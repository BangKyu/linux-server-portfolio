# Linux Log Management

Rocky Linux Server에서 `systemd-journald`, `journalctl`, `rsyslog`, `/var/log` 구조를 확인하고, 실제 Application Log를 대상으로 `logrotate`를 구성하여 Log Rotation과 압축을 검증하였다.

단순 Log 조회뿐 아니라 다음 흐름으로 실습하였다.

```text
Log 환경 확인
        ↓
journalctl 기본 조회
        ↓
Service / PID / 시간 / Priority 필터링
        ↓
logger를 이용한 Test Log 생성
        ↓
실시간 Log 확인
        ↓
/var/log File 확인
        ↓
logrotate 설정
        ↓
Debug Mode 검증
        ↓
강제 Rotation
        ↓
delaycompress 검증
        ↓
notifempty 검증
```

---

# 1. 실습 환경

| 구분 | 내용 |
|---|---|
| Server | Server-A |
| IP | 192.168.111.100 |
| OS | Rocky Linux |
| Journal | systemd-journald |
| File Logging | rsyslog |
| Rotation | logrotate |
| Test Log | `/var/log/rsync-server-b-backup.log` |

---

# 2. Log Package 확인

```bash
rpm -q rsyslog
rpm -q logrotate
```

실제:

```text
rsyslog-8.2510.0-2.el9.x86_64
logrotate-3.18.0-12.el9.x86_64
```

---

# 3. Log Service 상태

```bash
systemctl is-active systemd-journald
systemctl is-active rsyslog
systemctl is-enabled rsyslog
```

실제:

```text
active
active
enabled
```

즉:

```text
systemd-journald
→ 현재 실행 중

rsyslog
→ 현재 실행 중
→ 부팅 시 자동 시작
```

상태이다.

---

# 4. Linux Log 구조

Rocky Linux에서는 다양한 Service와 Kernel Log가 Journal에 저장되며 `journalctl`을 통해 조회할 수 있다.

또한 rsyslog를 통해 일부 Log는 `/var/log`의 일반 File 형태로 저장된다.

구조:

```text
Kernel / Service / Application
              |
              ↓
      systemd-journald
              |
              ↓
         journalctl

일부 Log
              |
              ↓
           rsyslog
              |
              ↓
          /var/log/
```

---

# 5. /var/log 확인

```bash
ls -lh /var/log | head -n 20
```

실제 환경에서 다음과 같은 Log를 확인하였다.

```text
boot.log
cron
dnf.log
dnf.rpm.log
firewalld
httpd/
lastlog
maillog
audit/
chrony/
```

예:

```text
-rw-------. 1 root root 16K /var/log/cron
```

cron 관련 File Log가 실제 존재하였다.

---

# 6. Backup Log 확인

이전 rsync + cron 실습에서 생성한 Log:

```bash
ls -lh /var/log/rsync-server-b-backup.log
```

실제:

```text
-rw-r--r--. 1 root root 494 ... /var/log/rsync-server-b-backup.log
```

이 File을 이번 `logrotate` 실습 대상으로 사용하였다.

---

# 7. journalctl

`journalctl`은 systemd Journal에 저장된 Log를 조회하는 Command이다.

기본:

```bash
journalctl
```

전체 Journal은 양이 많을 수 있으므로 실무에서는 조건을 이용하여 필요한 Log만 조회하는 것이 중요하다.

---

# 8. 현재 Boot Log 조회

```bash
journalctl -b -n 10 --no-pager
```

Option:

```text
-b
→ 현재 Boot의 Log

-n 10
→ 마지막 10개

--no-pager
→ less Pager 없이 바로 출력
```

실제 최근 Log에서 `named`의 DNS Resolution 관련 기록과 systemd Service 기록 등을 확인하였다.

예:

```text
Server-A named[1237]: network unreachable resolving ...
Server-A systemd[1]: sssd-kcm.service: Deactivated successfully.
```

---

# 9. Service별 Log 조회

특정 systemd Unit의 Log만 조회할 수 있다.

이번에는 `crond`:

```bash
journalctl -u crond \
--since "10 minutes ago" \
--no-pager
```

실제:

```text
Server-A CROND: (root) CMD (/usr/local/sbin/rsync-server-b-backup.sh ...)
Server-A CROND: (root) CMDEND (...)
Server-A crond: (*system*) RELOAD (/etc/cron.d/rsync-server-b-backup)
```

이전 실습에서 설정한 Backup cron Job이 실제 실행되고 설정 File이 Reload된 기록을 확인하였다.

---

# 10. -u Option

```bash
journalctl -u SERVICE
```

특정 systemd Unit에 해당하는 Log를 조회한다.

예:

```bash
journalctl -u crond
journalctl -u sshd
journalctl -u named
```

Service 장애 분석 시 전체 Log를 검색하는 것보다 빠르게 범위를 좁힐 수 있다.

---

# 11. 시간 기준 조회

최근 특정 시간만 조회:

```bash
journalctl --since "3 minutes ago" --no-pager
```

시작과 종료 시간 지정:

```bash
journalctl \
--since "2026-09-15 10:54:00" \
--until "2026-09-15 10:59:00" \
-u crond \
--no-pager
```

장애 발생 시간을 알고 있다면 해당 시간 범위만 분석할 수 있다.

---

# 12. Priority 필터

현재 Boot에서 Error 이상 Log 조회:

```bash
journalctl -b -p err --no-pager
```

실제 확인된 기록 중:

```text
sshd-session: error: maximum authentication attempts exceeded ...
systemd: Failed to start dnf makecache.
```

등이 있었다.

---

# 13. Journal Priority

대표 Priority:

```text
emerg
alert
crit
err
warning
notice
info
debug
```

심각도는 위에서 아래로 낮아진다.

예:

```bash
journalctl -p err
```

은 Error 이상 Priority를 조회한다.

---

# 14. PID 기준 조회

특정 Process PID의 Log만 조회할 수 있다.

이번 `named` PID:

```text
1237
```

조회:

```bash
journalctl _PID=1237 -n 5 --no-pager
```

실제:

```text
Server-A named[1237]: network unreachable resolving ...
```

PID 1237의 Log만 출력되었다.

---

# 15. PID와 Unit 조회 차이

PID:

```bash
journalctl _PID=1237
```

장점:

```text
특정 Process Instance만 확인
```

하지만 Service 재시작 시 PID가 변경될 수 있다.

Service 기준:

```bash
journalctl -u named
```

은 Service 전체 흐름을 추적할 때 더 적합할 수 있다.

---

# 16. logger

`logger` Command를 이용하면 Test Log를 Journal / Syslog에 생성할 수 있다.

이번 실습:

```bash
logger -t portfolio-log-test "journalctl live test"
```

Option:

```text
-t
→ Log Identifier 지정
```

Identifier:

```text
portfolio-log-test
```

---

# 17. Identifier 기준 조회

```bash
journalctl -t portfolio-log-test -n 5 --no-pager
```

실제:

```text
Server-A portfolio-log-test[4270]: journalctl live test
```

직접 생성한 Log가 Journal에 정상 저장되었다.

---

# 18. 실시간 Log 확인

`journalctl -f`는 새로운 Journal Log를 실시간으로 Follow한다.

실습:

```bash
timeout 5 journalctl -f -n 0 \
-t portfolio-log-test \
--no-pager &
```

다른 Process에서:

```bash
logger -t portfolio-log-test \
"journalctl live test"
```

새 Log가 실시간으로 출력되는 것을 확인하였다.

---

# 19. journalctl -f

```bash
journalctl -f
```

의 `-f`:

```text
Follow
```

새 Log가 생성될 때마다 계속 출력한다.

Application 장애를 실시간으로 재현하거나 Service 동작을 확인할 때 유용하다.

---

# 20. Journal 조회 기본 흐름

```text
전체 Log
   ↓
시간 범위 지정
   ↓
Service 지정
   ↓
Priority 지정
   ↓
PID / Identifier 지정
   ↓
필요한 Log만 분석
```

Log가 많을수록 Filtering이 중요하다.

---

# 21. logrotate란?

`logrotate`는 계속 증가하는 Log File을 주기적으로 분리하고 오래된 Log를 압축 / 삭제하여 관리하는 Tool이다.

Log를 Rotation하지 않으면:

```text
Application
      ↓
Log 계속 기록
      ↓
File Size 증가
      ↓
Disk 공간 증가
      ↓
File System Full 가능
```

상황이 발생할 수 있다.

---

# 22. Rotation 구조

이번 목표:

```text
rsync-server-b-backup.log
→ 현재 Log

rsync-server-b-backup.log.1
→ 직전 Log

rsync-server-b-backup.log.2.gz
→ 더 오래된 압축 Log
```

---

# 23. logrotate 설정 생성

설정 File:

```text
/etc/logrotate.d/rsync-server-b-backup
```

내용:

```text
/var/log/rsync-server-b-backup.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    nodateext
}
```

---

# 24. daily

```text
daily
```

하루 단위로 Rotation 필요 여부를 확인한다.

---

# 25. rotate 7

```text
rotate 7
```

과거 Rotation Log를 최대 7개까지 보관하도록 설정한다.

오래된 Log는 보관 개수를 초과하면 삭제 대상이 된다.

---

# 26. compress

```text
compress
```

오래된 Log를 gzip 형태로 압축한다.

예:

```text
rsync-server-b-backup.log.2.gz
```

---

# 27. delaycompress

```text
delaycompress
```

직전 Rotation File은 바로 압축하지 않는다.

예:

```text
현재
.log

직전
.log.1

더 오래된 Log
.log.2.gz
```

즉:

```text
1차 Rotation
→ .1

2차 Rotation
→ 기존 .1을 .2.gz로 압축
→ 새로운 .1 생성
```

---

# 28. missingok

```text
missingok
```

대상 Log File이 없어도 Error로 처리하지 않는다.

---

# 29. notifempty

```text
notifempty
```

Log File이 비어 있으면 Rotation하지 않는다.

이번 실습에서 실제 검증하였다.

---

# 30. create

```text
create 0640 root root
```

Rotation 후 새로운 현재 Log File을 생성한다.

설정:

```text
Permission
0640

Owner
root

Group
root
```

---

# 31. 0640

```text
Owner
rw-

Group
r--

Other
---
```

즉:

```text
root
→ Read / Write

root Group
→ Read

Others
→ 접근 불가
```

---

# 32. nodateext

```text
nodateext
```

날짜 기반 이름 대신:

```text
.log.1
.log.2
.log.3
```

형태의 번호 기반 Rotation File을 사용하도록 구성하였다.

---

# 33. logrotate Debug Mode

설정을 실제로 적용하기 전에:

```bash
logrotate -d \
/etc/logrotate.d/rsync-server-b-backup
```

로 확인하였다.

`-d`:

```text
Debug Mode
→ 실제 File 변경 없음
```

---

# 34. Debug 결과

실제:

```text
Handling 1 logs

rotating pattern: /var/log/rsync-server-b-backup.log after 1 days (7 rotations)

considering log /var/log/rsync-server-b-backup.log

Now: 2026-09-15 11:16
Last rotated at 2026-09-15 11:00

log does not need rotating
(log has already been rotated)
```

설정 문법이 정상적으로 읽혔으며 당시 상태에서는 `daily` 조건에 따라 Rotation이 필요하지 않다고 판단하였다.

---

# 35. 강제 Rotation

실습을 위해 시간 조건과 관계없이 강제로 Rotation:

```bash
logrotate -f \
/etc/logrotate.d/rsync-server-b-backup
```

`-f`:

```text
Force
```

---

# 36. 첫 번째 Rotation 결과

실제:

```text
-rw-r-----. 1 root root   0 ... rsync-server-b-backup.log
-rw-r--r--. 1 root root 494 ... rsync-server-b-backup.log.1
```

결과:

```text
현재 Log
→ 새 File 생성
→ 0 byte

기존 494B Log
→ .log.1로 이동
```

---

# 37. create 검증

```bash
stat -c '%A %U %G %n' \
/var/log/rsync-server-b-backup.log*
```

실제 현재 Log:

```text
-rw-r----- root root /var/log/rsync-server-b-backup.log
```

즉:

```text
create 0640 root root
```

설정이 정상 적용되었다.

---

# 38. 첫 번째 Rotation에서 압축되지 않은 이유

