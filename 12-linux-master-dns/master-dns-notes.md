# Master DNS Server 정리

## 1. Master DNS Server란?

Master DNS Server는 특정 Domain의 DNS 정보를 직접 가지고 관리하는 DNS Server이다.

현재는 Primary DNS Server라는 표현도 사용한다.

Master DNS Server는 자신이 관리하는 Domain에 대한 Zone File을 직접 가지고 있으며 DNS Query가 들어오면 해당 Zone File을 이용하여 응답한다.

예:

```text
bangkyu.com
```

이라는 Domain을 Master DNS Server가 관리한다고 하면:

```text
www.bangkyu.com
ftp.bangkyu.com
ns.bangkyu.com
```

등의 Host 정보를 직접 저장하고 관리할 수 있다.

구조:

```text
              Master DNS Server
               192.168.111.100
                      |
                bangkyu.com
                      |
          +-----------+-----------+
          |           |           |
          ns          www         ftp
          |           |           |
        .100        .100        .200
```

---

## 2. Master DNS의 특징

Master DNS Server의 주요 특징:

```text
특정 Domain의 원본 DNS 정보 관리
Zone File 직접 보유
DNS Record 직접 수정 가능
해당 Domain에 대해 Authoritative Answer 제공
Secondary DNS가 존재하면 Zone 정보를 전달 가능
```

즉 Master DNS Server는 해당 Domain의 DNS 정보를 관리하는 원본 서버이다.

---

## 3. Caching DNS와 Master DNS 차이

### Caching DNS Server

자신이 Domain 정보를 직접 관리하지 않고 다른 DNS Server에서 조회한 결과를 Cache에 저장한다.

```text
Client
   |
   v
Caching DNS
   |
   v
외부 DNS Server
```

목적:

```text
DNS 조회 결과 Cache
반복 Query 응답 속도 향상
```

---

### Master DNS Server

특정 Domain 정보를 직접 관리한다.

```text
Client
   |
   v
Master DNS
   |
   v
Zone File
```

예:

```text
www.bangkyu.com
→ 192.168.111.100
```

이 정보 자체를 Master DNS가 가지고 있다.

---

## 4. 이번 실습의 DNS Server

이번 실습에서는 이전에 구축한 Caching DNS Server에 Master DNS 역할을 추가하였다.

```text
DNS Server
192.168.111.100
```

따라서 현재 Server는:

```text
Caching DNS 역할
+
bangkyu.com Master DNS 역할
```

을 함께 수행한다.

---

## 5. Domain과 Zone

### Domain

인터넷에서 사용하는 이름의 전체 구조를 의미한다.

예:

```text
bangkyu.com
www.bangkyu.com
ftp.bangkyu.com
```

---

### Zone

DNS Server가 직접 관리하는 Domain 영역을 의미한다.

예:

```text
bangkyu.com Zone
```

을 등록하면 해당 DNS Server가 다음과 같은 이름을 직접 관리할 수 있다.

```text
bangkyu.com
ns.bangkyu.com
www.bangkyu.com
ftp.bangkyu.com
```

쉽게 표현하면:

```text
Domain
→ 전체 이름 구조

Zone
→ 그 Domain 중에서 현재 DNS Server가 담당하는 영역
```

---

## 6. Zone File

Zone File은 DNS Server가 직접 관리하는 DNS 주소록이다.

예:

```text
www.bangkyu.com → 192.168.111.100
ftp.bangkyu.com → 192.168.111.200
```

와 같은 실제 DNS Record가 저장된다.

이번 실습의 정방향 Zone File:

```text
/var/named/bangkyu.com.db
```

역방향 Zone File:

```text
/var/named/bangkyu.com.rev
```

---

## 7. BIND 주요 구성

DNS Server Software:

```text
BIND
```

Service / Daemon:

```text
named
```

Zone File 기본 저장 Directory:

```text
/var/named/
```

주요 설정:

```text
/etc/named.conf
/etc/named.rfc1912.zones
```

---

## 8. DNS 설정 구조

DNS Server 설정은 크게 두 부분으로 나눌 수 있다.

### Zone 선언

```text
/etc/named.rfc1912.zones
```

예:

```conf
zone "bangkyu.com" IN {
        type master;
        file "bangkyu.com.db";
};
```

의미:

```text
bangkyu.com이라는 Zone을
이 Server가 Master로 관리하며

실제 데이터는
/var/named/bangkyu.com.db
파일에 존재
```

---

### Zone 실제 데이터

