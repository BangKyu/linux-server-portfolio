# rsync + cron Backup 이론 정리

## 1. rsync란?

`rsync`는 File과 Directory를 동기화하는 Command이다.

단순 복사뿐 아니라 Source와 Destination을 비교하여 변경된 File 중심으로 동기화할 수 있다.

기본 구조:

```bash
rsync [옵션] [원본] [대상]
```

예:

```bash
rsync -av /source/ /backup/
```

의미:

```text
/source/ 내부 내용을
/backup/으로 동기화
```

---

# 2. rsync의 특징

대표적인 특징:

```text
Local 복사 가능
Remote 복사 가능
변경된 File 중심으로 동기화
Directory 재귀 복사 가능
File 속성 보존 가능
SSH와 함께 사용 가능
삭제 동기화 가능
Dry Run 가능
```

---

# 3. cp와 rsync 차이

`cp`:

```text
기본적으로 File을 복사
```

`rsync`:

```text
Source와 Destination 비교
        ↓
변경 사항 확인
        ↓
필요한 File 중심으로 동기화
```

따라서 반복 Backup이나 Server 간 동기화에 `rsync`가 많이 사용된다.

---

# 4. Local rsync

같은 Server 안에서도 사용할 수 있다.

예:

```bash
rsync -av /root/rsync-lab/source/ /root/rsync-lab/local-backup/
```

구조:

```text
Server-A

/root/rsync-lab/source/
        ↓
rsync
        ↓
/root/rsync-lab/local-backup/
```

---

# 5. Remote rsync

Remote Server로도 동기화할 수 있다.

예:

```bash
rsync -av \
/source/ \
user@192.168.111.200:/backup/
```

구조:

```text
Local Server
/source/
        |
        | Network
        ↓
Remote Server
/backup/
```

---

# 6. rsync over SSH

실무에서는 Remote rsync를 SSH와 함께 사용하는 경우가 많다.

이번 실습:

```bash
rsync -av \
-e "ssh -i /root/.ssh/id_ed25519_backup_server_b -o IdentitiesOnly=yes" \
/root/rsync-lab/source/ \
backupuser@192.168.111.200:/backup/server-a/
```

구조:

```text
Server-A
/root/rsync-lab/source/
        |
        | rsync
        | over SSH
        ↓
Server-B
/backup/server-a/
```

---

# 7. Remote Destination 형식

Remote 경로:

```text
backupuser@192.168.111.200:/backup/server-a/
```

구조:

```text
backupuser
→ Remote Login User

192.168.111.200
→ Remote Server IP

/backup/server-a/
→ Remote Destination Directory
```

---

# 8. -e Option

```bash
-e "ssh ..."
```

`-e`는 rsync가 Remote Shell로 어떤 Command를 사용할지 지정할 때 사용한다.

이번 실습에서는:

```text
SSH
```

를 사용하였다.

---

# 9. SSH Identity File

```bash
-i /root/.ssh/id_ed25519_backup_server_b
```

의미:

```text
SSH 인증에 사용할 Private Key 지정
```

---

# 10. IdentitiesOnly

```bash
-o IdentitiesOnly=yes
```

지정한 SSH Key만 인증에 사용하도록 한다.

여러 SSH Key가 존재하는 환경에서 어떤 Key를 사용할지 명확하게 할 수 있다.

---

# 11. Source 마지막 `/`

rsync에서 매우 중요하다.

다음 두 형태는 의미가 다를 수 있다.

```bash
rsync -av /source/ /backup/
```

```bash
rsync -av /source /backup/
```

---

# 12. source/

```bash
/source/
```

마지막 `/`가 있으면:

```text
source Directory 내부 내용
```

을 의미한다.

예:

```text
/source/
├── file1
└── file2
```

실행:

```bash
rsync -av /source/ /backup/
```

결과:

```text
/backup/
├── file1
└── file2
```

---

# 13. source

마지막 `/` 없이:

```bash
/source
```

를 지정하면 Directory 자체를 대상으로 처리하는 형태가 될 수 있다.

결과 구조가:

```text
/backup/source/
```

처럼 만들어질 수 있다.

따라서 rsync 사용 전:

```text
Directory 자체를 복사할 것인지
Directory 내부를 복사할 것인지
```

를 명확하게 해야 한다.

---

# 14. -a Option

```bash
-a
```

는 Archive Mode이다.

대표적으로:

```text
Directory 재귀 복사
Symbolic Link 보존
Permission 보존
Timestamp 보존
Owner / Group 등의 속성 보존 시도
```

등의 기능을 묶어서 사용한다.

---

# 15. -v Option

```bash
-v
```

Verbose Mode.

진행되는 File 등을 출력한다.

예:

```text
sending incremental file list
file1.txt
file2.txt
subdir/
```

---

