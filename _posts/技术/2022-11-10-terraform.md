---
layout: post
title: terraform探索和实践
category: 技术
---

# 相关概念

## why terraform

- Provisioning tool—Deploys infrastructure, not just applications.
- Easy to use—For all of us non-geniuses.
- Free and open source—Who doesn’t like free?
- Declarative—Say what you want, not how to do it.
- Cloud-agnostic—Deploy to any cloud using the same tool.
- Expressive and extendable—You aren’t limited by the language.

# terminlogy

## HCL language.

## terraform registry

“The Terraform Registry is a global store for sharing versioned provider binaries. When Terraform initializes, it automatically looks up and downloads any required providers from the registry.”

# syntax

## tf 之间关系

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled.png)

output， providers， variables，可以单独拆分出来 放在单独的文件。

所以terraform 执行时，会将 当前目录内所有 tf 文件，合并为一个大的的文件。

## summary

input 用来做配置的输入变量

local 用来保存表达式的值

output 用来在module 间传递变量，也可以返回给 user

## provisioner

attach在 resource上的语法，

一般用于 在 资源创建，或者资源销毁时，提供的一个 hook， 用来在 local 或者 remote 机器的OS上执行cmd

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%201.png)

## module

类似于 ansible的 roles ，功能reuse的单位。

module可以在本地，也可以在 远程

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%202.png)

```jsx
“Complex projects, such as multi-tiered web applications in AWS, are easy to design and deploy with the help of Terraform modules.
    
       The root module is the main entry point for your project. You configure variables at the root level by using a variables definition file (terraform.tfvars). These variables are then trickled down as necessary into child modules.
    

    
      Nested modules organize code into child modules. Child modules can be nested within other child modules without limit. Generally, you don’t want your module hierarchy to be more than three or four levels deep, because it makes it harder to understand.
    

    
      Many people have published modules in the public Terraform Registry. You can save a lot of time by using these open source modules instead of writing comparable code yourself; all it takes is learning how to use the module interface.
    

    
      Data is passed between modules using bubble-up and trickle-down techniques. Since this can result in a lot of boilerplate, it’s a good idea to optimize your code so that minimal data needs to be passed between modules.”

Excerpt From
Terraform in Action
Scott Winkler
This material may be protected by copyright.
```

## output

只是将 vpcid print ？  是的

另外，会用来在 module 之前传递变量。

```jsx
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

## variable

local vs variable ??  tf-pool 看到用 local 

local 内部使用的变量，一般用来保存复杂的表达式的值，方便多次调用。

[https://www.youtube.com/watch?v=Yo06KIsKKN8&t=10s](https://www.youtube.com/watch?v=Yo06KIsKKN8&t=10s)

## local

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%203.png)

## for

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%204.png)

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%205.png)

## resource vs data

resource 和 data 都是对 云厂商资源的封装（比如 ec2，vpc，s3，iam等资源），

data的作用在于 可以动态获取 资源的属性，然后用于 后续terraform的其他语句中，只有下图的read接口。

resource 提供的接口更丰富些。

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%206.png)

## data block

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%207.png)

## resource block

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%208.png)

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%209.png)

# 场景操作

## 创建aws ec2

code

```jsx
provider "aws" {
  region  = "us-west-2"
}

