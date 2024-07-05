## Node.js Event Loop

```javascript

async function getSomething() {
    console.log('test start');
    const test = await repo.getSomething(id);
    console.log(test);
    console.log('test end');
}

```
위 코드는 Node.js 내부에서 어떻게 실행될까? 이 글을 읽고 난 후에는 이해 할 수 있을 것이다.

---

&nbsp;
Node.js는 C++로 작성된 Javascript 런타임 환경이다. 그리고 Javascript는 Single Thread && 절대 블로킹 연상을 하지 않는다는 특성이 있다.
&nbsp;

이것이 Node.js는 싱글 스레드다. 라고 불리는 이유이다.
그런데 싱글 스레드라고 하면 I/O 작업이 끝날때까지 스레드가 멈춰버린다거나 해야하는데 그렇지는 않다. 이는 Node.js에 탑재된 libuv란 라이브러리와 그 유명한 Event Loop덕분이다.

### 들어가기 앞서
이글은 Node.js애서 코드가 어떻게 실행되는지에 대한 대략적인 설명이다. 따라서 비약이 많을 수 있다.

### libuv
libuv는 Node.js에서 비동기 작업을 처리할 때 사용하는 라이브러리이다. 로컬에 있는 파일의 I/O, DB에 Record생성, 불러오기, API 호출등은 모두 외부와의 통신에 속한다. 즉 Node.js의 실행 프로세스 외부의 일이란 것이다.

libuv는 이런 부분들을 처리해준다. 즉 libuv는 비동기 작업을 위한 Node.js의 인터페이스라고도 할 수 있을 것 같다. 

libuv는 운영체제(OS)에 작업을 요청하기도 하고, OS에서 지원하지 않는 작업 요청일 경우 libuv 내부의 Thread Pool에 작업을 요청한다.

> "libuv 내부의 Thread Pool" 이라고 했는데, 이는 Node.js의 메인스레드를 이야기 하는 것이 아니다. 이 Thread Pool 덕분에 Node.js가 싱글 스레드 이지만 Non-blocking으로 비동기 작업들(Asynchronous) 작업들을 할 수 있다. 

표현하기는 힘들지만 Node.js의 메인 스레드는 싱글 스레드가 맞다. 그리고 이 싱글 스레드가 이벤트 루프를 돌며 하나하나 작업들을 실행한다. 

그러나 이벤트 루프에 작업들(비동기 작업들)을 추가하고 작업이 완료된 비동기 task들을 추가하는 것은 libuv의 다른 Thread들 이다.

### Event Loop
*Event Loop는 자료구조나 어떤 구현체가 아니다. 6개의 Phase를 활용하는 방식이라고 생각하는 것이 편하다.*

윗 문단에서 Event Loop에 task(실행해야할 코드 라고 생각하면 된다.)들을 추가 하는 것이 libuv가 하는 역할이라고 설명했다. 그렇다면 이제 우리의 메인 스레드가 Event Loop를 어떻게 활용하는지 살펴보자.

Event Loop는 6개의 Phase로 구성되어있고 다음과 같다.
- Timer Phase
- Pending Callbacks Phase
- Idle, Prepare Phase
- Poll Phase
- Check Phase
- Close Callbacks Phase

각 Phase가 어떤역할을 하는지는 크게 중요한 내용은 아니라고 생각한다. 그냥 6개의 영역이 있고 따로따로 쓰는구나 라고 이해해도 좋다. 각 Phase에는 각자의 Queue를 가지고 있다.
&nbsp;

메인 스레드는 각 Phase를 언급된 순서대로 순회하면서 Queue에 있는 task들을 실행시킨다. 이렇게 메인 스레드가 한 Phase에서 다음 Phase로 넘어가는 것을 Tick이라고 한다.

&nbsp;
Tick이 발생하기 위해선(즉 메인스레드가 다음 Phase로 가기 위해선) 둘 중 하나의 조건을 만족해야하는데 다음과 같다.
1. Phase에 있는 Queue가 다 비었다.(Phase의 Task를 모두 실행시킴)
2. 시스템의 실행 한도에 다달랐다. (같은 Phase에서 무한대로 있는 상황을 방지)


### nextTickQueue와 microTaskQueue
nextTickQueue와 microTaskQueue 역시도 Node.js가 비동기 Task를 처리하는 중요한 요소인데 필요한 부분만 살펴보자

우선 가장 중요한 점은 nextTickQueue와 microTaskQueue는 Event Loop의 요소가 아니라는 점이다. (그러니까 아에 별개다)

그리고 두 번째는 이 Queue들 역시도 Task를 담아뒀다가 메인 스레드가 가져다가 실행시키라고 있는 Queue들이다.

그리고 마지막으로는 이 Queue들의 Task는 메인 스레드가 언제 실행시키는가 인데 그건 아래와 같다.

nextTickQueue :
- process.nextTick() 함수에 의해 추가된 콜백을 관리한다.
- 메인 스레드가 현재 Phase의 Task들을 모두 끝내고 현재 Phase가 끝난 직후에 실행된다.

microTaskQueue :
- 주로 Promise의 .then(), .catch(), .finally() 콜백에 의해 추가된다.
- 각 Phase가 끝난후 다음 Phase로 넘어가기 전에 실행된다. 
- 각 Phase가 끝날 때마다 비어있을 때까지 실행된다. 

nextTickQueue의 실행 순서가 microTaskQueue의 실행순서에 앞선다.



### 결론
```javascript

async function getSomething() {
    console.log('test start');
    const test = await repo.getSomething(id);
    console.log(test);
    console.log('test end');
}

```
코드를 다시 살펴보며 내가 메인 스레드다 생각하고 실행흐름을 살펴보자.

1. console.log('test start');가 실행된다. 이 Task는 비동기 작업이 아니기 때문에 동기적으로 바로 실행시켜버린다. 아직까지 Event Loop는 생성되지도 않았다.

2. 메인스레드가 비동기 작업을 만났다. 이제 Event Loop가 생성된다. 

3. 메인스레드가 libuv에 const test = await repo.getSomething(id); 작업을 요청하고 await 키워드로 인해서 repo.getSomthing(id); Promise가 반환 될 때까지 코드 실행을 멈춘다.

4. 메인스레드는 await로 인해서 const test = await repo.getSomething(id);가 중지되었고 그 아래 코드들은 microTaskQueue로 들어간다.
 
5. 더 이상 Poll Phase에 머물 이유가 없기 때문에 다음 Phase들로 이동한다. 

6. Promise가 해결되면 test 변수에 repo.getSomthing(id)의 resolve 값이 할당된다.

7. 메인 스레드는 Tick이 발생하는 사이에 microTaskQueue를 확인하여 
  console.log(test);
  console.log('test end');
  를 실행시킨다.

### 마무리
Node.js가 싱글 스레드이지만 여러 비동기 작업을 수행할 수 있는 이유는 libuv의 서브 스레드이 비동기 작업을 대신 해주고 => Event Loop나 nextTickQueue, microTaskQueue에 비동기 작업의 결과가 들어가고 => 메인 스레드가 Phase에서 Phase로 이벤트 루프를 돌며 Task들을 실행하기 때문이다. 

(위의 nextTickQueue, microTaskQueue는 Node 버전에 따라 조금씩 상이하다. v11.0.0 이상의 버전에서는 Phase와 Phase 사이가 아니라 현재 작업중이 Task가 끝날 때마다 Queue들을 확인해서 실행시킨다.)