---
layout: post
title: 검색 색인기 Search Indexer 고언어 Golang 개발기
author: Eunchan Lee
categories: [software,aws,es,elasticsearch,search,indexer,indexing,go,golang]
---

# 검색 색인기(Indexer)가 하는 일

각 서비스 팀의 데이터를 검색 엔진에 추가/업데이트 하여 검색 될 수 있도록 합니다.

# 검색 색인기에서 중요한 부분 3가지

1. 안정성

    개복치마냥 툭하면 꺼지고 그러면 안되겠죠!

    뿐만 아니라 너무 성능이 좋아서(?) Write 가 너무 빠르면 ES 가 부담을 느낄 수 있죠.

    그래서 Fail Tolerance, Adaptive Write 가 필요합니다.

2. 정합성

    필드가 잘못 색인 된다거나.. 인코딩이 깨진다거나 하면 안되겠죠?

    유닛 테스트는 사실 거진 쓸모가 없고.

    Integration Test 가 필요합니다.

3. 속도

    현재 2021년 3월 27일 기준 우리 회사 글 수는 2억개 입니다.

    이거 평균 초당 만개 넣어도 총 대여섯시간이 걸려요.

    이 때문에 검색 퀄리티 반영이 늦어지고 CS 처리가 늦어질 수 있어요.

    이런 속도를 줄여나가게 되면 결국 회사의 성장이 빨라지는 것으로 이어지게 됩니다.

# 세번이나 새로 짰습니다

v1, v2, v3 로 나누어 각 버전의 구현과 장점 그리고 문제점을 보여드리겠습니다.

# 검색 색인기 V1

