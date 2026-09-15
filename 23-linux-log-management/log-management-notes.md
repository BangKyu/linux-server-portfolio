# Linux Log Management 이론 정리

## 1. Log란?

Log는 System, Service, Application에서 발생한 Event를 기록한 정보이다.

예:

```text
Service 시작 / 종료
Login 성공 / 실패
Application Error
Network 문제
Disk 오류
Cron 실행
Package 설치
Kernel Event
```

Server 장애 분석에서 Log는 매우 중요한 자료이다.

---

# 2. Log가 필요한 이유

Server에 문제가 발생했을 때 현재 상태만 보면 원인을 알기 어려울 수 있다.

예:

```text
현재
Service Down
```

만 확인하면:

```text
왜 중지되었는가?
언제 중지되었는가?
어떤 Error가 발생했는가?
누가 설정을 변경했는가?
```

를 알기 어렵다.

Log를 이용하면 과거 Event를 추적할 수 있다.

---

# 3. Linux Log 기본 구조

Rocky Linux에서는 대표적으로:

```text
systemd-journald
rsyslog
/var/log
logrotate
```

를 이용하여 Log를 수집하고 관리한다.

구조:

```text
Kernel / Service / Application
             |
             ↓
     systemd-journald
             |
             ↓
        Journal 저장
             |
             ↓
        journalctl

일부 Log
             |
             ↓
          rsyslog
             |
             ↓
         /var/log
             |
             ↓
         logrotate
```

---

# 4. systemd-journald

`systemd-journald`는 systemd 기반 Linux에서 Log를 수집하는 Service이다.

확인:

```bash
systemctl status systemd-journald
```

간단 확인:

```bash
systemctl is-active systemd-journald
```

이번 환경:

```text
active
```

---

# 5. journald가 수집하는 정보

대표적으로:

```text
Kernel Message
systemd Service Output
Application Log
Syslog Message
Boot Log
Process Metadata
PID
UID
Service Unit
Priority
```

등을 수집할 수 있다.

---

# 6. Journal

`systemd-journald`가 관리하는 Log 저장 영역을 Journal이라고 한다.

Journal은 단순 Text File과 달리 여러 Metadata를 함께 저장할 수 있다.

예:

```text
Timestamp
Hostname
PID
UID
Unit
Priority
Identifier
Message
```

따라서 다양한 조건으로 Log를 검색할 수 있다.

---

# 7. journalctl

Journal을 조회하는 대표 Command:

```bash
journalctl
```

아무 Option 없이 실행하면 많은 Log가 출력될 수 있다.

운영 환경에서는 필요한 조건을 지정하여 범위를 줄이는 것이 중요하다.

---

# 8. 최근 Log 조회

```bash
journalctl -n 20
```

마지막 20개 Log를 확인한다.

Pager 없이:

```bash
journalctl -n 20 --no-pager
```

---

# 9. --no-pager

```text
--no-pager
```

는 `less` 등의 Pager를 사용하지 않고 바로 Terminal에 출력한다.

Script나 짧은 Log 확인에서 편리하다.

---

# 10. Boot 기준 조회

현재 Boot 이후 Log:

```bash
journalctl -b
```

최근 Log만:

```bash
journalctl -b -n 20 --no-pager
```

이번 실습에서도 현재 Boot 기준 Log를 확인하였다.

---

# 11. Boot Log가 중요한 이유

Server 재부팅 후 문제가 발생했다면:

```text
Boot 시작
        ↓
Kernel 초기화
        ↓
Service 시작
        ↓
Error 발생
```

흐름을 확인해야 한다.

이때:

```bash
journalctl -b
```

가 유용하다.

---

# 12. 이전 Boot 조회

환경에 따라 이전 Boot의 Journal이 보존되어 있다면:

```bash
journalctl --list-boots
```

로 Boot 목록을 확인할 수 있다.

예:

```bash
journalctl -b -1
```

은 이전 Boot Log를 조회하는 데 사용할 수 있다.

---

# 13. Service별 Log

특정 systemd Unit의 Log:

```bash
journalctl -u SERVICE
```

예:

```bash
journalctl -u sshd
journalctl -u crond
journalctl -u named
journalctl -u httpd
```

---

# 14. -u Option

```text
-u
→ systemd Unit 기준 조회
```

예:

```bash
journalctl -u crond
```

이면 crond 관련 Log만 볼 수 있다.

---

# 15. Service 장애 분석 흐름

