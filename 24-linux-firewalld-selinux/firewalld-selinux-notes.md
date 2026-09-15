# Firewalld + SELinux Notes

Rocky Linux Server에서 Network 접근 제어를 담당하는 **Firewalld**와 Process/File/Port 접근 제어를 담당하는 **SELinux**의 개념과 장애 분석 방법을 정리한다.

이번 실습에서는 단순히 설정 방법만 확인하지 않고 다음 장애를 직접 재현하였다.

```text
1. Firewalld 때문에 외부 Client가 Port에 접근하지 못하는 장애

2. SELinux File Context가 잘못되어
   Apache에서 403 Forbidden이 발생하는 장애

3. SELinux Port Type 때문에
   Apache가 특정 Port에 Bind하지 못하는 장애
```

---

# 1. Linux Server 보안 계층

Server에 Client가 접근할 때 단순히 하나의 설정만 통과하는 것이 아니다.

Web Server를 예로 들면 다음과 같은 여러 계층을 통과해야 한다.

```text
Client
  ↓
Network
  ↓
Firewalld
  ↓
Server Port
  ↓
Application(httpd)
  ↓
SELinux Policy
  ↓
File / Directory
```

따라서 Web Service에 접속되지 않는다고 해서 무조건 다음처럼 판단하면 안 된다.

```text
"접속 안 됨 = Firewall 문제"
```

실제로는 다음과 같은 여러 원인이 존재할 수 있다.

```text
Service 중지
Port 미Listen
Firewalld 차단
잘못된 Apache 설정
Linux Permission 문제
SELinux File Context 문제
SELinux Port Type 문제
SELinux Boolean 문제
```

---

# 2. Firewalld란?

Firewalld는 Linux에서 Network Packet 접근을 제어하는 Firewall 관리 Service이다.

Rocky Linux에서는 기본 Firewall 관리 도구로 사용된다.

확인:

```bash
systemctl status firewalld
```

간단하게:

```bash
systemctl is-active firewalld
systemctl is-enabled firewalld
```

---

# 3. Firewalld의 역할

Firewalld는 다음과 같은 질문에 관여한다.

```text
"외부 Client의 Packet이
이 Server의 특정 Port까지 들어올 수 있는가?"
```

예:

```text
Client-L
192.168.111.150

        ↓ TCP 8081

Server-A
192.168.111.100

        ↓

Firewalld가 8081 허용?
```

8081이 허용되어 있지 않다면 Application이 정상이어도 외부 Client에서 접근하지 못할 수 있다.

---

# 4. Firewalld Zone

Firewalld는 Network Interface를 **Zone**에 연결하여 정책을 관리한다.

확인:

```bash
firewall-cmd --get-active-zones
```

실습 환경:

```text
public
  interfaces: ens160
```

즉:

```text
ens160
→ public Zone 정책 적용
```

---

# 5. Zone 확인

현재 기본 Zone:

```bash
firewall-cmd --get-default-zone
```

현재 활성 Zone:

```bash
firewall-cmd --get-active-zones
```

특정 Zone 전체 설정:

```bash
firewall-cmd --zone=public --list-all
```

현재 사용 중인 Zone이라면 다음처럼 생략할 수도 있다.

```bash
firewall-cmd --list-all
```

---

# 6. Firewalld Service와 Port

Firewalld에서는 Port를 직접 허용할 수도 있고 미리 정의된 Service를 허용할 수도 있다.

예:

```bash
firewall-cmd --add-service=http
```

또는:

```bash
firewall-cmd --add-port=80/tcp
```

둘 다 결과적으로 HTTP 접근을 가능하게 할 수 있지만 관리 방식이 다르다.

---

# 7. Firewalld Service 방식

Service는 여러 Network 설정을 이름으로 관리하기 위한 방식이다.

예:

```text
http
https
ssh
dns
samba
nfs
ntp
```

현재 허용 Service:

```bash
firewall-cmd --list-services
```

특정 Service 확인:

```bash
firewall-cmd --query-service=http
```

결과 예:

```text
yes
```

---

# 8. Firewalld Port 방식

특정 Port를 직접 허용할 수도 있다.

예:

```bash
firewall-cmd --add-port=8081/tcp
```

확인:

```bash
firewall-cmd --query-port=8081/tcp
```

현재 열린 Port:

```bash
firewall-cmd --list-ports
```

---

# 9. Runtime 설정

다음 명령은 현재 실행 중인 Firewalld에 즉시 적용된다.

```bash
firewall-cmd --add-port=8081/tcp
```

이를 **Runtime Configuration**이라고 한다.

확인:

```bash
firewall-cmd --query-port=8081/tcp
```

실습 결과:

```text
yes
```

하지만:

```bash
firewall-cmd --permanent --query-port=8081/tcp
```

결과는:

```text
no
```

였다.

즉:

```text
Runtime     → 8081 허용
Permanent   → 8081 미허용
```

상태였다.

---

# 10. Permanent 설정

영구 설정은 `--permanent` 옵션을 사용한다.

```bash
firewall-cmd --permanent --add-port=8081/tcp
```

Permanent 설정 확인:

```bash
firewall-cmd --permanent --query-port=8081/tcp
```

하지만 중요한 점이 있다.

```text
--permanent 설정
→ 설정 파일에는 저장됨
→ 현재 Runtime에 즉시 반영되는 것은 아님
```

일반적으로 다음이 필요하다.

```bash
firewall-cmd --reload
```

---

# 11. Runtime과 Permanent 비교

정리하면:

| 구분 | Runtime | Permanent |
|---|---|---|
| 즉시 적용 | O | X |
| Reload 후 유지 | X | O |
| Reboot 후 유지 | X | O |
| 옵션 | 기본 | `--permanent` |

예:

```bash
firewall-cmd --add-port=8081/tcp
```

```text
Runtime에 즉시 적용
```

반면:

```bash
firewall-cmd --permanent --add-port=8081/tcp
```

```text
Permanent 설정에 저장
```

이후:

```bash
firewall-cmd --reload
```

하여 Runtime에도 반영한다.

---

# 12. firewall-cmd --reload

```bash
firewall-cmd --reload
```

는 Permanent 설정을 다시 읽어 현재 Runtime 정책에 반영한다.

실습에서는:

```text
Runtime 8081 = yes
Permanent 8081 = no
```

상태에서:

```bash
firewall-cmd --reload
```

를 실행한 후:

```text
Runtime 8081 = no
```

로 돌아가는 것을 확인하였다.

즉 Runtime에만 존재했던 설정이 사라졌다.

---

# 13. Firewalld 장애 분석 방법

예를 들어 Client에서 다음 Port에 접속할 수 없다고 가정한다.

```text
192.168.111.100:8081
```

무조건 Firewall부터 변경하지 말고 순서대로 확인한다.

---

## 13-1. Process 확인

```bash
ps aux | grep <process>
```

또는:

```bash
systemctl status <service>
```

---

## 13-2. Listen Port 확인

```bash
ss -lntp
```

특정 Port:

```bash
ss -lntp | grep ':8081'
```

예:

```text
0.0.0.0:8081 LISTEN
```

이면 Application이 해당 Port에서 Connection을 기다리고 있다는 의미이다.

---

## 13-3. Local 접속

```bash
curl http://127.0.0.1:8081
```

Local 접속이 성공하면:

```text
Application 실행
Port Listen
Local HTTP 처리
```

는 정상일 가능성이 높다.

---

## 13-4. Client 접속

다른 Host에서:

```bash
curl --connect-timeout 3 http://192.168.111.100:8081
```

Local은 성공하고 Client만 실패한다면 Network 계층을 의심할 수 있다.

---

## 13-5. Firewalld 확인

```bash
firewall-cmd --query-port=8081/tcp
```

결과:

```text
no
```

라면 Firewall 정책이 원인 후보가 된다.

---

# 14. "No route to host"의 의미 주의

Client에서 다음 오류가 발생할 수 있다.

```text
curl: (7) Failed to connect ...
호스트로 갈 루트가 없음
```

영문 환경에서는:

```text
No route to host
```

가 표시될 수 있다.

이 메시지만 보고 반드시 Routing Table 문제라고 판단하면 안 된다.

Firewall 정책이나 Network 차단 과정에 따라 Client Application에서 비슷한 오류가 표시될 수 있다.

따라서 반드시 다음을 함께 확인한다.

```bash
ping
ip route
ss
firewall-cmd
curl
```

---

# 15. SELinux란?

SELinux는 **Security-Enhanced Linux**의 약자이다.

일반적인 Linux Permission보다 추가적인 보안 정책을 제공한다.

기본 Linux 접근 제어:

```text
Owner
Group
Other

r / w / x
```

SELinux는 여기에 추가로:

```text
Process가 어떤 Resource에
어떤 행동을 할 수 있는가?
```

를 정책으로 제어한다.

---

# 16. DAC와 MAC

일반 Linux Permission은 대표적으로 **DAC** 방식이다.

```text
DAC
Discretionary Access Control
```

SELinux는 **MAC** 방식에 해당한다.

```text
MAC
Mandatory Access Control
```

즉 일반 Permission이 허용하더라도 SELinux 정책이 거부하면 접근하지 못할 수 있다.

예:

```text
chmod 644 index.html
```

이라고 해도 SELinux Context가 잘못되어 있으면 Apache가 읽지 못할 수 있다.

