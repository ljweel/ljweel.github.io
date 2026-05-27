---
title: "03. Rate Limiter 설계"
description: "rate limiter, token bucket, leaking bucket, sliding window, Redis, distributed system"
summary: "Rate Limiter의 개념과 알고리즘(token bucket, leaking bucket, fixed window, sliding window)을 다루고, 분산 환경에서의 설계까지 진행한 글입니다."
tags: ['system-design', '네트워크']
categories: ['study']
author: "ljweel"
date: 2026-05-07
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

## Rate Limiter 설계

네트워크 시스템에서 rate limiter는 클라이언트 또는 서비스가 전송하는 트래픽의 속도를 제어하는 데 사용된다. HTTP 환경에서 rate limiter는 지정된 시간 동안 클라이언트가 보낼 수 있는 요청 수를 제한한다. API 요청 수가 rate limiter가 정의한 임계값을 초과하면, 초과된 호출은 모두 차단된다. 몇 가지 예시는 다음과 같다.

- 사용자는 초당 최대 2개의 게시물을 작성할 수 있다.
- 동일한 IP 주소에서 하루에 최대 10개의 계정을 생성할 수 있다.
- 동일한 기기에서 일주일에 최대 5회 보상을 수령할 수 있다.

이 챕터에서는 rate limiter를 설계하는 과제를 다룬다. 설계를 시작하기 전에, API rate limiter를 사용할 때의 이점을 살펴본다.

- **DoS(Denial of Service) 공격으로 인한 resource starvation 방지.** 대형 기술 회사가 공개하는 거의 모든 API는 어떤 형태로든 rate limiting을 적용한다. 예를 들어, Twitter는 3시간에 300개 트윗으로 제한하며, Google Docs API는 사용자당 60초에 300회 읽기 요청을 기본 제한으로 둔다. Rate limiter는 의도적이든 의도치 않든 DoS 공격을 초과 호출 차단을 통해 예방한다.
- **비용 절감.** 초과 요청을 제한하면 서버 수가 줄어들고, 우선순위가 높은 API에 더 많은 자원을 할당할 수 있다. 유료 서드파티 API(신용 조회, 결제 처리, 의료 기록 조회 등)를 사용하는 회사에서는 호출 횟수 제한이 비용 절감에 매우 중요하다.
- **서버 과부하 방지.** Bot 또는 사용자의 오남용으로 인한 초과 요청을 걸러내어 서버 부하를 줄인다.

## Step 1 - 문제 이해 및 설계 범위 확립

Rate limiting은 각각 장단점이 있는 다양한 알고리즘으로 구현할 수 있다. 면접관과의 대화를 통해 우리가 구축할 rate limiter의 유형을 명확히 한다.

**지원자**: 어떤 종류의 rate limiter를 설계할 것인가요? client-side rate limiter인가요, server-side API rate limiter인가요?  
**면접관**: 좋은 질문입니다. server-side API rate limiter에 집중하겠습니다.

**지원자**: Rate limiter는 IP, 사용자 ID, 또는 다른 속성을 기반으로 API 요청을 제한하나요?  
**면접관**: Rate limiter는 다양한 throttle 규칙 세트를 지원할 수 있도록 유연해야 합니다.

**지원자**: 시스템의 규모는 어느 정도인가요? 스타트업용인가요, 대규모 사용자 기반을 가진 대기업용인가요?  
**면접관**: 시스템은 대량의 요청을 처리할 수 있어야 합니다.

**지원자**: 시스템은 분산 환경에서 동작하나요?  
**면접관**: 네.

**지원자**: Rate limiter는 별도의 서비스인가요, 아니면 애플리케이션 코드에 구현되어야 하나요?  
**면접관**: 설계 결정은 당신에게 맡깁니다.

**지원자**: Throttle된 사용자에게 알림을 보내야 하나요?  
**면접관**: 네.

### 요구사항

시스템 요구사항 요약은 다음과 같다.

- 과도한 요청을 정확하게 제한한다.
- Low latency. Rate limiter는 HTTP 응답 시간을 느리게 해서는 안 된다.
- 메모리를 최대한 적게 사용한다.
- Distributed rate limiting. Rate limiter는 여러 서버나 프로세스에 걸쳐 공유될 수 있다.
- Exception handling. 요청이 throttle될 때 사용자에게 명확한 예외를 표시한다.
- High fault tolerance. Rate limiter에 문제가 생기더라도(예: cache 서버 오프라인) 전체 시스템에 영향을 주지 않는다.

