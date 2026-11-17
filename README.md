# paas_gitops_dind — Jenkins CI/CD 파이프라인

Kubernetes 위의 Jenkins Agent Pod에서 **Spring Boot(Maven) 애플리케이션**을 빌드·검사·이미지화한 뒤, **GitOps 저장소에 매니페스트를 커밋하고 Argo CD로 배포**하는 Scripted Pipeline입니다.

- 이미지 빌드: **DinD(docker-in-docker)** 사이드카 방식 (기존 kaniko 대체)
- 품질/보안 게이트: **SonarQube**(코드), **Trivy**(이미지 취약점), **Polaris**(K8s 매니페스트)
- 배포: **Kustomize** 로 렌더링한 `kubernetes.yaml` → GitOps repo push → **Argo CD** sync
- 사내 CA: Vault에서 발급한 CA(`vault-ca-cert` Secret)를 각 컨테이너에 주입해 사설 TLS(GitLab, Harbor, Argo CD) 통신

> 파일: [paas_gitops_dind.jenkinsfile](paas_gitops_dind.jenkinsfile)

---

## 0. 사용 오픈소스

| 구분 | 오픈소스 | 버전 | 라이선스 |
|---|---|---|---|
| CI Server | [Jenkins](https://www.jenkins.io/) (inbound-agent) | 3345.v03dee9b_f88fc-1-jdk21 | MIT |
| CI Plugin | [Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/) | - | Apache-2.0 |
| CI Plugin | [SonarQube Scanner for Jenkins](https://plugins.jenkins.io/sonar/) | - | LGPL-3.0 |
| SCM | [GitLab](https://about.gitlab.com/) | - | MIT (CE) |
| Build | [Apache Maven](https://maven.apache.org/) | 3.9.5 | Apache-2.0 |
| Runtime | [Eclipse Temurin](https://adoptium.net/) (OpenJDK) | 17 | GPL-2.0 w/ Classpath Exception |
| Image Build | [Docker Engine / Moby](https://mobyproject.org/) (DinD) | 20.10 | Apache-2.0 |
| Registry | [Harbor](https://goharbor.io/) | - | Apache-2.0 |
| Code Quality | [SonarQube](https://www.sonarsource.com/products/sonarqube/) / sonar-scanner-cli | 5.0.1 (scanner) | LGPL-3.0 |
| Image Scan | [Trivy](https://trivy.dev/) | 0.60.0 | Apache-2.0 |
| Manifest | [Kustomize](https://kustomize.io/) | 5.0.1 | Apache-2.0 |
| Manifest Scan | [Polaris](https://polaris.docs.fairwinds.com/) (Fairwinds) | 9.6 | Apache-2.0 |
| CD | [Argo CD](https://argo-cd.readthedocs.io/) (CLI) | 3.1.8 | Apache-2.0 |
| Secret / PKI | [HashiCorp Vault](https://www.vaultproject.io/) (사내 CA 발급) | - | BUSL-1.1 |
| Base Image | [Alpine Linux](https://alpinelinux.org/) (argocli-custom) | 3.20 | MIT 외 |
| Util | curl / jq / git | - | curl / MIT / GPL-2.0 |
| Orchestration | [Kubernetes](https://kubernetes.io/) | - | Apache-2.0 |

> 버전이 `-`인 항목은 파이프라인 밖에서 운영되는 서버/플러그인이라 이 파일로는 버전을 확정할 수 없습니다. Vault는 1.14부터 BUSL-1.1이며, 그 이전 버전은 MPL-2.0입니다.

---

## 0-1. 엔드포인트 (URL 표준)

모든 외부 주소는 Jenkinsfile **최상단 상수 블록**에서 한 번만 정의하고, `Print env` 스테이지에서 같은 이름의 `env.*`로 노출합니다. 환경이 바뀌면 그 블록만 수정합니다.

| 시스템 | `*_HOST` (스킴 없음) | `*_URL` (스킴 포함) | 비고 |
|---|---|---|---|
| 공통 | `DOMAIN` = `example.com` | | |
| GitLab | `GITLAB_HOST` = `gitlab-y.${DOMAIN}` | `GITLAB_URL`, `GITLAB_API_URL`(`/api/v4`) | |
| Harbor | `HARBOR_HOST` = `harbor-y.${DOMAIN}` | `HARBOR_URL` | `HARBOR_LIBRARY` = `${HARBOR_HOST}/library` (도구/베이스 이미지) |
| Argo CD | `ARGOCD_HOST` = `argocd-y.${DOMAIN}` | `ARGOCD_URL` | |
| SonarQube | | `SONARQUBE_URL` = `http://sonarqube-sonarqube.devopsit.svc.cluster.local:9000` | 클러스터 내부 서비스 |
| 배포 클러스터 | | `K8S_API_URL` = `https://k8s-api.example.com:6443` | Argo CD `--dest-server` |

- `*_HOST`는 `git clone https://<id>:<pw>@<HOST>/...`, `docker login <HOST>`, `argocd --server <HOST>`, `certs.d/<HOST>` 등 스킴이 붙으면 안 되는 곳에 씁니다.
- `*_URL`은 브라우저/API 호출처럼 스킴이 필요한 곳에 씁니다.
- 저장소 변수(`SOURCE_REPO`, `GITOPS_REPO`, `MANIFEST_REPO`)는 `"${GITLAB_HOST}/<group>/<repo>.git"` 형식입니다.

---

## 1. 전체 아키텍처

```mermaid
flowchart LR
    subgraph DEV["개발자 / Devopsit"]
        U["Devopsit<br/>(Job 파라미터 전달)"]
    end

    subgraph K8S["Kubernetes - Jenkins Agent Pod (label: jenkinsBuildPipline)"]
        direction TB
        JNLP["jnlp<br/>(git clone, Dockerfile 생성)"]
        MVN["maven<br/>(compile / package)"]
        SQ["sonarqube<br/>(sonar-scanner)"]
        KZ["kustomize<br/>(overlay build)"]
        DCK["docker CLI<br/>(build / push / save)"]
        DIND["dind (privileged)<br/>dockerd :2375"]
        TV["trivy<br/>(offline image scan)"]
        PL["polaris<br/>(manifest audit)"]
        ARGO["argocli<br/>(git push, argocd CLI)"]
        DCK -- "tcp://localhost:2375" --> DIND
    end

    subgraph EXT["사내 플랫폼 (*.example.com)"]
        GL_SRC[("GitLab<br/>소스 repo")]
        GL_GO[("GitLab<br/>GitOps repo<br/>(kustomize base/overlays)")]
        GL_MF[("GitLab<br/>Manifest repo<br/>(배포용 kubernetes.yaml)")]
        HARBOR[("Harbor<br/>Image Registry")]
        SONAR["SonarQube Server<br/>(devopsit ns)"]
        ARGOCD["Argo CD"]
    end

    TARGET["대상 클러스터<br/>K8S_API_URL<br/>ns: sample-dev"]

    U --> JNLP
    GL_SRC -- clone --> JNLP
    GL_GO -- clone --> JNLP
    SQ -- 분석 결과 --> SONAR
    DIND -- push --> HARBOR
    ARGO -- "commit + tag push" --> GL_MF
    ARGO -- "repo/proj/app 등록, sync" --> ARGOCD
    ARGOCD -- pull --> GL_MF
    ARGOCD -- 배포 --> TARGET
    TARGET -- image pull --> HARBOR
```

### Agent Pod 내부 구성 (컨테이너 · 볼륨)

```mermaid
flowchart TB
    subgraph POD["Jenkins Agent Pod"]
        direction TB
        INIT["initContainer: build-truststore<br/>JDK cacerts + vault-ca → /truststore/cacerts"]

        subgraph C["containers"]
            JNLP[jnlp]
            DIND["dind (privileged)"]
            DCK[docker]
            MVN[maven]
            SQ[sonarqube]
            TV[trivy]
            KZ[kustomize]
            PL["polaris (root)"]
            AR[argocli]
        end

        V1[/"Secret: vault-ca-cert"/]
        V2[/"emptyDir: truststore-vol"/]
        V3[/"emptyDir: /built-images"/]
        V4[/"PVC jenkins-repo-pvc<br/>/shared/maven-repo"/]
        V5[/"PVC jenkins-tar-pvc<br/>/shared/trivy"/]
    end

    V1 --> INIT --> V2
    V1 -- "/etc/ssl/certs/vault-ca" --> JNLP
    V1 -- "/etc/docker/certs.d/HARBOR_HOST" --> DIND
    V1 -- "/vault-ca" --> AR
    V2 -- "/truststore" --> JNLP
    V2 -- "/truststore" --> SQ
    DCK -- "docker save *.tar" --> V3 --> TV
    V4 <--> MVN
    V5 -- "db.tar.gz / java-db.tar.gz" --> TV
```

| 컨테이너 | 이미지 | 역할 |
|---|---|---|
| `jnlp` | `jenkins-inbound-agent:3345...-jdk21` | Jenkins agent, git clone, Dockerfile 생성 |
| `dind` | `docker:20.10-dind` (privileged) | Docker 데몬. `tcp://0.0.0.0:2375`, bridge `192.168.250.1/24`, pool `192.168.240.0/20`, MTU 1450 |
| `docker` | `docker:20.10-dind` | Docker CLI 클라이언트 (`DOCKER_HOST=tcp://localhost:2375`) |
| `maven` | `maven:3.9.5-eclipse-temurin-17` | compile / package |
| `sonarqube` | `sonar-scanner-cli:5.0.1` | 정적 분석 (truststore로 TLS 신뢰) |
| `trivy` | `aquasec/trivy:0.60.0` | 이미지 tar 오프라인 취약점 스캔 |
| `kustomize` | `kustomize:v5.0.1` | overlay 이미지 태그 갱신 및 렌더링 |
| `polaris` | `fairwinds/polaris:9.6` | 매니페스트 베스트프랙티스 점검 |
| `argocli` | `argocli-custom:v4` | argocd CLI(v3.1.8), git, curl, jq |

> 데몬(`dind`)과 클라이언트(`docker`)를 분리해, 파이프라인 코드는 privileged 컨테이너에서 직접 실행되지 않습니다. 레지스트리와 TLS 핸드셰이크는 데몬이 하므로 Harbor CA는 `dind`의 `certs.d`에 마운트합니다.

---

## 2. 스테이지 흐름

```mermaid
flowchart TD
    S1["1. Print env<br/>변수/태그 생성"] --> S2["2. Checkout source<br/>소스 repo + GitOps repo clone"]
    S2 --> S3["3. Build first<br/>mvn clean compile"]
    S3 --> S4["4. SonarQube scan"]
    S4 --> S5{"5. Quality gate<br/>QUALITY_GATE_PASS == 'true'?"}
    S5 -- "아니오: waitForQualityGate()" --> QG{"status OK?"}
    QG -- 아니오 --> FAIL1(["❌ 중단"])
    QG -- 예 --> S6
    S5 -- "예: skip (기본)" --> S6["6. Code Artifact<br/>mvn package (테스트 skip)"]
    S6 --> S7["7. Manifest values write<br/>kustomization.yaml newName/newTag 갱신<br/>→ kustomize build → kubernetes.yaml"]
    S7 --> S8["8. Write Dockerfile<br/>DOCKER_PATH 없으면 템플릿 생성"]
    S8 --> S9["9. Build & Push Docker Image<br/>docker build → push → save(tar)"]
    S9 --> S10["10. Trivy image scan<br/>CRITICAL,HIGH (exit-code 0)"]
    S10 --> S11{"11. Polaris<br/>score > 50?"}
    S11 -- 아니오 --> FAIL2(["❌ 중단"])
    S11 -- 예 --> S12["12. Push to GitOps Repo<br/>Dockerfile, ARTIFACT.txt, kubernetes.yaml<br/>commit + git tag"]
    S12 --> S13["13. ArgoCD bootstrap<br/>CA 등록 · repo 등록 · project source · app upsert"]
    S13 --> S14["14. Deploy with ArgoCD<br/>hard-refresh → sync → wait --health"]
    S14 --> DONE(["✅ 배포 완료"])
```

| # | Stage | 컨테이너 | 주요 내용 |
|---|---|---|---|
| 1 | Print env | jnlp | `BUILD_DATE_TAG`(yyyyMMddHHmm), 이미지/Argo 변수 설정 |
| 2 | Checkout source | jnlp | 소스 repo(`GIT_BRANCH`) → `repo/`, GitOps repo(main) clone. 실패 시 `MANIFEST_REPO_FLAG=false` |
| 3 | Build first | maven | `mvn clean compile` (Sonar용 `target/classes` 생성) |
| 4 | SonarQube scan | sonarqube | `sonar.projectKey=$APPLICATION`, `repo/src/main/java/**` |
| 5 | Quality gate | jnlp | `QUALITY_GATE_PASS`가 `'true'`(기본)면 skip, 아니면 결과 대기(2분 timeout) |
| 6 | Code Artifact | maven | `mvn package -P$BUILD_PROFILE` (테스트 skip). `SKIP_CODE_ARTIFACT=true`면 생략 |
| 7 | Manifest values write | kustomize | `overlays/$OVERLAY_PATH/kustomization.yaml`의 `images` 갱신 → `kubernetes.yaml` |
| 8 | Write Dockerfile | jnlp | 실행 jar 1개 탐지 → `eclipse-temurin:<JAVA_VERSION>-jre` 기반, non-root(uid 1001) |
| 9 | Build & Push Docker Image | docker | Harbor login → build → push → `/built-images/$APPLICATION.tar` |
| 10 | Trivy image scan | trivy | PVC의 DB tar로 오프라인 스캔, `trivy-report/trivy-result.json` |
| 11 | Polaris scan and validate | polaris | Final score ≤ 50 이면 실패, `polaris-report.txt` 아카이브 |
| 12 | Push to GitOps Repo | argocli | Manifest repo에 commit, 태그 `${APPLICATION}_${IMAGE_TAG}` push |
| 13 | ArgoCD bootstrap | argocli | GitLab CA 등록, scoped repo 등록(최대 10회 재시도), project source/Namespace 허용, app upsert |
| 14 | Deploy with ArgoCD | argocli | `sync` / `wait --health` (각 300초) |

---

## 3. 저장소 관계

| 구분 | 변수 | 예시 | 용도 |
|---|---|---|---|
| 소스 | `SOURCE_REPO` | `${GITLAB_HOST}/sample/sample-petmaven-boot.git` | 애플리케이션 소스 |
| GitOps (kustomize 원본) | `GITOPS_REPO` | `.../sample-cicd/sample-petmaven-boot-gitops.git` | `base/`, `overlays/<env>/` |
| Manifest (배포 대상) | `MANIFEST_REPO` | `.../sample-cicd/sample-petmaven-boot-dev.git` | 렌더링된 `kubernetes.yaml`, Argo CD가 바라보는 repo |

GitOps repo의 예상 구조:

```
sample-petmaven-boot-gitops/
├── base/
└── overlays/
    └── dev/
        └── kustomization.yaml   # images: - name: <APPLICATION> / newName / newTag
```

---

## 4. 파라미터 & 변수

### Job 파라미터 (Devopsit 기본 파라미터명)

| 파라미터 | 필수 | 설명 |
|---|---|---|
| `GIT_BRANCH` | ✅ | 소스 repo 브랜치 |
| `GIT_CREDENTIAL` | ✅ | GitLab username/password credential ID |
| `IMAGE_REGISTRY_CREDENTIAL` | ✅ | Harbor username/password credential ID |
| `SONARQUBE_CREDENTIAL` | ✅ | SonarQube token credential ID |
| `ARGOCD_AUTH_TOKEN` | ✅ | Argo CD API 토큰 |
| `QUALITY_GATE_PASS` | | `true`(기본): Quality Gate skip / `false`: SonarQube 결과 대기 후 실패 시 중단 |
| `BUILD_PROFILE` | | Maven profile (`-P`) |
| `JAVA_VERSION` | | 런타임 JRE 버전 (기본 `17`) |
| `ACTIVE_PROFILE` | | `spring.profiles.active` (기본 `dev`) |

### 파이프라인 내 하드코딩 변수 (`Print env`)

프로젝트마다 수정해야 하는 값입니다.

| 변수 | 현재 값 | 설명 |
|---|---|---|
| 엔드포인트 (`DOMAIN`, `*_HOST`, `*_URL`) | [0-1 참고](#0-1-엔드포인트-url-표준) | 파일 최상단 상수 블록에서 정의 |
| `SOURCE_REPO` | `${GITLAB_HOST}/${APPLICATION_GROUP}/${APPLICATION}.git` | 소스 저장소 |
| `GITOPS_REPO` / `MANIFEST_REPO` | `${GITLAB_HOST}/sample-cicd/...` | 3절 표 참고 |
| `OVERLAY_PATH` | `dev` | kustomize overlay (브랜치명과 동일하게) |
| `GITOPS_NAME` | `sample-petmaven-boot-gitops` | GitOps repo 디렉터리명 |
| `APPLICATION` | `sample-petmaven-boot` | 앱명(= Devopsit 프로젝트명) |
| `APPLICATION_GROUP` | `sample` | Devopsit 서비스명 = Argo CD project |
| `NAMESPACE` | `sample-dev` | 배포 네임스페이스 |
| `DOCKER_PATH` | `''` | 소스에 Dockerfile이 있으면 해당 경로 |
| `SKIP_CODE_ARTIFACT` | `false` | jar 빌드 생략 여부 |
| `QUALITY_GATE_PASS` | 파라미터 값, 없으면 `'true'` | `'true'`면 Quality Gate skip, 그 외 값이면 결과 대기 |

### 생성되는 이름 규칙

```
IMAGE_REPO = <HARBOR_HOST>/<APPLICATION_GROUP>/<APPLICATION_GROUP>-<OVERLAY_PATH>/<APPLICATION>
IMAGE_TAG  = <yyyyMMddHHmm>-<NAMESPACE>-<GITOPS_NAME>-<BUILD_NUMBER>
ARGO_APP   = <NAMESPACE>-<APPLICATION>
GIT TAG    = <APPLICATION>_<IMAGE_TAG>
```

---

## 5. 사전 준비 사항

**Kubernetes (Agent 생성 네임스페이스)**
- Secret `vault-ca-cert` (`ca.crt` 키 포함)
- imagePullSecret `cicdbot-imagepull-secret`
- PVC `jenkins-repo-pvc` (Maven 로컬 저장소 캐시)
- PVC `jenkins-tar-pvc` — `db.tar.gz`, `java-db.tar.gz` (Trivy DB, 오프라인 스캔용으로 미리 적재)
- privileged 컨테이너 허용 (dind)

**Jenkins**
- Kubernetes plugin, SonarQube Scanner plugin (`installationName: sonarqube`)
- Credential: Git / Harbor / SonarQube

**외부 시스템**
- Harbor에 베이스 이미지 `library/eclipse-temurin:<JAVA_VERSION>-jre` 및 각 도구 이미지 미러링
- SonarQube: `SONARQUBE_URL`
- Argo CD: `ARGOCD_HOST`, project `<APPLICATION_GROUP>` 존재, 대상 클러스터 `K8S_API_URL` 등록

---

## 6. 산출물

| 산출물 | 위치 |
|---|---|
| 컨테이너 이미지 | Harbor `IMAGE_FULL_NAME` |
| `kubernetes.yaml`, `Dockerfile`, `ARTIFACT.txt`, `README.md`(태그명) | Manifest repo main 브랜치 + git tag |
| `polaris-report.txt` | Jenkins 빌드 아티팩트 |
| `trivy-report/trivy-result.json` | 워크스페이스 (아카이브 안 함) |

---

## 7. 알려진 제약 / TODO

- **Trivy는 게이트가 아님**: `--exit-code 0` 이므로 CRITICAL/HIGH가 있어도 통과합니다.
- **Quality Gate 기본 skip**: `QUALITY_GATE_PASS` 기본값이 `'true'`입니다. 전환하려면 Job 파라미터로 `false`를 넘기거나 `Print env`의 기본값을 `'false'`로 바꾸세요.
- **`MANIFEST_REPO_FLAG=false` 경로 미완성**: GitOps repo clone 실패 시 `kubernetes.yaml`이 만들어지지 않아 이후 `cat ./kubernetes.yaml`, Polaris, `cd ${GITOPS_NAME}` 단계에서 실패합니다.
- **자격증명 노출 위험**: `git clone https://${CICDBOT_ID}:${CICDBOT_PW}@...`, `--auth-token ${params.ARGOCD_AUTH_TOKEN}` 이 Groovy 문자열 보간으로 쉘에 전달됩니다. 작은따옴표 `sh '...'` + 쉘 변수 사용, `ARGOCD_AUTH_TOKEN`은 Secret Text credential로 전환을 권장합니다.
- `trivy` 컨테이너의 `mountPath: 'shared/trivy'` 앞에 `/` 가 빠져 있습니다 (pod-level PVC 마운트로 동작은 함).
- `GITOPS_REPO`, `MANIFEST_REPO`의 그룹/저장소명(`sample-cicd/...`)은 아직 프로젝트별 하드코딩입니다. 호스트 부분만 `GITLAB_HOST`로 표준화되어 있습니다.
- `Build first`는 워크스페이스 경로를 `/var/jenkins_home/workspace/${JOB_NAME}`로 가정합니다.
- `Print env`의 `env.BRANCH`는 정의되지 않아 `null`로 출력됩니다.
- 주석 처리된 스테이지: `Gitops repo create`(GitLab API로 그룹/repo 자동 생성), `Git tag push`.
