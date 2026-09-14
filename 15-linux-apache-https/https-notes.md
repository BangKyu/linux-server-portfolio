# Apache HTTPS / SSL / TLS 이론 정리

## 1. HTTPS란?

HTTPS는 HTTP 통신에 TLS 암호화를 적용한 Protocol이다.

```text
HTTP
+
TLS
=
HTTPS
```

일반 HTTP:

```text
Client
   ↓
TCP 80
   ↓
HTTP
   ↓
Web Server
```

HTTPS:

```text
Client
   ↓
TCP 443
   ↓
TLS
   ↓
HTTP
   ↓
Web Server
```

HTTPS를 사용하면 Client와 Server 사이의 통신을 암호화할 수 있다.

---

# HTTP와 HTTPS

## 2. HTTP

HTTP:

```text
HyperText Transfer Protocol
```

Web Browser와 Web Server가 데이터를 주고받기 위해 사용하는 Protocol이다.

기본 Port:

```text
TCP 80
```

예:

```text
http://site1.bangkyu.com
```

구조:

```text
Client
   ↓
TCP 80
   ↓
Apache
   ↓
HTTP Data
```

HTTP 자체는 통신 내용을 암호화하지 않는다.

---

## 3. HTTPS

HTTPS:

```text
HyperText Transfer Protocol Secure
```

HTTP에 TLS를 적용한 방식이다.

기본 Port:

```text
TCP 443
```

예:

```text
https://site1.bangkyu.com
```

구조:

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

---

## 4. HTTP와 HTTPS 비교

```text
HTTP

Port
→ TCP 80

암호화
→ 없음

URL
→ http://

Certificate
→ 필요 없음
```

```text
HTTPS

Port
→ TCP 443

암호화
→ TLS 사용

URL
→ https://

Certificate
→ 사용
```

---

# SSL과 TLS

## 5. SSL이란?

SSL:

```text
Secure Sockets Layer
```

Network 통신을 암호화하기 위해 만들어진 보안 Protocol이다.

과거에는 SSL이라는 이름이 사용되었지만 현재는 SSL의 후속 기술인 TLS가 사용된다.

Apache Package나 설정 이름에는 아직 다음과 같이 `SSL`이라는 표현이 많이 남아 있다.

```text
mod_ssl

ssl.conf

SSLCertificateFile

SSLCertificateKeyFile
```

실제 현대 HTTPS 통신은 TLS를 사용한다.

---

## 6. TLS란?

TLS:

```text
Transport Layer Security
```

Client와 Server 사이의 통신을 암호화하고, 통신 상대를 인증하며, 데이터 변조 여부를 확인하기 위한 Protocol이다.

이번 실습에서 확인된 Protocol:

```text
TLSv1.3
```

실제 출력:

```text
Protocol: TLSv1.3
```

---

## 7. HTTPS에서 TLS가 필요한 이유

Client와 Server가 평문으로 통신하면 Network 중간에서 Data가 노출될 수 있다.

TLS를 이용하면:

```text
Client
   ↓
암호화
   ↓
Network
   ↓
복호화
   ↓
Server
```

형태로 통신할 수 있다.

HTTPS에서 중요한 목적은 다음과 같다.

```text
기밀성
→ 통신 내용 암호화

무결성
→ Data가 중간에 변경되지 않았는지 확인

인증
→ 접속한 Server가 올바른 Server인지 Certificate로 확인
```

---

# Apache와 HTTPS

## 8. Apache의 HTTPS 지원

Rocky Linux의 Apache Package:

```text
httpd
```

기본 HTTP 기능만으로는 HTTPS 설정이 완성되지 않는다.

HTTPS 기능을 사용하기 위해:

```text
mod_ssl
```

Package를 사용한다.

설치:

```bash
dnf install -y mod_ssl
```

---

## 9. mod_ssl

`mod_ssl`은 Apache에서 SSL/TLS 기능을 사용할 수 있게 해주는 Module이다.

설치 후 대표적인 설정 파일:

```text
/etc/httpd/conf.d/ssl.conf
```

이번 실습에서 `mod_ssl` 설치 전:

```text
TCP 80
→ LISTEN

TCP 443
→ 사용하지 않음
```