# 16. 최초 rsync

Destination에 File이 없는 상태에서 처음 실행하면 대부분의 Source File이 복사된다.

예:

```text
Source
file1
file2
subfile
```

Destination:

```text
비어 있음
```

실행 후:

```text
전체 File 전송
```

---

# 17. Incremental 동작

같은 rsync를 다시 실행했는데 변경된 File이 없다면:

```text
sending incremental file list
```

만 나오고 실제 File 이름은 나타나지 않을 수 있다.

즉:

```text
변경 없음
→ 다시 File 전송할 필요 없음
```

이다.

---

# 18. File 수정 시

Source의 File 내용이 변경되면 해당 File이 다시 동기화된다.

예:

```text
file1.txt 변경
```

다시 rsync:

```text
file1.txt
```

이 전송 목록에 나타난다.

---

# 19. File 추가 시

새로운 File이 Source에 생기면 다음 rsync에서 Destination으로 복사된다.

예:

```text
Source
file4.txt 추가
        ↓
rsync
        ↓
Destination
file4.txt 생성
```

---

# 20. 기본 rsync의 삭제 동작

기본적으로 Source에서 File을 삭제했다고 Destination에서도 자동 삭제되는 것은 아니다.

예:

```text
Source
file2.txt 삭제

Destination
file2.txt 존재
```

일반 rsync:

```bash
rsync -av source/ backup/
```

결과:

```text
Destination의 file2.txt는 남아 있을 수 있음
```

---

# 21. --delete

Source에 없는 File을 Destination에서도 삭제하려면:

```bash
--delete
```

를 사용할 수 있다.

예:

```bash
rsync -av --delete source/ backup/
```

결과:

```text
Source에 없는 File
→ Destination에서도 삭제
```

---

# 22. --delete의 위험성

`--delete`는 강력하지만 위험하다.

예:

```text
잘못된 Source 지정
        ↓
Destination과 비교
        ↓
필요한 File을 Source에 없는 File로 판단
        ↓
삭제 가능
```

따라서 운영 환경에서는 경로 확인이 중요하다.

---

# 23. --dry-run

실제 변경 없이 어떤 작업이 수행될지 미리 확인할 수 있다.

```bash
rsync -av --delete --dry-run source/ backup/
```

예:

```text
deleting file2.txt
(DRY RUN)
```

실제 삭제는 발생하지 않는다.

---

# 24. --dry-run 활용

특히 다음 상황에서 유용하다.

```text
--delete 사용 전
대량 File 이동 전
Remote Backup 변경 전
경로가 확실하지 않을 때
```

추천 흐름:

```text
명령 작성
        ↓
--dry-run
        ↓
결과 확인
        ↓
실제 실행
```

---

# 25. Mirror Backup

이번 구성은:

```bash
rsync -av --delete
```

를 사용한다.

따라서 Destination을 Source와 최대한 동일하게 유지한다.

구조:

```text
Source 상태
        ↓
rsync --delete
        ↓
Destination 상태 동기화
```

이를 Mirror 성격의 Backup으로 볼 수 있다.

---

# 26. Mirror와 Version Backup 차이

Mirror:

```text
현재 Source 상태를 그대로 복제
```

Source에서 삭제:

```text
Destination에서도 삭제 가능
```

Version Backup:

```text
과거 Version 보존
```

예:

```text
오늘 File
어제 File
일주일 전 File
```

등을 별도로 보존할 수 있다.

---

# 27. Mirror는 완전한 Backup인가?

Mirror만으로 모든 Backup 요구사항을 만족하지는 않는다.

예를 들어:

```text
사용자가 Source에서 실수로 File 삭제
        ↓
Mirror 실행
        ↓
Backup에서도 삭제
```

될 수 있다.

따라서 중요한 운영 환경에서는:

```text
Version 관리
Snapshot
Retention Policy
Offline Backup
Remote Backup
```

등을 함께 고려한다.

---

# 28. Local과 Remote 양쪽 rsync 필요

이번 실습에서 실제 오류:

```text
bash: rsync: command not found
rsync: connection unexpectedly closed
rsync error: error in rsync protocol data stream
```

가 발생하였다.

원인은 Remote Server에 `rsync`가 없었기 때문이다.

---

# 29. 왜 Remote에도 rsync가 필요한가?

`rsync over SSH`는 구조상:

```text
Server-A
Local rsync
        |
        | SSH
        ↓
Server-B
Remote rsync
```

처럼 양쪽에서 rsync가 동작한다.

SSH는 전송 Channel 역할을 한다.

따라서:

```text
Local rsync 설치
Remote rsync 설치
```

가 모두 필요하다.

---

# 30. rsync 설치 확인

```bash
rpm -q rsync
```

또는:

```bash
which rsync
```

---

# 31. Backup 전용 User

