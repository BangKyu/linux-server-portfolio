# DNS + Web / FTP 통합 실습 정리

## 1. 이번 실습의 목적

이번 실습의 목적은 DNS Server에서 Domain Name을 IP 주소로 변환하는 것에서 끝나지 않고, 실제 Web Server와 FTP Server까지 연결하여 전체 서비스 흐름을 확인하는 것이다.

구성:

```text
Client-L
192.168.111.150
        |
        | DNS Query
        v
Master DNS
192.168.111.100
        |
        +-------------------------+
        |                         |
        v                         v
www.bangkyu.com           ftp.bangkyu.com
192.168.111.100            192.168.111.200
        |                         |
        v                         v
Apache httpd                 vsftpd
TCP 80                       TCP 21
```

---

# DNS와 실제 서비스

## 2. DNS의 역할

DNS는 Domain Name을 IP 주소로 변환한다.

예:

```text
www.bangkyu.com
        ↓
192.168.111.100
```

```text
ftp.bangkyu.com
        ↓
192.168.111.200
```

하지만 DNS는 IP 주소를 알려주는 역할까지만 수행한다.

실제 Web Page를 보여주는 것은:

```text
Apache
```

이고,

실제 FTP 파일 전송을 처리하는 것은:

```text
vsftpd
```

이다.

즉:

```text
DNS
→ 어디로 접속할지 알려줌

Web / FTP Server
→ 실제 서비스를 제공
```

---

## 3. DNS 조회와 서비스 접속의 차이

예:

```bash
nslookup www.bangkyu.com
```

결과:

```text
www.bangkyu.com
→ 192.168.111.100
```

이것은 DNS가 정상이라는 의미이다.

하지만 이것만으로 Web Server가 정상이라는 뜻은 아니다.

Web Server까지 확인하려면:

```bash
curl http://www.bangkyu.com
```

과 같이 실제 HTTP 접속을 해야 한다.

따라서:

```text
DNS 성공
≠
Web Server 성공
```

이다.

전체가 정상인지 확인하려면:

```text
DNS 조회
→ IP 확인
→ Port 접속
→ Service 응답
```

까지 확인해야 한다.

---

# Web Server

## 4. Apache란?

Apache는 대표적인 Web Server Software이다.

Rocky Linux에서는 Package 이름이:

```text
httpd
```

이다.

Service 이름도:

```text
httpd
```

이다.

주로 사용하는 Port:

```text
TCP 80
```

HTTPS는 일반적으로:

```text
TCP 443
```

을 사용한다.

---

## 5. Apache 설치

```bash
dnf install -y httpd
```

Service 실행:

```bash
systemctl enable --now httpd
```

의미:

```text
enable
→ 부팅 시 자동 실행

--now
→ 지금 즉시 실행
```

---

## 6. Apache Document Root

Apache 기본 Web Page Directory:

```text
/var/www/html/
```

대표적인 기본 Page:

```text
/var/www/html/index.html
```

이번 실습:

```bash
vi /var/www/html/index.html
```

내용:

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

---

## 7. HTTP Firewall 설정

Apache가 실행 중이어도 Firewall이 TCP 80을 차단하면 외부 Client가 접속하지 못한다.

이번 실습:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-services
```

결과에:

```text
http
```

가 있어야 한다.

---

## 8. Port Listen 확인

```bash
ss -lntp | grep ':80 '
```

실제 결과:

```text
LISTEN 0 511 *:80 *:* users:(("httpd",...))
```

의미:

```text
*:80
→ 모든 IPv4 Interface에서 TCP 80 접속 대기

httpd
→ Apache Process가 Port 사용
```

---

## 9. IP를 이용한 Web 접속

```bash
curl http://192.168.111.100
```

이 명령은 DNS가 필요 없다.

이유:

```text
이미 목적지 IP 주소를 직접 지정했기 때문
```

흐름:

```text
curl
↓
192.168.111.100
↓
TCP 80
↓
Apache
↓
index.html
```

---

## 10. Domain을 이용한 Web 접속

```bash
curl http://www.bangkyu.com
```

이 경우는 먼저 DNS 조회가 필요하다.

흐름:

```text
curl http://www.bangkyu.com
        ↓
