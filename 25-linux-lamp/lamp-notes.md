# LAMP Stack Notes

Rocky Linux 환경에서 Apache, PHP-FPM, MariaDB를 이용한 LAMP Stack의 구조와 동작 원리, Database 연동, DNS/VirtualHost 구성, 보안 설정 및 Troubleshooting 방법을 정리한다.

이번 실습에서는 단순히 Package를 설치하는 것에서 끝내지 않고 다음 전체 흐름을 직접 구성하였다.

```text
Client
  ↓
DNS
  ↓
Apache
  ↓
PHP-FPM
  ↓
PHP
  ↓
MariaDB
  ↓
Database 조회
  ↓
HTML 응답
```

---

# 1. LAMP란?

LAMP는 전통적인 Linux Web Application 환경을 구성하는 Software Stack이다.

```text
L = Linux
A = Apache
M = MariaDB / MySQL
P = PHP
```

각 구성 요소는 다음 역할을 담당한다.

```text
Linux
→ Server OS

Apache
→ HTTP 요청 처리

PHP
→ 동적 Web Application 실행

MariaDB
→ 데이터 저장 및 조회
```

---

# 2. Static Web과 Dynamic Web

정적인 HTML File만 제공하는 경우:

```text
Client
  ↓
Apache
  ↓
index.html
  ↓
Client
```

Apache가 File을 그대로 읽어서 반환하면 된다.

하지만 PHP와 Database를 사용하는 동적 Web Service는 구조가 다르다.

```text
Client
  ↓
Apache
  ↓
index.php
  ↓
PHP-FPM
  ↓
PHP Code 실행
  ↓
MariaDB Query
  ↓
결과 처리
  ↓
HTML 생성
  ↓
Client
```

즉 PHP File 자체가 Client에게 전달되는 것이 아니라 Server에서 실행된 결과가 전달된다.

---

# 3. LAMP 요청 전체 흐름

이번 실습 환경을 기준으로 보면 다음과 같다.

```text
Client-L
192.168.111.150
        ↓
lamp.bangkyu.com
        ↓
DNS Query
        ↓
192.168.111.100
        ↓
Firewalld
        ↓
Apache :80
        ↓
VirtualHost
lamp.bangkyu.com
        ↓
/var/www/lamp/index.php
        ↓
proxy_fcgi
        ↓
PHP-FPM
        ↓
PDO MySQL
        ↓
MariaDB
        ↓
lampdb.visitors
        ↓
PHP가 HTML 생성
        ↓
Apache
        ↓
Client-L
```

---

# 4. Apache의 역할

Apache HTTP Server는 Client의 HTTP Request를 수신한다.

Service 확인:

```bash
systemctl status httpd
```

간단히 확인:

```bash
systemctl is-active httpd
systemctl is-enabled httpd
```

Listen Port:

```bash
ss -lntp | grep httpd
```

일반적인 HTTP:

```text
TCP 80
```

HTTPS:

```text
TCP 443
```

---

# 5. Apache는 PHP를 직접 실행하는가?

구성 방식에 따라 다르다.

대표적인 방식은 다음과 같다.

```text
mod_php 방식

Apache
  ↓
Apache Process 내부에서 PHP 실행
```

또는:

```text
PHP-FPM 방식

Apache
  ↓
FastCGI
  ↓
PHP-FPM
  ↓
PHP 실행
```

이번 Rocky Linux 환경에서는 PHP-FPM 방식을 사용하였다.

---

# 6. 실제 PHP 처리 구조

Apache Module 확인:

```bash
httpd -M | grep -E 'php|proxy_fcgi|mpm'
```

실습 환경에서는:

```text
mpm_event_module
proxy_fcgi_module
```

가 확인되었다.

PHP 설정:

```apache
<FilesMatch \.(php|phar)$>
    SetHandler "proxy:unix:/run/php-fpm/www.sock|fcgi://localhost"
</FilesMatch>
```

따라서 구조는:

```text
Apache
  ↓
proxy_fcgi
  ↓
/run/php-fpm/www.sock
  ↓
PHP-FPM
```

이다.

---

# 7. PHP-FPM이란?

PHP-FPM은:

```text
PHP FastCGI Process Manager
```

의 약자이다.

PHP 실행 Process를 별도로 관리한다.

Apache는 PHP File 요청이 들어오면 직접 PHP를 실행하지 않고 PHP-FPM에 처리를 요청한다.

```text
Apache
→ Web Server

PHP-FPM
→ PHP 실행 Process 관리자
```

---

# 8. PHP-FPM Service

확인:

```bash
systemctl status php-fpm
```

실습에서는 다음과 같이 활성화하였다.

```bash
systemctl enable --now php-fpm
```

확인:

```bash
systemctl is-active php-fpm
systemctl is-enabled php-fpm
```

---

# 9. PHP-FPM Unix Socket

실습 환경:

```text
/run/php-fpm/www.sock
```

확인:

```bash
ls -l /run/php-fpm/www.sock
```

Unix Socket은 동일 Linux System 내부의 Process끼리 통신하는 방법 중 하나이다.

```text
Apache
   │
   │ Unix Socket
   ↓
PHP-FPM
```

Network TCP Port를 통하지 않고 Local IPC 방식으로 통신할 수 있다.

---

# 10. Apache MPM

MPM은:

```text
Multi-Processing Module
```

Apache가 Connection과 Process/Thread를 어떻게 처리할지 결정한다.

대표적으로:

```text
prefork
worker
event
```

가 있다.

실습 환경에서는:

```text
mpm_event_module
```

이 사용되었다.

---

# 11. event MPM

event MPM은 Connection을 효율적으로 처리하기 위한 Apache MPM이다.

