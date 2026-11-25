# 🐙 Week13 - GitOps (Argo CD) & 모니터링

> 📌 **Git이 곧 진실(Single Source of Truth)이다.**
> 클러스터는 항상 Git에 적힌 상태와 같아야 한다.

---

## 🎯 1. 이번 주 목표

- 지금까지 손으로 하던 배포(`kubectl set image`)를 **Argo CD로 자동화**한다.
- 배포 전략(Recreate / Rolling / Blue-Green / Canary)과 A/B 테스트를 구분한다.
- `kubectl top`, Grafana로 CPU/메모리 지표를 읽는다.

---

## 🔄 2. 전체 흐름

```
코드 수정 → PR → 머지 → GitHub Actions(빌드) → ghcr.io 이미지 게시
                                               ↓
               deployment.yaml의 image 태그를 바꾸는 PR → 머지
                                               ↓
                 Argo CD가 Git 변경 감지 → 쿠버네티스에 자동 배포
                                               ↓
                         Prometheus + Grafana로 모니터링
```

---

## ⚖️ 3. Push 방식 vs Pull 방식(GitOps)

| | Push (CI가 직접 배포) | Pull (GitOps, Argo CD) |
|---|---|---|
| 배포 주체 | 바깥의 CI 서버 | 클러스터 안의 Argo CD |
| 권한 | CI에 클러스터 접속 권한 필요 | 바깥에 권한 줄 필요 없음 |
| 배포 기록 | 파이프라인 로그 따로 관리 | **Git 히스토리 = 배포 기록** |
| 손으로 바꾸면 | 별도 감지 체계 필요 | selfHeal로 자동 원복 |
| 롤백 | 이전 파이프라인 재실행 | `git revert` |

---

## 🧪 4. 실습 기록

### 4-1. Argo CD 설치 및 접속

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w              # 전부 Running 될 때까지 대기
kubectl port-forward svc/argocd-server -n argocd 8443:443
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d ; echo
```

https://localhost:8443 에 접속해서 `admin` / 초기 비밀번호로 로그인했다.

---

### 4-2. 애플리케이션 등록

`argocd-application.yaml`은 자동으로 생기는 게 아니라 **내가 직접 작성하는 파일**이다. (처음엔 이걸 몰랐다..😅)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<내아이디>/devops.git
    targetRevision: main
    path: guestbook/k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true     # Git에서 지우면 클러스터에서도 삭제
      selfHeal: true  # 손으로 바꾸면 Git 상태로 원복
```

```bash
kubectl apply -f ~/devops/argocd-application.yaml
kubectl get application -n argocd   # Synced / Healthy 확인
```

```bash
kubectl -n argocd get application guestbook
```
```
NAME        SYNC STATUS   HEALTH STATUS
guestbook   Synced        Healthy
```

Git과 클러스터가 같은 상태(`Synced`)이고, 앱도 정상(`Healthy`)인 것을 확인했다.

> ⚠️ `guestbook/k8s/` 안에 두면 Argo CD가 자기 설정까지 관리하게 되므로 **저장소 루트에 저장**한다.

---

### 4-3. 실습 1 - Git 변경으로 스케일 (replicas 2 → 4)

`deployment.yaml`의 replicas를 4로 바꿔서 PR → 머지했다.

```
6eecb0b replicas 4 (#8)
```

**kubectl 명령 없이도** 약 3분 뒤 Pod가 4개가 되었다!

---

### 4-4. 실습 2 - 손으로 변경 (selfHeal)

```bash
kubectl scale deployment guestbook --replicas=10
```

잠깐 10개가 됐다가, Argo CD가 OutOfSync(drift)를 감지하고 **다시 4개로 원복**했다.
Git에 4개라고 적혀 있기 때문이다.

> 💡 **Git에 없는 변경은 인정되지 않는다.**

---

### 4-5. 실습 3 - 롤백 (`git revert`)

```bash
git revert "$SCALE_COMMIT" --no-edit   # 브랜치 → PR → 머지
```

```
4ad4b4f Revert "replicas 4 (#8)" (#9)
```
```bash
kubectl get pods
```
```
NAME                         READY   STATUS    RESTARTS   AGE
guestbook-7b6dfcf686-fhj57   1/1     Running   0          92m
guestbook-7b6dfcf686-p76q2   1/1     Running   0          92m
```