![](https://i.imgur.com/YNiURkc.png)

1. 각 서비스팀의 DB 에 직접 SQL 실행
2. Fetch
3. ES 에 Bulk Indexing

일단 배경은

이전에는 인덱서 개념이 없고 각 서비스팀에서 ES 에 직접 upsert 하고 있었습니다.

ES 가 많이 힘들어 했습니다. 거의 CPU 10~30% 정도를 먹는다 보시면 됩니다.

이 부분을 빠르게 개선하고자 거진 Logstash 수준으로 짜게 됐습니다.

### SQL

```sql
SELECT id, title, content, user_id
FROM articles
WHERE '2021-03-27T00:00:00' <= updated_at
			AND updated_at < '2021-03-27T00:01:00'
```

이런식으로 `updated_at` 필드를 사용해서 색인 했습니다.

이때 문제점은

1. 전체 재색인, 증분 작업 어려움
2. 재시도 없음, 속도 느림 등 완성도 떨어짐

전체 재색인, 증분 색인 작업이 어려웠습니다.

증분 색인이란전체도 최신도 아닌 중간 부분 다시 색인한다 보시면 됩니다.

그리고 재시도 등.. 사실 PoC 프로토 타이핑에 불과했습니다.

그냥 버전이 2개면 일 많이 안한 것 같아서 버전 3개 하려고 창피하지만 넣어봤습니다.

# 검색 색인기 V2

![](https://i.imgur.com/ixkvDzu.png)

1. CloudWatch Schedule Task 를 활용해 Lambda 호출
2. Lambda 에서 Indexer HTTP Server 에 Indexing 명령
3. SQS Push
4. SQS Pull
5. Indexing 명령에 따라 DB 접근, Fetch
6. ES 에 색인

### 왜 CloudWatch ?

Schedule 하는 영역은 따로 뺀 이유는 안정성 때문입니다. 스스로 하게된다면

1. 혼자 DeadLock 등의 문제 빠졌을 경우... Stop the world
2. Scale out 일 때 scheduling... 질서 없어 통제 불가능

이런 문제가 발생합니다.

### 왜 Lambda 서 Queue 에 직접 넣지 않았느냐?

우리 회사는 글로벌 서비스라서 위 색인기는 총 8개 환경에 배포됩니다.

그 모든 애들에게 인프라 세팅 해주는게 번거로웠습니다.

심지어 알파와 프로덕션은 계정 자체가 달라서 더욱 더 번거로웠죠.

HTTP Server 를 열어 두면 또 스크립트로 활용하기 편합니다.

### V1 문제점 해결 - 전체 재색인, 증분 색인 등

V2 는 from, to 를 설정 가능하게 하여 전체 재색인, 증분 색인이 가능했습니다.

### V1 문제점 해결 - 재시도

SQS 썼으니 뭐..ㅎㅎ 당연하게 구현됐죠. DLQ 랑 같이 해서 알림도 받고.. 네.

### V1 문제점 해결 - 속도

Golang 의 최대 장점인 Goroutine 을 십분 활용 했습니다.

![](https://i.imgur.com/oTbGPbi.png)

DB 에서 SQL 해서 Fetch 하면 Rows 나오죠.

Rows 는 사실 데이터가 실제로 온 것은 아니고 하나의 Cursor 죠.

Cursor Next (Rows.Next()) 해줘야지 가져오기 시작하죠.

사실 이걸 멀티 컨슘한다고 빨라지진 않죠.

이걸 빠르게 하고 싶으면 Query 튜닝, 멀티 쿼리 해야죠. Task Split 이 필요합니다.

~~Spring Batch 라면 이제 Partitioner 로 ㅎㅎ~~

그럼 이제 제가 한건 여기서 Scan 후에 일어나는 작업을 최적화 한 것 입니다.

![](https://i.imgur.com/z9sXgLd.png)

Rows 를 Next() 해가며 Scan 해서 DAO 를 만드는 것에 Goroutine 을 썼어요.

Channel 로 묶고.. WaitGroup 과 Mutex 를 적절히 쓰며..

사실 보여드릴 필요도 없는 너무 간단하고 평범한.. 모두가 다 쓰는 소스죠.. 부끄럽네요.. 별 것도 아닌 걸로..

자 이제 메모리상의 DAO 객체를 JSON 으로 꾸워주어야 합니다.

JSON 은 웹서버들의 전체 CPU 사용량 중 30% 를 차지한다는 꽤나 무거운 CPU Intensive 작업입니다.

바로 Goroutine 발라줍니다.

![](https://i.imgur.com/wzS2dc8.png)

Go 해보신 분은 아시겠지만 `er` 접미사 붙으면 Interface 입니다.

저는 모든 DAO 를 `Stringer` 로 만들었어요.

JSON 으로 Marshal 할 때 비즈니스 로직으로 nullable 이나 기본 값 등이 들어가기 때문에 직접 DAO 자신이 신경 쓰도록 해야 했어요.

DAO `Stringer` 관련해서 간단한 예시 코드를 드리면 이런 식입니다.

바로 옆에 있기 때문에 수정도 용이하고 디버깅도 매우 편리합니다.

```go
var _ Stringer = &ArticleDAO{}
type ArticleDAO struct {
	ID         int64
	Title      string
	Content    string
	WatchCount int64
}

func (dao *ArticleDAO) String() (string, error) {
  // 이 곳에서 기본값이나 nullable 혹은 field 추가/가공을 할 수 있습니다.
	b, err := json.Marshal(dao)
	if err != nil {
		return "", err
	}
	return string(b), nil
}

```

이렇게 JSON Marshal 도 Goroutine 으로 빠방하게 진행했습니다.

Indexer 를 작동시킬 때 CPU 가 100% 를 찍으며 살려줘 외칠 때가 가장 뿌듯했습니다.

마지막으로 ES 에 Bulk 로 쏴주는 것도. 당연히 Goroutine.

![](https://i.imgur.com/1pY0FwH.png)

하지만 위 Scan 이나 JSON 때 처럼 빡세게 쓰진 못했어요.

Bulk 를 Goroutine 두어개로만 실행해도 ES 가 죽을라 했어요.

ES 에 HTTP 로 전송하는 Body 의 양은 기본값으로 100MiB 가 최대에요.

보통 3초 이내로 100MiB 가 채워져서 날라가는데 이게 양이 최소 50만개의 문서에요.

HTTP Response 에만 몇 초가 걸려요. 그리고 그게 반복 되면 ES CPU 100% 찍고 정신 못차리더군요.

(Replica 0, Refresh -1, Translog Size 20GiB 상태로 진행했습니다)

![](https://i.imgur.com/T3ycVzZ.png)

결론적으로 이런식으로 해서 속도는 DB 병목을 제외하고는 최적화를 시켰습니다.

완성 직후엔 완벽했다 생각했지만

몇개월간 사용해보니 V2 의 문제점이 보였어요.

1. DB 부하 : from, to 전체 색인시 매우 불리

    ```sql
    SELECT id, title, content, user_id
    FROM articles
    WHERE '$from' <= updated_at
    			AND updated_at < '$to'
    ```

    이 때 전체 색인을 위해 from 과 to 를 회사 설립일과 now() 로 하게 된다면

    ```sql
    SELECT id, title, content, user_id
    FROM articles
    WHERE '2015-06-01' <= updated_at
    			AND updated_at < now()
    ```

    이렇게 되는데요, 사실 where 문이 없어도 되는 거라서 부하가 간다고 합니다.

    (실제로 DB 상에는 테스트 등으로 저 기간 외 값도 있어서 sequential read 하면서 where 조건으로 읽고 버리기 때문에..)

    또한 서비스에서는 작업을 하다보면 일괄 업데이트를 할 때가 있습니다.

    즉 100만개의 레코드 중 80만개가 5분 이내로 업데이트가 되었어요.

    이러면 이제 현재 구조(현재 쿼리)로는 박살나는 겁니다.

    쿼리도 엄청 느려집니다. 위와 같이 sequential read 하면서 일일이 비교하고 버리고..

    암튼 네 그렇습니다.. 그래서 이런 목적에 맞게 쿼리를 나누어야겠더라구요.

2. Datasource 가 여러개일때? (2개 이상의 DB, Redis)

    보통 글의 관심 수 등은 중복 처리 등을 위해서 Redis 를 많이 사용하죠.

    그동안 DB 만 읽으려 했기 때문에 그리고 또 한 DB 만 읽으려 했기 때문에

    2개 이상의 DB 그리고 Redis 참조하기 어려운 구조입니다.

3. 중간 매개체 필요 : 매번 항상 DB 를 참조할 수 없음

    검색팀은 상시로 전체 색인을 합니다.

    Lucene 의 segment 정리할 겸, 사전 업데이트 반영할 겸.

    그리고 그전에 없던 field 를 색인할 겸 해서 전체 재색인을 하는데

    그때마다 서비스팀의 DB 를 긁으면 그 분들이 마음 아파합니다.

    실제로 위에서 설명드린 레코드 80%가 5분이내로 업데이트가 되어서

    updated_at 으로 쿼리하다가 DB 가 CPU 100% 되버려서 서비스에 지장이 갈뻔 했습니다.

    그래서 중간 매개체로 DB나 File 를 사용하여

    색인은 그곳에서 빠르게 긁어오도록 하는 것이 필요했어요.

4. 정합성 테스트

    유저에게 서빙되는 매우 중요한 데이터를 다루기 때문에 이제 값의 정확성을 테스트 해야 했어요.

    현재 색인기의 경우 SQL 과 DAO Scan 에 대한 테스트가 필요했습니다.

# 검색 색인기 V3

![](https://i.imgur.com/nxu9gnx.png)

그림이 뭔가 복잡하네요.

V2 와 달라진 점은 그냥 중간에 S3 생겨서

Crawling 과 Indexing 두 단계로 나뉘어졌다는 점 입니다.

Crawling 은 순수하게 DB 에서 S3 까지.

Indexing 도 순수하게 S3 에서 ES 까지.

### V2 문제점 - DB 부하 (쿼리 최적화)

쿼리도 이제 Full / Recent 두 가지 버젼으로 나누어서 두가지에 맞춰서 최적화 했습니다.

Full 의 경우 전체 id 개수 가져와 id 로 partitioning 합니다.

```sql
SELECT max(id) as max from articles
```

해서 가져온 다음에 적절히 BATCH_SIZE 로 잘 나누어서 Partitioning 해서 Iterator 를 만듭니다.

이거 하는데 생각보다 좀 손이 많이 가더군요 ㅎㅎ.

실시간 색인을 위해 `updated_at`

전체 색인을 위해 `id` partitioning 을 하게 되었습니다.

### V2 문제점 - Datasource 가 여러개일때? (2개 이상의 DB, Redis)

Iterator 패턴을 사용해서 각 도메인(컬렉션) 마다 알아서 캡슐화 하게 끔해서

알아서 `Next()` 에서 하나의 DB가 끝나면 seamless 하게 다음 DB 가던지

자기 원하는대로 Redis 를 중간에 참조하던지 하게 끔 했습니다.

```go
func (it *ArticleIterator) Next() (json string, done bool, err error) {
  var ok bool
	if it.rows == nil || !it.rows.Next() {
		it.rows, ok = it.nextDBRows() // 이곳에서 DB Switch
		if !ok {
			return "", true, nil
		}
		return it.Next()
	}
	
	dao := ArticleDAO{}
	err = it.rows.Scan(&dao)
	if err != nil {
		return "", false, err
	}
	
	json, err = dao.String()
	if err != nil {
		return "", false, err
	}
	
	return json, false, nil
}
```

위 소스는 대충 쓴 거라서 안돌아갈 수 있습니다 ㅋㅋ..

디자인 패턴을 쓰려고 쓴건 아닌데 하다보니 엥 이게 Iterator 네 하고 이름을 Iterator 로 바꾸게 됐네요.

### V2 문제점 - 중간 매개체 필요 : 매번 항상 DB 를 참조할 수 없음

![](https://i.imgur.com/frq6lYL.png)

S3 에 NDJSON 으로 쌓아 두었습니다.

그래서 중간에 색인이 잘못되었거나 잘못된 유니코드가 들어있다던지 등을 여기 중간에서 확인해볼 수 있고

머신러닝 등의 작업에서 유용하게 가져다 쓸 수 있었습니다.

사실 이 부분은 아직 아쉬움이 남습니다.

S3 가 아니라 DB 였어야 했는데..

S3 로 하게 됐을 때 문제는 관리가 좀 불편해요.

파일 저거 엄청 쌓여요. 50MiB 짜리가 2100개 정도 됩니다.

그렇게 매일 새벽 4시에 쌓이고..

Retention Policy 로 Glacier 및 Version 삭제 등은 해놨는데 그래도 불편합니다..

왜 50MiB 냐구요?

저거 S3 가 Stream 이 안돼서 저거 주고 받고 할 때 다 메모리에 들고 있어야 하다보니..

그리고 저거 통째로 받자마자 ES 에 쏘려고 했는데

아시다시피 ES 100MiB 이하로만 body 받아서

50MiB 에 bulk indexing metadata 넣어주면 꽤 뜁니다.

그리고 100MiB 꽉 채워 보내는것보다 25~50MiB 씩 주는 것이 색인에 더 빠르더라구요 ㅎㅎ

흠.. 500MiB 로 좀 사이즈 늘리고 해볼법도하긴 하지만.. 

그리고 또 Stream 이 안되다보니 매번 저걸 다운로드해야해요. 그게 생각보다 꽤 오래 걸립니다.. 3~7초는 나오더라구요... DB Stream 은 그보다는 빠를 것 같네요..

DB 가 아니다보니 또 생각보다 활용하는데에 한계가 있어요. 

Spring 처럼 S3 Object 도 ResourceLoader 로 불러올 수 있는게 아니면...

다들 AWS SDK 사용해서 삽질 조금 하고 뭐 하는 등.. 좀 불편했어요.

DB 로 안한 이유는..

제 결정은 아니구요.. 팀장님께서 그렇게 하라고 하셔서..

팀원들은 모두 DB로 하고 싶어했어요..

그치만 일단 팀장님의 경험과 경력 실력을 믿고 S3 로 하게 됐어요.

이 글을 보시는 다른 분들은 DB 로 하셨으면 좋겠어요. 특히 Schema-less 몽고디비 같은걸루요.

검색 서비스는 field 가 많이 바뀌니.. ㅎㅎ ML 통해서 field 가 더 추가될 수도 있구요.

### V2 문제점 - 정합성 테스트

Docker Container 띄워서 DB, ES 그리고 Local Stack 으로 SQS, S3 까지 띄워서

Full Integration Test 했어요.

좋더라구요.ㅎㅎ

테스트 코드로 직접 Table, Records 만들고 색인 결과 값을 Answer 와 비교하고..ㅎㅎ

# V3 문제점 및 다음 개선에 기대하는 것

1. 색인기가 직접 서비스팀의 DB 를 긁어 오면 안된다.

    사실 사내에서도 말이 많았어요.

    이 부분도 저를 포함해 팀원들은 모두 반대했지만

    팀장님의 경험을 전적으로 믿고 진행한 부분이거든요.

    막상 해보니까 어떤 문제가 있냐면요.

    서비스 팀이 DB Column 을 바꿀 때 장애가 나요.

    서비스 팀에서는 더 이상 안쓰는 Column 이다 생각하고 지웠는데

    알고보니 색인기의 쿼리에서는 계속 사용하고 있던거죠.

    그럼 서비스팀이 매번 바꿀 때마다 검색팀에게 알려주어야 하는데

    우리 회사 조직 문화가 아무래도 그런 프로세스가 확립되어 있지 않아서 힘들었어요.

    그 뿐 아니라 서비스팀의 DB가 Postgres 에서 MySQL 로 옮기기도 했어요.

    혹은 DB Column 하나를 Redis 를 사용해서 역할 위임을 했다던지.

    혹은 DB Column 이 그대로 사용되는 것이 아니라 내부 로직을 통해 계산 된다던지

    (예를 들면 채팅수 = (유저채팅수+그룹채팅수)/2 이렇게 쓰이는 경우)

    즉 DB 를 직접 긁어온다는 것은 어떻게 보면 그 팀의 비즈니스 로직을 일부 구현해야 한다는 뜻이 되더라구요.

    그게 생각보다 큰 부담입니다.

    부하는 둘째고 서비스 발전하는 부분에 있어서 발목을 잡게 돼요.

    그래서 이보다는 각 서비스팀이 카프카를 통하던 DB에 직접 쓰던 해서

    서비스팀이 Produce 하는 방식이 좋은 것 같아요.

    각 팀에서 구현을 하는데에 부담이 갈 수 있다 생각하지만

    회사가 점차 커지며 서로의 데이터를 필요로 할 때가 많아요.

    결국 이것은 회사의 소프트웨어 시스템 고도화에 있어서 언젠간 해야하는 스텝이라 생각 돼요.

    서로 자신의 서비스를 다른 서비스가 활용하기 쉽도록 가공 해주는 것 말이죠.

2. 중간 매개체로 DB를 써야한다.

    이 부분도 참 말이 많았는데요. 물론 저를 포함해 팀원들은 같은 생각이긴 했어요.

    DB를 반대하신 이유는 관리 포인트 때문이었어요. 

    그래서 S3 를 사용해보았는데 Stream 지원 안되고 매번 다운로드 해야하고

    활용해서 분석하기 번거롭고 ML 등을 통해서 중간 수정 혹은 수동 수정이 어려웠어요.

    위 단점들이 DB를 쓰게 되면 해결이 됩니다.

3. Go 언어는 이 프로젝트에 적합하지 않다.

    이 부분에 있어서 할 말이 많아서 따로 글을 씁니다.

    DB(SQL) 작업과 Batch 성 작업 등에 적합하지 못했어요.

    위에서 Goroutine 을 잘 활용하지 않았냐 하실 수 있는데

    사실 Goroutine 쓰게 되면 WaitGroup, Mutex, Channel 등을 쓰다보면

    은근히 또 손이 많이가요.

    Spring Batch 와 비교해보시면 좋을 듯 합니다.

    단순히 구현에서 끝날 것 아니잖아요. 토이프로젝트도 아니고.

    Enterprise 패턴이 무엇인고 하면 뭐 쓰기 좋고 빠른 프레임워크를 뜻하는게 아닌 것 같더라구요.

    Logging, Retry, Discoverability 등 아주 사소하지만 그 edge 가 살아있는 부분들이

    그것들이 바로 Enterprise 급 제품이더라구요.

    그리고 그런 Enterprise 급 제품을 만들기에는 Go 가 적합하지 못합니다.

    너무 많은 것을 직접 짜야하는데

    그렇다고 이것을 패턴화 시키기에도 언어적인 한계가 큽니다.

    서로 잘 공유하지 않는, 모든 것을 자신이 직접 만들려고하는 Go 생태계? 분위기 자체도 한 몫 하구요.

# 끝마치며

![](https://i.imgur.com/zVD3F2V.png)

현재 V3 로 색인 하고 있구요. Datadog 을 통해서 ES 의 실시간 색인량 그래프를 그려보았어요.

1분마다 Scheduling 되어서 2분 어치씩 색인이 되고 있어요. 

즉 1분에 약 8천개 정도의 글이 색인이 되네요. (현재 2021.03.27 15:33. 피크 대비 60%)

흠.. 원체 글 솜씨도 없고.. 한 큐에 글을 쓰다보니 놓치거나 하는 부분이 많네요.

Fail Tolerance 에 관해 알림에다가 retry command 넣어 편의성 도모 했던 부분이나

ES indexing 할때 팁을 더 적으려 했는데... (중간에 지나가며 조금씩 적긴 했어요)

이게 생각보다 손이 매우 많이가고 귀찮고 하네요...

혹시 제 글에서 더 궁금한 점이나 혹은 가르쳐 주실 부분 있으시다면 언제든지 매일

3unch4n@gmail.com 으로 부탁드려요.

그리고 제 글이 부디 한분께라도 도움이 되었으면 좋겠어요. 그러려고 쓴 글이니까요..

감사합니다.
