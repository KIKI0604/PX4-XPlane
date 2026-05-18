# PX4 - X-Plane 12 - QGroundControl 연동 설치 매뉴얼

이 매뉴얼은 다음 환경에서 **X-Plane 12 + PX4 SITL + QGroundControl(QGC)** 를 함께 사용하는 것을 목표로 한다.

```text
Windows + WSL2 Ubuntu + PX4-Autopilot-Me + X-Plane 12 + px4xplane plugin + mavlink-router + QGroundControl
```

본 매뉴얼의 최종 운용 기준은 다음과 같다.

```text
X-Plane 12 실행
→ mavlink-routerd 실행
→ PX4 SITL 실행
→ QGroundControl 연결 확인
```

---

## 0. 전체 구성 개념

### 0-1. 연결 구조

PX4와 X-Plane 플러그인의 시뮬레이터 연결은 TCP 4560 기반이다.

```text
PX4 simulator_mavlink → TCP 4560 → X-Plane px4xplane plugin
```

하지만 본 매뉴얼은 QGroundControl까지 함께 사용하는 것을 전제로 한다.

따라서 실제 운용 구조는 다음과 같다.

```text
X-Plane 12
  └─ px4xplane plugin
       ↑
       │ TCP 4560
       ↓
PX4 SITL in WSL Ubuntu
       ↓
 MAVLink UDP
       ↓
mavlink-routerd
       ↓
QGroundControl / additional MAVLink endpoints
```

정리하면 다음과 같다.

```text
PX4 ↔ X-Plane 연결 자체 = TCP 4560 기반
PX4 + X-Plane + QGC 동시 운용 = mavlink-router 사용
```

본 매뉴얼에서는 QGC 사용을 전제로 하므로, **mavlink-routerd 실행을 최종 운용 절차의 필수 단계로 취급한다.**

---

### 0-2. 터미널 및 창 구분

혼동을 줄이기 위해 본 매뉴얼에서는 실행 창을 다음처럼 구분한다.

| 구분 | 실행 위치 | 용도 |
|---|---|---|
| 창 1 | Windows | X-Plane 12 실행 |
| 창 2 | WSL Ubuntu 터미널 1 | mavlink-routerd 실행 |
| 창 3 | WSL Ubuntu 터미널 2 | PX4 SITL 실행 |
| 창 4 | Windows | QGroundControl 실행 |
| 터미널 W | Windows PowerShell | WSL IP 확인, X-Plane 경로 확인 |
| PX4 콘솔 | PX4 실행 후 `pxh>` | uORB, listener 확인 |

중요:

```text
- mavlink-routerd 실행 창과 PX4 SITL 실행 창은 서로 다른 Ubuntu 터미널이어야 한다.
- mavlink-routerd 터미널은 실행 후 계속 켜둔다.
- PX4 SITL 터미널도 실행 후 계속 켜둔다.
- X-Plane과 QGroundControl도 동시에 실행 상태를 유지한다.
```

---

## 1. WSL 설치 및 설정

### 1-1. WSL2 Ubuntu 설치

Windows PowerShell에서 실행한다.

**[터미널 W = Windows PowerShell]**

```powershell
wsl --install -d Ubuntu-22.04
```

설치된 WSL 배포판과 WSL 버전을 확인한다.

```powershell
wsl -l -v
```

정상 예시:

```text
  NAME            STATE           VERSION
* Ubuntu-22.04    Stopped         2
```

Ubuntu 22.04에 접속한다.

```powershell
wsl -d Ubuntu-22.04
```

---

### 1-2. Ubuntu 기본 패키지 업데이트

WSL Ubuntu 안에서 실행한다.

**[창 2 또는 창 3 = WSL Ubuntu]**

```bash
sudo apt update
sudo apt upgrade -y
```

PX4 기본 작업에 필요한 도구를 설치한다.

```bash
sudo apt install git curl wget unzip python3 python3-pip -y
```

---

## 2. PX4 기본 설치

이 단계는 PX4 기본 SITL 동작 확인용이다.

X-Plane 연동용 실제 실행은 이후 `PX4-Autopilot-Me`를 사용한다.

### 2-1. PX4 기본 저장소 클론

WSL Ubuntu에서 실행한다.

