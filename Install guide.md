# PX4 - X-Plane 설치 매뉴얼

이 매뉴얼은 Windows + WSL2 Ubuntu + PX4 + X-Plane 12 + px4xplane 플러그인을 연동하기 위한 설치 과정을 간단히 정리한 문서입니다.

---

## 1. WSL 설치 및 설정

```powershell
wsl --install -d Ubuntu-22.04
```
WSL2 Ubuntu 22.04를 설치합니다.

```powershell
wsl -l -v
```
설치된 WSL 배포판과 WSL 버전을 확인합니다.

```powershell
wsl -d Ubuntu-22.04
```
Ubuntu 22.04에 접속합니다.

```bash
sudo apt update
```
Ubuntu 패키지 목록을 갱신합니다.

```bash
sudo apt upgrade -y
```
기존 패키지를 업데이트합니다.

```bash
sudo apt install git curl wget unzip python3 python3-pip -y
```
PX4 기본 작업에 필요한 도구를 설치합니다.

---

## 2. PX4 기본 설치

```bash
cd ~
```
홈 디렉토리로 이동합니다.

```bash
git clone --recursive --branch v1.15.0 https://github.com/PX4/PX4-Autopilot.git
```
PX4 v1.15.0 고정 버전을 클론합니다.

```bash
cd ~/PX4-Autopilot
```
PX4 폴더로 이동합니다.

```bash
git submodule update --init --recursive
```
PX4 서브모듈을 동기화합니다.

```bash
git describe --tags
```
현재 PX4 버전을 확인합니다.

정상 예시:

```text
v1.15.0
```

```bash
bash ./Tools/setup/ubuntu.sh
```
PX4 Ubuntu 개발환경을 설치합니다.

```powershell
wsl --shutdown
```
설치 후 WSL을 완전히 재시작합니다.

```powershell
wsl -d Ubuntu-22.04
```
Ubuntu에 다시 접속합니다.

```bash
cd ~/PX4-Autopilot
make px4_sitl none
```
PX4 SITL 기본 빌드 및 실행을 확인합니다.

정상 예시:

```text
pxh>
```

PX4 콘솔에서 종료하려면 다음을 입력합니다.

```text
shutdown
```

---

## 3. X-Plane 12 및 px4xplane 플러그인 설치

### X-Plane 12 설치

X-Plane 12는 일반 설치판 또는 Steam 설치판 모두 사용할 수 있습니다.

Steam 설치판의 기본 경로 예시는 다음과 같습니다.

<span style="color:red"> *데모 버전의 경우 경로가 다를 수 있으나 설치 방법은 동일합니다.* </span>

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12
```

X-Plane 12 설치 폴더는 보통 아래 구조를 가집니다.

```text
X-Plane 12
├─ Aircraft
├─ Resources
│  └─ plugins
├─ X-Plane.exe
└─ Log.txt
```

### X-Plane 설치 경로 확인

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12"
```
X-Plane 12 설치 폴더가 존재하는지 확인합니다.

정상 출력:

```text
True
```

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins"
```
X-Plane 플러그인 폴더가 존재하는지 확인합니다.

정상 출력:

```text
True
```

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

### px4xplane 플러그인 설치

압축 해제 후 `px4xplane` 폴더를 아래 경로에 복사합니다.

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins
```

최종 구조는 다음과 같아야 합니다.

```text
C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12
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

### px4xplane 설치 확인

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\win.xpl"
```
px4xplane 실행 플러그인 파일이 있는지 확인합니다.

정상 출력:

```text
True
```

```powershell
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini"
```
px4xplane 설정 파일이 있는지 확인합니다.

정상 출력:

```text
True
```

### px4xplane 설정 확인

```powershell
Select-String -Path "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Resources\plugins\px4xplane\64\config.ini" -Pattern "debug_|mavlink_sensor_rate_hz|mavlink_gps_rate_hz"
```
debug 설정과 MAVLink 전송률 설정을 확인합니다.

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

### px4xplane 설정 변경

필요하면 PowerShell에서 다음을 실행합니다.

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
debug log를 끄고 MAVLink sensor/GPS rate를 안정화 값으로 낮춥니다.

설정 변경 후에는 X-Plane을 완전히 종료했다가 다시 실행해야 합니다.

### X-Plane에서 플러그인 확인

X-Plane 12 실행 후 다음 메뉴에서 확인합니다.

```text
Plugins → Plugin Admin
```

목록에 `px4xplane`이 표시되면 플러그인 설치가 완료된 것입니다.

### ALIA-250 기체 확인

