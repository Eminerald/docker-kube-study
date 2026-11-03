# 서버 운영 학습 일지

> 맥(Apple Silicon)에서 Docker/K8s 모니터링 스택을 직접 띄우고, 일부러 망가뜨려서 알림까지 받아보기
> 이 문서 하나에 진행할 때마다 내용을 추가·수정해 나간다.

| rev | 날짜 | 내용 |
|---|---|---|
| rev1 | 2026-10-30 (금) | 1주차 Day 1 ~ Day 2·3 |
| rev2 | 2026-11-02 (월) | 1주차 Day 4 추가, 10/30 내용에 스크린샷 삽입 |
| rev3 | 2026-11-03 (화) | 1주차 Day 5 추가 (알림), "추가로 해볼 수 있는 것" 섹션 신설 |

---

## 진행 현황

| 주차 | 단계 | 내용 | 날짜 | 상태 |
|---|---|---|---|---|
| 1주차 | Day 1 | OrbStack, Ubuntu VM, kubectl/helm/k3d 설치 | 10/30 | 완료 |
| 1주차 | Day 2·3 | Prometheus + node-exporter + Grafana, PromQL, 대시보드 | 10/30 | 완료 |
| 1주차 | Day 4 | cAdvisor 컨테이너 지표, Node Exporter Full(1860) | 11/02 | 완료 |
| 1주차 | Day 5 | Alertmanager + 알림 규칙 (HighCPU, TargetDown 실습) | 11/03 | 완료 |
| 2주차 | Day 1~5 | k3d 클러스터 + kube-prometheus-stack, 장애 실습 | | 예정 |

