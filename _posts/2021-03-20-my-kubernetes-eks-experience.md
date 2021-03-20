---
layout: post
title: My Kubernetes(EKS) Experience
title: 나의 쿠버네티스(EKS) 경험
author: Eunchan Lee
categories: [software,eks,kubernetes]
---

2개월의 인턴 기간동안 Terraform, AWS, Kubernetes 를 공부하면서 인턴 기간이 끝나가는 지도 모른채 이거 저거 배포 등 일만 하다가 어찌 좋게 봐 주셨는지 같이 일할 기회를 주셨다.

인턴과 정규직으로서의 업무 차이는 따로 없었다, 어제 하던 배포 작업 이어서 했다.

아래는 내가 회사 업무에서 쿠버네티스(EKS) 관련하여 했던 업무(경험)들이다.

## 인프라 비용 아끼기
스타트업 답지 않게? 넉넉한 자본이 있어 인프라 요금을 신경 안쓰고 사용하고 있었다

그치만 너무 많이 나올 때는 한번씩 정리를 하곤 했다

쿠베 클러스터 자원 활용성 개선이 필요했다

 인스턴스 낮추고 앱 resource 와 replica 정리했다

자원 사용량 예상 및 측정을 위해 qos 도 guarenteed 로 했다

기존에 나와있는 다른 툴들에서 limit 을 모아서 보여주진 않아서

limit 기준으로 정리하기 위해서 직접 go 로 프로그램을 만들어 간단하게 그래프를 그려가며 작업했다

![img](https://i.imgur.com/t1W1h6f.png)

위에서 보듯이 limit share 가 46% 도 안차고 있다

cpu 는 throttling 걸리지만 memory 는 oom kill 이기 때문에 memory 위주로 보게 됐다

![img](https://i.imgur.com/7hzFaMy.png)

우선 resource request/limit 좀 정리했다

알파에서는 욕심 부리던 앱이 많아서 혼내줬고 프로덕션에서는 replica 만 건드렸다

![img](https://i.imgur.com/bE6EL9O.png)

이제 자원 활용성이 46% 이하인 노드는 죽였다 그래서 줄인 금액은 아래와 같다 (알파기준)

이전 : c5.xlarge(25) = $138 * 25 = $3456 (400만원/월)

이후 : m5.xlarge(6) = $169 * 6 = $1019 (115만원/월)

총 $2437(275만원/월) 정도 아끼게 됐다.

c 에서 m 으로 간 이유는, 알파는 cpu 보단 memory 가 더필요하다

성능 테스트는 그때마다 필요한 노드 그룹 띄우면 된다

앱 가동 시 더 큰 requirements 를 갖는 건 memory 이기 때문이다



## Kubernetes - kustomization as a Template Engine

지금까지 내 손으로 배포한 앱 수는 27개, 하면서 느낀게 kustomization 편했는데 템플릿 기능이 부족했다

kustomization 1.15 를 helm 마냥 템플릿으로 쓰려고

`comonAnnotation` `var reference` 등을 활용해봤지만

validation 때 yaml attribute 는 type 이 있기 때문에 안된다..

그냥 skeleton boilerplate 가 낫다 😅



## Kubernetes - istio envoy sidecar → kiali

서비스 매쉬 공부해보면서 내부 네트워크 트래킹이 필요하다 생각했다

istio sidecar 가 admission hook 에 안되어 있었다 (np default 에만 적용되어있었다)

그걸 잘 몰라서 그냥 sidecar injection tool 을 사용했다

그런데 막 생성된 소스가 너무 복잡 했고 무엇보다 오류가 잘났다 됐다가 안됐다가..

이쪽에 더 리소스 쓸 수 가없어 패스했다


## Kubernetes - storage usage

어느 날 앱이 잘 안떠서 event 보고 했더니 inode 어쩌고 했다

EBS 볼륨 하드웨어적인 문젠가? 심각한 표정 지었는데

그냥 docker image 가 무거운게 많이 쌓여서 그런거란다

20G → 50G 로 늘려서 해결됐다



## Kubernetes - IRSA

쿠베 앱은 aws sdk 쓸 때 AWS ACCESS KEY 를 쓰지 않고

더 보안적으로 안전하고 개발하기에도 편리해서 사용하게 됐다

aws sdk 인증 우선순위 중에서 거의 맨마지막에 있는 web token 방식이다

aws iam role 을 만들고 해당 arn 을 service account 의 annotation 에 달면

operator 가 token 받아와 file 로 마운트해준다 + 환경변수도

이때 이 file 을 로컬에 가져오면 디버깅도 가능하다 🤫


## Kubernetes - Logging Operator

fluentd 를 올려야해서 이를 operator 로 잘 만들어둔 logging operator 를 쓰려 했다

하지만 디버깅도 잘 안되고 뭔가 안되는게 많았다

용감하게 소스를 까봤다, 소스 까보는게 제일 좋았다, Go 라서 읽기 쉬웠다

file output 에는 테스트 코드도 없고 plugin 설치에 대한 확장 유연성이 떨어졌다

직접 docker 로 굽기엔 부담 됐다

사용을 철회하고 직접 daemonset 으로 fluentd forwarder 올렸다


## Kubernetes - External Secrets Hang 버그

gitops 이기에 secret 들을 yaml 로 올릴 수 없었다

aws 의 ssm parameter store 를 활용했다

godaddy 의 external secrets 을 사용했는데 문제가 있었다

한달 반 정도 지나면 갑자기 hang 걸려서 작동하지 않았다

용감하게 또 소스를 까봤는데

쿠베 이벤트 스트림을 파이프로 연결해서 리스닝할 때 deathQueue 에까지 도달 못하는 것 같았다

`error` 와 `end` event 에서만 deathQueue 에 enqueue 하는데

저 두 event 는 호출되지 않지만 쿠베 stream 이 작동안하는 행걸린 상태..

그냥 event 새로 파도 좋으니 좀 자주 파면 좋았을 것 같다 😳

그래서 그냥 작동안할 때마다 재시작했다


```js
try {
    while (true) {
      logger.debug('Starting watch stream')

      const stream = kubeClient
        .apis[customResourceManifest.spec.group]
        .v1.watch[customResourceManifest.spec.names.plural]
        .getStream()

      const jsonStream = new JSONStream()
      stream.pipe(jsonStream)

      jsonStream.on('data', eventQueue.put)

      jsonStream.on('error', (err) => {
        logger.warn(err, 'Got error on stream')
        deathQueue.put('ERROR')
      })

      jsonStream.on('end', () => {
        deathQueue.put('END')
      })

      await deathQueue.take()

      logger.debug('Stopping watch stream')
      eventQueue.put({ type: 'DELETED_ALL' })

      stream.abort()
    }
  } catch (err) {
    logger.error(err, 'Watcher crashed')
  }
}
```


## Kubernetes - External Secrets SSM 변경사항 자동 반영

AWS SSM Parameter Store 에서 값을 바꾸어도 쿠베엔 반영되지 않았다

sns 로 리스닝하고 관련된 key 값이 있는 CRD 를 retouch 해주면 될 것 같다


## AWS - Elasticache Redis Memory Usage Percentage

레디스 남은 메모리 용량은 CW의 Freeable Memory 지표로는 못가져온다 (부정확함

이를 위해 Datadog Agent 를 따로 연결하여 계산하고 그랬다

그 작업을 한지 이틀만엔가 AWS 에서 새로운 지표를 지원해줬다 🤯

계속 변하는 AWS , 계속 공부해야한다
