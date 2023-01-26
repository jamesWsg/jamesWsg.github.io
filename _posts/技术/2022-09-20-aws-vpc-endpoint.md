---
layout: post
title: aws VPC-Endpoint实践探索
category: 技术
---
# VPC相关概念和实践2/2(VPC-Endpoint)

## use scenario

一般vpc中的ec2需要访问aws service时，需要通过IGW（internet-gateway），需要走public网络。如果业务需要不走public网络访问 aws service，就需要使用 privateLink

you can use AWS PrivateLink to connect your VPC to AWS services as if they were in your VPC, without the use of an internet gateway.

### 对比示意图

- 不使用endpoint（PrivateLink）
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled.png)
    
- 使用endpoint（PrivateLink）后
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%201.png)
    

# 场景操作

## ec2通过 vpc endpoint(interface)访问s3服务

- prepare
    
    
    |  | default-vpc | my-vpc |  |
    | --- | --- | --- | --- |
    | vm | 172.31.80.132/20 | 172.16.10.77/24 |  |
    |  |  |  |  |
- 默认没有vpc-endpoint情况
    
    ```jsx
    [ec2-user@ip-172-31-80-132 ~]$ s3cmd ls s3://wsgbucket
                              DIR  s3://wsgbucket/test-folder/
    2022-11-09 09:17       226445  s3://wsgbucket/1Untitled.jpg
    2022-10-27 11:26          588  s3://wsgbucket/ansible.yml
    2022-11-10 04:49         2610  s3://wsgbucket/testCli
    [ec2-user@ip-172-31-80-132 ~]$
    
    //my-vpc vm
    [ec2-user@ip-172-16-10-77 ~]$ s3cmd ls s3://wsgbucket
                              DIR  s3://wsgbucket/test-folder/
    2022-11-09 09:17       226445  s3://wsgbucket/1Untitled.jpg
    2022-10-27 11:26          588  s3://wsgbucket/ansible.yml
    2022-11-10 04:49         2610  s3://wsgbucket/testCli
    [ec2-user@ip-172-16-10-77 ~]$
    [ec2-user@ip-172-16-10-77 ~]$ ping www.google.com
    PING www.google.com (142.251.16.103) 56(84) bytes of data.
    64 bytes from bl-in-f103.1e100.net (142.251.16.103): icmp_seq=1 ttl=50 time=1.58 ms
    64 bytes from bl-in-f103.1e100.net (142.251.16.103): icmp_seq=2 ttl=50 time=1.59 ms
    ^C
    
    此时应该是通过 IGW访问到的（可以验证将 IGW 拿掉后，是否可以访问 ，）
    ```
    
    - 去掉my-vpc的 IGW
        
        ```jsx
        [ec2-user@ip-172-16-10-77 ~]$ ping www.google.com
        PING www.google.com (142.251.16.99) 56(84) bytes of data.
        // now vm can not access the internet
        
        //check s3 is not accessiable, blocked as following
        [ec2-user@ip-172-16-10-77 ~]$ s3cmd ls s3://wsgbucket
        
        WARNING: Retrying failed request: /?location ([Errno 110] Connection timed out)
        WARNING: Waiting 3 sec...
        ```
        

- create endpoint
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%202.png)
    
    有4种类型的endpoint，用来标识 该endpoint可以访问的不同资源
    
    选择endpoint类型，vpc
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%203.png)
    
    选择subnet 和securigy group
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%204.png)
    