resource "aws_instance" "helloworld" {
  ami           = "ami-09dd2e08d601bff67"
  instance_type = "t2.micro"
  tags = {
    Name = "HelloWorld"
  }
}
```

- init
    
    ```jsx
    ➜  aws git:(main) ✗ terraform init
    
    Initializing the backend...
    
    Initializing provider plugins...
    - Finding latest version of hashicorp/aws...
    - Installing hashicorp/aws v4.46.0...
    - Installed hashicorp/aws v4.46.0 (signed by HashiCorp)
    ```
    
- plan
    
    ```jsx
    aws git:(main) ✗ terraform plan
    
    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
      + create
    
    Terraform will perform the following actions:
    
      # aws_instance.helloworld will be created
      + resource "aws_instance" "helloworld" {
          + ami                                  = "ami-09dd2e08d601bff67"
          + arn                                  = (known after apply)
          + associate_public_ip_address          = (known after apply)
          + availability_zone                    = (known after apply)
          + cpu_core_count                       = (known after apply)
          + cpu_threads_per_core                 = (known after apply)
          + disable_api_stop                     = (known after apply)
          + disable_api_termination              = (known after apply)
          + ebs_optimized                        = (known after apply)
          + get_password_data                    = false
          + host_id                              = (known after apply)
          + host_resource_group_arn              = (known after apply)
          + iam_instance_profile                 = (known after apply)
          + id                                   = (known after apply)
          + instance_initiated_shutdown_behavior = (known after apply)
          + instance_state                       = (known after apply)
          + instance_type                        = "t2.micro"
          + ipv6_address_count                   = (known after apply)
          + ipv6_addresses                       = (known after apply)
          + key_name                             = (known after apply)
          + monitoring                           = (known after apply)
          + outpost_arn                          = (known after apply)
          + password_data                        = (known after apply)
          + placement_group                      = (known after apply)
          + placement_partition_number           = (known after apply)
          + primary_network_interface_id         = (known after apply)
          + private_dns                          = (known after apply)
          + private_ip                           = (known after apply)
          + public_dns                           = (known after apply)
          + public_ip                            = (known after apply)
          + secondary_private_ips                = (known after apply)
          + security_groups                      = (known after apply)
          + source_dest_check                    = true
          + subnet_id                            = (known after apply)
          + tags                                 = {
              + "Name" = "HelloWorld"
            }
          + tags_all                             = {
              + "Name" = "HelloWorld"
            }
          + tenancy                              = (known after apply)
          + user_data                            = (known after apply)
          + user_data_base64                     = (known after apply)
          + user_data_replace_on_change          = false
          + vpc_security_group_ids               = (known after apply)
    
          + capacity_reservation_specification {
              + capacity_reservation_preference = (known after apply)
    
              + capacity_reservation_target {
                  + capacity_reservation_id                 = (known after apply)
                  + capacity_reservation_resource_group_arn = (known after apply)
                }
            }
    
          + ebs_block_device {
              + delete_on_termination = (known after apply)
              + device_name           = (known after apply)
              + encrypted             = (known after apply)
              + iops                  = (known after apply)
              + kms_key_id            = (known after apply)
              + snapshot_id           = (known after apply)
              + tags                  = (known after apply)
              + throughput            = (known after apply)
              + volume_id             = (known after apply)
              + volume_size           = (known after apply)
              + volume_type           = (known after apply)
            }
    
          + enclave_options {
              + enabled = (known after apply)
            }
    
          + ephemeral_block_device {
              + device_name  = (known after apply)
              + no_device    = (known after apply)
              + virtual_name = (known after apply)
            }
    
          + maintenance_options {
              + auto_recovery = (known after apply)
            }
    
          + metadata_options {
              + http_endpoint               = (known after apply)
              + http_put_response_hop_limit = (known after apply)
              + http_tokens                 = (known after apply)
              + instance_metadata_tags      = (known after apply)
            }
    
          + network_interface {
              + delete_on_termination = (known after apply)
              + device_index          = (known after apply)
              + network_card_index    = (known after apply)
              + network_interface_id  = (known after apply)
            }
    
          + private_dns_name_options {
              + enable_resource_name_dns_a_record    = (known after apply)
              + enable_resource_name_dns_aaaa_record = (known after apply)
              + hostname_type                        = (known after apply)
            }
    
          + root_block_device {
              + delete_on_termination = (known after apply)
              + device_name           = (known after apply)
              + encrypted             = (known after apply)
              + iops                  = (known after apply)
              + kms_key_id            = (known after apply)
              + tags                  = (known after apply)
              + throughput            = (known after apply)
              + volume_id             = (known after apply)
              + volume_size           = (known after apply)
              + volume_type           = (known after apply)
            }
        }
    
    Plan: 1 to add, 0 to change, 0 to destroy.
    ```
    
- apply
    
    ```jsx
    
    + network_interface {
              + delete_on_termination = (known after apply)
              + device_index          = (known after apply)
              + network_card_index    = (known after apply)
              + network_interface_id  = (known after apply)
            }
    
          + private_dns_name_options {
              + enable_resource_name_dns_a_record    = (known after apply)
              + enable_resource_name_dns_aaaa_record = (known after apply)
              + hostname_type                        = (known after apply)
            }
    
          + root_block_device {
              + delete_on_termination = (known after apply)
              + device_name           = (known after apply)
              + encrypted             = (known after apply)
              + iops                  = (known after apply)
              + kms_key_id            = (known after apply)
              + tags                  = (known after apply)
              + throughput            = (known after apply)
              + volume_id             = (known after apply)
              + volume_size           = (known after apply)
              + volume_type           = (known after apply)
            }
        }
    
    Plan: 1 to add, 0 to change, 0 to destroy.
    
    Do you want to perform these actions?
      Terraform will perform the actions described above.
      Only 'yes' will be accepted to approve.
    
      Enter a value: yes
    
    aws_instance.helloworld: Creating...
    aws_instance.helloworld: Still creating... [10s elapsed]
    aws_instance.helloworld: Still creating... [20s elapsed]
    aws_instance.helloworld: Still creating... [30s elapsed]
    aws_instance.helloworld: Creation complete after 36s [id=i-0d8aeabe82c6d4ee6]
    ```
    
- web console check
    
    ![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%2010.png)
    
- show
    
    ```jsx
    ➜  aws git:(main) ✗ terraform show   
    # aws_instance.helloworld:
    resource "aws_instance" "helloworld" {
        ami                                  = "ami-09dd2e08d601bff67"
        arn                                  = "arn:aws:ec2:us-west-2:359990655175:instance/i-0d8aeabe82c6d4ee6"
        associate_public_ip_address          = true
        availability_zone                    = "us-west-2a"
        cpu_core_count                       = 1
        cpu_threads_per_core                 = 1
        disable_api_stop                     = false
        disable_api_termination              = false
        ebs_optimized                        = false
        get_password_data                    = false
        hibernation                          = false
        id                                   = "i-0d8aeabe82c6d4ee6"
        instance_initiated_shutdown_behavior = "stop"
        instance_state                       = "running"
        instance_type                        = "t2.micro"
        ipv6_address_count                   = 0
        ipv6_addresses                       = []
        monitoring                           = false
        primary_network_interface_id         = "eni-05b14f8292506d2f8"
        private_dns                          = "ip-172-31-25-192.us-west-2.compute.internal"
        private_ip                           = "172.31.25.192"
        public_dns                           = "ec2-35-91-166-192.us-west-2.compute.amazonaws.com"
        public_ip                            = "35.91.166.192"
        secondary_private_ips                = []
        security_groups                      = [
            "default",
        ]
        source_dest_check                    = true
        subnet_id                            = "subnet-0d2bdcaa3fec24bf5"
        tags                                 = {
            "Name" = "HelloWorld"
        }
        tags_all                             = {
            "Name" = "HelloWorld"
        }
        tenancy                              = "default"
        user_data_replace_on_change          = false
        vpc_security_group_ids               = [
            "sg-0a1626f32b5dff096",
        ]
    
        capacity_reservation_specification {
            capacity_reservation_preference = "open"
        }
    
        credit_specification {
            cpu_credits = "standard"
        }
    
        enclave_options {
            enabled = false
        }
    
        maintenance_options {
            auto_recovery = "default"
        }
    
        metadata_options {
            http_endpoint               = "enabled"
            http_put_response_hop_limit = 1
            http_tokens                 = "optional"
            instance_metadata_tags      = "disabled"
        }
    
        private_dns_name_options {
            enable_resource_name_dns_a_record    = false
            enable_resource_name_dns_aaaa_record = false
            hostname_type                        = "ip-name"
        }
    
        root_block_device {
            delete_on_termination = true
            device_name           = "/dev/sda1"
            encrypted             = false
            iops                  = 100
            tags                  = {}
            throughput            = 0
            volume_id             = "vol-0100d95392bdd22b1"
            volume_size           = 8
            volume_type           = "gp2"
        }
    }
    ➜  aws git:(main) ✗
    ```
    
- destroy
    
    ```jsx
    ➜  aws git:(main) ✗ terraform destroy
    aws_instance.helloworld: Refreshing state... [id=i-0d8aeabe82c6d4ee6]
    
    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
      - destroy
    
    Terraform will perform the following actions:
    
      # aws_instance.helloworld will be destroyed
      - resource "aws_instance" "helloworld" {
          - ami                                  = "ami-09dd2e08d601bff67" -> null
          - arn                                  = "arn:aws:ec2:us-west-2:359990655175:instance/i-0d8aeabe82c6d4ee6" -> null
          - associate_public_ip_address          = true -> null
          - availability_zone                    = "us-west-2a" -> null
          - cpu_core_count                       = 1 -> null
          - cpu_threads_per_core                 = 1 -> null
          - disable_api_stop                     = false -> null
          - disable_api_termination              = false -> null
          - ebs_optimized                        = false -> null
          - get_password_data                    = false -> null
          - hibernation                          = false -> null
          - id                                   = "i-0d8aeabe82c6d4ee6" -> null
          - instance_initiated_shutdown_behavior = "stop" -> null
          - instance_state                       = "running" -> null
          - instance_type                        = "t2.micro" -> null
          - ipv6_address_count                   = 0 -> null
          - ipv6_addresses                       = [] -> null
          - monitoring                           = false -> null
          - primary_network_interface_id         = "eni-05b14f8292506d2f8" -> null
          - private_dns                          = "ip-172-31-25-192.us-west-2.compute.internal" -> null
          - private_ip                           = "172.31.25.192" -> null
          - public_dns                           = "ec2-35-91-166-192.us-west-2.compute.amazonaws.com" -> null
          - public_ip                            = "35.91.166.192" -> null
          - secondary_private_ips                = [] -> null
          - security_groups                      = [
              - "default",
            ] -> null
          - source_dest_check                    = true -> null
          - subnet_id                            = "subnet-0d2bdcaa3fec24bf5" -> null
          - tags                                 = {
              - "Name" = "HelloWorld"
            } -> null
          - tags_all                             = {
              - "Name" = "HelloWorld"
            } -> null
          - tenancy                              = "default" -> null
          - user_data_replace_on_change          = false -> null
          - vpc_security_group_ids               = [
              - "sg-0a1626f32b5dff096",
            ] -> null
    
          - capacity_reservation_specification {
              - capacity_reservation_preference = "open" -> null
            }
    
          - credit_specification {
              - cpu_credits = "standard" -> null
            }
    
          - enclave_options {
              - enabled = false -> null
            }
    
          - maintenance_options {
              - auto_recovery = "default" -> null
            }
    
          - metadata_options {
              - http_endpoint               = "enabled" -> null
              - http_put_response_hop_limit = 1 -> null
              - http_tokens                 = "optional" -> null
              - instance_metadata_tags      = "disabled" -> null
            }
    
          - private_dns_name_options {
              - enable_resource_name_dns_a_record    = false -> null
              - enable_resource_name_dns_aaaa_record = false -> null
              - hostname_type                        = "ip-name" -> null
            }
    
          - root_block_device {
              - delete_on_termination = true -> null
              - device_name           = "/dev/sda1" -> null
              - encrypted             = false -> null
              - iops                  = 100 -> null
              - tags                  = {} -> null
              - throughput            = 0 -> null
              - volume_id             = "vol-0100d95392bdd22b1" -> null
              - volume_size           = 8 -> null
              - volume_type           = "gp2" -> null
            }
        }
    
    Plan: 0 to add, 0 to change, 1 to destroy.
    
    Do you really want to destroy all resources?
      Terraform will destroy all your managed infrastructure, as shown above.
      There is no undo. Only 'yes' will be accepted to confirm.
    
      Enter a value: yes
    
    Enter a value: yes
    
    aws_instance.helloworld: Destroying... [id=i-0d8aeabe82c6d4ee6]
    aws_instance.helloworld: Still destroying... [id=i-0d8aeabe82c6d4ee6, 10s elapsed]
    aws_instance.helloworld: Still destroying... [id=i-0d8aeabe82c6d4ee6, 20s elapsed]
    aws_instance.helloworld: Still destroying... [id=i-0d8aeabe82c6d4ee6, 30s elapsed]
    aws_instance.helloworld: Destruction complete after 32s
    
    Destroy complete! Resources: 1 destroyed.
    ➜  aws git:(main) ✗ terraform show   
    
    ➜  aws git:(main) ✗
    ```
    
    <aside>
    💡 terraform show   return null mean destroy finish
    
    </aside>
    
    also can check from web console if ec2 is deleted
    
    ![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%2011.png)
    

## ami to ec2

code

```jsx
provider "aws" {
  region = "us-west-2"
}

