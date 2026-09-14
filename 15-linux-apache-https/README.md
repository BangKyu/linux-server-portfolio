# Apache HTTPS + Self-Signed Certificate 구축

기존 Apache VirtualHost 환경의 `site1.bangkyu.com`에 HTTPS를 적용하고, OpenSSL을 이용하여 Self-Signed Certificate를 직접 생성하였다.

Client에서 HTTPS 접속과 TLS 정보를 확인하고 Apache SSL Access Log를 통해 실제 Client 요청까지 검증하였다.

```text
Client-L
192.168.111.150
        |
        | https://site1.bangkyu.com
        v
DNS Server
192.168.111.100
        |
        | site1.bangkyu.com
        v
192.168.111.100
        |
      TCP 443
        |
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
| SSL Module | mod_ssl |
| TLS Library | OpenSSL |
| HTTP Port | TCP 80 |
| HTTPS Port | TCP 443 |
| Certificate | Self-Signed Certificate |

기존 DNS:

```text
site1.bangkyu.com
→ 192.168.111.100
```

기존 HTTP VirtualHost:

```text
site1.bangkyu.com
→ /var/www/site1
```

---

# 초기 상태 확인

## 2. SSL 관련 Package 확인

```bash
rpm -q mod_ssl
rpm -q openssl

openssl version
```

실제 결과:

```text
mod_ssl 패키지가 설치되어 있지 않습니다

openssl-3.5.5-6.el9_8.x86_64

OpenSSL 3.5.5 27 Jan 2026
(Library: OpenSSL 3.5.5 27 Jan 2026)
```

OpenSSL은 설치되어 있었지만 Apache에서 HTTPS 기능을 사용하기 위한 `mod_ssl`은 설치되어 있지 않았다.

---

## 3. 기존 Apache 상태 확인

```bash
systemctl is-active httpd

ss -lntp | grep -E ':80 |:443 '
```

결과:

```text
active
```

```text
LISTEN 0 511 *:80 *:* users:(("httpd",...))
```

기존 Apache HTTP Service는 정상적으로 실행 중이며 TCP 80에서 Listen 중이었다.

HTTPS TCP 443은 아직 활성화되지 않은 상태였다.

---

## 4. 기존 Firewall 확인

```bash
firewall-cmd --list-services
```

결과:

```text
cockpit dhcp dhcpv6-client dns http ssh
```

기존에는 HTTP만 허용되어 있고 HTTPS Service는 추가되어 있지 않았다.

---

## 5. 기존 VirtualHost 확인

```bash
cat /etc/httpd/conf.d/site1.conf
```

설정:

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

기존 HTTP 구조:

```text
site1.bangkyu.com
        ↓
TCP 80
        ↓
Apache
        ↓
/var/www/site1
```

---

# mod_ssl 설치

## 6. mod_ssl 설치

```bash
dnf install -y mod_ssl
```

설치 후:

```bash
rpm -q mod_ssl
```

Apache SSL 설정 파일:

```text
/etc/httpd/conf.d/ssl.conf
```

이 생성되었다.

---

## 7. 기본 SSL 설정 확인

```bash
grep -nE '^(Listen|SSLCertificateFile|SSLCertificateKeyFile)|<VirtualHost|ServerName' /etc/httpd/conf.d/ssl.conf
```

결과:

```text
5:Listen 443 https
40:<VirtualHost _default_:443>
44:#ServerName www.example.com:443
85:SSLCertificateFile /etc/pki/tls/certs/localhost.crt
93:SSLCertificateKeyFile /etc/pki/tls/private/localhost.key
```

`mod_ssl` 설치 후 Apache가 TCP 443을 사용하도록 설정되어 있었으며 기본 인증서 경로는 다음과 같았다.

```text
/etc/pki/tls/certs/localhost.crt
/etc/pki/tls/private/localhost.key
```

---

# 초기 SSL 오류

## 8. Apache 설정 검사 실패

```bash
httpd -t
```

결과:

```text
AH00526: Syntax error on line 85 of /etc/httpd/conf.d/ssl.conf:
SSLCertificateFile: file '/etc/pki/tls/certs/localhost.crt' does not exist or is empty
```

인증서 Directory 확인:

```bash
ls -l /etc/pki/tls/certs/
ls -l /etc/pki/tls/private/
```

실제 상태:

```text
/etc/pki/tls/certs/
→ localhost.crt 없음