PHP-FPM처럼 PHP 실행을 별도 Process로 분리하는 구조와 함께 사용하기 좋다.

구조:

```text
Apache event MPM
       ↓
HTTP Connection 처리
       ↓
PHP 요청
       ↓
proxy_fcgi
       ↓
PHP-FPM
```

---

# 12. PHP CLI와 Web PHP 차이

다음 명령:

```bash
php -v
```

또는:

```bash
php script.php
```

는 CLI 환경에서 PHP를 실행한다.

하지만 Web Browser에서:

```text
http://server/index.php
```

를 요청하는 것은:

```text
Apache
→ PHP-FPM
→ PHP
```

경로를 이용한다.

따라서:

```text
php CLI 정상
```

이라고 해서 Web PHP까지 반드시 정상이라는 뜻은 아니다.

---

# 13. PHP Module 확인

PHP와 MariaDB를 연동하려면 MySQL/MariaDB Driver가 필요하다.

확인:

```bash
php -m | grep -Ei 'mysqli|pdo_mysql'
```

실습 환경:

```text
mysqli
pdo_mysql
```

---

# 14. mysqli와 PDO

PHP에서 MySQL/MariaDB에 접근하는 대표적인 방법이다.

## mysqli

```text
MySQL Improved Extension
```

MySQL 계열 Database에 특화되어 있다.

## PDO

```text
PHP Data Objects
```

다양한 Database에 공통적인 방식으로 접근할 수 있는 Interface이다.

이번 실습에서는 PDO를 사용하였다.

---

# 15. PDO 연결 구조

예:

```php
$pdo = new PDO(
    "mysql:host={$host};dbname={$dbname};charset=utf8mb4",
    $username,
    $password
);
```

구조:

```text
PHP
  ↓
PDO
  ↓
pdo_mysql
  ↓
MariaDB
```

---

# 16. PDO Exception 처리

Database 연결 실패 시 오류를 처리할 수 있다.

예:

```php
try {

    // DB 연결

} catch (PDOException $e) {

    error_log($e->getMessage());

    http_response_code(500);

    die('Database connection failed');
}
```

사용자에게 Database Password나 내부 Query 오류를 그대로 보여주지 않는 것이 중요하다.

---

# 17. Database 오류를 그대로 출력하면 안 되는 이유

다음과 같은 정보를 Client에게 그대로 보여주면 보안상 좋지 않다.

```text
DB Host
DB User
Database Name
File Path
SQL Query
Stack Trace
```

따라서:

```text
사용자
→ 일반적인 오류 Message

Server Log
→ 상세 오류
```

로 분리하는 것이 좋다.

---

# 18. MariaDB란?

MariaDB는 Relational Database Management System이다.

확인:

```bash
mariadb --version
```

Service:

```bash
systemctl status mariadb
```

기본 TCP Port:

```text
3306/tcp
```

---

# 19. Database 기본 구조

MariaDB 내부를 단순화하면:

```text
MariaDB Server
   │
   ├── Database A
   │      ├── Table A
   │      └── Table B
   │
   └── Database B
          └── Table C
```

이번 실습:

```text
MariaDB
   ↓
lampdb
   ↓
visitors
```

---

# 20. Database와 Table

Database:

```sql
CREATE DATABASE lampdb;
```

Table:

```sql
CREATE TABLE visitors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    message VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

관계:

```text
lampdb
  ↓
visitors
  ↓
Row
```

---

# 21. visitors Table 구조

```text
id
→ 각 Row를 구분

name
→ 사용자 이름

message
→ Message

created_at
→ 생성 시각
```

`AUTO_INCREMENT`:

```text
id 값을 자동으로 증가
```

`PRIMARY KEY`:

```text
각 Row를 유일하게 식별
```

---

# 22. Application 전용 DB User

PHP Application에서 MariaDB의 `root` 계정을 사용하지 않았다.

전용 계정:

```text
lampuser@localhost
```

를 생성하였다.

왜 필요한가?

```text
DB root
→ 거의 모든 Database 관리 가능

Application User
→ Application에 필요한 작업만 수행
```

공격이나 Application 취약점 발생 시 피해 범위를 줄일 수 있다.

---

# 23. 최소 권한 원칙

Application에는 필요한 권한만 부여한다.

이번 실습:

```sql
GRANT SELECT ON lampdb.* TO 'lampuser'@'localhost';
```

즉:

```text
lampdb의 데이터 조회
→ 가능

다른 Database 관리
→ 불필요

User 생성
→ 불필요

Server 전체 관리
→ 불필요
```

이 원칙을:

```text
Least Privilege
최소 권한 원칙
```

이라고 한다.

---

# 24. SHOW GRANTS

DB User가 어떤 권한을 가지고 있는지 확인:

```sql
SHOW GRANTS FOR 'lampuser'@'localhost';
```

실습에서는:

```text
GRANT SELECT ON `lampdb`.* TO `lampuser`@`localhost`
```

가 확인되었다.

Application 장애나 권한 문제를 분석할 때 매우 중요한 명령이다.

---

# 25. MariaDB User의 Host 부분

MariaDB 계정은 단순히 User Name만으로 구분하지 않는다.

```text
'user'@'host'
```

형태이다.

예:

```text
'lampuser'@'localhost'
'lampuser'@'192.168.111.%'
'lampuser'@'%'
```

은 서로 다른 계정으로 취급될 수 있다.

---

# 26. localhost 계정의 의미

```text
'lampuser'@'localhost'
```

는 Local Server에서 접속하는 Application용 계정으로 구성할 수 있다.

이번 구성은:

```text
Apache
PHP-FPM
MariaDB
```

가 모두 Server-A에 존재하므로 Local Application 계정이 적합하다.

---

# 27. localhost와 127.0.0.1 차이

MySQL/MariaDB Client 계열에서는:

```text
localhost
```

와:

```text
127.0.0.1
```

의 동작이 다를 수 있다.

일반적으로:

```text
localhost
→ Unix Socket 사용 가능

