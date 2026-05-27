---
title: "01. 단일 서버에서 수백만 사용자까지"
description: "single server, load balancer, database replication, cache, CDN, sharding, stateless"
summary: "단일 서버 구성에서 시작해 수백만 사용자를 감당하는 시스템으로 점진적으로 확장하는 과정을 다룬 글입니다."
tags: ['system design', '시스템 디자인', '네트워크']
categories: ['study']
author: "ljweel"
date: 2026-04-26
draft: false
---

<!--
접기 기능
{{% collapse summary="" openByDefault=false %}}
{{% /collapse %}}

메모 기능
{{< memo "제목" "내용" >}}

코드블럭 기능 (제목 추가 / 접기 기능 추가)
```txt {title="파일명" openByDefault="true"}
```

diff 기능
{{< codediff title="" openByDefault="false" >}}
{{< before >}}
{{< /before >}}
{{< after >}}
{{< /after >}}
{{< /codediff >}}

터미널 명령어
{{< terminal >}}
$ 명령어
{{< /terminal >}}
-->

## 단일 서버에서 수백만 사용자까지

시스템은 처음부터 복잡하게 설계되지 않는다. 단일 서버에서 시작해 병목이 드러날 때마다 한 단계씩 구조를 개선해 나가는 것이 현실적인 접근 방식이다.

---

## 단일 서버 구성

모든 것이 하나의 서버 위에서 동작한다. 웹 앱, 데이터베이스, 캐시가 함께 올라가 있는 가장 단순한 형태이다.

요청 흐름은 다음과 같다.

```
User -> DNS 조회 -> IP 반환 -> HTTP 요청 -> Web Server -> HTML/JSON 응답
```

트래픽 출처는 두 가지다. 웹 애플리케이션은 서버 사이드 언어(Java, Python 등)와 클라이언트 사이드 언어(HTML, JavaScript)를 조합해 동작한다. 모바일 애플리케이션은 HTTP 프로토콜로 서버와 통신하며, JSON 형식으로 데이터를 주고받는다.

---

## 데이터베이스 분리

사용자가 늘어나면 웹 서버와 데이터베이스를 분리해야 한다. 각각을 독립적으로 확장할 수 있게 되기 때문이다.

데이터베이스는 크게 두 종류로 나뉜다. **관계형 데이터베이스(RDBMS)** 는 테이블과 행으로 데이터를 저장하며 SQL 조인을 지원한다. MySQL, PostgreSQL 등이 대표적이다. **비관계형 데이터베이스(NoSQL)** 는 키-값, 그래프, 컬럼, 문서 저장소로 나뉘며 조인을 지원하지 않는 대신 확장성이 뛰어나다.

**NoSQL을 선택하는 경우**: 매우 낮은 레이턴시가 필요하거나, 데이터가 {{< memo "비정형" "관계형 DB의 테이블처럼 행/열로 딱 떨어지지 않는 데이터 예를 들어 사용자마다 가진 필드가 다르거나, SNS 게시글처럼 구조가 제각각인 경우" >}}이거나, JSON/XML 등의 {{< memo "직렬화" "메모리 안의 객체(예: Python dict, Java 객체)를 네트워크로 전송하거나 저장할 수 있는 형태(JSON, XML 등 바이트 문자열)로 변환하는 것" >}}만 필요하거나, 방대한 양의 데이터를 저장해야 하는 경우에 적합하다.

---

## 수직 확장 vs 수평 확장

**수직 확장(scale up)** 은 기존 서버에 CPU, RAM 등을 추가하는 방식이다. 단순하지만 한계가 명확하다. 물리적 한계가 있고, 서버 하나가 다운되면 전체 서비스가 중단된다.

**수평 확장(scale out)** 은 서버를 여러 대 추가하는 방식이다. 대규모 서비스에서는 수평 확장이 현실적인 선택이다.

수평 확장을 도입하면 **load balancer**가 필요해진다. load balancer는 들어오는 트래픽을 여러 웹 서버에 균등하게 분산시킨다. 사용자는 load balancer의 public IP에만 접근하고, 웹 서버는 private IP로만 통신한다. 이를 통해 {{< memo "failover" "장애가 났을 때 예비 서버로 자동 전환되는 것" >}}와 {{< memo "가용성" "가용성(availability)은 서비스가 정상적으로 응답 가능한 시간의 비율" >}}이 확보된다.

```
User -> Load Balancer (public IP)
                ├── Server 1 (private IP: 10.0.0.1)
                └── Server 2 (private IP: 10.0.0.2)
```

---

## 데이터베이스 복제

단일 데이터베이스는 SPOF이다. SPOF(단일 장애 지점)이란 시스템의 일부로서, 해당 부분이 고장 나면 전체 시스템의 작동이 중단되는 부분을 뜻한다.
**database replication** 은 master/slave 구조로 이를 해결한다.

