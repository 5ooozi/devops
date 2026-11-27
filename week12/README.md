# ⚙️ Week12 - GitHub Actions로 CI 만들기

> 📌 **사람은 코드만 쓰고, 검사·테스트·빌드·게시는 로봇(Actions)에게 맡긴다.**
> 그리고 검사를 통과 못 한 코드는 main에 못 들어오게 막는다.

---

## 🎯 이번 주 목표

- CI와 CD의 차이를 설명한다.
- workflow / trigger / job / step / runner 구조를 이해한다.
- push하면 자동으로 **린트 → 테스트 → 빌드 → ghcr.io 푸시**가 되는 워크플로를 만든다.
- 일부러 빨간X를 만들고 초록✓로 고쳐본다.
- 브랜치 보호 규칙으로 깨진 코드가 main에 합쳐지지 않게 막는다.

---

## 🤔 1. 왜 CI가 필요할까?

지금까지 새 버전을 배포하려면 이걸 매번 손으로 했다.

```bash
# 1. 코드 수정
docker build -t ...
docker push ...
kubectl set image ...
```

- 하루에 10번 배포하면? → 4단계를 10번 반복 😵
- 팀원이 문법 오류가 있는 코드를 main에 합쳤는데 아무도 몰랐다면? → 그대로 배포돼서 장애

👉 이걸 **로봇(자동화 시스템)에게 맡기는 것**이 CI다.

### 단어 정리

| 단어 | 뜻 |
|---|---|
| Integration | 여러 사람의 코드를 하나로 합치기 |
| Continuous | 조금씩 **자주** 합치고, 합칠 때마다 검사 |
| Pipeline | 코드 → 검사 → 테스트 → 빌드 → 게시를 차례로 통과 |
| Lint | 실행 전에 코드의 "보풀"(버그 가능성, 품질 문제)을 찾는 도구 |
| Runner | 워크플로를 실제로 실행하는 컴퓨터 |
| Actions | 깃허브에 들어 있는 자동화 기능 |

---

## 🔄 2. CI / CD

| | 이름 | 하는 일 |
|---|---|---|
| **CI** | Continuous Integration (지속적 통합) | 코드를 합칠 때마다 자동으로 검사·빌드해서 문제를 **일찍** 발견 |
| **CD** | Continuous **Delivery** (지속적 제공) | 언제든 배포 가능한 상태 유지 (운영 반영은 승인 단계가 있을 수 있음) |
| | Continuous **Deployment** (지속적 배포) | 검증을 통과한 변경을 운영까지 **자동 반영** |

> ⚠️ CI 자체가 머지를 막는 건 아니다. 막는 건 **브랜치 보호 규칙**이다. (실습 2-4)

```
                [코드 푸시]
                     ↓
 ┌──────────────── CI ────────────────┐
 │ 1. 소스 가져오기                    │
 │ 2. 검사 및 테스트 (실패 시 중단)     │
 │ 3. 이미지 빌드                      │
 │ 4. ghcr.io에 푸시                   │
 └────────────────────────────────────┘
                     ↓
 ┌──────────────── CD ────────────────┐
 │ 5. 쿠버네티스에 자동 배포 (13주차)   │
 └────────────────────────────────────┘
```

하나라도 실패하면 **그 뒤 단계는 진행하지 않는다.**

---

## 🧩 3. 워크플로 구조

### 워크플로 읽는 법: 언제(on) → 어디서(runs-on) → 무엇을(steps)

```yaml
name: Build and Push Image                # 워크플로 이름
on:                                       # 언제? (트리거)
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:                                     # 할 일들
  build:                                  # job 이름
    runs-on: ubuntu-latest                # 어디서? (깃허브가 빌려주는 리눅스)
    steps:                                # 순서대로 실행
      - uses: actions/checkout@v7         # 남이 만든 액션 재사용
      - run: ruff check . --select=E9,F   # 직접 명령 실행
```