이번 실습에서는 Server-B에:

```text
backupuser
```

를 생성하였다.

Backup Destination:

```text
/backup/server-a/
```

Owner:

```text
backupuser:backupuser
```

Permission:

```text
750
```

---

# 32. 왜 Backup 전용 User를 사용하는가?

Remote Backup을 무조건 `root` 계정으로 수행하면 권한 범위가 너무 커질 수 있다.

Backup 전용 User를 사용하면:

```text
접근 가능한 Directory 제한
권한 분리
관리 목적 명확화
```

등의 장점이 있다.

---

# 33. SSH Key Authentication

자동 Backup에서는 매번 Password를 입력할 수 없다.

따라서 SSH Key 인증을 사용한다.

구조:

```text
Server-A
Private Key
        ↓
SSH Authentication
        ↓
Server-B
Authorized Public Key
```

---

# 34. SSH Key 생성

예:

```bash
ssh-keygen -t ed25519 \
-f /root/.ssh/id_ed25519_backup_server_b \
-N ''
```

구성:

```text
Private Key
/root/.ssh/id_ed25519_backup_server_b

Public Key
/root/.ssh/id_ed25519_backup_server_b.pub
```

---

# 35. Private Key와 Public Key

Private Key:

```text
Client 측 보관
외부 유출 금지
```

Public Key:

```text
Remote Server에 등록 가능
```

---

# 36. ssh-copy-id

Public Key를 Remote User에 등록:

```bash
ssh-copy-id \
-i /root/.ssh/id_ed25519_backup_server_b.pub \
backupuser@192.168.111.200
```

일반적으로 Remote User의:

```text
~/.ssh/authorized_keys
```

에 Public Key가 등록된다.

---

# 37. Password 없는 접속

등록 후:

```bash
ssh -i /root/.ssh/id_ed25519_backup_server_b \
-o IdentitiesOnly=yes \
backupuser@192.168.111.200
```

으로 Password 입력 없이 접속 가능한지 확인한다.

자동화 전에 반드시 수동 검증하는 것이 좋다.

---

# 38. cron과 SSH Password

cron Job은 사람이 직접 Terminal에서 입력하지 않는다.

따라서 실행 중:

```text
Password:
```

Prompt가 나오면 자동화가 중단될 수 있다.

그래서 SSH Key Authentication이 중요하다.

---

# 39. BatchMode

이번 Backup Script:

```bash
-o BatchMode=yes
```

를 사용하였다.

의미:

```text
Password 등 Interactive Prompt 사용하지 않음
```

자동화에서 인증이 실패하면 사용자 입력을 기다리는 대신 실패하도록 하는 데 유용하다.

---

# 40. Backup Script

긴 rsync Command를 cron에 직접 넣는 것보다 Script로 분리할 수 있다.

이번 Script:

```text
/usr/local/sbin/rsync-server-b-backup.sh
```

내용:

```bash
#!/bin/bash

/usr/bin/rsync -av --delete \
-e "/usr/bin/ssh -i /root/.ssh/id_ed25519_backup_server_b -o IdentitiesOnly=yes -o BatchMode=yes" \
/root/rsync-lab/source/ \
backupuser@192.168.111.200:/backup/server-a/
```

---

# 41. Script를 사용하는 이유

cron Entry에 모든 Command를 직접 넣는 것보다:

```text
설정 관리 쉬움
수정 쉬움
수동 실행 가능
Troubleshooting 쉬움
재사용 가능
```

이라는 장점이 있다.

---

# 42. Script 실행 권한

```bash
chmod 700 /usr/local/sbin/rsync-server-b-backup.sh
```

의미:

```text
Owner
→ rwx

Group
→ ---

Other
→ ---
```

root 전용 Script로 사용한다.

---

# 43. cron 등록 전 수동 테스트

자동화를 설정하기 전에 Script를 직접 실행해야 한다.

```bash
/usr/local/sbin/rsync-server-b-backup.sh
```

먼저 수동으로 성공해야 cron 문제인지 Script 문제인지 구분하기 쉽다.

---

# 44. cron이란?

`cron`은 Linux에서 Command나 Script를 특정 시간에 자동 실행하기 위한 Scheduler이다.

Daemon:

```text
crond
```

---

# 45. crond 상태 확인

```bash
systemctl is-active crond
```

```bash
systemctl is-enabled crond
```

이번:

```text
active
enabled
```

---

# 46. active와 enabled

```text
active
→ 현재 crond 실행 중

enabled
→ 부팅 시 자동 시작
```

---

# 47. cron 표현식

기본 구조:

```text
분 시 일 월 요일 command
```

형식:

```text
* * * * *
│ │ │ │ │
│ │ │ │ └─ 요일
│ │ │ └─── 월
│ │ └───── 일
│ └─────── 시
└───────── 분
```

