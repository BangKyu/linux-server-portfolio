# Chrony / NTP Server-Client 구축

Rocky Linux 환경에서 Chrony를 이용하여 내부 NTP Server / Client 환경을 구축하였다.

Server-A는 외부 NTP Server와 시간을 동기화하면서 내부 Network의 NTP Server 역할을 수행하도록 구성하고, Client-L은 외부 NTP Server를 직접 사용하지 않고 Server-A를 시간 동기화 Source로 사용하도록 구성하였다.

```text
외부 NTP Server
        |
        | NTP
        v
Server-A
192.168.111.100
Stratum 3
        |
        | UDP 123
        | NTP
        v
Client-L
192.168.111.150
Stratum 4
```

---

# 1. 실습 환경

| 구분 | Server-A | Client-L |
|---|---|---|
| 역할 | NTP Server | NTP Client |
| IP | 192.168.111.100 | 192.168.111.150 |
| OS | Rocky Linux | Rocky Linux |
| Package | chrony | chrony |
| Service | chronyd | chronyd |
| Time Zone | Asia/Seoul | Asia/Seoul |
| NTP Port | UDP 123 | - |

최종 구조:

```text
Server-A
→ 외부 NTP Source와 동기화
→ 내부 Client에게 시간 제공

Client-L
→ Server-A를 NTP Source로 사용
```

---

# 사전 확인

## 2. Chrony Package 확인

Server-A:

```bash
rpm -q chrony
```

실제 결과:

```text
chrony-4.8-1.el9.x86_64
```

Client-L:

```bash
rpm -q chrony
```

실제 결과:

```text
chrony-4.8-1.el9.x86_64
```

Server와 Client 모두 Chrony가 설치되어 있었다.

---

## 3. chronyd Service 상태 확인

Server-A:

```bash
systemctl is-active chronyd
systemctl is-enabled chronyd
```

실제 결과:

```text
active
enabled
```

Client-L 역시:

```text
active
enabled
```

상태였다.

---

# 기존 시간 동기화 상태

## 4. Server-A timedatectl 확인

```bash
timedatectl
```

실제 결과:

```text
Local time: 월 2026-09-14 17:00:33 KST
Universal time: 월 2026-09-14 08:00:33 UTC
RTC time: 월 2026-09-14 08:00:34
Time zone: Asia/Seoul (KST, +0900)
System clock synchronized: yes
NTP service: active
RTC in local TZ: no
```

기존부터 Server-A는 외부 NTP Source와 정상적으로 동기화되어 있었다.

---

## 5. Server-A Tracking 상태

```bash
chronyc tracking
```

실제 주요 결과:

```text
Reference ID    : AFC3A7C2 (mail.innotab.com)
Stratum         : 3
System time     : 0.002179991 seconds fast of NTP time
Leap status     : Normal
```

Server-A는:

```text
Stratum 3
```

상태로 외부 NTP Source와 정상 동기화 중이었다.

---

## 6. Server-A NTP Source 확인

```bash
chronyc sources -v
```

실제 주요 결과:

```text
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^- any.time.nl                   2  10   377   43m  +1767us[+3186us] +/-   22ms
^+ 193.123.243.2                 4  10   377   506   +601us[+2898us] +/- 3561us
^* mail.innotab.com              2  10   377    27  +3065us[+5459us] +/-   11ms
^- 115.15.44.33                  3  10   377   839   -793us[+1437us] +/-  103ms
```

`*`가 붙은:

```text
mail.innotab.com
```

이 현재 Server-A가 실제 시간 동기화 기준으로 선택한 Source였다.

---

# 기존 Client-L 상태

## 7. Client-L 초기 NTP Source

Client-L도 초기에는 외부 NTP Server와 직접 동기화하고 있었다.

```bash
chronyc sources -v
```

실제 결과:

```text
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 211.108.117.211               2   6   377    57   +613us[ +736us] +/- 5683us
^- 211.222.238.30                2   6   377    58  -1460us[-1338us] +/-   55ms
^- 158.247.202.103               2   6   377    62  -2057us[-1936us] +/-   21ms
^- 3.39.176.65                   2   6   372   189  +1516us[+1490us] +/- 8326us
```

초기 구조:

```text
외부 NTP
   |
   +----> Server-A

외부 NTP
   |
   +----> Client-L
```

이를 다음 구조로 변경하였다.

```text
외부 NTP
   |
   v
Server-A
   |
   v
Client-L
```

---

# NTP Server 구성

## 8. Server-A chrony.conf 백업

```bash
cp -a /etc/chrony.conf /etc/chrony.conf.before-ntp-server
```

기존 설정 파일을 백업한 후 Server 설정을 변경하였다.

---

## 9. Client Network 접근 허용

Server-A:

```bash
vi /etc/chrony.conf
```

다음 설정을 추가하였다.

```conf
allow 192.168.111.0/24
```

의미:

```text
192.168.111.0/24 Network의 Client가
Server-A의 chronyd에 NTP 요청 가능
```

즉 Server-A가 단순 NTP Client 역할뿐만 아니라 내부 NTP Server 역할도 수행하도록 구성하였다.

구조:

```text
외부 NTP Source
        ↓
Server-A chronyd
        ↓
192.168.111.0/24 Client에게 시간 제공
```

---

## 10. Chrony Service 재시작

```bash
systemctl restart chronyd
```

확인:

```bash
systemctl is-active chronyd
```

Chrony Service가 정상 동작하는 것을 확인하였다.

---

# Firewall

## 11. NTP Service 허용

Server-A:

```bash
firewall-cmd --permanent --add-service=ntp
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-services
```

실제 결과:

```text
cockpit dhcp dhcpv6-client dns http https mountd nfs ntp rpc-bind samba ssh
```

Firewall에:

```text
ntp
```

Service가 정상적으로 추가되었다.

---

# UDP 123

## 12. NTP Port 확인

Server-A:

```bash
ss -lunp | grep ':123'
```

실제 결과:

```text
UNCONN 0 0 0.0.0.0:123 0.0.0.0:* users:(("chronyd",pid=8827,fd=7))
```

NTP의 기본 Port:

```text
UDP 123
```

에서 `chronyd`가 요청을 받을 수 있는 상태임을 확인하였다.

---

# NTP Client 구성

## 13. Client-L 설정 백업

Client-L:

```bash
cp -a /etc/chrony.conf /etc/chrony.conf.before-server-a
```

---

## 14. Client-L NTP Source 변경

Client-L의 기존 외부 NTP Source 사용을 중지하고 Server-A를 직접 NTP Server로 사용하도록 변경하였다.

```bash
vi /etc/chrony.conf
```

기존 Source:

```conf
pool 2.rocky.pool.ntp.org iburst
sourcedir /run/chrony-dhcp
```

를 주석 처리하고:

```conf
#pool 2.rocky.pool.ntp.org iburst
#sourcedir /run/chrony-dhcp
```

Server-A를 추가하였다.

```conf
server 192.168.111.100 iburst
```

이번 실습에서는 Client-L의 시간 Source를 명확하게:

```text
Server-A
192.168.111.100
```

하나로 구성하였다.

---

## 15. Client Chrony 재시작

```bash
systemctl restart chronyd
```

초기 NTP Sample을 빠르게 수집하기 위해:

```bash
chronyc burst 4/4
```

을 실행하였다.

---

# Client NTP Source 검증

## 16. chronyc sources 확인

Client-L:

```bash
chronyc sources -v
```

실제 결과:

```text
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* www.bangkyu.com               3   6    37    56  +7851ns[ -138us] +/- 6106us
```

핵심:

```text
^*
```

의 의미:

```text
^
→ NTP Server Source

*
→ 현재 실제 시간 동기화 Source로 선택
```

