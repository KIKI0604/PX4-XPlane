# PX4 - X-Plane 설치 매뉴얼

이 매뉴얼은 **Windows + WSL2 Ubuntu + PX4 + X-Plane 12 + px4xplane 플러그인** 설치 및 연동을 위한 절차입니다.

핵심 연결 구조는 다음과 같습니다.

```text
PX4 simulator_mavlink → TCP 4560 → X-Plane px4xplane plugin
```

`mavlink-router`는 PX4와 X-Plane의 TCP 4560 연결 자체에는 필수 구성요소가 아닙니다.  
QGroundControl 또는 추가 MAVLink endpoint 라우팅이 필요할 때만 별도로 사용합니다.

---

## 전체 진행 순서

```text
1. WSL 설치 및 설정
2. PX4 기본 설치
3. 참고: mavlink-router
4. X-Plane 연동용 PX4 설치
5. X-Plane 12 및 px4xplane 플러그인 설치
6. Windows WSL IP 확인
7. X-Plane 연동 실행
8. 연동 상태 확인
9. 참고: sensor_gps RATE 0 문제
10. 참고: Position 모드 가능 조건
11. 부록 D. 최종 실행 방법 요약
```


---

## 터미널 / 창 구분 규칙

이 매뉴얼에서는 어떤 명령을 어디에서 실행해야 하는지 혼동하지 않도록 창을 다음처럼 구분합니다.

| 구분 | 실행 위치 | 용도 | 대표 명령 |
|---|---|---|---|
| 터미널 1 | Windows PowerShell | WSL 설치, X-Plane 경로 확인, WSL IP 확인 | `wsl -l -v`, `ipconfig`, `Test-Path` |
| 터미널 2 | WSL Ubuntu | PX4 설치, 빌드, SITL 실행, PX4 콘솔 확인 | `sudo apt update`, `make px4_sitl_default xplane_alia250`, `uorb top` |
| 터미널 3 | Windows X-Plane GUI | X-Plane 실행, 기체 로드, px4xplane 플러그인 활성화 | X-Plane 12 실행, `Plugins → Plugin Admin` |
| 터미널 4 | WSL Ubuntu 선택 사항 | mavlink-router를 별도로 실행할 때만 사용 | `mavlink-routerd ...` |

중요한 기준은 다음과 같습니다.

```text
PowerShell 명령 = Windows에서 실행
bash 명령       = WSL Ubuntu에서 실행
pxh> 명령       = PX4 SITL 실행 후 PX4 콘솔에서 실행
X-Plane 조작    = Windows의 X-Plane 프로그램 화면에서 실행
```

IP 확인 기준은 다음처럼 구분합니다.

```text
Windows에서 확인해야 하는 IP:
- ipconfig로 확인하는 vEthernet (WSL)의 IPv4 주소
- PX4_SIM_HOSTNAME에 넣을 값

Ubuntu에서 확인하는 값:
- echo $PX4_SIM_HOSTNAME
- PX4가 어떤 Windows IP로 접속하려는지 확인하는 값
```

예를 들어 Windows PowerShell에서 `ipconfig`로 다음 IP를 확인했다면,

```text
vEthernet (WSL) IPv4 주소: 172.17.160.1
```

WSL Ubuntu에서는 다음처럼 설정합니다.

```bash
export PX4_SIM_HOSTNAME=172.17.160.1
```

---

## 1. WSL 설치 및 설정

이 장은 **터미널 1: Windows PowerShell**에서 시작하고, Ubuntu 접속 후에는 **터미널 2: WSL Ubuntu**에서 이어서 진행합니다.

WSL2 Ubuntu 22.04를 설치합니다.

**터미널 1: Windows PowerShell**

```powershell
wsl --install -d Ubuntu-22.04
```

설치된 WSL 배포판과 WSL 버전을 확인합니다.

**터미널 1: Windows PowerShell**

```powershell
wsl -l -v
```

정상 예시:

```text
  NAME            STATE           VERSION
* Ubuntu-22.04    Stopped         2
```

Ubuntu 22.04에 접속합니다.

**터미널 1: Windows PowerShell**

```powershell
wsl -d Ubuntu-22.04
```

접속 후 열리는 Ubuntu shell을 이 매뉴얼에서는 **터미널 2: WSL Ubuntu**라고 부릅니다.

Ubuntu 패키지 목록을 갱신합니다.

**터미널 2: WSL Ubuntu**

```bash
sudo apt update
```

기존 패키지를 업데이트합니다.

```bash
sudo apt upgrade -y
```

PX4 기본 작업에 필요한 도구를 설치합니다.

```bash
sudo apt install git curl wget unzip python3 python3-pip -y
```

### 정상 출력이 아닐 때

#### 실패 예시 1: WSL 명령어를 찾을 수 없음

```text
'wsl' is not recognized as an internal or external command
```

해결 방법:

```text
1. Windows 기능에서 WSL이 활성화되어 있는지 확인
2. 관리자 PowerShell에서 wsl --install 재실행
3. Windows 재부팅
```

#### 실패 예시 2: Ubuntu 설치가 완료되지 않음

```text
There is no distribution with the supplied name.
```

해결 방법:

```powershell
wsl --list --online
wsl --install -d Ubuntu-22.04
```

설치 후 다시 확인합니다.

```powershell
wsl -l -v
```

#### 실패 예시 3: apt update 실패

```text
Temporary failure resolving 'archive.ubuntu.com'
```

해결 방법:

```bash
ping -c 4 google.com
```

인터넷 연결이 되지 않으면 Windows 네트워크, VPN, 방화벽, WSL DNS 설정을 확인합니다.

---

## 2. PX4 기본 설치

이 장은 **터미널 2: WSL Ubuntu**에서 진행합니다. 단, `wsl --shutdown`만 **터미널 1: Windows PowerShell**에서 실행합니다.

이 단계는 PX4 기본 SITL 빌드 환경이 정상인지 확인하기 위한 절차입니다.

홈 디렉토리로 이동합니다.

```bash
cd ~
```

PX4 고정 버전을 클론합니다.

> 아래 `v1.14.0`은 예시입니다. 실제 프로젝트 기준 버전이 따로 있으면 해당 태그 또는 브랜치로 변경합니다.

```bash
git clone --recursive --branch v1.14.0 https://github.com/PX4/PX4-Autopilot.git
```

PX4 폴더로 이동합니다.

```bash
cd ~/PX4-Autopilot
```

PX4 서브모듈을 동기화합니다.

```bash
git submodule update --init --recursive
```

현재 PX4 버전을 확인합니다.

```bash
git describe --tags
```

정상 예시:

```text
v1.14.0
```

PX4 Ubuntu 개발환경을 설치합니다.

```bash
bash ./Tools/setup/ubuntu.sh
```

