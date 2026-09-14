# HTTP → HTTPS Redirect 이론 정리

## 1. HTTP → HTTPS Redirect란?

사용자가 HTTP로 접속했을 때 Web Server가 HTTPS 주소로 다시 접속하도록 안내하는 방식이다.

예:

```text
http://site1.bangkyu.com
        ↓
HTTP 301 Redirect
        ↓
https://site1.bangkyu.com
```

이번 실습에서는 Apache의 HTTP VirtualHost가 Web Page를 직접 제공하지 않고 HTTPS 주소를 Client에게 전달하도록 구성하였다.

---

# 기존 구조

## 2. Redirect 적용 전

기존 `site1.bangkyu.com`은 HTTP TCP 80에서 직접 Web Page를 제공하였다.

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
        ↓
HTTP 200
```

Apache 설정:

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

---

# Redirect 적용 후

## 3. HTTP의 역할 변경

Redirect를 적용한 후 HTTP TCP 80에서는 Web Page를 직접 제공하지 않는다.

HTTP의 역할:

```text
HTTP 요청 수신
        ↓
HTTPS 주소 전달
        ↓
301 Moved Permanently
```

설정:

```apache
<VirtualHost *:80>
    ServerName site1.bangkyu.com

    Redirect permanent / https://site1.bangkyu.com/

    ErrorLog logs/site1-error.log
    CustomLog logs/site1-access.log combined
</VirtualHost>
```

---

## 4. HTTPS의 역할

실제 Web Page는 HTTPS TCP 443에서 제공한다.

```text
Client
        ↓
https://site1.bangkyu.com
        ↓
TCP 443
        ↓
TLS
        ↓
Apache
        ↓
DocumentRoot /var/www/site1
        ↓
index.html
        ↓
HTTP 200
```

즉:

```text
HTTP :80
→ Redirect 전용

HTTPS :443
→ 실제 Web Service
```

로 역할이 분리된다.

---

# Redirect

## 5. Redirect란?

Redirect는 Client가 요청한 Resource를 다른 URL로 다시 요청하도록 Web Server가 알려주는 기능이다.

Server가 대신 새로운 URL에 접속하는 것이 아니다.

Server는 Client에게:

```text
이 Resource는 다른 주소에 있으니
그 주소로 다시 요청하세요.
```

라고 알려준다.

---

## 6. Redirect 동작

Client 요청:

```text
GET / HTTP/1.1
Host: site1.bangkyu.com
```

Server 응답:

```text
HTTP/1.1 301 Moved Permanently
Location: https://site1.bangkyu.com/
```

그 후 Client가 새로운 요청을 생성한다.

```text
https://site1.bangkyu.com/
```

따라서 Redirect는 실제로 두 번의 HTTP 요청이 발생할 수 있다.

---

# HTTP 301

## 7. 301 Moved Permanently

HTTP Status Code:

```text
301
```

의 의미:

```text
요청한 Resource가
다른 위치로 영구적으로 이동
```

이번 실습:

```text
기존 주소

http://site1.bangkyu.com
```

새 주소:

```text
https://site1.bangkyu.com/
```

---

## 8. permanent

Apache 설정:

```apache
Redirect permanent / https://site1.bangkyu.com/
```

여기서:

```text
permanent
```

는 HTTP Status Code:

```text
301 Moved Permanently
```

를 의미한다.

---

## 9. 실제 301 응답

이번 실습:

```bash
curl -I http://site1.bangkyu.com
```

실제 결과:

```text
HTTP/1.1 301 Moved Permanently
Location: https://site1.bangkyu.com/
```

따라서 Apache가 HTTP 요청을 HTTPS로 Redirect하고 있음을 확인하였다.

---

# Location Header

## 10. Location Header란?

Redirect 응답에서 Client가 새로 접속해야 할 주소를 알려주는 HTTP Header이다.

이번 실습:

```text
Location: https://site1.bangkyu.com/
```

Client는 이 값을 이용하여 다음 요청을 수행할 수 있다.

```text
https://site1.bangkyu.com/
```

---

## 11. 301과 Location의 관계

Redirect 응답은 다음 두 정보가 중요하다.

```text
301
→ 이동해야 한다는 의미

Location
→ 어디로 이동해야 하는지 알려줌
```

즉:

```text
HTTP/1.1 301 Moved Permanently

