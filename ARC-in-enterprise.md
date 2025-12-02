
---

> 당신의 GitHub Actions Self-Hosted Runner 운영 전문가 입니다. 특히 ARC(Actions Runner Controller)를 자유자재로 적용할 수 있습니다. 우리 회사에서는 GitHub Enterprise Server 3.14.17 버전을 서비스 하고 있고, 외부 github.com 과 마찬가지로 Github-hosted runner에 대한 요구사항이 많이 있습니다. 수많은 Organization에서 runner 사용을 기대하고 있는데, 각각의 Organization 별로 최대 동시에 실행되는 workflows 수를 10개 이하로 설정하고 싶습니다. 현재 사내 자체 Cloud에서 Kubernetes cluster를 활용할 수 있고, 또는 VM 형태로 서버 자원을 받아 직접 설정할 수도 있습니다. 저는 Kubernetes에 대해 어느정도 지식이 있지만, 부서원들은 초심자입니다. kubernetes를 통해 self-hosted runner를 운영한다면 최소한의 노력으로 쉽게 설정할 수 있는 방법이 필요합니다. 또한 docker-in-docker 형태로 privileged 모드로 동작을 해야 합니다. 모든 organization에서 동일한 라벨인 code-linux 를 사용하기를 원합니다. 즉, workflows 실행 요청이 들어오면 실시간으로 새로운 runner를 해당 organization에 등록해서 수행할 수도 있고, 미리 다수의 runner 풀을 online 상태로 놓고 각각의 organization에서 취해갈 수도 있을 것 같습니다.최대한 우리 조직에 적합한 architecture를 제시해 주고, 설치 및 설정 가이드를 제공해 주세요. README.md 파일 등으로 작성해서 부서원들이 바로 참고할 수 있도록 하고 싶습니다.



# ARC Self-Hosted Runner 운영 가이드

GitHub Enterprise Server 3.14.17 환경에서 ARC(Actions Runner Controller)를 사용하여 Organization별 Self-Hosted Runner를 운영하기 위한 가이드입니다.

---

## 목차