## Step 2 - 고수준 설계 제안 및 동의 확보

기본적인 client-server 모델로 시작한다.

### Rate Limiter를 어디에 배치할 것인가?

Rate limiter는 client-side 또는 server-side 중 하나에 구현할 수 있다.

- **Client-side 구현.** 일반적으로 client는 rate limiting을 적용하기에 신뢰할 수 없는 위치다. 악의적인 행위자가 client 요청을 쉽게 위조할 수 있으며, client 구현을 제어하지 못할 수도 있다.
- **Server-side 구현.** Figure 1은 server-side에 배치된 rate limiter를 보여준다.

또 다른 방법은 API server에 rate limiter를 두는 대신, Figure 2처럼 API에 대한 요청을 throttle하는 rate limiter middleware를 만드는 것이다.

Figure 3의 예시를 통해 이 설계에서 rate limiting이 어떻게 동작하는지 설명한다. API가 초당 2개의 요청을 허용하고, 클라이언트가 1초 내에 3개의 요청을 보낸다고 가정한다. 첫 두 개의 요청은 API server로 라우팅된다. 그러나 rate limiter middleware는 세 번째 요청을 throttle하고 HTTP status code 429를 반환한다. HTTP 429는 사용자가 너무 많은 요청을 보냈음을 나타낸다.

Cloud microservice가 널리 보급되면서, rate limiting은 보통 API gateway라는 컴포넌트 내에서 구현된다. API gateway는 rate limiting, SSL termination, authentication, IP whitelisting, 정적 콘텐츠 제공 등을 지원하는 완전 관리형 서비스다.

Rate limiter를 server-side에 구현할지, gateway에 구현할지를 결정할 때 고려할 일반적인 가이드라인은 다음과 같다.

- 현재 기술 스택(프로그래밍 언어, cache 서비스 등)을 평가한다. 현재 언어가 server-side에서 rate limiting을 효율적으로 구현할 수 있는지 확인한다.
- 비즈니스 요구에 맞는 rate limiting 알고리즘을 파악한다. 모든 것을 server-side에 구현하면 알고리즘에 대한 완전한 제어권을 갖게 된다. 하지만 서드파티 gateway를 사용하면 선택지가 제한될 수 있다.
- 이미 microservice 아키텍처를 사용하고 있고 authentication, IP whitelisting 등을 위해 API gateway를 설계에 포함했다면, rate limiter를 API gateway에 추가할 수 있다.
- 자체 rate limiting 서비스를 구축하는 데는 시간이 걸린다. 충분한 엔지니어링 리소스가 없다면, 상용 API gateway가 더 나은 선택이다.

### Rate Limiting 알고리즘

Rate limiting은 다양한 알고리즘으로 구현할 수 있으며, 각각 뚜렷한 장단점이 있다. 알고리즘을 고수준에서 이해하면 use case에 맞는 알고리즘이나 조합을 선택하는 데 도움이 된다. 주요 알고리즘 목록은 다음과 같다.

- Token bucket
- Leaking bucket
- Fixed window counter
- Sliding window log
- Sliding window counter

#### Token bucket 알고리즘

Token bucket 알고리즘은 rate limiting에 널리 사용된다. 단순하고 잘 알려져 있으며, Amazon과 Stripe 모두 이 알고리즘으로 API 요청을 throttle한다.

동작 방식은 다음과 같다.

- Token bucket은 미리 정의된 용량을 가진 컨테이너다. 토큰은 사전에 정해진 비율로 주기적으로 버킷에 추가된다. 버킷이 가득 차면 더 이상 토큰이 추가되지 않는다. Figure 4처럼 token bucket 용량이 4이고, refiller가 매 초 2개의 토큰을 추가한다면, 버킷이 가득 찬 후 초과 토큰은 넘쳐 흐른다.
- 각 요청은 토큰 하나를 소비한다. 요청이 도착하면 버킷에 토큰이 충분한지 확인한다(Figure 5).
  - 토큰이 충분하면 요청당 토큰 하나를 꺼내고, 요청이 처리된다.
  - 토큰이 부족하면 요청은 버려진다.

