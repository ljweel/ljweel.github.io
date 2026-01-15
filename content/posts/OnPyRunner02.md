---
title: "OnPyRunner (2)" # 리스트에 표시될 제목
description: "온라인 파이썬 인터프리터 플랫폼에 대한 의식의 흐름" # 제목 아래에 보일 요약문
summary: "이 글은 OnPyRunner를 구현하는 중 특정 시점에서의 생각과 판단을 기록한 글입니다." # 리스트에서 미리보기로 보여줄 글자들
tags: ['OnPyRunner']
categories: ['dev']
author: "ljweel"
date: 2026-01-10
showToc: true
TocOpen: false
draft: false
---

## [이전 글](https://ljweel.github.io/posts/onpyrunner01/) 요약

- 온라인 파이썬 실행기를 만들고 싶다!
- 유스케이스와 다이어그램을 고민하고 문제점을 파악했다!
- 개선한 다이어그램을 그렸다!

## ExpressJs

사실 express 프레임워크를 사용안해봤기 때문에, 이참에 이번 프로젝트에서 써보려고 한다. 백엔드에 있어서 프레임워크는 별로 안 중요하다고 생각하는 편..~~그냥 써본 프레임워크가 적음~~

npm init 하고, redis, express install 하고, 시작해보자..

## 설계를 구현으로

사실 설계는 저렇게 했지만, 코드로 어떻게 구현하는지는 다른 이야기다. redis와 express 사용 경험이 없기 때문. 그래서 전체적인 설계를 세부적이고 구체적인 코드 구현으로 쪼개어 생각해야한다. 다음과 같이 쪼개어 코드를 작성해 보았다.

#### [API 호출]
1. 요청
2. job 생성 및 MQ에 job push
3. job id만 반환

#### [Worker 프로세스]
4. MQ에서 job pop
5. Python 실행 (격리)
6. stdout / stderr 반환
7. 결과 Redis 저장

#### [API 조회]
8. jobId로 Redis 조회
9. 상태/결과 반환


좀 찾아보니까 redis 메시지큐로 bullMQ를 많이 사용하는 거 같아서 이걸 쓰기로 했다.

[bullMQ 공식문서](https://docs.bullmq.io/)

bullMQ를 테스트 해보려면 도커에 redis를 띄워야한다. [참고 사이트](https://jindevelopetravel0919.tistory.com/391)

## 지금까지 한거

결국 app.js, worker.js, queue.js 3개를 만들어서 로컬에서 api 호출해서 worker에 도달하는지까지 테스트해 보았다. 코드는 다음과 같다.

**`app.js`**
```javascript
import express from 'express';
import { addJobs } from './queue.js';

const app = express();
app.use(express.json());
app.listen(3000);


app.post('/api/jobs', async (req, res) => { // 1. 요청
    const { code, input } = req.body;
    const job = await addJobs(code, input); // 2. job 생성 및 MQ에 job push

    console.log(code, input);

    res.status(202).json({ // 3. job.id 반환
        jobId: job.id,
        status: 'queued',
    });

});

// 8. jobId로 Redis 조회
// 9. 상태/결과 반환
```

**`queue.js`**

```javascript
import { Queue } from 'bullmq';
import IORedis from 'ioredis';

const jobQueue = new Queue('jobQueue');

async function addJobs(code, input) {
    return jobQueue.add('run-code', {'code': code, 'input': input});
}

const connection = new IORedis({ maxRetriesPerRequest: null });

export {jobQueue, addJobs, connection};
```

**`worker.js`**

```javascript
import { Worker } from 'bullmq';
import { connection } from './queue.js';

const worker = new Worker(
    'jobQueue',
    async (job) => { // 4. MQ에서 job pop
        const { code, input } = job.data;

        console.log('job popped:', job.id);
        console.log('job.data:', job.data);
        // 5. Python 실행 (격리)
        // 6. stdout / stderr 반환
        // 7. 결과 Redis 저장
    },
    { connection },
);
```


도커에 redis를 띄우고, node worker.js와 node app.js를 해주고 postman에서 api 요청을 날려보면 다음과 같이 나온다!

![alt text](/images/OnPyRunner02/img01.png)
![alt text](/images/OnPyRunner02/img02.png)
