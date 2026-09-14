# Master DNS Server 구축

Rocky Linux에서 BIND를 이용하여 `bangkyu.com` 도메인을 직접 관리하는 Master DNS Server를 구축하였다.

정방향 Zone과 역방향 Zone을 구성하고, A / NS / SOA / PTR Record를 등록한 뒤 Client에서 Domain → IP, IP → Domain 조회를 검증하였다.

또한 DNS 응답의 `aa(Authoritative Answer)` Flag를 확인하여 해당 DNS Server가 `bangkyu.com` Zone에 대한 권한 있는 DNS Server임을 확인하였다.

---

## 1. 실습 환경

| 구분 | 설정 |
|---|---|
| DNS Server | Rocky Linux |
| DNS Server IP | 192.168.111.100 |
| Interface | ens160 |
| Client | Rocky Linux |
| Client IP | 192.168.111.150/24 |
| Network | 192.168.111.0/24 |
| Domain | bangkyu.com |
| DNS Software | BIND |
| DNS Daemon | named |
| DNS Port | TCP / UDP 53 |

구성:

```text
                         Master DNS
                       192.168.111.100
                              |
                 +------------+------------+
                 |                         |
        www.bangkyu.com            ftp.bangkyu.com
        192.168.111.100             192.168.111.200
                 |
                 |
             DNS Client
          192.168.111.150
       DNS = 192.168.111.100
```

---

## 2. 기존 DNS Server 상태 확인

이전 DNS Caching Server 실습에서 구성한 `named` Service를 그대로 사용하였다.

```bash
systemctl is-active named
systemctl is-enabled named
```

확인 결과:

```text
active
enabled
```

기존 주요 설정:

```bash
grep -nE 'listen-on|allow-query|allow-recursion|recursion' /etc/named.conf
```

```text
11:     listen-on port 53 { any; };
12:     listen-on-v6 port 53 { none; };
19:     allow-query     { localhost; 192.168.111.0/24; };
20:     allow-recursion { localhost; 192.168.111.0/24; };
31:     recursion yes;
```

현재 DNS Server는 내부 Network의 DNS Query와 Recursive Query를 처리하도록 구성되어 있다.

---

## 3. 기존 설정 백업

Master DNS 설정 전 기존 BIND 설정을 백업하였다.

```bash
mkdir -p /backup/master-dns

cp -a /etc/named.conf /backup/master-dns/
cp -a /etc/named.rfc1912.zones /backup/master-dns/

ls -l /backup/master-dns
```

확인 결과:

```text
합계 8
-rw-r-----. 1 root named 1792  9월 14 11:17 named.conf
-rw-r-----. 1 root named 1029  8월 13 21:08 named.rfc1912.zones
```

---

## 4. 정방향 Zone 등록

`bangkyu.com` Domain을 Master DNS Server가 직접 관리하도록 Zone을 등록하였다.

```bash
vi /etc/named.rfc1912.zones
```

추가:

```conf
zone "bangkyu.com" IN {
        type master;
        file "bangkyu.com.db";
};
```

설정 의미:

```text
zone "bangkyu.com"
→ 관리할 Domain

type master
→ 해당 Zone의 원본 데이터를 직접 관리

file "bangkyu.com.db"
→ 실제 DNS Record가 저장된 Zone File
→ /var/named/bangkyu.com.db
```

---

## 5. 정방향 Zone File 생성

```bash
vi /var/named/bangkyu.com.db
```

설정:

```dns
$TTL 1D

@       IN      SOA     ns.bangkyu.com. admin.bangkyu.com. (
                        2026091401      ; Serial
                        1D              ; Refresh
                        1H              ; Retry
                        1W              ; Expire
                        3H )            ; Minimum TTL

@       IN      NS      ns.bangkyu.com.
@       IN      A       192.168.111.100

ns      IN      A       192.168.111.100
www     IN      A       192.168.111.100
ftp     IN      A       192.168.111.200
```

구성된 DNS Record:

```text
bangkyu.com
→ 192.168.111.100

ns.bangkyu.com
→ 192.168.111.100

www.bangkyu.com
→ 192.168.111.100

ftp.bangkyu.com
→ 192.168.111.200
```

---

## 6. Zone File 권한 설정

`named`가 Zone File을 읽을 수 있도록 소유 그룹과 Permission을 설정하였다.

```bash
chown root:named /var/named/bangkyu.com.db
chmod 640 /var/named/bangkyu.com.db
```

확인:

```bash
ls -l /var/named/bangkyu.com.db
```