예:

```text
httpd 장애
   ↓
systemctl status httpd
   ↓
journalctl -u httpd
   ↓
최근 Error 확인
   ↓
설정 File 확인
```

전체 Journal을 보는 것보다 Service 단위로 범위를 좁히는 것이 효율적이다.

---

# 16. 시간 기준 조회

최근 10분:

```bash
journalctl --since "10 minutes ago"
```

최근 1시간:

```bash
journalctl --since "1 hour ago"
```

---

# 17. 특정 시간 범위

```bash
journalctl \
--since "2026-09-15 10:54:00" \
--until "2026-09-15 10:59:00"
```

장애 발생 시간을 알고 있으면 매우 유용하다.

---

# 18. 시간 + Service 결합

```bash
journalctl \
-u crond \
--since "10 minutes ago" \
--no-pager
```

조건을 여러 개 함께 사용할 수 있다.

즉:

```text
Service
+
시간
```

으로 검색 범위를 더 줄일 수 있다.

---

# 19. Log Priority

Syslog 계열 Log에는 Priority 또는 Severity 개념이 있다.

대표 순서:

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

위쪽일수록 심각한 수준이다.

---

# 20. emerg

```text
emerg
```

System을 정상적으로 사용할 수 없을 정도의 매우 심각한 상황.

---

# 21. alert

```text
alert
```

즉각적인 조치가 필요한 수준.

---

# 22. crit

```text
crit
```

Critical 상태.

---

# 23. err

```text
err
```

Error 수준.

---

# 24. warning

```text
warning
```

경고 수준.

즉시 장애는 아니더라도 주의가 필요한 상태일 수 있다.

---

# 25. notice

```text
notice
```

정상 동작이지만 확인할 가치가 있는 Event.

---

# 26. info

```text
info
```

일반 정보성 Message.

---

# 27. debug

```text
debug
```

Troubleshooting이나 개발 과정에서 사용하는 상세 Debug 정보.

---

# 28. Priority 조회

Error 이상:

```bash
journalctl -p err
```

현재 Boot에서 Error 이상:

```bash
journalctl -b -p err --no-pager
```

---

# 29. -p err의 의미

```text
-p err
```

는 `err`만 정확히 한 종류를 의미하는 것이 아니라 일반적으로 해당 수준과 더 심각한 Priority까지 포함해서 조회한다.

즉:

```text
emerg
alert
crit
err
```

등을 확인할 수 있다.

---

# 30. 실제 Error Log

이번 실습에서는:

```text
sshd-session:
maximum authentication attempts exceeded

systemd:
Failed to start dnf makecache.
```

등의 Error를 확인하였다.

---

# 31. Error Log가 있다고 항상 현재 장애인가?

아니다.

Journal에는 과거 Error도 남아 있을 수 있다.

따라서:

```text
발생 시간
현재 Service 상태
재현 여부
관련 Log
```

를 함께 확인해야 한다.

---

# 32. PID 기준 조회

특정 PID Log:

```bash
journalctl _PID=1237
```

이번 실습:

```text
named PID
1237
```

에 대한 Log를 조회하였다.

---

# 33. PID 필터 장점

특정 Process Instance만 확인 가능하다.

예:

```text
Service에 여러 Process 존재
        ↓
특정 PID만 문제
        ↓
_PID 기준 조회
```

---

# 34. PID 필터 단점

Process가 재시작되면 PID가 변경된다.

예:

```text
named PID 1237
        ↓
Service restart
        ↓
named PID 5000
```

따라서 장기적인 Service 분석에는:

```bash
journalctl -u named
```

가 더 적합할 수 있다.

---

# 35. Identifier

Application이나 `logger`가 Identifier를 지정하여 Log를 기록할 수 있다.

예:

```bash
logger -t portfolio-log-test "journalctl live test"
```

여기서:

```text
portfolio-log-test
```

가 Identifier이다.

---

# 36. Identifier 기준 조회

```bash
journalctl -t portfolio-log-test
```

실제:

```text
Server-A portfolio-log-test[4270]:
journalctl live test
```

를 확인하였다.

---

# 37. logger

`logger`는 Shell에서 Syslog / Journal Message를 생성할 수 있는 Command이다.

기본:

```bash
logger "test message"
```

Tag 지정:

```bash
logger -t test-app "test message"
```

---

# 38. logger 활용

다음과 같은 경우 유용하다.