127.0.0.1
→ TCP 사용
```

이번 PHP 설정에서도:

```php
'host' => 'localhost'
```

를 사용하였다.

---

# 28. MariaDB Unix Socket

실습 환경에서:

```bash
mariadb -e "SHOW VARIABLES LIKE 'socket';"
```

결과:

```text
/var/lib/mysql/mysql.sock
```

PHP에서도:

```text
mysqli.default_socket
pdo_mysql.default_socket
```

이 같은 경로를 사용하였다.

즉 Local PHP Application은 Network TCP 3306을 사용하지 않고 Unix Socket을 통해 MariaDB와 통신할 수 있다.

---

# 29. PHP ↔ MariaDB Local 구조

```text
PHP-FPM
   ↓
PDO MySQL
   ↓
/var/lib/mysql/mysql.sock
   ↓
MariaDB
```

이 경우 외부 Network를 거치지 않는다.

---

# 30. MariaDB bind-address

초기 MariaDB는:

```text
*:3306
```

으로 Listen하였다.

이는 모든 Interface에 Bind된 상태를 의미한다.

실습 후:

```ini
bind-address=127.0.0.1
```

을 설정하였다.

결과:

```text
127.0.0.1:3306
```

만 Listen하였다.

---

# 31. 0.0.0.0과 127.0.0.1

```text
0.0.0.0:3306
또는
*:3306
```

의미:

```text
모든 IPv4 Interface에서 Connection 수신 가능
```

반면:

```text
127.0.0.1:3306
```

은:

```text
Local Host에서만 TCP Connection 가능
```

이다.

---

# 32. Bind Address와 Firewalld는 다르다

두 기능은 역할이 다르다.

```text
bind-address
→ Application이 어느 Interface에 Listen할지 결정

Firewalld
→ 외부 Packet을 허용할지 결정
```

예:

```text
MariaDB
*:3306 LISTEN

Firewalld
3306 차단
```

이라면 MariaDB는 외부 Interface에서도 Listen하지만 Firewall에서 접근을 막을 수 있다.

더 안전한 구조는 필요에 따라:

```text
MariaDB
127.0.0.1:3306

+

Firewalld
3306 미허용
```

처럼 여러 계층에서 제한하는 것이다.

---

# 33. Defense in Depth

이번 구성에서는 MariaDB 외부 접근을 두 단계로 제한하였다.

```text
1. bind-address=127.0.0.1

2. Firewalld에서 3306/mysql 미허용
```

이처럼 여러 보안 계층을 적용하는 것을:

```text
Defense in Depth
다중 계층 방어
```

라고 볼 수 있다.

---

# 34. Client에서 3306 접근 검증

Client에서 TCP Connection을 확인:

```bash
timeout 3 bash -c '</dev/tcp/192.168.111.100/3306' \
&& echo CONNECTED || echo BLOCKED
```

실습 결과:

```text
BLOCKED
```

반면 Web Application은 정상 동작하였다.

즉:

```text
외부 DB 직접 접근
→ 차단

Web Application을 통한 DB 사용
→ 정상
```

상태이다.

---

# 35. Apache VirtualHost

하나의 Apache Server에서 여러 Domain을 운영할 수 있다.

예:

```text
www.bangkyu.com
site1.bangkyu.com
site2.bangkyu.com
lamp.bangkyu.com
```

모두 동일 IP를 사용할 수 있다.

Apache는 HTTP `Host` Header를 보고 어떤 VirtualHost를 사용할지 결정한다.

---

# 36. LAMP VirtualHost

실습 설정:

```apache
<VirtualHost *:80>
    ServerName lamp.bangkyu.com
    DocumentRoot /var/www/lamp

    <Directory "/var/www/lamp">
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/lamp-error.log
    CustomLog /var/log/httpd/lamp-access.log combined
</VirtualHost>
```

---

# 37. ServerName

```apache
ServerName lamp.bangkyu.com
```

Apache가 해당 VirtualHost를 선택할 때 사용하는 Domain Name이다.

---

# 38. DocumentRoot

```apache
DocumentRoot /var/www/lamp
```

해당 VirtualHost에서 제공할 Web File의 기본 Directory이다.

이번 구성:

```text
lamp.bangkyu.com
        ↓
/var/www/lamp
        ↓
index.php
```

---

# 39. Directory Require

```apache
<Directory "/var/www/lamp">
    Require all granted
</Directory>
```

해당 Directory에 HTTP 접근을 허용한다.

Apache 2.4의 Access Control Directive이다.

---

# 40. Apache 설정 검사

설정 변경 후:

```bash
httpd -t
```

를 먼저 실행하는 습관이 중요하다.

정상:

```text
Syntax OK
```

하지만:

```text
Syntax OK
```

는 Apache 설정 문법만 정상이라는 의미이다.

다음을 보장하지는 않는다.

```text
PHP 정상
Database 정상
Permission 정상
SELinux 정상
DNS 정상
```

---

# 41. httpd -S

VirtualHost 구조 확인:

```bash
httpd -S
```

다음과 같은 내용을 확인할 수 있다.

```text
어떤 Port에 VirtualHost가 있는가?

기본 VirtualHost는 무엇인가?

ServerName은 무엇인가?

어떤 설정 File에서 정의되었는가?
```

VirtualHost Troubleshooting에 매우 유용하다.

---

# 42. DNS와 Apache VirtualHost 관계

DNS:

```text
lamp.bangkyu.com
→ 192.168.111.100
```

은 Client가 어느 IP로 접속해야 하는지 알려준다.

Apache:

```text
Host: lamp.bangkyu.com
```

을 보고 어떤 VirtualHost를 사용할지 결정한다.

즉:

```text
DNS
→ Server 위치 찾기

