
#### URDF·USD와 물리 엔진
URDF — 로봇 링크·관절 기술 — 트리, 좌표, 관성, 메시
USD — 장면과 물리 속성 기술 — RigidBody, Mass, Joint
MuJoCo — 동역학 계산 — 시험용 모델의 주행
Isaac Sim / PhysX — USD 기반 동역학 계산 — 실제 USD 로봇의 주행

#### CAD 부품을 링크로 묶는 기준
서로 고정된 부품은 같은 링크로 묶는다.
움직이는 경계에 관절을 둔다.
모든 CAD 바디가 정확히 한 링크에 속해야 한다.
base_link
  lh1_steer  → lh1_wheel
  rh1_steer  → rh1_wheel
  lh2_steer  → lh2_wheel
  rh2_steer  → rh2_wheel

#### 단위와 좌표를 먼저 통일
CAD mm를 m로 변환: 1,000으로 나누기
ROS 기준: X 전방, Y 좌측, Z 위쪽
월드 좌표 메시에서 링크 원점을 빼기
v_local = C @ v_CAD_mm / 1000 - origin_m
C = [[0, 1, 0], [-1, 0, 0], [0, 0, 1]]
[100, 200, 30] mm → [0.2, -0.1, 0.03] m

#### 관절 원점과 회전축
origin: 부모 링크에서 본 관절 위치·방향
axis: 관절 좌표계에서 본 회전 방향
거울 배치된 오른쪽 모듈의 축 부호 확인
0, +0.3, −0.3 rad에서 바퀴 중심과 방향 비교

#### 보이는 형상과 충돌 형상
visual: 화면에 보이는 메시와 색
collision: 접촉 계산용 단순 형상
바퀴는 원통, 출력 부품은 박스 근사
색은 설명용 추정 재질

실습 1 · 생성 환경 준비
```
cd code
py -3.11 -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\python lab.py build
.venv\Scripts\python lab.py check
```

URDF 링크에 들어가는 정보
```
<link name="arm_link">
  <inertial> ... 질량·질량중심·관성 ... </inertial>
  <visual><geometry> ... </geometry></visual>
  <collision><geometry> ... </geometry></collision>
</link>
```

URDF 관절 작성
```
<joint name="hinge" type="revolute">
  <parent link="base_link"/>
  <child link="arm_link"/>
  <origin xyz="0 0 0.1" rpy="0 0 0"/>
  <axis xyz="0 0 1"/>
  <limit lower="-1.5707963" upper="1.5707963"
         effort="1" velocity="1"/>
</joint>
```

실습 2 · 값 변경과 재생성
model.json의 arm 질량을 0.2→0.3 — 두 형식의 질량이 함께 변경
joint.origin의 z를 0.1→0.12 — URDF origin과 USD localPos0 변경
build 후 check 실행 — PASS_STATIC 출력
원래 값으로 복원 후 재생성 — 기준 모델과 결과 보관

USD 물리 모델의 구성
Stage  metersPerUnit=1, upAxis=Z, kg=1
/Robot  defaultPrim, ArticulationRootAPI
각 링크  RigidBodyAPI, MassAPI
충돌 도형  CollisionAPI
관절  RevoluteJoint + body0/body1 관계
장면  PhysicsScene / 바닥은 시험에 맞게 추가

URDF와 USD 관절의 대응
회전 한계 — radian — degree
연결 — parent / child — body0 / body1
관절 위치 — origin — localPos0 / localPos1
축 방향 — axis + origin rpy — axis + localRot0/1
예제 90도 — 1.5707963 — 90.0

실습 3 · 단위 오류 검출
python lab.py self-test

정상 모델: PASS_STATIC
USD 상한을 90° 대신 1.5707963으로 변경
오류 모델: angle unit mismatch 검출

정적 검사 통과 기준
구조 — 링크 9 / 구동 관절 8 / base_link
메시·수치 — 경로 존재 / NaN·Inf 없음
물성 — 총질량 4 kg / 유효 관성
형식 간 비교 — 질량·원점·축·연결 일치
메시 경계 — 좌표 차이 < 2×10⁻⁷ m
독립 FK — 0, +0.3, −0.3 rad 확인

Isaac Sim에서 실제 USD 시험
SimulationApp 초기화 후 모듈 import
.usd를 참조하여 articulation 구성
DOF 이름으로 관절 순서 매핑
원래 drive gain을 0으로 하고 effort 제어
시험마다 상태·속도·적분값 초기화

주행 합격 기준
위치 / yaw 오차 — < 0.05 m / < 5°
기울기 / 정지 속도 — < 10° / < 0.02 m/s
조향 / 바퀴 각속도 — ≤ 1.001 / ≤ 10.001 rad/s
최종 토크 — ≤ 1.000001 N·m
수치 상태 — 유한값 / 수치 불안정 없음

오류를 좁히는 순서
크기가 1,000배 차이 — mm→m / mesh scale / stage 단위
회전 방향이 반대 — joint axis / 좌우 모듈 부호
관절이 다른 부위를 움직임 — body0/1 / DOF 이름 매핑
힘 제한 초과 — 감쇠 추가 후 최종 토크 clip
기록은 정상, 순간 초과 발생 — 다운샘플 전 모든 step 최대값




































