**현재 구성 (최신)**: Prometheus v3.5.0 · node-exporter v1.9.1 · Grafana 12.1.0 · cAdvisor 0.56.1 (ghcr.io) · Alertmanager v0.28.1 — 최종 compose 파일은 [Day 5](#day-5--알림-alertmanager) 참고

### 접속 주소 (로컬, `docker compose up -d` 상태에서)

| 서비스 | 주소 | 용도 |
|---|---|---|
| Prometheus | http://localhost:9090 | PromQL 쿼리 (Query 탭) |
| Prometheus Targets | http://localhost:9090/targets | 수집 대상 UP/DOWN 확인 |
| Prometheus Alerts | http://localhost:9090/alerts | 알림 규칙 상태 (Day 5) |
| Grafana | http://localhost:3000 | 대시보드 (admin / admin) |
| node-exporter | http://localhost:9100/metrics | 호스트 지표 원본 |
| cAdvisor | http://localhost:8080 | 컨테이너 지표 웹 UI (`/metrics`는 원본) |
| Alertmanager | http://localhost:9093 | 알림 수신 확인 (Day 5) |

### 참고 사이트

| 주제 | 주소 |
|---|---|
| Prometheus 문서 | https://prometheus.io/docs/ |
| Prometheus + cAdvisor 가이드 | https://prometheus.io/docs/guides/cadvisor/ |
| Grafana 대시보드 1860 (Node Exporter Full) | https://grafana.com/grafana/dashboards/1860 |
| cAdvisor 저장소·릴리스 | https://github.com/google/cadvisor/releases |
| cAdvisor × Docker 29 containerd-snapshotter 이슈 | https://github.com/google/cadvisor/issues/3643 |
| kube-prometheus-stack 차트 (2주차) | https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack |
| k3d | https://k3d.io |
| OrbStack | https://orbstack.dev |

---

## 2026-10-30 (금) · Day 1 ~ Day 2·3

### Day 1 · 환경 준비

#### 설치한 것과 역할

| 도구 | 버전 | 무엇인가 |
|---|---|---|
| OrbStack | 2.2.5 | 맥에서 리눅스 VM과 Docker 엔진을 가볍게 돌려주는 앱. Docker Desktop 대체 |
| Docker | 29.6.1 | 컨테이너를 만들고 실행하는 도구 |
| kubectl | 1.37.2 | 쿠버네티스 클러스터에 명령을 내리는 CLI |
| Helm | 4.3.1 | 쿠버네티스용 패키지 매니저 (여러 리소스를 묶어서 한 번에 설치) |
| k3d | 5.9.0 (k3s 1.35.5) | Docker 컨테이너 안에 경량 쿠버네티스(k3s) 클러스터를 띄우는 도구 |
| Ubuntu VM `lab` | Ubuntu 26.04.1 LTS, aarch64 | 리눅스 운영 실습용 VM |

#### 맥 터미널 vs lab VM 역할 구분

- **맥 터미널** (`seongjong@Mac`): `brew`로 설치, `docker` / `kubectl` / `k3d` 실행
- **lab VM** (`seongjong@lab`, `orb -m lab`으로 접속): systemd, ufw, 계정 관리 같은 리눅스 운영 실습용. 패키지 설치는 `apt`
- VM 안에서 `brew`가 안 되는 건 정상 (리눅스는 `apt`)

![Ubuntu VM 생성 및 접속](screenshots/1030/02-docker-context-orbstack-vm.png)
*`orb create ubuntu lab` → `orb -m lab`으로 접속, Ubuntu 26.04.1 / aarch64 확인*

#### 확인 명령

```bash
docker run --rm hello-world
kubectl version --client
helm version
k3d version
```

![Day 1 도구 설치 완료](screenshots/1030/03-day1-tools-hello-world.png)
*kubectl·helm·k3d 설치 후 hello-world 성공, 각 도구 버전 확인*

---

### Day 2·3 · Docker compose로 모니터링 스택 띄우기

#### 구성 요소

| 컨테이너 | 역할 | 포트 |
|---|---|---|
| **Prometheus** | 메트릭을 주기적으로 긁어와(scrape) 시계열로 저장하고, PromQL로 조회 | 9090 |
| **node-exporter** | 호스트(리눅스)의 CPU·메모리·디스크 등 지표를 `/metrics`로 노출 | 9100 |
| **Grafana** | Prometheus 데이터를 대시보드로 시각화 | 3000 |

흐름: `node-exporter`가 지표를 노출 → `Prometheus`가 15초마다 가져가 저장 → `Grafana`가 Prometheus에 쿼리해서 그림으로 표시

#### 폴더 구조

```
~/server-study/week1/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
└── alertmanager/        (Day 5에서 사용)
```

#### docker-compose.yml

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.5.0
    volumes:
      - ./prometheus:/etc/prometheus
      - prom-data:/prometheus
    ports: ["9090:9090"]

  node-exporter:
    image: prom/node-exporter:v1.9.1
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/rootfs
    ports: ["9100:9100"]

  grafana:
    image: grafana/grafana:12.1.0
    ports: ["3000:3000"]
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  prom-data:
  grafana-data:
```

> 11/02 Day 4에서 `restart: unless-stopped`와 cAdvisor가 추가됨 → 최종본은 아래 [Day 4](#day-4--컨테이너-메트릭) 참고

**원래 계획서에서 바꾼 점**
- 이미지 태그를 `latest` → **버전 고정**. `latest`는 시점마다 다른 이미지가 받아져서 재현이 안 되고, 인수인계 문서로 쓸 수 없음
- node-exporter에 **`/:/rootfs` 마운트 + `--path.rootfs`** 추가. 이게 없으면 파일시스템 지표가 컨테이너 자신의 것만 보이거나 비어서 디스크 모니터링이 의미 없어짐

**설정 항목 설명**
- `pid: host` — node-exporter가 컨테이너가 아니라 호스트의 프로세스 정보를 보게 함
- `:ro` — 읽기 전용 마운트. 모니터링 도구가 호스트 파일을 실수로 바꾸지 못하게
- `volumes: prom-data, grafana-data` — 이름 있는 볼륨. 컨테이너를 지웠다 다시 만들어도 수집 데이터와 대시보드가 유지됨

#### prometheus/prometheus.yml

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100"]
```

- `scrape_interval: 15s` — 15초마다 각 타깃에서 지표를 가져옴
- `job_name` — 수집 대상 묶음 이름. 쿼리 결과의 `job` 라벨이 됨

#### 실행

```bash
cd ~/server-study/week1
docker compose up -d
docker compose ps
```

- http://localhost:9090/targets 에서 `prometheus`, `node` 둘 다 **UP** 확인
- OrbStack 앱에서 컨테이너 3개가 `week1` 그룹으로 묶여 보임 → compose 프로젝트 이름은 **폴더 이름**

![compose 실행 결과](screenshots/1030/06-compose-up-orbstack.png)
*`docker compose ps`로 컨테이너 3개 Up 확인, OrbStack 앱에서 `week1` 그룹으로 표시*

#### 핵심 개념: 서비스 이름으로 통신되는 이유

Grafana 데이터소스에 `localhost:9090`이 아니라 **`http://prometheus:9090`** 을 넣어야 했다.

- 컨테이너 안에서 `localhost`는 **그 컨테이너 자기 자신**이다. Grafana 컨테이너 안에는 Prometheus가 없으므로 `localhost:9090`은 실패한다.
- docker compose는 같은 파일의 서비스들을 **하나의 기본 네트워크**로 묶고, 그 네트워크 안에서 **서비스 이름을 DNS 이름처럼** 쓸 수 있게 해준다.
- 그래서 `prometheus.yml`의 `node-exporter:9100`, Grafana의 `prometheus:9090`이 동작한다.
- 반대로 브라우저(맥)에서 접속할 때는 `ports`로 열어둔 `localhost:9090`, `localhost:3000`을 쓴다.

![Grafana 데이터소스 연결](screenshots/1030/11-grafana-datasource.png)
*Grafana 데이터소스에 `http://prometheus:9090` 연결*

#### PromQL 쿼리

| 지표 | 쿼리 | 설명 |
|---|---|---|
| 수집 상태 | `up` | 타깃이 살아 있으면 1, 아니면 0 |
| CPU 사용률 | `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)` | 최근 1분간 idle(쉬는) 시간 비율을 구해 100에서 뺌 |
| 메모리 사용률 | `100 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100` | 가용 메모리 비율을 100에서 뺌 |
| 디스크 사용률 | `100 - node_filesystem_avail_bytes{mountpoint="/var/lib/docker"} / node_filesystem_size_bytes{mountpoint="/var/lib/docker"} * 100` | OrbStack 환경 기준 (아래 트러블슈팅 참고) |

- `rate(...[1m])` — 계속 증가하기만 하는 카운터 값을 "1분 동안 초당 얼마나 늘었나"로 바꿔줌
- `{mode="idle"}` — 라벨로 원하는 시계열만 골라냄
- 지표 값은 **맥이 아니라 OrbStack 리눅스 VM 기준**. 메모리 총량이 맥 24GB보다 작게 나오는 게 정상

![메모리 가용률 그래프](screenshots/1030/08-memory-available-graph.png)
*PromQL로 조회한 메모리 가용률. Y축이 93.2~93.9 사이로 확대돼 있어 출렁이는 것처럼 보임*

#### Grafana 대시보드 `week1 서버 모니터링`

| 패널 | 시각화 | 설정 |
|---|---|---|
| CPU 사용률 | Time series | Unit `Percent (0-100)`, Min 0 / Max 100 |
| 메모리 사용률 | Time series | Unit `Percent (0-100)`, Min 0 / Max 100 |
| 디스크 사용률 | Gauge | Unit `Percent (0-100)`, Min 0 / Max 100, Thresholds 70(노랑)·90(빨강) |

![CPU 패널 편집](screenshots/1030/12-grafana-cpu-panel-edit.png)
*패널 편집 화면. Code 모드로 쿼리 입력, 오른쪽 Standard options에서 Unit 지정*

![완성된 대시보드](screenshots/1030/14-grafana-dashboard-gauge.png)
*디스크 게이지(70/90 임계값) + 메모리·CPU 시계열 패널*

**설계 이유**
- **Min/Max 0~100 고정**: 고정하지 않으면 Grafana가 Y축을 값 범위에 맞춰 확대해서, 0.29%짜리 디스크도 크게 출렁이는 것처럼 보인다.
- **단위 Percent (0-100)**: 쿼리에서 `* 100`을 했으므로 0-100. `* 100`을 안 했다면 `Percent (0.0-1.0)`. 잘못 고르면 50%가 5000%로 표시됨. (단위는 `Misc` 카테고리에 있고, Unit 칸에 `percent`를 타이핑하면 빠름)
- **지표 방향 통일**: 세 패널 모두 "**높을수록 위험**"인 사용률로 통일. 가용률과 사용률이 섞이면 보는 사람이 반대로 해석할 수 있음
- **디스크는 게이지**: 디스크는 시간에 따른 변화보다 "지금 얼마나 찼나"가 중요. CPU처럼 출렁이는 값은 시계열 그래프가 맞음
- **임계값 90**: Day 5에 만들 디스크 알림 기준(90%)과 맞춤. 대시보드 색과 알림 기준이 다르면 운영 시 혼란

#### Grafana 그래프는 실시간으로 바뀌나?

- **데이터 수집**은 계속된다. Prometheus는 브라우저를 열든 말든 15초마다 값을 가져와 저장한다.
- **화면**은 기본적으로 자동 갱신되지 않는다. 대시보드를 연 시점에 한 번 쿼리해서 그린 그림이다.
- 자동 갱신: 대시보드 오른쪽 위 **Refresh ▾** → `15s` 또는 `30s` → **Save dashboard**
- 화면 갱신 주기는 **수집 주기(15초)보다 짧게 할 필요가 없다.** 5초마다 다시 그려도 같은 점을 반복해서 그릴 뿐이고, 여러 명이 대시보드를 띄워두는 서버에서는 Prometheus에 불필요한 부하가 된다.
- 맥이 **잠자기**에 들어가면 OrbStack VM도 멈춰서 그 구간은 수집되지 않는다 → 그래프에 끊긴 구간으로 보임. 실제 서버에서 이런 공백이 생기면 "서버나 수집기가 죽었었다"는 신호다.

---

### 막혔던 것 / 해결 방법 (10/30)

#### 1. `brew install orbstack` 실패 — TLS 에러

```
curl: (35) LibreSSL/3.3.6: error:1404B42E:SSL routines:ST_CONNECT:tlsv1 alert protocol version
```

![brew TLS 에러](screenshots/1030/01-brew-orbstack-tls-error.png)

- **원인**: Homebrew가 설치 파일을 받을 때 HTTPS 연결 협상(TLS) 단계에서 거부됨. macOS 기본 curl(LibreSSL)과 서버 설정이 안 맞거나, 네트워크가 HTTPS를 막는 경우
- **해결 방법 후보**: `brew install curl` 후 `export HOMEBREW_FORCE_BREWED_CURL=1`로 Homebrew판 curl 사용 / 공식 사이트에서 dmg 직접 다운로드 / 다른 네트워크(핫스팟)로 시도
- **실제로 해결된 방법**: (직접 기록)

#### 2. `docker version`에서 데몬 연결 실패

```
Cannot connect to the Docker daemon at unix:///Users/seongjong/.docker/run/docker.sock
```

- **원인**: Docker Desktop이 이미 설치돼 있어서 docker 컨텍스트가 `desktop-linux`(Docker Desktop)를 가리키고 있었고, Docker Desktop은 꺼져 있었음
- **해결**: `docker context use orbstack`
- **교훈**: Docker Desktop과 OrbStack을 동시에 쓰면 어느 엔진에 컨테이너를 띄웠는지 헷갈림. Docker Desktop은 종료하고 자동 실행 끄기

![docker 컨텍스트 전환](screenshots/1030/04-docker-context-which-kubectl.png)
*`orbstack` 컨텍스트로 전환, `which -a kubectl`로 kubectl이 여러 곳에 있음을 확인*

#### 3. kubectl 버전이 brew로 설치한 것과 다르게 나옴

- brew로 1.37.2를 설치했는데 `kubectl version --client`가 **1.36.1**
- `which -a kubectl`과 `ls -l`로 확인:
  - `/opt/homebrew/bin/kubectl` → brew의 1.37.2
  - `/usr/local/bin/kubectl` → Docker.app 안의 1.36.1
- **원인**: PATH 순서상 brew판이 먼저인데도, zsh가 예전에 찾아둔 명령 위치를 **캐시**해서 계속 Docker Desktop판을 실행함
- **해결**: `rehash` (zsh 명령 위치 캐시 초기화) → 1.37.2 확인
- **교훈**: "지금 실행되는 바이너리가 어느 것인지" 확인하는 습관 (`which -a`, `ls -l`). 서버에서도 자주 쓰임
- 참고: PATH에 `/opt/homebrew/bin`이 중복으로 들어가 있음 → `~/.zprofile`의 `brew shellenv` 중복 의심, 나중에 정리

#### 4. 명령어 옆 `# 주석`이 인자로 들어감

- `rehash   # zsh 캐시 초기화`를 붙여넣었더니 `rehash: too many arguments`
- **원인**: zsh는 대화형 터미널에서 `#`을 주석으로 인식하지 않는 게 기본값
- **해결**: 명령만 한 줄씩 실행. (원하면 `~/.zshrc`에 `setopt interactivecomments` 추가)

![zsh 주석 문제와 kubectl 경로](screenshots/1030/05-kubectl-path-zsh-comment.png)
*`#` 뒤 글자들이 `ls`의 인자로 들어가 에러. 아래 `ls -l` 결과로 두 kubectl의 실제 위치 확인*

#### 5. 디스크 쿼리 결과가 비어 있음 (`mountpoint="/"`)

- CPU·메모리 쿼리는 되는데 디스크 쿼리만 `Empty query result`

![디스크 쿼리 결과 없음](screenshots/1030/07-disk-query-empty.png)

- 조건을 빼고 `node_filesystem_size_bytes`로 전체 목록을 확인:
  - `device="mac"`, `fstype="virtiofs"` (~1TB): OrbStack이 **맥의 디스크를 VM에 공유**한 것. VM 디스크가 아니므로 모니터링 대상 아님
  - `device="/dev/vdb1"`, `fstype="btrfs"`: **OrbStack VM의 실제 디스크**. `/var/lib/docker`, `/var/lib/cni`, `/etc/rancher/node`, `/opt/orbstack-guest/data` 등 여러 곳에 마운트되어 선이 여러 개로 보임
  - **`mountpoint="/"`는 목록에 없음** → 그래서 결과가 비었던 것

![파일시스템 목록](screenshots/1030/09-filesystem-list.png)
*위쪽 큰 선(~1TB)은 맥 디스크 공유분(virtiofs), 아래쪽이 VM 실제 디스크(/dev/vdb1, btrfs)*

- **해결**: 컨테이너 이미지·볼륨이 쌓이는 `mountpoint="/var/lib/docker"` 기준으로 쿼리 변경

![/var/lib/docker 기준 디스크 사용률](screenshots/1030/10-disk-var-lib-docker.png)
*약 0.29%. Y축이 0.28~0.30으로 확대돼 있어 크게 변하는 것처럼 보이지만 실제로는 거의 변화 없음*

- **교훈**
  - 쿼리를 쓰기 전에 **조건 없이 먼저 목록을 보고** 라벨 값을 확인할 것
  - 알림 규칙도 환경마다 달라야 한다. 계획서 Day 5의 `DiskAlmostFull`도 `mountpoint="/"`라 이대로면 **에러 없이 영원히 안 울리는 알림**이 됨 → Day 5에 수정 예정
  - 학교 서버(일반 Ubuntu)라면 보통 `mountpoint="/"`, `fstype="ext4"`이고 데이터 디스크가 `/data` 등에 따로 있을 수 있음. 실제 서버에서도 목록부터 확인

#### 6. Prometheus 그래프 시간이 9시간 다름

- 11시대인데 그래프에 02시대로 표시 (위 메모리 그래프 스크린샷의 시간축 참고)
- **원인**: Prometheus UI 기본값이 UTC
- **해결**: 오른쪽 위 ⚙ → **Use local time**
- **교훈**: 서버 로그와 그래프를 대조할 때 시간대를 먼저 확인

#### 7. Grafana 메모리 패널 제목과 값이 안 맞음

- 제목은 "사용률"인데 쿼리는 가용률(약 93%) → 메모리가 거의 꽉 찬 것처럼 보임

![메모리 패널 제목·쿼리 불일치](screenshots/1030/13-grafana-cpu-memory-panels.png)
*제목은 `메모리 사용률`인데 값은 가용률(약 93%)*

- 수정 과정에서 쿼리와 제목을 둘 다 바꿔 다시 반대로 어긋남 (제목 `메모리 가용률`, 값은 사용률 약 7% — 위 "완성된 대시보드" 스크린샷)
- **해결**: 쿼리를 사용률(`100 - ...`)로, 제목을 `메모리 사용률`로 맞춤
- **교훈**: 대시보드에서 제목·단위·쿼리 방향이 어긋나면 운영자가 정반대로 판단할 수 있음. 패널을 고치면 **쿼리와 제목을 함께** 확인

---

### 그래프 읽는 법 메모

- 그래프는 **Y축 범위부터** 확인. 디스크 사용률 그래프가 크게 출렁여 보였지만 실제로는 0.28~0.30% 사이 변화
- 11:20 직전 CPU가 40~50% 근처로 튄 것, 디스크가 살짝 늘어난 것은 `docker compose up` 때 이미지 3개를 받고 컨테이너를 동시에 띄운 흔적

---

---

## 2026-11-02 (월) · Day 4

### Day 4 · 컨테이너 메트릭

#### cAdvisor란

| 수집기 | 보는 대상 | 답해주는 질문 |
|---|---|---|
| **node-exporter** | 호스트(VM) 전체 | 서버 전체가 얼마나 바쁜가? |
| **cAdvisor** | 컨테이너 하나하나 | 그중 **어떤 컨테이너가** 바쁜가? |

> **정리**: 서버가 느려지면 node-exporter로 이상을 감지하고, cAdvisor로 원인 컨테이너를 찾는다.
> 쿠버네티스에서는 kubelet 안에 cAdvisor가 내장되어 같은 역할을 한다 (2주차에서 확인).

#### docker-compose.yml (Day 4 시점)

> 11/03 Day 5에서 Alertmanager가 추가됨 → 최종본은 아래 [Day 5](#day-5--알림-alertmanager) 참고

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.5.0
    restart: unless-stopped
    volumes:
      - ./prometheus:/etc/prometheus
      - prom-data:/prometheus
    ports: ["9090:9090"]

  node-exporter:
    image: prom/node-exporter:v1.9.1
    restart: unless-stopped
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/rootfs
    ports: ["9100:9100"]

  grafana:
    image: grafana/grafana:12.1.0
    restart: unless-stopped
    ports: ["3000:3000"]
    volumes:
      - grafana-data:/var/lib/grafana

  cadvisor:
    image: ghcr.io/google/cadvisor:0.56.1
    restart: unless-stopped
    privileged: true
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    command:
      - --containerd=/run/docker/containerd/containerd.sock
      - --containerd-namespace=moby
    ports: ["8080:8080"]

volumes:
  prom-data:
  grafana-data:
```

**10/30 대비 바뀐 점**
- **`restart: unless-stopped`** (전 서비스): OrbStack이나 서버가 재시작돼도 컨테이너가 자동으로 다시 뜸. 직접 `docker compose stop`으로 멈춘 경우만 예외. 주말 동안 맥이 재시작되면서 컨테이너가 안 올라와 있던 걸 계기로 추가
- **cAdvisor 추가**
  - `privileged: true`, `/var/lib/docker` 마운트: 다른 컨테이너들의 정보를 들여다봐야 해서 높은 권한이 필요함. **모니터링 도구가 높은 권한을 갖는다는 점은 실제 서버에서 보안 검토 대상**
  - `command`의 containerd 옵션 두 줄: Docker 29 호환을 위해 필요 (아래 트러블슈팅 8 참고)

#### prometheus/prometheus.yml (Day 4 시점)

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]
```

#### 반영 절차

```bash
cd ~/server-study/week1
docker compose config --quiet
docker compose up -d
docker compose restart prometheus
docker compose ps
```

- `docker compose config --quiet`: YAML 문법만 미리 검사. **출력이 없으면 정상**, 들여쓰기가 깨졌으면 몇 번째 줄인지 알려줌. 파일을 통째로 바꿀 때 먼저 확인하는 습관
- `docker compose restart prometheus`: compose 정의는 그대로고 **설정 파일만** 바뀐 경우 `up -d`가 컨테이너를 다시 만들지 않으므로, 재시작해서 설정을 다시 읽게 함

![compose 반영](screenshots/1102/02-cadvisor-compose-up.png)
*cAdvisor 포함 4개 컨테이너 기동, cadvisor는 `(healthy)`*

#### 컨테이너별 CPU 쿼리

```
sum by (image) (rate(container_cpu_usage_seconds_total{name!=""}[1m]))
```

- `{name!=""}` — 이름 없는 시계열(cgroup 전체 합계 등)을 제외
- `sum by (image)` — 이미지 이름별로 묶어서 합침. 범례가 짧고 읽기 쉬워짐
- 우리 환경에서는 `name` 라벨이 컨테이너 **이름이 아니라 ID**로 나와서 `image` 라벨로 구분 (트러블슈팅 8 참고)

![컨테이너별 CPU](screenshots/1102/07-container-cpu-by-image.png)
*`image` 라벨별로 선이 나뉨 — prometheus, node-exporter 등 컨테이너 단위 구분 성공*

**단위 읽는 법**
- 값의 단위는 **CPU 코어 수**. Y축의 `4.50m` = **0.0045 코어** (코어 하나의 0.45%)
- `m`은 밀리(1/1000). 쿠버네티스에서 `cpu: 500m`(0.5코어)처럼 자원을 지정할 때 같은 단위를 씀
- 노드 CPU 사용률(%)과 단위가 다르니 주의

#### Node Exporter Full 대시보드 (ID 1860)

1. Grafana → **Dashboards** → **New** → **Import**
2. `1860` 입력 → **Load** → **Import**
3. 데이터소스 선택 칸이 없는 건 정상. 최신 버전은 Grafana의 **기본(default) 데이터소스**를 자동으로 사용

![1860 가져오기](screenshots/1102/08-grafana-import-1860.png)

![Node Exporter Full](screenshots/1102/09-node-exporter-full-1860.png)

**화면 해석**

| 보이는 것 | 의미 |
|---|---|
| CPU Cores 15 / RAM Total 12 GiB | 맥(24GB)이 아니라 **OrbStack VM 기준** 값 |
| Root FS Used: N/A | 이 대시보드도 `mountpoint="/"` 기준이라 OrbStack에서 값 없음 (트러블슈팅 5와 같은 원인). 학교 서버에서는 정상 표시 예상 |
| Pressure: No data | 커널 자원 압박 지표(PSI). OrbStack VM 환경에서 수집 안 되는 것으로 보임 |
| 그래프 중간 끊긴 구간 | 맥 **잠자기** 동안 수집이 멈춘 흔적 |
| Network Traffic 마이너스 값 | 고장 아님. 받는 트래픽(Rx)은 위로, 보내는 트래픽(Tx)은 **아래로** 그리는 표시 방식 |
| Disk Space에 `/Applications`, `/Users` 등 | 맥 디스크 공유분(virtiofs)이 같이 잡힘 |

**대시보드 변수**: 상단의 Datasource / Job / Nodename / Instance 드롭다운. 대시보드 하나로 서버 여러 대를 골라 볼 수 있게 해줌. 학교 서버 여러 대에 node-exporter를 깔면 목록에 서버들이 나란히 뜸

**교훈**
- 남이 만든 대시보드에는 **우리 환경에 안 맞는 부분**이 꼭 있다. 그대로 믿지 말고 패널별로 확인
- 커뮤니티 대시보드는 작성자가 계속 고친다 (이번에 받은 건 2026-10-31 업데이트판). 운영 환경에서는 가져온 대시보드를 **JSON으로 내보내 저장소에 보관** — 이미지 버전 고정과 같은 이유

---

### 막혔던 것 / 해결 방법 (11/02)

#### 8. 컨테이너별 지표가 안 잡힘 — cAdvisor ↔ Docker 29 호환 문제 (오늘의 메인)

`rate(container_cpu_usage_seconds_total{name!=""}[1m])` 결과가 비어 있음

![컨테이너 쿼리 결과 없음](screenshots/1102/03-container-query-empty.png)

조건을 빼고 조회하니 컨테이너 이름 없이 큰 묶음 4개(`/`, `/docker`, `/init.scope`, `/.lxc`)만 존재

![name 라벨 없음](screenshots/1102/04-cadvisor-no-name-label.png)

해결까지 세 단계를 거침.

**① 원인 파악 — cAdvisor 로그**

```bash
docker compose logs cadvisor | grep -i docker
```
```
Failed to create existing container: /docker/717673f7...
open /rootfs/var/lib/docker/image/overlayfs/layerdb/mounts/.../mount-id: no such file or directory
```

- cAdvisor가 컨테이너 정보를 얻으려고 Docker의 **옛날 이미지 저장 위치**(`/var/lib/docker/image/.../layerdb`)를 찾는데 파일이 없음
- **Docker 29부터 containerd 이미지 저장소(containerd-snapshotter)가 기본값**이 되면서 저장 구조가 바뀜
- cAdvisor v0.49.1(2024년 버전)은 옛 구조만 알아서 컨테이너를 하나도 등록하지 못함
- 처음에는 Docker API 버전 문제로 추측했지만 **로그를 보니 원인이 달랐음** → 추측보다 로그가 먼저

**② 최신 cAdvisor로 교체 — 이미지 태그 함정**

- containerd-snapshotter 지원은 최신 cAdvisor에 추가됨. 이 버전부터 이미지 저장소가 `gcr.io` → **`ghcr.io`** 로 이동
- `ghcr.io/google/cadvisor:v0.55.0` → **`not found`**
  - ghcr.io 이미지는 태그에 **`v`가 붙지 않음**: `0.56.1` 형식
  - `up -d`가 실패하면 **기존 컨테이너가 그대로 돌아감** → 로그가 그대로라 "고쳤는데 그대로네"로 착각하기 쉬움. 바꾼 뒤엔 `docker compose ps`의 IMAGE 열로 실제 버전 확인
- `ghcr.io/google/cadvisor:0.56.1`로 교체 성공. `mount-id` 에러는 사라짐

**③ containerd 소켓 위치 지정**

새 로그:
```
Registration of the docker container factory failed:
unable to create containerd client: ... dial unix /run/containerd/containerd.sock: no such file or directory
```

![cAdvisor 0.56.1 containerd 에러](screenshots/1102/05-cadvisor-0.56.1-containerd-error.png)

- 새 cAdvisor는 containerd에 **직접 접속**하는데, 기본 위치(`/run/containerd/containerd.sock`, 별도 설치된 containerd 위치)에서 소켓을 못 찾음
- Docker에 딸린 containerd는 다른 위치에 소켓을 둠 → cAdvisor 컨테이너 안에서 실제 위치 찾기:

```bash
docker compose exec cadvisor find /var/run /run -name "containerd.sock" 2>/dev/null
```
→ `/run/docker/containerd/containerd.sock`

![containerd 소켓 찾기](screenshots/1102/06-find-containerd-sock.png)

- cAdvisor **컨테이너 안에서** 찾은 이유: compose에서 `/var/run`을 마운트했으므로, cAdvisor 눈에 보이는 경로가 곧 설정에 써야 할 경로
- compose에 옵션 추가:
  ```yaml
      command:
        - --containerd=/run/docker/containerd/containerd.sock
        - --containerd-namespace=moby
  ```
  - `moby`: Docker가 containerd 안에서 자기 컨테이너들을 넣어두는 구역(namespace) 이름. 지정하지 않으면 쿠버네티스용 구역(`k8s.io`)을 찾아서 Docker 컨테이너가 안 보임
- **결과**: 컨테이너별 지표 수집 성공 (위 "컨테이너별 CPU" 스크린샷)
- **남은 한계**: Docker를 거치지 않고 containerd에서 직접 읽어서 `name` 라벨이 `week1-prometheus-1` 같은 이름이 아니라 **컨테이너 ID**로 나옴 → `image` 라벨로 구분

**교훈**
- 10/30에 "`latest` 대신 오래된 안정 버전으로 고정"한 선택이 **최신 Docker 엔진과 호환이 안 되는** 결과로 돌아옴. 버전 고정은 맞지만 **"무엇과 함께 돌아가는 버전인가"** 까지 확인해야 함
- 학교 서버도 Docker를 29로 올리는 순간 기존 cAdvisor가 **조용히** 깨질 수 있음 (컨테이너는 Up, 타깃도 UP인데 지표만 빠짐). **업그레이드 전 모니터링 스택 호환성 확인**은 서버장 체크리스트 항목
- 이미지 이름·태그는 **공식 README**에서 확인. 기억이나 블로그 글에 의존하면 `not found`

#### 9. prometheus.yml을 고쳤는데 반영이 안 됨

- 새 설정을 `week1/prometheus.yml`에 만들었는데, Prometheus가 읽는 파일은 **`week1/prometheus/prometheus.yml`** (폴더 안)
- compose에서 `./prometheus` **폴더**를 컨테이너의 `/etc/prometheus`로 연결했기 때문

![cadvisor 타깃 없음](screenshots/1102/01-targets-no-cadvisor.png)
*Targets에 cadvisor가 없음 — 예전 설정을 그대로 읽고 있었음*

- **해결**:
  ```bash
  mv prometheus.yml prometheus/prometheus.yml
  cat prometheus/prometheus.yml
  docker compose restart prometheus
  ```
- **교훈**: 설정을 바꿨는데 반영이 안 되면 **프로그램이 실제로 읽는 파일이 맞는지**부터 확인. 컨테이너에서는 compose의 `volumes`에 적힌 경로가 그 답. nginx, Prometheus 등 서버 운영에서 자주 겪는 유형

---

---

## 2026-11-03 (화) · Day 5

### Day 5 · 알림 (Alertmanager)

지금까지는 사람이 대시보드를 들여다봐야 이상을 알았다면, Day 5는 **조건을 걸어두고 넘으면 알려주게** 만드는 단계.

#### 구조

```
Prometheus ──(규칙 평가)──▶ 조건 충족 시 알림 발생 ──▶ Alertmanager ──▶ 수신처(Slack, Discord 등)
```

| 도구 | 하는 일 | 알림이 보이는 곳 |
|---|---|---|
| **Grafana** | 그래프로 **보여주기** | ✕ (대시보드는 그림만 그림) |
| **Prometheus** | 규칙으로 **판단하기** | http://localhost:9090/alerts |
| **Alertmanager** | 알림을 묶고, 중복을 거르고, **전달하기** | http://localhost:9093 |

#### 폴더 구조 (최종)

```
~/server-study/week1/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   └── alert-rules.yml      ← 새로 추가 (prometheus 폴더 안!)
└── alertmanager/
    └── alertmanager.yml     ← 새로 추가
```

#### 최종 docker-compose.yml

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.5.0
    restart: unless-stopped
    volumes:
      - ./prometheus:/etc/prometheus
      - prom-data:/prometheus
    ports: ["9090:9090"]

  node-exporter:
    image: prom/node-exporter:v1.9.1
    restart: unless-stopped
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/rootfs
    ports: ["9100:9100"]

  grafana:
    image: grafana/grafana:12.1.0
    restart: unless-stopped
    ports: ["3000:3000"]
    volumes:
      - grafana-data:/var/lib/grafana

  cadvisor:
    image: ghcr.io/google/cadvisor:0.56.1
    restart: unless-stopped
    privileged: true
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    command:
      - --containerd=/run/docker/containerd/containerd.sock
      - --containerd-namespace=moby
    ports: ["8080:8080"]

  alertmanager:
    image: prom/alertmanager:v0.28.1
    restart: unless-stopped
    volumes:
      - ./alertmanager:/etc/alertmanager
    ports: ["9093:9093"]

volumes:
  prom-data:
  grafana-data:
```

#### 최종 prometheus/prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - alert-rules.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]
```

- `rule_files` — 알림 규칙 파일 위치. `prometheus.yml`과 같은 폴더 기준 상대 경로
- `alerting` — 알림을 넘겨줄 Alertmanager 주소. compose 서비스 이름(`alertmanager`)으로 접근
- `evaluation_interval: 15s` — 규칙을 판단하는 주기. **기본값이 1분**이라 안 넣으면 조건을 넘고도 최대 1분 늦게 알아챔. 수집 주기(15초)에 맞춤

#### prometheus/alert-rules.yml

```yaml
groups:
  - name: node
    rules:
      - alert: HighCPU
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "CPU 사용률 80% 초과 (1분 지속)"

      - alert: DiskAlmostFull
        expr: 100 - node_filesystem_avail_bytes{mountpoint="/var/lib/docker"} / node_filesystem_size_bytes{mountpoint="/var/lib/docker"} * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "디스크 사용률 90% 초과"

      - alert: TargetDown
        expr: up == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.job }} 수집 대상이 응답하지 않음"
```

| 항목 | 의미 |
|---|---|
| `expr` | 이 쿼리 결과가 **참이면** 알림 후보. Day 2·3에서 만든 PromQL에 `> 80` 같은 조건을 붙인 것 |
| `for` | 조건이 **이 시간 동안 계속** 참이어야 진짜 알림(Firing). 그 전까지는 Pending |
| `labels.severity` | 심각도. 나중에 "critical은 Discord, warning은 이메일"처럼 **보낼 곳을 나누는 기준** |
| `annotations.summary` | 알림 설명 문구. `{{ $labels.job }}`처럼 **템플릿**으로 실제 라벨 값을 끼워 넣을 수 있음 |

**계획서에서 바꾼 점**
- **디스크 알림을 `mountpoint="/var/lib/docker"`로** — 10/30 트러블슈팅 5에서 확인한 대로 `"/"`는 OrbStack에 없어서, 원래대로면 **에러 없이 영원히 안 울리는 알림**이 됨
- **`TargetDown` 추가** — 실제 서버에서 가장 먼저 만드는 알림. 수집 대상이 죽으면 그 대상의 다른 알림이 전부 조용해지므로, **"감시 자체가 살아 있나"를 감시**
- **`evaluation_interval: 15s` 추가** (위 설명 참고)

#### alertmanager/alertmanager.yml

```yaml
route:
  receiver: default
receivers:
  - name: default
```

수신처를 비워둔 **최소 설정**. 알림이 Alertmanager까지 도착하는지 화면에서 확인만 하는 단계 (Discord 연결은 아래 "추가로 해볼 수 있는 것" 참고)

#### 반영 및 검사

```bash
cd ~/server-study/week1
docker compose config --quiet
docker compose up -d
docker compose exec prometheus promtool check rules /etc/prometheus/alert-rules.yml
docker compose restart prometheus
docker compose ps
```

- `promtool check rules` — Prometheus 이미지 안에 들어 있는 검사 도구로 **규칙 파일 문법만 미리 확인**. `SUCCESS: 3 rules found`면 정상
- `docker compose ps`에 alertmanager까지 **5개** 컨테이너가 Up이면 완료

![규칙 3개 Inactive](screenshots/1103/01-alerts-inactive.png)
*http://localhost:9090/alerts — `alert-rules.yml`의 규칙 3개가 모두 초록색 Inactive(조건 넘는 것 없음)*

#### 실습 1 · CPU 부하로 HighCPU 울리기

VM 코어가 15개라 **전체 평균 80%**를 넘기려면 코어 전부를 바쁘게 만들어야 함. 설치 없이 바로 되는 방법 사용:

```bash
docker run --rm --name cpu-load alpine sh -c 'for i in $(seq $(nproc)); do (while :; do :; done) & done; sleep 240'
```

- 코어 수(`nproc`)만큼 무한 반복을 동시에 돌리고 4분 뒤 자동 종료
- 계획서의 `ubuntu + apt install stress-ng` 방식은 패키지 설치 시간이 걸려서 대체

**상태 변화**

| 시간(대략) | 상태 | 의미 |
|---|---|---|
| 0 ~ 1분 | 🟢 Inactive | `rate(...[1m])`이 1분 평균이라 CPU 값이 서서히 올라오는 중 |
| 1분 전후 | 🟡 **Pending** | 80%를 넘었지만 `for: 1m` 동안 **계속 넘는지 지켜보는 중** |
| 2분 전후 | 🔴 **Firing** | 1분 내내 넘어서 **진짜 알림 발생** → Alertmanager로 전송 |
| 종료 후 | 🟢 Inactive | 부하가 사라지고 1분쯤 지나 원래대로 |

![HighCPU Pending](screenshots/1103/02-highcpu-pending.png)
*80%를 넘은 직후 — 노란색 Pending. 아직 알림을 보내지 않고 지켜보는 단계*

![HighCPU Firing](screenshots/1103/03-highcpu-firing.png)
*1분 동안 계속 넘어서 빨간색 Firing — 진짜 알림 발생*

![Grafana CPU 급상승](screenshots/1103/04-grafana-cpu-spike.png)
*같은 시간 Node Exporter Full의 CPU Basic 그래프. 13:15쯤 Busy User 87.8%까지 상승*

**핵심 개념: `for: 1m`이 필요한 이유 (Day 5 완료 기준)**

CPU가 80%를 넘은 **순간 바로 울리지 않고**, 노란색 Pending으로 1분 동안 "잠깐 튄 건지, 계속되는 건지"를 지켜본 뒤에야 Firing이 됐다. `for`가 없으면 `docker compose up` 때처럼 몇 초 튀는 순간마다 알림이 와서, 결국 아무도 알림을 안 보게 된다(알림 피로). **"진짜 문제일 때만 울리게 하는 장치"**다.

#### 실습 2 · 수집기를 멈춰서 TargetDown 울리기

```bash
docker compose stop node-exporter
```

- CPU 부하 없이 알림을 울리는 방법 (맥 팬이 돌지 않음)
- 멈춘 동안 서버 CPU·메모리·디스크 값이 수집되지 않아 **Grafana 그래프에 그 구간만 빈칸**이 생김. 에러 없이 **조용히 끊기기만** 하므로 알림이 없으면 아무도 모르고 지나감 → `TargetDown`이 필요한 이유

![TargetDown Firing](screenshots/1103/05-targetdown-firing.png)
*`for: 30s`라서 HighCPU보다 빨리 Firing. 규칙 정의의 `{{ $labels.job }}`이 실제 알림에서는 `node`로 바뀌어 "node 수집 대상이 응답하지 않음"으로 표시됨(템플릿)*

![Alertmanager에 도착한 알림](screenshots/1103/06-alertmanager-targetdown.png)
*http://localhost:9093 — `default` 수신처로 TargetDown 알림 도착. 시간 `04:29:54Z`는 UTC(`Z`)라 한국 시간 13:29:54*

확인 후 원상복구:

```bash
docker compose start node-exporter
```

- `TargetDown`의 `for`를 30초로 짧게 잡은 이유: 수집기가 죽는 건 순간적으로 튀는 값이 아니라 **확실한 이상**이므로 빨리 알리는 게 맞음

### 막혔던 것 / 궁금했던 것 (11/03)

#### 10. CPU 부하 중 맥 팬이 너무 돌아서 중간에 끄기

- 기존 터미널에서 `Ctrl+C`는 안 먹힐 수 있음 — 컨테이너 안에서 PID 1로 도는 `sh`가 종료 신호(SIGINT)를 무시하는 구조
- **해결**: 새 터미널 창에서 강제 종료
  ```bash
  docker kill cpu-load
  ```
- 교훈: 부하 테스트는 **멈추는 방법을 먼저 알고** 시작할 것. 실제 서버에서는 다른 사용자에게도 영향이 가므로 더 중요

#### 11. Firing(빨강)이 됐는데 알림이 아무 데도 안 뜸

- **정상 동작.** Firing은 "Prometheus가 문제라고 **판단했다**"는 뜻이지 "누군가에게 **알렸다**"는 뜻이 아님
- 흐름:
  ```
  CPU 80% 초과 → Prometheus: Pending → Firing → Alertmanager 도착 ✅
                                                    ↓
                                          수신처: 없음 → 여기서 끝 ❌
  ```
- `alertmanager.yml`에 수신처를 비워뒀기 때문. 맥 화면 팝업 같은 기능은 원래 없고, Prometheus·Alertmanager는 서버용이라 **Slack·Discord·이메일 같은 외부 채널**로 받는 게 기본
- Grafana 대시보드에도 Prometheus 알림은 표시되지 않음 (역할 분리: Grafana는 보여주기만)

---

## 추가로 해볼 수 있는 것

Day 5 기본 과제 이후 선택적으로 해볼 수 있는 확장 실습. 나중에 필요할 때 이 섹션만 보고 따라 할 수 있게 정리.

### 1. Discord로 알림 받기

Alertmanager는 Discord를 **정식 지원**(`discord_configs`, v0.28.1에 포함). 별도 프로그램 없이 웹후크 URL만 넣으면 됨.

**웹후크 만들기**
1. Discord에서 본인만 있는 서버 만들기 (왼쪽 `+` → 직접 만들기)
2. 채널 이름 옆 ⚙ **채널 편집** → **연동** → **웹후크** → **새 웹후크**
3. 이름을 `서버알림` 등으로 바꾸고 **웹후크 URL 복사**

**alertmanager/alertmanager.yml**

```yaml
route:
  receiver: discord
receivers:
  - name: discord
    discord_configs:
      - webhook_url: "<웹후크 URL>"
        send_resolved: true
```

- `send_resolved: true` — 문제가 해결됐을 때 `RESOLVED` 메시지도 한 번 더 보냄
- 반영: `docker compose restart alertmanager`
- 테스트: `docker compose stop node-exporter` → 1분 뒤 Discord에 TargetDown 메시지 도착 확인 → `docker compose start node-exporter` → RESOLVED 메시지 확인

> ⚠️ **웹후크 URL은 비밀번호와 같다.** URL을 아는 사람은 누구나 그 채널에 메시지를 보낼 수 있으므로, 이 저장소처럼 **공개된 곳에 실제 URL이 들어간 `alertmanager.yml`을 올리면 안 됨.** 일지에는 `<웹후크 URL>`처럼 가려서 기록.

- 연구실에서 Slack을 쓴다면 `discord_configs` 대신 `slack_configs`로 몇 줄만 바꾸면 그대로 옮길 수 있음
- 심각도별로 나누기: `route` 아래 `routes`에 `matchers: [severity="critical"]`를 두면 critical만 Discord, 나머지는 다른 곳으로 보낼 수 있음

### 2. Silence (알림 잠시 끄기)

- Alertmanager 화면 오른쪽 위 **New Silence**, 또는 알림 옆 **Silence** 버튼
- 라벨 조건(예: `alertname="TargetDown"`)과 기간을 정하면 그동안 해당 알림을 보내지 않음
- 용도: **서버 점검하느라 일부러 내릴 때** 알림이 쏟아지지 않게. 실제 서버 운영에서 자주 씀

### 3. 컨테이너 부하로 "범인 찾기"

- Day 4에서 건너뛴 실습. 특정 컨테이너 하나에만 부하를 걸고, node-exporter CPU 패널(서버 전체)과 cAdvisor 컨테이너별 그래프를 **나란히** 비교
- 서버 전체 그래프로는 "바쁘다"만 보이고, 컨테이너별 그래프로 **누가** 바쁜지 찾는 흐름 확인

---

## 다음에 할 것

- [x] **Day 5**: Alertmanager + 알림 규칙 → 11/03 완료
- [ ] **2주차 Day 1**: k3d 클러스터 만들기 (`k3d cluster create lab --agents 2`), 노드 3개 Ready 확인
  - 시작 전 1주차 compose 스택은 `docker compose down`으로 내려서 리소스 확보 (볼륨은 남으므로 데이터·대시보드 유지)
- [ ] (선택) "추가로 해볼 수 있는 것" — Discord 알림, Silence, 컨테이너 범인 찾기
- [ ] (선택) CPU·메모리 패널 범례를 `CPU %`, `메모리 %`로 정리 (Options → Legend → Custom)
- [ ] `week1 서버 모니터링`, 1860 대시보드 JSON 내보내서 저장소에 보관 (Export → Export as JSON)
- [ ] Docker Desktop 자동 실행 끄기, `~/.zprofile` PATH 중복 정리
