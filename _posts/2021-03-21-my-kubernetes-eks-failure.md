---
layout: post
title: 나의 쿠버네티스(EKS) 장애
author: Eunchan Lee
categories: [software,kubernetes,eks]
---

![](https://i.imgur.com/0q3pzBi.png)

## Kubernetes - 내가 만든 장애1 : worker 죽이기

Infra App 이라는 Lens 같은 Kubernetes GUI 를 쓰고 있었다

근데 여기서 Context 를 System wide 하게 바꿔버렸다

관리 권한 때문에 `aws-auth.yaml` 를 apply 하는데 Infra App 에서 프로덕션으로 바꾸어버렸다

콘솔에서 apply 를 했는데 프로덕션에 적용이 되었다

알파와 프로덕션의 `aws-auth.yaml` 자체가 호환이 안되기 때문에

몇분여만에 모든 worker 가 master 와 연결이 끊겼다

nlb 에 target group 으로 등록된 worker node 들이 다 맛 갔다

Infra App 을 삭제하고 read-only kubeconfig 을 만들어서 사용하게 됐다


## Kubernetes - 내가 만든 장애2 : master 죽이기

worker node 를 한번에 180개 추가 생성했다

메인 웹앱이 올라갈 노드를 미리 띄워보려 했다

그런데 master 가 죽어버렸다 너무 많은 worker 추가라서 죽었다

aws support 에 따르면 수천개가 연결을 시도했다 한다

이상해서 autoscaling group 을 살펴보니 lb 에 등록이 안돼서 죽이고

asg 는 desired count 맞추려고 재생성하고 해서 그렇게 됐다

lb 에 등록될 수 있는 target 개수는 500 개가 제한이고

http 와 https 룰에다가 등록하기 때문에 250개 node 가 제한이었던 것이다


asg 가 node는 계쏙 띄우려고하고.

node 는 lb 에 못붙으니 unhealthy 로 죽고.. 이게 무한반복... ㅎㅎ;;

노드를 뭐 적게 띄울 수는 없고. 

해결 법은 AWS 에 문의해서 Target 제한 풀었다.


## 장애 기사가 나다

![](https://i.imgur.com/HodQNWG.png)

이 장애는 좀 특별하다. 바로 장애가 기사로 나왔기 때문이다. 

[기사](https://www.asiatoday.co.kr/view.php?key=20200622001719216)

흑흑.. 

그래도 장애가 기사가 날 정도의 기업에서 일한 다는 것은 멋진 일이다.

항상 감사하다.

