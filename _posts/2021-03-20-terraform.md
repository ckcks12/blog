---
layout: post
title: Terraform
author: Eunchan Lee
categories: [software,internship]
---

첫 회사 면접 때 인프라 경험 없음에 우려를 표하시고 개발 위주의 플랫폼 팀 가는 것이 어떻겠냐 하셨다.

하지만 AWS, 클라우드 인프라 지식이 너무 탐나서 포기할 수 없다 말씀 드렸고 감사하게도 인턴 기회를 주셨다.

입사 첫 날 주어진 과제 : AWS Cloudwatch Schedule Task Admin UI 만들기

Terraform 으로 관리하던 Cloudwatch Schedule Task 를 UI 로 관리하기 위해서 였다. (200개가 넘어 테라폼으로 할때 꽤 힘드셨다고 한다)


## Terraform 과 AWS

[img](https://i.imgur.com/Qfvee9e.png)

당연히 나는 Terraform, AWS 가 둘다 처음이었다.

Terraform 은 쉬웠다.

마치 전생에 다루었던 툴 같았다.

나는 Terraform 의 Declarative 에 매료 되었고 가슴 깊이 이를 받아들였다.

Terraform 은 정적이다.

동적일 수 있는 부분은 모듈과 for_each 뿐이다. 

인프라, 특히 프로비저닝 및 리소스 관리에 있어서 동적인 것은 좋지 않다.

왜냐하면 소스의 흐름을 따라가야 무엇이 생성될 지 알 수 있기 때문이다. 

그 흐름이 복잡해질 수록 인프라 리소스에 대한 사이드 이펙트를 유추하기 매우 어려워진다.

인프라 리소스 사이드 이펙트를 예시로 들면. AWS Elasticache Redis Cluster Instance 를 2개에서 3개로 늘리면서 AZ 를 바꾸려고 했는데 기존에 있던 Redis 를 삭제하고 새로 생성하게 되는 것이다.

또한 기존에 있던 리소스를 재활용 하는 것도 쉽지 않다.

정적인 Terraform 의 모듈은 Depth 가 낮다. 차원이 1차원이다. 대부분 한 눈에 보인다.

하지만 CDK 등과 같이 동적으로 프로그래밍하게 되면. 매우 높은 확률로 질 나쁜 구조를 통해 한가지 일만을 겨우 처리하게 될 수 도 있다. 

Terraform 은 CDK 가 하는 일을 1차원 적으로 Flatten 한 뒤에 이를 Planning 을 통해 사이드 이펙트를 추적 가능하게 한 툴 일 뿐이다.


Terraform 의 단점으로는 자동화하기 어렵다는 점이 있다. 파이프라인 등에 사용하기가 단순 CDK 와 같은 스크립트 라이브러리에 비해 까다롭다. 

또한 협업하기도 쉽지 않다. 그래서 Terraform Cloud 등이 나오게 되었다.

이런 부분은 사실 도구의 단점으로 치부하기보다는 조직의 구조를 되돌아볼 필요가 있다.

아무나 Terraform 을 만진다. 그리고 이때문에 협업에 문제가 생긴다.

그렇기에 Terraform 이 좋지 않다...?

아무나 쉽게 Domain(Route53 Record) 를 생성하게 한다.

이 때문에 gitops 에서 github comit 충돌이 문제가 되고 자동화 파이프라인에 포함될 수 없는 Terraform 이 문제가 된다...?

Terraform 의 경우 plan 이후 이 Actionable 정보를 byte 로 내뿜을 수 있다. 그리고 추후 apply 시 해당 byte 을 인자로 넣으면 plan 에 해당했던 CRUD 작업을 하게 된다.

나는 실제로 그렇게 자동화 시켰었다.




AWS 는 어려웠다. AWS 는 예외가 많다. 화학 같다. 그 예외들을 단순 암기하기엔 양이 많았고 또 쉽게 접하기 어려운 개념이라 말 그대로 밟아봐야 알게 되는 지뢰들이 많았다.

예를 들어 Load Balancer 에 추가될 수 있는 Target 은 500개가 한정이라는 것.

물론 컨택해서 더 늘릴 수 있겠지만.. 저런걸 AWS 문서를 일일이 읽으며 하기에는 AWS 서비스가 너무 많다..

Terraform 은 수학에 가까웠다. 규칙이 있고 원리가 있어 처음에 몇개만 익히면 이후에는 수월했다. (IDE의 자동완성 도움이 필수이긴 했다.)

하지만 AWS 는 마치 (공부해본 적 없지만) 법학과 같았다. 그냥 이유 없이 안되는 것도 많고 복잡하게 생긴 것도 많다.

물론 클라우드는 이전 하드웨어 인프라서부터 점차 발전해오면서 그것들을 추상화하느라 처음 배우기 어려운 부분이 있을거라 생각한다.

그리고.. AWS 는 계속해서 발전하고 확장한다..

AWS 공부하는 와중에 새로운 AWS 서비스/기능이 나온다. 살면서 처음으로 RSS 의 필요성을 느꼈다.


