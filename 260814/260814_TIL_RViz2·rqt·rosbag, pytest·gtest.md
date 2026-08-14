
터틀봇의 RViz2와 Gazebo를 띄워보고 key 입력받아 움직여보자.
ros2 launch turtlebot3_bringup rviz2.launch.py
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
ros2 run turtlebot3_teleop teleop_keyboard

Gazebo를 띄웠는데 움직이지 않는다. 가끔 먹통이 되는 경우가 있다고 한다.
gazebo가 먹통인 경우 pkill gzserver; pkill gzclient


# 디버깅·시각화·테스트 (RViz2·rqt·rosbag, pytest·gtest)

### 1. 디버깅 도구 상자 — 목적별로 고르기
   문제의 성격에 따라 쓰는 도구가 다릅니다.
   ![[Pasted image 20260814101752.png]]
   - 해석: 연결 구조가 궁금하면 rqt_graph, 공간·센서를 보려면 RViz2, 수치 변화는 rqt_plot, 빠른 확인은 ros2 CLI, 문제 재현은 rosbag2, 로그 추적은 rqt_console. "무엇이 궁금한가"에 따라 도구를 고르는 것이 디버깅의 시작입니다.

   가장 자주 쓰는 CLI부터:
<pre><code>
ros2 topic list / echo / hz / info    # 토픽 존재·내용·주파수·QoS
ros2 node list / info                 # 노드와 그 연결
ros2 topic pub ...                    # 직접 메시지 주입해 테스트
rqt_graph                             # 노드-토픽 연결 시각화
</code></pre>
   "인지가 장애물을 놓친다" → ros2 topic echo /obstacles로 인지가 발행은 하는지, ros2 topic hz /scan으로 센서가 오긴 하는지, rqt_graph로 연결이 됐는지 — 데이터 흐름을 단계별로 추적하면 문제 지점이 좁혀집니다.


### 2. RViz2 — 로봇의 눈으로 3D 보기
   RViz2는 ROS2의 대표 3D 시각화 도구입니다. 추상적인 숫자 토픽을 공간 속 형상으로 보여 줍니다.
   * 센서 데이터: LiDAR 점군, 카메라 영상, 깊이 데이터를 3D 공간에 표시
   * TF 좌표계: 프레임 트리를 3D 축으로 — 각 좌표계가 로봇과 함께 움직이는 것을 눈으로 확인
   * 로봇 모델: 로봇의 형상(URDF)을 실제 자세로 렌더링
   * 경로·마커: 계획된 경로, 감지된 장애물을 시각적으로
   ![[Pasted image 20260814102743.png]]
   해석: 계산 결과를 Marker 밸시지에 담아 토픽으로 발행하고, RViz2 에서 그 토픽을 추가하면 화면에 뜼니다. 안 보일 때 원인은 대개 scale 0 · alpha 0 · Fixed Frame 불일치 셋 중 하나입니다.

   RViz2는 토픽에 있는 것만 그립니다. 그래서 "내가 계산한 결과"를 보려면 그것을 마커 토픽으로 발행해야 합니다. 표준 타입이 visualization_msgs/Marker 입니다.
<pre><code>
from visualization_msgs.msg import Marker

m = Marker()
m.header.frame_id = 'world'                  # 어느 좌표계 기준인가(14강)
m.header.stamp = self.get_clock().now().to_msg()
m.ns, m.id = 'waypoints', 0                  # 같은 ns+id 는 덮어쓰기, 다르면 따로 그림
m.type = Marker.SPHERE                       # SPHERE·CUBE·ARROW·LINE_STRIP·TEXT_VIEW_FACING
m.action = Marker.ADD
m.pose.position.x, m.pose.position.y = 2.0, 3.0
m.pose.orientation.w = 1.0                   # 회전 없음
m.scale.x = m.scale.y = m.scale.z = 0.2      # 크기 [m] — 0 이면 안 보인다
m.color.r, m.color.a = 1.0, 1.0              # a(알파) 0 이면 투명해서 안 보인다
self.marker_pub.publish(m)
</code></pre>
   RViz2 에서 **Add → By topic** 으로 그 토픽을 추가하고, 왼쪽 위 **Fixed Frame** 을 마커의 `frame_id`(위 예에서는 `world`)와 맞춰야 화면에 나타납니다. 마커가 안 보일 때 원인은 거의 셋 중 하나입니다 — **scale 이 0**, **color.a 가 0**, **Fixed Frame 불일치**. 여러 점을 한 번에 그릴 때는 `Marker.SPHERE_LIST` 나 `MarkerArray` 를 씍니다.

   RViz2가 강력한 이유는 **여러 정보를 하나의 공간에 겹쳐** 볼 수 있기 때문입니다. "LiDAR가 본 장애물"과 "인지가 검출한 장애물"과 "TF상 센서 위치"를 한 화면에 겹치면, 좌표 변환이 틀어졌는지 한눈에 보입니다. TF 디버깅에서 RViz2는 거의 필수입니다.


