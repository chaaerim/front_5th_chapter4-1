## 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://chaaerim-s3.s3-website.ap-northeast-2.amazonaws.com/
- CloudFrount 배포 도메인 이름: https://d1i8xvsglsnufd.cloudfront.net/

## 주요 개념

### GitHub Actions과 CI/CD 도구

- GitHub Actions
    - GitHub에서 제공하는 CI(Continuous Integration, 지속 통합)와 CD(Continuous Deployment, 지속 배포)를 위한 서비스이다. GitHub Actions를 사용하면 자동으로 코드 저장소에서 어떤 이벤트(event)가 발생했을 때 특정 작업이 일어나게 하거나 주기적으로 어떤 작업들을 반복해서 실행시킬 수 있다.
- Github Actions의 구성요소
    - Workflow
        - 최상위 개념
        - 여러 Job으로 구성되고, Event에 의해 트리거될 수 있는 자동화된 프로세스.
        - Workflow 파일은 YAML로 작성되고, .github/workflows 폴더 아래에 저장.
    - Event
        - Workflow를 트리거하는 특정 활동이나 규칙.
            - 특정 브랜치로 Push하거나
            - 특정 브랜치로 Pull Request하거나
            - 특정 시간대에 반복(Cron)
            - Webhook을 사용해 외부 이벤트를 통해 실행
    - Job
        - Job은 여러 Step으로 구성되고 가상 환경의 인스턴스에서 실행.
        - 여러 Job 간 의존 관계를 가질 수 있고 기본적으로 병렬 실행.
    - Step
        - Task들의 집합으로 커맨드를 실행하거나 action을 실행할 수 있음.
    - Action
        - Workflow의 가장 작은 블럭
        - Job을 만들기 위해 Step 들을 연결할 수 있음.
    - Runner
        - Github Action Runner 어플리케이션이 설치된 머신으로 Workflow가 실행될 인스턴스.
        - Github에서 호스팅해주는 Github-hosted runner와 직접 호스팅하는 Self-hosted runner로 나뉨.

- 과제 deployment.yaml

