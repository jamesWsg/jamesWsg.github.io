---
layout: post
title: 学习笔记之微博的自动化运维平台
category: 技术
---



# 背景



产品运维

拥抱快速变化



刘然，2011年加入微博，





微博平台架构



openAPI 转化为微博平台





# 业务架构

![1569725253048](../../images/weibo auto ops/service-architect1569725253048.png)







![1569725432865](../../images/weibo auto ops/op-roles1569725432865.png)



报警 ：贯穿 全部体系，监控和告警拆分

传统 ：脚本运维





#中控平台



产品经理思维 规划运维平台

系统可用性，易用性，



![1569725855962](../../images/weibo auto ops/op architect1569725855962.png)





ECO

![1569725932465](../../images/weibo auto ops/ECO-architect1569725932465.png)

CMDB 产品属性（服务属性），和传统区别



决策中心： 同样的问题可以沉淀下来，不要处理重复的问题





### DCP 混合云



![1569745606124](../../images/weibo auto ops/DCP-1569745606124.png)









###扩容案例

痛点，

解决方案

阿里云 slb，单slb 断连



![1569745750542](../../images/weibo auto ops/扩容-1569745750542.png)

 







## 公有云实践

150G 专线， 自己机房到 阿里机房



一半服务在 阿里云



私有云

