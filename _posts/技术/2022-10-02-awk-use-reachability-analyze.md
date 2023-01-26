---
layout: post
title: How to use AWS’s reachability analyzer
category: 技术
---

# How to use AWS’s reachability analyzer

Created by: Shengguo Wu
From Slab: No
Last edited by: Martijn Gonlag
Last edited time: November 24, 2022 6:42 PM
Status: WIP
Team(s): SRE, Technical Support

# **Index**

---

## Purpose

In AWS environment, the traditional tool (like ping and traceroute ) can not work. If you want to check the connectivity or the route-path  between the EC2 instance(which is in  different vpc), the best way is to use reachability analyzer provided by AWS.

This article mainly demo how to use it


## Procedure

Create a step by step procedure to complete this process. Add Miro boards, screen recordings, and images to provide a visual aid.

### Example1: check vpc-peering connectivity

there is two vpc configured with vpc peering, each vpc have a ec2 instance, need to know if vpc peering work properly 

- step1: create  reachability analyzer
    
    ![Untitled](../../images/How%20to%20use%20AWS%E2%80%99s%20reachability%20analyzer%20a8adf51b33ea4d3396966c461c459b6e/Untitled.png)
    
    then you can see it support many type (like ec2 instance, vpc enpoint, vpc peering connection)，in this example，we choose instance
    
    ![Untitled](../../images/How%20to%20use%20AWS%E2%80%99s%20reachability%20analyzer%20a8adf51b33ea4d3396966c461c459b6e/Untitled%201.png)
    
     choose the ec2 instance in each vpc
    
    ![Untitled](../../images/How%20to%20use%20AWS%E2%80%99s%20reachability%20analyzer%20a8adf51b33ea4d3396966c461c459b6e/Untitled%202.png)
    

- step2: check result
    
    
    ![Untitled](../../images/Customer%20Experience%20eb5d6f6f943e48018843458b00964109/Customer%20Support%20893fb4c3c4d24222aed03a79bc740c09/Pulsar%20%E4%B8%AD%E6%96%87%E6%8C%87%E5%8D%97%20027f3805b8804861a6aa13e98f9c47cc/VPC%E7%9B%B8%E5%85%B3%E6%A6%82%E5%BF%B5%E5%92%8C%E5%AE%9E%E8%B7%B51%202%EF%BC%88VPC-Peering%EF%BC%89%20ce00ae54048a48738f42852f24e883f8/Untitled.png)
    
    from the result, we now know the connectivity is blocked in the above route-table, so we check the route table and then rerun the  reachability analyzer as following, now it is ok
    
    ![Untitled](../../images/Customer%20Experience%20eb5d6f6f943e48018843458b00964109/Customer%20Support%20893fb4c3c4d24222aed03a79bc740c09/Projects%20b8ddf612d4c741cda374df0aaed266f0/Supported%20projects%209d07d2eef9234f77b29dc844f9f89831/HTSC%20Securities%20f053a52816bf4655ad6ddb21f675a90b/use%20aws%20rechability%20analyzer%2013bdecaa94004001862f2436a6538d84/Untitled.png)
    

### Example2: check vpc endpoint connectivity

we create a interface endpoint in our vpc，need to check if ec2 can connect to the endpoint ok? 

- step1: create  reachability analyzer
    
    we create an reachability analyzer （source as ec2 instance，destination as vpc endpoint）
    
    ![Untitled](../../images/Customer%20Experience%20eb5d6f6f943e48018843458b00964109/Customer%20Support%20893fb4c3c4d24222aed03a79bc740c09/Projects%20b8ddf612d4c741cda374df0aaed266f0/Supported%20projects%209d07d2eef9234f77b29dc844f9f89831/HTSC%20Securities%20f053a52816bf4655ad6ddb21f675a90b/use%20aws%20rechability%20analyzer%2013bdecaa94004001862f2436a6538d84/Untitled%201.png)
    
- step2: according the above result，modify the security group
    
    add inboud rule source is unlimited as following
    
    ![Untitled](../../images/Customer%20Experience%20eb5d6f6f943e48018843458b00964109/Customer%20Support%20893fb4c3c4d24222aed03a79bc740c09/Projects%20b8ddf612d4c741cda374df0aaed266f0/Supported%20projects%209d07d2eef9234f77b29dc844f9f89831/HTSC%20Securities%20f053a52816bf4655ad6ddb21f675a90b/use%20aws%20rechability%20analyzer%2013bdecaa94004001862f2436a6538d84/Untitled%202.png)
    

- check result again
    
    
    ![Untitled](../../images/Customer%20Experience%20eb5d6f6f943e48018843458b00964109/Customer%20Support%20893fb4c3c4d24222aed03a79bc740c09/Projects%20b8ddf612d4c741cda374df0aaed266f0/Supported%20projects%209d07d2eef9234f77b29dc844f9f89831/HTSC%20Securities%20f053a52816bf4655ad6ddb21f675a90b/use%20aws%20rechability%20analyzer%2013bdecaa94004001862f2436a6538d84/Untitled%203.png)
    
    now connect ok