| 개념 | 의미 |
|---|---|
| workflow | `.github/workflows/*.yml` 파일 하나 = 워크플로 하나 |
| trigger (`on`) | 언제 돌릴지 (push, pull_request, schedule, 수동) |
| job | 작업 단위. 각 job은 **깨끗한 새 서버**에서 시작. 여러 job은 기본 병렬 (`needs`로 순서 지정) |
| step | job 안의 한 단계. `uses`(남의 액션) 또는 `run`(명령) |
| runner (`runs-on`) | 실제로 돌아가는 서버. `ubuntu-latest`는 깃허브가 빌려줌 (public 저장소는 무료) |

### ✅ 초록✓ / ❌ 빨간X 는 종료 코드로 정해진다

```bash
ruff check . --select=E9,F ; echo $?
# All checks passed!
# 0        ← 0 = 성공 → 다음 단계로 (초록✓)

curl -f http://localhost:5000/nope ; echo $?
# curl: (22) The requested URL returned error: 404
# 22       ← 0이 아님 = 실패 → 거기서 멈춤 (빨간X)
```

> 💡 화면에 뭐가 찍히든 상관없다. **종료 코드가 0이냐 아니냐**만 본다.
> 그래서 CI에 넣는 도구는 실패하면 0이 아닌 값을 돌려줘야 한다. (ruff, unittest, `curl -f` 등)

### `${{ }}` 읽기

| 표현 | 의미 |
|---|---|
| `${{ github.actor }}` | 이 실행을 일으킨 사람/봇 (내 아이디) |
| `${{ github.repository_owner }}` | 저장소 주인 아이디 |
| `${{ github.event_name }}` | 무엇 때문에 돌았나 (`push`, `pull_request` ...) |
| `${{ github.ref }}` | 어느 브랜치? main push면 `refs/heads/main`, PR이면 `refs/pull/번호/merge` |
| `${{ secrets.GITHUB_TOKEN }}` | 깃허브가 실행마다 새로 주는 **임시 토큰** (끝나면 만료) |
| `${{ env.REGISTRY }}` | 위쪽 `env:`에 적은 값 |
| `${{ steps.meta.outputs.tags }}` | `id: meta` 단계가 만든 결과 (태그 목록) |

> ⚠️ 변수 문법 헷갈림 주의: compose `${REDIS_PASSWORD}` / k8s `$(REDIS_PASSWORD)` / Actions `${{ secrets.… }}`

### `uses` vs `run` vs `with`

```yaml
- uses: actions/checkout@v7
#       ─────── ──────── ──
#       만든 곳  액션 이름 버전
```

```yaml
- name: Python 준비
  uses: actions/setup-python@v7     # uses: 남이 만든 부품 호출
  with:                             # with: 그 부품에 넘기는 설정 값
    python-version: '3.12'

- name: 코드 검사 (Lint)
  run: |                            # run: 내가 직접 쓰는 명령
    pip install ruff
    ruff check . --select=E9,F
```

- `uses:` 미리 만들어진 액션(부품)을 호출
- `run:` 내가 직접 쓰는 명령
- `with:` 부품에 넘기는 설정 값 (예: `python-version: '3.12'`)

> ⚠️ 남의 코드가 **내 저장소 권한으로** 돈다. 믿을 만한 것만 쓰자. (오늘은 `actions/`, `docker/` 공식 액션만 사용)

### 문법 검사 vs 린트

```python
print(APP_TITL)     # APP_TITLE의 오타
```

- 문법 검사(compileall): **통과** (파이썬이 읽을 수는 있음) → 실행하면 실패
- 린트(ruff): 실행도 안 하고 **잡아냄**

```bash
ruff check . --select=E9,F   # E9: 문법 오류 / F: 정의 안 된 변수, 안 쓰는 import, 오타 등
```

### 태깅 전략

```yaml
tags: |
  type=sha,prefix=,format=short                        # 커밋 해시 7글자
  type=raw,value=latest,enable={{is_default_branch}}   # 기본 브랜치일 때만 latest
```

