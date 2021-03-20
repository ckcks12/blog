---
layout: post
title: AWS Route53 Geolocation Default Issue
author: Eunchan Lee
categories: [software,aws,route53]
---

글로벌 아키텍처를 어떻게 할까 하다가 어플리케이션은 반응을 위해서 리전별 배포하고

DB 같은건 한 곳에서 쓰기로 했었다 그리고 도메인을 하나로 하기로.

이를 위해 AWS Route53 의 Geolocation 을 쓰게 됐다

이때 주의할 점이 Geolocation 의 Default location 에 대한 설정을 하지 않으면 

가끔 geolocation 을 못 가져오는 DNS query source 에 대해서 dns 값을 못 주는 현상이 있었다

![](https://i.imgur.com/zyA4E2n.png)

> Route 53 uses [resolver-identity.cloudfront.net](http://resolver-identity.cloudfront.net/) to reflex the IP address where the DNS query came from
