---
layout: post
title: 웹앱(Ruby On Rails) ECS - Kubernetes 마이그레이션
author: Eunchan Lee
categories: [software,kubernetes,eks,ecs]
---

![](https://i.imgur.com/uPtx6VR.png)

우리 회사 메인 웹서버는 Ruby On Rails 로 되어 있다.

Dokerize 되어 ECS 에서 돌고 있다.

환경 변수는 모두 SSM 으로 되어 있었다.

Kubernetes 의 External Secret 이라는 Godday 의 Operator 를 사용해서 SSM 을 Secert 으로 주입할 수 있도록 했다.

또 URL 등과 같은 Plain 한 값들은 Config Map 으로 분리했다.

환경 변수가 100개가 넘었다. 이 모든 것을 추적하기는 어려웠다. 슬랙에서 일일이 물어보며 찾아다녔는데 결국 주인 없는 환경변수도 있었기에 임의로 처리했다. (무식하면 무서운 줄 모른다고.. 운이 좋았다)

배포는 회사가 Kubernetes 의 ArgoCD 를 이용하고 있었기에 이 GitOps 를 활용할 수 있도록 간단한 툴을 만들었다.

![](https://i.imgur.com/sAqEasu.png)

위 사진은 Infra Admin 에서 deployment 를 관리하는 간단한 대시보드로써 ArgoCD 의 App 상태를 나타낸다.

![](https://i.imgur.com/n0buQDz.png)

위 사진은 Infra Admin 에서 해당 ArgoCD App 의 디테일을 보는 화면으로 배포 락을 걸 수 있고 이미지 태그를 업데이트할 수 있으며 현재 떠있는 pod 의 목록과 SSH 접근용 주소 등을 얻을 수 있다.

매 배포시마다 이미지 태그를 이 곳에 와서 바꾸는 것은 아니다. 저 UI 도 결국 내부 Queue 에 쏘고 이 Queue Worker 가 실제로 Kuberenetes 관련 작업을 한다. 이를 통해 동시에 겹치거나 할 일이 없고 또 DLQ 를 통해 실패 작업에 관한 모니터링이 가능하다.





## Ruby On Rails 의 DB Schema Migration Job






Ruby On Rails 의 경우 DB Schema Migration 스크립트가 배포시에 1회(only once) 돌아야 한다고 들었다.

그래서 ArgoCD 의 [Pre Sync Hook](https://argoproj.github.io/argo-cd/user-guide/resource_hooks/) 을 이용했다.

해당 Hook 에 Job 을 만들어 실행시켰다.

> Using a PreSync hook to perform a database schema migration before deploying a new version of the app.

다만 DB Schema Migration 도 같은 Docker Image 라서 Up & Running 되기 위해 같은 환경변수 등이 필요 했다.

그리고 이 DB Schema Migration 이 실패하면 실제 웹서버들은 Sync 되면 안됐다.

![](https://i.imgur.com/pYKideO.png)

그래서 위와 같이 플로우를 짜게 됐다.

PreSync 에서 Sync Wave 를 주어 가장 기초적으로 필요한 Service Account, External Secet 그리고 Schema Migration ConfigMap 을 생성했다.

여기서 ConfigMap Generator 등을 사용하지 않은 이유는 아래와 같다.

해당 ConfigMap 을 참조할 DB Migration Job 이 Pre Sync(Wave -1) 에 작동된다.

그러면 ConfigMap 이 Pre Sync(Wave -2) 에 생성 되어야 한다. 그리고 이 Pre Sync Wave 등을 설정하려면 Annotation 이 지원되어야 한다.

하지만 현재 사용하고 있는 Kubernetes v1.14 에서는 ConfigMap Generator 에서 Annotation 을 먹일 수 없었다..


아무튼 이렇게 진행하게 되었을 때 또 다른 문제가 생겼다.

Replicas 수정 -> Out of Sync -> Sync!

Pre Sync(-2) -> Pre Sync(-1) -> Sync

이때 Pre Sync(-2) 에서 Service Account 와 External Secret 이 **매번** 재생성 되었다.

내가 수정한 것은 Replicas 였는데..

그 이유는 Pre Sync 는 기본적으로 매번 Delete/Create 한다. ([소스](https://github.com/argoproj/argo-cd/blob/v1.5.3/util/hook/delete_policy.go#L24))

그렇게 되면 기존 Pod 이 재시작 된다거나 등의 작업을 할 때 Service Account, Secret 등을 찾지 못한다는 오류를 내게 된다.

![](https://i.imgur.com/Lf2Ia3Z.png)

Pre Sync 말고 Sync Wave 순서로 작업하게 됐다.

대충 Table 로 비교해보면

||이전|개선|
|:-:|:-:|:-:|
|리소스 강제 재생성|O|X|
|사전 작업 실패시 배포 중단|O|O|
|Canary 지원 여부|X|O|

다만 이렇게 되면 Ruby On Rails 의 DB Schema Migration 을 Job 으로 실행시킬 수 없었다.

Job 은 이미지를 변경시킬 수 없었다. 한번 실행되면 끝이다.

Pre Sync 는 Delete/Create 했지만 Sycn Wave 는 순수하게 Diff 가 있을 때만 작동했다.

따라서 Job 의 이미지가 바뀌지 않아서 DB Schema Migration 스크립트가 정상적으로 돌지 않았다.

그래서 이를 Edge Deployment 로 해결 했다.

Edge 라는 이름의 Deployment 를 새로 만들었다. 그리고 여기에 `initContainer` 를 통해 DB Schema Migration 스크립트를 실행시켰다.

이외에도 이전 ECS 를 이용하는 Schedule Task 를 위해 Task Definition 을 update 했다.

Edge Deployment 를 사용하면서 APM Sampling 도 가능하게 되었다. Edge 에만 APM Enabled 했다.

Edge Deployment 와 Stable Deployment 로 나누고 이 둘은 Service 에서 둘다 포함시켜서 같이 트래픽 되도록 했다.

Canary 의 경우에도 Sync Wave 를 통해서 Wait For Approval Pod 을 넣어서 liveness 를 실패시키도록 하고 Approval 을 통해 liveness 를 통과시키면 가능했다.

이렇게 Ruby On Rails 의 DB Schema Migration 스크립트를 Kubernetes 와 ArgoCD 에서 실행할 수 있도록 했다.



## Application Load Balancer Migration

우리 회사는 Applicatoin Load Balancer(이하 ALB) 를 많이 쓴다.

가장 큰 장점으로 동적으로 Routing 을 할 수 있기 때문이다.

장애 발생시에 Routing 을 바꾸어 고정적인 안내 메시지를 줄 수도 있고 (Fixed Response)

마이크로 서비스로 분기해낼 때도 Path Routing 을 통해 우아하게, 즉

1. 마이크로 서비스 장애 시에 손쉽게 라우팅을 변경
2. 비율을 정해서 트래픽 분산 가능

등이 가능하다.

내게 주어진 과제는 Terraform 으로 관리 되고 있던 ALB (Rule, Target Group 포함) 을 Kubernetes 로 마이그레이션 하는 것이다.

ALB 를 만들어내고 룰 설정을 만들어내야 했고 또 Pod 를 Target Group 으로 등록해야 했다.

다행히 ALB Operator 가 있었다. 그래서 사용하는 도중 아래와 같은 고통이 있었다.

#### 1. Annotation 에 ALB Rule 을 JSON 으로 때려박기

이때 문제는 디버깅이었다. JSON 문법 실수(",} 등 빼뜨리기)나 ALB Rule 에서 지원하지 않는 Attribute 를 사용할 때 ALB Controller 는 오류를 내뱉었다.

그런데 어디쯤에서 어떻게 문제가 되는지는 알 수 없었다.

눈알 빠지게 디버깅 했다. 

또한 Operator 패턴이 비동기적이다 보니 언제 어떻게 무엇을 실행시키는지가 불명확 했다.

그리고 나중에 Operator 가 많아지거나 버그가 있거나 혹은 악의적인 코드가 담겨져 있을 경우 이것이 통제 가능한가에 대한 의문이 들었다. 

Operator 패턴이 큰규모, 대규모적인 인프라 관리에 적합한지 잘 모르겠다.

어찌됐든 회사 일인데 시력을 포기하며 일해서 ALB Rule JSON 디버깅을 마쳤다. 그리고 지하철 역 안내 문구가 잘 보이지 않게 되었다. 진짜 슬펐다.

#### 2. ALB 중간에 오류 났을 때 중간 리소스들이 미아가 된다.

ALB Controller 가 Reconcile 할때 LB 생성 후 Subnet 찾아서 Target Group 생성하고 Rule 을 생성한다.

이러한 일련의 작업 중에서 1. 권한 문제 / 2. Subnet Tag(Auto Discovery) 부재 / 3. Subnet IP 주소 부족(최소 8개) 와 같은 문제가 발생할 시에 중간에 생성된 리소스들이 미아가 된다.

또한 Security Group 도 `kubernets.io/cluster/ks-kr-p03: owned ` 이런식으로 태그가 있어야 한다.

따라서 이어서 작업할 시에 ALB Controller 는 바보멍청이가 되어서 어찌할 줄을 모르게 된다. ([관련 Github 이슈](https://github.com/kubernetes-sigs/aws-load-balancer-controller/issues/1187))

다행히 ALB 버젼 2.0 에서는 해결되었다고 한다 ^^

#### 3. (Kubernetes 보다) 느린 배포

새로운 Pod 이 떴을 때 ALB Controller 가 Kubernetes Event Stream 으로 부터 신호를 받고 AWS API 를 호출하고 이제 이를 Deregsier/Register 하는데에 걸리는 시간이 순수 Kubernetes 의 Rolling Update 에 비해 느렸다.

사실 당연하기도 하다.

Pod 이 하나일 때는 5xx 에러가 발생하며 Pod 이 여러개여도 Zero Downtime 을 위해서 Min Read Thresold 와 PreStop 그리고 maxUnavailable 등을 활용 했다.

지금 ALB Controller 는 더 좋을 듯 싶다. ㅎㅎ 

## 배포 시간

Rolling Update 를 한다면 새로 뜨는 녀석들이 Live/Ready 되어야 이전 Pod 들이 죽는다.

새로운 Pod 을 많이 띄운다면 그만큼 여유 공간(여유 노드)들이 많아야 한다.

그러면 돈이 아깝다..

140대의 Pod 이 떠있다.

Pod 이 새로 떠서 Live & Ready 되려면 보통 3분 정도가 필요 했다.

maxUnavailable 을 25% 로 잡아서 4번의 Rolling 이 일어난다고 하면

4번 * 3분 = 12 분이 걸린다.

그리고 이때 여유 공간은 35개 이상 잡아놔야 한다.

아직 우리 회사는 Cluster Autoscale 이 없었다.

이러한 이유로 여유 공간을 항상 준비해 두기 위해 웹앱 전용으로 노드 그룹을 만들어야 했다.

## 노드 그룹

우리 회사는 EKS 를 Terraform 으로 프로비저닝 한다. 그리고 kubelet args 를 userdata 를 통해 스크립트로 넘긴다.

그래서 나는 Terraform 에서 object for_each 를 통해 모듈을 만들어 아래와 같이 사용했다. 

```
  worker_additional_groups = {
    "webapp" = {
      name             = "webapp"
      instance_type    = "c5.4xlarge"
      desired_capacity = 5
      max_size         = 5
      min_size         = 5
      volume_size      = 50
    },
    "stress" = {
      name             = "stress"
      instance_type    = "c5.4xlarge"
      desired_capacity = 5
      max_size         = 5
      min_size         = 5
      volume_size      = 50
    }
  }
```

그리고 `--kubelet-extra-args` 에서 `--register-with-taints` 로 taint 를 거려는데... deprecated 였다. -_-

보통 deprecated 라고 해놨으면 use instead 가 있어야 하는데 없다.

Kubernetes 문서는 불친절하다.

이전 버전의 API Reference 는 삭제되거나 주소가 이동되는데 이를 찾을 방법이 생각보다 어렵다.

Kubernetes 는 문서에 매우 의존해야 한다. apiVersion 을 어떻게 외우겠는가..

문서 좀만 친절해졌으면 좋겠다 마치 Terraform Provider 처럼..

무튼 deprecated 였지만 그냥 사용했다.. ㅎㅎ


## 마이그레이션 성공

![](https://i.imgur.com/58MAkKo.png)

![](https://i.imgur.com/JIt2j0v.png)

![](https://i.imgur.com/rjLu52v.png)

다행히 운좋게도 한번에 성공했다.

마이그레이션 자체 작업은 어려울 것이 없었다.

ingress.yml 에서 weight 를 조금씩 조절해나가면 되었다.

1%, 5%, 10% 넣으면서 오류가 너무 없길래 바로 50% 100% 을 갔다.

살아있음을 느꼈다. 스릴이 넘쳤다.

Ben, Ted 사수분들이 너무 잘하셔서 옆에서 보고 배운 것으로 이렇게 하게 되었다.

그리고 중간 중간 막히고 어려울 때마다 오셔서 한방에 문제를 해결해 주시곤 했다.

클라우드 인프라는 하나도 몰랐는데 두 분 덕분에 조금이라도 알게 되어 너무 감사하다.


## 다음 작업

이제 글로벌로 나가 있는 웹앱도 같이 정리해야 한다.

다만 이를 위해서 배포 파이프라인을 개선해야 하는데..

현재 내가 했던 업무는 사실 기초적인 마이그레이션, 즉 이사였고 이제 인테리어, 리모델링을 해야할 때다.

더 자동화 되고 모니터링 쉽고 빠르고 간편한 그런 CI/CD 를 생각하면 항상 기분이 좋고 기대가 된다.

더 좋은 CI/CD 를 만들고 공부하고 알아내고 싶다.
