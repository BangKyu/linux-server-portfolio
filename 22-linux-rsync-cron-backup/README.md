# rsync + cron Remote Backup

Rocky Linux Server 환경에서 `rsync`와 SSH Key 인증을 이용하여 Server-A의 데이터를 Server-B로 원격 Backup하고, `cron`을 이용하여 매일 자동 실행되도록 구성하였다.

---

# 1. 실습 환경

| 구분 | Server-A | Server-B |
|---|---|---|
| 역할 | Backup Source | Backup Destination |
| IP | 192.168.111.100 | 192.168.111.200 |
| Backup User | root | backupuser |
| Source | `/root/rsync-lab/source/` | - |
| Destination | - | `/backup/server-a/` |

Backup 구조:

```text
Server-A
192.168.111.100

/root/rsync-lab/source/
        |
        | rsync over SSH
        ↓
Server-B
192.168.111.200

/backup/server-a/
```

---

# 2. rsync / cron 환경 확인

Server-A:

```bash
rpm -q rsync
rpm -q cronie
systemctl is-active crond
systemctl is-enabled crond
```

실제 결과:

```text
rsync-3.2.5-7.el9_8.2.x86_64
cronie-1.5.7-16.el9.x86_64
active
enabled
```

Server-B 연결 확인:

```bash
ping -c 2 192.168.111.200
```

실제:

```text
2 packets transmitted, 2 received, 0% packet loss
```

SSH Client:

```bash
ssh -V
```

실제:

```text
OpenSSH_9.9p1, OpenSSL 3.5.5 27 Jan 2026
```

---

# 3. Local rsync 실습

테스트 Directory 생성:

```bash
mkdir -p /root/rsync-lab/source/subdir
mkdir -p /root/rsync-lab/local-backup
```

테스트 File 생성:

```bash
printf 'file1 original\n' > /root/rsync-lab/source/file1.txt
printf 'file2 original\n' > /root/rsync-lab/source/file2.txt
printf 'sub file original\n' > /root/rsync-lab/source/subdir/subfile.txt
```

Local Backup:

```bash
rsync -av /root/rsync-lab/source/ /root/rsync-lab/local-backup/
```

실제 결과:

```text
sending incremental file list
file1.txt
file2.txt
subdir/
subdir/subfile.txt
```

Backup File 확인:

```bash
find /root/rsync-lab/local-backup -type f -printf '%p\n'
```

실제:

```text
/root/rsync-lab/local-backup/subdir/subfile.txt
/root/rsync-lab/local-backup/file1.txt
/root/rsync-lab/local-backup/file2.txt
```

---

# 4. rsync 증분 동작 확인

변경 없이 다시 실행:

```bash
rsync -av /root/rsync-lab/source/ /root/rsync-lab/local-backup/
```

실제:

```text
sending incremental file list

sent 151 bytes  received 13 bytes
```

전송 File 이름이 표시되지 않았다.

즉 원본과 대상이 동일하면 변경되지 않은 File은 다시 전송하지 않는다.

---

# 5. File 수정 / 추가

원본 수정:

```bash
printf 'file1 modified\n' > /root/rsync-lab/source/file1.txt
printf 'file3 new\n' > /root/rsync-lab/source/file3.txt
```

다시 rsync 실행 후 Backup 결과:

```text
file1 modified
file3 new
```

변경된 File과 새 File이 Backup에 반영되었다.

변경이 없는 상태에서 다시 실행하면 File 전송이 발생하지 않았다.

---

# 6. rsync Source 경로의 `/`

다음 Command:

```bash
rsync -av /root/rsync-lab/source/ /root/rsync-lab/local-backup/
```

Source의 마지막 `/`는 `source` Directory 자체가 아니라 내부 내용을 의미한다.

```text
source/
├── file1.txt
├── file3.txt
└── subdir/
    └── subfile.txt
```

결과:

```text
local-backup/
├── file1.txt
├── file3.txt
└── subdir/
    └── subfile.txt
```

즉:

```text
source/
→ Directory 내부 내용 동기화
```

---

# 7. --delete 동작 확인

원본에서:

```bash
rm -f /root/rsync-lab/source/file2.txt
```