따라서 Client-L이 Server-A를 실제 시간 동기화 기준으로 사용하고 있음을 확인하였다.

---

# 왜 IP가 아니라 www.bangkyu.com으로 보이는가?

## 17. DNS Name 표시

Client-L에는:

```conf
server 192.168.111.100 iburst
```

로 설정하였지만 `chronyc sources -v`에서는:

```text
www.bangkyu.com
```

으로 표시되었다.

현재 DNS에서:

```text
www.bangkyu.com
→ 192.168.111.100
```

으로 연결되어 있기 때문에 Chrony가 IP를 Hostname 형태로 표시한 것이다.

즉:

```text
www.bangkyu.com
=
192.168.111.100
=
Server-A
```

이다.

---

# Tracking 검증

## 18. Client-L chronyc tracking

```bash
chronyc tracking
```

실제 결과:

```text
Reference ID    : C0A86F64 (www.bangkyu.com)
Stratum         : 4
Ref time (UTC)  : Mon Sep 14 08:10:18 2026
System time     : 0.000039654 seconds slow of NTP time
Last offset     : -0.000145764 seconds
RMS offset      : 0.000145764 seconds
Frequency       : 2.980 ppm fast
Residual freq   : -3.692 ppm
Skew            : 8.961 ppm
Root delay      : 0.009158012 seconds
Root dispersion : 0.002478016 seconds
Update interval : 64.1 seconds
Leap status     : Normal
```

---

## 19. Reference ID 확인

Client-L:

```text
Reference ID : C0A86F64
```

16진수 값을 IPv4로 변환하면:

```text
C0 → 192
A8 → 168
6F → 111
64 → 100
```

따라서:

```text
C0A86F64
→ 192.168.111.100
→ Server-A
```

이다.

Client-L의 실제 시간 Source가 Server-A임을 확인하였다.

---

# Stratum

## 20. Server-A와 Client-L Stratum

Server-A:

```text
Stratum 3
```

Client-L:

```text
Stratum 4
```

구조:

```text
외부 NTP Source
Stratum 2
        ↓
Server-A
Stratum 3
        ↓
Client-L
Stratum 4
```

Client가 상위 NTP Server로부터 시간을 전달받으면 Stratum이 한 단계 증가하는 것을 확인하였다.

---

# Server에서 Client 확인

## 21. chronyc clients

Server-A:

```bash
chronyc clients
```

실제 결과:

```text
Hostname                      NTP   Drop Int IntL Last     Cmd   Drop Int  Last
===============================================================================
192.168.111.150                 5      0   4   -    18       0      0   -     -
```

Server-A에서:

```text
192.168.111.150
```

즉 Client-L의 실제 NTP 요청을 확인하였다.

주요 결과:

```text
Client IP
→ 192.168.111.150

NTP Request
→ 5

Drop
→ 0

Last
→ 약 18초 전 마지막 요청
```

이를 통해 Client-L이 실제로 Server-A의 NTP Service를 사용하고 있음을 Server 측에서도 검증하였다.

---

# Client 시간 동기화 확인

## 22. timedatectl

Client-L:

```bash
timedatectl
```

실제 결과:

```text
Local time: 월 2026-09-14 17:12:34 KST
Universal time: 월 2026-09-14 08:12:34 UTC
RTC time: 월 2026-09-14 08:12:34
Time zone: Asia/Seoul (KST, +0900)
System clock synchronized: yes
NTP service: active
RTC in local TZ: no
```

핵심:

```text
System clock synchronized: yes
NTP service: active
```

Client-L의 System Clock이 NTP를 통해 정상적으로 동기화되고 있음을 확인하였다.

---

# 전체 동작 구조

## 23. 최종 구성

