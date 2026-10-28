# DevOps 08. 쿠버네티스

## 1. 실습 결과

### `kubectl get nodes` 결과
> 사용 도구: **minikube**

<img width="662" height="65" alt="kubectl get nodes 결과" src="https://github.com/user-attachments/assets/d022180d-5dfb-40e3-9f10-017f9ea3ff6b" />

### Pod 삭제 후 새 Pod가 다시 생성되는 화면 (`kubectl get pods -w`)

<img width="897" height="206" alt="get pods -w 화면" src="https://github.com/user-attachments/assets/025df973-c620-4822-b178-605c28845f3c" />
<img width="872" height="896" alt="Pod 재생성 화면" src="https://github.com/user-attachments/assets/725c0641-2769-4bc7-8207-a6454581555a" />

### 쿠버네티스에서 실행한 방명록 브라우저 화면

<img width="931" height="622" alt="방명록 브라우저 화면" src="https://github.com/user-attachments/assets/b21b02b0-51e2-4e05-81e5-a27c9416c48a" />

---

## 2. "선언형"의 의미

쿠버네티스는 명령형 명령어도 지원하지만, 핵심은 원하는 상태를 선언하고 그 상태를 계속 유지하는 선언형 방식임.

| 구분 | 명령형 (Imperative) | 선언형 (Declarative) |
|---|---|---|
| 방식 | **"어떻게"** 할지를 하나하나 지시함 | **"무엇을"** 원하는지만 선언함 |
| 컨테이너가 죽으면 | 사람이 직접 다시 run 해야 함 | 쿠버네티스가 알아서 다시 살림 (자가 치유) |
| 상황이 바뀌면 | 그때마다 계속 지시해야 함 | 현재 상태를 계속 확인하고, 목표와 다르면 알아서 맞춤 |

### 🍝 식당 비유 - 음식 주문

- **명령형**: "팬에 올리브유 두르고, 마늘이랑 페페론치노 볶다가, 새우 넣고, 삶은 면 넣어서 볶아주세요~!!"
- **선언형**: (메뉴판 보고) "쉬림프 알리오 올리오 하나 주세요!"

→ 선언형은 결과만 말하면 셰프가 알아서 만들어줌.

### 💧 식당 비유 - 물 채우기

- **명령형**: 물병이 빈 걸 보고 "저기요, 물 좀 채워주세요~!" → 채워줌 → 다 마심 → "물 채워주세요!" → 계속 반복함
- **선언형**: "우리 테이블 물병은 항상 차 있게 해주세요!" → 직원이 물병을 계속 확인하다가 비면 알아서 채워줌

→ 쿠버네티스가 Pod 개수를 계속 세다가 부족하면 다시 띄우는 것과 같음.

---

### 🚨 현상
minikube에 guestbook 이미지를 배포하는 실습 중에 Pod가 `ErrImagePull` → `ImagePullBackOff` 상태를 반복함.

### 🔍 원인
- `kubectl describe pod`로 Events를 확인함. kubelet이 `ghcr.io/5oozi/guestbook:v1`을 ghcr에서 pull하려다 `403 Forbidden`으로 실패함.

  <img width="935" height="390" alt="describe pod 이벤트" src="https://github.com/user-attachments/assets/4f7b5acd-ca24-466d-8891-64f821c1e975" />

- 디플로이먼트의 이미지 경로가 실제 GitHub 아이디(`5ooozi`)와 달랐음(`5oozi`). 그래서 minikube 안에 있는 이미지와 이름이 맞지 않았음.
- minikube 안에 같은 이름의 이미지가 없으니, 로컬 이미지를 쓰지 않고 외부 레지스트리(ghcr)에서 받으려 함. 그런데 존재하지 않는 이미지라서 거부됨.

### ✅ 해결 방법
1. `kubectl set image`로 디플로이먼트의 이미지 경로를 실제 이름으로 수정함.
   ```bash
   kubectl set image deployment/guestbook guestbook=ghcr.io/5ooozi/guestbook:v1
   ```

### 💡 배운 점
- `minikube image load`는 로컬 도커 이미지를 minikube로 복사만 하는 명령임. 디플로이먼트의 이미지 이름과 한 글자라도 다르면 이미지를 못 찾아서 외부(ghcr)에서 pull을 시도함.
- ghcr의 `403` 에러는 이미지가 private이거나 존재하지 않을 때 발생함.
- 이미지 관련 에러는 `kubectl describe pod`의 Events부터 확인하면 원인을 빨리 찾을 수 있음.