DNS Query
        ↓
www.bangkyu.com = 192.168.111.100
        ↓
TCP 80 접속
        ↓
Apache
        ↓
index.html 반환
```

---

# DNS Resolver 문제

## 11. Server-A에서 Domain 접속 실패

처음 실행:

```bash
curl http://www.bangkyu.com
```

결과:

```text
curl: (6) Could not resolve host: www.bangkyu.com
```

하지만:

```bash
curl http://192.168.111.100
```

은 정상 동작하였다.

따라서 Apache 문제는 아니었다.

---

## 12. 원인

Server-A의 `/etc/resolv.conf`:

```text
nameserver 192.168.111.2
```

NetworkManager:

```text
IP4.DNS[1]: 192.168.111.2
```

Server-A는 자신이 구축한 Master DNS:

```text
192.168.111.100
```

이 아니라 VMware DNS:

```text
192.168.111.2
```

를 사용하고 있었다.

VMware DNS는 내부에서 만든:

```text
bangkyu.com
```

Zone을 모르기 때문에 이름 해석에 실패하였다.

---

## 13. DNS Server를 운영하는 것과 사용하는 것은 다르다

중요한 개념:

```text
named가 실행 중이다
```

와:

```text
현재 Linux가 그 named를 DNS Server로 사용한다
```

는 별개의 문제이다.

즉 Server-A가 DNS Server 역할을 하고 있어도 `/etc/resolv.conf`가 다른 DNS를 가리키고 있다면 Server-A 자신은 다른 DNS를 사용한다.

---

## 14. DNS Resolver 변경

NetworkManager를 사용하여 Server-A가 자신의 DNS Server를 사용하도록 변경하였다.

```bash
nmcli connection modify ens160 ipv4.ignore-auto-dns yes
nmcli connection modify ens160 ipv4.dns "192.168.111.100"

nmcli device reapply ens160
```

의미:

```text
ipv4.ignore-auto-dns yes
→ DHCP가 제공하는 DNS를 사용하지 않음

ipv4.dns "192.168.111.100"
→ 사용할 DNS Server 직접 지정
```

---

## 15. getent hosts

일반 Linux Resolver가 실제로 이름을 해석할 수 있는지 확인할 때 사용할 수 있다.

```bash
getent hosts www.bangkyu.com
```

실제 결과:

```text
192.168.111.100 www.bangkyu.com
```

FTP:

```bash
getent hosts ftp.bangkyu.com
```

결과:

```text
192.168.111.200 ftp.bangkyu.com
```

---

## 16. dig와 getent의 차이

예:

```bash
dig @192.168.111.100 www.bangkyu.com
```

은 DNS Server를 직접 지정한다.

반면:

```bash
getent hosts www.bangkyu.com
```

은 시스템의 Resolver 설정을 이용한다.

일반 Application:

```text
curl
ping
ssh
ftp
```

등도 일반적으로 시스템 Resolver를 사용한다.

따라서:

```text
dig @DNS주소
→ DNS Server 자체 테스트

getent / curl / ping
→ 실제 OS Resolver 환경 테스트
```

로 생각하면 이해하기 쉽다.

---

# FTP Server

## 17. FTP란?

FTP는:

```text
File Transfer Protocol
```

의 약자이다.

Network를 통해 파일을 전송하기 위한 Protocol이다.

이번 실습에서는:

```text
vsftpd
```

를 FTP Server로 사용하였다.

---

## 18. vsftpd

vsftpd:

```text
Very Secure FTP Daemon
```

Rocky Linux에서 대표적으로 사용할 수 있는 FTP Server이다.

Package:

```text
vsftpd
```

Service:

```text
vsftpd
```

기본 제어 Port:

```text
TCP 21
```

---

## 19. vsftpd 설치

```bash
dnf install -y vsftpd
```

실행:

```bash
systemctl enable --now vsftpd
```

확인:

```bash
systemctl status vsftpd
```

실제:

```text
Active: active (running)
```

---

## 20. FTP Port 확인

```bash
ss -lntp | grep ':21 '
```

실제 결과:

```text
LISTEN 0 32 *:21 *:* users:(("vsftpd",pid=2339,fd=3))
```

의미:

```text
TCP 21
→ FTP 접속 대기

