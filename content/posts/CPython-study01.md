---
title: "CPython 파헤치기 (1)" # 리스트에 표시될 제목
description: "CPython 파헤치기 책을 보고 공부하는 글입니다." # 제목 아래에 보일 요약문
summary: "CPython 파헤치기 책을 보고 공부하는 글입니다." # 리스트에서 미리보기로 보여줄 글자들
tags: ['cpython', 'python']
categories: ['study']
author: "ljweel"
date: 2026-01-20
showToc: true
TocOpen: false
draft: false
---


책'CPython 파헤치기' 공부를 시작했다.<br>
Vs Code + Ubuntu 22.04(WSL) 환경에서 기본적인 환경 설정을 했다.

### Cpython Clone하기 (3.9 버젼)
```
git clone --branch 3.9 https://github.com/python/cpython
cd cpython
```

### Vs Code Extension 다운
- C/C++ (ms-vscode.cpptools)
- Python (ms-python.python)
- reStructuredText (lextudio.restructuredtext)
- Task Explorer (spmeesseman.vscode-taskexplorer)

### tasks.json
```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "type": "shell",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "windows": {
                "command": "PCBuild/build.bat",
                "args": ["-p", "x64", "-c", "Debug"]
            },
            "linux": {
                "command": "make -j2 s"
            },
            "osx": {
                "command": "make -j2 s"
            }
        }
    ]
}
```


### ubuntu 에서 python compile 해보기
```bash
# 의존성 설치
sudo apt install build-essential

sudo apt install libssl-dev zlib1g-dev libncurses5-dev \
    libncursesw5-dev libreadline-dev libsqlite3-dev libgdbm-dev \
    libdb5.3-dev libbz2-dev libexpat1-dev liblzma-dev libffi-dev

# configure 스크립트 실행, --with-pydebug는 디버그 훅 활성화 flag
./configure --with-pydebug

make -j2 -s

./python
```