```text
                    External NTP Servers
                             |
                             | NTP
                             v
                         Server-A
                      192.168.111.100
                             |
                       chronyd Client
                             |
                         Stratum 3
                             |
                -------------------------
                |                       |
                | allow                 |
                | 192.168.111.0/24      |
                |                       |
                -------------------------
                             |
                          UDP 123
                             |
                             | NTP
                             v
                         Client-L
                      192.168.111.150
                             |
                      chronyd Client
                             |
                         Stratum 4
                             |
                             v
                  System Clock Synchronized
```

---

# 시간 동기화 흐름

## 24. Server-A

```text
외부 NTP Server
        ↓
chronyd
        ↓
Server-A System Clock
        ↓
Stratum 3
```

---

## 25. Client-L

```text
Server-A
192.168.111.100
        ↓
UDP 123
        ↓
chronyd
        ↓
Client-L System Clock
        ↓
Stratum 4
```

---

# 주요 설정

## 26. Server-A /etc/chrony.conf

핵심 추가 설정:

```conf
allow 192.168.111.0/24
```

의미:

```text
해당 Network의 Client에게
NTP Service 제공 허용
```

---

## 27. Client-L /etc/chrony.conf

핵심 설정:

```conf
server 192.168.111.100 iburst

#pool 2.rocky.pool.ntp.org iburst
#sourcedir /run/chrony-dhcp
```

Client-L이 외부 NTP Server 대신 Server-A만 사용하도록 구성하였다.

---

# 주요 명령어

## 28. Chrony Package 확인

```bash
rpm -q chrony
```

---

## 29. Chrony Service 확인

```bash
systemctl is-active chronyd
systemctl is-enabled chronyd
```

---

## 30. 현재 시간 상태

```bash
timedatectl
```

---

## 31. NTP Source 확인

```bash
chronyc sources -v
```

현재 사용 가능한 NTP Source와 실제 선택된 Source를 확인한다.

---

## 32. 시간 동기화 상세 확인

```bash
chronyc tracking
```

확인 가능:

```text
Reference ID
Stratum
System time
Offset
Root delay
Leap status
```

---

## 33. Client 접속 확인

NTP Server:

```bash
chronyc clients
```

Server에 실제 NTP 요청을 보내는 Client를 확인할 수 있다.

---

## 34. NTP Sample 요청

```bash
chronyc burst 4/4
```

초기 NTP Source 확인 과정에서 Sample을 빠르게 수집하기 위해 사용하였다.

---

## 35. NTP Port 확인

```bash
ss -lunp | grep ':123'
```

NTP 기본 Port:

```text
UDP 123
```

확인.

---

## 36. Firewall 확인

```bash
firewall-cmd --list-services
```

확인 항목:

```text
ntp
```

---

# chronyc sources 기호

## 37. Source Mode

```text
^
→ NTP Server

=
→ Peer

#
→ Local Clock
```

---

## 38. Source State

이번 실습에서 가장 중요한 상태:

```text
*
→ 현재 실제 동기화 Source
```

예:

```text
^* www.bangkyu.com
```

의미:

```text
^
→ NTP Server

*
→ 현재 선택된 기준 Source
```

즉 Client-L이 Server-A를 실제 시간 Source로 사용하고 있다는 의미이다.

---

# 시간 관련 용어

## 39. Local Time

```text
Local time
```

현재 System의 Time Zone을 적용한 시간.

이번 환경:

```text
Asia/Seoul
KST
UTC+9
```

---

## 40. Universal Time

```text
Universal time
```

UTC 기준 시간.

이번 결과:

```text
Local Time
17:12

UTC
08:12
```

로 약 9시간 차이가 있으며 이는:

```text
KST = UTC + 9
```

이기 때문이다.

---

## 41. RTC

```text
RTC
```

Real-Time Clock.

Mainboard의 Hardware Clock을 의미한다.

이번 환경:

```text
RTC in local TZ: no
```

이므로 RTC는 UTC 기준으로 관리되고 있다.

---

# Chrony 동작 확인 포인트

## 42. 정상 동기화 판단

