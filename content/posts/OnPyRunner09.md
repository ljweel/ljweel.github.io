---
title: "OnPyRunner (9)" # 리스트에 표시될 제목
description: "온라인 파이썬 실행기에 대한 의식의 흐름" # 제목 아래에 보일 요약문
summary: "이 글은 OnPyRunner를 구현하는 중 특정 시점에서의 생각과 판단을 기록한 글입니다." # 리스트에서 미리보기로 보여줄 글자들
tags: ['OnPyRunner']
categories: ['dev']
author: "ljweel"
date: 2026-02-11
showToc: true
TocOpen: false
draft: false
---

## [이전 글](https://ljweel.github.io/posts/onpyrunner08/) 요약
- worker 설계와 데이터 흐름도

## 테스트 자동화 도입

전체적으로 설계가 끝났기 때문에 로컬에서 docker compose up으로 컨테이너를 올리고 api를 테스트 해보고 있는데, postman으로 직접 localhost:8000/execute를 Send하고, 받은 job_id를 복사해서 localhost:8000/jobs/{job_id}에 붙여넣고 잘 나오는지 체크해보고 있었다. 근데 코드 고치고 테스트하는 일이 많아져서 이전에 생각했던 TDD가 필요해졌다. 그래서 코드를 고치고 api 테스트하는 일련의 과정을 개발 과정에서 자동화하기로 했다.  

- 코드를 고치고 docker compose down & docker compose up -d --build
  - docker compose watch와 volume 기능을 사용해서 파일이 수정되면 컨테이너를 재시작/재빌드를 자동으로 해준다.
- post /execute + get /jobs/{job_id} 과정을 테스트로 작성하여 기대하는 결과가 나오는지 체크
  - pytest + request를 사용해서 해당 과정을 class로 작성했다.

## nsjail stderr 처리

pytest를 진행하던 도중 stderr에 nsjail info가 불필요하게 많이 나오는걸 알게 되었다.
[snekbox](https://github.com/python-discord/snekbox/blob/main/snekbox/nsjail.py#L27)의 경우 링크의 코드를 보면 알겠지만 nsjail info를 패턴매칭하여 logging을 이용했다. 딱히 info 관련은 필요 없는 것 같아서 nsjail command option으로 '-q'를 주어 warning 이상의 로그만 나오도록 했다. 
### 참고 코드
- [snekbox log pattern matching](https://github.com/python-discord/snekbox/blob/main/snekbox/nsjail.py#L27)
- [nsjail cmdline -q](https://github.com/google/nsjail/blob/master/cmdline.cc#L539)

## cgroup
fork bomb를 막기 위해 cgroup 설정을 하고자 했지만 번번히 실패했다. 그래서 그 이유를 찾아보니 다음과 같았다. 현재의 구조는 docker가 api-server, redis, worker로 3개로 분리해서 만들었다. worker docker에서는 worker가 job을 낚고, nsjail을 실행시킨다.  

하지만, docker는 컨테이너 단위로 cgroup을 관리하게 되는데, 이때, 컨테이너 내부의 프로세스가 cgroup 에 대한 작성 권한을 위임받지 않기 때문에 nsjail에서 cgroup을 설정할 수 없게 되는 것이였다. 

따라서 이 문제를 해결하기 위해, 컨테이너 내부에서 cgroup 트리를 재구성하는 방식을 사용하였다.

cgorup v2의 [**no internal processes rule**](https://manpath.be/f35/7/cgroups#L539)에 따르면, 어떤 cgroup이 하위 cgroup에 컨트롤러를 위임하려면(cgroup.subtree_control에 +memory, +cpu, +pids 등을 설정하려면), 해당 cgroup은 프로세스를 직접 보유하고 있지 않아야 한다. 즉, 프로세스는 leaf 노드에만 존재해야 한다.

하지만 Docker 컨테이너 내부에서 /sys/fs/cgroup의 루트 cgroup에는 이미 컨테이너의 PID 1 프로세스를 포함한 여러 프로세스가 존재한다. 이 상태에서는 cgroup.subtree_control에 컨트롤러를 활성화하려 할 경우 Device or resource busy 오류가 발생한다. 이는 규칙 위반 때문이다.

이를 해결하기 위해 다음과 같은 절차를 거쳤다.

먼저 /sys/fs/cgroup/init이라는 하위 cgroup을 생성한다. 그리고 루트 cgroup에 속해 있던 모든 프로세스를 이 init 그룹으로 이동시킨다. 이렇게 하면 루트 cgroup은 더 이상 프로세스를 가지지 않는다. 즉, 컨트롤러를 위임할 수 있는 상태가 된다.

그 다음 루트 cgroup의 cgroup.subtree_control에 +memory, +cpu, +pids 등을 활성화한다. 이 단계는 하위 cgroup들이 해당 리소스 컨트롤러를 사용할 수 있도록 권한을 위임하는 과정이다.

이후 /sys/fs/cgroup/nsjail과 같은 별도의 하위 cgroup을 생성하고, 그 아래에서 nsjail이 새로운 cgroup을 만들도록 구성한다. 이렇게 하면 nsjail은 해당 하위 트리 안에서 memory.max, pids.max, cpu.max 등을 정상적으로 설정할 수 있게 된다.

정리하면,

1. 루트 cgroup에 존재하던 프로세스를 init 그룹으로 이동시켜 루트를 비운다.
2. 루트에서 subtree_control을 활성화하여 컨트롤러를 하위에 위임한다.
3. nsjail 전용 하위 cgroup을 만들고, 그 안에서 자식 cgroup을 생성하도록 한다.

이 과정을 통해 Docker 컨테이너 내부에서도 cgroup v2 규칙을 만족하는 구조를 만들 수 있었고, nsjail이 fork bomb 방지를 위한 pids.max나 메모리 제한을 정상적으로 설정할 수 있게 되었다.