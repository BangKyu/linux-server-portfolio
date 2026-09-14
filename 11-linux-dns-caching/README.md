# DNS Caching Server 구축

Rocky Linux에서 BIND를 이용하여 DNS Caching Server를 구축하였다.

내부 Client가 Local DNS Server를 사용하도록 설정하고, 동일한 Domain을 반복 조회하여 DNS Cache 적용 전후의 Query Time을 비교하였다.

---

## 1. 실습 환경

| 구분 | 설정 |
|---|---|
| DNS Server | Rocky Linux |
| DNS Server IP | 192.168.111.100 |
| Interface | ens160 |
| DNS Client | Rocky Linux |
| Client IP | 192.168.111.150/24 |
| Network | 192.168.111.0/24 |
| Default Gateway | 192.168.111.2 |
| DNS Port | TCP/UDP 53 |
| DNS Software | BIND |
| DNS Daemon | named |

구성:

```text
                    Internet
                       |
                 192.168.111.2
                 VMware NAT
                       |
        -------------------------------
        |                             |
DNS Caching Server                DNS Client
192.168.111.100                   192.168.111.150
Rocky Linux                       Rocky Linux
named                             DNS → 192.168.111.100
```

---

## 2. 기존 DHCP Server 중지

이전 DHCP Server 실습에서 실행한 `dhcpd`를 중지하고 자동 실행을 해제하였다.

```bash
systemctl disable --now dhcpd
systemctl is-active dhcpd
systemctl is-enabled dhcpd
```

확인 결과:

```text
inactive
disabled
```

VMware VMnet8의 DHCP 기능을 다시 활성화하여 Client가 VMware DHCP를 사용할 수 있도록 구성하였다.

Client의 네트워크 확인:

```bash
ip -br addr show ens160
ip route
nmcli -f GENERAL.DEVICE,IP4.ADDRESS,IP4.GATEWAY,IP4.DNS device show ens160
```

확인 결과:

```text
ens160    UP    192.168.111.150/24
```

```text
default via 192.168.111.2 dev ens160 proto dhcp src 192.168.111.150 metric 100
192.168.111.0/24 dev ens160 proto kernel scope link src 192.168.111.150 metric 100
```

```text
GENERAL.DEVICE:     ens160
IP4.ADDRESS[1]:     192.168.111.150/24
IP4.GATEWAY:        192.168.111.2
IP4.DNS[1]:         192.168.111.2
```

---

## 3. BIND 설치

DNS Caching Server 구축을 위해 BIND Package를 설치하였다.

```bash
dnf install -y bind bind-utils bind-libs
```

Package 확인:

```bash
rpm -q bind
rpm -q bind-utils
rpm -q bind-libs
```

확인 결과:

```text
bind-9.16.23-40.el9_8.8.x86_64
bind-utils-9.16.23-40.el9_8.8.x86_64
bind-libs-9.16.23-40.el9_8.8.x86_64
```

BIND 설치 직후 `named`는 실행되지 않은 상태였다.

```bash
systemctl status named --no-pager
```

```text
○ named.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/usr/lib/systemd/system/named.service; disabled; preset: disabled)
     Active: inactive (dead)
```

---

## 4. BIND 설정 파일 백업

설정 변경 전에 주요 DNS 관련 파일을 백업하였다.

```bash
mkdir -p /backup/dns-cache

cp -a /etc/hosts /backup/dns-cache/
cp -a /etc/resolv.conf /backup/dns-cache/
cp -a /etc/named.conf /backup/dns-cache/
cp -a /etc/named.rfc1912.zones /backup/dns-cache/
cp -a /etc/named.root.key /backup/dns-cache/
cp -a /etc/named /backup/dns-cache/
```

백업 확인:

```bash
ls -l /backup/dns-cache
```

```text
합계 20
-rw-r--r--. 1 root root   158  6월 23  2020 hosts
drwxr-x---. 2 root named    6  8월 13 21:12 named
-rw-r-----. 1 root named 1722  8월 13 21:12 named.conf
-rw-r-----. 1 root named 1029  8월 13 21:08 named.rfc1912.zones
-rw-r--r--. 1 root named  686  8월 13 21:12 named.root.key
-rw-r--r--. 1 root root    55  9월 14 10:56 resolv.conf
```

---

## 5. 기존 BIND 설정 확인

```bash
grep -nE 'listen-on|listen-on-v6|allow-query|recursion|dnssec-validation' /etc/named.conf
```

기존 설정:

```text
listen-on port 53 { 127.0.0.1; };
listen-on-v6 port 53 { ::1; };
allow-query     { localhost; };
recursion yes;
dnssec-validation yes;
```

