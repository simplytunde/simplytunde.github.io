---
layout: post
---
## Logs

EKS is the managed kubernetes offering by AWS that saves you the stress of managing your own control plane with a twist of offboarding some controls like what goes on in your control. The feature was not available when the service went GA but was recently made available recently. Here are the kinds of logs that it provides;

+ API server component logs: You know that component of your cluster that validates requests, provides api rest endpoint and so on? These are the logs from the apiserver which are very critical when trying to diagnose things like why your pods are not creating, admission controller issues etc.
```yaml
E0523 03:27:22.258958 1 memcache.go:134] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
```

+ Audit Logs: People make changes in your cluster and you want to know who, what and when. This logs gives you the ability to this.
```yaml
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1beta1",
    "metadata": {
        "creationTimestamp": "2019-05-23T02:08:34Z"
    },
    "level": "Request",
    "timestamp": "2019-05-23T02:08:34Z",
    "auditID": "84662c40-8d4f-4d3e-99b2-0d4005e44375",
    "stage": "ResponseComplete",
    "requestURI": "/api/v1/namespaces/default/services/kubernetes",
    "verb": "get",
    "user": {
        "username": "system:apiserver",
        "uid": "2d8ad7ed-25ed-4f37-a2f0-416d2af705e9",
        "groups": [
            "system:masters"
        ]
    },
    "sourceIPs": [
        "::1"
    ],
    "userAgent": "kube-apiserver/v1.12.6 (linux/amd64) kubernetes/d69f1bf",
    "objectRef": {
        "resource": "services",
        "namespace": "default",
        "name": "kubernetes",
        "apiVersion": "v1"
    },
    "responseStatus": {
        "metadata": {},
        "code": 200
    },
    "requestReceivedTimestamp": "2019-05-23T02:08:34.498973Z",
    "stageTimestamp": "2019-05-23T02:08:34.501446Z",
    "annotations": {
        "authorization.k8s.io/decision": "allow",
        "authorization.k8s.io/reason": ""
    }
}

```
+ Authenticator Logs: EKS uses this thing called aws-iam-authenticator to guess what? Authenticate against the EKS cluster using AWS credentials and roles. These logs contains event from these activities
```yaml
time="2019-05-16T22:19:48Z" level=info msg="Using assumed role for EC2 API" roleARN="arn:aws:iam::523447765480:role/idaas-kubernetes-cluster-idauto-dev-masters-role"
```
+ Controller manager: For those familiar with kubernetes objects such as Deployments, Replicas etc; these are managed by controllers which ships with kubernetes controller manager. To see what these controllers are doing under the hood, you need these.
```yaml
E0523 02:07:55.486872 1 horizontal.go:212] failed to compute desired number of replicas based on listed metrics for Deployment/routing/rapididentity-default-backend: failed to get memory utilization: unable to get metrics for resource memory: unable to fetch metrics from resource metrics API: the server is currently unable to handle the request (get pods.metrics.k8s.io)
```
+ Scheduler: This component of the control plane does what it name says, put pods on the right node after factoring a number of constraints and resources available. To see information on how this component is making its decision, check these logs.

```yaml
E0523 02:07:55.486872 1 horizontal.go:212] failed to compute desired number of replicas based on listed metrics for Deployment/routing/rapididentity-default-backend: failed to get memory utilization: unable to get metrics for resource memory: unable to fetch metrics from resource metrics API: the server is currently unable to handle the request (get pods.metrics.k8s.io)
```

# Enabling Logs

You can easily enable the logs in your EKS cluser console and AWS updates your cluster to enable those logs to ship to cloudwatch. The corresponding cloudwatch log group will be displayed in your console. For those using terraform to provision their cluster, you can just pass in the types of logs that you want to provision and also create the log group to ship it to.

```yaml
resource "aws_eks_cluster" "my_cluster" {
  depends_on = ["aws_cloudwatch_log_group.eks_log_group"]
  enabled_cluster_log_types = ["api", "audit"]
  name                      = "${var.cluster_name}"
  # ... other configuration ...
}
```
