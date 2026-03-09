---
title: "rlimit vs cgroup: 메모리 제한의 두 가지 레이어"
description: "OnPyRunner에서 샌드박스 메모리 설정 중 rlimit과 cgroup의 차이를 서술한 글입니다."
summary: "rlimit과 cgroup의 차이를 서술한 글입니다."
tags: ['Linux']
categories: ['dev']
author: "ljweel"
date: 2026-03-06
showToc: true
TocOpen: false
draft: false
---

## 배경

OnPyRunner에서 사용자 코드의 메모리를 제한하기 위해 nsjail을 사용한다.
nsjail은 내부적으로 **rlimit**과 **cgroup** 두 가지 메커니즘으로 메모리를 제한할 수 있다.

```text
# nsjail.cfg
rlimit_as: 128          # 가상 주소 공간 128MB 제한
cgroup_mem_max: 134217728  # 물리 메모리 128MB 제한 (cgroup)
```

exit_code 137(SIGKILL)이 발생했을 때, 이것이 TLE(Time Limit Exceeded)인지 MLE(Memory Limit Exceeded)인지 구분해야 하는 문제가 있었다. 이 글에서는 두 메커니즘의 차이를 정리하고, 현재 설정에서 exit_code를 어떻게 해석할 수 있는지 분석한다.

---

## 가상 메모리 vs 물리 메모리

### 메모리 할당은 예약과 사용의 2단계다

프로세스가 메모리를 요청하면(예: `malloc`), 내부적으로는 `brk` 또는 `mmap` 시스템 콜을 통해 커널에 **가상 주소 공간을 예약**한다. 이 시점에서 물리 메모리(RAM)는 아직 할당되지 않는다.

비유하면:
- **가상 주소 공간 예약** = 땅에 울타리를 치는 것 (주소 공간 확보)
- **물리 메모리 할당** = 그 땅에 건물을 짓는 것 (실제 데이터 write)

### demand paging

물리 메모리는 해당 가상 주소에 **처음 접근(read/write)할 때** 페이지 단위(보통 4KB)로 할당된다. 이를 **demand paging**이라 하며, 첫 접근 시 **page fault**가 발생하고 커널이 물리 페이지를 매핑한다.

```text
malloc(100MB) 직후  → 가상 주소 공간 100MB 예약, 물리 메모리 ~0
50MB 영역에 write   → 가상 100MB, 물리 ~50MB
100MB 전부 write    → 가상 100MB, 물리 ~100MB
```

이 구조 때문에 **가상 주소 공간 사용량 >= 물리 메모리 사용량**이 일반적으로 성립한다. 단, page cache나 cgroup accounting 방식에 따라 예외가 있으며, 이는 뒤에서 다룬다.

---

## rlimit_as: 가상 주소 공간 제한

- **제한 대상**: 프로세스의 **전체 가상 주소 공간**(Virtual Address Space)
  - heap, stack, mmap 영역, 공유 라이브러리 매핑 등 모든 매핑을 포함한다
- **체크 시점**: `brk`, `mmap` 등으로 가상 주소 공간을 확장하려는 시점
- **동작**: 가상 주소 공간의 총합이 제한을 초과하면 시스템 콜이 실패한다 (`mmap` → `ENOMEM`, `brk` → 실패)
- **결과**: Python의 경우, 메모리 할당 실패를 감지하여 `MemoryError` 예외를 발생시킨다 → **exit_code 1**

```python
# 한 번에 큰 할당 시도
a = [1] * 1000000000  # 내부적으로 대량의 메모리 요청 → rlimit_as 초과 → MemoryError

```
---

## cgroup_mem_max: 물리 메모리 제한 (cgroup)

- **제한 대상**: cgroup에 속한 **모든 프로세스**의 메모리 사용량 합산
  - RSS(Resident Set Size), page cache, tmpfs, 일부 커널 메모리(slab 등)가 포함된다
  - 즉, 프로세스의 가상 주소 공간이 아니라 **커널이 해당 cgroup에 청구(charge)한 물리 페이지**를 기준으로 한다
- **체크 시점**: 물리 페이지가 실제로 할당·청구되는 시점 (page fault 처리, 파일 I/O 등)
- **동작**: cgroup 전체 메모리 사용량이 제한을 초과하면 커널의 OOM Killer가 발동한다
- **결과**: SIGKILL(9) → **exit_code 137** (128 + 9)

cgroup은 가상 주소 공간 예약(울타리) 자체는 막지 않는다. 실제 물리 메모리 사용(건물)이 한도를 넘어야 개입한다.

