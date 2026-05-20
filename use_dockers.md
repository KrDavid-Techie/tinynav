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

## GUI 없이 Foxglove로 보는 docker compose 구성

RealSense 카메라를 직접 연결하고, RViz 대신 Foxglove로 토픽을 보면서, 로봇 제어는 `unitree_ros2` 쪽 command API 토픽으로 넘기고 싶다면 `docker compose` 기반 장기 실행 구성이 더 맞습니다.

이 경우 핵심 전제는 다음과 같습니다.

- Linux 호스트에서 Docker를 사용한다.
- RealSense 장치는 호스트에 직접 연결되어 있다.
- Foxglove Studio는 웹 또는 데스크톱 앱으로 `ws://HOST:8765` 에 접속한다.
- TinyNav의 기본 주행 체인은 ROS 2 토픽 `/cmd_vel` 까지를 만든다.
- sit/stand 같은 액션 명령이 필요하면 `/service/command` 는 app/backend 또는 외부 publisher 가 추가로 넣어줘야 한다.
- `unitree_ros2` 쪽 command API topic 은 이미 준비되어 있거나, 별도 브리지 노드로 `/cmd_vel` 과 필요 시 `/service/command` 를 해당 토픽으로 변환할 수 있다.

중요한 점 하나는, 이 시나리오에서는 [tinynav/platforms/unitree_control.py](tinynav/platforms/unitree_control.py) 를 기본 제어 경로로 쓰지 않는 것이 맞다는 것입니다. 이 파일은 ROS 2 토픽이 아니라 Unitree SDK DDS 채널 `rt/cmd_vel`, `rt/service/command`, `rt/lf/lowstate` 를 직접 구독하고, 네트워크 인터페이스도 `enP8p1s0` 로 하드코딩되어 있습니다. 즉, 이미 `unitree_ros2` 기반 ROS command API 가 있다면, TinyNav 쪽에서는 주행 명령 `/cmd_vel` 을 만들고, 필요하면 별도로 `/service/command` 를 주입한 뒤, 그 다음 단계는 `unitree_ros2` 브리지/노드가 맡는 구조가 더 자연스럽습니다.

### 이 구성에서의 실제 데이터 흐름

GUI 없이 Foxglove만 쓰는 최소 실시간 구성은 보통 다음 흐름입니다.

1. RealSense 드라이버가 `/camera/camera/infra1/image_rect_raw`, `/camera/camera/infra2/image_rect_raw`, `/camera/camera/imu`, `/camera/camera/infra2/camera_info` 를 발행
2. [tinynav/core/perception_node.py](tinynav/core/perception_node.py) 가 `/slam/depth`, `/slam/odometry_visual`, `/slam/disparity_vis` 등을 생성
3. 같은 프로세스 안의 `ImuPropagatorNode` 가 `/slam/odometry` 를 생성
4. [tinynav/core/planning_node.py](tinynav/core/planning_node.py) 가 `/planning/trajectory_path`, `/planning/height_map`, `/planning/occupied_voxels` 등을 발행
5. [tinynav/platforms/cmd_vel_control.py](tinynav/platforms/cmd_vel_control.py) 가 `/planning/trajectory_path` 와 `/slam/odometry` 를 받아 `/cmd_vel` 을 발행
6. Foxglove bridge 가 위 토픽들을 WebSocket으로 노출
7. `unitree_ros2` 브리지 또는 기존 command API 노드가 `/cmd_vel` 을 로봇 제어 토픽으로 전달하고, 필요하면 `/service/command` 도 함께 전달

### 왜 compose가 적합한가

이 경로에서는 RViz/X11 이 필요 없으므로 다음 항목을 모두 제거할 수 있습니다.

- `DISPLAY`
- `/tmp/.X11-unix` 마운트
- `xhost +local:root`

반면 다음은 그대로 유지하는 편이 좋습니다.

- `network_mode: host`
- `privileged: true`
- `/dev:/dev`
- 충분한 shared memory (`ipc: host` 또는 `shm_size`)

### 실제 compose 파일

실제 실행 파일은 루트의 [compose.yaml](compose.yaml) 입니다. 이 파일은 바로 `docker compose up -d` 로 올릴 수 있게 만들어 두었습니다.

[compose.yaml](compose.yaml)에 포함된 서비스는 다음과 같습니다.

- `tinynav-init`: 첫 실행 시 TensorRT `.plan` 생성만 담당하는 초기화 서비스
- `realsense`: [scripts/run_realsense_sensor.sh](scripts/run_realsense_sensor.sh) 기반 RealSense 드라이버
- `perception`: [tinynav/core/perception_node.py](tinynav/core/perception_node.py)
- `planning`: [tinynav/core/planning_node.py](tinynav/core/planning_node.py)
- `cmd-vel-control`: [tinynav/platforms/cmd_vel_control.py](tinynav/platforms/cmd_vel_control.py)
- `foxglove-image-repub`: 컬러 영상을 Foxglove 친화 포맷으로 재퍼블리시
- `foxglove-bridge`: Foxglove WebSocket 브리지

기본값으로는 아래 환경변수를 사용합니다.

- `TINYNAV_IMAGE=uniflexai/tinynav:latest`
- `ROS_DOMAIN_ID=0`
- `FOXGLOVE_PORT=8765`
- `TINYNAV_HOST_DATA_DIR=./tinynav_data`