HTTPS 구성 후:

```text
TCP 80
→ HTTP

TCP 443
→ HTTPS
```

구조가 된다.

---

# OpenSSL

## 10. OpenSSL이란?

OpenSSL은 SSL/TLS와 암호화 관련 기능을 제공하는 Tool 및 Library이다.

이번 실습에서 확인:

```text
OpenSSL 3.5.5
```

OpenSSL로 수행할 수 있는 작업:

```text
Private Key 생성

Certificate 생성

Certificate 정보 확인

TLS Server 연결 테스트

Certificate 검증
```

---

## 11. 이번 실습에서 사용한 OpenSSL 기능

Certificate 생성:

```bash
openssl req
```

Certificate 정보 확인:

```bash
openssl x509
```

TLS 연결 확인:

```bash
openssl s_client
```

---

# HTTPS Certificate

## 12. Certificate란?

HTTPS Certificate는 Server의 신원 정보와 Public Key 등을 포함하는 전자 문서이다.

이번 실습의 Certificate:

```text
/etc/pki/tls/certs/site1.bangkyu.com.crt
```

Certificate를 통해 Client는:

```text
이 Certificate가 어느 Domain용인지

누가 발급했는지

언제까지 유효한지

어떤 Public Key를 사용하는지
```

등을 확인할 수 있다.

---

## 13. Private Key란?

이번 실습의 Private Key:

```text
/etc/pki/tls/private/site1.bangkyu.com.key
```

Private Key는 Server가 비밀리에 보관해야 하는 Key이다.

```text
Certificate
→ Client에게 공개 가능

Private Key
→ Server 외부에 공개하면 안 됨
```

따라서 Private Key의 File Permission을 제한하였다.

```bash
chmod 600 /etc/pki/tls/private/site1.bangkyu.com.key
```

`600`:

```text
Owner
→ Read / Write

Group
→ 권한 없음

Other
→ 권한 없음
```

---

# Public Key와 Private Key

## 14. 비대칭키 구조

TLS에서는 Public Key와 Private Key를 사용하는 비대칭 암호 기술이 사용된다.

개념적으로:

```text
Private Key
→ Server가 비밀 보관

Public Key
→ Certificate를 통해 공개
```

Private Key와 Public Key는 서로 관련된 Key Pair이다.

이번 실습에서는:

```text
RSA 2048 bit
```

Key를 생성하였다.

---

## 15. RSA 2048 bit

Certificate 생성 명령:

```bash
openssl req -x509 -nodes -newkey rsa:2048 ...
```

여기서:

```text
rsa:2048
```

은 RSA 방식의 2048 bit Key Pair를 생성한다는 의미이다.

실제 TLS 확인:

```text
Server public key is 2048 bit
```

---

# Certificate의 주요 정보

## 16. Subject

Certificate가 누구를 위한 것인지 나타낸다.

실제:

```text
subject=
C=KR,
ST=Seoul,
L=Seoul,
O=BangKyuLab,
OU=Linux,
CN=site1.bangkyu.com
```

주요 정보:

```text
C
→ Country

ST
→ State

L
→ Locality

O
→ Organization

OU
→ Organizational Unit

CN
→ Common Name
```

이번 실습의 CN:

```text
site1.bangkyu.com
```

---

## 17. Issuer

Certificate를 발급하고 서명한 주체를 의미한다.

실제:

```text
issuer=
C=KR,
ST=Seoul,
L=Seoul,
O=BangKyuLab,
OU=Linux,
CN=site1.bangkyu.com
```

이번 실습에서는:

```text
Subject
=
Issuer
```

였다.

이유:

```text
Self-Signed Certificate
```

를 사용했기 때문이다.

---

# Self-Signed Certificate

## 18. Self-Signed Certificate란?

Certificate를 공인 CA가 서명하지 않고 자신이 직접 서명한 Certificate이다.

구조:

```text
Server가 Certificate 생성
        ↓
Server 자신이 서명
        ↓
Self-Signed Certificate
```

이번 실습:

```text
Subject
CN=site1.bangkyu.com

Issuer
CN=site1.bangkyu.com
```

두 값이 동일하였다.

---

## 19. Self-Signed Certificate의 특징

