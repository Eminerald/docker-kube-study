# 재부팅 후 다시 켜기 메모

> 맥을 껐다 켜면 터미널 창과 port-forward가 모두 꺼진다.
> 1주차 컨테이너와 k3d 클러스터는 대부분 **자동으로 다시 뜨고**, 매번 직접 해줘야 하는 건 2주차 **port-forward 두 개**뿐이다.

---

## 방법 A · 스크립트로 한 번에 (추천)

처음 한 번만 `start.sh`를 `~/server-study/`에 넣고 실행 권한 주기:

```bash
chmod +x ~/server-study/start.sh
```

그다음부터는 재부팅 후 이것만:

```bash
~/server-study/start.sh
```

- 재부팅 직후엔 2주차 Grafana·Prometheus 파드가 다시 뜨는 데 1~3분 걸린다. 스크립트가 **준비될 때까지 기다렸다가** port-forward를 시작한다 (11/09 수정)
- port-forward는 파드가 재시작돼서 끊기면 **3초 뒤 자동으로 다시 연결**된다
- port-forward가 이 터미널 창 **뒤에서** 돌기 때문에, 스크립트를 실행한 **창을 닫으면 3001·9091 연결이 끊긴다.** 창은 열어둘 것
- 다시 연결하고 싶으면 스크립트를 한 번 더 실행하면 됨 (이전 연결은 자동으로 정리)

---

## 방법 B · 직접 하나씩

### 1. OrbStack 실행

```bash
open -a OrbStack
```

10~20초 뒤 Docker 확인:

```bash
docker ps
```

### 2. 1주차 스택 (Docker compose)

```bash
cd ~/server-study/week1
```

```bash
docker compose up -d
```

`restart: unless-stopped` 덕분에 보통 이미 떠 있음. 떠 있으면 아무 일도 안 하니 습관처럼 쳐도 됨.

### 3. 2주차 클러스터 (k3d)

```bash
k3d cluster start lab
```

```bash
kubectl get nodes
```

노드 3개 Ready 확인 후 파드 상태:

```bash
kubectl get pods -A
```

### 4. port-forward (새 탭 2개, 켜둔 동안만 연결)

탭 1 (`Cmd+T`) — 2주차 Grafana:

```bash
kubectl port-forward -n monitoring svc/kps-grafana 3001:80
```

탭 2 (`Cmd+T`) — 2주차 Prometheus:

```bash
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9091:9090
```

---

## 접속 주소 (북마크용)

### 1주차 · Docker compose

| 서비스 | 주소 | 무엇을 보는 곳인가 |
|---|---|---|
| Grafana | http://localhost:3000 | 대시보드 (`week1 서버 모니터링`, `Node Exporter Full`). 평소엔 여기만 |
| Prometheus | http://localhost:9090 | PromQL로 수집 데이터 직접 조회 |
| Prometheus Targets | http://localhost:9090/targets | 수집 대상이 UP/DOWN인지. 그래프가 비면 먼저 볼 곳 |
| Prometheus Alerts | http://localhost:9090/alerts | 알림 규칙 상태 (Inactive / Pending / Firing) |
| Alertmanager | http://localhost:9093 | 알림이 실제로 도착했는지, Silence 설정 |
| cAdvisor | http://localhost:8080 | 컨테이너별 자원 사용량 화면 |

### 2주차 · k3d (port-forward 켜둔 동안만)

| 서비스 | 주소 | 무엇을 보는 곳인가 |
|---|---|---|
| Grafana | http://localhost:3001 | 쿠버네티스용 대시보드 20여 개 (클러스터·노드·파드별) |
| Prometheus | http://localhost:9091 | PromQL로 클러스터 데이터 직접 조회 |
| Prometheus Targets | http://localhost:9091/targets | ServiceMonitor로 등록된 수집 대상 (podinfo 등) |
| Prometheus Alerts | http://localhost:9091/alerts | 기본 알림 100여 개 + 직접 만든 `PodinfoDown` |

> 브라우저에서 위 주소를 모두 북마크 폴더 하나에 넣어두면, 폴더 우클릭 → **모두 열기**로 한 번에 띄울 수 있다.

---

## 로그인 정보

| 대상 | 아이디 | 비밀번호 |
|---|---|---|
| 1주차 Grafana | admin | 직접 정한 것 (처음엔 admin) |
| 2주차 Grafana | admin | 아래 명령으로 확인 |

```bash
kubectl --namespace monitoring get secrets kps-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

---

## 안 될 때 확인할 것

| 증상 | 확인 / 해결 |
|---|---|
| `Cannot connect to the Docker daemon` | OrbStack이 아직 안 켜짐 → 잠시 기다리기. 계속되면 `docker context use orbstack` |
| `kubectl`이 엉뚱한 클러스터를 봄 | `kubectl config current-context`가 `k3d-lab`인지 확인. 아니면 `kubectl config use-context k3d-lab` |
| 3001 / 9091이 안 열림 | port-forward 탭(또는 start.sh 창)이 꺼진 것 → 다시 실행 |
| port-forward가 `address already in use` | 이전 port-forward가 남아 있음 → `pkill -f "kubectl port-forward"` 후 다시 실행 |
| 1주차·2주차 Grafana가 서로 로그아웃시킴 | 1주차 compose에 `GF_AUTH_LOGIN_COOKIE_NAME=grafana_session_week1`이 있는지 확인 (study-log 트러블슈팅 14) |
| 파드가 `Pending` / `ContainerCreating`에서 오래 멈춤 | 재부팅 직후엔 1~2분 걸릴 수 있음. 계속되면 `kubectl describe pod 파드이름`의 Events 확인 |

---

## 끌 때 (선택)

맥을 그냥 꺼도 되지만, 리소스를 아끼고 싶을 때:

```bash
k3d cluster stop lab
```

```bash
cd ~/server-study/week1 && docker compose stop
```

`stop`은 멈추기만 하고 지우지 않으므로 데이터·대시보드·클러스터 설정은 그대로 남는다. (`down -v`, `k3d cluster delete`는 지우는 명령이니 주의)