설치 후 WSL을 완전히 재시작합니다.

별도의 PowerShell을 열고 다음 명령을 실행합니다.

```powershell
wsl --shutdown
```

Ubuntu에 다시 접속합니다.

```powershell
wsl -d Ubuntu-22.04
```

PX4 SITL 기본 빌드 및 실행을 확인합니다.

```bash
cd ~/PX4-Autopilot
make px4_sitl none
```

정상 예시:

```text
pxh>
```

PX4 콘솔에서 종료하려면 다음을 입력합니다.

```text
shutdown
```

### 정상 출력이 아닐 때

#### 실패 예시 1: git clone 실패

```text
fatal: unable to access 'https://github.com/PX4/PX4-Autopilot.git/'
```

해결 방법:

```bash
ping -c 4 github.com
```

네트워크 연결, 회사망 프록시, VPN, 방화벽을 확인합니다.

#### 실패 예시 2: submodule 누락

```text
fatal: clone of ... failed
```

해결 방법:

```bash
cd ~/PX4-Autopilot
git submodule sync --recursive
git submodule update --init --recursive
```

#### 실패 예시 3: 빌드 도중 의존성 오류

```text
ModuleNotFoundError
```

또는

```text
command not found
```

해결 방법:

```bash
cd ~/PX4-Autopilot
bash ./Tools/setup/ubuntu.sh
```

이후 WSL을 재시작합니다.

```powershell
wsl --shutdown
wsl -d Ubuntu-22.04
```

다시 빌드합니다.

```bash
cd ~/PX4-Autopilot
make px4_sitl none
```

---

## 3. 참고: mavlink-router

이 장은 선택 사항입니다. 설치 확인은 **터미널 2: WSL Ubuntu**에서 진행합니다. 단, mavlink-router를 계속 실행한 상태로 PX4 SITL도 같이 실행하려면 **터미널 4: WSL Ubuntu 추가 창**을 별도로 열어 사용합니다.

`mavlink-router`는 **PX4와 X-Plane TCP 연결 자체에는 필수 구성요소가 아닙니다.**

PX4 ↔ X-Plane 연결의 핵심 구조는 다음입니다.

```text
PX4 simulator_mavlink → TCP 4560 → X-Plane px4xplane plugin
```

`mavlink-router`는 다음과 같은 경우에만 별도로 사용합니다.

```text
- QGroundControl을 동시에 연결해야 하는 경우
- 여러 MAVLink endpoint로 데이터를 라우팅해야 하는 경우
- PX4 SITL, QGC, 외부 프로그램을 동시에 붙여야 하는 경우
```

따라서 **PX4-XPlane 연결 확인 단계에서는 mavlink-router 없이 먼저 진행하는 것을 권장합니다.**

### mavlink-router 설치 시도

Ubuntu에서 다음 명령을 실행합니다.

```bash
sudo apt update
sudo apt install mavlink-router -y
```

정상 설치 시 예시는 다음과 같습니다.

```text
Setting up mavlink-router ...
```

설치 여부를 확인합니다.

```bash
which mavlink-routerd
```

정상 출력 예시:

```text
/usr/bin/mavlink-routerd
```

또는 버전을 확인합니다.

```bash
mavlink-routerd --version
```

### 정상 출력이 아닐 때

#### 실패 예시 1: 패키지를 찾을 수 없음

Ubuntu 기본 저장소에서 다음 오류가 발생할 수 있습니다.

```text
E: Unable to locate package mavlink-router
```

이 경우 **mavlink-router 설치를 건너뛰고 PX4-XPlane 연결부터 확인**합니다.

```text
mavlink-router 없음 → PX4-XPlane TCP 4560 연결에는 영향 없음
```

다음 단계로 그대로 진행합니다.

```bash
cd ~/PX4-Autopilot-Me
export PX4_SIM_HOSTNAME=<Windows WSL vEthernet IP>
make px4_sitl_default xplane_alia250
```

#### 실패 예시 2: 명령어가 없음

설치 후 다음 오류가 나올 수 있습니다.

```text
mavlink-routerd: command not found
```

설치가 완료되지 않았거나 PATH에 등록되지 않은 상태입니다.

확인합니다.

```bash
dpkg -l | grep mavlink-router
```

결과가 없으면 설치되지 않은 상태입니다.

```bash
sudo apt update
sudo apt install mavlink-router -y
```

그래도 설치가 되지 않으면 이 단계에서는 생략합니다.

```text
PX4-XPlane 연결 확인에는 mavlink-router가 필요하지 않으므로 생략 가능
```

#### 실패 예시 3: 설정 파일 없음

mavlink-router 실행 시 다음 메시지가 나올 수 있습니다.

```text
Could not open conf file '/etc/mavlink-router/main.conf' (No such file or directory)
```

이 메시지는 설정 파일이 없다는 경고에 가깝고, CLI 옵션으로 endpoint를 직접 지정하면 동작할 수 있습니다.

예시:

```bash
mavlink-routerd -e 172.17.160.1:14550 0.0.0.0:14540
```

다만 이 매뉴얼에서는 우선순위를 다음처럼 둡니다.

```text
1순위: PX4 ↔ X-Plane TCP 4560 연결 확인
2순위: QGroundControl 연결 필요 시 mavlink-router 추가
```

---

## 4. X-Plane 연동용 PX4 설치

이 장은 **터미널 2: WSL Ubuntu**에서 진행합니다.

X-Plane 연동용 PX4 fork를 클론합니다.

```bash
cd ~
git clone --recursive -b px4xplane-sitl https://github.com/alireza787b/PX4-Autopilot-Me.git PX4-Autopilot-Me
```

X-Plane fork 폴더로 이동하고 서브모듈을 동기화합니다.

```bash
cd ~/PX4-Autopilot-Me
git submodule update --init --recursive
```

현재 브랜치를 확인합니다.

```bash
git branch --show-current
```

정상 출력:

```text
px4xplane-sitl
```

작업 트리 상태를 확인합니다.

```bash
git status
```

정상 예시:

```text
nothing to commit, working tree clean
```

### 정상 출력이 아닐 때

#### 실패 예시 1: 브랜치가 다름

```text
main
```

또는

```text
master
```

처럼 출력되면 X-Plane 연동용 브랜치가 아닐 수 있습니다.

해결 방법:

```bash
cd ~/PX4-Autopilot-Me
git fetch --all
git checkout px4xplane-sitl
git submodule update --init --recursive
```

다시 확인합니다.

```bash
git branch --show-current
```

정상 출력:

```text
px4xplane-sitl
```

#### 실패 예시 2: 이미 폴더가 존재함

```text
fatal: destination path 'PX4-Autopilot-Me' already exists and is not an empty directory.
```

기존 폴더가 이미 있는 상태입니다.