실습이나 내부 Test 환경에서는 쉽게 사용할 수 있다.

하지만 일반 Client의 Trust Store에는 해당 Certificate를 신뢰할 근거가 등록되어 있지 않다.

따라서:

```bash
curl https://site1.bangkyu.com
```

실행 시:

```text
SSL certificate problem: self-signed certificate
```

오류가 발생하였다.

---

## 20. Self-Signed 오류가 HTTPS 장애인가?

아니다.

이번 실습에서:

```bash
curl https://site1.bangkyu.com
```

은 실패했지만:

```bash
curl -k https://site1.bangkyu.com
```

은 정상적으로 Web Page를 가져왔다.

즉:

```text
TCP 443
→ 정상

TLS 연결
→ 정상

Apache HTTPS
→ 정상

Web Page
→ 정상

Certificate Trust
→ 실패
```

상태이다.

따라서 Self-Signed Certificate 오류와 HTTPS Service 장애를 구분해야 한다.

---

# CA

## 21. CA란?

CA:

```text
Certificate Authority
```

Certificate를 발급하거나 서명하여 신뢰 관계를 제공하는 기관 또는 시스템이다.

일반적인 HTTPS 환경:

```text
Web Server
   ↓
Certificate 발급 요청
   ↓
CA
   ↓
CA가 Certificate 서명
   ↓
Server에 설치
```

Client가 해당 CA를 신뢰하면 Server Certificate도 신뢰할 수 있다.

---

## 22. 공인 인증서와 Self-Signed 비교

```text
공인 CA Certificate

Issuer
→ 공인 CA

Client 기본 신뢰
→ 일반적으로 가능

Browser Warning
→ 정상 Certificate라면 없음

운영 환경
→ 일반적인 방식
```

```text
Self-Signed Certificate

Issuer
→ 자기 자신

Client 기본 신뢰
→ 안 함

Browser / curl Warning
→ 발생

실습 / 내부 Test
→ 사용 가능
```

---

# CN과 SAN

## 23. CN

CN:

```text
Common Name
```

Certificate의 대표 이름을 나타내는 항목이다.

이번 실습:

```text
CN=site1.bangkyu.com
```

---

## 24. SAN

SAN:

```text
Subject Alternative Name
```

Certificate에서 사용할 수 있는 Domain Name 등의 정보를 지정하는 Extension이다.

이번 실습:

```text
X509v3 Subject Alternative Name:
    DNS:site1.bangkyu.com
```

Certificate 생성 시:

```bash
-addext "subjectAltName=DNS:site1.bangkyu.com"
```

을 사용하였다.

---

## 25. Domain과 Certificate

Client가:

```text
https://site1.bangkyu.com
```

으로 접속하면 Certificate에 해당 Domain이 올바르게 포함되어 있는지 확인할 수 있다.

이번 실습:

```text
접속 Domain
site1.bangkyu.com

Certificate SAN
DNS:site1.bangkyu.com
```

으로 일치하였다.

---

# Certificate 유효기간

## 26. notBefore / notAfter

Certificate에는 사용 가능한 기간이 존재한다.

이번 실습:

```text
notBefore=Sep 14 04:19:44 2026 GMT

notAfter=Sep 14 04:19:44 2027 GMT
```

즉 Certificate가 유효한 시작 시점과 만료 시점이다.

Certificate 생성 시:

```text
-days 365
```

를 사용하여 약 1년의 유효기간을 설정하였다.

---

# Apache SSL 설정

## 27. ssl.conf

Apache HTTPS의 주요 설정 파일:

```text
/etc/httpd/conf.d/ssl.conf
```

이번 실습의 핵심 설정:

```apache
DocumentRoot "/var/www/site1"

ServerName site1.bangkyu.com:443

SSLCertificateFile /etc/pki/tls/certs/site1.bangkyu.com.crt

SSLCertificateKeyFile /etc/pki/tls/private/site1.bangkyu.com.key
```

---

## 28. ServerName

```apache
ServerName site1.bangkyu.com:443
```

HTTPS VirtualHost가 사용할 Server Name을 지정한다.

```text
Domain
→ site1.bangkyu.com

Port
→ 443
```

---

## 29. DocumentRoot