---

## 두 메커니즘 비교

| 구분 | rlimit_as | cgroup_mem_max |
|------|-----------|----------------|
| 제한 대상 | 가상 주소 공간 (VAS) | 물리 메모리 (cgroup 단위) |
| 적용 범위 | 프로세스 개별 | cgroup 전체 (부모+자식 합산) |
| 체크 시점 | brk/mmap 호출 시 | 물리 페이지 할당(charge) 시 |
| 실패 방식 | 시스템 콜 실패 → 예외 | OOM Kill → SIGKILL |
| exit_code | 1 (MemoryError) | 137 (SIGKILL) |

---

## 공유 라이브러리의 영향

Python 프로세스가 시작할 때 libc, libpython 등 공유 라이브러리를 로드한다.

- **rlimit_as**: 공유 라이브러리를 가상 주소 공간에 **매핑하는 것 자체**가 카운트된다. 다른 프로세스와 공유하더라도 해당 프로세스의 가상 주소 공간을 차지한다.
- **cgroup_mem_max**: 공유 라이브러리의 read-only 페이지는 이미 물리 메모리에 존재하는 페이지를 참조만 하므로, 해당 cgroup에 추가로 청구되지 않는 경우가 많다. 단, copy-on-write로 인한 private dirty page가 생기면 그 부분은 청구된다.

이 차이는 가상 주소 공간과 물리 메모리 사용량 사이의 갭을 벌리는 요인 중 하나다. 공유 라이브러리가 많을수록 가상 주소 공간은 크게 잡히지만, 물리 메모리 사용량은 상대적으로 적다.

---

## 같은 값으로 설정했을 때 어느 쪽이 먼저 발동하는가?

**일반적인 경우, rlimit_as가 먼저 발동할 가능성이 높다.**

이유:
1. demand paging으로 인해 가상 주소 공간 사용량이 물리 메모리 사용량보다 크거나 같다
2. 공유 라이브러리 매핑이 가상 주소 공간에는 잡히지만 물리 메모리에는 추가 청구되지 않는 부분이 있다
3. Python 인터프리터 자체가 시작 시 상당한 가상 주소 공간을 사용한다

따라서 **단순히 메모리를 할당하는 패턴**(list, dict 등의 자료구조 확장)에서는 rlimit_as가 먼저 한도에 도달하여 `MemoryError`가 발생하고, cgroup OOM Kill까지 가지 않는다.

---

## 예외: cgroup이 먼저 발동하는 경우

위의 판단은 "일반적인 메모리 할당 패턴"에서만 성립한다. 다음과 같은 경우에는 cgroup이 먼저 발동할 수 있다.

### 1. page cache를 통한 파일 I/O

파일 I/O(`read()`, `write()` 시스템 콜)를 수행하면 커널은 **page cache**에 파일 내용을 올린다.

- page cache는 커널이 관리하는 영역이다. 프로세스의 가상 주소 공간에 매핑되지 않으므로 **rlimit_as에 카운트되지 않는다**.
- 그러나 **cgroup memory accounting에서는 page cache도 해당 cgroup에 청구**된다.

```text
read('huge_file.txt') 실행 시:

프로세스 관점:
  - 유저 버퍼 (read 결과를 받을 공간)만 가상 주소 공간에 존재 → rlimit_as에 카운트

커널 관점:
  - 디스크에서 읽은 데이터를 page cache에 올림 → 프로세스의 VAS 밖 → rlimit_as와 무관
  - 하지만 해당 cgroup에 청구됨 → cgroup_mem_max에 카운트
```

이 경우 rlimit_as에는 잡히지 않는 물리 메모리(page cache)가 cgroup에는 누적되므로, cgroup_mem_max가 먼저 한도에 도달할 수 있다.

### 2. fork/clone으로 인한 다중 프로세스

rlimit_as는 **프로세스 개별** 단위로 적용되지만, cgroup_mem_max는 **cgroup 전체** (부모 + 자식 프로세스 합산) 단위로 적용된다.

```text
fork()를 여러 번 실행한 경우:

rlimit_as: 각 프로세스가 개별적으로 128MB 이내 → 제한에 걸리지 않음
cgroup_mem_max: 부모 + 자식 프로세스의 물리 메모리 합산 → 128MB 초과 가능 → OOM Kill
```

### 3. tmpfs / shared memory

`/dev/shm` 등 tmpfs에 쓰거나 POSIX shared memory를 사용하면, 해당 메모리는 cgroup에 청구되지만 rlimit_as에는 mmap 매핑 크기만 반영된다. 대량의 tmpfs 사용 시 cgroup이 먼저 발동할 수 있다.

