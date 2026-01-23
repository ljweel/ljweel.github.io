---
title: "메모장" # 리스트에 표시될 제목
description: "기억해야 할 것 같은 내용을 작성하는 곳, 일종의 wiki" # 제목 아래에 보일 요약문
summary: "헷갈리는 개념을 적어 공부해봅니다." # 리스트에서 미리보기로 보여줄 글자들
tags: ['memo']
categories: ['study']
author: "ljweel"
date: 2026-01-22
showToc: true
TocOpen: false
draft: false
pinned: true
weight: 0
---

## Request vs Job
Request 
- HTTP 프로토콜 수준의 단위
- 클라이언트 <-> 서버 연결의 생명주기
- 짧아야 한다
- 실패하면 즉시 응답해야 한다
- 재시도 개념이 약하다

Job
- 시간이 걸릴 수 있는 실행 단위
- HTTP와 무관하게 독립된 생명주기 보유
- 실행 / 대기 / 종료 상태를 가진다
- timeout, kill, 재시도가 가능하다
- Worker가 실행한다

설계 판단 기준

>- 실행 시간이 항상 짧고 (<100ms),
>- 실패가 거의 없으며,
>- timeout 관리가 필요 없다면

-> Request

>- 실행 시간이 길 수 있고
>- 무한루프 가능성이 있으며
>- 강제 종료가 필요하다면

-> Job

## 책임 분리
설계 판단 기준: 이 코드가 죽으면 서비스 전체가 죽어도 되나?  
YES -> 같은 책임  
NO -> 책임 분리

## 동기/비동기와 블로킹/논블로킹
- 동기/비동기
  - 작업의 순차적 흐름 보장
  - 작업의 완료 여부를 따지면 동기, 아니면 비동기
  - 
- 블로킹/논블로킹: 
  - 현재 작업을 처리하기 위해 실행 중인 작업을 블락(차단/대기)하는지가 중요
  - 작업을 막으면 블로킹, 아니면 논블로킹
  - 호출한 스레드가 멈추면 블로킹, 아니면 논블로킹

참고 링크
- [완벽히 이해하는 동기/비동기 & 블로킹/논블로킹](https://inpa.tistory.com/entry/%F0%9F%91%A9%E2%80%8D%F0%9F%92%BB-%EB%8F%99%EA%B8%B0%EB%B9%84%EB%8F%99%EA%B8%B0-%EB%B8%94%EB%A1%9C%ED%82%B9%EB%85%BC%EB%B8%94%EB%A1%9C%ED%82%B9-%EA%B0%9C%EB%85%90-%EC%A0%95%EB%A6%AC#%EB%8F%99%EA%B8%B0%EC%99%80_%EB%B9%84%EB%8F%99%EA%B8%B0%EB%8A%94_%EC%9E%91%EC%97%85_%EC%88%9C%EC%84%9C_%EC%B2%98%EB%A6%AC_%EC%B0%A8%EC%9D%B4)

- [블로킹(Blocking)/논블로킹(Non-Blocking), 동기(Sync)/비동기(Async) 구분하기](https://joooing.tistory.com/entry/%EB%8F%99%EA%B8%B0%EB%B9%84%EB%8F%99%EA%B8%B0-%EB%B8%94%EB%A1%9C%ED%82%B9%EB%85%BC%EB%B8%94%EB%A1%9C%ED%82%B9)