vsftpd
→ 해당 Port를 사용하는 Process
```

---

## 21. FTP Firewall

```bash
firewall-cmd --permanent --add-service=ftp
firewall-cmd --reload
```

FTP Service를 Firewall에서 허용하였다.

---

# FTP 로그인

## 22. ftp.bangkyu.com 접속

Client-L에서:

```bash
ftp ftp.bangkyu.com
```

흐름:

```text
ftp.bangkyu.com
        ↓
DNS Query
        ↓
192.168.111.200
        ↓
TCP 21
        ↓
vsftpd
```

실제:

```text
Connected to ftp.bangkyu.com (192.168.111.200).
```

Domain이 DNS를 통해 Server-B의 IP로 정상 변환되었음을 확인할 수 있다.

---

## 23. FTP 로그인

`guest` 계정으로 로그인하였다.

성공 시:

```text
230 Login successful.
```

이 출력된다.

현재 Directory 확인:

```text
ftp> pwd
```

실제:

```text
257 "/home/guest" is the current directory
```

즉 `guest` 계정의 FTP 작업 Directory가:

```text
/home/guest
```

임을 확인하였다.

---

# FTP 명령어

## 24. FTP 내부 주요 명령어

현재 Remote Directory 확인:

```text
pwd
```

Remote Directory 목록:

```text
ls
```

Local Directory 확인/변경:

```text
lcd
```

Upload:

```text
put
```

Download:

```text
get
```

종료:

```text
quit
```

---

## 25. FTP에서 Local과 Remote

FTP에서는 Client와 Server의 File System을 구분해야 한다.

```text
Local
→ FTP Client의 File System

Remote
→ FTP Server의 File System
```

예:

```text
Client-L
/tmp/ftp-test.txt
```

은 Local File이다.

Server-B:

```text
/home/guest/ftp-test.txt
```

은 Remote File이다.

---

# Passive Mode

## 26. Passive Mode란?

FTP에서는 제어 연결과 데이터 연결을 별도로 사용한다.

제어:

```text
TCP 21
```

Directory Listing이나 파일 전송은 별도의 Data Connection을 사용한다.

실제 출력:

```text
227 Entering Passive Mode (192,168,111,200,185,99).
```

이것은 FTP Client가 Passive Mode로 데이터 연결을 준비했다는 의미이다.

이후:

```text
150 Here comes the directory listing.
226 Directory send OK.
```

이 나왔다.

즉 Directory Listing이 정상적으로 완료되었다.

---

## 27. FTP 응답 코드

이번 실습에서 확인한 주요 Code:

```text
220
→ FTP Server 준비 완료

331
→ Password 필요

230
→ 로그인 성공

227
→ Passive Mode 진입

150
→ Data Transfer 시작 준비

226
→ Data Transfer 정상 완료

257
→ 현재 Directory 정보

221
→ FTP 연결 정상 종료

553
→ Server에서 File 생성 실패
```

---

# FTP Upload

## 28. Upload 테스트 파일

Client-L:

```bash
cd /tmp
echo "FTP upload test from Client-L" > ftp-test.txt
```

확인:

```bash
cat ftp-test.txt
```

결과:

```text
FTP upload test from Client-L
```

---

## 29. put 명령

FTP Upload 명령:

```text
put
```

형식:

```text
put <Local File> <Remote File>
```

예:

```text
put /tmp/ftp-test.txt ftp-test.txt
```

의미:

```text
Client-L
/tmp/ftp-test.txt

        ↓ FTP Upload

