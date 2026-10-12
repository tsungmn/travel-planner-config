# travel-planner-config

**여행길**(travel-planner) 서비스를 쿠버네티스 클러스터에 배포하는 설정 저장소입니다.

이 저장소에는 애플리케이션 코드가 없습니다. 어떤 이미지를, 어떤 설정으로, 어떤 순서로 띄울지만 담고 있습니다. ArgoCD가 이 저장소를 계속 지켜보다가 변경이 생기면 클러스터에 그대로 반영합니다. 그래서 이 저장소의 `main` 브랜치가 곧 운영 환경의 설계도입니다.

이 문서는 두 가지 목적으로 씁니다. 하나는 평소에 배포하고 운영하는 사람을 위한 안내이고, 다른 하나는 몇 년이 지나 서버가 사라졌거나 모든 내용을 잊은 뒤에도 이 문서만 보고 서비스를 처음부터 다시 세울 수 있게 하는 기록입니다. 그래서 명령어뿐 아니라 그렇게 하는 이유도 함께 적었습니다. 화면 메뉴 이름이나 도구 버전은 시간이 지나면 바뀌므로, 각 단계가 무엇을 위한 것인지 이해해 두면 환경이 바뀌어도 같은 일을 할 수 있습니다.

> **작성 기준일: 2026-10-12.**
> 버전, 메뉴 이름, 외부 서비스의 정책은 이 시점의 내용입니다. 오래된 문서를 열었다면 먼저 [13. 오래 지난 뒤 점검할 것](#13-오래-지난-뒤-점검할-것)을 읽어 주세요.

| 항목 | 내용 |
| --- | --- |
| 서비스 주소 | https://travel.tsungmn.com |
| 배포 방식 | GitOps. 이 저장소의 커밋이 곧 배포입니다. |
| 외부 접속 | Cloudflare Tunnel. Ingress, cert-manager, 공유기 포트포워딩을 쓰지 않습니다. |
| 이미지 저장소 | GHCR (`ghcr.io/tsungmn/...`) |
| 실행 환경 | 홈서버의 쿠버네티스 클러스터 |

## 상황별로 어디를 읽을까

| 상황 | 갈 곳 |
| --- | --- |
| 처음 보거나 오랜만에 열었다 | [1. 한눈에 보기](#1-한눈에-보기) → [4. 이 저장소 밖에 있는 것](#4-이-저장소-밖에-있는-것) |
| 서버를 새로 만들었거나 모든 것을 다시 세워야 한다 | [6. 처음부터 다시 만들기](#6-처음부터-다시-만들기) |
| 새 버전을 배포하고 싶다 | [7. 평소 운영](#7-평소-운영) |
| 배포했더니 문제가 생겼다 | [7의 롤백](#롤백), [14. 문제 해결](#14-문제-해결) |
| DB를 되살려야 한다 | [9. DB 백업과 복원](#9-db-백업과-복원) |
| 토큰이 만료됐거나 서버가 죽었다 | [10. 사고가 났을 때](#10-사고가-났을-때) |
| 몇 년이 지나 모든 버전이 낡았다 | [13. 오래 지난 뒤 점검할 것](#13-오래-지난-뒤-점검할-것) |

## 목차

1. [한눈에 보기](#1-한눈에-보기)
2. [이름과 계정](#2-이름과-계정)
3. [저장소 구조](#3-저장소-구조)
4. [이 저장소 밖에 있는 것](#4-이-저장소-밖에-있는-것)
5. [설정 값과 서로 맞아야 하는 값](#5-설정-값과-서로-맞아야-하는-값)
6. [처음부터 다시 만들기](#6-처음부터-다시-만들기)
7. [평소 운영](#7-평소-운영)
8. [앱 저장소가 지켜야 하는 약속](#8-앱-저장소가-지켜야-하는-약속)
9. [DB 백업과 복원](#9-db-백업과-복원)
10. [사고가 났을 때](#10-사고가-났을-때)
11. [설정 결정 이유](#11-설정-결정-이유)
12. [dev와 prod의 구분](#12-dev와-prod의-구분)
13. [오래 지난 뒤 점검할 것](#13-오래-지난-뒤-점검할-것)
14. [문제 해결](#14-문제-해결)
15. [서브도메인을 바꿔야 한다면](#15-서브도메인을-바꿔야-한다면)
16. [이 문서가 쓰인 시점의 상태](#16-이-문서가-쓰인-시점의-상태)

---

## 1. 한눈에 보기

### 구성 요소

| 구성 요소 | 하는 일 | 이미지 | 포트 | 정의된 곳 |
| --- | --- | --- | --- | --- |
| api | FastAPI 백엔드. 시작하기 전에 DB 마이그레이션을 실행합니다. | `ghcr.io/tsungmn/travel-planner-api` | 컨테이너 8000, Service 80 | `base/api` |
| web | Next.js 웹 서버 | `ghcr.io/tsungmn/travel-planner-web` | 컨테이너 3000, Service 80 | `base/web` |
| postgres | PostgreSQL. StatefulSet이고 데이터 디스크는 5Gi입니다. | `postgres:17` | 5432 | `db` |
| postgres-backup | 매일 04:00(KST)에 `pg_dump`를 실행하는 CronJob | `postgres:17` | 없음 | `db/backup.yaml` |
| cloudflared | Cloudflare로 나가는 터널을 만들고, 들어온 트래픽을 api와 web으로 전달합니다. | `cloudflare/cloudflared:2026.9.3` | 상태 확인용 2000 | `base/cloudflared` |
| ArgoCD | 이 저장소를 감시해서 클러스터에 반영합니다. | (클러스터에 따로 설치) | 없음 | `argocd` |

### 트래픽이 흐르는 길

```
브라우저 → Cloudflare → cloudflared(파드) ─┬─ /api/*  → Service api → API 파드 → Service postgres
                                         └─ 그 외   → Service web → Web 파드
                                                      └─ 서버 렌더링 때 API를 부를 땐 http://api (클러스터 내부)
```

한 도메인에서 경로로만 나누므로 브라우저 입장에서는 같은 출처입니다. 쿠키와 CORS 문제가 없고, Google 로그인 콜백도 이 도메인 기준으로 동작합니다. API 경로가 처음부터 `/api/...`라서 경로를 다시 쓰는 설정도 필요 없습니다.

web 코드에는 `/api/*` 요청을 백엔드로 넘기는 `rewrites` 설정이 있습니다. 개발할 때(`npm run dev`)는 이 설정이 `/api`를 로컬 백엔드로 보내 주고, prod에서는 Cloudflare가 `/api`를 먼저 api로 갈라 주므로 거의 거치지 않습니다. 한편 web 서버가 서버 렌더링 중에 로그인 상태를 확인하는 요청은 `rewrites`가 아니라 `API_ORIGIN`(클러스터 내부 주소 `http://api`)으로 직접 갑니다.

### 세 저장소의 역할

| 저장소 | 하는 일 |
| --- | --- |
| [`travel-planner-api`](https://github.com/tsungmn/travel-planner-api) | API 코드, Dockerfile, 이미지를 빌드하고 이 저장소의 태그를 갱신하는 워크플로(`.github/workflows/deploy.yml`) |
| [`travel-planner-web`](https://github.com/tsungmn/travel-planner-web) | 웹 코드, Dockerfile, 같은 역할의 워크플로 |
| [`travel-planner-config`](https://github.com/tsungmn/travel-planner-config) (이 저장소) | 어떤 이미지를 어떤 설정으로 띄울지 |

### 배포가 일어나는 과정

```
앱 저장소(api 또는 web)의 main 에 push
   │
   ▼
GitHub Actions 가 이미지를 빌드해서 ghcr.io/tsungmn/travel-planner-<api|web>:<커밋 SHA 40자> 로 push
   │
   ▼
GitHub Actions 가 이 저장소 overlays/prod/kustomization.yaml 의 newTag 를 바꾸는 커밋을 push
   │
   ▼
ArgoCD 가 변경을 감지(몇 분 주기)해서 클러스터에 반영, 새 파드가 뜸
```

> **주의: 앱 저장소의 `main`에 push하는 것은 곧 운영 배포입니다.** 서버가 켜져 있으면 ArgoCD가 곧바로 새 이미지로 교체합니다. 서버를 확인할 수 없는 상황이라면 `main`에 병합하지 말고 브랜치에서 작업하세요. 브랜치 push는 이미지를 시험 빌드만 하고 배포하지 않습니다.

---

## 2. 이름과 계정

### 이름

이 프로젝트에는 이름이 여러 개 나오는데, 쓰이는 층이 서로 다릅니다.

| 층 | 이름 | 쓰이는 곳 |
| --- | --- | --- |
| 브랜드 (사용자에게 보임) | 여행길 | 화면 문구, 브라우저 탭 제목, Google 동의 화면, 개인정보처리방침 |
| 코드네임, 서브도메인 | `travel-planner`, `travel` | 저장소, 컨테이너 이미지, namespace, `travel.tsungmn.com` |
| 데이터 용어 | `trip` (여행 한 건) | `/trips`, `/api/trips`, `trips` 테이블 |

`travel-planner`는 제품 이름이고 `trip`은 그 안의 데이터 한 건을 부르는 말이라 섞여 있어도 문제가 아닙니다. 이 저장소에서는 코드네임 `travel-planner`로 통일합니다.

### 계정

| 서비스 | 계정 | 용도 |
| --- | --- | --- |
| GitHub **조직** | `tsungmn` | 저장소 세 개의 소유자이자 GHCR 이미지 경로의 소유자입니다. 이미지 경로와 저장소 주소에는 모두 이 이름이 들어갑니다. |
| GitHub **개인 계정** | `tsungmnli` | 조직의 멤버입니다. `ghcr-pull`용 토큰을 발급하는 계정이고, 클러스터 Secret의 `--docker-username`에 들어갑니다. |
| Cloudflare | (본인 계정) | `tsungmn.com`의 DNS와 Zero Trust 터널 |
| Google Cloud | (본인 계정) | prod 전용 OAuth 클라이언트 |
| Geoapify | (본인 계정) | prod 전용 API 키 |

`ghcr.io`는 GitHub의 레지스트리라서 이미지 경로에는 조직 이름(`tsungmn`)이 들어가고, 이미지를 받을 때 로그인하는 사용자 이름에는 **토큰을 발급한 개인 계정**이 들어갑니다. 둘이 다르다는 점이 가장 헷갈리는 부분입니다.

---

## 3. 저장소 구조

```
travel-planner-config/
├─ argocd/                         # ArgoCD Application (처음 한 번만 kubectl apply)
│  ├─ travel-planner-prod.yaml     # api, web, cloudflared 를 배포
│  └─ travel-planner-db.yaml       # DB 를 따로 배포
├─ base/
│  ├─ api/          deployment.yaml  service.yaml  kustomization.yaml
│  ├─ web/          deployment.yaml  service.yaml  kustomization.yaml
│  └─ cloudflared/  deployment.yaml  kustomization.yaml
├─ db/                             # API 배포가 DB에 영향을 주지 않도록 분리
│  └─ statefulset.yaml  service.yaml  backup.yaml  kustomization.yaml
└─ overlays/
   └─ prod/         kustomization.yaml   # 이미지 태그, 환경 변수 등 prod 값 전부
```

| 위치 | 역할 |
| --- | --- |
| `base/api` | API 서버. 시작하기 전에 `initContainer`(`migrate`)로 `python -m app.migrate`를 실행해 DB 스키마를 맞춥니다. |
| `base/web` | Next.js 웹 서버 |
| `base/cloudflared` | Cloudflare 터널 연결 |
| `db` | PostgreSQL StatefulSet, 백업 CronJob과 백업용 디스크 |
| `overlays/prod` | namespace, 이미지 태그, 환경 변수처럼 prod에서만 달라지는 값 |
| `argocd` | "이 경로를 감시해서 배포하라"고 ArgoCD에 알려 주는 Application |

`base`에는 환경에 따라 달라지는 값이 들어 있지 않습니다. 달라지는 값은 모두 `overlays/prod`에 모았습니다. dev 환경은 클러스터에 올리지 않기 때문에 overlay는 `prod` 하나뿐입니다. staging이 필요해지면 `overlays/staging`을 추가하면 됩니다.

---

## 4. 이 저장소 밖에 있는 것

**Git에 없어서, 서버가 사라지면 이 저장소만으로는 되살릴 수 없는 것들입니다.** 아래 목록은 비밀번호 관리자 같은 곳에 따로 보관해 두는 것을 권장합니다.

### 비밀과 계정

| 항목 | 어디에 있나 | 잃어버리면 |
| --- | --- | --- |
| `postgres-secret` | 클러스터의 Secret | **DB 비밀번호가 사라집니다.** 디스크에 남은 DB는 처음 만들 때의 비밀번호를 기억하므로, 다른 값으로 Secret을 새로 만들면 API가 접속하지 못합니다. 같은 값으로 다시 만들거나, [10장](#10-사고가-났을-때)의 방법으로 DB 비밀번호를 바꿉니다. |
| `api-secret` | 클러스터의 Secret | 새로 만들면 됩니다. `SESSION_SECRET`이 바뀌면 모든 사용자가 로그아웃됩니다. |
| `cloudflared-secret` | 클러스터의 Secret | Cloudflare 대시보드에서 토큰을 다시 받으면 됩니다. |
| `ghcr-pull` | 클러스터의 Secret | `read:packages` 권한의 토큰을 새로 만들면 됩니다. |
| Google OAuth prod 클라이언트 | Google Cloud 콘솔 | 클라이언트 ID와 시크릿은 콘솔에서 다시 볼 수 있습니다. |
| Geoapify prod 키 | Geoapify 대시보드 | 새로 발급합니다. |
| Cloudflare Tunnel과 Public Hostname | Cloudflare 대시보드 | [6-4](#6-4-cloudflare-tunnel-만들기)를 다시 하면 됩니다. |
| DNS와 도메인 `tsungmn.com` | Cloudflare, 도메인 등록 업체 | 도메인 갱신이 끊기면 서비스 전체가 멈춥니다. 만료일을 확인하세요. |
| DB 데이터 | 클러스터의 디스크 (`data-postgres-0`) | 백업 디스크에 14일치 덤프가 있습니다. 같은 서버에 있으므로 [다른 곳에도 복사](#9-db-백업과-복원)해야 합니다. |

### GitHub 쪽 설정

| 항목 | 어디에 있나 | 잃어버리면 |
| --- | --- | --- |
| 이미지 빌드 워크플로 | 각 앱 저장소의 `.github/workflows/deploy.yml` | 이 저장소에는 없습니다. 앱 저장소 코드와 함께 보관됩니다. |
| `CONFIG_REPO_TOKEN` | 조직 또는 앱 저장소의 Actions 시크릿 | 만료되면 이미지는 빌드되지만 이 저장소의 태그 갱신이 실패합니다. [10장](#10-사고가-났을-때)을 보세요. |
| 패키지 공개 여부와 접근 권한 | 조직 → Packages | 비공개면 `ghcr-pull`이 필요하고, 토큰 계정에 읽기 권한이 있어야 합니다. |
| 조직의 토큰 정책과 SSO | 조직 Settings | fine-grained 토큰 승인 요구나 SAML SSO가 켜져 있으면 토큰을 따로 승인해야 합니다. |

### 서버 기록

서버는 이 저장소 밖에 있는 가장 큰 것입니다. 나중에 새 서버로 옮길 때를 위해 아래 표를 채워 두세요.

| 항목 | 기록 |
| --- | --- |
| 하드웨어와 CPU 구조 (`uname -m`) | (여기에 적어 두세요) |
| OS와 버전 | (여기에 적어 두세요) |
| 쿠버네티스 배포판과 버전 | (여기에 적어 두세요) |
| ArgoCD 설치 방법과 버전 | (여기에 적어 두세요) |
| 접속 방법 (SSH, Tailscale 등) | (여기에 적어 두세요) |
| DB 덤프를 따로 보관하는 곳 | (여기에 적어 두세요) |

---

## 5. 설정 값과 서로 맞아야 하는 값

### 이 저장소에서 확인해야 하는 값

| 파일 | 값 | 설명 |
| --- | --- | --- |
| `overlays/prod/kustomization.yaml` | api, web `newTag` | 앱 저장소의 커밋 SHA 40자입니다. 평소에는 CI가 바꿉니다. |
| `overlays/prod/kustomization.yaml` | `namespace: travel-planner` | 바꾸려면 `argocd/*.yaml`의 `destination.namespace`와 `db/kustomization.yaml`의 `namespace`도 함께 바꿔야 합니다. |
| `overlays/prod/kustomization.yaml` | `PUBLIC_URL=https://travel.tsungmn.com` | [15장](#15-서브도메인을-바꿔야-한다면)의 세 곳을 같이 바꿉니다. |
| `overlays/prod/kustomization.yaml` | `APP_ENV=prod`, `COOKIE_SECURE=true`, `DOCS_ENABLED=false` | 코드가 prod 값을 검증하므로 어긋나면 API가 시작되지 않습니다. |
| `overlays/prod/kustomization.yaml` | `NOMINATIM_USER_AGENT` | Nominatim 사용 정책상 연락처가 들어가 있어야 합니다. 저장소를 공개한다면 공개해도 되는 연락처인지 확인하세요. |
| `overlays/prod/kustomization.yaml` | `API_ORIGIN=http://api` | web 서버가 서버 렌더링 때 API를 부르는 클러스터 내부 주소입니다. Service 이름 `api`와 맞아야 합니다. |
| `base/cloudflared/deployment.yaml` | `cloudflare/cloudflared:2026.9.3` | `latest`를 쓰지 않고 버전을 고정합니다. 올리기 전에 이미지가 실제로 있는지 확인하세요. |
| `db/statefulset.yaml` | `postgres:17`, 데이터 디스크 5Gi | 버전을 올리는 방법은 [13장](#13-오래-지난-뒤-점검할-것)을 보세요. |
| `db/backup.yaml` | 매일 `0 4 * * *` (KST), 14일 보관, 백업 디스크 5Gi | 보관 일수는 `find`의 `-mtime +14`입니다. |
| `base/api/deployment.yaml` | `replicas: 1`, `strategy: Recreate`, `runAsUser: 10001` | 이유는 [11장](#11-설정-결정-이유)을 보세요. |
| `argocd/*.yaml` | `repoURL: https://github.com/tsungmn/travel-planner-config.git`, `targetRevision: main` | 저장소를 옮기거나 기본 브랜치 이름을 바꾸면 고쳐야 합니다. |

### 서로 맞아야 하는 값

한 곳만 바꾸면 조용히 깨지는 값들입니다. 하나를 바꿀 때는 같은 줄의 다른 곳도 함께 확인하세요.

| 값 | 같아야 하는 곳 |
| --- | --- |
| 이미지 경로 `ghcr.io/tsungmn/travel-planner-<api\|web>` | `overlays/prod/kustomization.yaml`, `base/api/deployment.yaml`(두 곳), `base/web/deployment.yaml`, 각 앱 저장소 워크플로가 만드는 이미지 이름 |
| kustomization의 이미지 항목 모양 | 앱 저장소 워크플로의 정규식 ([8장](#8-앱-저장소가-지켜야-하는-약속)) |
| UID `10001` | API Dockerfile의 `appuser`와 `base/api/deployment.yaml`의 `runAsUser` |
| Service 이름 `api` | `overlays/prod/kustomization.yaml`의 `API_ORIGIN=http://api`, web 이미지를 빌드할 때의 `API_ORIGIN` 빌드 인자, Cloudflare Public Hostname의 서비스 주소 |
| 포트 | API 컨테이너 8000, web 컨테이너 3000. Service는 둘 다 80으로 받습니다. Deployment의 `containerPort`, Service의 `targetPort`, Cloudflare의 서비스 주소가 맞아야 합니다. |
| 사이트 주소 `https://travel.tsungmn.com` | `PUBLIC_URL`, Cloudflare Public Hostname, Google OAuth redirect URI, web 워크플로의 `SITE_URL` |
| namespace `travel-planner` | `overlays/prod`, `db/kustomization.yaml`, `argocd/*.yaml`, Secret을 만든 namespace, Cloudflare 서비스 주소의 `.travel-planner.svc` |
| `postgres-secret`의 비밀번호 | `api-secret`의 `DATABASE_URL`에 들어 있는 비밀번호 |

이미지 경로가 한 번에 맞는지는 이렇게 확인합니다.

```bash
grep -rn "ghcr.io/" base overlays
```

---

## 6. 처음부터 다시 만들기

서버를 새로 만들었거나, 이 서비스를 처음 올리는 경우의 전체 순서입니다.

**이미지가 아직 없는 상태에서 ArgoCD Application부터 등록하면 `ImagePullBackOff`가 납니다.** 그래서 순서는 항상 이렇습니다. 이미지를 먼저 만들고, 그 태그가 이 저장소에 적히게 한 다음, 마지막에 Application을 등록합니다.

### 6-0. 준비물

| 준비물 | 용도 |
| --- | --- |
| 쿠버네티스 클러스터와 `kubectl` 접근 | 배포 대상 |
| GitHub 조직 `tsungmn`과 저장소 세 개 | 코드, 이미지 빌드, 이 설정 |
| Cloudflare 계정과 `tsungmn.com` DNS | 터널과 도메인 |
| Google Cloud 콘솔 접근 | OAuth 클라이언트 |
| Geoapify 계정 | API 키 |
| 비밀번호 관리자 | 아래에서 만드는 비밀을 보관 |
| (복원이라면) DB 덤프 파일 | [9장](#9-db-백업과-복원) |

### 6-1. 클러스터 준비

- Kubernetes 1.27 이상 (백업 CronJob의 `timeZone` 필드 때문입니다)
- ArgoCD
- 기본 StorageClass (k3s라면 `local-path`)

Ingress 컨트롤러와 cert-manager는 **필요 없습니다.** 외부 접속은 Cloudflare Tunnel이 맡습니다.

서버의 CPU 구조(`uname -m`)가 `x86_64`인지 `aarch64`인지도 기록해 둡니다. 앱 이미지가 그 구조용으로 빌드되어야 하기 때문입니다.

> ingress-nginx(쿠버네티스 커뮤니티 버전)는 2026년 3월에 종료되어 더 이상 보안 패치가 없습니다. 이 서비스는 Ingress를 거치지 않으므로 영향이 없습니다. 클러스터의 다른 서비스가 쓰고 있다면 그쪽만 교체 계획이 필요합니다.
>
> ```bash
> kubectl get deploy -A -o wide | grep -i nginx
> # registry.k8s.io/ingress-nginx/controller → 종료된 커뮤니티 버전
> # nginx/nginx-ingress                      → F5 버전 (계속 지원)
> ```

### 6-2. GitHub 준비

**조직과 저장소.** 조직 `tsungmn` 아래에 `travel-planner-api`, `travel-planner-web`, `travel-planner-config`가 있어야 합니다. 이름을 바꾸면 [5장](#5-설정-값과-서로-맞아야-하는-값)의 값을 모두 고쳐야 합니다.

**CI가 이 저장소에 쓸 수 있는 토큰 (`CONFIG_REPO_TOKEN`).** 앱 저장소의 워크플로가 이 저장소의 태그를 바꾸는 커밋을 올릴 때 씁니다.

1. 본인 계정의 개인 설정에서 Developer settings → Personal access tokens → **Fine-grained tokens**를 엽니다. (조직 설정이 아니라 개인 설정입니다.)
2. **Resource owner**를 본인 계정이 아니라 **조직 `tsungmn`**으로 고릅니다.
3. **Repository access**는 `travel-planner-config` 하나만 선택하고, 권한은 **Contents: Read and write**로 합니다.
4. 만든 토큰을 `CONFIG_REPO_TOKEN`이라는 이름의 Actions 시크릿으로 등록합니다. 조직 시크릿으로 만들고 접근 가능한 저장소를 `travel-planner-api`, `travel-planner-web` 두 개로 제한하는 방식을 권장합니다. 토큰을 갱신할 때 한 곳만 고치면 되기 때문입니다.

막히는 경우는 이렇습니다.

- Resource owner에 조직이 보이지 않으면 조직이 fine-grained 토큰을 허용하지 않은 상태입니다. 조직 Owner가 조직 설정의 Personal access tokens 메뉴에서 허용해야 합니다.
- 토큰이 Pending 상태면 조직이 승인을 요구하는 설정입니다. 같은 메뉴의 Pending requests에서 Owner가 승인합니다.
- 이 토큰은 **발급한 개인 계정에 묶입니다.** 그 계정이 조직에서 빠지면 배포가 멈추고, 만료일이 지나도 멈춥니다. 오래 운영하려면 개인 계정에 묶이지 않는 **GitHub App**(설치 토큰을 워크플로에서 발급)으로 바꾸는 것도 방법입니다.

**이미지를 받는 쪽 (`ghcr-pull`).** 이미지 패키지가 비공개이면 클러스터가 받을 수 있도록 토큰이 필요합니다.

- 방법 A. 패키지를 **public**으로 바꿉니다. 그러면 `ghcr-pull` Secret과 두 Deployment의 `imagePullSecrets`가 모두 필요 없습니다. 조직 설정에서 public 패키지가 허용되어 있어야 하고, 이미지에 비밀이 들어 있지 않아야 합니다.
- 방법 B. 비공개를 유지하고 **classic 개인 액세스 토큰**(`read:packages`)을 씁니다. 이 문서를 쓴 시점에는 GHCR이 fine-grained 토큰을 지원하지 않는 것으로 알려져 있으니, 바뀌었는지 GitHub 문서를 확인하세요. 조직이 SAML SSO를 쓰면 토큰 옆의 **Configure SSO**에서 조직을 승인해야 합니다. 이 토큰을 발급한 계정이 두 패키지를 읽을 권한을 가져야 하고, 그 계정 이름이 Secret의 `--docker-username`이 됩니다.

방법 B의 토큰도 발급한 사람에게 묶이므로 만료일을 기록해 두세요.

### 6-3. 앱 이미지 만들기와 첫 태그

각 앱 저장소의 `main`에 push하면 워크플로가 이미지를 빌드해 GHCR에 올리고, 이 저장소의 태그를 갱신합니다. 동작 원리와 필수 조건은 [8장](#8-앱-저장소가-지켜야-하는-약속)에 있습니다.

1. `overlays/prod/kustomization.yaml`의 `images:` 항목에 api와 web 이름이 이미 있는지 확인합니다. 태그 값은 아무 문자열이어도 됩니다. 워크플로가 항목을 찾아 값을 바꿉니다. (항목 자체가 없으면 워크플로가 "이미지 항목을 찾지 못했습니다"라며 실패합니다.)
2. 두 앱 저장소에서 워크플로를 실행합니다. 각 저장소의 Actions 탭에서 빌드가 성공했는지 확인합니다.
3. 워크플로의 deploy 잡이 성공하면 이 저장소의 태그가 자동으로 40자 SHA로 바뀝니다. `CONFIG_REPO_TOKEN`을 아직 못 만들었다면 GitHub의 패키지 페이지에서 실제로 올라간 태그를 복사해 직접 붙여 넣으세요.

   ```yaml
   images:
     - name: ghcr.io/tsungmn/travel-planner-api
       newTag: "<API 이미지 태그>"
     - name: ghcr.io/tsungmn/travel-planner-web
       newTag: "<web 이미지 태그>"
   ```

4. 렌더링이 되는지 로컬에서 확인한 뒤 커밋합니다.

   ```bash
   kubectl kustomize overlays/prod | grep -E "image:|<"
   # image 줄이 모두 실제 경로와 태그여야 하고, "<" 가 들어간 줄이 없어야 합니다.
   ```

### 6-4. Cloudflare Tunnel 만들기

`tsungmn.com`의 DNS가 Cloudflare에 있어야 합니다. (메뉴 이름은 시간이 지나면 바뀔 수 있으니 대시보드에서 비슷한 이름을 찾으세요.)

1. Cloudflare 대시보드의 **Zero Trust → Networks → Tunnels → Create a tunnel**에서 **Cloudflared**를 고릅니다.
2. 화면에 나오는 **토큰**을 복사해 둡니다. [6-5](#6-5-namespace와-secret-만들기)에서 씁니다.
3. 같은 터널에 **Public Hostname**을 두 개 추가합니다. **순서가 중요합니다.** `/api` 규칙이 위에 있어야 합니다.

| 순서 | 호스트 이름 | 경로 | 서비스 |
| --- | --- | --- | --- |
| 1 | `travel.tsungmn.com` | `^/api/` | `http://api.travel-planner.svc.cluster.local:80` |
| 2 | `travel.tsungmn.com` | (비움) | `http://web.travel-planner.svc.cluster.local:80` |

호스트 이름을 추가하면 `travel` CNAME 레코드가 DNS에 자동으로 만들어집니다. 이미 `travel` 레코드가 있으면 생성이 실패하므로 DNS 화면에서 먼저 확인하세요.

경로 입력 형식(정규식인지 `/api/*` 같은 패턴인지)은 Cloudflare 화면에서 확인해야 합니다. [6-9](#6-9-확인)의 `curl` 확인이 통과하지 않으면 형식을 바꿔서 다시 시도하세요.

### 6-5. namespace와 Secret 만들기

Secret은 **Git에 올리지 않고** 클러스터에 직접 만듭니다. 사용한 값, 특히 DB 비밀번호는 비밀번호 관리자에 보관하세요.

```bash
# namespace
kubectl create namespace travel-planner

# DB 계정
kubectl -n travel-planner create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=app \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -hex 24)" \
  --from-literal=POSTGRES_DB=travel

# API 설정 (DB 비밀번호는 위에서 만든 값을 그대로 가져옵니다)
PGPW=$(kubectl -n travel-planner get secret postgres-secret \
  -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d)

kubectl -n travel-planner create secret generic api-secret \
  --from-literal=DATABASE_URL="postgres://app:${PGPW}@postgres:5432/travel" \
  --from-literal=SESSION_SECRET="$(openssl rand -base64 32)" \
  --from-literal=GOOGLE_CLIENT_ID='<prod 클라이언트 ID>' \
  --from-literal=GOOGLE_CLIENT_SECRET='<prod 시크릿>' \
  --from-literal=GEOAPIFY_API_KEY='<prod 키>'

# Cloudflare Tunnel 토큰 ([6-4]에서 복사한 값)
kubectl -n travel-planner create secret generic cloudflared-secret \
  --from-literal=TUNNEL_TOKEN='<토큰>'

# 이미지가 비공개(GHCR)라면
kubectl -n travel-planner create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username='<토큰을 발급한 개인 GitHub 계정>' \
  --docker-password='<read:packages 권한의 classic 토큰>'
```

- `--docker-username`은 조직 이름(`tsungmn`)이 아니라 **토큰을 발급한 개인 계정**입니다.
- 비밀 값이 셸 히스토리에 남지 않게 주의하세요. 명령 앞에 공백을 두고 `HISTCONTROL=ignorespace`를 설정하는 방법이 있습니다.
- `ghcr-pull`을 바꿀 때는 이렇게 덮어씁니다. 이미 떠 있는 파드에는 영향이 없고, 새로 이미지를 받을 때부터 적용됩니다.

  ```bash
  kubectl -n travel-planner create secret docker-registry ghcr-pull \
    --docker-server=ghcr.io --docker-username='<계정>' --docker-password='<토큰>' \
    --dry-run=client -o yaml | kubectl apply -f -
  ```

### 6-6. Google OAuth와 Geoapify

Google Cloud 콘솔에서 **prod 전용 OAuth 클라이언트**를 만들고, 승인된 리디렉션 URI에 아래 주소를 등록합니다. 동의 화면의 앱 이름은 **여행길**로 합니다.

```
https://travel.tsungmn.com/api/auth/google/callback
```

dev와 prod의 Google 키, Geoapify 키는 각각 따로 발급합니다. 한쪽이 유출되어도 다른 쪽은 안전합니다. 만든 값은 [6-5](#6-5-namespace와-secret-만들기)의 `api-secret`에 넣습니다.

### 6-7. ArgoCD가 저장소를 읽을 수 있게 하기

이 저장소가 비공개라면 ArgoCD에 접근 권한(SSH deploy key 등)을 등록해야 합니다. 공개 저장소라면 건너뜁니다.

ArgoCD UI는 외부에 열지 않고 `port-forward`로만 접속합니다.

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# 브라우저에서 https://localhost:8080
```

### 6-8. Application 등록

```bash
kubectl apply -f argocd/travel-planner-db.yaml     # DB 먼저
kubectl apply -f argocd/travel-planner-prod.yaml
```

처음 한 번만 합니다. 이후에는 Git 커밋만으로 배포됩니다.

| Application | 감시 경로 | 특징 |
| --- | --- | --- |
| `travel-planner-db` | `db` | `prune: false`. DB 리소스는 자동으로 삭제되지 않습니다. |
| `travel-planner-prod` | `overlays/prod` | `prune: true`, `selfHeal: true`. 클러스터에서 손으로 바꿔도 Git 상태로 되돌립니다. |

### 6-9. 확인

```bash
kubectl -n travel-planner get pods
kubectl -n travel-planner logs deploy/api -c migrate   # 마이그레이션 로그

curl -i https://travel.tsungmn.com/api/healthz   # {"ok":true} 가 나오면 API로 간 것
curl -I https://travel.tsungmn.com/              # Next 페이지가 나와야 함
```

`postgres-0`, `api`, `web`, `cloudflared` 파드가 모두 `Running`이어야 합니다.

API 파드가 처음에 `Init:Error`를 몇 번 보이다가 자리를 잡는 것은 정상일 수 있습니다. 마이그레이션 컨테이너는 DB에 연결하지 못하면 재시도 없이 종료하고, 쿠버네티스가 알아서 다시 시작합니다. `postgres-0`이 `Running`이 되면 곧 해결됩니다. 계속 실패하면 위의 `migrate` 로그를 보세요.

브라우저에서도 확인합니다. 로그인(Google), 항공편 입력의 공항 검색, 지도 표시, 공유 링크가 모두 동작해야 합니다.

### 6-10. 기존 데이터를 복원해야 한다면

서비스가 이미 있었고 덤프 파일이 있다면 [9장의 복원](#복원)을 따릅니다. 새 DB가 비어 있는 상태에서 API가 한 번 뜬 뒤에 복원하는 것이 순서상 안전합니다.

---

## 7. 평소 운영

### 배포하기

배포는 **이 저장소의 이미지 태그를 바꾸는 커밋**입니다. 보통은 앱 저장소의 `main`에 push하면 CI가 알아서 해 주므로 직접 할 일이 없습니다. 빨리 반영하고 싶으면 ArgoCD UI에서 `travel-planner-prod`의 **Refresh → Sync**를 누르세요.

`latest` 태그나 `imagePullPolicy: Always`는 쓰지 않습니다. 이미지가 바뀌어도 Git에 기록이 남지 않고 롤백도 할 수 없기 때문입니다.

### CI 없이 손으로 배포하기

CI가 고장 났거나 토큰이 만료됐을 때의 방법입니다.

1. 앱 저장소에서 이미지를 빌드해 GHCR에 올립니다. (워크플로를 수동으로 다시 실행하는 것이 가장 쉽습니다.)
2. `overlays/prod/kustomization.yaml`의 해당 `newTag`를 새 태그로 바꿉니다.
3. 커밋하고 push합니다.

```bash
git commit -am "deploy: api <태그 앞 7자>" && git push
```

### 설정 값 바꾸기

`overlays/prod/kustomization.yaml`의 `configMapGenerator` 값을 고치고 커밋합니다. 값이 바뀌면 ConfigMap 이름의 해시가 바뀌어 파드가 자동으로 재시작됩니다.

Secret 값(`api-secret` 등)은 Git에 없으므로 `kubectl`로 바꿔야 하고, 바꾼 뒤에는 파드를 직접 재시작해야 합니다.

```bash
kubectl -n travel-planner rollout restart deploy/api
```

### 롤백

태그를 바꾼 커밋을 되돌립니다.

```bash
git log --oneline -- overlays/prod/kustomization.yaml   # 되돌릴 커밋 찾기
git revert <커밋>
git push
```

`kubectl edit`이나 `kubectl scale`로 직접 고치면 `selfHeal` 때문에 곧바로 Git 상태로 되돌아갑니다. 급하게 직접 손대야 한다면 ArgoCD UI에서 `travel-planner-prod`의 Auto-Sync를 먼저 끄세요. 작업이 끝나면 다시 켭니다.

주의할 점이 하나 있습니다. DB 마이그레이션은 API가 시작할 때 자동으로 실행되고 **되돌려지지 않습니다.** 이전 버전의 API가 새 스키마에서 동작하는지는 마이그레이션 내용에 달려 있으니, 스키마를 바꾼 배포를 롤백할 때는 마이그레이션 코드를 먼저 확인하세요.

### 자주 쓰는 명령

```bash
kubectl -n travel-planner get pods                                  # 전체 상태
kubectl -n travel-planner logs deploy/api --tail=100                # API 로그
kubectl -n travel-planner logs deploy/web --tail=100                # web 로그
kubectl -n travel-planner logs deploy/api -c migrate                # 마이그레이션 로그
kubectl -n travel-planner rollout status deploy/web                 # 배포 진행 확인
kubectl -n argocd get app                                           # ArgoCD 상태 (Synced, Healthy)
kubectl -n travel-planner get deploy api web \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}'   # 떠 있는 이미지 태그
kubectl kustomize overlays/prod                                     # 렌더링 결과 확인
```

---

## 8. 앱 저장소가 지켜야 하는 약속

이 저장소와 앱 저장소는 서로 몇 가지를 약속으로 주고받습니다. 한쪽을 바꾸면 반대쪽도 맞춰야 합니다.

### 이 저장소가 지켜야 할 모양

앱 저장소의 워크플로는 `overlays/prod/kustomization.yaml`에서 아래 모양의 항목을 **정규식으로** 찾아 `newTag` 값만 바꿉니다.

```yaml
images:
  - name: ghcr.io/tsungmn/travel-planner-api
    newTag: "<40자 커밋 SHA>"
```

- `- name: <이미지 경로>` 바로 다음 줄이 `newTag:`여야 합니다. 항목이 정확히 하나여야 하고, 없거나 둘이면 워크플로가 실패합니다.
- 이 모양을 바꾸거나(예: `digest` 사용, 줄 순서 변경) 이미지 경로를 바꾸면 워크플로의 정규식도 같이 고쳐야 합니다.

### 워크플로가 하는 일 (api와 web 공통)

1. 커밋 SHA(40자)를 태그로 이미지를 빌드해 GHCR에 올립니다. 이미지 이름은 조직 이름을 소문자로 바꿔서 `ghcr.io/<조직>/travel-planner-<api|web>`로 만듭니다.
2. 이 저장소를 `CONFIG_REPO_TOKEN`으로 받아 `newTag`를 바꾸고 커밋·push합니다. api와 web이 동시에 push하면 한쪽이 거절될 수 있어서 `git pull --rebase` 후 push합니다. web 워크플로는 여기에 더해 최대 5번까지 재시도하고, api 워크플로에는 아직 재시도가 없으니 같은 방식으로 맞춰 두는 것을 권장합니다.
3. 변경이 없으면 커밋하지 않습니다.

### api 저장소의 약속

- Dockerfile은 Python 3.12 기반이고, 컨테이너는 UID `10001`(`appuser`)로 8000번 포트에서 실행됩니다.
- `python -m app.migrate`가 작업 디렉터리 `/srv`에서 `db/migrations`를 찾아 스키마를 맞춥니다. 이 명령은 `initContainer`로 실행됩니다.
- `/api/healthz`(프로세스만)와 `/api/readyz`(DB까지)가 `{"ok": true}`를 돌려줍니다.
- `APP_ENV=prod`일 때 코드가 `PUBLIC_URL`, `COOKIE_SECURE=true`, `DOCS_ENABLED=false`, DB 호스트(로컬 호스트 목록에 없을 것)를 검증합니다.
- Dockerfile의 `--forwarded-allow-ips "*"`는 프록시 헤더를 신뢰하는 설정입니다. 클러스터 안에서는 같은 namespace의 다른 파드도 API에 닿을 수 있다는 점을 알아 두세요.

### web 저장소의 약속

- Dockerfile은 `node:22-slim` 기반의 여러 단계 빌드이고, 컨테이너는 `node` 계정(UID 1000)으로 3000번 포트에서 `node server.js`를 실행합니다.
- **`next.config.ts`에 `output: "standalone"`이 있어야 합니다.** 없으면 Dockerfile이 빌드 중에 멈추고 안내 메시지를 냅니다.
- **`API_ORIGIN`은 빌드 시점에 확정됩니다.** `next.config.ts`의 `rewrites`가 빌드 때 라우팅 표에 굳어지기 때문에, 런타임 환경변수(`web-config`)로는 바꿀 수 없습니다. 워크플로가 빌드 인자 `API_ORIGIN=http://api`를 넣습니다. Service 이름이 바뀌면 이 값도 바꾸고 이미지를 다시 빌드해야 합니다.
- `NEXT_PUBLIC_SITE_URL`도 빌드 시점에 번들에 들어갑니다. 워크플로가 `https://travel.tsungmn.com`을 넣습니다. 비워 두는 경우에는 Dockerfile이 변수 자체를 설정하지 않은 것으로 처리합니다. (빈 문자열로 들어가면 공유 링크의 주소가 비어 버립니다.)
- **공항 데이터 `public/data/airports.json`(약 1.5MB)은 git에 커밋되어 있어야 합니다.** 이 파일이 없으면 이미지는 빌드되지만 항공편 입력의 공항 검색이 동작하지 않습니다. Dockerfile은 파일이 없으면 빌드를 멈춥니다. 원본과 중간 데이터는 `data/`에 있고 git에서 제외되어 있으며(`.gitignore`와 `.dockerignore`), 갱신은 로컬에서 `npm run build:airports`를 실행한 뒤 결과 파일을 커밋하는 방식입니다. 절차는 web 저장소 README의 "공항 데이터 준비"에 있습니다.
- `package-lock.json`이 `package.json`과 맞아야 합니다. 어긋나 있으면 이미지 빌드의 `npm ci` 단계에서 실패합니다. 의존성을 바꾼 뒤에는 `npm install`로 lock을 갱신해 함께 커밋하세요.
- `npm ci`는 `postinstall` 스크립트를 실행해야 합니다. 이 스크립트가 지도 라이브러리의 워커 파일을 `public/`에 복사합니다. `--ignore-scripts`를 쓰면 지도가 동작하지 않습니다.
- `/data/*`는 브라우저가 1시간 캐시하도록 `next.config.ts`에서 설정되어 있습니다.

### web 워크플로의 브랜치 규칙

| push 대상 | 하는 일 |
| --- | --- |
| `main` 이외의 모든 브랜치 | amd64 이미지를 빌드해서 컨테이너를 띄우고 스모크 테스트만 합니다. 공항 데이터가 3천 개 넘게 들어 있는지, 지도 워커 파일이 있는지, root가 아닌 계정으로 실행되는지 확인합니다. 이미지를 올리지도 배포하지도 않습니다. |
| `main` | 위 테스트를 통과하면 amd64와 arm64 이미지를 GHCR에 올리고, 이 저장소의 web 태그를 갱신합니다. |

서버가 amd64 한 종류뿐이면 워크플로의 `platforms`를 `linux/amd64`로 줄여 `main` 빌드를 빠르게 할 수 있습니다.

### 이미지 라벨

각 Dockerfile에는 `org.opencontainers.image.source` 라벨이 있어서 GHCR 패키지가 해당 저장소와 연결됩니다. 연결되어 있으면 패키지의 접근 권한을 저장소 기준으로 관리하기 쉽습니다.

---

## 9. DB 백업과 복원

### 백업 현황

`postgres-backup` CronJob이 매일 04:00(KST)에 `pg_dump`를 실행해 `postgres-backup` 디스크에 `travel-<날짜>-<시각>.sql.gz`로 저장하고, 14일이 지난 파일을 지웁니다.

지금 바로 백업을 한 번 만들려면 이렇게 합니다.

```bash
kubectl -n travel-planner create job --from=cronjob/postgres-backup manual-backup-$(date +%s)
kubectl -n travel-planner get jobs
```

**백업 디스크는 DB와 같은 서버에 있습니다.** 서버 디스크가 고장 나면 DB와 백업을 함께 잃습니다. 중요한 시점(큰 변경 전, 서버를 옮기기 전)에는 덤프를 다른 곳으로 꺼내 두세요. 정기적으로(예: 한 달에 한 번) 꺼내 두는 습관을 권장합니다.

```bash
# 덤프를 로컬 PC로 바로 받기
kubectl -n travel-planner exec postgres-0 -- \
  sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" | gzip' > travel-$(date +%F).sql.gz
```

### 복원

1. ArgoCD UI에서 `travel-planner-prod`의 **Auto-Sync를 끕니다.** (`selfHeal`이 켜져 있으면 아래에서 줄인 replicas가 곧바로 되돌아옵니다.)
2. API를 멈춥니다.

   ```bash
   kubectl -n travel-planner scale deploy/api --replicas=0
   ```

3. 복원할 덤프를 준비합니다. 위의 방법으로 로컬 PC에 받아 둔 파일을 쓰는 것이 가장 단순합니다. 백업 디스크 안의 파일을 쓰려면 임시 파드로 들여다보고 `kubectl cp`로 꺼냅니다.

   ```bash
   kubectl -n travel-planner run backup-shell --rm -it --restart=Never --image=postgres:17 \
     --overrides='{"spec":{"containers":[{"name":"backup-shell","image":"postgres:17","command":["sh"],"stdin":true,"tty":true,"volumeMounts":[{"name":"b","mountPath":"/backup"}]}],"volumes":[{"name":"b","persistentVolumeClaim":{"claimName":"postgres-backup"}}]}}' \
     -- sh
   # 파드 안에서: ls -l /backup
   ```

4. 덤프를 DB에 넣습니다. 기존 데이터를 지우고 넣는 방법입니다.

   ```bash
   kubectl -n travel-planner exec -i postgres-0 -- \
     sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"'
   gunzip -c travel-<날짜>.sql.gz | kubectl -n travel-planner exec -i postgres-0 -- \
     sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
   ```

5. Auto-Sync를 다시 켜면 API가 원래 replicas로 돌아옵니다. 그러면 `initContainer`의 마이그레이션이 실행되어 스키마가 최신이 됩니다.

복원은 미리 해 본 적이 없으면 실제 사고 때 실패하기 쉽습니다. 서비스가 안정되면 한 번 연습해 두세요.

---

## 10. 사고가 났을 때

| 상황 | 증상 | 대처 |
| --- | --- | --- |
| 서버나 디스크가 완전히 사라졌다 | 서비스 전체 중단 | [6장](#6-처음부터-다시-만들기)으로 클러스터를 다시 세우고, [9장](#복원)의 방법으로 따로 보관한 덤프를 복원합니다. 서버 안의 백업 디스크는 같이 사라졌다고 보세요. |
| `postgres-secret`의 비밀번호를 잃었다 (DB 디스크는 멀쩡함) | API가 DB 인증에 실패 | 컨테이너 안에서는 비밀번호 없이 로컬 접속이 되는 것이 공식 이미지의 기본 동작입니다. 아래 명령으로 비밀번호를 새 값으로 바꾸고, `postgres-secret`과 `api-secret`의 `DATABASE_URL`을 같은 새 값으로 다시 만든 뒤 API를 재시작합니다. |
| `CONFIG_REPO_TOKEN`이 만료됐다 | 이미지는 빌드되는데 이 저장소의 태그가 안 바뀌고 워크플로의 deploy 잡이 실패 | 토큰을 다시 발급해 시크릿을 교체하고 워크플로를 다시 실행합니다. 급하면 [CI 없이 손으로 배포하기](#ci-없이-손으로-배포하기)로 태그를 직접 고칩니다. |
| `ghcr-pull` 토큰이 만료됐다 | 새 파드가 `ImagePullBackOff` (이미 떠 있는 파드는 영향 없음) | 새 토큰으로 [6-5](#6-5-namespace와-secret-만들기)의 방법으로 Secret을 덮어씁니다. |
| 도메인이 만료됐거나 Cloudflare 설정이 사라졌다 | 사이트 접속 불가, Cloudflare 오류 화면 | 도메인을 갱신하고, [6-4](#6-4-cloudflare-tunnel-만들기)의 터널과 Public Hostname을 확인합니다. |
| cloudflared 버전이 너무 낡아 거절당한다 | cloudflared 파드가 재시작을 반복 | [13장](#13-오래-지난-뒤-점검할-것)에 따라 새 안정 버전으로 올립니다. |
| GitHub 조직이나 개인 계정에 접근할 수 없다 | 이미지 빌드와 배포 불가 | 서비스는 이미 떠 있는 이미지로 계속 동작합니다. 접근을 복구하거나, 앱 코드를 다른 곳에 올려 이미지를 다시 만들어 이미지 경로를 모두 바꿉니다([5장](#5-설정-값과-서로-맞아야-하는-값)). |

비밀번호를 바꾸는 명령은 이렇습니다.

```bash
kubectl -n travel-planner exec -it postgres-0 -- \
  sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "ALTER USER \"$POSTGRES_USER\" PASSWORD '"'"'<새 비밀번호>'"'"';"'
```

---

## 11. 설정 결정 이유

설정을 지우거나 바꾸기 전에 아래 이유를 먼저 확인해 주세요.

| 설정 | 이유 |
| --- | --- |
| API `replicas: 1`, `strategy: Recreate` | Nominatim 호출 간격 제한이 프로세스 안에서만 지켜집니다. 파드가 둘이면 제한을 어깁니다. 업데이트 중에도 둘이 동시에 뜨지 않게 합니다. |
| 마이그레이션을 PreSync Job이 아니라 `initContainer`로 | PreSync 훅은 ConfigMap보다 먼저 실행되어 `configMapGenerator`의 해시 이름과 충돌하기 쉽습니다. 마이그레이션은 멱등이고 advisory lock이 있습니다. |
| `/api/readyz`와 `/api/healthz` 분리 | readyz는 DB 연결까지 확인하고, healthz는 프로세스만 확인합니다. DB가 잠깐 죽었다고 API 파드가 계속 재시작되지 않게 하려는 것입니다. |
| API 메모리 limit 512Mi | argon2 비밀번호 해시가 요청당 수십 MB를 씁니다. |
| `enableServiceLinks: false` | Service 이름을 딴 환경 변수(예: `POSTGRES_PORT`)가 파드에 주입되어 설정과 충돌하는 것을 막습니다. |
| `runAsUser: 10001` | API Dockerfile의 `appuser` UID입니다. 이름만으로는 non-root 검증이 안 되어 숫자로 지정합니다. |
| DB 리소스의 `Prune=false,Delete=false` | sync나 Application 삭제로 DB와 백업이 지워지지 않게 합니다. |
| Application을 api/web과 DB로 분리 | API 쪽 배포나 prune이 DB에 영향을 주지 않게 합니다. |
| prod `APP_ENV` 검증 | prod에서 localhost, HTTP, docs 공개 설정이면 서버가 시작되지 않습니다. dev는 로컬 DB 호스트만 허용해서 prod DB에 붙지 못합니다. |
| 이미지 태그를 커밋 SHA 40자로 고정 | 어떤 코드가 떠 있는지 Git 기록으로 추적하고, 롤백이 `git revert` 한 번으로 끝납니다. |
| web 이미지를 `output: "standalone"`으로 | 실행에 필요한 파일만 이미지에 담아 이미지를 작게 하고, `node server.js` 한 줄로 실행되게 합니다. 개발 의존성은 빌드 단계에서만 쓰고 최종 이미지에는 넣지 않습니다. |
| web 컨테이너를 `node` 계정으로 실행 | root 실행을 피합니다. 숫자 UID 없이 `runAsNonRoot`만 켜면 쿠버네티스가 non-root를 확인하지 못해 파드가 뜨지 않으므로, 켠다면 `runAsUser: 1000`을 함께 넣어야 합니다. |
| 외부 접속을 Cloudflare Tunnel로 | 공유기 포트포워딩과 공인 IP, 인증서 관리가 필요 없고, 서버가 밖으로만 연결합니다. |

---

## 12. dev와 prod의 구분

| 장치 | 효과 |
| --- | --- |
| `APP_ENV=prod`는 ConfigMap에서만 설정하고 코드가 검증 | prod에서 localhost, HTTP, docs 공개 설정이면 서버가 시작되지 않습니다. |
| dev는 로컬 DB 호스트(`localhost`, `db` 등)만 허용 | dev가 prod DB(`postgres` Service)에 붙으면 시작에 실패합니다. |
| dev는 클러스터에 없고, namespace와 Secret이 분리됨 | dev 데이터와 prod 데이터가 물리적으로 섞일 수 없습니다. |
| Google OAuth, Geoapify 키를 환경별로 발급 | 한쪽 키가 유출되어도 다른 쪽은 안전합니다. |
| 이미지는 SHA 태그로 고정하고 환경별 값은 overlay에만 둠 | prod가 어떤 코드와 설정으로 떠 있는지 Git 기록으로 추적됩니다. |
| web의 API 주소는 빌드 인자로만 주입 | 개발 중에는 `API_ORIGIN`이 없으면 `http://127.0.0.1:8000`을 쓰고, prod 이미지에는 `http://api`가 박힙니다. |
| web의 `.dockerignore`가 `.env*`를 제외 | 로컬의 `.env.local`이 prod 이미지에 섞이지 않습니다. |

dev에서는 로컬에서 `docker compose`(api와 DB)와 `npm run dev`(web)로 실행합니다. web의 mock은 web 저장소가 담당하고, prod 빌드에서 mock이 켜져 있으면 빌드가 실패합니다.

---

## 13. 오래 지난 뒤 점검할 것

### 이 문서를 쓴 시점의 버전

| 항목 | 값 (2026-10-12) |
| --- | --- |
| Kubernetes | 1.27 이상 필요 |
| ArgoCD | (설치한 버전을 [서버 기록](#서버-기록)에 적어 두세요) |
| PostgreSQL | 17 |
| cloudflared | 2026.9.3 |
| API 베이스 이미지 | `python:3.12-slim` |
| web 베이스 이미지 | `node:22-slim`, Next.js 16 |
| GitHub Actions | `actions/checkout@v4`, `docker/setup-qemu-action@v3`, `docker/setup-buildx-action@v3`, `docker/login-action@v3`, `docker/build-push-action@v6` |

### 점검 항목

| 항목 | 점검 방법 |
| --- | --- |
| PostgreSQL 버전 | `postgres:17`은 영구히 지원되지 않습니다. 지원 종료 시점을 확인하세요. 메이저 버전은 이미지 태그만 바꿔서 올릴 수 없고, **덤프를 받아 새 버전의 빈 DB에 복원**해야 합니다. 순서는 (1) 덤프를 받아 둔다 (2) Auto-Sync를 끄고 API를 멈춘다 (3) StatefulSet과 데이터 디스크를 지우고 새 이미지 태그로 다시 만든다 (4) 덤프를 복원한다 (5) 백업 CronJob의 이미지 태그도 같이 올린다 입니다. |
| cloudflared 버전 | 너무 오래된 버전은 Cloudflare가 연결을 거부할 수 있습니다. [릴리스 페이지](https://github.com/cloudflare/cloudflared/releases)에서 새 안정 버전을 고르고, 올리기 전에 `docker manifest inspect cloudflare/cloudflared:<버전>`으로 이미지가 실제로 있는지 확인하세요. 없는 태그를 쓰면 `ImagePullBackOff`가 납니다. |
| Kubernetes, ArgoCD | 클러스터와 ArgoCD를 올린 뒤에도 `kubectl kustomize overlays/prod`가 오류 없이 나오는지, 사용 중인 API 버전(`apps/v1`, `batch/v1` 등)이 아직 지원되는지 확인합니다. |
| 베이스 이미지의 보안 업데이트 | 앱 저장소의 Dockerfile을 주기적으로 다시 빌드해 새 태그로 배포합니다. Python 3.12와 Node 22도 지원 종료 시점이 있습니다. |
| GitHub Actions의 버전 | `actions/*`, `docker/*`의 메이저 버전이 낡아 경고가 나오기 시작하면 올립니다. |
| `ghcr-pull` 토큰 | 만료되면 새 파드가 이미지를 받지 못합니다. 만료일을 확인하세요. |
| `CONFIG_REPO_TOKEN` | 만료되면 태그 갱신이 실패합니다. 만료일을 확인하세요. |
| GHCR 인증 방식 | fine-grained 토큰 지원 여부 등 정책이 바뀌었는지 GitHub 문서를 확인합니다. |
| 도메인 | `tsungmn.com` 갱신일 |
| Cloudflare 대시보드 | 메뉴 이름과 입력 형식이 바뀌었을 수 있습니다. [6-4](#6-4-cloudflare-tunnel-만들기)를 단계의 의도 중심으로 읽으세요. |
| 공항 데이터 | 노선과 공항 정보는 바뀝니다. web 저장소의 방법으로 다시 만들어 커밋합니다. |
| 백업 | 덤프를 서버 밖으로 마지막으로 꺼낸 날짜, 복원을 마지막으로 연습한 날짜를 확인합니다. |

### 점검 주기 제안

| 주기 | 할 일 |
| --- | --- |
| 한 달에 한 번 | 덤프를 서버 밖으로 복사, `kubectl get pods`와 ArgoCD 상태 확인 |
| 석 달에 한 번 | 베이스 이미지 보안 업데이트 반영, 토큰 만료일 확인 |
| 일 년에 한 번 | PostgreSQL, cloudflared, Kubernetes, ArgoCD 버전 점검, 복원 연습, 도메인 갱신 확인 |

---

## 14. 문제 해결

| 증상 | 확인할 것 |
| --- | --- |
| `/api/healthz`가 Next 페이지로 응답함 | Cloudflare Public Hostname의 `/api` 규칙이 위에 있는지, 경로 형식이 맞는지 |
| 파드가 `ImagePullBackOff` | 이미지 경로의 조직 이름(`tsungmn`), 태그가 GHCR에 실제로 있는지, `ghcr-pull` Secret의 계정과 토큰 만료, 토큰의 SSO 승인, 토큰 계정의 패키지 읽기 권한. 패키지를 public으로 바꾸면 해결됩니다. |
| `cloudflared`가 `ImagePullBackOff` | `base/cloudflared/deployment.yaml`의 버전이 실제로 존재하는지 (`docker manifest inspect`) |
| 렌더링에서 태그에 `<`가 들어 있다는 오류 | `overlays/prod/kustomization.yaml`의 자리표시자를 채우지 않음 (새로 세울 때만) |
| API 파드가 `Init:Error` 또는 `Init:CrashLoopBackOff` | `kubectl -n travel-planner logs deploy/api -c migrate`, `api-secret`의 `DATABASE_URL`, DB 파드 상태. 처음 배포 직후라면 DB가 준비되는 동안 몇 번 재시작되는 것은 정상입니다. |
| API가 DB 인증에 실패함 | `postgres-secret`을 새로 만들면서 처음과 다른 비밀번호를 넣었을 수 있습니다. [10장](#10-사고가-났을-때)을 보세요. |
| API 파드가 시작하지 않음 | prod 검증 값(`APP_ENV`, `PUBLIC_URL`, `COOKIE_SECURE`, `DOCS_ENABLED`) |
| web은 뜨는데 로그인이나 `/api` 호출이 500 | web 이미지를 빌드할 때의 `API_ORIGIN`이 `http://api`가 아니었을 수 있습니다. 빌드 때 확정되므로 워크플로의 `API_ORIGIN`을 확인하고 이미지를 다시 빌드하세요. |
| 항공편 입력에서 공항 목록이 안 나옴 | 브라우저의 `/data/airports.json`이 404인지 확인합니다. web 저장소에 `public/data/airports.json`이 커밋되어 있는지 보세요. |
| 지도가 안 나옴 | `/maplibre-gl-worker.mjs`가 200인지 확인합니다. 이미지 빌드에서 `npm ci`가 `postinstall`을 실행했는지 보세요. |
| web 이미지 빌드가 `npm ci`에서 실패 | `package-lock.json`이 `package.json`과 어긋남. `npm install`로 lock을 갱신해 커밋하세요. |
| web 이미지 빌드가 `standalone` 안내와 함께 멈춤 | `next.config.ts`에 `output: "standalone"`이 있는지 확인 |
| web 파드 안에서 `curl localhost:3000`이 안 됨 | 쿠버네티스가 `HOSTNAME` 환경변수를 파드 이름으로 덮어써서 서버가 파드 IP에만 바인딩된 것일 수 있습니다. Service와 프로브는 파드 IP로 접속하므로 영향이 없고, 필요하면 Deployment에 `env: HOSTNAME=0.0.0.0`을 추가합니다. |
| cloudflared가 계속 재시작됨 | `cloudflared-secret`의 토큰, 이미지 버전이 실제로 존재하는지 |
| 사이트가 Cloudflare 오류 화면(1033 등)을 보여 줌 | cloudflared 파드가 떠 있는지, 터널이 대시보드에서 Healthy인지 |
| ArgoCD가 변경을 반영하지 않음 | 저장소 접근 권한, Application 상태(OutOfSync / Degraded), Refresh 해 보기 |
| 로그인이 안 됨 | Google OAuth redirect URI가 `PUBLIC_URL`과 정확히 일치하는지 |
| 이미지는 빌드됐는데 배포가 안 됨 | 워크플로의 deploy 잡 로그. `CONFIG_REPO_TOKEN` 만료, 조직 승인 대기, 이 저장소의 `main` 보호 규칙(직접 push 거부)이 흔한 원인입니다. |
| deploy 잡이 "이미지 항목을 찾지 못했습니다"로 실패 | `overlays/prod/kustomization.yaml`의 `images:` 항목이 [8장](#이-저장소가-지켜야-할-모양)의 모양인지, 이미지 경로가 워크플로가 만드는 이름과 같은지 |

---

## 15. 서브도메인을 바꿔야 한다면

서브도메인은 여러 곳에 박혀 있어서 나중에 바꾸면 이미 만든 공유 링크가 깨지고 로그인 쿠키도 사라집니다. 바꿔야 한다면 아래를 함께 수정하세요.

1. `overlays/prod/kustomization.yaml`의 `PUBLIC_URL`
2. Cloudflare Tunnel의 Public Hostname (경로 두 개 모두)
3. Google OAuth prod 클라이언트의 redirect URI (`https://<새 호스트>/api/auth/google/callback`)
4. web 저장소 워크플로의 `SITE_URL` (바꾼 뒤 web 이미지를 다시 빌드해야 반영됩니다.)

---

## 16. 이 문서가 쓰인 시점의 상태

이 표는 문서를 고칠 때마다 함께 고쳐 주세요. 현재 설정이 어디까지 확인된 상태인지 알려 줍니다.

**2026-10-12 기준**

| 항목 | 상태 |
| --- | --- |
| cloudflared `2026.9.3` 이미지 | 존재 확인됨 |
| 클러스터에서 도는 이미지 태그 | 각 앱 저장소의 최신 커밋과 같음 |
| api 저장소의 `deploy.yml` | 저장소에 커밋됨 |
| GitHub 조직 `tsungmn`, 이미지 경로 | 조직 기준으로 통일됨 (`tsungmnli`는 개인 계정) |
| web 저장소의 Dockerfile과 워크플로 | 준비됨. 브랜치에서 시험 빌드를 확인한 뒤 `main`에 병합해 첫 배포를 해야 합니다. |
| web 이미지의 공항 데이터 포함 | `public/data/airports.json`을 커밋하고 시험 빌드의 스모크 테스트로 확인해야 합니다. |
| `ghcr-pull`의 조직 계정 기준 설정 | 서버를 방문해서 Secret을 교체하고 이미지 받기를 확인해야 합니다. |
| DB 덤프의 서버 밖 보관 | 아직 정기적으로 하고 있지 않음. [서버 기록](#서버-기록)의 칸을 채워 주세요. |

**변경 이력**

| 날짜 | 내용 |
| --- | --- |
| 2026-10-12 | 조직 계정(`tsungmn`) 기준으로 이미지 경로를 정리하고, web의 이미지 빌드와 공항 데이터, 토큰 구성, 재구축 순서, 사고 대응, 점검 주기를 추가했습니다. |