# Apache VirtualHost 이론 정리

## 1. VirtualHost란?

VirtualHost는 하나의 Apache Web Server에서 여러 개의 Web Site를 운영할 수 있도록 하는 기능이다.

예:

```text
site1.bangkyu.com
site2.bangkyu.com
www.bangkyu.com
```

세 Domain을 하나의 Apache Server에서 서비스할 수 있다.

이번 실습에서는 모든 Domain이 같은 IP를 사용하였다.

```text
www.bangkyu.com
→ 192.168.111.100

site1.bangkyu.com
→ 192.168.111.100

site2.bangkyu.com
→ 192.168.111.100
```

Port 역시 동일하다.

```text
TCP 80
```

하지만 접속한 Domain Name에 따라 서로 다른 Web Page를 제공한다.

---

## 2. VirtualHost를 사용하는 이유

Web Site마다 Server를 한 대씩 사용하면 다음과 같은 구조가 된다.

```text
Site 1
→ Server 1

Site 2
→ Server 2

Site 3
→ Server 3
```

Site가 많아질수록 필요한 Server도 증가한다.

VirtualHost를 사용하면 하나의 Server에서 여러 Site를 운영할 수 있다.

```text
                Apache Server
             192.168.111.100
                     |
         +-----------+-----------+
         |           |           |
         v           v           v
       www         site1       site2
```

따라서 Server 자원을 효율적으로 사용할 수 있다.

---

# VirtualHost 종류

## 3. Name-based VirtualHost

Domain Name을 기준으로 Web Site를 구분하는 방식이다.

이번 실습에서 사용한 방식이다.

```text
site1.bangkyu.com
        ↓
192.168.111.100:80
        ↓
Apache
        ↓
/var/www/site1
```

```text
site2.bangkyu.com
        ↓
192.168.111.100:80
        ↓
Apache
        ↓
/var/www/site2
```

IP와 Port는 동일하지만 Domain Name이 다르다.

```text
IP
192.168.111.100

Port
80

Domain
site1.bangkyu.com
site2.bangkyu.com
```

Apache는 HTTP 요청에 포함된 Host 정보를 확인하여 사용할 VirtualHost를 결정한다.

---

## 4. IP-based VirtualHost

서로 다른 IP 주소를 이용하여 Web Site를 구분하는 방식이다.

예:

```text
192.168.111.101
→ SITE 1

192.168.111.102
→ SITE 2
```

구조:

```text
SITE 1
→ 192.168.111.101:80

SITE 2
→ 192.168.111.102:80
```

각 Site마다 별도의 IP가 필요하다.

---

## 5. Port-based VirtualHost

Port 번호를 이용하여 Site를 구분할 수도 있다.

예:

```text
192.168.111.100:80
→ SITE 1

192.168.111.100:8080
→ SITE 2
```

하지만 사용자가 URL에 Port 번호를 직접 입력해야 할 수 있다.

예:

```text
http://192.168.111.100:8080
```

일반적인 Web Service에서는 Name-based VirtualHost가 많이 사용된다.

---

# Name-based VirtualHost 동작 원리

## 6. DNS의 역할

Client가 다음 주소에 접속한다고 가정한다.

```text
http://site1.bangkyu.com
```

먼저 DNS를 이용하여 IP를 확인한다.

```text
site1.bangkyu.com
        ↓
DNS Query
        ↓
192.168.111.100
```

DNS의 역할은 여기까지이다.

즉 DNS는:

```text
Domain
→ IP
```

변환만 수행한다.

DNS가 `/var/www/site1`을 선택하는 것은 아니다.

---

## 7. Apache의 역할

DNS를 통해 목적지 IP를 확인한 후 Client가 Web Server에 접속한다.

```text
192.168.111.100:80
```

Apache는 HTTP 요청 안에 들어있는 Host Name을 확인한다.

예:

```text
Host: site1.bangkyu.com
```

Apache 설정에:

```apache
ServerName site1.bangkyu.com
```

이 존재하면 해당 VirtualHost를 선택한다.

결과:

```text
Host: site1.bangkyu.com
        ↓
ServerName site1.bangkyu.com
        ↓
DocumentRoot /var/www/site1
```

---

# HTTP Host Header

## 8. Host Header란?

HTTP 요청에는 Client가 어떤 Domain으로 접속했는지 나타내는 정보가 포함된다.

예:

```http
GET / HTTP/1.1
Host: site1.bangkyu.com
```

여기서:

```text
Host: site1.bangkyu.com
```

이 Apache가 Name-based VirtualHost를 선택하는 데 중요한 정보이다.

---

## 9. 같은 IP인데 다른 Site가 나오는 이유

두 Domain의 DNS 결과가 다음과 같다고 가정한다.

```text
site1.bangkyu.com
→ 192.168.111.100

site2.bangkyu.com
→ 192.168.111.100
```

둘 다 같은 Server에 접속한다.

하지만 HTTP Host가 다르다.

SITE 1:

```text
Host: site1.bangkyu.com
```

SITE 2:

```text
Host: site2.bangkyu.com
```

Apache가 이 값을 보고 서로 다른 VirtualHost를 선택한다.

```text
site1.bangkyu.com
        ↓
/var/www/site1
```

```text
site2.bangkyu.com
        ↓
/var/www/site2
```

즉:

```text
DNS
→ 어느 Server로 갈 것인지 결정

Apache VirtualHost
→ Server 안에서 어느 Site를 보여줄지 결정
```

이라고 이해하면 된다.

---

# VirtualHost 설정

## 10. 기본 형식

예:

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

## 11. `<VirtualHost *:80>`

```apache
<VirtualHost *:80>
```

의미:

```text
*:80
```

`*`:

```text
Server의 모든 IP Interface
```

`80`:

```text
TCP 80 Port
```

즉:

```text
Server의 TCP 80으로 들어오는 HTTP 요청
```

을 처리하는 VirtualHost이다.

---

## 12. ServerName

```apache
ServerName site1.bangkyu.com
```

해당 VirtualHost의 대표 Domain Name을 설정한다.

Apache는 Client의 Host 정보와 `ServerName`을 비교한다.

```text
Client 요청

Host: site1.bangkyu.com

        ↓ 비교

ServerName site1.bangkyu.com

        ↓

일치
        ↓
해당 VirtualHost 선택
```

Name-based VirtualHost에서 매우 중요한 설정이다.

---

## 13. ServerAlias

하나의 VirtualHost가 추가 Domain Name도 처리하도록 설정할 수 있다.

예:

```apache
ServerName www.bangkyu.com
ServerAlias bangkyu.com
```

그러면 다음 두 Domain을 같은 Site가 처리할 수 있다.

```text
www.bangkyu.com
bangkyu.com
```

예:

```apache
<VirtualHost *:80>
    ServerName www.bangkyu.com
    ServerAlias bangkyu.com

    DocumentRoot /var/www/html
</VirtualHost>
```

구조:

```text
www.bangkyu.com ─┐
                 ├→ /var/www/html
bangkyu.com ─────┘
```

---

## 14. DocumentRoot

```apache
DocumentRoot /var/www/site1
```

해당 Web Site의 문서가 저장되는 기본 Directory이다.

예:

```text
DocumentRoot
/var/www/site1
```

Client 요청:

```text
http://site1.bangkyu.com/
```

Apache:

```text
/var/www/site1/index.html
```

을 제공한다.

---

## 15. Domain과 Directory의 관계

예:

```apache
ServerName site1.bangkyu.com
DocumentRoot /var/www/site1
```

의미:

```text
site1.bangkyu.com
        ↓
/var/www/site1
```

다른 VirtualHost:

```apache
ServerName site2.bangkyu.com
DocumentRoot /var/www/site2
```

의미:

```text
site2.bangkyu.com
        ↓
/var/www/site2
```

최종:

```text
site1.bangkyu.com
→ /var/www/site1/index.html

site2.bangkyu.com
→ /var/www/site2/index.html
```

---

# Directory 설정

## 16. `<Directory>`

예:

```apache
<Directory /var/www/site1>
    Require all granted
</Directory>
```

Apache가 해당 Directory를 Web을 통해 접근할 때 적용할 규칙을 설정한다.

---

## 17. Require all granted

```apache
Require all granted
```

의미:

```text
모든 Client의 접근 허용
```

즉 외부 Client가 해당 Web Directory의 내용을 요청할 수 있도록 허용한다.

반대로 접근 정책이 제한되어 있으면 파일이 존재해도 Web에서 접근하지 못할 수 있다.

---

# Apache Log

## 18. Access Log

```apache
CustomLog logs/site1-access.log combined
```

SITE 1에 접속한 Client의 요청을 기록한다.

이번 실습의 실제 Log:

```text
192.168.111.150 - - [14/Sep/2026:13:08:36 +0900] "GET / HTTP/1.1" 200 194 "-" "curl/7.76.1"
```

주요 의미:

```text
192.168.111.150
→ 접속한 Client IP

GET /
→ Web Root 요청

HTTP/1.1
→ 사용한 HTTP Version

200
→ 요청 정상 처리

194
→ 응답 Data 크기

curl/7.76.1
→ 요청한 Client Program
```

---

## 19. Error Log

```apache
ErrorLog logs/site1-error.log
```

해당 VirtualHost에서 발생한 오류를 기록한다.

예를 들어:

```text
파일 없음
Permission 문제
설정 문제
Application 오류
```

등을 확인할 때 사용한다.

---

## 20. Site별 Log 분리

VirtualHost마다 로그를 따로 지정할 수 있다.

예:

```text
SITE 1

/var/log/httpd/site1-access.log
/var/log/httpd/site1-error.log
```

```text
SITE 2

/var/log/httpd/site2-access.log
/var/log/httpd/site2-error.log
```

장점:

```text
어느 Site에 누가 접속했는지 확인 가능

Site별 오류 분석 가능

Site별 Traffic 확인 가능
```

---

# Default VirtualHost

## 21. Default VirtualHost란?

Client가 요청한 Host Name과 정확하게 일치하는 VirtualHost가 없을 경우 Apache가 기본으로 선택하는 VirtualHost이다.

`httpd -S`에서 확인할 수 있다.

예:

```text
default server site1.bangkyu.com
```

이 경우 Apache가 매칭되는 VirtualHost를 찾지 못하면 SITE 1이 기본적으로 처리될 수 있다.

---

## 22. 첫 번째 VirtualHost

같은 IP와 Port를 사용하는 Name-based VirtualHost에서는 설정을 읽은 순서상 첫 번째 VirtualHost가 기본 VirtualHost가 될 수 있다.

예:

```text
00-www.conf
site1.conf
site2.conf
```

처럼 구성하면 `00-www.conf`가 먼저 읽히도록 만들 수 있다.

따라서 기존 `www.bangkyu.com`을 기본 Site로 사용할 수 있다.

---

## 23. 왜 00-www.conf를 사용하는가?

Apache의 `/etc/httpd/conf.d/` 아래 `.conf` 파일은 설정으로 읽힌다.

파일 이름을:

```text
00-www.conf
```

처럼 앞쪽에 오도록 만들면 다른 VirtualHost 설정보다 먼저 처리되도록 구성하기 쉽다.

예:

```text
00-www.conf
site1.conf
site2.conf
```

이렇게 하면 기존 www Site를 기본 VirtualHost로 관리하기 편하다.

---

# Global ServerName

## 24. Global ServerName이란?

Apache Server 자체의 기본 이름을 지정한다.

예:

```apache
ServerName www.bangkyu.com
```