- check创建好的endpoint
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%205.png)
    
    - dns not ok
        
        ```jsx
        [ec2-user@ip-172-16-10-77 ~]$ sudo hping3 -c 10 -S -p 443 vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com
        HPING vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com (eth0 172.16.10.143): S set, 40 headers + 0 data bytes
        
        --- vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com hping statistic ---
        10 packets transmitted, 0 packets received, 100% packet loss
        round-trip min/avg/max = 0.0/0.0/0.0 ms
        [ec2-user@ip-172-16-10-77 ~]$ ^C
        [ec2-user@ip-172-16-10-77 ~]$
        ```
        
    
    - 调整endpoint管理的security group（可以参考[runbook](https://www.notion.so/How-to-use-AWS-s-reachability-analyzer-a8adf51b33ea4d3396966c461c459b6e)）
        
        ```jsx
        [ec2-user@ip-172-16-10-77 ~]$ sudo hping3 -c 10 -S -p 443 vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com
        HPING vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com (eth0 172.16.10.143): S set, 40 headers + 0 data bytes
        len=44 ip=172.16.10.143 ttl=57 id=63404 sport=443 flags=SA seq=0 win=512 rtt=2.5 ms
        len=44 ip=172.16.10.143 ttl=247 id=22533 sport=443 flags=SA seq=1 win=512 rtt=1.8 ms
        len=44 ip=172.16.10.143 ttl=247 id=28645 sport=443 flags=SA seq=2 win=512 rtt=2.5 ms
        len=44 ip=172.16.10.143 ttl=248 id=52089 sport=443 flags=SA seq=3 win=512 rtt=2.6 ms
        len=44 ip=172.16.10.143 ttl=57 id=44856 sport=443 flags=SA seq=4 win=512 rtt=2.6 ms
        len=44 ip=172.16.10.143 ttl=247 id=10791 sport=443 flags=SA seq=5 win=512 rtt=2.0 ms
        len=44 ip=172.16.10.143 ttl=247 id=41632 sport=443 flags=SA seq=6 win=512 rtt=1.8 ms
        ^C
        --- vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com hping statistic ---
        7 packets transmitted, 7 packets received, 0% packet loss
        round-trip min/avg/max = 1.8/2.2/2.6 ms
        [ec2-user@ip-172-16-10-77 ~]$
        
        ```
        
    - aws cli endpoint test
        
        ```jsx
        //by endpoint
        
        [ec2-user@ip-172-16-10-77 ~]$ aws --endpoint-url https://bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com s3 ls
        2022-10-27 11:24:36 wsgbucket
        //需要在endpoint的name前面加bucket前缀
        
        // by public 
        [ec2-user@ip-172-31-80-132 ~]$ aws --endpoint-url https://s3-eu-west-1.amazonaws.com s3 ls
        2022-11-10 07:42:06 wsgbucket
        ```
        

## ec2 use vpc endpoint(gateway)访问s3

- prepare
    
    创建一个
    

- create gateway endpoint
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%206.png)
    
    其中有关联vpc中的路由表的配置（这个是和interface endpoint不同的地方）
    
- check
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%207.png)
    
    check route table
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%208.png)
    
    check prefix list
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%209.png)
    

- test s3 access
    
    ```jsx
    
    [ec2-user@ip-172-16-2-99 ~]$ aws s3 ls
    2022-10-27 11:24:36 wsgbucket
    [ec2-user@ip-172-16-2-99 ~]$
    
    ```
    
    - if now delete the gateway endpoint
        
        ```jsx
        [ec2-user@ip-172-16-2-99 ~]$ aws s3 ls
        
        Connect timeout on endpoint URL: "https://s3.amazonaws.com/"
        ```
        

# FAQ

## aws network

- Direct connect
    
    是一个独立的dashboard，需要注意在固定的aws 机房才能创建
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2010.png)
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2011.png)
    
    可用的机房
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2012.png)
    
    AWS Direct Connect helps in establishing a dedicated network from your premises to AWS. It enables a private and secure connection between AWS and the data center. It is compatible with AWS services and supports a high bandwidth for a more consistent network and better speed. The starting speed is around 50 Mbps and supports scaling up to 100 Gbps.
    
- privateLink(endpoint)
- AWS Site-to-Site VPN
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2013.png)
    

## VPC-Endpoints vs VPC-Peering

|  | VPC-Endpoints | VPC-Peering |
| --- | --- | --- |
| 实现原理 | 通过域名+NAT | 路由表 |
| 用途 | 为访问某个具体服务 | 打通网络，访问所有服务 |
| 作用范围 | 可以跨region | 在某个region内部有效，无法跨region |

本质上，interface-endpoint是DNS-hack， gateway-endpoint是 route-hack（[ref](https://itnext.io/what-exactly-are-vpc-endpoints-and-why-they-need-real-inter-region-support-283a9987fe51)er）

## s3的服务看起来有这几个，啥区别？

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2014.png)

