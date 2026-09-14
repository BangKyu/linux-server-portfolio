# HTTP → HTTPS Redirect 구축

기존 `site1.bangkyu.com`의 HTTP Service를 HTTPS로 자동 전환하도록 Apache Redirect를 구성하였다.

HTTP TCP 80으로 들어온 요청은 Web Page를 직접 제공하지 않고 HTTPS TCP 443으로 Redirect하며, 실제 Web Page는 TLS가 적용된 HTTPS VirtualHost에서 제공하도록 구성하였다.

```text
Client-L
192.168.111.150
        |
        | http://site1.bangkyu.com
        v
Apache HTTP :80
        |
        | 301 Moved Permanently
        v
https://site1.bangkyu.com
        |
        | TCP 443
        v
TLS
        |
        v
Apache HTTPS
        |
        v
/var/www/site1/index.html
```

---

## 1. 실습 환경

| 구분 | 값 |
|---|---|
| Server | Server-A |
| Server IP | 192.168.111.100 |
| Client | Client-L |
| Client IP | 192.168.111.150 |
| Domain | site1.bangkyu.com |
| Web Server | Apache httpd |
| HTTP Port | TCP 80 |
| HTTPS Port | TCP 443 |
| HTTPS Certificate | Self-Signed Certificate |
| Redirect Code | HTTP 301 |

기존 구성:

```text
site1.bangkyu.com
→ 192.168.111.100
```

HTTP:

```text
http://site1.bangkyu.com
→ TCP 80
```

HTTPS:

```text
https://site1.bangkyu.com
→ TCP 443
→ TLS
```

---

# 기존 HTTP VirtualHost

## 2. 기존 site1 설정

기존 `/etc/httpd/conf.d/site1.conf`:

```apache
<VirtualHost *:80>
    ServerName site1.bangkyu.com
    DocumentRoot /var/www/site1

    <Directory /var/www/site1>
        Require all granted
    </Directory>

    ErrorLog logs/site1-error.log
    CustomLog logs/site1-access.log combined
</VirtualHost>
```

기존 동작:

```text
Client
        ↓
http://site1.bangkyu.com
        ↓
TCP 80
        ↓
Apache
        ↓
DocumentRoot /var/www/site1
        ↓
index.html
```

HTTP VirtualHost가 직접 Web Page를 제공하는 구조였다.

---

# Redirect 구성

## 3. 기존 설정 백업

```bash
cp -a /etc/httpd/conf.d/site1.conf /etc/httpd/conf.d/site1.conf.before-redirect
```

설정 변경 전 원본 파일을 백업하였다.

---

## 4. HTTP VirtualHost를 Redirect 전용으로 변경

```bash
vi /etc/httpd/conf.d/site1.conf
```

변경 후:

```apache
<VirtualHost *:80>
    ServerName site1.bangkyu.com

    Redirect permanent / https://site1.bangkyu.com/

    ErrorLog logs/site1-error.log
    CustomLog logs/site1-access.log combined
</VirtualHost>
```

핵심 설정:

```apache
Redirect permanent / https://site1.bangkyu.com/
```

의미:

```text
site1.bangkyu.com으로
HTTP 요청이 들어오면

현재 HTTP Page를 제공하지 않고

https://site1.bangkyu.com/

으로 Redirect
```

`permanent`는 HTTP Status Code:

```text
301 Moved Permanently
```

를 사용한다.

---

# DocumentRoot 역할 변경

## 5. HTTP VirtualHost에서 DocumentRoot 제거

기존에는 HTTP VirtualHost에서:

```apache
DocumentRoot /var/www/site1
```

을 이용하여 직접 Web Page를 제공하였다.

Redirect 적용 후 HTTP TCP 80의 역할은:

```text
Web Page 제공
X

HTTPS Redirect
O
```

가 되었다.

따라서 HTTP VirtualHost에서는 더 이상:

```text
/var/www/site1/index.html
```

을 직접 제공하지 않는다.

---

## 6. HTTPS에서 실제 Web Page 제공

실제 Web Page는 HTTPS TCP 443에서 제공한다.

HTTPS 설정:

```text
ServerName
site1.bangkyu.com:443

DocumentRoot
/var/www/site1
```

구조:

```text
HTTP :80
→ Redirect 전용
```

```text
HTTPS :443
→ 실제 Web Service
→ /var/www/site1
```

즉 Web File 자체를 제거한 것이 아니라 실제 Page를 제공하는 역할을 HTTP에서 HTTPS로 이동시킨 것이다.

---

# Apache 설정 검사

## 7. 설정 문법 확인

```bash
httpd -t
```

정상:

```text
Syntax OK
```

설정을 적용하기 전에 Apache Configuration 문법을 먼저 확인하였다.

---

## 8. Apache 재시작

```bash
systemctl restart httpd
systemctl is-active httpd
```

Service가 정상적으로 실행되는 것을 확인하였다.

---

# HTTP Redirect 검증

## 9. HTTP Header 확인

Server-A:

```bash
curl -I http://site1.bangkyu.com
```

실제 결과:

```text
HTTP/1.1 301 Moved Permanently
Date: Mon, 14 Sep 2026 06:10:38 GMT
Server: Apache/2.4.62 (Rocky Linux) OpenSSL/3.5.5
Location: https://site1.bangkyu.com/
Content-Type: text/html; charset=iso-8859-1
```

핵심:

```text
HTTP/1.1 301 Moved Permanently
```

```text
Location: https://site1.bangkyu.com/
```

Apache가 HTTP 요청에 Web Page를 직접 반환하지 않고 HTTPS URL을 Client에게 전달하였다.

---

# Redirect Follow Test

## 10. Redirect를 따라 HTTPS 접속

```bash
curl -k -L http://site1.bangkyu.com
```

`-L`:

```text
HTTP Redirect 자동 Follow
```

`-k`:

```text
Self-Signed Certificate의
신뢰 검증 생략
```

실제 결과:

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Site 1</title>
</head>
<body>
    <h1>site1.bangkyu.com</h1>
    <p>Apache VirtualHost - SITE 1</p>
</body>
</html>
```

전체 동작:

```text
HTTP
        ↓
301 Redirect
        ↓
HTTPS
        ↓
TLS
        ↓
SITE 1
```

가 정상적으로 수행되었다.

---

# 다른 VirtualHost 영향 확인

## 11. SITE 2 HTTP 확인

```bash
curl -I http://site2.bangkyu.com
```

실제 결과:

```text
HTTP/1.1 200 OK
Date: Mon, 14 Sep 2026 06:11:09 GMT
Server: Apache/2.4.62 (Rocky Linux) OpenSSL/3.5.5
Last-Modified: Mon, 14 Sep 2026 04:00:53 GMT
ETag: "c2-65b697b063a9b"
Accept-Ranges: bytes
Content-Length: 194
Content-Type: text/html; charset=UTF-8
```

SITE 2는 Redirect되지 않고 기존 HTTP Service를 정상적으로 제공하였다.

---

## 12. 기존 www Site 확인

```bash
curl -I http://www.bangkyu.com
```

실제 결과:

```text
HTTP/1.1 200 OK
Date: Mon, 14 Sep 2026 06:11:14 GMT
Server: Apache/2.4.62 (Rocky Linux) OpenSSL/3.5.5
Last-Modified: Mon, 14 Sep 2026 03:16:42 GMT
ETag: "f8-65b68dcfdb5a2"
Accept-Ranges: bytes
Content-Length: 248
Content-Type: text/html; charset=UTF-8
```

기존 `www.bangkyu.com` 역시 영향을 받지 않았다.

따라서:

```text
site1.bangkyu.com
→ HTTP 301 Redirect

site2.bangkyu.com
→ HTTP 200

www.bangkyu.com
→ HTTP 200
```

로 Site별 설정이 정상적으로 분리되었다.

---

# Client-L Redirect 검증

## 13. Client-L에서 HTTP 요청

Client-L에서:

```bash
curl -I http://site1.bangkyu.com
```

HTTP 요청이 HTTPS로 Redirect되는 것을 확인하였다.

---

## 14. Client-L에서 Redirect Follow

```bash
curl -k -L http://site1.bangkyu.com
```

흐름:

```text
Client-L
192.168.111.150
        ↓
