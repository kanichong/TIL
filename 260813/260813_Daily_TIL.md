ROS2의 publisher로 message를 topic하고 listener로 받는 코드를 적용해본다.
colcon build를 해보고 setup.bash로 환경변수를 등록 후에 launch를 실행.

# Lifecycle·Composition과 빌드·실행 (colcon·launch)

1. Lifecycle Node — 관리되는 생애
   - 일반 노드는 실행되는 즉시 동작을 시작합니다. 하지만 로봇 시스템에서는 "준비는 시키되 아직 동작은 시키지 않는" 상태가 필요합니다. 센서 드라이버를 켜서 하드웨어에 연결(준비)만 해두고, 모든 노드가 준비된 뒤 일제히 동작을 시작하고 싶은 것입니다.
   - Lifecycle Node는 노드의 생애를 명시적 상태로 관리합니다.
   ![[Pasted image 20260813145635.png]]
- 해석: 노드는 Unconfigured(초기) → configure() → Inactive(설정 완료, 대기) → activate() → Active(실제 동작) 순으로 전이합니다. deactivate() 로 다시 대기로, shutdown() 으로 종료로 갑니다. 핵심은 Inactive 상태 — 자원(센서 연결, 메모리)은 준비됐지만 아직 데이터를 발행/처리하지 않는 "대기" 상태입니다.

이 구조가 푸는 문제:

- **시작 순서**: 모든 노드를 configure(준비)해 둔 뒤, 준비가 확인되면 순서대로 activate. "센서가 아직 준비 안 됐는데 제어가 돌기 시작"하는 사고를 막습니다.
- **안전한 정지·재시작**: 문제가 생긴 노드를 deactivate로 멈췄다가 다시 activate. 프로세스를 죽이지 않고 동작만 제어합니다.
- **자원 관리**: configure에서 자원을 잡고, 각 상태 전이 콜백(on_configure, on_activate 등)에서 할 일을 명확히 나눕니다.

Nav2(Level 2의 자율주행 스택)가 Lifecycle Node로 구성돼 있어, 전체 주행 스택을 관리되는 상태로 일괄 기동·정지합니다.


2. Composition — 노드를 한 프로세스에 합치기
   지금까지 각 노드는 독립 프로세스였습니다. 프로세스 격리는 견고하지만 비용이 있습니다 — 노드 간 통신이 프로세스 경계를 넘나들며 데이터를 복사·직렬화합니다. 카메라 영상을 프로세스 간에 계속 복사하면 낭비가 큽니다.
   Composition은 여러 노드를 하나의 프로세스에 함께 올리는 기법입니다. 그러면 같은 프로세스 안의 노드끼리는 메모리를 직접 공유(intra-process communication)해 복사 없이 데이터를 주고받습니다 — shared_ptr가 여기서 빛을 발합니다(포인터만 넘기면 되니까).

|  | 독립 프로세스 (기본) | Composition (합성) |
| --- | --- | --- |
| 견고성 | 높음(하나 죽어도 격리) | 낮음(같이 죽음) |
| 통신 비용 | 복사·직렬화 | 메모리 공유(제로 카피) |
| 적합 | 대부분의 노드 | 대용량 데이터를 주고받는 노드 묶음 |

   즉 트레이드오프입니다. 카메라→인지처럼 큰 데이터가 오가고 성능이 중요한 노드들은 합성하고, 안전·독립성이 중요한 노드는 따로 둡니다. 1강의 "데이터는 근처에서 처리"가 프로세스 배치 수준으로 내려온 결정입니다.

3. colcon — 워크스페이스 빌드
   여러 패키지(노드·인터페이스)를 한데 모은 것이 워크스페이스이고, 이를 빌드하는 도구가 colcon입니다. CMake와 setup.py를 의존성 순서대로 호출해 주는 상위 도구입니다.
   
   #### 설치와 소싱 — `ros2` 명령이 어디서 오는가
   ROS2 를 apt 로 설치해도 **터미널은 아직 ROS2 를 모릅니다.** `source` 로 환경 변수를 채우야 합니다.
<pre><code>
sudo apt install ros-humble-desktop python3-colcon-common-extensions -y
source /opt/ros/humble/setup.bash # 이 터미널에만 적용 → ~/.bashrc 에 넣어 둔다
export ROS_DOMAIN_ID=42 # 같은 네트워킹의 남의 노드와 섞이지 않게
</code></pre>
소싱이 채우는 것은 세 개입니다 — `AMENT_PREFIX_PATH`(ROS2 가 패키지를 찾는 경로), `PYTHONPATH`(노드 모듈 import), `LD_LIBRARY_PATH`(공유 라이브러리). 내 워크스페이스를 소싱하면 설치본 위에 **오버레이**로 얹힐고, 같은 이름이면 **나중에 소싱한 쪽이 이깁니다**(`ros2 pkg prefix <패키지>` 로 확인).