- **master DB**: 쓰기(write) 전용. insert, update, delete 요청을 처리한다.
- **slave DB**: 읽기(read) 전용. master로부터 데이터를 복제받는다.

대부분의 서비스는 읽기 비율이 쓰기보다 훨씬 높기 때문에, slave를 여러 대 두어 읽기 부하를 분산시킨다.  

장애 시 동작 방식은 다음과 같다.

1. slave가 다운되면 읽기 요청이 master로 임시 전환되고, 새 slave가 교체된다.  
2. master가 다운되면 slave 중 하나가 새 master로 승격된다.  
3. 다만 slave의 데이터가 최신 상태가 아닐 수 있어, 데이터 복구 스크립트가 필요할 수 있다.


복제의 이점은 세 가지다. 
- 읽기를 분산해 **성능**이 향상된다. 
- 여러 위치에 데이터가 복제되어 **안정성**이 높아진다. 
- 하나의 DB가 다운되어도 서비스가 유지되는 **고가용성**을 확보한다.

---

## 캐시

매 요청마다 데이터베이스를 조회하면 성능이 크게 저하된다. **cache**는 자주 읽히는 데이터를 메모리에 저장해 이를 완화한다.

가장 일반적인 패턴은 **read-through cache**이다.

```
요청 -> 캐시 확인
        ├── hit: 캐시에서 즉시 반환
        └── miss: DB 조회 -> 캐시 저장 -> 반환
```

```txt {title="Memcached API 예시"}
SECONDS = 1
cache.set('myKey', 'hi there', 3600 * SECONDS)
cache.get('myKey')
```

캐시 도입 시 고려할 사항은 다음과 같다. 
- 데이터가 자주 읽히고 수정이 드물 때 캐시를 사용한다. 
- 캐시는 휘발성 메모리이므로 중요한 데이터는 영구 저장소에 별도 보관해야 한다. 
- 만료 정책(expiration policy)을 반드시 설정해야 한다. 너무 짧으면 DB 부하가 증가하고, 너무 길면 데이터가 최신 상태로 유지되지 못한다. 
- DB와 캐시가 동기화된 상태로 유지되어야 한다.
- 단일 캐시 서버는 SPOF이므로 여러 데이터 센터에 분산 배치한다. 
- 캐시가 가득 찰 때의 퇴출 정책(eviction policy)도 결정해야 하며, 일반적으로 {{< memo "LRU" "Least Recently Used" >}}가 사용된다.

---

## CDN

**CDN(Content Delivery Network)** 은 정적 콘텐츠(이미지, CSS, JavaScript, 동영상 등)를 지리적으로 분산된 서버에 캐시해 사용자에게 더 빠르게 전달한다.

```
User -> 가장 가까운 CDN 서버
        ├── hit: CDN에서 즉시 반환
        └── miss: Origin 서버에서 파일 가져오기 -> CDN 저장 -> 반환
```

CDN 도입 시 고려할 사항이 있다. 
- 자주 사용되지 않는 자산은 CDN에 올릴 실익이 없다. 
- TTL(Time-to-Live)은 콘텐츠 특성에 맞게 설정해야 한다. 
- CDN 장애 시 origin으로 fallback할 수 있어야 한다. 
- 파일 갱신이 필요할 때는 API로 무효화(invalidate)하거나 URL에 버전 파라미터(`image.png?v=2`)를 붙인다.

---

## 무상태(Stateless) 웹 티어

**stateful** 서버는 사용자 세션 데이터를 서버 자체에 저장한다. 이 경우 같은 사용자의 요청이 항상 같은 서버로 가야 하며(sticky session), 서버 추가/제거가 복잡해진다.

**stateless** 서버는 세션 데이터를 서버 밖의 공유 저장소(관계형 DB, Redis, NoSQL 등)에 저장한다. 어느 서버로 요청이 가도 동일하게 처리할 수 있다.

```
stateful:  User -> (항상) Server 1 -> 세션 in Server 1
stateless: User -> (아무) Server N -> 세션 in Shared Store
```

stateless 구조가 되면 트래픽 증감에 따라 web server를 자유롭게 추가/제거하는 **auto-scaling** 이 가능해진다.

---

## 데이터 센터

서비스가 국제적으로 성장하면 여러 데이터 센터가 필요하다. **geoDNS** 를 사용해 사용자를 가장 가까운 데이터 센터로 라우팅한다.

데이터 센터 운영 시 해결해야 할 과제가 있다. 트래픽 리다이렉션은 GeoDNS로 처리한다. 데이터 동기화는 여러 데이터 센터 간 데이터를 복제해야 하며, 비동기 복제 전략이 일반적이다. 테스트 및 배포는 자동화된 배포 도구로 모든 데이터 센터에서 서비스 일관성을 유지해야 한다.