Figure 6은 token 소비, refill, rate limiting 로직이 어떻게 동작하는지 보여준다. 이 예시에서 token bucket 크기는 4이고, refill rate는 분당 4다.

Token bucket 알고리즘은 두 가지 파라미터를 취한다.

- **Bucket size**: 버킷에 허용되는 최대 토큰 수
- **Refill rate**: 초당 버킷에 추가되는 토큰 수

몇 가지 예시를 통해 버킷이 얼마나 필요한지 살펴본다.

- API endpoint마다 별도의 버킷이 필요한 경우가 많다. 예를 들어, 사용자가 초당 1개 게시물 작성, 하루 150명 친구 추가, 초당 5개 좋아요가 허용된다면, 각 사용자에게 버킷 3개가 필요하다.
- IP 주소 기반으로 throttle하는 경우, 각 IP 주소마다 버킷이 필요하다.
- 시스템이 초당 최대 10,000개 요청을 허용한다면, 모든 요청이 공유하는 global bucket을 두는 것이 합리적이다.

{{% collapse summary="장단점" openByDefault=false %}}
**장점**
- 알고리즘이 구현하기 쉽다.
- 메모리 효율적이다.
- Token bucket은 단기간의 트래픽 burst를 허용한다. 토큰이 남아 있는 한 요청이 처리된다.

**단점**
- bucket size와 token refill rate, 두 가지 파라미터를 적절히 튜닝하기 어려울 수 있다.
{{% /collapse %}}

#### Leaking bucket 알고리즘

Leaking bucket 알고리즘은 token bucket과 유사하지만 요청이 고정된 속도로 처리된다는 점이 다르다. 보통 FIFO(First-In-First-Out) 큐로 구현된다. 동작 방식은 다음과 같다.

- 요청이 도착하면 큐가 가득 찼는지 확인한다. 가득 차지 않았으면 요청을 큐에 추가한다.
- 가득 찼으면 요청은 버려진다.
- 요청은 일정한 간격으로 큐에서 꺼내져 처리된다.

Figure 7은 이 알고리즘의 동작을 설명한다.

Leaking bucket 알고리즘은 두 가지 파라미터를 취한다.

- **Bucket size**: 큐 크기와 같다. 큐는 고정 속도로 처리될 요청을 담는다.
- **Outflow rate**: 고정 속도(보통 초 단위)로 처리할 수 있는 요청 수를 정의한다.

Shopify는 rate-limiting에 leaky bucket을 사용한다.

{{% collapse summary="장단점" openByDefault=false %}}
**장점**
- 큐 크기가 제한되어 있어 메모리 효율적이다.
- 요청이 고정 속도로 처리되므로, 안정적인 outflow rate가 필요한 use case에 적합하다.

**단점**
- 트래픽이 burst하면 큐가 오래된 요청으로 가득 차고, 최근 요청이 rate limited될 수 있다.
- 두 가지 파라미터를 적절히 튜닝하기 어려울 수 있다.
{{% /collapse %}}

#### Fixed window counter 알고리즘

Fixed window counter 알고리즘은 다음과 같이 동작한다.

- 타임라인을 고정 크기의 time window로 나누고, 각 window에 counter를 할당한다.
- 각 요청은 counter를 1 증가시킨다.
- counter가 사전에 정의된 임계값에 도달하면, 새로운 time window가 시작될 때까지 새 요청은 버려진다.

Figure 8의 구체적인 예시를 보면, 시간 단위가 1초이고 시스템이 초당 최대 3개의 요청을 허용한다. 각 1초 window에서 3개 이상의 요청이 수신되면 초과 요청은 버려진다.

이 알고리즘의 주요 문제는 time window 경계에서의 트래픽 급증이 허용 할당량을 초과하는 요청을 통과시킬 수 있다는 것이다. Figure 9의 예시처럼, 시스템이 분당 최대 5개 요청을 허용하고 할당량이 정각 분 단위로 리셋된다고 가정한다. 2:00:30부터 2:01:30까지 1분 window에서 10개의 요청이 통과되는데, 이는 허용된 요청의 두 배다.

