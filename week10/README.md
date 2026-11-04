# Week10 - 쿠버네티스 배포와 서비스

## 1. replicas 5로 늘리기

```bash
kubectl scale deployment/guestbook --replicas=5
kubectl get pods
```

guestbook 파드 5개가 모두 Running 상태가 된 것을 확인했다.

<img width="901" height="73" alt="스크린샷 2026-11-04 201223" src="https://github.com/user-attachments/assets/43f2f019-2c60-4189-9cf5-3a0a7efc8c25" />
<img width="805" height="161" alt="스크린샷 2026-11-04 201247" src="https://github.com/user-attachments/assets/1d513504-f0ca-474d-846d-45585ca91413" />

---

## 2. 서비스를 통과한 요청에서 서로 다른 Pod hostname 확인

브라우저에서 새로고침할 때마다 화면의 Pod/Host 값이 바뀌는 것을 확인했다.
서비스가 요청을 여러 Pod로 나눠서 보내기 때문이다.

<img width="542" height="102" alt="스크린샷 2026-11-04 201358" src="https://github.com/user-attachments/assets/dee4fa1d-3de5-4a4c-a2d9-13a73095bcfc" />
<img width="552" height="107" alt="스크린샷 2026-11-04 201407" src="https://github.com/user-attachments/assets/5084254d-a749-4da0-b191-79e1a50686ae" />
<img width="528" height="96" alt="스크린샷 2026-11-04 201419" src="https://github.com/user-attachments/assets/c26e7db9-92ff-488e-b051-0f2c5cf53cf7" />
<img width="547" height="96" alt="스크린샷 2026-11-04 201427" src="https://github.com/user-attachments/assets/1ac30ed6-4e44-41c2-88d6-ea59f31b8c1f" />
<img width="532" height="96" alt="스크린샷 2026-11-04 201433" src="https://github.com/user-attachments/assets/6e51c497-59c4-48dd-b173-134fedeec5f3" />

---

## 3. Pod 강제 삭제 → 자동 복구 확인

### Pod 삭제

```bash
kubectl delete pod guestbook-79fff89bcb-nfxzb
```

guestbook-79fff89bcb-nfxzb를 삭제했다.

<img width="892" height="65" alt="스크린샷 2026-11-04 201605" src="https://github.com/user-attachments/assets/5f576bc8-18a4-4e8a-b608-c83d9868b392" />

### 자동 복구

```bash
kubectl get pods
```

새로운 파드 guestbook-79fff89bcb-ghxgh가 자동으로 생성되어 다시 5개가 유지되었다.

<img width="868" height="166" alt="스크린샷 2026-11-04 201700" src="https://github.com/user-attachments/assets/d5420ff0-91a8-4192-951e-35c6271901df" />

---

## 4. v1 → v2 롤링 업데이트

```bash
kubectl set image deployment/guestbook web=ghcr.io/5ooozi/guestbook:v2
kubectl rollout status deployment/guestbook
kubectl annotate deployment/guestbook kubernetes.io/change-cause="v1 -> v2 롤링 업데이트" --overwrite
kubectl rollout history deployment/guestbook
```

<img width="897" height="196" alt="스크린샷 2026-11-04 200751" src="https://github.com/user-attachments/assets/7605d680-be11-40eb-98af-c5ca0ca05c25" />
<img width="925" height="637" alt="스크린샷 2026-11-04 200646" src="https://github.com/user-attachments/assets/ee923bc5-5db7-49ee-938a-72041261e035" />

---

## 5. 롤백

```bash
kubectl rollout undo deployment/guestbook
kubectl annotate deployment/guestbook kubernetes.io/change-cause="v1: 롤백 완료" --overwrite
kubectl rollout history deployment/guestbook
```

<img width="902" height="213" alt="스크린샷 2026-11-04 200838" src="https://github.com/user-attachments/assets/1da60a48-15a6-4d6c-835f-355d25b3a111" />
<img width="920" height="612" alt="스크린샷 2026-11-04 200908" src="https://github.com/user-attachments/assets/b044d9a0-b4d3-48ab-bdd9-4edc7e00937e" />

---

## 😵 오늘의 트러블슈팅..

> **결론**
> 1. `docker build`는 기본으로 `Dockerfile`을 쓴다.
> 2. 같은 태그로 다른 이미지를 빌드하면 minikube가 예전 이미지를 계속 써서 반영 안된다.  
>    minikube에 이미지가 캐시로 남아 있기에 `minikube image rm`으로 지우고 `minikube image load`로 다시 넣어야 한다.

---

### 알아간 과정..

교안과 똑같이 inhatc devops 방명록을 실습때 쓰고싶었는데 `index.html`을 아무리 수정하고 v2로 굽고 푸시하고 서비스를 실행해도 다른 방명록이 나왔다.

<img width="887" height="587" alt="스크린샷 2026-11-04 194420" src="https://github.com/user-attachments/assets/6d9ac714-05e7-40af-b390-ac89b9366366" />

뭔가 잘못된 것을 느껴 ai의 도움을 받아

