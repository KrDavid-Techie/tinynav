# TinyNav 분석 요약

## 한 줄 요약

TinyNav는 ROS 2 Humble 위에서 동작하는 경량 지향의 비전 기반 내비게이션 스택이다. 저장소를 기준으로 보면 단순한 SLAM 데모 수준이 아니라, 스테레오 인지, 키프레임 기반 맵 구축, ESDF 기반 지역 경로 계획, 맵 재위치추정, 로봇 제어, 그리고 이를 다루는 웹/모바일 앱까지 포함한 비교적 완결된 로보틱스 플랫폼에 가깝다.

## 프로젝트가 실제로 하는 일

코드와 문서를 함께 보면 TinyNav는 다음 문제를 하나의 파이프라인으로 묶는다.

- 스테레오 카메라와 IMU로 현재 자세와 깊이를 추정한다.
- 키프레임을 축적해 맵을 만들고, 이미 구축된 맵에 현재 위치를 재정렬한다.
- 현재 위치와 주변 장애물 분포를 바탕으로 안전한 지역 궤적을 고른다.
- 선택된 경로를 실제 로봇 제어 명령으로 바꾼다.
- FastAPI + Flutter 기반 앱으로 상태 조회, 맵 확인, POI 지정, 내비게이션 제어를 수행한다.

즉, TinyNav의 핵심은 “비전 기반 인지 + 맵 기반 재위치추정 + 지역 경로 계획 + 제어 + 운영 UI”를 한 저장소 안에서 통합하려는 데 있다.

## 주요 구성 요소

| 계층 | 핵심 파일 | 역할 |
| --- | --- | --- |
| 인지 | [tinynav/core/perception_node.py](tinynav/core/perception_node.py) | 스테레오 영상과 IMU를 받아 오도메트리, 깊이, 키프레임을 생성 |
| 계획 | [tinynav/core/planning_node.py](tinynav/core/planning_node.py) | 3D 점유 그리드와 ESDF를 만들고 후보 궤적을 평가 |
| 맵/재위치추정 | [tinynav/core/map_node.py](tinynav/core/map_node.py) | 루프 폐쇄, 포즈 그래프, 맵 저장, 전역 경로 및 재위치추정 처리 |
| 데이터 저장 | [tinynav/core/build_map_node.py](tinynav/core/build_map_node.py) | TinyNavDB 기반 맵/키프레임 저장과 후처리 |
| 제어 | [tinynav/platforms/cmd_vel_control.py](tinynav/platforms/cmd_vel_control.py) | 계획된 경로를 cmd_vel로 변환 |
| 플랫폼 연동 | [tinynav/platforms/lekiwi_control.py](tinynav/platforms/lekiwi_control.py), [tinynav/platforms/unitree_control.py](tinynav/platforms/unitree_control.py), [tinynav/platforms/simulator_control.py](tinynav/platforms/simulator_control.py) | 로봇별 제어 인터페이스 |
| 운영 백엔드 | [app/backend/main.py](app/backend/main.py), [app/backend/node_manager.py](app/backend/node_manager.py) | FastAPI API, WebSocket, ROS 2 노드 생명주기 관리 |
| 운영 프론트엔드 | [app/frontend/pubspec.yaml](app/frontend/pubspec.yaml) | Flutter 기반 웹/모바일 UI |
| 실행 스크립트 | [scripts/run_navigation.sh](scripts/run_navigation.sh), [scripts/start_app.sh](scripts/start_app.sh) | 로컬/데모/앱 실행 자동화 |

## 런타임 데이터 흐름

### 1. 입력

주요 입력은 RealSense 또는 Looper 계열 스테레오 카메라 토픽과 IMU다. 문서와 코드 기준 대표 입력은 다음과 같다.

- `/camera/camera/infra1/image_rect_raw`
- `/camera/camera/infra2/image_rect_raw`
- `/camera/camera/imu`
- `/camera/camera/infra2/camera_info`