```text
/var/named/bangkyu.com.db
```

예:

```dns
www     IN      A       192.168.111.100
ftp     IN      A       192.168.111.200
```

---

## 9. type master

Zone 설정:

```conf
type master;
```

의미:

```text
이 DNS Server가 해당 Zone의
원본 데이터를 직접 관리
```

이번 실습:

```conf
zone "bangkyu.com" IN {
        type master;
        file "bangkyu.com.db";
};
```

따라서 `192.168.111.100` Server가 `bangkyu.com`의 Master DNS Server가 된다.

---

## 10. 정방향 DNS

정방향 DNS는 Domain Name을 IP 주소로 변환한다.

```text
Domain
   ↓
IP Address
```

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

정방향 Zone에서는 주로 A Record 등을 사용한다.

---

## 11. 정방향 Zone 등록

이번 실습에서는:

```bash
vi /etc/named.rfc1912.zones
```

다음을 등록하였다.

```conf
zone "bangkyu.com" IN {
        type master;
        file "bangkyu.com.db";
};
```

---

## 12. 정방향 Zone File

```bash
vi /var/named/bangkyu.com.db
```

실습 설정:

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

---

## 13. $TTL

TTL은 다음의 약자이다.

```text
Time To Live
```

DNS 조회 결과를 다른 DNS Server나 Resolver가 Cache에 얼마나 오래 저장할 수 있는지를 나타낸다.

설정:

```dns
$TTL 1D
```

의미:

```text
1D
→ 1 Day
→ 1일
```

시간 단위 표현:

```text
W → Week
D → Day
H → Hour
M → Minute
```

예:

```text
$TTL 3H
→ 3시간

$TTL 1D
→ 1일
```

---

## 14. @ 기호

Zone File에서:

```text
@
```

는 현재 Zone Domain을 의미한다.

이번 Zone:

```text
bangkyu.com
```

이므로:

```dns
@       IN      A       192.168.111.100
```

은:

```text
bangkyu.com
→ 192.168.111.100
```

이라는 의미이다.

---

## 15. IN

DNS Record에 사용되는:

```text
IN
```

은 Internet Class를 의미한다.

예:

```dns
www     IN      A       192.168.111.100
```

구조:

```text
www
→ Host 이름

IN
→ Internet Class

A
→ Record Type

192.168.111.100
→ 값
```

---

## 16. SOA Record

SOA는 다음의 약자이다.

```text
Start Of Authority
```

Zone의 권한 및 기본 관리 정보를 나타내는 Record이다.

이번 설정:

```dns
@       IN      SOA     ns.bangkyu.com. admin.bangkyu.com. (
                        2026091401
                        1D
                        1H
                        1W
                        3H )
```

주요 정보:

```text
ns.bangkyu.com.
→ Primary Name Server

admin.bangkyu.com.
→ 관리자 정보

2026091401
→ Serial

1D
→ Refresh

1H
→ Retry

1W
→ Expire

3H
→ Minimum
```

---

## 17. SOA의 관리자 표현

일반적인 Email 주소가:

```text
admin@bangkyu.com
```

이라면 SOA에서는 `@` 대신 `.`을 사용하여 표현한다.

```text
admin.bangkyu.com.
```

따라서:

```dns
SOA ns.bangkyu.com. admin.bangkyu.com.
```

처럼 작성하였다.

---

## 18. Serial

Serial은 Zone File의 Version Number이다.

이번 실습:

```text
2026091401
```

형태:

```text
YYYYMMDDNN
```

예:

```text
2026 09 14 01

2026091401
```

같은 날 Zone File을 수정한다면:

```text
2026091402
2026091403
```

처럼 값을 증가시켜 관리할 수 있다.

Secondary DNS Server는 Serial 값을 이용하여 Master DNS의 Zone 데이터가 변경되었는지 판단할 수 있다.

---

## 19. Refresh

```text
Refresh
```

Secondary DNS Server가 Master DNS Server의 Zone 데이터 변경 여부를 확인하는 주기이다.

이번 설정:

```text
1D
```

즉:

```text
1일
```

---

## 20. Retry

Secondary DNS가 Master DNS에 접근하지 못했을 때 다시 연결을 시도할 시간을 나타낸다.

이번 설정:

```text
1H
```

즉:

```text
1시간 후 다시 시도
```

---

## 21. Expire

Secondary DNS가 오랫동안 Master DNS와 통신하지 못했을 때 기존 Zone 정보를 언제까지 신뢰할 것인지 나타낸다.