Location:
https://site1.bangkyu.com/
```

두 정보가 함께 사용된다.

---

# HTTP 200

## 12. 200 OK

HTTP Status Code:

```text
200
```

은 요청이 정상적으로 처리되었다는 의미이다.

Redirect 후 HTTPS 요청:

```text
https://site1.bangkyu.com/
```

이 정상 처리되면:

```text
200 OK
```

가 반환된다.

---

## 13. 이번 실습의 Status Code 흐름

```text
HTTP 요청
        ↓
301
        ↓
HTTPS 요청
        ↓
200
```

즉:

```text
301
→ HTTPS로 이동하라는 응답

200
→ HTTPS에서 실제 요청 처리 성공
```

이다.

---

# HTTP와 HTTPS 역할 분리

## 14. HTTP TCP 80

이번 실습에서:

```text
TCP 80
```

의 역할:

```text
Client의 HTTP 요청 수신
        ↓
HTTPS 주소 반환
        ↓
301
```

실제 Web Page를 직접 제공하지 않는다.

---

## 15. HTTPS TCP 443

```text
TCP 443
```

의 역할:

```text
TLS 연결
        ↓
암호화된 HTTP 통신
        ↓
실제 Web Page 제공
```

DocumentRoot:

```text
/var/www/site1
```

---

## 16. DocumentRoot를 HTTP에서 제거한 이유

기존 HTTP 설정:

```apache
DocumentRoot /var/www/site1
```

은 TCP 80에서 실제 Web File을 제공하기 위해 필요하였다.

하지만 Redirect 적용 후 TCP 80에서는:

```text
Web File 제공 X
HTTPS Redirect O
```

이므로 HTTP VirtualHost에 `DocumentRoot`가 필요하지 않다.

실제 Web File:

```text
/var/www/site1/index.html
```

은 삭제되지 않는다.

HTTPS TCP 443에서 계속 사용한다.

---

## 17. Directory 설정을 제거한 이유

기존:

```apache
<Directory /var/www/site1>
    Require all granted
</Directory>
```

은 `/var/www/site1`의 Web 접근을 허용하는 설정이었다.

그러나 HTTP VirtualHost가 해당 Directory의 파일을 직접 제공하지 않으므로 HTTP :80 설정에서는 필요하지 않게 된다.

실제 Page는 HTTPS 설정에서 제공된다.

---

# 전체 요청 과정

## 18. 첫 번째 요청

Client:

```text
http://site1.bangkyu.com
```

DNS:

```text
site1.bangkyu.com
→ 192.168.111.100
```

연결:

```text
192.168.111.100:80
```

Apache:

```text
site1 HTTP VirtualHost
```

응답:

```text
301 Moved Permanently
```

---

## 19. 두 번째 요청

Client는 `Location` Header를 확인한다.

```text
https://site1.bangkyu.com/
```

다시 접속:

```text
192.168.111.100:443
```

TLS 연결 후 Apache가:

```text
/var/www/site1/index.html
```

을 제공한다.

응답:

```text
200 OK
```

---

## 20. 전체 구조

```text
Client-L
192.168.111.150
        |
        | HTTP
        v
site1.bangkyu.com:80
        |
        v
Apache
        |
        | 301
        v
Location:
https://site1.bangkyu.com/
        |
        | 새로운 요청
        v
site1.bangkyu.com:443
        |
        v
TLS
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

# curl

## 21. curl -I

```bash
curl -I http://site1.bangkyu.com
```

`-I` 옵션은 Response Body 전체를 받는 대신 Header를 중심으로 확인한다.

이번 실습에서는 Redirect 상태 확인에 사용하였다.

실제:

```text
HTTP/1.1 301 Moved Permanently
Location: https://site1.bangkyu.com/
```

---

## 22. HEAD Method

`curl -I`를 사용하면 Server Log에 다음과 같이 기록될 수 있다.

```text
HEAD / HTTP/1.1
```

실제:

```text
"HEAD / HTTP/1.1" 301
```

`HEAD`는 `GET`과 비슷하지만 Response Body를 전달받지 않고 Header 정보를 확인하는 데 사용된다.

---

## 23. GET Method

일반적인 Web Page 요청:

```text
GET /
```

실제 Redirect 적용 후 HTTP Log:

```text
"GET / HTTP/1.1" 301
```