```bash
kubectl exec <실습때 생성했던 파드이름> -- cat templates/index.html > templates/index.html
```

이렇게 덮어씌우고 다시 굽고 깃에 올리고 서비스를 실행했는데도 똑같이 오지의 방명록이 나오는것이다!! ㅠㅠ

---

### v1 이미지 확인

그래서 내가 처음부터 v2를 오지의 방명록으로 잘못 굽고 또 덮어씌워서 그대로인것이라 생각해서  
v1 이미지를 꺼내 임시파일에 먼저 받아서 확인해봤다.

```bash
cd ~/devops/guestbook
docker run --rm ghcr.io/5ooozi/guestbook:v1 cat templates/index.html > /tmp/index_v1.html
nano /tmp/index_v1.html
```

열어보니 분명 교안의 inhatc devops 방명록이 맞았다.

```bash
cp /tmp/index_v1.html templates/index.html
```

이렇게 원래 자리에 복사하고

```bash
docker build -t ghcr.io/5ooozi/guestbook:v2 .
docker push ghcr.io/5ooozi/guestbook:v2
```

다시 v2로 굽고

```bash
minikube image rm ghcr.io/5ooozi/guestbook:v2
minikube image load ghcr.io/5ooozi/guestbook:v2
```

예전 v2는 지우고 새 v2 넣고

```bash
kubectl set image deployment/guestbook web=ghcr.io/5ooozi/guestbook:v1
kubectl rollout status deployment/guestbook

kubectl set image deployment/guestbook web=ghcr.io/5ooozi/guestbook:v2
kubectl rollout status deployment/guestbook

kubectl annotate deployment/guestbook kubernetes.io/change-cause="v1 -> v2 롤링 업데이트"
```

롤링 업데이트도 하고!!

...

<img width="887" height="587" alt="스크린샷 2026-11-04 194420" src="https://github.com/user-attachments/assets/e787b13f-ad6d-43c3-a207-caa9b68fce15" />

ㅠㅠ 오지의 방명록 v2가 되었다..

---

### 롤백

일단 반틈 포기하고 v1으로 롤백했다

```bash
kubectl rollout undo deployment/guestbook
kubectl annotate deployment/guestbook kubernetes.io/change-cause="v1: 롤백 완료" --overwrite
kubectl rollout history deployment/guestbook
```

<img width="887" height="572" alt="스크린샷 2026-11-04 194636" src="https://github.com/user-attachments/assets/63c8a45d-0e02-426a-9583-129e34e690b9" />

???  
내가 계속 찾던 inhatc devops가 나왔다

v1에는 분명 교안의 inhatc devops 방명록이 들어있는데 v1의 index.html을 가져와서 v2를 다시 구워도 자꾸 오지의 방명록이 나오는 이유를 정말 모르겠어서 그만 포기하고싶었는데 지금까지 삽질한게 아까워서 클로드에게 폭풍질문을 하고..

---

### 🔍 원인 발견

```bash
grep -n "오지\|InhaTC" -r .
```

어느 파일에 오지 방명록과 inhatc 방명록이 있는지 가져와보았더니

```
./Dockerfile:26:ENV APP_TITLE="오지의 멋진  방명록ㅎㅎ" \
./Dockerfile.custom:2:ENV APP_TITLE="다인이 깃허브에서 가져온  오지 방명록" \ THEME_COLOR="7C9A82"
./Dockerfile.multistage:18:ENV APP_TITLE="InhaTC DevOps 방명록" \
```

**v1은 `Dockerfile.multistage`로 구웠던거고 v2는 `Dockerfile`로 구웠던것이다.**  
`docker build`는 기본으로 `Dockerfile`을 쓰기 때문에 `-f Dockerfile.multistage`로 지정하지않는이상 내가 원하는대로 빌드될 일은 없었던것이다!!!!!! ㅠㅠㅠ

---

### 🔧 해결

```bash
docker build -f Dockerfile.multistage -t ghcr.io/5ooozi/guestbook:v2 .
docker push ghcr.io/5ooozi/guestbook:v2
```

```bash
minikube image rm ghcr.io/5ooozi/guestbook:v2
minikube image ls | grep guestbook
```

기존의 v2는 지우고 grep해서 v2가 없는지 확인하고

```bash
minikube image load ghcr.io/5ooozi/guestbook:v2
minikube image ls | grep guestbook
```

새 v2 넣고 확인하고..

```bash
kubectl set image deployment/guestbook web=ghcr.io/5ooozi/guestbook:v2
kubectl rollout status deployment/guestbook
```

롤링 업데이트 하고..  
실행했더니...

<img width="925" height="637" alt="스크린샷 2026-11-04 200646" src="https://github.com/user-attachments/assets/022a231a-9f37-4325-b8cf-931d0eab5151" />

드디어 나왔다!!!!!!!111ㅠㅠㅠㅠㅠㅠㅠㅠ

캡쳐하고.. 롤백하고.. 리드미작성하고.. 여기에 넋두리를 풀어본다..  
흑흑