### 2. 인지 단계

[tinynav/core/perception_node.py](tinynav/core/perception_node.py)에서 입력 영상과 IMU를 동기화한다. 구현상 다음 특징이 있다.

- `ApproximateTimeSynchronizer`와 `InputAligner`를 함께 사용해 스테레오 프레임과 IMU를 맞춘다.
- TensorRT 래퍼인 `SuperPointTRT`, `LightGlueTRT`, `StereoEngineTRT`를 사용한다.
- 초기 중력 방향을 IMU 평균값으로 잡아 초기 자세를 정한다.
- 처리 결과로 `/slam/odometry`, `/slam/depth`, `/slam/disparity_vis`, `/slam/keyframe_*` 토픽을 발행한다.

이 노드는 로컬 SLAM의 출발점이자, 이후 planning/map 노드에 공급되는 핵심 데이터 생산자다.

### 3. 계획 단계

[tinynav/core/planning_node.py](tinynav/core/planning_node.py)는 깊이와 오도메트리를 받아 지역 장애물 표현과 후보 궤적을 만든다.

- `run_raycasting_loopy`가 깊이 영상을 3D 점유 그리드로 투영한다.
- 점유 정보를 바탕으로 장애물 맵과 ESDF 성격의 거리장을 계산한다.
- `generate_trajectory_library_3d`로 샘플 기반 궤적 집합을 만들고, `score_trajectories_by_ESDF`로 안전도를 평가한다.
- 결과를 `/planning/trajectory_path`, `/planning/height_map` 등으로 발행한다.

구조적으로 보면 TinyNav의 계획기는 최적화 기반 전역 planner보다는, 실시간성을 중시하는 샘플링 기반 local planner에 가깝다.

### 4. 맵 및 재위치추정 단계

[tinynav/core/map_node.py](tinynav/core/map_node.py)는 키프레임 중심의 맵 계층을 담당한다.

- `/slam/keyframe_image`, `/slam/keyframe_odom`, `/slam/keyframe_depth`를 동기화해서 입력으로 받는다.
- `SuperPointTRT`, `LightGlueTRT`, `Dinov2TRT`를 활용해 특징 추출, 매칭, 재위치추정을 수행한다.
- 저장된 맵에서 임베딩과 포즈를 불러오고, 현재 프레임을 맵에 정렬한다.
- `TinyNavDB`와 numpy 파일을 함께 사용해 맵 자산을 보관한다.
- `/mapping/current_pose_in_map`, `/map/relocalization`, `/mapping/global_plan`, `/control/target_pose` 같은 토픽을 발행한다.

이 계층 덕분에 TinyNav는 단순한 로컬 장애물 회피를 넘어, 기존 맵을 기준으로 목적지 기반 내비게이션을 시도할 수 있다.

### 5. 제어 단계

플랫폼별 제어 모듈은 [tinynav/platforms](tinynav/platforms) 아래에 분리되어 있다. 공통 흐름은 planning 결과를 받아 실제 로봇이 이해하는 명령으로 바꾸는 것이다. 이 설계는 perception/planning/map 논리를 비교적 보존한 채, 하드웨어별 제어만 바꿔 끼울 수 있게 한다.

### 6. 운영 UI 단계

앱 계층은 로봇 시스템에서 흔히 빠지는 운영 경험을 보완한다.

- [app/backend/main.py](app/backend/main.py)는 FastAPI 앱을 띄우고 라우터를 묶는다.
- [app/backend/node_manager.py](app/backend/node_manager.py)는 ROS 2 spin thread와 HTTP/WebSocket 계층 사이의 브리지를 맡는다.
- 백엔드는 `/slam/odometry`, `/mapping/current_pose_in_map`, `/planning/trajectory_path`, `/planning/height_map` 등을 구독해 프론트엔드에 전달한다.
- 프론트엔드는 [app/frontend/pubspec.yaml](app/frontend/pubspec.yaml) 기준 Riverpod, Dio, WebSocket을 사용한다.