설정에:

```text
delaycompress
```

가 있기 때문이다.

첫 번째 Rotation:

```text
.log
→ 새 Log

.log.1
→ 이전 Log
→ 아직 압축 안 함
```

---

# 39. 두 번째 Rotation 준비

새 현재 Log가 0 byte이므로 테스트 내용을 추가하였다.

```bash
printf 'second rotation test\n' \
>> /var/log/rsync-server-b-backup.log
```

내용:

```text
second rotation test
```

---

# 40. 두 번째 강제 Rotation

```bash
logrotate -f \
/etc/logrotate.d/rsync-server-b-backup
```

결과:

```text
-rw-r-----. 1 root root   0 ... rsync-server-b-backup.log
-rw-r-----. 1 root root  21 ... rsync-server-b-backup.log.1
-rw-r--r--. 1 root root 167 ... rsync-server-b-backup.log.2.gz
```

---

# 41. 두 번째 Rotation 구조

```text
rsync-server-b-backup.log
→ 새로운 현재 Log
→ 0 byte

rsync-server-b-backup.log.1
→ 직전 Log
→ second rotation test

rsync-server-b-backup.log.2.gz
→ 최초 Backup Log
→ gzip 압축
```

---

# 42. .1 Log 확인

```bash
cat /var/log/rsync-server-b-backup.log.1
```

실제:

```text
second rotation test
```

---

# 43. gzip Log 확인

압축 File을 해제하지 않고 내용 확인:

```bash
gzip -cd \
/var/log/rsync-server-b-backup.log.2.gz
```

실제 기존 rsync Backup Log 내용이 정상 출력되었다.

예:

```text
sent 222 bytes received 13 bytes ...
total size is 101 ...
sending incremental file list
```

---

# 44. compress + delaycompress 검증

실제 흐름:

```text
초기 Log
494B
   ↓
1차 Rotation
   ↓
.log.1
494B
압축 안 됨
   ↓
2차 Rotation
   ↓
.log.1
21B

.log.2.gz
167B
```

따라서:

```text
compress
→ 오래된 Log 압축

delaycompress
→ 직전 Log의 압축을 한 Rotation 늦춤
```

동작을 직접 확인하였다.

---

# 45. notifempty 검증

두 번째 Rotation 이후 현재 Log:

```text
0 byte
```

상태에서 다시:

```bash
logrotate -f \
/etc/logrotate.d/rsync-server-b-backup
```

실행하였다.

이후:

```bash
ls -lh /var/log/rsync-server-b-backup.log*
```

실제:

```text
-rw-r-----. 1 root root   0 ... rsync-server-b-backup.log
-rw-r-----. 1 root root  21 ... rsync-server-b-backup.log.1
-rw-r--r--. 1 root root 167 ... rsync-server-b-backup.log.2.gz
```

구조가 변경되지 않았다.

즉:

```text
현재 Log가 Empty
        ↓
notifempty
        ↓
Rotation 수행하지 않음
```

을 실제로 확인하였다.

---

# 46. Log Rotation 전체 흐름

```text
Application Log 기록
        ↓
rsync-server-b-backup.log
        ↓
logrotate 실행
        ↓
.log.1
        ↓
다음 Rotation
        ↓
.log.2.gz
        ↓
rotate 7 초과
        ↓
가장 오래된 Log 삭제
```

---

# 47. journalctl과 /var/log 차이

`journalctl`:

```text
systemd Journal 조회
Service / Boot / PID / Priority 등
다양한 Metadata 기반 검색 가능
```

`/var/log`:

```text
일반 File 형태 Log
cat / less / tail / grep 등으로 조회 가능
```

둘 중 하나만 사용하는 것이 아니라 환경에 따라 함께 사용한다.

---

# 48. 장애 분석 기본 흐름

```text
서비스 이상 발생
        ↓
systemctl status SERVICE
        ↓
journalctl -u SERVICE
        ↓
발생 시간 범위 확인
        ↓
Priority 확인
        ↓
필요하면 PID / Identifier 확인
        ↓
/var/log Application Log 확인
        ↓
원인 분석
```

---

# 49. Service 장애 확인

예:

```bash
systemctl status sshd
```