/etc/pki/tls/private/
→ localhost.key 없음
```

즉 `ssl.conf`는 존재하지만 설정에서 참조하는 기본 인증서와 Private Key가 존재하지 않아 Apache 설정 검사에 실패하였다.

---

# Self-Signed Certificate 생성

## 9. 인증서와 Private Key 생성

OpenSSL을 이용하여 `site1.bangkyu.com` 전용 Self-Signed Certificate를 생성하였다.

```bash
openssl req -x509 -nodes -newkey rsa:2048 \
-keyout /etc/pki/tls/private/site1.bangkyu.com.key \
-out /etc/pki/tls/certs/site1.bangkyu.com.crt \
-days 365 \
-subj "/C=KR/ST=Seoul/L=Seoul/O=BangKyuLab/OU=Linux/CN=site1.bangkyu.com" \
-addext "subjectAltName=DNS:site1.bangkyu.com"
```

생성 파일:

```text
Private Key
/etc/pki/tls/private/site1.bangkyu.com.key

Certificate
/etc/pki/tls/certs/site1.bangkyu.com.crt
```

---

## 10. Private Key 권한

Private Key는 외부에 공개되면 안 되는 중요한 파일이므로 권한을 제한하였다.

```bash
chmod 600 /etc/pki/tls/private/site1.bangkyu.com.key
```

---

# Certificate 확인

## 11. 인증서 정보 확인

```bash
openssl x509 \
-in /etc/pki/tls/certs/site1.bangkyu.com.crt \
-noout \
-subject \
-issuer \
-dates \
-ext subjectAltName
```

실제 결과:

```text
subject=C=KR, ST=Seoul, L=Seoul, O=BangKyuLab, OU=Linux, CN=site1.bangkyu.com

issuer=C=KR, ST=Seoul, L=Seoul, O=BangKyuLab, OU=Linux, CN=site1.bangkyu.com

notBefore=Sep 14 04:19:44 2026 GMT
notAfter=Sep 14 04:19:44 2027 GMT

X509v3 Subject Alternative Name:
    DNS:site1.bangkyu.com
```

`subject`와 `issuer`가 동일하므로 자체적으로 서명한 Self-Signed Certificate임을 확인하였다.

Domain 정보:

```text
CN
→ site1.bangkyu.com

SAN
→ DNS:site1.bangkyu.com
```

---

# Apache SSL 설정

## 12. ssl.conf 백업

```bash
cp -a /etc/httpd/conf.d/ssl.conf /etc/httpd/conf.d/ssl.conf.backup
```

---

## 13. HTTPS VirtualHost 설정

```bash
vi /etc/httpd/conf.d/ssl.conf
```

주요 설정을 다음과 같이 변경하였다.

```apache
DocumentRoot "/var/www/site1"

ServerName site1.bangkyu.com:443
```

Certificate:

```apache
SSLCertificateFile /etc/pki/tls/certs/site1.bangkyu.com.crt
```

Private Key:

```apache
SSLCertificateKeyFile /etc/pki/tls/private/site1.bangkyu.com.key
```

확인:

```bash
grep -nE 'ServerName|DocumentRoot|SSLCertificateFile|SSLCertificateKeyFile' /etc/httpd/conf.d/ssl.conf
```

실제 결과:

```text
43:DocumentRoot "/var/www/site1"
44:ServerName site1.bangkyu.com:443
85:SSLCertificateFile /etc/pki/tls/certs/site1.bangkyu.com.crt
93:SSLCertificateKeyFile /etc/pki/tls/private/site1.bangkyu.com.key
```

---

## 14. Apache 설정 문법 확인

```bash
httpd -t
```

결과:

```text
Syntax OK
```

기존에 발생했던 인증서 파일 오류가 해결되었다.

---

# HTTPS Service 활성화

## 15. Firewall HTTPS 허용

```bash
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