Argo CD가 반영해서 replicas가 2로 복구되었다.

---

### 4-6. 실습 4 - prune

`guestbook/k8s/hello.yaml`(ConfigMap)을 추가 → 머지 → `kubectl get cm hello`로 생성을 확인했다.

```
Message:  configmap/hello created
```
```bash
kubectl get cm hello
```
```
NAME    DATA   AGE
hello   1      14m
```

그다음 파일을 삭제(`git rm`) → 머지하니 `NotFound`가 떴다. 클러스터에서도 지워진 것이다.

```bash
kubectl get cm hello
```
```
Error from server (NotFound): configmaps "hello" not found
```

> 💡 클러스터에서 지울 땐 `kubectl delete`가 아니라 **Git에서 삭제**한다.

---

### 4-7. 실습 5 - 새 이미지 배포

1. `index.html` 제목 수정 → PR → 머지 → CI가 `ghcr.io/.../guestbook:<커밋해시>` 게시
2. `git switch main && git pull --ff-only` 후 `git rev-parse --short HEAD`로 **main의 해시** 확인
3. `deployment.yaml`의 `image:`를 그 태그로 수정 → PR → 머지
4. Argo CD가 롤링 업데이트 → 브라우저 새로고침해서 변경 확인

> ⚠️ 이미지를 push하는 것만으로는 Git의 `image:`가 안 바뀌어서 Argo CD는 배포하지 않는다!
> 그리고 태그는 **브랜치 해시가 아니라 머지된 main의 해시**를 써야 한다. (아래 트러블슈팅에서 제대로 당함..)

<img width="928" height="536" alt="스크린샷 2026-11-25 121840" src="https://github.com/user-attachments/assets/037d1ab3-8dfd-4d76-89a4-a9f173fa8e11" />


---

### 4-8. 정리하기

```bash
kubectl -n argocd delete application guestbook   # GitOps 관리 중단
```

애플리케이션 등록만 해제하는 거라, guestbook Pod/Service 같은 리소스는 그대로 남아 있다.

---

## 🚦 5. Argo CD 상태 읽기

| 구분 | 값 | 의미 |
|---|---|---|
| Sync | Synced | 클러스터 = Git |
| | OutOfSync | 클러스터 ≠ Git |
| Health | Healthy | 정상 |
| | Progressing | 롤링 업데이트 중 |
| | Degraded | 문제 있음 (ImagePullBackOff 등) |
| | Missing | Git엔 있는데 클러스터엔 없음 |

> 💡 **Synced + Degraded** = Git대로 정확히 배포했지만 Git 내용이 틀림 (예: 없는 이미지 태그)

---

## 🚀 6. 배포 전략

> 새 버전에 버그가 있으면? 한 번에 다 바꾸면 모든 사용자가 같은 버그를 겪는다.
> → **새 버전을 "어떻게" 내보낼지** 정하는 게 배포 전략

### 한눈에 보기
| 전략 | 비유 | 방식 | 장점 | 단점 |
|---|---|---|---|---|
| Recreate | 가게 문 닫고 인테리어 | 옛 Pod 전부 내림 → 새 Pod 올림 | 단순함, 두 버전이 절대 안 섞임 | 그 사이 **서비스 중단** |
| RollingUpdate (기본값) | 직원을 한 명씩 교대 | 새 Pod 하나씩 늘리고 옛 Pod 하나씩 줄임 | 중단 없음, 추가 자원 거의 없음, 쿠버네티스 기본 | 잠깐 **두 버전이 섞임**, 롤백이 즉시가 아님 |
| Blue/Green | 옆에 새 매장 차리고 간판만 바꿔 달기 | v2를 다 띄워두고 Service selector로 한 번에 전환 | **즉시 전환 + 즉시 롤백**, 미리 v2 확인 가능 | 두 환경 유지 동안 **자원 2배** |
| Canary | 손님 10%만 새 메뉴로 | 일부 사용자에게만 v2 → 괜찮으면 점점 늘림 | 문제가 생겨도 **피해가 일부만**, 실제 트래픽으로 검증 | 지켜볼 **모니터링 필요**, 설정이 복잡 |

### 전략별 정리