| 옵션 | 뜻 |
|---|---|
| `type=sha` | 태그를 **커밋 해시**로 만든다. SHA = Secure Hash Algorithm(해시 알고리즘)이고, 깃의 커밋 ID가 바로 이 SHA 해시값이다 |
| `prefix=` | prefix = **접두사**(앞에 붙는 글자). `type=sha`는 기본으로 앞에 `sha-`를 붙이는데(`sha-c559a29`), 빈 값으로 두면 **`sha-`를 떼고** `c559a29`만 남는다 |
| `format=short` | 40글자 전체 해시 대신 **앞 7글자**만 사용 (`format=long`이면 전체) |
| `type=raw,value=latest` | 정해진 글자(`latest`)를 그대로 태그로 사용 |
| `enable={{is_default_branch}}` | 기본 브랜치(main)일 때만 그 태그를 만든다 |

- `latest`는 계속 덮어써진다 → 어제의 latest ≠ 오늘의 latest → **롤백할 버전을 모름**
- 커밋 해시(`fe9db50`)나 버전(`v1.2`)으로 태그하면 "이 이미지는 이 커밋으로 만들었다"를 **추적**할 수 있다

### ghcr.io를 쓰는 이유

```yaml
- uses: docker/login-action@v4
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}   # 자동 제공!
```

`GITHUB_TOKEN`은 깃허브가 자동으로 발급해서 **시크릿을 따로 만들 필요가 없다.** (도커 허브는 토큰을 직접 등록해야 함)

---

## 🧪 4. 실습 1 - 첫 워크플로

### 준비

```bash
cd ~/devops
git status                                        # 커밋 안 한 변경 확인
gh repo view --json visibility -q .visibility     # PUBLIC인지 확인 (무료 러너)
```

```
~/devops/
├── .github/workflows/build.yml   # 워크플로 (꼭 저장소 루트에!)
└── guestbook/                    # 빌드 대상
    └── tests/test_app.py         # 테스트 코드
```

> ⚠️ 6주차에 CLI로 만든 guestbook 패키지는 저장소와 연결이 안 돼 있어서 푸시할 때 **403**이 날 수 있다.
> Packages → guestbook → Package settings → **Manage Actions access** → 저장소 추가 → Role **Write**

---

### 4-1. 테스트 코드 작성

```bash
git switch main && git pull --ff-only && git switch -c feature/ci-setup
mkdir -p guestbook/tests
```

`guestbook/tests/test_app.py`

```python
import unittest
import app as guestbook

class GuestbookTest(unittest.TestCase):
    def setUp(self):                                # 각 테스트 전에 실행
        guestbook.app.config.update(TESTING=True)
        guestbook._memory_entries.clear()           # 메모리 저장소 비우기
        self.client = guestbook.app.test_client()

    def test_health(self):
        response = self.client.get('/healthz')
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.get_json()['status'], 'ok')

    def test_add_and_read(self):
        response = self.client.post('/add', data={'name': 'student', 'message': 'hello'})
        self.assertEqual(response.status_code, 302)
        entries = self.client.get('/api/entries').get_json()
        self.assertEqual(entries['count'], 1)
        self.assertEqual(entries['entries'][0]['message'], 'hello')
```

| 코드 | 의미 |
|---|---|
| `import unittest` | 파이썬 기본 테스트 도구 |
| `class ...(unittest.TestCase)` | 테스트 묶음 하나 |
| `setUp` | 각 테스트 전에 호출. 방명록을 비워서 테스트끼리 영향 안 주게 함 |
| `app.test_client()` | 서버 실행 없이 요청을 보낼 수 있는 가상 브라우저 |
| `def test...` | `test`로 시작하는 함수만 테스트로 실행됨 (파일 이름도 `test_`로 시작) |
| `assertEqual(a, b)` | a ≠ b면 실패 → 종료 코드 1 → 빨간X |
| `302` | 글 등록 후 첫 화면으로 돌려보내는(redirect) 응답 코드 |

**push 전에 컨테이너로 미리 확인**

