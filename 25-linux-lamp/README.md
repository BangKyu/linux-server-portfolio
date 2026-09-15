# LAMP Stack 구축 및 연동

Rocky Linux Server에서 Apache, PHP-FPM, MariaDB를 이용하여 LAMP Stack을 구축하고 DNS와 연동하였다.

PHP에서 MariaDB의 데이터를 조회하여 실제 Web Page에 출력하고, MariaDB의 외부 Network 접근 제한과 Application 계정 최소 권한 설정까지 진행하였다.

또한 DB 설정 File을 DocumentRoot 외부로 분리하는 과정에서 발생한 HTTP 500 장애를 Linux Permission 관점에서 분석하고 해결하였다.

---

# 1. 실습 환경

| 구분 | 내용 |
|---|---|
| Server | Server-A |
| Server IP | 192.168.111.100 |
| Client | Client-L |
| Client IP | 192.168.111.150 |
| Domain | lamp.bangkyu.com |
| Web Server | Apache HTTP Server |
| PHP | PHP 8.0.30 |
| PHP 실행 방식 | PHP-FPM + proxy_fcgi |
| Database | MariaDB 10.5.29 |
| Database Name | lampdb |
| Application User | lampuser |

전체 구조:

```text
Client-L
   ↓
DNS
lamp.bangkyu.com
   ↓
192.168.111.100
   ↓
Apache
   ↓
PHP-FPM
   ↓
PHP PDO
   ↓
MariaDB
   ↓
lampdb.visitors
```

---

# 2. 초기 상태 확인

Apache는 기존 실습에서 이미 설치되어 있었다.

```bash
rpm -q httpd
systemctl is-active httpd
systemctl is-enabled httpd
```

확인 결과:

```text
httpd-2.4.62-13.el9_8.6.x86_64
active
enabled
```

PHP와 MariaDB Server는 설치되지 않은 상태였다.

```text
php 패키지가 설치되어 있지 않습니다
php-cli 패키지가 설치되어 있지 않습니다
php-mysqlnd 패키지가 설치되어 있지 않습니다
mariadb-server 패키지가 설치되어 있지 않습니다
```

---

# 3. PHP 및 MariaDB 설치

필요 Package 설치:

```bash
dnf install -y php php-cli php-mysqlnd mariadb-server
```

설치 후 확인:

```bash
rpm -q php
rpm -q php-cli
rpm -q php-mysqlnd
rpm -q mariadb-server
```

결과:

```text
php-8.0.30-8.el9_8.x86_64
php-cli-8.0.30-8.el9_8.x86_64
php-mysqlnd-8.0.30-8.el9_8.x86_64
mariadb-server-10.5.29-3.el9_7.x86_64
```

PHP:

```bash
php -v
```

```text
PHP 8.0.30
```

MariaDB:

```bash
mariadb --version
```

```text
Distrib 10.5.29-MariaDB
```

---

# 4. MariaDB Service 기동

```bash
systemctl enable --now mariadb
```

확인:

```bash
systemctl is-active mariadb
systemctl is-enabled mariadb
```

결과:

```text
active
enabled
```

초기 MariaDB는 모든 Interface의 TCP 3306에서 Listen하고 있었다.

```bash
ss -lntp | grep ':3306'
```

```text
*:3306
```

---

# 5. Apache와 PHP-FPM 연동 확인

PHP-FPM Package:

```bash
rpm -q php-fpm
```

결과:

```text
php-fpm-8.0.30-8.el9_8.x86_64
```

Apache PHP 설정 확인:

```bash
grep -Ev '^[[:space:]]*(#|$)' /etc/httpd/conf.d/php.conf
```

주요 설정:

```apache
<FilesMatch \.(php|phar)$>
    SetHandler "proxy:unix:/run/php-fpm/www.sock|fcgi://localhost"
</FilesMatch>
```

Apache Module 확인:

```bash
httpd -M | grep -E 'php|proxy_fcgi|mpm'
```

결과:

```text
mpm_event_module (shared)
proxy_fcgi_module (shared)
```