{{% collapse summary="장단점" openByDefault=false %}}
**장점**
- 메모리 효율적이다.
- 이해하기 쉽다.
- 단위 time window 종료 시 가용 할당량 리셋이 특정 use case에 적합하다.

**단점**
- Window 경계에서의 트래픽 급증으로 허용 할당량보다 많은 요청이 통과될 수 있다.
{{% /collapse %}}

#### Sliding window log 알고리즘

Fixed window counter 알고리즘의 주요 문제, 즉 window 경계에서 더 많은 요청이 통과되는 문제를 sliding window log 알고리즘이 해결한다. 동작 방식은 다음과 같다.

- 알고리즘은 요청의 timestamp를 추적한다. Timestamp 데이터는 보통 Redis의 sorted set 같은 cache에 저장된다.
- 새 요청이 들어오면 만료된 timestamp를 모두 제거한다. 만료된 timestamp는 현재 time window 시작 이전의 것들이다.
- 새 요청의 timestamp를 log에 추가한다.
- Log 크기가 허용 횟수 이하이면 요청이 수락된다. 초과하면 거부된다.

Figure 10의 예시에서 rate limiter는 분당 2개의 요청을 허용한다.

- 1:00:01에 새 요청이 도착할 때 log가 비어 있으므로 요청이 허용된다.
- 1:00:30에 새 요청이 도착하여 timestamp 1:00:30이 log에 삽입된다. 삽입 후 log 크기는 2로, 허용 횟수를 초과하지 않으므로 요청이 허용된다.
- 1:00:50에 새 요청이 도착하여 timestamp가 log에 삽입된다. 삽입 후 log 크기는 3으로, 허용 크기 2를 초과한다. 따라서 이 요청은 거부되지만 timestamp는 log에 남는다.
- 1:01:40에 새 요청이 도착한다. [1:00:40, 1:01:40) 범위의 요청은 최신 time frame 안에 있지만, 1:00:40 이전에 전송된 요청은 만료된 것이다. 만료된 timestamp 1:00:01과 1:00:30이 log에서 제거된다. 제거 후 log 크기가 2가 되므로 요청이 수락된다.

{{% collapse summary="장단점" openByDefault=false %}}
**장점**
- 이 알고리즘으로 구현된 rate limiting은 매우 정확하다. 어떤 rolling window에서도 요청이 rate limit을 초과하지 않는다.

**단점**
- 요청이 거부된 경우에도 timestamp가 메모리에 저장될 수 있어 메모리를 많이 소비한다.
{{% /collapse %}}

#### Sliding window counter 알고리즘

Sliding window counter 알고리즘은 fixed window counter와 sliding window log를 결합한 hybrid 방식이다. Figure 11은 이 알고리즘의 동작을 보여준다.

Rate limiter가 분당 최대 7개의 요청을 허용하고, 이전 분에 5개, 현재 분에 3개의 요청이 있다고 가정한다. 현재 분의 30% 지점에 새 요청이 도착하면, rolling window의 요청 수는 다음 공식으로 계산된다.

```txt
현재 window의 요청 수 + 이전 window의 요청 수 × rolling window와 이전 window의 겹치는 비율
= 3 + 5 × 0.7 = 6.5
```

use case에 따라 올림 또는 내림할 수 있다. 이 예시에서는 내림하여 6이 된다.

Rate limiter가 분당 최대 7개 요청을 허용하므로, 현재 요청은 통과된다. 그러나 요청을 한 번 더 받으면 한계에 도달한다.

{{< memo "Cloudflare 실험" "Cloudflare의 실험에 따르면, 4억 건의 요청 중 잘못 허용되거나 rate limited된 요청은 0.003%에 불과하다고 한다." >}}

{{% collapse summary="장단점" openByDefault=false %}}
**장점**
- 이전 window의 평균 속도를 기반으로 rate를 산정하므로 트래픽 급증을 완화한다.
- 메모리 효율적이다.

**단점**
- 엄밀하지 않은 look back window에서만 동작한다. 이전 window의 요청이 균등하게 분포한다고 가정하기 때문에 실제 rate의 근사값이다.
{{% /collapse %}}