```bash
cd ~/devops/guestbook
docker run --rm -v "$PWD":/app -w /app -e PYTHONDONTWRITEBYTECODE=1 python:3.12-slim \
  sh -c "pip install -q ruff -r requirements.txt && ruff check . --select=E9,F --no-cache \
         && python -m unittest discover -s tests -v"
```
```
All checks passed!
test_add_and_read (test_app.GuestbookTest.test_add_and_read) ... ok
test_health (test_app.GuestbookTest.test_health) ... ok
Ran 2 tests in 0.038s
OK
```


> 💡 **test_app.py vs build.yml**
> - `test_app.py` = 시험 문제와 정답 ("**무엇이** 올바른 동작인가?", 파이썬)
> - `build.yml` = 그 시험을 **언제, 어디서, 어떤 순서로** 볼지 정한 지시서 (YAML)

---

### 4-2. 워크플로 작성

```bash
cd ~/devops
mkdir -p .github/workflows
nano .github/workflows/build.yml
```

```yaml
name: Build and Push Image

on:
  push:                             # main + paths 조건을 만족할 때
    branches: [main]
    paths:                          # 앱이 바뀐 경우에만
      - 'guestbook/**'
      - '.github/workflows/build.yml'
  pull_request:
    branches: [main]                # PR 검사
  workflow_dispatch:                # 수동 실행 버튼

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository_owner }}/guestbook

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read                # 코드 읽기만
      packages: write               # ghcr.io에 이미지 업로드
    defaults:
      run:
        working-directory: guestbook   # run: 명령은 guestbook/에서 실행
    steps:
      - name: 소스 코드 가져오기
        uses: actions/checkout@v7
      - name: Python 준비
        uses: actions/setup-python@v7
        with:
          python-version: '3.12'
      - name: 코드 검사 (Lint)
        run: |
          pip install ruff
          ruff check . --select=E9,F
      - name: 앱 동작 테스트
        env:
          REDIS_HOST: ''              # 메모리 모드로 테스트
        run: |
          python -m pip install -r requirements.txt
          python -m unittest discover -s tests -v
      - name: QEMU 준비
        uses: docker/setup-qemu-action@v4
      - name: Buildx 준비
        uses: docker/setup-buildx-action@v4
      - name: ghcr.io 로그인
        if: github.event_name != 'pull_request' && github.ref == 'refs/heads/main'
        uses: docker/login-action@v4
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: 이미지 태그 결정
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=,format=short
            type=raw,value=latest,enable={{is_default_branch}}
      - name: 빌드 & 푸시
        uses: docker/build-push-action@v7
        with:
          context: ./guestbook        # 앱 폴더를 빌드 재료로
          push: ${{ github.event_name != 'pull_request' && github.ref == 'refs/heads/main' }}
          tags: ${{ steps.meta.outputs.tags }}
          platforms: linux/amd64,linux/arm64
```

| 포인트 | 의미 |
|---|---|
| `paths:` | 앱 파일이 바뀐 push에만 실행 (`**` = 하위 폴더 전체) |
| `working-directory` | `run:`에만 적용, `uses:`에는 적용 안 됨 |
| `run: \|` | 여러 줄 명령. 한 줄이라도 실패하면 그 단계에서 멈춤 |
| 린트 → 테스트 순서 | **싸고 빠른 검사부터** 해서 실패를 일찍 발견 |
| QEMU / Buildx | 다른 CPU 아키텍처용(amd64 + arm64) 멀티플랫폼 이미지 빌드 |
| 로그인·푸시의 `if` | **main에 push될 때만** 로그인하고 이미지를 올림. PR에서는 빌드만 |

> ⚠️ `branches:`를 `branch:`로 쓰면 유효하지 않은 워크플로가 된다.

---

### 4-3. 커밋 → PR → 머지

변경 확인용으로 제목에 `new`를 붙였다.

```bash
sed -i 's/{{ title }} v2/{{ title }} new/g' guestbook/templates/index.html
grep '{{ title }}' guestbook/templates/index.html

git add .github guestbook/tests guestbook/templates
git commit -m "CI 파이프라인 추가"
git push -u origin feature/ci-setup
gh pr create --fill
gh pr checks --watch            # PR 만들고 조금 기다렸다가
gh pr merge --squash --delete-branch
git switch main && git pull --ff-only
```