VirtualHost 내부의 `ServerName`과는 목적이 조금 다르다.

---

## 25. AH00558 Warning

처음 `httpd -t` 실행 시 다음 Warning이 발생하였다.

```text
AH00558: httpd: Could not reliably determine the server's fully qualified domain name
```

의미:

```text
Apache가 자기 자신의 FQDN을 정확하게 결정하지 못함
```

이다.

Service가 반드시 실패했다는 뜻은 아니다.

실제로:

```text
Syntax OK
```

가 출력되었다면 설정 문법 자체는 정상이다.

---

## 26. Global ServerName 설정

```bash
vi /etc/httpd/conf.d/servername.conf
```

내용:

```apache
ServerName www.bangkyu.com
```

이후:

```bash
httpd -t
```

결과:

```text
Syntax OK
```

Global ServerName을 명확하게 지정하여 Warning을 제거할 수 있다.

---

# DNS와 VirtualHost

## 27. DNS A Record

VirtualHost를 Domain으로 접근하려면 DNS에서 해당 Domain을 Web Server IP로 연결해야 한다.

이번 실습:

```dns
site1   IN   A   192.168.111.100
site2   IN   A   192.168.111.100
```

결과:

```text
site1.bangkyu.com
→ 192.168.111.100

site2.bangkyu.com
→ 192.168.111.100
```

---

## 28. DNS만 설정하면 되는가?

아니다.

DNS만 설정하면:

```text
Domain
→ Server IP
```

까지만 가능하다.

Apache에도 VirtualHost 설정이 필요하다.

예:

```text
DNS

site1.bangkyu.com
→ 192.168.111.100
```

Apache:

```apache
ServerName site1.bangkyu.com
DocumentRoot /var/www/site1
```

두 설정이 모두 있어야 원하는 Site가 정상적으로 동작한다.

---

## 29. DNS와 Apache의 역할 비교

DNS:

```text
site1.bangkyu.com
        ↓
192.168.111.100
```

Apache:

```text
Host: site1.bangkyu.com
        ↓
VirtualHost 선택
        ↓
/var/www/site1
```

정리:

```text
DNS
= Server 찾기

VirtualHost
= Server 내부에서 Site 찾기
```

---

# DNS Serial

## 30. Zone Serial

DNS Zone File을 수정할 때 SOA Serial 값을 증가시킨다.

기존:

```text
2026091401
```

변경 후:

```text
2026091402
```

이번 실습에서는:

```dns
site1   IN   A   192.168.111.100
site2   IN   A   192.168.111.100
```

Record를 추가했기 때문에 Serial도 증가시켰다.

---

# Apache 설정 파일

## 31. `/etc/httpd/conf.d/`

Rocky Linux Apache에서 추가 설정 파일을 저장하는 Directory이다.

이번 실습:

```text
/etc/httpd/conf.d/00-www.conf
/etc/httpd/conf.d/site1.conf
/etc/httpd/conf.d/site2.conf
/etc/httpd/conf.d/servername.conf
```

구조:

```text
/etc/httpd/
└── conf.d/
    ├── 00-www.conf
    ├── servername.conf
    ├── site1.conf
    └── site2.conf
```

---

# Apache 설정 검사

## 32. `httpd -t`

Apache Configuration의 문법 오류를 확인한다.

```bash
httpd -t
```

정상:

```text
Syntax OK
```

Apache 설정을 변경한 후 Service를 Restart하기 전에 먼저 실행하는 것이 좋다.

```text
설정 수정
   ↓
httpd -t
   ↓
Syntax OK
   ↓
Apache Restart
```

---

## 33. `httpd -S`

Apache의 VirtualHost 구성을 확인한다.

```bash
httpd -S
```

확인 가능 내용:

```text
VirtualHost 목록

Default VirtualHost

ServerName

Port

설정 파일 위치
```

예:

```text
port 80 namevhost site1.bangkyu.com
port 80 namevhost site2.bangkyu.com
```