관련 Log:

```bash
journalctl -u sshd
```

최근 Log:

```bash
journalctl -u sshd -n 30 --no-pager
```

---

# 50. 최근 시간만 조회

```bash
journalctl -u sshd \
--since "10 minutes ago" \
--no-pager
```

전체 과거 Log보다 필요한 범위만 보는 것이 효율적이다.

---

# 51. Boot 기준 Log

현재 Boot:

```bash
journalctl -b
```

이전 Boot 정보도 Journal 보존 상태에 따라 확인할 수 있다.

재부팅 직후 장애가 발생했다면 Boot 기준 조회가 유용하다.

---

# 52. Log를 너무 많이 출력하지 않는 이유

운영 Server에서는 Log가 매우 많을 수 있다.

따라서:

```bash
journalctl
```

전체를 무조건 출력하기보다:

```text
Service
시간
Priority
PID
Identifier
마지막 N줄
```

등으로 범위를 줄이는 것이 중요하다.

---

# 53. tail

File Log의 최신 부분 확인:

```bash
tail -n 20 /var/log/FILE
```

실시간:

```bash
tail -f /var/log/FILE
```

---

# 54. journalctl과 tail 비교

Journal:

```bash
journalctl -f
```

File Log:

```bash
tail -f /var/log/application.log
```

둘 다 실시간 Log Monitoring에 사용할 수 있다.

---

# 55. Log Rotation이 필요한 이유

Log File이 계속 증가하면:

```text
Disk 사용량 증가
        ↓
File System 부족
        ↓
Application Log 기록 실패
        ↓
Service 문제 가능
```

이 발생할 수 있다.

따라서 Log 관리도 Server 운영의 중요한 부분이다.

---

# 56. rotate 개수와 보관 기간

이번:

```text
daily
rotate 7
```

구성은 대략적으로 최근 Daily Rotation File을 최대 7개 보관하는 정책이다.

하지만 정확한 보관 기간은 실제 Rotation 실행 여부와 Log 생성 상태에 따라 달라질 수 있다.

---

# 57. 빈 Log와 Rotation

```text
notifempty
```

를 설정하면 Log가 비어 있는 상황에서 불필요한 Rotation File이 생성되는 것을 막을 수 있다.

이번 실습에서 강제 Rotation을 사용해도 빈 File은 Rotation되지 않는 것을 확인하였다.

---

# 58. Log 압축

오래된 Log를 gzip으로 압축하면 Disk 사용량을 줄일 수 있다.

확인:

```bash
ls -lh *.gz
```

내용 확인:

```bash
gzip -cd FILE.gz
```

또는:

```bash
zcat FILE.gz
```

---

# 59. logrotate 설정 위치

일반 설정:

```text
/etc/logrotate.conf
```

개별 Service 설정:

```text
/etc/logrotate.d/
```

이번:

```text
/etc/logrotate.d/rsync-server-b-backup
```

---

# 60. logrotate 실무 확인 순서

```text
Log File 확인
        ↓
/etc/logrotate.d 설정 작성
        ↓
logrotate -d
        ↓
설정 / 판단 확인
        ↓
필요하면 Test 환경에서 -f
        ↓
Rotation File 확인
        ↓
Permission 확인
        ↓
압축 확인
```

---

# 61. -d와 -f 차이

```text
-d
→ Debug
→ 실제 변경하지 않음
```

```text
-f
→ Force
→ 조건과 관계없이 강제 Rotation 시도
```

실제 운영 Log에서 `-f` 사용 시 의도하지 않은 Rotation이 발생할 수 있으므로 주의해야 한다.

---

# 62. 주요 Command

Journal 최근 Log:

```bash
journalctl -b -n 20 --no-pager
```

Service Log:

```bash
journalctl -u SERVICE
```

시간:

```bash
journalctl --since "10 minutes ago"
```

Priority:

```bash
journalctl -p err
```

PID:

```bash
journalctl _PID=PID
```

Identifier:

```bash
journalctl -t IDENTIFIER
```

실시간:

```bash
journalctl -f
```

---

# 63. logger

Test Log 생성:

```bash
logger -t test "message"
```

조회:

```bash
journalctl -t test
```

---