HTTP :80
        ↓
301
        ↓
HTTPS :443
        ↓
TLS
        ↓
SITE 1
```

Client에서도 실제 Redirect 이후 HTTPS Page가 정상적으로 출력되었다.

---

# HTTP Access Log

## 15. site1 HTTP Access Log 확인

Server-A:

```bash
tail -n 10 /var/log/httpd/site1-access.log
```

실제 결과:

```text
192.168.111.100 - - [14/Sep/2026:13:04:40 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
192.168.111.100 - - [14/Sep/2026:13:04:55 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
192.168.111.150 - - [14/Sep/2026:13:08:36 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
192.168.111.100 - - [14/Sep/2026:15:10:38 +0900] "HEAD / HTTP/1.1" 301 - "-" "curl/7.76.1"
192.168.111.100 - - [14/Sep/2026:15:11:03 +0900] "GET / HTTP/1.1" 301 234 "-" "curl/7.76.1"
192.168.111.150 - - [14/Sep/2026:15:13:46 +0900] "HEAD / HTTP/1.1" 301 - "-" "curl/7.76.1"
192.168.111.150 - - [14/Sep/2026:15:13:52 +0900] "GET / HTTP/1.1" 301 234 "-" "curl/7.76.1"
```

Redirect 적용 전 Client-L 요청:

```text
192.168.111.150
GET /
200
```

Redirect 적용 후:

```text
192.168.111.150
GET /
301
```

로 변경되었다.

---

# HTTPS Access Log

## 16. SSL Access Log 확인

```bash
tail -n 10 /var/log/httpd/ssl_access_log
```

실제 결과:

```text
192.168.111.100 - - [14/Sep/2026:14:57:34 +0900] "GET / HTTP/1.1" 200 194
192.168.111.150 - - [14/Sep/2026:14:59:05 +0900] "GET / HTTP/1.1" 200 194
192.168.111.100 - - [14/Sep/2026:15:11:03 +0900] "GET / HTTP/1.1" 200 194
192.168.111.150 - - [14/Sep/2026:15:13:52 +0900] "GET / HTTP/1.1" 200 194
```

Client-L:

```text
192.168.111.150
```

의 요청이 HTTPS Log에서도 확인되었다.

HTTP:

```text
301
```

HTTPS:

```text
200
```

의 연속적인 동작이 실제 Log에서 확인되었다.

---

# Status Code 흐름

## 17. Redirect 전

```text
Client
        ↓
HTTP :80
        ↓
GET /
        ↓
200 OK
        ↓
Web Page
```

---

## 18. Redirect 후

```text
Client
        ↓
HTTP :80
        ↓
GET /
        ↓
301 Moved Permanently
        ↓
Location: https://site1.bangkyu.com/
        ↓
HTTPS :443
        ↓
GET /
        ↓
200 OK
        ↓
Web Page
```

---

# HEAD와 GET

## 19. curl -I

```bash
curl -I http://site1.bangkyu.com
```

`-I`는 HTTP Header를 확인하기 위한 요청이다.

Log:

```text
"HEAD / HTTP/1.1" 301
```

즉 Response Body 전체를 가져오지 않고 Header 중심으로 Redirect 상태를 확인하였다.

---

## 20. curl -L

```bash
curl -L http://site1.bangkyu.com
```

`-L`은 Server가 반환한 `Location` Header를 따라 새로운 URL로 다시 요청한다.

이번 실습에서는 Self-Signed Certificate를 사용하므로:

```bash
curl -k -L http://site1.bangkyu.com
```

을 사용하였다.

---

# HTTP 301

## 21. 301 Moved Permanently

HTTP Status Code:

```text
301
```

의미:

```text
요청한 Resource가
새로운 URL로 영구적으로 이동
```

이번 실습:

```text
기존

http://site1.bangkyu.com
```

새로운 위치:

```text
https://site1.bangkyu.com/
```

Apache는 `Location` Header를 이용하여 새로운 주소를 Client에게 알려준다.

---

# HTTP 200

## 22. 200 OK

HTTPS Redirect 이후:

```text
"GET / HTTP/1.1" 200 194
```

가 확인되었다.

`200`:

```text
Client 요청이 정상 처리됨
```

즉 Redirect된 HTTPS 요청이 실제 Web Page까지 정상적으로 도달하였다.

---

# HTTP와 HTTPS 역할 분리

## 23. HTTP :80

```text
site1.bangkyu.com:80
```

역할:

```text
Web Page 직접 제공
X

HTTPS Redirect
O
```

설정:

```apache
Redirect permanent / https://site1.bangkyu.com/
```

---

## 24. HTTPS :443

```text
site1.bangkyu.com:443
```

역할:

```text
TLS 암호화
O

실제 Web Page 제공
O
```

DocumentRoot:

```text
/var/www/site1
```

---

# 전체 동작 구조

## 25. 최종 요청 흐름

```text
                         Client-L
                      192.168.111.150
                             |
                             | HTTP
                             v
                 http://site1.bangkyu.com
                             |
                           TCP 80
                             |
                             v
                   Apache HTTP VirtualHost
                             |
                             | 301
                             v
                 https://site1.bangkyu.com/
                             |
                           TCP 443
                             |
                             v
                       TLS Handshake
                             |
                             v
                      Apache HTTPS
                             |
                             v
                    /var/www/site1
                             |
                             v
                         index.html
                             |
                             v
                          200 OK
```

---

# 주요 명령어

## 26. 설정 백업

```bash
cp -a /etc/httpd/conf.d/site1.conf /etc/httpd/conf.d/site1.conf.before-redirect
```

---

## 27. Apache 설정 검사

```bash
httpd -t
```

---

## 28. Apache Service

```bash
systemctl restart httpd
systemctl is-active httpd
```

---

## 29. Redirect Header 확인

```bash
curl -I http://site1.bangkyu.com
```

---

## 30. Redirect Follow

```bash
curl -k -L http://site1.bangkyu.com
```

---

## 31. 다른 VirtualHost 확인

```bash
curl -I http://site2.bangkyu.com
curl -I http://www.bangkyu.com
```

---

## 32. HTTP Log

```bash
tail -n 10 /var/log/httpd/site1-access.log
```

---

## 33. HTTPS Log

```bash
tail -n 10 /var/log/httpd/ssl_access_log
```

---

# 실습 결과

```text
HTTP Redirect
✓ site1.bangkyu.com HTTP 요청 확인
✓ HTTP 301 Moved Permanently 확인
✓ Location Header 확인

HTTPS
✓ Redirect 이후 HTTPS 연결
✓ TCP 443 / TLS 사용
✓ SITE 1 Page 정상 출력
✓ HTTPS HTTP 200 확인

VirtualHost
✓ site1만 HTTPS Redirect 적용
✓ site2는 기존 HTTP 200 유지
✓ www는 기존 HTTP 200 유지

Log
✓ Client-L HTTP 요청 확인
✓ Client-L IP 192.168.111.150 확인
✓ HTTP Access Log에서 301 확인
✓ SSL Access Log에서 200 확인
```

---

# 최종 구성

```text
site1.bangkyu.com

HTTP :80
   |
   | GET /
   v
301 Moved Permanently
   |
   | Location:
   | https://site1.bangkyu.com/
   v
HTTPS :443
   |
   | TLS
   v
Apache
   |
   v
/var/www/site1/index.html
   |
   v
200 OK
```

Apache의 `Redirect permanent` 설정을 이용하여 `site1.bangkyu.com`의 HTTP 요청을 HTTPS로 자동 전환하였다.

HTTP VirtualHost는 실제 Web Page를 직접 제공하지 않고 HTTP 301과 `Location` Header를 이용하여 HTTPS URL로 Redirect하도록 역할을 변경하였다.

또한 `curl -k -L`을 이용하여 Redirect를 실제로 따라간 뒤 HTTPS Web Page가 정상적으로 출력되는 것을 확인하였다.

마지막으로 Apache HTTP Access Log와 SSL Access Log에서 Client-L의 요청이 각각 `301 → 200`으로 기록되는 것을 확인하여 HTTP → HTTPS Redirect 전체 흐름을 검증하였다.