---

## OnPyRunner에서의 실제 판별 로직

이 판별 로직은 다음 두 가지 전제 조건 위에서 성립한다.

### 전제 1: nsjail이 fork와 파일 I/O를 제한한다

OnPyRunner의 nsjail 설정에서는 fork/clone이 제한되고, 사용자 코드가 접근할 수 있는 파일 시스템이 최소화되어 있다. 따라서 앞서 다룬 예외 상황(page cache 누적, 다중 프로세스의 cgroup 합산)이 발생할 가능성이 낮다. 이 덕분에 MLE는 cgroup OOM Kill(exit_code 137)이 아닌 rlimit_as에 의한 `MemoryError`(exit_code 1)로 나타난다고 기대할 수 있다.

### 전제 2: MemoryError를 except로 잡아도 문제없다

사용자가 `except`로 `MemoryError`를 잡으면 exit_code 1이 아닌 0으로 정상 종료될 수 있다.

```python
try:
    a = [1] * 1000000000
except:
    pass  # MemoryError가 잡혀서 exit_code 0으로 종료
```

그러나 이 경우 프로그램이 crash 없이 실행을 계속한 것이므로, MLE가 아닌 정상 종료(SUCCESS)로 판정하는 것이 적절하다. 메모리 한도에 도달했더라도 프로그램 스스로 이를 처리한 것이기 때문이다.

### 판별 로직

위 전제 조건 하에서, 현재 설정(`rlimit_as: 128`, `cgroup_mem_max: 134217728`, 둘 다 128MB)의 판별 로직은 다음과 같다:

| exit_code | stderr 내용 | 판정 |
|-----------|-------------|------|
| 0 | - | SUCCESS |
| 1 | "MemoryError" 포함 | MEMORY_LIMIT_EXCEEDED |
| 1 | 그 외 | RUNTIME_ERROR |
| 137 | - | TIME_LIMIT_EXCEEDED |
| 그 외 | - | UNKNOWN_ERROR |

단, 향후 입력 크기 제한을 늘리거나, 사용자 코드가 파일 I/O를 수행할 수 있는 환경이 된다면, exit_code 137이 MLE일 가능성도 고려해야 한다. 이 경우 cgroup의 `memory.events` 파일에서 OOM 발생 여부를 확인하는 방식으로 TLE와 MLE를 구분할 수 있다.

---

## 만약 cgroup이 먼저 발동하게 하려면?

`cgroup_mem_max`를 `rlimit_as`보다 **낮게** 설정하면 된다.

```text
rlimit_as: 128           # 가상 주소 공간 128MB
cgroup_mem_max: 13421     # 물리 메모리 ~13KB
```

이 경우 Python 인터프리터 자체가 시작할 때 물리 메모리 사용량이 즉시 한도를 초과하여 OOM Kill된다. Python 코드가 한 줄도 실행되지 않고 exit_code 137이 발생한다.

---

## 한줄 정리

- **rlimit_as**: 프로세스의 가상 주소 공간(VAS) 총합을 제한한다. `brk`/`mmap` 시점에 체크.
- **cgroup_mem_max**: cgroup에 청구된(charged) 물리 페이지 총합을 제한한다. 물리 페이지 할당 시점에 체크.

가장 큰 차이는 **accounting 대상**이 다르다는 것이다:
- rlimit_as: 가상 주소 공간에 매핑된 모든 영역 (heap, stack, mmap, 공유 라이브러리 등)
- cgroup_mem_max: 해당 cgroup에 청구된 물리 페이지 (RSS, page cache, tmpfs 등)

같은 값으로 설정했을 때, 순수 메모리 할당 패턴에서는 rlimit_as가 먼저 한도에 도달하는 것이 일반적이다. 그러나 page cache, fork, tmpfs 등의 요인으로 cgroup이 먼저 발동할 수 있으므로, "항상 rlimit_as가 먼저"라고 단정할 수는 없다.

---

## 면접 팁

- "가상 메모리와 물리 메모리의 차이"는 OS 면접 단골 질문이다
- demand paging, page fault, copy-on-write 등과 연결된다
- cgroup은 컨테이너(Docker) 기술의 핵심이므로 "Docker가 메모리를 어떻게 제한하는가?"라는 질문과도 연결된다
- "rlimit과 cgroup의 차이"를 설명할 수 있으면, 리소스 격리에 대한 깊은 이해를 보여줄 수 있다
