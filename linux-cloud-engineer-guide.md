# Linux 완벽 가이드 — Cloud Engineer 대상
## Ubuntu 24.04 LTS 기준 | 기초 → 고급 + 글로벌 IT 기업 인터뷰 문제

---

# PART 1: 기초 개념 (Foundation)

---

## Chapter 1: Linux 아키텍처와 부팅 프로세스

### 1.1 Linux 커널 아키텍처

Linux는 **모놀리식 커널(Monolithic Kernel)** 기반 운영체제다. 커널은 하드웨어와 사용자 공간(User Space) 사이의 추상화 계층으로 동작하며, 다음 핵심 서브시스템으로 구성된다.

**커널 공간(Kernel Space)의 주요 구성 요소:**

- **Process Scheduler**: CFS(Completely Fair Scheduler)를 사용하여 CPU 시간을 프로세스에 공정하게 분배한다. `nice` 값(-20~19)과 `priority` 값으로 스케줄링 우선순위를 조정한다.
- **Memory Manager**: 가상 메모리 관리, 페이지 테이블 관리, swap 관리, OOM Killer 등을 담당한다. 물리 메모리를 4KB 페이지 단위로 관리하며, Copy-on-Write(CoW) 기법으로 fork 시 메모리 효율을 극대화한다.
- **VFS (Virtual File System)**: ext4, XFS, Btrfs 등 다양한 파일시스템을 통합된 인터페이스로 추상화한다. 모든 것을 파일로 취급하는 Unix 철학의 핵심 구현체다.
- **Network Stack**: OSI 7계층 모델을 구현하며, Netfilter(iptables/nftables의 기반), TCP/IP 스택, 소켓 인터페이스를 제공한다.
- **IPC (Inter-Process Communication)**: 파이프, 시그널, 공유 메모리, 세마포어, 메시지 큐, Unix 소켓 등 프로세스 간 통신 메커니즘을 제공한다.

**User Space vs Kernel Space:**

```
┌─────────────────────────────────────────────┐
│              User Space                      │
│  ┌─────────┐ ┌─────────┐ ┌──────────────┐  │
│  │ 애플리케이션│ │  Shell  │ │ System Daemons│  │
│  └────┬────┘ └────┬────┘ └──────┬───────┘  │
│       │           │              │           │
│  ┌────▼───────────▼──────────────▼───────┐  │
│  │         GNU C Library (glibc)          │  │
│  └────────────────┬──────────────────────┘  │
├───────────────────┼─────────────────────────┤
│  System Call Interface (syscall)             │
├───────────────────┼─────────────────────────┤
│              Kernel Space                    │
│  ┌────────────────▼──────────────────────┐  │
│  │  Process │ Memory │  VFS  │ Network   │  │
│  │  Mgmt    │ Mgmt   │       │ Stack     │  │
│  └────────────────┬──────────────────────┘  │
│  ┌────────────────▼──────────────────────┐  │
│  │        Device Drivers / Modules        │  │
│  └────────────────┬──────────────────────┘  │
├───────────────────┼─────────────────────────┤
│  ┌────────────────▼──────────────────────┐  │
│  │           Hardware (CPU, RAM, Disk)    │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

커널 버전 확인:
```bash
uname -r          # 예: 6.8.0-31-generic
uname -a          # 전체 시스템 정보
cat /proc/version # 커널 빌드 정보
```

### 1.2 부팅 프로세스 (Boot Process) 상세

Ubuntu 24.04의 부팅 과정은 다음 단계를 거친다:

**① UEFI/BIOS → ② GRUB2 → ③ Kernel → ④ systemd (PID 1) → ⑤ Target(Runlevel)**

**단계별 상세:**

1. **UEFI/BIOS POST**: 하드웨어 초기화 및 자가 진단(Power-On Self-Test). UEFI는 GPT 파티션의 EFI System Partition(ESP)에서 부트로더를 탐색한다.

2. **GRUB2 (GRand Unified Bootloader 2)**: `/boot/grub/grub.cfg`를 읽어 커널 이미지와 initramfs를 메모리에 로드한다.
```bash
# GRUB 설정 수정
sudo vim /etc/default/grub
sudo update-grub  # grub.cfg 재생성

# 주요 GRUB 파라미터
GRUB_TIMEOUT=5
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""  # 커널 파라미터 추가 위치
```

3. **Kernel 초기화**: initramfs(초기 RAM 파일시스템)를 마운트하고, 하드웨어 드라이버를 로드한 뒤, 실제 루트 파일시스템으로 `pivot_root`를 수행한다. 이후 PID 1인 `/sbin/init` → systemd를 실행한다.

4. **systemd 초기화**: 의존성 그래프(dependency graph)에 따라 서비스를 병렬로 시작한다.
```bash
# 부팅 시간 분석
systemd-analyze                    # 전체 부팅 시간
systemd-analyze blame              # 서비스별 소요 시간
systemd-analyze critical-chain     # 크리티컬 패스 표시
systemd-analyze plot > boot.svg    # 시각화

# 부팅 로그 확인
journalctl -b        # 현재 부팅 로그
journalctl -b -1     # 이전 부팅 로그
journalctl --list-boots  # 모든 부팅 기록
```

5. **Target 도달**: `default.target`(보통 `multi-user.target` 또는 `graphical.target`)에 도달하면 부팅 완료.
```bash
systemctl get-default                        # 현재 기본 타겟
sudo systemctl set-default multi-user.target # 서버용 (GUI 없음)
sudo systemctl set-default graphical.target  # 데스크톱용
```

---

## Chapter 2: 파일시스템과 디렉토리 구조

### 2.1 FHS (Filesystem Hierarchy Standard)

```
/                   # 루트 디렉토리
├── bin -> usr/bin  # 필수 사용자 명령어 (Ubuntu 24.04는 /usr/bin으로 심볼릭 링크)
├── boot/           # 커널 이미지, initramfs, GRUB 파일
├── dev/            # 장치 파일 (udev가 동적 생성)
│   ├── sda         # SCSI/SATA 디스크
│   ├── nvme0n1     # NVMe SSD
│   ├── null        # /dev/null (블랙홀)
│   ├── zero        # 무한 0 바이트 스트림
│   ├── random      # 엔트로피 기반 난수
│   └── urandom     # 비차단 난수
├── etc/            # 시스템 설정 파일 (Editable Text Configuration)
│   ├── fstab       # 파일시스템 마운트 테이블
│   ├── passwd      # 사용자 계정 정보
│   ├── shadow      # 암호화된 패스워드
│   ├── group       # 그룹 정보
│   ├── hostname    # 호스트명
│   ├── hosts       # 정적 호스트명 해석
│   ├── resolv.conf # DNS 설정
│   ├── netplan/    # Ubuntu 네트워크 설정 (Netplan YAML)
│   ├── ssh/        # SSH 서버 설정
│   ├── systemd/    # systemd 설정
│   └── apt/        # APT 패키지 관리자 설정
├── home/           # 일반 사용자 홈 디렉토리
├── lib -> usr/lib  # 공유 라이브러리
├── media/          # 이동식 미디어 자동 마운트 지점
├── mnt/            # 임시 마운트 지점
├── opt/            # 서드파티 애플리케이션
├── proc/           # 프로세스 및 커널 정보 (가상 파일시스템)
│   ├── cpuinfo     # CPU 정보
│   ├── meminfo     # 메모리 정보
│   ├── loadavg     # 시스템 부하
│   ├── [PID]/      # 프로세스별 정보
│   └── sys/        # 커널 튜닝 파라미터
├── root/           # root 사용자 홈 디렉토리
├── run/            # 런타임 데이터 (tmpfs)
├── sbin -> usr/sbin # 시스템 관리 명령어
├── srv/            # 서비스 데이터 (웹서버 등)
├── sys/            # sysfs (커널 장치 트리)
├── tmp/            # 임시 파일 (재부팅 시 삭제)
├── usr/            # 사용자 프로그램, 라이브러리
│   ├── bin/        # 사용자 명령어
│   ├── sbin/       # 시스템 관리 명령어
│   ├── lib/        # 라이브러리
│   ├── local/      # 로컬 설치 프로그램
│   └── share/      # 아키텍처 독립 데이터
└── var/            # 가변 데이터
    ├── log/        # 시스템 로그
    ├── cache/      # 애플리케이션 캐시
    ├── spool/      # 큐 데이터 (메일, 프린트)
    └── lib/        # 상태 정보 (DB 등)
```

### 2.2 파일시스템 유형과 마운트

```bash
# 디스크 및 파티션 확인
lsblk                    # 블록 장치 트리 구조
lsblk -f                 # 파일시스템 포함 표시
fdisk -l                 # 디스크 파티션 상세
blkid                    # UUID 및 파일시스템 타입

# 파일시스템 생성
sudo mkfs.ext4 /dev/sdb1          # ext4 생성
sudo mkfs.xfs /dev/sdb2           # XFS 생성 (대용량 파일에 유리)
sudo mkfs.btrfs /dev/sdb3         # Btrfs 생성 (스냅샷, 압축 지원)

# 마운트
sudo mount /dev/sdb1 /mnt/data
sudo mount -t nfs server:/share /mnt/nfs   # NFS 마운트
sudo mount -o loop disk.iso /mnt/iso       # ISO 파일 마운트

# 영구 마운트 (/etc/fstab)
# <device>    <mount>    <type>  <options>        <dump> <pass>
UUID=xxxx     /data      ext4    defaults,noatime  0      2
tmpfs         /tmp       tmpfs   defaults,noexec   0      0

# fstab 변경 후 검증 (재부팅 없이)
sudo mount -a          # fstab의 모든 항목 마운트
sudo findmnt --verify  # fstab 문법 검증

