ReAct와 Tool Use
13강 · LLM · VLM · VLA Robotics · Waffle MPX의 센서 관측으로 다음 행동 결정하기

문제 제기 · 로봇은 언제 사진을 찍어야 할까?
“가까이에 물체가 있으면 사진을 한 장 남겨 주세요.”
얼마나 가까워야 할까? 사진을 찍었다는 사실은 어떻게 확인할까?
정할 것  -  이번 실습의 답  -  이유
“가까이”의 기준  -  임계값 1.5 m  -  숫자 기준이 있어야 참/거짓이 정해짐
촬영 확인  -  센서 거리 + 저장된 파일  -  “찍었습니다”라는 모델 출력이 아니라 디스크의 파일이 증거

새로운 관측 · 계획을 바꾸는 값 (가상 사례)
작업 계획: A에 있는 상자를 B로 옮긴 뒤 충전기로 돌아감
상태 조회: 배터리 18% 관측 · 이동 허용 잔량은 20% 이상
다음 행동: 이동을 보류하고 충전기 위치를 조회 · 잔량의 출처와 관측 시각을 기록
행동으로 얻은 새 정보가 다음 선택을 바꾸는 것이 ReAct의 핵심입니다. (실습에서 실제로 읽는 센서는 라이다와 카메라)

ReAct · Reasoning + Acting (Yao 외, 2022)
Thought (생각)  -  지금 무엇을 알아야 하는지 판단
Action (행동)  -  등록된 도구 하나를 이름 · 인자와 함께 호출 (= Tool Use)
Observation (관측)  -  도구가 돌려준 결과 → 다음 Thought의 입력
모델은 센서를 읽거나 모터를 돌리지 못합니다. 도구 이름과 인자만 출력하고 실제 실행은 프로그램이 맡습니다.

한 턴 · 누가 무엇을 결정할까?
담당  -  수행하는 일  -  기록할 내용
도구 선택 정책  -  목표와 이전 관측으로 다음 호출을 고름  -  도구 이름과 인자
실행기  -  등록된 도구인지 확인하고 실행  -  호출 결과 또는 오류
다음 턴  -  관측과 남은 예산으로 계속할지 결정  -  선택 근거와 종료 사유

목표 “1.5 m 안이면 촬영”  →  read_scan 선택  →  실행기 등록 검사 · 실행  →  관측 0.86 m  →  take_photo 선택  →  photo_saved
정책은 “제안”, 실행기는 “최종 승인”. 모델이 없는 도구를 지어내도 로봇을 움직이는 문은 실행기 하나뿐입니다.

```python
TOOLS = {"read_scan": read_scan, "take_photo": take_photo}   # 등록 목록
budget = 3                                                   # 호출 예산

while True:
    if budget == 0:
        stop("budget_exhausted"); break
    call = policy.choose(goal, history)        # {"tool": ..., "args": {...}}
    if call["tool"] not in TOOLS:
        stop("unknown_tool"); break            # 실행 자체를 거부
    obs = TOOLS[call["tool"]](**call["args"])
    budget -= 1
    history.append({"call": call, "obs": obs}) # 턴별 JSON 로그
    if obs.get("status") == "timeout":
        stop("timeout"); break                 # 거리 모름 → 판단 불가
    if call["tool"] == "read_scan" and obs["nearest_m"] > THRESHOLD:
        stop("outside_threshold"); break
    if call["tool"] == "take_photo" and file_exists(obs["path"]):
        stop("photo_saved"); break
```
(개념 설명용 의사코드 · 실제 저장소 코드와 이름이 다를 수 있음)

관측의 근거 · read_scan이 반환한 첫 관측
필드  -  값  -  뜻
source  -  /scan  -  어느 ROS 2 토픽에서 왔는지 (라이다)
frame  -  base_scan  -  어느 좌표계 기준인지
stamp  -  12.000 s  -  메시지 시각 (시뮬레이션 시간)
rays  -  400  -  한 바퀴 광선 수 (간격 0.9°)
nearest_m  -  0.8593  -  유효한 거리만 골라 구한 최솟값
유효한 거리: range_min ~ range_max 안의 값만 사용 · inf, NaN 제외
질문: “센서를 확인했습니다”라는 문장만으로는? → 어느 센서를, 언제, 몇 m로, 어느 좌표계로 읽었는지 알 수 없음 → 관측은 필드가 있는 JSON으로 남김

