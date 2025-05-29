# front_5th_chapter4-1

### 배포 파이프라인 프로세스

![아키텍처 다이어그램](/diagram.png)

1. 코드 변경 후 github 에 push
2. github actions 가 트리거
3. next.js 어플리케이션 빌드 및 테스트
4. 빌드된 정적 파일을 S3 버킷에 업로드
5. CloudFront 캐시 무효화
6. 글로벌 엣지 로케이션에 새 콘텐츠 배포
7. 사용자가 최신 버전에 즉시 접근 가능

## 주요 링크

#### S3 버킷 웹사이트 엔드포인트

- http://hanghae-frontend-chapter-4.s3-website-us-east-1.amazonaws.com

#### CloudFrount 배포 도메인 이름

- https://d347q9yh20adyy.cloudfront.net/

## 주요 개념

#### CI/CD 도구와 GitHub Actions의 역할

CI: Continuous Integration 지속적 통합
코드 변경사항을 중앙 저장소에 통합하는 것으로 아래 과정을 거친다.
`코드 커밋 -> 자동 빌드 -> 자동 테스트 -> 피드백`

CD: Continuous Deployment 지속적 배포
코드 변경사항이 자동으로 프로덕션 환경까지 배포되는 것으로 아래 과정을 거친다.

과제의 deployment.yml 파일에서는 다음과 같이 워크플로우의 이름을 정의하고 Action 섹션에서 같은 이름의 워크플로우 UI 를 확인할 수 있다.

`name: Deploy Next.js to S3 and invalidate CloudFront`

그리고 아래와 같이 트리거를 설정하여 main 브랜치에서 git push 이벤트가 발생할 때 워크플로우를 실행하도록 한다.

```yaml
on:
	push:
		branches:
		- main
```

아래와 같이 배포 환경을 우분투로 설정하고 깃헙 액션에서 체크아웃 하는 작업 단계를 정의한다.

```yaml
deploy:
	runs-on: ubuntu-latest

	steps:
	- name: Checkout repository
	  uses: actions/checkout@v4
```

#### 스토리지의 개념과 S3란?

데이터를 저장하는 모든 방식을 스토리지라고 통칭한다.
스토리지는 3가지 종류가 있는데 블록 스토리지, 파일 스토리지, 객체 스토리지가 있다.
각 예시는 다음과 같다.

- 블록 스토리지 (하드디스크, SSD)

```
구조: [블록1][블록2][블록3][블록4]...
접근: 직접 블록 주소로 접근
```

- 파일 스토리지 (NAS, 파일 시스템)

```
구조: /폴더1/폴더2/파일.txt
접근: 파일 경로로 접근
```

- 객체 스토리지 (S3, Google Cloud Storage)

```
구조: 버킷/객체키 + 메타데이터
접근: HTTP URL로 접근
```

객체 스토리지가 다른 스토리지와 비교했을 때,
키-메타데이터의 구조로 이루어져 있어 계층적인 구조보다 확장성이 크다.
그리고 각 파일이 URL 을 고유한 키 값으로 가지기 때문에 HTTP 기반 접근이 쉽다는 특징이 있다.

대표적인 객체 스토리지 서비스가 AWS의 Amazon S3(Simple Storage Service)이다.
주로 파일을 저장하거나 정적 웹사이트를 호스팅하는데 사용하고 액세스 제어, 버전 관리 등의 기능을 지원한다.
프론트엔드에서는 주로 빌드된 정적 파일들(HTML, CSS, JS, 이미지 등)을 저장하고 웹 서비스에서 이미지 링크를 통해 접근하는 사례가 많다.

#### CDN와 CloudFront의 역할

- CDN 이란?
  전 세계에 분산된 서버 네트워크를 통해 사용자와 가장 가까운 위치에서 콘텐츠를 제공하는 시스템이다.

- CDN 동작 원리

1. 사용자(서울) -> 가장 가까운 CDN 서버
2. 캐시가 있다면 -> 즉시 응답
3. 캐시가 없다면 -> origin 서버에서 가져와서 캐시 후 응답

그렇기 때문에 CDN 을 통하지 않고 사용자(서울) -> origin 서버(미국)로 통신한다면 기본 통신시간과 네트워크 지연으로 인해 추가적으로 200ms 이상의 응답시간이 소요될 수 있다.

하지만 CDN 을 사용한다면 사용자(서울) -> CDN 엣지(서울) -> 캐시된 콘텐츠로 접근하기 때문에 응답시간은 20ms 정도로 줄어들게 된다.

그리고 Amazon CloudFront는 AWS의 글로벌 CDN(Content Delivery Network) 서비스이다. 오리진 서버(S3 등)의 부하를 줄이고, HTTPS 적용, 압축, 보안 헤더 추가 등의 추가적인 최적화 기능을 제공한다.

그리고 CloudFront 에서는 다음과 같이 설정하게 되면 정적 파일 압축으로 최적화가 가능하다.

```yaml
CloudFront 설정:
  Compress: true # Gzip 압축 자동
  HTTP2Support: true # HTTP/2 지원
  IPV6: true # IPv6 지원
  PriceClass: PriceClass_100 # 북미/유럽/아시아만 사용
```

#### 캐시 무효화(Cache Invalidation)