```jsx
name: Deploy Next.js to S3 and invalidate CloudFront

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: latest

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build
        run: pnpm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      # ✅ 캐싱 가능한 정적 자산 (Next.js 빌드된 JS, CSS 등)
      - name: Upload static assets with long cache
        run: |
          aws s3 sync out/_next/ s3://${{ secrets.S3_BUCKET_NAME }}/_next/ \
            --cache-control "public, max-age=31536000, immutable" \
            --delete

      # ✅ HTML 및 기타 파일 (항상 최신화가 필요)
      - name: Upload HTML and other assets with no cache
        run: |
          aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }}/ \
            --exclude "_next/*" \
            --cache-control "public, max-age=0, must-revalidate" \
            --delete

      # ✅ CloudFront 캐시 무효화
      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

- 위 deployment.yaml파일은 yaml파일로 정의된 배포 파이프라인으로 main branch에 push가 발생할 때마다 트리거 되며 workflow_dispatch로 직접 실행시킬 수도 있다. `ubuntu-latest` GitHub-hosted runner에서 실행되며 코드 체크아웃 후에 pnpm setup, 의존성 설치, 빌드를 거치고 Github에 저장된 secrets을 이용하여 AWS 자격 증명 설정을 하게 된다. 이후 빌드된 정적 결과물들을 S3 버킷에 업로드하며 이때 cache-control 메타데이터도 설정하여 적절히 캐싱을 활용할 수 있도록 했다. 마지막으로 CloudFront 캐시 무효화를 시켜 변경된 빌드 결과물이 반영되어 배포될 수 있도록 한다.

<br/>

### S3와 스토리지

- 컴퓨터에서 **스토리지**란 데이터를 저장하고 보존하는 역할을 하는 장치를 말한다. 클라우드 서비스 업체에서는 인터넷을 통해 원격 서버에 데이터를 저장, 액세스, 보호 및 분석을 할 수 있는 **스토리지 서비스**를 제공한다. Amazon S3(Simple Storage Service)는 아마존 웹 서비스(AWS)가 제공하는 클라우드 스토리지 서비스이다. 웹 사이트, 모바일 애플리케이션, 백업 및 복원, 빅 데이터 분석 등 다양한 사용 사례에서 원하는 양의 데이터를 저장하고 보호할 수 있다.
- 데이터를 객체 단위로 저장이 되며, 객체 별 접근 허가 Access Control List, ACL 를 통해 데이터에 접근 가능한 사용자 특정이 가능하다.

<br/>


### CloudFront와 CDN

- 콘텐츠 전송 네트워크(CDN)는 지리적 제약 없이 전세계 사용자에게 빠르고 안전하게 콘텐츠를 전송할 수 있게 하는 기술이다. 사용자가 웹 사이트를 방문하면 해당 웹 서버의 데이터는 사용자의 컴퓨터에 도달하기 위해 **인터넷**을 통해 이동한다. 따라서 사용자가 서버로부터 멀리 있는 경우에는 대용량 파일을 로드하는 데 오랜 시간이 걸린다. 이러한 방법 대신 웹 사이트 콘텐츠를 **사용자와 지리적으로 가까운 CDN 서버**에 저장하면 데이터가 훨씬 빨리 도달하게 할 수 있다.
- Amazon CloudFront는 AWS에서 제공하는 Global CDN 서비스이다. CloudFront는 전 세계에 분포된 엣지 로케이션을 사용하여 콘텐츠를 캐시하고 분배한다. 사용자들은 가까운 엣지 로케이션에서 콘텐츠를 전달 받아 빠르게 콘텐츠에 접근할 수 있다. 즉 원래 데이터가 저장되어있는 원본 서버에서 직접 콘텐츠를 가져오는 것보다 데이터 전송 속도가 빠를 수 밖에 없다. 물리적으로 거리가 가깝기 때문이다. 또한 S3와 연동하면, 캐싱을 지원하여 S3에 저장된 컨텐츠에 직접 접근하지 않아도 되어 응답이 빠르고 S3의 비용 또한 절감할 수 있다.
- 또한 CDN을 사용하면 네트워크 왕복 지연이 감소될 수 있다. 기본적으로 https를 사용하기 위해서는 TLS handshake 과정이 필요하다. 이 과정에서 TCP 3 way handshake가 일어나며 패킷을 주고 받고 이후 TLS ClientHello ⇄ ServerHello 등 두번의 왕복이 더 필요하게 된다. 여기서 대부분의 시간은 클라이언트와 서버 간 패킷을 전송하는데 걸리는 시간이다. 따라서 데이터가 왕복하는데 걸리는 시간인 RTT 시간을 줄이는 것이 TLS handshake를 맺는 시간을 줄이게 된다. CDN은 물리적으로 유저와 가까운 곳에 위치하기 때문에 RTT가 대폭 감소하게 되어 데이터에 대한 응답을 받기 전에 걸리는 시간 자체를 줄이는데도 CDN을 사용하면 매우 유리하다.

<br/>


### 캐시와 캐시 무효화(Cache Invalidation)

- 캐시는 일시적인 시간동안 데이터의 일부를 미리 저장하는 계층이다.
    - 서버와 HTTP 통신을 할 때 헤더의 Cache-Control 값에 따라 HTTP 응답으로 받은 리소스를 캐싱할 수 있다.
    - 이때 캐시의 유효기간은 max-age로 설정하게 되며 이 시간 동안 캐시가 유효하게 된다.
    - 한 번 받아온 리소스의 유효 기간이 지나기 전이라면, 브라우저는 서버에 요청을 보내지 않고 디스크 또는 메모리에서만 캐시를 읽어와 계속 사용한다.
- 캐시 무효화는 캐시에 저장된 데이터를 유요하지 않은 데이터로 마킹하는 작업이다. 다음 요청이 왔을 때, 유효하지 않은 데이터는 반드시 `캐시-미스` 로 다루고 원본 저장소에서 값을 재생성 하도록 해야 한다.
    - 캐시된 데이터는 실제 원본 데이터가 아니다. 원본 데이터는 데이터베이스에 있거나 서비스가 생성해야 합니다. 문제는 DB 의 데이터가 변경되는 경우 발생하는데 이 경우 캐시된 데이터는 더 이상 유효하지 않다. 만약 캐시된 데이터가 유효하지 않다면, 모순되거나 부정확한 정보를 제공할 수 있기 때문에 적절히 캐시를 무효화시키는 것이 중요하다.
    - 일반적으로 캐시를 없애기 위해서 “CDN Invalidation”을 수행한다. 그러나 브라우저의 캐시는 다른 곳에 위치하기 때문에 CDN 캐시를 삭제한다고 해서 브라우저 캐시가 삭제되지는 않는다. 따라서 Cache-Control의 max-age 값은 신중히 설정하여야 한다.

<br/>


### Repository secret과 환경변수

GitHub 레포지토리 설정에서 중요한 정보를 레포지토리 환경에 저장하면 해당 secret을 Github Actions에서 사용할 수 있다. 

<br/>


## CDN과 성능최적화

### 배포 파이프라인
<img width="907" alt="image" src="https://github.com/user-attachments/assets/0789de9e-5e8d-4fac-8fea-937c45fa4a95" />


### 캐싱 전략 
- **CloudFront 사용 중이라면** S3에서 설정한 `Cache-Control`이 그대로 전달된다.
- 브라우저 캐시 외에도 CDN 캐시를 `max-age`로 제어할 수 있다.
- 따라서 _next/static 하위에 빌드된 정적 자산들은 변경될 가능성이 적다고 판단하여 `max-age=31536000, immutable`로 Cache-Control을 설정했고 index.html은 항상 변경된 최신 페이지를 렌더해야하기 때문에 `max-age=0, must-revalidate로` Cache-Control을 설정하였다.

### cdn 적용 전 s3로 배포한 결과 (cache control 세팅 안했을 때)
![image](https://github.com/user-attachments/assets/69a387ef-d6ae-4cce-8187-14a19fde7ebb)


### cdn 적용 후(cache control 설정 안했을 때)
![image](https://github.com/user-attachments/assets/79cba6dd-d3ff-4264-9828-c480f7ec6094)
- s3만을 사용해서 배포를 했을 때에는 document 파일의 size는 12.5kB, 응답에 걸린 시간은 45ms였다. cdn을 적용한 결과 document의 size는 3.2kB, 응답에 걸린 시간은 32ms로 감소했다.
    - cdn을 사용했을 때 document의 크기는 약 74.7%, 응답 시간은 28.9% 감소했다.
    - 물리적으로 엔드유저와 거리가 가까워지는 것 뿐만 아니라, CloudFront 자체가 HTML 파일을 압축해서 전송하기 때문이다.



### cache control 적용하여 s3 배포(s3에 cache control 적용시 cdn에도 적용)
![image](https://github.com/user-attachments/assets/5f41d040-cdc5-48a0-a081-94b93e5f2248)
![image](https://github.com/user-attachments/assets/747bec05-41c1-4fb5-bdfe-c3087301015f)
- cdn만 적용하는 것 보다 Cache-Control를 적용하면 응답 속도가 매우 큰 폭으로 감소하는 것을 확인할 수 있다.
- size에 disk cache로 표기된 리소스는 응답 시간에 걸린 시간이 5ms 이하로 매우 빠른 속도로 리소스를 가져오는 것을 확인할 수 있다. 
- 이는 Cache-Control 헤더를 설정하면 브라우저가 해당 리소스를 로컬 HTTP 캐시에 저장하고, 유효 기간(또는 기타 정책) 안에서는 서버에 재요청 없이 바로 로드하도록 지시하기 때문이다.
- _next/하위의 정적 리소스들은 Cache-Control 헤더를 `public, max-age=31536000, immutable` 로 설정했기 때문에
    - public 브라우저뿐만 아니라 프록시 서버와 같은 중간 캐시에서도 해당 리소스를 저장할 수 있음. 
    - max-age=31536000 캐싱된 리소스가 1년 동안 유효함을 뜻함.
    - immutable 해당 리소스가 변경되지 않는 리소스라는 것을 브라우저에게 알려줌. 
- 다만 index.html 같은 경우 배포 이후 변경된 index.html이 새로 렌더되어야 하기 때문에 `max-age=0, must-revalidate`로 Cache-Control 헤더를 설정해주었다.
    - must-revalidate: 만료 후 반드시 서버에 재검증을 요구.
- 위에서도 다뤘지만, 한번 브라우저에 캐시가 저장되면 만료될 때까지 캐시는 계속 브라우저에 남아있다. 따라서 CDN Invalidation 작업을 진행하더라도 브라우저에 유효한 캐시를 지우기는 어렵다. 따라서 프로젝트에 알맞는 캐싱 전략을 세우고 적용하는 것이 중요하다.

<br/>



---
### 질문
- 리소스 별로 캐시 전략을 다르게 가져가야할 것 같은데 어떤 식으로 캐싱 전략을 세우고 적용하고 계신가요? (캐싱 전략을 잘못 세우고 적용하게 될 경우 엔드 유저에게 정확하지 못한 정보를 전달할 수도 있어 캐싱 전략을 어떻게 가져가야할지 고민입니다. )
- Next.js나 서버의 배포가 필요한 경우 리소스 산정을 어떻게 하고 배포하고 계신가요? (ex. 인스턴스 or pod을 몇대 띄울지, 이중화나 DR 측면에서는 어떻게 설계할지.)


