---
layout: diagram_post
title: 使用 markdown 画各种流程图
tags: ["markdown"]
---

前一阵在负责支付中心系统的搭建，要给下游的服务调用方和前端介绍整个支付流程的流转。需要画一些流程图，想着 markdown 会不会有这些功能呢？果不其然，这个世界总给极客提供一种快捷的方式做一些事~ 下面带大家快速体验一下吧，10 分钟包教包会！

## 流程图

流程图是最常用的场景了。

<div class="flow">
<textarea class="flowcode">
st=>start: Start 
e=>end           
ldata=>operation: 进入csdn 

st->ldata->e 
</textarea>
</div>


```
APP 端->后端: 1. 发起创建订单请求
后端->APP 端: 2. 返回订单编号
APP 端->后端: 3. 发起支付请求(需验签)
后端->支付中心: 4. 获取支付参数，用订单信息(包括金额、订单号)
支付中心->后端: 5. 返回支付相关参数
后端->APP 端: 6. 返回支付相关参数
Note right of APP 端: 7. 唤起支付 APP 进行支付
三方支付 --> 支付中心: 8. 发送回调通知
Note right of 支付中心: 保存支付结果并发送 MQ
Note right of 后端: 9. 处理回调并更新订单状态
```

## 序列图
## 甘特图
## Mermaid 流程图
## Mermaid 序列图
## Mermaid 序列图