Server-B
현재 Remote Directory의 ftp-test.txt
```

현재 Remote Directory가:

```text
/home/guest
```

라면 최종 저장 위치:

```text
/home/guest/ftp-test.txt
```

---

## 30. Upload 중 발생한 553 오류

처음 실행:

```text
put /tmp/ftp-test.txt
```

결과:

```text
local: /tmp/ftp-test.txt remote: /tmp/ftp-test.txt
553 Could not create file.
```

FTP Client가 Remote File도:

```text
/tmp/ftp-test.txt
```

로 사용하려고 하면서 파일 생성에 실패하였다.

Remote File Name을 명확하게 지정하면:

```text
put /tmp/ftp-test.txt ftp-test.txt
```

처럼 사용할 수 있다.

---

## 31. Upload 확인

Server-B에서:

```bash
ls -l /home/guest
```

실제 결과:

```text
-rw-r--r--. 1 guest guest 30 Sep 14 12:26 ftp-test.txt
```

내용 확인:

```bash
cat /home/guest/ftp-test.txt
```

결과:

```text
FTP upload test from Client-L
```

따라서 Client-L → Server-B Upload가 성공하였다.

---

# FTP Download

## 32. get 명령

FTP Download:

```text
get
```

형식:

```text
get <Remote File> <Local File>
```

이번 실습에서는 Server-B의:

```text
/home/guest/ftp-test.txt
```

를 Client-L의:

```text
/tmp/ftp-download.txt
```

로 Download하였다.

---

## 33. Upload와 Download 흐름

Upload:

```text
Client-L
/tmp/ftp-test.txt
        |
        | put
        v
Server-B
/home/guest/ftp-test.txt
```

Download:

```text
Server-B
/home/guest/ftp-test.txt
        |
        | get
        v
Client-L
/tmp/ftp-download.txt
```

---

# 파일 무결성

## 34. 무결성이란?

File Integrity는 파일 내용이 전송 전후에 변경되지 않았는지를 확인하는 것이다.

예:

```text
전송 전 File
=
전송 후 File
```

인지 검증한다.

---

## 35. SHA-256

이번 실습에서는 SHA-256 Hash를 이용하여 파일 무결성을 확인하였다.

Server-B:

```bash
sha256sum /home/guest/ftp-test.txt
```

결과:

```text
465afcc258aab4e540233253410e37a9090d8e1082cd5762bec4d261b72ddb0c
```

Client-L:

```bash
sha256sum /tmp/ftp-download.txt
```

결과:

```text
465afcc258aab4e540233253410e37a9090d8e1082cd5762bec4d261b72ddb0c
```

두 Hash가 동일하였다.

따라서:

```text
FTP 전송 과정에서
파일 내용이 변경되지 않음
```

을 확인하였다.

---

## 36. Hash의 특징

Hash는 File 내용으로부터 일정한 길이의 값을 계산한다.

같은 File:

```text
동일한 Hash
```

내용이 변경된 File:

```text
다른 Hash
```

가 나올 가능성이 매우 높다.

따라서 파일 전송 후 무결성 확인에 사용할 수 있다.

---

# Hostname

## 37. Linux Hostname

이번 실습에서는 Server를 쉽게 구분하기 위해 Hostname을 변경하였다.

Server-A:

```bash
hostnamectl set-hostname Server-A
```

Server-B:

```bash
hostnamectl set-hostname Server-B
```

Client:

```bash
hostnamectl set-hostname Client-L
```

결과:

```text
[root@Server-A ~]#
[root@Server-B ~]#
[guest@Client-L ~]$
```

처럼 어떤 시스템에서 작업 중인지 쉽게 확인할 수 있다.

---

## 38. Hostname과 DNS Name 차이

Linux Hostname:

```text
Server-A
Server-B
Client-L
```

DNS Name:

```text
www.bangkyu.com
ftp.bangkyu.com
```

둘은 같은 개념이 아니다.

Hostname:

```text
Linux System 자체의 이름
```

DNS Name:

```text
Network에서 Service 또는 Host를 찾기 위해 사용하는 이름
```

이다.

---

# 전체 서비스 흐름

## 39. Web 접속 전체 과정

Client가:

```text
http://www.bangkyu.com
```

에 접속하면:

```text
1. www.bangkyu.com 이름 확인