---

# 48. 매분 실행

```cron
* * * * *
```

의미:

```text
매 1분마다
```

이번 실습에서는 자동화 검증용으로 잠시 사용하였다.

---

# 49. 매일 02:00

최종 설정:

```cron
0 2 * * *
```

의미:

```text
분
0

시
2

일
*

월
*

요일
*
```

즉:

```text
매일 02:00
```

이다.

---

# 50. /etc/cron.d

이번 설정:

```text
/etc/cron.d/rsync-server-b-backup
```

내용:

```cron
0 2 * * * root /usr/local/sbin/rsync-server-b-backup.sh >> /var/log/rsync-server-b-backup.log 2>&1
```

---

# 51. /etc/cron.d 형식

`/etc/cron.d/` 아래의 File에서는 일반 User의 `crontab -e`와 달리 실행 User Field가 포함된다.

구조:

```text
분 시 일 월 요일 USER COMMAND
```

이번:

```text
0 2 * * * root COMMAND
```

---

# 52. root

cron Entry:

```text
root
```

는 해당 Command를 root User로 실행한다는 의미이다.

---

# 53. cron File Permission

이번:

```text
/etc/cron.d/rsync-server-b-backup
```

Permission:

```text
644
```

즉:

```text
root
→ 읽기 / 쓰기

나머지
→ 읽기
```

---

# 54. cron 환경은 Login Shell과 다를 수 있다

cron은 일반 Terminal에서 실행한 환경과 다를 수 있다.

예:

```text
PATH 다름
환경변수 다름
현재 Directory 다름
Interactive 입력 불가
```

따라서 Script에서는 가능하면 절대 경로를 사용하는 것이 좋다.

---

# 55. Command 절대 경로

이번 Script:

```bash
/usr/bin/rsync
/usr/bin/ssh
```

처럼 절대 경로를 사용하였다.

이는 cron 환경에서 PATH 문제를 줄이는 데 도움이 된다.

---

# 56. cron Log Redirect

이번 설정:

```bash
>> /var/log/rsync-server-b-backup.log 2>&1
```

---

# 57. >>

```bash
>>
```

Standard Output을 File 끝에 Append한다.

기존 내용을 지우지 않고 추가한다.

---

# 58. 2>&1

File Descriptor:

```text
0
→ stdin

1
→ stdout

2
→ stderr
```

`2>&1`:

```text
stderr를 stdout과 같은 곳으로 보냄
```

따라서:

```bash
>> logfile 2>&1
```

은:

```text
정상 출력
+
Error 출력
```

모두 같은 Log File에 저장한다.

---

# 59. Backup Log

이번:

```text
/var/log/rsync-server-b-backup.log
```

확인:

```bash
tail -n 15 /var/log/rsync-server-b-backup.log
```

---

# 60. cron 자체 Log

cron이 Command를 실행했는지는:

```bash
journalctl -u crond
```

로 확인할 수 있다.

이번 실습:

```text
CROND: (root) CMD (...)
CROND: (root) CMDEND (...)
```

를 확인하였다.

---

# 61. CMD

```text
CMD
```

cron이 Command를 실행했다는 기록.

---

# 62. CMDEND

```text
CMDEND
```

cron이 해당 Command 실행을 끝냈다는 기록.

---

# 63. cron 실행과 Backup 성공은 다르다

중요하다.

`journalctl`에서:

```text
CMD
```

가 보인다고 해서 Backup File이 반드시 정상 복사된 것은 아니다.

이는:

```text
cron이 Script를 실행했다
```

는 의미이다.

실제 성공 확인은:

```text
rsync Log
Remote File
File 내용
```

등을 별도로 확인해야 한다.

---

# 64. 자동 Backup 검증

이번 실습에서는 Source에:

```text
cron-auto-test.txt
```

를 생성하고 Script를 직접 실행하지 않았다.

cron 실행 후 Server-B:

```text
automatic cron backup test
```

내용이 확인되었다.

따라서:

```text
cron
→ Script
→ rsync
→ SSH
→ Server-B
```

전체 자동화 Chain이 정상 동작했음을 확인하였다.

---

# 65. Backup 흐름

```text
Server-A Source
        ↓
cron Trigger
        ↓
Backup Script
        ↓
rsync
        ↓
SSH Key 인증
        ↓
Server-B
        ↓
/backup/server-a/
```

---

# 66. Backup 성공 확인 방법

단계별 확인:

```text
1. crond active 확인
2. cron Entry 확인
3. journalctl에서 CMD 확인
4. Backup Script Log 확인
5. Remote Destination File 확인
6. File 내용 확인
```

---

# 67. Backup Troubleshooting 기본 순서