Apache VirtualHost
→ Server 안에서 Site 선택
```

이다.

---

# 43. DNS A Record

이번 실습:

```text
lamp    IN    A    192.168.111.100
```

즉:

```text
lamp.bangkyu.com
→ 192.168.111.100
```

이다.

---

# 44. DNS Serial

Master DNS Zone File을 수정하면 SOA Serial을 증가시킨다.

실습:

```text
2026091402
→
2026091501
```

Serial은 Zone Data의 Version처럼 사용된다.

Slave DNS가 존재하는 환경에서는 Zone 변경 여부 판단에 특히 중요하다.

---

# 45. DNS 설정 검증

Zone File:

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db
```

전체 BIND 설정:

```bash
named-checkconf
```

DNS 확인:

```bash
dig lamp.bangkyu.com +short
```

---

# 46. DNS 없이 VirtualHost 테스트

DNS를 등록하기 전에도 Host Header를 직접 넣을 수 있다.

```bash
curl -H 'Host: lamp.bangkyu.com' http://127.0.0.1/
```

이 방식은 다음을 분리해서 테스트할 수 있다.

```text
DNS 문제인지?

Apache VirtualHost 문제인지?
```

매우 유용한 Troubleshooting 방법이다.

---

# 47. HTTP 200

정상 Web 요청:

```text
HTTP/1.1 200 OK
```

이번 실습에서는:

```text
DNS
Apache
PHP
MariaDB
```

전체 연결 후 200을 확인하였다.

---

# 48. HTTP 500

```text
500 Internal Server Error
```

는 Server 내부에서 요청 처리 중 문제가 발생했다는 의미이다.

원인은 매우 다양하다.

```text
PHP Syntax Error
PHP Runtime Error
Permission
DB Connection
잘못된 require/include
SELinux
Application Logic
```

따라서:

```text
500 = Database 문제
```

처럼 바로 단정하면 안 된다.

---

# 49. 이번 실습의 HTTP 500 장애

Database Password를 PHP Main Source에서 분리하기 위해:

```text
/var/www/lamp-private/db-config.php
```

를 생성하였다.

구조:

```text
/var/www/lamp/index.php
       ↓
require
       ↓
/var/www/lamp-private/db-config.php
```

설정 분리 직후 HTTP 500이 발생하였다.

---

# 50. PHP 문법 검사의 의미

확인:

```bash
php -l /var/www/lamp/index.php
php -l /var/www/lamp-private/db-config.php
```

결과:

```text
No syntax errors detected
```

따라서 PHP Syntax 자체가 원인이 아니라는 것을 알 수 있었다.

이처럼 장애 분석에서는 하나씩 원인을 제거해 나가는 것이 중요하다.

---

# 51. Directory Execute Permission

Linux Directory에서 `x` Permission은 단순히 Program 실행 의미가 아니다.

Directory에서:

```text
x
→ 해당 Directory를 Traverse할 수 있는 권한
```

이다.

즉 File에 Read 권한이 있어도 상위 Directory를 통과할 수 없다면 File을 읽지 못할 수 있다.

---

# 52. HTTP 500 발생 당시 구조

문제 당시:

```text
/var/www/lamp-private
root:root 750
```

File:

```text
db-config.php
root:apache 640
```

였다.

PHP-FPM Process가 `apache` 권한으로 File을 읽으려 해도:

```text
apache
  ↓
/var/www/lamp-private
  ↓
Group = root
  ↓
Directory Traverse 불가
```

상태가 되었다.

---

# 53. HTTP 500 해결

Directory Group 변경:

```bash
chown root:apache /var/www/lamp-private
chmod 750 /var/www/lamp-private
```

File:

```bash
chown root:apache /var/www/lamp-private/db-config.php
chmod 640 /var/www/lamp-private/db-config.php
```

최종 구조:

```text
Directory
root:apache 750

File
root:apache 640
```

---

# 54. 750 Permission

```text
750
```

을 나누면:

```text
Owner
7 = rwx

Group
5 = r-x

Other
0 = ---
```

Directory의 경우 Group `apache`가:

```text
r
→ Directory Entry 확인 가능

x
→ Directory Traverse 가능
```

상태가 된다.

---

# 55. 640 Permission

```text
640
```

File Permission:

```text
Owner
6 = rw-

Group
4 = r--

Other
0 = ---
```

따라서:

```text
root
→ 읽기/수정

apache
→ 읽기

other
→ 접근 불가
```

구조가 된다.

---

# 56. Secret을 DocumentRoot 밖에 두는 이유

DocumentRoot:

```text
/var/www/lamp
```

는 Web Content 제공 대상이다.

DB 인증정보 File:

```text
/var/www/lamp-private/db-config.php
```

를 DocumentRoot 밖에 두면 Web Server 설정 오류가 발생하더라도 직접 HTTP 경로로 노출될 가능성을 줄일 수 있다.

구조:

```text
/var/www/lamp
→ 공개 Web Content

/var/www/lamp-private
→ Application 내부 설정
```

---

# 57. Password를 Git에 저장하면 안 되는 이유

Git Repository에 Password를 Commit하면 나중에 File을 삭제해도 Git History에 Secret이 남을 수 있다.

따라서 다음을 Commit하지 않는 것이 중요하다.

```text
DB Password
API Key
Private Key
Token
Cloud Credential
```

---

# 58. 설정 File 관리 방식

실제 Secret:

```text
db-config.php
```

Git에는 넣지 않고, 대신 예제 File을 사용할 수 있다.

예:

```text
db-config.example.php
```

내용:

```php
<?php

return [
    'host' => 'localhost',
    'dbname' => 'lampdb',
    'username' => 'lampuser',
    'password' => '<DB_PASSWORD>'
];
```

그리고 실제 File은 `.gitignore` 대상에 둘 수 있다.

---

# 59. .gitignore 예

Application Source가 Git Repository 안에 있을 경우:

```gitignore
db-config.php
.env
```

등을 사용할 수 있다.

단 이미 Commit한 Secret은 `.gitignore`만 추가한다고 Git History에서 사라지는 것이 아니다.

---

# 60. 환경 변수 방식

실제 Application에서는 Secret을 File에 직접 작성하는 방식 대신 환경 변수를 사용할 수도 있다.

개념:

```text
DB_HOST
DB_NAME
DB_USER
DB_PASSWORD
```

PHP:

```php
$password = getenv('DB_PASSWORD');
```

환경과 Application 구조에 따라 Secret 관리 방식을 선택한다.

---

# 61. SELinux Context

Web Content 확인:

```bash
ls -Z /var/www/lamp/index.php
```

실습에서는:

```text
httpd_sys_content_t
```

가 적용되었다.

이 Type은 Apache가 일반적인 Web Content로 읽을 수 있는 대표적인 SELinux Type이다.

---

# 62. restorecon

SELinux Context를 정책에 맞게 적용:

```bash
restorecon -Rv /var/www/lamp
```

일반 Permission만 맞는다고 Apache 접근이 항상 가능한 것은 아니다.

```text
Linux Permission
+
SELinux Policy
```

둘 다 확인해야 한다.

---

# 63. httpd_can_network_connect_db

SELinux Boolean:

```bash
getsebool httpd_can_network_connect_db
```

실습 이전 확인에서는:

```text
off
```

상태였다.

그럼에도 이번 PHP → MariaDB 연결은 정상 동작하였다.

---

# 64. 왜 DB Boolean이 off인데 연결되었는가?

이번 구성에서는 PHP에서:

```text
localhost
```

를 사용하였고 PHP와 MariaDB가 동일 Server에 있었다.

또한 PHP와 MariaDB의 Socket 경로가:

```text
/var/lib/mysql/mysql.sock
```

로 일치하였다.

즉 이번 Application은 Local Unix Socket을 사용할 수 있었기 때문에 외부 TCP Database Connection과는 구조가 다르다.

개념적으로:

```text
외부 DB Server TCP Connection
→ Network 관련 SELinux 정책 영향 가능

Local Unix Socket
→ Local Socket 정책 사용
```

이라고 구분할 수 있다.

---

# 65. Remote Database를 사용할 경우

예를 들어:

```text
Web Server
192.168.111.100

DB Server
192.168.111.200
```

처럼 분리한다면 PHP가 Network를 통해 DB Server에 연결해야 한다.

```text
PHP/httpd
   ↓ TCP 3306
Remote MariaDB
```

이때는 다음을 추가로 확인해야 할 수 있다.

```text
Routing
Firewalld
MariaDB bind-address
DB User Host
SELinux Boolean
```

즉 Local LAMP와 Remote DB 구조는 Troubleshooting 범위가 다르다.

---

# 66. Firewalld에서 HTTP

확인:

```bash
firewall-cmd --query-service=http
```

실습 최종 상태:

```text
yes
```

따라서 외부 Client의 HTTP 접근이 허용되었다.

---

# 67. Firewalld에서 MariaDB

확인:

```bash
firewall-cmd --query-port=3306/tcp
firewall-cmd --query-service=mysql
```

실습 최종 상태:

```text
no
no
```

DB가 Local Application 전용이므로 외부에 MariaDB Port를 열지 않았다.

---

# 68. Application Port를 무조건 열면 안 되는 이유

Application이 사용한다고 해서 모든 Port를 Client에 공개할 필요는 없다.

이번 구조:

```text
Client가 필요한 Port
→ HTTP 80

Application 내부에서만 필요한 Port
→ MariaDB 3306
```

따라서:

```text
80 공개
3306 비공개
```

로 구성할 수 있다.

---

# 69. Apache Access Log

VirtualHost별 Access Log:

```text
/var/log/httpd/lamp-access.log
```

설정:

```apache
CustomLog /var/log/httpd/lamp-access.log combined
```

확인:

```bash
tail /var/log/httpd/lamp-access.log
```

실제 요청의:

```text
Client IP
시간
HTTP Method
URL
Status Code
Response Size
```

등을 확인할 수 있다.

---

# 70. 장애 전후 Access Log

이번 실습에서 실제로:

```text
500
500
500
200
200
```

흐름이 확인되었다.

즉:

```text
설정 변경
   ↓
장애 발생
   ↓
HTTP 500
   ↓
Permission 수정
   ↓
HTTP 200
```

을 Log로 증명할 수 있었다.

---

# 71. Error Log

VirtualHost Error Log:

```text
/var/log/httpd/lamp-error.log
```

설정:

```apache
ErrorLog /var/log/httpd/lamp-error.log
```

Application 장애 시 Access Log에서 500을 확인한 뒤 Error Log와 PHP-FPM Log를 함께 확인한다.

---

# 72. PHP-FPM Log 확인

Service Log:

```bash
journalctl -u php-fpm
```

최근 Log:

```bash
journalctl -u php-fpm --since "10 minutes ago"
```

PHP가 실행되지 않거나 PHP-FPM Service 문제가 있을 경우 확인한다.

---

# 73. MariaDB Log

실습 설정:

```text
/var/log/mariadb/mariadb.log
```

Service Log:

```bash
journalctl -u mariadb
```

Database Service 기동 실패나 설정 오류 등을 확인할 수 있다.

---

# 74. LAMP Troubleshooting 1단계 - DNS

