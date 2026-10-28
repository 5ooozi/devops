# Week 07

## 1. docker compose ps 결과 (healthy포함)

```bash
hyozi@Win11-01:~/devops/guestbook$ docker compose ps
NAME                IMAGE            COMMAND                  SERVICE   CREATED          STATUS                    PORTS
guestbook-nginx-1   nginx:alpine     "/docker-entrypoint.…"   nginx     12 minutes ago   Up 12 minutes             0.0.0.0:8080->80/tcp, [::]:8080->80/tcp
guestbook-redis-1   redis:7-alpine   "docker-entrypoint.s…"   redis     12 minutes ago   Up 12 minutes (healthy)   6379/tcp
guestbook-web-1     guestbook-web    "python app.py"          web       12 minutes ago   Up 12 minutes (healthy)   5000/tcp
guestbook-web-2     guestbook-web    "python app.py"          web       12 minutes ago   Up 12 minutes (healthy)   5000/tcp
guestbook-web-3     guestbook-web    "python app.py"          web       12 minutes ago   Up 12 minutes (healthy)   5000/tcp
```

## 2. scale후 각 웹 컨테이너 호스트명(컨테이너ID)이 바뀐 결과 캡쳐

<img width="500" alt="스크린샷 2026-10-14 120748" src="https://github.com/user-attachments/assets/a3e2191c-1210-44bf-96ad-200521e6fe45" />
<img width="500"  alt="스크린샷 2026-10-14 120757" src="https://github.com/user-attachments/assets/63839de1-71fc-4c94-9610-3b7fbe4f7ffa" />
<img width="500"  alt="스크린샷 2026-10-14 120802" src="https://github.com/user-attachments/assets/e0bb94d0-511a-4559-9343-0fb3d5bcdb31" />



## 3. down후 up과 down -v후 up 차이 비교

docker compose down를 하면 컨테이너와 네트워크가 전부 삭제됨<br>
docker compose ps로 아무것도 없는 걸 확인 가능함<br>
docker compose up -d로 다시 생성하고 브라우저에 다시 접속하면 글이 여전히 남아있는 것을 볼 수 있음<br>
컨테이너를 삭제해도 별도로 생성한 볼륨을 기본적으로 삭제하지 않기 때문임

docker compose down -v를 실행하면 볼륨까지 삭제되어 브라우저 재접속시 글이 사라져있는 것을 확인 가능함.

### down vs down -v

- **down:** 컨테이너와 네트워크만 지움. 데이터는 남아있음
- **down -v:** 볼륨까지 지움. 데이터(방명록 글)도 사라짐

## 4. .env를 Git에 올리지 않는 이유 적어보기

.env 파일 안에는 데이터베이스 비밀번호, api 키 등 외부에 노출되면 안 되는 민감한 정보가 포함되어 있는데

이를 Github 같은 공개 저장소에 올리면 탈취 및 해킹이 가능함

.env는 로컬에만 두고 .gitignore에 등록하여 Git 원격에 올리면 안댄다
