
PC와 Jetson의 일을 먼저 나눈다
시뮬레이션과 학습은 PC에서, 로봇의 판단에 필요한 추론은 Jetson에서 합니다.

PC / RTX Sim 6.1 · Lab 3.0  →  모델 패키지 ONNX + 설정 + 해시  →  Jetson / Arm64 TensorRT / ROS2  →  MCU / 구동기 주기 제어 · 타임아웃

이번 강의의 버전 고정표
구성  -  강의 기준  -  확인할 점
Jetson 호스트  -  JetPack 7.2 / Ubuntu 24.04  -  Orin 대상 BSP·플래시 구성
ROS 2 / Isaac ROS  -  Lyrical / release-5.0  -  NVIDIA Noble 빌드팜 경로
PC 시뮬레이터  -  Isaac Sim 6.1  -  ROS bridge 지원 조합 별도 확인
Isaac Lab  -  v3.0.0-EA  -  Sim 6.1 대응 태그; main은 6.2
TensorRT / CUDA / ORT  -  설치된 정확한 버전 기록  -  TRT 10.x와 11.x 명령 분리
같은 제품 이름이어도 태그·CPU 아키텍처·드라이버가 다르면 재현되지 않습니다.

모델·파일 형식·실행기의 구분
이름  -  무엇인가  -  이번 실습에서의 역할
PyTorch / TensorFlow  -  모델을 정의·학습하는 프레임워크  -  학습된 모델과 기준 출력
ONNX  -  Open Neural Network Exchange  -  연산 그래프·가중치 교환 형식
ONNX Runtime  -  ONNX 등을 실행하는 런타임  -  CPU·CUDA·TensorRT EP 실행
TensorRT  -  NVIDIA 추론 최적화 SDK  -  대상 GPU용 엔진 빌드와 실행
TensorRT 엔진  -  최적화된 실행 계획  -  Jetson에서 로드할 .plan 파일
학습, 파일 저장, 최적화, 실행을 서로 다른 단계로 구분합니다.

우리가 ONNX를 사용하는 이유
문제  -  ONNX로 해결하는 부분
학습 PC와 Jetson의 환경이 다름  -  학습된 계산 그래프·가중치를 공통 형식으로 전달
PyTorch 모델을 GPU 엔진으로 바꾸고 싶음  -  TensorRT Parser가 읽는 배포 입력으로 사용
변환 후 결과가 달라졌는지 확인해야 함  -  같은 입력으로 원본·ORT 출력 비교
우리 퍽 인식에 적용  -  YOLO best.pt를 ONNX로 내보내고 클래스·전처리를 함께 저장
ONNX는 모델 전달 형식입니다. 변환 후 정확도 검사가 필요하며, GPU 속도 향상을 자동 보장하지 않습니다.

ONNX 파일 내부의 구성
구성 요소  -  저장하는 내용  -  확인할 이유
ModelProto  -  IR 버전·opset·생성자·그래프  -  읽는 도구와의 호환성
GraphProto  -  입력·출력·노드·initializer  -  계산의 연결 구조
NodeProto  -  Conv·MatMul·Add 등 연산과 속성  -  지원 연산인지 확인
TensorProto  -  가중치·편향 등 텐서 데이터  -  dtype·shape·값 확인
ValueInfoProto  -  입출력 이름·자료형·차원  -  런타임 입력 계약
노드는 연산, 연결선은 텐서가 전달되는 관계를 표현합니다.

ONNX는 모델의 교환용 설계도
학습 모델 가중치 + 연산  →  ONNX 그래프 입력/출력 계약  →  ONNX Runtime 실행 제공자 EP  →  추론 결과 후처리 포함 비교

파일 변환 성공과 같은 결과를 내는 것은 별개의 검사입니다.

ONNX 그래프 읽기: 선형 회귀
![[pdf_p09.png]]
Y = X × A + B에서 입력, 행렬 곱, 덧셈, 출력을 찾습니다.

