# DHCP Server 정리

## 1. DHCP란?

DHCP(Dynamic Host Configuration Protocol)는 Client가 네트워크에 연결될 때 필요한 네트워크 정보를 자동으로 할당하는 프로토콜이다.

DHCP를 사용하지 않으면 각 Client에 IP 주소, Gateway, DNS 등을 직접 설정해야 하지만 DHCP Server를 사용하면 이러한 정보를 자동으로 제공할 수 있다.

---

## 2. DHCP Server가 제공하는 정보

DHCP Server는 Client에게 다음과 같은 정보를 제공할 수 있다.

```text
IP Address
Subnet Mask
Default Gateway
DNS Server
Lease Time
```

예시:

```text
IP Address      192.168.111.150
Subnet Mask     255.255.255.0
Default Gateway 192.168.111.2
DNS Server      168.126.63.1
                8.8.8.8
Lease Time      86400초
```

---

## 3. DHCP Port

DHCP는 UDP 프로토콜을 사용한다.

| 구분 | Port | 역할 |
|---|---:|---|
| DHCP Server | UDP 67 | Client의 DHCP 요청 수신 |
| DHCP Client | UDP 68 | DHCP Server의 응답 수신 |

```text
DHCP Server : UDP 67
DHCP Client : UDP 68
```

DHCP 초기 통신에서는 Client가 아직 자신의 IP 주소를 가지고 있지 않기 때문에 Broadcast 통신이 사용된다.

---

## 4. DHCP DORA 과정

DHCP Client가 IP 주소를 할당받는 기본 과정을 DORA라고 한다.

```text
D = Discover
O = Offer
R = Request
A = ACK
```

### 4-1. DHCPDISCOVER

Client가 사용 가능한 DHCP Server를 찾기 위해 요청을 전송한다.

```text
Client
   |
   | DHCPDISCOVER
   v
Network
```

Client는 아직 DHCP Server의 위치를 모르기 때문에 DHCP Server를 찾는다.

---

### 4-2. DHCPOFFER

DHCP Server가 Client에게 사용 가능한 IP 주소와 네트워크 정보를 제안한다.

예:

```text
DHCPOFFER
IP : 192.168.111.150
```

---

### 4-3. DHCPREQUEST

Client가 DHCP Server가 제안한 IP 주소를 사용하겠다고 요청한다.

예:

```text
DHCPREQUEST
192.168.111.150 사용 요청
```

---

### 4-4. DHCPACK

DHCP Server가 Client의 요청을 승인한다.

```text
DHCPACK
192.168.111.150 할당 승인
```

최종적으로:

```text
Client                         DHCP Server

DHCPDISCOVER  -------------------->
              <-------------------- DHCPOFFER

DHCPREQUEST   -------------------->
              <-------------------- DHCPACK

Client IP 할당 완료
```

실습에서는 다음 로그를 통해 DORA 과정을 직접 확인하였다.

```text
DHCPDISCOVER from 00:0c:29:8f:7d:df via ens160
DHCPOFFER on 192.168.111.150 to 00:0c:29:8f:7d:df via ens160
DHCPREQUEST for 192.168.111.150 (192.168.111.100) from 00:0c:29:8f:7d:df via ens160
DHCPACK on 192.168.111.150 to 00:0c:29:8f:7d:df via ens160
```

---

## 5. DHCP 실습 구성

실습 환경:

```text
DHCP Server
192.168.111.100/24
Rocky Linux
ens160
      |
      | VMnet8
      |
DHCP Client
Rocky Linux
ens160
      |
      +--> DHCP를 통해 192.168.111.150 할당
```

Network:

```text
192.168.111.0/24
```

Default Gateway:

```text
192.168.111.2
```

DHCP 동적 할당 범위:

```text
192.168.111.150 ~ 192.168.111.180
```

DHCP Server의 고정 IP 주소는 DHCP 할당 범위에 포함하지 않는다.

```text
Server IP : 192.168.111.100

DHCP Range:
192.168.111.150 ~ 192.168.111.180
```

고정 IP 주소와 DHCP 동적 할당 범위가 겹치면 IP 충돌이 발생할 수 있으므로 서로 분리하여 구성한다.

---

## 6. DHCP Server Package

Rocky Linux에서 DHCP Server 구축에 사용하는 Package:

```text
dhcp-server
```

설치:

```bash
dnf install -y dhcp-server
```

Package 확인:

```bash
rpm -q dhcp-server
```

---

## 7. DHCP 주요 파일

### DHCP Server 설정 파일

```text
/etc/dhcp/dhcpd.conf
```

DHCP Server가 Client에게 제공할 네트워크 정보와 IP 주소 범위를 설정한다.

### DHCP Lease 파일

```text
/var/lib/dhcpd/dhcpd.leases
```

DHCP Server가 Client에게 할당한 IP 주소의 Lease 정보를 저장한다.

### Service

```text
dhcpd.service
```

상태 확인:

```bash
systemctl status dhcpd
```

---

## 8. dhcpd.conf 주요 설정

실습에서 사용한 설정:

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

---

## 9. dhcpd.conf 설정 항목