```apache
DocumentRoot "/var/www/site1"
```

HTTPS로 접속했을 때 제공할 Web File의 기본 Directory이다.

구조:

```text
https://site1.bangkyu.com/
        ↓
/var/www/site1/
        ↓
index.html
```

---

## 30. SSLCertificateFile

```apache
SSLCertificateFile /etc/pki/tls/certs/site1.bangkyu.com.crt
```

Apache가 Client에게 제공할 Server Certificate를 지정한다.

---

## 31. SSLCertificateKeyFile

```apache
SSLCertificateKeyFile /etc/pki/tls/private/site1.bangkyu.com.key
```

Certificate에 대응하는 Server Private Key를 지정한다.

Certificate와 Private Key가 올바른 Pair여야 한다.

---

# mod_ssl 설치 후 발생한 오류

## 32. 초기 오류

`mod_ssl` 설치 후:

```bash
httpd -t
```

실행 결과:

```text
SSLCertificateFile:
file '/etc/pki/tls/certs/localhost.crt'
does not exist or is empty
```

---

## 33. 오류 원인

기본 `ssl.conf`에는:

```apache
SSLCertificateFile /etc/pki/tls/certs/localhost.crt
```

```apache
SSLCertificateKeyFile /etc/pki/tls/private/localhost.key
```

가 설정되어 있었다.

하지만 실제 File은 존재하지 않았다.

따라서:

```text
ssl.conf
        ↓
localhost.crt 요청
        ↓
File 없음
        ↓
Apache Configuration 검사 실패
```

가 발생하였다.

---

## 34. 오류 해결

직접 다음 File을 생성하였다.

```text
/etc/pki/tls/certs/site1.bangkyu.com.crt

/etc/pki/tls/private/site1.bangkyu.com.key
```

그리고 `ssl.conf`를 해당 경로로 수정하였다.

이후:

```bash
httpd -t
```

결과:

```text
Syntax OK
```

---

# Apache 설정 검사

## 35. httpd -t

```bash
httpd -t
```

Apache Configuration의 문법과 일부 참조 File 문제를 확인한다.

정상:

```text
Syntax OK
```

설정을 변경한 후 바로 Apache를 Restart하는 것보다:

```text
설정 변경
   ↓
httpd -t
   ↓
Syntax OK
   ↓
systemctl restart httpd
```

순서로 진행하는 것이 안전하다.

---

# Firewall

## 36. HTTPS Firewall

HTTPS는 기본적으로 TCP 443을 사용한다.

Firewalld:

```bash
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-services
```

HTTPS가 Firewall에서 허용되어야 외부 Client가 접근할 수 있다.

---

# TCP 443

## 37. HTTPS Listen 확인

Apache HTTPS가 활성화되면 TCP 443에서 Listen한다.

확인:

```bash
ss -lntp | grep ':443 '
```

전체 흐름:

```text
Client
   ↓
TCP 443
   ↓
Server-A
   ↓
httpd
```

---

# TLS Handshake

## 38. TLS Handshake란?

HTTPS 통신을 시작하기 전에 Client와 Server가 TLS 통신에 필요한 정보를 협상하는 과정이다.

단순화하면:

```text
Client
   ↓
TLS 연결 요청
   ↓
Server
   ↓
Certificate 제공
   ↓
Client
   ↓
Certificate 확인
   ↓
암호화 방식 협상
   ↓
암호화된 통신 시작
```

이 과정을 TLS Handshake라고 한다.

---

## 39. Certificate 전달

Server는 TLS Handshake 과정에서 Client에게 Certificate를 제공한다.

이번 실습:

```text
site1.bangkyu.com.crt
```

Client는 Certificate를 확인하여:

```text
Domain이 맞는가?

유효기간이 맞는가?

신뢰할 수 있는 Issuer인가?
```

등을 검사한다.

---

# Cipher

## 40. Cipher란?

TLS 통신에서 실제 암호화에 사용할 알고리즘 조합을 의미한다.

이번 실습:

```text
Cipher is TLS_AES_256_GCM_SHA384
```

확인된 Cipher:

```text
TLS_AES_256_GCM_SHA384
```

TLS 1.3 연결에서 협상된 암호 Suite이다.

