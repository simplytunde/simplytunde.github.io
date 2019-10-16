---
layout: post
---

For most kubernetes clusters using EKS, there is  the fear of running out of IPs since each pod gets one IP address from the VPC. For large enterprise clusters, this is a problem. Now the question is, how do we solve this? We can leverage one of the feature of AWS VPC to use secondary IP range combined with customize AWS VPC CNI configuration.

### Secondary IPs

For thoser that are not aware, AWS released a feature back in [2017](https://aws.amazon.com/about-aws/whats-new/2017/08/amazon-virtual-private-cloud-vpc-now-allows-customers-to-expand-their-existing-vpcs/) which allows you to extend your VPC with secondary CIDRs. This means, let's say your primary CIDR is 10.2.0.0/16,  you now have the ability to use another secondary CIDR like 172.14.0.0/16 with the VPC also. We will be taking advantage of this to extend our EKS cluster combined with ability of [AWS VPC  CNI](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html) to use custom CNI config.

### The Cluster

We will be creating a VPC cluster that has primary CIDR `10.0.0.0/16`  and additional secondary CIDR `172.2.0.0/16`. In this VPC, we will create 3 private subnets,  two belonging to `10.0.0.0/16 ` while the other  to 172.2.0.0/16.

```yaml
resource "aws_vpc" "eks_vpc" {
  cidr_block = "10.0.0.0/16" 
  enable_dns_support = true //These configuration are needed for private EKS cluster
  enable_dns_hostnames = true
  tags = {
    "kubernetes.io/cluster/test-cluster" = "shared" 
  }
}
...
resource "aws_vpc_ipv4_cidr_block_association" "secondary_cidr" {
  vpc_id     = aws_vpc.eks_vpc.id
  cidr_block = "172.2.0.0/16"
}
...
resource "aws_subnet" "private_1" {
  vpc_id     = aws_vpc.eks_vpc.id
  availability_zone = "us-east-1a"
  cidr_block = "10.0.3.0/24"
  tags = {
    Name = "private_1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared" #We are adding this because EKS automatically does this anyway.
  }
}

resource "aws_subnet" "private_2" {
  vpc_id     = aws_vpc.eks_vpc.id
  availability_zone = "us-east-1b"
  cidr_block = "10.0.4.0/24"
  tags = {
    Name = "private_2"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared" 
  }
}

#This is the secondary CIDR subnet.
resource "aws_subnet" "private_3" {
  vpc_id     = aws_vpc.eks_vpc.id
  availability_zone = "us-east-1a" #The secondary subnet must be in the same AZ for AWS CNI to use its IPs.
  cidr_block = "172.2.3.0/24"
  tags = {
    Name = "private_3"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared" 
  }
}
```

Now that our VPC has been setup, lets go ahead and create our EKS cluster to launch  into `private_1` and `private_2` subnets both belonging to `10.0.0.0/16` CIDR. For our demo, we will be launching our workers node into one of the subnets in us-east-1a.

```yaml
resource "aws_eks_cluster" "test_cluster" {
  name     = var.cluster_name
  role_arn = aws_iam_role.cluster.arn

  vpc_config {
    subnet_ids = [aws_subnet.private_1.id, aws_subnet.private_2.id] #mininum of 2 is required
    security_group_ids = [aws_security_group.cluster.id]
  }
}
...
resource "aws_launch_template" "eks-cluster-worker-nodes" {
  iam_instance_profile        {
    arn  = aws_iam_instance_profile.workers-node.arn
  }
  image_id                    = data.aws_ami.eks-worker.id
  instance_type               = "t3.medium"
  key_name                    = "mykey.pem"
  vpc_security_group_ids      = [aws_security_group.workers-node.id]
  user_data                   = "${base64encode(local.workers-node-userdata)}"
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "eks-cluster-worker-nodes-spot" {
  ...
  mixed_instances_policy {
    ...
    launch_template {
      launch_template_specification {
        launch_template_id = "${aws_launch_template.eks-cluster-worker-nodes.id}" 
        version = "$Latest"
      }

      override {
        instance_type = "t3.medium"
      }
    }
  }
   ...
}
```

To connect the cluster, we will need our `awsauth` config, the `kubeconfig` as well as `ENIConfig` which informs AWS VPC CNI which subnet to use for a particular node. These will be generated from terraform

```yaml
locals {
  kubeconfig = <<KUBECONFIG
apiVersion: v1
clusters:
- cluster:
    server: ${aws_eks_cluster.test_cluster.endpoint}
    certificate-authority-data: ${aws_eks_cluster.test_cluster.certificate_authority.0.data}
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: aws
  name: aws
current-context: aws
kind: Config
preferences: {}
users:
- name: aws
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "${var.cluster_name}"
KUBECONFIG

  config_map_aws_auth = <<CONFIGMAPAWSAUTH
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: ${aws_iam_role.workers-node.arn}
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
CONFIGMAPAWSAUTH

  awsauth = <<AWSAUTH
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: ${aws_iam_role.workers-node.name}
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
AWSAUTH
}

resource "local_file" "kubeconfig" {
  content  = "${local.kubeconfig}"
  filename = "kubeconfig"
}

resource "local_file" "aws_auth" {
  content  = "${local.config_map_aws_auth}"
  filename = "awsauth.yaml"
}
resource "local_file" "eni-a" {
  content  = "${local.eni_a}"
  filename = "eni-${aws_subnet.private_1.availability_zone}.yaml"
}
...
```

### DEMO

Once terraform apply is complete, these files will be generated and should be applied following the procedures below;

- export KUBECONFIG to the generated `kubeconfig` file.

- Run kubectl apply -f `eni-us-east-1a.yaml` to create CRD which informs the CNI  which subnets to create workers pods on. Update the CNI daemonset.

  ```yaml
  kubectl apply -f eni-us-east-1a.yaml
  kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
  kubectl set env daemonset aws-node -n kube-system ENI_CONFIG_LABEL_DEF=failure-domain.beta.kubernetes.io/zone
  ```

- Run `kubectl apply -f awsauth.yaml`  for the workers node to be able to join the cluster.

- Once joined, you should see the pods scheduled on `172.2.3.0/24` subnet rather than the primary interface ENI.

```yaml
$ kubectl get pods -n kube-system -owide
NAME                       READY   STATUS    RESTARTS   AGE    IP            NODE                         NOMINATED NODE   READINESS GATES
aws-node-7tv4z             1/1     Running   0          2m     10.0.3.232    ip-10-0-3-232.ec2.internal   <none>           <none>
coredns-69bc49bfdd-s5t75   1/1     Running   0          3m9s   172.2.3.218   ip-10-0-3-232.ec2.internal   <none>           <none>
coredns-69bc49bfdd-wk48q   1/1     Running   0          3m9s   172.2.3.230   ip-10-0-3-232.ec2.internal   <none>           <none>
kube-proxy-fm564           1/1     Running   0          2m     10.0.3.232    ip-10-0-3-232.ec2.internal   <none>           <none>
```

What happened here is that `L-IPAMD` launches an ENI and instead of attaching secondary IPs from the primary ENI subnet, it uses the IPs from subnet specified in the ENIConfig associated with the node. Note that the subnet in the ENIConfig must be in the same AZ as the subnet of the primary ENI.
