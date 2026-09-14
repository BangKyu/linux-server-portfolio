# DHCP Server 구축

Rocky Linux에서 DHCP Server를 구축하고 Client가 IP 주소, Gateway, DNS 정보를 자동으로 할당받는 과정을 실습하였다.

서버 로그를 통해 DHCP의 DORA 과정을 확인하고, Lease 정보와 실제 네트워크 통신까지 검증하였다.

---

## 1. 실습 환경

| 구분 | 설정 |
|---|---|
| DHCP Server | Rocky Linux |
| Server Interface | ens160 |
| Server IP | 192.168.111.100/24 |
| Default Gateway | 192.168.111.2 |
| DHCP Client | Rocky Linux |
| Network | 192.168.111.0/24 |
| DHCP Range | 192.168.111.150 ~ 192.168.111.180 |
| DNS | 168.126.63.1, 8.8.8.8 |
| DHCP Server Port | UDP 67 |
| DHCP Client Port | UDP 68 |

VMware VMnet8 환경에서 실습하였으며 VMware DHCP와 Rocky Linux DHCP Server의 중복 응답을 방지하도록 구성하였다.

---

## 2. DHCP Server 설치

```bash
dnf install -y dhcp-server
```

DHCP Server의 주요 설정 파일은 다음과 같다.

```text
/etc/dhcp/dhcpd.conf
```

---

## 3. DHCP Server 설정

```bash
vi /etc/dhcp/dhcpd.conf
```

설정 내용:

```conf
authoritative;
ddns-update-style none;

subnet 192.168.111.0 netmask 255.255.255.0 {
    option routers 192.168.111.2;
    option domain-name-servers 168.126.63.1, 8.8.8.8;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.111.255;

    range 192.168.111.150 192.168.111.180;

    default-lease-time 86400;
    max-lease-time 864000;
}
```

설정한 DHCP 할당 범위:

```text
192.168.111.150 ~ 192.168.111.180
```

Server의 고정 IP `192.168.111.100`과 다른 고정 IP에서 사용하는 주소가 DHCP 할당 범위와 겹치지 않도록 구성하였다.

---

## 4. DHCP 설정 문법 검사

서비스를 시작하기 전에 설정 파일의 문법을 검사하였다.

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

실행 결과:

```text
Internet Systems Consortium DHCP Server 4.4.2b1
Copyright 2004-2019 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/
ldap_gssapi_principal is not set,GSSAPI Authentication for LDAP will not be used
Not searching LDAP since ldap-server, ldap-port and ldap-base-dn were not specified in the config file
Config file: /etc/dhcp/dhcpd.conf
Database file: /var/lib/dhcpd/dhcpd.leases
PID file: /var/run/dhcpd.pid
Source compiled to use binary-leases
```

별도의 문법 오류가 출력되지 않는 것을 확인하였다.

---

## 5. 방화벽 DHCP 서비스 허용

```bash
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --reload
firewall-cmd --list-services
```

확인 결과:

```text
cockpit dhcp dhcpv6-client ssh
```

방화벽에서 DHCP 서비스가 허용된 것을 확인하였다.

---

## 6. DHCP Server 시작

```bash
systemctl enable --now dhcpd
systemctl status dhcpd --no-pager
```

확인 결과:

```text
● dhcpd.service - DHCPv4 Server Daemon
     Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; enabled; preset: disabled)
     Active: active (running) since Mon 2026-09-14 10:54:15 KST
       Docs: man:dhcpd(8)
             man:dhcpd.conf(5)
   Main PID: 3166 (dhcpd)
     Status: "Dispatching packets..."
```

DHCP Server가 정상적으로 실행 중인 것을 확인하였다.

---

## 7. UDP 67 포트 확인

```bash
ss -lunp | grep ':67'
```

확인 결과:

```text
UNCONN 0      0      0.0.0.0:67      0.0.0.0:*    users:(("dhcpd",pid=3166,fd=7))
```

`dhcpd` 프로세스가 UDP 67 포트에서 Client 요청을 대기하고 있는 것을 확인하였다.

---

## 8. DHCP Client IP 할당 확인

DHCP 갱신 전 Client의 IP 주소:

```bash
ip -br addr show ens160
ip route
```

```text
ens160    UP    192.168.111.130/24

default via 192.168.111.2 dev ens160 proto dhcp src 192.168.111.130 metric 100
192.168.111.0/24 dev ens160 proto kernel scope link src 192.168.111.130 metric 100
```

NetworkManager 연결을 다시 활성화하여 DHCP Server에 새로운 네트워크 설정을 요청하였다.

```bash
sudo nmcli connection down ens160
sudo nmcli connection up ens160
```

갱신 후 확인:

```bash
ip -br addr show ens160
ip route
nmcli -f GENERAL.DEVICE,IP4.ADDRESS,IP4.GATEWAY,IP4.DNS device show ens160
```

실행 결과:

```text
ens160    UP    192.168.111.150/24

default via 192.168.111.2 dev ens160 proto dhcp src 192.168.111.150 metric 100
192.168.111.0/24 dev ens160 proto kernel scope link src 192.168.111.150 metric 100
```

```text
GENERAL.DEVICE:                         ens160
IP4.ADDRESS[1]:                         192.168.111.150/24
IP4.GATEWAY:                            192.168.111.2
IP4.DNS[1]:                             168.126.63.1
IP4.DNS[2]:                             8.8.8.8
```