---

### 4-4. 관찰

- Actions 탭에서 워크플로가 **초록✓** 로 바뀌는 걸 확인했다.
- step을 클릭하면 실행 로그를 볼 수 있다.
- Packages에서 **커밋 해시 태그**가 붙은 새 이미지를 확인했다.


```bash
gh run list --limit 5           # 최근 실행 5개
gh run watch                    # 진행 중인 실행을 실시간으로
gh run view --log-failed        # 실패한 단계 로그만
gh run rerun --failed           # 실패한 job만 다시 실행
gh workflow run build.yml --ref main   # 수동 실행 (workflow_dispatch)
```

> 💡 **내가 한 건 `git push`, PR, 머지뿐이다.** `docker build`도 `docker push`도 안 했는데 이미지가 올라갔다!
> 단, push만 한다고 무조건 올라가는 건 아니다. **검사·빌드 성공 + 게시 조건(main push)** 까지 만족해야 한다.

**CI가 만든 이미지를 쿠버네티스로**

```bash
kubectl set image deployment/guestbook web=ghcr.io/5ooozi/guestbook:<커밋태그>
kubectl rollout status deployment/guestbook
```

<img width="932" height="622" alt="스크린샷 2026-11-18 121958" src="https://github.com/user-attachments/assets/f30bff51-8657-4b9d-a9d8-0a45a9991352" />


제목이 **`방명록 (k8s) new`** 로 바뀐 것을 확인했다! 🎉
`docker build`, `docker push` 없이 PR 머지만으로 만들어진 이미지다.


이 마지막 단계까지 자동으로 되면 CD가 된다. → 13주차 Argo CD

---

## 🔴 5. 실습 2 - 빨간X → 초록✓

### 5-1. 문법 오류

```bash
git switch main && git pull --ff-only
git switch -c feature/ci-errors
echo "def broken(:" >> guestbook/app.py
git commit -am "문법 오류 확인"        # -am: 추적 중인 파일만 add + commit (새 파일은 X)
git push -u origin feature/ci-errors
gh pr create --fill
gh run view --log-failed
```
```
invalid-syntax: Expected a parameter or the end of the parameter list
 --> app.py:136:12
##[error]Process completed with exit code 1.
```

검사 단계에서 멈춰서 **이미지 푸시까지 가지 않았다.**


```bash
sed -i '$ d' guestbook/app.py          # 마지막 줄 삭제
git commit -am "문법 오류 수정" && git push
```


---

### 5-2. 오타 (린트가 잡음)

`index()` 함수 안에 한 줄 추가했다.

```python
print(APP_TITL)       # APP_TITLE 오타
```

```bash
python3 -m compileall guestbook/app.py ; echo $?   # 0 → 문법 검사는 통과!
git commit -am "오타 실험" && git push
```

Actions 탭 → build → **코드 검사 (Lint)** 에서 빨간X.
`compileall`은 통과시킨 코드를 **린트가 잡아냈다.** 그대로 배포됐으면 그 페이지에서 HTTP 500이 났을 것이다.


---

### 5-3. 테스트가 잡는 버그

```bash
git revert HEAD --no-edit               # 오타 되돌리기
```

`healthz`의 응답 코드를 200 → 500으로 바꿨다.

```python
return jsonify({"status": "ok", "hostname": HOSTNAME}), 500
```

```bash
git commit -am "테스트 실험" && git push
```
```
FAIL: test_health (test_app.GuestbookTest.test_health)
AssertionError: 500 != 200
FAILED (failures=1)
```

코드 검사는 **성공**, 앱 동작 테스트는 **실패**.
그대로 배포됐으면 liveness probe가 계속 실패해서 컨테이너가 재시작을 반복했을 것이다. (11주차에서 본 그거!)

---


### 📊 정리: 문법 검사 vs 린트 vs 테스트