기존 작업을 보존하려면 이름을 바꿉니다.

```bash
mv ~/PX4-Autopilot-Me ~/PX4-Autopilot-Me_backup
```

다시 클론합니다.

```bash
cd ~
git clone --recursive -b px4xplane-sitl https://github.com/alireza787b/PX4-Autopilot-Me.git PX4-Autopilot-Me
```

#### 실패 예시 3: 서브모듈 오류

```text
fatal: clone of ... failed
```

또는 일부 submodule이 누락되면 다시 동기화합니다.

```bash
cd ~/PX4-Autopilot-Me
git submodule sync --recursive
git submodule update --init --recursive
```

---

## 5. X-Plane 12 및 px4xplane 플러그인 설치

이 장은 주로 **터미널 1: Windows PowerShell**과 **터미널 3: Windows X-Plane GUI**에서 진행합니다. X-Plane 설치 경로 확인, 플러그인 파일 확인, `Log.txt` 확인은 PowerShell에서 실행합니다.

### X-Plane 12 설치

X-Plane 12는 일반 설치판 또는 Steam 설치판 모두 사용할 수 있습니다.

Steam 설치판의 기본 경로 예시는 다음과 같습니다.

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12
```

D드라이브에 Steam 라이브러리를 만든 경우에는 다음과 같은 경로일 수 있습니다.

```text
D:\SteamLibrary\steamapps\common\X-Plane 12
```

또는 사용자가 직접 설치한 경우 다음처럼 별도 경로일 수 있습니다.

```text
D:\X-Plane 12
```

이후 명령어에서 경로가 다르면, 본인 PC의 실제 X-Plane 설치 경로로 바꿔서 실행합니다.

X-Plane 12 설치 폴더는 보통 아래 구조를 가집니다.

```text
X-Plane 12
├─ Aircraft
├─ Resources
│  └─ plugins
├─ X-Plane.exe
└─ Log.txt
```

---

### X-Plane 설치 경로 확인

기본 Steam 경로를 사용하는 경우 다음을 확인합니다.

**터미널 1: Windows PowerShell**

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12"
```

정상 출력:

```text
True
```

D드라이브 Steam 라이브러리에 설치한 경우 예시는 다음과 같습니다.

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12"
```

정상 출력:

```text
True
```

직접 `D:\X-Plane 12`에 설치한 경우 예시는 다음과 같습니다.

```powershell
Test-Path "D:\X-Plane 12"
```

정상 출력:

```text
True
```

### 정상 출력이 아닐 때

정상 출력이 아니라면 다음처럼 나옵니다.

```text
False
```

가능한 원인은 다음입니다.

```text
- X-Plane 12가 설치되지 않음
- Steam 설치 경로가 기본 경로가 아님
- D드라이브 또는 다른 SteamLibrary에 설치됨
- 데모 버전 또는 별도 경로에 설치됨
- 폴더명이 다르거나 드라이브가 다름
```

해결 방법:

1. Steam에서 설치 경로를 확인합니다.

```text
Steam → Library → X-Plane 12 → Manage → Browse local files
```

2. 실제 열린 폴더 경로를 복사합니다.

예시:

```text
D:\SteamLibrary\steamapps\common\X-Plane 12
```

3. 복사한 경로로 다시 확인합니다.

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12"
```

정상 출력:

```text
True
```

---

### X-Plane 실행 파일 확인

X-Plane 설치 폴더가 맞는지 확인하려면 `X-Plane.exe` 존재 여부를 확인합니다.

기본 C드라이브 Steam 경로:

**터미널 1: Windows PowerShell**

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\X-Plane.exe"
```

D드라이브 Steam 경로 예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\X-Plane.exe"
```

직접 설치 경로 예시:

```powershell
Test-Path "D:\X-Plane 12\X-Plane.exe"
```

정상 출력:

```text
True
```

`False`가 나오면 해당 경로는 X-Plane 설치 루트가 아닙니다.

---

### X-Plane 플러그인 폴더 확인

기본 C드라이브 Steam 경로:

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins"
```

정상 출력:

```text
True
```

D드라이브 Steam 경로 예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins"
```

정상 출력:

```text
True
```

직접 설치 경로 예시:

```powershell
Test-Path "D:\X-Plane 12\Resources\plugins"
```

정상 출력:

```text
True
```

### 정상 출력이 아닐 때

정상 출력이 아니라면 다음처럼 나옵니다.

```text
False
```

가능한 원인은 다음입니다.

```text
- X-Plane 설치 경로가 다름
- Resources 폴더 구조가 손상됨
- plugins 폴더가 삭제됨
```

먼저 X-Plane 설치 폴더가 맞는지 확인합니다.

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\X-Plane.exe"
```

`X-Plane.exe`는 있는데 `Resources\plugins`가 없으면 폴더를 생성합니다.

```powershell
New-Item -ItemType Directory -Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins"
```

다시 확인합니다.

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins"
```

정상 출력:

```text
True
```

단, `X-Plane.exe` 자체가 없다면 경로가 잘못된 것입니다.  
이 경우 Steam의 **Browse local files**로 실제 경로를 먼저 찾아야 합니다.

---

### px4xplane 플러그인 다운로드

GitHub에서 Windows용 px4xplane 플러그인을 다운로드합니다.

검색어:

```text
alireza787b px4xplane releases
```

다운로드 파일 예시:

```text
px4xplane-windows-vX.X.X.zip
```

다운로드 후 압축을 해제합니다.

---

### px4xplane 플러그인 설치

압축 해제 후 `px4xplane` 폴더를 X-Plane의 `Resources\plugins` 아래에 복사합니다.

기본 C드라이브 Steam 경로:

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins
```

D드라이브 Steam 경로 예시:

```text
D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins
```

직접 설치 경로 예시:

```text
D:\X-Plane 12\Resources\plugins
```

최종 구조는 다음과 같아야 합니다.

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

D드라이브 Steam 설치 예시 기준 정상 구조는 다음입니다.

```text
D:\SteamLibrary\steamapps\common\X-Plane 12
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

즉, `px4xplane-windows-vX.X.X` 폴더 자체를 넣는 것이 아니라, 그 안의 `px4xplane` 폴더가 `plugins` 바로 아래에 있어야 합니다.

---

### px4xplane 설치 확인

기본 C드라이브 Steam 경로:

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl"
```

정상 출력:

```text
True
```

D드라이브 Steam 경로 예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl"
```

정상 출력:

```text
True
```

직접 설치 경로 예시:

```powershell
Test-Path "D:\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl"
```

정상 출력:

```text
True
```

### 정상 출력이 아닐 때

정상 출력이 아니라면 다음처럼 나옵니다.

```text
False
```

대부분 플러그인 폴더 구조가 잘못된 상태입니다.