HTTPS는 기본적으로:

```text
TCP 443
```

을 사용한다.

---

## 16. Apache 적용

```bash
systemctl restart httpd
```

HTTPS 구조:

```text
site1.bangkyu.com
        ↓
192.168.111.100
        ↓
TCP 443
        ↓
TLS
        ↓
Apache
        ↓
/var/www/site1
```

---

# Server-A HTTPS 검증

## 17. 일반 curl 접속

Server-A에서:

```bash
curl https://site1.bangkyu.com
```

실제 결과:

```text
curl: (60) SSL certificate problem: self-signed certificate
```

HTTPS Server 자체의 장애가 아니라 Client가 Self-Signed Certificate를 신뢰하지 않기 때문에 발생한 오류이다.

---

## 18. 인증서 검증을 생략한 HTTPS 접속

```bash
curl -k https://site1.bangkyu.com
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

HTTPS 연결을 통해 SITE 1 Web Page가 정상적으로 제공되는 것을 확인하였다.

`-k` 옵션은 인증서 신뢰 검증을 생략하고 TLS 연결을 수행한다.

---

# TLS 정보 확인

## 19. OpenSSL을 이용한 TLS 연결

```bash
openssl s_client \
-connect site1.bangkyu.com:443 \
-servername site1.bangkyu.com \
</dev/null
```

연결:

```text
Connecting to 192.168.111.100
CONNECTED(00000003)
```

Certificate:

```text
CN=site1.bangkyu.com
```

Self-Signed Certificate 검증:

```text
verify error:num=18:self-signed certificate
```

---

## 20. Certificate Chain

실제 확인:

```text
Certificate chain

0 s:C=KR, ST=Seoul, L=Seoul,
    O=BangKyuLab, OU=Linux,
    CN=site1.bangkyu.com

  i:C=KR, ST=Seoul, L=Seoul,
    O=BangKyuLab, OU=Linux,
    CN=site1.bangkyu.com
```

Subject와 Issuer가 동일하였다.

```text
Subject
= 인증서의 대상

Issuer
= 인증서를 발급한 주체
```

Self-Signed Certificate이므로 자신이 자신의 인증서를 서명한다.

---

## 21. Public Key / Signature 확인

```text
PKEY: RSA, 2048 (bit)
sigalg: sha256WithRSAEncryption
```

인증서 생성 시 RSA 2048 bit Key를 사용한 것을 확인하였다.

---

## 22. TLS Version 확인

실제 연결 결과:

```text
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384

Protocol: TLSv1.3

Server public key is 2048 bit
```

확인된 TLS 정보:

```text
TLS Version
→ TLS 1.3

Cipher
→ TLS_AES_256_GCM_SHA384

Server Public Key
→ RSA 2048 bit
```

---

## 23. 인증서 검증 결과

```text
Verify return code: 18 (self-signed certificate)
```

TLS 연결은 정상적으로 이루어졌지만 Self-Signed Certificate이므로 기본 신뢰 저장소에서 인증서를 신뢰하지 않는 상태이다.

---

# Client-L HTTPS 검증

## 24. DNS 확인

Client-L:

```bash
nslookup site1.bangkyu.com
```

실제 결과:

```text
Server:         192.168.111.100
Address:        192.168.111.100#53

Name:   site1.bangkyu.com
Address: 192.168.111.100
```

Client-L이 Master DNS Server를 통해 `site1.bangkyu.com`을 정상적으로 `192.168.111.100`으로 조회하였다.

---

## 25. Client-L 일반 HTTPS 접속

```bash
curl https://site1.bangkyu.com
```

결과:

```text
curl: (60) SSL certificate problem: self-signed certificate
```

Server-A와 동일하게 Self-Signed Certificate를 신뢰하지 않아 인증서 검증에 실패하였다.

---

## 26. Client-L HTTPS 접속 성공

```bash
curl -k https://site1.bangkyu.com
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

Client-L에서 DNS를 이용하여 Server-A에 접근한 후 HTTPS를 통해 Web Page를 정상적으로 전달받았다.

---

# SSL Access Log