2. DNS Server 192.168.111.100에 Query

3. A Record 확인

   www.bangkyu.com
   → 192.168.111.100

4. Client가 192.168.111.100 TCP 80 연결

5. httpd가 요청 수신

6. /var/www/html/index.html 반환

7. Client가 Web Page 확인
```

---

## 40. FTP 접속 전체 과정

Client가:

```bash
ftp ftp.bangkyu.com
```

을 실행하면:

```text
1. ftp.bangkyu.com 이름 확인

2. DNS Server에 Query

3. A Record 확인

   ftp.bangkyu.com
   → 192.168.111.200

4. Client가 Server-B TCP 21 연결

5. vsftpd가 연결 수신

6. guest 로그인

7. /home/guest 접근

8. put / get으로 File Transfer
```

---

# 문제 발생 시 확인 순서

## 41. Domain 접속이 안 될 때

예:

```text
Could not resolve host
```

확인:

```bash
cat /etc/resolv.conf
```

```bash
nmcli device show ens160 | grep IP4.DNS
```

```bash
nslookup www.bangkyu.com
```

```bash
getent hosts www.bangkyu.com
```

즉 먼저:

```text
DNS 문제인지
```

확인한다.

---

## 42. DNS는 되는데 Web 접속이 안 될 때

확인:

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
curl http://192.168.111.100
```

확인 순서:

```text
Service 실행 여부
↓
Port Listen 여부
↓
Firewall
↓
Local 접속
↓
Remote 접속
```

---

## 43. FTP 접속이 안 될 때

확인:

```bash
systemctl status vsftpd
```

```bash
ss -lntp | grep ':21 '
```

```bash
firewall-cmd --list-services
```

```bash
nslookup ftp.bangkyu.com
```

---

## 44. FTP Login은 되는데 Upload가 안 될 때

확인할 항목:

```text
1. Remote 현재 Directory
2. File / Directory Permission
3. FTP Write 설정
4. Remote File 이름
5. SELinux
```

FTP에서:

```text
pwd
```

로 현재 위치를 먼저 확인한다.

---

# 주요 명령어 정리

## 45. Apache

설치:

```bash
dnf install -y httpd
```

실행:

```bash
systemctl enable --now httpd
```

상태:

```bash
systemctl status httpd
```

Port:

```bash
ss -lntp | grep ':80 '
```

Web Page:

```text
/var/www/html/index.html
```

접속:

```bash
curl http://192.168.111.100
curl http://www.bangkyu.com
```

---

## 46. DNS Resolver

확인:

```bash
cat /etc/resolv.conf
```

```bash
nmcli device show ens160 | grep IP4.DNS
```

설정:

```bash
nmcli connection modify ens160 ipv4.ignore-auto-dns yes
nmcli connection modify ens160 ipv4.dns "192.168.111.100"
nmcli device reapply ens160
```

조회:

```bash
getent hosts www.bangkyu.com
getent hosts ftp.bangkyu.com
```

---

## 47. vsftpd

설치:

```bash
dnf install -y vsftpd
```

실행:

```bash
systemctl enable --now vsftpd
```

상태:

```bash
systemctl status vsftpd
```

Port:

```bash
ss -lntp | grep ':21 '
```

---

## 48. FTP Client

접속:

```bash
ftp ftp.bangkyu.com
```

FTP 내부:

```text
pwd
ls
lcd
put
get
quit
```

---

