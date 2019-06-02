---
layout: post
---
So you have a workload that is CPU senstive and you want to optimize things by providing better CPU performance to your workflow, CPU Manager can help. Now, howexactly does it help you; Before we can talk about this, lets try to understand CFS(Completely Fair Scheduler) slang;
### CFS Share
No, this is not like stock market share, we are talking CPU here. Think about a fixed time that everyone is trying to take a slice of. CPU Shares simply implies how much of system CPU time do you have access to.
* CPU Share: This determines your power when assigned to a CPU core under excess load. Lets say two processes(A and B) jumps on a CPU core and they both get allocated 1024 shares each(default allocation unless you change things), it means they both carry the same weight in terms of time allocation with each getting 1/2 CPU core time. Now, if we make things interesting and make process B share  updated to 512, it means B gets (512/(1024+512)) = 1/3 of the CPU  time. Now, one more thing to remember is that, if process A goes idle, process B can use some of that CPU time provided we only have A and B on the core.
* CPU Period: This is part of the CFS bandwidth control and it determines the what a period means to a CPU. What is a Period? Think of it as a time that represent a CPU cycle, usually, 100ms(100,000) for most system and it is expressed as ```cfs_period_us```.

* CPU Quota: A process with 20ms(20,000) quota will get 1/5 of time during a CPU period of 100ms. So quota is basically, how much of the time slice do you get to access? You see this variable expressed as ```cfs_quota_us```.

Ok, enough of the jargons, how does kubernetes translate a container with 100m(0.1 CPU) to shares and quota? You can see the answer below.