```bash
cd ~
git clone --recursive --branch v1.14.0 https://github.com/PX4/PX4-Autopilot.git
```

PX4 폴더로 이동한다.

```bash
cd ~/PX4-Autopilot
```

서브모듈을 동기화한다.

```bash
git submodule update --init --recursive
```

현재 PX4 버전을 확인한다.

```bash
git describe --tags
```

정상 예시:

```text
v1.14.0
```

---

### 2-2. PX4 Ubuntu 개발환경 설치

```bash
bash ./Tools/setup/ubuntu.sh
```

설치 후 WSL을 완전히 재시작한다.

Windows PowerShell에서 실행한다.

**[터미널 W = Windows PowerShell]**

```powershell
wsl --shutdown
```

다시 Ubuntu에 접속한다.

```powershell
wsl -d Ubuntu-22.04
```

---

### 2-3. PX4 기본 SITL 실행 확인

WSL Ubuntu에서 실행한다.

```bash
cd ~/PX4-Autopilot
make px4_sitl none
```

정상 예시:

```text
pxh>
```

PX4 콘솔에서 종료하려면 다음을 입력한다.

```text
shutdown
```

---

## 3. mavlink-router 설치 및 실행 준비

### 3-1. mavlink-router 역할

본 매뉴얼은 **X-Plane + PX4 SITL + QGroundControl** 동시 사용을 기준으로 한다.

따라서 이 매뉴얼에서는 `mavlink-routerd`를 최종 운용 절차에 포함한다.

```text
PX4 ↔ X-Plane TCP 4560 연결 자체에는 mavlink-router가 직접 관여하지 않는다.
하지만 QGroundControl까지 함께 연결하는 본 매뉴얼의 최종 구성에서는 mavlink-routerd를 사용한다.
```

---

### 3-2. mavlink-router 설치

WSL Ubuntu에서 실행한다.

**[창 2 = Ubuntu 터미널 1]**

```bash
sudo apt update
sudo apt install mavlink-router -y
```

설치 여부를 확인한다.

```bash
which mavlink-routerd
```

정상 출력 예시:

```text
/usr/bin/mavlink-routerd
```

또는 버전을 확인한다.

```bash
mavlink-routerd --version
```

---

### 3-3. 설치 실패 시

다음 오류가 발생할 수 있다.

```text
E: Unable to locate package mavlink-router
```

이 경우 현재 Ubuntu 저장소에서 바로 설치되지 않는 상태이다.

이때 판단은 다음과 같다.

```text
- PX4 ↔ X-Plane TCP 4560 연결만 확인하는 것은 가능할 수 있다.
- 하지만 본 매뉴얼은 QGC 사용이 포함된 최종 운용을 목표로 한다.
- 따라서 mavlink-router 설치 문제를 해결한 뒤 최종 운용 절차를 진행하는 것을 권장한다.
```

대안은 다음 중 하나다.

```text
1. Ubuntu 패키지 저장소 갱신 후 재시도
2. mavlink-router를 소스 빌드로 설치
3. QGC 연결 없이 PX4-XPlane TCP 연결만 먼저 확인
```

우선 재시도:

```bash
sudo apt update
sudo apt install mavlink-router -y
```

그래도 실패하면, 이 매뉴얼의 최종 QGC 운용 절차에서는 `mavlink-routerd`가 필요하므로 설치 문제를 먼저 해결해야 한다.

---

### 3-4. mavlink-router 프로세스 확인 및 종료

기존에 실행 중인 PX4 또는 mavlink-router 프로세스가 남아 있으면 포트 충돌이 발생할 수 있다.

WSL Ubuntu에서 확인한다.

**[창 2 = Ubuntu 터미널 1]**

```bash
pgrep -a px4
pgrep -a mavlink-routerd
```

실행 중인 프로세스를 종료하려면 다음을 사용한다.

```bash
pkill -f px4
pkill -f mavlink-routerd
```

다시 확인한다.

```bash
pgrep -a px4
pgrep -a mavlink-routerd
```

필요 시 PX4를 강제 종료한다.

```bash
pkill -9 -f px4
```

주의:

```text
pkill -9는 강제 종료이므로 일반 종료가 되지 않을 때만 사용한다.
```

---