필요하면 셸에서 export 한 뒤 `docker compose` 를 실행하면 됩니다.

이 실제 compose 파일에는 `unitree_ros2` 브리지 서비스는 넣지 않았습니다. 이유는 사용자가 이미 `unitree_ros2` command API topic 을 준비할 수 있다는 전제였고, 그 경우 TinyNav 쪽은 `/cmd_vel` 까지만 만들면 충분하기 때문입니다. 즉, 기본 compose 스택을 올린 뒤 같은 host network 와 같은 `ROS_DOMAIN_ID` 위에서 사용자의 `unitree_ros2` 노드가 `/cmd_vel` 을 구독하게 두는 방식이 가장 단순합니다. sit/stand 같은 액션 명령이 필요하면 `/service/command` 는 외부 publisher 나 app/backend 가 별도로 넣어주면 됩니다.

### 실행 순서

첫 실행에서는 모델 초기화를 먼저 끝내는 편이 좋습니다.

```bash
docker compose --profile init run --rm tinynav-init
```

그 다음 본 스택을 올립니다.

```bash
docker compose up -d
```

로그 확인:

```bash
docker compose logs -f realsense perception planning cmd-vel-control foxglove-bridge
```

Foxglove 연결 주소:

```text
ws://<Linux-host-ip>:8765
```

### Foxglove에서 보면 좋은 토픽

다음 토픽만 열어도 상태 확인이 거의 됩니다.

- `/slam/odometry`
- `/slam/odometry_visual`
- `/slam/depth`
- `/slam/disparity_vis`
- `/planning/trajectory_path`
- `/planning/height_map`
- `/planning/occupied_voxels`
- `/planning/footprint`
- `/cmd_vel`
- `/control/target_pose`

### 목표점이 없으면 로봇은 움직이지 않습니다

[tinynav/core/planning_node.py](tinynav/core/planning_node.py) 는 `/control/target_pose` 가 없으면 경로를 publish 하지 않습니다. GUI를 쓰지 않는다면, 가장 단순한 방법은 외부에서 목표점을 한 번 넣는 것입니다.

```bash
docker compose exec planning bash -lc 'source /opt/ros/humble/setup.bash && ros2 topic pub --once /control/target_pose nav_msgs/msg/Odometry "{header: {frame_id: world}, pose: {pose: {position: {x: 1.0, y: 0.0, z: 0.0}, orientation: {w: 1.0}}}}"'
```

이 명령은 map 기반 전역 내비게이션이 아니라, 현재 local planner 에게 직접 목표점을 주는 방식입니다. 기존 맵과 POI를 쓰는 전체 내비게이션이 필요하면 `map_node` 를 compose 서비스에 추가하고, 맵 경로를 mount 한 뒤 `/mapping/cmd_pois` 또는 앱 백엔드를 함께 써야 합니다.

### 이 구성을 쓸 때 자주 막히는 지점

#### 1. `unitree_control.py` 를 같이 띄우면 안 되는가

보통은 같이 띄우지 않는 편이 낫습니다. [tinynav/platforms/unitree_control.py](tinynav/platforms/unitree_control.py) 는 `unitree_ros2` API 용 ROS bridge 가 아니라, Unitree SDK DDS 채널 직접 구독기입니다. `unitree_ros2` 를 이미 쓰는 경우에는 제어 경로가 중복되거나 엇갈릴 가능성이 큽니다. 현재 실제 [compose.yaml](compose.yaml) 도 이 이유 때문에 `unitree_control.py` 를 포함하지 않습니다.

#### 2. RealSense만 뜨고 planning 이 조용한가

대개 다음 중 하나입니다.

- `/camera/camera/infra2/camera_info` 가 아직 안 들어옴
- `/control/target_pose` 를 주지 않음
- 첫 모델 생성이 끝나지 않음

#### 3. Foxglove는 붙는데 그림이 없는가

다음부터 먼저 확인하면 됩니다.

- `foxglove-bridge` 로그에서 topic whitelist 파싱 오류가 없는지
- `realsense` 와 `perception` 로그에서 실제 토픽 publish 가 되는지
- Foxglove 연결 주소가 `ws://HOST:8765` 인지

### compose 기반 구성의 한 줄 요약

RealSense + TinyNav + Foxglove + `unitree_ros2` 조합에서는, 실제 [compose.yaml](compose.yaml) 로 TinyNav 스택을 올리고, 외부의 `unitree_ros2` 노드가 같은 host network 에서 `/cmd_vel` 을 받아가게 두는 구성이 가장 현실적입니다. `/service/command` 는 액션 명령이 필요할 때만 추가로 연결하면 됩니다.

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

### 1. RViz 창이 안 뜰 때

- Linux 호스트에서 `xhost +local:root`를 먼저 실행했는지 확인
- `DISPLAY` 환경변수와 `/tmp/.X11-unix` 마운트가 들어갔는지 확인
- Wayland 환경이면 XWayland 설정이 필요한지 확인

### 2. RealSense가 안 잡힐 때

- `--privileged`와 `-v /dev:/dev`를 빼지 않았는지 확인
- 호스트에 실제 장치가 연결되어 있는지 확인
- 컨테이너 안에서 `rs-enumerate-devices`가 동작하는지 확인

### 3. 첫 실행이 느릴 때

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
