---
title:  "stdout/stderr 제한과 subprocess 파이프 데드락 해결"
description: "subprocess.PIPE로 stdout/stderr를 동시에 읽을 때 발생하는 데드락의 원인과 스레드를 이용한 해결"
summary: "무한 출력 이슈를 해결하다 파이프 데드락 문제를 발견하여, 원인 분석부터 해결까지의 과정을 기록하였습니다."
tags: ['OnPyRunner', 'Python', 'subprocess', 'Deadlock']
categories: ['Troubleshooting']
author: "ljweel"
date: 2026-03-16
showToc: true
TocOpen: false
draft: false
---


## 문제 제기: stdout/stderr를 제한해야 하는 이유

며칠전에 [첫 issue](https://github.com/ljweel/OnPyRunner/issues/1)가 올라왔다. 
내용은 사용자가 `while True: print("1")`를 실행하면 사이트가 크래시 난다는 것이였다.
원인을 고민해보니 다음과 같았다.

nsjail이 CPU 시간과 메모리를 제한하지만, stdout/stderr는 nsjail의 cgroup 제한에 포함되지 않는다.
stdout/stderr는 자식 프로세스가 `write()` syscall로 데이터를 넘기면 커널 파이프 버퍼나 부모 쪽에 쌓이기 때문이다.  
출력이 아무리 많아도 자식 프로세스의 메모리가 아니라 워커 프로세스의 메모리를 잡아먹는 것이였다.  
nsjail의 `time_limit: 3`이 3초 후 프로세스를 종료시키긴 하지만, 3초 동안 수백 MB의 출력이 쌓일 수 있으므로 별도의 출력 제한이 필요하다고 판단했다.



## subprocess.run을 subprocess.Popen으로

기존 코드에서는 `subprocess.run`을 사용했다.

```python
result = subprocess.run(cmd, stdin=file, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
```

`run()`은 프로세스가 끝난 뒤에 stdout을 한꺼번에 반환하기 때문에 실행 도중에 크기를 체크하고 프로세스를 종료시키는 것이 불가능하다.  
만약 3초 동안 `while True: print("A" * 100000)`가 모두 돌아간다면, 수백 MB가 메모리에 올라온 뒤에야 크기를 확인할 수 있다.

반면, `Popen`은 프로세스가 실행되는 동안 stream으로 stdout/stderr을 읽을 수 있어서, 중간에 제한을 걸고 프로세스를 종료시킬 수 있다.  
그래서 `subprocess.run`에서 `subprocess.Popen`으로 전환했다.

## 순차 읽기의 함정
 
`Popen`으로 전환한 뒤, stdout과 stderr를 순차적으로 읽으면서 각각 크기를 제한하는 코드를 다음과 같이 작성했다.
 
```python
def _consume_stream(self, proc, stream, max_size):
    """
    stream을 실시간으로 읽으며 크기를 제한하는 함수
    설정된 max_size를 초과할 경우, proc를 즉시 종료(terminate)
    """
    output_size = 0
    output = []
    while True:
        chunk = stream.read(4096)
        if not chunk: break
        output_size += len(chunk)
        output.append(chunk)
        if output_size > max_size:
            proc.terminate()
            break
    return b"".join(output)[:max_size].decode("utf-8")

proc = subprocess.Popen(cmd, stdin=f, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
stdout = self._consume_stream(proc, proc.stdout, MAX_STDOUT_SIZE)
stderr = self._consume_stream(proc, proc.stderr, MAX_STDERR_SIZE)
```
이 코드가 정상 동작할 것 같았지만, 다음과 같은 testcase에서 계속 터지는 현상을 발견했다.

```python
import sys
while True: print("A"*100000, file=sys.stderr)
```
stderr에만 대량 출력하는 코드인데, 워커가 stdout부터 읽고 있으니 stderr를 아무도 소비하지 않는 상황이었다.  
원인을 파보니 리눅스 파이프 버퍼의 크기 제한에 의한 데드락이었다.


### 데드락 시나리오

Linux에서 파이프는 커널이 관리하는 고정 크기 버퍼를 갖는다. 기본값은 64KB.

프로세스가 파이프에 데이터를 쓸 때:
- 버퍼에 공간이 있으면 -> 즉시 write 완료
- 버퍼가 가득 차면 -> 누군가 읽어줄 때까지 write가 블로킹

이를 바탕으로 위 코드의 핵심인 만 남기자면 다음과 같다.
```python
proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
stdout = proc.stdout.read()
stderr = proc.stderr.read()
```
위 코드에서 stdout을 먼저 읽고 stderr를 읽을 때, 자식 프로세스가 stderr에 64KB 이상을 쓰려고 하면 다음과 같은 일이 발생한다.

```text
[자식 프로세스]
  stderr에 64KB 씀 -> 버퍼 가득 참 -> 부모가 stderr을 read할 때까지 write 블로킹

[부모 프로세스]
  stdout.read() 실행 중 -> 자식이 끝나야 EOF가 올때까지 블로킹
  
결과: 자식은 stderr 쓰기에서 멈추고, 부모는 stdout 읽기에서 멈춤
      -> 서로를 기다리며 영원히 진행 불가 (데드락)
```

지금까지 말한 데드락 문제의 간단한 재현은 다음과 같이 할 수 있다.
```python
import subprocess

proc = subprocess.Popen(
    ["python3", "-c", "import sys; print('A'*100000, file=sys.stderr)"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)

stdout = proc.stdout.read()
stderr = proc.stderr.read()
```

## 해결: 스레드로 동시 읽기

실제로 [여기](https://docs.python.org/3/library/subprocess.html#subprocess.Popen.wait)를 보면 
> **Note:** This will deadlock when using `stdout=PIPE` or `stderr=PIPE` and the child process generates enough output to a pipe such that it blocks waiting for the OS pipe buffer to accept more data. Use `Popen.communicate()` when using pipes to avoid that.

라고 OS의 pipe buffer보다 더 많은 데이터를 넣을때, PIPE를 사용하면 데드락이 발생할 수 있다고 경고하고, `communicate()`를 사용하라고 한다.

하지만, subprocess의 `communicate()`는 내부적으로 스레드를 사용해 stdout/stderr를 동시에 읽지만,
출력을 전부 읽은 뒤에 반환하기 때문에 실시간 크기 제한이 불가능하다.  
그래서 스레드를 사용해 stdout과 stderr를 동시에 읽으면서 각각 크기를 제한하는 방법을 택했다.

```python
with ThreadPoolExecutor(max_workers=2) as executor:
    stdout_future = executor.submit(
    self._consume_stream, result, result.stdout, MAX_STDOUT_SIZE
    )
    stderr_future = executor.submit(
        self._consume_stream, result, result.stderr, MAX_STDERR_SIZE
    )
    stdout = stdout_future.result()
    stderr = stderr_future.result()
```

이 구조를 사용하여 데드락을 방지하면서 stdout 128KB, stderr 128KB 각각 독립적으로 제한할 수 있게 되었다.  
또한 [web page](https://run.ljweel.dev)에서 stdout/stdeer이 초과될 경우 alert()를 호출하도록 하여 UX를 개선하였다.

## 참고 자료
- [python-discord/snekbox](https://github.com/python-discord/snekbox/blob/main/snekbox/nsjail.py)
- [Python 공식 문서: subprocess](https://docs.python.org/3/library/subprocess.html)
- [Python 공식 문서: ThreadPoolExecutor](https://docs.python.org/ko/3/library/concurrent.futures.html#concurrent.futures.ThreadPoolExecutor)
- [Linux pipe(7) 매뉴얼](https://man7.org/linux/man-pages/man7/pipe.7.html)