## 4. X-Plane 연동용 PX4 설치

X-Plane 연동용 PX4 fork를 클론한다.

WSL Ubuntu에서 실행한다.

**[창 3 = Ubuntu 터미널 2 또는 일반 WSL Ubuntu]**

```bash
cd ~
git clone --recursive -b px4xplane-sitl https://github.com/alireza787b/PX4-Autopilot-Me.git PX4-Autopilot-Me
```

X-Plane fork 폴더로 이동하고 서브모듈을 동기화한다.

```bash
cd ~/PX4-Autopilot-Me
git submodule update --init --recursive
```

현재 브랜치를 확인한다.

```bash
git branch --show-current
```

정상 출력:

```text
px4xplane-sitl
```

작업 트리 상태를 확인한다.

```bash
git status
```

정상 예시:

```text
nothing to commit, working tree clean
```

---

### 4-1. 브랜치가 다른 경우

다음처럼 출력되면 X-Plane 연동용 브랜치가 아닐 수 있다.

```text
main
```

또는

```text
master
```

해결:

```bash
cd ~/PX4-Autopilot-Me
git fetch --all
git checkout px4xplane-sitl
git submodule update --init --recursive
```

다시 확인한다.

```bash
git branch --show-current
```

정상 출력:

```text
px4xplane-sitl
```

---

### 4-2. 폴더가 이미 있는 경우

다음 오류가 발생할 수 있다.

```text
fatal: destination path 'PX4-Autopilot-Me' already exists and is not an empty directory.
```

기존 작업을 보존하려면 이름을 바꾼다.

```bash
mv ~/PX4-Autopilot-Me ~/PX4-Autopilot-Me_backup
```

다시 클론한다.

```bash
cd ~
git clone --recursive -b px4xplane-sitl https://github.com/alireza787b/PX4-Autopilot-Me.git PX4-Autopilot-Me
```

---

### 4-3. 서브모듈 오류가 발생한 경우

다음처럼 서브모듈 오류가 발생할 수 있다.

```text
fatal: clone of ... failed
```

해결:

```bash
cd ~/PX4-Autopilot-Me
git submodule sync --recursive
git submodule update --init --recursive
```

---

## 5. X-Plane 12 및 px4xplane 플러그인 설치

### 5-1. X-Plane 12 설치 경로 예시

X-Plane 12는 일반 설치판 또는 Steam 설치판 모두 사용할 수 있다.

Steam 기본 설치 경로 예시는 다음과 같다.

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12
```

D드라이브 Steam Library에 설치한 경우 예시는 다음과 같다.

```text
D:\SteamLibrary\steamapps\common\X-Plane 12
```

또는 사용자가 직접 지정한 경로일 수 있다.

```text
D:\X-Plane 12
D:\Games\SteamLibrary\steamapps\common\X-Plane 12
```

X-Plane 12 설치 폴더는 보통 아래 구조를 가진다.

```text
X-Plane 12
├─ Aircraft
├─ Resources
│  └─ plugins
├─ X-Plane.exe
└─ Log.txt
```

---

### 5-2. X-Plane 설치 폴더 확인

Windows PowerShell에서 실행한다.

**[터미널 W = Windows PowerShell]**

C드라이브 기본 경로 확인:

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12"
```

정상 출력:

```text
True
```

D드라이브 설치 예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12"
```

정상 출력:

```text
True
```

정상 출력이 아니라면 다음처럼 나온다.

```text
False
```

가능한 원인:

```text
- X-Plane 12가 설치되지 않음
- Steam 설치 경로가 기본 경로가 아님
- D드라이브 또는 다른 드라이브에 설치됨
- 데모 버전 또는 별도 경로에 설치됨
- 폴더명이 다름
```

해결 방법:

```text
Steam → Library → X-Plane 12 → Manage → Browse local files
```

열린 폴더 경로를 복사한 뒤 그 경로로 다시 `Test-Path`를 수행한다.

예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12"
```

---

### 5-3. X-Plane 실행 파일 확인

C드라이브 예시:

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\X-Plane.exe"
```

D드라이브 예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\X-Plane.exe"
```

정상 출력:

```text
True
```

`False`이면 X-Plane 설치 경로가 잘못된 것이다.

---

