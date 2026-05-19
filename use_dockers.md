# TinyNav Docker 실행 가이드

이 저장소는 Docker로 실행할 수 있지만, 실제로는 일반적인 웹앱 컨테이너보다 요구사항이 훨씬 강합니다. [Dockerfile](Dockerfile)은 ROS 2 Humble, TensorRT, Gazebo, RealSense, Flutter web build까지 한 이미지에 넣고 있고, [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json)은 그 이미지를 GPU, host network, X11, /dev 마운트와 함께 실행하도록 구성합니다.

결론부터 말하면 가장 안전한 경로는 다음 두 가지입니다.

1. Linux + NVIDIA 또는 Jetson 환경에서 공개 이미지 `uniflexai/tinynav:latest`를 Dev Container로 실행한다.
2. 같은 공개 이미지를 `docker run`으로 직접 띄운 뒤, 컨테이너 안에서 [scripts/run_rosbag_examples.sh](scripts/run_rosbag_examples.sh) 같은 스크립트를 실행한다.

## 지원 환경 요약

| 호스트 환경 | 권장 여부 | 이유 |
|---|---|---|
| Ubuntu x86_64 + NVIDIA GPU | 권장 | `--gpus all`, X11, `--network host`, TensorRT 요구사항과 가장 잘 맞음 |
| Jetson Orin + JetPack 6.2+ | 권장 | README와 Dockerfile이 arm64 경로를 직접 고려함 |
| macOS + Docker Desktop | 비권장 | NVIDIA runtime, 실제 `/dev` 접근, Linux 방식의 `--network host`, `/tmp/.X11-unix` 전제가 맞지 않음 |

맥에서 이 프로젝트를 실제로 돌리려면, 맥 안에서 Docker Desktop으로 직접 실행하기보다 Ubuntu/NVIDIA PC 또는 Jetson에 원격 접속해서 그 Linux 호스트에서 아래 절차를 수행하는 것이 맞습니다. 특히 이 프로젝트는 TensorRT와 ROS 2 DDS 통신을 전제로 하므로, macOS 로컬 Docker는 개발 문서 확인 정도를 제외하면 실사용 경로가 아닙니다.

## 이 프로젝트에서 Docker가 실제로 어떻게 쓰이는가

- [README.md](README.md)는 빠른 시작 경로로 Dev Container를 안내합니다.
- [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json)은 공개 이미지 `uniflexai/tinynav:latest`를 사용합니다.
- 같은 설정에서 다음 런타임 옵션을 강하게 전제합니다.
	- `--gpus all`
	- `--privileged`
	- `--network host`
	- `/tmp/.X11-unix`, `/dev`, `/etc/localtime` 마운트
	- `--shm-size=16gb`
	- 워크스페이스를 `/tinynav`로 bind mount
- [Dockerfile](Dockerfile)은 CI에서 멀티 아키텍처 이미지 빌드에 사용됩니다.
- [scripts/start_app.sh](scripts/start_app.sh)는 FastAPI 백엔드와 Flutter web 정적 파일 서버를 같은 컨테이너 안에서 tmux로 올립니다.

즉, 이 프로젝트의 Docker 실행은 보통 "컨테이너 한 번 띄우고 내부에서 ROS/앱 스크립트를 실행하는 개발 런타임"에 가깝습니다.

## 가장 권장하는 방법: Dev Container

### 1. Linux 호스트 준비

필수 조건:

- Docker
- Git LFS
- x86_64라면 NVIDIA Container Toolkit
- Jetson이라면 JetPack 6.2 이상

Linux 호스트에서는 먼저 환경 점검 스크립트를 돌리는 것이 좋습니다.

```bash
git clone https://github.com/UniflexAI/tinynav.git
cd tinynav
git lfs install
git lfs pull
bash scripts/check_env.sh
```

[scripts/check_env.sh](scripts/check_env.sh)은 Docker, docker group, NVIDIA runtime, Git LFS를 확인합니다.

### 2. VS Code에서 컨테이너 열기

가장 단순한 흐름은 README와 동일합니다.

CLI를 쓸 계획이면 먼저 설치합니다.

```bash
npm install -g @devcontainers/cli
```

```bash
devcontainer up --workspace-folder .
devcontainer exec --workspace-folder . bash
```

VS Code GUI를 쓴다면 저장소를 연 뒤 Dev Containers 확장으로 다시 열면 됩니다.

### 3. 컨테이너 안에서 예제 실행

```bash
bash /tinynav/scripts/run_rosbag_examples.sh
```

이 스크립트는 다음을 한 번에 띄웁니다.

- `perception_node.py`
- `planning_node.py`
- Hugging Face에서 예제 rosbag 다운로드 후 재생
- RViz