---

## 41. TLS Version 확인

OpenSSL:

```bash
openssl s_client \
-connect site1.bangkyu.com:443 \
-servername site1.bangkyu.com
```

실제 결과:

```text
Protocol: TLSv1.3
```

따라서 Client와 Server가 TLS 1.3으로 연결되었음을 확인하였다.

---

# openssl s_client

## 42. s_client의 역할

`openssl s_client`는 TLS Server에 직접 연결하여 TLS 정보를 확인할 수 있는 Tool이다.

사용:

```bash
openssl s_client \
-connect site1.bangkyu.com:443 \
-servername site1.bangkyu.com
```

확인 가능:

```text
Certificate

Subject

Issuer

TLS Version

Cipher

Public Key

Certificate 검증 결과
```

---

## 43. -servername

```text
-servername site1.bangkyu.com
```

TLS 연결 시 사용할 Server Name을 전달한다.

HTTPS에서 여러 Domain을 하나의 Server가 처리하는 환경에서는 Server Name 정보가 중요할 수 있다.

---

# SNI

## 44. SNI란?

SNI:

```text
Server Name Indication
```

TLS Handshake 과정에서 Client가 자신이 접속하려는 Domain Name을 Server에게 전달하는 기능이다.

HTTP에서는:

```text
Host Header
```

를 이용하여 VirtualHost를 구분할 수 있다.

하지만 HTTPS에서는 HTTP 요청 전에 TLS Handshake가 먼저 발생한다.

따라서 Server가 어떤 Certificate를 사용할지 결정하기 위해 TLS 단계에서 Domain Name을 알아야 할 수 있다.

이때 SNI가 사용된다.

---

## 45. HTTP Host와 SNI 차이

HTTP:

```text
Host: site1.bangkyu.com
```

Apache가 HTTP 요청에서 사용할 VirtualHost를 구분하는 데 사용한다.

SNI:

```text
site1.bangkyu.com
```

TLS Handshake 과정에서 어떤 HTTPS Site에 접속하려는지 Server에게 알려주는 데 사용된다.

구조:

```text
TCP 연결
   ↓
TLS Handshake
   ↓
SNI
   ↓
Certificate / TLS 설정 선택
   ↓
HTTPS 연결
   ↓
HTTP 요청
   ↓
Host Header
```

---

# curl HTTPS Test

## 46. 일반 curl

```bash
curl https://site1.bangkyu.com
```

curl은 HTTPS 접속 시 Certificate를 검증한다.

이번 실습:

```text
curl: (60) SSL certificate problem: self-signed certificate
```

가 발생하였다.

---

## 47. curl -k

```bash
curl -k https://site1.bangkyu.com
```

`-k`:

```text
--insecure
```

Certificate 신뢰 검증을 생략한다.

이번 실습에서는:

```text
Self-Signed Certificate가 적용된
HTTPS Service 자체가 정상 동작하는지
확인하기 위해 사용
```

하였다.

---

## 48. curl -k 사용 시 주의

`-k`는 Certificate 검증을 생략한다.

따라서:

```text
Server Identity 검증을 하지 않음
```

이라는 의미도 있다.

실제 운영 환경에서 무조건 `-k`를 사용하는 방식은 보안상 적절하지 않다.

실습에서:

```text
Self-Signed Certificate 환경의
HTTPS 연결 Test
```

목적으로 사용하였다.

---

# DNS와 HTTPS

## 49. DNS의 역할

Client-L:

```bash
nslookup site1.bangkyu.com
```

실제:

```text
site1.bangkyu.com
→ 192.168.111.100
```

DNS는 HTTPS 통신 전에 목적지 Server의 IP 주소를 찾는 역할을 한다.

---

## 50. DNS와 TLS는 별개

DNS:

```text
Domain
→ IP
```

TLS:

```text
Client
↔
Server 사이의 보안 통신
```

따라서:

```text
DNS가 정상
```

이라고 해서:

```text
HTTPS까지 정상
```

이라는 뜻은 아니다.

각 단계별 확인이 필요하다.

---

# 전체 HTTPS 요청 흐름

## 51. Client-L HTTPS 접속 과정

Client가:

```text
https://site1.bangkyu.com
```