삭제 후 일반 rsync를 실행하면 대상의 `file2.txt`는 그대로 남아 있었다.

기본 rsync는:

```text
추가 File
→ 복사

수정 File
→ 갱신

원본에서 삭제된 File
→ 대상에서 자동 삭제하지 않음
```

---

# 8. --dry-run

실제 삭제 전 확인:

```bash
rsync -av --delete --dry-run \
/root/rsync-lab/source/ \
/root/rsync-lab/local-backup/
```

실제:

```text
sending incremental file list
deleting file2.txt

(DRY RUN)
```

실제 File을 변경하지 않고 삭제 예정 항목을 확인하였다.

---

# 9. --delete 실제 적용

```bash
rsync -av --delete \
/root/rsync-lab/source/ \
/root/rsync-lab/local-backup/
```

실제:

```text
deleting file2.txt
```

결과 확인:

```text
/root/rsync-lab/local-backup/subdir/subfile.txt
/root/rsync-lab/local-backup/file1.txt
/root/rsync-lab/local-backup/file3.txt
```

원본에 없는 `file2.txt`가 Backup에서도 삭제되었다.

---

# 10. Server-B Backup User 생성

Server-B에 원격 Backup 전용 User:

```text
backupuser
```

를 사용하였다.

실제:

```text
uid=1001(backupuser)
gid=1001(backupuser)
groups=1001(backupuser)
```

Backup Directory:

```bash
mkdir -p /backup/server-a
chown backupuser:backupuser /backup/server-a
chmod 750 /backup/server-a
```

실제:

```text
drwxr-x---. 2 backupuser backupuser /backup/server-a
```

---

# 11. SSH Key 인증 구성

Server-A에서 Backup 전용 SSH Key 생성:

```bash
ssh-keygen -t ed25519 \
-f /root/.ssh/id_ed25519_backup_server_b \
-N ''
```

Public Key 등록:

```bash
ssh-copy-id \
-i /root/.ssh/id_ed25519_backup_server_b.pub \
backupuser@192.168.111.200
```

Password 없이 접속 확인:

```bash
ssh -i /root/.ssh/id_ed25519_backup_server_b \
-o IdentitiesOnly=yes \
backupuser@192.168.111.200
```

실제:

```text
backupuser
Server-B
/home/backupuser
```

---

# 12. Remote Directory 쓰기 권한 확인

Server-A:

```bash
ssh -i /root/.ssh/id_ed25519_backup_server_b \
-o IdentitiesOnly=yes \
backupuser@192.168.111.200 \
'touch /backup/server-a/ssh-write-test && \
ls -l /backup/server-a/ssh-write-test && \
rm -f /backup/server-a/ssh-write-test'
```

실제:

```text
-rw-r--r--. 1 backupuser backupuser 0 ... /backup/server-a/ssh-write-test
```

`backupuser`가 `/backup/server-a`에 정상적으로 File을 생성할 수 있음을 확인하였다.

---

# 13. Remote rsync 오류 및 해결

최초 원격 rsync 실행 시:

```text
bash: line 1: rsync: command not found
rsync: connection unexpectedly closed
rsync error: error in rsync protocol data stream (code 12)
```

오류가 발생하였다.

원인:

```text
Server-A
→ rsync 설치됨

Server-B
→ rsync 미설치
```

`rsync over SSH`는 SSH 연결만 사용하는 것이 아니라 Remote Host에서도 `rsync` Command를 실행한다.

따라서 양쪽 Server에 rsync가 필요하다.

Server-B에 `rsync` 설치 후 정상 동작하였다.

---

# 14. Server-A → Server-B Remote rsync

Server-A:

```bash
rsync -av \
-e "ssh -i /root/.ssh/id_ed25519_backup_server_b -o IdentitiesOnly=yes" \
/root/rsync-lab/source/ \
backupuser@192.168.111.200:/backup/server-a/
```

실제:

```text
sending incremental file list
./
file1.txt
file3.txt
subdir/
subdir/subfile.txt
```

Server-B 확인:

```bash
ssh -i /root/.ssh/id_ed25519_backup_server_b \
-o IdentitiesOnly=yes \
backupuser@192.168.111.200 \
'find /backup/server-a -type f -printf "%p\n"'
```