| 알고리즘 | Burst 처리 | 메모리 사용 | 정확성 | 출력 속도 균등성 | 구현 난이도 | 적합한 상황 |
|---|---|---|---|---|---|---|
| Token Bucket | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">허용</span> | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | <span style="background:#FAEEDA;color:#854F0B;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">보통</span> | <span style="background:#FCEBEB;color:#A32D2D;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | 순간적인 요청 폭증을 허용해야 할 때. 평균 rate만 맞으면 되는 일반 API rate limiting. |
| Leaky Bucket | <span style="background:#FAEEDA;color:#854F0B;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">입력 burst 흡수 가능</span> | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | <span style="background:#FAEEDA;color:#854F0B;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">보통</span> | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">높음</span> | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | downstream 시스템이 일정 속도만 처리 가능할 때. 결제, 메시지 큐, 외부 API 보호. |
| Fixed Window Counter | <span style="background:#FAEEDA;color:#854F0B;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">경계 burst 가능</span> | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | <span style="background:#FCEBEB;color:#A32D2D;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | <span style="background:#FCEBEB;color:#A32D2D;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">매우 낮음</span> | 빠르고 단순하게 구현해야 할 때. Redis 기반 간단 rate limit, abuse 방지. |
| Sliding Window Log | <span style="background:#FCEBEB;color:#A32D2D;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">불허</span> | <span style="background:#FCEBEB;color:#A32D2D;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">높음</span> | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">매우 높음</span> | <span style="background:#FCEBEB;color:#A32D2D;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | <span style="background:#FAEEDA;color:#854F0B;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">중간</span> | 초과 요청을 절대 허용하면 안 되는 경우. 금융, 인증, 보안 시스템. |
| Sliding Window Counter | <span style="background:#FAEEDA;color:#854F0B;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">경계 burst 완화</span> | <span style="background:#EAF3DE;color:#3B6D11;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | <span style="background:#FAEEDA;color:#854F0B;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">보통(근사치)</span> | <span style="background:#FCEBEB;color:#A32D2D;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">낮음</span> | <span style="background:#FAEEDA;color:#854F0B;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:600;">중간</span> | 정확성과 메모리 효율의 균형이 필요할 때. 대규모 분산 시스템, CDN, API Gateway. |

### 고수준 아키텍처

Rate limiting 알고리즘의 기본 아이디어는 단순하다. 고수준에서 동일한 사용자, IP 주소 등에서 전송된 요청 수를 추적하는 counter가 필요하다. Counter가 한계를 초과하면 요청이 거부된다.

Counter를 어디에 저장할 것인가? 디스크 접근이 느리기 때문에 database는 좋은 선택이 아니다. In-memory cache는 빠르고 time-based expiration 전략을 지원하기 때문에 선택된다. Redis는 rate limiting 구현에 자주 사용되는 in-memory store로, 다음 두 가지 명령어를 제공한다.

- `INCR`: 저장된 counter를 1 증가시킨다.
- `EXPIRE`: counter에 timeout을 설정한다. timeout이 만료되면 counter는 자동으로 삭제된다.

Figure 12는 rate limiting의 고수준 아키텍처를 보여준다.

- 클라이언트가 rate limiting middleware에 요청을 보낸다.
- Rate limiting middleware는 Redis의 해당 bucket에서 counter를 가져와 한계에 도달했는지 확인한다.
- 한계에 도달했으면 요청이 거부된다.
- 한계에 도달하지 않았으면 요청이 API server로 전달된다. 동시에 시스템은 counter를 증가시키고 Redis에 저장한다.

## Step 3 - 상세 설계

Figure 12의 고수준 설계는 다음 질문에 답하지 않는다.

- Rate limiting 규칙은 어떻게 생성되는가? 규칙은 어디에 저장되는가?
- Rate limited된 요청은 어떻게 처리되는가?

### Rate limiting 규칙

Lyft는 자체 rate-limiting 컴포넌트를 오픈소스로 공개했다. 컴포넌트 내부를 살펴보면 rate limiting 규칙의 예시를 볼 수 있다.

```txt {title="messaging 도메인 규칙 예시"}
domain: messaging
descriptors:
  - key: message_type
    value: marketing
    rate_limit:
      unit: day
      requests_per_unit: 5
```