```text
Log 실습
Shell Script Log 기록
Syslog Test
Monitoring Test
Log Pipeline Test
```

---

# 39. 실시간 Journal

```bash
journalctl -f
```

의:

```text
-f
→ Follow
```

새로운 Log가 발생하면 계속 출력한다.

---

# 40. 실시간 Monitoring 예

```bash
journalctl -f -u sshd
```

SSH Login을 시도하면서 실시간으로 Log를 확인할 수 있다.

또는:

```bash
journalctl -f -u httpd
```

로 Web Service Event를 관찰할 수 있다.

---

# 41. journalctl -f와 tail -f

Journal:

```bash
journalctl -f
```

Text File Log:

```bash
tail -f /var/log/application.log
```

둘 다 새로운 Log를 실시간으로 확인하는 데 사용한다.

---

# 42. 너무 많은 Log 출력 방지

운영 Server에서:

```bash
journalctl
```

전체를 보는 것은 비효율적일 수 있다.

추천:

```text
Service
시간
Priority
PID
Identifier
마지막 N줄
```

을 조합한다.

예:

```bash
journalctl \
-u sshd \
--since "10 minutes ago" \
-p warning \
--no-pager
```

---

# 43. rsyslog

`rsyslog`는 Linux에서 Syslog Message를 처리하고 File이나 Remote Server 등에 전달할 수 있는 Logging System이다.

확인:

```bash
rpm -q rsyslog
systemctl status rsyslog
```

---

# 44. journald와 rsyslog

둘은 동일한 것이 아니다.

간단히:

```text
systemd-journald
→ systemd Journal 중심 Log 수집

rsyslog
→ Syslog 규칙 기반 Log 처리
→ File 저장
→ Remote 전송 가능
```

환경에 따라 함께 사용한다.

---

# 45. /var/log

전통적인 Linux File Log가 저장되는 대표 Directory:

```text
/var/log
```

확인:

```bash
ls -lh /var/log
```

---

# 46. 주요 Log File

환경에 따라 다음과 같은 File이 있다.

```text
/var/log/messages
/var/log/secure
/var/log/cron
/var/log/maillog
/var/log/boot.log
/var/log/dnf.log
/var/log/audit/audit.log
```

배포판과 설정에 따라 실제 File 구성은 다를 수 있다.

---

# 47. /var/log/cron

Cron 관련 File Log.

이번 환경에서도:

```text
/var/log/cron
```

이 존재하였다.

---

# 48. /var/log/secure

환경에 따라 Authentication / Security 관련 Log가 기록될 수 있다.

예:

```text
SSH Login
sudo
Authentication
```

Rocky Linux 계열에서 자주 확인하는 Log 중 하나이다.

---

# 49. /var/log/messages

일반적인 System Message가 저장될 수 있다.

다만 journald / rsyslog 설정에 따라 실제 구성은 달라질 수 있다.

---

# 50. /var/log/audit

SELinux나 Audit 관련 Log:

```text
/var/log/audit/audit.log
```

SELinux Troubleshooting에서 매우 중요하다.

---

# 51. Log File 확인 Command

전체 보기:

```bash
cat FILE
```

한 화면씩:

```bash
less FILE
```

최근 Log:

```bash
tail FILE
```

최근 20줄:

```bash
tail -n 20 FILE
```

실시간:

```bash
tail -f FILE
```

---

# 52. grep과 Log

특정 문자열 검색:

```bash
grep ERROR logfile
```

대소문자 무시:

```bash
grep -i error logfile
```

여러 조건:

```bash
grep -E 'error|failed|denied' logfile
```

---

# 53. Log File이 계속 커지는 문제

Application이 계속 Log를 쓰면:

```text
10 MB
100 MB
1 GB
10 GB
...
```

처럼 계속 증가할 수 있다.

결과:

```text
Disk 공간 부족
        ↓
File System Full
        ↓
Application 장애
```

로 이어질 수 있다.

---

# 54. logrotate

`logrotate`는 Log File을 주기적으로 회전시키고 오래된 Log를 압축 / 삭제하는 Tool이다.

예:

```text
app.log
app.log.1
app.log.2.gz
app.log.3.gz
```

---

# 55. logrotate Package 확인

```bash
rpm -q logrotate
```

이번 환경:

```text
logrotate-3.18.0-12.el9.x86_64
```

---

# 56. logrotate 설정 위치

Main Configuration:

```text
/etc/logrotate.conf
```

Service별 Configuration:

```text
/etc/logrotate.d/
```

이번 실습:

```text
/etc/logrotate.d/rsync-server-b-backup
```

---

# 57. 이번 설정

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

# 58. daily

```text
daily
```

Daily 기준으로 Rotation 필요 여부를 판단한다.

즉 매 실행마다 무조건 Rotation하는 의미가 아니다.

---

# 59. weekly

예:

```text
weekly
```

주 단위 Rotation 정책.

---

# 60. monthly

```text
monthly
```

월 단위 Rotation 정책.

---

# 61. size

크기를 기준으로 Rotation할 수도 있다.

예:

```text
size 100M
```

Log가 특정 Size 조건에 도달했을 때 Rotation하는 방식이다.

실제 정책은 Service 특성에 맞게 설정한다.

---

# 62. rotate 7

```text
rotate 7
```

Rotation된 과거 Log를 최대 7개까지 보관하도록 한다.

예:

```text
.log.1
.log.2.gz
.log.3.gz
...
```

보관 개수를 초과한 오래된 Log는 삭제될 수 있다.

---

# 63. compress

```text
compress
```

오래된 Log를 gzip으로 압축한다.

예:

```text
application.log.2.gz
```

---

# 64. Log 압축 장점

Text Log는 압축률이 좋은 경우가 많다.

따라서:

```text
Disk 공간 절약
오래된 Log 보존
```

에 도움이 된다.

---

# 65. gzip Log 읽기

압축 해제 없이:

```bash
zcat FILE.gz
```

또는:

```bash
gzip -cd FILE.gz
```

사용 가능.

---

# 66. delaycompress

```text
delaycompress
```

가 있으면 가장 최근 Rotation File은 바로 압축하지 않는다.

구조:

```text
현재
application.log

직전
application.log.1

그 이전
application.log.2.gz
```

---

# 67. delaycompress가 필요한 이유

일부 Application이 Rotation 직후에도 기존 Log File을 잠시 참조할 수 있는 경우가 있다.

또는 최근 Log를 일반 Text 형태로 바로 확인하고 싶은 경우 유용하다.

---

# 68. compress + delaycompress 흐름

```text
초기
app.log
   ↓
1차 Rotation
   ↓
app.log
app.log.1
   ↓
2차 Rotation
   ↓
app.log
app.log.1
app.log.2.gz
```

이번 실습에서 실제로 이 구조를 확인하였다.

---

# 69. missingok

```text
missingok
```

대상 Log File이 없어도 Error로 취급하지 않는다.

예:

```text
Service 미실행
Log File 없음
```

이어도 logrotate 전체 작업이 불필요하게 실패하는 것을 줄일 수 있다.

---

# 70. notifempty

```text
notifempty
```

Log File이 비어 있으면 Rotation하지 않는다.

이번 실습에서:

```text
현재 Log = 0 byte
```

인 상태에서 강제 실행했지만 Rotation이 발생하지 않았다.

---

# 71. create

```text
create 0640 root root
```

Rotation 후 새로운 Log File을 생성한다.

구조:

```text
기존 Log
→ .1

새 Log
→ 지정 Permission / Owner / Group으로 생성
```

---

# 72. Permission 0640

```text
0 6 4 0

Owner
6 = rw-

Group
4 = r--

Other
0 = ---
```

즉:

```text
root
→ Read + Write

root Group
→ Read

Other
→ 접근 불가
```

---

# 73. nodateext

기본 환경에서는 날짜가 포함된 Rotation File 이름을 사용할 수도 있다.

이번에는:

```text
nodateext
```

를 사용하여:

```text
.log.1
.log.2.gz
```

형태로 확인하였다.

---

# 74. dateext

반대로 날짜 기반 이름을 사용하는 Option도 있다.

예:

```text
application.log-20260915
```

정확한 Naming 정책은 Configuration에 따라 달라진다.

---

# 75. logrotate -d

```bash
logrotate -d CONFIG
```

`-d`는 Debug Mode.

실제 File을 변경하지 않는다.

---

# 76. Debug Mode가 중요한 이유

설정 작성 후 바로 Rotation하지 않고:

```text
Syntax
대상 File
Rotation 조건
State
```

등을 먼저 확인할 수 있다.

추천 흐름:

```text
설정 작성
   ↓
logrotate -d
   ↓
판단 확인
   ↓
실제 적용
```

---

# 77. 이번 Debug 결과

실제:

```text
rotating pattern:
after 1 days
(7 rotations)

log does not need rotating
```

를 확인하였다.

---

# 78. logrotate State

logrotate는 이전 Rotation 상태를 저장하여:

```text
마지막 Rotation 시간
```

등을 기준으로 다음 Rotation 필요 여부를 판단한다.

따라서:

```text
daily
```

설정이라고 해서 Command 실행 시마다 무조건 회전하지 않는다.

---

# 79. logrotate -f

```bash
logrotate -f CONFIG
```

`-f`:

```text
Force
```

조건과 관계없이 강제로 Rotation을 시도한다.

---

# 80. -f 사용 주의

운영 환경에서는 실제 Log File이 Rotation될 수 있다.

따라서:

```text
Test 환경
대상 File 확인
Config 확인
```

후 사용하는 것이 좋다.

---

# 81. 첫 번째 Rotation

실습 전:

```text
rsync-server-b-backup.log
494B
```

강제 Rotation 후:

```text
rsync-server-b-backup.log
0B

rsync-server-b-backup.log.1
494B
```

---

# 82. 새 Log 생성

현재 Log:

```text
-rw-r----- root root
```

로 생성되어:

```text
create 0640 root root
```

가 적용된 것을 확인하였다.

---

# 83. 첫 Rotation에서 .gz가 없는 이유

```text
delaycompress
```

때문이다.

즉 직전 Log:

```text
.log.1
```

은 아직 Text 상태로 유지된다.

---

# 84. 두 번째 Rotation

현재 Log에:

```text
second rotation test
```

를 기록한 뒤 다시 Rotation하였다.

결과:

```text
.log
→ 0B

.log.1
→ 21B
→ second rotation test

.log.2.gz
→ 최초 Backup Log
```

---

# 85. 실제 delaycompress 확인

```text
1차
.log.1
압축 안 됨

2차
기존 .log.1
→ .log.2.gz
```

따라서 설정이 실제 동작함을 확인하였다.

---

# 86. notifempty 실제 확인

현재:

```text
rsync-server-b-backup.log
0B
```

상태에서 다시:

```bash
logrotate -f ...
```

실행.

File 구조가 그대로 유지되었다.

```text
.log
.log.1
.log.2.gz
```

즉 빈 File은 Rotation하지 않았다.

---

# 87. Log Rotation과 Application

중요한 개념이다.

Application이 Log File을 계속 Open한 상태에서 단순히 File 이름만 변경하면 Application이 예전 File Descriptor를 계속 사용할 수도 있다.

따라서 Service에 따라:

```text
postrotate
prerotate
copytruncate
Service Reload
```

등이 필요할 수 있다.

---

# 88. postrotate

Rotation 후 특정 Command를 실행할 수 있다.

예시 개념:

```text
postrotate
    Service reload
endscript
```

Application이 새 Log File을 다시 열도록 하는 데 사용할 수 있다.

---

# 89. copytruncate

Application이 Log File을 다시 Open할 수 없는 경우 사용하는 방식 중 하나이다.

개념:

```text
현재 Log 내용 Copy
        ↓
원본 Log 내용 Truncate
```

하지만 Copy와 Truncate 사이에 짧은 Log 유실 가능성이 있을 수 있어 무조건 좋은 방식은 아니다.

---

# 90. 이번 rsync Log에서 create 사용이 가능한 이유

이번 Backup Log는 cron Script 실행 시:

```bash
>> /var/log/rsync-server-b-backup.log
```

로 매 실행마다 File을 Append Open한다.

즉 장시간 Process가 Log File Descriptor를 계속 잡고 있는 Service와는 성격이 다르다.

그래서 Rotation 후 생성된 새 Log File에 다음 cron 실행이 다시 기록될 수 있다.

---

# 91. Log Ownership

Log File을 생성할 때 Owner / Group이 잘못되면 Application이 새 Log에 쓰지 못할 수 있다.

따라서:

```text
Application 실행 User
Log Owner
Log Group
Permission
```

을 확인해야 한다.

---

# 92. Log Rotation 후 Permission 확인

```bash
stat -c '%A %U %G %n' FILE
```

이번:

```text
-rw-r----- root root
```

확인.

---

# 93. Disk 사용량 확인

Log 문제 분석 시:

```bash
df -h
```

로 File System 사용량을 확인한다.

특정 Log Directory:

```bash
du -sh /var/log
```

큰 File:

```bash
du -ah /var/log | sort -h | tail
```