data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  owners = ["099720109477"]
}

resource "aws_instance" "helloworld" {
  ami           = data.aws_ami.ubuntu.id  ##使用动态变量
  instance_type = "t2.micro"
  tags = {
    Name = "HelloWorld"
  }
}
```

- ami check
    
    ![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%2012.png)
    

- plan
    
    data 在plan时，就已经得到值了。
    
    ```jsx
    ➜  ami-to-ec2 git:(main) ✗ terraform plan   
    data.aws_ami.ubuntu: Reading...
    data.aws_ami.ubuntu: Read complete after 2s [id=ami-0d31d7c9fc9503726]
    
    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
      + create
    
    Terraform will perform the following actions:
    
      # aws_instance.helloworld will be created
      + resource "aws_instance" "helloworld" {
          + ami                                  = "ami-0d31d7c9fc9503726"
          + arn                                  = (known after apply)
          + associate_public_ip_address          = (known after apply)
          + availability_zone                    = (known after apply)
          + cpu_core_count                       = (known after apply)
          + cpu_threads_per_core                 = (known after apply)
          + disable_api_stop                     = (known after apply)
          + disable_api_termination              = (known after apply)
          + ebs_optimized                        = (known after apply)
          + get_password_data                    = false
          + host_id                              = (known after apply)
          + host_resource_group_arn              = (known after apply)
          + iam_instance_profile                 = (known after apply)
          + id                                   = (known after apply)
          + instance_initiated_shutdown_behavior = (known after apply)
          + instance_state                       = (known after apply)
          + instance_type                        = "t2.micro"
          + ipv6_address_count                   = (known after apply)
          + ipv6_addresses                       = (known after apply)
          + key_name                             = (known after apply)
          + monitoring                           = (known after apply)
          + outpost_arn                          = (known after apply)
          + password_data                        = (known after apply)
          + placement_group                      = (known after apply)
          + placement_partition_number           = (known after apply)
          + primary_network_interface_id         = (known after apply)
          + private_dns                          = (known after apply)
          + private_ip                           = (known after apply)
          + public_dns                           = (known after apply)
          + public_ip                            = (known after apply)
          + secondary_private_ips                = (known after apply)
          + security_groups                      = (known after apply)
          + source_dest_check                    = true
          + subnet_id                            = (known after apply)
          + tags                                 = {
              + "Name" = "HelloWorld"
            }
          + tags_all                             = {
              + "Name" = "HelloWorld"
            }
          + tenancy                              = (known after apply)
          + user_data                            = (known after apply)
          + user_data_base64                     = (known after apply)
          + user_data_replace_on_change          = false
          + vpc_security_group_ids               = (known after apply)
    
          + capacity_reservation_specification {
              + capacity_reservation_preference = (known after apply)
    
              + capacity_reservation_target {
                  + capacity_reservation_id                 = (known after apply)
                  + capacity_reservation_resource_group_arn = (known after apply)
                }
            }
    
          + ebs_block_device {
              + delete_on_termination = (known after apply)
              + device_name           = (known after apply)
              + encrypted             = (known after apply)
              + iops                  = (known after apply)
              + kms_key_id            = (known after apply)
              + snapshot_id           = (known after apply)
              + tags                  = (known after apply)
              + throughput            = (known after apply)
              + volume_id             = (known after apply)
              + volume_size           = (known after apply)
              + volume_type           = (known after apply)
            }
    
          + enclave_options {
              + enabled = (known after apply)
            }
    
          + ephemeral_block_device {
    ```
    

## local file

```jsx
➜  local-file git:(main) ✗ terraform show    
# local_file.literature:
resource "local_file" "literature" {
    content              = <<-EOT
        Sun Tzu said: The art of war is of vital importance to the State.
        It is a matter of life and death, a road either to safety or to 
        ruin. Hence it is a subject of inquiry which can on no account be
        neglected.
    EOT
    directory_permission = "0777"
    file_permission      = "0777"
    filename             = "art_of_war.txt"
    id                   = "3c2d5b372ad55dc2902bdd2836efb6fc8b914e87"
}
```

after change file content,  will recreate infra ,id has changed

```jsx
# local_file.literature:
resource "local_file" "literature" {
    content              = <<-EOT
        Sun Tzu said: The art of war is of vital importance to the State.
        It is a matter of life and death, a road either to safety or to 
        ruin. Hence it is a subject of inquiry which can on no account be
        neglected.
        add by wsg,haha
    EOT
    directory_permission = "0777"
    file_permission      = "0777"
    filename             = "art_of_war.txt"
    id                   = "39d9064ddaa3c67b737b17b035f2ee346d9938ce"
}
➜  local-file git:(main) ✗
```

如果其他人误操作 手动改了上面的文件，当terraform plan时，就会 （configuration drift ）

此时，正确的做法时，

```jsx
➜  local-file git:(main) ✗ terraform refresh
local_file.literature: Refreshing state... [id=39d9064ddaa3c67b737b17b035f2ee346d9938ce]
➜  local-file git:(main) ✗ terraform show

➜  local-file git:(main) ✗ “terraform apply -auto-approve” //paste erro
➜  local-file git:(main) ✗ terraform apply -auto-approve

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

----
➜  local-file git:(main) ✗ terraform show
# local_file.literature:
resource "local_file" "literature" {
    content              = <<-EOT
        Sun Tzu said: The art of war is of vital importance to the State.
        It is a matter of life and death, a road either to safety or to 
        ruin. Hence it is a subject of inquiry which can on no account be
        neglected.
        add by wsg,haha
    EOT
    directory_permission = "0777"
    file_permission      = "0777"
    filename             = "art_of_war.txt"
    id                   = "39d9064ddaa3c67b737b17b035f2ee346d9938ce"
}

//可以看到，id 没有变化，说明并没有delete
```

# sn-cluster practical

## 待解决问题

### how interpret

proximabeta 

```jsx
module "sn_crds" {
  source  = "streamnative/charts/helm//modules/crds"
  version = "v0.8.1"

  depends_on = [
    module.sn_cluster
  ]
}

```

- module.sn_cluster
    
    ```jsx
    module "sn_cluster" {
      source  = "streamnative/cloud/aws"
      version = "2.2.4-alpha"
    
      cluster_enabled_log_types = []
    
      add_vpc_tags             = true
      cluster_name             = local.cluster_name
      cluster_version          = "1.20"
      enable_func_pool         = true
      hosted_zone_id           = aws_route53_zone.main.zone_id
      map_additional_iam_roles = local.cluster_role_mapping
      node_pool_instance_types = [local.instance_type]
      node_pool_disk_size      = 40
      node_pool_desired_size   = floor(local.desired_num_nodes / length(module.vpc.private_subnet_ids)) # Floor here to keep the desired count lower, autoscaling will take care of the rest
      node_pool_min_size       = 1
      node_pool_max_size       = ceil(local.max_num_nodes / length(module.vpc.private_subnet_ids)) # Ceiling here to keep the upper limits on the high end
      private_subnet_ids       = module.vpc.private_subnet_ids
      public_subnet_ids        = module.vpc.public_subnet_ids
      region                   = local.region
      vpc_id                   = module.vpc.vpc_id
    
      enable_node_group_private_networking = false
    
      ### Istio Configuration
      enable_istio       = true
      istio_mesh_id      = local.cluster_name
      istio_network      = "default"
      istio_trust_domain = local.cluster_name
      service_domain     = local.service_domain
    
      ### Func
      func_pool_min_size = 1
    
      depends_on = [
        module.vpc
      ]
    }
    ```
    

# 知识点

## immutable vs mutable

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%2013.png)

## what plan do

![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%2014.png)

- save plan
    
    ```jsx
    “terraform plan -out plan.out”
    
    ```
    

- if have state file
    
    ![Untitled](../../images/terraform%20ce23544f48784c35a9e58eaf0d8fd2ce/Untitled%2015.png)
    

## debug

“For more verbose logs, you can turn on trace-level logging by setting the environment variable TF_LOG =trace to a non-zero value, e.g. export TF_LOG=trace.”

“If terraform plan is running slowly, turn off trace-level logging and consider increasing parallelism (-parallelism=n).”

Excerpt From
Terraform in Action
Scott Winkler
This material may be protected by copyright.

## 默认行为

会将目录内的所有tf 文件拼接在一起

“Make sure the folder doesn’t contain any existing configuration code, because Terraform concatenates all .tf files together”

Excerpt From
Terraform in Action
Scott Winkler
This material may be protected by copyright.
