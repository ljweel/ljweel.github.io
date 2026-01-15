---
title: "OnPyRunner (1)" # 리스트에 표시될 제목
description: "온라인 파이썬 인터프리터 플랫폼에 대한 의식의 흐름" # 제목 아래에 보일 요약문
summary: "이 글은 OnPyRunner를 구현하는 중 특정 시점에서의 생각과 판단을 기록한 글입니다." # 리스트에서 미리보기로 보여줄 글자들
tags: ['OnPyRunner']
categories: ['dev']
author: "ljweel"
date: 2026-01-08
showToc: true
TocOpen: false
draft: false
---

## 어쩌다 만들게 되었나
백준 문제를 풀다보면 폰으로 코딩하는 일이 빈번하다. 사실 온라인 파이썬 실행기는 많이 있다. 하지만, 많은 사이트들을 찾아봤는데 아쉬웠던 것은
- 최신 버전의 파이썬이 아니였다.
- pypy가 지원되지 않았다.
- 폰코딩이 가능하고, 편리하다.

정도의 주요한 이유가 있었다.

내가 알고리즘 문제를 풀 때 가장 유용하게 썼던 것은 [tio.run](https://tio.run/#python38pr)인데 이거에 기반으로 좀 프로젝트 영감을 얻었긴 하다. ~~사실상 python 최신버젼이 가능한 tio.run~~ 물론 설계나 아키텍쳐, 프레임워크 공부 같은 이유도 있긴하다.

## 주요 기능 정리

그래서 내 프로젝트에는 어떤 주요 기능이 있는지 고민해 보았다.

- **코드와 입력이 주어질 때, 실행 버튼을 누르면 출력해주기 (가장 중요)**
- TBD

해당 기능에 대해서 유스케이스를 생각해보았다.
1) 유저가 코드와 입력을 작성 후 실행 버튼을 누름
2) 코드와 입력을 건네 받아 컨테이너 같은 공간에서 파이썬을 실행시키고 출력
3) 받은 출력을 유저에게 보여주기

이를 다이어그램으로 그려보면

```mermaid
sequenceDiagram
    autonumber
    actor A as USER
    participant B as API SERVER
    participant C as ISOLATED ENV
    A->>B: 코드 실행 요청
    B->>C: 코드 전달 및 실행 명령
    C->>C: 코드 실행 중
    C->>B: 실행 결과 반환
    B->>A: 최종 결과 응답
```
## 가장 큰 문제점


- 보안적으로 문제가 없나? os.system('rm -rf')같은걸 어떻게 막을 건데?
  - Sandbox 컨테이너 격리
- 자원 독점 관리: while True: pass 하면 CPU 다 먹는데?
  - CPU 점유율 관리와 메모리 제한 설정
- 동시 접속자가 늘면 코드 실행 끝날때까지 블로킹
  - 메세지 큐 기반으로 비동기 처리


## 개선 후 다이어그램

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant API as API Server (CT 1)
    participant Redis as Redis (CT 2: MQ/DB)
    participant Worker as Worker (CT 3: 1..N)
    participant Sandbox

    Note over User, Redis: [1단계: 비동기 요청 접수]
    User->>API: (1) 코드 실행 요청
    API->>Redis: (2) Job 생성 & 대기열 투입
    API-->>User: (3) Job ID 반환

    Note over Redis, Sandbox: [2단계: 백그라운드 격리 실행]
    Worker->>Redis: (4) Job 낚아채기
    Worker->>Sandbox: (5) 컨테이너 생성 및 코드 주입
    activate Sandbox
    Sandbox->>Sandbox: (6) 파이썬 실행
    Sandbox-->>Worker: (7) 실행 결과 반환
    deactivate Sandbox
    Worker-->>Redis: (8) 최종 결과 및 상태저장

    Note over User, Redis: [3단계: 결과 확인]
    User->>API: (9) 결과 조회
    API->>Redis: (10) 결과 데이터 확인
    Redis-->>API: 데이터 반환
    API-->>User: (11) 최종 결과 반환

```

이런 식으로 전체적으로 처음에 생각했던 3단계를 세부적으로 쪼갰다.
