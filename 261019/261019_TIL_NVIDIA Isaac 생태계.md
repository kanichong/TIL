


## NVIDIA Isaac 생태계

#### NVIDIA 로봇 생태계

GPU 하드웨어
RTX, Jetson, DGX
Vram의 용량이 커야 좋은 AI모델을 돌릴 수 있다.

계산과 모델 실행
CUDA, cuDNN, TensorRT

가상 환경과 로봇 개발
Omniverse, Isaac, GR00T
가상공간에서 로봇을 학습시키고 보상함수를 설정하고 움직임을 학습. 전력 문제가 관건.

#### CUDA와 GPU 계산
CUDA
NVIDIA GPU의 병렬 계산 플랫폼

로봇에서 쓰는 계산
영상 처리, 신경망, 병렬 환경

성능 확인
GPU 연산과 전체 지연을 함께 측정

#### CPU, GPU와 메모리
CPU - 순서와 분기가 많은 처리 (ROS 노드, 파일, 작업 관리)
GPU - 비슷한 연산을 대량으로 처리 (영상, 행렬, 물리 계산)
VRAM - GPU가 사용하는 메모리 (모델, 텍스처, 센서 출력)
Jetson -  메모리 CPU와 GPU가 시스템 메모리 공유 (모델과 ROS 노드가 함께 사용)
TensorRT - 학습한 모델의 추론 최적화 (지원 장치에서 모델 실행)

#### 계산 속도와 로봇 반응 속도
전체 처리 흐름
카메라 입력, 전처리, 연산, 발행

느린 단계 찾기
메모리 복사와 CPU 처리도 관찰

측정 지표
초당 처리량과 끝까지 걸리는 지연

#### Isaac Sim, Lab, ROS 비교
Isaac Sim - 가상 로봇과 센서를 시험 - 장면, 센서 데이터, 시험 기록
Isaac Lab - 학습 실험과 정책 평가 - 정책 체크포인트, 실험 로그
Isaac ROS - ROS 2에서 가속 기능 실행 - 인식, 위치, 지도 등의 메시지

#### Isaac Sim
가상 환경
로봇과 물리 법칙, 센서를 구성
시험
제어 명령에 따른 움직임을 확인
활용
주행 검증과 학습용 데이터 생성

#### 가상 로봇의 구성
형상과 관절
URDF 또는 USD 자산 준비
물리 설정
질량, 충돌, 마찰, 관절 한계
센서와 제어
카메라와 제어 명령을 연결

#### OpenUSD와 장면 구성
OpenUSD
장면과 여러 자산을 조합하는 기반
로봇 가져오기
모델을 연 뒤 물리 속성 확인
시연
카메라 위치를 바꾸고 출력 비교

#### 가상 센서와 합성 데이터
센서 출력
RGB, 깊이, 분할 등의 데이터
정답 라벨
가상 환경의 상태를 활용
활용 범위
실제 수집과 라벨링을 보완

#### Isaac Lab
학습 실험
관측, 행동, 보상과 종료 정의
학습과 평가
강화학습과 모방학습에 활용
실행 경로
3.0 계열은 여러 백엔드 지원

#### 병렬 환경에서 경험 수집
같은 과제를 여러 번
각 환경의 상태와 시작점은 다름
환경 수 증가
더 많은 경험과 더 큰 메모리 사용
강의 실습
16개 환경으로 동작부터 확인

#### Isaac Lab의 학습 흐름
![[Pasted image 20261019101717.png]]
환경과 관측·행동을 구성하고, 학습 라이브러리와 연결한다.
정책과 로그를 저장한 뒤 새로운 조건에서 평가한다.

#### ROS 2와 Isaac ROS
ROS 2
로봇 프로그램을 메시지로 연결
Isaac ROS
NVIDIA 가속 기능과 개발 도구
사용 예
cuVSLAM, AprilTag, DNN 추론