에 접속한다고 가정한다.

```text
1. Client가 Domain Name 확인

   site1.bangkyu.com

2. DNS Query

   site1.bangkyu.com
   → 192.168.111.100

3. Client가 Server-A의 TCP 443으로 연결

4. TLS Handshake 시작

5. Server가 Certificate 제공

6. Client가 Certificate 확인

7. TLS Version / Cipher 협상

8. 암호화된 TLS 연결 생성

9. HTTP 요청 전달

10. Apache가 /var/www/site1/index.html 처리

11. HTTP Response 반환
```

---

# SSL Access Log

## 52. HTTPS Access Log

이번 실습:

```text
/var/log/httpd/ssl_access_log
```

확인:

```bash
tail -n 10 /var/log/httpd/ssl_access_log
```

실제:

```text
192.168.111.100 - - [14/Sep/2026:14:57:34 +0900] "GET / HTTP/1.1" 200 194

192.168.111.150 - - [14/Sep/2026:14:59:05 +0900] "GET / HTTP/1.1" 200 194
```

---

## 53. Log 해석

```text
192.168.111.150
```

Client-L의 IP이다.

```text
GET /
```

Web Root Page 요청이다.

```text
HTTP/1.1
```

HTTP Protocol Version이다.

```text
200
```

정상 처리된 HTTP Status Code이다.

```text
194
```

전송된 Response Body의 크기와 관련된 값이다.

---

# 문제 해결 순서

## 54. HTTPS가 접속되지 않을 때

확인 흐름:

```text
DNS
   ↓
TCP 443
   ↓
Firewall
   ↓
Apache
   ↓
mod_ssl
   ↓
Certificate
   ↓
Private Key
   ↓
TLS
   ↓
HTTP
```

한 단계씩 확인하는 것이 중요하다.

---

## 55. DNS 확인

```bash
nslookup site1.bangkyu.com
```

또는:

```bash
getent hosts site1.bangkyu.com
```

정상:

```text
192.168.111.100
```

---

## 56. Apache 상태

```bash
systemctl status httpd
```

또는:

```bash
systemctl is-active httpd
```

정상:

```text
active
```

---

## 57. Port 확인

```bash
ss -lntp | grep ':443 '
```

TCP 443에서 `httpd`가 Listen하는지 확인한다.

---

## 58. Firewall 확인

```bash
firewall-cmd --list-services
```

다음이 포함되어 있는지 확인:

```text
https
```

---

## 59. Apache 설정 확인

```bash
httpd -t
```

정상:

```text
Syntax OK
```

---

## 60. Certificate 확인

```bash
openssl x509 \
-in /etc/pki/tls/certs/site1.bangkyu.com.crt \
-noout \
-subject \
-issuer \
-dates \
-ext subjectAltName
```

확인:

```text
Domain

Issuer

유효기간

SAN
```

---

## 61. TLS 직접 확인

```bash
openssl s_client \
-connect site1.bangkyu.com:443 \
-servername site1.bangkyu.com \
</dev/null
```

확인:

```text
CONNECTED

Certificate

TLS Version

Cipher

Verify return code
```

---

# 주요 파일

## 62. Apache SSL 설정

```text
/etc/httpd/conf.d/ssl.conf
```

---

## 63. Certificate

```text
/etc/pki/tls/certs/site1.bangkyu.com.crt
```

---

## 64. Private Key

```text
/etc/pki/tls/private/site1.bangkyu.com.key
```

---

## 65. Web DocumentRoot

```text
/var/www/site1
```

---

## 66. HTTPS Access Log

```text
/var/log/httpd/ssl_access_log
```

---

# 주요 명령어

## 67. Package

```bash
dnf install -y mod_ssl

rpm -q mod_ssl
rpm -q openssl
```

---

## 68. OpenSSL Version

```bash
openssl version
```

---

## 69. Self-Signed Certificate 생성

```bash
openssl req -x509 -nodes -newkey rsa:2048 \
-keyout /etc/pki/tls/private/site1.bangkyu.com.key \
-out /etc/pki/tls/certs/site1.bangkyu.com.crt \
-days 365 \
-subj "/C=KR/ST=Seoul/L=Seoul/O=BangKyuLab/OU=Linux/CN=site1.bangkyu.com" \
-addext "subjectAltName=DNS:site1.bangkyu.com"
```