This kubernetes go code explains it all for those interested in  how kubernetes does all [these](https://github.com/kubernetes/kubernetes/blob/a21a7502496418d8ef26a5d1522f8f246110f5fc/pkg/kubelet/kuberuntime/helpers_linux.go#L33).
```go
// milliCPUToShares converts milliCPU to CPU shares
func milliCPUToShares(milliCPU int64) int64 {
	if milliCPU == 0 {
		// Return 2 here to really match kernel default for zero milliCPU.
		return minShares
	}
	// Conceptually (milliCPU / milliCPUToCPU) * sharesPerCPU, but factored to improve rounding.
	shares := (milliCPU * sharesPerCPU) / milliCPUToCPU
        // for example, share := (100m/1024) * 1000 = (100/1000) * 1024 = 102.4 shares
	if shares < minShares {
		return minShares
	}
	return shares
}

// milliCPUToQuota converts milliCPU to CFS quota and period values
func milliCPUToQuota(milliCPU int64) (quota int64, period int64) {
	// CFS quota is measured in two values:
	//  - cfs_period_us=100ms (the amount of time to measure usage across)
	//  - cfs_quota=20ms (the amount of cpu time allowed to be used across a period)
	// so in the above example, you are limited to 20% of a single CPU
	// for multi-cpu environments, you just scale equivalent amounts
	if milliCPU == 0 {
		return
	}

	// we set the period to 100ms by default
	period = quotaPeriod

	// we then convert your milliCPU to a value normalized over a period
	quota = (milliCPU * quotaPeriod) / milliCPUToCPU


	// quota needs to be a minimum of 1ms.
	if quota < minQuotaPeriod {
		quota = minQuotaPeriod
	}

	return
```
##### CPU Manager and Scheduling
Nice that we have gone through all these terms. The question remains, how does CPU manager work to help with my CPU sensitive workload. so basically, it uses cpuset feature in linux to place containers on specific cpu. It takes slice of cpu, equals to specified requests(or limit) in the container, separates it and assign it to your container thereby preventing context switching and noisy-neighbour issue. Let's look under the cover; it creates different  pool of memory as shown below
+ Shared Memory Pool: This is the pool of memory that every scheduled container gets assigned to until decision is made to move them elsewhere.
+ Reserved Pool: Remember that your kubelet can reserve cpu right? Yes, those are these guys. Simply, cpu that you can not touch in the shared pool.
+ Assignable: This is where containers with that meets exclusivity requirement get their CPU from. They are taking from remaining CPU left after removing the reserved pool for kubelet. Once they are assigned to a container, they get removed from the shared pool.
+ Exclusive Allocations; This pool contains those cpuset assigned to containers.

The next question, who qualified for Assignable Pool? Any Guaranteed container(request = limit) with integer CPU. Yes, integer! Containers like the one below;
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      limits:
        cpu: 1
        memory: "200Mi" 
      requests:
        cpu: 1
        memory: "200Mi"
    command: ["stress"]
```
Ok, I mentioned everyone gets assigned to shared pool at first. What moves them to exclusive pool? Well, kubelet does what we call resync(configurable kubelet option) by checking the containers in the shared pool every certain period and move those who qualified to exclusive pool. That means, your pod could be in shared pool until next resync. Also, please note it is possible that kubelet or system process will be running on the exclusive CPU set because manager only guarantees exclusivity for pods.Other processes in the system, thats a not kubelet business.
###Show me the money
Enough theory, let's get dirty. How do we enable this feature on our kubelet? Just enable feature-gate CPUManager and pass in static policy. I used this in my test kubeadm setup
```yaml
kind: KubeletConfiguration
featureGates:
  CPUManager: true
cpuManagerPolicy: static
systemReserved:
  cpu: 500m
  memory: 256M
kubeReserved:
  cpu: 500m
  memory: 256M
```
Lets go ahead and create this pod for example;
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    resources:
      requests:
        memory: "24Mi"
        cpu: "150m"
      limits:
        memory: "28Mi"
        cpu: "160m"
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```
After creating the pod on the cluster with CPU Manager feature gate enabled, it get scheduled onto node. At this point, it is a  burstable pod with 153(150/1000 * 1024) share of CPU and 16000 CPU Quota(160/1000 * 100,000). You can confirm this by looking the container cgroup. 
```bash
cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/podf19e6b4b-6eb0-11e9-898e-062ad3dc4fe4/138208e13ba
73882fc0a5c06862b7b0bc7f6d3f43116d61ecf2488fae11d6004/cpu.shares 
153
cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/podf19e6b4b-6eb0-11e9-898e-062ad3dc4fe4/138208e13ba
73882fc0a5c06862b7b0bc7f6d3f43116d61ecf2488fae11d6004/cpu.cfs_quota_us 
16000
``` 
The above pod is a burstable pod but to test CPU Manager, we need a guaranteed pod with **whole number** CPU. Once the pod is scheduled, the kubelet should configure our container runtime to run the pod on particular core(s) using cpuset.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    resources:
      requests:
        memory: "38Mi"
        cpu: "1"
      limits:
        memory: "38Mi"
        cpu: "1"
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```
We can see this in the container docker inspect as well as the cgroup.
```bash
#  docker inspect 59964c06d765 | grep -i cpu
            "CpuShares": 1024,
            "NanoCpus": 0,
            "CpuPeriod": 100000,
            "CpuQuota": 100000,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "1",
            "CpusetMems": "",
            "CpuCount": 0,
            "CpuPercent": 0,
# cat /sys/fs/cgroup/cpuset/kubepods/pod273e9d00-706c-11e9-a529-062ad3dc4fe4/59964c06d7657face0585c9db37
5d8773dcb1b351a2d7e87204e89a2e47c2b97/cpuset.effective_cpus
1
```
If you look at other burstable containers, they will be restricted to  shared cores.
```bash
# docker inspect 6ca5ea998f0d | grep -i cpu
            "CpuShares": 153,
            "NanoCpus": 0,
            "CpuPeriod": 100000,
            "CpuQuota": 16000,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "0,2-3",
            "CpusetMems": "",
            "CpuCount": 0,
            "CpuPercent": 0,
# cat /sys/fs/cgroup/cpuset/kubepods/burstable/podd024aac6-706b-11e9-a529-062ad3dc4fe4/539b9b1c4ec3e49fa
49b84b22ff4a06058a1c0bd6db57667cb30786927d3a380/cpuset.cpus
0,2-3
```
### Helpful Links
* [https://stupefied-goodall-e282f7.netlify.com/contributors/design-proposals/node/cpu-manager/](https://stupefied-goodall-e282f7.netlify.com/contributors/design-proposals/node/cpu-manager/)
* [https://goldmann.pl/blog/2014/09/11/resource-management-in-docker/#_cpu](https://goldmann.pl/blog/2014/09/11/resource-management-in-docker/#_cpu)
* [https://software.intel.com/en-us/blogs/2018/08/07/cpu-manager-for-performance-sensitive-applications](https://software.intel.com/en-us/blogs/2018/08/07/cpu-manager-for-performance-sensitive-applications)
* [https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)
* [https://medium.com/@mcastelino/kubernetes-resource-management-deep-dive-b337ba15359c](https://medium.com/@mcastelino/kubernetes-resource-management-deep-dive-b337ba15359c)
