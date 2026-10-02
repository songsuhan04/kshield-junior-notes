# 1. Linux

[← 목차로](README.md)

- [1-1. 실습 환경 구축 (Rocky Linux)](#1-1-실습-환경-구축-rocky-linux)
- [1-2. 유닉스와 리눅스](#1-2-유닉스와-리눅스)
- [1-3. 쉘(Shell)](#1-3-쉘shell)
- [1-4. 기본 명령어](#1-4-기본-명령어)
- [1-5. vim 편집기](#1-5-vim-편집기)
- [1-6. 네트워크 설정](#1-6-네트워크-설정)
- [1-7. 파일 시스템 구조와 링크](#1-7-파일-시스템-구조와-링크)
- [1-8. 파티션, 마운트, LVM](#1-8-파티션-마운트-lvm)
- [1-9. 사용자 관리](#1-9-사용자-관리)
- [1-10. 파일 권한](#1-10-파일-권한)
- [1-11. 프로세스](#1-11-프로세스)

---

## 1-1. 실습 환경 구축 (Rocky Linux)

1. Rocky Linux 공식 사이트 → 다운로드 → 기본 이미지 → **DVD ISO**
2. VMware: `File` → `New Virtual Machine` → **Custom** → *I will install the operating system later*

| 설정 항목 | 값 |
|---|---|
| Guest OS / Version | Linux / Rocky Linux 64-bit |
| Processors | 2 processors, 1 core |
| Memory | 2 GB |
| Network Type | NAT |
| SCSI Controller | LSI Logic |
| Disk Type | NVMe, *Create a new virtual disk*, 20 GB |
| CD/DVD | *Customize Hardware* → *Use ISO image file* (Rocky Linux 9.5) |

3. ▶ 실행 → *Install Rocky Linux* → 설치 대상 지정, root 비밀번호 설정 후 설치

```bash
su -        # root 계정으로 전환 (root 비밀번호 입력)
```

| 프롬프트 | 의미 |
|---|---|
| `$` | 일반 사용자 계정 |
| `#` | 관리자 계정 (리눅스에서는 **root**) |

> 💡 **보충** — 실습용이라도 비밀번호를 문서나 저장소에 그대로 적어두지 않는 습관이 중요합니다.

---

## 1-2. 유닉스와 리눅스

### 유닉스 (UNIX)
- 리눅스 개발의 뿌리가 된 운영체제, **C 언어**로 작성되어 **이식성**이 높음
- 두 계열로 발전 → 현재는 양쪽 장점을 모두 취한 형태
  - **System V 계열**: AT&T, 상업 지향 (SVR4, SCO UNIX, Solaris, HP-UX, AIX)
  - **BSD 계열**: UC 버클리, 연구·개발 지향 (SunOS, FreeBSD, macOS의 기반)

### 리눅스 (Linux)
- 상용 유닉스에 반대해 **오픈 라이선스**를 내세우며 탄생 (1991, 리누스 토르발스)
- 오픈 소스, **GNU**(*GNU's Not Unix*) 프로젝트의 도구들과 결합, 이식성이 높음

| 계열 | 배포판 |
|---|---|
| **Debian** | Ubuntu, Kali Linux(모의 해킹), Linux Mint |
| **Red Hat** | Fedora(데스크톱), RHEL(유료 기술 지원), CentOS(EOS, 지원 종료), **Rocky Linux** |
| 기타 | Mandriva, Slackware, Gentoo, Arch, Asianux, TmaxOS(국산) |

> 💡 **보충** — CentOS가 지원 종료(EOS)된 뒤 RHEL과 호환되는 무료 대안으로 등장한 것이 Rocky Linux입니다. 패키지 관리자는 Debian 계열이 `apt`, Red Hat 계열이 `dnf`(`yum`)입니다.

---

## 1-3. 쉘(Shell)

**쉘**: 사용자와 운영체제(커널) 사이에서 사용자의 명령을 해석해 실행하는 프로그램

- **기능**: 명령어 해석기 · 프로그래밍(스크립트) · 사용자 환경 설정
- **종류**: GUI 쉘 / CLI 쉘 (Bourne Shell `sh`, C Shell `csh`, Korn Shell `ksh`, **Bash** 등)

```bash
echo $SHELL                    # 현재 로그인 쉘
chsh -l                        # 사용 가능한 쉘 목록 (= cat /etc/shells)
chsh                           # 로그인 쉘 변경
cat /etc/passwd | grep kisec   # kisec 계정의 로그인 쉘 확인 → 마지막 필드가 /bin/bash
```

### echo
문자열을 터미널에 출력

| 옵션 | 의미 |
|---|---|
| `-n` | 마지막 개행 문자를 출력하지 않음 |
| `-e` | 백슬래시 이스케이프 문자를 해석 (`\a` 경고음, `\t` 수평 탭, `\v` 수직 탭, `\n` 줄바꿈) |

### 리다이렉션 & 파이프

| 기호 | 의미 |
|---|---|
| `>` | 표준 출력을 파일에 저장 (**덮어쓰기**) |
| `>>` | 표준 출력을 파일 끝에 **이어쓰기** |
| `\|` | 앞 명령의 출력을 뒤 명령의 입력으로 연결 (파이프) |

```bash
echo "hello" > test.txt
echo "world" >> test.txt
cat /etc/passwd | grep root
```

> 💡 **보충** — 표준 에러는 `2>`로 따로 보낼 수 있습니다. 예) `find / -name passwd 2>/dev/null`

---

## 1-4. 기본 명령어

### 종료 · 재시작

```bash
shutdown -h now   # 즉시 종료 (= halt, init 0)
shutdown -h +10   # 10분 후 종료
shutdown -c       # 예약된 종료 취소
shutdown -r now   # 즉시 재시작 (= reboot, init 6)
```

### 파일 · 디렉터리

| 명령어 | 의미 | 주요 옵션 |
|---|---|---|
| `ls` | 파일 목록 (*list*) | `-a` 숨김 포함, `-l` 상세, `-s` 블록 크기, `-t` 수정 시간순, `-R` 하위 포함, `--color` |
| `pwd` | 현재 작업 디렉터리 출력 (*Print Working Directory*) | |
| `cd` | 디렉터리 이동 — 절대경로(`/`로 시작), 상대경로(`.`, `..`) | |
| `cp` | 복사 — `cp [옵션] [원본] [대상]` | `-a` 속성 유지, `-b` 백업, `-d` 심볼릭 링크 유지, `-f` 강제, `-p` 소유자·권한·시간 보존, `-r` 하위 디렉터리 포함, `-u` 원본이 더 새로울 때만 |
| `mv` | 이동 / 이름 변경 | `mv a.txt dir/`, `mv a.txt b.txt` |
| `rm` | 삭제 | `-d` 빈 디렉터리, `-f` 확인 없이, `-i` 하나씩 확인, `-r` 하위 포함, `-v` 과정 출력 |
| `mkdir` | 디렉터리 생성 | `-p` 상위 디렉터리까지 한 번에 |
| `rmdir` | **빈** 디렉터리 삭제 | `-p` 상위까지 |
| `find` | 조건 검색 — `find [경로] [조건] [동작]` | `-name`, `-type`, `-perm`, `-user` |

### 내용 확인 · 검색

| 명령어 | 의미 |
|---|---|
| `cat` | 파일 내용 출력 (*concatenate*) |
| `more` | 화면 단위로 나눠 출력 (`cmd \| more`) |
| `grep` | 패턴이 포함된 줄 출력 (*Global Regular Expression Print*) |
| `history` | 이전 명령 목록. 계정 홈의 `.bash_history`에 저장 (**정상 로그아웃 시** 기록) |
| `man` | 명령어 매뉴얼 |

### 프로세스 · 사용자

| 명령어 | 의미 |
|---|---|
| `ps` | 현재 프로세스 상태 (*process status*) — `ps -ef`, `ps aux` |
| `kill` | 프로세스에 시그널 전송 — `kill [-시그널] PID` (`-9` 강제 종료) |
| `who` | 현재 로그인한 사용자 목록 (`w`는 접속자가 하는 작업까지 표시) |

### 디스크 · 파일 시스템

| 명령어 | 의미 |
|---|---|
| `df -h` | 파일 시스템별 디스크 사용량 |
| `du -sh [경로]` | 디렉터리·파일 용량 |
| `mount` | 장치를 디렉터리에 연결해 사용 가능하게 함 |
| `mkfs` | 장치를 지정한 파일 시스템으로 포맷 (*make file system*) |
| `fsck` | 파일 시스템 무결성 검사 (*file system consistency check*) |

### 가상 콘솔
- 리눅스는 **멀티 유저** OS → 여러 가상 콘솔 제공
- `Ctrl + Alt + F1~F6`: 텍스트 콘솔 전환, 그래픽 환경은 별도 콘솔
- `Ctrl + D`: 로그아웃

> 💡 **보충 (보안 관점)** — `history`, `/var/log/secure`, `last` 명령은 침해 사고 시 공격자의 흔적을 확인하는 기본 도구입니다.

---

## 1-5. vim 편집기

**vim** (*Vi IMproved*): vi를 확장한 터미널 텍스트 편집기

| 모드 | 진입 | 용도 |
|---|---|---|
| 일반 모드 | `Esc` | 이동, 복사(`yy`), 붙여넣기(`p`), 삭제(`dd`) |
| 입력 모드 | `i`, `a`, `o` | 텍스트 입력 |
| 명령 모드 | `:` | 저장(`:w`), 종료(`:q`), 강제 종료(`:q!`), 설정(`:set nu`) |
| 비주얼 모드 | `v`, `V` | 범위 선택 |

```vim
:set nu        " 줄 번호 표시
:set ts=4      " 탭 크기
:%s/old/new/g  " 전체 치환
```

---

## 1-6. 네트워크 설정

- 같은 네트워크 대역 안에서 **중복되지 않는 IP 주소**를 지정해야 함
- 리눅스는 장치까지 모두 **파일**로 다루므로, 네트워크 설정도 설정 파일 형태로 관리됨

> 💡 **보충** — Rocky Linux 9은 NetworkManager를 사용합니다.
> ```bash
> ip addr                     # 인터페이스 / IP 확인
> nmcli connection show       # 연결 목록
> nmtui                       # 텍스트 UI로 IP 설정
> ```
> 설정 파일은 `/etc/NetworkManager/system-connections/`에 저장됩니다.

---

## 1-7. 파일 시스템 구조와 링크

리눅스 파일 시스템은 `/`(root)를 꼭대기로 하는 **트리 구조**이며, 디렉터리 · 일반 파일 · 특수 파일(장치 등)로 구성됩니다.

### `ls -l` 결과 읽기

```
-rw-rw-r--. 1 kisec kisec 16 Mar  9 09:08 test.txt
```

| 필드 | 값 | 의미 |
|---|---|---|
| 파일 종류 | `-` | `-` 일반 파일, `d` 디렉터리, `l` 심볼릭 링크 |
| 권한 | `rw-rw-r--` | 소유자 / 그룹 / 기타 사용자 권한 |
| 링크 수 | `1` | 하드 링크 개수 |
| 소유자 | `kisec` | 파일 소유 계정 |
| 그룹 | `kisec` | 소유 그룹 |
| 크기 | `16` | 바이트 |
| 수정 일시 | `Mar 9 09:08` | 마지막 변경 시각 |
| 이름 | `test.txt` | 파일 이름 |

### inode
- 리눅스는 파일에 접근할 때 이름이 아니라 이름과 매핑된 **번호(inode number)**로 접근
- inode에는 파일의 위치, 크기, 권한 등 메타데이터가 저장됨 (`ls -i`로 확인)

### 링크 파일

| 구분 | 하드 링크 | 심볼릭 링크 |
|---|---|---|
| 생성 | `ln 원본 링크` | `ln -s 원본 링크` |
| 가리키는 대상 | 원본과 **같은 inode** | 원본 파일의 **경로(이름)** |
| 원본 삭제 시 | 데이터 유지 (링크 수만 감소) | 링크가 깨짐 |
| 비유 | 같은 데이터에 이름 하나 더 붙이기 | 윈도우의 **바로가기** |

> inode는 자신을 가리키는 이름(하드 링크)이 하나도 남지 않으면 삭제됩니다.

---

## 1-8. 파티션, 마운트, LVM

| 용어 | 의미 |
|---|---|
| 파일 시스템 | 데이터를 저장·검색하는 방식을 제어하는 체계 (ext4, xfs 등) |
| 파티션 | 하나의 디스크를 여러 영역으로 나눈 것 — **고정적·물리적** 개념 |
| 볼륨 | 디스크 위의 논리적 저장 단위 — **유동적·논리적** 개념 |
| 마운트 | 저장 장치를 디렉터리 구조의 특정 경로에 연결하는 것 |

### 장치 이름 읽기: `/dev/sda3`

| 글자 | 의미 |
|---|---|
| `sd` | SCSI/SATA 디스크 (NVMe는 `nvme0n1`) |
| `a` | 첫 번째 디스크 (두 번째는 `b`) |
| `3` | 그 디스크의 **3번째 파티션** |

### 디스크 추가 절차

```
전원 OFF → 디스크 장착 → 파티션 생성(fdisk) → 파일 시스템 생성(mkfs)
→ 마운트 포인트 생성(mkdir) → mount → 부팅 시 자동 연결을 위해 /etc/fstab 등록
```

### LVM (Logical Volume Manager)
디스크를 묶고 나눠 **유연하게 크기를 조절**할 수 있게 해주는 기술

```
디스크 장착 → 파티션 타입을 LVM(8e)으로 생성
→ PV(물리 볼륨) 생성 → VG(볼륨 그룹)로 묶기 → LV(논리 볼륨) 생성 → 포맷 후 마운트
```

> 💡 **보충** — 대응 명령어
> ```bash
> pvcreate /dev/sdb1
> vgcreate myvg /dev/sdb1
> lvcreate -L 5G -n mylv myvg
> mkfs.xfs /dev/myvg/mylv
> mount /dev/myvg/mylv /data
> ```

---

## 1-9. 사용자 관리

```bash
useradd kisec        # 사용자 추가 (adduser도 가능)
passwd kisec         # 비밀번호 설정
usermod -aG wheel kisec   # 그룹 추가 (wheel = sudo 권한 그룹)
userdel -r kisec     # 홈 디렉터리까지 삭제
```

> 💡 **보충** — 관련 파일
> | 파일 | 내용 |
> |---|---|
> | `/etc/passwd` | 계정 정보 (이름:x:UID:GID:설명:홈:쉘) |
> | `/etc/shadow` | 암호화된 비밀번호 (root만 읽기 가능) |
> | `/etc/group` | 그룹 정보 |

---

## 1-10. 파일 권한

- **대상**: 사용자(u, owner) · 그룹(g) · 기타(o)
- **권한**: 읽기(`r`=4) · 쓰기(`w`=2) · 실행(`x`=1)

```bash
chmod 754 file.sh    # rwx r-x r--
chmod u+x file.sh    # 소유자에게 실행 권한 추가
chown user:group file
```

### umask
새 파일·디렉터리의 **기본 권한**을 정하는 값. 기본값에서 umask에 해당하는 비트를 빼는(보수와 AND 연산) 방식입니다.

| 대상 | 기본 최대 권한 | umask `022` 적용 결과 |
|---|---|---|
| 파일 | `666` (rw-rw-rw-) | `644` (rw-r--r--) |
| 디렉터리 | `777` (rwxrwxrwx) | `755` (rwxr-xr-x) |

### 특수 권한

| 권한 | 숫자 | 의미 | 예 |
|---|---|---|---|
| **SetUID** | 4000 | 실행하는 동안 **파일 소유자** 권한으로 실행 | `/usr/bin/passwd` (`-rwsr-xr-x`) |
| **SetGID** | 2000 | 실행하는 동안 **파일 그룹** 권한으로 실행 | |
| Sticky Bit | 1000 | 디렉터리 안 파일은 소유자만 삭제 가능 | `/tmp` (`drwxrwxrwt`) |

> 💡 **보충 (보안 관점)** — root 소유 SetUID 파일은 권한 상승 공격에 악용될 수 있어 보안 점검 대상입니다.
> ```bash
> find / -user root -perm -4000 2>/dev/null
> ```

---

## 1-11. 프로세스

- 리눅스는 수백 개 이상의 프로그램을 동시에 실행할 수 있으며, 실행 중인 프로그램을 **프로세스**라 함
- 각 프로세스는 고유 번호 **PID**로 관리됨
- 프로세스의 정의
  - 실행 중인 프로그램
  - **PCB**(Process Control Block)를 가진 프로그램
  - 프로그램 카운터를 가진, 순차적으로 수행되는 능동적 개체
- **포어그라운드** / **백그라운드** 프로세스로 구분

```bash
sleep 100 &    # 백그라운드 실행
jobs           # 작업 목록
fg %1          # 포어그라운드로 가져오기
ps -ef | grep sleep
kill -9 <PID>
```
