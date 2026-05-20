# TinyNav Navigation Flow 검증 메모

## 결론

제시된 6단계 설명은 큰 흐름에서는 대체로 맞습니다. 실제로 [tinynav/core/planning_node.py](tinynav/core/planning_node.py) 는 깊이 맵과 위치 정보를 받아 3D 점유 격자를 갱신하고, 장애물 맵과 거리 맵을 만든 뒤, 후보 궤적을 생성하고 평가해서 최종 경로를 발행합니다.

다만 그대로 사실이라고 보기에는 몇 가지 중요한 차이가 있습니다.

## 코드 기준으로 틀리거나 빠진 부분

### 1. 입력은 3개만으로 충분하지 않습니다

원문은 목표점, Odometry, Depth Map 3가지만 있으면 된다고 읽히지만, 실제 [tinynav/core/planning_node.py](tinynav/core/planning_node.py) 는 다음 입력을 받습니다.

- `/slam/depth`
- `/slam/odometry_visual`
- `/camera/camera/infra2/camera_info`
- `/control/target_pose`
- `/mapping/poi_change`

특히 `camera_info` 로부터 `K` 와 baseline 을 받기 전에는 `sync_callback()` 초반에서 바로 return 합니다. 즉, 카메라 내부 파라미터도 실제 필수 입력입니다.

### 2. 동기화되는 것은 Depth와 `/slam/odometry_visual` 뿐입니다

원문은 목표점, 현재 위치, 센서 입력이 함께 정렬되는 것처럼 읽히지만, 실제 `TimeSynchronizer` 는 `/slam/depth` 와 `/slam/odometry_visual` 두 개만 묶습니다. 목표점 `/control/target_pose` 는 별도 subscription 으로 최신 값만 저장합니다.

또한 planning 에 쓰는 위치는 `/slam/odometry` 가 아니라 `/slam/odometry_visual` 입니다.

### 3. 장애물 맵 생성은 단순한 "높이 영역 슬라이싱"보다 더 구체적입니다

원문은 "로봇 키에 해당하는 영역만 걸러낸다"고 요약했지만, 실제 [tinynav/core/planning_node.py](tinynav/core/planning_node.py) 의 `build_obstacle_map()` 는 다음 조건을 함께 사용합니다.

- `robot_z_bottom` 부터 `robot_z_top` 범위만 사용
- 점유값이 `occ_threshold` 를 넘는 voxel 만 고려
- z 방향 점유 span 이 `min_wall_span_m` 이상인 경우만 장애물로 판정
- 마지막에 `binary_dilation` 으로 장애물을 팽창

즉, 단순히 특정 높이 단면을 잘라 2D 장애물 맵을 만드는 수준이 아니라, 벽처럼 세로로 충분히 이어진 구조만 장애물로 보는 후처리가 들어 있습니다.

### 4. 코드가 만드는 것은 엄밀한 signed distance field 가 아닙니다

변수 이름은 `ESDF_map` 이지만, 실제 구현은 다음 한 줄입니다.

- `distance_transform_edt(~obstacle_mask) * resolution`

이 계산은 장애물 셀에서 0, 자유 공간에서 양수 거리를 가지는 2D 거리 변환입니다. 즉, 일반적으로 말하는 signed distance field 처럼 장애물 내부가 음수가 되는 형태는 아닙니다. 따라서 "ESDF" 라는 표현은 코드 변수명 기준으로는 맞지만, 구현 자체는 signed field 보다는 obstacle distance map 에 가깝습니다.

### 5. 후보 궤적은 "수십~수백 개"가 아니라 기본 설정상 55개입니다

원문은 수십~수백 개 후보라고 했지만, 기본 `generate_trajectory_library_3d()` 설정은 `num_samples=11` 이고 실제 생성 개수는 다음 조합입니다.

- 가속도 샘플 5개
- 회전 샘플 11개
- 총 55개

즉, 현재 기본값 기준으로는 "수십 개"는 맞지만 "수백 개"는 아닙니다.

### 6. 점수는 보상 점수가 아니라 penalty 에 가깝습니다

원문은 "안전할수록 좋은 점수를 준다"고 설명했지만, 실제 `score_trajectories_by_ESDF()` 는 낮을수록 좋은 값입니다.

- 충돌 궤적: `inf`
- 충분히 안전한 궤적: `0.0`
- 장애물에 가까운 궤적: 더 큰 양수 penalty

즉, 안전한 경로에 높은 점수를 주는 구조가 아니라, 위험한 경로에 큰 비용을 부과하는 구조입니다.

