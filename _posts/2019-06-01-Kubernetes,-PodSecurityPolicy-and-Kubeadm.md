---
layout: post
---
I will assume you have a little background about PodSecurityPolicy and now its time to setup your cluster with PodSecurityPolicy admission controller enabled. For our setup, here is the kubeadm config;
```yaml
kind: InitConfiguration
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
featureGates:
  AppArmor: true
cpuManagerPolicy: static
systemReserved:
  cpu: 500m
  memory: 256M
kubeReserved:
  cpu: 500m
  memory: 256M
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
apiServer:
  extraArgs:
    enable-admission-plugins:  PodSecurityPolicy,LimitRanger,ResourceQuota,AlwaysPullImages,DefaultStorageClass
```
Let's start the cluster, 
```yaml
$ sudo kubeadm init --config kubeadm.json 
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#join a worker node to the master
$ sudo kubeadm join 10.100.11.231:6443 --token 6yel1a.ce3le6eel3kfnxsz --discovery-token-ca-cert-hash sha256:99ee2e4ea
302c5270f2047c7a0093533b69105a8c91bf20f48b230dce9fd3f3a
$ kubectl get no
NAME               STATUS     ROLES    AGE    VERSION
ip-10-100-11-199   NotReady   <none>   109s   v1.13.1
ip-10-100-11-231   NotReady   master   3m3s   v1.13.1
```
As you can see, our cluster is not ready because we need to install a network plugin which you can see if you describe the node.
```bash
$ kubectl describe no ip-10-100-11-231
...
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Fri, 31 May 2019 22:50:36 +0000   Fri, 31 May 2019 22:46:39 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Fri, 31 May 2019 22:50:36 +0000   Fri, 31 May 2019 22:46:39 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Fri, 31 May 2019 22:50:36 +0000   Fri, 31 May 2019 22:46:39 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Fri, 31 May 2019 22:50:36 +0000   Fri, 31 May 2019 22:46:39 +0000   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
Addresses:
...
```
We will be using calico as our network plugin and you can follow the install instruction right [here](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
```bash
$ kubectl apply -f https://docs.projectcalico.org/v3.7/manifests/calico.yaml
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.extensions/calico-node created
serviceaccount/calico-node created
deployment.extensions/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```
To our surprise, there is no pod running in kube-system as we would expect. Let look at calico-node daemonset;
```yaml
$ kubectl describe daemonset calico-node  -n kube-system
Name:           calico-node
Selector:       k8s-app=calico-node
Node-Selector:  beta.kubernetes.io/os=linux
Labels:         k8s-app=calico-node
...
Events:
  Type     Reason        Age                  From                  Message
  ----     ------        ----                 ----                  -------
  Warning  FailedCreate  20s (x15 over 102s)  daemonset-controller  Error creating: pods "calico-node-" is forbidden: no providers available to validate pod request
```
Ok, it says no provider and that means you do not have any PSP defined that will validate the daemonset pods. It is a very confusing error message.
Now, we will apply this PSP below.
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: calico-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: false
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  hostIPC: true
  hostPID: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```
```bash
$ kubectl apply -f calico-psp.yaml 
podsecuritypolicy.policy/calico-psp created
$ kubectl get psp
NAME             PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
calico-psp   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *
```
Ok everything should be good now? Well, if you describe the daemonset, you will still see the same message as above. You need to delete and re-apply the calico plugin manifest. After this, let's go ahead and describe our daemonset again.

```yaml
$ kubectl describe daemonset calico-node  -n kube-system
Name:           calico-node
Selector:       k8s-app=calico-node
Node-Selector:  beta.kubernetes.io/os=linux
Labels:         k8s-app=calico-node
...
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/cni/networks
    HostPathType:  
Events:
  Type     Reason        Age                From                  Message
  ----     ------        ----               ----                  -------
  Warning  FailedCreate  4s (x12 over 14s)  daemonset-controller  Error creating: pods "calico-node-" is forbidden: unable to validate against any pod security policy: []
```
The error means we have a psp but not something we can use to validate our daemonset. This is because our daemonset pod container cannot use our PSP and we need to modify the daemonset clusterrole to be able to use this specific PSP and add this rule. Yep, you need to recreate.
```yaml
# Include a clusterrole for the calico-node DaemonSet,
# and bind it to the calico-node serviceaccount.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-node
rules:
  - apiGroups: ["extensions"]
    resources:
       - podsecuritypolicies
    resourceNames:
       - calico-psp
    verbs:
       - use
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
---
...
```
After the clusterrole has been modified, we will run into another error;
```yaml
...
   host-local-net-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/cni/networks
    HostPathType:  
Events:
  Type     Reason        Age               From                  Message
  ----     ------        ----              ----                  -------
  Warning  FailedCreate  3s (x11 over 9s)  daemonset-controller  Error creating: pods "calico-node-" is forbidden: unable to validate against any pod security policy: [spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]
...
```
Our calico pods are forbidden because the PSP it is using forbids privileged container and Calico needs one to handle network policy. Now, lets go ahead and fix that in our PSP by updating;
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: calico-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true #update this to true
  allowPrivilegeEscalation: true
...
```
Once the update has been applied, calico will create the pods and our node will become healthy.
```yaml
...
Events:
  Type     Reason            Age                     From                  Message
  ----     ------            ----                    ----                  -------
  Warning  FailedCreate      3m10s (x16 over 5m54s)  daemonset-controller  Error creating: pods "calico-node-" is forbidden: unable to validate against any pod security policy: [spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]
  Normal   SuccessfulCreate  26s                     daemonset-controller  Created pod: calico-node-hwb2b
  Normal   SuccessfulCreate  26s                     daemonset-controller  Created pod: calico-node-gtrm2

$ kubectl get no
NAME               STATUS   ROLES    AGE     VERSION
ip-10-100-11-199   Ready    <none>   7m2s    v1.13.1
ip-10-100-11-231   Ready    master   7m33s   v1.13.1
```

Yayy!!! Our nodes are ready.