---

# 17. SELinux Mode

확인:

```bash
getenforce
```

주요 Mode:

```text
Enforcing
Permissive
Disabled
```

---

## Enforcing

```text
SELinux 정책 적용
위반 동작 차단
Audit Log 기록
```

운영 환경에서 일반적으로 사용하는 상태이다.

---

## Permissive

```text
SELinux 정책 위반을 차단하지 않음
하지만 Audit Log는 기록
```

문제 분석 목적으로 일시적으로 사용할 수 있다.

---

## Disabled

```text
SELinux 기능 자체 비활성화
```

일반적인 문제 해결 방법으로 권장되지 않는다.

---

# 18. SELinux Context

SELinux에서는 Process와 File 등에 Security Context가 존재한다.

확인:

```bash
ls -Z
```

Directory:

```bash
ls -Zd /var/www/html
```

예:

```text
system_u:object_r:httpd_sys_content_t:s0
```

구조:

```text
user : role : type : level
```

실무에서 Troubleshooting할 때 가장 자주 확인하는 부분은:

```text
type
```

이다.

---

# 19. Apache Process Type

Apache는 SELinux 환경에서 일반적으로 다음 Domain으로 실행된다.

```text
httpd_t
```

Audit Log에서:

```text
scontext=system_u:system_r:httpd_t:s0
```

형태로 확인할 수 있다.

---

# 20. Apache Web Content Type

Apache가 일반적인 정적 Web Content로 읽을 수 있는 대표 Type:

```text
httpd_sys_content_t
```

실습 환경의:

```bash
ls -Zd /var/www/html
```

결과에도:

```text
httpd_sys_content_t
```

가 적용되어 있었다.

---

# 21. 잘못된 File Context

실습에서는:

```text
/srv/selinux-web
```

Directory를 만들었다.

Context:

```text
var_t
```

였다.

즉:

```text
Apache Process
→ httpd_t

Web File
→ var_t
```

상태였다.

Linux Permission은:

```text
Directory 755
File      644
```

로 정상이어도 Apache 요청 결과는:

```text
HTTP/1.1 403 Forbidden
```

이었다.

---

# 22. Permission과 SELinux의 관계

다음과 같이 생각할 수 있다.

```text
Linux Permission
       +
SELinux Policy
       ↓
최종 접근 가능 여부 결정
```

예:

```text
Linux Permission 허용 ✅
SELinux 허용 ✅
→ 접근 가능
```

```text
Linux Permission 거부 ❌
SELinux 허용 ✅
→ 접근 불가
```

```text
Linux Permission 허용 ✅
SELinux 거부 ❌
→ 접근 불가
```

따라서 SELinux 환경에서는 `chmod`만 봐서는 안 된다.

---

# 23. AVC란?

SELinux가 접근을 거부하면 Audit Log에 **AVC** 기록이 남을 수 있다.

AVC:

```text
Access Vector Cache
```

최근 SELinux 거부 기록:

```bash
ausearch -m AVC -ts recent
```

---

# 24. 실제 AVC File Denial 분석

실습에서 다음 Log가 확인되었다.

```text
avc: denied { getattr }
comm="httpd"
path="/srv/selinux-web/index.html"
scontext=system_u:system_r:httpd_t:s0
tcontext=unconfined_u:object_r:var_t:s0
tclass=file
permissive=0
```

각 부분의 의미를 보면 다음과 같다.

---

## denied

```text
avc: denied
```

SELinux가 동작을 거부했다는 의미이다.

---

## Operation

```text
{ getattr }
```

File의 Attribute를 조회하려는 동작이 거부되었다.

---

## comm

```text
comm="httpd"
```

접근을 시도한 Process가 Apache라는 의미이다.

---

## path

```text
path="/srv/selinux-web/index.html"
```

문제가 발생한 대상 File이다.

---

## scontext

```text
scontext=system_u:system_r:httpd_t:s0
```

Source Context이다.

즉 접근을 시도한 Process의 SELinux Context.

핵심 Type:

```text
httpd_t
```

---

## tcontext

```text
tcontext=unconfined_u:object_r:var_t:s0
```

Target Context이다.

즉 접근 대상 File의 SELinux Context.

핵심 Type:

```text
var_t
```

---

## tclass

```text
tclass=file
```

대상이 File이라는 의미이다.

---

## permissive

```text
permissive=0
```

정책이 실제로 강제되는 Enforcing 상황임을 나타낸다.

---

# 25. SELinux File Context 확인

File:

```bash
ls -Z /srv/selinux-web/index.html
```

Directory:

```bash
ls -Zd /srv/selinux-web
```

Troubleshooting 시 일반 `ls -l`뿐만 아니라 `ls -Z`도 확인하는 습관이 중요하다.