위 예시에서 시스템은 하루 최대 5개의 마케팅 메시지를 허용하도록 구성된다. 또 다른 예시는 다음과 같다.

```txt {title="auth 도메인 규칙 예시"}
domain: auth
descriptors:
  - key: auth_type
    value: login
    rate_limit:
      unit: minute
      requests_per_unit: 5
```

이 규칙은 클라이언트가 1분에 5번 이상 로그인할 수 없음을 나타낸다. 규칙은 일반적으로 설정 파일에 작성되고 디스크에 저장된다.

### Rate limit 초과 처리

요청이 rate limited된 경우, API는 클라이언트에 HTTP 응답 코드 429(too many requests)를 반환한다. use case에 따라, rate limited된 요청을 나중에 처리할 수 있도록 큐에 넣을 수 있다. 예를 들어, 시스템 과부하로 일부 주문이 rate limited된 경우, 해당 주문들을 나중에 처리할 수 있도록 보관할 수 있다.

#### Rate limiter headers

클라이언트는 어떻게 throttle되고 있다는 것을 알 수 있는가? HTTP response header에 그 답이 있다. Rate limiter는 클라이언트에 다음 HTTP header를 반환한다.

```txt {title="Rate limiter HTTP headers"}
X-Ratelimit-Remaining: window 내 남은 허용 요청 수.
X-Ratelimit-Limit: 클라이언트가 time window당 수행할 수 있는 호출 수.
X-Ratelimit-Retry-After: throttle 없이 다시 요청할 수 있을 때까지 대기할 초(seconds) 수.
```

사용자가 너무 많은 요청을 보내면 `429 too many requests` 오류와 `X-Ratelimit-Retry-After` header가 클라이언트에 반환된다.

### 상세 설계

Figure 13은 시스템의 상세 설계를 보여준다.

- 규칙은 디스크에 저장된다. Worker는 주기적으로 규칙을 디스크에서 가져와 cache에 저장한다.
- 클라이언트가 서버에 요청을 보내면, 요청은 먼저 rate limiter middleware로 전달된다.
- Rate limiter middleware는 cache에서 규칙을 로드한다. Redis cache에서 counter와 마지막 요청 timestamp를 가져온다. 응답에 따라 rate limiter는 다음을 결정한다.
  - 요청이 rate limited되지 않으면 API server로 전달된다.
  - 요청이 rate limited되면 rate limiter는 클라이언트에 `429 too many requests` 오류를 반환한다. 동시에 요청은 버려지거나 큐로 전달된다.

### 분산 환경에서의 Rate Limiter

단일 서버 환경에서 동작하는 rate limiter를 구축하는 것은 어렵지 않다. 그러나 여러 서버와 concurrent thread를 지원하도록 시스템을 확장하는 것은 다른 이야기다. 두 가지 과제가 있다.

- Race condition
- Synchronization 문제

#### Race condition

고수준에서 rate limiter는 다음과 같이 동작한다.

- Redis에서 `counter` 값을 읽는다.
- `(counter + 1)`이 임계값을 초과하는지 확인한다.
- 초과하지 않으면 Redis의 counter 값을 1 증가시킨다.

Figure 14에서 보듯이, 높은 concurrency 환경에서 race condition이 발생할 수 있다.

Redis의 `counter` 값이 3이라고 가정한다. 두 개의 요청이 어느 쪽도 값을 다시 쓰기 전에 동시에 `counter` 값을 읽으면, 각각 `counter`를 1씩 증가시켜 상대 스레드를 확인하지 않고 값을 다시 쓴다. 두 요청(스레드) 모두 올바른 `counter` 값이 4라고 믿는다. 그러나 올바른 `counter` 값은 5여야 한다.

Lock이 race condition을 해결하는 가장 명확한 방법이다. 그러나 lock은 시스템 속도를 크게 저하시킨다. 일반적으로 두 가지 전략이 사용된다: Lua script와 Redis의 sorted sets 자료구조. 이 전략에 관심 있는 독자는 해당 참고 자료를 참조한다.

#### Synchronization 문제

Synchronization은 분산 환경에서 고려해야 할 또 다른 중요한 요소다. 수백만 명의 사용자를 지원하려면 rate limiter 서버 하나로 트래픽을 처리하기에 충분하지 않을 수 있다. 여러 rate limiter 서버를 사용하면 synchronization이 필요하다.

