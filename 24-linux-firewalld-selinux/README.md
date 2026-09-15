# Firewalld + SELinux Troubleshooting

Rocky Linux 환경에서 Firewalld와 SELinux로 인해 발생할 수 있는 Web Service 장애를 직접 재현하고 원인을 분석한 뒤 정상적으로 해결하는 실습을 진행하였다.

단순히 보안 기능을 비활성화하는 방식이 아니라, 실제 Server 운영 환경에서 사용할 수 있도록 원인을 확인하고 필요한 정책만 수정하는 것을 목표로 하였다.

---

## 1. 실습 환경

| 구분 | 내용 |
|---|---|
| OS | Rocky Linux |
| Server | Server-A |
| Server IP | 192.168.111.100 |
| Client | Client-L |
| Client IP | 192.168.111.150 |
| Web Server | Apache HTTP Server |
| Firewall | firewalld |
| SELinux | Enforcing |

---

## 2. 초기 상태 확인

### Firewalld

```bash
systemctl is-active firewalld
systemctl is-enabled firewalld

firewall-cmd --get-active-zones
firewall-cmd --list-all
```

확인 결과:

```text
active
enabled

public
  interfaces: ens160
```

`public` Zone에 다음 Service들이 허용되어 있었다.

```text
cockpit dhcp dhcpv6-client dns http https mountd nfs ntp rpc-bind samba ssh
```

---

### SELinux

```bash
getenforce
sestatus | head -n 10
```

결과:

```text
Enforcing
```

SELinux가 활성화되어 있으며 Enforcing Mode로 동작 중이었다.

---

### Apache

```bash
systemctl is-active httpd
```

결과:

```text
active
```

기존 Apache Web Service는 정상 동작 중이었다.

---

### 기본 Web Content SELinux Context

```bash
ls -Zd /var/www/html
ls -Z /var/www/html/index.html
```

결과:

```text
system_u:object_r:httpd_sys_content_t:s0 /var/www/html
unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/html/index.html
```

Apache가 제공하는 일반적인 정적 Web Content에는 `httpd_sys_content_t` Type이 적용되어 있음을 확인하였다.

---

# 3. Firewalld 장애 재현

기존 Apache의 80/443 Port를 변경하지 않고 별도의 TCP 8081 Port를 사용하여 Firewall 장애를 재현하였다.

---

## 테스트 Web Server 실행

```bash
mkdir -p /tmp/firewall-web
printf 'Firewalld Test OK\n' > /tmp/firewall-web/index.html

cd /tmp/firewall-web
python3 -m http.server 8081 --bind 0.0.0.0 &
FW_PID=$!
```

Process 확인:

```bash
ps -p "$FW_PID" -o pid,ppid,user,stat,cmd
```

실제 확인 결과:

```text
PID    PPID USER     STAT CMD
4725   3333 root     S    python3 -m http.server 8081 --bind 0.0.0.0
```

Port 확인:

```bash
ss -lntp | grep ':8081'
```

결과:

```text
LISTEN 0 5 0.0.0.0:8081 0.0.0.0:* users:(("python3",pid=4725,fd=3))
```

---

## Server 내부 접속 확인

```bash
curl http://127.0.0.1:8081
```

결과:

```text
Firewalld Test OK
```

Application과 Port Listen 자체에는 문제가 없음을 확인하였다.

---

## Firewalld Port 확인

```bash
firewall-cmd --query-port=8081/tcp
```

결과:

```text
no
```

TCP 8081 Port는 Firewall에서 허용되지 않은 상태였다.

---

## Client-L 접속 테스트

Client-L:

```bash
curl --connect-timeout 3 http://192.168.111.100:8081
```

결과:

```text
curl: (7) Failed to connect to 192.168.111.100 port 8081: 호스트로 갈 루트가 없음
```

Server 내부에서는 접속이 가능하지만 외부 Client에서는 접속할 수 없었다.

---

# 4. Firewalld Runtime 설정

8081 Port를 Runtime 설정에 추가하였다.

```bash
firewall-cmd --add-port=8081/tcp
```

Runtime과 Permanent 설정을 비교하였다.

```bash
firewall-cmd --query-port=8081/tcp
firewall-cmd --permanent --query-port=8081/tcp
```

결과:

```text
yes
no
```

즉 현재 실행 중인 Firewall에는 8081이 허용되었지만 Permanent 설정에는 저장되지 않은 상태였다.

Client-L에서 다시 접속하자 Web Page가 정상적으로 반환되었다.

```text
Firewalld Test OK
```

---

## Runtime 설정 Reload 테스트

```bash
firewall-cmd --reload
```

다시 확인:

```bash
firewall-cmd --query-port=8081/tcp
```

결과:

```text
no
```

Client-L에서도 다시 접속에 실패하였다.

