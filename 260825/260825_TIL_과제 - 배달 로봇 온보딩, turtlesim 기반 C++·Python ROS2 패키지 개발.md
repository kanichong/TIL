



# 과제 1 - 배달 로봇 온보딩
### 원격 접속(SSH)과 센서 장치 위치 고정
**온보드 컴퓨터를 대신할 로그인 대상을 설명합니다.**

**내 서버 sshd를 설치 한다.(openssh)**
sudo apt install openssh-server

**active(running) 확인**
systemctl status ssh
**22번 포트 확인**
ss -tlnp | grep :22
**localhost로 접속**
ssh 사용자@localhost

키 페어 - 암호와 방식

**SSH키 생성**
ssh-keygen -t ed25519
**키를 해당 서버에 등록**
ssh-copy-id 사용자@아이피

**접속하지 않고 명령만 실행**
ssh 사용자@서버 'uname -a'
**파일 복사**
scp 파일 사용자@아이피:절대위치

**접속되어 있는 사용자 확인**
who
echo $SSH_CONNECTION

**시리얼 장치가 있는지 확인**
ls -l /dev/tty*

**가상 sensor 장치 생성**
mkdir -p ~/fake_sensors && cd ~/fake_sensors
**파일생성 10M 크기로**
truncate -s 10M lidar.img imu.img
**블록장치로 연결**
sudo losetup -f --show lidar.img
sudo losetup -f --show imu.img

**udev데몬의 단계별로 속성 출력**
udevadm info --attribute-walk /dev/loop(N)

**규칙파일 수정**
/etc/udev/rules.d/99-robot-sensor.rules

**규칙**
SUBSYSTEM: 장치가 속한 udev 서브시스템. 블록 장치=block, USB=tty
KERNEL: 커널이 부여한 장치 이름 패턴을 조건으로 비교합니다.
ATTR{...}: sysfs의 장치 속성을 읽어 조건으로 비교하거나 값을 설정합니다. 실제 USB 센서는 ATTRS{idVendor}, ATTRS{idProduct}, ATTRS{serial}처럼 식별에 주로 씁니다.
SYMLINK: 기존 장치는 유지하면서, 같은 장치를 가리키는 추가 심볼릭 링크.
MODE: 장치 파일의 권한을 설정합니다. 일반적으로 그룹에 읽기·쓰기 권한을 주기 위해 "0660"을 사용합니다.
GROUP: 장치 파일의 소유 그룹을 설정합니다. 해당 그룹에 사용자를 넣으면 sudo 없이 장치에 접근하게 할 수 있습니다.
== : **비교 연산자**. 장치의 현재 속성이 오른쪽 값과 일치할 때만 규칙이 적용됩니다.
= : **값 설정 연산자**. 해당 키의 값을 지정합니다. 목록 성격의 키에서는 기존 값을 대체할 수 있습니다.
+= : **추가 연산자**. 기존 값은 유지하고 새 항목을 덧붙입니다. SYMLINK에는 보통 이것을 사용합니다.

**블록장치 고정이름 SYMLINK**
/dev/robot_lidar, `/dev/robot_imu`

**udev데몬 규칙 다시 읽기**
sudo udevadm control --reload-rules

**기존 장치 이벤트 재발생**
sudo udevadm trigger --subsystem-match=tty
(--subsystem-match : 해당 장치만)

ls -l /dev/robot_*





# 과제 2 - turtlesim 기반 C++·Python ROS2 패키지 개발

**라이브러리, 툴 설치 확인**
sudo apt install ros-humble-turtlesim
g++ 11 이상, C++17, CMake 3.22 이상
Python 3.10, rclpy / rclcpp, colcon, ament_python / ament_cmake
RViz2, rqt, rosbag2, pytest 7.0 이상

## 1. C++ 빌드 체계 세우기 — g++ 다중 파일 빌드와 CMake 전환

## 2. 현대 C++로 센서 계층 구현 — RAII·다형성·STL
## 3. rclpy 노드 작성 — 거북이 상태 발행자와 구독자
## 4. rclcpp 노드 작성 — C++ 발행자와 구독자