---

# 26. semanage fcontext

File이나 Directory가 어떤 SELinux Context를 가져야 하는지를 **정책에 등록**할 때 사용한다.

예:

```bash
semanage fcontext -a -t httpd_sys_content_t "/srv/selinux-web(/.*)?"
```

의미:

```text
/srv/selinux-web
그리고 그 아래 모든 경로

→ httpd_sys_content_t 사용
```

---

# 27. 정규식 (/.*)?

다음 표현:

```text
/srv/selinux-web(/.*)?
```

은:

```text
/srv/selinux-web
```

자체와:

```text
/srv/selinux-web/...
```

하위 모든 File/Directory를 포함하기 위해 사용한다.

---

# 28. semanage fcontext와 restorecon 차이

매우 중요하다.

```bash
semanage fcontext ...
```

는:

```text
"이 경로의 올바른 SELinux Context는 이것이다"
```

라는 규칙을 등록한다.

하지만 이 명령만으로 현재 File의 Label이 바로 변경되지 않을 수 있다.

실제 Label 적용:

```bash
restorecon -Rv /srv/selinux-web
```

---

## 쉽게 표현

```text
semanage fcontext
→ 정답표 등록

restorecon
→ 정답표를 실제 File에 적용
```

---

# 29. restorecon

기본 또는 사용자 정의 SELinux File Context 정책을 기준으로 실제 Label을 복구한다.

```bash
restorecon -Rv /srv/selinux-web
```

옵션:

```text
-R
→ Recursive
→ 하위 Directory까지 처리

-v
→ 변경 내용 표시
```

---

# 30. chcon과 semanage fcontext 차이

SELinux Label은 `chcon`으로도 변경할 수 있다.

예:

```bash
chcon -t httpd_sys_content_t file
```

하지만 `chcon`은 영구 정책 자체를 정의하는 방식이 아니다.

나중에:

```bash
restorecon
```

을 수행하면 정책에 정의된 원래 Context로 돌아갈 수 있다.

따라서 영구적으로 관리하려면 일반적으로:

```text
semanage fcontext
+
restorecon
```

방식을 사용한다.

---

# 31. chmod 777이 해결책이 아닌 이유

SELinux 문제에서 다음과 같이 접근하는 경우가 있다.

```bash
chmod 777 file
```

하지만 SELinux Denial이라면 Unix Permission을 777로 바꿔도 SELinux 정책은 별도로 적용된다.

즉:

```text
chmod 777
→ SELinux 정책을 변경하지 않음
```

불필요하게 Permission만 약화시킬 수 있다.

---

# 32. setenforce 0이 최종 해결책이 아닌 이유

```bash
setenforce 0
```

을 실행하면 SELinux가 Permissive 상태가 되어 차단하지 않는다.

그래서:

```text
setenforce 0 후 정상 작동
```

한다면 SELinux가 원인이라는 진단 단서로 사용할 수 있다.

그러나:

```text
SELinux 때문에 문제 발생
→ SELinux 꺼버림
```

은 올바른 운영 해결 방식이 아니다.

올바른 방향:

```text
AVC 확인
       ↓
잘못된 Context/Port/Boolean 확인
       ↓
필요한 정책만 수정
       ↓
SELinux Enforcing 유지
```

---

# 33. SELinux Boolean

SELinux Boolean은 특정 정책 기능을 간단히 On/Off할 수 있도록 제공되는 Switch이다.

전체 확인:

```bash
getsebool -a
```

Apache 관련:

```bash
getsebool -a | grep httpd
```

---

# 34. httpd_can_network_connect

확인:

```bash
getsebool httpd_can_network_connect
```

실습 결과:

```text
httpd_can_network_connect --> off
```

이 Boolean은 Apache/httpd Domain에서 일반적인 외부 Network Connection을 허용하는 것과 관련된다.

중요:

```text
httpd_can_network_connect
```

는 Apache가 HTTP Port에 **Listen하는 설정 그 자체와 동일한 개념이 아니다.**

Port Bind 문제는 `http_port_t` 같은 Port Type을 확인해야 한다.

---

# 35. httpd_can_network_connect_db

```bash
getsebool httpd_can_network_connect_db
```

실습 결과:

```text
off
```

Web Application이 Network를 통해 Database에 연결해야 하는 구성에서는 관련될 수 있다.

예:

```text
Apache/PHP
   ↓
MariaDB/MySQL
```

---

# 36. httpd_enable_homedirs

```bash
getsebool httpd_enable_homedirs
```

실습 결과:

```text
off
```

사용자의 Home Directory에 있는 Web Content 제공과 관련된 SELinux Boolean이다.

---

# 37. Boolean 변경