Apache가 두 Site를 별도의 Name-based VirtualHost로 인식하고 있다는 의미이다.

---

# SELinux

## 34. Web Directory와 SELinux

Rocky Linux는 SELinux를 사용한다.

Web Page를 새로운 Directory에 생성한 경우 Apache가 정상적으로 접근할 수 있는 SELinux Context인지 확인할 필요가 있다.

이번 실습:

```bash
restorecon -Rv /var/www/site1
restorecon -Rv /var/www/site2
```

`restorecon`은 정책에 정의된 기본 SELinux Context를 파일 또는 Directory에 다시 적용하는 명령이다.

SELinux를 단순히 Disable하는 대신 올바른 Context를 사용하는 것이 좋다.

---

# 전체 요청 흐름

## 35. SITE 1 접속 과정

Client가:

```text
http://site1.bangkyu.com
```

에 접근한다.

전체 과정:

```text
1. Client가 site1.bangkyu.com의 IP 확인

2. DNS Server에 Query

3. A Record 확인

   site1.bangkyu.com
   → 192.168.111.100

4. Client가 192.168.111.100 TCP 80 연결

5. HTTP 요청 전송

   Host: site1.bangkyu.com

6. Apache가 VirtualHost 검색

   ServerName site1.bangkyu.com

7. DocumentRoot 선택

   /var/www/site1

8. index.html 반환

9. Client가 SITE 1 확인
```

---

## 36. SITE 2 접속 과정

```text
site2.bangkyu.com
        ↓
DNS
        ↓
192.168.111.100
        ↓
TCP 80
        ↓
HTTP Host: site2.bangkyu.com
        ↓
Apache VirtualHost
        ↓
ServerName site2.bangkyu.com
        ↓
DocumentRoot /var/www/site2
        ↓
index.html
```

---

# 문제 해결

## 37. Domain 자체가 조회되지 않을 때

확인:

```bash
dig @192.168.111.100 site1.bangkyu.com A +short
```

또는:

```bash
nslookup site1.bangkyu.com
```

확인할 항목:

```text
DNS A Record
Zone File
Zone Serial
named Service
Client DNS 설정
```

---

## 38. DNS는 되지만 Web 접속이 안 될 때

DNS 결과:

```text
site1.bangkyu.com
→ 192.168.111.100
```

까지 정상인데 Web이 안 된다면 Apache 쪽을 확인한다.

```bash
systemctl status httpd
```

```bash
ss -lntp | grep ':80 '
```

```bash
firewall-cmd --list-services
```

```bash
httpd -t
```

---

## 39. 엉뚱한 Site가 나타날 때

확인:

```bash
httpd -S
```

주요 확인 항목:

```text
ServerName이 올바른지

Default VirtualHost가 무엇인지

설정 파일이 실제로 읽히는지

DocumentRoot가 올바른지
```

Client 요청 Host와 Apache `ServerName`이 일치해야 한다.

---

## 40. 403 Forbidden

가능한 확인 대상:

```text
Apache Directory 접근 설정

Linux File Permission

SELinux Context
```

VirtualHost에서:

```apache
<Directory /var/www/site1>
    Require all granted
</Directory>
```

와 같은 설정을 확인한다.

---

## 41. 404 Not Found

일반적으로 요청한 파일을 Apache가 찾지 못할 때 발생한다.

확인:

```text
DocumentRoot

index.html 존재 여부

요청 URL

파일명
```

예:

```bash
ls -l /var/www/site1/index.html
```

---

## 42. HTTP Status Code 200

이번 Access Log에서:

```text
"GET / HTTP/1.1" 200
```

을 확인하였다.

`200`:

```text
HTTP 요청 정상 처리
```

를 의미한다.

즉 Client-L에서 VirtualHost에 보낸 요청이 Apache에서 정상적으로 처리되었다.

---

# 주요 설정 정리

## 43. SITE 1

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