#### ROS 2의 기본 단위
Node - 기능을 수행하는 프로그램 단위 (카메라 드라이버, 위치 추정기)
Topic - 연속 메시지를 주고받는 통로 (이미지, odometry, 속도 명령)
TF - 좌표계 사이의 관계 (카메라와 로봇 기준의 변환)
시간 - 측정과 처리의 시각 (영상과 IMU의 시각 정렬)

#### VSLAM과 위치 추정
VSLAM
카메라로 지도와 위치를 함께 추정
cuVSLAM
NVIDIA의 가속 시각 위치 추정 기술
Nav2
위치와 지도, 장애물로 주행 계획

#### 위치 추정에 필요한 조건
보정과 좌표
카메라 보정값과 TF가 맞는가
영상과 시각
특징이 보이고 측정 시각이 맞는가
결과 확인
위치 추정과 주행용 지도를 구분

#### VSLAM, VLM, VLA
VSLAM - 연속 영상과 센서 정보 - 기하학적 지도와 위치 추정
VLM - 영상과 언어 - 장면 설명과 질의 응답
VLA - 영상, 지시, 로봇 상태 - 로봇 행동 또는 행동 토큰

#### VLA의 입력과 행동 출력
![[Pasted image 20261019102153.png]]
영상, 언어 지시와 로봇 상태를 행동 생성에 연결한다.
출력 규격에 맞춘 로봇별 변환과 제어기가 필요하다.

#### 로봇 행동 모델과 실제 장치
![[Pasted image 20261019102230.png]]
GR00T
로봇 행동 모델과 개발 도구
행동 표현
관절 목표 또는 말단 이동량 등
실행 준비
로봇의 관측과 제어 규격에 연결

#### Isaac을 사용하는 프로젝트
![[Pasted image 20261019102315.png]]
자율주행: Sim에서 시험하고 ROS 2와 Nav2로 주행을 연결한다.
로봇 학습: Lab에서 정책을 학습하고 실제 제어기에 연결한다.

#### 강화학습의 기본 구조
![[Pasted image 20261019102402.png]]
정책이 관측으로 행동을 고르고, 환경이 결과와 보상을 돌려준다.
경험을 이용해 장기적으로 좋은 결과를 내는 정책을 학습한다.

#### Cartpole 예제
![[Pasted image 20261019102440.png]]
관측
카트 위치·속도, 막대 각도·각속도
행동
카트를 밀어 주는 힘
목표
막대가 넘어지지 않게 균형 유지

#### 강화학습 용어
Agent / Policy - 행동을 고르는 주체와 규칙 (카트를 제어하는 신경망)
Environment - 행동의 결과가 생기는 환경 (카트와 막대 물리 계산)
Observation - 정책이 읽는 정보 (위치, 속도, 각도)
Action / Reward - 제어 입력과 평가 신호 (힘 입력과 균형 유지 보상)
Episode - 초기화부터 종료까지 한 시도 (시작 후 실패 또는 시간 종료)

#### 이동 로봇의 학습 과제 설계
관측 - 목표 방향, 거리, 속도, 장애물 거리 - 실제 로봇에서도 얻을 수 있나?
행동 - 전진 속도와 회전 속도 목표 - 속도와 가속 한계 안에 있나?
보상 - 목표 접근, 도착, 충돌 - 점수만 얻는 편법은 없나?
종료와 초기화 - 도착, 충돌, 시간 종료 / 시작점 변경 - 새 조건에도 성공하는가?

#### 보상 설계 예시
한 스텝의 보상: 2 × 목표 접근 거리 − 0.01 × 경과 시간
추가 보상: 도착 시 +10 / 충돌 시 −20
관찰할 행동: 목표 근처 진동, 시간 끌기, 센서 오차 이용