```text
Backup 실패
   ↓
crond 상태 확인
   ↓
cron 설정 확인
   ↓
Script 수동 실행
   ↓
SSH 연결 확인
   ↓
Remote rsync 설치 확인
   ↓
Destination Permission 확인
   ↓
Backup Log 확인
```

---

# 68. Script 수동 실행이 중요한 이유

cron 문제인지 Backup Script 문제인지 구분할 수 있다.

예:

```text
Script 수동 실행 성공
cron 자동 실행 실패
→ cron 설정 / 환경 문제 의심
```

반대로:

```text
Script 수동 실행도 실패
→ rsync / SSH / Permission 문제 의심
```

---

# 69. Remote Connection 확인

```bash
ssh -i /root/.ssh/id_ed25519_backup_server_b \
-o IdentitiesOnly=yes \
backupuser@192.168.111.200
```

확인:

```text
접속 가능 여부
Password Prompt 여부
```

---

# 70. BatchMode Test

자동화 환경처럼 확인하려면:

```bash
ssh -i /root/.ssh/id_ed25519_backup_server_b \
-o IdentitiesOnly=yes \
-o BatchMode=yes \
backupuser@192.168.111.200
```

---

# 71. Destination Permission

Server-B:

```bash
ls -ld /backup/server-a
```

이번:

```text
backupuser:backupuser
750
```

---

# 72. 750 의미

```text
Owner
7 = rwx

Group
5 = r-x

Other
0 = ---
```

Owner인 `backupuser`가 Directory 안에 File을 생성할 수 있다.

---

# 73. Write Test

Backup 전에 Remote User가 실제로 File 생성 가능한지 확인할 수 있다.

예:

```bash
touch /backup/server-a/test
```

성공:

```text
Permission 정상
```

실패:

```text
Permission / SELinux / File System 상태 확인
```

---

# 74. SELinux 고려

Rocky Linux에서는 SELinux가 활성화되어 있을 수 있다.

Permission이 맞는데도 특정 Service나 Context에서 접근 실패가 발생하면:

```text
File Permission
Owner
SELinux Context
Audit Log
```

등을 함께 확인해야 한다.

이번 실습에서는 SSH User의 일반 File Access로 정상 동작하였다.

---

# 75. Network 확인

Remote Backup 전 기본 확인:

```bash
ping -c 2 192.168.111.200
```

하지만 `ping` 성공만으로 SSH가 반드시 정상이라는 뜻은 아니다.

추가로:

```text
SSH Service
Firewall
Port 22
Authentication
```

을 확인해야 한다.

---

# 76. SSH와 rsync 관계

구조:

```text
rsync
→ File Synchronization Logic

SSH
→ Secure Remote Transport / Authentication
```

즉 역할이 다르다.

---

# 77. Backup User 권한 최소화

운영에서는 Backup User에게 필요한 Directory만 쓸 수 있도록 제한하는 것이 좋다.

예:

```text
/backup/server-a
→ Write 가능

다른 System Directory
→ 불필요한 권한 부여하지 않음
```

Principle of Least Privilege에 해당한다.

---

# 78. Backup Key 보안

Private Key:

```text
/root/.ssh/id_ed25519_backup_server_b
```

는 중요한 Credential이다.

주의:

```text
외부 공유 금지
Permission 제한
Backup Script에 Key 내용 직접 삽입 금지
Git Repository에 Commit 금지
```

---

# 79. SSH Private Key를 GitHub에 올리면 안 되는 이유

Private Key가 유출되면 해당 Key로 인증 가능한 Server에 Unauthorized Access가 발생할 수 있다.

Git에는:

```text
Private Key
Password
Credential File
Secret Token
```

을 올리지 않는다.

---

# 80. Backup Log 관리

현재:

```text
/var/log/rsync-server-b-backup.log
```

에 계속 Append한다.

장기간 운영하면 Log가 계속 커질 수 있다.

따라서 운영 환경에서는:

```text
logrotate
```

같은 Log Rotation을 고려한다.

이는 이후 Log Management 실습에서 다룰 수 있다.

---

# 81. cron 자체는 File Version을 보존하지 않는다

cron은 단순히:

```text
정해진 시간에 Command 실행
```

하는 Scheduler이다.

Backup 정책은 실제 Script와 rsync Option이 결정한다.

---

# 82. Backup Schedule 결정

무조건 자주 실행한다고 좋은 것은 아니다.

고려:

```text
Data 변경 빈도
Network Traffic
Disk I/O
Backup 용량
서비스 중요도
복구 요구사항
```

이번 실습에서는:

```text
매일 02:00
```

으로 설정하였다.

---

# 83. 왜 새벽 시간인가?

일반적으로 업무 Traffic이 적은 시간에 Backup을 실행하면:

```text
Application 부하 감소
Network 경쟁 감소
Disk I/O 경쟁 감소
```

에 도움이 될 수 있다.