## interface endpoint和gateway endpoint 区别，应用场景？

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2015.png)

- vpc 自动创建的到s3的 gateway endpoint
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2016.png)
    
- manual created interface endpoint
    
    ![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2017.png)
    
    上面关联的dns name 可以解析
    
    ```jsx
    [ec2-user@ip-172-16-10-77 ~]$ nslookup vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com
    Server:		172.16.0.2
    Address:	172.16.0.2#53
    
    Non-authoritative answer:
    Name:	vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com
    Address: 172.16.10.143
    
    [ec2-user@ip-172-16-10-77 ~]$
    ```
    

interface endpoint可以在vpc 之外使用

gateway endpoint 只能在vpc之内使用。

## vpc endpoint policy

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2018.png)

## interface endpoint 创建的网卡 ENI

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2019.png)

```jsx
[ec2-user@ip-172-16-10-77 ~]$ nslookup vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com
Server:		172.16.0.2
Address:	172.16.0.2#53

Non-authoritative answer:
Name:	vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com
Address: 172.16.10.143

[ec2-user@ip-172-16-10-77 ~]$
```

## S3 端口80，443

```jsx

[ec2-user@ip-172-31-80-132 ~]$ sudo hping3 -c 10 -S -p 80 s3.amazonaws.com
HPING s3.amazonaws.com (eth0 52.216.53.32): S set, 40 headers + 0 data bytes
len=44 ip=52.216.53.32 ttl=58 id=54189 sport=80 flags=SA seq=0 win=512 rtt=0.6 ms
len=44 ip=52.216.53.32 ttl=58 id=19177 sport=80 flags=SA seq=1 win=512 rtt=0.7 ms
len=44 ip=52.216.53.32 ttl=58 id=52965 sport=80 flags=SA seq=2 win=512 rtt=0.7 ms
len=44 ip=52.216.53.32 ttl=58 id=59158 sport=80 flags=SA seq=3 win=512 rtt=0.7 ms
len=44 ip=52.216.53.32 ttl=58 id=28259 sport=80 flags=SA seq=4 win=512 rtt=1.2 ms
len=44 ip=52.216.53.32 ttl=58 id=65407 sport=80 flags=SA seq=5 win=512 rtt=0.6 ms
len=44 ip=52.216.53.32 ttl=58 id=44786 sport=80 flags=SA seq=6 win=512 rtt=0.7 ms
len=44 ip=52.216.53.32 ttl=58 id=20560 sport=80 flags=SA seq=7 win=512 rtt=0.6 ms
len=44 ip=52.216.53.32 ttl=58 id=25452 sport=80 flags=SA seq=8 win=512 rtt=0.6 ms
len=44 ip=52.216.53.32 ttl=58 id=32064 sport=80 flags=SA seq=9 win=512 rtt=0.8 ms

--- s3.amazonaws.com hping statistic ---
10 packets transmitted, 10 packets received, 0% packet loss
round-trip min/avg/max = 0.6/0.7/1.2 ms
[ec2-user@ip-172-31-80-132 ~]$ sudo hping3 -c 10 -S -p 443 s3.amazonaws.com
HPING s3.amazonaws.com (eth0 52.217.194.208): S set, 40 headers + 0 data bytes
len=44 ip=52.217.194.208 ttl=250 id=30235 sport=443 flags=SA seq=0 win=512 rtt=0.9 ms
len=44 ip=52.217.194.208 ttl=250 id=9993 sport=443 flags=SA seq=1 win=512 rtt=1.0 ms
len=44 ip=52.217.194.208 ttl=250 id=31185 sport=443 flags=SA seq=2 win=512 rtt=0.9 ms
len=44 ip=52.217.194.208 ttl=250 id=2487 sport=443 flags=SA seq=3 win=512 rtt=0.9 ms
len=44 ip=52.217.194.208 ttl=250 id=63389 sport=443 flags=SA seq=4 win=512 rtt=0.9 ms
len=44 ip=52.217.194.208 ttl=250 id=21709 sport=443 flags=SA seq=5 win=512 rtt=1.0 ms
len=44 ip=52.217.194.208 ttl=250 id=58702 sport=443 flags=SA seq=6 win=512 rtt=1.0 ms
len=44 ip=52.217.194.208 ttl=250 id=51756 sport=443 flags=SA seq=7 win=512 rtt=0.9 ms
len=44 ip=52.217.194.208 ttl=250 id=7490 sport=443 flags=SA seq=8 win=512 rtt=0.9 ms
len=44 ip=52.217.194.208 ttl=250 id=13394 sport=443 flags=SA seq=9 win=512 rtt=1.0 ms

--- s3.amazonaws.com hping statistic ---
10 packets transmitted, 10 packets received, 0% packet loss
round-trip min/avg/max = 0.9/0.9/1.0 ms
[ec2-user@ip-172-31-80-132 ~]$
```