이번 설정:

```text
1W
```

즉:

```text
1주
```

---

## 22. Minimum

이번 Zone File:

```text
3H
```

Zone의 SOA에 포함되는 Minimum 값을 설정하였다.

```text
3H
→ 3시간
```

---

## 23. NS Record

NS는 Name Server Record이다.

해당 Domain을 관리하는 DNS Server의 이름을 지정한다.

이번 설정:

```dns
@       IN      NS      ns.bangkyu.com.
```

의미:

```text
bangkyu.com의 Name Server
→ ns.bangkyu.com
```

그리고 Name Server 자체의 IP도 A Record로 등록하였다.

```dns
ns      IN      A       192.168.111.100
```

따라서:

```text
ns.bangkyu.com
→ 192.168.111.100
```

---

## 24. Domain 뒤의 점

Zone File에서 완전한 Domain Name을 작성할 때 마지막에 `.`을 붙인다.

예:

```text
ns.bangkyu.com.
```

```text
www.bangkyu.com.
```

마지막 `.`은 Root Domain까지 포함된 완전한 Domain Name이라는 것을 나타낸다.

이번 NS Record:

```dns
@ IN NS ns.bangkyu.com.
```

---

## 25. A Record

A Record는 Domain / Host Name을 IPv4 주소에 연결한다.

형식:

```text
Host명    IN    A    IPv4주소
```

이번 실습:

```dns
@       IN      A       192.168.111.100
ns      IN      A       192.168.111.100
www     IN      A       192.168.111.100
ftp     IN      A       192.168.111.200
```

결과:

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

## 26. AAAA Record

AAAA Record는 Domain Name을 IPv6 주소와 연결한다.

```text
Domain
→ IPv6
```

IPv4에서:

```text
A Record
```

를 사용하는 것과 같은 역할을 IPv6에서는:

```text
AAAA Record
```

가 수행한다.

예:

```dns
www     IN      AAAA    2001:db8::100
```

이번 실습에서는 IPv4 기반이므로 AAAA Record를 별도로 구성하지 않았다.

---

## 27. CNAME Record

CNAME은 Domain Name을 다른 Domain Name에 연결할 때 사용하는 Record이다.

```text
Domain
→ Domain
```

이번 Master DNS 실습에서는 CNAME Record를 추가하지 않았으며 A / NS / SOA / PTR Record를 중심으로 구성하였다.

---

## 28. Zone File Permission

Zone File을 `named`가 읽을 수 있도록 권한을 설정하였다.

정방향:

```bash
chown root:named /var/named/bangkyu.com.db
chmod 640 /var/named/bangkyu.com.db
```

결과:

```text
root:named
640
```

Permission:

```text
owner
rw-

group
r--

other
---
```

따라서:

```text
root
→ 읽기 / 쓰기

named 그룹
→ 읽기

other
→ 접근 불가
```

---

## 29. SELinux Context

Zone File 생성 후 SELinux Context를 기본 정책에 맞게 적용하였다.

```bash
restorecon -v /var/named/bangkyu.com.db
```

역방향 Zone File도:

```bash
restorecon -v /var/named/bangkyu.com.rev
```

로 처리하였다.

---

## 30. named-checkconf

BIND의 설정 파일 문법을 검사한다.

```bash
named-checkconf
```

정상인 경우 별도의 오류 메시지가 출력되지 않는다.

Zone File Load까지 함께 검사:

```bash
named-checkconf -z
```

이번 실습에서는:

```text
zone bangkyu.com/IN: loaded serial 2026091401
```

을 확인하였다.

---

## 31. named-checkzone

특정 Zone File의 문법과 구조를 검사한다.

형식:

```bash
named-checkzone <Zone 이름> <Zone File>
```

