---
layout: post
title: Dapper, 구글의 Large Scale Distributed System Tracer
author: Eunchan Lee
categories: [software,dapper,google,large,scale,distributed,system,tracer]
---

![](https://i.imgur.com/Tc5Xtii.png)

구글이 2010년 4월 발표한 논문이다. [링크](https://research.google/pubs/pub36356/)

11년 전, 아니 13년 전에 이미 사내에서 쓰고 있었다니..

# Dapper = Span + Annotation

![](https://i.imgur.com/hkKeSPJ.png)

Dapper 는 timestamp 로 span 을 기록하고 annotation 으로 추가 정보를 달아 이를 묶어 root span(trace) 로 관리한다.

위 그림은 Span 을 보여준다.

![](https://i.imgur.com/zVSx6pg.png)

위 그림은 Span 의 Annotation 을 보여준다.

# 서버, 클라의 시간 동기화 어떻게 하나?

> It is important to note that a span can contain information from multiple hosts; in fact, every RPC span contains annotations from both the client and server processes, making two-host spans the most common ones.
> Since the timestamps on client and server come from
> different host machines, we have to be mindful of clock
> skew. In our analysis tools, we take advantage of the fact
> that an RPC client always sends a request before a server
> receives it, and vice versa for the server response. In
> this way, we have a lower and upper bound for the span
> timestamps on the server side of RPCs.

RPC 특성상 (아니 사실 서버-클라 구조 특성상) 클라가 먼저 호출하고 서버가 받고, 서버가 응답하면 클라가 받으니 이걸 응용한다고 한다.

클라의 request 시간을 벗어나는 서버의 response 는 없는 것을 가정한 것이다.

하지만 이때문에 해당 논문에서는 async batch job 등의 케이스에서 trace 가 힘들다고 한다.

Datadog Agent 는 그래서 처음 시작할때 time 을 맞추고 시작하는 것 같아 보였다. (잘 기억 안남)

# Dapper 의 세가지 설계 목표
1. Low Overhead : 느리지 않게
2. Applicaton-level Transparency : 적용하기 쉽도록
3. Scalability : 확장 가능하게

여기서 와닿았던건 2번 적용하기 쉬운 것에 대해서 구글은 RPC 로 거의 통일 되고 C++, JAVA 를 주로 썼기 때문에 라이브러리를 만들어 지원하기 쉬웠다고 한다.

회사는 언어나 라이브러리 선택에 보수적이어야 하는 것 같다.

# Trace 구조

![](https://i.imgur.com/Zy8GiIr.png)

1. Trace Context 를 만든다.
> root span 생성시 204ns (globally unique id 생성때문) 일반 span 176ns
2. Thread-local Storage 에 넣는다. (async callback 에서도 context 복원하여 trace 를 이어나간다)
> 넣고 빼고 하는데 9ns
3. trace 정보를 채운다. (timestamp, annotation)
> annotation 넣는데 40ns
4. log file 로 i/o write 한다.
5. host machine 에서 log file 수집하여 collector 에게 보낸다.
6. collector 가 Bigtable 에 보낸다.

1~6 의 작업 즉 data collection 의 median latency 는 15초 정도라고 한다.

# Sampling

![](https://i.imgur.com/awfdO73.png)

Google Web Search 는 매우 많은 요청이 있어서 모든 요청을 trace 하면 부하가 심하다.

1/1024 정도로 할때 괜찮았다고 한다.

더 발전해서는 adaptive sampling 으로 요청 많을 때는 드물게, 요청 적을 때는 잦게 sampling 한다.

또한 collector 에서 Bigtable 에 보낼 때도 trace id 를 hash 해서 0 < z <= 1 해서 collection sample coefficient 보다 크면 버리고 하는 등 sampling 한다고 한다.

# Sampling 에 대한 걱정

Dapper 처음 쓰는 구글러들은 너무 적게 수집하면 분석이 되겠냐 걱정했단다.

너무 적게 sampling => 문제가 있으면 적은 sampling 에도 잡히기 마련이다.

사실 논문 시작 부분에도 "경험을 통해 깨달은 것은 수천개 중의 한 trace 만으로도 충분한 정보를 얻을 수 있었다" 고 했다.

# 보안을 위해 payload 를 버린다

계좌 정보, 배송지 정보 등.. 이런 정보는 trace 되면 안된다.

대신에 annotation 등으로 디버깅에 도움이 될 정보를 넣으라고 한다.

# 보안 점검이 가능하다

gRPC 를 예를 들어 withInsecure() 와 같은 protocol 레벨의 보안 취약점을 같이 수집하여 본다고 한다.

# Dapper 의 부족한 점

1. firefighting, 빠르게 대처해야 할때 샘플링을 바로 얻고 싶을 때, sahred storage service 와 같은 경우에 유용하지 못하다.
2. 여러 trace 가 한 span 을 공유해서 가질 수 없다. (Coalescing effects)
graphql batchloader 의 경우 여러 request 가 한 span 을 바라보게 될 수 있다. 이러한 경우에 Daper 의 span tree 구조로 표현하기 어렵다고 한다.
3. batch 에 적합하지 않다.
위에서도 언급했듯 서버-클라 구조가 설계 뿌리부터 박혀있기 때문에 ㅎㅎ;;
4. root cause 찾기 어렵다.
다른 request 때문에 queue 가 밀려서 처리가 느려질 수 있는데 이런 문제를 찾아내기 쉽지 않다. (Datadog 도 그러하다.)
또한 kernel level 의 정보도 필요한데 이 부분이 지원되지 않는다고 한다. (file description, cpu i/o, epoll 등)

# 느낀점

책 읽는 것보다 논문이 훨씬 낫다. 액기스만 담겨져 있다.

구글의 논문은 특히다 경험 기반 + lessons learned 가 있어 더 좋다.

지금 회사에서 graphql batchloader 를 쓰는데 이 부분을 11년전에 이미 예상하고 논문을 쓰고 있었다니..

아마 죽어도 이 구글러들을 못따라가겠다 싶다..

