# 5. Linux 보안 점검 실습

[← 목차로](README.md)

> 💡 **이 문서 전체가 보충 내용입니다.**
> [1. Linux](01-linux.md)에서 배운 명령어를 **보안 점검과 침해 사고 확인**에 어떻게 쓰는지 상황별로 정리했습니다.
> 실습 환경은 Rocky Linux 9이며, 대부분의 명령은 root 권한(`su -` 또는 `sudo`)이 필요합니다.

- [5-0. 실습 전 준비](#5-0-실습-전-준비)
- [5-1. 서버 현황 파악하기](#5-1-서버-현황-파악하기)
- [5-2. 로그에서 공격 흔적 찾기](#5-2-로그에서-공격-흔적-찾기)
- [5-3. 텍스트 처리 도구](#5-3-텍스트-처리-도구)
- [5-4. 계정 점검](#5-4-계정-점검)
- [5-5. SSH 보안 설정](#5-5-ssh-보안-설정)
- [5-6. 서비스와 열린 포트 관리](#5-6-서비스와-열린-포트-관리)
- [5-7. 방화벽 (firewalld)](#5-7-방화벽-firewalld)
- [5-8. 지속성(Persistence) 점검](#5-8-지속성persistence-점검)
- [5-9. 의심스러운 파일과 프로세스 찾기](#5-9-의심스러운-파일과-프로세스-찾기)
- [5-10. 패키지 무결성 확인](#5-10-패키지-무결성-확인)
- [5-11. 증거 보존](#5-11-증거-보존)
- [보안 점검 체크리스트](#-보안-점검-체크리스트)

---

## 5-0. 실습 전 준비

- 설정을 바꾸는 실습(SSH, 방화벽) 전에는 VMware **스냅샷**을 찍어두면 잘못돼도 바로 되돌릴 수 있습니다.
- 원격 접속 설정을 바꿀 때는 **기존 접속 창을 닫지 않은 채** 새 창으로 접속을 테스트합니다.

---

## 5-1. 서버 현황 파악하기

점검을 시작할 때 "이 서버가 어떤 상태인지"부터 확인합니다.

```bash
hostnamectl                 # 호스트 이름, OS 버전, 커널
uptime                      # 가동 시간, 부하
ip addr                     # IP 주소
ss -tulnp                   # 열려 있는 포트와 프로세스
systemctl list-units --type=service --state=running   # 실행 중인 서비스
w                           # 현재 접속자와 하는 작업
```

<!-- 📸 실습 캡처: images/5-1-ss-tulnp.png -->

---

## 5-2. 로그에서 공격 흔적 찾기

### 주요 로그 위치

| 파일 / 명령 | 내용 |
|---|---|
| `/var/log/secure` | 인증 관련 로그 (SSH 로그인 성공·실패, `su`, `sudo`) |
| `/var/log/messages` | 시스템 전반 로그 |
| `/var/log/cron` | 예약 작업 실행 기록 |
| `last` | 로그인 성공 기록 (`/var/log/wtmp`) |
| `lastb` | 로그인 **실패** 기록 (`/var/log/btmp`) |
| `journalctl` | systemd 저널 (서비스별 로그) |

### 상황: SSH 무차별 대입 공격이 의심될 때

```bash
# 1) 실패한 로그인 기록 보기
grep "Failed password" /var/log/secure

# 2) 공격 IP별 시도 횟수 집계 (많은 순)
grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head

# 3) 존재하지 않는 계정으로 시도한 기록 (계정 이름 추측 공격)
grep "Invalid user" /var/log/secure

# 4) 공격 이후 로그인에 성공한 기록이 있는지 확인 ← 가장 중요
grep "Accepted" /var/log/secure
last -a | head
```

> 로그 한 줄 예시: `Failed password for root from 192.168.10.5 port 51234 ssh2`
> 끝에서부터 `ssh2`(NF) · `51234`(NF-1) · `port`(NF-2) · **IP(NF-3)** 이므로 `$(NF-3)`으로 IP를 뽑습니다.

```bash
# 실시간으로 로그 지켜보기
tail -f /var/log/secure

# 서비스별 / 기간별 로그
journalctl -u sshd --since today
journalctl -u sshd --since "2025-03-09 09:00" --until "2025-03-09 12:00"
```

**직접 해보기**: 호스트 PC에서 일부러 틀린 비밀번호로 VM에 몇 번 SSH 접속한 뒤, 위 명령으로 내 IP가 집계되는지 확인합니다.

<!-- 📸 실습 캡처: images/5-2-failed-login.png -->

---

## 5-3. 텍스트 처리 도구

로그 분석은 대부분 **필터링 → 추출 → 정렬 → 집계**를 파이프(`|`)로 이어 붙이는 작업입니다.

| 명령 | 역할 | 예시 |
|---|---|---|
| `grep` | 패턴이 있는 줄 고르기 | `grep -i "error" /var/log/messages` (`-i` 대소문자 무시, `-v` 반대) |
| `cut` | 구분자 기준으로 열 자르기 | `cut -d: -f1,7 /etc/passwd` (계정 이름과 쉘) |
| `awk` | 열 추출과 조건 처리 | `awk -F: '$3 >= 1000 {print $1}' /etc/passwd` (일반 사용자 계정) |
| `sort` | 정렬 | `sort -rn` (숫자 기준 내림차순) |
| `uniq` | 연속 중복 제거와 개수 세기 | `sort \| uniq -c` (반드시 정렬 후 사용) |
| `wc` | 줄·단어 수 세기 | `grep "Failed" /var/log/secure \| wc -l` |
| `head` / `tail` | 앞·뒤 일부 보기 | `tail -n 50`, `tail -f` (실시간) |
| `sed` | 치환·특정 줄 출력 | `sed -n '10,20p' file` |

---

## 5-4. 계정 점검

```bash
# root 외에 UID가 0인 계정 (관리자 권한 백도어 계정 의심)
awk -F: '$3 == 0 {print $1}' /etc/passwd

# 비밀번호가 비어 있는 계정
awk -F: '$2 == "" {print $1}' /etc/shadow

# 로그인 가능한 쉘을 가진 계정
grep -v -E "nologin|false" /etc/passwd

# 비밀번호 만료 정책 확인
chage -l kisec

# sudo 권한을 가진 그룹 구성원 (wheel)
getent group wheel
```

| 점검 항목 | 기대 결과 |
|---|---|
| UID 0 계정 | `root` 하나뿐 |
| 빈 비밀번호 계정 | 없음 |
| 로그인 가능한 계정 | 실제 사용하는 계정만 |
| `wheel` 그룹 | 관리자 계정만 |

> `sudo` 권한 설정 파일 `/etc/sudoers`는 문법 오류가 나면 sudo를 아예 쓸 수 없게 되므로, 직접 열지 말고 **`visudo`**로 편집합니다.

<!-- 📸 실습 캡처: images/5-4-account-check.png -->

---

## 5-5. SSH 보안 설정

설정 파일: `/etc/ssh/sshd_config` (Rocky 9은 `/etc/ssh/sshd_config.d/*.conf`에 따로 써도 적용됩니다)

> `sshd_config.d`의 파일은 이름 순서로 읽고 **먼저 나온 값이 적용**됩니다. 설치할 때 root SSH 로그인을 허용했다면 `01-permitrootlogin.conf`가 생기므로, 내 설정 파일 이름은 `00-`으로 시작해야 덮어쓸 수 있습니다. 적용된 최종 값은 `sshd -T`로 확인합니다.

| 설정 | 권장 값 | 이유 |
|---|---|---|
| `PermitRootLogin` | `no` | root로 바로 접속하는 것을 막아 공격 대상 계정을 줄임 |
| `MaxAuthTries` | `3` | 한 번 접속에서 시도할 수 있는 비밀번호 횟수 제한 |
| `PasswordAuthentication` | `no` (키 등록 후) | 비밀번호 대신 **SSH 키**로만 로그인 → 무차별 대입 무력화 |
| `PermitEmptyPasswords` | `no` | 빈 비밀번호 로그인 금지 |

```bash
vi /etc/ssh/sshd_config.d/00-hardening.conf   # 위 설정 작성
sshd -t                                        # 문법 검사 (출력 없으면 정상)
systemctl restart sshd
```

```bash
# SSH 키 로그인 설정 (접속하는 PC에서)
ssh-keygen -t ed25519
ssh-copy-id kisec@<서버 IP>
```

> ⚠️ 포트를 22에서 바꾸려면 SELinux(`semanage port -a -t ssh_port_t -p tcp 2222`)와 방화벽에도 새 포트를 열어야 합니다. 하나라도 빠지면 접속이 끊기니 **기존 세션을 유지한 채** 테스트합니다.

---

## 5-6. 서비스와 열린 포트 관리

```bash
systemctl status sshd                 # 상태 확인
systemctl stop cups                   # 중지
systemctl disable cups                # 부팅 시 자동 시작 해제
systemctl enable --now httpd          # 자동 시작 등록 + 바로 시작
systemctl list-unit-files --type=service --state=enabled   # 부팅 시 켜지는 서비스 목록
```

```bash
ss -tulnp                    # 대기 중인 포트 (t:TCP u:UDP l:LISTEN n:숫자 p:프로세스)
ss -tnp state established    # 현재 연결된 세션
```

**원칙**: 쓰지 않는 서비스는 끄고, 열려 있는 포트 하나하나가 어떤 서비스인지 설명할 수 있어야 합니다.
모르는 포트가 열려 있다면 `ss -tulnp`의 프로세스 이름과 PID로 무엇인지 추적합니다.

---

## 5-7. 방화벽 (firewalld)

Rocky Linux의 기본 방화벽은 **firewalld**이며, `firewall-cmd`로 관리합니다.

```bash
firewall-cmd --state                     # 동작 여부
firewall-cmd --list-all                  # 현재 허용된 서비스·포트

firewall-cmd --permanent --add-service=http       # 서비스 허용
firewall-cmd --permanent --add-port=8080/tcp      # 포트 허용
firewall-cmd --permanent --remove-service=cockpit # 허용 해제
firewall-cmd --reload                             # 적용

# 특정 IP 차단 (5-2에서 찾은 공격 IP)
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.5" reject'
firewall-cmd --reload
```

> `--permanent`를 빼면 즉시 적용되지만 재부팅하면 사라지고, 붙이면 `--reload` 후에 적용됩니다.

<!-- 📸 실습 캡처: images/5-7-firewall.png -->

---

## 5-8. 지속성(Persistence) 점검

공격자는 침입한 뒤 재부팅되거나 접속이 끊겨도 다시 들어올 수 있도록 **자동 실행 장치**를 심어둡니다. 아래 위치에 모르는 항목이 있는지 확인합니다.

| 위치 | 확인 명령 |
|---|---|
| 사용자 예약 작업 | `crontab -l`, `crontab -l -u <계정>`, `ls /var/spool/cron/` |
| 시스템 예약 작업 | `cat /etc/crontab`, `ls /etc/cron.d/ /etc/cron.hourly/ /etc/cron.daily/` |
| systemd 타이머·서비스 | `systemctl list-timers`, `ls /etc/systemd/system/` |
| 쉘 시작 스크립트 | `~/.bashrc`, `~/.bash_profile`, `/etc/profile.d/` |
| SSH 키 | `~/.ssh/authorized_keys` (모르는 공개키가 등록되어 있지 않은지) |
| 부팅 스크립트 | `/etc/rc.d/rc.local` |

```bash
# 모든 계정의 crontab 한 번에 보기
for u in $(cut -d: -f1 /etc/passwd); do crontab -l -u "$u" 2>/dev/null | sed "s/^/$u: /"; done
```

---

## 5-9. 의심스러운 파일과 프로세스 찾기

### 파일

```bash
# 최근 하루 사이 수정된 파일 (가상 파일 시스템 제외)
find / -xdev -type f -mtime -1 2>/dev/null

# root 소유 SetUID 파일 (권한 상승에 악용 가능)
find / -xdev -user root -perm -4000 -type f 2>/dev/null

# 누구나 쓸 수 있는 파일
find / -xdev -type f -perm -o+w 2>/dev/null

# 소유자가 없는 파일 (삭제된 계정이 남긴 파일)
find / -xdev -nouser -o -nogroup 2>/dev/null

# 공격 도구가 자주 숨는 임시 디렉터리
ls -la /tmp /var/tmp /dev/shm
```

| 옵션 | 의미 |
|---|---|
| `-mtime -1` | 내용이 수정된 지 1일 이내 (`-mmin -30`은 30분 이내) |
| `-perm -4000` | SetUID 비트가 켜진 파일 |
| `-xdev` | 다른 파일 시스템(`/proc` 등)으로 넘어가지 않음 |
| `2>/dev/null` | 권한 오류 메시지 숨기기 |

> SetUID 목록은 **평소에 한 번 저장해두고**(`> suid_baseline.txt`) 나중에 `diff`로 비교하면 새로 생긴 파일을 쉽게 찾을 수 있습니다.

### 프로세스

```bash
ps aux --sort=-%cpu | head       # CPU를 많이 쓰는 프로세스 (채굴 악성코드 의심)
ps -ef --forest                  # 부모-자식 관계로 보기
ls -l /proc/<PID>/exe            # 프로세스의 실제 실행 파일 경로
ls -l /proc/<PID>/cwd            # 프로세스의 작업 디렉터리
```

> 실행 파일 경로가 `/tmp`, `/dev/shm`이거나 `(deleted)`로 표시되면 실행 후 파일을 지운 것이라 강하게 의심해야 합니다.

---

## 5-10. 패키지 무결성 확인

```bash
rpm -qa | sort                 # 설치된 패키지 목록
rpm -qf /usr/bin/passwd        # 이 파일이 어느 패키지 소속인지
rpm -V openssh-server          # 패키지 설치 당시와 달라진 파일 확인
rpm -Va                        # 전체 패키지 검사 (시간 오래 걸림)
dnf history                    # 언제 무엇을 설치·삭제했는지
```

`rpm -V` 결과에서 `S`(크기), `5`(해시), `M`(권한), `T`(수정 시간)가 표시되면 원본과 다르다는 뜻입니다.
설정 파일(`c`)이 바뀐 건 정상일 수 있지만, **`/usr/bin` 같은 실행 파일의 해시(`5`)가 바뀌었다면** 바꿔치기를 의심합니다.

```
S.5....T.  c /etc/ssh/sshd_config     ← 설정 파일 수정 (정상일 수 있음)
S.5....T.    /usr/bin/ls              ← 실행 파일 변조 의심!
```

---

## 5-11. 증거 보존

침해가 의심되면 분석하기 전에 원본을 먼저 보존하고, 해시로 **무결성**을 증명합니다.

```bash
tar czf /root/evidence_$(date +%F).tar.gz /var/log/secure* /var/log/messages* /etc/passwd /etc/shadow
sha256sum /root/evidence_*.tar.gz > /root/evidence.sha256
sha256sum -c /root/evidence.sha256    # 나중에 변조되지 않았는지 검증
```

> [4. 정보보호 개론](04-security-basics.md)의 **무결성**과 **감사 추적** 개념이 실제로 쓰이는 지점입니다.

---

## ✅ 보안 점검 체크리스트

| 분류 | 점검 항목 | 확인 방법 | 결과 |
|---|---|---|---|
| 계정 | UID 0 계정이 root 하나뿐인가 | `awk -F: '$3==0' /etc/passwd` | ☐ |
| 계정 | 빈 비밀번호 계정이 없는가 | `awk -F: '$2==""' /etc/shadow` | ☐ |
| 계정 | 불필요한 로그인 가능 계정이 없는가 | `grep -v nologin /etc/passwd` | ☐ |
| SSH | root 직접 로그인이 차단되어 있는가 | `sshd -T \| grep permitrootlogin` | ☐ |
| SSH | 로그인 시도 횟수가 제한되어 있는가 | `sshd -T \| grep maxauthtries` | ☐ |
| 서비스 | 불필요한 서비스가 꺼져 있는가 | `systemctl list-unit-files --state=enabled` | ☐ |
| 네트워크 | 모든 열린 포트의 용도를 아는가 | `ss -tulnp` | ☐ |
| 방화벽 | firewalld가 동작 중이고 필요한 것만 허용했는가 | `firewall-cmd --list-all` | ☐ |
| 권한 | 새로 생긴 SetUID 파일이 없는가 | `find / -perm -4000` + 기준 목록 비교 | ☐ |
| 권한 | 누구나 쓸 수 있는 파일이 없는가 | `find / -perm -o+w -type f` | ☐ |
| 지속성 | 모르는 cron·타이머·authorized_keys가 없는가 | 5-8 표 참고 | ☐ |
| 로그 | 로그인 실패가 몰린 IP가 없는가 | 5-2 집계 명령 | ☐ |
| 무결성 | 실행 파일이 변조되지 않았는가 | `rpm -Va` | ☐ |