- test s3 in some region
    
    ```jsx
    [ec2-user@ip-172-31-80-132 ~]$ sudo hping3 -c 10 -S -p 80 s3-eu-west-1.amazonaws.com
    HPING s3-eu-west-1.amazonaws.com (eth0 52.218.26.139): S set, 40 headers + 0 data bytes
    len=44 ip=52.218.26.139 ttl=227 id=27733 sport=80 flags=SA seq=0 win=512 rtt=65.7 ms
    len=44 ip=52.218.26.139 ttl=227 id=48914 sport=80 flags=SA seq=1 win=512 rtt=64.5 ms
    len=44 ip=52.218.26.139 ttl=227 id=42733 sport=80 flags=SA seq=2 win=512 rtt=66.7 ms
    len=44 ip=52.218.26.139 ttl=226 id=8635 sport=80 flags=SA seq=3 win=512 rtt=66.2 ms
    len=44 ip=52.218.26.139 ttl=226 id=45237 sport=80 flags=SA seq=4 win=512 rtt=66.5 ms
    len=44 ip=52.218.26.139 ttl=226 id=7886 sport=80 flags=SA seq=5 win=512 rtt=65.5 ms
    len=44 ip=52.218.26.139 ttl=227 id=30341 sport=80 flags=SA seq=6 win=512 rtt=66.7 ms
    len=44 ip=52.218.26.139 ttl=226 id=49450 sport=80 flags=SA seq=7 win=512 rtt=67.7 ms
    len=44 ip=52.218.26.139 ttl=227 id=9406 sport=80 flags=SA seq=8 win=512 rtt=65.3 ms
    len=44 ip=52.218.26.139 ttl=227 id=36253 sport=80 flags=SA seq=9 win=512 rtt=66.3 ms
    
    --- s3-eu-west-1.amazonaws.com hping statistic ---
    10 packets transmitted, 10 packets received, 0% packet loss
    round-trip min/avg/max = 64.5/66.1/67.7 ms
    
    [ec2-user@ip-172-31-80-132 ~]$ sudo hping3 -c 10 -S -p 443 s3-eu-west-1.amazonaws.com
    HPING s3-eu-west-1.amazonaws.com (eth0 52.218.110.115): S set, 40 headers + 0 data bytes
    len=44 ip=52.218.110.115 ttl=227 id=61001 sport=443 flags=SA seq=0 win=512 rtt=66.6 ms
    len=44 ip=52.218.110.115 ttl=227 id=2823 sport=443 flags=SA seq=1 win=512 rtt=67.2 ms
    len=44 ip=52.218.110.115 ttl=227 id=24644 sport=443 flags=SA seq=2 win=512 rtt=65.8 ms
    len=44 ip=52.218.110.115 ttl=227 id=22339 sport=443 flags=SA seq=3 win=512 rtt=66.0 ms
    len=44 ip=52.218.110.115 ttl=227 id=49355 sport=443 flags=SA seq=4 win=512 rtt=65.8 ms
    len=44 ip=52.218.110.115 ttl=227 id=37562 sport=443 flags=SA seq=5 win=512 rtt=67.2 ms
    len=44 ip=52.218.110.115 ttl=227 id=50522 sport=443 flags=SA seq=6 win=512 rtt=65.9 ms
    len=44 ip=52.218.110.115 ttl=226 id=27811 sport=443 flags=SA seq=7 win=512 rtt=65.2 ms
    len=44 ip=52.218.110.115 ttl=227 id=12233 sport=443 flags=SA seq=8 win=512 rtt=65.4 ms
    len=44 ip=52.218.110.115 ttl=226 id=26528 sport=443 flags=SA seq=9 win=512 rtt=67.1 ms
    
    --- s3-eu-west-1.amazonaws.com hping statistic ---
    10 packets transmitted, 10 packets received, 0% packet loss
    round-trip min/avg/max = 65.2/66.2/67.2 ms
    [ec2-user@ip-172-31-80-132 ~]$
    ```
    

