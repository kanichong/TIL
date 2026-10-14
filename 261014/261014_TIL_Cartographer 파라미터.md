

ROS launch: 파일 경로, 토픽 remap, use_sim_time
options: 센서 입력, 프레임, 발행 주기
TRAJECTORY_BUILDER: 로컬 정합과 submap 구성
POSE_GRAPH: 재방문 제약과 전역 최적화

Gazebo: odom 기준 base_footprint의 연속 이동을 발행한다
Cartographer: map 기준 odom의 보정을 발행한다
tracking_frame=base_footprint, published_frame=odom
provide_odom_frame=false로 Gazebo의 odom TF와 역할을 나눈다

입력 점검
ros2 topic hz /scan
ros2 topic hz /odom
ros2 topic echo /clock --once
ros2 run tf2_ros tf2_echo odom base_footprint

환경·시간·TF 오류를 먼저 해결한다
알고리즘 파라미터로 끊긴 센서나 잘못된 축을 보정하지 않는다

거리 필터
min_range=0.12m보다 가까운 측정은 제거한다
max_range=3.5m를 넘긴 측정은 실제 hit로 삽입하지 않는다
missing_data_ray_length=3m로 먼 측정의 자유 공간 갱신을 다룬다
상한을 줄이면 먼 노이즈와 유효 구조가 함께 줄 수 있다

Voxel 필터
고정 voxel_filter_size는 일정한 크기로 점을 줄인다
adaptive_voxel_filter는 점 수 기준을 보며 voxel 크기를 조절한다
loop_closure_adaptive_voxel_filter는 루프 탐색용 점군에 적용한다
서로 다른 단계의 필터를 같은 설정으로 취급하지 않는다

로컬 스캔 정합
운동 예측으로 이번 스캔의 초기 자세를 만든다
선택한 correlative 탐색으로 주변 후보를 비교한다
Ceres가 점유 구조와 초기 자세의 균형을 최적화한다
로컬 submap 내부의 변형을 전역 루프만으로 해결할 수는 없다

Motion filter 판정
마지막 채택 이후 시간·거리·각도 변화가 모두 기준 이하면 삽입을 생략한다
하나라도 기준을 초과하면 새 삽입을 허용할 수 있다
실습 각도 기준은 0.1° = 약 0.001745rad다
필터는 SLAM 계산 전체나 ROS 자세 발행을 끄는 설정이 아니다

Submap 크기
2D num_range_data 기본값은 90이다
새 submap은 N회 삽입 간격으로 만들고 약 2N회 후 완성한다
큰 submap은 구조를 넓게 담지만 로컬 오차도 더 오래 누적할 수 있다
N은 거리 m나 시간 s가 아니라 채택된 삽입 수다

점유 확률 갱신
hit_probability=0.55는 관측 끝점의 점유 증거다
miss_probability=0.49는 ray가 지나간 자유 공간의 증거다
0.5는 중립 기준이며 hit와 miss의 누적 강도를 구분한다
insert_free_space=false이면 자유 공간 갱신을 제한한다

루프 폐합
과거 submap과 현재 node의 재방문 후보를 탐색한다
점수 기준을 통과한 정합으로 제약을 추가한다
pose graph 최적화가 여러 제약의 모순을 줄인다
반복된 복도에서는 비슷한 구조의 잘못된 연결을 점검한다

전역 최적화 주기
optimize_every_n_nodes=90이 기본 주기다
더 자주 실행하면 보정 결과를 자주 받지만 풀이 호출도 증가한다
0은 주기적 최적화를 중지한다
종료 시 최종 최적화와 제약 후보 탐색은 별도로 구분한다

CPU 지연 점검
rosbag 입력률, 스캔 수, 점 수와 처리 지연을 먼저 기록한다
local 지연은 voxel·누적·온라인 탐색·Ceres 반복 수를 비교한다
global 지연은 sampling·검색 범위·최적화 주기·백그라운드 수를 비교한다
스레드만 늘리면 CPU 경쟁으로 지연이 커질 수 있다

CPU 점유율 확인 - htop














