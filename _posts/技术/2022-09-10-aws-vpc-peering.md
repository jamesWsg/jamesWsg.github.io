---
layout: post
title: aws VPC-Peering实践探索
category: 技术
---

# VPC相关概念和实践1/2（VPC-Peering）

VPC可以理解为一个虚拟的数据中心，数据中心会部署不同类型的应用，不同应用对网络的访问要求不一样（比如有些应用需要和internet双向访问，有些应用需要单向访问internet，有些应用需要禁止访问internet，数据中心内部之前的网络访问也可以设置），这些网络控制的需求都可以在VPC中实现

## VPC网络整体示意图

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled.png)

- 相关术语
    
    IGW： vpc中访问internet的虚拟设备
    
    NGW：vpc中，ec2没有public ip但是有希望访问internet，会用到该虚拟设备
    

一个vpc中会划分不同的subnet，subnet可以选择不同AZ（availability zone）

可以根据需要创建internet-gateway （IGW）

和NAT-gateway（NGW）

route table (有vpc全局的main route table，也有每个subnet自己的route table)

vpc-peeing： 将2个vpc（可以是同一个账号下的2个vpc，也可以是不同账号下的vpc）网络打通（前提是2个vpc的CIDR网段没有重叠（overlap））

## 带路由表的示意图

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%201.png)

## vpc property

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%202.png)

### vpc-routetable

有vpc全局层面的route-table（main route table）

每个subnet也可以创建自己的route-table

## default vpc

“VPC comes preconfigured with the following set of configurations:

- The default VPC is always created with a CIDR block of /16, which means it supports 65,536 IP addresses in it.
- A default subnet is created in each AZ of your selected region. Instances launched in these default subnets have both a public and a private IP address by default as well.
- An Internet Gateway is provided to the default VPC for instances to have Internet connectivity.
- A few necessary route tables, security groups, and ACLs are also created by default that enable the instance traffic to pass through to the Internet. Refer to the following figure:”

## VPC peering

Inter-Region VPC peering provides a simple and cost-effective way to share resources between regions or replicate data for geographic redundancy.

[https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)

### limit

[https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)

# 操作

## vpc 属性检查

- network ACL
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%203.png)
    
- route-table
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%204.png)
    
- route-internet-gateway
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%205.png)
    
    igw（internet-gateway）可以从vpc中detach
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%206.png)
    

## vpc CIDR(subnet)分配后不可更改？

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%207.png)

## 可以查看vpc相关资源在所有region的分布

admin用户查看

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%208.png)

## prefix list

默认会看到这些 prefix

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%209.png)

## aws cli

- 创建vpc
    
    ```jsx
    ➜  aws-cli aws ec2 create-vpc --cidr-block 172.16.0.0/16
    ➜  aws-cli
    
    ```
    
