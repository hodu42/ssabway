# 🧭 SSabway

> 한국을 찾은 여행객이 화상으로 실시간 상담을 받는 서비스.
> 표지판 인식·개인정보 모자이크 AI, 경로 안내, 상담 녹취 요약을 제공합니다.

| 기간 | 팀 구성 | 담당 |
|---|---|---|
| 2026.07 ~ 2026.08 (6주) | 7인 | **CI/CD · 인프라** |

SSAFY 15기 2학기 공통 프로젝트로 진행했습니다.

---

## ✨ 주요 기능

<!-- TODO: 기능별 스크린샷 또는 GIF. 화상통화·모자이크는 GIF 권장 -->

| 기능 | 설명 |
|---|---|
| 실시간 화상 상담 | OpenVidu 기반 WebRTC 영상통화로 여행객과 상담원을 연결 |
| 표지판 인식 | 카메라에 잡힌 표지판을 AI가 인식해 안내 |
| 개인정보 모자이크 | 영상에 노출되는 개인정보를 실시간으로 가림 |
| 경로 안내 | 목적지까지의 이동 경로 제공 |
| 실시간 번역 자막 | 상담 중 음성을 인식해 번역 자막으로 표시 |
| 상담 녹취 요약 | 상담 종료 후 대화 내용을 자동 요약 |

---

## 🛠️ 기술 스택

| 구분 | 사용 기술 |
|---|---|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS 4, Zustand, TanStack Query, React Router 7 |
| 다국어 · PWA | i18next, react-i18next, vite-plugin-pwa |
| WebRTC | OpenVidu 2 (`openvidu-browser`, `openvidu-java-client`) |
| Backend | Spring Boot 4, Spring Security · OAuth2 Client, JPA, JWT(jjwt), MySQL, Redis, AWS S3 |
| AI | FastAPI, PyTorch, Ultralytics YOLOv8, OpenCV(YuNet), Azure Speech |
| 외부 API | ODsay(대중교통 경로), Azure Speech(음성 인식·번역), OpenAI 호환 LLM(상담 요약) |
| Infra | AWS EC2, Docker Compose, Jenkins, GitLab CI, Nginx |
| 개발 도구 | MSW, oxlint, Prettier |

---

## 🏗️ 시스템 구성

![시스템 아키텍처](docs/%EC%8B%9C%EC%8A%A4%ED%85%9C%20%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90.png)

애플리케이션을 역할에 따라 셋으로 분리했습니다.

| 애플리케이션 | 스택 | 역할 |
|---|---|---|
| 메인 API 서버 | Spring Boot | 사용자·상담 관리, 경로 안내(ODsay), 상담 요약(LLM) |
| 시그널링 서버 | Spring Boot + OpenVidu 2 | WebRTC 세션 관리, 녹음 파일 S3 업로드 |
| AI 서버 | FastAPI + PyTorch | 표지판 분류, 얼굴 검출(모자이크), 실시간 번역 자막 |

각 백엔드 내부는 모듈러 모놀리스 구조입니다.

배포 환경은 SSAFY에서 제공한 **EC2 1대(메모리 15GB)로 고정**되어 있었습니다.
서버를 늘릴 수 없는 조건에서 컨테이너 7개(nginx · frontend · api · signaling · ai · mysql · redis)와 Jenkins를 한 호스트에 올려야 했고, 아래 작업 대부분이 이 제약에서 출발했습니다.

---

## 🙋 담당 구현

전체 823 커밋 중 82건을 작성했습니다. 이 중 46건은 `develop` · `main` 통합 머지로, 인프라와 함께 릴리스 병합도 맡았습니다.

배포 자동화 경험이 없는 상태에서 인프라를 맡았습니다.
AWS 강의를 완강해 EC2 구축·Route53·ELB·HTTPS·RDS·S3·CloudFront를 실습하고, 개인 계정에서 수동 배포를 끝까지 해본 뒤 파이프라인 작성에 들어갔습니다.

### 배포 파이프라인 (Jenkins)