이번 정방향 Zone:

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db
```

결과:

```text
zone bangkyu.com/IN: loaded serial 2026091401
OK
```

즉:

```text
SOA
NS
A
```

등 Zone File의 구문이 정상이라는 것을 확인하였다.

---

## 32. Zone File 이름 불일치 오류

실습 중 다음과 같이 Zone을 등록하였다.

```conf
zone "bangkyu.com" IN {
        type master;
        file "soldesk.com.db";
};
```

하지만 실제 생성한 파일은:

```text
/var/named/bangkyu.com.db
```

였다.

따라서:

```bash
named-checkconf -z
```

실행 시:

```text
zone bangkyu.com/IN: loading from master file soldesk.com.db failed: file not found
zone bangkyu.com/IN: not loaded due to errors.
_default/bangkyu.com/IN: file not found
```

오류가 발생하였다.

원인:

```text
Zone 설정의 file 이름
≠
실제 Zone File 이름
```

수정:

```conf
zone "bangkyu.com" IN {
        type master;
        file "bangkyu.com.db";
};
```

이후:

```text
zone bangkyu.com/IN: loaded serial 2026091401
OK
```

로 정상 처리되었다.

---

## 33. Zone 내부 Domain 불일치

Zone File을 만들면서 처음 NS Record가:

```dns
@       IN      NS      ns.soldesk.com.
```

으로 남아 있었다.

현재 Zone은:

```text
bangkyu.com
```

이므로 다음과 같이 수정하였다.

```dns
@       IN      NS      ns.bangkyu.com.
```

Master DNS를 구성할 때는 다음 항목을 동일한 Domain 기준으로 맞춰야 한다.

```text
Zone 이름
Zone File 이름
SOA Name Server
NS Record
Host Record
```

---

## 34. Authoritative DNS

Master DNS Server는 자신이 관리하는 Zone에 대해 권한 있는 응답을 제공한다.

이번 실습:

```bash
dig @192.168.111.100 www.bangkyu.com A +norecurse
```

응답 Flag:

```text
flags: qr aa ra
```

여기서:

```text
aa
```

는:

```text
Authoritative Answer
```

를 의미한다.

즉 `192.168.111.100`이 `bangkyu.com`에 대한 원본 Zone 정보를 직접 가지고 응답했다는 것을 확인할 수 있다.

---

## 35. aa와 ra

이번 응답:

```text
qr aa ra
```

### aa

```text
Authoritative Answer
```

현재 Server가 해당 Zone에 대한 권한 있는 응답을 제공했다는 의미이다.

---

### ra

```text
Recursion Available
```

Recursive Query 기능도 사용할 수 있음을 의미한다.

따라서 이번 Server는:

```text
bangkyu.com
→ Master / Authoritative DNS

그 외 Domain
→ Recursive / Caching DNS
```

역할을 함께 수행하도록 구성되어 있다.

---

## 36. dig로 A Record 확인

```bash
dig @192.168.111.100 bangkyu.com A +noall +answer
```

결과:

```text
bangkyu.com.    86400    IN    A    192.168.111.100
```

---

```bash
dig @192.168.111.100 ns.bangkyu.com A +noall +answer
```

```text
ns.bangkyu.com.    86400    IN    A    192.168.111.100
```

---

```bash
dig @192.168.111.100 www.bangkyu.com A +noall +answer
```

```text
www.bangkyu.com.    86400    IN    A    192.168.111.100
```

---

```bash
dig @192.168.111.100 ftp.bangkyu.com A +noall +answer
```

```text
ftp.bangkyu.com.    86400    IN    A    192.168.111.200
```

---

## 37. NS Record 조회

```bash
dig @192.168.111.100 bangkyu.com NS +noall +answer
```

결과:

```text
bangkyu.com.    86400    IN    NS    ns.bangkyu.com.
```

따라서:

```text
bangkyu.com의 Name Server
→ ns.bangkyu.com
```

을 확인하였다.

---

## 38. SOA Record 조회

```bash
dig @192.168.111.100 bangkyu.com SOA +noall +answer
```

결과:

```text
bangkyu.com. 86400 IN SOA ns.bangkyu.com. admin.bangkyu.com. 2026091401 86400 3600 604800 10800
```

각 값은 Zone File에서 설정한:

```text
2026091401
1D
1H
1W
3H
```

가 초 단위로 표시된 것이다.

---

# 역방향 DNS

## 39. 역방향 DNS란?

정방향 DNS:

```text
Domain
→ IP
```

역방향 DNS:

```text
IP
→ Domain
```

예:

```text
www.bangkyu.com
→ 192.168.111.100
```

반대로:

```text
192.168.111.100
→ www.bangkyu.com
```

---

## 40. PTR Record

역방향 DNS에서 사용하는 주요 Record가 PTR Record이다.

이번 실습:

```dns
100     IN      PTR     www.bangkyu.com.
200     IN      PTR     ftp.bangkyu.com.
```

결과:

```text
192.168.111.100
→ www.bangkyu.com

