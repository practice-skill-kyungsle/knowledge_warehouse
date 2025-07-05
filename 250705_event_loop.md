## 1. 핵심 개념: 세 가지 실행 큐의 차이점

### 1.1 브라우저 이벤트 루프의 실행 순서

브라우저는 하나의 이벤트 루프 턴에서 다음 순서로 작업을 처리합니다:

```
한 번의 이벤트 루프 턴:
┌─────────────────────────────────────────────────────────────┐
│ 1. 동기 코드 실행 (Call Stack)                               │
│    ↓                                                        │
│ 2. 마이크로태스크 큐 모두 실행 ⭐ (가장 빠름)                  │
│    - Promise.then(), queueMicrotask()                      │
│    - 큐가 빌 때까지 계속 실행                                │
│    ↓                                                        │
│ 3. 매크로태스크 큐에서 하나만 실행 ⭐ (중간 속도)              │
│    - setTimeout(), setInterval()                           │
│    - 하나 실행 후 다시 1번으로                               │
│    ↓                                                        │
│ 4. 렌더링 (필요한 경우)                                      │
│    - 스타일 계산, 레이아웃, 페인트                            │
│    ↓                                                        │
│ 5. requestAnimationFrame 실행 ⭐ (가장 느림)                  │
│    - 렌더링 직전에 실행                                      │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 속도 차이의 핵심 원리

**마이크로태스크가 빠른 이유**: 큐가 빌 때까지 **연속으로** 실행
**매크로태스크가 느린 이유**: **하나씩만** 실행 후 다른 작업 처리
**프레임이 가장 느린 이유**: **60fps 주기**에 맞춰 실행

## 2. 실제 속도 측정과 분석

### 2.1 정확한 실행 시간 측정

```jsx
// 실제 실행 시간을 측정하는 함수
function measureExecutionTime(name, asyncFn) {
    const start = performance.now();

    asyncFn(() => {
        const end = performance.now();
        console.log(`${name}: ${(end - start).toFixed(3)}ms`);
    });
}

// 동시에 모든 방법을 테스트
function speedComparison() {
    console.log('=== 속도 비교 시작 ===');

    // 동기 코드 실행
    console.log('1. 동기 코드 실행');

    // 마이크로태스크들 (가장 빠름)
    measureExecutionTime('queueMicrotask', (callback) => {
        queueMicrotask(callback);
    });

    measureExecutionTime('Promise.then', (callback) => {
        Promise.resolve().then(callback);
    });

    // 매크로태스크들 (중간 속도)
    measureExecutionTime('setTimeout 0ms', (callback) => {
        setTimeout(callback, 0);
    });

    measureExecutionTime('setTimeout 1ms', (callback) => {
        setTimeout(callback, 1);
    });

    // 프레임 기반 (가장 느림)
    measureExecutionTime('requestAnimationFrame', (callback) => {
        requestAnimationFrame(callback);
    });

    console.log('6. 동기 코드 끝');
}

// 실행 결과 예시:
// 1. 동기 코드 실행
// 6. 동기 코드 끝
// queueMicrotask: 0.045ms        ← 가장 빠름
// Promise.then: 0.052ms          ← 두 번째로 빠름
// setTimeout 0ms: 4.123ms        ← 최소 4ms 지연
// setTimeout 1ms: 4.234ms        ← 여전히 4ms 지연
// requestAnimationFrame: 16.667ms ← 다음 프레임까지 대기

```

### 2.2 왜 이런 속도 차이가 발생하는가?

### A. 마이크로태스크 큐의 특성

```jsx
// 마이크로태스크는 현재 실행 스택이 비워지자마자 즉시 실행
function microtaskDemo() {
    console.log('시작');

    // 3개의 마이크로태스크를 연속으로 추가
    queueMicrotask(() => console.log('마이크로태스크 1'));
    queueMicrotask(() => console.log('마이크로태스크 2'));
    queueMicrotask(() => console.log('마이크로태스크 3'));

    // 매크로태스크 하나 추가
    setTimeout(() => console.log('매크로태스크 1'), 0);

    console.log('끝');
}

// 실행 결과:
// 시작
// 끝
// 마이크로태스크 1  ← 이 3개가 연속으로 실행
// 마이크로태스크 2  ←
// 마이크로태스크 3  ←
// 매크로태스크 1    ← 이후에 실행

```

### B. 매크로태스크 큐의 제한

```jsx
// 매크로태스크는 하나씩만 실행되고 다른 작업에 양보
function macrotaskDemo() {
    console.log('시작');

    // 3개의 매크로태스크를 추가
    setTimeout(() => {
        console.log('매크로태스크 1');
        // 이 시점에서 마이크로태스크 큐 확인
        queueMicrotask(() => console.log('매크로태스크 1 후 마이크로태스크'));
    }, 0);

    setTimeout(() => console.log('매크로태스크 2'), 0);
    setTimeout(() => console.log('매크로태스크 3'), 0);

    console.log('끝');
}

// 실행 결과:
// 시작
// 끝
// 매크로태스크 1
// 매크로태스크 1 후 마이크로태스크  ← 매크로태스크 사이에 마이크로태스크 실행
// 매크로태스크 2
// 매크로태스크 3

```

### C. 프레임 기반 실행의 특성

```jsx
// requestAnimationFrame은 브라우저 렌더링 주기에 맞춰 실행
function frameDemo() {
    let frameCount = 0;
    const startTime = performance.now();

    function measureFrame() {
        const currentTime = performance.now();
        const elapsed = currentTime - startTime;

        console.log(`프레임 ${++frameCount}: ${elapsed.toFixed(1)}ms`);

        if (frameCount < 5) {
            requestAnimationFrame(measureFrame);
        }
    }

    requestAnimationFrame(measureFrame);
}