확인 명령:

```powershell
Get-ChildItem "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins" -Recurse -Filter "win.xpl"
```

만약 다음처럼 나오면 잘못 설치된 것입니다.

```text
...\Resources\plugins\px4xplane-windows-vX.X.X\px4xplane\64\win.xpl
```

해결 방법:

```text
px4xplane-windows-vX.X.X 폴더 안의 px4xplane 폴더만 꺼내서
Resources\plugins 바로 아래에 복사해야 합니다.
```

정상 위치:

```text
...\Resources\plugins\px4xplane\64\win.xpl
```

---

### px4xplane 설정 파일 확인

기본 C드라이브 Steam 경로:

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini"
```

정상 출력:

```text
True
```

D드라이브 Steam 경로 예시:

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini"
```

정상 출력:

```text
True
```

직접 설치 경로 예시:

```powershell
Test-Path "D:\X-Plane 12\Resources\plugins\px4xplane\64\config.ini"
```

정상 출력:

```text
True
```

### 정상 출력이 아닐 때

정상 출력이 아니라면 다음처럼 나옵니다.

```text
False
```

가능한 원인은 다음입니다.

```text
- 플러그인 압축 해제가 잘못됨
- config.ini가 64 폴더 안에 없음
- px4xplane 릴리즈 파일 구성이 다름
```

확인합니다.

```powershell
Get-ChildItem "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane" -Recurse -Filter "config.ini"
```

`config.ini`가 다른 위치에 있으면 `64` 폴더 안으로 복사합니다.

정상 위치:

```text
...\Resources\plugins\px4xplane\64\config.ini
```

---

### px4xplane 설정 확인

debug 설정과 MAVLink 전송률 설정을 확인합니다.

C드라이브 기본 경로:

```powershell
Select-String -Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini" -Pattern "debug_|mavlink_sensor_rate_hz|mavlink_gps_rate_hz"
```

D드라이브 Steam 경로 예시:

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

### 설정 변경

필요하면 PowerShell에서 다음을 실행합니다.

C드라이브 기본 경로:

```powershell
(Get-Content "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini") `
-replace "debug_verbose_logging = true", "debug_verbose_logging = false" `
-replace "debug_log_sensor_timing = true", "debug_log_sensor_timing = false" `
-replace "debug_log_sensor_values = true", "debug_log_sensor_values = false" `
-replace "debug_log_ekf_innovations = true", "debug_log_ekf_innovations = false" `
-replace "debug_log_accel_pipeline = true", "debug_log_accel_pipeline = false" `
-replace "mavlink_sensor_rate_hz = 200", "mavlink_sensor_rate_hz = 50" `
-replace "mavlink_gps_rate_hz = 20", "mavlink_gps_rate_hz = 5" |
Set-Content "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini"
```

D드라이브 Steam 경로 예시:

```powershell
(Get-Content "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini") `
-replace "debug_verbose_logging = true", "debug_verbose_logging = false" `
-replace "debug_log_sensor_timing = true", "debug_log_sensor_timing = false" `
-replace "debug_log_sensor_values = true", "debug_log_sensor_values = false" `
-replace "debug_log_ekf_innovations = true", "debug_log_ekf_innovations = false" `
-replace "debug_log_accel_pipeline = true", "debug_log_accel_pipeline = false" `
-replace "mavlink_sensor_rate_hz = 200", "mavlink_sensor_rate_hz = 50" `
-replace "mavlink_gps_rate_hz = 20", "mavlink_gps_rate_hz = 5" |
Set-Content "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini"
```

`debug log`를 끄고 MAVLink sensor/GPS rate를 안정화 값으로 낮춥니다.

설정 변경 후에는 X-Plane을 완전히 종료했다가 다시 실행해야 합니다.

---

### X-Plane에서 플러그인 확인

X-Plane 12 실행 후 다음 메뉴에서 확인합니다.

```text
Plugins → Plugin Admin
```

목록에 `px4xplane`이 표시되면 플러그인 설치가 완료된 것입니다.

목록에 표시되지 않으면 다음을 확인합니다.

```text
1. win.xpl 위치가 올바른지 확인
2. config.ini 위치가 올바른지 확인
3. X-Plane을 완전히 종료 후 재실행
4. X-Plane Log.txt에서 px4xplane 로딩 오류 확인
```

C드라이브 기본 경로의 Log.txt:

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Log.txt
```

D드라이브 Steam 경로 예시의 Log.txt:

```text
D:\SteamLibrary\steamapps\common\X-Plane 12\Log.txt
```

PowerShell에서 px4xplane 관련 로그만 확인합니다.

```powershell
Select-String -Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Log.txt" -Pattern "px4xplane|win.xpl|plugin"
```

---

### ALIA-250 기체 확인

C드라이브 기본 경로:

```powershell
Get-ChildItem "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Aircraft" -Recurse -Filter "*alia*"
```

D드라이브 Steam 경로 예시:

```powershell
Get-ChildItem "D:\SteamLibrary\steamapps\common\X-Plane 12\Aircraft" -Recurse -Filter "*alia*"
```

정상 예시:

```text
BETA Technologies Alia-250
ALIA-250.acf
```

### 정상 출력이 아닐 때

정상 출력이 없다면 다음을 확인합니다.

```text
- ALIA-250 기체가 설치되지 않음
- Aircraft 폴더가 다른 위치에 있음
- 파일명이 alia가 아닌 다른 이름으로 되어 있음
```

Aircraft 폴더 전체에서 `.acf` 파일을 확인합니다.

```powershell
Get-ChildItem "D:\SteamLibrary\steamapps\common\X-Plane 12\Aircraft" -Recurse -Filter "*.acf"
```

ALIA-250이 없다면 X-Plane 내 Aircraft 설치 상태를 먼저 확인해야 합니다.

---

## 6. Windows WSL IP 확인

이 장은 **터미널 1: Windows PowerShell**에서 확인합니다. 여기서 확인한 IP를 나중에 **터미널 2: WSL Ubuntu**의 `PX4_SIM_HOSTNAME`에 입력합니다.

Windows PowerShell에서 WSL용 가상 네트워크 IP를 확인합니다.

**터미널 1: Windows PowerShell**

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
172.17.160.1
```

VMware IP가 아니라 `vEthernet (WSL)`의 IPv4 주소를 사용해야 합니다.

### 정상 출력이 아닐 때

#### 실패 예시 1: vEthernet WSL 항목이 보이지 않음

가능한 원인:

```text
- WSL이 실행 중이 아님
- WSL2가 아니라 WSL1로 설정됨
- Windows 네트워크 어댑터 표시가 갱신되지 않음
```

해결 방법:

PowerShell에서 WSL을 실행합니다.

```powershell
wsl -d Ubuntu-22.04
```