즉, TinyNav는 “알고리즘 저장소”가 아니라 실제 운영을 염두에 둔 제품형 구조를 이미 어느 정도 갖춘 상태다.

## 기술 스택과 설계 의도

### 핵심 스택

- ROS 2 Humble
- Python 3.10
- FastAPI, Uvicorn
- Flutter
- TensorRT
- pybind11 + scikit-build-core
- NumPy, SciPy, OpenCV, Numba

[pyproject.toml](pyproject.toml)을 보면 이 프로젝트는 순수 Python 프로젝트가 아니다. Python을 주 실행 언어로 쓰되, 병목은 TensorRT와 C++ 바인딩으로 밀어내는 하이브리드 구조다.

### 성능 지향 포인트

코드 기준으로 성능 설계 포인트는 명확하다.

- 추론은 TensorRT 엔진으로 처리한다.
- 광선 추적과 수치 루프는 Numba JIT로 가속한다.
- 자세 그래프/번들 조정/일부 레이캐스팅은 C++ 바인딩을 사용한다.
- ROS 토픽 기반 분리로 병렬 실행과 모듈 단위 디버깅이 쉽다.

이 선택은 “연구용 알고리즘 검증”보다 “실시간 로봇에서 돌아가야 하는 시스템”이라는 목표에 더 가깝다.

## 운영 방식 분석

스크립트 구조를 보면 TinyNav는 단일 진입 실행 파일보다 tmux 오케스트레이션에 많이 의존한다.

- [scripts/run_navigation.sh](scripts/run_navigation.sh)는 perception, planning, map, control, RViz, POI 발행까지 한 번에 띄운다.
- [scripts/run_rosbag_examples.sh](scripts/run_rosbag_examples.sh)와 [scripts/run_rosbag_build_map.sh](scripts/run_rosbag_build_map.sh)는 데모와 맵 구축 흐름을 분리한다.
- [scripts/start_app.sh](scripts/start_app.sh)는 백엔드와 프론트엔드 정적 서빙을 함께 띄운다.

이 접근은 개발자 경험 측면에서는 빠르지만, 반대로 프로세스 관리가 셸 스크립트와 tmux에 묶여 있어 운영 자동화나 서비스화 관점에서는 다소 거칠다.

## 테스트 관점에서 본 상태

[tests](tests) 아래에는 planning, map, backend, TRT 모델, disjoint set 등에 대한 테스트가 있다. 다만 구조를 보면 다음 해석이 가능하다.

- 핵심 수학/유틸/렌더링/백엔드 보조 로직에 대한 테스트는 존재한다.
- 전체 센서 입력부터 제어 출력까지 이어지는 완전한 end-to-end 자동 검증은 제한적이다.
- GPU, ROS 2, 실제 센서/모델 파일 의존성이 강해서 CI에서 완전 재현하기는 쉽지 않다.

즉, 테스트는 “중요한 부분을 골라 막는” 형태에 가깝고, 완전한 시뮬레이션 기반 제품 검증 체계라고 보긴 어렵다.

## 이 저장소의 강점

### 1. 기능 통합도가 높다

인지, 맵, 계획, 제어, 앱을 하나의 저장소에서 이어서 볼 수 있다. 로보틱스 프로젝트에서 자주 생기는 “알고리즘은 있는데 운영 도구가 없다”는 문제가 상대적으로 적다.

### 2. 성능 최적화 전략이 현실적이다

Python만 고집하지 않고, Numba와 TensorRT, C++ 바인딩을 혼합해 병목을 분산시킨다. 실시간 시스템으로 가려는 방향이 코드에 그대로 드러난다.

### 3. 플랫폼 확장 여지가 있다

LeKiwi, Unitree, Simulator 등의 제어 경로를 분리해 두어, 로봇 본체가 달라도 상위 스택을 재사용하기 쉽다.

### 4. 맵 기반 내비게이션까지 시야에 넣고 있다