등을 사용할 수 있다.

---

# 94. Disk Full과 Log

예:

```text
Application Error 반복
        ↓
Log 폭증
        ↓
/var 사용량 증가
        ↓
Disk 100%
        ↓
새 Log 기록 실패
        ↓
다른 Service까지 영향
```

가능하다.

---

# 95. Log 폭증 원인

대표적으로:

```text
Application Loop
Authentication Attack
Network Error 반복
Debug Logging 활성화
Service 재시작 반복
DNS Resolution 실패 반복
```

등이 있다.

단순히 Log를 삭제하기보다 원인을 해결해야 한다.

---

# 96. Log를 무조건 rm 하면 안 되는 이유

운영 Log는:

```text
장애 분석
보안 분석
감사
장애 발생 시각 확인
```

에 필요할 수 있다.

따라서:

```bash
rm -f huge.log
```

로 즉시 삭제하기 전에 보존 필요성과 Application 동작을 고려해야 한다.

---

# 97. 삭제된 Log와 Open File

Process가 File을 Open한 상태에서 Log File을 삭제하면 File 이름은 사라졌지만 Disk 공간이 바로 반환되지 않는 상황도 있을 수 있다.

이런 경우 Process가 File Descriptor를 닫아야 실제 공간이 반환될 수 있다.

따라서 Log 관리에는 Rotation 방식이 중요하다.

---

# 98. Log Troubleshooting 기본 순서

```text
장애 발생
   ↓
발생 시간 확인
   ↓
Service 상태 확인
   ↓
journalctl -u SERVICE
   ↓
Priority 확인
   ↓
Application File Log 확인
   ↓
관련 PID 확인
   ↓
Error 반복 여부 확인
   ↓
설정 / Resource / Network 확인
```

---

# 99. SSH 문제 예

```bash
systemctl status sshd
```

```bash
journalctl -u sshd \
--since "10 minutes ago" \
--no-pager
```

필요하면:

```bash
journalctl -u sshd -p warning
```

---

# 100. Cron 문제 예

```bash
systemctl status crond
```

```bash
journalctl -u crond
```

그리고 Application 자체 Log:

```bash
tail /var/log/rsync-server-b-backup.log
```

---

# 101. DNS 문제 예

```bash
journalctl -u named
```

이번 실습에서는 `named`에서:

```text
network unreachable resolving ...
```

Log를 확인하였다.

해당 Message는 IPv6 Resolution 시도와 현재 Network 상태 등을 함께 고려해야 한다.

---

# 102. 한 줄 Log만 보고 결론 내리지 않는다

예:

```text
network unreachable
```

한 줄만 보고:

```text
DNS 전체 장애
```

라고 바로 판단하면 안 된다.

함께 확인:

```text
Service active 여부
IPv4 Query 성공 여부
Client Resolution
관련 Network 설정
반복 빈도
```

등이 필요하다.

---

# 103. Log와 현재 상태 비교

과거 Error가 Journal에 남아 있어도 현재 Service가 정상일 수 있다.

따라서:

```text
Log
+
현재 상태
+
재현 테스트
```

를 함께 본다.

---

# 104. Log Timestamp

Log 분석에서 시간은 매우 중요하다.

확인할 사항:

```text
System Time
Timezone
NTP Synchronization
```

시간이 틀리면 여러 Server 간 Event 순서를 비교하기 어려워진다.

---

# 105. Chrony와 Log 분석

이전에 구성한 Chrony/NTP는 Log 분석에서도 중요하다.

예:

```text
Server-A
10:00:00

Server-B
09:55:00
```

처럼 시간이 다르면 장애 Event 순서 분석이 어려워진다.

---

# 106. 여러 Server의 Log

분산 System에서는:

```text
Web Server
DB Server
DNS Server
Backup Server
```

등 여러 Host의 Log를 동시에 분석해야 할 수 있다.

이때 Time Synchronization이 중요하다.

---

# 107. Centralized Logging

운영 환경에서는 여러 Server의 Log를 중앙 Log Server로 보낼 수 있다.

예:

```text
Server-A ─┐
Server-B ─┼→ Central Log Server
Server-C ─┘
```

rsyslog Remote Logging이나 별도 Logging Platform을 사용할 수 있다.

---

# 108. Centralized Logging 장점

```text
한 곳에서 검색
Server 장애 시에도 Log 보존 가능
보안 감사
여러 Server Event 비교
Monitoring 연동
```