일시적인 변경:

```bash
setsebool <boolean> on
```

예:

```bash
setsebool httpd_can_network_connect on
```

영구 변경:

```bash
setsebool -P httpd_can_network_connect on
```

`-P`:

```text
Persistent
```

단, 무조건 Boolean을 활성화하는 것이 아니라 Application 요구사항에 필요한 경우에만 변경해야 한다.

---

# 38. SELinux Port Type

SELinux는 File뿐 아니라 Network Port에도 Type을 적용한다.

Apache용 대표 Port Type:

```text
http_port_t
```

확인:

```bash
semanage port -l | grep '^http_port_t'
```

실습 초기 결과:

```text
http_port_t tcp 80, 81, 443, 488, 8008, 8009, 8443, 9000
```

---

# 39. Firewalld Port와 SELinux Port 차이

이 부분이 매우 중요하다.

## Firewalld

질문:

```text
외부 Client가 Server의 이 Port로 들어올 수 있는가?
```

---

## SELinux Port Type

질문:

```text
이 Process Domain이 이 Port에 Bind할 수 있는가?
```

---

## 예

Apache가 8082에서 Listen하려고 한다.

```text
SELinux http_port_t에 8082 없음
```

이면:

```text
httpd
→ 8082 Bind 실패
```

가능성이 있다.

반대로:

```text
SELinux에서는 8082 허용
Firewalld에서는 8082 차단
```

이라면:

```text
Apache 8082 LISTEN 가능
localhost 접속 가능
외부 Client 접속 실패
```

가 될 수 있다.

---

# 40. SELinux Port Bind 장애

실습에서는 Apache에:

```apache
Listen 8082
```

를 추가하였다.

문법 검사:

```bash
httpd -t
```

결과:

```text
Syntax OK
```

즉 Apache Configuration 문법 자체에는 문제가 없었다.

하지만 SELinux AVC Log에서는:

```text
avc: denied { name_bind }
comm="httpd"
src=8082
scontext=system_u:system_r:httpd_t:s0
tcontext=system_u:object_r:us_cli_port_t:s0
tclass=tcp_socket
```

가 확인되었다.

---

# 41. name_bind

```text
denied { name_bind }
```

은 Process가 Network Port에 Bind하려는 동작을 SELinux가 차단했다는 의미이다.

실습에서는:

```text
httpd_t
→ TCP 8082
```

Bind 시도가 차단되었다.

---

# 42. 왜 8082에서 -a가 아닌 -m을 사용했는가?

처음에는 8082가 `http_port_t` 목록에 없었다.

하지만 AVC에서:

```text
tcontext=...:us_cli_port_t:s0
```

가 확인되었다.

또한:

```bash
semanage port -l | grep -w '8082'
```

로 보면 8082가 이미 다른 SELinux Port Type에 정의되어 있었다.

따라서 새로운 Port 정의를 추가하는:

```bash
semanage port -a ...
```

가 아니라 기존 Mapping을 변경하는:

```bash
semanage port -m -t http_port_t -p tcp 8082
```

를 사용하였다.

---

# 43. semanage port 옵션

새 Port Mapping 추가:

```bash
semanage port -a -t <type> -p tcp <port>
```

기존 Mapping 변경:

```bash
semanage port -m -t <type> -p tcp <port>
```

Local Customization 삭제:

```bash
semanage port -d -t <type> -p tcp <port>
```

---

# 44. Local SELinux Customization

사용자가 추가하거나 수정한 SELinux 정책만 보고 싶다면:

```bash
semanage port -l -C
```

File Context:

```bash
semanage fcontext -l -C
```

여기서:

```text
-C
```

는 Local Customization 확인에 매우 유용하다.

실습에서:

```bash
semanage port -l -C | grep '8082'
```

결과:

```text
http_port_t tcp 8082
```

가 확인되었다.

---

# 45. 기본 정책과 Local Override

실습 중 `semanage port -l` 출력에서는 다음처럼 보였다.

```text
http_port_t   tcp 8082, ...
us_cli_port_t tcp 8082, 8083
```

그리고:

```bash
semanage port -l -C
```

에서는:

```text
http_port_t tcp 8082
```

라는 Local Customization을 확인하였다.

실습 종료 후 Local Customization을 삭제하면 기본 정책 Mapping이 다시 사용된다.

실제 최종 확인:

```text
us_cli_port_t tcp 8082, 8083
us_cli_port_t udp 8082, 8083
```

---

# 46. SELinux Troubleshooting에서 Audit Log 중요성

SELinux 문제를 추측으로 해결하지 말고 실제 AVC 기록을 확인해야 한다.

기본 명령:

```bash
ausearch -m AVC -ts recent
```

최근 일부:

```bash
ausearch -m AVC -ts recent | tail
```

httpd 관련:

```bash
ausearch -m AVC -ts recent | grep httpd
```

---

# 47. ausearch 시간 조건

최근 기록:

```bash
ausearch -m AVC -ts recent
```

오늘:

```bash
ausearch -m AVC -ts today
```

특정 Service 관련 Log를 찾을 때 시간 범위를 좁히면 분석이 편하다.

---

# 48. AVC를 볼 때 핵심 항목

```text
denied
comm
path
scontext
tcontext
tclass
name_bind
read
write
getattr
permissive
```

특히 다음 질문에 답할 수 있어야 한다.

```text
누가?
→ scontext / comm

무엇에?
→ tcontext / path

무슨 행동을?
→ { read }, { write }, { name_bind } 등

왜 차단됐나?
→ Source Type과 Target Type 정책 관계
```

---

# 49. Firewalld와 SELinux 문제 구분

## Case 1

```text
Service active
Port LISTEN
localhost 접속 성공
Client 접속 실패
Firewall Port 미허용
```

의심:

```text
Firewalld / Network
```

---

## Case 2

```text
Apache active
Firewall 정상
Linux Permission 정상
HTTP 403
AVC denied 발생
잘못된 File Context
```

의심:

```text
SELinux File Context
```

---

## Case 3

```text
Apache 설정 Syntax OK
새 Port Listen 실패
AVC name_bind denied
```

의심:

```text
SELinux Port Type
```

---

# 50. HTTP 403과 SELinux

HTTP:

```text
403 Forbidden
```

이라고 해서 반드시 SELinux 문제라는 뜻은 아니다.

403 원인은 여러 가지가 있다.

```text
Apache Directory 설정
Require 설정
Linux Permission
File/Directory 소유권
SELinux Context
.htaccess
```

따라서:

```text
403 = SELinux
```

라고 단정하지 않고 실제 Log와 Context를 확인해야 한다.

---

# 51. Apache Troubleshooting 순서

Web Service 접속 장애 시 다음 순서가 유용하다.

### 1. Service

```bash
systemctl status httpd
```

---

### 2. Apache Configuration

```bash
httpd -t
```

---

### 3. Listen Port

```bash
ss -lntp | grep httpd
```

---

### 4. Local HTTP Test

```bash
curl -I http://127.0.0.1/
```

---

### 5. Firewalld

```bash
firewall-cmd --list-all
```

---

### 6. Linux Permission

```bash
ls -ld <directory>
ls -l <file>
```

---

### 7. SELinux Context

```bash
ls -Zd <directory>
ls -Z <file>
```

---

### 8. SELinux Port

```bash
semanage port -l
```

---

### 9. SELinux Boolean

```bash
getsebool -a | grep httpd
```

---

### 10. AVC Audit Log

```bash
ausearch -m AVC -ts recent
```

---

# 52. Service가 active라고 정상은 아니다

다음만 보고:

```bash
systemctl is-active httpd
```

```text
active
```

라고 나온다고 모든 Web 기능이 정상이라는 뜻은 아니다.

예를 들어:

```text
Process active
80 Listen
443 Listen
특정 VirtualHost 설정 오류
File Context 오류
```

등 다른 문제가 존재할 수 있다.

따라서:

```text
Service 상태
+
Port 상태
+
실제 요청
```

을 함께 확인해야 한다.

---

# 53. httpd -t

Apache 설정 변경 후에는 먼저:

```bash
httpd -t
```

를 실행하는 습관이 중요하다.

정상:

```text
Syntax OK
```

하지만 이것은:

```text
Apache 설정 문법이 올바르다
```

는 뜻이지:

```text
SELinux
Firewall
Network
File Permission
```

까지 정상이라는 뜻은 아니다.

실제로 이번 8082 실습에서도:

```text
Syntax OK
```

였지만 SELinux `name_bind` 정책에 의해 Port Bind가 차단되었다.

---

# 54. ss 명령의 중요성

실제 Port가 Listen 중인지 확인:

```bash
ss -lntp
```

옵션:

```text
-l
Listening

-n
Port를 숫자로 표시

-t
TCP

-p
Process 표시
```

예:

```bash
ss -lntp | grep ':8082'
```

---

# 55. 127.0.0.1과 0.0.0.0

Application이:

```text
127.0.0.1:8080
```

에만 Listen하면 Local Host에서만 접근할 수 있다.

반면:

```text
0.0.0.0:8080
```

은 모든 IPv4 Interface에서 Connection을 받을 수 있다는 의미이다.

따라서 외부 Client 접속 문제에서는 Bind Address도 확인해야 한다.

---

# 56. Firewalld Troubleshooting Checklist