### 3. rosbag2 — 기록하고 재생하기
   로봇 디버깅의 가장 큰 어려움은 **재현**입니다. "야외에서 오후 3시에 딱 한 번 일어난 오작동"을 어떻게 다시 볼까요? 로봇을 다시 거기 데려갈 수도 없습니다.

   **rosbag2**가 이를 해결합니다. 토픽에 흐르는 모든 메시지를 **타임스탬프와 함께 파일로 기록**했다가, 나중에 **똑같이 재생**합니다.
   ![[Pasted image 20260814103530.png]]
   해석: (왼쪽) 실제 로봇에서 토픽을 record로 기록해 .db3 파일로 저장하고, 책상에서 play로 재생하면 로봇 없이도 그때의 데이터 흐름이 그대로 재현됩니다. 인지·제어 노드를 이 재생 위에서 돌리며 수십 번 디버깅할 수 있습니다.
<pre><code>
ros2 bag record /scan /image /tf      # 지정 토픽 기록 (-a는 전체)
ros2 bag record -a -o field_test_01   # 전체를 이름 붙여 기록
ros2 bag play field_test_01           # 재생
ros2 bag info field_test_01           # 내용 요약(토픽·기간·메시지 수)
</code></pre>
   rosbag은 디버깅을 넘어 데이터셋 구축(AI 학습용 로봇 데이터), 회귀 테스트(같은 입력에 알고리즘이 여전히 잘 동작하는지)에도 쓰입니다. "현장에서 한 번 기록, 책상에서 무한 재생"은 로봇 개발 생산성의 핵심 기법입니다.


### 4. 예외 처리와 로깅 — 노드가 죽지 않게
   ![[Pasted image 20260814103727.png]]
   해석: 콜백은 실패할 수 있는 계산만 감싸고, 아는 예외는 경고 로그와 함께 그 주기를 건너뜁니다. 로그는 레벨로 나눠 상태 바뀌는 순간만 info, 반복되는 이상은 warn 으로 묶어 쓸니다.

   노트북 코드가 죽으면 셀에 발간 글자가 뜼지만, 노드가 죽으면 로봇이 마지막 명령 상태로 남습니다. 콜백 안에서 예외가 새어 나가면 executor가 그 노드를 멈춰 세우게 됩니다. 그래서 콜백은 실패할 수 있는 최소한의 코드만 try 로 감쌉니다.
<pre><code>
def on_scan(self, msg):
    try:
        d = self.nearest(msg)                 # 실패할 수 있는 계산
    except (ValueError, ZeroDivisionError):   # 대응 방법을 아는 예외만
        self.get_logger().warn('스캔 한 프레임 건너뜀')
        return                                # 다음 주기에 다시 시도
    self.publish(d)
</code></pre>
   * 좁게 잡습니다. except Exception 으로 뭉뚱그리면 내가 만든 버그(NameError 등)까지 삼켜 원인을 못 찾습니다.
   * 복구 불가한 실패는 살려 두지 않습니다. 모터 통신이 끊겨 상태를 모르는 채 명령을 계속 내리는 것이 멈춘 로봇보다 위험합니다. 정지 명령을 보낸 뒤 종료합니다.
   * 종료 경로를 만듭니다. finally 또는 destroy_node() 직전에 정지 명령과 포트 정리를 넣어 Ctrl+C 로도 안전하게 내려오게 합니다.
   로그는 print 대신 노드 로거를 씍니다. 레벨이 붙어 rqt_console(1절)에서 필터링되고, 어느 노드에서 나온 로그인지 남습니다.
   