단순 obstacle avoidance가 아니라, 키프레임 DB와 재위치추정, 전역 목적지 개념까지 포함한다. 이는 시스템 완성도를 크게 끌어올리는 요소다.

### 5. 개발자 온보딩 자료가 비교적 좋다

README, docs, 실행 스크립트, app README가 갖춰져 있어 처음 따라가기가 어렵지 않은 편이다.

## 한계와 리스크

### 1. 환경 의존성이 강하다

ROS 2, TensorRT, GPU, 모델 파일, C++ 확장 빌드, 특정 Python 버전까지 맞아야 한다. 즉, 동작 환경을 맞추는 비용이 높다.

### 2. “Tiny”라는 이름과 실제 운영 복잡도 사이에 간극이 있다

핵심 노드만 보면 간결성을 지향하지만, 저장소 전체는 앱, 스크립트, 모델, 데이터베이스, 바인딩, 다양한 플랫폼 대응까지 포함해 이미 꽤 큰 시스템이다. 학습 관점에서는 좋지만, 유지보수 관점에서는 결코 작은 프로젝트가 아니다.

### 3. 실행 제어가 스크립트 중심이다

tmux 스크립트는 빠르게 띄우기 좋지만, 장애 복구, 로그 수집, 서비스 재시작 정책, 배포 일관성 같은 운영 요소는 별도 체계가 필요하다.

### 4. 하드웨어 친화적이지만 범용성은 제한된다

TensorRT와 NVIDIA 런타임 중심 설계는 Jetson/데스크톱 GPU에는 강하지만, GPU가 약한 환경이나 비-NVIDIA 환경에서는 활용성이 낮다.

### 5. 테스트 자동화 범위가 시스템 복잡도를 완전히 따라가지는 못한다

코어 테스트는 있으나 실제 로봇 운용 수준의 통합 회귀를 자동화했다고 보긴 어렵다. 기능이 늘수록 이 부분이 병목이 될 가능성이 있다.

## 종합 판단

TinyNav는 “가볍고 해킹 가능한 비전 내비게이션 스택”이라는 표방을 유지하면서도, 실제로는 꽤 야심찬 통합 로보틱스 시스템으로 확장된 상태다. 저장소 기준 가장 인상적인 점은 다음 두 가지다.

- 연구용 프로토타입처럼 보이지만 운영 UI와 관리 계층까지 포함해 실사용 시나리오를 의식하고 있다.
- Python 중심 생산성을 유지하면서도 성능 병목은 GPU와 C++로 분리해 현실적인 절충을 택하고 있다.

따라서 TinyNav는 다음과 같은 팀에 특히 적합해 보인다.

- Jetson 또는 데스크톱 GPU 환경에서 비전 내비게이션을 빠르게 실험하려는 팀
- SLAM, planning, control, 운영 UI를 한 저장소에서 함께 다루고 싶은 팀
- 완전히 범용적인 프레임워크보다는 실전형 reference stack이 필요한 팀

반대로, 설치 단순성, 하드웨어 독립성, 강한 CI 재현성을 최우선으로 두는 팀이라면 환경 구성 비용과 시스템 복잡도를 먼저 감수해야 한다.

## 참고한 주요 파일

- [README.md](README.md)
- [pyproject.toml](pyproject.toml)
- [tinynav/core/perception_node.py](tinynav/core/perception_node.py)
- [tinynav/core/planning_node.py](tinynav/core/planning_node.py)
- [tinynav/core/map_node.py](tinynav/core/map_node.py)
- [app/README.md](app/README.md)
- [app/backend/main.py](app/backend/main.py)
- [app/backend/node_manager.py](app/backend/node_manager.py)
- [app/frontend/pubspec.yaml](app/frontend/pubspec.yaml)
- [tests/test_backend.py](tests/test_backend.py)
- [scripts/run_navigation.sh](scripts/run_navigation.sh)
- [scripts/start_app.sh](scripts/start_app.sh)