#### 1) Recreate
- 옛 버전을 모두 종료한 뒤 새 버전을 시작
- **언제?** DB 구조가 바뀌어서 두 버전이 같이 돌면 안 될 때 (+ 점검 공지)

#### 2) RollingUpdate (기본값)
- Pod를 조금씩 교체해서 서비스는 계속 응답
- 새 Pod가 Ready가 안 되면 롤링이 멈추고 **기존 Pod가 계속 응답**함
  (예: 이미지 태그 오타 → ImagePullBackOff여도 사용자는 서비스 사용 가능)
- **언제?** 평소의 작은 배포

#### 3) Blue/Green
- Blue(v1)가 손님을 받는 동안 Green(v2)을 미리 띄워 확인
- 전환은 Service selector 한 줄만 변경
  ```bash
  kubectl patch svc guestbook -p '{"spec":{"selector":{"app":"guestbook","version":"green"}}}'
  ```
- 문제가 생기면 `"version": "blue"`로 되돌리면 끝 (옛 Pod가 그대로 살아 있음)
- **언제?** 문제가 생기면 즉시 통째로 되돌려야 할 때

#### 4) Canary
- 이름 유래: 옛날 광부들이 가스를 감지하려고 카나리아를 데리고 갱도에 들어감
- 예: stable 3개(약 75%) + canary 1개(약 25%). Service가 version을 안 보고 둘 다로 보냄
- 에러율, 응답 시간을 **모니터링**해서 괜찮으면 canary를 늘리고 stable을 줄임
- **언제?** 큰 변경이라 일부 사용자에게 먼저 내보낼 때

### 🧪 A/B 테스트 (배포 전략 X, 배포 위에서 하는 실험)
| | Canary | A/B 테스트 |
|---|---|---|
| 묻는 질문 | 새 버전은 **안전한가?** | 어느 쪽이 **더 좋은가?** |
| 대상 | 무작위 일부 → 점점 늘림 | 그룹으로 나눠 고정 |
| 기간 | 몇 분 ~ 몇 시간 | 며칠 ~ 몇 주 |
| 보는 숫자 | 에러율, 응답 시간 (운영) | 클릭률, 가입, 매출 (비즈니스) |
| 끝나면 | 전체 교체 또는 롤백 | 결과 분석 후 더 좋은 쪽 선택 |

- 예: 버튼 색깔(THEME_COLOR) 중 어떤 색이 클릭을 더 많이 받을까?

> 💡 정답은 없음 **상황에 맞게** 고르면 된다.

---

## 📊 7. 모니터링

| 종류 | 무엇 | 도구 |
|---|---|---|
| 메트릭 | 시간에 따른 숫자 (CPU 80%) | Prometheus |
| 로그 | 텍스트 기록 ("ERROR: DB 연결 실패") | Loki, ELK |
| 트레이스 | 요청 하나의 여정 | Jaeger |

- 골든 시그널: 지연(latency), 트래픽(traffic), 에러(errors), 포화(saturation)
- Prometheus(수집, Pull 방식) → Grafana(시각화) → Alertmanager(알림)

```bash
minikube addons enable metrics-server
kubectl top pods          # 1m = CPU 0.1%, 28Mi ≈ 30MB

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

# 부하 걸기 → Grafana에서 CPU 변화 관찰 → 꼭 정리!
kubectl run load --image=busybox --restart=Never -- \
  sh -c "while true; do wget -q -O /dev/null http://guestbook/; done"
kubectl delete pod load
```

Grafana에서 Dashboards → Kubernetes / Compute Resources / Pod로 들어가서, namespace `default`, Last 15 minutes로 확인했다.

<img width="901" height="351" alt="스크린샷 2026-11-25 183637" src="https://github.com/user-attachments/assets/a5279b71-2330-4da9-9b28-f28236ef4840" />
<img width="910" height="355" alt="스크린샷 2026-11-25 185032" src="https://github.com/user-attachments/assets/747ead4e-a7e4-4917-8768-ec83c1e32ed9" />


---

## 😵 8. 오늘의 트러블슈팅