즉 PHP 처리 구조는 다음과 같다.

```text
Apache
   ↓
proxy_fcgi
   ↓
/run/php-fpm/www.sock
   ↓
PHP-FPM
```

---

# 6. PHP-FPM 기동

```bash
systemctl enable --now php-fpm
```

확인:

```bash
systemctl is-active php-fpm
systemctl is-enabled php-fpm
```

결과:

```text
active
enabled
```

Socket:

```bash
ls -l /run/php-fpm/www.sock
```

PHP CLI 동작 확인:

```bash
php -r 'echo "PHP CLI Test OK\n";'
```

결과:

```text
PHP CLI Test OK
```

MariaDB Driver 확인:

```bash
php -m | grep -Ei 'mysqli|pdo_mysql'
```

결과:

```text
mysqli
pdo_mysql
```

---

# 7. MariaDB Database 구성

Application 전용 Database 생성:

```sql
CREATE DATABASE lampdb
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Application 계정 생성:

```sql
CREATE USER 'lampuser'@'localhost'
IDENTIFIED BY '<DB_PASSWORD>';
```

Database에 필요한 최소 권한만 부여:

```sql
GRANT SELECT ON lampdb.* TO 'lampuser'@'localhost';
```

Table 생성:

```sql
USE lampdb;

CREATE TABLE visitors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    message VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

테스트 Data 입력:

```sql
INSERT INTO visitors (name, message)
VALUES
    ('BangKyu', 'LAMP Stack Test'),
    ('Linux Lab', 'Apache PHP MariaDB OK');
```

> 실제 Database Password는 Git Repository나 README에 기록하지 않는다.

---

# 8. Application 계정 권한 확인

```bash
mariadb -e "
SHOW GRANTS FOR 'lampuser'@'localhost';
"
```

주요 결과:

```text
GRANT SELECT ON `lampdb`.* TO `lampuser`@`localhost`
```

즉 Application 계정에는 전체 DB 권한을 주지 않고 `lampdb`에 대한 `SELECT` 권한만 부여하였다.

---

# 9. Database 데이터 확인

```bash
mariadb -e "
USE lampdb;
SHOW TABLES;
SELECT * FROM visitors;
"
```

최종 데이터:

```text
+----+-----------+-----------------------+---------------------+
| id | name      | message               | created_at          |
+----+-----------+-----------------------+---------------------+
|  1 | BangKyu   | LAMP Stack Test       | 2026-09-15 12:50:18 |
|  2 | Linux Lab | Apache PHP MariaDB OK | 2026-09-15 12:50:18 |
+----+-----------+-----------------------+---------------------+
```

Application 계정으로도 직접 접속하여 조회를 확인하였다.

```bash
mariadb -u lampuser -p lampdb
```

```sql
SELECT * FROM visitors;
```

정상적으로 2개의 Row가 조회되었다.

---

# 10. LAMP 전용 DocumentRoot 생성

```bash
mkdir -p /var/www/lamp

chmod 755 /var/www/lamp
chmod 644 /var/www/lamp/index.php

restorecon -Rv /var/www/lamp
```

SELinux Context 확인:

```bash
ls -Zd /var/www/lamp
ls -Z /var/www/lamp/index.php
```

결과:

```text
unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/lamp
unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/lamp/index.php
```

---

# 11. PHP Database 조회 Page

`/var/www/lamp/index.php`에서 PDO를 이용하여 MariaDB의 `visitors` Table을 조회하였다.

주요 구조:

```php
<?php

$config = require '/var/www/lamp-private/db-config.php';

$host = $config['host'];
$dbname = $config['dbname'];
$username = $config['username'];
$password = $config['password'];

try {
    $pdo = new PDO(
        "mysql:host={$host};dbname={$dbname};charset=utf8mb4",
        $username,
        $password,
        [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
        ]
    );

    $stmt = $pdo->query(
        "SELECT id, name, message, created_at
         FROM visitors
         ORDER BY id"
    );

    $visitors = $stmt->fetchAll();

} catch (PDOException $e) {
    error_log($e->getMessage());
    http_response_code(500);
    die('Database connection failed');
}
?>
```