첫 실행에서 `/tinynav/tinynav/models` 아래에 `.plan` 파일이 없으면, 이미지의 entrypoint가 TensorRT 엔진 생성을 물어봅니다. 이 단계는 플랫폼별 최적화 과정이라 정상입니다.

### 4. 중국 네트워크 환경

[.devcontainer/devcontainer_cn.json](.devcontainer/devcontainer_cn.json)에 Hugging Face와 pip 미러 설정이 따로 들어 있습니다. 또한 이미지 이름도 `docker.1ms.run/uniflexai/tinynav:latest`로 바뀌어 있습니다. 미러가 필요한 환경이면 그 파일을 기준으로 같은 값을 적용하면 됩니다.

## Docker CLI로 직접 실행하는 방법

Dev Container를 쓰지 않아도, [Dockerfile](Dockerfile)과 [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json)에 맞춰 비슷한 옵션으로 실행하면 됩니다.

### 1. 공개 이미지 받기

```bash
docker pull uniflexai/tinynav:latest
```

이 이미지는 CI와 Dev Container가 공통으로 쓰는 기본 이미지입니다.

### 2. GUI가 필요하면 X11 허용

RViz를 띄울 Linux X11 환경이라면 호스트에서 한 번 허용해야 합니다.

```bash
xhost +local:root
```

RViz를 쓰지 않을 거라면 아래 `docker run`에서 `DISPLAY`와 `/tmp/.X11-unix` 관련 옵션은 빼도 됩니다.

### 3. 컨테이너 실행

```bash
docker run --rm -it \
	--name tinynav-dev \
	--gpus all \
	--privileged \
	--network host \
	-e DISPLAY=$DISPLAY \
	-e GDK_SCALE=2 \
	-v /tmp/.X11-unix:/tmp/.X11-unix \
	-v /dev:/dev \
	-v /etc/localtime:/etc/localtime:ro \
	--device-cgroup-rule='c 81:* rwm' \
	--device-cgroup-rule='c 234:* rwm' \
	--shm-size=16gb \
	-v "$HOME/.local/share/tinynav:/root/.local/share/tinynav" \
	-v "$PWD:/tinynav" \
	-w /tinynav \
	uniflexai/tinynav:latest \
	bash
```

이 명령은 Dev Container 설정을 거의 그대로 옮긴 것입니다.

- `--network host`: ROS 2 DDS discovery와 로컬 툴 연결을 단순하게 유지
- `-v /dev:/dev`, `--privileged`: RealSense 같은 장치 접근
- `-v "$PWD:/tinynav"`: 현재 저장소를 컨테이너 안 `/tinynav`에 마운트
- `-v "$HOME/.local/share/tinynav:/root/.local/share/tinynav"`: rosbag/캐시/산출물 유지

### 4. 컨테이너 안에서 실행할 대표 명령

#### 예제 데모

```bash
bash scripts/run_rosbag_examples.sh
```

#### 예제 rosbag으로 맵 생성

```bash
bash scripts/run_rosbag_build_map.sh
```

[scripts/run_rosbag_build_map.sh](scripts/run_rosbag_build_map.sh)는 기본적으로 예제 rosbag을 받아 `/tinynav/output/map_go2_looper`에 맵을 만듭니다.

#### 맵 기반 내비게이션

[scripts/run_navigation.sh](scripts/run_navigation.sh)은 바로 실행하면 안 됩니다. 파일 상단의 `map_path="PATH/TO/MAP"`를 실제 맵 경로로 바꾼 뒤 실행해야 합니다. 예를 들면 방금 생성한 맵을 쓸 때는 다음처럼 수정하면 됩니다.

```bash
map_path="/tinynav/output/map_go2_looper"
```

그 뒤 실행합니다.

```bash
bash scripts/run_navigation.sh
```

#### RealSense 센서 드라이버

실제 카메라가 Linux 호스트에 연결되어 있고 컨테이너를 위 옵션으로 띄웠다면 다음 스크립트를 사용할 수 있습니다.

```bash
bash scripts/run_realsense_sensor.sh
```

녹화까지 같이 하려면:

```bash
bash scripts/run_map_record.sh
```

## 앱 백엔드와 프런트엔드를 Docker 안에서 실행하기

[scripts/start_app.sh](scripts/start_app.sh)은 다음 두 프로세스를 tmux로 실행합니다.

- FastAPI 백엔드: 기본 8000 포트
- Flutter web 정적 파일 서버: 기본 80 포트

다만 한 가지 주의할 점이 있습니다. Docker 이미지 빌드 시점에는 Flutter web이 이미 빌드되지만, 실제 실행에서는 보통 `-v "$PWD:/tinynav"`로 워크스페이스를 bind mount합니다. 그러면 이미지 안에 있던 `/tinynav/app/frontend/build/web`가 호스트 파일로 덮이기 때문에, 호스트 저장소에 빌드 산출물이 없으면 프런트엔드 서버가 바로 뜨지 않습니다.