// 실행 결과 (60fps 환경):
// 프레임 1: 16.7ms
// 프레임 2: 33.3ms
// 프레임 3: 50.0ms
// 프레임 4: 66.7ms
// 프레임 5: 83.3ms

```

## 3. React 상태 업데이트와의 상호작용

### 3.1 프레임보다 마이크로태스크/매크로태스크가 먼저인 이유

**핵심 원리**: 브라우저의 이벤트 루프는 JavaScript 실행을 렌더링보다 **항상 우선시**합니다.

### A. 브라우저 렌더링 파이프라인과 이벤트 루프의 관계

```jsx
// 이벤트 루프의 실제 동작 순서
한 번의 이벤트 루프 턴:
1. 동기 코드 실행 (Call Stack)
2. 마이크로태스크 큐 모두 실행        ← JavaScript 실행 영역
3. 매크로태스크 큐에서 하나 실행      ← JavaScript 실행 영역
4. 렌더링 (필요한 경우)              ← 브라우저 렌더링 영역
   - 스타일 재계산
   - 레이아웃 (리플로우)
   - 페인트 (리페인트)
   - 컴포지트
5. requestAnimationFrame 실행        ← 렌더링 직전에 실행

```

### B. 실제 실행 순서 증명

```jsx
// 프레임이 가장 늦게 실행되는 이유 실험
function provePriority() {
    console.log('=== 실행 순서 증명 실험 ===');

    // 모든 작업을 동시에 스케줄링
    const startTime = performance.now();

    // 1. 마이크로태스크 (가장 빠름)
    queueMicrotask(() => {
        const elapsed = performance.now() - startTime;
        console.log(`1. 마이크로태스크: ${elapsed.toFixed(3)}ms`);
    });

    // 2. 매크로태스크 (중간)
    setTimeout(() => {
        const elapsed = performance.now() - startTime;
        console.log(`2. 매크로태스크: ${elapsed.toFixed(3)}ms`);
    }, 0);

    // 3. 프레임 (가장 늦음)
    requestAnimationFrame(() => {
        const elapsed = performance.now() - startTime;
        console.log(`3. 프레임: ${elapsed.toFixed(3)}ms`);
    });

    console.log('모든 작업 스케줄링 완료');
}

// 실행 결과:
// 모든 작업 스케줄링 완료
// 1. 마이크로태스크: 0.045ms     ← 즉시 실행
// 2. 매크로태스크: 4.123ms       ← 최소 4ms 후
// 3. 프레임: 16.667ms            ← 다음 프레임까지 대기

```

### C. 왜 프레임이 가장 늦을까?

```jsx
// requestAnimationFrame의 실행 조건
const animationFrameConditions = {
    timing: '다음 리페인트 직전',
    frequency: '60fps = 16.67ms 간격',
    dependency: '브라우저 렌더링 주기에 종속',
    blocking: '탭이 비활성화되면 일시정지',

    // 가장 중요한 이유
    executionOrder: [
        '1. 모든 JavaScript 작업 완료 대기',
        '2. 마이크로태스크 큐 비우기',
        '3. 매크로태스크 처리',
        '4. 렌더링 필요 여부 확인',
        '5. requestAnimationFrame 실행',
        '6. 실제 렌더링 수행'
    ]
};

```

### D. 브라우저가 JavaScript를 우선시하는 이유

```jsx
// 브라우저 철학: "JavaScript First, Rendering Second"
const browserPhilosophy = {
    reason: 'JavaScript는 프로그램 로직이고, 렌더링은 시각적 표현',
    priority: [
        '1. 프로그램 로직 실행 (JavaScript)',
        '2. 상태 변경 처리 (마이크로태스크)',
        '3. 예약된 작업 처리 (매크로태스크)',
        '4. 화면 업데이트 (렌더링)'
    ],

    // 실제 증명
    proof: `
        // 만약 프레임이 더 빨랐다면?
        requestAnimationFrame(() => {
            // 이 코드가 먼저 실행되고
            console.log('프레임 실행');
        });

        queueMicrotask(() => {
            // 이 코드가 나중에 실행될 것입니다
            console.log('마이크로태스크 실행');
        });

        // 하지만 실제로는 마이크로태스크가 먼저 실행됩니다!
        // 왜냐하면 브라우저는 렌더링 전에 모든 JavaScript를 처리하기 때문입니다.
    `
};

```

## 4. 결론

React에서 비동기 작업을 처리할 때는 다음과 같은 우선순위를 고려해야 합니다:

1. **즉시 실행 필요**: `queueMicrotask()` 또는 `Promise.resolve().then()` (0~1ms)
2. **적절한 지연 필요**: `setTimeout(fn, 0)` (4ms)
3. **렌더링 동기화 필요**: `requestAnimationFrame()` (16.67ms)

**핵심 원리**는 마이크로태스크 큐가 매크로태스크 큐보다 우선순위가 높고, 프레임 기반 실행은 브라우저의 렌더링 주기에 맞춰 가장 느리게 실행된다는 것입니다.

이러한 차이를 이해하고 상황에 맞게 적절한 방법을 선택하면, 더 반응성이 좋고 성능이 뛰어난 React 애플리케이션을 만들 수 있습니다.

나는 아래와 같이 실제 사용을 해보았다.

```jsx
RootNavigationRef.current.dispatch(
  CommonActions.navigate(routeName, params)
);
queueMicrotask(() => {
  listener(deeplink);
});
return;
```