### 5-4. X-Plane 플러그인 폴더 확인

C드라이브 예시:

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins"
```

D드라이브 예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins"
```

정상 출력:

```text
True
```

정상 출력이 아니라면 다음처럼 나온다.

```text
False
```

가능한 원인:

```text
- X-Plane 설치 경로가 다름
- Resources 폴더 구조가 손상됨
- plugins 폴더가 삭제됨
```

`X-Plane.exe`는 있는데 `Resources\plugins`가 없으면 폴더를 생성한다.

C드라이브 예시:

```powershell
New-Item -ItemType Directory -Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins"
```

D드라이브 예시:

```powershell
New-Item -ItemType Directory -Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins"
```

---

### 5-5. px4xplane 플러그인 다운로드

GitHub에서 Windows용 px4xplane 플러그인을 다운로드한다.

검색어:

```text
alireza787b px4xplane releases
```

다운로드 파일 예시:

```text
px4xplane-windows-vX.X.X.zip
```

---

### 5-6. px4xplane 플러그인 설치

압축 해제 후 `px4xplane` 폴더를 X-Plane 플러그인 폴더에 복사한다.

C드라이브 예시:

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins
```

D드라이브 예시:

```text
D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins
```

최종 구조는 다음과 같아야 한다.

```text
X-Plane 12
└─ Resources
   └─ plugins
      └─ px4xplane
         ├─ 64
         │  ├─ win.xpl
         │  └─ config.ini
         └─ px4_airframes
```

잘못된 구조:

```text
...\Resources\plugins\px4xplane-windows-vX.X.X\px4xplane\64\win.xpl
```

정상 구조:

```text
...\Resources\plugins\px4xplane\64\win.xpl
```

즉, 압축 파일의 최상위 폴더가 아니라 **그 안의 `px4xplane` 폴더 자체**가 `Resources\plugins` 바로 아래에 있어야 한다.

---

### 5-7. px4xplane 설치 확인

C드라이브 예시:

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl"
```

D드라이브 예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl"
```

정상 출력:

```text
True
```

정상 출력이 아니라면:

```text
False
```

이 경우 대부분 플러그인 폴더 구조가 잘못된 상태이다.

확인 명령:

C드라이브 예시:

```powershell
Get-ChildItem "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins" -Recurse -Filter "win.xpl"
```

D드라이브 예시:

```powershell
Get-ChildItem "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins" -Recurse -Filter "win.xpl"
```

만약 다음처럼 나오면 잘못 설치된 것이다.

```text
...\Resources\plugins\px4xplane-windows-vX.X.X\px4xplane\64\win.xpl
```

해결:

```text
px4xplane-windows-vX.X.X 폴더 안의 px4xplane 폴더만 꺼내서
Resources\plugins 바로 아래에 복사한다.
```

---

### 5-8. px4xplane 설정 파일 확인

C드라이브 예시:

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini"
```

D드라이브 예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini"
```

정상 출력:

```text
True
```

정상 출력이 아니라면:

```text
False
```

확인:

C드라이브 예시:

```powershell
Get-ChildItem "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane" -Recurse -Filter "config.ini"
```

D드라이브 예시:

```powershell
Get-ChildItem "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane" -Recurse -Filter "config.ini"
```

정상 위치:

```text
...\Resources\plugins\px4xplane\64\config.ini
```

---

### 5-9. px4xplane 설정 확인

debug 설정과 MAVLink 전송률 설정을 확인한다.

C드라이브 예시:

```powershell
Select-String -Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini" -Pattern "debug_|mavlink_sensor_rate_hz|mavlink_gps_rate_hz"
```

D드라이브 예시:

```powershell
Select-String -Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini" -Pattern "debug_|mavlink_sensor_rate_hz|mavlink_gps_rate_hz"
```

권장 설정:

```text
debug_verbose_logging = false
debug_log_sensor_timing = false
debug_log_sensor_values = false
debug_log_ekf_innovations = false
debug_log_accel_pipeline = false
debug_accel_bypass_calibration = false
mavlink_sensor_rate_hz = 50
mavlink_gps_rate_hz = 5
```

설정 변경 후에는 X-Plane을 완전히 종료했다가 다시 실행해야 한다.

---

### 5-10. X-Plane에서 플러그인 확인

X-Plane 12 실행 후 다음 메뉴에서 확인한다.

```text
Plugins → Plugin Admin
```

목록에 `px4xplane`이 표시되면 플러그인 설치가 완료된 것이다.

목록에 표시되지 않으면 다음을 확인한다.

```text
1. win.xpl 위치가 올바른지 확인
2. config.ini 위치가 올바른지 확인
3. X-Plane을 완전히 종료 후 재실행
4. X-Plane Log.txt에서 px4xplane 로딩 오류 확인
```

Log.txt 위치 예시:

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Log.txt
D:\SteamLibrary\steamapps\common\X-Plane 12\Log.txt
```

