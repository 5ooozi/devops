# 5주차 - Docker 실습 결과

## 1. nginx1, nginx2, nginx3 컨테이너 실행 및 curl 결과
- **nginx1 (8091)**
HTTP/1.1 200 OK
Server: nginx/1.31.6
Date: Wed, 30 Sep 2026 03:35:03 GMT
Content-Type: text/html
Content-Length: 27
Last-Modified: Wed, 30 Sep 2026 03:09:24 GMT
Connection: keep-alive
ETag: "6abc7d64-1b"
Accept-Ranges: bytes


- **Nginx2 (8092)**
HTTP/1.1 200 OK
Server: nginx/1.31.6
Date: Wed, 30 Sep 2026 03:35:03 GMT
Content-Type: text/html
Content-Length: 27
Last-Modified: Wed, 30 Sep 2026 03:10:04 GMT
Connection: keep-alive
ETag: "6abc7d8c-1b"
Accept-Ranges: bytes


- **Nginx3 (8093)**
HTTP/1.1 200 OK
Server: nginx/1.31.6
Date: Wed, 30 Sep 2026 03:35:03 GMT
Content-Type: text/html
Content-Length: 27
Last-Modified: Wed, 30 Sep 2026 03:10:18 GMT
Connection: keep-alive
ETag: "6abc7d9a-1b"
Accept-Ranges: bytes


## 2. Docker PS 결과
CONTAINER ID   IMAGE     COMMAND                  CREATED              STATUS              PORTS                                     NAMES
782d92dae510   nginx     "/docker-entrypoint.…"   About a minute ago   Up About a minute   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   web