출력 시 `htmlspecialchars()`를 사용하여 DB 값을 HTML에 출력하였다.

---

# 12. Apache VirtualHost 설정

설정 File:

```text
/etc/httpd/conf.d/lamp.conf
```

내용:

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

문법 검사:

```bash
httpd -t
```

결과:

```text
Syntax OK
```

Apache 적용:

```bash
systemctl reload httpd
```

VirtualHost 확인:

```bash
httpd -S
```

결과에 다음 Host가 추가되었다.

```text
port 80 namevhost lamp.bangkyu.com
```

---

# 13. Server 내부 LAMP 연동 테스트

DNS 등록 전 Host Header를 직접 지정하여 테스트하였다.

```bash
curl -i -H 'Host: lamp.bangkyu.com' http://127.0.0.1/
```

결과:

```text
HTTP/1.1 200 OK
X-Powered-By: PHP/8.0.30
```

Web Page에 실제 MariaDB 데이터가 출력되었다.

```text
BangKyu
LAMP Stack Test

Linux Lab
Apache PHP MariaDB OK
```

이를 통해 다음 전체 연결이 정상임을 확인하였다.

```text
Apache
   ↓
PHP-FPM
   ↓
PDO MySQL
   ↓
MariaDB
   ↓
lampdb.visitors
```

---

# 14. DNS Record 추가

Forward Zone:

```text
/var/named/bangkyu.com.db
```

기존 Serial:

```text
2026091402
```

변경:

```text
2026091501
```

A Record 추가:

```text
lamp    IN      A       192.168.111.100
```

Zone 검사:

```bash
named-checkzone bangkyu.com /var/named/bangkyu.com.db
```

결과:

```text
zone bangkyu.com/IN: loaded serial 2026091501
OK
```

전체 BIND 설정 확인:

```bash
named-checkconf
```

정상인 경우 별도 오류 출력 없음.

Zone Reload:

```bash
rndc reload bangkyu.com
```

확인:

```bash
dig @127.0.0.1 lamp.bangkyu.com +short
```

결과:

```text
192.168.111.100
```

---

# 15. Client-L DNS 및 Web 접속 확인

Client-L:

```bash
dig lamp.bangkyu.com +short
```

결과:

```text
192.168.111.100
```

Web 요청:

```bash
curl -i http://lamp.bangkyu.com/
```

결과:

```text
HTTP/1.1 200 OK
X-Powered-By: PHP/8.0.30
```

MariaDB 데이터도 정상적으로 출력되었다.

```text
BangKyu
LAMP Stack Test

Linux Lab
Apache PHP MariaDB OK
```

---

# 16. MariaDB Network 보안 점검

초기 MariaDB 상태:

```bash
ss -lntp | grep ':3306'
```

```text
*:3306
```

MariaDB는 모든 Network Interface에서 TCP 3306을 Listen하고 있었다.

그러나 Firewalld에서는 MariaDB 접근을 허용하지 않았다.

```bash
firewall-cmd --query-port=3306/tcp
firewall-cmd --query-service=mysql
```

결과:

```text
no
no
```

---

# 17. MariaDB Localhost 전용 설정

LAMP Stack의 Apache/PHP/MariaDB가 모두 같은 Server에서 실행되므로 외부 Host가 MariaDB에 직접 접근할 필요가 없다.

설정 File:

```text
/etc/my.cnf.d/mariadb-server.cnf
```

`[mysqld]`에 다음 설정을 추가하였다.

```ini
bind-address=127.0.0.1
```

MariaDB 재시작:

```bash
systemctl restart mariadb
```

확인:

```bash
ss -lntp | grep ':3306'
```

결과:

```text
LISTEN 0 80 127.0.0.1:3306 0.0.0.0:*
```

즉 MariaDB의 TCP 3306이 Loopback Interface에서만 Listen하도록 제한하였다.

---

