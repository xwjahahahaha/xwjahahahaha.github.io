---
title: k8s的bug记录
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-02-06 09:44:57
---

## 1、创建ingress时ingress-nginx controller报错：ingress does not contain a valid IngressClass

原因：`ingress-nginx`新版本对新创建的ingress要求必须要创建`IngressClass`

After some googling it turns out the Nginx controller was updated back in August to now require a IngressClass to be specified on your ingresses. You can read about it in the release notes [here](https://github.com/kubernetes/ingress-nginx/releases/tag/controller-v1.0.0).

https://robearlam.com/blog/nginx-ingress-breaking-change-ingress.class-now-required

<!-- more -->

解决：

创建ingress时加入注释：`kubernetes.io/ingress.class: "nginx"`

例如：

