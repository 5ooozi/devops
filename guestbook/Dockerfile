# ---------------------------------------------------------------
# 학생 과제: 아래 각 줄이 무엇을 하는지 주석으로 설명을 달아보세요.
# ---------------------------------------------------------------

FROM python:3.12-slim
# python3.12의 가벼운 버전으로 시작한다

WORKDIR /app
# /app 디렉터리를 생성한다. 컨테이너 내부에 작업할 폴더를 지정하고 이동함.

COPY requirements.txt .
# requirements.txt를 내 이미지안의 디렉토리로 불러온다.

RUN pip install --no-cache-dir -r requirements.txt
# 캐시 없이 requirements.txt를 설치해서 이미지에 실행한다.

COPY . .
# 나머지 모든 소스코드와 파일을 이미지 안으로 복사한다.

RUN useradd -m appuser
#  appuser를 생성함. 보안을 위해 권한이 제한된 사용자 계정을 만든다.

USER appuser
# root 말고 방금 만든 appuser로 실행한다.

ENV APP_TITLE="오지의 멋진  방명록ㅎㅎ" \
    THEME_COLOR="#16a34a"
# 환경변수를 이용해서 문구와 색상을 수정한다.

EXPOSE 5000
# 포트번호를 문서화 한다

CMD ["python", "app.py"]
# 컨테이너 시작할 때 자동으로 app.py를 실행한다.