```text
-rw-r-----. 1 root named 522  9월 14 11:46 /var/named/bangkyu.com.db
```

---

## 7. 정방향 Zone 문법 검사

Zone File 검사:

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db
```

확인 결과:

```text
zone bangkyu.com/IN: loaded serial 2026091401
OK
```

전체 BIND 설정 검사:

```bash
named-checkconf
```

별도의 오류가 출력되지 않아 설정 문법이 정상임을 확인하였다.

Zone File까지 포함한 전체 검사:

```bash
named-checkconf -z
```

확인 결과:

```text
zone localhost.localdomain/IN: loaded serial 0
zone localhost/IN: loaded serial 0
zone 1.0.0.127.in-addr.arpa/IN: loaded serial 0
zone 0.in-addr.arpa/IN: loaded serial 0
zone bangkyu.com/IN: loaded serial 2026091401
```

`bangkyu.com` Zone이 정상적으로 Load되는 것을 확인하였다.

---

## 8. Zone File 이름 오류 해결

처음 Zone을 등록할 때 Domain 이름은 `bangkyu.com`이었지만 Zone File 이름을 기존 실습 예제인 `soldesk.com.db`로 잘못 지정하였다.

잘못된 설정:

```conf
zone "bangkyu.com" IN {
        type master;
        file "soldesk.com.db";
};
```

실제 생성한 File:

```text
/var/named/bangkyu.com.db
```

`named-checkconf -z` 실행 결과 다음 오류가 발생하였다.

```text
zone bangkyu.com/IN: loading from master file soldesk.com.db failed: file not found
zone bangkyu.com/IN: not loaded due to errors.
_default/bangkyu.com/IN: file not found
```

Zone 선언을 다음과 같이 수정하였다.

```conf
zone "bangkyu.com" IN {
        type master;
        file "bangkyu.com.db";
};
```

또한 Zone File 내부의 NS Record도 Domain에 맞게 수정하였다.

```text
기존
ns.soldesk.com.

변경
ns.bangkyu.com.
```

수정 후:

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db
named-checkconf -z
```

결과:

```text
zone bangkyu.com/IN: loaded serial 2026091401
OK

zone bangkyu.com/IN: loaded serial 2026091401
```

Zone 이름과 Zone File 이름을 동일한 Domain 기준으로 맞추어 문제를 해결하였다.

---

## 9. 정방향 DNS 조회 확인

DNS Server를 직접 지정하여 각 Record를 조회하였다.

### bangkyu.com

```bash
dig @192.168.111.100 bangkyu.com A +noall +answer
```

```text
bangkyu.com.    86400    IN    A    192.168.111.100
```

### ns.bangkyu.com

```bash
dig @192.168.111.100 ns.bangkyu.com A +noall +answer
```

```text
ns.bangkyu.com.    86400    IN    A    192.168.111.100
```

### www.bangkyu.com

```bash
dig @192.168.111.100 www.bangkyu.com A +noall +answer
```

```text
www.bangkyu.com.    86400    IN    A    192.168.111.100
```

### ftp.bangkyu.com

```bash
dig @192.168.111.100 ftp.bangkyu.com A +noall +answer
```

```text
ftp.bangkyu.com.    86400    IN    A    192.168.111.200
```

Zone File에 등록한 A Record가 정상적으로 반환되는 것을 확인하였다.

---

## 10. NS Record 확인

```bash
dig @192.168.111.100 bangkyu.com NS +noall +answer
```

확인 결과:

```text
bangkyu.com.    86400    IN    NS    ns.bangkyu.com.
```

`bangkyu.com`의 Name Server가 `ns.bangkyu.com`으로 정상 등록된 것을 확인하였다.

---

## 11. SOA Record 확인

```bash
dig @192.168.111.100 bangkyu.com SOA +noall +answer
```

확인 결과:

```text
bangkyu.com.    86400    IN    SOA    ns.bangkyu.com. admin.bangkyu.com. 2026091401 86400 3600 604800 10800
```

확인된 정보:

```text
Primary Name Server
→ ns.bangkyu.com.

Administrator
→ admin.bangkyu.com.

Serial
→ 2026091401

Refresh
→ 86400

Retry
→ 3600

Expire
→ 604800

Minimum TTL
→ 10800
```

---

## 12. Authoritative Answer 확인

Master DNS Server가 `bangkyu.com`에 대해 직접 권한 있는 응답을 제공하는지 확인하였다.