등의 장점이 있다.

---

# 109. Journal과 Persistent Storage

Journal 저장 방식은 System 설정에 따라:

```text
Memory 중심
Disk Persistent
```

등으로 구성될 수 있다.

Persistent Journal이 구성되어야 Reboot 후 이전 Boot Log를 계속 확인할 수 있는 경우가 있다.

---

# 110. Journal Disk Usage

Journal 사용량 확인:

```bash
journalctl --disk-usage
```

오래된 Journal 정리가 필요한 환경에서는 Retention Policy를 고려해야 한다.

---

# 111. Journal Vacuum

환경에 따라 다음과 같은 방식으로 오래된 Journal을 정리할 수 있다.

예:

```bash
journalctl --vacuum-time=30d
```

또는 Size 기준 정책도 존재한다.

운영에서는 보존 정책을 확인한 후 사용해야 한다.

---

# 112. 로그 보존 정책

Log를 무한정 보존할 수는 없다.

고려:

```text
Disk 용량
보안 정책
감사 요구사항
장애 분석 기간
Compliance
```

등에 따라 Retention 기간을 정한다.

---

# 113. rotate 7의 의미 다시 정리

이번:

```text
daily
rotate 7
```

은 Daily Rotation File을 최대 7개 보관하도록 설정한 것이다.

하지만:

```text
빈 Log라 Rotation 안 됨
logrotate 실행 안 됨
```

등의 조건에 따라 실제 Calendar 기준 정확히 7일과 항상 동일하다고 단정하면 안 된다.

---

# 114. Logrotate와 Scheduler

logrotate 자체는 Rotation Logic을 담당한다.

실제 실행 주기는 System에서:

```text
systemd timer
cron
```

등을 통해 관리될 수 있다.

환경에 따라 실행 방식이 다를 수 있다.

---

# 115. logrotate 상태 확인

설정 Test:

```bash
logrotate -d /etc/logrotate.d/CONFIG
```

강제 Test:

```bash
logrotate -f /etc/logrotate.d/CONFIG
```

File 결과:

```bash
ls -lh LOGFILE*
```

---

# 116. Configuration Error

logrotate 설정 Syntax가 잘못되면 정상 Rotation되지 않을 수 있다.

따라서 새 설정 작성 후:

```bash
logrotate -d CONFIG
```

로 먼저 확인하는 습관이 좋다.

---

# 117. 현재 Log와 Old Log

현재 Application이 기록하는 File:

```text
application.log
```

Rotation된 과거 File:

```text
application.log.1
application.log.2.gz
```

현재 장애 분석은 최신 Log를 먼저 보고, 필요하면 과거 Rotation File까지 확인한다.

---

# 118. 압축 Log 검색

압축 Log에서도:

```bash
zgrep ERROR logfile.gz
```

등을 이용하여 검색할 수 있다.

즉 반드시 압축을 풀 필요는 없다.

---

# 119. 여러 Rotation Log 검색

예:

```bash
zgrep -i error /var/log/application.log.*.gz
```

과거 Error를 찾는 데 활용할 수 있다.

---

# 120. Log 분석 시 기본 질문

```text
언제 발생했는가?
어떤 Service인가?
어떤 Host인가?
어떤 PID인가?
어떤 User인가?
Error Level은 무엇인가?
한 번인가 반복인가?
장애 전후 어떤 Event가 있었는가?
```

---

# 121. 반복 Log

같은 Error가 초당 수백 번 반복되면:

```text
Log 자체가 장애 원인
```

이 될 수도 있다.

예:

```text
Disk Write 증가
Log File 폭증
CPU 증가
```

따라서 Error 원인과 Log Rate를 함께 확인한다.

---

# 122. journalctl 기본 Command 정리

현재 Boot:

```bash
journalctl -b
```

최근 20개:

```bash
journalctl -n 20
```

Service:

```bash
journalctl -u SERVICE
```

최근 시간:

```bash
journalctl --since "10 minutes ago"
```

시간 범위:

```bash
journalctl \
--since "YYYY-MM-DD HH:MM:SS" \
--until "YYYY-MM-DD HH:MM:SS"
```

---

# 123. Priority Command

```bash
journalctl -p err
```

현재 Boot:

```bash
journalctl -b -p err
```

---

# 124. PID / Identifier

PID:

```bash
journalctl _PID=PID
```

Identifier:

```bash
journalctl -t TAG
```

---

