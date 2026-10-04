# travel-planner-config

**여행길**(travel-planner)을 쿠버네티스 클러스터에 배포하는 설정 저장소입니다.
Kustomize로 매니페스트를 만들고, ArgoCD가 이 저장소를 감시해서 클러스터에 자동으로 반영합니다.
이 저장소에는 애플리케이션 코드가 없습니다. 어떤 이미지를, 어떤 설정으로 띄울지만 담습니다.

> 이 저장소를 처음 보거나 오랜만에 여는 분은 먼저 [확인해야 할 커스텀 값](#확인해야-할-커스텀-값)과 [이 저장소 밖에 있는 것](#이-저장소-밖에-있는-것)을 읽어 주세요.
> 그다음 상황에 맞는 절로 이동하면 됩니다.
>
> | 상황 | 갈 곳 |
> | --- | --- |
> | 처음 배포하거나, 서버를 새로 만들었다 | [처음 배포하기](#처음-배포하기) |
> | 새 버전을 배포하고 싶다 | [평소 배포하기](#평소-배포하기) |
> | 배포했더니 문제가 생겼다 | [롤백](#롤백), [문제 해결](#문제-해결) |
> | DB를 되살려야 한다 | [DB 백업과 복원](#db-백업과-복원) |
> | 몇 년이 지나 버전이 낡았다 | [오래 지난 뒤 점검할 것](#오래-지난-뒤-점검할-것) |

- 서비스 주소: https://travel.tsungmn.com
- 배포 방식: GitOps (이 저장소의 커밋이 곧 배포)
- 외부 접속: Cloudflare Tunnel (Ingress, cert-manager, 공유기 포트포워딩을 쓰지 않습니다)
- 이미지 저장소: GHCR (`ghcr.io/tsungmnli/...`)

---

## 이름 정리

이 프로젝트에는 이름이 여러 개 나오지만 쓰이는 층이 서로 다릅니다.

| 층 | 이름 | 쓰이는 곳 |
| --- | --- | --- |
| 브랜드 (사용자에게 보임) | 여행길 | 화면 문구, 브라우저 탭 제목, Google 동의 화면, 개인정보처리방침 |
| 코드네임, 서브도메인 | `travel-planner`, `travel` | repo, 컨테이너 이미지, namespace, `travel.tsungmn.com` |
| 데이터 용어 | `trip` (여행 한 건) | `/trips`, `/api/trips`, `trips` 테이블 |

`travel-planner`는 제품 이름이고 `trip`은 그 안의 데이터 한 건을 부르는 말이라 섞여 있어도 문제가 아닙니다.
이 저장소에서는 코드네임 `travel-planner`로 통일합니다.

계정 이름도 둘이라 헷갈리기 쉽습니다.

| 서비스 | 계정 |
| --- | --- |
| GitHub (그리고 GHCR) | `tsungmnli` |
| Docker Hub | `tsungmn` (이 프로젝트에서는 쓰지 않습니다) |

`ghcr.io`는 GitHub의 레지스트리이므로 경로에는 **GitHub 계정**(`tsungmnli`)이 들어갑니다.

---

## 구조

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
| `base/api` | API 서버. 시작하기 전에 `initContainer`로 DB 마이그레이션(`python -m app.migrate`)을 실행합니다. |
| `base/web` | Next.js 웹 서버 |
| `base/cloudflared` | Cloudflare로 나가는 터널 연결을 만들고, 들어온 트래픽을 api와 web Service로 전달합니다. |
| `db` | PostgreSQL StatefulSet, 그리고 매일 04:00(KST)에 도는 백업 CronJob |
| `overlays/prod` | namespace, 이미지 태그, 환경 변수 등 prod에서만 달라지는 값 |
| `argocd` | "이 경로를 감시해서 배포하라"고 ArgoCD에 알려 주는 Application |

dev 환경은 클러스터에 올리지 않으므로 overlay는 `prod` 하나뿐입니다.
staging이 필요해지면 `overlays/staging`을 추가하면 됩니다.

### 트래픽이 흐르는 길

```
브라우저 → Cloudflare → cloudflared(파드) ─┬─ /api/*  → Service api → API 파드 → Service postgres
                                           └─ 그 외   → Service web → Web 파드
```

한 도메인에서 경로로만 나누기 때문에 브라우저 입장에서는 같은 출처입니다. 쿠키와 CORS 문제가 없습니다.
API 경로가 처음부터 `/api/...`라서 경로를 다시 쓰는(rewrite) 설정도 필요 없습니다.

---

## 이 저장소 밖에 있는 것

**Git에 없어서, 서버가 사라지면 이 저장소만으로는 되살릴 수 없는 것들입니다.**
아래 항목은 비밀번호 관리자 같은 곳에 따로 보관해 두는 것을 권장합니다.

| 항목 | 어디에 있나 | 잃어버리면 |
| --- | --- | --- |
| `postgres-secret` | 클러스터의 Secret | **DB 비밀번호가 사라집니다.** 디스크에 남은 DB는 처음 만들 때의 비밀번호를 그대로 기억하므로, 새 값으로 Secret을 만들면 API가 접속하지 못합니다. 같은 값으로 다시 만들어야 합니다. |
| `api-secret` | 클러스터의 Secret | 새로 만들면 됩니다. `SESSION_SECRET`이 바뀌면 모든 사용자가 로그아웃됩니다. |
| `cloudflared-secret` | 클러스터의 Secret | Cloudflare 대시보드에서 토큰을 다시 받으면 됩니다. |
| `ghcr-pull` | 클러스터의 Secret | GitHub에서 `read:packages` 토큰을 새로 만들면 됩니다. |
| Google OAuth prod 클라이언트 | Google Cloud 콘솔 | 클라이언트 ID/시크릿은 콘솔에서 다시 볼 수 있습니다. redirect URI는 아래 [서브도메인을 바꿔야 한다면](#서브도메인을-바꿔야-한다면)을 보세요. |
| Geoapify prod 키 | Geoapify 대시보드 | 새로 발급합니다. |
| Cloudflare Tunnel과 Public Hostname | Cloudflare 대시보드 | 처음 배포하기의 2단계를 다시 하면 됩니다. |
| DNS (`tsungmn.com`) | Cloudflare | 도메인 갱신이 끊기면 서비스 전체가 멈춥니다. 도메인 만료일을 확인하세요. |
| DB 데이터 | 클러스터의 PVC (`data-postgres-0`) | 백업 PVC에 14일치 덤프가 있습니다. 같은 서버에 있으니 [다른 곳에도 복사](#db-백업과-복원)하세요. |
| 이미지 빌드 CI | 각 앱 저장소 (API, web)의 `.github/workflows` | 이 저장소에는 없습니다. |
| CI가 이 저장소에 push할 권한 | 앱 저장소의 Secrets (토큰 또는 deploy key) | 만료되면 이미지는 빌드되지만 태그 갱신 커밋이 실패합니다. |

---

## 확인해야 할 커스텀 값

이 저장소에서 사람이 채우거나 확인해야 하는 값의 목록입니다.
상태는 이 README를 쓴 시점 기준이고, 채우면 표를 고쳐 주세요.

범례: ✅ 채워져 있음 / ⬜ 아직 안 채움 / 🔎 채웠는지 확인 필요

| 파일 | 값 | 설명 / 바꿀 때 주의 |
| --- | --- | --- |
| `overlays/prod/kustomization.yaml` | api `newTag: "<커밋 SHA>"` | API 이미지 태그. 채우는 방법은 [처음 배포하기의 4단계](#4-이미지를-처음-만들고-태그-채우기)입니다. |
| `overlays/prod/kustomization.yaml` | web `newTag: "<커밋 SHA>"` | Web 이미지 태그. 위와 같습니다. |
| `base/cloudflared/deployment.yaml` | `image: cloudflare/cloudflared:<고정할 버전>` | `latest`를 쓰지 않고 안정 버전을 고정합니다. [cloudflared 릴리스 페이지](https://github.com/cloudflare/cloudflared/releases)에서 최신 안정 버전을 확인해 넣으세요. |
| 클러스터 | 4개의 Secret | [처음 배포하기의 3단계](#3-namespace와-secret-만들기)대로 만들었는지 `kubectl -n travel-planner get secret`으로 확인합니다. |
| `overlays/prod/kustomization.yaml` | `namespace: travel-planner` | `argocd/*.yaml`의 `destination.namespace`와 `db/kustomization.yaml`의 `namespace`도 같이 바꿔야 합니다. |
| `overlays/prod/kustomization.yaml` | `PUBLIC_URL=https://travel.tsungmn.com` | [서브도메인을 바꿔야 한다면](#서브도메인을-바꿔야-한다면)의 세 곳을 같이 바꿉니다. |
| `overlays/prod/kustomization.yaml` | `APP_ENV=prod`, `COOKIE_SECURE=true`, `DOCS_ENABLED=false` | 코드가 prod 값을 검증하므로 이 값이 어긋나면 API가 시작되지 않습니다. |
| `overlays/prod/kustomization.yaml` | `NOMINATIM_USER_AGENT` | Nominatim 사용 정책상 연락처가 들어가 있어야 합니다. 저장소를 공개한다면 공개해도 되는 연락처인지 확인하세요. |
| `overlays/prod/kustomization.yaml` | `API_ORIGIN=http://api` | 서버 사이드 렌더링이 API를 부르는 클러스터 내부 주소입니다. Service 이름 `api`와 맞아야 합니다. |
| `argocd/*.yaml` | `repoURL: https://github.com/tsungmnli/travel-planner-config.git`, `targetRevision: main` | 저장소를 옮기거나 기본 브랜치 이름을 바꾸면 고쳐야 합니다. |
| `db/backup.yaml` | 매일 `0 4 * * *` (KST), 14일 보관, 백업 PVC 5Gi | 보관 일수는 `-mtime +14`입니다. |
| `db/statefulset.yaml` | `postgres:17`, 데이터 PVC 5Gi | 버전 올리기는 [오래 지난 뒤 점검할 것](#오래-지난-뒤-점검할-것)을 보세요. |
| `base/api/deployment.yaml` | `replicas: 1`, `strategy: Recreate`, `runAsUser: 10001` | 이유는 [설정 결정 이유](#설정-결정-이유)를 보세요. `10001`은 API Dockerfile의 `appuser` UID와 같아야 합니다. |

```bash
grep -rn "ghcr.io/" base overlays
```

---

## 처음 배포하기

서버를 새로 만들었거나, 이 서비스를 처음 올리는 경우의 전체 순서입니다.
**이미지가 아직 없는 상태에서 ArgoCD Application부터 등록하면 `ImagePullBackOff`가 납니다.**
그래서 이미지를 먼저 만들고, 그 태그를 이 저장소에 적은 다음, 마지막에 Application을 등록합니다.

### 1. 클러스터에 먼저 있어야 하는 것

- Kubernetes 1.27 이상 (백업 CronJob의 `timeZone` 필드 때문입니다)
- ArgoCD
- 기본 StorageClass (k3s라면 `local-path`)

Ingress 컨트롤러와 cert-manager는 **필요 없습니다.**

> ingress-nginx(쿠버네티스 커뮤니티 버전)는 2026년 3월에 종료되어 더 이상 보안 패치가 없습니다.
> 이 서비스는 Ingress를 거치지 않으므로 영향이 없습니다.
> 클러스터의 다른 서비스가 ingress-nginx를 쓰고 있다면 그쪽만 교체 계획이 필요합니다.
>
> ```bash
> kubectl get deploy -A -o wide | grep -i nginx
> # registry.k8s.io/ingress-nginx/controller → 종료된 커뮤니티 버전
> # nginx/nginx-ingress                      → F5 버전 (계속 지원)
> ```

### 2. Cloudflare Tunnel 만들기

`tsungmn.com`의 DNS가 Cloudflare에 있어야 합니다.
(메뉴 이름은 시간이 지나면 바뀔 수 있습니다. 대시보드에서 비슷한 이름을 찾으세요.)

1. Cloudflare 대시보드에서 **Zero Trust → Networks → Tunnels → Create a tunnel**을 선택하고 **Cloudflared**를 고릅니다.
2. 화면에 나오는 **토큰**을 복사해 둡니다. (3단계에서 씁니다.)
3. 같은 터널에 **Public Hostname**을 두 개 추가합니다. **순서가 중요합니다.** `/api` 규칙이 위에 있어야 합니다.

| 순서 | 호스트 이름 | 경로 | 서비스 |
| --- | --- | --- | --- |
| 1 | `travel.tsungmn.com` | `^/api/` | `http://api.travel-planner.svc.cluster.local:80` |
| 2 | `travel.tsungmn.com` | (비움) | `http://web.travel-planner.svc.cluster.local:80` |

호스트 이름을 추가하면 `travel` CNAME 레코드가 DNS에 자동으로 만들어집니다.
이미 `travel` 레코드가 있으면 생성이 실패하므로 DNS 화면에서 먼저 확인하세요.

경로 입력 형식(정규식인지 `/api/*` 같은 패턴인지)은 Cloudflare 화면에서 확인해야 합니다.
7단계의 `curl` 확인이 통과하지 않으면 형식을 바꿔서 다시 시도하세요.

### 3. namespace와 Secret 만들기

Secret은 **Git에 올리지 않고** 클러스터에 직접 만듭니다.
사용한 값(특히 DB 비밀번호)은 비밀번호 관리자에 따로 보관해 두세요.

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

# Cloudflare Tunnel 토큰 (2단계에서 복사한 값)
kubectl -n travel-planner create secret generic cloudflared-secret \
  --from-literal=TUNNEL_TOKEN='<토큰>'

# 이미지가 비공개(GHCR)라면
kubectl -n travel-planner create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username=tsungmnli \
  --docker-password='<read:packages 권한 GitHub 토큰>'
```

- `--docker-username`은 Docker Hub 계정(`tsungmn`)이 아니라 **GitHub 계정(`tsungmnli`)** 입니다.
- 비밀 값이 셸 히스토리에 남지 않게 주의하세요. 명령 앞에 공백을 두고 `HISTCONTROL=ignorespace`를 설정하는 방법이 있습니다.
- 이미지를 공개 패키지로 바꿨다면 `ghcr-pull`과 두 Deployment의 `imagePullSecrets`는 없어도 됩니다.

**Google OAuth**: Google Cloud 콘솔에서 **prod 전용 OAuth 클라이언트**를 만들고, 승인된 리디렉션 URI에 아래 주소를 등록합니다. 동의 화면의 앱 이름은 **여행길**로 합니다.

```
https://travel.tsungmn.com/api/auth/google/callback
```

dev와 prod의 Google 키, Geoapify 키는 각각 따로 발급합니다. 한쪽이 유출되어도 다른 쪽은 안전합니다.

### 4. 이미지를 처음 만들고 태그 채우기

이 저장소의 `overlays/prod/kustomization.yaml`에는 태그 자리표시자가 있습니다.
태그는 **앱 저장소의 커밋 SHA**이고, 그 SHA로 빌드된 이미지가 GHCR에 있어야 합니다.

1. API 저장소와 web 저장소를 각각 GitHub에 push합니다. (`main`에 push하면 CI가 이미지를 빌드해 올립니다. CI는 각 앱 저장소에 있습니다.)
2. 각 저장소의 Actions 탭에서 빌드가 성공했는지 확인합니다.
3. GitHub의 해당 패키지 페이지(저장소 → Packages)에서 **실제로 올라간 태그**를 복사합니다. 앱 저장소에서 `git rev-parse HEAD`로 얻은 값과 같아야 하지만, 태그 형식(전체 40자인지 앞 7자인지)은 CI 설정에 따르므로 패키지 페이지의 값을 그대로 쓰는 편이 안전합니다.
4. `overlays/prod/kustomization.yaml`에 붙여 넣습니다.

   ```yaml
   images:
     - name: ghcr.io/tsungmnli/travel-planner-api
       newTag: "<API 이미지 태그>"
     - name: ghcr.io/tsungmnli/travel-planner-web
       newTag: "<web 이미지 태그>"
   ```

5. `base/cloudflared/deployment.yaml`의 `<고정할 버전>`도 채웁니다.
6. 렌더링이 되는지 로컬에서 확인한 뒤 커밋하고 push합니다.

   ```bash
   kubectl kustomize overlays/prod | grep -E "image:|<"
   # image 줄이 모두 실제 경로와 태그여야 하고, "<" 가 들어간 줄이 없어야 합니다.
   git add -A && git commit -m "deploy: 첫 이미지 태그" && git push
   ```

### 5. ArgoCD가 저장소를 읽을 수 있게 하기

이 저장소가 비공개라면 ArgoCD에 접근 권한(SSH deploy key 등)을 등록해야 합니다. 공개 저장소라면 건너뜁니다.

ArgoCD UI는 외부에 열지 않고 `port-forward`로만 접속합니다.

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# 브라우저에서 https://localhost:8080
```

### 6. Application 등록

```bash
kubectl apply -f argocd/travel-planner-db.yaml     # DB 먼저
kubectl apply -f argocd/travel-planner-prod.yaml
```

처음 한 번만 합니다. 이후에는 Git 커밋만으로 배포됩니다.

| Application | 감시 경로 | 특징 |
| --- | --- | --- |
| `travel-planner-db` | `db` | `prune: false`. DB 리소스는 자동으로 삭제되지 않습니다. |
| `travel-planner-prod` | `overlays/prod` | `prune: true`, `selfHeal: true`. 클러스터에서 손으로 바꿔도 Git 상태로 되돌립니다. |

### 7. 확인

```bash
kubectl -n travel-planner get pods
kubectl -n travel-planner logs deploy/api -c migrate   # 마이그레이션 로그

curl -i https://travel.tsungmn.com/api/healthz   # {"ok":true} 가 나오면 API로 간 것
curl -I https://travel.tsungmn.com/              # Next 페이지가 나와야 함
```

`postgres-0`, `api`, `web`, `cloudflared` 파드가 모두 `Running`이어야 합니다.
API 파드가 처음에 `Init`에서 잠깐 머무는 것은 DB가 준비되기를 기다리는 정상 동작입니다.

---

## 평소 배포하기

배포는 **이 저장소의 이미지 태그를 바꾸는 커밋**입니다.

```
앱 저장소(API/web)에 push
  → CI가 이미지를 빌드해 :<커밋 SHA> 로 GHCR에 push
  → CI가 이 저장소의 overlays/prod/kustomization.yaml 태그를 바꾸는 커밋을 생성
  → ArgoCD가 변경을 감지해 배포
```

ArgoCD는 기본적으로 몇 분 주기로 저장소를 확인하므로 반영까지 시간이 걸릴 수 있습니다.
빨리 반영하려면 ArgoCD UI에서 `travel-planner-prod`의 **Refresh → Sync**를 누르세요.

### CI 없이 손으로 배포하기

CI가 고장 났거나 아직 없을 때의 방법입니다.

1. 앱 저장소에서 이미지를 빌드해 GHCR에 올립니다. (CI를 수동 실행하는 것이 가장 쉽습니다.)
2. `overlays/prod/kustomization.yaml`의 해당 `newTag`를 새 태그로 바꿉니다.
3. 커밋하고 push합니다.

```bash
git commit -am "deploy: api <태그 앞 7자>" && git push
```

`latest` 태그나 `imagePullPolicy: Always`는 쓰지 않습니다.
이미지가 바뀌어도 Git에 기록이 남지 않고, 롤백도 할 수 없기 때문입니다.

### 설정 값 바꾸기

`overlays/prod/kustomization.yaml`의 `configMapGenerator` 값을 고치고 커밋합니다.
값이 바뀌면 ConfigMap 이름의 해시가 바뀌어 파드가 자동으로 재시작됩니다.

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

주의할 점이 하나 있습니다. DB 마이그레이션은 API가 시작할 때 자동으로 실행되고, **되돌려지지 않습니다.**
이전 버전의 API가 새 스키마에서 동작하는지는 마이그레이션 내용에 달려 있으니, 스키마를 바꾼 배포를 롤백할 때는 마이그레이션 코드를 확인하세요.

---

## DB 백업과 복원

### 백업 현황

`postgres-backup` CronJob이 매일 04:00(KST)에 `pg_dump`를 실행해 `postgres-backup` PVC에 `travel-<날짜>-<시각>.sql.gz`로 저장하고, 14일이 지난 파일을 지웁니다.

지금 바로 백업을 한 번 만들려면 이렇게 합니다.

```bash
kubectl -n travel-planner create job --from=cronjob/postgres-backup manual-backup-$(date +%s)
kubectl -n travel-planner get jobs
```

**백업 PVC는 DB와 같은 서버에 있습니다.** 서버 디스크가 고장 나면 DB와 백업을 함께 잃습니다.
중요한 시점(큰 변경 전, 서버 이전 전)에는 덤프를 다른 곳으로 꺼내 두세요.

```bash
# 덤프를 로컬 PC로 바로 받기
kubectl -n travel-planner exec postgres-0 -- \
  sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" | gzip' > travel-$(date +%F).sql.gz
```

### 복원

1. ArgoCD UI에서 `travel-planner-prod`의 **Auto-Sync를 끕니다.** (`selfHeal` 때문에 켜 두면 아래에서 줄인 replicas가 곧바로 되돌아옵니다.)
2. API를 멈춥니다.

   ```bash
   kubectl -n travel-planner scale deploy/api --replicas=0
   ```

3. 복원할 덤프 파일을 고릅니다. 백업 PVC의 파일 목록은 임시 파드로 봅니다.

   ```bash
   kubectl -n travel-planner run backup-shell --rm -it --restart=Never --image=postgres:17 \
     --overrides='{"spec":{"containers":[{"name":"backup-shell","image":"postgres:17","command":["sh"],"stdin":true,"tty":true,"volumeMounts":[{"name":"b","mountPath":"/backup"}]}],"volumes":[{"name":"b","persistentVolumeClaim":{"claimName":"postgres-backup"}}]}}' \
     -- sh
   # 파드 안에서: ls -l /backup
   ```

4. 덤프를 DB에 넣습니다. 기존 데이터를 지우고 넣는 방법입니다.

   ```bash
   # (3번의 임시 파드 안이 아니라, 덤프를 로컬 PC에 받아 둔 경우)
   kubectl -n travel-planner exec -i postgres-0 -- \
     sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"'
   gunzip -c travel-<날짜>.sql.gz | kubectl -n travel-planner exec -i postgres-0 -- \
     sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
   ```

   백업 PVC 안의 파일을 쓰려면 3번의 임시 파드에서 `kubectl cp`로 로컬 PC에 꺼낸 뒤 위 명령을 쓰는 것이 가장 단순합니다.

5. Auto-Sync를 다시 켜면 API가 원래 replicas로 돌아옵니다.
   그러면 `initContainer`의 마이그레이션이 실행되어 스키마가 최신이 됩니다.

복원은 미리 해 본 적이 없으면 실제 사고 때 실패하기 쉽습니다. 서비스가 안정되면 한 번 연습해 두는 것을 권장합니다.

---

## 설정 결정 이유

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

---

## 오래 지난 뒤 점검할 것

시간이 지나면 이 저장소의 값이 낡습니다. 오랜만에 다시 배포한다면 아래 항목을 점검해 주세요.

| 항목 | 점검 방법 |
| --- | --- |
| PostgreSQL 버전 | `postgres:17`은 영구히 지원되지 않습니다. 지원 종료 시점을 확인하세요. 메이저 버전은 이미지 태그만 바꿔서 올릴 수 없고, **덤프를 받아 새 버전의 빈 DB에 복원**해야 합니다. |
| cloudflared 버전 | 너무 오래된 버전은 Cloudflare가 연결을 거부할 수 있습니다. 릴리스 페이지에서 새 안정 버전으로 올립니다. |
| Kubernetes, ArgoCD | 클러스터와 ArgoCD를 올린 뒤에도 `kubectl kustomize overlays/prod`가 오류 없이 나오는지, 사용 중인 API 버전(`apps/v1`, `batch/v1` 등)이 아직 지원되는지 확인합니다. |
| 베이스 이미지의 보안 업데이트 | 앱 저장소의 Dockerfile을 주기적으로 다시 빌드해 새 태그로 배포합니다. |
| GHCR 접근 토큰 | `ghcr-pull`의 토큰이 만료되면 새 파드가 이미지를 받지 못합니다. 만료일을 확인하세요. |
| CI의 config 저장소 push 토큰 | 만료되면 태그 갱신 커밋이 실패합니다. |
| 도메인 | `tsungmn.com` 갱신일 |
| Cloudflare 대시보드 | 메뉴 이름과 입력 형식이 바뀌었을 수 있습니다. |

---

## 서브도메인을 바꿔야 한다면

서브도메인은 여러 곳에 박혀 있어서 나중에 바꾸면 이미 만든 공유 링크가 깨지고 로그인 쿠키도 사라집니다.
바꿔야 한다면 아래 세 곳을 함께 수정하세요.

1. `overlays/prod/kustomization.yaml`의 `PUBLIC_URL`
2. Cloudflare Tunnel의 Public Hostname (경로 두 개 모두)
3. Google OAuth prod 클라이언트의 redirect URI (`https://<새 호스트>/api/auth/google/callback`)

---

## dev와 prod의 구분

| 장치 | 효과 |
| --- | --- |
| `APP_ENV=prod`는 ConfigMap에서만 설정, 코드에서 검증 | prod에서 localhost나 HTTP, docs 공개 설정이면 서버가 시작되지 않습니다. |
| dev는 로컬 DB 호스트(`localhost`, `db` 등)만 허용 | dev가 prod DB(`postgres` 서비스)에 붙으면 시작에 실패합니다. |
| dev는 클러스터에 없음, namespace와 Secret 분리 | dev 데이터와 prod 데이터가 물리적으로 섞일 수 없습니다. |
| Google OAuth, Geoapify 키를 환경별로 발급 | 한쪽 키가 유출되어도 다른 쪽은 안전합니다. |
| 이미지는 SHA 태그 고정, 환경별 값은 overlay에만 | prod가 어떤 코드와 설정으로 떠 있는지 Git 기록으로 추적됩니다. |

web의 mock은 web 저장소에서 담당합니다. prod 빌드에서 mock이 켜져 있으면 빌드가 실패하고, dev에서만 동적으로 불러옵니다.

---

## 문제 해결

| 증상 | 확인할 것 |
| --- | --- |
| `/api/healthz`가 Next 페이지로 응답함 | Cloudflare Public Hostname의 `/api` 규칙이 위에 있는지, 경로 형식이 맞는지 |
| 파드가 `ImagePullBackOff` | 이미지 경로의 계정(`tsungmnli`), 태그가 GHCR에 실제로 있는지, `ghcr-pull` Secret과 토큰 만료 |
| 태그에 `<` 가 들어 있다는 오류 | `overlays/prod/kustomization.yaml`의 자리표시자를 안 채움 |
| API 파드가 `Init:Error` 또는 `Init:CrashLoopBackOff` | `kubectl -n travel-planner logs deploy/api -c migrate`, `api-secret`의 `DATABASE_URL`, DB 파드 상태 |
| API가 DB 인증에 실패함 | `postgres-secret`을 새로 만들면서 **처음과 다른 비밀번호**를 넣었을 수 있습니다. DB는 처음 비밀번호를 기억합니다. |
| API 파드가 시작하지 않음 | prod 검증 값(`APP_ENV`, `PUBLIC_URL`, `COOKIE_SECURE`, `DOCS_ENABLED`) |
| cloudflared가 계속 재시작됨 | `cloudflared-secret`의 토큰, 이미지 버전 자리표시자를 채웠는지 |
| 사이트가 Cloudflare 오류 화면(1033 등)을 보여 줌 | cloudflared 파드가 떠 있는지, 터널이 대시보드에서 Healthy인지 |
| ArgoCD가 변경을 반영하지 않음 | 저장소 접근 권한, Application 상태(OutOfSync / Degraded), Refresh 해 보기 |
| 로그인이 안 됨 | Google OAuth redirect URI가 `PUBLIC_URL`과 정확히 일치하는지 |
| CI가 이미지는 빌드했는데 배포가 안 됨 | CI가 이 저장소에 push할 토큰이 만료되었는지 |