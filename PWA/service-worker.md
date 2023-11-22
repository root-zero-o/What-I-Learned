# Service Worker
2023.11.21 <br/>
[GeunYeong Kim](https://github.com/root-zero-o)

## Service Worker?
- 브라우저가 백그라운드에서 실행하는 스크립트로, Javascript 파일 형태
- Worker context에서 실행
  - DOM에 접근할 수 없고, Javascript 메인 쓰레드와는 다른 쓰레드에서 실행
  - XHR, Web Storage는 Service Worker 안에서 사용할 수 없다
- HTTPS에서만 실행됩니다.(보안상의 이유)
- 프록시 서버와 비슷하게 동작 : 네트워크 요청을 중간에 가로채고, 리소스를 캐싱하는 역할 -> 브라우저의 외부에 위치하기 때문

##  Lifecycle
### 1. 설치중(installing)
- ServiceWorkerContainer.register() 메소드로 Service Worker가 등록됨
- 등록된 후 자바스크립트 파일이 다운로드되고 파싱되고 나면 **설치중** 상태
- 설치 성공 시 **설치됨(installed)** 상태가 되고, 실패 시 **중복(redundant)** 상태

### 2. 설치됨/대기중(installed/waiting)
- 서비스 워커가 설치되면 **설치됨** 상태
- 활성화 된 서비스 워커가 없다면 즉시 **활성화 중** 상태가 되고, 다른 서비스 워커가 앱을 제어하고 있다면 **대기중** 상태

### 3. 활성화중(activating)
- 서비스 워커가 활성화되기 직전의 상태
- 활성화 직전 activate 이벤트 발생. 이 때 waitUntil() 메서드로 **활성화 중** 상태로 유지

### 4. 활성화됨(activated)
- 서비스 워커가 활성화 된 상태
- 서비스 워커가 앱 제어, 동작 이벤트를 처리

### 5. 중복(Redundant)
- 서비스 워커가 등록 중 혹은 설치 중 실패하거나 새로운 버전으로 교체되면 **중복** 상태
- 서비스 워커가 앱에 아무런 영향을 미칠 수 없음

----

## CRA 코드 이해하기

### serviceWorkerRegistration.ts  
: service-worker.js 를 등록하고 이벤트를 동시에 처리하는 파일

- register() : service worker 등록하는 function

```typescript
// service worker 등록
export function register(config?: Config) {
  	// production 환경인지 && 서비스 워커를 지원하는지
	if (process.env.NODE_ENV === 'production' && 'serviceWorker' in navigator) {
    const publicUrl = new URL(process.env.PUBLIC_URL, window.location.href);
    if (publicUrl.origin !== window.location.origin) {
		// PUBLIC_URL이 다른 origin에 있으면 동작하지 않음
      return;
    }
	// 브라우저 로드가 완료되면
    window.addEventListener('load', () => {
      const swUrl = `${process.env.PUBLIC_URL}/service-worker.js`;
	  // localhost 일 때
      if (isLocalhost) {
        // 1) service worker가 존재하는 지 체크
        checkValidServiceWorker(swUrl, config);

        // 2) 친절하게 worker/PWA documentation 안내 ^^
        navigator.serviceWorker.ready.then(() => {
          console.log(
            'This web app is being served cache-first by a service ' +
              'worker. To learn more, visit https://cra.link/PWA',
          );
        });
      } else {
        // localhost가 아닐 때 -> 등록
        registerValidSW(swUrl, config);
      }
    });
  }
}
```

- checkValidServiceWorker() 
```typescript
function checkValidServiceWorker(swUrl: string, config?: Config) {
  fetch(swUrl, {
    headers: { 'Service-Worker': 'script' },
  })
    .then((response) => {
      // service worker가 있는지 확인
      const contentType = response.headers.get('content-type');
	  // 1) 없으면 -> 새로고침 
      if (
        response.status === 404 ||
        (contentType != null && contentType.indexOf('javascript') === -1)
      ) {
        navigator.serviceWorker.ready.then((registration) => {
          registration.unregister().then(() => {
            window.location.reload();
          });
        });
      } 
	  // 2) 있으면 -> 정상적으로 진행
		else {
        registerValidSW(swUrl, config);
      }
    })
    .catch(() => {
      console.log(
        'No internet connection found. App is running in offline mode.',
      );
    });
}

```

- registerValidSW() : 유효한 서비스 워커를 등록(```navigator.serviceWorker.register()```)
```typescript
function registerValidSW(swUrl: string, config?: Config) {
  // 등록!
  navigator.serviceWorker
    .register(swUrl)
    .then((registration) => {
      // eslint-disable-next-line no-param-reassign
      registration.onupdatefound = () => {
        const installingWorker = registration.installing;
        if (installingWorker == null) {
          return;
        }
        installingWorker.onstatechange = () => {
		  // installed 상태가 되면
          if (installingWorker.state === 'installed') {
			// 이전 서비스 워커가 아직 일하는 중
            if (navigator.serviceWorker.controller) {
              console.log(
                'New content is available and will be used when all ' +
                  'tabs for this page are closed. See https://cra.link/PWA.',
              );
              // Execute callback
              if (config && config.onUpdate) {
                config.onUpdate(registration);
              }
            } else {
              console.log('Content is cached for offline use.');
              // Execute callback
              if (config && config.onSuccess) {
                config.onSuccess(registration);
              }
            }
          }
        };
      };
    })
    .catch((error) => {
      console.error('Error during service worker registration:', error);
    });
}

```

- unregister() : 서비스 워커 사용하지 않을 때
```typescript
export function unregister() {
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.ready
      .then((registration) => {
        registration.unregister();
      })
      .catch((error) => {
        console.error(error.message);
      });
  }
}

```

### index.tsx
: 위에서 정의한 함수를 사용해 index.tsx에서 등록
```typescript
// 서비스 워커 등록
serviceWorkerRegistration.register();

// 사용 안할 때는
// serviceWorkerRegistration.unregister();
```

### service-worker.js
:  서비스 워커 🥹 (CRA에서는 기본적으로 Workbox 모듈을 사용)


```typescript
declare const self: ServiceWorkerGlobalScope; // Service Worker API 사용
```

```typescript
clientsClaim()
// 서비스 워커가 등록되고 설치되자마자 페이지들을 즉시 제어할 수 있도록
// top level에서 사용되어야 함(이벤트 핸들러 안에서 사용 불가)
```

```typescript
precacheAndRoute(self.__WB_MANIFEST);
// 파라미터로 주어진 path 의 엔트리들을 precache 리스트에 넣어 캐싱
// 캐싱이 진행된 path에 대해 라우팅이 일어날 경우 이에 응답
```

- ```self.__WB_MANIFEST``` 
  - webpack에서 설정한 precache 항목들
  - Webpack 이 service-worker.js 파일을 만들 때 참고하는 placeholder
  - CRA에는 이미 설정이 되어 있음
  
```typescript
// registerRoute
// Regex, string, 혹은 함수와 handler를 입력받아 원하는 asset이나 path, 파일에 따라 원하는 방식의 캐싱을 설정

registerRoute(
  // Return false to exempt requests from being fulfilled by index.html.
  ({ request, url }: { request: Request; url: URL }) => {
    // If this isn't a navigation, skip.
    if (request.mode !== 'navigate') {
      return false;
    }

    // If this is a URL that starts with /_, skip.
    if (url.pathname.startsWith('/_')) {
      return false;
    }

    // If this looks like a URL for a resource, because it contains
    // a file extension, skip.
    if (url.pathname.match(fileExtensionRegexp)) {
      return false;
    }

    // Return true to signal that we want to use the handler.
    return true;
  },
  createHandlerBoundToURL(process.env.PUBLIC_URL + '/index.html'),
);

// precache로 핸들되지 않은 요청에 대해 런타임 캐싱
registerRoute(
  ({ url }) =>
	// same-origin에서 .png 요청(/public)을 캐싱
    url.origin === self.location.origin && url.pathname.endsWith('.png'),
  // 캐싱 방식은 변경할 수 있음(5가지) - 아래 참고
  new StaleWhileRevalidate({
    cacheName: 'images',
    plugins: [
	  // 런타임 캐시가 최대 사이즈에 도달하면, 가장 최근에 사용되지 않은 이미지부터 삭제
      new ExpirationPlugin({ maxEntries: 50 }),
    ],
  }),
);
```

- registerRoute 캐싱 방식 5가지
  - ```StaleWhileRevalidate``` : 캐싱된 response가 있으면 바로 응답, 아니면 네트워크 요청 fallback이 일어난다. 캐싱된 response로 응답한 뒤에 백그라운드에서 네트워크 요청으로 캐시 업데이트
  - ```CacheFirst``` : 캐싱된 response가 있으면 바로 응답, 캐시 업데이트는 안함
  - ````NetworkFirst```` : 네트워크 요청 먼저, 실패했을 경우 캐시로 응답
  - ```NetworkOnly``` : 캐싱을 전혀 사용 안함
  - ```CacheOnly``` : 응답은 캐시에서만 받아오는 방식. Precaching이 수동으로 진행되는 경우에만 의미 있는 방식
- 콘솔에서 캐싱된 images를 확인할 수 있음(CacheStorage)

```typescript
// 웹앱에서 skitWaiting을 registration.waiting.postMessage({type: 'SKIP_WAITING'})로 트리거 했을 때
self.addEventListener('message', (event) => {
  if (event.data && event.data.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});
```


## Reference
- [Service Worker API - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [서비스 워커에 대해 알아보고 Mock Response 만들기 | 카카오엔터테인먼트 FE 기술블로그](https://fe-developers.kakaoent.com/2022/221208-service-worker/)
- [Workbox - Chrome for Developers](https://developer.chrome.com/docs/workbox/)
- [Progressive Web App \(2\) React PWA 적용 방법](https://elice.io/newsroom/pwa_2)
- [Workbox로 오프라인 페이지 처리방법](https://dawan0111.github.io/javascript/workbox%EB%A1%9C-%EC%BA%90%EC%8B%B1-%EB%B0%8F-%EC%98%A4%ED%94%84%EB%9D%BC%EC%9D%B8-%ED%8E%98%EC%9D%B4%EC%A7%80-%EC%B2%98%EB%A6%AC%EB%B0%A9%EB%B2%95/)