## 27. HTTPS Client 접속 확인

Server-A:

```bash
tail -n 10 /var/log/httpd/ssl_access_log
```

실제 결과:

```text
192.168.111.100 - - [14/Sep/2026:14:57:34 +0900] "GET / HTTP/1.1" 200 194
192.168.111.150 - - [14/Sep/2026:14:59:05 +0900] "GET / HTTP/1.1" 200 194
```

첫 번째 요청:

```text
192.168.111.100
→ Server-A 자체 HTTPS 테스트
```

두 번째 요청:

```text
192.168.111.150
→ Client-L HTTPS 접속
```

HTTP Status:

```text
200
→ 요청 정상 처리
```

따라서 Client-L에서 HTTPS를 이용하여 Apache Web Server에 실제로 접속한 것을 Server Log에서도 확인하였다.

---

# HTTP와 HTTPS 비교

## 28. HTTP

```text
Client
   ↓
TCP 80
   ↓
HTTP
   ↓
Apache
```

HTTP 자체에는 TLS 암호화가 적용되지 않는다.

---

## 29. HTTPS

```text
Client
   ↓
TCP 443
   ↓
TLS
   ↓
HTTP
   ↓
Apache
```

HTTPS는:

```text
HTTP
+
TLS
```

를 이용하여 Client와 Server 사이의 통신을 보호한다.

---

# Certificate와 Private Key

## 30. Certificate

이번 실습:

```text
/etc/pki/tls/certs/site1.bangkyu.com.crt
```

Certificate에는 Server의 공개 정보와 Public Key 등이 포함된다.

Client에게 전달되는 정보이므로 Private Key와 달리 비밀 정보가 아니다.

---

## 31. Private Key

이번 실습:

```text
/etc/pki/tls/private/site1.bangkyu.com.key
```

Private Key는 Server만 가지고 있어야 하는 비밀 정보이다.

```text
Certificate
→ Client에게 공개 가능

Private Key
→ Server만 보관
```

Private Key가 유출되면 보안에 문제가 발생할 수 있으므로 접근 권한을 제한해야 한다.

---

# Self-Signed Certificate

## 32. Self-Signed Certificate 특징

이번 실습의 인증서는 공인 CA가 발급한 인증서가 아니라 직접 생성하고 직접 서명하였다.

따라서:

```text
Subject
=
Issuer
```

형태가 확인되었다.

장점:

```text
별도의 CA 없이 생성 가능
실습 환경에서 사용하기 편리
TLS 암호화 구조 실습 가능
```

단점:

```text
Client가 기본적으로 신뢰하지 않음
Browser / curl에서 인증서 Warning 발생
```

---

## 33. curl -k의 의미

```bash
curl -k https://site1.bangkyu.com
```

`-k`:

```text
Certificate Trust 검증을 생략
```

한다.

따라서 Self-Signed Certificate 환경에서 TLS 접속 여부를 테스트할 때 사용할 수 있다.

하지만 실제 운영 환경에서 무조건 `-k`를 사용하는 것은 인증서 검증을 무력화하기 때문에 적절하지 않다.

---

# 전체 동작 흐름

## 34. HTTPS 접속 과정

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
        | TCP 443
        v
TLS Handshake
        |
        | Certificate 전달
        v
site1.bangkyu.com.crt
        |
        v
Apache HTTPS
        |
        v
/var/www/site1/index.html
        |
        v
HTTP 200
```

---

# 주요 문제 해결

## 35. mod_ssl 설치 후 Apache 설정 오류

증상:

```text
SSLCertificateFile:
file '/etc/pki/tls/certs/localhost.crt'
does not exist or is empty
```

원인:

```text
ssl.conf 존재
        ↓
localhost.crt를 참조
        ↓
실제 Certificate 없음
        ↓
httpd -t 실패
```

해결:

```text
site1.bangkyu.com 전용 인증서 생성
        ↓
ssl.conf 인증서 경로 변경
        ↓
httpd -t
        ↓
Syntax OK
```

---

## 36. curl HTTPS 접속 오류

증상:

```text
SSL certificate problem: self-signed certificate
```

원인:

```text
TLS Server 장애 X