```bash
dig lamp.bangkyu.com +short
```

정상:

```text
192.168.111.100
```

잘못된 IP가 나온다면 Apache보다 먼저 DNS를 확인해야 한다.

---

# 75. LAMP Troubleshooting 2단계 - Network

Server 접근:

```bash
ping 192.168.111.100
```

Routing:

```bash
ip route
```

Firewall:

```bash
firewall-cmd --list-all
```

---

# 76. LAMP Troubleshooting 3단계 - Apache

```bash
systemctl is-active httpd
```

설정:

```bash
httpd -t
```

VirtualHost:

```bash
httpd -S
```

Port:

```bash
ss -lntp | grep ':80'
```

---

# 77. LAMP Troubleshooting 4단계 - HTTP

Server Local:

```bash
curl -I http://127.0.0.1/
```

VirtualHost 직접 지정:

```bash
curl -H 'Host: lamp.bangkyu.com' http://127.0.0.1/
```

Client:

```bash
curl -i http://lamp.bangkyu.com/
```

---

# 78. LAMP Troubleshooting 5단계 - PHP-FPM

```bash
systemctl is-active php-fpm
```

Socket:

```bash
ls -l /run/php-fpm/www.sock
```

PHP Module:

```bash
php -m
```

PHP Syntax:

```bash
php -l file.php
```

---

# 79. LAMP Troubleshooting 6단계 - Database

MariaDB:

```bash
systemctl is-active mariadb
```

Listen:

```bash
ss -lntp | grep ':3306'
```

Database 조회:

```bash
mariadb -e "SHOW DATABASES;"
```

Application User 접속:

```bash
mariadb -u lampuser -p lampdb
```

---

# 80. LAMP Troubleshooting 7단계 - DB 권한

```sql
SHOW GRANTS FOR 'lampuser'@'localhost';
```

다음과 같은 오류가 있다면 User/Host/권한을 확인한다.

```text
Access denied for user
```

---

# 81. LAMP Troubleshooting 8단계 - File Permission

```bash
ls -ld /path/to/directory
ls -l /path/to/file
```

File만 보지 말고 상위 Directory Permission까지 확인해야 한다.

필요하면:

```bash
namei -l /var/www/lamp-private/db-config.php
```

처럼 경로 전체 Permission을 단계별로 확인하는 방법도 있다.

---

# 82. LAMP Troubleshooting 9단계 - SELinux

```bash
getenforce
ls -Zd /var/www/lamp
ls -Z /var/www/lamp/index.php
```

AVC:

```bash
ausearch -m AVC -ts recent
```

Permission이 정상인데 Access Denial이 발생하면 SELinux도 확인한다.

---

# 83. LAMP Troubleshooting 10단계 - Log

```text
DNS 정상?
↓
Network 정상?
↓
Apache 정상?
↓
PHP-FPM 정상?
↓
PHP Syntax 정상?
↓
DB 정상?
↓
Permission 정상?
↓
SELinux 정상?
↓
Log 확인
```

실제 Log를 근거로 장애 지점을 좁히는 것이 중요하다.

---

# 84. 대표 장애 Case - DNS 실패

증상:

```text
Could not resolve host
```

확인:

```bash
dig
cat /etc/resolv.conf
```

의심:

```text
DNS Record
Client DNS Server
named Service
Zone 설정
```

---

# 85. 대표 장애 Case - Connection Refused

증상:

```text
Connection refused
```

주로:

```text
Service 미실행
Port 미Listen
잘못된 Bind Address
```

등을 확인한다.

```bash
systemctl
ss
```

---

# 86. 대표 장애 Case - Timeout

증상:

```text
Connection timed out
```

확인:

```text
Firewall
Routing
Network
Security Policy
```

단 오류 Message만으로 원인을 단정하지 않는다.

---

# 87. 대표 장애 Case - 403 Forbidden

확인 대상:

```text
Apache Require 설정
Linux Permission
SELinux Context
DocumentRoot
Directory 설정
```

---

# 88. 대표 장애 Case - 500 Internal Server Error

확인 대상:

```text
PHP Syntax
PHP Runtime Error
require/include File
Permission
DB Connection
Application Logic
SELinux
```

우선:

```bash
tail /var/log/httpd/lamp-access.log
tail /var/log/httpd/lamp-error.log
php -l <file>
```

등으로 확인한다.

---

# 89. 대표 장애 Case - DB Access Denied

예:

```text
Access denied for user
```

확인:

```text
Username
Password
User Host
GRANT
DB Name
```

```sql
SELECT User,Host FROM mysql.user;
SHOW GRANTS FOR 'user'@'host';
```

---

# 90. 대표 장애 Case - DB Connection Refused

확인:

```text
mariadb Service
Port 3306
bind-address
Host 설정
Firewall
```

Local Socket을 사용하는 경우 Socket 경로도 확인한다.

---

# 91. PHP에서 htmlspecialchars()

DB 데이터를 HTML에 출력할 때:

```php
htmlspecialchars($row['name'])
```

형태를 사용하였다.

이는 HTML 특수문자를 Escape하여 출력하는 데 사용한다.

예:

```text
<
>
&
"
```

등을 HTML Entity 형태로 처리한다.

---

# 92. XSS와 출력 Escaping

DB에 저장된 문자열을 그대로 HTML에 출력하면 경우에 따라 XSS 위험이 생길 수 있다.

따라서 사용자 입력이나 외부 데이터를 HTML에 출력할 때는 Context에 맞는 Escaping이 필요하다.

PHP HTML 출력에서는:

```php
htmlspecialchars()
```

가 기본적인 방법 중 하나이다.

---

# 93. SQL Injection

이번 실습에서는 고정 SELECT Query만 사용하였다.