앱을 띄우기 전에 컨테이너 안에서 먼저 한 번 빌드하는 편이 안전합니다.

```bash
cd /tinynav/app/frontend
flutter pub get
flutter build web --release
cd /tinynav
BACKEND_PORT=8000 FRONTEND_PORT=8080 bash scripts/start_app.sh
```

그러면 Linux 호스트 기준으로 다음 주소를 사용할 수 있습니다.

- 백엔드: `http://localhost:8000`
- 프런트엔드: `http://localhost:8080`

`TINYNAV_DB_PATH`를 바꾸고 싶다면 실행 전에 환경변수로 넘기면 됩니다.

```bash
TINYNAV_DB_PATH=/tinynav/tinynav_db BACKEND_PORT=8000 FRONTEND_PORT=8080 bash scripts/start_app.sh
```

## 이미지를 직접 다시 빌드해야 할 때

대부분의 사용자는 공개 이미지 `uniflexai/tinynav:latest`를 쓰는 편이 낫습니다. 이 이미지는 CI에서 멀티 아키텍처로 빌드됩니다. 직접 빌드가 필요한 경우는 보통 다음 둘 중 하나입니다.

- Dockerfile을 수정했을 때
- 공개 이미지 대신 사내/개인 이미지로 재배포하고 싶을 때

로컬 빌드는 무겁습니다. TensorRT, ROS 2, RealSense, Flutter, 추가 워크스페이스 빌드까지 포함되어 시간이 오래 걸립니다. 그래도 필요하면 buildx로 현재 대상 아키텍처에 맞춰 빌드합니다.

```bash
# x86_64 Linux 데스크톱
docker buildx build --load --platform linux/amd64 -t tinynav:local .

# Jetson/arm64
docker buildx build --load --platform linux/arm64 -t tinynav:local .
```

그 다음 위 `docker run` 명령에서 이미지 이름만 `tinynav:local`로 바꾸면 됩니다.

주의할 점:

- 이 Dockerfile은 [Dockerfile](Dockerfile)에서 `uniflexai/base_image:latest`를 베이스로 사용합니다.
- 첫 컨테이너 진입 시 TensorRT `.plan` 파일이 없으면 model build prompt가 뜰 수 있습니다.
- 로컬 빌드보다 공개 이미지를 먼저 검증하는 편이 실패 가능성이 낮습니다.

## 자주 막히는 지점

### 1. macOS에서 왜 안 되나

이 프로젝트는 NVIDIA GPU 런타임과 Linux 장치 접근을 전제로 합니다. Docker Desktop이 제공하는 macOS 가상화 계층에서는 다음이 맞지 않습니다.

- `--gpus all`
- Jetson/NVIDIA TensorRT 런타임
- Linux 방식의 `/dev` 장치 접근
- `/tmp/.X11-unix` 기반 RViz 표시
- ROS 2에서 기대하는 host network 동작

따라서 맥이 주 개발 머신이라면, Linux GPU 머신에 Remote SSH로 접속한 뒤 그 머신에서 Dev Container 또는 `docker run`을 쓰는 흐름이 현실적입니다.

### 2. RViz 창이 안 뜰 때

- Linux 호스트에서 `xhost +local:root`를 먼저 실행했는지 확인
- `DISPLAY` 환경변수와 `/tmp/.X11-unix` 마운트가 들어갔는지 확인
- Wayland 환경이면 XWayland 설정이 필요한지 확인

### 3. RealSense가 안 잡힐 때

- `--privileged`와 `-v /dev:/dev`를 빼지 않았는지 확인
- 호스트에 실제 장치가 연결되어 있는지 확인
- 컨테이너 안에서 `rs-enumerate-devices`가 동작하는지 확인

### 4. 첫 실행이 느릴 때

정상일 수 있습니다. 다음 작업들이 처음에만 오래 걸립니다.

- TensorRT `.plan` 생성
- Hugging Face 예제 rosbag 다운로드
- Flutter web build

## 추천 실행 순서

처음 올릴 때는 아래 순서가 가장 단순합니다.

1. Linux + NVIDIA 또는 Jetson 준비
2. `git lfs pull` 수행
3. Dev Container 또는 위의 `docker run`으로 컨테이너 진입
4. 첫 진입 시 model build prompt가 뜨면 필요에 따라 진행
5. `bash scripts/run_rosbag_examples.sh`로 데모 확인
6. 이후 `bash scripts/run_rosbag_build_map.sh`, `bash scripts/run_navigation.sh`, `bash scripts/start_app.sh` 순으로 확장

한 줄로 요약하면, 이 저장소는 "Linux GPU 호스트에서 공개 TinyNav 이미지를 Dev Container 또는 `docker run`으로 띄우고, 컨테이너 안에서 제공된 스크립트를 실행하는 방식"으로 사용하는 것이 맞습니다.