실제 운영 Schedule은 서비스 특성에 맞게 결정해야 한다.

---

# 84. RPO

Backup을 이해할 때 참고할 개념.

RPO:

```text
Recovery Point Objective
```

얼마나 최근 시점까지 복구해야 하는지를 나타내는 목표이다.

예:

```text
하루 1회 Backup
```

이면 상황에 따라 최대 하루 정도의 변경 Data가 Backup에 없을 수 있다.

---

# 85. RTO

RTO:

```text
Recovery Time Objective
```

장애 발생 후 서비스를 얼마나 빨리 복구해야 하는지에 대한 목표이다.

Backup이 있어도 복구에 너무 오래 걸린다면 RTO 요구사항을 만족하지 못할 수 있다.

---

# 86. Backup과 Restore

Backup의 진짜 목적은:

```text
복구
```

이다.

따라서 Backup File이 존재하는 것만 확인하는 것이 아니라 실제 Restore Test도 중요하다.

---

# 87. Restore Test 필요성

예:

```text
Backup Job 성공
        ↓
File 존재
        ↓
하지만 File 손상
또는 Permission 문제
또는 필요한 File 누락
```

일 수 있다.

따라서 중요한 환경에서는 주기적으로 Restore를 검증해야 한다.

---

# 88. 3-2-1 Backup 개념

일반적인 Backup 전략으로 자주 언급되는 개념:

```text
3
→ Data Copy 3개

2
→ 서로 다른 Storage 종류 2개

1
→ 1개는 별도 장소 / Offsite
```

이번 실습은:

```text
Server-A
→ Server-B
```

Remote Copy를 만드는 기본 단계에 해당한다.

---

# 89. rsync만으로 모든 장애를 막을 수는 없다

예:

```text
Source Data 손상
        ↓
rsync
        ↓
손상된 Data도 Destination에 반영
```

또는:

```text
Source에서 File 삭제
        ↓
--delete
        ↓
Destination에서도 삭제
```

가능하다.

따라서 Version / Snapshot / Separate Backup 정책이 중요하다.

---

# 90. rsync 전송 보안

SSH를 사용하면:

```text
Authentication
Encryption
Integrity
```

가 적용된 Channel을 통해 Remote rsync를 수행할 수 있다.

---

# 91. rsync Output 해석

예:

```text
sending incremental file list
file1.txt

sent 300 bytes
received 40 bytes
total size is 100
speedup is 0.29
```

중요한 것은 File 이름이 나타나는지 확인하는 것이다.

변경 File 이름이 나오면 해당 File이 동기화 대상이 된 것이다.

---

# 92. sent / received

```text
sent
→ Local에서 전송된 Protocol / Data 양

received
→ Remote에서 받은 Protocol 관련 Data 양
```

File 크기와 정확히 동일하게 생각할 필요는 없다.

---

# 93. total size

```text
total size
```

동기화 대상 File Set의 전체 Logical Size와 관련된다.

실제 Network 전송량과 항상 동일한 것은 아니다.

---

# 94. speedup

```text
speedup
```

rsync가 전체 Data Size와 실제 Transfer 관련 Data를 비교해 보여주는 통계이다.

작은 Lab File에서는 숫자가 크게 의미 없을 수 있다.

---

# 95. Directory 확인 시 주의

예:

```bash
ls /backup/server-a/subdir/
```

는:

```text
subdir 내부
```

만 확인한다.

전체 Backup Directory:

```bash
ls /backup/server-a/
```

하위 전체 File:

```bash
find /backup/server-a -type f
```

---

# 96. find 활용

Remote Backup 전체 File 확인:

```bash
find /backup/server-a -type f
```

Subdirectory까지 전체 File을 볼 수 있다.

---

# 97. rsync 테스트용 Directory 사용

`--delete` 같은 위험한 Option은 운영 Directory보다 먼저 별도 Test Directory에서 확인하는 것이 좋다.

이번 실습:

```text
/root/rsync-lab/source/
/root/rsync-lab/local-backup/
```

를 사용하였다.

---

# 98. 운영 Backup 전에 확인할 사항

```text
Source 경로
Destination 경로
마지막 / 여부
--delete 여부
SSH User
SSH Key
Remote Permission
Disk 여유 공간
Network 연결
rsync 설치 여부
cron Schedule
Log 위치
```

---

# 99. Disk 공간 확인

Backup Destination 용량이 부족하면 Backup이 실패할 수 있다.

확인:

```bash
df -h /backup
```

필요에 따라:

```bash
du -sh /backup/server-a
```

로 사용량을 확인할 수 있다.

---

# 100. Source 변경과 Backup

Backup은 Source 변경 시점과 cron 실행 시점에 따라 결과가 달라진다.

예:

```text
01:00 File 생성
02:00 Backup
→ 포함

02:01 File 생성
다음 Backup 02:00 다음날
→ 다음 실행까지 Backup되지 않음
```

---

# 101. cron Schedule 해석 주의

```cron
0 2 * * *
```

는:

```text
2시간마다
```

가 아니다.

정확한 의미:

```text
매일 02시 00분
```

이다.

---

# 102. */2와 2 차이

예:

```cron
0 */2 * * *
```

은:

```text
2시간 간격
```

의 의미로 사용할 수 있다.

반면:

```cron
0 2 * * *
```

은:

```text
매일 02시
```

이다.

---

# 103. crontab과 /etc/cron.d 차이

User Crontab:

```bash
crontab -e
```

형식:

```text
분 시 일 월 요일 COMMAND
```

`/etc/cron.d`:

```text
분 시 일 월 요일 USER COMMAND
```

즉 `/etc/cron.d`에는 User Field가 추가된다.

---

# 104. Cron Job 확인

`/etc/cron.d` 방식:

```bash
cat /etc/cron.d/rsync-server-b-backup
```

User Crontab을 확인할 때는:

```bash
crontab -l
```

을 사용할 수 있다.

---

# 105. cron Troubleshooting 시 시간 확인

```bash
date
```

로 현재 System 시간을 확인한다.

Server Timezone이나 시간이 잘못되어 있으면 예상한 시각과 다른 시각에 cron이 실행될 수 있다.

이전에 구성한 Chrony/NTP가 중요한 이유 중 하나이다.

---

# 106. 시간 동기화와 Backup

cron은 System Clock을 기준으로 동작한다.

따라서:

```text
System Time 오류
        ↓
Backup 실행 시간 오류
```

가 발생할 수 있다.

NTP / Chrony 설정은 Scheduler 운영에도 중요하다.

---

# 107. 실무 Troubleshooting 예

증상:

```text
오늘 Backup File이 없음
```

확인 순서:

```text
1. date
2. systemctl status crond
3. cron File 확인
4. journalctl -u crond
5. Backup Log 확인
6. Script 직접 실행
7. SSH Test
8. Destination Permission 확인
9. Disk 공간 확인
```

---

# 108. rsync command not found

Remote에서:

```text
bash: rsync: command not found
```

라면 Remote Server에 설치:

```bash
dnf install rsync
```

---

# 109. Permission denied

가능한 원인:

```text
SSH Login Permission
SSH Key 문제
Destination Directory Permission
Owner / Group 문제
SELinux
```

---

# 110. Host Key 관련 문제

첫 SSH 연결에서는 Host Key 확인이 나타날 수 있다.

예:

```text
Are you sure you want to continue connecting?
```

자동화 전에 수동 SSH 연결을 한 번 수행하여 Host Key를 확인해 두는 것이 좋다.

---

# 111. Known Hosts

SSH로 연결한 Host 정보는 일반적으로:

```text
~/.ssh/known_hosts
```

에 저장된다.

Host Key가 변경되면 SSH가 경고를 표시할 수 있다.

이는 MITM 등의 보안 문제를 탐지하기 위한 기능이기도 하다.

---

# 112. Password 없는 Key의 주의점

이번 Lab에서는 cron 자동화를 위해 Passphrase 없는 Key를 사용하였다.

장점:

```text
자동화 쉬움
```

위험:

```text
Private Key 유출 시 즉시 악용 가능
```

운영 환경에서는:

```text
Key 권한 제한
전용 User
접근 제한
SSH Configuration 제한
Secret 관리
```

등을 함께 고려해야 한다.

---

# 113. Backup Script Log 증가

현재:

```bash
>> /var/log/rsync-server-b-backup.log
```

로 계속 Append한다.

시간이 지나면:

```text
Log File 크기 증가
```

가 발생한다.

따라서 추후:

```text
logrotate
```

설정이 필요할 수 있다.

---

# 114. Backup Monitoring

자동 Backup은 설정했다고 끝이 아니다.

확인해야 할 것:

```text
마지막 성공 시간
실패 여부
Backup 용량
Destination Disk 사용량
Network 오류
SSH 인증 오류
```

운영 환경에서는 실패 Alert도 중요하다.

---

# 115. Backup File Ownership

이번 Server-B Backup File:

```text
backupuser:backupuser
```

로 생성되었다.

이는 Remote rsync가:

```text
backupuser
```

권한으로 실행되기 때문이다.

---

# 116. -a와 Owner 보존

`-a`는 Owner / Group 보존 기능을 포함하지만 Remote User 권한에 따라 원본의 root Ownership을 그대로 설정하지 못할 수 있다.

이번처럼 일반 User인:

```text
backupuser
```

로 Remote Destination에 저장하면 실제 Destination File Ownership은 Remote User 권한의 영향을 받는다.

