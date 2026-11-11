# 🛠️ Week11 - 쿠버네티스 설정과 상태 점검

## 1. Secret을 base64 -d로 읽은 화면

```bash
kubectl get secret guestbook-secret -o jsonpath='{.data.REDIS_PASSWORD}' | base64 -d; echo
```

<img width="1917" height="361" alt="스크린샷 2026-11-11 200042" src="https://github.com/user-attachments/assets/6a4a0c4a-09ee-4f45-9f62-63fc438ca15b" />

> ⚠️ **왜 안전하지 않을까?**
> Secret은 암호화가 아니라 base64 인코딩만 되어 있다.
> 인코딩, 디코딩은 정해진 방식대로 변환되는 것이기 때문에 규칙만 알고 있다면 키 없이 누구나 복원이 가능하다.
> Secret을 조회할 권한만 있으면 `base64 -d` 한 줄로 비밀번호가 보인다.

---

## 🎨 2. ConfigMap만 변경하여 색 바꾸기

**원래 화면**

<img width="942" height="653" alt="스크린샷 2026-11-11 134748" src="https://github.com/user-attachments/assets/80d5c872-7cba-4c61-871a-c155f5f197f1" />

**바뀐 화면** ✨

<img width="921" height="647" alt="스크린샷 2026-11-11 140637" src="https://github.com/user-attachments/assets/acdc61bd-2c14-44ad-882f-9d23d36663c4" />

```bash
kubectl edit configmap guestbook-config # THEME_COLOR 수정함
kubectl rollout restart deployment guestbook
```

`kubectl edit`으로 ConfigMap의 `THEME_COLOR`만 수정하였다.
이미지를 다시 굽지 않고 설정만 바꿨는데 색이 바뀌었다!
환경변수는 파드가 켜질 때 한 번만 읽기 때문에 `rollout restart`로 파드를 다시 띄워야 반영된다.

---

## 3. readiness / liveness 실패

### ⏳ readiness 실패 (READY 0/1)

<img width="830" height="47" alt="스크린샷 2026-11-11 194203" src="https://github.com/user-attachments/assets/7b589f91-a0f7-47e3-a8a6-e8247fba37e5" />

파드는 살아 있지만(Running) 준비 검사에 실패해서 `READY 0/1`이 되었다.
준비 안 된 파드에는 서비스가 트래픽을 보내지 않는다.

### 🔁 liveness 실패 (RESTARTS 증가)

<img width="842" height="160" alt="스크린샷 2026-11-11 195328" src="https://github.com/user-attachments/assets/4b6444c4-bc1f-42db-b52d-47212401b231" />

```bash
kubectl patch deployment guestbook -p '{"spec":{"template":{"spec":{"containers":[{"name":"web","livenessProbe":{"httpGet":{"path":"/devops","port":5000},"periodSeconds":5,"failureThreshold":2}}]}}}}'
```

검사 경로를 없는 주소(`/devops`)로 바꾸니 404가 나서, 앱은 멀쩡한데도 계속 재시작되어 `RESTARTS`가 올라갔다.

> 💡 **probe 경로를 잘못 쓰면 멀쩡한 앱도 계속 재시작된다.**

---

## 🧭 4. 트러블슈팅 3단 루틴

| 단계 | 명령어 | 보는 것 |
|---|---|---|
| 1️⃣ 상태 확인 | `kubectl get pods` | STATUS, READY, RESTARTS → **무슨 문제인지** |
| 2️⃣ 이유 확인 | `kubectl describe pod <파드이름>` | 맨 아래 **Events** → **왜 그런지** (대부분 정답이 이곳에!) |
| 3️⃣ 앱 확인 | `kubectl logs <파드이름>` | **앱이 뭐라고 하는지** |
| | `kubectl logs <파드이름> --previous` | 재시작 전 로그 (CrashLoopBackOff일 때) |

> 📌 **get으로 무슨 문제인지 보고 → describe로 왜 그런지 보고 → logs로 앱이 뭐라는지 본다!**

---

## 😵 5. 오늘의 트러블슈팅

### 🚨 사건: 계속 "메모리 모드로 동작합니다"가 뜸

**logs** - 앱 로그부터 확인했다.

```bash
kubectl logs -l app=guestbook | grep Redis
```
```
[WARN] Redis 연결 실패 (Error 111 connecting to redis:6379. Connection refused.) → 메모리 모드로 동작합니다.
```