### authoritative

```conf
authoritative;
```

해당 네트워크에 대해 이 DHCP Server가 DHCP 서비스를 제공하도록 설정할 때 사용한다.

---

### ddns-update-style

```conf
ddns-update-style none;
```

Dynamic DNS Update를 사용하지 않도록 설정한다.

---

### subnet

```conf
subnet 192.168.111.0 netmask 255.255.255.0 {
}
```

DHCP 서비스를 제공할 Network 대역을 지정한다.

실습 Network:

```text
192.168.111.0/24
```

---

### option routers

```conf
option routers 192.168.111.2;
```

Client에게 제공할 Default Gateway 주소이다.

Client에서는 다음과 같이 확인할 수 있다.

```text
default via 192.168.111.2
```

---

### option domain-name-servers

```conf
option domain-name-servers 168.126.63.1, 8.8.8.8;
```

Client에게 제공할 DNS Server 주소이다.

실습 확인 결과:

```text
IP4.DNS[1]: 168.126.63.1
IP4.DNS[2]: 8.8.8.8
```

---

### option subnet-mask

```conf
option subnet-mask 255.255.255.0;
```

Client에게 제공할 Subnet Mask이다.

```text
255.255.255.0
=
/24
```

---

### option broadcast-address

```conf
option broadcast-address 192.168.111.255;
```

Client에게 제공할 Broadcast 주소이다.

`192.168.111.0/24`의 Broadcast 주소:

```text
192.168.111.255
```

---

### range

```conf
range 192.168.111.150 192.168.111.180;
```

DHCP Server가 Client에게 동적으로 할당할 IP 주소 범위이다.

```text
시작 : 192.168.111.150
끝   : 192.168.111.180
```

---

### default-lease-time

```conf
default-lease-time 86400;
```

기본 IP 임대 시간이다.

```text
86400초
=
24시간
=
1일
```

---

### max-lease-time

```conf
max-lease-time 864000;
```

최대 IP 임대 시간이다.

```text
864000초
=
10일
```

---

## 10. Lease란?

DHCP는 IP 주소를 Client에게 영구적으로 주는 것이 아니라 일정 시간 동안 임대한다.

이를 Lease라고 한다.

```text
DHCP Server
     |
     | 192.168.111.150
     | Lease
     v
DHCP Client
```

DHCP Server의 Lease 정보 확인:

```bash
cat /var/lib/dhcpd/dhcpd.leases
```

실습 결과:

```text
lease 192.168.111.150 {
  starts 1 2026/09/14 01:58:30;
  ends 2 2026/09/15 01:58:30;
  binding state active;
  hardware ethernet 00:0c:29:8f:7d:df;
}
```

주요 정보:

```text
lease
→ Client에게 임대한 IP 주소

starts
→ Lease 시작 시간

ends
→ Lease 종료 시간

binding state active
→ 현재 사용 중인 Lease

hardware ethernet
→ IP를 할당받은 Client의 MAC 주소
```

---

## 11. DHCP Client 설정

Rocky Linux에서는 NetworkManager를 사용하여 DHCP 자동 설정을 사용할 수 있다.

DHCP Client의 연결을 다시 활성화:

```bash
nmcli connection down ens160
nmcli connection up ens160
```

IP 확인:

```bash
ip -br addr show ens160
```

Route 확인:

```bash
ip route
```

IP, Gateway, DNS 확인:

```bash
nmcli -f GENERAL.DEVICE,IP4.ADDRESS,IP4.GATEWAY,IP4.DNS device show ens160
```

실습 결과:

```text
IP4.ADDRESS[1]: 192.168.111.150/24
IP4.GATEWAY:    192.168.111.2
IP4.DNS[1]:     168.126.63.1
IP4.DNS[2]:     8.8.8.8
```

---

## 12. VMware DHCP Server와의 충돌

VMware VMnet8에는 자체 DHCP Server 기능이 존재할 수 있다.

직접 Rocky Linux DHCP Server를 구축하면서 VMware DHCP도 동시에 실행하면 한 Network에 DHCP Server가 두 개 존재하게 된다.

```text
                +--> VMware DHCP Server
Client ---------|
                +--> Rocky Linux DHCP Server
```

이 경우 Client가 어느 DHCP Server의 응답을 받을지 예측하기 어렵다.

따라서 실습에서는 VMware DHCP Server를 중지하고 Rocky Linux DHCP Server만 동작하도록 구성하였다.

VMware Workstation:

```text
Edit
→ Virtual Network Editor
→ Change Settings
→ VMnet8
→ Use local DHCP service to distribute IP address to VMs
   체크 해제
→ Apply
```

VMware DHCP를 중지해도 VMnet8의 NAT 및 Gateway 기능은 사용할 수 있다.

---

## 13. Firewall 설정

DHCP Server가 Client의 요청을 받을 수 있도록 Firewall에서 DHCP 서비스를 허용한다.

```bash
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-services
```

실습 확인:

```text
cockpit dhcp dhcpv6-client ssh
```

---

## 14. DHCP Server 시작

서비스 시작 및 부팅 시 자동 시작 설정:

```bash
systemctl enable --now dhcpd
```

상태 확인:

```bash
systemctl status dhcpd
```

정상 상태:

```text
Active: active (running)
Status: "Dispatching packets..."
```

UDP 67 포트 확인:

```bash
ss -lunp | grep ':67'
```

실습 결과:

```text
0.0.0.0:67
users:(("dhcpd",pid=3166,fd=7))
```

DHCP Server가 UDP 67 포트에서 Client의 요청을 기다리고 있음을 확인할 수 있다.

---

## 15. DHCP 설정 문법 검사

DHCP Server를 시작하기 전에 설정 파일의 문법을 검사할 수 있다.

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

문법 오류가 존재할 경우 Service를 시작하기 전에 문제를 확인할 수 있다.

---

## 16. DHCP Log 확인

DHCP Server의 동작 과정 확인:

```bash
journalctl -u dhcpd -n 50 --no-pager
```

Client가 DHCP 요청을 하면 다음과 같은 로그를 확인할 수 있다.

```text
DHCPDISCOVER
DHCPOFFER
DHCPREQUEST
DHCPACK
```

따라서 DHCP가 단순히 IP를 할당했다는 결과뿐만 아니라 실제 DORA 과정도 확인할 수 있다.

---

## 17. 실습 중 발생한 오류

DHCP 설정 전에 `dhcpd`를 실행했을 때 Service 시작에 실패하였다.

로그:

```text
No subnet declaration for ens160 (192.168.111.100).
Ignoring requests on ens160.
Not configured to listen on any interfaces!
```

원인:

```text
/etc/dhcp/dhcpd.conf
```

파일에 `ens160`이 연결된 Network에 대한 `subnet` 선언이 존재하지 않았다.

Server Interface:

```text
192.168.111.100/24
```

따라서 다음 Network 설정이 필요하였다.

```conf
subnet 192.168.111.0 netmask 255.255.255.0 {
    ...
}
```

설정 후:

```text
Listening on LPF/ens160/.../192.168.111.0/24
Server starting service.
```

가 출력되면서 DHCP Server가 정상 실행되었다.

---

## 18. unknown lease

Client가 기존에 사용하던 IP 주소 `192.168.111.130`을 요청했을 때 다음 로그가 확인되었다.

```text
DHCPREQUEST for 192.168.111.130 from 00:0c:29:8f:7d:df via ens160: unknown lease 192.168.111.130.
```

새로 구축한 Rocky Linux DHCP Server의 Lease 정보에는 기존 `192.168.111.130`에 대한 기록이 없었기 때문이다.

이후 Client는 DHCPDISCOVER를 수행하고 새로운 IP 주소를 할당받았다.

```text
192.168.111.150
```

---

## 19. DHCP 동작 확인 순서

DHCP Server를 구축한 후 다음 순서로 확인할 수 있다.

```text
1. dhcp-server Package 설치

2. /etc/dhcp/dhcpd.conf 설정

3. dhcpd.conf 문법 검사

4. Firewall DHCP 허용

5. dhcpd Service 시작

6. UDP 67 Listen 확인

7. Client DHCP 연결 갱신

8. Client IP / Gateway / DNS 확인

9. DHCP Server Log에서 DORA 확인

10. dhcpd.leases에서 Lease 확인

11. Server / Gateway / 외부 Network 통신 확인
```

---

## 20. 주요 명령어 정리

### Server

```bash
dnf install -y dhcp-server
```

```bash
vi /etc/dhcp/dhcpd.conf
```

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

```bash
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --reload
```

```bash
systemctl enable --now dhcpd
```

```bash
systemctl status dhcpd
```

```bash
ss -lunp | grep ':67'
```

```bash
journalctl -u dhcpd -n 50 --no-pager
```

```bash
cat /var/lib/dhcpd/dhcpd.leases
```

### Client

```bash
nmcli connection down ens160
nmcli connection up ens160
```

```bash
ip -br addr show ens160
```

```bash
ip route
```

```bash
nmcli -f GENERAL.DEVICE,IP4.ADDRESS,IP4.GATEWAY,IP4.DNS device show ens160
```

---

## 21. 핵심 정리

```text
DHCP
→ Client에게 Network 정보를 자동으로 제공

Server Port
→ UDP 67

Client Port
→ UDP 68

DORA
→ Discover
→ Offer
→ Request
→ ACK

주요 설정 파일
→ /etc/dhcp/dhcpd.conf

Lease 파일
→ /var/lib/dhcpd/dhcpd.leases

Service
→ dhcpd

실습 DHCP Server
→ 192.168.111.100

DHCP 할당 범위
→ 192.168.111.150 ~ 192.168.111.180

Client 실제 할당 IP
→ 192.168.111.150

Default Gateway
→ 192.168.111.2

DNS
→ 168.126.63.1
→ 8.8.8.8
```

DHCP Server 구축에서 중요한 것은 단순히 Client가 IP 주소를 받았는지만 확인하는 것이 아니라, Server의 `dhcpd` 상태, UDP 67 Port, Client에 적용된 Gateway/DNS, DORA Log, Lease 정보까지 함께 확인하는 것이다.
