---
layout: post
---
Terraform 0.12 was recently released and you can check out the details more in-dept at [terraform blogs](https://www.hashicorp.com/blog/announcing-terraform-0-12). The focus of this blog post will be  the new features released and how to use them. You can find the code used in this [Github repo](https://github.com/simplytunde/tutorial/tree/master/terraform). Below is the content of the ec2 main.tf.
```hcl
module "test_ec2" {
      source = "./ec2_module"
      //I am passing the subnet object instead of the id
      subnet = aws_subnet.private.0
      instance_tags = [
        {
          Key = "Name"
          Value = "Test"
        },
        {
          Key = "Environment"
          Value = "prd"
        }
    ]
}
```
Here is the content of the ec2 module;

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-trusty-14.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  subnet_id  = var.subnet.id
  user_data  = templatefile("${path.module}/userdata.tpl", { instance_tags = var.instance_tags })
  tags = {
     for tag in var.instance_tags:
       tag.Key => tag.Value
  }
}

variable "instance_type" {
     default = "t2.micro"
}

variable "instance_tags" {
     type = list
}

variable "subnet" {
    type = object({
        id = string
    })
}
```

### For expression

From my perspective, this is one of the best feature of 0.12 which allows you to iterate over a list or map and you can  do whatever you want with each item. From above, you can see that we generated the instance tags property from the user passed in  list of maps.

```yaml
  tags = {
     for tag in var.instance_tags:
       tag.Key => tag.Value
  }
```
### First class expression

You can see from above that we did not have to interpolate over the variables or object values instead they were used directly as first class variable. This will save some ink and allows for more complex operational usage with terraform.

```yaml
...
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  subnet_id  = var.subnet.id
...
```

### Rich Value Type

My impression of this feature so far is that we can have user-defined type and existing type are supported as first class type without quote. We defined a type that expects map with id attribute and if passing a non-matching object, it will be rejected which is pretty cool.

```yaml
variable "instance_tags" {
     type = list
}

variable "subnet" {
    type = object({
        id = string
    })
}
```

### New Type Interpolation

New terraform now gives you the capability to loop in your user data. For our example, here is the userdata;

```
#!/bin/bash

%{ for tag in instance_tags~}
cat ${tag.Key}=${tag.Value}
%{ endfor ~}
```

We were able to use forloop to go over the tags and use it in instance user data which is lovely without having to through hacks.

One other feature that was not discussed here is the *dynamic block* which you can checkout on terraform [blog page](https://www.hashicorp.com/blog/hashicorp-terraform-0-12-preview-for-and-for-each).