가중치와 입력의 구분
![[pdf_p10.png]]
학습으로 정한 A·B와 매 프레임 바뀌는 X를 구분합니다.

입출력 계약을 표로 고정한다
항목  -  YOLO 퍽 검출 예  -  잘못 바꾸면
입력  -  images / 1×3×640×640  -  shape 불일치
dtype / 채널  -  float32 / RGB / NCHW  -  색상·축 오인식
전처리  -  letterbox + 0‒1 정규화  -  박스 좌표 편향
출력  -  실제 ONNX 메타데이터로 확인  -  디코더 차원 오류
후처리  -  conf·IoU·NMS·좌표 복원 고정  -  mAP 비교 불공정
내보낸 그래프의 메타데이터를 확인한 뒤 ROS 디코더를 연결합니다.

내 ONNX의 구조를 직접 출력하기
```python
import onnx
m = onnx.load("model.onnx")
onnx.checker.check_model(m)
print("IR", m.ir_version)
print("opsets", [(x.domain, x.version) for x in m.opset_import])
print("inputs", [x.name for x in m.graph.input])
print("weights", len(m.graph.initializer))
for n in m.graph.node[:10]:
    print(n.op_type, list(n.input), list(n.output))
```
구조 검사를 통과한 뒤 같은 입력으로 원본과 출력값을 비교합니다.

연산 융합은 메모리 왕복을 줄인다
Conv 중간 결과 기록  →  BatchNorm 다시 읽고 기록  →  ReLU 다시 읽고 기록  →  융합 커널 중간 왕복 감소

커널 수가 줄어도 전체 지연이 얼마나 줄었는지는 측정해야 합니다.

ONNX Runtime과 실행 제공자
![[pdf_p14.png]]
하나의 런타임 API 아래에서 실행 제공자가 하드웨어별 계산을 담당합니다.

ONNX Runtime이 필요한 시점
필요  -  우리 실습에서 하는 일
ONNX가 실제로 실행되는지 확인  -  CPU EP로 먼저 입력·출력 오류 분리
원본 모델과 변환 모델 비교  -  같은 퍽 이미지로 수치·검출 결과 정합 확인
GPU 실행과 fallback 분석  -  CUDA·TensorRT EP별 노드 배치와 지연 측정
사용 판단  -  직접 TensorRT만 배포하더라도 ORT는 변환 검증 기준으로 활용
ONNX Runtime은 ONNX를 실행하는 도구입니다. TensorRT 엔진과 비교할 기준선을 만드는 데 사용합니다.

EP 경계에서 복사가 생길 수 있다
CUDA EP 지원 연산  →  CPU fallback 미지원 연산  →  CUDA EP 다음 연산  →  프로파일 확인 복사 + 대기

GPU가 목록에 있다는 사실만으로 모든 연산이 GPU에서 실행되지는 않습니다.

그래프 분할과 CPU fallback
![[pdf_p17.png]]
지원되는 부분 그래프가 어느 EP에 배정되는지 확인합니다.

ORT의 기준선을 만든다
```python
import onnxruntime as ort
so = ort.SessionOptions()
so.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL
so.enable_profiling = True
sess = ort.InferenceSession("model.onnx", so,
    providers=["CUDAExecutionProvider", "CPUExecutionProvider"])
print(sess.get_inputs()[0].name, sess.get_inputs()[0].shape)
print(sess.get_providers())
# 동일 입력 x로 warm-up → 반복 측정 → end_profiling()
```
원본·ORT CPU·ORT CUDA를 같은 입력과 같은 후처리로 비교합니다.

최적화 전후의 구간별 지연 (ms)
![[pdf_p19.png]]
교육용 예시(Jetson 실측 아님): 추론만 줄이면 전체 이득은 제한됩니다.