다른 PowerShell에서 다시 확인합니다.

```powershell
ipconfig
```

#### 실패 예시 2: WSL 버전이 1로 표시됨

확인:

```powershell
wsl -l -v
```

예시:

```text
Ubuntu-22.04    Running    1
```

해결:

```powershell
wsl --set-version Ubuntu-22.04 2
```

다시 확인합니다.

```powershell
wsl -l -v
```

정상 예시:

```text
Ubuntu-22.04    Running    2
```

---

## 7. X-Plane 연동 실행

이 장에서는 창을 동시에 사용합니다. **터미널 3: X-Plane GUI**를 먼저 준비하고, **터미널 2: WSL Ubuntu**에서 PX4 SITL을 실행합니다.

### X-Plane 준비

먼저 Windows의 X-Plane 프로그램 화면에서 다음을 진행합니다.

**터미널 3: Windows X-Plane GUI**

```text
1. X-Plane 12 실행
2. ALIA-250 기체 로드
3. Pause 해제
4. Plugins → Plugin Admin → px4xplane Enabled 확인
5. px4xplane 메뉴에서 Enable / Start / Connect 실행
```

### PX4 SIM 호스트네임 설정

WSL Ubuntu에서 다음을 실행합니다. 이때 IP는 **터미널 1: Windows PowerShell**에서 `ipconfig`로 확인한 `vEthernet (WSL)` IPv4 주소를 넣습니다.

**터미널 2: WSL Ubuntu**

```bash
cd ~/PX4-Autopilot-Me
export PX4_SIM_HOSTNAME=172.17.160.1
```

PX4가 Windows에서 실행 중인 X-Plane 플러그인에 접속할 IP를 설정합니다.

IP는 `ipconfig`에서 확인한 `vEthernet (WSL)`의 IPv4 주소로 변경합니다.

설정값을 확인합니다.

**터미널 2: WSL Ubuntu**

```bash
echo $PX4_SIM_HOSTNAME
```

정상 출력 예시:

```text
172.17.160.1
```

### X-Plane ALIA-250 연동용 PX4 SITL 실행

X-Plane ALIA-250 연동용 PX4 SITL을 실행합니다. 이 명령은 실행 후 PX4 콘솔 `pxh>`로 들어가므로, 이 창은 종료하지 말고 유지합니다.

**터미널 2: WSL Ubuntu**

```bash
cd ~/PX4-Autopilot-Me
export PX4_SIM_HOSTNAME=172.17.160.1
make px4_sitl_default xplane_alia250
```

정상 로그 예시:

```text
INFO  [init] PX4_SIM_HOSTNAME: 172.17.160.1
INFO  [simulator_mavlink] using TCP on remote host 172.17.160.1 port 4560
INFO  [simulator_mavlink] Simulator connected on TCP port 4560.
pxh>
```

### 정상 출력이 아닐 때

#### 실패 예시 1: TCP 4560 연결 실패

```text
ERROR [simulator_mavlink] poll timeout
```

또는

```text
Simulator not connected
```

확인 순서:

```text
1. X-Plane이 먼저 실행되어 있는지 확인
2. ALIA-250 기체가 로드되어 있는지 확인
3. px4xplane 플러그인이 Enabled 상태인지 확인
4. px4xplane 메뉴에서 Enable / Start / Connect를 실행했는지 확인
5. PX4_SIM_HOSTNAME이 WSL vEthernet IP인지 확인
6. Windows 방화벽에서 X-Plane 통신이 차단되지 않았는지 확인
```

PX4_SIM_HOSTNAME 확인:

```bash
echo $PX4_SIM_HOSTNAME
```

Windows PowerShell에서 WSL IP 재확인:

```powershell
ipconfig
```

다시 실행:

```bash
cd ~/PX4-Autopilot-Me
export PX4_SIM_HOSTNAME=<vEthernet WSL IPv4 주소>
make px4_sitl_default xplane_alia250
```

#### 실패 예시 2: make target 없음

```text
No rule to make target 'xplane_alia250'
```

가능한 원인:

```text
- PX4-Autopilot-Me가 아니라 기본 PX4-Autopilot 폴더에서 실행함
- px4xplane-sitl 브랜치가 아님
- 서브모듈 또는 airframe 파일이 누락됨
```

확인:

```bash
pwd
git branch --show-current
```

정상 예시:

```text
/home/<user>/PX4-Autopilot-Me
px4xplane-sitl
```

해결:

```bash
cd ~/PX4-Autopilot-Me
git checkout px4xplane-sitl
git submodule update --init --recursive
make px4_sitl_default xplane_alia250
```

#### 실패 예시 3: 포트 또는 방화벽 문제

Windows Defender 방화벽에서 X-Plane 또는 플러그인 통신이 차단되면 TCP 연결이 되지 않을 수 있습니다.

해결 방법:

```text
1. Windows 보안 → 방화벽 및 네트워크 보호
2. 앱이 방화벽을 통과하도록 허용
3. X-Plane.exe 허용
4. 개인 네트워크 허용 체크
5. 필요 시 공용 네트워크도 테스트 목적으로 임시 허용
```

---

## 8. 연동 상태 확인

이 장은 **터미널 2: WSL Ubuntu**의 PX4 콘솔 `pxh>`에서 진행합니다. `make px4_sitl_default xplane_alia250` 실행 후 `pxh>`가 떠 있는 같은 창에서 입력합니다.

PX4 콘솔 `pxh>`에서 다음을 확인합니다.

명령 앞에 `pxh>`가 보이는 상태에서 입력합니다. 일반 Ubuntu shell이 아니라 PX4 콘솔입니다.

### IMU / HIL_SENSOR 수신률 확인

```bash
uorb top sensor_combined
```

정상 목표:

```text
sensor_combined RATE: 40~60 Hz 근처
```

### GPS / HIL_GPS 수신률 확인

```bash
uorb top sensor_gps
```

정상 목표:

```text
sensor_gps RATE: 1~5 Hz
```

### EKF local position 생성 여부 확인

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

### vehicle_status 확인

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

### 정상 출력이 아닐 때

#### 실패 예시 1: sensor_combined RATE가 0

가능한 원인:

```text
- X-Plane px4xplane 플러그인이 연결되지 않음
- TCP 4560 연결 실패
- X-Plane Pause 상태
- px4xplane에서 sensor MAVLink 전송이 꺼져 있음
```

확인:

```text
1. X-Plane Pause 해제
2. px4xplane Enabled 확인
3. px4xplane Start / Connect 확인
4. PX4 로그에 Simulator connected on TCP port 4560 표시 확인
```

#### 실패 예시 2: sensor_gps RATE가 0

가능한 원인:

```text
- px4xplane에서 HIL_GPS 송신 문제
- PX4-Autopilot-Me의 HIL_GPS 처리 문제
- GPS instance id가 불안정하여 publish가 정상적으로 이어지지 않음
```

이 경우 9장 `sensor_gps RATE 0 문제`를 확인합니다.

#### 실패 예시 3: vehicle_local_position이 유효하지 않음

예시:

```text
xy_valid: False
z_valid: False
```

가능한 원인:

```text
- IMU/GPS 데이터 수신 불안정
- EKF 초기화 미완료
- GPS rate 0
- heading 또는 local position 조건 불충족
```

확인 순서:

```bash
uorb top sensor_combined
uorb top sensor_gps
listener vehicle_local_position
listener vehicle_status
```

---

## 9. 참고: sensor_gps RATE 0 문제

이 장의 PX4 코드 확인과 빌드는 **터미널 2: WSL Ubuntu**에서 진행합니다. X-Plane `Log.txt` 확인이 필요하면 **터미널 1: Windows PowerShell**에서 확인합니다.

정상적으로는 다음처럼 GPS uORB publish rate가 나와야 합니다.

```text
sensor_gps 0 RATE 5
```

만약 X-Plane 로그에서는 `[HIL_GPS #...]`가 증가하는데 PX4에서 다음처럼 나오면 문제가 있습니다.

```text
sensor_gps 0 RATE 0
sensor_gps 1 RATE 0
```

이 경우 `PX4-Autopilot-Me`의 `SimulatorMavlink.cpp`에서 HIL_GPS publish 경로를 확인해야 합니다.

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

문제 구조 예시:

```cpp
for (size_t i = 0; i < sizeof(_gps_ids) / sizeof(_gps_ids[0]); i++) {
    if (_sensor_gps_pubs[i] && _gps_ids[i] == hil_gps.id) {
        _sensor_gps_pubs[i]->publish(gps);
        break;
    }

    if (_sensor_gps_pubs[i] == nullptr) {
        _sensor_gps_pubs[i] = new uORB::PublicationMulti<sensor_gps_s> {ORB_ID(sensor_gps)};
        _gps_ids[i] = hil_gps.id;
        _sensor_gps_pubs[i]->publish(gps);
        break;
    }
}
```

X-Plane SITL에서는 HIL_GPS id가 안정적으로 고정되지 않을 수 있으므로, 필요 시 sensor_gps instance 0으로 고정 publish하는 패치를 검토합니다.

패치 예시:

```cpp
gps.timestamp = hrt_absolute_time();

// X-Plane HIL_GPS may provide non-stable or changing GPS IDs.
// For X-Plane SITL, keep a single stable simulated GPS instance.
constexpr size_t gps_instance = 0;

if (_sensor_gps_pubs[gps_instance] == nullptr) {
    _sensor_gps_pubs[gps_instance] = new uORB::PublicationMulti<sensor_gps_s> {ORB_ID(sensor_gps)};
}

_gps_ids[gps_instance] = 0;

device::Device::DeviceId device_id;
device_id.devid_s.bus_type = device::Device::DeviceBusType::DeviceBusType_SIMULATION;
device_id.devid_s.bus = 0;
device_id.devid_s.address = gps_instance;
device_id.devid_s.devtype = DRV_GPS_DEVTYPE_SIM;
gps.device_id = device_id.devid;

_sensor_gps_pubs[gps_instance]->publish(gps);
```

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

### 패치 전 주의사항

이 패치는 X-Plane SITL에서 GPS instance가 불안정한 경우를 위한 임시 보정 성격입니다.  
실기체 펌웨어나 일반 PX4 upstream 코드에 그대로 적용하는 용도로 보면 안 됩니다.

권장 절차:

```text
1. 먼저 px4xplane 플러그인 연결 상태 확인
2. X-Plane Log.txt에서 HIL_GPS 송신 여부 확인
3. PX4 uorb top sensor_gps 확인
4. 그래도 RATE 0이면 SimulatorMavlink.cpp publish 경로 검토
5. X-Plane SITL 한정 패치 적용 여부 판단
```

---

## 10. 참고: Position 모드 가능 조건

이 장은 **터미널 2: WSL Ubuntu**의 PX4 콘솔 `pxh>`에서 확인합니다.

Position 모드는 GPS만 있다고 바로 되는 것이 아니라 EKF local position이 정상이어야 합니다.

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

`listener vehicle_local_position`에서 값이 출력되지 않거나 `xy_valid: False`이면 Position 모드는 불가 상태입니다.

### Position 모드가 되지 않을 때 확인 순서

```text
1. sensor_combined RATE 확인
2. sensor_gps RATE 확인
3. vehicle_local_position의 xy_valid / z_valid 확인
4. vehicle_status의 pre_flight_checks_pass 확인
5. failsafe / gcs_connection_lost 확인
6. EKF 관련 메시지 확인
```

PX4 콘솔에서 로그 메시지를 확인합니다.

```bash
dmesg
```

또는 관련 listener를 추가로 확인합니다.

```bash
listener estimator_status
```

확인할 수 있는 항목:

```text
gps_check_fail_flags
pos_horiz_accuracy
pos_vert_accuracy
output_tracking_error
```

### 판단 기준

```text
GPS rate가 0이면 → GPS publish 문제부터 해결
sensor_combined rate가 0이면 → X-Plane sensor/HIL_SENSOR 연결 문제부터 해결
xy_valid가 False이면 → EKF local position 생성 문제
pre_flight_checks_pass가 False이면 → arming/position mode 진입 조건 미충족
failsafe가 True이면 → failsafe 원인 제거 필요
```

---

## 부록 A. 경로를 바꿔서 쓰는 방법

이 매뉴얼에서는 C드라이브 기본 Steam 경로와 D드라이브 예시를 함께 사용합니다.

기본 C드라이브 Steam 경로:

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12
```

D드라이브 Steam 라이브러리 예시:

```text
D:\SteamLibrary\steamapps\common\X-Plane 12
```

D드라이브 직접 설치 예시:

```text
D:\X-Plane 12
```

본인 PC의 실제 X-Plane 설치 경로가 다르면, 모든 PowerShell 명령에서 해당 경로만 바꿔서 사용합니다.

예를 들어 기존 명령이 다음이라면:

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl"
```

D드라이브 Steam 설치자는 다음처럼 바꿉니다.

```powershell
Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl"
```

직접 D드라이브 설치자는 다음처럼 바꿉니다.

```powershell
Test-Path "D:\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl"
```

---

## 부록 B. 최소 성공 기준

최종적으로 다음 조건이 만족되면 PX4-XPlane 기본 연동은 성공으로 판단할 수 있습니다.