192.168.111.200
→ ftp.bangkyu.com
```

---

## 41. in-addr.arpa

IPv4 역방향 DNS에서는:

```text
in-addr.arpa
```

Namespace를 사용한다.

현재 Network:

```text
192.168.111.0/24
```

은 역순으로:

```text
111.168.192.in-addr.arpa
```

Zone이 된다.

---

## 42. IP가 역순이 되는 이유

IPv4 주소:

```text
192.168.111.100
```

역방향 DNS Query에서는:

```text
100.111.168.192.in-addr.arpa
```

형태가 된다.

Network Zone은 `/24`이므로:

```text
111.168.192.in-addr.arpa
```

이고 마지막 Octet만 Zone File의 PTR Record에 작성한다.

```text
100
200
```

---

## 43. Reverse Zone 등록

```bash
vi /etc/named.rfc1912.zones
```

추가:

```conf
zone "111.168.192.in-addr.arpa" IN {
        type master;
        file "bangkyu.com.rev";
};
```

---

## 44. Reverse Zone File

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

---

## 45. Reverse Zone 검사

```bash
named-checkzone 111.168.192.in-addr.arpa /var/named/bangkyu.com.rev
```

실제 결과:

```text
zone 111.168.192.in-addr.arpa/IN: loaded serial 2026091401
OK
```

전체 검사:

```bash
named-checkconf -z
```

결과:

```text
zone bangkyu.com/IN: loaded serial 2026091401
zone 111.168.192.in-addr.arpa/IN: loaded serial 2026091401
```

정방향 / 역방향 Zone이 모두 정상적으로 Load되었다.

---

## 46. 역방향 조회 명령어

`dig`:

```bash
dig @192.168.111.100 -x 192.168.111.100
```

또는:

```bash
nslookup 192.168.111.100
```

---

## 47. Client 정방향 조회

Client DNS:

```text
192.168.111.100
```

Client에서:

```bash
nslookup www.bangkyu.com
```

결과:

```text
Server:         192.168.111.100
Address:        192.168.111.100#53

Name:   www.bangkyu.com
Address: 192.168.111.100
```

---

```bash
nslookup ftp.bangkyu.com
```

결과:

```text
Server:         192.168.111.100
Address:        192.168.111.100#53

Name:   ftp.bangkyu.com
Address: 192.168.111.200
```

---

## 48. Client 역방향 조회

```bash
nslookup 192.168.111.100
```

결과:

```text
100.111.168.192.in-addr.arpa
name = www.bangkyu.com.
```

---

```bash
nslookup 192.168.111.200
```

결과:

```text
200.111.168.192.in-addr.arpa
name = ftp.bangkyu.com.
```

---

## 49. 정방향과 역방향 관계

### www

```text
www.bangkyu.com
        |
        | A Record
        v
192.168.111.100
        |
        | PTR Record
        v
www.bangkyu.com
```

### ftp

```text
ftp.bangkyu.com
        |
        | A Record
        v
192.168.111.200
        |
        | PTR Record
        v
ftp.bangkyu.com
```

---

## 50. FCrDNS

역방향 DNS로 확인한 Domain을 다시 정방향 DNS로 조회하여 원래 IP 주소와 일치하는지 확인하는 방식이 있다.

```text
Forward-confirmed Reverse DNS
FCrDNS
```

개념:

```text
Domain
→ IP

IP
→ Domain

두 결과가 서로 일치하는지 확인
```

예:

```text
www.bangkyu.com
→ 192.168.111.100

192.168.111.100
→ www.bangkyu.com
```

메일 서버의 신뢰성 확인이나 시스템 식별 등에 역방향 DNS가 사용될 수 있다.

---

## 51. 역방향 DNS가 유용한 경우

역방향 DNS를 사용하면 IP 주소를 Host / Domain Name으로 확인할 수 있다.

예:

```text
192.168.111.200
→ ftp.bangkyu.com
```

단순 IP만 확인하는 것보다 어떤 시스템인지 파악하기 쉬워진다.

따라서 다음과 같은 상황에서 활용할 수 있다.

```text
메일 서버 확인
서버 식별
Log 분석
보안 이벤트 분석
장애 분석
```

---

## 52. Zone File 변경 시 주의점

Zone File을 수정했다면 다음을 확인한다.

```text
1. Serial 증가

2. Zone File 문법 확인

3. BIND 전체 설정 확인

4. named에 변경 설정 적용

5. 실제 DNS Query 검증
```

예:

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db
```

```bash
named-checkconf -z
```

```bash
systemctl restart named
```

---

## 53. Master DNS 설정 확인 순서