## 49. Firewall

HTTP:

```bash
firewall-cmd --permanent --add-service=http
```

FTP:

```bash
firewall-cmd --permanent --add-service=ftp
```

적용:

```bash
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-services
```

---

## 50. File Integrity

```bash
sha256sum <파일>
```

이번 실습:

```bash
sha256sum /home/guest/ftp-test.txt
```

```bash
sha256sum /tmp/ftp-download.txt
```

두 Hash 비교:

```text
같음
→ 파일 내용 동일
```

---

# 핵심 정리

## 51. DNS + Web

```text
www.bangkyu.com
        ↓
DNS
        ↓
192.168.111.100
        ↓
TCP 80
        ↓
Apache
        ↓
index.html
```

---

## 52. DNS + FTP

```text
ftp.bangkyu.com
        ↓
DNS
        ↓
192.168.111.200
        ↓
TCP 21
        ↓
vsftpd
        ↓
guest
        ↓
File Upload / Download
```

---

## 53. 이번 실습에서 이해해야 할 핵심

```text
DNS
→ 이름을 IP 주소로 변환

Apache
→ HTTP Web Service 제공

vsftpd
→ FTP File Transfer Service 제공

Firewall
→ Service Port 접근 허용

ss
→ 실제 Port Listen 확인

systemctl
→ Service 상태 관리

curl
→ HTTP Service 확인

ftp
→ FTP Service 접속

getent
→ OS Resolver 기준 이름 해석 확인

sha256sum
→ File Integrity 확인
```

---

## 54. 장애 분석 핵심

이번 실습에서 실제 발생한 문제:

```text
curl: (6) Could not resolve host
```

원인:

```text
Server-A가
192.168.111.100이 아니라
192.168.111.2 DNS를 사용
```

해결:

```text
NetworkManager DNS 설정 변경
→ 192.168.111.100
```

또한 FTP Upload 과정에서는:

```text
553 Could not create file
```

을 확인하였으며 Local File과 Remote File 경로를 구분하여 처리하였다.

따라서 Service 문제를 해결할 때는 단순히 프로그램만 확인하는 것이 아니라:

```text
DNS
Network
Service
Port
Firewall
Permission
Application 설정
```

순서로 확인하는 것이 중요하다.

---

## 55. 최종 구성

```text
Server-A
Hostname: Server-A
IP: 192.168.111.100

Role:
- Master DNS
- Caching DNS
- Apache Web Server

DNS:
www.bangkyu.com
→ 192.168.111.100
```

```text
Server-B
Hostname: Server-B
IP: 192.168.111.200

Role:
- vsftpd FTP Server

DNS:
ftp.bangkyu.com
→ 192.168.111.200
```

```text
Client-L
IP: 192.168.111.150

DNS:
192.168.111.100

사용 Service:
- DNS
- HTTP
- FTP
```

---

## 56. 최종 실습 흐름

```text
                    Client-L
                 192.168.111.150
                        |
                        |
                        v
                Master DNS Server
                 192.168.111.100
                        |
            +-----------+-----------+
            |                       |
            |                       |
            v                       v
    www.bangkyu.com         ftp.bangkyu.com
    192.168.111.100          192.168.111.200
            |                       |
          TCP 80                   TCP 21
            |                       |
            v                       v
        Apache                    vsftpd
            |                       |
            v                       v
        Web Page             guest Login
                                    |
                              File Transfer
                              put / get
                                    |
                               SHA-256
                                    |
                              Integrity OK
```

이번 실습을 통해 DNS가 단순히 Domain을 IP로 변환하는 기능에 그치지 않고, 실제 Web Server와 FTP Server Service를 찾기 위한 기반으로 사용되는 전체 과정을 확인하였다.

또한 DNS Resolver 설정 오류, FTP File Upload 문제를 직접 확인하고 해결하면서 DNS → Network → Port → Service → Application으로 이어지는 Server Troubleshooting 흐름을 실습하였다.