```text
1. X-Plane 12 실행됨
2. ALIA-250 기체 로드됨
3. px4xplane 플러그인 Enabled 상태
4. PX4_SIM_HOSTNAME이 WSL vEthernet IP로 설정됨
5. PX4 로그에 Simulator connected on TCP port 4560 출력
6. sensor_combined RATE가 40~60 Hz 근처
7. sensor_gps RATE가 1~5 Hz 근처
8. vehicle_local_position에서 xy_valid / z_valid가 True
```

정상 연결 핵심 로그:

```text
INFO  [simulator_mavlink] using TCP on remote host <WSL vEthernet IP> port 4560
INFO  [simulator_mavlink] Simulator connected on TCP port 4560.
pxh>
```



---

## 부록 C. 터미널별 빠른 확인표

| 해야 할 일 | 사용하는 창 | 명령/조작 |
|---|---|---|
| WSL 설치 확인 | 터미널 1: Windows PowerShell | `wsl -l -v` |
| Ubuntu 접속 | 터미널 1 → 터미널 2 | `wsl -d Ubuntu-22.04` |
| PX4 소스 클론/빌드 | 터미널 2: WSL Ubuntu | `git clone`, `make ...` |
| X-Plane 설치 경로 확인 | 터미널 1: Windows PowerShell | `Test-Path "...\X-Plane 12"` |
| D드라이브 X-Plane 경로 확인 | 터미널 1: Windows PowerShell | `Test-Path "D:\SteamLibrary\steamapps\common\X-Plane 12"` |
| WSL IP 확인 | 터미널 1: Windows PowerShell | `ipconfig` |
| PX4_SIM_HOSTNAME 설정 | 터미널 2: WSL Ubuntu | `export PX4_SIM_HOSTNAME=<vEthernet WSL IPv4>` |
| X-Plane 실행/기체 로드 | 터미널 3: X-Plane GUI | X-Plane 실행, ALIA-250 로드 |
| px4xplane 플러그인 확인 | 터미널 3: X-Plane GUI | `Plugins → Plugin Admin` |
| PX4-XPlane SITL 실행 | 터미널 2: WSL Ubuntu | `make px4_sitl_default xplane_alia250` |
| uORB rate 확인 | 터미널 2: PX4 콘솔 `pxh>` | `uorb top sensor_combined`, `uorb top sensor_gps` |
| mavlink-router 실행 | 터미널 4: WSL Ubuntu 추가 창 | 필요할 때만 `mavlink-routerd ...` |

가장 중요한 연결 흐름은 다음입니다.

```text
터미널 3에서 X-Plane + px4xplane 준비
        ↓
터미널 1에서 vEthernet (WSL) IPv4 확인
        ↓
터미널 2에서 PX4_SIM_HOSTNAME=<확인한 IP> 설정
        ↓
터미널 2에서 make px4_sitl_default xplane_alia250 실행
        ↓
터미널 2의 pxh>에서 uorb top으로 센서 수신 확인
```

---

## 부록 D. 최종 실행 방법 요약

이 부록은 설치가 완료된 뒤, 실제로 PX4-XPlane 연동을 실행할 때 사용하는 **최종 실행 순서 요약**입니다.

권장 창 구성은 다음과 같습니다.

| 구분 | 창 이름 | 실행 위치 | 목적 |
|---|---|---|---|
| 터미널 1 | Windows / X-Plane | Windows GUI | X-Plane 12 실행, ALIA-250 로드, px4xplane 활성화 |
| 터미널 2 | Ubuntu 창 1 | WSL Ubuntu | 기존 PX4/mavlink-router 프로세스 확인 및 정리, 필요 시 mavlink-router 실행 |
| 터미널 3 | Ubuntu 창 2 | WSL Ubuntu | PX4-Autopilot-Me 실행 |
| 터미널 4 | PX4 콘솔 | `pxh>` | `uorb top`, `listener` 등 연동 상태 확인 |

주의할 점은 다음입니다.

```text
- X-Plane은 Windows에서 실행합니다.
- mavlink-router와 PX4는 WSL Ubuntu에서 실행합니다.
- PX4_SIM_HOSTNAME에는 Windows PowerShell의 ipconfig에서 확인한 vEthernet (WSL) IPv4 주소를 넣습니다.
- 아래 예시의 172.20.192.1은 사용자 PC 환경에 따라 달라질 수 있습니다.
```

---

### 1단계: 기존 프로세스 확인 및 정리

**Ubuntu 창 1**을 엽니다.

Windows PowerShell에서 Ubuntu에 접속합니다.

```powershell
wsl -d Ubuntu-22.04
```

Ubuntu 창 1에서 현재 실행 중인 PX4와 mavlink-router 프로세스를 확인합니다.

```bash
pgrep -a px4
pgrep -a mavlink-routerd
```

정상적으로 아무것도 실행 중이 아니면 출력이 없을 수 있습니다.

실행 중인 PX4 또는 mavlink-router가 있으면 먼저 종료합니다.

```bash
pkill -f px4
pkill -f mavlink-routerd
```

다시 확인합니다.

```bash
pgrep -a px4
pgrep -a mavlink-routerd
```

출력이 없으면 정리 완료입니다.

만약 `pkill -f px4` 이후에도 PX4 프로세스가 남아 있으면 마지막 수단으로 강제 종료합니다.

```bash
pkill -9 -f px4
```

강제 종료 후 다시 확인합니다.

```bash
pgrep -a px4
pgrep -a mavlink-routerd
```

판단 기준은 다음입니다.

```text
출력 없음                 → 해당 프로세스 실행 중 아님
px4 경로 또는 PID 출력     → PX4가 아직 실행 중
mavlink-routerd PID 출력  → mavlink-router가 아직 실행 중
```

---

### 2단계: X-Plane 실행

**터미널 1: Windows / X-Plane GUI**에서 진행합니다.

```text
1. X-Plane 12 실행
2. ALIA-250 기체 로드
3. Pause 해제
4. Plugins → Plugin Admin → px4xplane Enabled 확인
5. px4xplane 메뉴에서 Enable / Start / Connect 실행
```

이 단계에서는 X-Plane 창을 닫지 않습니다.  
PX4가 이후 `TCP 4560`으로 X-Plane px4xplane plugin에 접속해야 합니다.

---

### 3단계: mavlink-router 실행

**Ubuntu 창 1**에서 진행합니다.

이미 Ubuntu에 접속되어 있지 않다면 Windows PowerShell에서 다시 접속합니다.

```powershell
wsl -d Ubuntu-22.04
```

mavlink-router가 설치되어 있고 QGroundControl 또는 추가 endpoint 라우팅이 필요하면 다음처럼 실행합니다.

```bash
mavlink-routerd -e 172.20.192.1:14540 -e 172.20.192.1:14550 -e 172.20.192.1:14569 -e 127.0.0.1:14569 0.0.0.0:14550
```