```text
curl: (7) Failed to connect to 192.168.111.100 port 8081: 호스트로 갈 루트가 없음
```

이를 통해 Runtime에만 추가한 설정은 Reload 후 유지되지 않는 것을 확인하였다.

---

# 5. Firewalld Permanent 설정

Permanent 설정에 8081 Port를 추가하였다.

```bash
firewall-cmd --permanent --add-port=8081/tcp
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --query-port=8081/tcp
firewall-cmd --permanent --query-port=8081/tcp
```

결과:

```text
yes
yes
```

Runtime과 Permanent 설정 모두 8081 Port를 허용한 상태가 되었다.

---

## Firewalld 테스트 설정 제거

실습 후 추가했던 Port를 제거하였다.

```bash
firewall-cmd --permanent --remove-port=8081/tcp
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --query-port=8081/tcp
firewall-cmd --permanent --query-port=8081/tcp
```

결과:

```text
no
no
```

임시 Python Web Server도 종료하였다.

```bash
kill -15 "$FW_PID"
```

---

# 6. SELinux File Context 장애 재현

이번에는 Linux Permission은 정상이나 SELinux File Context가 잘못되어 Apache가 Web File을 읽지 못하는 장애를 재현하였다.

---

## 테스트 Directory 생성

```bash
mkdir -p /srv/selinux-web
printf 'SELinux Test OK\n' > /srv/selinux-web/index.html

chmod 755 /srv/selinux-web
chmod 644 /srv/selinux-web/index.html
```

SELinux Context 확인:

```bash
ls -Zd /srv/selinux-web
ls -Z /srv/selinux-web/index.html
```

결과:

```text
unconfined_u:object_r:var_t:s0 /srv/selinux-web
unconfined_u:object_r:var_t:s0 /srv/selinux-web/index.html
```

Apache의 일반적인 Web Content Type인 `httpd_sys_content_t`가 아닌 `var_t`가 적용되어 있었다.

---

## Apache Alias 설정

```apache
Alias /selinux-test/ "/srv/selinux-web/"

<Directory "/srv/selinux-web">
    Require all granted
</Directory>
```

Apache에서 해당 Directory에 접근하도록 설정하였다.

---

## HTTP 접속 테스트

```bash
curl -i http://127.0.0.1/selinux-test/
```

결과:

```text
HTTP/1.1 403 Forbidden
```

Linux Permission을 정상적으로 설정했음에도 Apache가 File을 읽지 못하였다.

---

# 7. AVC Log를 이용한 SELinux 장애 분석

SELinux Audit Log를 확인하였다.

```bash
ausearch -m AVC -ts recent | tail -n 15
```

핵심 Log:

```text
avc: denied { getattr }
comm="httpd"
path="/srv/selinux-web/index.html"
scontext=system_u:system_r:httpd_t:s0
tcontext=unconfined_u:object_r:var_t:s0
tclass=file
permissive=0
```

Apache Process는:

```text
httpd_t
```

Type으로 실행되고 있었고 대상 File은:

```text
var_t
```

Type이었다.

SELinux 정책에 의해 `httpd_t` Process가 해당 File에 접근하지 못해 HTTP 403이 발생한 것을 확인하였다.

---

# 8. SELinux File Context 수정

Apache가 읽을 수 있도록 `/srv/selinux-web`의 SELinux File Context를 `httpd_sys_content_t`로 설정하였다.

```bash
semanage fcontext -a -t httpd_sys_content_t "/srv/selinux-web(/.*)?"
```

실습 환경에서는 해당 경로에 대한 File Context 정의가 이미 존재하여 다음 메시지가 출력되었다.

```text
File context for /srv/selinux-web(/.*)? already defined, modifying instead
```

정책 확인:

```bash
semanage fcontext -l | grep '/srv/selinux-web'
```

결과:

```text
/srv/selinux-web(/.*)? all files system_u:object_r:httpd_sys_content_t:s0
```

실제 File에 Context 적용:

```bash
restorecon -Rv /srv/selinux-web
```

확인:

```bash
ls -Zd /srv/selinux-web
ls -Z /srv/selinux-web/index.html
```

결과:

```text
unconfined_u:object_r:httpd_sys_content_t:s0 /srv/selinux-web
unconfined_u:object_r:httpd_sys_content_t:s0 /srv/selinux-web/index.html
```

---

## HTTP 재검증

```bash
curl -i http://127.0.0.1/selinux-test/
```

결과:

```text
HTTP/1.1 200 OK
```

본문:

```text
SELinux Test OK
```

SELinux File Context 수정 후 Apache가 정상적으로 Web Content를 제공하였다.

---

# 9. SELinux Boolean 확인

Apache 관련 SELinux Boolean을 확인하였다.

```bash
getsebool httpd_can_network_connect
getsebool httpd_can_network_connect_db
getsebool httpd_enable_homedirs
```

결과:

```text
httpd_can_network_connect --> off
httpd_can_network_connect_db --> off
httpd_enable_homedirs --> off
```

Boolean은 SELinux 정책에서 특정 기능의 허용 여부를 제어하는 데 사용된다.

이번 실습에서는 값을 변경하지 않고 현재 상태만 확인하였다.

---

# 10. SELinux HTTP Port Type 확인

Apache가 사용할 수 있는 SELinux Port Type을 확인하였다.

```bash
semanage port -l | grep '^http_port_t'
```

초기 결과:

```text
http_port_t tcp 80, 81, 443, 488, 8008, 8009, 8443, 9000
```

현재 Apache는 실제로 80과 443 Port를 사용하고 있었다.

```bash
ss -lntp | grep httpd
```

```text
*:443
*:80
```

---

# 11. SELinux Port Label 장애 재현

Apache가 TCP 8082 Port에서도 Listen하도록 테스트 설정을 추가하였다.

```apache
Listen 8082
```

Apache 문법 검사:

```bash
httpd -t
```

결과:

```text
Syntax OK
```

그러나 Apache Reload 시 다음 상태가 확인되었다.

```text
httpd.service is not active, cannot reload.
```

SELinux AVC Log를 확인하였다.

```bash
ausearch -m AVC -ts recent | grep -E 'httpd|8082|name_bind' | tail -n 10
```

핵심 Log:

```text
avc: denied { name_bind }
comm="httpd"
src=8082
scontext=system_u:system_r:httpd_t:s0
tcontext=system_u:object_r:us_cli_port_t:s0
tclass=tcp_socket
permissive=0
```

TCP 8082는 기존 SELinux 정책에서 `us_cli_port_t` Type으로 정의되어 있었기 때문에 `httpd_t`가 해당 Port에 Bind하는 것을 SELinux가 차단하였다.

---

# 12. SELinux HTTP Port Type 수정

기존에 존재하는 8082 Port의 Type을 Apache용 `http_port_t`로 변경하였다.

```bash
semanage port -m -t http_port_t -p tcp 8082
```

Apache 시작:

```bash
systemctl start httpd
systemctl is-active httpd
```

결과:

```text
active
```

8082 Listen 확인:

```bash
ss -lntp | grep ':8082'
```

결과:

```text
LISTEN 0 511 *:8082 *:* users:(("httpd",...))
```

Server 내부에서 HTTP 요청:

```bash
curl -I http://127.0.0.1:8082/
```

결과:

```text
HTTP/1.1 200 OK
```

SELinux Port Type 변경 후 Apache가 정상적으로 TCP 8082에 Bind할 수 있음을 확인하였다.

---

# 13. SELinux와 Firewalld 차이 확인

SELinux에서는 8082 사용이 허용되었지만 Firewalld에는 아직 Port가 열려 있지 않았다.

```bash
firewall-cmd --query-port=8082/tcp
```

결과:

```text
no
```

SELinux Local Customization 확인:

```bash
semanage port -l -C | grep '8082'
```

결과:

```text
http_port_t tcp 8082
```

Firewalld Runtime에 8082를 허용하였다.

```bash
firewall-cmd --add-port=8082/tcp
```

확인:

```bash
firewall-cmd --query-port=8082/tcp
```

결과:

```text
yes
```

Client-L에서 접속하였다.

```bash
curl --connect-timeout 3 http://192.168.111.100:8082/
```

Apache Web Page가 정상적으로 반환되었다.

---

# 14. Firewalld와 SELinux 역할 비교

이번 실습을 통해 Firewalld와 SELinux의 역할 차이를 확인하였다.

```text
Firewalld
외부 Client의 Network Packet이
Server의 특정 Port까지 들어올 수 있는지 제어

SELinux Port Type
httpd와 같은 Process가
특정 Port에 Bind할 수 있는지 제어

SELinux File Context
httpd와 같은 Process가
특정 File 또는 Directory에 접근할 수 있는지 제어
```

따라서 Web Service 장애 발생 시 단순히 Firewall만 확인하는 것이 아니라 다음 항목을 함께 확인해야 한다.

```text
Application Process
        ↓
Listen Port
        ↓
Firewalld
        ↓
Linux Permission
        ↓
SELinux File Context
        ↓
SELinux Port Type / Boolean
        ↓
Audit Log
```

---

# 15. 실습 환경 원상복구

8082 Apache 테스트 설정을 제거하였다.

```bash
rm -f /etc/httpd/conf.d/selinux-port-test.conf

httpd -t
systemctl restart httpd
```

SELinux Local Port Customization을 제거하였다.

```bash
semanage port -d -t http_port_t -p tcp 8082
```

Firewalld Runtime 설정도 제거하였다.

```bash
firewall-cmd --remove-port=8082/tcp
```

SELinux File Context 테스트 설정과 Directory도 제거하였다.