PowerShell에서 px4xplane 관련 로그만 확인한다.

C드라이브 예시:

```powershell
Select-String -Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Log.txt" -Pattern "px4xplane|win.xpl|plugin"
```

D드라이브 예시:

```powershell
Select-String -Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Log.txt" -Pattern "px4xplane|win.xpl|plugin"
```

---

### 5-11. ALIA-250 기체 확인

C드라이브 예시:

```powershell
Get-ChildItem "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Aircraft" -Recurse -Filter "*alia*"
```

D드라이브 예시:

```powershell
Get-ChildItem "D:\SteamLibrary\steamapps\common\X-Plane 12\Aircraft" -Recurse -Filter "*alia*"
```

정상 예시:

```text
BETA Technologies Alia-250
ALIA-250.acf
```

정상 출력이 없다면 다음을 확인한다.

```text
- ALIA-250 기체가 설치되지 않음
- Aircraft 폴더가 다른 위치에 있음
- 파일명이 alia가 아닌 다른 이름으로 되어 있음
```

Aircraft 폴더 전체에서 `.acf` 파일을 확인한다.

C드라이브 예시:

```powershell
Get-ChildItem "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Aircraft" -Recurse -Filter "*.acf"
```

D드라이브 예시:

```powershell
Get-ChildItem "D:\SteamLibrary\steamapps\common\X-Plane 12\Aircraft" -Recurse -Filter "*.acf"
```

---

## 6. Windows에서 WSL IP 확인

PX4 SITL은 WSL Ubuntu 안에서 실행되고, X-Plane은 Windows에서 실행된다.

따라서 PX4가 Windows의 X-Plane 플러그인으로 접속하려면 Windows의 WSL 가상 네트워크 IP를 알아야 한다.

이 단계는 반드시 **Windows PowerShell**에서 진행한다.

**[터미널 W = Windows PowerShell]**

```powershell
ipconfig
```

확인할 항목:

```text
이더넷 어댑터 vEthernet (WSL (Hyper-V firewall)):
   IPv4 주소 . . . . . . . . . : 172.xx.xxx.1
```

예시:

```text
172.20.192.1
```

이 값은 이후 Ubuntu에서 다음에 사용한다.

```bash
export PX4_SIM_HOSTNAME=172.20.192.1
```

주의:

```text
- VMware IP를 쓰면 안 된다.
- Wi-Fi 어댑터 IP를 쓰면 안 된다.
- Ethernet 물리 어댑터 IP를 쓰면 안 된다.
- 반드시 vEthernet (WSL)의 IPv4 주소를 사용한다.
```

---

## 7. X-Plane 연동 실행

### 7-1. 실행 창 구분

실행할 때는 아래처럼 창을 구분한다.

```text
창 1 = X-Plane 12
창 2 = Ubuntu 터미널 1 (mavlink-routerd)
창 3 = Ubuntu 터미널 2 (PX4 SITL)
창 4 = QGroundControl
```

---

### 7-2. 실행 순서

```text
1. X-Plane 실행
2. Ubuntu 터미널 1에서 mavlink-routerd 실행
3. Ubuntu 터미널 2에서 PX4 SITL 실행
4. QGroundControl 연결 확인
```

---

### 7-3. 창 1: X-Plane 준비

Windows에서 X-Plane 12를 먼저 실행한다.

**[창 1 = X-Plane 12]**

진행 순서:

```text
1. X-Plane 12 실행
2. ALIA-250 기체 로드
3. Pause 해제
4. Plugins → Plugin Admin → px4xplane Enabled 확인
5. px4xplane 메뉴에서 Enable / Start / Connect 실행
```