사용자 입력이 SQL Query에 들어가는 Application이라면 문자열을 직접 이어 붙이지 않는 것이 중요하다.

위험 예:

```php
$sql = "SELECT * FROM users WHERE name = '$name'";
```

권장 방식:

```text
Prepared Statement
```

이다.

---

# 94. Prepared Statement 개념

예:

```php
$stmt = $pdo->prepare(
    "SELECT * FROM users WHERE name = :name"
);

$stmt->execute([
    ':name' => $name
]);
```

사용자 입력과 SQL 구조를 분리하여 SQL Injection 위험을 줄인다.

---

# 95. Web/DB 계정 분리

Linux 계정과 Database 계정은 서로 다른 개념이다.

```text
Linux User
→ OS Permission

MariaDB User
→ Database Permission
```

예:

```text
apache
→ Linux Process User

lampuser
→ MariaDB User
```

서로 역할이 다르다.

---

# 96. root 계정도 서로 다르다

```text
Linux root
```

과:

```text
MariaDB root
```

도 서로 다른 보안 영역의 계정이다.

Linux root 권한이 있다고 해서 모든 Database Application이 root 계정을 사용해야 하는 것은 아니다.

Application은 전용 DB User를 사용하는 것이 좋다.

---

# 97. LAMP Security 기본 원칙

```text
Web Server
→ 필요한 Port만 공개

Database
→ 가능하면 외부 직접 공개 제한

DB User
→ 최소 권한

Secret
→ Source Code와 분리

File Permission
→ 필요한 Process만 읽기

SELinux
→ Enforcing 유지

Firewall
→ 필요한 Service만 허용

Log
→ 장애 발생 시 근거로 사용
```

---

# 98. expose_php

실습 응답 Header에는:

```text
X-Powered-By: PHP/8.0.30
```

가 확인되었다.

이는 PHP Version 정보를 노출할 수 있다.

운영 환경에서는 필요에 따라 `php.ini`의:

```ini
expose_php = Off
```

설정을 검토할 수 있다.

이번 실습에서는 변경하지 않았다.

---

# 99. Development와 Production 차이

학습 환경에서는 오류를 쉽게 확인하기 위해 상세 정보를 볼 수 있지만 운영 환경에서는 민감한 내부 정보를 사용자에게 노출하지 않는 것이 중요하다.

개념:

```text
Development
→ Debug 편의성 중시

Production
→ 보안 / 안정성 중시
```

---

# 100. Database Backup도 필요하다

LAMP Application이 정상 작동하더라도 Database Data가 손실되면 Service 복구가 어렵다.

따라서 실제 운영 환경에서는:

```text
Database Backup
Application Source Backup
Config Backup
Restore Test
```

가 함께 필요하다.

이전 rsync Backup 실습과 연결하여 생각할 수 있다.

---

# 101. LAMP와 이전 실습 연결

이번 LAMP 실습은 앞에서 학습한 여러 내용을 종합한다.

```text
DNS
→ lamp.bangkyu.com

Apache
→ VirtualHost

Linux Permission
→ lamp-private 권한

SELinux
→ httpd_sys_content_t

Firewalld
→ HTTP 허용 / DB 비공개

Process
→ httpd / php-fpm / mariadb

Log
→ Access / Error / journalctl

Security
→ 최소 권한 / Secret 분리
```

즉 개별 실습이 실제 Web Service 구성에서 서로 연결된다.

---

# 102. 실습 최종 Architecture

```text
                     Client-L
                  192.168.111.150
                         │
                         │ DNS
                         ▼
                 lamp.bangkyu.com
                         │
                         │ A Record
                         ▼
                  192.168.111.100
                         │
                         │ HTTP :80
                         ▼
                    Firewalld
                         │
                         ▼
                      Apache
                         │
                         │ VirtualHost
                         ▼
                 /var/www/lamp
                         │
                         │ index.php
                         ▼
                    proxy_fcgi
                         │
                         ▼
                     PHP-FPM
                         │
                         ├── db-config.php
                         │   /var/www/lamp-private
                         │
                         ▼
                       PDO
                         │
                         │ Unix Socket
                         ▼
                     MariaDB
                /var/lib/mysql/mysql.sock
                         │
                         ▼
                      lampdb
                         │
                         ▼
                     visitors
```

MariaDB TCP:

```text
127.0.0.1:3306
```

외부 Client:

```text
192.168.111.100:3306
→ BLOCKED
```

---

# 103. 면접 질문 - LAMP가 무엇인가?

```text
LAMP는 Linux, Apache, MySQL/MariaDB, PHP로 구성된
전통적인 Web Application Stack입니다.

Apache가 HTTP 요청을 받고 PHP가 Application Logic을 처리하며,
MariaDB가 데이터를 저장하고 Linux가 전체 실행 환경을 제공합니다.
```

---

# 104. 면접 질문 - PHP-FPM을 사용하는 이유는?

```text
PHP 실행을 Web Server Process와 분리하여 관리할 수 있으며
FastCGI를 통해 Apache와 PHP를 연동할 수 있습니다.

이번 환경에서는 Apache event MPM과 proxy_fcgi를 이용해
Unix Socket으로 PHP-FPM에 요청을 전달했습니다.
```

---

# 105. 면접 질문 - DB root를 Application에서 사용하지 않은 이유는?

```text
Application에 필요한 권한보다 훨씬 큰 권한을 주게 되기 때문입니다.

전용 DB User를 생성하고 필요한 Database에 필요한 권한만 부여하여
최소 권한 원칙을 적용하는 것이 안전합니다.
```

---

# 106. 면접 질문 - MariaDB 3306을 왜 외부에 열지 않았는가?