```bash
dig @192.168.111.100 www.bangkyu.com A +norecurse | grep -E 'HEADER|flags:|ANSWER SECTION'
```

확인 결과:

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50774
;; flags: qr aa ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2
;; ANSWER SECTION:
```

Flag:

```text
aa
```

가 포함되어 있다.

```text
aa = Authoritative Answer
```

따라서 `192.168.111.100` DNS Server가 `bangkyu.com` Zone의 원본 정보를 직접 보유하고 해당 Domain에 대한 Authoritative DNS Server로 동작함을 확인하였다.

현재 Server에는 `ra` Flag도 확인되므로 기존 Caching DNS의 Recursive 기능도 함께 제공하고 있다.

---

## 13. 역방향 DNS Zone 등록

정방향 DNS:

```text
Domain → IP
```

역방향 DNS:

```text
IP → Domain
```

현재 Network:

```text
192.168.111.0/24
```

에 대한 Reverse Zone은 다음과 같다.

```text
111.168.192.in-addr.arpa
```

`/etc/named.rfc1912.zones`에 Reverse Zone을 추가하였다.

```conf
zone "111.168.192.in-addr.arpa" IN {
        type master;
        file "bangkyu.com.rev";
};
```

최종 Zone 구성:

```conf
zone "bangkyu.com" IN {
        type master;
        file "bangkyu.com.db";
};

zone "111.168.192.in-addr.arpa" IN {
        type master;
        file "bangkyu.com.rev";
};
```

---

## 14. 역방향 Zone File 생성

```bash
vi /var/named/bangkyu.com.rev
```

설정:

```dns
$TTL 1D

@       IN      SOA     ns.bangkyu.com. admin.bangkyu.com. (
                        2026091401      ; Serial
                        1D              ; Refresh
                        1H              ; Retry
                        1W              ; Expire
                        3H )            ; Minimum TTL

@       IN      NS      ns.bangkyu.com.

100     IN      PTR     www.bangkyu.com.
200     IN      PTR     ftp.bangkyu.com.
```

PTR Record:

```text
192.168.111.100
→ www.bangkyu.com.

192.168.111.200
→ ftp.bangkyu.com.
```

---

## 15. 역방향 Zone File 권한 설정

```bash
chown root:named /var/named/bangkyu.com.rev
chmod 640 /var/named/bangkyu.com.rev
restorecon -v /var/named/bangkyu.com.rev
```

---

## 16. 역방향 Zone 문법 검사

```bash
named-checkzone 111.168.192.in-addr.arpa /var/named/bangkyu.com.rev
```

확인 결과:

```text
zone 111.168.192.in-addr.arpa/IN: loaded serial 2026091401
OK
```

전체 Zone 검사:

```bash
named-checkconf -z
```

확인 결과:

```text
zone localhost.localdomain/IN: loaded serial 0
zone localhost/IN: loaded serial 0
zone 1.0.0.127.in-addr.arpa/IN: loaded serial 0
zone 0.in-addr.arpa/IN: loaded serial 0
zone bangkyu.com/IN: loaded serial 2026091401
zone 111.168.192.in-addr.arpa/IN: loaded serial 2026091401
```

정방향 Zone과 역방향 Zone이 모두 정상적으로 Load되는 것을 확인하였다.

---

## 17. Client 정방향 DNS 조회

Client는 DNS Server로 `192.168.111.100`을 사용하도록 설정되어 있다.

Client에서:

```bash
nslookup www.bangkyu.com
nslookup ftp.bangkyu.com
```

### www.bangkyu.com

```text
Server:         192.168.111.100
Address:        192.168.111.100#53

Name:   www.bangkyu.com
Address: 192.168.111.100
```

### ftp.bangkyu.com

```text
Server:         192.168.111.100
Address:        192.168.111.100#53

Name:   ftp.bangkyu.com
Address: 192.168.111.200
```

Client가 직접 구축한 Master DNS Server를 이용하여 Domain Name을 정상적으로 IP 주소로 변환하는 것을 확인하였다.

---

## 18. Client 역방향 DNS 조회

Client에서 IP 주소를 이용한 Reverse DNS 조회를 수행하였다.

```bash
nslookup 192.168.111.100
nslookup 192.168.111.200
```

확인 결과:

```text
100.111.168.192.in-addr.arpa    name = www.bangkyu.com.
```

```text
200.111.168.192.in-addr.arpa    name = ftp.bangkyu.com.
```

정방향과 역방향 DNS가 다음과 같이 서로 대응하는 것을 확인하였다.

```text
www.bangkyu.com
        ↓