---

### 7-4. 창 2: Ubuntu 터미널 1에서 mavlink-routerd 실행

새 Ubuntu 터미널을 연다.

**[창 2 = Ubuntu 터미널 1]**

Windows PowerShell 또는 Windows Terminal에서 실행:

```powershell
wsl -d Ubuntu-22.04
```

기존 실행 중인 프로세스를 확인한다.

```bash
pgrep -a px4
pgrep -a mavlink-routerd
```

필요 시 종료한다.

```bash
pkill -f px4
pkill -f mavlink-routerd
```

다시 확인한다.

```bash
pgrep -a px4
pgrep -a mavlink-routerd
```

필요하면 PX4를 강제 종료한다.

```bash
pkill -9 -f px4
```

이후 `mavlink-routerd`를 실행한다.

```bash
mavlink-routerd -e 172.20.192.1:14540 -e 172.20.192.1:14550 -e 172.20.192.1:14569 -e 127.0.0.1:14569 0.0.0.0:14550
```

주의:

```text
- 172.20.192.1은 예시이다.
- 반드시 Windows PowerShell의 ipconfig에서 확인한 vEthernet (WSL) IPv4 주소로 바꾼다.
- 이 터미널은 계속 켜둔다.
```

mavlink-router 실행 시 다음 메시지가 보일 수 있다.

```text
Could not open conf file '/etc/mavlink-router/main.conf' (No such file or directory)
```

CLI 옵션으로 endpoint를 직접 지정해서 실행하는 경우에는 설정 파일이 없어도 동작할 수 있다.

즉, 아래처럼 endpoint가 열리면 진행 가능하다.

```text
Opened UDP Client ...
Opened UDP Server ...
```

---

### 7-5. 창 3: Ubuntu 터미널 2에서 PX4 SITL 실행

새 Ubuntu 터미널을 하나 더 연다.

**[창 3 = Ubuntu 터미널 2]**

Windows PowerShell 또는 Windows Terminal에서 실행:

```powershell
wsl -d Ubuntu-22.04
```

PX4 X-Plane fork 디렉터리로 이동한다.

```bash
cd ~/PX4-Autopilot-Me
```

주의:

```bash
~/PX4-Autopilot-Me
```

위처럼 입력하면 안 된다. 이는 경로 문자열일 뿐, 디렉터리 이동 명령이 아니다.

반드시 다음처럼 입력한다.

```bash
cd ~/PX4-Autopilot-Me
```

`PX4_SIM_HOSTNAME`을 설정한다.

```bash
export PX4_SIM_HOSTNAME=172.20.192.1
```

설정값을 확인한다.

```bash
echo $PX4_SIM_HOSTNAME
```

정상 출력 예시:

```text
172.20.192.1
```

주의:

```text
172.20.192.1은 예시이다.
실제 값은 Windows PowerShell의 ipconfig에서 확인한 vEthernet (WSL) IPv4 주소를 사용한다.
```

PX4 SITL을 실행한다.

```bash
make px4_sitl_default xplane_alia250
```

정상 로그 예시:

```text
INFO  [init] PX4_SIM_HOSTNAME: 172.20.192.1
INFO  [simulator_mavlink] using TCP on remote host 172.20.192.1 port 4560
INFO  [simulator_mavlink] Simulator connected on TCP port 4560.
pxh>
```

---

### 7-6. 창 4: QGroundControl 연결 확인

Windows에서 QGroundControl을 실행한다.

**[창 4 = QGroundControl]**

정상 연결되면 QGC에서 기체가 인식된다.

이 매뉴얼 기준의 최종 구성은 다음이다.

```text
X-Plane + px4xplane plugin + PX4 SITL + mavlink-routerd + QGroundControl
```

따라서 QGC까지 사용할 목적이면 `mavlink-routerd` 실행을 생략하지 않는다.

---

## 8. 연동 상태 확인

PX4 실행 터미널에서 `pxh>` 콘솔이 보이면 다음을 확인한다.

**[PX4 콘솔 = 창 3 안의 pxh>]**

### 8-1. IMU / HIL_SENSOR 수신률 확인

```bash
uorb top sensor_combined
```

정상 목표:

```text
sensor_combined RATE: 40~60 Hz 근처
```

---

### 8-2. GPS / HIL_GPS 수신률 확인

```bash
uorb top sensor_gps
```

정상 목표:

```text
sensor_gps RATE: 1~5 Hz
```

---

### 8-3. EKF local position 확인

```bash
listener vehicle_local_position
```

확인할 항목:

```text
xy_valid
z_valid
v_xy_valid
v_z_valid
dead_reckoning
```

Position 모드 가능 조건에 중요하다.

---

### 8-4. vehicle_status 확인

```bash
listener vehicle_status
```

확인할 항목:

```text
pre_flight_checks_pass
failsafe
gcs_connection_lost
nav_state
arming_state
```

---

## 9. 참고: sensor_gps RATE 0 문제

정상적으로는 다음처럼 GPS uORB publish rate가 나와야 한다.

```text
sensor_gps 0 RATE 5
```

만약 X-Plane 로그에서는 `[HIL_GPS #...]`가 증가하는데 PX4에서 다음처럼 나오면 문제가 있다.

```text
sensor_gps 0 RATE 0
sensor_gps 1 RATE 0
```

이 경우 `PX4-Autopilot-Me`의 `SimulatorMavlink.cpp`에서 HIL_GPS publish 경로를 확인해야 한다.

확인 명령:

```bash
cd ~/PX4-Autopilot-Me
grep -R "HIL_GPS\|hil_gps\|MAVLINK_MSG_ID_HIL_GPS" -n src/modules/simulation src/modules/mavlink | head -80
```

```bash
grep -R "sensor_gps" -n src/modules/simulation src/modules/mavlink | head -80
```

수정 대상 후보:

```text
src/modules/simulation/simulator_mavlink/SimulatorMavlink.cpp
handle_message_hil_gps()
```

X-Plane SITL에서는 HIL_GPS id가 안정적으로 고정되지 않을 수 있으므로, 필요 시 sensor_gps instance 0으로 고정 publish하는 패치를 검토한다.

패치 후 빌드:

```bash
cd ~/PX4-Autopilot-Me
make px4_sitl_default xplane_alia250
```

검증:

```bash
uorb top sensor_gps
```

정상 목표:

```text
sensor_gps 0 RATE 5
```

---

## 10. 참고: Position 모드 가능 조건

Position 모드는 GPS만 있다고 바로 되는 것이 아니라 EKF local position이 정상이어야 한다.

최소 확인 항목:

```bash
uorb top sensor_combined
uorb top sensor_gps
listener vehicle_local_position
listener vehicle_status
```

Position 모드 가능 최소 조건:

```text
sensor_combined RATE: 40~60 Hz 근처
sensor_gps RATE: 1~5 Hz
xy_valid: True
z_valid: True
v_xy_valid: True
v_z_valid: True
pre_flight_checks_pass: True
failsafe: False
```

권장 조건:

```text
dead_reckoning: False
gcs_connection_lost: False
heading_good_for_control: True
```

`listener vehicle_local_position`에서 값이 출력되지 않거나 `xy_valid: False`이면 Position 모드는 불가 상태이다.

---

## 부록 A. C드라이브 / D드라이브 경로 빠른 비교

| 항목 | C드라이브 예시 | D드라이브 예시 |
|---|---|---|
| X-Plane 설치 폴더 | `C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12` | `D:\SteamLibrary\steamapps\common\X-Plane 12` |
| plugins 폴더 | `C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins` | `D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins` |
| px4xplane win.xpl | `C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl` | `D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl` |
| X-Plane Log.txt | `C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Log.txt` | `D:\SteamLibrary\steamapps\common\X-Plane 12\Log.txt` |

---

## 부록 B. IP 확인 기준

| 확인 대상 | 확인 위치 | 명령 | 사용 목적 |
|---|---|---|---|
| vEthernet (WSL) IPv4 | Windows PowerShell | `ipconfig` | `PX4_SIM_HOSTNAME`, mavlink-router endpoint |
| PX4_SIM_HOSTNAME | WSL Ubuntu | `echo $PX4_SIM_HOSTNAME` | PX4가 X-Plane plugin으로 접속할 Windows IP |
| VMware IP | 사용하지 않음 | 사용하지 않음 | 본 매뉴얼에서는 사용하지 않음 |