```powershell
Get-ChildItem "C:\Program Files (x86)\Steam\steamapps\common\X-Plane 12\Aircraft" -Recurse -Filter "*alia*"
```
ALIA-250 기체 파일이 있는지 확인합니다.

정상 예시:

```text
BETA Technologies Alia-250
ALIA-250.acf
```

---

## 4. X-Plane 연동용 PX4 설치

```bash
cd ~
git clone --recursive -b px4xplane-sitl https://github.com/alireza787b/PX4-Autopilot-Me.git PX4-Autopilot-Me
```
X-Plane 연동용 PX4 fork를 클론합니다.

```bash
cd ~/PX4-Autopilot-Me
git submodule update --init --recursive
```
X-Plane fork 폴더로 이동하고 서브모듈을 동기화합니다.

```bash
git branch --show-current
```
현재 브랜치를 확인합니다.

정상 출력:

```text
px4xplane-sitl
```

```bash
git status
```
작업 트리 상태를 확인합니다.

정상 예시:

```text
nothing to commit, working tree clean
```

---

## 5. Windows WSL IP 확인

```powershell
ipconfig
```
Windows PowerShell에서 WSL용 가상 네트워크 IP를 확인합니다.

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

---

## 6. X-Plane 연동 실행

### X-Plane 준비

Windows에서 다음을 먼저 진행합니다.

```text
1. X-Plane 12 실행
2. ALIA-250 기체 로드
3. Pause 해제
4. Plugins → Plugin Admin → px4xplane Enabled 확인
5. px4xplane 메뉴에서 Enable / Start / Connect 실행
```

### PX4 SIM 호스트네임 설정

WSL Ubuntu에서 다음을 실행합니다.

```bash
cd ~/PX4-Autopilot-Me
export PX4_SIM_HOSTNAME=172.17.160.1
```
PX4가 Windows에서 실행 중인 X-Plane 플러그인에 접속할 IP를 설정합니다.

IP는 `ipconfig`에서 확인한 `vEthernet (WSL)`의 IPv4 주소로 변경합니다.

```bash
echo $PX4_SIM_HOSTNAME
```
설정값을 확인합니다.

정상 출력 예시:

```text
172.17.160.1
```

### X-Plane ALIA-250 연동용 PX4 SITL 실행

```bash
cd ~/PX4-Autopilot-Me
export PX4_SIM_HOSTNAME=172.17.160.1
make px4_sitl_default xplane_alia250
```
X-Plane ALIA-250 연동용 PX4 SITL을 실행합니다.

정상 로그 예시:

```text
INFO  [init] PX4_SIM_HOSTNAME: 172.17.160.1
INFO  [simulator_mavlink] using TCP on remote host 172.17.160.1 port 4560
INFO  [simulator_mavlink] Simulator connected on TCP port 4560.
pxh>
```

---

## 7. 연동 상태 확인

PX4 콘솔 `pxh>`에서 다음을 확인합니다.

```bash
uorb top sensor_combined
```
IMU/HIL_SENSOR 수신률을 확인합니다.

정상 목표:

```text
sensor_combined RATE: 40~60 Hz 근처
```

```bash
uorb top sensor_gps
```
GPS/HIL_GPS 수신률을 확인합니다.

정상 목표:

```text
sensor_gps RATE: 1~5 Hz
```

```bash
listener vehicle_local_position
```
EKF local position 생성 여부를 확인합니다.

확인할 항목:

```text
xy_valid
z_valid
v_xy_valid
v_z_valid
dead_reckoning
```

```bash
listener vehicle_status
```
preflight, failsafe, GCS 연결 상태를 확인합니다.

확인할 항목:

```text
pre_flight_checks_pass
failsafe
gcs_connection_lost
nav_state
arming_state
```

---

## 8. 참고: mavlink-router

mavlink-router는 PX4와 X-Plane TCP 연결 자체에는 필수는 아닙니다.

PX4 ↔ X-Plane 연결의 핵심은 다음입니다.

```text
simulator_mavlink → TCP 4560 → X-Plane px4xplane plugin
```

mavlink-router는 QGroundControl 또는 추가 MAVLink endpoint 라우팅이 필요할 때 별도로 사용합니다.

Ubuntu 기본 저장소에서 다음 명령이 실패할 수 있습니다.

```bash
sudo apt install mavlink-router -y
```

실패 예시:

```text
E: Unable to locate package mavlink-router
```

이 경우 우선 mavlink-router 없이 PX4-XPlane 연결부터 확인합니다.

---

## 9. 참고: sensor_gps RATE 0 문제

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

---

## 10. 참고: Position 모드 가능 조건

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