| 레벨    | 언제          | 로봇 예              |
| ----- | ----------- | ----------------- |
| debug | 개발 중 상세 추적  | 매 주기 중간값          |
| info  | 상태가 바뀌는 순간  | "라이다 연결됨" "목표 도달" |
| warn  | 이상하지만 계속 가능 | "스캔 3개 누락"        |
| error | 기능 하나가 실패   | "지도 저장 실패"        |
| fatal | 계속할 수 없음    | "모터 통신 단절 - 정지"   |
   모든 주기에 info 를 찍으면 하루 만에 수십만 줄이 쌓여 아무도 읽지 않습니다. 상태 변화만 info, 반복되는 이상은 횟수로 묶어 warn("최근 10초간 37개 누락")이 실무 기준입니다.


### 5. 테스트 — 문제가 생기기 전에 잡기
   디버깅이 "터진 것을 고치는" 것이라면, 테스트는 "터지지 않게 막는" 것입니다. 로봇에서는 버그가 물리적 사고이므로, 테스트의 가치가 특히 큽니다.
   ![[Pasted image 20260814105958.png]]
   해석(오른쪽 피라미드): 아래로 갈수록 빠르고 많이, 위로 갈수록 현실적이나 느림. 단위 테스트(함수·클래스)를 가장 많이 두고, 통합 테스트(노드 간 통신), 맨 위에 시뮬/실기 테스트를 둡니다. 하단일수록 CI에서 자동화하기 좋습니다.

### 단위 테스트 — pytest (Python)
   순수 로직은 ROS2 없이도 테스트할 수 있습니다.

'''python
#test_safety.py
from robot_utils.safety import compute_stop_distance

def test_stop_distance_zero_speed():
    assert compute_stop_distance(0.0) == 0.0

def test_stop_distance_increases_with_speed():
    assert compute_stop_distance(2.0) > compute_stop_distance(1.0)

def test_stop_distance_formula():
    # v=1.0, decel=1.5 → 1.0/(2*1.5)
    assert abs(compute_stop_distance(1.0) - 1/3) < 1e-6
''''

<pre><code>
pytest test_safety.py -v
</code></pre>
   좋은 테스트는 경계값(0, 음수, 최대)과 불변식(속도가 크면 정지거리도 크다)을 확인합니다. 제어·안전 로직처럼 물리와 직결된 함수일수록 촘촘히 테스트합니다.

### 단위 테스트 — gtest (C++)
   C++은 gtest를 씁니다. 구조는 동일합니다.

'''css
#include <gtest/gtest.h>
#include "robot_utils/safety.hpp"

TEST(SafetyTest, ZeroSpeed) {
    EXPECT_DOUBLE_EQ(computeStopDistance(0.0), 0.0);
}
TEST(SafetyTest, IncreasesWithSpeed) {
    EXPECT_GT(computeStopDistance(2.0), computeStopDistance(1.0));
}
'''

### CI와의 연결
   이 테스트들을 4강의 CI(GitHub Actions) 에 넣으면, PR마다 자동으로 실행됩니다. "안전 로직을 건드린 PR이 테스트를 깨면 병합 불가" — 물리적 사고로 이어질 회귀를 코드 단계에서 차단합니다.
   - **심화** — 통합 테스트와 시뮬레이션 기반 검증
    
    단위 테스트는 함수를 검증하지만, "노드 A가 발행한 것을 노드 B가 제대로 받아 처리하는가" 같은 **상호작용**은 못 잡습니다. ROS2는 `launch_test`로 여러 노드를 띄우고 토픽 흐름을 검증하는 **통합 테스트**를 지원합니다.
    
    더 위로 가면 **시뮬레이션 검증**입니다 — Gazebo(Level 2)에서 가상 로봇을 띄워 "장애물 앞에서 실제로 멈추는가"를 하드웨어 없이 확인합니다. rosbag 재생과 결합하면 "현장 데이터로 알고리즘 회귀 테스트"까지 자동화할 수 있습니다.
    
    핵심 원칙은 테스트 피라미드입니다 — **빠른 단위 테스트를 두껍게, 느린 실기 테스트를 얇게.** 모든 것을 실로봇으로 확인하려 하면 개발이 멈추고, 모든 것을 단위 테스트로만 하면 통합 문제를 놓칩니다. 계층을 나누는 것이 2강의 "빠른 것과 느린 것의 분리"와 같은 사고입니다.




a




<pre><code>
</code></pre>

<pre><code>
</code></pre>
<pre><code>
</code></pre>