핵심:

```text
PX4_SIM_HOSTNAME = Windows PowerShell ipconfig에서 확인한 vEthernet (WSL)의 IPv4 주소
```

---

## 부록 C. 최종 실행 방법 요약

실사용 시에는 아래 순서만 따라가면 된다.

### C-1. 최종 실행 순서

```text
1. X-Plane 실행
2. Ubuntu 창 1에서 mavlink-routerd 실행
3. Ubuntu 창 2에서 PX4 SITL 실행
4. QGroundControl 연결 확인
```

---

### C-2. 창별 구분

```text
창 1 = X-Plane 12
창 2 = Ubuntu 터미널 1 (mavlink-routerd)
창 3 = Ubuntu 터미널 2 (PX4 SITL)
창 4 = QGroundControl
```

---

### C-3. 창 1: X-Plane

```text
- X-Plane 12 실행
- ALIA-250 로드
- Pause 해제
- Plugins → Plugin Admin → px4xplane Enabled 확인
- px4xplane 메뉴에서 Enable / Start / Connect 실행
```

---

### C-4. 창 2: Ubuntu 터미널 1

```bash
wsl -d Ubuntu-22.04

pgrep -a px4
pgrep -a mavlink-routerd

pkill -f px4
pkill -f mavlink-routerd

pgrep -a px4
pgrep -a mavlink-routerd

pkill -9 -f px4

mavlink-routerd -e 172.20.192.1:14540 -e 172.20.192.1:14550 -e 172.20.192.1:14569 -e 127.0.0.1:14569 0.0.0.0:14550
```

주의:

```text
172.20.192.1은 예시이다.
실제 값은 Windows PowerShell ipconfig에서 확인한 vEthernet (WSL) IPv4 주소로 바꾼다.
```

---

### C-5. 창 3: Ubuntu 터미널 2

```bash
wsl -d Ubuntu-22.04

cd ~/PX4-Autopilot-Me
export PX4_SIM_HOSTNAME=172.20.192.1
echo $PX4_SIM_HOSTNAME
make px4_sitl_default xplane_alia250
```

주의:

```text
172.20.192.1은 예시이다.
실제 값은 Windows PowerShell ipconfig에서 확인한 vEthernet (WSL) IPv4 주소로 바꾼다.
```

---

### C-6. 창 4: QGroundControl

```text
- QGC 실행
- 기체 연결 확인
- 필요 시 14550 / 14540 관련 연결 상태 확인
```

---

## 부록 D. 자주 발생하는 실수

### D-1. `~/PX4-Autopilot-Me`만 입력하는 경우

잘못된 입력:

```bash
~/PX4-Autopilot-Me
```

올바른 입력:

```bash
cd ~/PX4-Autopilot-Me
```

---

### D-2. VMware IP를 사용하는 경우

잘못된 기준:

```text
VMware Network Adapter VMnet IP
```

올바른 기준:

```text
vEthernet (WSL)의 IPv4 주소
```

---

### D-3. mavlink-routerd 창을 닫는 경우

QGC까지 사용하는 본 매뉴얼 구성에서는 `mavlink-routerd` 창을 닫으면 QGC 연결이 끊길 수 있다.

따라서 다음 두 터미널은 계속 유지한다.

```text
Ubuntu 터미널 1 = mavlink-routerd
Ubuntu 터미널 2 = PX4 SITL
```

---

### D-4. X-Plane plugin 폴더 구조가 한 단계 깊은 경우

잘못된 구조:

```text
...\Resources\plugins\px4xplane-windows-vX.X.X\px4xplane\64\win.xpl
```

올바른 구조:

```text
...\Resources\plugins\px4xplane\64\win.xpl
```

---

### D-5. QGC 사용인데 mavlink-router를 생략하는 경우

PX4와 X-Plane의 TCP 4560 연결만 확인할 때는 mavlink-router 없이도 일부 연결이 성립할 수 있다.

하지만 본 매뉴얼은 QGroundControl까지 함께 사용하는 최종 운용을 기준으로 한다.

따라서 본 매뉴얼의 실행 절차에서는 `mavlink-routerd`를 생략하지 않는다.