192.168.111.100
        ↓
www.bangkyu.com.
```

```text
ftp.bangkyu.com
        ↓
192.168.111.200
        ↓
ftp.bangkyu.com.
```

---

## 19. 정방향 DNS와 역방향 DNS

### 정방향 DNS

Domain Name을 IP 주소로 변환한다.

```text
www.bangkyu.com
        ↓
192.168.111.100
```

주요 Record:

```text
A Record
```

---

### 역방향 DNS

IP 주소를 Domain Name으로 변환한다.

```text
192.168.111.100
        ↓
www.bangkyu.com
```

주요 Record:

```text
PTR Record
```

Reverse DNS에서는 다음 Namespace를 사용한다.

```text
in-addr.arpa
```

---

## 20. 주요 DNS Record

### SOA

```text
Start Of Authority
```

Zone의 기본 관리 정보를 저장한다.

이번 실습:

```text
ns.bangkyu.com.
admin.bangkyu.com.
Serial 2026091401
```

### NS

해당 Domain을 관리하는 Name Server를 지정한다.

```text
bangkyu.com
→ ns.bangkyu.com
```

### A

Host Name을 IPv4 주소에 연결한다.

```text
www.bangkyu.com
→ 192.168.111.100

ftp.bangkyu.com
→ 192.168.111.200
```

### PTR

IPv4 주소를 Domain Name에 연결하는 Reverse DNS Record이다.

```text
192.168.111.100
→ www.bangkyu.com

192.168.111.200
→ ftp.bangkyu.com
```

---

## 21. 주요 파일

### BIND 전체 설정

```text
/etc/named.conf
```

### Zone 선언

```text
/etc/named.rfc1912.zones
```

### 정방향 Zone File

```text
/var/named/bangkyu.com.db
```

### 역방향 Zone File

```text
/var/named/bangkyu.com.rev
```

---

## 22. 주요 검증 명령어

### BIND 설정 검사

```bash
named-checkconf
```

### 전체 Zone Load 검사

```bash
named-checkconf -z
```

### 정방향 Zone 검사

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db
```

### 역방향 Zone 검사

```bash
named-checkzone 111.168.192.in-addr.arpa /var/named/bangkyu.com.rev
```

### A Record 조회

```bash
dig @192.168.111.100 www.bangkyu.com A
```

### NS Record 조회

```bash
dig @192.168.111.100 bangkyu.com NS
```

### SOA Record 조회

```bash
dig @192.168.111.100 bangkyu.com SOA
```

### Authoritative Answer 확인

```bash
dig @192.168.111.100 www.bangkyu.com A +norecurse
```

### Client 정방향 조회

```bash
nslookup www.bangkyu.com
nslookup ftp.bangkyu.com
```

### Client 역방향 조회

```bash
nslookup 192.168.111.100
nslookup 192.168.111.200
```

---

## 23. 실습 결과

- BIND 기반 Master DNS Server 구성
- `bangkyu.com` 정방향 Zone 등록
- SOA Record 구성
- NS Record 구성
- A Record 구성
- Zone File Permission 설정
- `named-checkzone`을 통한 Zone File 검사
- `named-checkconf -z`를 통한 전체 Zone Load 검사
- Zone File 이름 불일치 오류 확인 및 해결
- `bangkyu.com` 정방향 DNS 조회 성공
- `aa` Flag를 통한 Authoritative Answer 확인
- `192.168.111.0/24` Reverse Zone 등록
- PTR Record 구성
- 역방향 DNS 조회 성공
- Client에서 Master DNS Server를 이용한 정방향 / 역방향 조회 확인

최종 DNS 구성:

```text
DNS Server
192.168.111.100

Forward DNS
bangkyu.com       → 192.168.111.100
ns.bangkyu.com    → 192.168.111.100
www.bangkyu.com   → 192.168.111.100
ftp.bangkyu.com   → 192.168.111.200

Reverse DNS
192.168.111.100   → www.bangkyu.com
192.168.111.200   → ftp.bangkyu.com

Authoritative Answer
aa Flag 확인
```

Rocky Linux에서 BIND를 이용하여 `bangkyu.com` Domain의 Master DNS Server를 구축하고, 정방향 및 역방향 Zone을 직접 관리하도록 구성하였다.

Zone File의 문법 검사, Authoritative Answer 확인, Client의 실제 Domain/IP 조회를 통해 Master DNS Server가 정상적으로 동작하는 것을 검증하였다.