> **결론**
> 1. 이상하면 `~/.bash_history`에서 **실제로 친 명령**부터 확인한다. (`gir`..)
> 2. squash 머지를 하면 main에 **새 커밋 해시**가 생긴다. 이미지 태그는 그 해시를 써야 한다.
> 3. `matadata` 오타.. 11주차에 이어 또 matadata 막이럼ㅁ.. 😇

---

### 🚨 사건 1: 브랜치를 만들었는데 브랜치가 없다

교안대로 첫 줄을 치고 hello.yaml을 만들어 커밋했는데, push하려니까 에러가 났다.

```
error: src refspec ops/add-hello does not match any
```

확인해보니 나는 계속 **main**에 있었고, 커밋도 로컬 main에 들어가 있었다.

```bash
git branch --show-current     # main
git log --oneline -3          # 9da569c (HEAD -> main) hello ConfigMap 추가
```

분명 브랜치를 만들었는데..? `~/.bash_history`에서 그때 친 명령을 찾아봤다.

```bash
grep -n -B2 -A3 "add-hello" ~/.bash_history
```
```
1423:git switch main && git pull --ff-only && gir switch -c ops/add-hello
```

**`gir`** 😱 .. `git`을 `gir`로 쳐서 마지막 명령만 `command not found`로 실패한 것이었다.
앞의 `switch main`과 `pull`은 성공해서 main에 그대로 있었고, 그 상태로 커밋까지 해버렸다.

👉 다행히 push 전이라 커밋을 새 브랜치로 옮기고 main만 되돌렸다.

```bash
git switch -c ops/add-hello          # 지금 커밋 그대로 새 브랜치 생성
git branch -f main origin/main       # main 포인터만 원래대로
git push -u origin ops/add-hello
gh pr create --fill
gh pr merge --squash --delete-branch
```

`git log --oneline -1 main`이 원격과 같은 커밋(`4ad4b4f`)으로 돌아왔고, hello 커밋은 `ops/add-hello`로 옮겨져서 PR로 정상 머지됐다. ✅

> 💡 브랜치는 **커밋을 가리키는 포인터**일 뿐이다. `branch -f`는 main 포인터만 옮기는 거라 커밋은 새 브랜치에 그대로 남는다.
> (main에 있는 상태에서는 못 쓰니까 꼭 다른 브랜치로 옮긴 뒤에!)

---

### 🔍 사건 2: 머지했는데 hello가 안 생긴다

PR도 머지했고 파일도 main에 잘 들어가 있었는데..

```bash
kubectl get cm hello
```
```
Error from server (NotFound): configmaps "hello" not found
```

처음엔 Argo CD가 아직 안 읽어간 줄 알고 새로고침을 시켰다. (기본으로 약 3분마다 Git을 확인한다)

```bash
kubectl -n argocd annotate application guestbook argocd.argoproj.io/refresh=hard --overwrite
kubectl -n argocd get application guestbook     # OutOfSync / Healthy
```

계속 `OutOfSync`라서 이유를 봤다.

```bash
kubectl -n argocd describe application guestbook | grep -A10 -iE "message|condition"
```
```
Message:  Failed last sync attempt to [a255bab...]: one or more objects failed to apply (dry run) (retried 5 times).
...
Name: "", Namespace: "default" ... resource name may not be empty
```

ConfigMap 이름이 **비어 있다**고 한다. 들여쓰기 문제인 줄 알고 `cat -A`로 확인했는데..

```bash
cat -A guestbook/k8s/hello.yaml
```
```
apiVersion: v1$
kind: ConfigMap$
matadata:$          ← 😱
  name: hello$
```

**`matadata`** .. 또 이 오타다 ㅜㅜ
쿠버네티스가 모르는 필드라 통째로 무시해서 `name`까지 같이 사라진 것이었다.
👉 `metadata`로 고쳐서 다시 PR → 머지하니 `configmap/hello created`가 떴다! 🎉

> 💡 sync가 5번 실패하면 Argo CD는 같은 커밋으로 다시 시도하지 않는다. **고친 새 커밋**이 들어가야 다시 sync한다.

---

### 🤯 사건 3: 삭제 PR이 충돌남 + 삭제 커밋이 안 보임

prune 실습으로 hello.yaml을 지우는 PR을 올렸는데 머지가 안 됐다.

```
X Pull request #12 is not mergeable: the merge commit cannot be cleanly created.
deleted by us:   guestbook/k8s/hello.yaml
```