```bash
rm -f /etc/httpd/conf.d/selinux-test.conf
rm -rf /srv/selinux-web

semanage fcontext -d "/srv/selinux-web(/.*)?"
```

Apache를 다시 시작하였다.

```bash
httpd -t
systemctl restart httpd
```

---

## 최종 상태 확인

```bash
systemctl is-active httpd
ss -lntp | grep httpd
```

결과:

```text
active
```

Apache는 기존 Port만 정상적으로 Listen하였다.

```text
*:443
*:80
```

테스트 Directory와 Apache 설정 File도 제거된 것을 확인하였다.

```text
ls: cannot access '/srv/selinux-web': 그런 파일이나 디렉터리가 없습니다
ls: cannot access '/etc/httpd/conf.d/selinux-test.conf': 그런 파일이나 디렉터리가 없습니다
```

---

# 16. Troubleshooting 정리

## Firewalld 장애

```text
8081 Process 실행 정상
        ↓
0.0.0.0:8081 LISTEN 정상
        ↓
localhost 접속 성공
        ↓
Client-L 접속 실패
        ↓
firewall-cmd 확인
        ↓
8081/tcp 미허용
        ↓
Firewall Port 허용
        ↓
Client-L 접속 성공
```

---

## SELinux File Context 장애

```text
Apache 정상
        ↓
Linux Permission 정상
        ↓
HTTP 403 Forbidden
        ↓
ausearch AVC 확인
        ↓
httpd_t → var_t 접근 거부
        ↓
semanage fcontext
        ↓
restorecon
        ↓
httpd_sys_content_t
        ↓
HTTP 200 OK
```

---

## SELinux Port 장애

```text
Apache Listen 8082 설정
        ↓
httpd -t
Syntax OK
        ↓
Apache 8082 Bind 실패
        ↓
AVC 확인
        ↓
denied { name_bind }
        ↓
8082 = us_cli_port_t
        ↓
semanage port
        ↓
8082 = http_port_t
        ↓
Apache 정상 기동
        ↓
8082 LISTEN
        ↓
HTTP 200 OK
```

---

# 17. 주요 명령어

### Firewalld

```bash
firewall-cmd --list-all
firewall-cmd --query-port=8081/tcp

firewall-cmd --add-port=8081/tcp
firewall-cmd --remove-port=8081/tcp

firewall-cmd --permanent --add-port=8081/tcp
firewall-cmd --permanent --remove-port=8081/tcp

firewall-cmd --reload
```

### SELinux 상태

```bash
getenforce
sestatus
```

### SELinux File Context

```bash
ls -Z
ls -Zd

semanage fcontext -l
semanage fcontext -l -C

restorecon -Rv <directory>
```

### SELinux Port

```bash
semanage port -l
semanage port -l -C
```

### SELinux Boolean

```bash
getsebool <boolean>
```

### SELinux Audit Log

```bash
ausearch -m AVC -ts recent
```

---

# 18. 실습을 통해 확인한 내용

- Firewalld의 Runtime 설정과 Permanent 설정의 차이 확인
- `firewall-cmd --reload` 동작 확인
- Application 정상 여부와 Firewall 장애 구분
- SELinux Enforcing Mode에서 실제 Access Denial 재현
- `httpd_t`와 `httpd_sys_content_t` 관계 확인
- 잘못된 File Context로 인한 HTTP 403 장애 분석
- AVC Audit Log를 이용한 SELinux 원인 분석
- `semanage fcontext`와 `restorecon`을 이용한 File Context 관리
- SELinux `name_bind` 거부 장애 재현
- `semanage port`를 이용한 Apache Port Type 관리
- Firewalld와 SELinux Port 정책의 역할 차이 확인
- 실습 후 변경 사항 원상복구

---

# 19. 핵심 정리

Server에서 특정 Service에 접속할 수 없다고 해서 바로 Firewall 문제라고 판단해서는 안 된다.

먼저 Process와 Listen Port를 확인한 뒤 Network 접근과 SELinux 정책을 단계적으로 확인해야 한다.

```text
1. systemctl
   → Service 실행 상태 확인

2. ss
   → 실제 Listen Port 확인

3. curl
   → Local Application 동작 확인

4. firewall-cmd
   → 외부 Network 접근 허용 여부 확인

5. ls -Z
   → SELinux File Context 확인

6. semanage port
   → SELinux Port Type 확인

7. ausearch
   → 실제 SELinux Denial Log 확인
```

SELinux 장애가 발생했을 때 단순히 다음과 같이 처리하는 것은 올바른 해결 방법이 아니다.

```bash
setenforce 0
```

또는 무조건적인 Permission 변경:

```bash
chmod 777
```

대신 실제 AVC Log와 SELinux Context를 확인하여 필요한 정책만 정확하게 수정하는 것이 중요하다.