Redirect 후 HTTPS Log:

```text
"GET / HTTP/1.1" 200
```

---

# curl -L

## 24. -L 옵션

기본 `curl`은 Redirect 응답을 확인해도 자동으로 새 URL을 따라가지 않을 수 있다.

`-L`:

```text
--location
```

옵션을 사용하면 `Location` Header를 따라 새로운 URL로 다시 요청한다.

예:

```bash
curl -L http://site1.bangkyu.com
```

흐름:

```text
HTTP
↓
301
↓
Location
↓
HTTPS
```

---

## 25. 이번 실습에서 -k도 사용한 이유

현재 HTTPS 환경에서는 Self-Signed Certificate를 사용한다.

따라서:

```bash
curl -L http://site1.bangkyu.com
```

으로 HTTPS까지 따라가면 인증서 검증 단계에서 실패할 수 있다.

그래서:

```bash
curl -k -L http://site1.bangkyu.com
```

을 사용하였다.

---

## 26. -k

```text
-k
```

는:

```text
--insecure
```

옵션이다.

Certificate Trust 검증을 생략한다.

이번 실습에서는 Self-Signed Certificate 환경에서 HTTPS Redirect 동작을 검증하기 위해 사용하였다.

실제 운영 환경에서 무조건 `-k`를 사용하는 것은 권장되지 않는다.

---

# Redirect와 TLS

## 27. Redirect 자체는 암호화가 아니다

중요한 점:

```text
301 Redirect
```

자체가 통신을 암호화하는 것은 아니다.

Redirect는 단순히:

```text
HTTPS 주소로 다시 접속하세요.
```

라고 알려주는 기능이다.

실제 암호화는 Client가 TCP 443으로 다시 접속하여 TLS 연결을 만든 뒤 시작된다.

---

## 28. HTTP 요청은 처음에 존재한다

사용자가:

```text
http://site1.bangkyu.com
```

으로 접속하면 첫 요청 자체는 HTTP이다.

```text
HTTP :80
        ↓
301
        ↓
HTTPS :443
```

이기 때문에 첫 HTTP 요청과 Redirect 응답은 TLS 보호를 받는 HTTPS 요청과는 구분해야 한다.

---

# Apache Redirect Directive

## 29. Redirect Directive

이번 실습:

```apache
Redirect permanent / https://site1.bangkyu.com/
```

Apache가 URL을 다른 위치로 Redirect하도록 설정한다.

구조:

```text
Redirect
상태
기존 Path
새 URL
```

이번 설정:

```text
Redirect
permanent
/
https://site1.bangkyu.com/
```

---

## 30. `/`의 의미

```apache
Redirect permanent / https://site1.bangkyu.com/
```

에서:

```text
/
```

은 해당 Site의 Root부터 시작하는 요청을 대상으로 한다.

예:

```text
http://site1.bangkyu.com/
```

뿐만 아니라 해당 경로 아래의 요청도 Redirect 대상이 될 수 있다.

---

# 다른 VirtualHost

## 31. site1만 Redirect

이번 설정은:

```text
site1.bangkyu.com
```

의 VirtualHost 안에 적용하였다.

따라서:

```text
site1
→ 301
```

이 된다.

---

## 32. site2는 기존 HTTP 유지

실제:

```bash
curl -I http://site2.bangkyu.com
```

결과:

```text
HTTP/1.1 200 OK
```

즉 site2는 Redirect되지 않았다.

---

## 33. www도 기존 HTTP 유지

```bash
curl -I http://www.bangkyu.com
```

실제:

```text
HTTP/1.1 200 OK
```

따라서 `www.bangkyu.com`도 기존 HTTP Service를 유지하였다.

---

## 34. VirtualHost별 설정 독립성

Apache에서는 VirtualHost별로 서로 다른 동작을 구성할 수 있다.

현재:

```text
www.bangkyu.com
→ HTTP 200

site1.bangkyu.com
→ HTTP 301 → HTTPS

site2.bangkyu.com
→ HTTP 200
```

이다.

즉 동일한 Apache Server에서도 Domain별로 정책을 다르게 적용할 수 있다.

---

# Log

## 35. HTTP Access Log

site1 HTTP:

```text
/var/log/httpd/site1-access.log
```

Redirect 적용 후 Client-L:

```text
192.168.111.150 ... "HEAD / HTTP/1.1" 301
192.168.111.150 ... "GET / HTTP/1.1" 301
```

HTTP 요청이 실제로 Redirect 처리된 것을 확인하였다.

---

## 36. HTTPS Access Log

HTTPS:

```text
/var/log/httpd/ssl_access_log
```

Client-L:

```text
192.168.111.150 ... "GET / HTTP/1.1" 200 194
```

Redirect 이후 HTTPS 요청이 정상 처리된 것을 확인하였다.

---

## 37. 두 Log를 같이 봐야 하는 이유

HTTP Log:

```text
301
```

만 보면:

```text
Redirect 응답을 보냈다
```

는 것만 확인할 수 있다.

HTTPS Log:

```text
200
```

까지 확인하면:

```text
Client가 실제로 Redirect를 따라가
HTTPS Site까지 접속했다
```

는 것을 확인할 수 있다.

따라서:

```text
HTTP Log
301
        +
HTTPS Log
200
        =
Redirect End-to-End 검증
```

이다.

---

# Redirect 적용 전후 Log 비교

## 38. 적용 전

Client-L:

```text
192.168.111.150 ... "GET / HTTP/1.1" 200 194
```

의미:

```text
HTTP :80
→ 직접 Web Page 제공
```

---

## 39. 적용 후

HTTP:

```text
192.168.111.150 ... "GET / HTTP/1.1" 301
```

HTTPS:

```text
192.168.111.150 ... "GET / HTTP/1.1" 200 194
```

의미:

```text
HTTP
→ Redirect

HTTPS
→ 실제 Page 제공
```

---

# 301과 302

## 40. 301과 302 차이

대표적인 Redirect Status Code:

```text
301 Moved Permanently
→ 영구적인 이동

302 Found
→ 일시적인 이동
```

이번 실습은 HTTP를 지속적으로 HTTPS로 전환하는 목적이므로:

```text
301
```

을 사용하였다.

---

## 41. 301 사용 시 주의

301은 영구적인 Redirect 의미이므로 Browser나 Client가 Redirect 정보를 Cache할 수 있다.

설정 테스트 중 Redirect 정책을 자주 변경해야 하는 환경이라면 Cache의 영향을 고려해야 한다.

실습에서는 명령줄 `curl`을 이용하여 Response를 직접 확인하였다.

---

# Redirect Loop

## 42. Redirect Loop란?

Redirect 설정을 잘못하면 반복적으로 Redirect되는 문제가 생길 수 있다.

예:

```text
HTTP
→ HTTPS

HTTPS
→ 다시 HTTP

HTTP
→ 다시 HTTPS

...
```

이런 상태를 Redirect Loop라고 한다.

---

## 43. 이번 구성에서 Loop가 발생하지 않는 이유

HTTP :80:

```text
HTTPS로 Redirect
```

HTTPS :443:

```text
실제 Page 제공
```

으로 역할이 명확하게 분리되어 있다.

```text
HTTP
→ HTTPS
→ 종료
```

구조이므로 정상적으로 Page가 출력된다.

---

# Redirect 문제 해결

## 44. 301이 나오지 않을 때

확인:

```bash
cat /etc/httpd/conf.d/site1.conf
```

다음 설정 확인:

```apache
Redirect permanent / https://site1.bangkyu.com/
```

Apache 문법:

```bash
httpd -t
```

Service:

```bash
systemctl is-active httpd
```

---

## 45. Redirect는 되는데 HTTPS가 안 될 때

HTTP에서:

```text
301
```

이 나오더라도 HTTPS가 실패할 수 있다.

확인:

```bash
ss -lntp | grep ':443 '
```

```bash
firewall-cmd --list-services
```

```bash
curl -k https://site1.bangkyu.com
```

즉:

```text
Redirect 성공
≠
HTTPS 성공
```

이다.

두 단계를 별도로 확인해야 한다.

---

## 46. HTTPS Certificate 오류

이번 환경:

```text
Self-Signed Certificate
```

이므로 일반:

```bash
curl https://site1.bangkyu.com
```

은:

```text
self-signed certificate
```

오류가 발생할 수 있다.

실습 검증:

```bash
curl -k https://site1.bangkyu.com
```

---

## 47. 다른 Site까지 Redirect될 때

확인:

```bash
httpd -S
```

각 VirtualHost의:

```text
ServerName

설정 파일

Default VirtualHost
```

를 확인한다.

Redirect Directive가 원하는 VirtualHost 내부에 들어 있는지도 확인한다.

---

# 주요 명령어

## 48. HTTP Header

```bash
curl -I http://site1.bangkyu.com
```

---

## 49. Redirect Follow

```bash
curl -k -L http://site1.bangkyu.com
```

---

## 50. Apache 설정 검사

```bash
httpd -t
```

---

## 51. Apache VirtualHost

```bash
httpd -S
```

---

## 52. Apache Service

```bash
systemctl restart httpd
systemctl is-active httpd
```

---

## 53. HTTP Log

```bash
tail -n 10 /var/log/httpd/site1-access.log
```

---

## 54. HTTPS Log

```bash
tail -n 10 /var/log/httpd/ssl_access_log
```

---

# 핵심 정리

## 55. Redirect 핵심

```text
Redirect
=
Client에게 다른 URL로
다시 요청하도록 알려주는 기능
```

---

## 56. HTTP 301 핵심

```text
301 Moved Permanently
=
Resource의 위치가
영구적으로 변경됨
```

새 주소는:

```text
Location Header
```

로 전달한다.

---

## 57. 이번 설정의 핵심

```apache
Redirect permanent / https://site1.bangkyu.com/
```

의미:

```text
site1의 HTTP 요청
        ↓
301
        ↓
HTTPS 주소 전달
```

---

## 58. Port 역할

```text
TCP 80
→ HTTP
→ Redirect 전용
```

```text
TCP 443
→ HTTPS
→ TLS
→ 실제 Web Page 제공
```

---

## 59. Status Code 흐름

```text
HTTP :80
        ↓
301 Moved Permanently
        ↓
HTTPS :443
        ↓
200 OK
```

---

## 60. curl 옵션

```text
-I
→ Header 확인

-L
→ Redirect Follow

-k
→ Certificate 신뢰 검증 생략
```

이번 실습:

```bash
curl -k -L http://site1.bangkyu.com
```

은:

```text
HTTP 요청
→ Redirect 따라가기
→ Self-Signed Certificate 검증 생략
→ HTTPS Page 확인
```

의 의미이다.

---

## 61. 로그를 통한 검증

HTTP Log:

```text
192.168.111.150
GET /
301
```

HTTPS Log:

```text
192.168.111.150
GET /
200
```

따라서:

```text
Client-L
        ↓
HTTP
        ↓
301
        ↓
HTTPS
        ↓
200
```

전체 동작을 실제 Server Log에서 검증하였다.

---

# 최종 구조

## 62. 최종 서비스 흐름

```text
                         Client-L
                      192.168.111.150
                             |
                             | DNS
                             v
                   site1.bangkyu.com
                    192.168.111.100
                             |
                             | HTTP
                             v
                          TCP 80
                             |
                             v
                 Apache HTTP VirtualHost
                             |
                             | 301
                             v
                    Location Header
                             |
                             v
              https://site1.bangkyu.com/
                             |
                             | HTTPS
                             v
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

## 63. 최종 이해

이번 실습에서 가장 중요한 흐름은 다음과 같다.

```text
HTTP Request
        ↓
Apache :80
        ↓
301 Redirect
        ↓
Location Header
        ↓
HTTPS Request
        ↓
TCP 443
        ↓
TLS
        ↓
Apache
        ↓
Web Page
        ↓
200 OK
```

HTTP → HTTPS Redirect는 HTTP 통신 자체를 HTTPS로 변환하는 것이 아니라, HTTP 요청을 받은 Server가 Client에게 HTTPS URL로 다시 요청하도록 알려주는 방식이다.

따라서 최초 HTTP 요청과 Redirect 응답이 존재하고, 이후 Client가 HTTPS TCP 443으로 새로운 연결을 생성한다.

이번 실습에서는 `site1.bangkyu.com`의 HTTP VirtualHost를 Redirect 전용으로 변경하고 실제 Web Page는 HTTPS VirtualHost에서 제공하도록 역할을 분리하였다.

또한 HTTP Access Log에서 `301`, HTTPS SSL Access Log에서 `200`을 확인하여 Client-L이 실제로 HTTP → HTTPS Redirect를 따라 정상적으로 Web Page에 접근한 전체 과정을 검증하였다.