| | 문법 검사 | 린트 | 테스트 |
|---|---|---|---|
| 묻는 것 | 파이썬이 **읽을 수** 있나? | 읽을 수는 있는데 **버그 가능성**은? | **의도한 대로 동작**하나? |
| 코드 실행 | X | X | O (함수 호출) |
| 예시 | `def broken(:` | `print(APP_TITL)` | `/healthz`가 500 |
| 속도 | 아주 빠름 | 빠름 | 조금 느림 |

> 💡 테스트를 통과했다고 버그가 없는 게 아니다. **테스트한 부분만 보장**한다.
> 기능을 추가하거나 버그를 고칠 때 테스트도 같이 늘려가야 한다.

### 🔍 로그는 아래에서 위로 읽는다

```
Run ruff check . --select=E9,F
F821 Undefined name `APP_TITL`               # 3. 원인
  --> app.py:95:11                           # 2. 어디서 (파일:줄:칸)
95 |     print(APP_TITL)
Found 1 error.
Error: Process completed with exit code 1.   # 1. 결과
```

```bash
git revert HEAD --no-edit && git push        # 고치고 초록✓ 확인
gh pr merge --squash --delete-branch
git switch main && git pull --ff-only
```

---

### 5-4. main 브랜치에 문지기 두기 (브랜치 보호)

> ⚠️ **CI 검사 실패 ≠ 머지 차단.** CI는 빨간X를 띄울 뿐이고, 머지를 막는 건 보호 규칙이다.

**방법 1. 룰세트**
Settings → Rules → Rulesets → New branch ruleset
- 이름 정하기, **Active**
- Add target → **Include default branch**
- ✅ Require a pull request before merging
- ✅ Require status checks to pass → Add checks → **build**
- Create

**방법 2. 클래식**
Settings → Branches → Add classic branch protection rule
- Branch name pattern: `main`
- ✅ Require a pull request before merging (Require approvals는 **끄기**, 혼자라 승인해줄 사람이 없음)
- ✅ Require status checks to pass before merging → **build**
- ✅ Do not allow bypassing the above settings (관리자인 나에게도 적용)


**확인**

```bash
# 룰세트
gh api "repos/$(gh repo view --json nameWithOwner --jq '.nameWithOwner')/rules/branches/main" --jq '.[].type'
# → pull_request, required_status_checks 가 보이면 OK

# 클래식
gh api "repos/$(gh repo view --json nameWithOwner --jq '.nameWithOwner')/branches/main" --jq '.protected'
# → true

# main에 직접 push 시도
touch t.txt && git add t.txt && git commit -m "test" && git push
# ! [remote rejected] main -> main (push declined due to repository rule violations)
git reset --hard origin/main      # 되돌리기
```

---

### 5-5. 깨진 코드로 PR → 머지 버튼 잠김

```bash
git switch -c feature/broken-test
echo "def broken2(:" >> guestbook/app.py
git commit -am "깨진 코드로 PR"
git push -u origin feature/broken-test
gh pr create --fill
gh pr view --web
```

**머지 버튼이 비활성화**됐다. 🔒


```bash
sed -i '$ d' guestbook/app.py
git commit -am "수정" && git push
```

PR 화면에서 검사가 자동으로 다시 돌고, 초록✓가 뜨자 머지 버튼이 활성화됐다. 🔓


> 💡 **CI의 진짜 가치는 빌드 자체가 아니라 main 브랜치를 지키는 것이다.**

---

## 📮 6. 이제 README 제출도 PR로

보호 규칙을 걸었으니 main에 직접 push가 안 된다. **보고서만 고친 PR도 검사를 통과해야 머지된다.**

```bash
git switch main && git pull --ff-only
git switch -c week12-report
mkdir -p week12 && nano week12/README.md
git add week12 && git commit -m "week12: 깃허브 액션"
git push -u origin week12-report
gh pr create --fill && gh pr checks --watch
gh pr merge --squash --delete-branch
git switch main && git pull --ff-only
git tag week12 && git push origin week12
```