### 7. 목표점은 비용 함수에만 쓰이고, 후보 생성에는 직접 쓰이지 않습니다

원문은 목표점을 바탕으로 후보 경로를 생성하는 것처럼 읽힐 수 있지만, 실제 후보 생성은 현재 자세, 현재 속도, 현재 orientation 에서 시작하는 고정 trajectory library 입니다. 목표점 `target_pose` 는 마지막 `cost_function()` 에서 trajectory 끝점과의 거리 계산에만 사용됩니다.

또한 `target_pose_callback()` 는 Odometry 메시지의 orientation 을 쓰지 않고, position `(x, y, z)` 만 읽습니다.

### 8. planning_node 가 직접 로봇을 움직이는 것은 아닙니다

원문 마지막 문장은 개념적으로는 맞지만, 구현상 한 단계가 더 있습니다.

- [tinynav/core/planning_node.py](tinynav/core/planning_node.py) 는 `/planning/trajectory_path` 를 publish
- [tinynav/platforms/cmd_vel_control.py](tinynav/platforms/cmd_vel_control.py) 는 그 Path 와 `/slam/odometry` 를 구독
- 같은 노드가 최종적으로 `/cmd_vel` 을 publish

즉, planning_node 가 최종 경로를 내보내면, 실제 속도 제어는 별도 제어 노드가 담당합니다.

## 코드 기준으로 다시 쓴 정정본

### 1. 입력 수신과 준비

planning_node 는 `/slam/depth` 와 `/slam/odometry_visual` 을 시간 동기화해서 받고, 별도로 `/camera/camera/infra2/camera_info` 에서 카메라 내부 파라미터를 받습니다. 또한 `/control/target_pose` 에서 목표 위치를 별도 subscription 으로 저장합니다.

### 2. 전처리와 속도 추정

동기화된 깊이 맵과 시각 Odometry 에서 현재 pose 를 복원하고, 직전 pose 와의 위치 차이를 이용해 속도를 추정합니다. 이 속도는 지수평활된 `smoothed_velocity` 로 유지됩니다.

### 3. 3D 점유 격자 갱신

현재 카메라 pose 와 depth 를 이용해 raycasting 을 수행하고, rolling 3D occupancy grid 를 갱신합니다. 로봇이 격자 중심에서 멀어지면 grid origin 자체도 함께 이동합니다.

### 4. 2D 장애물 맵과 거리 맵 생성

3D occupancy grid 에서 로봇 높이 근처의 점유 voxel 중 z span 이 충분히 큰 구조만 장애물로 간주하고, dilation 으로 마진을 줍니다. 그 결과 2D obstacle mask 를 만들고, 여기에 대해 거리 변환을 계산해 obstacle distance map 을 생성합니다.

### 5. 후보 궤적 생성과 충돌 평가

현재 로봇 중심 위치, 현재 orientation, 평활 속도를 초기 상태로 사용해 기본 설정상 55개의 후보 궤적을 생성합니다. 각 궤적에 대해 로봇 footprint 중심과 네 꼭짓점의 최소 장애물 거리를 계산해 collision 여부와 안전 비용을 평가합니다.

### 6. 목표점 기반 비용 계산과 경로 발행

후보 궤적마다 다음 비용을 합쳐 최종 1등 궤적을 고릅니다.

- 장애물 회피 penalty
- 이전 control parameter 와의 차이
- 목표점까지의 거리

단, 목표점이 없으면 경로를 publish 하지 않고 return 합니다. 충돌하지 않는 경로가 남아 있으면 선택된 궤적을 `/planning/trajectory_path` 로 발행합니다. 이후 실제 `/cmd_vel` 변환과 속도 제한, stale-path 처리 등은 [tinynav/platforms/cmd_vel_control.py](tinynav/platforms/cmd_vel_control.py) 가 담당합니다.

## 최종 판정

원문 설명은 "planning_node 내부 알고리즘의 큰 줄기"는 맞습니다. 하지만 아래처럼 고쳐 이해하는 것이 더 정확합니다.

- 입력은 3개가 아니라 `camera_info` 까지 포함해 봐야 함
- planning 입력 Odometry 는 `/slam/odometry_visual` 임
- ESDF 는 엄밀한 signed field 가 아니라 2D 거리 변환 결과에 가까움
- 후보 궤적 기본 개수는 55개임
- 점수는 reward 가 아니라 penalty 구조임
- planning_node 가 직접 모터를 구동하는 것이 아니라 Path 를 publish 하고, 별도 제어 노드가 `/cmd_vel` 로 변환함