## endpoint 访问s3报错

sg 开启了所有 tcp

```jsx
[ec2-user@ip-172-16-10-77 ~]$ aws --endpoint-url https://vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com s3 ls

SSL validation failed for https://vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com/ hostname 'vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com' doesn't match either of 's3.us-east-1.amazonaws.com', 'bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1f.s3.us-east-1.vpce.amazonaws.com', '*.s3-control.us-east-1.amazonaws.com', 'bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1c.s3.us-east-1.vpce.amazonaws.com', '*.accesspoint.vpce-0a77994aa2124c299-9tcntpvn-us-east-1c.s3.us-east-1.vpce.amazonaws.com', '*.control.vpce-0a77994aa2124c299-9tcntpvn-us-east-1a.s3.us-east-1.vpce.amazonaws.com', '*.s3.us-east-1.amazonaws.com', '*.bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1a.s3.us-east-1.vpce.amazonaws.com', '*.bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com', '*.control.vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com', '*.accesspoint.vpce-0a77994aa2124c299-9tcntpvn-us-east-1e.s3.us-east-1.vpce.amazonaws.com', '*.bucket.vpce-0a77994aa2124c299-9tcntpvn.s3.us-east-1.vpce.amazonaws.com', '*.control.vpce-0a77994aa2124c299-9tcntpvn.s3.us-east-1.vpce.amazonaws.com', '*.control.vpce-0a77994aa2124c299-9tcntpvn-us-east-1e.s3.us-east-1.vpce.amazonaws.com', '*.bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1c.s3.us-east-1.vpce.amazonaws.com', '*.accesspoint.vpce-0a77994aa2124c299-9tcntpvn.s3.us-east-1.vpce.amazonaws.com', 'bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1e.s3.us-east-1.vpce.amazonaws.com', '*.bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1f.s3.us-east-1.vpce.amazonaws.com', '*.accesspoint.vpce-0a77994aa2124c299-9tcntpvn-us-east-1b.s3.us-east-1.vpce.amazonaws.com', '*.control.vpce-0a77994aa2124c299-9tcntpvn-us-east-1b.s3.us-east-1.vpce.amazonaws.com', '*.s3-accesspoint.us-east-1.amazonaws.com', '*.bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1e.s3.us-east-1.vpce.amazonaws.com', '*.control.vpce-0a77994aa2124c299-9tcntpvn-us-east-1c.s3.us-east-1.vpce.amazonaws.com', 'bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1b.s3.us-east-1.vpce.amazonaws.com', 'bucket.vpce-0a77994aa2124c299-9tcntpvn.s3.us-east-1.vpce.amazonaws.com', '*.accesspoint.vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com', 'bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1a.s3.us-east-1.vpce.amazonaws.com', 'bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com', '*.accesspoint.vpce-0a77994aa2124c299-9tcntpvn-us-east-1a.s3.us-east-1.vpce.amazonaws.com', '*.bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1b.s3.us-east-1.vpce.amazonaws.com', '*.control.vpce-0a77994aa2124c299-9tcntpvn-us-east-1f.s3.us-east-1.vpce.amazonaws.com', '*.accesspoint.vpce-0a77994aa2124c299-9tcntpvn-us-east-1f.s3.us-east-1.vpce.amazonaws.com'
```

- 解决方法，访问s3的 正确url是这个
    
    ```jsx
    [ec2-user@ip-172-16-10-77 ~]$ aws --endpoint-url https://bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com s3 ls
    2022-10-27 11:24:36 wsgbucket
    [ec2-user@ip-172-16-10-77 ~]$
    
    //需要指定bucket前缀
    
    ```
    

## vpc1中创建的interface endpoint 看起来可以被 vpc2中的ec2使用