Self-Signed Certificate를
Client가 신뢰하지 않음
```

검증용 접속:

```bash
curl -k https://site1.bangkyu.com
```

결과:

```text
SITE 1 정상 출력
```

---

# 주요 명령어

## 37. SSL Package

```bash
dnf install -y mod_ssl

rpm -q mod_ssl
rpm -q openssl

openssl version
```

---

## 38. Certificate 생성

```bash
openssl req -x509 -nodes -newkey rsa:2048 \
-keyout /etc/pki/tls/private/site1.bangkyu.com.key \
-out /etc/pki/tls/certs/site1.bangkyu.com.crt \
-days 365 \
-subj "/C=KR/ST=Seoul/L=Seoul/O=BangKyuLab/OU=Linux/CN=site1.bangkyu.com" \
-addext "subjectAltName=DNS:site1.bangkyu.com"
```

---

## 39. Certificate 확인

```bash
openssl x509 \
-in /etc/pki/tls/certs/site1.bangkyu.com.crt \
-noout \
-subject \
-issuer \
-dates \
-ext subjectAltName
```

---

## 40. Apache 설정 검사

```bash
httpd -t
```

정상:

```text
Syntax OK
```

---

## 41. HTTPS Firewall

```bash
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

---

## 42. HTTPS 접속

일반:

```bash
curl https://site1.bangkyu.com
```

Self-Signed 테스트:

```bash
curl -k https://site1.bangkyu.com
```

---

## 43. TLS 연결 확인

```bash
openssl s_client \
-connect site1.bangkyu.com:443 \
-servername site1.bangkyu.com \
</dev/null
```

---

## 44. SSL Access Log

```bash
tail -n 10 /var/log/httpd/ssl_access_log
```

---

# 실습 결과

```text
DNS
✓ site1.bangkyu.com → 192.168.111.100

Apache
✓ 기존 HTTP VirtualHost 유지
✓ mod_ssl 설치
✓ HTTPS 설정
✓ TCP 443 기반 Web Service 구성

Certificate
✓ RSA 2048 bit Key 생성
✓ Self-Signed Certificate 생성
✓ CN = site1.bangkyu.com
✓ SAN = DNS:site1.bangkyu.com
✓ 인증서 유효기간 1년

TLS
✓ TLS 1.3 연결 확인
✓ TLS_AES_256_GCM_SHA384 확인

HTTPS
✓ Server-A에서 HTTPS 접속
✓ Client-L에서 HTTPS 접속
✓ Self-Signed Certificate 검증 오류 확인
✓ curl -k를 이용한 HTTPS Service 검증

Log
✓ Server-A HTTPS 요청 확인
✓ Client-L 192.168.111.150 접속 확인
✓ HTTP Status Code 200 확인
```

---

# 최종 구성

```text
                         Client-L
                      192.168.111.150
                             |
                             | DNS Query
                             v
                       Master DNS
                    192.168.111.100
                             |
                             | A Record
                             v
                   site1.bangkyu.com
                    192.168.111.100
                             |
                             | TCP 443
                             v
                      TLS Handshake
                             |
                    Self-Signed Certificate
                             |
              CN  = site1.bangkyu.com
              SAN = site1.bangkyu.com
                             |
                             v
                         Apache
                             |
                             v
                    /var/www/site1
                             |
                             v
                         index.html
                             |
                             v
                        HTTP 200
```

OpenSSL을 이용하여 `site1.bangkyu.com`의 RSA 2048 bit Private Key와 Self-Signed Certificate를 생성하고 Apache `mod_ssl`과 연결하여 HTTPS Service를 구축하였다.

또한 Server-A와 Client-L에서 실제 HTTPS 접속을 수행하고, Self-Signed Certificate의 신뢰 오류와 `curl -k`를 이용한 접속 차이를 확인하였다.

마지막으로 OpenSSL을 이용하여 TLS 1.3 연결과 Cipher를 확인하고 Apache SSL Access Log에서 Client-L의 실제 HTTPS 요청과 HTTP Status Code 200을 검증하였다.
