
#### 실습 PC와 버전
운영체제 — Windows 11 x64 / Python 3.12 — Ubuntu 24.04 x86_64 / Python 3.12
터미널 — PowerShell — Bash
Sim / Lab — Sim 6.1.0.0 / Lab v3.0.0-EA — Sim 6.1.0.0 / Lab v3.0.0-EA
공식 최소 예시 — RAM 32 GB / VRAM 16 GB — RAM 32 GB / VRAM 16 GB
시험 드라이버 — 595.97 — 595.58.03

#### 8 GB 실습 설정과 VM 제한
Sim 화면 — 640 x 360 / AA 끄기 — 빈 장면부터 시작
카메라 — 320 x 240 / 카메라 1개 — RGB·깊이 파일 확인
Cartpole — 환경 1개 / 작은 과제 — 안정 실행 후 환경 수 증가
Ubuntu VirtualBox — 패키지 설치·CPU 보상 예제 — VMSVGA는 NVIDIA RTX가 아님
Ubuntu GPU 실행 — NVIDIA GPU를 직접 쓰는 환경 — 네이티브 설치 또는 적합한 GPU VM

#### 실습 ZIP과 작업 폴더
Windows 11 · PowerShell
```
Set-Location C:\isaac-course\windows_lab
Get-ChildItem
Get-ChildItem .\examples
```
Ubuntu 24.04 · Bash
```
cd "$HOME/pa-course/windows_lab"
ls
ls examples
```


#### 기본 도구 준비
Windows 11 · PowerShell
```
# Python 3.12 x64와 Git을 먼저 설치
# 긴 경로 설정은 관리자 PowerShell
$key = 'HKLM:\SYSTEM\CurrentControlSet'
$key += '\Control\FileSystem'
New-ItemProperty -Path $key `
-Name LongPathsEnabled -Value 1 `
-PropertyType DWORD -Force
```
Ubuntu 24.04 · Bash
```
sudo apt update
sudo apt install -y \
git curl build-essential cmake \
python3.12-venv python3-pip
```

#### GPU와 Python 확인
Windows 11 · PowerShell
```
nvidia-smi
py -3.12 --version
git --version
```
Ubuntu 24.04 · Bash
```
nvidia-smi
python3.12 --version
git --version
free -h
df -h "$HOME"
```

#### uv 설치
Windows 11 · PowerShell
```
$url = 'https://astral.sh/uv/install.ps1'
irm $url | iex
$env:Path = "$env:USERPROFILE" + `
"\.local\bin;" + $env:Path
uv --version
```
Ubuntu 24.04 · Bash
```
curl -LsSf \
https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
uv --version
```

#### Isaac Sim 전용 Python 환경
Windows 11 · PowerShell
```
$venv = 'C:\isaac-course\simenv'
if (Test-Path $venv) {
throw '기존 폴더를 먼저 확인하세요.'
}
py -3.12 -m venv $venv
$py = "$venv\Scripts\python.exe"
& $py -m pip install --upgrade pip
```
Ubuntu 24.04 · Bash
```
VENV="$HOME/pa-course/.venv-isaacsim"
test ! -e "$VENV" || exit 1
python3.12 -m venv "$VENV"
PY="$VENV/bin/python"
"$PY" -m pip install --upgrade pip
```

#### Isaac Sim 6.1 패키지 설치
Windows 11 · PowerShell
```
$py = 'C:\isaac-course\simenv' + `
'\Scripts\python.exe'
$base = 'https://download.pytorch.org'
& $py -m pip install torch==2.11.0 `
--index-url "$base/whl/cu130"
& $py -m pip install `
'isaacsim[all,extscache]==6.1.0.0' `
--extra-index-url https://pypi.nvidia.com
& $py -m pip check
```
Ubuntu 24.04 · Bash
```
VENV="$HOME/pa-course/.venv-isaacsim"
PY="$VENV/bin/python"
BASE='https://download.pytorch.org'
"$PY" -m pip install torch==2.11.0 \
--index-url "$BASE/whl/cu130"
"$PY" -m pip install \
'isaacsim[all,extscache]==6.1.0.0' \
--extra-index-url https://pypi.nvidia.com
"$PY" -m pip check
```

#### 호환성 검사와 첫 실행
Windows 11 · PowerShell
```
$sim = 'C:\isaac-course\simenv' + `
'\Scripts\isaacsim.exe'
& $sim isaacsim.exp.compatibility_check
# 검사 창을 닫은 뒤 실행
& $sim
```
Ubuntu 24.04 · Bash
```
VENV="$HOME/pa-course/.venv-isaacsim"
SIM="$VENV/bin/isaacsim"
"$SIM" isaacsim.exp.compatibility_check
# 검사 창을 닫은 뒤 실행
"$SIM"
```

