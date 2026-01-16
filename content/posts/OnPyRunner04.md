---
title: "OnPyRunner (4)" # 리스트에 표시될 제목
description: "온라인 파이썬 실행기에 대한 의식의 흐름" # 제목 아래에 보일 요약문
summary: "이 글은 OnPyRunner를 구현하는 중 특정 시점에서의 생각과 판단을 기록한 글입니다." # 리스트에서 미리보기로 보여줄 글자들
tags: ['OnPyRunner']
categories: ['dev']
author: "ljweel"
date: 2026-01-15
showToc: true
TocOpen: false
draft: false
---


## [이전 글](https://ljweel.github.io/posts/onpyrunner03/) 요약
- 설계 미스로 원하던 구조의 실행기가 아니게 됨.
- 어디서 미스난지 파악하고 설계 개선

## 현재 고민
서버에서 Python을 실행시키려면 제약 조건이 많아진다. 신뢰할 수 없는 코드에 대해서도 실행가능해야하는데 다음과 같은 고려 사항을 떠올렸다.
- 네트워크 격리
- CPU, 시간, 메모리와 같은 자원 제한
- syscall 필터
- 파일 시스템 제한

## 현재의 선택지
### seccomp
seccomp는 리눅스에서 sandbox 기반으로 시스템콜을 허용 및 차단하여 공격의 가능성을 막는 리눅스 보안 메커니즘이다.
### Nsjail
네임스페이스, 리소스 제한 및 seccomp-bpf 시스템 호출 필터를 사용하는 Linux 프로세스 격리 도구이다. <br>
내가 원하던 네트워크 격리, syscall, 자원 제한, 파일시스템 제한 등 많은 기능이 있었다.

## Nsjail로 결정
다른 사람들이 어떻게 했는지 구글링하다가 같은 생각을 한 사람을 발견했다. 
[글1](https://stackoverflow.com/questions/3068139/how-can-i-sandbox-python-in-pure-python) 
[글2](https://publish.obsidian.md/kruzenshtern/writings/2021-05-21-run-python-in-a-sandbox-with-nsjail)
정도 유용하게 쓰였고, 글2가 많은 도움을 주었다.
구글에서도 untrust python code를 실행시키기 위해 nsjail을 썼다고 한다. ~~그냥 자기들이 만든거 쓴거같은데~~<br>
결론은 OnPyRunner에 Nsjail을 써서 격리환경을 구성할 것 같다.

## 테스트 주도 개발 도입
내가 nsjail 코드를 작성하고 코드를 돌렸을 때, 내가 원했던 조건들이 적용되었는지 하나하나씩 일일이 체크하는건 어렵다. 테스트케이스를 만들어서 코드가 주어질때 stdout, stderr를 예측하도록 하기 위해 테스트 주도 개발을 하려고 한다. <br>
runCode라는 함수에 대해서 jest를 사용해서 unit test를 진행해야겠다. 

## Test Case 만들어보기
먼저 runCode를 통해 Hello, World! 출력을 확인하는 테스트 코드를 작성해보았다.
```js
test("stdout에 Hello, World!가 출력된다", () => {
    const result = runCode({
        code: "print('Hello, World!')",
        input: "",
    });

    expect(result.stdout.trim()).toBe("Hello, World!");
});
```
이제 아래 코드를 nsjail을 사용해서 python이 실행되도록 만들면 된다.
```js
function runCode({code, input}) {
    return {
        stdout: '',
        stderr: '',
        exitCode: 0,
    };
}
```

## js에서 nsjail 실행시키기
node에서 nsjail을 실행시키려면 child_process의 spawn을 사용하면 된다. 이 때, arg와 config 파일 설정을 통해 nsjail을 커스텀할 수 있다.

## 삽질하면서 알게 된 것
- spawn은 비동기함수라서 Jest에서 실행시키면 항상 undefined를 받으므로 spawnSync를 사용해야한다.
- 실행하다가 "spawnSync /usr/local/bin/nsjail ENOENT"라는 에러가 떴는데, 이것은 node.js가 nsjail 경로를 찾지 못해서였는데, 알고보니 ubuntu용 node가 없고, window node로 실행되어서 window node가 해당 경로로 가게 되어 nsjail을 찾지 못하게 되는 것이였다. 그래서 바로 nvm을 깔아 주었다.

수많은 에러끝에 성공해냄.
![alt text](/images/OnPyRunner04/img01.png)


## 참고 자료
- [How can I sandbox Python in pure Python?](https://stackoverflow.com/questions/3068139/how-can-i-sandbox-python-in-pure-python)
- [Run Python in a sandbox with nsjail](https://publish.obsidian.md/kruzenshtern/writings/2021-05-21-run-python-in-a-sandbox-with-nsjail)
- [nsjail.dev](https://nsjail.dev/)
- [How to run nsjail on Google Cloud Run without prctl() errors?](https://stackoverflow.com/questions/79829221/how-to-run-nsjail-on-google-cloud-run-without-prctl-errors)
- [how to run nsjail in js 구글링했을때 AI 뜨는거](https://www.google.com/search?q=how+to+run+nsjail+in+js&sca_esv=07973ddbbf4d42d1&sxsrf=ANbL-n4AFyQDWnrbs-CWBX5c6q0b9dzAOg%3A1768474719587&ei=X8hoadXHI_Dh2roP2vzpwAc&ved=0ahUKEwiV89HbsY2SAxXwsFYBHVp-GngQ4dUDCBE&uact=5&oq=how+to+run+nsjail+in+js&gs_lp=Egxnd3Mtd2l6LXNlcnAiF2hvdyB0byBydW4gbnNqYWlsIGluIGpzMgoQABiwAxjWBBhHMgoQABiwAxjWBBhHMgoQABiwAxjWBBhHMgoQABiwAxjWBBhHMgoQABiwAxjWBBhHSMUKUABYAHABeAGQAQCYAQCgAQCqAQC4AQPIAQCYAgGgAgOYAwCIBgGQBgWSBwExoAcAsgcAuAcAwgcDMC4xyAcCgAgA&sclient=gws-wiz-serp)
- [[Jest] 테스트 코드로 JS 의 기능 및 로직 점검하기](https://velog.io/@skyu_dev/Jest-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%BD%94%EB%93%9C%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-JS%EC%9D%98-%EA%B8%B0%EB%8A%A5-%EC%A0%90%EA%B2%80%ED%95%98%EA%B8%B0)
- [[Linux] Node.js 설치하기 / 백엔드 가동하기](https://chan-co.tistory.com/139)