# 64. logrotate

Debug:

```bash
logrotate -d /etc/logrotate.d/CONFIG
```

Force:

```bash
logrotate -f /etc/logrotate.d/CONFIG
```

---

# 65. Rotation File 확인

```bash
ls -lh /var/log/rsync-server-b-backup.log*
```

Permission:

```bash
stat -c '%A %U %G %n' \
/var/log/rsync-server-b-backup.log*
```

---

# 실습 결과

```text
Log 환경
✓ rsyslog 설치 확인
✓ logrotate 설치 확인
✓ systemd-journald active 확인
✓ rsyslog active / enabled 확인
✓ /var/log File 확인

journalctl
✓ 현재 Boot Log 조회
✓ 최근 N개 Log 조회
✓ Service별 조회
✓ 시간 범위 조회
✓ Priority 조회
✓ PID별 조회
✓ Identifier별 조회
✓ logger Test Log 생성
✓ 실시간 Follow 확인

기존 환경 연계
✓ crond Backup Log 확인
✓ cron RELOAD 확인
✓ Backup cron CMD / CMDEND 확인

logrotate
✓ 실제 Backup Log를 Rotation 대상으로 사용
✓ daily 설정
✓ rotate 7 설정
✓ compress 설정
✓ delaycompress 설정
✓ missingok 설정
✓ notifempty 설정
✓ create 0640 root root 설정
✓ nodateext 설정
✓ Debug Mode 확인
✓ 강제 Rotation 확인
✓ 새 Log File 생성 확인
✓ .log.1 생성 확인
✓ .log.2.gz 생성 확인
✓ gzip 내용 확인
✓ notifempty 실제 동작 확인
```

---

# 최종 구조

```text
          Application / Service / Kernel
                     |
             ----------------
             |              |
             ↓              ↓
     systemd-journald     rsyslog
             |              |
             ↓              ↓
        journalctl       /var/log/
                            |
                            ↓
                         logrotate
                            |
             -----------------------------
             |             |             |
             ↓             ↓             ↓
        current.log      .log.1       .log.2.gz
```

---

# 최종 결과

Rocky Linux Server의 Log 관리 구조를 확인하고 `journalctl`을 이용하여:

```text
Boot
Service
시간
Priority
PID
Identifier
```

기준으로 필요한 Log만 조회하였다.

`logger`를 이용하여 Test Log를 직접 생성하고:

```text
portfolio-log-test[4270]: journalctl live test
```

가 Journal에 기록되는 것도 확인하였다.

이후 기존 rsync Backup Log:

```text
/var/log/rsync-server-b-backup.log
```

를 대상으로:

```text
daily
rotate 7
compress
delaycompress
missingok
notifempty
create 0640 root root
nodateext
```

정책을 구성하였다.

첫 번째 강제 Rotation에서는:

```text
rsync-server-b-backup.log
→ 0B 새 File

rsync-server-b-backup.log.1
→ 기존 494B Log
```

를 확인하였다.

두 번째 Rotation에서는:

```text
rsync-server-b-backup.log
→ 현재 Log

rsync-server-b-backup.log.1
→ second rotation test

rsync-server-b-backup.log.2.gz
→ 과거 Backup Log 압축
```

구조가 생성되어 `compress`와 `delaycompress` 동작을 직접 확인하였다.

마지막으로 현재 Log가 0B인 상태에서 다시 강제 Rotation을 수행하였지만 File 구조가 변경되지 않아:

```text
notifempty
→ 빈 Log는 Rotation하지 않음
```

까지 검증하였다.

이번 실습을 통해 Linux Server에서 Log를 단순히 읽는 것을 넘어:

```text
필요한 Log 검색
실시간 Monitoring
Service 장애 분석
Log File Rotation
과거 Log 압축
Log 보관 정책
```

까지 기본적인 Log Management 흐름을 실습하였다.

---

## 관련 이론

`systemd-journald`, `journalctl`, Journal Priority, `rsyslog`, Syslog, `/var/log`, `logger`, `logrotate`, Rotation 정책, `compress`, `delaycompress`, `notifempty`, `create`, Log 보관 및 장애 분석 흐름은 `log-management-notes.md`에서 정리한다.