기본 설정에서는 localhost의 DNS 요청만 처리하도록 구성되어 있었다.

---

## 6. DNS Caching Server 설정

설정 파일 수정:

```bash
vi /etc/named.conf
```

주요 설정:

```conf
options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { none; };

        directory       "/var/named";

        allow-query     { localhost; 192.168.111.0/24; };
        allow-recursion { localhost; 192.168.111.0/24; };

        recursion yes;

        dnssec-validation yes;
};
```

주요 설정 의미:

```text
listen-on port 53 { any; };
→ IPv4 Interface에서 DNS 요청 수신

listen-on-v6 port 53 { none; };
→ IPv6 DNS Listen 비활성화

allow-query
→ localhost 및 192.168.111.0/24의 DNS 질의 허용

allow-recursion
→ 내부 Network의 Recursive DNS 질의 허용

recursion yes
→ 다른 DNS Server에 질의한 결과를 Client에게 반환하고 Cache

dnssec-validation yes
→ DNSSEC 검증 사용
```

내부 Network에서만 Recursive DNS 기능을 사용할 수 있도록 접근 범위를 제한하였다.

---

## 7. 설정 문법 검사

```bash
named-checkconf
```

별도의 오류가 출력되지 않아 `/etc/named.conf`의 문법이 정상임을 확인하였다.

설정 확인:

```bash
grep -nE 'listen-on|listen-on-v6|allow-query|allow-recursion|recursion|dnssec-validation' /etc/named.conf
```

```text
11:     listen-on port 53 { any; };
12:     listen-on-v6 port 53 { none; };
19:     allow-query     { localhost; 192.168.111.0/24; };
20:     allow-recursion { localhost; 192.168.111.0/24; };
31:     recursion yes;
33:     dnssec-validation yes;
```

---

## 8. Firewall DNS 허용

DNS 요청을 받을 수 있도록 Firewall에서 DNS Service를 허용하였다.

```bash
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload
firewall-cmd --list-services
```

확인 결과:

```text
cockpit dhcp dhcpv6-client dns ssh
```

Firewall에서 DNS Service가 정상적으로 허용된 것을 확인하였다.

---

## 9. named Service 실행

```bash
systemctl enable --now named
systemctl status named --no-pager -l
```

확인 결과:

```text
● named.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; preset: disabled)
     Active: active (running) since Mon 2026-09-14 11:18:05 KST
   Main PID: 5000 (named)
     CGroup: /system.slice/named.service
             └─5000 /usr/sbin/named -u named -c /etc/named.conf
```

로그에서도 DNS Zone 로딩과 Service 실행을 확인하였다.

```text
all zones loaded
running
Started Berkeley Internet Name Domain (DNS).
```

---

## 10. DNS Port 확인

DNS Server의 TCP/UDP 53 Port Listen 상태를 확인하였다.

```bash
ss -luntp | grep ':53'
```

확인 결과:

```text
udp   UNCONN 0  0  192.168.111.100:53  0.0.0.0:*  users:(("named",pid=5000,...))
udp   UNCONN 0  0        127.0.0.1:53  0.0.0.0:*  users:(("named",pid=5000,...))

tcp   LISTEN 0 10  192.168.111.100:53  0.0.0.0:*  users:(("named",pid=5000,...))
tcp   LISTEN 0 10        127.0.0.1:53  0.0.0.0:*  users:(("named",pid=5000,...))
```

`named`가 `192.168.111.100`의 TCP/UDP 53 Port에서 DNS 요청을 대기하고 있음을 확인하였다.

---

## 11. DNS Cache 초기화

BIND의 기존 Cache를 비우기 위해 `rndc`를 사용하였다.

```bash
rndc flush
```

`rndc`는 실행 중인 `named` Service를 관리할 수 있는 명령어이며 Service를 재시작하지 않고 Cache를 삭제할 수 있다.

---

## 12. DNS Caching 동작 확인

Local DNS Server를 직접 지정하여 `www.naver.com`을 조회하였다.

첫 번째 조회:

```bash
dig @192.168.111.100 www.naver.com | grep -E 'status:|Query time|SERVER:'
```

결과:

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8202
;; Query time: 2128 msec
;; SERVER: 192.168.111.100#53(192.168.111.100)
```

동일한 Domain을 다시 조회하였다.

```bash
dig @192.168.111.100 www.naver.com | grep -E 'status:|Query time|SERVER:'
```

결과:

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41174
;; Query time: 2 msec
;; SERVER: 192.168.111.100#53(192.168.111.100)
```

Query Time 비교:

```text
첫 번째 조회 : 2128 msec
두 번째 조회 :    2 msec
```