여기서 `172.20.192.1`은 예시입니다.  
실제 사용 시에는 Windows PowerShell의 `ipconfig`에서 확인한 `vEthernet (WSL)` IPv4 주소로 변경합니다.

예시:

```text
vEthernet (WSL) IPv4 주소가 172.17.160.1이면
명령어의 172.20.192.1 부분을 172.17.160.1로 바꿔야 합니다.
```

예시 변경:

```bash
mavlink-routerd -e 172.17.160.1:14540 -e 172.17.160.1:14550 -e 172.17.160.1:14569 -e 127.0.0.1:14569 0.0.0.0:14550
```

mavlink-router 실행 후 다음과 비슷한 출력이 나오면 실행 중인 상태입니다.

```text
Opened UDP Client ...
Opened UDP Server ...
```

이 창은 닫지 않습니다.

참고로 mavlink-router는 PX4-XPlane의 TCP 4560 연결 자체에는 필수가 아닙니다.  
설치되어 있지 않거나 다음 오류가 나오면 우선 생략하고 PX4-XPlane 연결부터 확인합니다.

```text
E: Unable to locate package mavlink-router
mavlink-routerd: command not found
```

---

### 4단계: PX4-Autopilot-Me 실행

**Ubuntu 창 2**를 새로 엽니다.

Windows PowerShell에서 Ubuntu에 접속합니다.

```powershell
wsl -d Ubuntu-22.04
```

PX4-Autopilot-Me 폴더로 이동합니다.

```bash
cd ~/PX4-Autopilot-Me
```

주의: 아래처럼 `~/PX4-Autopilot-Me`만 입력하면 폴더 이동이 아닙니다.

```bash
~/PX4-Autopilot-Me
```

반드시 `cd`를 붙여야 합니다.

```bash
cd ~/PX4-Autopilot-Me
```

PX4가 접속할 Windows 쪽 X-Plane IP를 설정합니다.

```bash
export PX4_SIM_HOSTNAME=172.20.192.1
```

여기서도 `172.20.192.1`은 예시입니다.  
Windows PowerShell의 `ipconfig`에서 확인한 `vEthernet (WSL)` IPv4 주소로 바꿔야 합니다.

설정값을 확인합니다.

```bash
echo $PX4_SIM_HOSTNAME
```

정상 출력 예시:

```text
172.20.192.1
```

X-Plane ALIA-250 연동용 PX4 SITL을 실행합니다.

```bash
make px4_sitl_default xplane_alia250
```

정상 연결 로그 예시는 다음입니다.

```text
INFO  [init] PX4_SIM_HOSTNAME: 172.20.192.1
INFO  [simulator_mavlink] using TCP on remote host 172.20.192.1 port 4560
INFO  [simulator_mavlink] Simulator connected on TCP port 4560.
pxh>
```

`Simulator connected on TCP port 4560.`이 출력되면 PX4와 X-Plane px4xplane plugin의 기본 TCP 연결은 성공입니다.

---

### 5단계: PX4 콘솔에서 연동 상태 확인

PX4 실행 후 `pxh>` 프롬프트가 보이면 같은 Ubuntu 창 2에서 확인합니다.

IMU/HIL_SENSOR 수신률 확인:

```bash
uorb top sensor_combined
```

정상 목표:

```text
sensor_combined RATE: 40~60 Hz 근처
```

GPS/HIL_GPS 수신률 확인:

```bash
uorb top sensor_gps
```

정상 목표:

```text
sensor_gps RATE: 1~5 Hz 근처
```

EKF local position 확인:

```bash
listener vehicle_local_position
```

확인 항목:

```text
xy_valid
z_valid
v_xy_valid
v_z_valid
dead_reckoning
```

vehicle status 확인:

```bash
listener vehicle_status
```

확인 항목:

```text
pre_flight_checks_pass
failsafe
gcs_connection_lost
nav_state
arming_state
```

---

### 6단계: 전체 실행 순서만 간단히 보기

최종 실행 순서만 압축하면 다음입니다.

```text
1. Windows에서 X-Plane 12 실행
2. ALIA-250 로드
3. px4xplane Enabled / Start / Connect 확인
4. Ubuntu 창 1에서 기존 px4, mavlink-routerd 프로세스 정리
5. 필요하면 Ubuntu 창 1에서 mavlink-routerd 실행
6. Ubuntu 창 2에서 PX4-Autopilot-Me 이동
7. Ubuntu 창 2에서 PX4_SIM_HOSTNAME 설정
8. Ubuntu 창 2에서 make px4_sitl_default xplane_alia250 실행
9. pxh>에서 uorb top / listener로 센서와 EKF 상태 확인
```

복사용 최소 명령 묶음은 다음입니다.

**Ubuntu 창 1: 프로세스 정리**

```bash
pgrep -a px4
pgrep -a mavlink-routerd
pkill -f px4
pkill -f mavlink-routerd
pgrep -a px4
pgrep -a mavlink-routerd
```

필요 시 PX4 강제 종료:

```bash
pkill -9 -f px4
```

**Ubuntu 창 1: mavlink-router 실행이 필요한 경우**

```bash
mavlink-routerd -e 172.20.192.1:14540 -e 172.20.192.1:14550 -e 172.20.192.1:14569 -e 127.0.0.1:14569 0.0.0.0:14550
```

**Ubuntu 창 2: PX4 실행**

```bash
cd ~/PX4-Autopilot-Me
export PX4_SIM_HOSTNAME=172.20.192.1
echo $PX4_SIM_HOSTNAME
make px4_sitl_default xplane_alia250
```

**PX4 콘솔 pxh>: 상태 확인**

```bash
uorb top sensor_combined
uorb top sensor_gps
listener vehicle_local_position
listener vehicle_status
```

---

### 7단계: 자주 틀리는 부분

```text
문제 1: Ubuntu에서 ~/PX4-Autopilot-Me만 입력함
해결: cd ~/PX4-Autopilot-Me 로 이동해야 함
```

```text
문제 2: PX4_SIM_HOSTNAME에 VMware IP를 넣음
해결: Windows PowerShell ipconfig의 vEthernet (WSL) IPv4 주소를 넣어야 함
```

```text
문제 3: X-Plane을 나중에 실행함
해결: X-Plane + px4xplane plugin을 먼저 준비한 뒤 PX4를 실행하는 순서가 안전함
```

```text
문제 4: mavlink-router가 안 깔려서 진행을 멈춤
해결: mavlink-router는 TCP 4560 기반 PX4-XPlane 연결에는 필수가 아니므로 생략 가능
```

```text
문제 5: Ubuntu 창 1에서 mavlink-router를 실행한 뒤 같은 창에서 PX4를 실행하려고 함
해결: mavlink-router 창은 그대로 두고, Ubuntu 창 2를 새로 열어 PX4를 실행해야 함
```