패키지 골격은 

- **의존성 순서**: colcon이 패키지의 `package.xml`에 선언된 의존성을 읽어 빌드 순서를 정합니다. 인터페이스 → 그것을 쓰는 노드 순.
- **install + source**: 빌드 결과는 `install/`에 놓이고, `source install/setup.bash`로 그 경로를 환경에 등록해야 `ros2 run`이 노드를 찾습니다. "빌드는 됐는데 노드를 못 찾겠다"는 대부분 source 누락(3강).
- **증분 빌드**: CMake처럼, 바뀐 패키지만 다시 빌드합니다.

4. launch — 시스템 기동 오케스트레이션
   여러 노드·파라미터·설정을 하나의 파일로 선언해 한 번에 기동하는 것이 launch입니다. ROS2에서는 주로 Python으로 작성합니다(조건·반복 같은 로직을 넣을 수 있어서).
<pre><code>`ros2 pkg create` 가 만들어 줍니다. 신분증은 `package.xml`(의존성 선언 → colcon 뱜드 순서와 `rosdep` 의 근거), 실행 파일 등록은 Python 이면 `setup.py` 의 `entry_points`, C++ 이면 `CMakeLists.txt` 의 `install(TARGETS ...)` 입니다.

| 증상 | 원인 | 대우 |
| --- | --- | --- |
| Package 'x' not found | install/setup.bash 미소싱 또는 뱜드 실패 | 소싱 확인 → 뱜드 로그의 첫 에러부터 읽기 |
| 뱜드는 성공, ros2 run 이 실행 파일을 못 찾음 | entry_points·install(TARGETS) 누락·오타 | ros2 pkg executables <패키지> 로 등록 이름 확인 |
| 코드를 고쳐는데 예전 동작 | --symlink-install 없이 뱜드 | 옵션 붙여 재뱜드 |
| 원인 불명의 뱜드 실패가 계속됨 | 이전 뱜드 캐시 오염 | rm -rf build install log 후 새로 뱜드(최후 수단) |
![[Pasted image 20260813150940.png]]
해석: (왼쪽) 워크스페이스는 src/(내 패키지 소스), build/(중간 산출물), install/(설치 결과), log/로 구성됩니다. colcon build는 src/의 패키지들을 의존성 순서로 빌드합니다 — 예를 들어 my_interfaces를 먼저 빌드해야 그것을 쓰는 perception_pkg를 빌드할 수 있습니다. (오른쪽) 빌드 후 launch로 여러 노드를 한 번에 기동합니다.
<pre><code>
cd ~/ros2_ws
colcon build                              # 전체 빌드 (의존성 순서 자동)
colcon build --packages-select control_pkg # 특정 패키지만
source install/setup.bash                  # 실행 환경 구성 (3강의 source!)
</code></pre>
핵심 감각:

- **의존성 순서**: colcon이 패키지의 `package.xml`에 선언된 의존성을 읽어 빌드 순서를 정합니다. 인터페이스 → 그것을 쓰는 노드 순.
- **install + source**: 빌드 결과는 `install/`에 놓이고, `source install/setup.bash`로 그 경로를 환경에 등록해야 `ros2 run`이 노드를 찾습니다. "빌드는 됐는데 노드를 못 찾겠다"는 대부분 source 누락(3강).
- **증분 빌드**: CMake처럼, 바뀐 패키지만 다시 빌드합니다.

4. launch — 시스템 기동 오케스트레이션
   여러 노드·파라미터·설정을 하나의 파일로 선언해 한 번에 기동하는 것이 launch입니다. ROS2에서는 주로 Python으로 작성합니다(조건·반복 같은 로직을 넣을 수 있어서).
<pre><code>
# robot.launch.py
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(package='camera_pkg', executable='camera_node'),
        Node(package='perception_pkg', executable='perception_node',
             parameters=[{'min_confidence': 0.7}]),          # 파라미터 주입(11강)
        Node(package='control_pkg', executable='control_node',
             parameters=['config/control.yaml']),            # 파일로 주입
    ])
</code></pre>
<pre><code>
ros2 launch robot_bringup robot.launch.py
</code></pre>
launch의 가치:

- **재현 가능한 기동**: "이 로봇을 켜는 법"이 파일 하나로 문서화됩니다. 누가 켜도 같은 시스템이 뜹니다.
- **파라미터 분리**: 같은 노드를 실내용/실외용 파라미터 파일만 바꿔 실행. 코드 수정 없이(11강).
- **구성 요소화**: launch 파일이 다른 launch 파일을 포함할 수 있어, "센서 묶음", "제어 묶음"을 조합합니다.

#### 네임스페이스 — 같은 노드를 여러 번 띄우기
![[Pasted image 20260813151251.png]]
해석: 네임스페이스가 없으면 두 노드가 같은 /state 를 쓰며 부딛치고, robot1·robot2 를 주면 /robot1/state·/robot2/state 로 갈립니다. 코드는 한 벌, 이름만 갈라지는 것이 핵심입니다.

로봇이 두 대이거나 같은 종료의 센서가 둘일 때, 같은 노드를 두 번 띄우면 토픽 이름이 충돌합니다. 이때 namespace 를 주면 그 노드의 토픽·서버스 이름 앞에 접드사가 붙습니다.
<pre><code>
Node(package='sensor_pkg', executable='state_node', namespace='robot1'),
Node(package='sensor_pkg', executable='state_node', namespace='robot2'),
</code></pre>
<pre><code>
ros2 topic list
# /robot1/state
# /robot2/state        # 같은 코드인데 이름이 갈렸다
</code></pre>
이름 앞에 / 를 붙여 /global_topic 처럼 절대 이름으로 선언한 토픽은 네임스페이스가 붙지 않습니다. "이 토픽은 로봇마다 따로인가, 시스템 전체에서 하나인가"를 정해 상대 이름과 절대 이름을 고르는 것이 설계 포인트입니다.

#### 파라미터를 YAML 로 분리하기
파라미터를 코드나 launch 파일에 박아 두면 값을 바꿀 때마다 파일을 고쳐야 합니다. YAML 로 빼면 재뱜드 없이 값만 바꿔 실행할 수 있습니다.
<pre><code>
# config/params.yaml
/**:                          # 모든 노드에 적용 (노드 이름을 적으면 그 노드만)
  ros__parameters:
    publish_rate: 10.0
    warn_distance: 3.0
</code></pre>
<pre><code>
ros2 param get /state_node publish_rate # 주입된 값 확인
</code></pre>

#### C++ 패키지의 CMakeLists 선언
![[Pasted image 20260813151805.png]]
해석: 찾고(find_package) → 만들고(add_executable) → 연결하고(ament_target_dependencies) → 설치합니다(install). 마지막 단계를 버리면 빌드는 성공하는데 ros2 run 이 실행 파일을 못 찾는 상태가 됩니다.

rclcpp 노드를 빌드할 때는 CMakeLists.txt 에 네 가지가 필요합니다. 하나라도 바지면 빌드나 실행에서 막힙니다.
<pre><code>
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)                     # 1) 의존 패키지 찾기
find_package(std_msgs REQUIRED)

add_executable(state_node src/state_node.cpp)     # 2) 실행 파일 정의
ament_target_dependencies(state_node rclcpp std_msgs)   # 3) 헤더·라이브러리 연결

install(TARGETS state_node DESTINATION lib/${PROJECT_NAME})   # 4) install/ 로 설치
ament_package()
</code></pre>
ament_target_dependencies 가 include_directories + target_link_libraries 를 대신합니다. install(TARGETS ...) 를 버면 빌드는 성공하지만 ros2 run 이 실행 파일을 못 찾습니다 — Python 패키지에서 setup.py 의 entry_points 를 빼먹는 것과 같은 실수입니다.

전체 흐름 정리: .msg 정의 → colcon build(인터페이스→노드 순) → source install/setup.bash → ros2 launch(여러 노드 일괄 기동). 이것이 ROS2 프로젝트를 빌드하고 돌리는 표준 사이클입니다.
- **심화** — package.xml과 두 빌드 타입(ament_cmake vs ament_python)
  ROS2 패키지는 `package.xml`(메타데이터·의존성)을 반드시 갖고, 빌드 타입이 둘입니다.
    - **ament_python**: 순수 Python 패키지. `setup.py`(5강)로 빌드. rclpy 노드에 적합.
    - **ament_cmake**: C++ 또는 인터페이스 패키지. `CMakeLists.txt`(7강)로 빌드. rclcpp 노드·커스텀 인터페이스에 적합.
  `package.xml`의 `<depend>` 태그가 colcon의 빌드 순서와 `rosdep`(시스템 의존성 설치)의 근거가 됩니다. 즉 지금까지 배운 조각들 — CMake·setup.py·인터페이스·의존성 — 이 `package.xml`을 중심으로 colcon 아래 하나로 묶입니다.




<pre><code>
##
</code></pre>