실제:

```text
/backup/server-a/subdir/subfile.txt
/backup/server-a/file1.txt
/backup/server-a/file3.txt
```

---

# 15. Remote rsync Command 구조

```bash
rsync -av \
-e "ssh -i /root/.ssh/id_ed25519_backup_server_b -o IdentitiesOnly=yes" \
/root/rsync-lab/source/ \
backupuser@192.168.111.200:/backup/server-a/
```

구조:

```text
rsync [Option] [Source] [Destination]
```

Source:

```text
Server-A
/root/rsync-lab/source/
```

Destination:

```text
Server-B
/backup/server-a/
```

즉:

```text
Server-A의
/root/rsync-lab/source/ 내부 내용을

SSH를 통해

Server-B의
/backup/server-a/

로 동기화
```

한다.

---

# 16. Remote 증분 Backup

Server-A에서:

```bash
printf 'file1 remote modified\n' > /root/rsync-lab/source/file1.txt
printf 'file4 remote new\n' > /root/rsync-lab/source/file4.txt
```

Remote rsync 후 Server-B 확인:

```bash
cat /backup/server-a/file1.txt
cat /backup/server-a/file4.txt
```

실제:

```text
file1 remote modified
file4 remote new
```

변경이 없는 상태에서 다시 rsync:

```text
sending incremental file list

sent 188 bytes  received 13 bytes
```

File 이름이 출력되지 않아 변경되지 않은 File은 다시 전송되지 않는 것을 확인하였다.

---

# 17. Backup Script 생성

Server-A:

```bash
vi /usr/local/sbin/rsync-server-b-backup.sh
```

내용:

```bash
#!/bin/bash

/usr/bin/rsync -av --delete \
-e "/usr/bin/ssh -i /root/.ssh/id_ed25519_backup_server_b -o IdentitiesOnly=yes -o BatchMode=yes" \
/root/rsync-lab/source/ \
backupuser@192.168.111.200:/backup/server-a/
```

실행 권한:

```bash
chmod 700 /usr/local/sbin/rsync-server-b-backup.sh
```

---

# 18. BatchMode

Backup Script의 SSH Option:

```text
-o BatchMode=yes
```

는 SSH가 Password 등 사용자 입력을 요구하지 않도록 하는 Option이다.

자동화 작업에서는 입력 Prompt가 발생하면 cron이 정상적으로 Backup을 완료할 수 없으므로 SSH Key 인증과 함께 사용하였다.

---

# 19. Backup Script 수동 검증

Server-A:

```bash
printf 'cron backup test\n' \
> /root/rsync-lab/source/cron-test.txt
```

Script 실행:

```bash
/usr/local/sbin/rsync-server-b-backup.sh
```

Server-B 확인:

```bash
cat /backup/server-a/cron-test.txt
```

실제:

```text
cron backup test
```

cron에 등록하기 전에 Script 자체가 정상적으로 동작하는 것을 먼저 확인하였다.

---

# 20. cron 테스트 구성

1분마다 자동 실행하도록 임시 설정하였다.

```bash
vi /etc/cron.d/rsync-server-b-backup
```

테스트 설정:

```cron
* * * * * root /usr/local/sbin/rsync-server-b-backup.sh >> /var/log/rsync-server-b-backup.log 2>&1
```

권한:

```bash
chmod 644 /etc/cron.d/rsync-server-b-backup
```

---

# 21. cron 자동 Backup 검증

cron만으로 전송되는지 확인하기 위해 새로운 File 생성:

```bash
printf 'automatic cron backup test\n' \
> /root/rsync-lab/source/cron-auto-test.txt
```

직접 Backup Script를 실행하지 않고 cron 실행을 기다렸다.

Server-B 확인:

```bash
cat /backup/server-a/cron-auto-test.txt
```

실제:

```text
automatic cron backup test
```

cron을 통해 자동으로 File이 Backup된 것을 확인하였다.

---

# 22. crond 실행 확인

```bash
journalctl -u crond --since "5 minutes ago" --no-pager
```

실제:

```text
10:54:01 Server-A CROND: (root) CMD (/usr/local/sbin/rsync-server-b-backup.sh ...)
10:55:01 Server-A CROND: (root) CMD (/usr/local/sbin/rsync-server-b-backup.sh ...)
10:55:01 Server-A CROND: (root) CMDEND (...)
10:56:01 Server-A CROND: (root) CMD (/usr/local/sbin/rsync-server-b-backup.sh ...)
10:56:02 Server-A CROND: (root) CMDEND (...)
```

`crond`가 지정된 Script를 실제 실행하고 있음을 확인하였다.

---

# 23. Backup Log 확인

```bash
tail -n 15 /var/log/rsync-server-b-backup.log
```

실제:

```text
sending incremental file list
./
cron-auto-test.txt

sent 299 bytes  received 39 bytes
total size is 101

sending incremental file list

sent 222 bytes  received 13 bytes
total size is 101
```

첫 실행:

```text
cron-auto-test.txt
```

이 실제 전송되었다.

이후 실행:

```text
File 이름 없음
```

으로 변경 사항이 없는 경우 실제 File 전송이 발생하지 않았다.

---

# 24. 최종 cron 설정

자동 Backup 검증 후 테스트용 매분 실행을 제거하고 운영형 Schedule로 변경하였다.

```cron
0 2 * * * root /usr/local/sbin/rsync-server-b-backup.sh >> /var/log/rsync-server-b-backup.log 2>&1
```

확인:

```bash
cat /etc/cron.d/rsync-server-b-backup
```

실제:

```text
0 2 * * * root /usr/local/sbin/rsync-server-b-backup.sh >> /var/log/rsync-server-b-backup.log 2>&1
```

의미:

```text
0
→ 00분

2
→ 02시

*
→ 매일

*
→ 매월

*
→ 모든 요일
```

즉:

```text
매일 02:00
```

Server-A → Server-B Backup이 자동 실행된다.

---

# 25. crond 최종 상태

```bash
systemctl is-active crond
systemctl is-enabled crond
```

실제:

```text
active
enabled
```

Server 재부팅 후에도 `crond`가 자동 시작하도록 구성되어 있다.

---

# 26. rsync -a

이번 실습에서 사용한:

```bash
rsync -av
```

중:

```text
-a
→ Archive Mode

-v
→ Verbose
```

이다.

Archive Mode는 Directory를 재귀적으로 복사하면서 주요 File 속성을 보존하는 데 사용된다.

---

# 27. --delete 주의사항

Backup Script:

```bash
/usr/bin/rsync -av --delete ...
```

에는 `--delete`가 포함되어 있다.

따라서:

```text
Server-A Source File 삭제
        ↓
다음 Backup 실행
        ↓
Server-B에서도 해당 File 삭제
```

된다.

즉 현재 구성은 과거 File을 계속 보관하는 Version Backup보다는:

```text
Source 상태를 Destination에 동일하게 유지하는
Mirror / Synchronization Backup
```

성격이 강하다.

운영 환경에서는 `--delete` 적용 전 경로와 Backup 정책을 반드시 확인해야 한다.

필요할 경우 먼저:

```bash
rsync --delete --dry-run ...
```

으로 삭제 예정 File을 확인한다.

---

# 28. rsync와 cp 차이

일반적인 `cp`는 File을 복사한다.

`rsync`는 Source와 Destination 상태를 비교하여 변경된 Data 중심으로 동기화할 수 있다.

```text
cp
→ File 복사

rsync
→ Source / Destination 비교
→ 변경된 File 중심 전송
→ Remote 전송 가능
→ --delete를 이용한 Mirror 가능
```

---

# 29. 자동 Backup 구조

```text
              Server-A
          192.168.111.100
                 |
        /root/rsync-lab/source/
                 |
                 |
          cron 매일 02:00
                 |
                 ↓
 /usr/local/sbin/rsync-server-b-backup.sh
                 |
                 ↓
             rsync
                 |
                 ↓
         SSH Key Authentication
                 |
                 ↓
              Server-B
          192.168.111.200
                 |
                 ↓
        /backup/server-a/
```

---

# 30. Troubleshooting

## Remote rsync에서 command not found

증상:

```text
bash: rsync: command not found
rsync error: error in rsync protocol data stream
```

확인:

```bash
rpm -q rsync
which rsync
```

