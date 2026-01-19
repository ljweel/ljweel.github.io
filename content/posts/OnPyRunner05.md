---
title: "OnPyRunner (5)" # 리스트에 표시될 제목
description: "온라인 파이썬 실행기에 대한 의식의 흐름" # 제목 아래에 보일 요약문
summary: "이 글은 OnPyRunner를 구현하는 중 특정 시점에서의 생각과 판단을 기록한 글입니다." # 리스트에서 미리보기로 보여줄 글자들
tags: ['OnPyRunner']
categories: ['dev']
author: "ljweel"
date: 2026-01-19
showToc: true
TocOpen: false
draft: false
---

## [이전 글](https://ljweel.github.io/posts/onpyrunner04/) 요약
- nsjail을 이용해서 sandbox 구축하기로 결정
- Jest를 이용해 TDD 시작


## 테스트 코드 추가
### 네트워크 차단 검증
jail 내부에서 외부 네트워크와 연결이 되지 못해야한다. 
socket.connet 코드가 정상적으로 Network is unreachable을 뱉는지 검사하면 된다.<br>

### 파일 시스템 차단 검증
#### 어떤걸 막아야할까?
open 함수를 제한할 수는 없기 때문에, jail 내부에서 python 실행에 필요한 최소 파일 빼고는 다 open할 수 없게 해야한다. 하지만...

## 지옥의 로컬테스트
nsjail은 chroot + mount로 jail내의 파일 시스템을 재구성한다. 하지만 로컬 우분투에서 이를 그대로 적용하면 로컬 우분투의 파일 시스템을 사용하여 nsjail을 실행시키게 되고, 배포와 개발 코드 사이의 괴리가 커지게 되지만, 무엇보다 로컬 테스트 디버깅이 너무 어려웠다. nsjail 조차 돌리기가 어렵기 때문에 python이 돌아가는지 테스트가 안되는 상황이 발생했다. ~~살려줘~~

## Docker 도입
Docker를 사용하자. Docker를 사용하면 일관된 환경으로 nsjail 설정이 쉬워질 뿐더러 개발과 배포간의 괴리가 사라진다. 

## 로컬 개발 환경 재구축

- OnPyRunner = api 서버 + Docker 실행 로직
- Docker = Python 런타임 + Python 라이브러리 + nsjail + sandbox 실행 dir

같은 느낌으로 개발 환경을 바꾸었다.

기존의 runCode는 nsjail을 실행시킨 반면, 현재의 runCode는 Docker(nsjail + python)를 실행시킨다.

바뀐 runCode는 docker를 실행시킨다.

## 왜 안됨?????
/tmp와 /etc를 open하는 test가 계속 실패하는 상황이 발생했다.
.cfg 기준으로는 mount 된 경로만 nsajil 내부에 생기게 되지만, mount 하지 않은 /tmp와 /etc가 계속 open이 되는 상황 발생<br>
-> 알고보니 clone_newns: false로 되어있었음... <br>
clone_newns는 새 namespace를 만드는지 여부를 체크하게 되는데, false로 되어있어서, docker namespace와 nsjail namespace가 같아져서 mount하지도 않은 /tmp와 /etc가 생김. clone_newns: true를 해서 test 성공

## 추가해야할 테스트
- 프로세스 생성 차단 테스트
- 리소스 제한 테스트