`Connection refused`는 redis 서비스는 찾았지만, 그 뒤에서 연결을 받아줄 Redis 파드가 없다는 뜻이다.
(가게 주소는 찾았는데 문이 닫혀 있는 상황🚪)
문제는 guestbook이 아니라 Redis 쪽이라고 판단하고, Redis 파드를 3단 루틴대로 확인했다.

---

### 🔍 1단계 - Redis 파드 상태 확인

```bash
kubectl get pods
kubectl logs redis-5d4d98b4-w59sf
```
```
Error from server (BadRequest): container "reids" in pod "redis-5d4d98b4-w59sf" is waiting to start: trying and failing to pull image
```

컨테이너 이름이 **`reids`** 😱.. 오타 ㅜㅜ 난리

---

### 🔎 2단계 - 어느 파일에 오타가 있는지 찾기

에러에는 파일 이름이 안 나와서 어디가 틀린 건지 한참 헤맸다.
파드 이름이 `redis-...`니까 redis 디플로이먼트를 만든 `k8s/redis.yaml`이었다.

```bash
grep -rn "reids\|image:" k8s/                 # 오타 위치 찾기
```

`reids` → `redis`로 고쳤다.

---

### 😩 3단계 - 파일을 다 고쳤는데 계속 실패

**첫 번째..: 고치기만 하고 적용을 안 했다**
파드 이름이 계속 `redis-5d4d98b4-w59sf` 그대로였다.
👉 파일을 고쳐도 `kubectl apply`를 해야 클러스터에 반영된다.

**두 번째..: apply가 거부됐다**

```bash
kubectl apply -f k8s/redis.yaml
```
```
service/redis configured
The Deployment "redis" is invalid: spec.selector: Invalid value: {"matchLabels":{"app":"redis"}}: field is immutable
```

처음 만들 때 selector에도 오타가 있었는데, selector는 한 번 만들면 수정이 안 되는 항목이었다.
👉 지우고 새로 만들었다.

```bash
kubectl delete deployment redis
kubectl apply -f k8s/redis.yaml
```

**세 번째..: Redis는 정상인데 또 WARN** 🤯

```bash
kubectl get pods -l app=redis      # Running 1/1 ✅
kubectl get endpoints redis        # 10.244.0.11:6379 ✅
kubectl rollout restart deployment/guestbook
kubectl logs -l app=guestbook | grep Redis    # 또 WARN...?
```

redis 파드도 Running, 서비스도 연결돼 있는데 또 실패 로그가 나왔다.
그래서 `-l`로 묶어 보지 않고 파드 하나를 지정해서 로그를 봤다.

```bash
kubectl logs guestbook-69c7b4879d-8gc5p
```
```
[INFO] Redis 연결 성공: redis:6379
[INFO] 방명록 (k8s) 시작 — http://0.0.0.0:5000
```

🎉 **이미 연결 성공한 상태였다!**
`kubectl logs -l`은 아직 종료 중인 **예전 파드 로그**까지 같이 보여줘서 헷갈렸던 것이다.

---

### 🐛 그 외에 만난 에러들

| 에러 | 원인 | 해결 |
|---|---|---|
| `resource name may not be empty` | configmap.yaml에 `matadata` 오타 | `metadata`로 수정 |
| `unknown field "...resources.requsets"` | deployment.yaml에 `requsets` 오타 | `requests`로 수정 |
| `unknown field "...livenessProbe.readinessProbe"` | readinessProbe가 livenessProbe 안으로 들여쓰기 됨 | 같은 칸으로 맞춤 |
| `connection refused` (kubectl) | 컴퓨터 재부팅 후 minikube가 꺼짐 | `minikube start` |

> 💡 apply 에러 메시지의 **경로(`spec.template.spec...`)를 따라가면** yaml 어디가 틀렸는지 찾을 수 있다!

---

## ✍️ 오늘의 교훈

1. 문제가 생기면 **get → describe → logs** 순서로 본다.
2. 파일을 고쳤으면 **꼭 apply**하고, 파드 이름이 바뀌었는지 확인한다.
3. **selector는 수정 불가**라서 틀리면 지우고 다시 만들어야 한다.
4. 로그가 이상하면 `-l` 말고 **파드 하나를 지정**해서 본다.
5. 오타 조심... 진짜로 😇