#### 실습 1 · 상자 낙하
![[Pasted image 20261020102250.png]]
입력 조건
중심 높이 2 m
질량 1 kg과 10 kg
관찰
무거운 상자가 먼저 닿는가?
마지막 중심 높이는 얼마인가?
저장
시작 장면 USD
물리 시각별 높이 CSV

#### 상자 낙하 실행
Windows 11 · PowerShell
```
Set-Location C:\isaac-course\windows_lab
$py = 'C:\isaac-course\simenv' + `
'\Scripts\python.exe'
& $py .\examples\01_drop_cubes.py
```
Ubuntu 24.04 · Bash
```
cd "$HOME/pa-course/windows_lab"
VENV="$HOME/pa-course/.venv-isaacsim"
PY="$VENV/bin/python"
"$PY" examples/01_drop_cubes.py
```

#### 높이와 중력 바꾸기
Windows 11 · PowerShell
```
# 하나씩 실행하고 종료를 기다린다.
& $py .\examples\01_drop_cubes.py `
--height 4
& $py .\examples\01_drop_cubes.py `
--gravity 1.62
```
Ubuntu 24.04 · Bash
```
# 하나씩 실행하고 종료를 기다린다.
"$PY" examples/01_drop_cubes.py \
--height 4
"$PY" examples/01_drop_cubes.py \
--gravity 1.62
```

#### 높이 CSV와 시작 장면 확인
Windows 11 · PowerShell
```
explorer .\outputs
```
Ubuntu 24.04 · Bash
```
xdg-open ./outputs
```
질문: 상자가 바닥에 있어도 중심 z가 0이 아닌 이유는?
01_drop 폴더의 heights.csv와 initial_scene.usda를 확인
CSV에서 z ≤ 0.27 m인 첫 행을 접촉의 근사 시각으로 비교한다.

#### 실습 2 · 카메라와 합성 데이터
![[Pasted image 20261020102812.png]]
입력 조건
seed에 따라 색·위치 변경
해상도 320 x 240
데이터
RGB 영상과 깊이 배열
장면 정보를 함께 기록
관찰
seed를 바꿔 비교
카메라 시점도 바꿔보기

#### 카메라 촬영 두 번 실행
Windows 11 · PowerShell
```
& $py .\examples\02_camera_capture.py `
--seed 7
& $py .\examples\02_camera_capture.py `
--seed 42
explorer .\outputs
```
Ubuntu 24.04 · Bash
```
"$PY" examples/02_camera_capture.py \
--seed 7
"$PY" examples/02_camera_capture.py \
--seed 42
xdg-open ./outputs
```

#### 카메라 출력 파일
rgb.png — 렌더링한 색 영상 — 상자·바닥이 보이는가
depth_m.npy — 깊이 원본 배열 — 단위 m, 이미지 평면까지 거리
depth_preview.png — 0~5 m 보기용 영상 — 실제 거리 분석에는 NPY 사용
scene_metadata.json — seed·색·위치·해상도 — 영상과 같은 실행 폴더인가
scene.usda — 카메라를 포함한 장면 — 시점을 바꿔 다시 확인

#### 실습 3 · 보상 계산
Windows 11 · PowerShell
```
Set-Location C:\isaac-course\windows_lab
py -3.12 .\examples\03_reward_design.py
Get-Content .\outputs\reward_examples.csv
```
Ubuntu 24.04 · Bash
```
cd "$HOME/pa-course/windows_lab"
python3.12 examples/03_reward_design.py
cat outputs/reward_examples.csv
```

#### 보상 값과 편법 행동
0.1초 동안 0.2 m 접근 — 0.399 — 계산식에서 거리 항은 얼마인가
0.1초 동안 0.2 m 후퇴 — −0.401 — 왕복하면 점수가 남는가
0.1초 동안 정지 — −0.001 — 시간 비용이 없다면 어떤가
0.2 m 접근하며 도착 — 10.399 — 도착 보상을 계속 주면 어떤가
0.2 m 접근하며 충돌 — −19.601 — 충돌 벌점이 너무 작다면?

#### 실습 4 · Cartpole 정책 학습
![[Pasted image 20261020103810.png]]
관측과 행동
수레·막대의 위치와 속도
수레에 가하는 힘
학습
RSL-RL의 PPO
환경 1개부터 실행
비교
무작위 행동
학습한 정책 재생

#### Isaac Lab 고정 버전 받기
Windows 11 · PowerShell
```
Set-Location C:\isaac-course
$repo = 'https://github.com/isaac-sim/'
$repo += 'IsaacLab.git'
git clone --depth 1 `
--branch v3.0.0-EA `
$repo IsaacLab-6.1
Set-Location .\IsaacLab-6.1
git rev-parse HEAD
```
Ubuntu 24.04 · Bash
```
cd "$HOME/pa-course"
REPO='https://github.com/isaac-sim/'
REPO+="IsaacLab.git"
git clone --depth 1 \
--branch v3.0.0-EA \
"$REPO" IsaacLab-6.1
cd IsaacLab-6.1
git rev-parse HEAD
```

#### 커밋 확인과 Lab 환경 설치
Windows 11 · PowerShell
```
$pin = 'ae37b028ea415c91ea2b'
$pin += 'c32609efcd759ed2b974'
if ((git rev-parse HEAD) -ne $pin) {
throw '수업 기준 커밋과 다릅니다.'
}
uv sync --frozen --extra isaacsim
```
Ubuntu 24.04 · Bash
```
PIN='ae37b028ea415c91ea2b'
PIN+='c32609efcd759ed2b974'
test "$(git rev-parse HEAD)" = "$PIN" \
|| exit 1
uv sync --frozen --extra isaacsim
```

#### Lab의 Sim 버전과 과제 확인
Windows 11 · PowerShell
```
Set-Location C:\isaac-course\IsaacLab-6.1
$check = 'from importlib.metadata '
$check += 'import version; '
$check += "print(version('isaacsim'))"
uv run --frozen --extra isaacsim `
python -c $check
uv run --frozen --extra isaacsim `
isaaclab list_envs --show_presets
```
Ubuntu 24.04 · Bash
```
cd "$HOME/pa-course/IsaacLab-6.1"
CHECK='from importlib.metadata '
CHECK+='import version; '
CHECK+="print(version('isaacsim'))"
uv run --frozen --extra isaacsim \
python -c "$CHECK"
uv run --frozen --extra isaacsim \
isaaclab list_envs --show_presets
```

#### Cartpole 무작위 행동
Windows 11 · PowerShell
```
uv run --frozen --extra isaacsim `
isaaclab random_agent `
--task Isaac-Cartpole `
--num_envs 1 `
physics=isaacsim_physx --viz kit
```
Ubuntu 24.04 · Bash
```
uv run --frozen --extra isaacsim \
isaaclab random_agent \
--task Isaac-Cartpole \
--num_envs 1 \
physics=isaacsim_physx --viz kit
```

#### PPO 학습 실행
Windows 11 · PowerShell
```
uv run --frozen --extra isaacsim `
isaaclab train `
--rl_library rsl_rl `
--task Isaac-Cartpole `
--num_envs 1 --seed 42 `
--max_iterations 300 `
physics=isaacsim_physx --viz kit
```
Ubuntu 24.04 · Bash
```
uv run --frozen --extra isaacsim \
isaaclab train \
--rl_library rsl_rl \
--task Isaac-Cartpole \
--num_envs 1 --seed 42 \
--max_iterations 300 \
physics=isaacsim_physx --viz kit
```

#### 학습한 정책 재생
Windows 11 · PowerShell
```
uv run --frozen --extra isaacsim `
isaaclab play `
--rl_library rsl_rl `
--task Isaac-Cartpole `
--num_envs 1 --seed 123 `
--checkpoint latest `
physics=isaacsim_physx --viz kit
```
Ubuntu 24.04 · Bash
```
uv run --frozen --extra isaacsim \
isaaclab play \
--rl_library rsl_rl \
--task Isaac-Cartpole \
--num_envs 1 --seed 123 \
--checkpoint latest \
physics=isaacsim_physx --viz kit
```

#### 화면 없이 학습
Windows 11 · PowerShell
```
uv run --frozen --extra isaacsim `
isaaclab train `
--rl_library rsl_rl `
--task Isaac-Cartpole `
--num_envs 1 --seed 42 `
--max_iterations 300 `
physics=isaacsim_physx --viz none
```
Ubuntu 24.04 · Bash
```
uv run --frozen --extra isaacsim \
isaaclab train \
--rl_library rsl_rl \
--task Isaac-Cartpole \
--num_envs 1 --seed 42 \
--max_iterations 300 \
physics=isaacsim_physx --viz none
```

#### 학습 전후 관찰 기록
균형 유지 경향 — 직접 기록 — 직접 기록
실패·초기화 — 시간 제한과 실패 구분 — 같은 관찰 범위로 비교
카트 위치 — 끝으로 치우치는지 — 끝으로 치우치는지
실행 조건 — 환경 수·seed·GPU — 갱신 횟수·체크포인트

#### 실행이 안 될 때
Sim 모듈을 찾지 못함 —  $py가 simenv Python인가 —  $PY가 Sim 전용 Python인가
줄 연결 문법 오류 — 백틱 뒤 공백이 없는가 —  역슬래시 뒤 공백이 없는가
GPU·RTX 오류 —  nvidia-smi, 지원 드라이버 —  nvidia-smi, 데스크톱 세션
Lab 과제·인자 오류 —  EA 커밋·작업 폴더 확인 —  EA 커밋·작업 폴더 확인
체크포인트 없음 —  Train 로그와 실제 파일 확인 — Train 로그와 실제 파일 확인

#### 배포 예제 자체 점검
Windows 11 · PowerShell
```
Set-Location C:\isaac-course\windows_lab
py -3.12 .\99_self_check.py
```
Ubuntu 24.04 · Bash
```
cd "$HOME/pa-course/windows_lab"
python3.12 99_self_check.py
```

#### 실습 결과 정리
물리 — heights.csv / initial_scene.usda — 높이·중력·접촉 시각
카메라 — RGB 2장 / 깊이 / 메타데이터 — seed·해상도·카메라 시점
보상 — reward_examples.csv / 답안 — 점수 계산·편법 행동 설명
강화학습 — 로그 / 체크포인트 / 관찰표 — 버전·커밋·GPU·seed·환경 수

#### 부록 · 기존 Workstation 설치 사용
Windows 11 · PowerShell
```
# 아래 경로를 실제 6.1 설치 위치로 변경
Set-Location C:\isaac-course\windows_lab
& 'C:\isaacsim\python.bat' `
.\examples\01_drop_cubes.py
```
Ubuntu 24.04 · Bash
```
# 아래 경로를 실제 6.1 설치 위치로 변경
cd "$HOME/pa-course/windows_lab"
"$HOME/isaacsim/python.sh" \
examples/01_drop_cubes.py
```






