> ⚠️ 태그는 **머지가 끝난 main에** 단다. squash 머지는 커밋을 새로 만들기 때문이다.

---

## 😵 7. 트러블슈팅

### 🧩 워크플로 들여쓰기 오류 (`uses:`가 step 밖으로 나감)

**사건**

`build.yml`을 nano로 옮겨 적다가 steps 앞부분을 이렇게 썼다.

```yaml
      - name: 소스 코드 가져오기
      uses: actions/checkout@v7
      - name: Python 준비
      uses: actions/setup-python@v7
      with:
        python-version: '3.12'
```

**확인**

- `uses:`, `with:`가 `- name:`의 `-`와 **같은 칸**에 있었다.
- YAML에서는 `- name:`으로 시작하는 항목 안의 키들이 `name`과 **같은 칸**에 있어야 한 step으로 묶인다.
- 이대로 push했으면 `Invalid workflow file`이 떴을 것이다.

**해결**

하위 키를 2칸 더 들여 써서 `name`이랑 줄을 맞췄다.

```yaml
      - name: 소스 코드 가져오기
        uses: actions/checkout@v7
      - name: Python 준비
        uses: actions/setup-python@v7
        with:
          python-version: '3.12'
```


**배운 점**

- YAML은 들여쓰기가 곧 구조다. 한 칸만 틀려도 워크플로가 통째로 무효가 된다.
- `defaults.run.working-directory`는 `run:`에만 적용된다. 그래서 `build-push-action`의 `context`는 저장소 루트 기준으로 `./guestbook`이라고 따로 써야 한다.


### 🧩 교안 FAQ 정리

| 증상 | 원인 | 확인 / 해결 |
|---|---|---|
| Actions 탭에 아무것도 안 뜸 | 파일이 `.github/workflows/`에 없거나 확장자가 다름 | `ls -a .github/workflows` |
| `Invalid workflow file` | YAML 들여쓰기·키 이름 오류 | 화면의 줄 번호 확인 |
| `No module named 'app'` | `working-directory` 빠짐 | `defaults:` 확인 |
| `NO TESTS RAN` 빨간X | 테스트 파일 위치·이름 (`test_`로 시작) | `guestbook/tests/test_app.py` |
| 빌드 & 푸시에서 `denied` / 403 | 패키지에 이 저장소 쓰기 권한 없음 | Manage Actions access |
| 머지했는데 이미지가 안 생김 | `paths`에 안 걸리는 변경이거나 아직 실행 중 | `gh run list` |
| 쿠버네티스에서 `manifest unknown` | 태그 오타 (7자리 커밋 해시) | Packages 화면의 태그 |
| 잘 되던 워크플로가 갑자기 실패 | `ubuntu-latest`가 26.04로 바뀌는 중 | `runs-on: ubuntu-24.04`로 고정해서 비교 |
| Settings에서 build 검사가 검색 안 됨 | 워크플로가 한 번도 실행 안 됨 | PR 올린 뒤 다시 확인 |
| 머지 버튼이 계속 회색 | 검사가 아직 실행 중 / 승인 요구가 켜져 있음 | PR 아래 Checks, Require approvals 끄기 |
| 보고서만 고친 PR이 검사를 계속 기다림 | PR 트리거에 `paths`를 씀 | `pull_request:`에서 `paths` 제거 |
| `git push`가 거부됨 (main) | 보호 규칙이 **정상 동작** 중 | PR로 올리기 |
| revert 했는데도 빨간X | 되돌릴 커밋을 잘못 고름 | `git log --oneline` |

---

## ✅ 8. 개념 체크

### Q1. CI와 CD의 차이는?

> **CI는 합칠 때 검사하고, CD는 배포까지 이어준다.**

- **CI**: 코드를 통합할 때마다 자동으로 검사·빌드해서 오류를 일찍 발견
  - 브랜치 보호 규칙과 같이 쓰면 검사 실패 코드가 main에 못 들어감
- **CD**
  - Delivery: 언제든 배포할 수 있는 상태 유지
  - Deployment: 검증 통과한 변경을 운영까지 자동 반영