Local뿐만 아니라 Remote Server에도 `rsync`가 설치되어 있어야 한다.

---

## SSH 자동화 실패

확인:

```bash
ssh -i /root/.ssh/id_ed25519_backup_server_b \
-o IdentitiesOnly=yes \
-o BatchMode=yes \
backupuser@192.168.111.200
```

Password Prompt 없이 접속 가능한지 확인한다.

---

## Destination 쓰기 실패

Server-B:

```bash
ls -ld /backup/server-a
```

확인:

```text
Owner
Group
Permission
```

이번 구성:

```text
backupuser:backupuser
750
```

---

## cron은 실행되는데 Backup이 안 되는 경우

확인:

```bash
systemctl status crond
```

```bash
journalctl -u crond
```

```bash
cat /var/log/rsync-server-b-backup.log
```

그리고 Script는 cron 등록 전에 반드시 직접 실행하여 검증한다.

```bash
/usr/local/sbin/rsync-server-b-backup.sh
```

---

# 31. 주요 File

Backup Script:

```text
/usr/local/sbin/rsync-server-b-backup.sh
```

cron 설정:

```text
/etc/cron.d/rsync-server-b-backup
```

Backup Log:

```text
/var/log/rsync-server-b-backup.log
```

SSH Private Key:

```text
/root/.ssh/id_ed25519_backup_server_b
```

Server-B Backup Destination:

```text
/backup/server-a/
```

---

# 실습 결과

```text
rsync / cron 환경
✓ Server-A rsync 확인
✓ cronie 확인
✓ crond active / enabled 확인
✓ Server-A ↔ Server-B Network 확인

Local rsync
✓ 최초 전체 복사
✓ 변경 없을 때 재전송 없음 확인
✓ File 수정 반영
✓ File 추가 반영
✓ Source 마지막 / 의미 확인

Delete
✓ 기본 rsync 삭제 미반영 확인
✓ --delete --dry-run 확인
✓ --delete 실제 삭제 확인

Remote Backup
✓ Server-B backupuser 구성
✓ /backup/server-a 구성
✓ SSH Key 인증 구성
✓ Password 없는 SSH 접속 확인
✓ Remote 쓰기 권한 확인
✓ Remote rsync 수행
✓ 변경된 File만 전송 확인

Troubleshooting
✓ Remote rsync 미설치 오류 확인
✓ Remote rsync 설치 후 정상화

Automation
✓ Backup Script 작성
✓ Script 수동 실행 검증
✓ cron 1분 테스트
✓ 자동 File Backup 확인
✓ crond Log 확인
✓ rsync Backup Log 확인
✓ 최종 Schedule 매일 02:00 설정
```

---

# 최종 결과

Server-A의:

```text
/root/rsync-lab/source/
```

내용을 Server-B의:

```text
/backup/server-a/
```

로 `rsync over SSH`를 이용하여 Backup하도록 구성하였다.

SSH Key 인증을 이용하여 Password 입력 없이 자동 접속할 수 있도록 하였으며, Backup Script:

```text
/usr/local/sbin/rsync-server-b-backup.sh
```

를 생성하였다.

`cron` 테스트에서는 1분마다 Backup Script를 자동 실행하여:

```text
automatic cron backup test
```

File이 실제 Server-B로 전송되는 것을 확인하였다.

최종적으로:

```cron
0 2 * * * root /usr/local/sbin/rsync-server-b-backup.sh >> /var/log/rsync-server-b-backup.log 2>&1
```

를 설정하여:

```text
매일 02:00
Server-A → Server-B
자동 Remote Backup
```

구성을 완료하였다.

또한 `rsync`의 증분 동작, `--delete`, `--dry-run`, SSH Key 인증, cron 실행 Log 및 원격 `rsync`가 양쪽 Server에 필요하다는 점까지 실제 오류와 검증을 통해 확인하였다.

---

## 관련 이론

`rsync`, Archive Mode, Source 경로의 `/`, 증분 동기화, `--delete`, `--dry-run`, SSH Key 인증, `BatchMode`, cron 표현식, Mirror Backup과 Version Backup의 차이 및 Backup 운영 시 주의사항은 `rsync-cron-backup-notes.md`에서 정리한다.
