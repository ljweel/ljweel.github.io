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