도구 실행 조건
도구 이름  -  실행 조건  -  반환값 또는 효과
read_scan  -  유효 범위의 거리값을 수신  -  최근접 거리(m)와 시각
take_photo  -  rgb8 프레임 수신 (픽셀당 R·G·B 각 8비트)  -  지정한 경로에 RGB 사진 저장
locate_cup  -  실행기의 등록 목록에 없음  -  unknown_tool로 종료
그럴듯한 이름이어도 실행기는 비슷한 도구를 추측해 실행하지 않고 멈춤 → 환각된 도구 호출 차단
주행 도구로 확장할 때: 좌표계와 인자 범위 검사 · 같은 이동 요청은 request_id로 구별해 중복 실행 방지

값이 없을 때 vs 거리가 멀 때
구분  -  수신 시간 초과 (timeout)  -  임계값 초과 (outside_threshold)
무슨 일  -  정해진 시간 안에 새 메시지가 오지 않음  -  유효 거리가 기준보다 큼
거리를 아는가  -  모른다  -  안다
기록  -  이전 관측의 시각을 보존  -  관측값과 함께 종료 사유 기록
토론: timeout에 거리 0 m를 대입하면? → 0 m는 모든 임계값보다 작으므로 실행기가 take_photo를 선택 (주행이면 충돌 직전으로 오판)
모르는 값은 숫자로 메우지 말고 “모름”으로 남깁니다.

준비 · 한국어 요청에서 동작까지
역할  -  모델 또는 실행기  -  출력
한국어 명령 해석  -  Ollama · Qwen2.5 1.5B  -  작업 종류와 거리(m)
사진 저장 선택  -  SmolVLM2-500M · RGB와 거리 조건  -  저장 또는 생략
색상 블록 운반  -  Waffle 색상 VLA · SmolVLM2와 궤적 예측층  -  관절 · 그리퍼 목표 92개
LLM은 말 → 구조화된 명령, VLM은 그림 + 말 → 판단, VLA는 그림 + 말 → 행동
차체 전진은 Qwen이 해석한 거리를 휠 제어기에 그대로 전달, 로봇팔은 학습된 VLA가 예측한 궤적을 실행

준비 · 색상 VLA는 팔을 어떻게 움직일까?
단계  -  입력과 처리  -  출력
인코더  -  SmolVLM2에 시작 RGB와 선택 색상 지시 입력  -  시각 · 언어 특징
상태 결합  -  로봇 상태 7개를 연결 후 학습된 은닉층 통과  -  128차원 특징
궤적 예측  -  색상 중심 · 크기를 더해 회귀층에 전달  -  92개의 5축 목표
물리 실행  -  4 Hz로 관절 4개와 그리퍼 제어  -  상승 높이와 목표 안착
5축 = 관절 4 + 그리퍼 1 · 92개 ÷ 4 Hz ≈ 23초 분량의 동작
처음에 전체 궤적을 한 번에 예측 → 실행 중 다시 보며 고치지 않음 (개루프)
물체의 실제 위치는 모델 입력이 아니라 성공 판정에만 사용 · 가중치: waffle-hf-vla.pt, waffle-color-73.npz

준비 · GitHub 코드와 모델 받기
시작점: SpartaPA/waffle-mpx-lyrical 저장소의 lessons/13-react
단계  -  할 일
① LLM  -  Ollama 실행 후 ollama pull qwen2.5:1.5b
② VLA  -  Isaac Lab의 PyTorch 환경에 learning/requirements.txt 설치, 고정 리비전의 SmolVLM2 받기
③ 환경 확인  -  ROS 2 Lyrical과 Isaac Sim 6.1 준비, README의 환경 설정과 체크섬 확인 명령 실행