1. [아키텍처 개요](#아키텍처-개요)
2. [사전 요구사항](#사전-요구사항)
3. [PAT 토큰 생성](#pat-토큰-생성)
4. [ARC Controller 설치](#arc-controller-설치)
5. [Runner Scale Set 배포](#runner-scale-set-배포)
6. [자동화 스크립트 사용법](#자동화-스크립트-사용법)
7. [운영 및 모니터링](#운영-및-모니터링)
8. [트러블슈팅](#트러블슈팅)
9. [FAQ](#faq)

---

## 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌───────────────────────────────────────────────────────────┐    │
│   │           gha-runner-scale-set-controller                  │    │
│   │           (arc-systems namespace)                          │    │
│   │           - 전체 Runner Scale Set 관리                     │    │
│   │           - Webhook 수신 및 Runner 자동 스케일링            │    │
│   └───────────────────────────────────────────────────────────┘    │
│                              │                                      │
│              ┌───────────────┼───────────────┐                     │
│              ▼               ▼               ▼                     │
│   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐     │
│   │ org-alpha       │ │ org-beta        │ │ org-gamma       │     │
│   │ Runner Scale Set│ │ Runner Scale Set│ │ Runner Scale Set│ x50+│
│   ├─────────────────┤ ├─────────────────┤ ├─────────────────┤     │
│   │ maxRunners: 10  │ │ maxRunners: 10  │ │ maxRunners: 10  │     │
│   │ minRunners: 0   │ │ minRunners: 0   │ │ minRunners: 0   │     │
│   │ label:code-linux│ │ label:code-linux│ │ label:code-linux│     │
│   │ DinD: privileged│ │ DinD: privileged│ │ DinD: privileged│     │
│   └─────────────────┘ └─────────────────┘ └─────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  GitHub Enterprise Server     │
                    │  3.14.17                      │
                    │  https://github.example.com   │
                    └───────────────────────────────┘
```

### 핵심 특징

| 항목 | 설정값 | 설명 |
|------|--------|------|
| Runner 라벨 | `code-linux` | 모든 Organization에서 동일한 라벨 사용 |
| 최대 동시 실행 | 10개/Org | Organization별 maxRunners 설정 |
| 컨테이너 모드 | DinD (Docker-in-Docker) | privileged 모드로 Docker 빌드 지원 |
| 스케일링 | 0 ~ 10 | 요청 없을 시 0으로 축소, 최대 10개까지 확장 |

---

## 사전 요구사항

### 필수 도구

| 도구 | 최소 버전 | 설치 확인 명령어 |
|------|----------|-----------------|
| kubectl | 1.24+ | `kubectl version --client` |
| Helm | 3.8+ | `helm version` |
| bash | 4.0+ | `bash --version` |

### Kubernetes 클러스터 요구사항

- Kubernetes 버전: 1.24 이상
- 네임스페이스 생성 권한
- Secret 및 ConfigMap 생성 권한
- RBAC 설정 권한

### 네트워크 요구사항

- Kubernetes 클러스터에서 GitHub Enterprise Server로의 HTTPS 접근 (포트 443)
- GitHub Enterprise Server에서 Kubernetes 클러스터로의 Webhook 접근 (선택사항, Webhook 모드 사용 시)

---

## PAT 토큰 생성

### 필요한 권한 (Scopes)

GitHub Enterprise Server에서 Personal Access Token을 생성할 때 다음 권한이 필요합니다:

| Scope | 설명 |
|-------|------|
| `admin:org` | Organization runner 등록/삭제 권한 |
| `repo` | Repository 접근 권한 (private repo workflow 실행 시 필요) |

### 토큰 생성 단계

1. GitHub Enterprise Server 웹 UI 접속
2. 우측 상단 프로필 클릭 → **Settings**
3. 좌측 메뉴 하단 **Developer settings** 클릭
4. **Personal access tokens** → **Tokens (classic)** 선택
5. **Generate new token (classic)** 클릭
6. 다음 설정 입력:
   - **Note**: `ARC Runner Token` (식별용 이름)
   - **Expiration**: 조직 정책에 따라 설정 (권장: 90일 이후 갱신)
   - **Select scopes**: `repo`, `admin:org` 체크
7. **Generate token** 클릭
8. 생성된 토큰을 안전한 곳에 저장 (다시 확인 불가)

> **주의**: 토큰은 생성 직후에만 확인 가능합니다. 반드시 안전한 곳에 저장하세요.

---

## ARC Controller 설치

### 1. Helm Repository 추가

```bash
# GitHub Actions Runner Controller Helm repository 추가
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller

# Repository 업데이트
helm repo update
```

### 2. Controller 네임스페이스 생성

```bash
kubectl create namespace arc-systems
```

### 3. Controller 설치

```bash
# values 파일을 사용하여 설치
helm install arc \
  --namespace arc-systems \
  -f values/controller-values.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

### 4. 설치 확인

```bash
# Controller Pod 상태 확인
kubectl get pods -n arc-systems

# 예상 출력:
# NAME                                     READY   STATUS    RESTARTS   AGE
# arc-gha-runner-scale-set-controller-xxx  1/1     Running   0          1m
```

### Controller 업그레이드

새 버전으로 업그레이드가 필요한 경우:

```bash
helm repo update

helm upgrade arc \
  --namespace arc-systems \
  -f values/controller-values.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

---

## Runner Scale Set 배포

### 수동 배포 (단일 Organization)

#### 1. Runner 네임스페이스 생성

```bash
kubectl create namespace arc-runners
```

#### 2. PAT Secret 생성

```bash
kubectl create secret generic github-pat \
  --namespace arc-runners \
  --from-literal=github_token='YOUR_PAT_TOKEN_HERE'
```

#### 3. Runner Scale Set 설치

```bash
# ORG_NAME을 실제 Organization 이름으로 변경
ORG_NAME="your-org-name"

helm install arc-runner-${ORG_NAME} \
  --namespace arc-runners \
  --set githubConfigUrl="https://github.example.com/${ORG_NAME}" \
  --set githubConfigSecret="github-pat" \
  -f values/runner-scale-set-template.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

### 배포 확인

```bash
# Runner Scale Set 상태 확인
kubectl get pods -n arc-runners

# AutoscalingRunnerSet 리소스 확인
kubectl get autoscalingrunnersets -n arc-runners
```

---

## 자동화 스크립트 사용법

50개 이상의 Organization을 관리해야 하므로 자동화 스크립트를 제공합니다.

### 파일 구조

```
arc-runners/
├── README.md                          # 본 가이드
├── values/
│   ├── controller-values.yaml         # Controller 설정
│   └── runner-scale-set-template.yaml # Runner 템플릿
└── scripts/
    ├── organizations.txt              # Organization 목록
    ├── deploy-runners.sh              # 일괄 배포 스크립트
    └── cleanup-runners.sh             # 일괄 삭제 스크립트
```

### Organization 목록 설정

`scripts/organizations.txt` 파일에 배포할 Organization 이름을 한 줄에 하나씩 입력합니다:

```
org-alpha
org-beta
org-gamma
# 주석은 #으로 시작
# 빈 줄은 무시됩니다
```

### 일괄 배포

```bash
# 스크립트 실행 권한 부여
chmod +x scripts/deploy-runners.sh

# 환경 변수 설정 및 배포 실행
export GITHUB_PAT="your-pat-token-here"
export GHES_URL="https://github.example.com"

./scripts/deploy-runners.sh
```

### 일괄 삭제

```bash
# 스크립트 실행 권한 부여
chmod +x scripts/cleanup-runners.sh

# 모든 Runner Scale Set 삭제
./scripts/cleanup-runners.sh
```

### 특정 Organization만 배포/삭제

```bash
# 특정 Organization만 배포
./scripts/deploy-runners.sh org-alpha org-beta

# 특정 Organization만 삭제
./scripts/cleanup-runners.sh org-alpha org-beta
```

---

## 운영 및 모니터링

### 기본 모니터링 명령어

```bash
# 모든 Runner Pod 상태 확인
kubectl get pods -n arc-runners -o wide

# 특정 Organization Runner 확인
kubectl get pods -n arc-runners -l "actions.github.com/organization=org-alpha"

# AutoscalingRunnerSet 상태 확인
kubectl get autoscalingrunnersets -n arc-runners

# Runner 상세 정보 확인
kubectl describe autoscalingrunnersets -n arc-runners
```

### 로그 확인

```bash
# Controller 로그 확인
kubectl logs -n arc-systems -l app.kubernetes.io/name=gha-runner-scale-set-controller -f

# 특정 Runner Pod 로그 확인
kubectl logs -n arc-runners <pod-name> -c runner
```

### 리소스 사용량 확인

```bash
# Node 리소스 사용량
kubectl top nodes

# Runner Pod 리소스 사용량
kubectl top pods -n arc-runners
```

### GitHub Enterprise Server에서 Runner 상태 확인

1. GitHub Enterprise Server 웹 UI 접속
2. Organization 페이지 이동
3. **Settings** → **Actions** → **Runners** 메뉴
4. 등록된 Runner 목록 및 상태 확인

---

## 트러블슈팅

### 문제 1: Runner Pod가 Pending 상태로 유지됨

**원인**: 클러스터 리소스 부족 또는 노드 스케줄링 문제

**해결 방법**:
```bash
# Pod 상태 및 이벤트 확인
kubectl describe pod <pod-name> -n arc-runners

# 노드 리소스 확인
kubectl describe nodes
```

### 문제 2: Runner가 GitHub에 등록되지 않음

**원인**: PAT 토큰 권한 부족 또는 네트워크 문제

**해결 방법**:
```bash
# Secret 확인
kubectl get secret github-pat -n arc-runners -o yaml

# Controller 로그에서 에러 확인
kubectl logs -n arc-systems -l app.kubernetes.io/name=gha-runner-scale-set-controller | grep -i error
```

### 문제 3: Docker 빌드가 실패함 (DinD 문제)

**원인**: privileged 모드가 적용되지 않았거나 Docker 소켓 문제

**해결 방법**:
```bash
# Runner Pod의 securityContext 확인
kubectl get pod <pod-name> -n arc-runners -o yaml | grep -A5 securityContext

# DinD 사이드카 컨테이너 로그 확인
kubectl logs <pod-name> -n arc-runners -c dind
```

### 문제 4: Workflow가 "Queued" 상태로 대기 중

**원인**: 사용 가능한 Runner가 없거나 라벨 불일치

**해결 방법**:
1. Runner 라벨이 `code-linux`인지 확인
2. maxRunners 제한에 도달했는지 확인
3. Runner가 해당 Organization에 등록되어 있는지 확인

```bash
# AutoscalingRunnerSet 상태 확인
kubectl get autoscalingrunnersets -n arc-runners -o wide
```

### 문제 5: PAT 토큰 만료

**원인**: Personal Access Token 유효기간 만료

**해결 방법**:
```bash
# 새 토큰으로 Secret 업데이트
kubectl delete secret github-pat -n arc-runners
kubectl create secret generic github-pat \
  --namespace arc-runners \
  --from-literal=github_token='NEW_PAT_TOKEN_HERE'

# Runner Scale Set 재시작 (Pod 삭제하면 자동 재생성)
kubectl delete pods -n arc-runners --all
```

---

## FAQ

### Q1: Runner 수를 Organization별로 다르게 설정할 수 있나요?

**A**: 네, 각 Organization별로 별도의 values 파일을 만들어 `maxRunners` 값을 다르게 설정할 수 있습니다.

```bash
# 예: org-alpha는 20개, org-beta는 5개
helm upgrade arc-runner-org-alpha ... --set maxRunners=20
helm upgrade arc-runner-org-beta ... --set maxRunners=5
```

### Q2: 특정 Repository에만 Runner를 할당할 수 있나요?

**A**: Repository 레벨 Runner도 지원됩니다. `githubConfigUrl`을 Repository URL로 변경하면 됩니다:

```bash
--set githubConfigUrl="https://github.example.com/org-name/repo-name"
```

### Q3: Runner 이미지를 커스텀할 수 있나요?

**A**: 네, `values/runner-scale-set-template.yaml`에서 `template.spec.containers[].image`를 변경하여 커스텀 이미지를 사용할 수 있습니다.

### Q4: minRunners를 0이 아닌 값으로 설정하면 어떻게 되나요?

**A**: `minRunners`를 1 이상으로 설정하면 항상 해당 수만큼의 Runner가 대기 상태로 유지됩니다. Workflow 시작 시간이 빨라지지만 리소스 비용이 증가합니다.

### Q5: 여러 라벨을 사용할 수 있나요?

**A**: 네, `runnerScaleSetName`과 별도로 추가 라벨을 설정할 수 있습니다. `values/runner-scale-set-template.yaml`의 `labels` 섹션을 참조하세요.

---

## 버전 정보

| 구성 요소 | 버전 |
|----------|------|
| GitHub Enterprise Server | 3.14.17 |
| ARC (gha-runner-scale-set-controller) | 0.9.x |
| ARC (gha-runner-scale-set) | 0.9.x |

---

## 참고 자료

- [GitHub Actions Runner Controller 공식 문서](https://github.com/actions/actions-runner-controller)
- [ARC Helm Chart](https://github.com/actions/actions-runner-controller/tree/master/charts)
- [GitHub Enterprise Server Self-Hosted Runner 문서](https://docs.github.com/en/enterprise-server/actions/hosting-your-own-runners)




### 파일 구조

arc-runners/
├── README.md                              # 종합 운영 가이드
├── values/
│   ├── controller-values.yaml             # ARC Controller 설정
│   └── runner-scale-set-template.yaml     # Runner Scale Set 템플릿 (DinD 포함)
└── scripts/
    ├── organizations.txt                  # Organization 목록 샘플
    ├── deploy-runners.sh                  # 일괄 배포 스크립트
    └── cleanup-runners.sh                 # 일괄 삭제 스크립트


### 생성된 파일 요약

파일	설명
README.md | 아키텍처, 사전 요구사항, PAT 생성, ARC Controller 설치, Runner 배포, 모니터링, 트러블슈팅 등 전체 가이드
values/controller-values.yaml | ARC Controller Helm 설정 (리소스, 보안, 로깅 등)
values/runner-scale-set-template.yaml | DinD privileged 모드, code-linux 라벨, maxRunners: 10 설정 포함
scripts/deploy-runners.sh | 50개 이상 Organization 일괄 배포 자동화
scripts/organizations.txt | 배포할 Organization 목록 (샘플 포함)
scripts/cleanup-runners.sh | Runner Scale Set 일괄 삭제


### 빠른 시작 가이드

부서원들이 바로 사용할 수 있도록 간단한 순서를 안내드립니다:

```
# 1. ARC Controller 설치 (한 번만)helm install arc \
  --namespace arc-systems \
  --create-namespace
 \  -f values/controller-values.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

# 2. Organization 목록 편집
vi scripts/organizations.txt

# 3. 환경 변수 설정 및 일괄 배포
export GITHUB_PAT="your-pat-token"
export GHES_URL="https://github.example.com"
chmod +x scripts/deploy-runners.sh
./scripts/deploy-runners.sh
```

모든 Organization에서 workflow 파일에 runs-on: code-linux를 사용하면 자동으로 Runner가 할당됩니다.





---

> 관리하고 있는 Organization이 1000개 정도 돼. 하지만 서버 자원은 한정적이라 모든 Organization에서 요청되는 workflows를 동시에 처리할 수 없어. 즉, 미리 runner를 각 Organization에 등록해 둘 수 없다는 얘기야. 이러한 경우도 구성한 ARC 구조의 설계로 지원 가능한가요?



좋은 질문입니다. 솔직하게 말씀드리면, 현재 제안한 아키텍처는 1000개 Organization을 모두 지원하기에는 적합하지 않습니다.

### 현재 아키텍처의 한계

각 Organization별로 Runner Scale Set을 배포하면:

항목 | 1000개 Org 기준
Helm Release 수 | 1,000개
Listener Pod 수 | 1,000개 (minRunners=0이어도 필요)
예상 메모리 | ~128MB × 1,000 = ~128GB (Listener만)
예상 CPU | ~50m × 1,000 = ~50 cores (Listener만)

minRunners: 0으로 설정해도 각 Runner Scale Set은 GitHub와 통신하는 Listener Pod를 항상 유지해야 합니다. 이것만으로도 상당한 리소스가 필요합니다.

###  대안 아키텍처 제안

1000개 Organization + 한정된 자원 + Organization별 제한이라는 요구사항을 모두 만족시키려면 Hybrid 접근이 필요합니다:

#### 옵션 1: Enterprise 레벨 Runner (권장)

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
├─────────────────────────────────────────────────────────────┤
│   ┌─────────────────────────────────────────────────────┐   │
│   │     gha-runner-scale-set-controller                  │   │
│   └─────────────────────────────────────────────────────┘   │
│                            │                                 │
│                            ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │         Enterprise Level Runner Scale Set            │   │
│   │         - maxRunners: 100 (전체 동시 실행 제한)       │   │
│   │         - minRunners: 0                              │   │
│   │         - label: code-linux                          │   │
│   │         - Listener Pod: 1개만 필요                   │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
              GitHub Enterprise Server (Enterprise 레벨 등록)
              └── 1000개 Organization에서 공유 사용
```

- 장점:
  - Listener Pod 1개만 필요
  - 리소스 효율적
  - 관리 단순
- 단점:
  - Organization별 10개 제한 불가 (전체 Enterprise 레벨 제한만 가능)

#### 옵션 2: Active Organization만 동적 관리

자주 사용하는 Organization만 Runner Scale Set을 유지하고, 나머지는 요청 시 동적으로 생성/삭제:

```
활성 Organization (Top 50~100개)
  └── Runner Scale Set 상시 유지
  └── Organization별 10개 제한 적용비활성 Organization (나머지 900+개)
  └── Enterprise 레벨 Runner 사용
  └── 또는 요청 시 동적 생성 (Webhook 기반)
```

#### 옵션 3: GitHub Actions의 Concurrency 기능 활용

Organization별 runner 제한 대신, Workflow 레벨에서 concurrency를 제한:

```
# 각 Repository의 workflow 파일에 추가
concurrency:
  group: ${{ github.repository }}-${{ github.workflow }}
  cancel-in-progress: false
```

또는 Organization 정책으로 workflow concurrency 템플릿 강제:

```
# Organization 레벨 concurrency 제한
concurrency:
  group: org-${{ github.repository_owner }}-workflow
  max-parallel: 10  # GHES 3.14에서 지원 여부 확인 필요
```

### 추천 방향

1000개 Organization 환경에서는 옵션 1 (Enterprise 레벨 Runner)을 권장드립니다:

비교 항목 | 현재 제안 (Org별) | Enterprise 레벨
Listener Pod | 1,000개 | 1개
Helm Release | 1,000개 | 1개
관리 복잡도 | 높음 | 낮음
Org별 10개 제한 | 가능 | 불가 (전체 제한만)
리소스 효율 | 낮음 | 높음




---

> Organization 별 최대 10개 제한은 꼭 필요해. 직접 Enterprise webhook을 받아서 처리할 수 있는 기술력은 가지고 있어. 예를들어 celery + message queue + redis db + worker 등등의 기술 스택은 보유하고 있어. 꼭 ARC를 사용하진 않아도 되지만 가능하다면 사용하고 싶어. 1번은 제외해야 할 것 같고, 현재 가용한 node는 20개 수준이야. 즉, 업무시간에는 대기는 무조건 발생할거고, 들어온 순서대로 순차적으로 처리가 되면 좋겠어. 단, 특정 Org에 10개가 넘으면 해당 org의 추가 요청은 잠시 홀드하고, 다른 Org에서 먼저 들어온 요청을 처리하면 돼. 이것은 아마 GitHub 자체적으로 스케쥴링을 하니깐 특별히 고려할 필요는 없을것 같아.



이해했습니다. 요구사항을 정리하면:
- 1000개 Organization, Organization별 10개 제한 (필수)
- 가용 노드 20개 (전체 동시 실행 ~200개 정도로 제한)
- Celery + Redis + Worker 기술 스택 보유
- ARC 사용 가능하면 사용, 아니면 직접 구현도 가능

### 제안: Custom JIT Runner Manager + ARC Hybrid
ARC의 Runner Scale Set 방식 대신, JIT(Just-In-Time) Runner를 직접 관리하는 방식을 제안합니다.

#### 아키텍처

```
GitHub Enterprise Server
        │
        │ Enterprise Webhook (workflow_job 이벤트)
        ▼
┌─────────────────────────────────────────────────────────────┐
│              Webhook Receiver (FastAPI/Flask)                │
│  - workflow_job.queued → Redis 체크 → Celery task          │
│  - workflow_job.completed → Runner 정리                     │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                         Redis                                │
│  - org:{org_name}:running_count → 현재 실행 중 runner 수    │
│  - org:{org_name}:pending_jobs → 대기 중인 job ID 목록      │
│  - global:total_running → 전체 실행 중 runner 수            │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                    Celery Workers                            │
│  Task 1: process_workflow_job                               │
│    - Org별 10개 제한 확인                                    │
│    - 전체 200개 제한 확인                                    │
│    - GitHub API로 JIT runner token 발급                     │
│    - Kubernetes에 Runner Pod 생성                           │
│                                                              │
│  Task 2: cleanup_runner                                      │
│    - 완료된 runner Pod 삭제                                  │
│    - Redis 카운터 감소                                       │
│    - 대기 중인 job 재처리                                    │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Runner Pods (동적 생성/삭제)                        │    │
│  │  - 최대 200개 동시 실행 (20 nodes × 10 pods/node)   │    │
│  │  - DinD privileged 모드                              │    │
│  │  - Ephemeral (1회 실행 후 삭제)                      │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

#### 핵심 흐름

```
1. workflow_job.queued 이벤트 수신
        │
        ▼
2. Redis에서 org:{org_name}:running_count 조회
        │
        ├── < 10 이고 전체 < 200 ──→ 3. JIT Runner 생성
        │                                   │
        │                                   ▼
        │                            4. GitHub API로 runner token 발급
        │                                   │
        │                                   ▼
        │                            5. K8s Runner Pod 생성
        │                                   │
        │                                   ▼
        │                            6. Redis 카운터 증가
        │
        └── >= 10 또는 전체 >= 200 ──→ GitHub이 자동으로 대기열 유지
                                        (아무것도 하지 않음)

7. workflow_job.completed 이벤트 수신
        │
        ▼
8. Redis 카운터 감소 + Pod 삭제
```

#### 장점

항목	ARC (Org별 Scale Set)	JIT Runner Manager
Listener Pod	1,000개 필요	0개
메모리 오버헤드	~128GB	~2GB (Webhook + Celery)
Org별 10개 제한	가능	가능
1000개 Org 지원	어려움	쉬움
관리 복잡도	Helm 1000개	코드 관리


#### ARC 활용 가능 부분
ARC를 완전히 버리지 않고, Runner Pod 템플릿만 참고하여 DinD 설정을 재사용할 수 있습니다:

```
# Runner Pod 템플릿 (직접 생성)
apiVersion: v1
kind: Pod
metadata:
  name: runner-{job_id}
  labels:
    app: github-runner
    org: {org_name}
spec:
  restartPolicy: Never
  containers:
    - name: runner
      image: ghcr.io/actions/actions-runner:latest
      env:
        - name: RUNNER_TOKEN
          value: "{jit_token}"
        - name: RUNNER_URL
          value: "https://github.example.com/{org_name}"
        - name: DOCKER_HOST
          value: "unix:///var/run/docker.sock"
      volumeMounts:
        - name: dind-sock
          mountPath: /var/run
    - name: dind
      image: docker:dind
      securityContext:
        privileged: true
      volumeMounts:
        - name: dind-sock
          mountPath: /var/run
  volumes:
    - name: dind-sock
      emptyDir: {}
```

#### 진행

- 아키텍처 문서 (README.md 업데이트)
- Webhook Receiver (FastAPI 기반)
- Celery Tasks (runner 생성/정리)
- Kubernetes 연동 (Python kubernetes client)
- Redis 스키마 설계
- Helm Chart (Webhook + Celery + Redis 배포용)
- Enterprise Webhook 설정 가이드





## 생성된 파일 구조

jit-runner-manager/
├── README.md                              # 종합 운영 가이드
├── requirements.txt                       # Python 의존성
├── .env.example                          # 환경 변수 예시
│
├── app/                                   # Python 애플리케이션
│   ├── __init__.py
│   ├── config.py                         # 설정 관리
│   ├── main.py                           # FastAPI 엔트리포인트
│   ├── webhook_handler.py                # Webhook 처리
│   ├── celery_app.py                     # Celery 설정
│   ├── tasks.py                          # Runner 생성/정리 태스크
│   ├── redis_client.py                   # Redis 상태 관리
│   ├── github_client.py                  # GitHub API (JIT token)
│   └── k8s_client.py                     # Kubernetes Pod 관리
│
├── k8s/                                   # Kubernetes 매니페스트
│   ├── namespace.yaml                    # 네임스페이스 정의
│   ├── runner-pod-template.yaml          # Runner Pod 템플릿 (참고용)
│   └── rbac.yaml                         # RBAC 설정
│
├── helm/jit-runner-manager/              #  Helm Chart
│   ├── Chart.yaml
│   ├── values.yaml                       # 설정값 (수정 필요)
│   └── templates/
│       ├── _helpers.tpl
│       ├── configmap.yaml
│       ├── deployment.yaml               # Webhook Receiver
│       ├── deployment-worker.yaml        # Celery Worker + Beat
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── serviceaccount.yaml
│       ├── rbac.yaml
│       └── namespace-runner.yaml
│
├── docker/
│   ├── Dockerfile                        # 컨테이너 이미지
│   └── docker-compose.yaml               # 로컬 개발용
│
└── docs/
    └── enterprise-webhook-setup.md       # Enterprise Webhook 설정 가이드

## 빠른 시작
### 1. Secrets 생성

```
# GitHub PAT
kubectl create secret generic github-credentials \
  --namespace jit-runner-manager \
  --from-literal=pat='YOUR_GITHUB_PAT'

# Webhook Secret
kubectl create secret generic webhook-secret \
  --namespace jit-runner-manager \
  --from-literal=secret='YOUR_WEBHOOK_SECRET'
```

### 2. Helm Chart 설치

```
# values.yaml 수정 (GHES URL, Ingress 등)
vi helm/jit-runner-manager/values.yaml

# 설치
helm install jit-runner-manager ./helm/jit-runner-manager \
  --namespace jit-runner-manager \
  --create-namespace
```

### 3. Enterprise Webhook 설정

```
GitHub Enterprise Server에서 Enterprise 레벨 Webhook을 설정합니다:
Payload URL: https://your-ingress/webhook
Events: Workflow jobs
Secret: 위에서 생성한 값과 동일
자세한 내용은 docs/enterprise-webhook-setup.md를 참조하세요.
```


## 핵심 동작 원리

단계 | 동작
1 | Workflow 시작 → GitHub이 workflow_job.queued 이벤트 전송
2 | FastAPI Webhook Receiver가 수신
3 | Redis에서 Org별 실행 중인 Runner 수 확인 (< 10?)
4 | Celery Worker가 GitHub API로 JIT Runner token 발급
5 | Kubernetes에 Runner Pod (DinD) 생성
6 | Workflow 완료 → workflow_job.completed 이벤트
7 | Runner Pod 삭제 + Redis 카운터 감소

이 아키텍처를 통해 1000개 Organization에서 Org별 10개 제한을 효율적으로 지원할 수 있습니다. Listener Pod 없이 Webhook 기반으로 동작하므로 리소스 오버헤드가 최소화됩니다.