구조:

```text
site1.bangkyu.com
        ↓
192.168.111.100:80
        ↓
Apache
        ↓
/var/www/site1
```

---

## 44. SITE 2

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

구조:

```text
site2.bangkyu.com
        ↓
192.168.111.100:80
        ↓
Apache
        ↓
/var/www/site2
```

---

# 주요 명령어

## 45. DNS

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db

dig @192.168.111.100 site1.bangkyu.com A +short
dig @192.168.111.100 site2.bangkyu.com A +short
```

---

## 46. Apache

문법 검사:

```bash
httpd -t
```

VirtualHost 확인:

```bash
httpd -S
```

Service:

```bash
systemctl restart httpd
systemctl status httpd
```

Port:

```bash
ss -lntp | grep ':80 '
```

---

## 47. HTTP 확인

```bash
curl http://www.bangkyu.com
curl http://site1.bangkyu.com
curl http://site2.bangkyu.com
```

---

## 48. Log 확인

```bash
tail /var/log/httpd/site1-access.log
tail /var/log/httpd/site2-access.log
```

오류:

```bash
tail /var/log/httpd/site1-error.log
tail /var/log/httpd/site2-error.log
```

---

# 핵심 정리

## 49. VirtualHost 핵심

```text
VirtualHost
=
하나의 Apache Server에서
여러 Web Site를 운영하는 기능
```

---

## 50. Name-based VirtualHost 핵심

```text
동일한 IP
+
동일한 Port
+
서로 다른 Domain
```

을 사용한다.

예:

```text
site1.bangkyu.com ─┐
                   |
                   ├→ 192.168.111.100:80
                   |
site2.bangkyu.com ─┘
```

Apache는 HTTP Host Name을 확인하여 Site를 구분한다.

---

## 51. 핵심 설정

```text
VirtualHost
→ 적용할 IP / Port

ServerName
→ 해당 Site의 Domain

ServerAlias
→ 추가 Domain

DocumentRoot
→ Web File 위치

Directory
→ Directory 접근 정책

ErrorLog
→ 오류 기록

CustomLog
→ 접속 기록
```

---

## 52. DNS와 VirtualHost 관계

```text
DNS
        ↓
Domain을 IP로 변환

site1.bangkyu.com
        ↓
192.168.111.100
        ↓
Apache
        ↓
HTTP Host 확인
        ↓
VirtualHost 선택
        ↓
DocumentRoot 선택
```

---

## 53. 이번 실습의 최종 구조

```text
                        Client-L
                     192.168.111.150
                            |
                            | Domain 요청
                            v
                      Master DNS
                    192.168.111.100
                            |
                            | A Record
                            v
                    192.168.111.100
                            |
                         TCP 80
                            |
                            v
                         Apache
                            |
                     HTTP Host 확인
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
 www.bangkyu.com   site1.bangkyu.com  site2.bangkyu.com
          |                 |                 |
          v                 v                 v
 /var/www/html      /var/www/site1     /var/www/site2
          |                 |                 |
          v                 v                 v
     기존 Site           SITE 1            SITE 2
```

---

## 54. 최종 이해

이번 실습에서 가장 중요한 것은 다음 흐름이다.

```text
Domain Name
        ↓
DNS
        ↓
Server IP
        ↓
TCP 80
        ↓
Apache
        ↓
HTTP Host Name
        ↓
ServerName 비교
        ↓
VirtualHost 선택
        ↓
DocumentRoot
        ↓
Web Page
```

즉 DNS는 Domain이 어느 Server를 가리키는지를 결정하고, Apache VirtualHost는 해당 Server로 들어온 요청이 어느 Web Site에 대한 요청인지를 구분한다.

Name-based VirtualHost를 이용하면 하나의 Server IP와 하나의 TCP 80 Port만으로 여러 Domain의 Web Site를 각각 독립적인 DocumentRoot와 Log로 운영할 수 있다.
