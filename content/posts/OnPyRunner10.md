---
title: "OnPyRunner (10)" # 리스트에 표시될 제목
description: "온라인 파이썬 실행기에 대한 의식의 흐름" # 제목 아래에 보일 요약문
summary: "이 글은 OnPyRunner를 구현하는 중 특정 시점에서의 생각과 판단을 기록한 글입니다." # 리스트에서 미리보기로 보여줄 글자들
tags: ['OnPyRunner']
categories: ['dev']
author: "ljweel"
date: 2026-02-22
showToc: true
TocOpen: false
draft: false
---

## [이전 글](https://ljweel.github.io/posts/onpyrunner09/) 요약
- 테스트 자동화
- nsjail log 처리 및 cgroup 설정

## 배포 및 UI 제작
이제 MVP는 거의 제작된거 같아서 배포 및 UI 제작을 해야한다. 배포는 vulcan님의 도움을 받아 vulcan 서버에서 하게 되었고, 클플 터널링 + nginx로 리버스 프록시를 해서 기존에 존재하던 서비스와 충돌하지 않게 했다. ui는 [run.xo.dev](run.xo.dev)를 참고해서 중앙을 기준으로 좌를 code, stdin을 입력하는 곳, 우를 stdout, stderr 를 보여주는 곳으로 했다. 

## 전체적인 구조 정리

### Excution Flow(Sequence Diagram)
전체적인 실행흐름을 시퀀스 다이어그램으로 나타내면 다음과 같다.


```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Redis Queue
    participant Worker
    participant Sandbox
    participant Redis Storage

    Client->>API: POST /execute
    API->>Redis Storage: SET job:{jobId} = PENDING
    API->>Redis Queue: LPUSH job_queue
    API-->>Client: return jobId

    Redis Queue-->>Worker: BRPOP job_queue
    Worker->>Sandbox: Execute code
    Sandbox-->>Worker: stdout, stderr, exit_code
    Worker->>Redis Storage: SET job:{jobId} = COMPLETED
    
    loop Polling
        Client->>API: GET /jobs/{jobId}
        API->>Redis Storage: GET job:{jobId}
        Redis Storage-->>API: status
        API-->>Client: status
    end
```

다른 것도 정리했는데 여기보다는 [여기](https://github.com/ljweel/OnPyRunner/blob/main/docs/architecture.md)에 작성했다.

## 드디어 오픈

[ljweel.dev](https://ljweel.dev)처럼 nginx로 호스팅했다.
- run.ljweel.dev -> 사이트 UI(html + js)를 담당
- run.ljweel.dev에서 /execute와 /jobs/{jobId} -> api endpoint

[run.ljweel.dev](https://run.ljweel.dev) 해당링크에서 실행해볼 수 있다.