# 18. MariaDB Unix Socket 확인

MariaDB Socket:

```bash
mariadb -e "SHOW VARIABLES LIKE 'socket';"
```

결과:

```text
/var/lib/mysql/mysql.sock
```

PHP 설정:

```bash
php -i | grep -E 'mysqli.default_socket|pdo_mysql.default_socket'
```

결과:

```text
mysqli.default_socket => /var/lib/mysql/mysql.sock
pdo_mysql.default_socket => /var/lib/mysql/mysql.sock
```

PHP Application에서는 DB Host를:

```text
localhost
```

로 사용하였다.

동일 Host에서 `localhost`를 사용하므로 PHP와 MariaDB 사이에 Local Unix Socket 연결을 사용할 수 있다.

---

# 19. MariaDB 외부 접근 차단 검증

MariaDB를 Loopback으로 제한한 후에도 Client-L의 Web 요청은 정상 동작하였다.

```bash
curl -i http://lamp.bangkyu.com/
```

결과:

```text
HTTP/1.1 200 OK
```

반면 Client-L에서 MariaDB TCP 3306으로 직접 연결을 시도하였다.

```bash
timeout 3 bash -c '</dev/tcp/192.168.111.100/3306' \
&& echo CONNECTED || echo BLOCKED
```

결과:

```text
BLOCKED
```

따라서 최종 구조는 다음과 같다.

```text
Client-L
   │
   │ HTTP 80
   ▼
Apache
   │
   ▼
PHP-FPM
   │
   │ Local Connection
   ▼
MariaDB
127.0.0.1:3306

Client-L
   │
   └── TCP 3306 직접 접근 → 차단
```

---

# 20. Database 인증 정보 분리

초기 PHP File에는 Database 인증정보가 직접 포함되어 있었다.

운영 관점에서 인증정보를 Web DocumentRoot와 분리하기 위해 다음 Directory를 생성하였다.

```text
/var/www/lamp-private
```

Database 설정 File:

```text
/var/www/lamp-private/db-config.php
```

형식:

```php
<?php

return [
    'host' => 'localhost',
    'dbname' => 'lampdb',
    'username' => 'lampuser',
    'password' => '<DB_PASSWORD>'
];
```

> 실제 Password가 포함된 `db-config.php`는 Git Repository에 Commit하지 않는다.

---

# 21. Private 설정 File 권한

최종 권한:

```bash
ls -ldZ /var/www/lamp-private
ls -lZ /var/www/lamp-private/db-config.php
```

결과:

```text
drwxr-x--- root apache ... httpd_sys_content_t ... /var/www/lamp-private
-rw-r----- root apache ... httpd_sys_content_t ... /var/www/lamp-private/db-config.php
```

즉:

```text
Directory
root:apache 750

db-config.php
root:apache 640
```

구조로 설정하였다.

일반 사용자는 접근할 수 없고 Apache/PHP Process가 필요한 설정 File만 읽을 수 있도록 제한하였다.

---

# 22. HTTP 500 장애 발생

DB 설정 File을 분리한 직후 LAMP Page 요청에서 HTTP 500이 발생하였다.

Access Log:

```text
127.0.0.1 - - [15/Sep/2026:13:05:28 +0900] "GET / HTTP/1.1" 500 -
192.168.111.150 - - [15/Sep/2026:13:05:34 +0900] "GET / HTTP/1.1" 500 -
127.0.0.1 - - [15/Sep/2026:13:05:43 +0900] "GET / HTTP/1.1" 500 -
```

PHP 문법 검사에서는 오류가 없었다.

```bash
php -l /var/www/lamp/index.php
php -l /var/www/lamp-private/db-config.php
```

결과:

```text
No syntax errors detected
```

---

# 23. HTTP 500 원인 분석

문제 당시 설정 File 자체는 다음 권한이었다.

```text
root:apache 640
```

하지만 상위 Directory의 Group이 `root`여서 Apache Process가 Directory를 Traverse하지 못하는 상태였다.

즉:

```text
PHP-FPM
   ↓
/var/www/lamp-private
root:root 750
   ↓
접근 불가
   ↓
db-config.php
root:apache 640
```

File에 읽기 권한이 있어도 해당 File까지 도달하기 위해서는 상위 Directory에 대한 실행(`x`) 권한이 필요하다.

---

# 24. HTTP 500 해결

Directory Group을 `apache`로 변경하였다.

```bash
chown root:apache /var/www/lamp-private
chmod 750 /var/www/lamp-private
```

최종 구조:

```text
/var/www/lamp-private
→ root:apache 750

db-config.php
→ root:apache 640
```

이후 다시 요청하였다.

```bash
curl -i -H 'Host: lamp.bangkyu.com' http://127.0.0.1/
```

결과:

```text
HTTP/1.1 200 OK
```

Database 데이터도 다시 정상적으로 출력되었다.

---

# 25. 장애 전후 Access Log

실제 Access Log:

```text
127.0.0.1 - - [15/Sep/2026:13:05:28 +0900] "GET / HTTP/1.1" 500 -
192.168.111.150 - - [15/Sep/2026:13:05:34 +0900] "GET / HTTP/1.1" 500 -
127.0.0.1 - - [15/Sep/2026:13:05:43 +0900] "GET / HTTP/1.1" 500 -
127.0.0.1 - - [15/Sep/2026:13:06:36 +0900] "GET / HTTP/1.1" 200 634 -
192.168.111.150 - - [15/Sep/2026:13:06:44 +0900] "GET / HTTP/1.1" 200 634 -
```

이를 통해:

```text
설정 File 분리
   ↓
HTTP 500
   ↓
Permission 분석
   ↓
Directory Group 수정
   ↓
HTTP 200 복구
```

흐름을 실제 Log로 확인하였다.

---

# 26. 최종 Service 상태

```bash
systemctl is-active httpd
systemctl is-active php-fpm
systemctl is-active mariadb
```

결과:

```text
active
active
active
```

---

# 27. 최종 Listen Port

```bash
ss -lntp | grep -E ':80 |:3306 '
```

결과:

```text
127.0.0.1:3306
*:80
```

즉:

```text
Apache
→ 외부 HTTP 요청 수신

MariaDB
→ Localhost에서만 TCP 연결 허용
```

구조이다.

---

# 28. 최종 Firewall 상태

```bash
firewall-cmd --query-service=http
firewall-cmd --query-port=3306/tcp
firewall-cmd --query-service=mysql
```

결과:

```text
yes
no
no
```

즉:

```text
HTTP
→ 외부 접근 허용

MariaDB 3306
→ 외부 접근 미허용
```

상태이다.

---

# 29. 최종 DNS 상태

```bash
dig @127.0.0.1 lamp.bangkyu.com +short
```

결과:

```text
192.168.111.100
```

---

# 30. 최종 Database 상태

Application 계정:

```text
lampuser@localhost
```

권한:

```text
lampdb SELECT
```

데이터:

```text
+----+-----------+-----------------------+
| id | name      | message               |
+----+-----------+-----------------------+
|  1 | BangKyu   | LAMP Stack Test       |
|  2 | Linux Lab | Apache PHP MariaDB OK |
+----+-----------+-----------------------+
```

---

# 31. LAMP 구성 요소 역할

## Linux

전체 Server OS 및 Service 실행 환경을 제공한다.

## Apache

Client의 HTTP 요청을 받아 Web Content를 제공한다.

## PHP

동적인 Web Application Logic을 처리한다.

## PHP-FPM

Apache에서 전달받은 PHP 요청을 실제 PHP Process에서 실행한다.

## MariaDB

Application이 사용하는 데이터를 저장하고 조회한다.

---

# 32. 전체 요청 흐름