내 브랜치는 hello.yaml을 **삭제**했는데, 그사이 main에서는 오타 수정으로 같은 파일이 **수정**돼서 충돌이 난 것이다.
👉 PR 목적이 삭제니까 삭제 쪽으로 정했다.

```bash
gh pr checkout 12 && git fetch origin main && git merge origin/main
git rm guestbook/k8s/hello.yaml
git commit --no-edit
git push
```

머지하고 나서 로그를 보니 **삭제 커밋이 안 보였다..?** 😵

```bash
git show --stat 6424c8e
```
```
ops/remove hello (#12)
    * hello ConfigMap metadata 오타 수정
    * hello ConfigMap 삭제

 guestbook/k8s/hello.yaml | 6 ------
```

삭제는 제대로 됐다. **squash 머지**가 PR 안의 커밋들을 하나로 합쳐서 따로 안 보였던 것이다.
`kubectl get cm hello` → `NotFound` ✅ prune 성공!

---

### 🐛 사건 4: 새 이미지를 못 받아옴 (ImagePullBackOff)

제목을 바꾸고 `deployment.yaml`의 이미지 태그를 내 커밋 해시로 바꿨는데 새 파드가 안 떴다.

```bash
kubectl get pods -w
```
```
guestbook-774dc4d658-9nc68   0/1   ErrImagePull       0   3s
guestbook-774dc4d658-9nc68   0/1   ImagePullBackOff   0   14s
```
```bash
kubectl describe pod guestbook-774dc4d658-9nc68 | tail -15
```
```
Failed to pull image "ghcr.io/5ooozi/guestbook:c028c12": ... not found
```

분명 내 해시인데 이미지가 없다고 한다. 워크플로 파일을 열어봤다.

```bash
grep -n -iE "tags|push:|sha" .github/workflows/*.yml
```
```
60:            type=sha,prefix=,format=short
66:          push: ${{ github.event_name != 'pull_request' && github.ref == 'refs/heads/main' }}
```

- 이미지는 **main에 push될 때만** 올라간다. PR 빌드는 테스트만 한다.
- 내가 넣은 `c028c12`는 **브랜치**의 커밋 해시였다.
- squash 머지하면서 main에는 **새 해시** `c559a29`가 생겼고, 이미지도 그걸로 올라가 있었다.

👉 `c559a29`로 다시 고쳐서 PR을 올렸다.

```bash
git switch main && git pull --ff-only
git log --oneline -1                    # main 해시 확인
sed -i 's#guestbook:c028c12#guestbook:c559a29#' guestbook/k8s/deployment.yaml
```

> 💡 롤링 업데이트라 새 파드가 실패하는 동안에도 **예전 파드 2개는 계속 Running**이었다. 서비스는 안 끊겼다!

---

### 🧩 그 외에 만난 에러들

| 에러 | 원인 | 해결 |
|---|---|---|
| `the base branch policy prohibits the merge` | 브랜치 보호 규칙. 체크가 끝나기 전에 머지 시도 | `gh pr checks <번호> --watch` 후 머지 (체크가 안 돌면 `--admin`) |
| `must be on a branch named differently than "main"` | main에서 `gh pr create` 실행 | `git switch -c 브랜치` 후 PR 생성 |
| 내 브랜치가 옛날 커밋 위에 있음 | main을 pull 안 하고 브랜치 생성 | `git rebase --onto origin/main <옛날 기준 커밋>` |
| `kubectl delete -f argocd-application.yaml` 파일 없음 | 파일을 `ops/argocd-app` 브랜치에만 커밋하고 main에 머지 안 함 → main에서는 파일이 안 보임 | `kubectl -n argocd delete application guestbook`으로 이름으로 삭제 |


---

## ✍️ 오늘의 교훈

1. `&&`로 이어진 명령은 하나만 실패해도 뒤가 실행 안 된다. 실행 후 `git branch --show-current`로 **어느 브랜치인지 꼭 확인**한다.
2. 새 작업 전에는 항상 `git switch main && git pull --ff-only`부터 한다.
3. Argo CD가 이상하면 `describe application`의 **Message**부터 본다.
4. 이미지 태그는 **머지된 main의 해시**로 쓴다.
5. 제발오타좀 😇

---