정합을 통과해야 다음 단계로 간다
검사  -  방법  -  완료 기준
구조  -  onnx.checker + 입출력 메타데이터  -  파싱 성공·계약 일치
수치  -  동일 입력의 abs/rel 오차  -  모델별 허용 오차 사전 정의
과제  -  같은 test 세트 mAP/성공률  -  기준 성능 유지
실행  -  EP 프로파일·복사 구간  -  원치 않는 fallback 설명
제출  -  ONNX·해시·환경·로그  -  다른 사람이 재실행 가능
퀴즈: 출력은 비슷한데 박스 위치가 어긋난다면 letterbox 복원을 먼저 확인합니다.

TensorRT는 대상 GPU용 실행 계획을 만든다
ONNX  →  그래프 최적화  →  tactic 탐색 작업 공간 선택  →  엔진 .plan 대상에서 실행

PC에서 만든 엔진을 Jetson에 그대로 복사하는 것을 기본 배포 경로로 삼지 않습니다.

TensorRT의 빌더와 런타임
구분  -  빌드할 때  -  매 추론 시 실행할 때
목적  -  대상 GPU용 실행 계획 생성  -  입력 데이터로 예측 계산
입력  -  ONNX·shape 범위·빌드 설정  -  저장된 엔진·입력 텐서
주요 객체  -  Parser / Network / BuilderConfig  -  Runtime / Engine / Context
주요 작업  -  융합·tactic 탐색·메모리 계획  -  버퍼 연결·GPU 작업 제출
결과  -  직렬화 엔진 .plan 또는 .engine  -  출력 텐서와 후처리 결과
빌드는 배포 준비 단계에서 수행하고, 엔진은 프로세스 시작 시 로드해 재사용합니다.

우리 로봇에 TensorRT가 필요한 이유
현장의 문제  -  TensorRT를 쓰는 이유
카메라 프레임마다 추론이 필요  -  대상 GPU에 맞춘 커널·연산 융합으로 추론 비용 절감
8GB 메모리를 여러 노드가 공유  -  지원 정밀도와 실행 메모리를 비교해 배포 구성 선택
늦은 인식 결과로 집기 위치가 어긋남  -  실측 지연을 줄여 결과의 신선도 확보
채택 조건  -  같은 평가 세트의 정확도·중심 오차를 유지하며 지연 목표 달성
TensorRT는 최적화 수단입니다. 실제 Jetson에서 전처리·후처리를 포함한 효과를 측정한 뒤 채택합니다.

프레임워크에서 TensorRT까지
![[pdf_p24.png]]
ONNX와 직렬화 엔진 사이에서 GPU에 맞춘 최적화가 일어납니다.

엔진이 만들어지는 내부 단계
순서  -  객체/과정  -  설계할 항목
1  -  ONNX Parser  -  연산 지원·입출력 이름 확인
2  -  NetworkDefinition  -  텐서 연결·dtype·동적 차원
3  -  BuilderConfig  -  workspace 상한·profile·플러그인
4  -  그래프 최적화·tactic 탐색  -  레이어 융합·실제 GPU 커널 선택
5  -  직렬화 엔진 저장  -  GPU·TRT·CUDA·모델 해시 기록
tactic은 같은 계산을 수행하는 여러 구현 중 선택한 실행 방식입니다.

카메라 한 프레임의 실행 단계
순서  -  TensorRT / CUDA 동작  -  애플리케이션의 책임
시작 시  -  Runtime이 엔진 역직렬화  -  엔진 호환·해시 검사
시작 시  -  Engine에서 Context 생성  -  버퍼·CUDA stream 준비
프레임마다  -  입력 shape / tensor address 설정  -  전처리·데이터 수명 유지
프레임마다  -  execute_async_v3 / enqueueV3  -  올바른 stream에 실행 제출
완료 후  -  출력 텐서 사용  -  완료 동기화·후처리·결과 age 검사
GPU에 작업을 제출한 시점과 GPU가 계산을 마친 시점은 다릅니다.
