# Apache Name-based VirtualHost 구축

하나의 Apache Web Server에서 동일한 IP와 TCP 80 Port를 사용하면서 Domain Name에 따라 서로 다른 Web Site를 제공하도록 Name-based VirtualHost를 구성하였다.

```text
site1.bangkyu.com ─┐
                   │
                   ├─ 192.168.111.100:80
                   │        ↓
site2.bangkyu.com ─┘      Apache
                            │
                    Host Name으로 구분
                    ┌───────┴───────┐
                    ↓               ↓
             /var/www/site1   /var/www/site2
```

---

## 1. 실습 환경

| 구분 | 값 |
|---|---|
| Server | Server-A |
| IP | 192.168.111.100 |
| DNS Domain | bangkyu.com |
| Web Server | Apache httpd |
| Protocol | HTTP |
| Port | TCP 80 |

구성한 Site:

```text
www.bangkyu.com
→ 192.168.111.100
→ /var/www/html

site1.bangkyu.com
→ 192.168.111.100
→ /var/www/site1

site2.bangkyu.com
→ 192.168.111.100
→ /var/www/site2
```

---

# 초기 상태 확인

## 2. Apache / DNS Service 확인

```bash
systemctl is-active httpd
systemctl is-active named
```

결과:

```text
active
active
```

Apache와 DNS Server가 모두 실행 중임을 확인하였다.

---

## 3. Apache 설정 문법 확인

```bash
httpd -t
```

초기 결과:

```text
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using fe80::20c:29ff:fedd:cda%ens160. Set the 'ServerName' directive globally to suppress this message
Syntax OK
```

Apache 설정 문법은 정상이나 Global `ServerName`이 설정되지 않아 Warning이 발생하였다.

---

## 4. 기존 DNS Zone 확인

```bash
cat /var/named/bangkyu.com.db
```

기존 설정:

```dns
$TTL 1D

@       IN      SOA     ns.bangkyu.com. admin.bangkyu.com. (
                        2026091401      ; Serial
                        1D              ; Refresh
                        1H              ; Retry
                        1W              ; Expire
                        3H )            ; Minimum TTL

@       IN      NS      ns.bangkyu.com.
@       IN      A       192.168.111.100

ns      IN      A       192.168.111.100
www     IN      A       192.168.111.100
ftp     IN      A       192.168.111.200
```

---

# DNS 설정

## 5. VirtualHost용 DNS Record 추가

`site1`과 `site2`를 동일한 Web Server IP에 연결하였다.

```bash
vi /var/named/bangkyu.com.db
```

Serial 변경:

```text
2026091401
↓
2026091402
```

추가한 A Record:

```dns
site1   IN      A       192.168.111.100
site2   IN      A       192.168.111.100
```

최종 구조:

```text
site1.bangkyu.com
→ 192.168.111.100

site2.bangkyu.com
→ 192.168.111.100
```

---

## 6. DNS Zone 검사

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db
```

실제 결과:

```text
zone bangkyu.com/IN: loaded serial 2026091402
OK
```

Zone File 문법이 정상임을 확인하였다.

---

## 7. DNS 조회 확인

```bash
dig @192.168.111.100 site1.bangkyu.com A +short
dig @192.168.111.100 site2.bangkyu.com A +short
```

결과:

```text
192.168.111.100
192.168.111.100
```

두 Domain이 모두 동일한 Web Server IP를 가리키도록 설정되었다.

---

# Web Site 구성

## 8. DocumentRoot 생성

```bash
mkdir -p /var/www/site1
mkdir -p /var/www/site2
```

구조:

```text
/var/www/
├── html/
├── site1/
└── site2/
```

---

## 9. SITE 1 Web Page

```bash
vi /var/www/site1/index.html
```

내용:

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

---

## 10. SITE 2 Web Page

```bash
vi /var/www/site2/index.html
```

내용:

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Site 2</title>
</head>
<body>
    <h1>site2.bangkyu.com</h1>
    <p>Apache VirtualHost - SITE 2</p>
</body>
</html>
```

SELinux Context 적용:

```bash
restorecon -Rv /var/www/site1
restorecon -Rv /var/www/site2
```

---

# Apache VirtualHost

## 11. SITE 1 VirtualHost

```bash
vi /etc/httpd/conf.d/site1.conf
```

내용:

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

주요 설정:

```text
<VirtualHost *:80>
→ TCP 80으로 들어오는 HTTP 요청 처리

ServerName site1.bangkyu.com
→ site1.bangkyu.com 요청을 해당 VirtualHost가 처리

DocumentRoot /var/www/site1
→ Web 문서 위치

Require all granted
→ Client의 Web 접근 허용

site1-access.log
→ SITE 1 접속 기록

site1-error.log
→ SITE 1 오류 기록
```

---

## 12. SITE 2 VirtualHost

```bash
vi /etc/httpd/conf.d/site2.conf
```

내용:

```apache
<VirtualHost *:80>
    ServerName site2.bangkyu.com
    DocumentRoot /var/www/site2

    <Directory /var/www/site2>
        Require all granted
    </Directory>

    ErrorLog logs/site2-error.log
    CustomLog logs/site2-access.log combined
</VirtualHost>
```

---

## 13. 기존 www Site 유지

기존 `www.bangkyu.com`도 `/var/www/html`을 사용하도록 구성하였다.

```bash
vi /etc/httpd/conf.d/00-www.conf
```

내용:

```apache
<VirtualHost *:80>
    ServerName www.bangkyu.com
    ServerAlias bangkyu.com

    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Require all granted
    </Directory>

    ErrorLog logs/www-error.log
    CustomLog logs/www-access.log combined
</VirtualHost>
```

구조:

```text
www.bangkyu.com
→ /var/www/html

site1.bangkyu.com
→ /var/www/site1

site2.bangkyu.com
→ /var/www/site2
```

---

# Global ServerName

## 14. Apache FQDN Warning 처리

초기 `httpd -t` 실행 시 다음 Warning이 발생하였다.

```text
AH00558: Could not reliably determine the server's fully qualified domain name
```

Global ServerName을 설정하였다.

```bash
vi /etc/httpd/conf.d/servername.conf
```

내용:

```apache
ServerName www.bangkyu.com
```

다시 검사:

```bash
httpd -t
```

결과:

```text
Syntax OK
```

Apache Configuration Warning이 해결되었다.

---

# VirtualHost 확인

## 15. Apache VirtualHost 설정 검사

```bash
httpd -S
```

실제 확인 결과:

```text
VirtualHost configuration:
*:80                   is a NameVirtualHost
         default server site1.bangkyu.com (/etc/httpd/conf.d/site1.conf:1)
         port 80 namevhost site1.bangkyu.com (/etc/httpd/conf.d/site1.conf:1)
         port 80 namevhost site2.bangkyu.com (/etc/httpd/conf.d/site2.conf:1)
```

Apache가 동일한 TCP 80 Port에서 `site1`과 `site2`를 각각 별도의 Name-based VirtualHost로 인식하는 것을 확인하였다.

---

# Web 접속 검증

## 16. SITE 1 접속

```bash
curl http://site1.bangkyu.com
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

---

## 17. SITE 2 접속

```bash
curl http://site2.bangkyu.com
```

실제 결과:

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Site 2</title>
</head>
<body>
    <h1>site2.bangkyu.com</h1>
    <p>Apache VirtualHost - SITE 2</p>
</body>
</html>
```

동일한 IP와 Port를 사용하지만 Domain Name에 따라 서로 다른 Web Page가 출력되었다.

---

## 18. 기존 www Site 확인

```bash
curl http://www.bangkyu.com
```

결과:

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>BangKyu Web Server</title>
</head>
<body>
    <h1>www.bangkyu.com</h1>
    <p>Rocky Linux Apache Web Server</p>
    <p>DNS + Web Server 통합 실습</p>