예를 들어, Figure 15의 왼쪽에서 client 1은 rate limiter 1로, client 2는 rate limiter 2로 요청을 보낸다. Web tier는 stateless이므로, 클라이언트는 오른쪽에서처럼 다른 rate limiter로 요청을 보낼 수 있다. Synchronization이 없으면 rate limiter 1은 client 2에 대한 데이터를 갖지 않아 rate limiter가 제대로 동작하지 않는다.

한 가지 가능한 해결책은 클라이언트가 동일한 rate limiter에 트래픽을 보낼 수 있도록 sticky session을 사용하는 것이다. 이 해결책은 확장성도 없고 유연하지도 않아 권장되지 않는다. 더 나은 방법은 Figure 16처럼 Redis와 같은 중앙화된 data store를 사용하는 것이다.

### 성능 최적화

성능 최적화는 시스템 설계 면접에서 흔한 주제다. 두 가지 영역을 다룬다.

첫째, multi-data center 설정은 rate limiter에 매우 중요하다. 데이터 센터에서 멀리 있는 사용자에게는 latency가 높기 때문이다. 대부분의 cloud service provider는 전 세계에 많은 edge server 위치를 구축한다. 예를 들어, 2020년 5월 기준으로 Cloudflare는 194개의 지리적으로 분산된 edge server를 보유하고 있다. 트래픽은 자동으로 가장 가까운 edge server로 라우팅되어 latency를 줄인다.

둘째, eventual consistency model로 데이터를 동기화한다. Eventual consistency model이 불분명하다면, "Design a Key-value Store" 챕터의 "Consistency" 섹션을 참조한다.

### 모니터링

Rate limiter가 배치된 후에는 rate limiter가 효과적인지 확인하기 위해 분석 데이터를 수집하는 것이 중요하다. 주로 다음 두 가지를 확인한다.

- Rate limiting 알고리즘이 효과적인가.
- Rate limiting 규칙이 효과적인가.

예를 들어, rate limiting 규칙이 너무 엄격하면 유효한 요청이 많이 버려진다. 이 경우 규칙을 조금 완화해야 한다. 또 다른 예로, 플래시 세일 같은 트래픽 급증 시 rate limiter가 비효과적임을 발견할 수 있다. 이 시나리오에서는 burst 트래픽을 지원하도록 알고리즘을 교체할 수 있다. Token bucket이 여기에 잘 맞는다.

## Step 4 - 마무리

이 챕터에서는 rate limiting의 다양한 알고리즘과 그 장단점을 논의했다. 다룬 알고리즘은 다음과 같다.

- Token bucket
- Leaking bucket
- Fixed window
- Sliding window log
- Sliding window counter

그리고 시스템 아키텍처, 분산 환경에서의 rate limiter, 성능 최적화, 모니터링을 논의했다. 시간이 허락된다면 언급할 수 있는 추가 주제는 다음과 같다.

- **Hard vs soft rate limiting.**
  - Hard: 요청 수가 임계값을 초과할 수 없다.
  - Soft: 요청이 짧은 기간 동안 임계값을 초과할 수 있다.
- **다양한 레이어에서의 rate limiting.** 이 챕터에서는 application 레벨(HTTP: layer 7)의 rate limiting만 다뤘다. 다른 레이어에서도 rate limiting을 적용할 수 있다. 예를 들어, Iptables(IP: layer 3)를 사용하여 IP 주소로 rate limiting을 적용할 수 있다. OSI 모델은 총 7개의 레이어를 가진다: Layer 1(Physical), Layer 2(Data link), Layer 3(Network), Layer 4(Transport), Layer 5(Session), Layer 6(Presentation), Layer 7(Application).
- **Rate limited되지 않도록 클라이언트 설계 시 권장 사항.**
  - 빈번한 API 호출을 피하기 위해 client cache를 사용한다.
  - 한계를 이해하고 짧은 시간 내에 너무 많은 요청을 보내지 않는다.
  - 예외나 오류를 처리하는 코드를 포함하여 클라이언트가 예외에서 gracefully recover할 수 있도록 한다.
  - Retry 로직에 충분한 back off 시간을 추가한다.