# 디스크 사용량
df -h          # 마운트된 파일시스템 사용량
df -i          # inode 사용량 (파일 수 제한 확인)
du -sh /var/*  # 디렉토리별 사용량
du -h --max-depth=1 /  # 루트 직하 디렉토리 용량
```

### 2.3 inode와 링크

**inode**는 파일의 메타데이터(권한, 소유자, 크기, 타임스탬프, 데이터 블록 포인터)를 저장하는 데이터 구조다. 파일명은 inode에 저장되지 않으며, 디렉토리 엔트리가 파일명과 inode 번호를 매핑한다.

```bash
ls -i file.txt          # inode 번호 확인
stat file.txt           # 파일 메타데이터 전체 확인

# 하드 링크: 동일 inode를 공유 (같은 파일시스템 내에서만 가능, 디렉토리 불가)
ln original.txt hardlink.txt

# 심볼릭 링크: 경로를 가리키는 별도 파일 (다른 파일시스템, 디렉토리 가능)
ln -s /path/to/original symlink.txt

# 차이점 확인
ls -li original.txt hardlink.txt symlink.txt
# 하드링크: inode 동일, 링크 카운트 증가
# 심볼릭 링크: 별도 inode, 원본 삭제 시 깨짐 (dangling symlink)
```

---

## Chapter 3: 사용자, 그룹, 퍼미션 관리

### 3.1 사용자와 그룹

```bash
# 사용자 생성
sudo useradd -m -s /bin/bash -G sudo,docker username
# -m: 홈 디렉토리 생성
# -s: 기본 셸 지정
# -G: 보조 그룹 추가

sudo passwd username           # 비밀번호 설정
sudo usermod -aG docker user   # 기존 사용자에 그룹 추가 (-a 필수!)
sudo userdel -r username       # 사용자 삭제 (-r: 홈 디렉토리도 삭제)

# 사용자 정보 확인
id username              # UID, GID, 그룹 목록
whoami                   # 현재 사용자
groups username          # 소속 그룹
getent passwd username   # /etc/passwd 엔트리

# /etc/passwd 형식
# username:x:1000:1000:Full Name:/home/username:/bin/bash
# 필드: 사용자명:패스워드(x=shadow):UID:GID:설명:홈:셸

# /etc/shadow 형식 (root만 읽기 가능)
# username:$6$salt$hash:19500:0:99999:7:::
# 필드: 사용자명:해시:마지막변경:최소:최대:경고:비활성:만료

# 그룹 관리
sudo groupadd developers
sudo groupdel developers
getent group groupname
```

### 3.2 파일 퍼미션 (Permission)

```
퍼미션 표기법:
-rwxr-xr-- 1 user group 4096 Jan 1 00:00 file.txt
│└┬┘└┬┘└┬┘
│ │  │  └── Others: r-- (4) = 읽기만
│ │  └───── Group:  r-x (5) = 읽기+실행
│ └──────── Owner:  rwx (7) = 읽기+쓰기+실행
└────────── Type:   - (일반파일), d (디렉토리), l (심볼릭링크)
            c (문자장치), b (블록장치), s (소켓), p (파이프)
```

```bash
# chmod: 퍼미션 변경
chmod 755 script.sh        # rwxr-xr-x (8진수)
chmod u+x file             # 소유자에 실행 권한 추가
chmod g-w file             # 그룹에서 쓰기 권한 제거
chmod o=r file             # 기타 사용자 읽기만 설정
chmod -R 750 /app          # 재귀적 적용

# chown: 소유자/그룹 변경
sudo chown user:group file
sudo chown -R www-data:www-data /var/www

# 디렉토리에서의 퍼미션 의미:
# r (4): 디렉토리 내 파일 목록 조회 (ls)
# w (2): 디렉토리 내 파일 생성/삭제
# x (1): 디렉토리 진입 (cd) 및 파일 접근
```

### 3.3 특수 퍼미션 (SUID, SGID, Sticky Bit)

```bash
# SUID (Set User ID) - 4000
# 실행 시 파일 소유자 권한으로 실행 (보안 주의!)
chmod u+s /usr/bin/passwd   # 또는 chmod 4755
ls -l /usr/bin/passwd       # -rwsr-xr-x (s = SUID)

# SGID (Set Group ID) - 2000
# 파일: 실행 시 그룹 소유자 권한으로 실행
# 디렉토리: 하위 생성 파일이 디렉토리의 그룹을 상속
chmod g+s /shared/project   # 또는 chmod 2775
ls -ld /shared/project      # drwxrwsr-x

# Sticky Bit - 1000
# 디렉토리 내 파일을 소유자만 삭제 가능 (/tmp에 기본 설정)
chmod +t /shared            # 또는 chmod 1777
ls -ld /tmp                 # drwxrwxrwt (t = sticky bit)
```

### 3.4 ACL (Access Control List)

기본 Unix 퍼미션으로 부족할 때 세밀한 접근 제어를 위해 사용한다.

```bash
# ACL 설정
setfacl -m u:alice:rwx /data/project     # alice에게 rwx
setfacl -m g:devteam:rx /data/project    # devteam 그룹에 rx
setfacl -m d:g:devteam:rx /data/project  # default ACL (하위 파일에 상속)
setfacl -x u:alice /data/project         # alice의 ACL 제거
setfacl -b /data/project                 # 모든 ACL 제거

# ACL 확인
getfacl /data/project
# ls -l에서 퍼미션 끝에 '+' 표시가 나타남
```

### 3.5 umask

새 파일/디렉토리 생성 시 기본 퍼미션을 결정한다.

```bash
umask          # 현재 umask 확인 (보통 0022)
umask 0027     # 설정 변경

# 계산법:
# 파일 기본값: 666 - umask = 666 - 022 = 644 (rw-r--r--)
# 디렉토리 기본값: 777 - umask = 777 - 022 = 755 (rwxr-xr-x)

# 영구 설정: ~/.bashrc 또는 /etc/profile에 추가
```

---

## Chapter 4: 프로세스 관리

### 4.1 프로세스 기본 개념

프로세스는 실행 중인 프로그램의 인스턴스다. 각 프로세스는 고유한 PID를 가지며, 부모 프로세스(PPID)로부터 fork()된다.

**프로세스 상태:**
- **R (Running)**: CPU에서 실행 중 또는 실행 대기
- **S (Sleeping)**: 인터럽트 가능한 대기 (I/O 등)
- **D (Disk Sleep)**: 인터럽트 불가능한 대기 (디스크 I/O)
- **Z (Zombie)**: 종료되었지만 부모가 wait()하지 않은 상태
- **T (Stopped)**: 시그널에 의해 중지됨

```bash
# 프로세스 조회
ps aux                      # 모든 프로세스 (BSD 스타일)
ps -ef                      # 모든 프로세스 (System V 스타일)
ps -ef --forest             # 트리 구조
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem  # 메모리 순 정렬

# 실시간 모니터링
top                         # 기본 모니터
htop                        # 향상된 모니터 (설치: sudo apt install htop)

# 프로세스 검색
pgrep -a nginx              # 이름으로 PID 검색
pidof sshd                  # 정확한 이름으로 PID

# /proc/[PID] 활용
cat /proc/1/cmdline         # PID 1의 실행 명령
ls -l /proc/1/fd            # 열린 파일 디스크립터
cat /proc/1/status          # 프로세스 상태 상세
cat /proc/1/maps            # 메모리 맵

# 프로세스 종료
kill PID                    # SIGTERM (15) - 정상 종료 요청
kill -9 PID                 # SIGKILL (9) - 강제 종료 (최후 수단)
kill -HUP PID               # SIGHUP (1) - 설정 리로드
killall nginx               # 이름으로 모든 프로세스 종료
pkill -f "python app.py"    # 패턴 매칭으로 종료
```

### 4.2 Job Control

```bash
# 백그라운드/포그라운드
command &                   # 백그라운드 실행
Ctrl+Z                      # 현재 작업 일시 중지
bg %1                       # 작업 1을 백그라운드로
fg %1                       # 작업 1을 포그라운드로
jobs                        # 현재 셸의 작업 목록

# nohup: 셸 종료 후에도 계속 실행
nohup long_running_command > output.log 2>&1 &

# disown: 이미 실행 중인 작업을 셸에서 분리
disown %1
```

### 4.3 systemd 서비스 관리

```bash
# 서비스 제어
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx        # 설정만 리로드 (무중단)
sudo systemctl enable nginx        # 부팅 시 자동 시작
sudo systemctl disable nginx
sudo systemctl status nginx
sudo systemctl is-active nginx
sudo systemctl is-enabled nginx

# 서비스 유닛 파일 작성 (/etc/systemd/system/myapp.service)
```
```ini
[Unit]
Description=My Application
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/start.sh
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
LimitNOFILE=65536
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```
```bash
# 유닛 파일 수정 후
sudo systemctl daemon-reload
sudo systemctl restart myapp

# 서비스 로그
journalctl -u nginx -f              # 실시간 로그
journalctl -u nginx --since today   # 오늘 로그
journalctl -u nginx -p err          # 에러만
```

---

## Chapter 5: 패키지 관리 (APT)

```bash
# 저장소 업데이트 및 패키지 설치
sudo apt update                     # 패키지 목록 갱신
sudo apt upgrade                    # 설치된 패키지 업그레이드
sudo apt full-upgrade               # 의존성 변경 포함 업그레이드
sudo apt install nginx              # 패키지 설치
sudo apt remove nginx               # 패키지 제거 (설정 유지)
sudo apt purge nginx                # 패키지 + 설정 완전 제거
sudo apt autoremove                 # 미사용 의존성 제거

# 패키지 검색 및 정보
apt search keyword
apt show nginx                      # 패키지 상세 정보
apt list --installed                # 설치된 패키지 목록
dpkg -l | grep nginx                # dpkg로 검색
dpkg -L nginx                       # 패키지가 설치한 파일 목록
dpkg -S /usr/sbin/nginx             # 파일이 속한 패키지

# 저장소 관리
sudo add-apt-repository ppa:deadsnakes/ppa
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

# .deb 파일 직접 설치
sudo dpkg -i package.deb
sudo apt install -f                 # 의존성 해결

# 패키지 고정 (업그레이드 방지)
sudo apt-mark hold nginx
sudo apt-mark unhold nginx
apt-mark showhold
```

---

# PART 2: 중급 개념 (Intermediate)

---

## Chapter 6: 셸 스크립팅 (Bash)

### 6.1 Bash 기본 문법

```bash
#!/bin/bash
set -euo pipefail  # 프로덕션 스크립트 필수!
# -e: 에러 발생 시 즉시 종료
# -u: 미정의 변수 사용 시 에러
# -o pipefail: 파이프라인 내 에러도 감지

# 변수
NAME="Cloud Engineer"
readonly DB_HOST="10.0.1.100"    # 상수
export PATH="$PATH:/opt/bin"     # 환경 변수 (자식 프로세스에 전달)

# 변수 확장 (Parameter Expansion)
echo ${VAR:-default}    # VAR 비어있으면 default 반환
echo ${VAR:=default}    # VAR 비어있으면 default 할당 및 반환
echo ${VAR:+alternate}  # VAR 설정되어있으면 alternate 반환
echo ${VAR:?error msg}  # VAR 비어있으면 에러 메시지 출력 후 종료
echo ${#VAR}            # 문자열 길이
echo ${VAR%.txt}        # 뒤에서 .txt 제거 (shortest match)
echo ${VAR%%.*}         # 뒤에서 .* 제거 (longest match)
echo ${VAR#*/}          # 앞에서 */ 제거 (shortest match)
echo ${VAR##*/}         # 앞에서 */ 제거 (longest match) = basename

# 조건문
if [[ "$STATUS" -eq 0 ]]; then
    echo "Success"
elif [[ "$STATUS" -eq 1 ]]; then
    echo "Warning"
else
    echo "Error"
fi

# 문자열 비교: ==, !=, =~ (정규식)
# 숫자 비교: -eq, -ne, -lt, -le, -gt, -ge
# 파일 테스트: -f (파일), -d (디렉토리), -e (존재), -r (읽기가능),
#             -w (쓰기가능), -x (실행가능), -s (크기>0), -L (심볼릭링크)

# 반복문
for server in web{1..5}.example.com; do
    ssh "$server" "uptime"
done

for file in /var/log/*.log; do
    gzip "$file"
done

while IFS= read -r line; do
    echo "Processing: $line"
done < servers.txt

# 함수
check_disk_usage() {
    local threshold="${1:-80}"  # 로컬 변수, 기본값 80
    local usage
    usage=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
    if (( usage > threshold )); then
        echo "WARNING: Disk usage ${usage}% exceeds ${threshold}%"
        return 1
    fi
    return 0
}

# 배열
servers=("web1" "web2" "db1" "db2")
echo "${servers[0]}"         # 첫 번째 요소
echo "${servers[@]}"         # 모든 요소
echo "${#servers[@]}"        # 배열 길이
servers+=("cache1")          # 요소 추가

# 연관 배열 (Bash 4+)
declare -A config
config[host]="10.0.1.100"
config[port]="5432"
echo "${config[host]}"

# Trap (시그널 핸들링)
cleanup() {
    echo "Cleaning up temp files..."
    rm -f /tmp/myapp_*
}
trap cleanup EXIT ERR     # 종료/에러 시 cleanup 실행
trap 'echo "Interrupted"' INT  # Ctrl+C 핸들링
```

### 6.2 실전 스크립트 예제: 서버 헬스체크

```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="/var/log/healthcheck.log"
ALERT_EMAIL="ops@company.com"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

log() { echo "[$TIMESTAMP] $1" | tee -a "$LOG_FILE"; }

check_disk() {
    while IFS= read -r line; do
        usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
        mount=$(echo "$line" | awk '{print $6}')
        if (( usage > 85 )); then
            log "CRITICAL: Disk ${mount} at ${usage}%"
            return 1
        fi
    done < <(df -h | grep -E '^/dev/')
    log "OK: Disk usage normal"
}

check_memory() {
    local mem_available
    mem_available=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
    local mem_total
    mem_total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
    local pct=$(( (mem_total - mem_available) * 100 / mem_total ))
    if (( pct > 90 )); then
        log "CRITICAL: Memory usage at ${pct}%"
        return 1
    fi
    log "OK: Memory usage at ${pct}%"
}

check_load() {
    local cores
    cores=$(nproc)
    local load
    load=$(awk '{print $1}' /proc/loadavg)
    local load_int=${load%.*}
    if (( load_int > cores * 2 )); then
        log "WARNING: Load average $load (cores: $cores)"
        return 1
    fi
    log "OK: Load average $load"
}

check_services() {
    local services=("nginx" "postgresql" "redis-server")
    for svc in "${services[@]}"; do
        if ! systemctl is-active --quiet "$svc"; then
            log "CRITICAL: $svc is not running"
            return 1
        fi
    done
    log "OK: All services running"
}

main() {
    local exit_code=0
    check_disk    || exit_code=1
    check_memory  || exit_code=1
    check_load    || exit_code=1
    check_services || exit_code=1

    if (( exit_code != 0 )); then
        log "ALERT: Issues detected - sending notification"
        # mail -s "Server Alert" "$ALERT_EMAIL" < "$LOG_FILE"
    fi
    return $exit_code
}

main "$@"
```

---

## Chapter 7: 텍스트 처리와 데이터 파이프라인

### 7.1 핵심 텍스트 처리 도구

```bash
# grep: 패턴 검색
grep -r "ERROR" /var/log/           # 재귀 검색
grep -i "error" log.txt             # 대소문자 무시
grep -n "pattern" file.txt          # 줄 번호 표시
grep -c "error" log.txt             # 매칭 횟수
grep -v "DEBUG" log.txt             # 역매칭 (제외)
grep -E "error|warn|critical" log   # 확장 정규식 (egrep)
grep -P '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}' log  # Perl 정규식 (IP 매칭)
grep -A3 -B2 "ERROR" log.txt       # 전후 라인 (After 3, Before 2)
grep -l "TODO" *.py                 # 매칭된 파일명만 출력

# sed: 스트림 편집기
sed 's/old/new/' file               # 첫 번째만 치환
sed 's/old/new/g' file              # 전체 치환
sed -i 's/old/new/g' file           # in-place 수정
sed -i.bak 's/old/new/g' file       # 백업 생성 후 수정
sed -n '10,20p' file                # 10~20번 라인만 출력
sed '/^#/d' config.conf             # 주석 라인 삭제
sed '/^$/d' file                    # 빈 라인 삭제
sed -n '/START/,/END/p' file        # 범위 출력

# awk: 패턴 스캐닝 & 텍스트 처리
awk '{print $1, $3}' file           # 1, 3번째 필드
awk -F: '{print $1}' /etc/passwd    # 구분자 지정
awk '$3 > 1000' /etc/passwd         # 조건 필터
awk '{sum+=$5} END {print sum}' file # 합계 계산
awk 'NR==1 {header=$0; next}       # 헤더 스킵
     {print NR": "$0}' file         # 라인번호와 함께 출력
awk '{count[$1]++} END {for (k in count) print k, count[k]}' access.log  # 빈도 집계

# cut, sort, uniq, wc
cut -d: -f1,3 /etc/passwd           # 필드 추출
sort -t: -k3 -n /etc/passwd         # 3번째 필드 숫자 정렬
sort -u file                        # 중복 제거 + 정렬
uniq -c sorted_file                 # 연속 중복 카운트
wc -l file                          # 라인 수
wc -w file                          # 단어 수

# tr: 문자 변환
echo "HELLO" | tr 'A-Z' 'a-z'      # 소문자 변환
echo "hello world" | tr -d ' '     # 공백 삭제
echo "aabbcc" | tr -s 'a-z'        # 연속 중복 제거

# xargs: 표준 입력을 인자로 변환
find /tmp -name "*.tmp" -print0 | xargs -0 rm -f   # null 구분
cat servers.txt | xargs -I{} ssh {} "uptime"         # 치환 문자열
find . -name "*.log" | xargs -P4 gzip               # 병렬 처리 (4 프로세스)
```

### 7.2 실전 로그 분석 파이프라인

```bash
# Nginx 액세스 로그에서 상위 10개 IP 추출
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# HTTP 5xx 에러를 발생시킨 URL 분석
awk '$9 ~ /^5/' /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head -20

# 시간대별 요청 수 (히스토그램)
awk '{print $4}' access.log | cut -d: -f2 | sort | uniq -c | sort -k2 -n

# 실시간 로그 모니터링 + 필터
tail -f /var/log/syslog | grep --line-buffered "error" | while read line; do
    echo "[$(date)] $line" >> /var/log/filtered_errors.log
done

# 대용량 로그에서 특정 시간 범위 추출
awk '/2024-01-15 14:00/,/2024-01-15 15:00/' app.log

# JSON 로그 처리 (jq 활용)
cat app.log | jq -r 'select(.level == "error") | [.timestamp, .message] | @tsv'
```

---

## Chapter 8: 네트워크 기초

### 8.1 네트워크 설정 (Netplan - Ubuntu 24.04)

Ubuntu 24.04는 **Netplan**을 기본 네트워크 설정 도구로 사용한다.

```yaml
# /etc/netplan/01-config.yaml
network:
  version: 2
  renderer: networkd    # 서버: networkd, 데스크톱: NetworkManager
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 10.0.1.100/24
      routes:
        - to: default
          via: 10.0.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search:
          - example.com
      mtu: 9000            # Jumbo Frame (클라우드 VPC에서 사용)
    ens34:
      dhcp4: true

  bonds:                   # 네트워크 본딩 (HA)
    bond0:
      interfaces:
        - ens35
        - ens36
      parameters:
        mode: 802.3ad      # LACP
        mii-monitor-interval: 100
      addresses:
        - 10.0.2.100/24

  vlans:                   # VLAN
    vlan100:
      id: 100
      link: ens33
      addresses:
        - 10.100.0.10/24
```

```bash
# Netplan 적용
sudo netplan try            # 테스트 (120초 후 자동 롤백)
sudo netplan apply          # 영구 적용
sudo netplan generate       # 설정 생성 (디버그)

# 네트워크 인터페이스 조회
ip addr show                # IP 주소 (ifconfig 대체)
ip -4 addr show             # IPv4만
ip link show                # 링크 상태
ip route show               # 라우팅 테이블
ip neigh show               # ARP 테이블 (arp -a 대체)

# 인터페이스 관리
sudo ip link set ens33 up
sudo ip link set ens33 down
sudo ip addr add 10.0.1.200/24 dev ens33   # IP 추가
sudo ip route add 10.10.0.0/16 via 10.0.1.1  # 정적 라우트 추가
```

### 8.2 DNS 설정 및 진단

```bash
# systemd-resolved (Ubuntu 24.04 기본)
resolvectl status           # DNS 설정 상태
resolvectl dns              # 사용 중인 DNS 서버
resolvectl query example.com

# DNS 진단
dig example.com             # 상세 DNS 조회
dig +short example.com      # IP만
dig @8.8.8.8 example.com    # 특정 DNS 서버로 조회
dig example.com MX          # MX 레코드
dig example.com AAAA        # IPv6 레코드
dig -x 8.8.8.8              # 역방향 조회
nslookup example.com
host example.com

# /etc/hosts: 로컬 호스트명 해석 (DNS보다 우선)
# /etc/nsswitch.conf: 이름 해석 순서 설정
```

### 8.3 네트워크 진단 도구

```bash
# 연결 테스트
ping -c 4 8.8.8.8          # ICMP 에코 (4회)
ping6 ::1                   # IPv6
traceroute 8.8.8.8          # 경로 추적 (UDP)
tracepath 8.8.8.8            # MTU 발견 포함
mtr 8.8.8.8                 # 실시간 traceroute + ping

# 포트 및 연결 상태
ss -tuln                    # TCP/UDP 리스닝 포트 (netstat 대체)
ss -tunap                   # 프로세스 포함
ss -s                       # 소켓 통계 요약
ss state established         # 연결된 소켓만

# nc (netcat): 네트워크 스위스 아미 나이프
nc -zv host 80              # 포트 스캔
nc -l 8080                  # TCP 리스너
echo "test" | nc host 8080  # 데이터 전송

# curl: HTTP 디버깅
curl -v https://api.example.com           # 상세 출력
curl -o /dev/null -s -w "%{http_code}\n" URL  # 상태 코드만
curl -X POST -H "Content-Type: application/json" -d '{"key":"val"}' URL
curl --connect-timeout 5 --max-time 10 URL  # 타임아웃

# tcpdump: 패킷 캡처
sudo tcpdump -i ens33 port 80             # HTTP 트래픽
sudo tcpdump -i any host 10.0.1.100       # 특정 호스트
sudo tcpdump -i ens33 -w capture.pcap     # 파일로 저장
sudo tcpdump -i ens33 -n 'tcp port 443 and host 10.0.1.50'  # 복합 필터

# 대역폭 측정
iperf3 -s                   # 서버
iperf3 -c server_ip         # 클라이언트
```

---

# PART 3: 고급 개념 (Advanced)

---

## Chapter 9: 보안 심화 (Security)

### 9.1 SSH 보안 강화

```bash
# SSH 키 생성 (Ed25519 권장)
ssh-keygen -t ed25519 -C "user@company.com"
ssh-keygen -t rsa -b 4096 -C "user@company.com"  # RSA 사용 시

# 공개키 배포
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# SSH 에이전트
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l                  # 등록된 키 확인

# SSH Config (~/.ssh/config)
Host production
    HostName 10.0.1.100
    User deploy
    IdentityFile ~/.ssh/prod_ed25519
    Port 2222
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User admin
    IdentityFile ~/.ssh/bastion_key

# SSH 터널링
ssh -L 8080:db-server:5432 bastion   # 로컬 포워딩
ssh -R 9090:localhost:3000 server    # 리모트 포워딩
ssh -D 1080 server                   # SOCKS 프록시 (다이나믹)

# sshd 보안 설정 (/etc/ssh/sshd_config)
```
```
Port 2222                          # 기본 포트 변경
PermitRootLogin no                 # root 로그인 차단
PasswordAuthentication no          # 패스워드 인증 비활성화
PubkeyAuthentication yes           # 키 인증만 허용
MaxAuthTries 3                     # 인증 시도 제한
ClientAliveInterval 300            # 유휴 타임아웃
ClientAliveCountMax 2
AllowUsers deploy admin            # 허용 사용자 지정
AllowGroups ssh-users              # 허용 그룹
X11Forwarding no                   # X11 포워딩 비활성화
Protocol 2                         # SSHv2만 허용
LoginGraceTime 30                  # 로그인 유예 시간
Banner /etc/ssh/banner             # 경고 배너
```
```bash
# 적용
sudo systemctl restart sshd
sudo sshd -t  # 설정 문법 검증
```

### 9.2 방화벽 (UFW / nftables / iptables)

```bash
# UFW (Uncomplicated Firewall) - Ubuntu 기본
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp                     # SSH
sudo ufw allow from 10.0.0.0/8 to any port 5432  # PostgreSQL (내부망만)
sudo ufw allow 80,443/tcp                 # HTTP/HTTPS
sudo ufw deny from 203.0.113.0/24        # 특정 대역 차단
sudo ufw status verbose
sudo ufw status numbered
sudo ufw delete 3                         # 규칙 번호로 삭제
sudo ufw reload

# iptables (전통적 방화벽 - 세밀한 제어)
# 체인: INPUT(수신), OUTPUT(송신), FORWARD(전달)
sudo iptables -L -v -n                    # 규칙 조회
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -j DROP            # 나머지 차단 (마지막에!)

# Rate Limiting (DDoS 완화)
sudo iptables -A INPUT -p tcp --dport 22 -m connlimit --connlimit-above 3 -j REJECT
sudo iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT

# NAT (Network Address Translation)
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE  # SNAT
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.1.10:8080  # DNAT

# 규칙 영구 저장
sudo apt install iptables-persistent
sudo netfilter-persistent save

# nftables (iptables 후속, Ubuntu 24.04 권장)
sudo nft list ruleset
sudo nft add table inet filter
sudo nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
sudo nft add rule inet filter input tcp dport 22 accept
sudo nft add rule inet filter input ct state established,related accept
```

### 9.3 SELinux / AppArmor

Ubuntu 24.04는 **AppArmor**를 기본 MAC(Mandatory Access Control) 시스템으로 사용한다.

```bash
# AppArmor 상태
sudo aa-status
sudo apparmor_status

# 프로파일 모드
# enforce: 정책 위반 차단 + 로그
# complain: 정책 위반 로그만 (학습 모드)
# unconfined: 비활성

sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx    # enforce 모드
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx   # complain 모드
sudo aa-disable /etc/apparmor.d/usr.sbin.nginx    # 비활성화

# 프로파일 예시 (/etc/apparmor.d/usr.sbin.nginx)
# /usr/sbin/nginx {
#   /var/www/** r,
#   /var/log/nginx/** w,
#   /run/nginx.pid rw,
#   network inet tcp,
#   deny /etc/shadow r,
# }

# AppArmor 로그 확인
sudo journalctl | grep apparmor
sudo dmesg | grep apparmor
```

### 9.4 감사 및 침입 탐지

```bash
# auditd: 시스템 감사
sudo apt install auditd
sudo systemctl enable auditd

# 감사 규칙 추가 (/etc/audit/rules.d/audit.rules)
-w /etc/passwd -p wa -k identity_changes       # 파일 변경 감시
-w /etc/shadow -p wa -k identity_changes
-w /etc/sudoers -p wa -k sudoers_changes
-a always,exit -F arch=b64 -S execve -k commands  # 모든 명령 실행 기록
-w /var/log/ -p wa -k log_tampering            # 로그 변조 감시

# 감사 로그 검색
ausearch -k identity_changes
ausearch -ui 1000 --start today
aureport --summary

# fail2ban: 브루트포스 차단
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
```ini
# /etc/fail2ban/jail.local
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
banaction = ufw

[sshd]
enabled = true
port    = 2222
logpath = %(sshd_log)s
maxretry = 3
bantime = 24h
```
```bash
sudo systemctl enable fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip 192.168.1.100  # 차단 해제

# CIS Benchmark 기반 보안 점검
# AIDE (Advanced Intrusion Detection Environment)
sudo apt install aide
sudo aideinit                       # 초기 데이터베이스 생성
sudo aide --check                   # 무결성 검사
```

### 9.5 TLS/SSL 인증서 관리

```bash
# Let's Encrypt (Certbot)
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
sudo certbot renew --dry-run       # 갱신 테스트

# 자체 서명 인증서 (테스트/내부용)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout private.key -out certificate.crt

# 인증서 확인
openssl x509 -in certificate.crt -text -noout
openssl s_client -connect example.com:443 -servername example.com

# 인증서 만료일 확인
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | \
    openssl x509 -noout -dates
```

---

## Chapter 10: 스토리지 고급 관리

### 10.1 LVM (Logical Volume Manager)

```
Physical Volume (PV) → Volume Group (VG) → Logical Volume (LV) → Filesystem

디스크 /dev/sdb ──┐
                   ├──▶ VG "data_vg" ──▶ LV "app_lv"  ──▶ /app  (ext4)
디스크 /dev/sdc ──┘                  └──▶ LV "log_lv"  ──▶ /logs (xfs)
```

```bash
# PV 생성
sudo pvcreate /dev/sdb /dev/sdc
sudo pvs                           # PV 목록

# VG 생성
sudo vgcreate data_vg /dev/sdb /dev/sdc
sudo vgs                           # VG 목록
sudo vgextend data_vg /dev/sdd     # VG에 디스크 추가

# LV 생성
sudo lvcreate -n app_lv -L 50G data_vg       # 50GB
sudo lvcreate -n log_lv -l 100%FREE data_vg  # 남은 공간 전부
sudo lvs                           # LV 목록

# 파일시스템 생성 및 마운트
sudo mkfs.ext4 /dev/data_vg/app_lv
sudo mount /dev/data_vg/app_lv /app

# LV 확장 (무중단!)
sudo lvextend -L +20G /dev/data_vg/app_lv    # LV 크기 확장
sudo resize2fs /dev/data_vg/app_lv            # ext4 파일시스템 확장
# XFS의 경우: sudo xfs_growfs /mountpoint

# LV 스냅샷 (백업/롤백용)
sudo lvcreate -s -n app_snap -L 10G /dev/data_vg/app_lv
# 스냅샷에서 복원
sudo lvconvert --merge /dev/data_vg/app_snap

# LV 씬 프로비저닝 (over-provisioning)
sudo lvcreate --type thin-pool -n thin_pool -L 100G data_vg
sudo lvcreate --type thin -n thin_vol -V 500G --thinpool thin_pool data_vg
```

### 10.2 RAID

```bash
# mdadm으로 소프트웨어 RAID 구성
sudo apt install mdadm

# RAID 1 (미러링)
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# RAID 5 (패리티 분산)
sudo mdadm --create /dev/md1 --level=5 --raid-devices=3 /dev/sdd /dev/sde /dev/sdf

# RAID 상태
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# 디스크 교체 (RAID 1/5)
sudo mdadm /dev/md0 --fail /dev/sdb       # 장애 표시
sudo mdadm /dev/md0 --remove /dev/sdb     # 제거
sudo mdadm /dev/md0 --add /dev/sdg        # 새 디스크 추가 (자동 리빌드)

# 설정 저장
sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

---

## Chapter 11: 성능 튜닝과 모니터링

### 11.1 시스템 모니터링

```bash
# CPU
top / htop
mpstat -P ALL 1             # CPU별 통계 (1초 간격)
sar -u 1 10                 # 10초간 CPU 사용률

# 메모리
free -h                     # 메모리 사용량
vmstat 1                    # 가상 메모리 통계
sar -r 1 10                 # 메모리 통계

# 디스크 I/O
iostat -xz 1                # 디스크 I/O 통계
iotop                       # I/O별 프로세스 (top과 유사)
sar -d 1 10                 # 디스크 활동

# 네트워크
iftop                       # 실시간 네트워크 트래픽
nethogs                     # 프로세스별 네트워크 사용량
sar -n DEV 1 10             # 네트워크 인터페이스 통계
nload                       # 대역폭 모니터

# 종합 모니터링
dstat                       # CPU, 디스크, 네트워크, 페이징 종합
glances                     # 현대적 시스템 모니터
atop                        # 상세 시스템/프로세스 모니터

# /proc 기반 모니터링
cat /proc/loadavg           # 로드 평균 (1분, 5분, 15분)
cat /proc/meminfo           # 상세 메모리 정보
cat /proc/stat              # CPU 통계
cat /proc/diskstats         # 디스크 통계
```

### 11.2 커널 파라미터 튜닝 (sysctl)

```bash
# 현재 설정 조회
sysctl -a                                    # 모든 파라미터
sysctl vm.swappiness                         # 특정 파라미터

# 런타임 변경
sudo sysctl -w vm.swappiness=10
sudo sysctl -w net.core.somaxconn=65535

# 영구 설정 (/etc/sysctl.d/99-custom.conf)
```
```ini
# 메모리
vm.swappiness = 10                           # 스왑 사용 최소화
vm.dirty_ratio = 40                          # 더티 페이지 비율
vm.dirty_background_ratio = 10
vm.overcommit_memory = 1                     # Redis 등에 필요

# 네트워크 (고성능 서버)
net.core.somaxconn = 65535                   # 리스닝 소켓 백로그
net.core.netdev_max_backlog = 65536          # 네트워크 큐 크기
net.ipv4.tcp_max_syn_backlog = 65536         # SYN 백로그
net.ipv4.ip_local_port_range = 1024 65535    # 아웃바운드 포트 범위
net.ipv4.tcp_tw_reuse = 1                    # TIME_WAIT 소켓 재사용
net.ipv4.tcp_fin_timeout = 15                # FIN_WAIT2 타임아웃
net.ipv4.tcp_keepalive_time = 300            # TCP 킵얼라이브
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_max_tw_buckets = 1440000        # TIME_WAIT 소켓 수 제한
net.ipv4.tcp_window_scaling = 1              # TCP 윈도우 스케일링
net.core.rmem_max = 16777216                 # 수신 버퍼 최대
net.core.wmem_max = 16777216                 # 송신 버퍼 최대

# 보안
net.ipv4.conf.all.rp_filter = 1             # 역경로 필터링
net.ipv4.conf.all.accept_redirects = 0       # ICMP 리다이렉트 거부
net.ipv4.conf.all.send_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1     # 스머프 공격 방지
net.ipv4.conf.all.accept_source_route = 0    # 소스 라우팅 비활성화
net.ipv4.ip_forward = 1                      # IP 포워딩 (라우터/컨테이너용)
```
```bash
sudo sysctl -p /etc/sysctl.d/99-custom.conf  # 적용
```

### 11.3 리소스 제한 (ulimit / cgroup)

```bash
# ulimit: 프로세스별 리소스 제한
ulimit -a                    # 현재 제한 확인
ulimit -n 65535              # 파일 디스크립터 제한 (세션)

# 영구 설정 (/etc/security/limits.conf)
# <user>  <type>  <item>   <value>
*         soft    nofile   65535
*         hard    nofile   65535
*         soft    nproc    65535
root      soft    nofile   131072

# systemd 서비스 제한 (유닛 파일 내)
# LimitNOFILE=65536
# LimitNPROC=65536

# cgroups v2 (systemd 기반 리소스 제어)
# CPU 제한
sudo systemctl set-property myapp.service CPUQuota=200%   # 2코어 제한
# 메모리 제한
sudo systemctl set-property myapp.service MemoryMax=2G
sudo systemctl set-property myapp.service MemoryHigh=1.5G

# cgroup 직접 제어
cat /sys/fs/cgroup/system.slice/myapp.service/cpu.max
cat /sys/fs/cgroup/system.slice/myapp.service/memory.current
```

---

## Chapter 12: 컨테이너와 가상화

### 12.1 Linux 네임스페이스와 cgroups

컨테이너의 핵심 기술은 **네임스페이스(isolation)**와 **cgroups(resource limit)**다.

```bash
# 리눅스 네임스페이스 종류:
# - PID: 프로세스 ID 격리
# - NET: 네트워크 스택 격리
# - MNT: 마운트 포인트 격리
# - UTS: 호스트명/도메인명 격리
# - IPC: IPC 리소스 격리
# - USER: 사용자/그룹 ID 격리
# - CGROUP: cgroup 루트 격리

# 네임스페이스 확인
lsns                                 # 현재 네임스페이스 목록
ls -la /proc/self/ns/                # 현재 프로세스의 네임스페이스

# unshare: 네임스페이스 생성 (수동 컨테이너)
sudo unshare --pid --mount --fork --mount-proc bash
# 이 셸 안에서는 자체 PID 1을 가진 격리된 환경

# nsenter: 기존 네임스페이스 진입 (컨테이너 디버깅에 유용)
sudo nsenter -t $(docker inspect -f '{{.State.Pid}}' container_name) -n ip addr
```

### 12.2 Docker 핵심 (Cloud Engineer 필수)

```bash
# Docker 설치 (Ubuntu 24.04)
sudo apt install docker.io docker-compose-v2
sudo usermod -aG docker $USER
newgrp docker

# 이미지 관리
docker pull nginx:alpine
docker images
docker rmi image_id
docker image prune -a                # 미사용 이미지 전체 삭제

# 컨테이너 실행
docker run -d --name web \
    -p 80:80 \
    -v /data/web:/usr/share/nginx/html:ro \
    -e NGINX_HOST=example.com \
    --restart unless-stopped \
    --memory 512m --cpus 1.5 \
    --network mynet \
    nginx:alpine

# 컨테이너 관리
docker ps -a
docker logs -f --tail 100 web
docker exec -it web /bin/sh        # 컨테이너 내부 접속
docker stats                       # 리소스 사용량
docker inspect web                 # 상세 정보

# 네트워크
docker network create --driver bridge mynet
docker network ls
docker network inspect mynet

# 볼륨
docker volume create data_vol
docker volume ls
docker volume inspect data_vol

# Docker Compose
docker compose up -d
docker compose down
docker compose logs -f
docker compose ps

# 디스크 정리
docker system df                   # 디스크 사용량
docker system prune -a --volumes   # 전체 정리
```

---

## Chapter 13: 자동화와 설정 관리

### 13.1 cron과 systemd timer

```bash
# crontab
crontab -e                  # 편집
crontab -l                  # 조회
crontab -r                  # 삭제

# 형식: 분 시 일 월 요일 명령
# ┌───── 분 (0-59)
# │ ┌───── 시 (0-23)
# │ │ ┌───── 일 (1-31)
# │ │ │ ┌───── 월 (1-12)
# │ │ │ │ ┌───── 요일 (0-7, 0=7=일요일)
# * * * * * command
0 2 * * * /opt/backup/daily.sh               # 매일 02:00
*/5 * * * * /opt/monitoring/check.sh          # 5분마다
0 0 * * 0 /opt/cleanup/weekly.sh              # 매주 일요일 자정
30 6 1 * * /opt/reports/monthly.sh            # 매월 1일 06:30
0 */4 * * 1-5 /opt/sync/business_hours.sh     # 평일 4시간마다

# systemd timer (cron 대체, 더 강력)
```
```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup Timer

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true           # 놓친 실행 보상
RandomizedDelaySec=300     # 랜덤 지연 (동시 실행 방지)

[Install]
WantedBy=timers.target
```
```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
ExecStart=/opt/backup/daily.sh
User=backup
```
```bash
sudo systemctl enable --now backup.timer
systemctl list-timers --all
```

---

# PART 4: 글로벌 IT 기업 인터뷰 문제 & 해설

---

## Section A: 기초~중급 인터뷰 문제

### Q1. [Google/Amazon] Linux 부팅 프로세스를 설명하세요. 부팅이 안 될 때 어떻게 트러블슈팅하나요?

**모범 답변:**

부팅은 UEFI/BIOS → GRUB2 → Kernel+initramfs → systemd → default.target 순서로 진행됩니다.

트러블슈팅 접근법은 단계별로 다릅니다. GRUB 단계에서 문제가 생기면 GRUB rescue 모드에 진입하거나 Live USB로 부팅 후 `grub-install`과 `update-grub`으로 복구합니다. 커널 패닉이 발생하면 GRUB 메뉴에서 이전 커널을 선택하거나, `init=/bin/bash` 커널 파라미터로 단일 사용자 모드에 진입합니다.

systemd 단계에서는 `systemctl --failed`로 실패한 유닛을 확인하고, `journalctl -xb`로 현재 부팅 로그를 분석합니다. 파일시스템 문제라면 `fsck`를 실행하며, fstab 오류로 부팅 불가 시 `mount -o remount,rw /`로 루트를 읽기/쓰기로 재마운트한 뒤 `/etc/fstab`를 수정합니다.

실무에서는 `systemd-analyze blame`으로 부팅 지연 서비스를 식별하고, `dmesg`와 `journalctl -b -1`(이전 부팅 로그)을 비교 분석합니다.

---

### Q2. [Microsoft/Meta] 하드 링크와 심볼릭 링크의 차이를 설명하세요.

**모범 답변:**

하드 링크는 동일한 inode를 가리키는 또 다른 디렉토리 엔트리입니다. 원본 파일을 삭제해도 하드 링크를 통해 데이터에 접근 가능한데, inode의 링크 카운트가 0이 되어야 실제 데이터가 삭제되기 때문입니다. 제약으로는 같은 파일시스템 내에서만 생성 가능하고 디렉토리에는 만들 수 없습니다(순환 참조 방지).

심볼릭 링크는 대상 파일의 **경로명**을 저장하는 별도의 파일(별도 inode)입니다. 파일시스템을 넘어서 생성 가능하고, 디렉토리에도 사용할 수 있습니다. 단, 원본을 삭제하면 dangling symlink가 됩니다.

Cloud 환경에서 심볼릭 링크는 애플리케이션 배포 시 버전 전환(`/app/current -> /app/v2.1.0`)이나 설정 파일 관리에 많이 사용됩니다.

---

### Q3. [Amazon/Netflix] 프로세스와 스레드의 차이는? 좀비 프로세스란 무엇이며, 어떻게 처리하나요?

**모범 답변:**

프로세스는 독립된 메모리 공간(코드, 데이터, 힙, 스택)을 가진 실행 단위이고, 스레드는 같은 프로세스 내에서 메모리 공간을 공유하는 실행 단위입니다. Linux에서는 커널 레벨에서 둘 다 `task_struct`로 관리하며, 스레드는 `clone()` 시스템 콜로 생성되어 `CLONE_VM`, `CLONE_FS` 등의 플래그로 공유 범위를 지정합니다.

좀비 프로세스(Z 상태)는 자식 프로세스가 종료되었지만 부모 프로세스가 `wait()`/`waitpid()`를 호출하지 않아 exit status가 프로세스 테이블에 남아있는 상태입니다. 좀비 자체는 리소스를 거의 소비하지 않지만(PID 엔트리만 차지), 대량 발생 시 PID 고갈 문제가 생길 수 있습니다.

처리 방법: 부모 프로세스에 `SIGCHLD`를 보내거나(`kill -SIGCHLD <PPID>`), 부모 프로세스를 종료하면 좀비의 부모가 PID 1(systemd)로 변경되어 자동으로 회수(reaping)됩니다.

```bash
# 좀비 프로세스 찾기
ps aux | awk '$8=="Z" {print}'
```

---

### Q4. [Google/Apple] /proc 파일시스템은 무엇이며, 어떻게 활용하나요?

**모범 답변:**

`/proc`는 커널이 런타임에 생성하는 가상 파일시스템(pseudo-filesystem)으로, 디스크에 실제 존재하지 않습니다. 커널과 프로세스 상태를 사용자 공간에 파일 형태로 노출하며, 읽기/쓰기를 통해 커널과 상호작용합니다.

주요 활용 사례: `/proc/cpuinfo`로 CPU 정보 확인, `/proc/meminfo`로 메모리 상세(MemFree, Cached, Buffers 등) 확인, `/proc/loadavg`로 시스템 부하 확인, `/proc/[PID]/`로 특정 프로세스의 상태(status), 메모리 맵(maps), 열린 파일(fd), 환경 변수(environ), 실행 파일(exe) 등을 확인합니다.

`/proc/sys/` 하위는 sysctl로도 접근 가능한 커널 튜닝 파라미터입니다. 예를 들어 `echo 1 > /proc/sys/net/ipv4/ip_forward`로 IP 포워딩을 활성화할 수 있습니다. Cloud 환경에서는 컨테이너 디버깅, 성능 모니터링, 커널 파라미터 튜닝에 핵심적으로 사용됩니다.

---

### Q5. [Amazon/Microsoft] chmod 755, chmod 644의 의미를 설명하고, SUID의 보안 위험성을 설명하세요.

**모범 답변:**

755는 소유자 rwx(7), 그룹 r-x(5), 기타 r-x(5)로, 소유자만 쓰기 가능하고 나머지는 읽기/실행 가능합니다. 주로 디렉토리나 실행 스크립트에 사용합니다. 644는 소유자 rw-(6), 그룹 r--(4), 기타 r--(4)로, 소유자만 수정 가능하고 나머지는 읽기만 가능합니다. 일반 파일(설정 파일, HTML 등)에 사용합니다.

SUID(Set User ID) 비트가 설정된 실행 파일은 실행자가 아닌 **파일 소유자의 권한으로** 실행됩니다. `/usr/bin/passwd`가 대표적 예로, 일반 사용자가 실행해도 root 권한으로 `/etc/shadow`를 수정합니다.

보안 위험: SUID가 root 소유 파일에 설정되면, 해당 프로그램의 취약점을 통해 권한 상승(privilege escalation) 공격이 가능합니다. 보안 점검 시 `find / -perm -4000 -type f 2>/dev/null`로 SUID 파일을 정기 감사해야 합니다. 컨테이너 환경에서는 `--no-new-privileges` 옵션으로 SUID 실행을 차단합니다.

---

## Section B: 고급 인터뷰 문제

### Q6. [Google SRE] 서버의 로드 평균(Load Average)이 매우 높은데 CPU 사용률은 낮습니다. 원인은?

**모범 답변:**

Linux의 로드 평균은 **CPU를 사용 중이거나 사용 대기 중인 프로세스 + I/O 대기(D state) 프로세스**의 수를 반영합니다. 따라서 CPU 사용률이 낮은데 로드 평균이 높다면, I/O 대기 프로세스가 다수 존재한다는 의미입니다.

진단 절차:

1. `vmstat 1`에서 `b` 열(blocked/uninterruptible sleep)과 `wa`(I/O wait) 확인
2. `iostat -xz 1`에서 디스크별 `%util`, `await`, `avgqu-sz` 확인 → 특정 디스크에 병목이 있는지 식별
3. `iotop`으로 어떤 프로세스가 I/O를 유발하는지 확인
4. `ps aux | awk '$8~/D/'`로 D(Disk Sleep) 상태의 프로세스 확인

일반적 원인: NFS/네트워크 스토리지 응답 지연, 디스크 하드웨어 장애, 과도한 스왑 발생, 데이터베이스의 과도한 디스크 I/O, RAID 리빌드 진행 중 등입니다.

Cloud 환경에서는 EBS 볼륨의 IOPS 한도 초과, 네트워크 스토리지 레이턴시 등이 빈번한 원인입니다.

---

### Q7. [Netflix/Amazon] 서버의 메모리가 부족하다는 알림이 왔습니다. 어떻게 진단하나요?

**모범 답변:**

단계별 접근:

1. **현황 파악**: `free -h`로 total, used, free, available 확인. Linux는 사용 가능한 메모리를 캐시로 적극 활용하므로, `available` 열이 실제 사용 가능한 메모리입니다. `used`만 보고 판단하면 안 됩니다.

2. **프로세스별 분석**: `ps aux --sort=-%mem | head -20`으로 메모리 상위 프로세스 확인. `smem -t -k`로 PSS(Proportional Set Size) 기반 정확한 메모리 측정.

3. **상세 분석**:
   - `/proc/meminfo`에서 `Slab`, `PageTables`, `Buffers`, `Cached` 등 세부 항목 확인
   - `slabtop`으로 커널 슬랩 메모리 확인 (dentry, inode 캐시 등)
   - 특정 프로세스의 `/proc/[PID]/smaps_rollup`으로 RSS, PSS, Shared, Private 메모리 분석

4. **메모리 누수 확인**: 프로세스의 RSS가 시간에 따라 계속 증가하는지 모니터링. `valgrind`(개발), `pmap -x [PID]`로 메모리 맵 분석.

5. **OOM Killer 확인**: `dmesg | grep -i "oom\|killed"` 또는 `journalctl -k | grep -i oom`으로 OOM Killer가 프로세스를 종료했는지 확인.

6. **대응**: 불필요한 캐시 해제(`echo 3 > /proc/sys/vm/drop_caches` - 임시 조치), 메모리 누수 프로세스 재시작, `vm.overcommit_memory` 및 `vm.overcommit_ratio` 튜닝, swap 크기 조정.

---

### Q8. [Google/Meta] iptables의 테이블과 체인 구조를 설명하고, 패킷이 서버에 도착했을 때 처리 흐름을 설명하세요.

**모범 답변:**

iptables에는 5개의 테이블이 있습니다: **raw**(연결 추적 예외), **mangle**(패킷 헤더 수정), **nat**(주소 변환), **filter**(패킷 필터링, 기본), **security**(SELinux MAC).

각 테이블에는 체인이 있으며, 주요 체인은 **PREROUTING**, **INPUT**, **FORWARD**, **OUTPUT**, **POSTROUTING**입니다.

패킷 흐름 (서버로 들어오는 패킷):

```
네트워크 → PREROUTING(raw→mangle→nat) → 라우팅 결정
         ↓                                    ↓
    로컬 프로세스 대상                     포워딩 대상
         ↓                                    ↓
    INPUT(mangle→filter)               FORWARD(mangle→filter)
         ↓                                    ↓
    로컬 프로세스                       POSTROUTING(mangle→nat) → 네트워크
         ↓
    OUTPUT(raw→mangle→nat→filter)
         ↓
    POSTROUTING(mangle→nat) → 네트워크
```

Kubernetes 환경에서 kube-proxy는 nat 테이블의 PREROUTING, OUTPUT 체인에 Service ClusterIP → Pod IP 변환 규칙을 삽입합니다. 이것이 iptables 모드의 서비스 로드밸런싱 원리입니다.

---

### Q9. [Amazon/Cloudflare] TCP 3-way Handshake와 TIME_WAIT 상태를 설명하세요. TIME_WAIT가 대량 발생했을 때 어떻게 대응하나요?

**모범 답변:**

**3-way Handshake** (연결 수립):
1. 클라이언트 → SYN → 서버 (SEQ=x)
2. 서버 → SYN+ACK → 클라이언트 (SEQ=y, ACK=x+1)
3. 클라이언트 → ACK → 서버 (ACK=y+1)

**4-way Teardown** (연결 종료):
1. 클라이언트 → FIN → 서버
2. 서버 → ACK → 클라이언트
3. 서버 → FIN → 클라이언트
4. 클라이언트 → ACK → 서버 → **TIME_WAIT** (2*MSL, 보통 60초)

**TIME_WAIT의 목적**: 지연된 패킷이 새 연결에서 잘못 처리되는 것을 방지하고, 마지막 ACK가 유실됐을 때 재전송이 가능하도록 보장합니다.

**대량 TIME_WAIT 대응:**
```bash
# 현황 파악
ss -s
ss state time-wait | wc -l

# sysctl 튜닝
net.ipv4.tcp_tw_reuse = 1       # TIME_WAIT 소켓 재사용 (안전)
net.ipv4.tcp_fin_timeout = 15   # FIN_WAIT2 타임아웃 단축
net.ipv4.ip_local_port_range = 1024 65535  # 이용 가능 포트 확장
net.ipv4.tcp_max_tw_buckets = 1440000     # TIME_WAIT 상한
```

근본적으로는 HTTP Keep-Alive를 활성화하여 연결 재사용을 극대화하고, 커넥션 풀링을 적용합니다.

---

### Q10. [Google SRE/Amazon] 서버가 갑자기 느려졌습니다. 60초 안에 무엇을 확인하겠습니까?

**모범 답변:**

Netflix의 Brendan Gregg가 제안한 **USE Method** (Utilization, Saturation, Errors)와 **60-Second Checklist**를 따릅니다:

```bash
# 1. 전체 개요 (10초)
uptime                    # 로드 평균 추세 확인 (1/5/15분)
dmesg -T | tail           # 최근 커널 메시지 (하드웨어 오류, OOM 등)

# 2. CPU (10초)
vmstat 1 5                # r(run queue), b(blocked), wa(I/O wait)
mpstat -P ALL 1 3         # CPU별 사용률 (하나만 100%면 단일 스레드 병목)

# 3. 메모리 (10초)
free -h                   # available 확인
vmstat 1 3                # si/so(스왑 in/out) 확인 → 0이 아니면 메모리 부족

# 4. 디스크 I/O (10초)
iostat -xz 1 3            # %util, await(ms), avgqu-sz → 높으면 I/O 병목

# 5. 네트워크 (10초)
sar -n DEV 1 3            # 인터페이스별 패킷/초, 에러
ss -s                     # 소켓 상태 요약

# 6. 프로세스 (10초)
pidstat 1 3               # 프로세스별 CPU, 메모리, I/O
top -b -n 1               # 리소스 상위 프로세스
```

이후 특정 병목이 식별되면 `perf top`, `strace -p PID`, `tcpdump` 등으로 심층 분석합니다.

---

### Q11. [Meta/Stripe] Linux에서 OOM Killer는 어떻게 동작하나요? 특정 프로세스를 보호하려면?

**모범 답변:**

OOM(Out of Memory) Killer는 커널의 메모리 관리 서브시스템이 물리 메모리와 스왑을 모두 소진했을 때, 시스템 크래시를 방지하기 위해 프로세스를 강제 종료하는 메커니즘입니다.

**동작 원리**: 각 프로세스에 `oom_score`(0~1000)를 계산하며, 메모리 사용량이 많을수록, 실행 시간이 짧을수록, 우선순위가 낮을수록 점수가 높습니다. 점수가 가장 높은 프로세스부터 SIGKILL로 종료합니다.

**보호 방법:**
```bash
# oom_score_adj 조정 (-1000 ~ 1000)
echo -1000 > /proc/$(pidof critical_app)/oom_score_adj  # OOM 대상에서 제외
echo 1000 > /proc/$(pidof sacrificial_app)/oom_score_adj # 우선 종료 대상

# systemd 서비스에서 설정
# [Service]
# OOMScoreAdjust=-900

# 현재 점수 확인
cat /proc/$(pidof nginx)/oom_score
cat /proc/$(pidof nginx)/oom_score_adj

# OOM 이벤트 확인
dmesg | grep -i "oom\|killed process"
journalctl -k --grep="oom"
```

`vm.overcommit_memory`를 2로 설정하면 overcommit을 비활성화하여 OOM을 원천 방지할 수 있지만, 메모리 할당 실패가 빈번해질 수 있어 주의가 필요합니다.

---

### Q12. [Google/Amazon] 디스크가 df에서는 여유가 있는데, 파일 생성이 안 됩니다. 왜?

**모범 답변:**

두 가지 가능성이 있습니다:

**1. inode 고갈**: 파일시스템의 inode가 모두 사용된 경우입니다. `df -i`로 확인하면 IUse%가 100%일 수 있습니다. 수많은 작은 파일(세션 파일, 캐시 파일 등)이 원인이며, `find / -xdev -type d -exec sh -c 'echo "$(find "$1" -maxdepth 1 | wc -l) $1"' _ {} \; | sort -rn | head`로 파일이 많은 디렉토리를 찾습니다.

**2. 삭제된 파일이 열려있는 경우**: 프로세스가 파일을 열고 있는 상태에서 파일을 삭제하면 `ls`에는 보이지 않지만 디스크 공간은 해제되지 않습니다. `lsof +L1`로 삭제됐지만 열려있는 파일을 확인하고, 해당 프로세스를 재시작하면 공간이 회복됩니다.

```bash
# inode 확인
df -i

# 삭제된 파일이 열려있는 경우
lsof +L1 | grep deleted
# 프로세스 재시작 없이 파일 비우기 (로그 파일 등)
: > /proc/<PID>/fd/<FD>  # 또는 truncate
```

---

### Q13. [Cloudflare/Netflix] TCP SYN Flood 공격을 받았습니다. Linux에서 어떻게 완화하나요?

**모범 답변:**

SYN Flood는 대량의 SYN 패킷을 보내 서버의 SYN 백로그를 고갈시켜 정상 연결을 방해하는 DDoS 공격입니다.

**커널 레벨 완화:**
```bash
# SYN Cookies 활성화 (가장 중요! 백로그 고갈 방지)
sysctl -w net.ipv4.tcp_syncookies=1

# SYN 백로그 크기 증가
sysctl -w net.ipv4.tcp_max_syn_backlog=65536
sysctl -w net.core.somaxconn=65535

# SYN-ACK 재시도 횟수 감소
sysctl -w net.ipv4.tcp_synack_retries=2

# 연결 타임아웃 단축
sysctl -w net.ipv4.tcp_abort_on_overflow=1
```

**iptables/nftables 레벨:**
```bash
# SYN 패킷 rate limiting
iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# 연결 수 제한 (IP당)
iptables -A INPUT -p tcp --dport 80 --syn -m connlimit --connlimit-above 20 -j DROP

# 비정상 패킷 차단
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
```

**모니터링:**
```bash
ss -s                         # SYN-RECV 상태 소켓 수
netstat -n | awk '/SYN_RECV/ {print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn
```

근본적 방어는 Cloud 레벨(AWS Shield, Cloudflare 등)의 DDoS 완화 서비스를 사용하는 것입니다.

---

### Q14. [Amazon/Google] /etc/fstab에서 실수로 잘못된 항목을 추가하여 부팅이 안 됩니다. 어떻게 복구하나요?

**모범 답변:**

**방법 1 - Recovery Mode (GRUB)**:
1. GRUB 메뉴에서 "Advanced options" → "Recovery mode" 선택
2. root shell 선택
3. `mount -o remount,rw /`로 루트를 쓰기 가능하게 재마운트
4. `vim /etc/fstab`로 잘못된 항목 수정 또는 주석 처리
5. `reboot`

**방법 2 - GRUB 커널 파라미터 수정**:
1. GRUB에서 커널 라인 끝에 `init=/bin/bash` 또는 `systemd.unit=emergency.target` 추가
2. `mount -o remount,rw /`
3. fstab 수정
4. `exec /sbin/init` 또는 `reboot -f`

**방법 3 - Live USB**:
1. Live USB로 부팅
2. `mount /dev/sda2 /mnt` (루트 파티션 마운트)
3. `vim /mnt/etc/fstab` 수정
4. 재부팅

**예방법**: fstab 수정 전 `sudo cp /etc/fstab /etc/fstab.bak` 백업. 새 항목 추가 후 `mount -a`로 반드시 테스트. `nofail` 옵션을 사용하면 마운트 실패 시에도 부팅이 계속됩니다.

---

### Q15. [Netflix/Stripe] strace와 ltrace는 무엇이며, 프로덕션에서 어떻게 활용하나요?

**모범 답변:**

`strace`는 프로세스의 **시스템 콜**을 추적하고, `ltrace`는 **라이브러리 함수 호출**을 추적합니다.

```bash
# strace 기본
strace -p PID                        # 실행 중인 프로세스 추적
strace -f -e trace=network command   # 네트워크 syscall만 추적 (fork 포함)
strace -e trace=file command         # 파일 관련 syscall만
strace -c command                    # syscall 통계 요약
strace -T -e trace=read,write -p PID # 각 syscall 소요 시간

# 실전 예시: 프로세스가 어떤 파일을 열려다 실패하는지
strace -e trace=open,openat -p PID 2>&1 | grep ENOENT

# 실전 예시: DNS 해석 지연 추적
strace -e trace=network -T curl https://api.example.com
```

**프로덕션 주의사항**: strace는 프로세스를 ptrace로 attach하므로 성능 오버헤드가 크고(2~100배 느려짐), 대상 프로세스가 잠시 중단될 수 있습니다. 프로덕션에서는 BPF 기반의 `bpftrace`나 `perf`를 우선 사용하고, strace는 단시간 사용하거나 `-c`(통계 모드)로 제한합니다.

---

### Q16. [Google/Microsoft] Linux의 I/O 스케줄러를 설명하고, 워크로드에 따른 최적 선택은?

**모범 답변:**

Linux 커널(5.x+)의 주요 I/O 스케줄러:

**mq-deadline**: 요청에 데드라인을 부여하여 기아(starvation)를 방지합니다. 읽기 우선, 데이터베이스 워크로드에 적합합니다. HDD와 SSD 모두에서 안정적입니다.

**bfq (Budget Fair Queueing)**: 프로세스별 I/O 대역폭을 공정하게 배분합니다. 데스크톱이나 인터랙티브 워크로드에서 체감 성능이 좋습니다. 오버헤드가 있어 고처리량 서버에는 부적합할 수 있습니다.

**none (noop)**: 스케줄링을 하지 않고 FIFO로 전달합니다. NVMe SSD처럼 자체 스케줄링이 있는 장치에 최적입니다. Cloud VM의 가상 디스크에도 적합합니다(호스트 레벨에서 이미 스케줄링).

```bash
# 현재 스케줄러 확인
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# 변경 (런타임)
echo none > /sys/block/nvme0n1/queue/scheduler

# 영구 변경 (udev rule)
# /etc/udev/rules.d/60-iosched.rules
# ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"
# ACTION=="add|change", KERNEL=="sd*", ATTR{queue/scheduler}="mq-deadline"
```

**권장**: NVMe SSD → none, SATA SSD → mq-deadline 또는 none, HDD → mq-deadline, Cloud VM → none.

---

### Q17. [Amazon/Google] 대규모 서버에서 SSH 키 관리를 어떻게 하시겠습니까?

**모범 답변:**

규모에 따른 전략:

**소규모 (수십 대)**: `ssh-copy-id`로 배포하고, Ansible의 `authorized_key` 모듈로 자동화합니다. 키 로테이션 정책을 수립하고 주기적으로 실행합니다.

**중규모 (수백 대)**: LDAP/FreeIPA 기반 중앙 인증과 결합하거나, SSH Certificate Authority를 구축합니다. CA 키로 사용자 인증서를 발급하면, 서버에는 CA 공개키만 등록하면 되어 개별 키 배포가 필요 없습니다.

```bash
# SSH CA 구축
ssh-keygen -f ca_key -t ed25519          # CA 키 생성
ssh-keygen -s ca_key -I user_id -n user -V +52w user_key.pub  # 인증서 발급

# 서버 설정 (sshd_config)
# TrustedUserCAKeys /etc/ssh/ca_key.pub
```

**대규모/Cloud (수천 대 이상)**: HashiCorp Vault의 SSH Secrets Engine으로 동적 SSH 인증서를 발급합니다(TTL 기반 자동 만료). AWS의 경우 SSM Session Manager, GCP의 OS Login을 사용하여 IAM과 통합합니다. 이 경우 SSH 키 자체를 관리할 필요가 없어집니다. 모든 접근은 감사 로그에 기록됩니다.

---

### Q18. [Netflix/Meta] 커널 패닉(Kernel Panic)이 발생했습니다. 어떻게 디버깅하나요?

**모범 답변:**

**즉시 확인:**
```bash
# 시리얼 콘솔/IPMI/Cloud 콘솔에서 패닉 메시지 확인
# 재부팅 후 이전 부팅 로그 확인
journalctl -b -1 -p err     # 이전 부팅의 에러 로그
dmesg | grep -i "panic\|oops\|bug\|error"
cat /var/crash/              # 크래시 덤프 (kdump 설정 시)
```

**kdump 설정 (사전 준비):**
```bash
sudo apt install linux-crashdump kdump-tools
# /etc/default/kdump-tools에서 USE_KDUMP=1
# GRUB_CMDLINE_LINUX에 "crashkernel=256M" 추가
sudo update-grub && reboot

# 크래시 덤프 분석
sudo apt install crash linux-image-$(uname -r)-dbgsym
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/dump
# crash> bt       # 백트레이스
# crash> log      # 커널 로그
# crash> ps       # 프로세스 목록
# crash> vm       # 메모리 정보
```

**일반적 원인**: 하드웨어 오류(메모리 불량 → `memtest86+`), 커널 모듈 버그(최근 설치/업데이트한 모듈), 드라이버 호환성 문제, 파일시스템 손상, 메모리 부족(OOM에서 회복 실패).

Cloud 환경에서는 인스턴스의 시스템 로그(AWS: `get-console-output`, GCP: `getSerialPortOutput`)를 확인합니다.

---

### Q19. [Google/Amazon] 컨테이너 내에서 네트워크 문제를 디버깅하는 방법은?

**모범 답변:**

컨테이너는 경량 이미지를 사용하므로 디버깅 도구가 없는 경우가 많습니다. 이때 호스트에서 컨테이너 네임스페이스에 진입하여 디버깅합니다.

```bash
# 1. 컨테이너의 PID 확인
PID=$(docker inspect -f '{{.State.Pid}}' container_name)

# 2. 네트워크 네임스페이스에 진입
sudo nsenter -t $PID -n ip addr          # IP 확인
sudo nsenter -t $PID -n ss -tuln         # 리스닝 포트
sudo nsenter -t $PID -n ping 8.8.8.8     # 외부 연결
sudo nsenter -t $PID -n cat /etc/resolv.conf  # DNS 설정

# 3. 호스트에서 컨테이너 트래픽 캡처
# 컨테이너의 veth 인터페이스 찾기
VETH=$(ip link | grep -A1 "$(sudo nsenter -t $PID -n ip link show eth0 | head -1 | awk '{print $2}' | cut -d@ -f2 | tr -d ':')" | head -1 | awk '{print $2}' | cut -d@ -f1)
sudo tcpdump -i $VETH -n

# 4. Docker 네트워크 진단
docker network inspect bridge
docker exec container_name cat /etc/hosts

# 5. iptables 규칙 확인 (Docker는 nat, filter 테이블에 규칙 추가)
sudo iptables -t nat -L -n -v | grep DOCKER
```

**Kubernetes 환경:**
```bash
kubectl exec -it pod-name -- /bin/sh     # Pod 내부 접속
kubectl run debug --image=nicolaka/netshoot -it --rm  # 디버그 Pod
kubectl logs pod-name -c container-name   # 로그 확인
kubectl describe pod pod-name             # 이벤트 확인
```

---

### Q20. [Stripe/Cloudflare] Linux에서 파일 디스크립터(File Descriptor) 제한을 설명하고, 대규모 트래픽 서버에서 어떻게 튜닝하나요?

**모범 답변:**

파일 디스크립터(FD)는 열린 파일, 소켓, 파이프 등을 참조하는 정수형 핸들입니다. 각 TCP 연결은 하나의 FD를 소비하므로, 대규모 동시 연결을 처리하는 서버에서는 FD 제한이 병목이 됩니다.

**제한 확인:**
```bash
# 시스템 전체 제한
cat /proc/sys/fs/file-max            # 시스템 최대 FD 수
cat /proc/sys/fs/file-nr             # 현재 사용 중 / 할당 / 최대

# 프로세스별 제한
ulimit -n                            # 현재 셸의 soft limit
ulimit -Hn                           # hard limit
cat /proc/$(pidof nginx)/limits      # 특정 프로세스의 제한
ls /proc/$(pidof nginx)/fd | wc -l   # 현재 사용 중인 FD 수
```

**튜닝:**
```bash
# 1. 시스템 전체
echo "fs.file-max = 2097152" >> /etc/sysctl.d/99-custom.conf
sysctl -p

# 2. 사용자별 (/etc/security/limits.conf)
nginx    soft    nofile    131072
nginx    hard    nofile    131072

# 3. systemd 서비스별
# [Service]
# LimitNOFILE=131072

# 4. PAM 설정 확인 (/etc/pam.d/common-session)
# session required pam_limits.so
```

C10K/C100K 문제를 해결하려면 FD 제한 외에도 `net.core.somaxconn`, `net.ipv4.tcp_max_syn_backlog`, 이벤트 드리븐 아키텍처(epoll) 등을 함께 최적화해야 합니다.

---

## Section C: 시나리오 기반 심화 문제

### Q21. [종합] 다음 상황을 단계별로 해결하세요:
**"프로덕션 웹 서버에서 502 Bad Gateway가 간헐적으로 발생합니다."**

**체계적 접근:**

```bash
# 1단계: 현상 파악
curl -v https://your-site.com          # 응답 헤더 확인
tail -f /var/log/nginx/error.log       # Nginx 에러 로그 실시간

# 2단계: 업스트림(백엔드) 확인
# 502는 프록시/로드밸런서가 백엔드로부터 유효한 응답을 못 받은 것
systemctl status myapp                  # 백엔드 서비스 상태
ss -tuln | grep 8080                   # 백엔드 포트 리스닝 확인
curl localhost:8080/health             # 직접 백엔드 호출

# 3단계: 리소스 확인
free -h                                # 메모리 (OOM으로 백엔드 종료?)
dmesg | grep -i oom                    # OOM Killer 확인
df -h                                  # 디스크 풀 (로그 파일 등)
ulimit -n                              # FD 제한 도달?
ss -s                                  # 소켓 상태 (TIME_WAIT 폭증?)

# 4단계: 네트워크 레벨
ss -tuln | grep -E "8080|80|443"       # 포트 리스닝
sudo tcpdump -i lo port 8080 -c 20    # 루프백 트래픽 캡처
# upstream connection refused / reset 확인

# 5단계: 애플리케이션 로그
journalctl -u myapp --since "1 hour ago" -p err
# 백엔드 프로세스 크래시, 타임아웃, 연결 풀 고갈 확인

# 6단계: Nginx 설정 확인
nginx -t                                # 설정 문법 검증
# proxy_connect_timeout, proxy_read_timeout 값 확인
# upstream 블록의 서버 수와 상태 확인
```

**일반적 원인과 해결:**
- 백엔드 프로세스 크래시 → 재시작 + 원인 분석 (메모리 누수, 예외 등)
- 백엔드 과부하/타임아웃 → 워커 수 증가, 타임아웃 조정
- FD/소켓 고갈 → ulimit 튜닝
- 업스트림 커넥션 풀 고갈 → keepalive 설정, 커넥션 풀 크기 조정

---

### Q22. [종합] 프로덕션 서버 보안 하드닝 체크리스트를 작성하세요.

```bash
# ─── OS 레벨 ───
[ ] 불필요한 패키지/서비스 제거 (sudo apt remove --purge telnet ftp)
[ ] 자동 보안 업데이트 활성화 (unattended-upgrades)
[ ] 커널 파라미터 보안 설정 (sysctl: rp_filter, accept_redirects 등)
[ ] root 직접 로그인 비활성화
[ ] sudo 설정 최소 권한 원칙 (/etc/sudoers)
[ ] 불필요한 SUID/SGID 파일 제거

# ─── 인증/접근 ───
[ ] SSH: 키 인증만 허용, root 로그인 차단, 포트 변경
[ ] fail2ban 설정
[ ] PAM 설정 (비밀번호 복잡도, 계정 잠금)
[ ] 불필요한 사용자 계정 비활성화/삭제

# ─── 네트워크 ───
[ ] 방화벽 기본 정책: deny all incoming
[ ] 필요한 포트만 허용 (최소 권한)
[ ] 불필요한 네트워크 서비스 비활성화
[ ] NTP 동기화 설정 (chrony/systemd-timesyncd)

# ─── 파일시스템 ───
[ ] /tmp를 noexec,nosuid로 마운트
[ ] 중요 디렉토리 권한 검증 (/etc/shadow 640 등)
[ ] AIDE/Tripwire 무결성 모니터링

# ─── 로깅/감사 ───
[ ] auditd 설정 및 규칙 배포
[ ] 중앙 로그 수집 (rsyslog → ELK/Splunk)
[ ] 로그 로테이션 설정 (logrotate)
[ ] NTP 동기화 (로그 타임스탬프 정확성)

# ─── 컨테이너 (해당 시) ───
[ ] 비root 사용자로 실행
[ ] 읽기 전용 파일시스템 (--read-only)
[ ] 불필요한 capability 제거 (--cap-drop ALL)
[ ] 리소스 제한 설정 (--memory, --cpus)
[ ] 이미지 취약점 스캔 (Trivy, Grype)
```

---

# 부록: 자주 사용하는 명령어 치트시트

```bash
# ─── 파일/디렉토리 ───
find / -name "*.conf" -mtime -7 -type f     # 7일 내 수정된 .conf 파일
find / -size +100M -type f -exec ls -lh {} \;  # 100MB 이상 파일
find / -perm -4000 -type f 2>/dev/null      # SUID 파일 탐색
locate filename                              # 빠른 파일 검색 (updatedb 필요)
tar czf backup.tar.gz /data                  # 압축
tar xzf backup.tar.gz -C /restore           # 해제
rsync -avz --progress /src/ user@server:/dst/  # 동기화

# ─── 시스템 정보 ───
hostnamectl                    # 호스트명, OS, 커널 정보
lscpu                          # CPU 정보
lsmem                          # 메모리 정보
lspci                          # PCI 장치
lsusb                          # USB 장치
dmidecode                      # BIOS/하드웨어 정보

# ─── 시간/날짜 ───
timedatectl                    # 시간대, NTP 동기화 상태
timedatectl set-timezone Asia/Seoul
chronyc tracking               # NTP 동기화 상세

# ─── 로그 ───
journalctl -f                  # 실시간 시스템 로그
journalctl --disk-usage        # 저널 디스크 사용량
journalctl --vacuum-size=500M  # 저널 크기 제한
last                           # 로그인 기록
lastb                          # 로그인 실패 기록
who                            # 현재 로그인 사용자
w                              # 현재 로그인 사용자 + 활동
```

---

*이 가이드는 Ubuntu 24.04 LTS 기준으로 작성되었으며, Cloud Engineer/SRE 직무의 글로벌 기업 인터뷰에 자주 출제되는 주제를 중심으로 구성했습니다. 모든 명령어는 실제 환경에서 테스트하며 학습하는 것을 권장합니다.*