```text
Web Application과 MariaDB가 동일 Server에 있기 때문에
외부 Client가 Database에 직접 접근할 필요가 없었습니다.

따라서 MariaDB를 127.0.0.1에 Bind하고
Firewalld에서도 MySQL/3306을 허용하지 않아
Database 직접 접근 범위를 줄였습니다.
```

---

# 107. 면접 질문 - localhost와 127.0.0.1의 차이는?

```text
MySQL/MariaDB Client 환경에서는 localhost를 사용하면
Unix Socket을 사용하는 구성이 일반적이고,
127.0.0.1을 지정하면 TCP Loopback Connection을 사용합니다.

실제 사용 방식은 Client Library와 설정을 확인해야 합니다.
```

---

# 108. 면접 질문 - HTTP 500을 어떻게 해결했는가?

```text
DB 설정 File을 DocumentRoot 밖으로 분리한 뒤
HTTP 500이 발생했습니다.

PHP 문법 검사는 정상이라 Syntax 문제를 제외했고,
File과 상위 Directory Permission을 확인했습니다.

설정 File은 apache Group이 읽을 수 있었지만
상위 Directory가 root:root 750이라
apache Process가 Directory를 Traverse하지 못했습니다.

Directory Group을 apache로 변경하고 750,
설정 File을 root:apache 640으로 유지한 뒤
HTTP 200으로 복구되는 것을 확인했습니다.
```

---

# 109. 면접 질문 - File은 읽기 권한이 있는데 왜 읽지 못했는가?

```text
Linux에서 File에 접근하려면 해당 File의 Permission뿐만 아니라
경로에 포함된 상위 Directory들을 Traverse할 수 있어야 합니다.

Directory의 x 권한은 해당 Directory를 통과할 수 있는 권한이므로
상위 Directory에 x 권한이 없으면 File 자체가 readable이어도
접근하지 못할 수 있습니다.
```

---

# 110. 면접 질문 - DNS와 VirtualHost의 차이는?

```text
DNS는 Domain Name을 Server IP로 변환합니다.

Apache VirtualHost는 동일 Server에 도착한 HTTP Request의
Host Header를 기준으로 어떤 Site를 제공할지 결정합니다.

DNS는 Server를 찾고,
VirtualHost는 Server 안에서 Site를 선택합니다.
```

---

# 111. 실무 Troubleshooting 핵심

LAMP 장애 발생 시 한 번에 여러 설정을 바꾸지 않는다.

```text
DNS 확인
    ↓
Network 확인
    ↓
Port 확인
    ↓
Apache 확인
    ↓
VirtualHost 확인
    ↓
PHP-FPM 확인
    ↓
PHP Syntax 확인
    ↓
Database 확인
    ↓
DB User/GRANT 확인
    ↓
Permission 확인
    ↓
SELinux 확인
    ↓
Log 확인
    ↓
한 가지 원인 수정
    ↓
재검증
```

이렇게 계층별로 확인해야 원인을 정확하게 찾을 수 있다.

---

# 112. 주요 명령어 정리

Apache:

```bash
systemctl status httpd
systemctl is-active httpd
httpd -t
httpd -S
ss -lntp | grep httpd
```

PHP:

```bash
php -v
php -m
php -l file.php
php -i
```

PHP-FPM:

```bash
systemctl status php-fpm
ls -l /run/php-fpm/www.sock
journalctl -u php-fpm
```

MariaDB:

```bash
systemctl status mariadb
mariadb
mariadb -e "SHOW DATABASES;"
ss -lntp | grep ':3306'
```

DB User:

```sql
SELECT User,Host FROM mysql.user;
SHOW GRANTS FOR 'user'@'host';
```

DNS:

```bash
named-checkzone
named-checkconf
rndc reload
dig
```

Firewall:

```bash
firewall-cmd --list-all
firewall-cmd --query-service=http
firewall-cmd --query-service=mysql
firewall-cmd --query-port=3306/tcp
```

Permission:

```bash
ls -l
ls -ld
namei -l <path>
```

SELinux:

```bash
getenforce
ls -Z
ls -Zd
restorecon
ausearch -m AVC -ts recent
```

Log:

```bash
tail /var/log/httpd/lamp-access.log
tail /var/log/httpd/lamp-error.log
journalctl -u httpd
journalctl -u php-fpm
journalctl -u mariadb
```

---

# 113. 최종 핵심 정리

LAMP Stack의 핵심은 각 Package를 설치하는 것만이 아니다.

```text
Linux
Apache
PHP
MariaDB
```

각 Component 사이의 연결이 정상이어야 실제 Application이 동작한다.

이번 실습에서는 다음 전체 흐름을 확인하였다.

```text
DNS
→ Apache
→ PHP-FPM
→ PHP
→ PDO
→ MariaDB
→ SQL
→ HTML
→ Client
```

보안 측면에서는:

```text
DB root 미사용
→ Application 전용 User 사용

DB User
→ SELECT 최소 권한

MariaDB
→ 127.0.0.1 Bind

Firewalld
→ 3306 외부 미허용

DB Password
→ DocumentRoot 외부 분리

Private Directory
→ root:apache 750

DB Config File
→ root:apache 640

SELinux
→ Enforcing 유지
```

구조를 적용하였다.

Troubleshooting 측면에서는:

```text
HTTP 500 발생
        ↓
Service 확인
        ↓
PHP Syntax 확인
        ↓
Permission 확인
        ↓
상위 Directory Traverse 문제 발견
        ↓
필요한 Group Permission만 수정
        ↓
HTTP 200 복구
```

과정을 확인하였다.

최종적으로 중요한 것은:

```text
"Service가 안 되면 설정을 아무거나 바꾸는 것"

이 아니라

"요청이 통과하는 계층을 순서대로 확인하고
실제 증거를 이용해 원인을 좁히는 것"
```

이다.