```text
1. BIND / named 상태 확인

2. Zone 선언
   /etc/named.rfc1912.zones

3. 정방향 Zone File 생성
   /var/named/bangkyu.com.db

4. SOA / NS / A Record 작성

5. Permission 설정

6. named-checkzone 검사

7. named-checkconf -z 검사

8. named 재시작

9. A / NS / SOA 조회

10. aa Flag 확인

11. Reverse Zone 선언

12. Reverse Zone File 생성

13. PTR Record 작성

14. Reverse Zone 검사

15. named 재시작

16. Client 정방향 조회

17. Client 역방향 조회
```

---

## 54. 주요 파일 정리

```text
/etc/named.conf
→ BIND 전체 설정

/etc/named.rfc1912.zones
→ 관리할 Zone 선언

/var/named/bangkyu.com.db
→ bangkyu.com 정방향 Zone File

/var/named/bangkyu.com.rev
→ 192.168.111.0/24 역방향 Zone File
```

---

## 55. 주요 명령어 정리

### named 상태

```bash
systemctl status named
```

### BIND 설정 검사

```bash
named-checkconf
```

### 전체 Zone 검사

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

### DNS Service 재시작

```bash
systemctl restart named
```

### A Record 확인

```bash
dig @192.168.111.100 www.bangkyu.com A
```

### NS Record 확인

```bash
dig @192.168.111.100 bangkyu.com NS
```

### SOA Record 확인

```bash
dig @192.168.111.100 bangkyu.com SOA
```

### Authoritative Answer 확인

```bash
dig @192.168.111.100 www.bangkyu.com A +norecurse
```

### Reverse DNS

```bash
dig @192.168.111.100 -x 192.168.111.100
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

## 56. 주요 DNS Record 정리

| Record | 역할 |
|---|---|
| SOA | Zone의 권한 및 기본 관리 정보 |
| NS | Domain을 관리하는 Name Server 지정 |
| A | Domain / Host Name → IPv4 |
| AAAA | Domain / Host Name → IPv6 |
| CNAME | Domain Name → 다른 Domain Name |
| PTR | IPv4 → Domain Name |

이번 실습에서 직접 사용한 Record:

```text
SOA
NS
A
PTR
```

---

## 57. 이번 실습 최종 구성

```text
Master DNS Server
192.168.111.100

Domain
bangkyu.com
```

정방향:

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

역방향:

```text
192.168.111.100
→ www.bangkyu.com

192.168.111.200
→ ftp.bangkyu.com
```

Zone:

```text
Forward Zone
bangkyu.com

Reverse Zone
111.168.192.in-addr.arpa
```

Zone File:

```text
/var/named/bangkyu.com.db
/var/named/bangkyu.com.rev
```

---

## 58. 핵심 정리

```text
Master DNS
→ 특정 Domain의 원본 DNS 정보를 직접 관리

Zone
→ DNS Server가 담당하는 Domain 영역

Zone File
→ 실제 DNS Record가 저장된 파일

type master
→ 해당 Server가 Zone 원본 관리

정방향 DNS
→ Domain → IP

역방향 DNS
→ IP → Domain

SOA
→ Zone 권한 / 관리 정보

NS
→ Domain의 Name Server

A
→ Host → IPv4

PTR
→ IPv4 → Domain

Zone 설정
→ /etc/named.rfc1912.zones

정방향 Zone File
→ /var/named/bangkyu.com.db

역방향 Zone File
→ /var/named/bangkyu.com.rev

정방향 Zone
→ bangkyu.com

역방향 Zone
→ 111.168.192.in-addr.arpa

설정 검사
→ named-checkconf

Zone 검사
→ named-checkzone

Authoritative 확인
→ aa Flag
```

---

## 59. 이번 실습 핵심 흐름

```text
Client
   |
   | www.bangkyu.com?
   v
192.168.111.100
Master DNS
   |
   | /var/named/bangkyu.com.db 확인
   v
192.168.111.100 응답
```

역방향:

```text
Client
   |
   | 192.168.111.200의 이름?
   v
192.168.111.100
Master DNS
   |
   | /var/named/bangkyu.com.rev 확인
   v
ftp.bangkyu.com 응답
```

이번 실습을 통해 BIND 기반 Master DNS Server에서 Zone을 직접 생성하고 관리하며, SOA / NS / A / PTR Record를 이용하여 정방향 및 역방향 DNS를 구성하는 과정을 확인하였다.

또한 `aa` Flag를 통해 해당 DNS Server가 `bangkyu.com`에 대한 Authoritative DNS Server로 동작하는 것을 검증하였다.