vpc1 中创建了 interface endpoint

vpc2中的vm 测试（vpc2的vm可以访问internet）

```jsx
[ec2-user@ip-172-31-80-132 ~]$ aws --endpoint-url https://bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com s3 ls
2022-10-27 11:24:36 wsgbucket
[ec2-user@ip-172-31-80-132 ~]$
[ec2-user@ip-172-31-80-132 ~]$
[ec2-user@ip-172-31-80-132 ~]$ aws --endpoint-url https://bucket.vpce-0a77994aa2124c299-9tcntpvn-us-east-1d.s3.us-east-1.vpce.amazonaws.com s3 ls s3://wsgbucket
                           PRE test-folder/
2022-11-09 09:17:15     226445 1Untitled.jpg
2022-10-27 11:26:13        588 ansible.yml
2022-11-10 04:49:39       2610 testCli
[ec2-user@ip-172-31-80-132 ~]$

```

- 同时vpc2中的vm也可以通过public s3 endpoint 访问s3
    
    ```jsx
    [ec2-user@ip-172-31-80-132 ~]$ aws --endpoint-url https://s3-eu-west-1.amazonaws.com s3 ls
    2022-11-10 07:42:06 wsgbucket
    [ec2-user@ip-172-31-80-132 ~]$ aws --endpoint-url https://s3-eu-west-1.amazonaws.com s3 ls s3://wsgbucket
                               PRE test-folder/
    2022-11-09 09:17:15     226445 1Untitled.jpg
    2022-10-27 11:26:13        588 ansible.yml
    2022-11-10 04:49:39       2610 testCli
    [ec2-user@ip-172-31-80-132 ~]$
    
    ```
    

## 每个vpc中 默认的dns服务器

```jsx
//vpc1 解析另外一个 vpc中的 ec2 dns-name
[ec2-user@ip-172-31-80-132 ~]$ nslookup ip-172-16-2-99.ec2.internal
Server:		172.31.0.2
Address:	172.31.0.2#53

Non-authoritative answer:
Name:	ip-172-16-2-99.ec2.internal
Address: 172.16.2.99

//同一个vpc中
[ec2-user@ip-172-16-2-99 ~]$ nslookup ip-172-16-2-99.ec2.internal
Server:		172.16.0.2
Address:	172.16.0.2#53

** server can't find ip-172-16-2-99.ec2.internal: NXDOMAIN

[ec2-user@ip-172-16-2-99 ~]$

```

## endpoint service 类型？

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2020.png)

需要选择 loadbalance

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2021.png)

# 遗留问题

interface endpoint 使用NAT，和gateway endpoint 一样？如何验证？

# ref

[https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)

# draft

endpoint 也叫privateLink，目的是为了让vpc中的资源 以private ip的方式访问vpc外面的资源（比如aws的DB和S3，其他vpc中的服务）

## endpoint-service

You can create your own application in your VPC and configure it as an AWS PrivateLink-powered service (referred to as an endpoint service). Other AWS principals can create a connection from their VPC to your endpoint service using an interface VPC endpoint or a Gateway Load Balancer endpoint, depending on the type of service. You are the service provider, and the AWS principals that create connections to your service are service consumers.

### vpc-endpoints

endpoint是AWS管理的虚拟设备，实现VPC中的EC2 instance和外部服务通信（通过EC2的private网络通信）

“VPC endpoints basically allow you to securely connect your VPC with other AWS services. These are virtual devices that are highly available and fault tolerant by design. They are scaled and managed by AWS itself, so you don't have to worry about the intricacies of maintaining them. All you need to do is create a VPC endpoint connection between your VPC and an AWS service of your choice”

“The instances in the VPC communicate with other services using their private IP addresses itself, so there's no need to route the traffic over the Internet.”  

vs vpc peering

![Untitled](../../images/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B52%202(VPC-Endpoint)%2023e069f28bd246d782fd124278530e38/Untitled%2022.png)

**[Gateway endpoint](https://docs.aws.amazon.com/vpc/latest/userguide/vpce-gateway.html)**
 is a gateway that you specify as a target for a route in your route table for traffic destined to a supported AWS service. Currently supports S3 and DynamoDB services.