DHCP Server에서 설정한 정보가 Client에 정상적으로 적용된 것을 확인하였다.

```text
IP Address : 192.168.111.150/24
Gateway    : 192.168.111.2
DNS        : 168.126.63.1
             8.8.8.8
```

---

## 9. DHCP DORA 과정 확인

DHCP Server에서 로그를 확인하였다.

```bash
journalctl -u dhcpd -n 50 --no-pager
```

Client가 기존 주소 `192.168.111.130`을 요청했지만 DHCP Server에 해당 Lease 정보가 존재하지 않는 것을 확인하였다.

```text
DHCPREQUEST for 192.168.111.130 from 00:0c:29:8f:7d:df via ens160: unknown lease 192.168.111.130.
```

이후 DHCP DORA 과정이 정상적으로 수행되었다.

```text
DHCPDISCOVER from 00:0c:29:8f:7d:df via ens160
DHCPOFFER on 192.168.111.150 to 00:0c:29:8f:7d:df via ens160
DHCPREQUEST for 192.168.111.150 (192.168.111.100) from 00:0c:29:8f:7d:df via ens160
DHCPACK on 192.168.111.150 to 00:0c:29:8f:7d:df via ens160
```

동작 과정:

```text
Client                         DHCP Server

DHCPDISCOVER  -------------------->
              <-------------------- DHCPOFFER

DHCPREQUEST   -------------------->
              <-------------------- DHCPACK

Client IP : 192.168.111.150
```

Client가 DHCP Server를 탐색하고, Server가 `192.168.111.150`을 제안한 뒤 Client의 요청을 승인하는 과정을 확인하였다.

---

## 10. DHCP Lease 확인

DHCP Server의 Lease 파일을 확인하였다.

```bash
cat /var/lib/dhcpd/dhcpd.leases
```

확인 결과:

```text
lease 192.168.111.150 {
  starts 1 2026/09/14 01:58:30;
  ends 2 2026/09/15 01:58:30;
  cltt 1 2026/09/14 01:58:30;
  binding state active;
  next binding state free;
  rewind binding state free;
  hardware ethernet 00:0c:29:8f:7d:df;
  uid "\001\000\014)\217}\337";
}
```

다음 정보를 확인하였다.

```text
할당 IP     : 192.168.111.150
Lease 상태  : active
Client MAC  : 00:0c:29:8f:7d:df
```

DHCP Server가 Client에게 할당한 IP 정보를 Lease 파일에 정상적으로 기록하고 있음을 확인하였다.

---

## 11. 네트워크 통신 확인

DHCP로 네트워크 정보를 할당받은 Client에서 통신을 확인하였다.

### DHCP Server 통신

```bash
ping -c 3 192.168.111.100
```

```text
3 packets transmitted, 3 received, 0% packet loss
```

### Default Gateway 통신

```bash
ping -c 3 192.168.111.2
```

```text
3 packets transmitted, 3 received, 0% packet loss
```

### 외부 네트워크 통신

```bash
ping -c 3 8.8.8.8
```

```text
3 packets transmitted, 3 received, 0% packet loss
```

DHCP로 할당받은 네트워크 설정을 이용하여 DHCP Server, Gateway, 외부 네트워크까지 정상적으로 통신하는 것을 확인하였다.

---

## 12. 트러블슈팅

DHCP 설정 전에 서비스를 실행했을 때 서비스 시작에 실패하였다.

```bash
systemctl start dhcpd
```

로그 확인:

```bash
journalctl -u dhcpd -n 50 --no-pager
```

오류 내용:

```text
No subnet declaration for ens160 (192.168.111.100).
Ignoring requests on ens160.
Not configured to listen on any interfaces!
```

`ens160`이 연결된 `192.168.111.0/24` 네트워크에 대한 `subnet` 설정이 `/etc/dhcp/dhcpd.conf`에 존재하지 않아 DHCP Server가 해당 Interface에서 요청을 받을 수 없는 상태였다.

`dhcpd.conf`에 다음 네트워크를 설정한 후 문제를 해결하였다.

```conf
subnet 192.168.111.0 netmask 255.255.255.0 {
    ...
}
```

설정 후 로그:

```text
Listening on LPF/ens160/00:0c:29:dd:0c:da/192.168.111.0/24
Sending on   LPF/ens160/00:0c:29:dd:0c:da/192.168.111.0/24
Server starting service.
```

DHCP Server가 `ens160`의 `192.168.111.0/24` 네트워크에서 정상적으로 요청을 처리하는 것을 확인하였다.

---

## 13. 실습 결과

- DHCP Server 설치 및 설정
- DHCP IP 할당 범위 구성
- DHCP 방화벽 서비스 허용
- DHCP Server UDP 67 포트 확인
- DHCP Client 자동 IP 할당 확인
- Gateway 및 DNS 자동 설정 확인
- DHCP DORA 과정 로그 확인
- DHCP Lease 정보 확인
- DHCP Server, Gateway, 외부 네트워크 통신 확인
- subnet 미설정으로 인한 DHCP Server 실행 오류 원인 확인 및 해결

Rocky Linux DHCP Server를 구축하여 Client에 네트워크 설정을 자동으로 제공하고, 서버 로그와 Lease 정보를 통해 실제 DHCP 동작 과정을 검증하였다.