실습 1 · 한국어로 촬영 요청하기
터미널 A에서 README의 scene.py로 장면을 열고, 터미널 B에서 실행
```bash
source ~/waffle_mpx/src/tools/lyrical_env.bash
export ROS_DOMAIN_ID=113 ROS_LOCALHOST_ONLY=1
cd ~/waffle_mpx/src/lessons/13-react
python3 natural_command.py --output ~/photo-01
# 프롬프트: 1.5미터 안에 물체가 있으면 사진을 찍어 줘.
```
ROS_DOMAIN_ID  -  다른 로봇 · 수강생과 토픽이 섞이지 않게 나누는 번호
ROS_LOCALHOST_ONLY=1  -  통신을 이 PC 안으로 제한
출력 폴더는 실행마다 새 이름 → 이전 photo.png를 이번 결과로 착각하지 않기 위해
확인할 파일: request.json (Qwen 해석) · outcome.json (종료 사유 · 관측) · photo.png

촬영 정책의 출력 · Yes / No 로짓
실행 조건  -  실제 모델 출력 (Yes / No 로짓)  -  실행 결과
1.5 m · 관측 0.9036 m  -  8.398 / 7.887 (차 +0.511)  -  capture · RGB 저장
0.2 m · 관측 0.9033 m  -  7.938 / 8.117 (차 −0.179)  -  skip · 저장 생략
실행기 제약  -  거리 조건을 넘으면 저장 차단  -  모델 선택과 별도로 검사
로짓은 확률이 아님 (8.398을 84%로 읽으면 안 됨). 두 토큰만 비교하면:
```text
P(Yes) = 1 / (1 + e^-(Yes - No))
1.5 m 조건: 1 / (1 + e^-0.511) ≈ 0.625
0.2 m 조건: 1 / (1 + e^+0.179) ≈ 0.455
```
두 경우 모두 차이가 근소 → 작은 VLM은 숫자 비교를 확신 있게 못 함 → 실행기가 거리 조건을 다시 검사하는 이유

실습 2 · 차체를 10 cm 전진시키기
명령: 새 출력 폴더로 natural_command.py 실행 → “차체를 앞으로 10센티미터 전진해 줘.”
제어: Qwen이 drive와 0.1 m를 반환 (cm → m 변환까지 처리) → 좌우 휠 속도 제어 → 도달하면 정지
피드백: 휠 이동 환산 거리 왼쪽 0.1001 m, 오른쪽 0.1098 m
환산 = 바퀴 회전각(rad) × 반지름 (s = rθ)
오른쪽이 약 1 cm 더 굴렀음 → 차체가 약간 왼쪽으로 회전 · 미끄러짐이 있으면 실제 이동과 다를 수 있어 장면 창에서도 확인

실습 3 · 색상을 지시해 블록 집어 놓기
기본 장면을 종료하고 팔 실습 장면을 엶 · --show로 물리 실행 표시
```bash
cd ~/waffle_mpx/src/lessons/13-react
python3 natural_command.py --output ~/red-01 --show
# 프롬프트: 빨간 블록을 집어서 초록색 사각형 위에 놓아 줘.
# 다음 실행은 ~/blue-01로 바꾸고 파란 블록 요청
```
두 색을 같은 초기 배치에서 실행 → 바꾸는 변수는 색 하나 (배치가 다르면 차이의 원인을 구분할 수 없음)

빨강 · 파랑 지시 · 실제 블록의 상승과 안착
![[pdf_p15.png]]
Waffle 색상 VLA · Isaac Sim 6.1 · 파란 블록과 초록 목표 사각형, 위쪽에 그리퍼
outcome.json 필드  -  뜻  -  빨강  -  파랑
selected  -  Qwen이 고른 작업  -  pick_red  -  pick_blue
max_lift_m  -  블록이 들린 최대 높이  -  59.3 mm  -  59.4 mm
wrong_pick  -  다른 색 블록을 들었는지  -  아니오  -  아니오
stable_s  -  목표 위에서 안정될 때까지  -  2.20 s  -  1.95 s
두 결과가 거의 같음 → VLA가 색 지시에 따라 올바른 블록을 비슷한 품질로 옮김

비교 자료 · 규칙 기반 실행기의 RGB 프레임
![[pdf_p17.png]]
라이다가 0.8593 m를 관측한 뒤 전방 D435의 RGB 프레임 저장 (12.200 s · 640 × 480 · camera_color_optical_frame)
센서  -  시각  -  시야  -  좌표계
라이다  -  12.000 s  -  360° 전체  -  base_scan
카메라 D435  -  12.200 s  -  전방 일부  -  camera_color_optical_frame
라이다의 최근접 물체가 옆이나 뒤에 있을 수 있고, 0.2초 사이에 움직였을 수도 있음 → 사진만으로 최근접 물체의 신원을 확정하기 어려움