첫 번째 요청에서는 외부 DNS 조회가 필요했지만, 두 번째 요청에서는 이전 조회 결과가 Cache에 저장되어 빠르게 응답하는 것을 확인하였다.

두 요청 모두 다음 DNS Server를 사용하였다.

```text
192.168.111.100#53
```

이를 통해 Local DNS Caching Server가 정상적으로 동작하고 있음을 확인하였다.

---

## 13. Client DNS Server 변경

기존 Client의 DNS Server는 VMware NAT DNS인 `192.168.111.2`였다.

```text
기존 DNS
192.168.111.2
```

Client가 Local DNS Caching Server를 사용하도록 변경하였다.

```bash
sudo nmcli connection modify ens160 ipv4.ignore-auto-dns yes
sudo nmcli connection modify ens160 ipv4.dns "192.168.111.100"

sudo nmcli connection down ens160
sudo nmcli connection up ens160
```

DNS 설정 확인:

```bash
nmcli -f GENERAL.DEVICE,IP4.ADDRESS,IP4.GATEWAY,IP4.DNS device show ens160
cat /etc/resolv.conf
```

변경 후 DNS Server:

```text
192.168.111.100
```

---

## 14. Client DNS Server 확인

Client에서 `nslookup`을 사용하여 현재 사용 중인 DNS Server를 확인하였다.

```bash
nslookup
```

확인 결과:

```text
Server:         192.168.111.100
Address:        192.168.111.100#53
```

Client의 DNS 질의가 기존 VMware DNS `192.168.111.2`가 아닌 직접 구축한 Local DNS Caching Server `192.168.111.100`으로 전달되는 것을 확인하였다.

---

## 15. 실습 중 확인한 IPv6 Log

`named` 실행 시 다음과 같은 로그가 확인되었다.

```text
network unreachable resolving './NS/IN': 2001:500:1::53#53
network unreachable resolving './DNSKEY/IN': 2001:503:ba3e::2:30#53
```

현재 실습 환경에서 외부 IPv6 Route를 사용할 수 없어 IPv6 DNS Server로의 요청이 실패한 Log이다.

그러나 이후 다음 메시지가 정상적으로 출력되었다.

```text
all zones loaded
running
```

또한 `named` Service가 `active (running)` 상태이고 IPv4 TCP/UDP 53 Port도 정상적으로 Listen하고 있어 IPv4 기반 DNS Caching 실습에는 문제가 없음을 확인하였다.

---

## 16. 주요 확인 명령어

### BIND Package

```bash
rpm -q bind
rpm -q bind-utils
rpm -q bind-libs
```

### 설정 확인

```bash
grep -nE 'listen-on|listen-on-v6|allow-query|allow-recursion|recursion|dnssec-validation' /etc/named.conf
```

### 설정 문법 검사

```bash
named-checkconf
```

### Service 확인

```bash
systemctl status named
```

### DNS Port 확인

```bash
ss -luntp | grep ':53'
```

### DNS Cache 초기화

```bash
rndc flush
```

### DNS 직접 조회

```bash
dig @192.168.111.100 www.naver.com
```

### Query Time 확인

```bash
dig @192.168.111.100 www.naver.com | grep -E 'status:|Query time|SERVER:'
```

### Client DNS 확인

```bash
nmcli -f GENERAL.DEVICE,IP4.ADDRESS,IP4.GATEWAY,IP4.DNS device show ens160
```

```bash
nslookup
```

---

## 17. 실습 결과

- BIND Package 설치
- `/etc/named.conf` 설정
- 내부 Network DNS Query 허용
- 내부 Network Recursive DNS Query 허용
- `named-checkconf`를 이용한 설정 문법 검사
- Firewall DNS Service 허용
- `named` Service 자동 시작 설정
- TCP/UDP 53 Port Listen 확인
- `rndc flush`를 이용한 DNS Cache 초기화
- Local DNS Server를 통한 Domain 조회 성공
- 첫 번째 / 두 번째 DNS Query Time 비교
- DNS Cache에 의한 응답 속도 향상 확인
- Client DNS Server를 `192.168.111.100`으로 변경
- Client가 Local DNS Caching Server를 사용하는 것 확인

```text
DNS Caching Server : 192.168.111.100

첫 번째 Query  : 2128 msec
두 번째 Query  : 2 msec

Client DNS
192.168.111.2
        ↓
192.168.111.100
```

Rocky Linux에서 BIND 기반 DNS Caching Server를 구축하고, Client의 DNS 요청을 직접 처리하도록 구성하였다.

동일한 Domain의 반복 조회 시간을 비교하여 DNS Cache가 실제로 적용되는 것을 확인하였다.