# 125. 실시간

전체:

```bash
journalctl -f
```

Service:

```bash
journalctl -f -u SERVICE
```

---

# 126. File Log Command 정리

```bash
cat FILE
less FILE
tail FILE
tail -n 20 FILE
tail -f FILE
grep PATTERN FILE
```

---

# 127. logrotate 주요 Option 정리

```text
daily
→ Daily Rotation

weekly
→ Weekly Rotation

monthly
→ Monthly Rotation

rotate N
→ 과거 Log N개 보관

compress
→ gzip 압축

delaycompress
→ 직전 Log 압축을 한 Cycle 늦춤

missingok
→ File 없어도 Error 무시

notifempty
→ 빈 Log Rotation 안 함

create
→ Rotation 후 새 Log 생성

nodateext
→ 번호 기반 이름 사용
```

---

# 128. Debug / Force

```bash
logrotate -d CONFIG
```

```text
Debug
실제 변경 없음
```

```bash
logrotate -f CONFIG
```

```text
Force Rotation
실제 변경 가능
```

---

# 129. 이번 실습 흐름

```text
journald 상태 확인
        ↓
rsyslog 상태 확인
        ↓
/var/log 확인
        ↓
journalctl Boot Log
        ↓
crond Unit Log
        ↓
Error Priority
        ↓
named PID Log
        ↓
logger Test
        ↓
실시간 Follow
        ↓
Backup Log 확인
        ↓
logrotate 설정
        ↓
Debug
        ↓
1차 Rotation
        ↓
create 확인
        ↓
2차 Rotation
        ↓
compress / delaycompress 확인
        ↓
빈 Log Rotation
        ↓
notifempty 확인
```

---

# 130. 장애 분석 핵심 흐름

```text
장애 발견
   ↓
현재 Service 상태
systemctl status
   ↓
최근 Journal
journalctl -u
   ↓
발생 시간 확인
--since / --until
   ↓
Error Priority 확인
-p
   ↓
Application File Log 확인
   ↓
PID / User / Resource 확인
   ↓
설정 / Network / Disk / Permission 확인
   ↓
원인 판단
```

---

# 131. Log 관리와 Troubleshooting 차이

Log Management:

```text
Log 수집
저장
보존
압축
삭제 정책
```

Troubleshooting:

```text
필요한 Log 검색
Event 연결
Error 원인 분석
```

두 영역은 서로 연결되어 있다.

---

# 132. 좋은 Log 관리의 목적

```text
필요한 순간
필요한 Log를
쉽게 찾고
Disk를 과도하게 사용하지 않으면서
필요한 기간만큼 보존하는 것
```

이 핵심이다.

---

# 133. 이번 실습 핵심

Journal:

```text
systemd-journald
→ Log 수집

journalctl
→ Journal 조회
```

File Log:

```text
rsyslog
→ Syslog 처리

/var/log
→ Text Log 저장
```

Rotation:

```text
logrotate
→ Log File 크기 / 보관 관리
```

---

# 134. 최종 정리

Linux Server에서 Log는 단순 Text 기록이 아니라 장애 분석과 운영 상태 추적의 핵심 자료이다.

이번 실습에서는:

```text
systemd-journald
journalctl
rsyslog
logger
/var/log
logrotate
```

를 연결하여 전체 Log 관리 흐름을 확인하였다.

`journalctl`에서는:

```text
Boot
Service
시간
Priority
PID
Identifier
실시간 Follow
```

조건을 이용하여 필요한 Log만 조회하였다.

`logrotate`에서는:

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

를 구성하였다.

실제 Rotation 결과:

```text
현재 Log
rsync-server-b-backup.log

직전 Log
rsync-server-b-backup.log.1

과거 압축 Log
rsync-server-b-backup.log.2.gz
```

를 확인하였다.

또한:

```text
create
→ 새 Log Permission 적용

delaycompress
→ 직전 Log는 Text 유지

compress
→ 이전 Log gzip 압축

notifempty
→ Empty Log Rotation 방지
```

까지 실제 동작을 검증하였다.

따라서 Server 운영 시:

```text
문제 발생
        ↓
Log 범위 좁히기
        ↓
관련 Event 분석
        ↓
과거 Log 확인
        ↓
원인 파악
```

과 동시에:

```text
Log 증가
        ↓
Rotation
        ↓
압축
        ↓
보관
        ↓
오래된 Log 삭제
```

를 관리하는 것이 중요하다.