```text
[ ] firewalld active인가?

[ ] Interface가 어떤 Zone에 들어가 있는가?

[ ] 올바른 Zone을 수정했는가?

[ ] 필요한 Service가 허용되어 있는가?

[ ] 필요한 Port가 허용되어 있는가?

[ ] Runtime인가 Permanent인가?

[ ] --reload 후 설정이 유지되는가?

[ ] Application이 실제 Port를 Listen하고 있는가?

[ ] localhost에서는 접속되는가?

[ ] Client에서만 실패하는가?
```

---

# 57. SELinux Troubleshooting Checklist

```text
[ ] SELinux가 Enforcing인가?

[ ] Linux Permission은 정상인가?

[ ] ls -Z Context는 정상인가?

[ ] Process Domain은 무엇인가?

[ ] File Type은 무엇인가?

[ ] ausearch에 AVC denied가 있는가?

[ ] SELinux Port Type은 정상인가?

[ ] 필요한 Boolean이 있는가?

[ ] Local Customization이 존재하는가?

[ ] setenforce 0으로 회피하지 않았는가?
```

---

# 58. 보안 장애를 해결할 때 피해야 할 방식

## SELinux Disable

```bash
setenforce 0
```

만으로 문제를 끝내지 않는다.

---

## 무조건 chmod 777

```bash
chmod -R 777 /some/path
```

은 보안을 약화시킬 수 있고 SELinux 문제를 해결하지 못할 수도 있다.

---

## Firewalld 전체 비활성화

```bash
systemctl stop firewalld
systemctl disable firewalld
```

로 Service 접근 문제를 해결하는 것은 적절한 운영 방식이 아니다.

필요한 Service 또는 Port만 허용하는 것이 바람직하다.

---

# 59. 최소 권한 원칙

보안 설정은 가능한 최소 범위만 허용한다.

예를 들어 Web Server에 TCP 8081이 필요하다면:

```text
모든 Firewall 기능 해제 ❌

필요한 8081/tcp만 허용 ✅
```

SELinux도:

```text
SELinux 전체 비활성화 ❌

필요한 File Context 또는 Port Type만 설정 ✅
```

이 원칙을 **Least Privilege**라고 한다.

---

# 60. Firewalld와 SELinux를 함께 사용하는 이유

두 기술은 역할이 겹치는 것이 아니라 서로 다른 계층을 보호한다.

```text
Firewalld
→ Network 접근 제어

SELinux
→ Process / Resource 접근 제어
```

예를 들어 공격자가 Firewall을 통과해 Apache까지 도달했더라도 SELinux 정책은 Apache Process가 시스템의 아무 File이나 읽지 못하도록 추가로 제한할 수 있다.

즉 여러 보안 계층을 두는 **Defense in Depth** 관점으로 볼 수 있다.

---

# 61. 실습에서 확인한 Firewalld 장애

실제 흐름:

```text
Python Web Server
TCP 8081
        ↓
Process 정상
        ↓
0.0.0.0:8081 LISTEN
        ↓
127.0.0.1 접속 성공
        ↓
Client-L 접속 실패
        ↓
firewall-cmd --query-port
        ↓
no
        ↓
8081 Runtime 허용
        ↓
Client-L 접속 성공
```

이를 통해 Application 장애와 Firewall 장애를 구분하였다.

---

# 62. 실습에서 확인한 Runtime/Permanent

```text
Runtime 8081
→ yes

Permanent 8081
→ no
```

상태에서 Reload 후:

```text
Runtime 8081
→ no
```

가 되었다.

Permanent까지 설정한 뒤에는:

```text
Runtime 8081
→ yes

Permanent 8081
→ yes
```

를 확인하였다.

---

# 63. 실습에서 확인한 SELinux File 장애

```text
/srv/selinux-web
→ var_t
```

Apache 요청:

```text
403 Forbidden
```

AVC:

```text
httpd_t
→ var_t
→ denied { getattr }
```

해결:

```bash
semanage fcontext ...
restorecon ...
```

수정 후:

```text
httpd_sys_content_t
```

HTTP 결과:

```text
200 OK
SELinux Test OK
```

---

# 64. 실습에서 확인한 SELinux Port 장애

Apache:

```apache
Listen 8082
```

문법:

```text
Syntax OK
```

하지만 AVC:

```text
denied { name_bind }
src=8082
httpd_t
us_cli_port_t
```

확인.

수정:

```bash
semanage port -m -t http_port_t -p tcp 8082
```

이후:

```text
httpd active
*:8082 LISTEN
HTTP 200 OK
```

를 확인하였다.

---

# 65. 장애 분석 핵심 흐름

실무적인 흐름으로 정리하면:

```text
접속 장애 발생
      ↓
Service가 실행 중인가?
      ↓
Port가 LISTEN 중인가?
      ↓
Local 접속은 가능한가?
      ↓
Client에서만 실패하는가?
      ↓
Firewalld 확인
      ↓
Application Log 확인
      ↓
Linux Permission 확인
      ↓
SELinux Context 확인
      ↓
AVC Log 확인
      ↓
Port Type / Boolean 확인
      ↓
정확한 원인만 수정
      ↓
재검증
```

---

# 66. 주요 Firewalld 명령어

상태:

```bash
systemctl status firewalld
```

Zone:

```bash
firewall-cmd --get-active-zones
firewall-cmd --get-default-zone
firewall-cmd --list-all
```

Service:

```bash
firewall-cmd --list-services
firewall-cmd --query-service=http
```

Port:

```bash
firewall-cmd --list-ports
firewall-cmd --query-port=8081/tcp
```

Runtime 추가:

```bash
firewall-cmd --add-port=8081/tcp
```

Runtime 삭제:

```bash
firewall-cmd --remove-port=8081/tcp
```

Permanent 추가:

```bash
firewall-cmd --permanent --add-port=8081/tcp
```

Permanent 삭제:

```bash
firewall-cmd --permanent --remove-port=8081/tcp
```

적용:

```bash
firewall-cmd --reload
```

---

# 67. 주요 SELinux 명령어

Mode:

```bash
getenforce
sestatus
```

File Context:

```bash
ls -Z
ls -Zd
```

File Context 정책:

```bash
semanage fcontext -l
semanage fcontext -l -C
```

Context 적용:

```bash
restorecon -Rv <path>
```

Port:

```bash
semanage port -l
semanage port -l -C
```

Boolean:

```bash
getsebool -a
getsebool <boolean>
```

Audit:

```bash
ausearch -m AVC -ts recent
```

---

# 68. 시험/면접식 핵심 질문

## Q. Firewalld와 SELinux의 차이는?

```text
Firewalld는 Network Packet 접근을 제어하고,
SELinux는 Process가 File, Port 등의 Resource에
접근할 수 있는지를 정책으로 제어한다.
```

---

## Q. Runtime과 Permanent 차이는?

```text
Runtime은 현재 실행 중인 Firewall에 즉시 적용되지만
Reload 또는 Reboot 후 유지되지 않을 수 있다.

Permanent는 영구 설정에 저장되며,
일반적으로 reload 후 Runtime에도 적용한다.
```

---

## Q. chmod 777인데 Apache가 File을 못 읽는 이유는?

```text
Linux Permission과 SELinux는 별도의 접근 제어 계층이므로
SELinux File Context가 잘못되어 있다면
Permission이 허용되어 있어도 접근이 차단될 수 있다.
```

---

## Q. SELinux File Context를 영구적으로 변경하는 방법은?

```bash
semanage fcontext ...
restorecon ...
```

---

## Q. Apache가 새로운 Port에 Bind하지 못하고 AVC에서 name_bind가 나오면?

```text
해당 Port의 SELinux Port Type을 확인한다.
```

```bash
semanage port -l
```

필요한 경우 Apache가 사용할 수 있는 `http_port_t`로 관리한다.

---

## Q. SELinux 문제면 setenforce 0 하면 되지 않는가?

```text
Permissive 전환은 원인 진단에는 사용할 수 있지만
운영 환경의 최종 해결책으로 SELinux를 비활성화하는 것은
보안 기능을 제거하는 것이므로 적절하지 않다.

AVC Log를 분석하여 필요한 정책만 수정해야 한다.
```

---

# 69. 최종 핵심 정리

```text
Firewalld
→ 외부 Network 접근을 제어

SELinux
→ Process가 Resource를 사용하는 방법을 제어
```

Firewalld:

```text
Client
   ↓
TCP/UDP Port
   ↓
허용 / 차단
```

SELinux:

```text
Process Domain
   ↓
Policy
   ↓
File Type / Port Type / Boolean
   ↓
허용 / 차단
```

Server 장애 분석 시 가장 중요한 것은 보안 기능을 먼저 끄는 것이 아니라 **어느 계층에서 차단되는지 단계적으로 확인하는 것**이다.

```text
systemctl
→ Service 상태

ss
→ Port LISTEN

curl
→ 실제 Application 응답

firewall-cmd
→ Network 접근

ls -Z
→ SELinux File Context

semanage port
→ SELinux Port Mapping

getsebool
→ SELinux Boolean

ausearch
→ SELinux AVC Denial
```

최종 원칙:

```text
문제 발생
→ 보안 기능 Disable

이 아니라

문제 발생
→ 증거 확인
→ 원인 분리
→ 필요한 정책만 수정
→ 재검증
```