- 커밋에서 변경된 파일 경로를 확인해 **영향받는 서비스만 재배포**. 전체 재빌드는 단일 서버에 부담이 큽니다.
- `disableConcurrentBuilds`로 동시 빌드 차단, 서비스별 순차 빌드로 자원 사용 피크 분산
- 배포 후 컨테이너 상태와 외부 HTTPS 응답 코드를 함께 검증
- 실패 시 직전 5분 로그를 자동 수집하고 롤백 절차 안내, 성공 시 last-good 커밋 기록

### 병합 전 검증 파이프라인 (GitLab CI)

- `interruptible: true` — 새 커밋이 올라오면 이전 파이프라인 자동 취소
- **테스트 병합**을 수행해 문법이 아닌 논리 충돌을 병합 전에 감지
- 충돌 마커(`<<<<<<<`) 잔재 검사
- Gradle · npm 캐시로 실행 시간 단축
- 프론트엔드 3종 검증: `tsc --noEmit` → lint → build

### 협업 규칙 정립

gitmoji 커밋 컨벤션, Jira 이슈 키 기반 브랜치 네이밍(`S15P11D104-xxx`), MR·이슈 템플릿,
protected branch 전략(`develop` 기본 / `main` 배포 전용)을 정리해 팀에 적용했습니다.

### 도입 전 검증

OpenVidu는 팀에 제안하기 전에 데모 앱을 띄워 다른 네트워크의 휴대폰과 실제로 통화가 되는지 확인했습니다.
NAT 환경에서 연결되는지가 이 프로젝트의 전제 조건이었기 때문입니다.

---

## 🧯 트러블슈팅

### 1. 빌드가 시작되면 DB가 죽음

**증상** — 배포를 돌릴 때마다 MySQL 컨테이너가 내려갔습니다.

**원인** — 15GB 서버 한 대에 컨테이너 6개와 Jenkins가 함께 올라가 있었습니다. 빌드가 시작되면 메모리가 부족해지고, 가장 먼저 MySQL이 종료됐습니다.

**해결** — 파이프라인 진입 시점에 가용 메모리를 확인해 **2500MB 미만이면 빌드를 시작하지 않도록** 했습니다. 동시 빌드를 막고 서비스를 순차적으로 빌드해 피크도 낮췄습니다. 이후 배포로 인한 DB 종료는 재발하지 않았습니다.

### 2. 배포 직후 간헐적 502

**원인** — nginx가 재생성된 컨테이너의 이전 IP를 캐시한 상태로 요청을 보내고 있었습니다.

**해결** — nginx 설정을 볼륨 마운트로 분리해둔 구조라 이미지를 다시 만들 필요가 없었습니다. 재빌드 대신 **restart만 수행**하도록 바꿔 캐시를 초기화했습니다.

### 3. 서버에서 고친 설정이 배포로 사라짐

**원인** — 장애 대응 중 서버에서 직접 수정한 파일을, 이후 배포가 조용히 덮어썼습니다. 되돌아간 사실을 아무도 모르니 원인 파악이 어려워집니다.

**해결** — 배포 스크립트의 병합을 `git merge --ff-only`로 바꿨습니다. 히스토리가 어긋나면 배포가 멈추고 상황이 드러납니다.

### 4. 웹훅 누락으로 배포가 통째로 빠짐

**해결** — 웹훅은 한 번 유실되면 해당 배포가 그대로 사라집니다. `pollSCM`을 15분 주기로 걸어 fallback 경로를 뒀습니다.

---

## 🔭 아쉬웠던 점

- **테스트 코드 부재** — 프론트엔드에 테스트가 없어 `tsc --noEmit` · lint · build 세 단계가 유일한 안전망이었습니다. 타입과 빌드가 통과해도 동작이 맞는지는 보장하지 못합니다.
- **무중단 배포 불가** — 서버가 한 대로 고정돼 있어 배포 중에는 짧은 중단이 생깁니다. 인스턴스를 늘릴 수 없는 조건에서 선택할 수 있는 방법이 없었습니다.

---

## ⚙️ 실행 방법

```bash
git clone https://github.com/hodu42/ssabway.git
cd ssabway

cp .env.example .env   # 환경변수 설정

# 로컬 개발 (override 자동 적용)
docker compose up -d

# 운영 구성 (nginx · ai 포함)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