- check vpc
    
    ```jsx
    ➜  aws-cli aws ec2 describe-vpcs --vpc-ids vpc-0b696b859f722d87b
    
    {
        "Vpcs": [
            {
                "CidrBlock": "172.16.0.0/16",
                "DhcpOptionsId": "dopt-02e2df729b14fb866",
                "State": "available",
                "VpcId": "vpc-0b696b859f722d87b",
                "OwnerId": "359990655175",
                "InstanceTenancy": "default",
                "CidrBlockAssociationSet": [
                    {
                        "AssociationId": "vpc-cidr-assoc-09fd3439beb8f53d8",
                        "CidrBlock": "172.16.0.0/16",
                        "CidrBlockState": {
                            "State": "associated"
                        }
                    }
                ],
                "IsDefault": false
            }
        ]
    }
    
    //与default vpc 对比
    ➜  aws-cli aws ec2 describe-vpcs --vpc-ids vpc-0d33f1d31b002d5c7
    {
        "Vpcs": [
            {
                "CidrBlock": "172.31.0.0/16",
                "DhcpOptionsId": "dopt-02e2df729b14fb866",
                "State": "available",
                "VpcId": "vpc-0d33f1d31b002d5c7",
                "OwnerId": "359990655175",
                "InstanceTenancy": "default",
                "CidrBlockAssociationSet": [
                    {
                        "AssociationId": "vpc-cidr-assoc-02519e6b9bd118b4d",
                        "CidrBlock": "172.31.0.0/16",
                        "CidrBlockState": {
                            "State": "associated"
                        }
                    }
                ],
                "IsDefault": true
            }
        ]
    }
    (END)
    ```
    
    # 场景操作
    
    ## 同一个账号下，同一个region(virginia)的2个vpc创建 vpc peering
    
    - 当前环境检查
        
        
        | Name | VPC ID | State | IPv4 CIDR |
        | --- | --- | --- | --- |
        | default-vpc | https://us-east-1.console.aws.amazon.com/vpc/home?region=us-east-1#VpcDetails:VpcId=vpc-0d33f1d31b002d5c7 | Available | 172.31.0.0/16 |
        | my-vpc | https://us-east-1.console.aws.amazon.com/vpc/home?region=us-east-1#VpcDetails:VpcId=vpc-0b696b859f722d87b | Available | 172.16.0.0/16 |
        
        my-vpc中的vm ip: 172.16.10.160 (因为没有公网ip，无法通过公网登陆)
        
        ```jsx
        
        ```
        
        default-vpc中的vm ip: 172.31.80.132
        
        ```jsx
        [ec2-user@ip-172-31-80-132 ~]$ ip add
        1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
            link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
            inet 127.0.0.1/8 scope host lo
               valid_lft forever preferred_lft forever
            inet6 ::1/128 scope host
               valid_lft forever preferred_lft forever
        2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP group default qlen 1000
            link/ether 12:83:fe:31:b3:71 brd ff:ff:ff:ff:ff:ff
            inet 172.31.80.132/20 brd 172.31.95.255 scope global dynamic eth0
               valid_lft 2261sec preferred_lft 2261sec
            inet6 fe80::1083:feff:fe31:b371/64 scope link
               valid_lft forever preferred_lft forever
        
        //
        [ec2-user@ip-172-31-80-132 ~]$ traceroute 172.16.10.160
        traceroute to 172.16.10.160 (172.16.10.160), 30 hops max, 60 byte packets
         1  * * *
         2  * * *
         3  * * *
         4  * * *
         5  * * *
         6  * * *
        ```
        
        目标：vm(172.31.80.132) can connect to 172.16.10.160
        
    - 发起vpc-peering request
        
        
    - 接受 request
        
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2010.png)
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2011.png)
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2012.png)
        
    - 检查创建好的vpc-peering
        
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2013.png)
        
    - 增加路由表
        
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2014.png)
        
        对端vpc路由表
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2015.png)
        
    - 验证vpc之间联通性（reachability Analyzer）
        
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2016.png)
        
        可以看到网络包的双向的路由path
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2017.png)
        
    - hping3 检查vm之间网络延迟
        
        平均0.8ms，
        
        ```jsx
        [ec2-user@ip-172-31-80-132 ~]$ sudo hping3 -c 10 -S -p 22  172.16.10.160
        HPING 172.16.10.160 (eth0 172.16.10.160): S set, 40 headers + 0 data bytes
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=0 win=62727 rtt=0.5 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=1 win=62727 rtt=0.5 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=2 win=62727 rtt=0.6 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=3 win=62727 rtt=0.5 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=4 win=62727 rtt=0.5 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=5 win=62727 rtt=0.5 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=6 win=62727 rtt=0.6 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=7 win=62727 rtt=3.2 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=8 win=62727 rtt=0.5 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=9 win=62727 rtt=0.5 ms
        
        --- 172.16.10.160 hping statistic ---
        10 packets transmitted, 10 packets received, 0% packet loss
        round-trip min/avg/max = 0.5/0.8/3.2 ms
        [ec2-user@ip-172-31-80-132 ~]$
        ```
        
        - 上面测试的2个vm位与同一个 AZ上， 如果位于不同的AZ上，结果类似
            
            所以看起来，同一个 region内部，AZ之间网络和内网类似
            
            ```jsx
            [ec2-user@ip-172-31-80-132 ~]$ sudo hping3 -c 10 -S -p 22  172.16.2.105
            HPING 172.16.2.105 (eth0 172.16.2.105): S set, 40 headers + 0 data bytes
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=0 win=62727 rtt=1.2 ms
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=1 win=62727 rtt=0.7 ms
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=2 win=62727 rtt=0.6 ms
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=3 win=62727 rtt=0.6 ms
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=4 win=62727 rtt=0.8 ms
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=5 win=62727 rtt=0.8 ms
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=6 win=62727 rtt=1.3 ms
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=7 win=62727 rtt=0.8 ms
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=8 win=62727 rtt=0.8 ms
            len=44 ip=172.16.2.105 ttl=255 DF id=0 sport=22 flags=SA seq=9 win=62727 rtt=0.8 ms
            
            --- 172.16.2.105 hping statistic ---
            10 packets transmitted, 10 packets received, 0% packet loss
            round-trip min/avg/max = 0.6/0.8/1.3 ms
            [ec2-user@ip-172-31-80-132 ~]$
            ```
            
    
    ## 不同账号，不同region vpc peering创建
    
    - 环境准备
        
        
        | 资源 | account1 | account2（mama） |
        | --- | --- | --- |
        | vpc | myvpc(virgina) | wsg-vpc(tokyo) |
        | vm | 172.16.10.160 | 10.0.13.59（public subnet） |
        |  |  | 10.0.129.120（private subnet） |
        |  |  |  |
    - account1 创建vpc peering
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2018.png)
        
        其中需要填入 account2 的 account-ID 和 VPC ID
        
        创建成功后，可以看到vpc peering request已经成功发送到对端account
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2019.png)
        
    - account2 中接受 vpc-peering reques
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2020.png)
        
        accept后，就可以看到vpc-peering的状态（可以看到2端 vpc的CIDR（网段））
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2021.png)
        
    - 2个account的对应vpc中分别增加到对端的路由
        - account1 vpc 增加路由
            
            ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2022.png)
            
            ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2023.png)
            
        - account2 的 vpc的 public subnet 增加路由
            
            ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2024.png)
            
            ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2025.png)
            
    - check
        
        ```jsx
        [ec2-user@ip-172-16-10-160 ~]$ telnet 10.0.13.59 22
        Trying 10.0.13.59...
        Connected to 10.0.13.59.
        Escape character is '^]'.
        SSH-2.0-OpenSSH_7.4
        ^C^C^C^C^C^C
        
        [ec2-user@ip-10-0-129-120 ~]$ sudo hping3 -c 10 -S -p 22 172.16.10.160
        HPING 172.16.10.160 (eth0 172.16.10.160): S set, 40 headers + 0 data bytes
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=0 win=62727 rtt=167.9 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=1 win=62727 rtt=167.9 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=2 win=62727 rtt=166.9 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=3 win=62727 rtt=169.2 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=4 win=62727 rtt=166.0 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=5 win=62727 rtt=167.6 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=6 win=62727 rtt=168.1 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=7 win=62727 rtt=167.2 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=8 win=62727 rtt=176.3 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=9 win=62727 rtt=167.5 ms
        
        --- 172.16.10.160 hping statistic ---
        10 packets transmitted, 10 packets received, 0% packet loss
        round-trip min/avg/max = 166.0/168.5/176.3 ms
        [ec2-user@ip-10-0-129-120 ~]$
        
        //from ec2 10.0.13.59
        [ec2-user@ip-10-0-13-59 ~]$ sudo hping3 -c 10 -S -p 22 172.16.10.160
        HPING 172.16.10.160 (eth0 172.16.10.160): S set, 40 headers + 0 data bytes
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=0 win=62727 rtt=167.5 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=1 win=62727 rtt=167.4 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=2 win=62727 rtt=167.8 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=3 win=62727 rtt=166.1 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=4 win=62727 rtt=167.6 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=5 win=62727 rtt=166.6 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=6 win=62727 rtt=167.4 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=7 win=62727 rtt=166.9 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=8 win=62727 rtt=168.3 ms
        len=44 ip=172.16.10.160 ttl=255 DF id=0 sport=22 flags=SA seq=9 win=62727 rtt=167.2 ms
        
        --- 172.16.10.160 hping statistic ---
        10 packets transmitted, 10 packets received, 0% packet loss
        round-trip min/avg/max = 166.1/167.3/168.3 ms
        [ec2-user@ip-10-0-13-59 ~]$
        
        ```
        
        - 与account2 中 vpc的private subnet 没有创建路由，所以不通
            
            ```jsx
            [ec2-user@ip-172-16-10-160 ~]$ telnet 10.0.129.120 22
            Trying 10.0.129.120...
            ```
            
            在 private subnet中再补上 路由，
            
            ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2026.png)
            
            再次验证
            
            ```jsx
            [ec2-user@ip-172-16-10-160 ~]$ telnet 10.0.129.120 22
            Trying 10.0.129.120...
            Connected to 10.0.129.120.
            Escape character is '^]'.
            SSH-2.0-OpenSSH_7.4
            
            ```
            
    
    # FAQ
    
    ## 为啥我的vm无法ssh登陆
    
    新建了一个vpc，划分为多个 subnet
    
    - 现象
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2027.png)
        
        我的ec2 已经分配了54.168.57.130
        
        ```jsx
        ➜  ~ ssh -i ~/.ssh/mama.pem ec2-user@54.168.57.130
        ssh: connect to host 54.168.57.130 port 22: Operation timed out
        ➜  ~
        ```
        
    - check subnet
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2028.png)
        
        this is a subnet route table
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2029.png)
        
    - subnet route table add route
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2030.png)
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2031.png)
        
        ```jsx
        ➜  ~ ssh -i ~/.ssh/mama.pem ec2-user@54.168.57.130
        Last login: Sat Oct 29 09:49:01 2022 from 103.183.218.24
        
               __|  __|_  )
               _|  (     /   Amazon Linux 2 AMI
              ___|\___|___|
        
        https://aws.amazon.com/amazon-linux-2/
        -bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory
        [ec2-user@ip-10-0-13-59 ~]$ ip add
        1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
            link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
            inet 127.0.0.1/8 scope host lo
               valid_lft forever preferred_lft forever
            inet6 ::1/128 scope host
               valid_lft forever preferred_lft forever
        2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP group default qlen 1000
            link/ether 06:6c:a1:e4:ba:1f brd ff:ff:ff:ff:ff:ff
            inet 10.0.13.59/20 brd 10.0.15.255 scope global dynamic eth0
               valid_lft 3395sec preferred_lft 3395sec
            inet6 fe80::46c:a1ff:fee4:ba1f/64 scope link
               valid_lft forever preferred_lft forever
        [ec2-user@ip-10-0-13-59 ~]$
        ```
        
    
    ## vpc peering 中创建的路由，会把 igw的路由自动删除？
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2032.png)
    
    如果需要internet访问，需要把igw的路由再加上
    
    （）
    
    ## VPC中的main route table
    
    和route table的区别，main route table看起来都是一样的？？ 
    
    每个subnet都可以有自己路由表
    
    [https://medium.com/@mda590/aws-routing-101-67879d23014d](https://medium.com/@mda590/aws-routing-101-67879d23014d)
    
    ## 如何确认一个subnet是否允许公网访问？
    
    创建vpc过程中，自动创建的subnet有public和private 2种，如果 vm放在private的subnet中，即使vm自动分配了公网ip，还是无法ssh登陆。
    
    ## 如何确认一个region有哪些AZ（）
    
    在创建subnet时，这边可以列出所有的az，
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2033.png)
    
    ## 延迟
    
    [https://blog.cnlabs.net/wp-tools/aws_network_test/](https://blog.cnlabs.net/wp-tools/aws_network_test/)
    
    [https://www.cloudping.co/grid/latency/timeframe/1D](https://www.cloudping.co/grid/latency/timeframe/1D)
    
    - 同一个vpc中，同一个AZ中的 vm之间延迟
        
        ```jsx
        [ec2-user@ip-172-31-80-132 ~]$ sudo hping3 -c 10 -S -p 22 172.31.81.200
        HPING 172.31.81.200 (eth0 172.31.81.200): S set, 40 headers + 0 data bytes
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=0 win=62727 rtt=0.9 ms
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=1 win=62727 rtt=0.7 ms
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=2 win=62727 rtt=0.6 ms
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=3 win=62727 rtt=0.6 ms
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=4 win=62727 rtt=0.6 ms
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=5 win=62727 rtt=0.7 ms
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=6 win=62727 rtt=0.7 ms
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=7 win=62727 rtt=0.6 ms
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=8 win=62727 rtt=0.7 ms
        len=44 ip=172.31.81.200 ttl=255 DF id=0 sport=22 flags=SA seq=9 win=62727 rtt=0.6 ms
        
        --- 172.31.81.200 hping statistic ---
        10 packets transmitted, 10 packets received, 0% packet loss
        round-trip min/avg/max = 0.6/0.7/0.9 ms
        [ec2-user@ip-172-31-80-132 ~]$
        ```
        
    
    ## vpc中的ip不够用了怎么办？
    
    vpc可以增加 secondary CIDR block（需要满足一定要求，[https://www.youtube.com/watch?v=ZdydaX1ds8I](https://www.youtube.com/watch?v=ZdydaX1ds8I)）
    
    The RFC1918 address space includes the following networks:
    
    - 10.0.0.0 – 10.255.255.255 (10/8 prefix)
    - 172.16.0.0 – 172.31.255.255 (172.16/12 prefix)
    - 192.168.0.0 – 192.168.255.255 (192.168/16 prefix)
    
    ## public subnet中 ec2无法访问 internet
    
    *** ec2 需要public ip ，igw 最终才能正常访问 internet ***（是双向访问）
    
    或者 ec2 没有public ip，通过ngw，igw 最终访问internet（只能是单向访问）
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2034.png)
    
    ```jsx
    通过有public ip的vm 跳转到上面的 vm里面
    [ec2-user@ip-172-16-10-160 ~]$ ping www.google.com
    PING www.google.com (172.253.122.104) 56(84) bytes of data.
    ^C
    --- www.google.com ping statistics ---
    8 packets transmitted, 0 received, 100% packet loss, time 7169ms
    
    [ec2-user@ip-172-16-10-160 ~]$ ^C
    
    ```
    
    - 给该ec2 增加一个 public ip（创建一个 E IP，然后attach到该 ec2的 ENI上）
        
        
        ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled%2035.png)
        
        then check connet to internet,ok
        
        ```jsx
        [ec2-user@ip-172-16-10-160 ~]$ ping www.google.com
        PING www.google.com (172.253.122.104) 56(84) bytes of data.
        64 bytes from bh-in-f104.1e100.net (172.253.122.104): icmp_seq=1 ttl=50 time=1.37 ms
        64 bytes from bh-in-f104.1e100.net (172.253.122.104): icmp_seq=2 ttl=50 time=1.35 ms
        64 bytes from bh-in-f104.1e100.net (172.253.122.104): icmp_seq=3 ttl=50 time=1.27 ms
        ^C
        --- www.google.com ping statistics ---
        3 packets transmitted, 3 received, 0% packet loss, time 2002ms
        rtt min/avg/max/mdev = 1.277/1.333/1.371/0.050 ms
        [ec2-user@ip-172-16-10-160 ~]$ ^C
        [ec2-user@ip-172-16-10-160 ~]$
        ```