### Q2. workflow, trigger, job, step, runner를 한 줄씩 설명하면?

> **파일 하나(workflow)가, 언제(trigger), 어떤 작업(job)을, 어떤 순서(step)로, 어디서(runner) 돌릴지 정한다.**

- **workflow**: `.github/workflows/*.yml` 파일 하나
- **trigger (`on`)**: 워크플로를 언제 실행할지
- **job**: 새 서버에서 시작하는 작업 단위
- **step**: job 안의 한 단계 (`uses` 또는 `run`)
- **runner**: 실제로 실행되는 서버 (`ubuntu-latest`는 깃허브가 빌려줌)

### Q3. 단계의 성공/실패는 무엇으로 판단하나?

> **명령의 종료 코드.**

- `0` → 성공 (초록✓)
- `0`이 아니면 → 실패 (빨간X)
- 화면에 찍히는 내용은 판단 기준이 아님

### Q4. 문법 검사, 린트, 테스트는 각각 무엇을 찾나?

> **읽히나? → 버그 냄새 나나? → 진짜로 잘 동작하나?**

- **문법 검사**: 파이썬이 읽을 수 있는지 (예: `def broken(:`)
- **린트**: 읽히지만 버그 가능성 있는 코드를 **실행 없이** 확인 (예: `print(APP_TITL)`)
- **테스트**: **실제로 실행**해서 의도대로 동작하는지 확인 (예: `/healthz`가 500)

### Q5. `uses:`와 `run:`의 차이는?

> **남이 만든 부품을 쓰느냐, 내가 명령을 직접 쓰느냐.**

- **uses**: 다른 사람이 만든 액션(부품)을 가져다 씀 (예: `actions/checkout@v7`)
- **run**: 내가 쓴 셸 명령을 실행 (예: `ruff check .`)

### Q6. `${{ secrets.GITHUB_TOKEN }}`은 누가 만들고, 언제 사라지나?

> **깃허브가 실행마다 자동 발급하고, 실행이 끝나면 만료된다.**

- 시크릿을 직접 만들 필요 없음 → ghcr.io 로그인이 편함
- 단, 쓸 권한은 `permissions:`로 줘야 함 (`packages: write`)

### Q7. PR에서는 왜 이미지를 푸시하지 않나?

> **PR은 아직 main에 합쳐지지 않은 코드라서, 빌드만 해서 문제없는지 확인한다.**

- 이미지는 main에 push되거나, main 대상으로 수동 실행한 워크플로가 성공했을 때만 게시
- 워크플로에서 `push: ${{ github.event_name != 'pull_request' && github.ref == 'refs/heads/main' }}`로 구현

### Q8. latest 태그만 쓰면 안 되는 이유는?

> **계속 덮어써져서, 문제가 생겼을 때 이전 버전이 뭐였는지 알 수 없다.**

- 이전 버전을 찾아서 되돌리기 어려움
- 커밋 해시나 버전으로 태그해서 **추적**할 수 있게 함

### Q9. Branch Protection의 "Require status checks"는 무엇을 막나?

> **검사에 실패한 PR이 main에 머지되는 것을 막는다.**

- 검사가 통과하기 전에는 머지 버튼이 비활성화됨

### Q10. Actions 로그에서 원인을 찾을 때 어디부터 읽나?

> **맨 아래에서부터 위로.**

- 위로 올라가며 **첫 번째 오류 줄** 찾기
- **파일 이름과 줄 번호** 확인 (`app.py:95:11` = 파일:줄:칸)

---

## ✍️ 오늘의 교훈

1. 검사는 **싸고 빠른 것부터**: 문법 → 린트 → 테스트 → 빌드
2. 성공/실패는 **종료 코드**가 정한다.
3. CI는 빨간X를 띄울 뿐, **머지를 막는 건 브랜치 보호 규칙**이다.
4. 이미지는 **main에 push될 때만** 올라가고, 태그는 **커밋 해시**로 단다.
5. 로그는 **아래에서 위로** 읽는다.

---