---

# 117. root Remote Backup 주의

root로 Remote rsync를 실행하면 Ownership 등을 더 폭넓게 보존할 수 있지만 보안 위험도 커진다.

따라서 필요하지 않다면 전용 제한 User를 사용하는 것이 좋다.

---

# 118. 현재 구성 목적

이번 Lab의 목적은:

```text
완전한 Enterprise Backup System 구성
```

이 아니라:

```text
rsync 동기화 원리
SSH 기반 Remote Backup
Key Authentication
cron 자동화
Log 확인
Troubleshooting
```

을 이해하는 것이다.

---

# 119. 전체 구성

```text
                 Server-A
            192.168.111.100
                   |
                   |
         /root/rsync-lab/source/
                   |
                   |
            cron Scheduler
              매일 02:00
                   |
                   ↓
 /usr/local/sbin/rsync-server-b-backup.sh
                   |
                   ↓
                rsync
                   |
                   ↓
                 SSH
                   |
          ED25519 Key 인증
                   |
                   ↓
                 Server-B
            192.168.111.200
                   |
              backupuser
                   |
                   ↓
          /backup/server-a/
```

---

# 120. 실습 Troubleshooting 흐름

이번 실습에서 실제 발생한 문제:

```text
Remote rsync 실패
        ↓
bash: rsync: command not found
        ↓
Server-B rsync 미설치 확인
        ↓
Server-B rsync 설치
        ↓
Remote rsync 정상화
```

이런 식으로 Error Message를 먼저 읽고 원인을 좁히는 것이 중요하다.

---

# 121. 핵심 Command 정리

Local rsync:

```bash
rsync -av source/ backup/
```

Delete 동기화:

```bash
rsync -av --delete source/ backup/
```

Dry Run:

```bash
rsync -av --delete --dry-run source/ backup/
```

Remote rsync:

```bash
rsync -av \
-e "ssh -i PRIVATE_KEY -o IdentitiesOnly=yes" \
source/ \
user@remote:/backup/
```

---

# 122. SSH Key 생성

```bash
ssh-keygen -t ed25519 -f KEY_PATH
```

Public Key 등록:

```bash
ssh-copy-id -i KEY.pub user@remote
```

---

# 123. cron 확인

```bash
systemctl status crond
```

```bash
systemctl is-active crond
```

```bash
systemctl is-enabled crond
```

---

# 124. cron Log 확인

```bash
journalctl -u crond
```

최근 기록:

```bash
journalctl -u crond --since "5 minutes ago"
```

---

# 125. Backup Log 확인

```bash
tail /var/log/rsync-server-b-backup.log
```

---

# 126. 최종 핵심

```text
rsync
→ File / Directory Synchronization

-a
→ Archive Mode

-v
→ 상세 출력

--delete
→ Source에 없는 File을 Destination에서도 삭제

--dry-run
→ 실제 변경 없이 작업 예상 결과 확인

-e ssh
→ SSH를 Remote Shell로 사용

SSH Key
→ Password 없는 자동 인증

BatchMode=yes
→ Interactive 인증 Prompt 방지

cron
→ 정해진 시간에 Script 자동 실행
```

---

# 127. Backup 실무 관점 핵심

Backup을 구성할 때 단순히:

```text
File이 복사되었다
```

만 확인하면 안 된다.

다음 전체 흐름을 확인해야 한다.

```text
Source 존재
        ↓
Backup Script 정상
        ↓
SSH 인증 정상
        ↓
Remote Permission 정상
        ↓
rsync 정상
        ↓
cron 정상 실행
        ↓
Backup Log 정상
        ↓
Destination File 정상
        ↓
Restore 가능 여부 확인
```

---

# 128. 최종 정리

이번 구성은:

```text
Server-A
/root/rsync-lab/source/
```

를:

```text
Server-B
/backup/server-a/
```

로 자동 동기화한다.

전송 방식:

```text
rsync over SSH
```

인증 방식:

```text
ED25519 SSH Key
```

실행 방식:

```text
cron
매일 02:00
```

Backup Script:

```text
/usr/local/sbin/rsync-server-b-backup.sh
```

cron 설정:

```text
/etc/cron.d/rsync-server-b-backup
```

Log:

```text
/var/log/rsync-server-b-backup.log
```

최종 구조:

```text
Source
Server-A
        ↓
cron
        ↓
Backup Script
        ↓
rsync
        ↓
SSH Key Authentication
        ↓
Server-B
Destination
```

이번 실습을 통해 단순 File 복사를 넘어:

```text
증분 동기화
Remote Backup
SSH Authentication
Backup User 권한
--delete
--dry-run
cron Scheduling
Backup Logging
Remote rsync Troubleshooting
```

까지 실제 Server Backup 자동화의 기본 흐름을 구성하였다.