#### 학습, 추론과 평가
학습 - 경험 수집 후 정책 파라미터 갱신 - 보상, 손실, 체크포인트
추론 - 고정된 정책으로 행동 생성 - 행동과 실행 시간
평가 - 새 조건에서 반복 시험 - 성공률, 충돌, 실패 유형

#### PPO와 정책 개선
경험 수집
관측, 행동과 보상을 기록
정책 갱신
수집한 경험으로 조금씩 개선
역할 구분
Lab은 환경, 학습 라이브러리는 최적화

#### 모방학습과 시연 데이터
모방학습
시연의 관측과 행동 관계를 학습
강화학습
상호작용과 보상으로 행동 개선
조합
시연으로 시작한 뒤 추가 학습 가능

#### Sim to Real
차이가 생기는 곳
마찰, 질량, 모터 응답, 센서와 지연
도메인 랜덤화
조건을 바꾸어 다양한 경험 제공
배포 확인
단위, 순서, 행동 스케일, 제어 주기

#### 설치 환경과 버전
Jetson +microSD - JetPack 7.2.1 / Ubuntu 24.04 / Lyrical - USB 플래시와 ROS 통신
RTX PC - Isaac Sim 6.2 / Python 3.12 - 가상 로봇 실행
RTX PC 학습 - Lab 3.0 계열의 고정 커밋 - Cartpole 학습과 재생
Isaac ROS 확장 - 5.0 / 공식 Jetson 표: JP 7.2 + NVMe - SD 기본 실습 이후 선택

#### Jetson 설치 준비물
장치
공식 Orin Nano 개발 키트와 DC 전원
저장장치
microSD 64GB 이상, 강의용 128GB권장
호스트
Ubuntu 24.04 ×86_64와 데이터 USB

플래시 대상 SD의 기존 데이터는 지워진다.

#### SDK Manager 설치와 로그인
다운로드
Ubuntu x86_64용 deb 파일
설치
sudo apt install
./sdkmanager_*_amd64.deb
실행
sdkmanager 실행 후 NVIDIA 로그인
![[Pasted image 20261019104107.png]]

#### USB 연결과 복구 모드
전원을 끄고 SD와 USB를 연결한다. J14의 9번과 10번을 점퍼로 연결한다.
DC 전원을 켠 뒤 장치가 인식되면 점퍼를 제거한다. 노트북에서 lsusb 확인.
40핀 GPIO 번호와 구분한다. 다른 캐리어 보드는 해당 제조사 절차를 따른다.
![[Pasted image 20261019104056.png]]
공식 Orin Nano의 J14 위치와 복구 연결 예시

#### Jetson 대상 선택
제품군
Jetson
대상 하드웨어
실제 장비와 일치하는 Orin Nano
SDK 버전
Orin Nano용 JetPack 7.2.1 지원 항목
![[Pasted image 20261019104144.png]]
SDK Manager의 Orin Nano 선택 예시

#### Direct Flash와 설치 구성
설치 방식
Direct Flash 선택
기본 실습
Jetson Linux를 우선 설치
추가 구성
CUDA와 TensorRT는 필요할 때 추가
![[Pasted image 20261019104231.png]]
Direct Flash 공식 화면 예시. 캡처의 장치명은 선택 기준과 다를 수 있다.

#### 플래시 전 대상 확인
복구 진입
Manual Setup 선택
인식 확인
연결한 Jetson을 확인
다음 단계
대상 저장장치와 초기 계정 설정
![[Pasted image 20261019104317.png]]
Orin Nano의 플래시 준비 화면 예시

#### SD Card 플래시
OEM Configuration
Runtime: 첫 부팅에서 계정 설정
Storage Device
SD Card 선택
Flash
완료까지 USB와 DC 전원 유지
SD Card를 선택한 공식 화면 예시
![[Pasted image 20261019104334.png]]
저장장치가 SD Card인지 확인한 뒤 Flash를 시작한다.

