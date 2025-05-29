## 주요 링크

- S3 버킷 웹사이트 엔드포인트: [http://hanghae-front-5th-chapter4-1.s3-website.ap-northeast-2.amazonaws.com](http://hanghae-front-5th-chapter4-1.s3-website.ap-northeast-2.amazonaws.com/)
- CloudFrount 배포 도메인 이름: [https://d1iyh21jcqiile.cloudfront.net](https://d1iyh21jcqiile.cloudfront.net/)

## 주요 개념

- GitHub Actions과 CI/CD 도구
    - **GitHub Actions**는 GitHub에서 제공하는 자동화 도구로, 코드가 푸시되거나 PR이 머지될 때 특정 작업(워크플로우)을 자동으로 수행할 수 있게 합니다.
    - **CI/CD**는 Continuous Integration(지속적 통합)과 Continuous Deployment(지속적 배포)를 의미하며, 코드 변경이 생기면 테스트, 빌드, 배포까지 자동으로 이어지게 합니다.
    - 예를 들어 `.github/workflows/deployment.yml`에 배포 스크립트를 작성해두면, `main` 브랜치에 푸시할 때마다 자동으로 빌드 후 S3/CloudFront에 배포됩니다.
    - 수동 배포에서 발생하는 실수와 반복작업을 줄이고, 일관된 품질과 빠른 배포를 가능하게 합니다.
- S3와 스토리지
    - S3 (Simple Storage Service)는 AWS에서 제공하는 객체 스토리지 서비스로, 정적 웹사이트 파일(html, js, css, 이미지 등)을 저장하고 서빙하는 데 적합합니다.
    - Next.js 프로젝트를 `npm run build`로 정적 파일로 변환한 뒤, 해당 결과물(`out/`, `.next/`, `public/` 등)을 S3 버킷에 업로드하여 사용자에게 서비스합니다.
    - S3는 `정적 웹 호스팅` 기능을 제공하므로, 퍼블릭 액세스를 설정하면 웹사이트 도메인처럼 직접 접속할 수 있습니다.
    - S3는 비용이 저렴하고 가용성이 높아 정적 리소스 배포에 매우 유리합니다.
- CloudFront와 CDN
    - **CloudFront**는 AWS에서 제공하는 CDN(Content Delivery Network) 서비스입니다. 전 세계 엣지 로케이션에 콘텐츠를 캐싱해두고 사용자에게 더 가까운 서버에서 콘텐츠를 서빙함으로써 지연 시간(Latency)을 줄입니다.
    - S3와 CloudFront를 연결해 정적 리소스를 빠르게 전송할 수 있으며, SSL 인증서 및 사용자 정의 도메인을 연결할 수 있는 장점이 있습니다.
    - 특히 한국처럼 S3 버킷이 멀리 떨어져 있는 리전(예: 미국 us-east-1)에 있을 경우, CloudFront를 사용하지 않으면 웹사이트 응답 속도가 느릴 수 있습니다.
    - 글로벌 사용자 대상 서비스에서 필수적으로 사용되며, 성능 최적화 및 보안(HTTPS) 측면에서도 중요합니다.
- 캐시 무효화(Cache Invalidation)
    - CloudFront는 콘텐츠를 캐싱하므로, 정적 파일이 변경되더라도 바로 반영되지 않는 문제가 있습니다.
    - 이를 해결하기 위해 배포 후 다음 명령을 사용해 기존 캐시를 무효화합니다.
        
        ```jsx
        aws cloudfront create-invalidation --distribution-id <배포 ID> --paths "/*"
        ```
        
    - 이 과정은 GitHub Actions 워크플로우 내에서 자동화되고 있습니다.
    - 무분별한 캐시 무효화는 비용이 발생할 수 있어, 변경된 리소스만 선택적으로 무효화하는 전략도 중요합니다.
- Repository secret과 환경변수
    - AWS 자격 증명 등 민감한 정보는 코드에 직접 입력하지 않고, GitHub Repository의 **Secrets** 기능을 사용해 관리합니다.
 

## 성능 최적화

### **측정 환경 세팅**

- **테스트 대상**: AWS S3 정적 웹사이트
- **도구**: WebPageTest
- **환경**: Desktop / Chrome v136 / Cable 네트워크
- **측정 위치**: 한국 서울, 미국 버지니아
- **조건**: CloudFront 사용 여부에 따라 비교

## 1. 🇺🇸 미국 기준 성능 비교 (Dulles, Virginia / Desktop, Chrome v136, Cable)
### S3 직접 접속
<img width="702" alt="image" src="https://github.com/user-attachments/assets/b711e592-2939-490e-9203-289b1c239ef3" />

### CloudFront 접속
<img width="705" alt="image" src="https://github.com/user-attachments/assets/be8f2faa-16b9-4065-a162-3c9cd1479bee" />


| 지표 | S3 (직접 접속) | CloudFront | 개선 정도 |
| --- | --- | --- | --- |
| Time to First Byte | 1.939s | 0.330s | 🔽 1.609s (약 83% 감소) |
| Start Render | 2.500s | 1.100s | 🔽 1.400s |
| First Contentful Paint | 2.542s | 1.145s | 🔽 1.397s |
| Speed Index | 2.596s | 1.139s | 🔽 1.457s |
| Largest Contentful Paint | 2.740s | 1.145s | 🔽 1.595s |
| Cumulative Layout Shift | 0 | 0 | 동일 |
| Total Blocking Time | 0.000s | 0.000s | 동일 |
| Page Weight | 486 KB | 194 KB | 🔽 292 KB (약 60% 감소) |

📌 **요약**

- CloudFront 도입 시 모든 주요 지표가 눈에 띄게 개선되었습니다.
- 특히 **TTFB(Time to First Byte)**가 약 1.9초 → 0.33초로 83% 이상 개선되었습니다.
- FCP, LCP, Start Render 등 렌더링 관련 지표도 1.4초 이상 단축되었습니다.
- 페이지 용량도 60% 줄어 네트워크 및 로딩 부담이 줄었습니다.

---

## 2. 한국 기준 성능 비교 (Seoul, Korea / Desktop, Chrome v136, Cable)
### S3 직접 접속

<img width="704" alt="image" src="https://github.com/user-attachments/assets/bde5d11e-7e6d-4cc7-a946-512c6994802f" />

### CloudFront 접속

<img width="707" alt="image" src="https://github.com/user-attachments/assets/4b44c14c-e39b-4a91-ac4e-8d816e00d5c2" />


| 지표 | S3 (직접 접속) | CloudFront | 개선 정도 |
| --- | --- | --- | --- |
| Time to First Byte | 3.106s | 0.165s | 🔽 2.941s |
| Start Render | 3.400s | 0.500s | 🔽 2.900s |
| First Contentful Paint | 3.426s | 0.533s | 🔽 2.893s |
| Speed Index | 3.432s | 0.506s | 🔽 2.926s |
| Largest Contentful Paint | 3.426s | 0.533s | 🔽 2.893s |
| Cumulative Layout Shift | 0 | 0 | 동일 |
| Total Blocking Time | 0.000s | 0.000s | 동일 |
| Page Weight | 486 KB | 193 KB | 🔽 293 KB 감소 |

📌 **요약**

- CloudFront 사용 시, 렌더링 지연 시간이 약 **3초 이상** 감소되었습니다.
- **TTFB** 기준 3.1초 → 0.165초로 **18배 이상 개선**되었습니다.
- 페이지 용량도 약 293KB 감소. gzip 압축이나 정적 리소스 최적화 적용 가능성이 높습니다.

---

## 3.  국가별 성능 비교 (S3 직접 접근 vs CloudFront)

| 항목 | S3 (🇰🇷 한국) | S3 (🇺🇸 미국) | CloudFront (🇰🇷 한국) | CloudFront (🇺🇸 미국) |
| --- | --- | --- | --- | --- |
| Time to First Byte | **3.106s** | **1.804s** | **0.165s** | **0.194s** |
| Start Render | 3.400s | 2.015s | 0.500s | 0.500s |
| First Contentful Paint | 3.426s | 2.051s | 0.533s | 0.533s |
| Speed Index | 3.432s | 2.066s | 0.506s | 0.506s |
| Largest Contentful Paint | 3.426s | 2.051s | 0.533s | 0.533s |
| Page Weight | 486 KB | 486 KB | 193 KB | 193 KB |

---

## 4. 분석

1️⃣ **S3 직접 접근 시**
서울 리전에 있는 S3 버킷인데도 미국에서 더 빨랐습니다.
- TTFB 기준: 미국 1.8초, 한국 3.1초
- **원인 가능성**
    - `S3 static website endpoint`는 CloudFront처럼 지리적 최적화가 되지 않습니다.
    - S3 정적 웹사이트 접근 시 리전과 무관하게 느린 라우팅이 걸릴 수 있습니다.
    - 한국 내 DNS 라우팅 경로나 백엔드 네트워크 구조의 이슈 가능성이 있습니다.
- 결과적으로, **S3만 단독으로 사용 시 지역 간 성능 편차** 발생 가능합니다.

2️⃣ **cloudFront 사용 시**
서울, 미국 모두 거의 동일한 성능이었습니다.
- TTFB 기준: 한국 0.16초, 미국: 0.19초 (사실상 오차 수준)
- CloudFront가 각 지역의 **엣지 로케이션**을 활용해 정적 콘텐츠를 캐싱하기 때문입니다.
- 사용자와 가까운 서버에서 콘텐츠 제공 → TTFB 및 렌더링 지연 최소화됩니다.
- 한국에서 3초 가까이 걸리던 TTFB가 0.165초로 개선되었습니다.

✅ **결론**
- S3 버킷은 서울 리전에 있었음에도 불구하고, CloudFront 미사용 상태에서는 한국보다 미국에서 더 빠른 속도를 보였습니다.
- CloudFront 사용 시, 지역 간 차이가 거의 없었습니다.
- CloudFront가 위치와 무관하게 항상 S3보다 빠릅니다. (최대 3초 이상 차이)