비교 자료 · 규칙 기반 실행기의 종료 사유
waffle_react.py (현재 GitHub에서는 baseline_command.py) · 정해진 문형으로 거리와 촬영 표현을 해석
실행 조건  -  실제 관측 또는 검사  -  종료 사유
거리 1.5 m  -  0.8593 m · 사진 저장 (near.ppm)  -  photo_saved
거리 0.2 m  -  0.8596 m · 촬영 생략 (far.ppm 없음)  -  outside_threshold
예산 1회  -  0.8604 m · 다음 호출 불가  -  budget_exhausted
미등록 도구  -  locate_cup 호출 거부  -  unknown_tool
발행자 없는 도메인  -  2초간 /scan 미수신  -  timeout
호출 예산: 한 요청에서 도구를 부를 수 있는 최대 횟수 → 무한 반복 호출 방지 · 예산 1회면 read_scan에 쓰고 take_photo를 못 부름
같은 장면인데 0.8593 / 0.8596 / 0.8604 m로 다름 → 실행기를 새로 시작할 때마다 측정값이 조금씩 달라짐 · 임계값 근처 값이면 결과가 바뀔 수 있음

실습 · 실패 사유 비교하기
```bash
# ① 도구 이름 변경
python3 waffle_react.py --first-tool locate_cup
# ② 발행자가 없는 도메인에서 실행
ROS_DOMAIN_ID=112 python3 waffle_react.py --timeout 2
```
①은 등록 검사에서 끝남 → 센서를 읽기 전이라 관측에 source가 없음
②는 /scan을 2초 기다린 뒤 timeout → source는 /scan이지만 값이 없음
도메인 112에 다른 /scan 발행자가 없는지 먼저 확인 (ROS_DOMAIN_ID=112 ros2 topic list)
(원본 슬라이드는 이 실습도 “실습 3”으로 표기)

심화 · 라이다가 본 물체는 카메라에도 보일까?
현재 코드는 360° 전체 광선에서 최솟값을 구함 · 전방 카메라는 그중 일부 방향만 봄
각도 = angle_min + i × angle_increment → 정면 ±15°만 선택해 어떤 도구가 선택되는지 비교
```python
import math

def nearest_front(scan, half_deg=15.0):
    lim = math.radians(half_deg)
    best = None
    for i, r in enumerate(scan.ranges):
        a = scan.angle_min + i * scan.angle_increment
        a = math.atan2(math.sin(a), math.cos(a))   # -π~π로 정규화
        if abs(a) > lim:
            continue                               # 정면 ±15° 밖 제외
        if not (scan.range_min <= r <= scan.range_max) or math.isinf(r):
            continue                               # 유효 범위 밖 제외
        best = r if best is None else min(best, r)
    return best   # None이면 정면에 유효한 물체 없음
```
400개 광선이면 ±15°에 약 33개 · 360°에서 0.86 m였던 물체가 옆에 있었다면 정면 거리는 1.5 m를 넘어 outside_threshold로 끝날 수 있음

마무리 · 어디서 멈춰야 할까?
장면  -  상황  -  종료 사유  -  근거
A · 거리 조건  -  임계값 0.2 m, 최근접 0.8596 m  -  outside_threshold  -  관측 거리가 임계값보다 큼
B · 예산 제한  -  거리 조건은 만족, 남은 호출 예산 0  -  budget_exhausted  -  촬영에 쓸 호출이 남지 않음
C · 수신 실패  -  /scan 구독, 2초 동안 메시지 없음  -  timeout  -  거리값이 없어 조건을 판정할 수 없음
셋 다 “사진 없음”이지만 대처가 다름: A는 정상 동작 · B는 예산 조정 · C는 센서 · 도메인 · 발행자 점검
제출물: 실행 조건 · 턴별 JSON 로그 · 기본 실행의 사진 · 임계값과 호출 예산이 결과에 미친 차이 설명