#### 설치 완료와 첫 부팅
플래시 완료
성공 결과와 대상 장치 확인
정상 부팅
복구 점퍼 제거 확인
초기 설정
언어, 계정, 네트워크 설정
![[Pasted image 20261019104916.png]]
SDK Manager 플래시 완료 화면 예시

#### Jetson 첫 부팅 확인
. /etc/os-release
echo "$PRETTY_NAME"
dpkg --print-architecture
cat /etc/nv_tegra_release
findmnt -no SOURCE /
df -h /
Jetson 터미널에서 실행한다.
Ubuntu 24.04, arm64, R39 / REVISION 2.1을 확인한다.
루트 장치가 SD의 /dev/mmcblk… 파티션인지 확인한다.

#### Lyrical 저장소 준비
sudo apt update
sudo apt install -y curl gnupg ca-certificates \
software-properties-common
sudo add-apt-repository -y universe
export LANG=C.UTF-8
Jetson에서 네트워크와 Ubuntu Noble 환경을 확인한다.
NVIDIA는 Ubuntu 24.04용 Lyrical 패키지를 Buildfarm에서 제공한다.

#### NVIDIA 패키지 키와 저장소
repo=https://isaac.download.nvidia.com/isaac-ros
keyring=/usr/share/keyrings/nvidia-isaac-ros.gpg
arch=$(dpkg --print-architecture)
curl -fsSL "$repo/repos.key" | \
sudo gpg --dearmor --yes -o "$keyring"
echo "deb [arch=$arch signed-by=$keyring] $repo/ubuntu/main noble main" | \
sudo tee /etc/apt/sources.list.d/nvidia-isaac-ros-buildfarm.list
이 명령은 새로 설치한 Jetson을 기준으로 한다.
저장소 경로의 noble과 아키텍처 arm64를 확인한다.

#### 개발 도구용 ROS 저장소
ros_src=https://github.com/ros-infrastructure/ros-apt-source
curl -fL "$ros_src/releases/download/1.2.0/ros2-apt-source_1.2.0.noble_all.deb" \
-o /tmp/ros2-apt-source.deb
sudo dpkg -i /tmp/ros2-apt-source.deb
sudo apt update
apt-cache policy ros-lyrical-ros-base
Candidate에 설치 가능한 ros-lyrical-ros-base 버전이 보여야 한다.
(none)이면 키, 저장소, 네트워크를 먼저 확인한다.

#### ROS 2 설치와 환경 설정
sudo apt install -y ros-lyrical-ros-base \
python3-colcon-common-extensions python3-rosdep
grep -qxF 'source /opt/ros/lyrical/setup.bash' ~/.bashrc || \
echo 'source /opt/ros/lyrical/setup.bash' >> ~/.bashrc
source /opt/ros/lyrical/setup.bash
sudo rosdep init
rosdep update
echo "$ROS_DISTRO"
python3 -c "import rclpy; print('rclpy OK')"
ROS_DISTRO는 lyrical, Python 출력은 rclpy OK인지 확인한다.
rosdep이 이미 초기화되어 있으면 rosdep update부터 실행한다.

#### 두 터미널의 ROS 2 통신
터미널 1: 발행 터미널 2: 구독
source /opt/ros/lyrical/setup.bash
ros2 topic pub --rate 1 /test_topic \
std_msgs/msg/String "{data: 'hello'}"
source /opt/ros/lyrical/setup.bash
ros2 topic echo /test_topic
data: hello가 반복되면 성공. 문자열을 바꿔 보고 Ctrl+C로 종료한다.

#### Isaac Sim 실행 화면
PC 요구 사항
지원 RTX GPU와 드라이버
공식 최소 예시
RAM 32GB / VRAM 16GB / SSD
50GB
첫 실행
확장과 셰이더 준비 시간을 확보
![[Pasted image 20261019105228.png]]
Isaac Sim 기본 화면 예시
Orin Nano는 이번 강의의 Isaac Sim 실행 장치가 아니다.
