</body>
</html>
```

기존 Web Site도 정상적으로 유지되는 것을 확인하였다.

---

# Access Log 검증

## 19. SITE 1 Access Log

```bash
tail -n 5 /var/log/httpd/site1-access.log
```

실제 결과:

```text
192.168.111.100 - - [14/Sep/2026:13:04:40 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
192.168.111.100 - - [14/Sep/2026:13:04:55 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
192.168.111.150 - - [14/Sep/2026:13:08:36 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
```

Client-L의 IP:

```text
192.168.111.150
```

에서 SITE 1에 접속하였으며 HTTP Status Code `200`을 확인하였다.

---

## 20. SITE 2 Access Log

```bash
tail -n 5 /var/log/httpd/site2-access.log
```

실제 결과:

```text
192.168.111.100 - - [14/Sep/2026:13:04:40 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
192.168.111.100 - - [14/Sep/2026:13:05:02 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
192.168.111.150 - - [14/Sep/2026:13:08:43 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
```

SITE 2에서도 Client-L의 요청이 정상적으로 처리되었음을 확인하였다.

```text
192.168.111.150
→ Client-L

GET /
→ Web Root 요청

HTTP/1.1

200
→ 요청 정상 처리
```

---

# Name-based VirtualHost 동작 구조

## 21. SITE 1

```text
Client-L
192.168.111.150
        |
        | site1.bangkyu.com
        v
DNS Server
192.168.111.100
        |
        | A Record
        v
192.168.111.100
        |
        | TCP 80
        v
Apache
        |
        | Host: site1.bangkyu.com
        v
site1 VirtualHost
        |
        v
/var/www/site1/index.html
```

---

## 22. SITE 2

```text
Client-L
192.168.111.150
        |
        | site2.bangkyu.com
        v
DNS Server
192.168.111.100
        |
        | A Record
        v
192.168.111.100
        |
        | TCP 80
        v
Apache
        |
        | Host: site2.bangkyu.com
        v
site2 VirtualHost
        |
        v
/var/www/site2/index.html
```

---

# 핵심 원리

## 23. 동일 IP + 동일 Port에서 Site 구분

두 Domain의 DNS 결과는 동일하다.

```text
site1.bangkyu.com
→ 192.168.111.100

site2.bangkyu.com
→ 192.168.111.100
```

Port 역시 동일하다.

```text
TCP 80
```

하지만 HTTP 요청에는 접속하려는 Host Name 정보가 포함된다.

예:

```text
Host: site1.bangkyu.com
```

Apache는 해당 Host Name을 VirtualHost의 `ServerName`과 비교하여 사용할 Site를 결정한다.

```text
Host: site1.bangkyu.com
        ↓
ServerName site1.bangkyu.com
        ↓
/var/www/site1
```

```text
Host: site2.bangkyu.com
        ↓
ServerName site2.bangkyu.com
        ↓
/var/www/site2
```

따라서 하나의 IP와 하나의 TCP Port로 여러 Web Site를 운영할 수 있다.

---

# 주요 명령어

## DNS

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db

dig @192.168.111.100 site1.bangkyu.com A +short
dig @192.168.111.100 site2.bangkyu.com A +short
```

## Apache

```bash
httpd -t
httpd -S

systemctl restart httpd
systemctl is-active httpd
```

## Web 접속

```bash
curl http://www.bangkyu.com
curl http://site1.bangkyu.com
curl http://site2.bangkyu.com
```

## Access Log

```bash
tail -n 5 /var/log/httpd/site1-access.log
tail -n 5 /var/log/httpd/site2-access.log
```

---

# 실습 결과

```text
DNS
✓ site1.bangkyu.com → 192.168.111.100
✓ site2.bangkyu.com → 192.168.111.100

Apache
✓ Name-based VirtualHost 구성
✓ 동일 IP 사용
✓ 동일 TCP 80 Port 사용
✓ Domain별 DocumentRoot 분리

Web Site
✓ www.bangkyu.com 기존 Site 유지
✓ site1.bangkyu.com → SITE 1
✓ site2.bangkyu.com → SITE 2

Log
✓ SITE 1 Access Log 분리
✓ SITE 2 Access Log 분리
✓ Client-L 192.168.111.150 접속 확인
✓ HTTP Status Code 200 확인
```

최종 구성:

```text
                         Client-L
                      192.168.111.150
                             |
                             | DNS
                             v
                       bangkyu.com
                             |
                             v
                     192.168.111.100
                             |
                        Apache :80
                             |
                  HTTP Host Name 확인
                             |
          +------------------+------------------+
          |                  |                  |
          v                  v                  v
 www.bangkyu.com    site1.bangkyu.com   site2.bangkyu.com
          |                  |                  |
          v                  v                  v
 /var/www/html       /var/www/site1      /var/www/site2
          |                  |                  |
          v                  v                  v
 기존 Web Site          SITE 1             SITE 2
```

하나의 Apache Web Server에서 동일한 IP와 TCP 80 Port를 사용하면서 HTTP Host Name을 기준으로 각각 다른 DocumentRoot를 제공하는 Name-based VirtualHost를 구축하였다.

또한 DNS A Record와 Apache VirtualHost를 연동하고 Client-L의 실제 접속 기록을 Site별 Access Log에서 확인하여 DNS → HTTP → VirtualHost → DocumentRoot로 이어지는 전체 동작을 검증하였다.