---

## 70. Private Key 권한

```bash
chmod 600 /etc/pki/tls/private/site1.bangkyu.com.key
```

---

## 71. Certificate 정보

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

## 72. Apache 설정 검사

```bash
httpd -t
```

---

## 73. Firewall

```bash
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
firewall-cmd --list-services
```

---

## 74. Apache

```bash
systemctl restart httpd
systemctl is-active httpd
```

---

## 75. TCP 443

```bash
ss -lntp | grep ':443 '
```

---

## 76. HTTPS

```bash
curl https://site1.bangkyu.com
```

Self-Signed Test:

```bash
curl -k https://site1.bangkyu.com
```

---

## 77. TLS

```bash
openssl s_client \
-connect site1.bangkyu.com:443 \
-servername site1.bangkyu.com \
</dev/null
```

---

## 78. Log

```bash
tail -n 10 /var/log/httpd/ssl_access_log
```

---

# 핵심 정리

## 79. HTTPS 핵심 구조

```text
HTTPS
=
HTTP + TLS
```

```text
Client
   ↓
DNS
   ↓
Server IP
   ↓
TCP 443
   ↓
TLS Handshake
   ↓
Certificate
   ↓
Encrypted Connection
   ↓
HTTP
   ↓
Apache
```

---

## 80. Certificate와 Private Key

```text
Certificate

→ Public Key 포함
→ Client에게 제공
→ Server의 신원 정보 포함
```

```text
Private Key

→ Server만 보관
→ 외부 공개 금지
→ Certificate와 Pair로 사용
```

---

## 81. Self-Signed 핵심

```text
Self-Signed

Subject
=
Issuer
```

자체 서명 인증서라 기본 Client Trust Store에서 신뢰하지 않는다.

따라서:

```text
curl https://...
→ Certificate 검증 실패

curl -k https://...
→ 검증 생략 후 TLS 연결
```

이 발생하였다.

---

## 82. 이번 실습에서 확인한 TLS 정보

```text
Domain
→ site1.bangkyu.com

Protocol
→ TLS 1.3

Cipher
→ TLS_AES_256_GCM_SHA384

Public Key
→ RSA 2048 bit

Certificate
→ Self-Signed

SAN
→ DNS:site1.bangkyu.com
```

---

## 83. 최종 구조

```text
                         Client-L
                      192.168.111.150
                             |
                             | 1. DNS Query
                             v
                       Master DNS
                    192.168.111.100
                             |
                             | A Record
                             v
                   site1.bangkyu.com
                    192.168.111.100
                             |
                             | 2. TCP 443
                             v
                       TLS Handshake
                             |
                             | 3. Certificate
                             v
              site1.bangkyu.com.crt
                             |
                     CN / SAN 확인
                             |
                             | 4. TLS 연결
                             v
                         Apache
                             |
                             | 5. HTTP Request
                             v
                    /var/www/site1
                             |
                             v
                         index.html
                             |
                             v
                        HTTP 200
```

---

## 84. 최종 이해

이번 실습에서 가장 중요한 흐름은 다음과 같다.

```text
Domain
   ↓
DNS
   ↓
Server IP
   ↓
TCP 443
   ↓
TLS Handshake
   ↓
Certificate 확인
   ↓
암호화 연결
   ↓
HTTP Request
   ↓
Apache
   ↓
Web Page
```

HTTPS는 단순히 TCP 443을 사용하는 HTTP가 아니라 TLS를 통해 Client와 Server 사이에 보안 연결을 만든 뒤 그 연결 위에서 HTTP를 사용하는 방식이다.

또한 HTTPS Service가 정상적으로 동작하는 것과 Certificate가 Client에게 신뢰되는 것은 서로 구분해서 이해해야 한다.

이번 실습의 Self-Signed Certificate는 Client가 기본적으로 신뢰하지 않았기 때문에 인증서 검증 오류가 발생하였지만, `curl -k`와 `openssl s_client`를 통해 TLS 연결 자체와 Apache HTTPS Service가 정상적으로 동작하고 있음을 확인하였다.