---

## 메시지 큐

**message queue** 는 producer와 consumer를 분리(decoupling)하는 비동기 통신 구성 요소이다.

```
Producer -> [Message Queue] -> Consumer
```

producer는 consumer가 없어도 메시지를 큐에 발행할 수 있고, consumer는 producer가 없어도 큐에서 메시지를 소비할 수 있다. 큐가 쌓이면 consumer(worker)를 늘리고, 큐가 비어 있으면 줄인다. producer와 consumer가 독립적으로 확장된다.

---

## 데이터베이스 확장

데이터가 계속 증가하면 데이터베이스 자체도 확장해야 한다.

**수직 확장** 은 더 강력한 서버로 교체하는 것이다. Amazon RDS 기준 24TB RAM까지 가능하지만, 하드웨어 한계와 SPOF 위험, 높은 비용이 문제다.

**수평 확장(sharding)** 은 데이터베이스를 shard 단위로 분산하는 방식이다. 각 shard는 동일한 스키마를 갖지만 서로 다른 데이터를 보유한다.

```
user_id % 4 == 0 -> Shard 0
user_id % 4 == 1 -> Shard 1
user_id % 4 == 2 -> Shard 2
user_id % 4 == 3 -> Shard 3
```

sharding key 선택 시 데이터를 균등하게 분산할 수 있는 키를 골라야 한다.

샤딩의 한계
- **resharding**: 특정 shard가 가득 차거나 데이터가 불균등하게 분산되면 shard를 재분배해야 한다. consistent hashing이 이를 완화한다.
- **celebrity problem(hotspot key)**: 특정 shard에 접근이 집중되는 문제다. 인기 있는 데이터는 별도 shard 또는 추가 파티셔닝이 필요하다.
- **join 불가**: shard 간 join이 어렵다. 일반적으로 데이터베이스를 비정규화(de-normalization)해 단일 테이블 쿼리로 해결한다.

---

## 정리

수백만 사용자를 지원하는 시스템으로 확장하는 과정은 한 번에 이루어지지 않는다. 병목이 드러날 때마다 아래의 원칙들을 적용해 점진적으로 개선한다.

- 단일 서버가 버거워짐 -> 웹 서버와 DB 분리
- DB 읽기 부하가 커짐 -> replication으로 읽기 분산 or cache 도입
- 웹 서버 하나가 한계 -> load balancer + 서버 추가
- 정적 파일 전송이 느림 -> CDN 도입
- 웹 서버에 상태가 묶임 -> stateless로 분리해서 auto-scaling 가능하게
- 무거운 작업이 응답 느리게 -> message queue로 비동기 분리
- DB 자체가 한계 -> sharding

db 쓰기는?
## 쓰기 병목 완화

쓰기는 읽기와 달리 단순히 여러 대로 분산하기 어렵다. 데이터 정합성을 보장해야 하기 때문이다.  
읽기 -> 여러 slave가 동시에 처리 가능   
쓰기 -> 동시에 두 곳에서 같은 데이터를 수정하면 충돌 발생  
-> 정합성 보장을 위해 단일 master에 몰릴 수밖에 없음  
완전히 해결할 수는 없지만 완화하는 방법들이 있다.

- **샤딩**으로 쓰기를 여러 master에 분산한다. shard 간 충돌은 없지만 shard 내에서는 여전히 단일 master다.
- **Message Queue**로 쓰기 부하를 시간적으로 분산한다. 요청을 큐에 쌓아두고 worker가 순차적으로 DB에 쓴다. 사용자는 빠르게 응답받지만 실제 DB 반영은 약간 늦을 수 있다. 이를 최종 일관성(eventual consistency)이라고 한다.
- **Write-Back 캐시**로 DB 쓰기 자체를 줄인다. 캐시에만 먼저 쓰고 나중에 비동기로 DB에 반영한다. 빠르지만 캐시가 죽으면 데이터가 유실될 위험이 있다.

쓰기 병목의 근본 원인은 빠른 쓰기를 원할수록 정합성을 일부 포기해야 하는 트레이드오프가 존재한다. 이것이 분산 시스템에서 가장 어려운 문제 중 하나다.
다음은 쓰기 병목 완화 방법 비교표이다.

| 방법 | 원리 | 단점 |
|---|---|---|
| 샤딩 | 쓰기를 여러 master로 분산 | 복잡도 증가 |
| Message Queue | 쓰기를 시간적으로 분산 | 최종 일관성 (즉시 반영 안 됨) |
| Write-Back 캐시 | DB 쓰기 자체를 줄임 | 캐시 죽으면 데이터 유실 |
| Multi-master | master를 여러 대로 | 충돌 해결 로직이 복잡 |