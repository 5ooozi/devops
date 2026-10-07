# week06

## 1. 내 이미지 주소

`ghcr.io/5ooozi/guestbook:v2`

## 2. 친구 이미지 실행 결과 캡쳐

<img width="1386" height="632" alt="image" src="https://github.com/user-attachments/assets/3faff8a5-8878-46f0-9975-cf6f1b2346c3" />

## 3. Dockerfile의 각 줄이 하는 일을 본인의 언어로 설명

#### `FROM python:3.12-slim`

: python3.12의 가벼운 버전으로 시작한다

#### `WORKDIR /app`

: /app 디렉터리를 생성한다. 컨테이너 내부에 작업할 폴더를 지정하고 이동함.

#### `COPY requirements.txt .`

: requirements.txt를 내 이미지안의 디렉토리로 불러온다.

#### `RUN pip install --no-cache-dir -r requirements.txt`

: 캐시 없이 requirements.txt를 설치해서 이미지에 실행한다.

#### `COPY . .`

: 나머지 모든 소스코드와 파일을 이미지 안으로 복사한다.

#### `RUN useradd -m appuser`

: appuser를 생성함. 보안을 위해 권한이 제한된 사용자 계정을 만든다.

#### `USER appuser`

: root 대신 방금 만든 appuser로 실행한다.

#### `ENV APP_TITLE="오지의 멋진  방명록ㅎㅎ" \`

#### `THEME_COLOR="#16a34a"`

: 환경변수를 이용해서 문구와 색상을 수정한다.

#### `EXPOSE 5000`

: 포트번호를 문서화 한다

#### `CMD ["python", "app.py"]`

: 컨테이너 시작할 때 자동으로 app.py를 실행한다.

## 4.빌드 캐시가 동작한 로그 (CACHED)

<img width="926" height="572" alt="image" src="https://github.com/user-attachments/assets/6e3f3782-dba8-4eb1-84d9-86eab92a4b80" />

## 5. 설정을 넣는 세 가지 방법

* **코드(app.py 수정)**

파일을 열어서 코드를 직접 고치는 방법. 우선순위가 가장 낮다.

ex) # app.py 안에서 기본값을 직접 수정

```python
APP_TITLE = os.environ.get("APP_TITLE", "오지의 기본 방명록")
```

* **env (Dockerfile의 ENV 이용)**

이미지에 아예 구워버리기. 이미지를 만들 때 기본 설정을 고정하는 방법. 코드보다 우선순위가 높다

ex)

```dockerfile
ENV APP_TITLE="오지의 멋진 방명록" \
    THEME_COLOR="#16a34a"
```

* **-e (docker run)**

컨테이너를 띄울 때(실행할 때) 바깥에서 값을 쏙 주입하는 방법. 세 가지 중 우선순위가 가장 높다.

ex)

```bash
docker run -d -p 8080:5000 -e APP_TITLE="실행할 때 바꾼 방명록 제목" ghcr.io/5ooozi/guestbook:v2
```