Client-L에서 다음 세 가지를 함께 확인하였다.

```text
chronyc sources -v
→ ^* Server-A

chronyc tracking
→ Stratum 4
→ Reference Server-A
→ Leap status Normal

timedatectl
→ System clock synchronized: yes
→ NTP service: active
```

---

# Server / Client 양방향 검증

## 43. Client 관점

Client-L:

```text
^* www.bangkyu.com
```

즉 Server-A를 현재 NTP Source로 선택하였다.

---

## 44. Server 관점

Server-A:

```text
192.168.111.150
```

가 `chronyc clients`에 나타났다.

즉 Server-A가 Client-L의 NTP 요청을 실제로 받고 있다.

---

## 45. End-to-End 검증

```text
Client-L
        |
        | NTP Request
        v
Server-A UDP 123
        |
        | NTP Response
        v
Client-L
        |
        v
System Clock Synchronized
```

Client와 Server 양쪽에서 NTP 통신을 확인하였다.

---

# 실습 결과

```text
Chrony
✓ chrony Package 설치 확인
✓ chronyd active
✓ chronyd enabled

Server-A
✓ 기존 외부 NTP 동기화 확인
✓ Stratum 3 확인
✓ allow 192.168.111.0/24 설정
✓ NTP Server 역할 구성
✓ Firewall ntp Service 허용
✓ UDP 123 Listen 확인
✓ chronyc clients에서 Client-L 확인

Client-L
✓ 기존 외부 NTP Source 확인
✓ Server-A를 NTP Source로 변경
✓ 외부 Source 직접 사용 중지
✓ chronyd 재시작
✓ Server-A를 실제 Source로 선택
✓ ^* 상태 확인
✓ Reference ID가 Server-A임을 확인
✓ Stratum 4 확인
✓ Leap status Normal
✓ System clock synchronized: yes
✓ NTP service: active
```

---

# 최종 결과

Server-A가 외부 NTP Server를 이용하여 자신의 시간을 동기화하면서 내부 Network의 NTP Server 역할도 수행하도록 구성하였다.

Server-A의 `/etc/chrony.conf`에:

```conf
allow 192.168.111.0/24
```

를 설정하여 내부 Client의 NTP 요청을 허용하고 Firewall에서 NTP Service를 허용하였다.

Server-A의:

```bash
ss -lunp | grep ':123'
```

결과에서 `chronyd`가 UDP 123을 사용하고 있음을 확인하였다.

Client-L은 기존 외부 NTP Source를 직접 사용하지 않고:

```conf
server 192.168.111.100 iburst
```

를 이용하여 Server-A를 NTP Source로 사용하도록 변경하였다.

Client-L의:

```bash
chronyc sources -v
```

결과:

```text
^* www.bangkyu.com
```

을 통해 Server-A가 현재 실제 시간 동기화 Source로 선택된 것을 확인하였다.

또한:

```text
Server-A
Stratum 3

Client-L
Stratum 4
```

로 NTP 계층 구조가 형성된 것을 확인하였다.

Server-A의:

```bash
chronyc clients
```

결과에서도:

```text
192.168.111.150
```

Client-L의 실제 NTP 요청을 확인하였다.

최종적으로 Client-L의:

```text
System clock synchronized: yes
NTP service: active
```

상태까지 확인하여 NTP Server / Client 시간 동기화가 정상적으로 동작하는 것을 검증하였다.

```text
External NTP
        ↓
Server-A
Stratum 3
        ↓
UDP 123
        ↓
Client-L
Stratum 4
        ↓
System Clock Synchronized
```

---

## 관련 이론

Chrony와 NTP의 개념, `chronyd` / `chronyc`, Stratum, `sources`, `tracking`, `clients`, `iburst`, `allow`, UDP 123, System Clock / RTC, Offset 및 시간 동기화 Troubleshooting에 대한 자세한 내용은 `chrony-notes.md`에서 정리한다.