```text
Client-L
   ↓
DNS Query
   ↓
lamp.bangkyu.com
   ↓
192.168.111.100
   ↓
Firewalld
HTTP 허용
   ↓
Apache VirtualHost
   ↓
/var/www/lamp/index.php
   ↓
proxy_fcgi
   ↓
PHP-FPM
   ↓
db-config.php
   ↓
PDO MySQL
   ↓
MariaDB
   ↓
lampdb.visitors
   ↓
PHP HTML 생성
   ↓
Apache
   ↓
Client-L
```

---

# 33. Troubleshooting 흐름

LAMP Service 장애 발생 시 다음 순서로 확인할 수 있다.

```text
1. DNS
   dig

2. Apache
   systemctl status httpd

3. Apache 설정
   httpd -t
   httpd -S

4. Listen Port
   ss -lntp

5. HTTP 응답
   curl

6. PHP-FPM
   systemctl status php-fpm

7. PHP 문법
   php -l

8. PHP Module
   php -m

9. MariaDB
   systemctl status mariadb

10. DB 직접 접속
    mariadb

11. DB 권한
    SHOW GRANTS

12. Linux Permission
    ls -l
    ls -ld

13. SELinux Context
    ls -Z
    ls -Zd

14. Firewall
    firewall-cmd

15. Log
    Apache Access/Error Log
    journalctl
```

---

# 34. 보안 관점에서 확인한 내용

이번 실습에서는 기능 동작뿐 아니라 다음 보안 요소도 함께 적용하였다.

```text
Application에서 DB root 계정 사용하지 않음

lampuser 전용 계정 생성

필요한 SELECT 권한만 부여

MariaDB 3306 외부 공개하지 않음

MariaDB bind-address를 127.0.0.1로 제한

DB 인증정보를 DocumentRoot 외부로 분리

Private 설정 Directory 접근 권한 제한

실제 Password를 Git Repository에 저장하지 않음
```

---

# 35. 실습을 통해 확인한 내용

- Apache + PHP + MariaDB LAMP Stack 구성
- Apache event MPM 및 proxy_fcgi 확인
- PHP-FPM Unix Socket 연동 확인
- PHP `mysqli`, `pdo_mysql` Module 확인
- MariaDB Database 및 Table 생성
- Application 전용 DB User 생성
- 최소 권한 `SELECT` 적용
- PHP PDO를 이용한 MariaDB 데이터 조회
- Apache Name-based VirtualHost 구성
- BIND DNS A Record 추가
- Client에서 DNS → Web → PHP → DB 전체 연동 확인
- MariaDB TCP 3306 Localhost 제한
- Firewalld를 통한 DB 외부 접근 차단 확인
- PHP/MariaDB Unix Socket 경로 확인
- DB 인증정보 DocumentRoot 외부 분리
- Linux Directory Permission으로 인한 HTTP 500 장애 분석
- Permission 수정 후 HTTP 200 복구
- Apache Access Log를 통한 장애/복구 검증

---

# 36. 핵심 정리

LAMP Stack은 각각의 Component가 독립적으로 정상인 것만으로는 충분하지 않다.

```text
Apache 정상
PHP 정상
MariaDB 정상
```

이어도 Component 사이의 연결이 잘못되어 있다면 Application은 정상적으로 동작하지 않는다.

따라서 다음 **전체 연결 경로**를 확인하는 것이 중요하다.

```text
DNS
→ Firewall
→ Apache
→ PHP-FPM
→ PHP
→ DB 인증정보
→ MariaDB
→ SQL
→ Web 응답
```

또한 장애 발생 시 무조건 Service를 재설치하거나 Permission을 크게 풀기보다 실제 장애 지점을 단계적으로 좁혀야 한다.

이번 실습에서는 HTTP 500 발생 시 PHP 문법과 Service 상태가 정상이었고, 최종적으로 Private 설정 Directory의 Group/Execute Permission 문제를 확인하여 필요한 권한만 수정하였다.

```text
장애 발생
→ 상태 확인
→ Log 확인
→ Permission 확인
→ 원인 수정
→ 재검증
```

이 과정을 통해 LAMP Stack 구축뿐 아니라 실제 Server 운영에서 필요한 기본적인 Troubleshooting 흐름까지 확인하였다.