브라우저 또는 CDN 에 저장된 기존 캐시를 삭제하여 새로운 콘텐츠를 가져오도록 하는 프로세스이다.  
예를 들어, 웹사이트 업데이트 시 사용자가 이전 버전의 캐시된 파일을 보지 않도록 하는 역할을 한다.

아래와 같이 워크플로우 yaml 파일에서 작성할 경우, CI/CD 파이프라인에서 새로운 코드가 배포될 때마다 캐시 무효화도 자동으로 실행된다.

```yaml
# .github/workflows/deploy.yml
- name: Invalidate CloudFront
  run: |
    aws cloudfront create-invalidation \
      --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
      --paths "/*"
```

#### 환경변수와 Repository secret

환경변수는 애플리케이션 실행 시 필요한 설정값이다.
코드 외부에서 주입하는 방식 개발/스테이징/프로덕션 등 다른 환경에서 다른 값을 사용하여 주입하는 방식으로 사용된다.

Repository secret 은 GitHub Repository에서 API 키, 액세스 토큰, 비밀번호 등의 중요 정보를 안전하게 저장하고 관리하는 기능이다. 그리고 암호화되어 저장되며 GitHub Actions 워크플로우에서만 접근 가능하다.

## CDN과 성능최적화

#### 웹 애플리케이션의 초기 로딩 시간 단축

첫번째 사진은 S3 정적 호스팅으로 접근한 응답 헤더고 두번째 사진은 CloudFront 호스팅으로 접근한 응답헤더이다. CloudFront 로 접근할 때 첫번째 요청 이후에는 Cache Hit 로 S3 까지 가지 않고 가장 가까운 엣지 서버에서 즉시 응답한다. 이때 두번째 사진처럼 X-Cache 속성 값이 Hit from cloudfront 로 표시되는 것을 확인할 수 있다.

![S3Cache](/S3Cache.png)

![CFCache](/CFCache.png)

그렇기 때문에 페이지가 로딩 되는 속도 또한 단축되고 아래와 같이 첫번째 사진에서는 518ms 의 시간이 소요되지만 두번째 CloudFront 로 배포한 주소로 접근 시 23ms 로 로딩 시간이 약 95.6% 감소한 것을 확인할 수 있다.

![CFLoadTime](/CFLoadTime.png)

![CFLoadTime](/CFLoadTime.png)

#### Core Web Vitals 지표 향상

Lighthouse 를 통해 성능 보고서를 확인할 수 있는데 이때, LCP(Largest Contentful Paint)가 1.8초에서 0.4초로 78% 개선되었고 전체 성능 점수가 90점에서 100점으로 향상되었다.

![S3LightHouse](/S3LightHouse.png)

![CFLightHouse](/CFLightHouse.png)

#### Lighthouse 란?

Google Chrome에서 제공하는 웹앱 진단 도구이고 성능, 접근성, SEO 등 사이트에 대한 전반적인 체크가 가능하다. 성능의 측정 항목은 6 가지로 각 성능 지표는 페이지가 로드 되는 속도를 다양한 측면에서 측정한다. 성능지표마다 속도와 점수에 대한 기준은 다르다.

#### LCP 란?

먼저 가장 변화가 큰 지표부터 살펴보자면 LCP(Largest Contentful Paint) 이다.
명칭에서 알 수 있듯이 페이지에서 가장 큰 콘텐츠 요소가 렌더링되는 시간을 측정한 값이다.
짧을수록 사용자가 체감하는 로딩 속도가 빠른 것이다.

LCP로 측정되는 콘텐츠 요소는 다음과 같다.

- `<img>` 요소
- `<svg>` 안에 있는 `<image>` 요소
- `<video>` 요소
- background-image 속성의 url()로 로드되는 요소
- block 레벨 요소 (elements containing text nodes or other inline-level text elements children)

#### FCP 란?

그리고 FCP(First Contentful Paint)는 초기 DOM 콘텐츠를 렌더링하는데 걸리는 시간을 측정한 값이다.
사용자가 봤을 때 첫번째 텍스트 또는 이미지가 표시되는 시간을 의미한다. FCP 측정 시점을 시작점으로 하는 다른 지표들이 있어서 가장 먼저 찍히는 FCP는 다른 항목들의 점수 및 전체 점수에 영향을 줄 수 있다.

또, DOMContentLoaded 시점과 같이 볼 필요가 있다.
DOMContentLoaded 는 HTML 파싱이 완료되고 DOM 트리가 구성된 시점으로 CSS, 이미지 등은 아직 로딩 중일 수 있다. 하지만 FCP 는 사용자가 실제로 콘텐츠를 보는 시점이기 때문에 DOMContentLoaded 시점과 FCP 시점의 차이가 클수록 CSS/폰트 최적화가 필요하다는 것을 의미한다.

#### TBT 란?

TBT(Total Blocking Time) 는 FCP부터 TTI(Time to Interactive)까지 메인 스레드가 차단된 총 시간이다.
메인 스레드가 50ms 이상 연속으로 실행되는 작업 시간들만 합한 값으로 즉, TBT 가 높을수록 사용자의 인터렉션에 대한 반응이 늦어질 수 있는 확률이 커서 사용자 경험이 나빠진다는 것을 의미한다.
