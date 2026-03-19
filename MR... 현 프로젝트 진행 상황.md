# AgriBot 프로젝트 진행 상황

## 2026-03-19

### 이번에 진행한 작업

- `agribot`의 목표 주행 속도를 `0.30 m/s`에서 `1.60 m/s`로 상향했다.
- 단순히 최고 속도만 올리지 않고, 그 변화에 따라 깨질 수 있는 Nav2 / Gazebo / SLAM / AMCL 관련 하드코딩된 파라미터들도 함께 조정했다.
- 변경 후 실제 런타임 검증까지 수행했고, 결과를 확인한 뒤 커밋했다.

### 수정한 파일

- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_description/models/agribot/model.sdf`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/nav2_params.yaml`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/nav2_mapping_params.yaml`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/slam_mapping.yaml`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/amcl.yaml`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml`

### 핵심 변경 내용

- Gazebo diff-drive 제한값
  - `max_linear_velocity: 0.30 -> 1.60`
  - `max_linear_acceleration: 0.60 -> 2.40`
  - `max_angular_velocity: 1.20 -> 3.20`
  - `max_angular_acceleration: 1.20 -> 3.20`
- LiDAR update rate
  - `10 Hz -> 20 Hz`
- Nav2 controller
  - `desired_linear_vel: 1.60`
  - `controller_frequency: 30 Hz`
  - lookahead / curvature / regulated scaling 관련 값들을 고속 주행 기준으로 재조정
- Costmap
  - local costmap 크기 `4x4 -> 8x8`
  - local/global update 주기 상향
  - obstacle range / inflation radius 확장
- SLAM / AMCL
  - 고속 이동 중 샘플 누락이 줄어들도록 최소 이동 거리, 최소 시간 간격, beam 수, update threshold 등을 조정
- 순찰 메타데이터
  - `recommended_travel_speed_mps: 1.60`으로 반영

### 검증 내용

- `pytest src/agribot_navigation/test/test_harvest_routing.py`
  - `3 passed`
- YAML 파싱 검증 통과
- `xmllint`로 `model.sdf` XML 검증 통과
- `compileall` 검증 통과
- 실제 런타임 검증도 진행
  - `autonomous_mapping.launch.py use_rviz:=false patrol_autostart:=false`로 스택 기동
  - 런타임에서 아래 값 확인
    - `controller_frequency = 30.0`
    - `FollowPath.desired_linear_vel = 1.6`
    - `FollowPath.lookahead_dist = 1.2`
    - `FollowPath.max_lookahead_dist = 2.6`
    - `FollowPath.regulated_linear_scaling_min_radius = 3.0`
  - `mapping_patrol/start` 호출 후 실제 관측값
    - 최대 선속도 `1.5997 m/s`
    - 최대 각속도 `1.9618 rad/s`

### 참고 사항

- 이번 수정은 "속도를 높였을 때 바로 무너지는 비율"을 줄이기 위한 튜닝이다.
- 다만 `1.6 m/s`는 기존 대비 상당히 공격적인 값이라서, 실제 데모/발표 전에는 직선 구간, 회전 구간, 온실 통로 충돌 여유를 다시 한 번 눈으로 확인하는 것이 좋다.
- LiDAR는 설정상 `20 Hz`로 올렸지만 이번 검증 런에서는 약 `15.7 Hz` 수준으로 측정됐다. 현재 Gazebo 실시간율과 렌더링 영향 가능성이 있다.

### 커밋

- 커밋 해시: `5b4a882`
- 메시지: `Tune agribot for 1.6 m/s navigation`

### 다시 빌드 / 실행할 때 명령어

```bash
cd /home/ssafy/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --packages-select agribot_description agribot_navigation agribot_bringup --symlink-install
source install/setup.bash
```

정적 맵 네비게이션:

```bash
ros2 launch agribot_bringup simulation.launch.py
```

SLAM 기반 자율 매핑:

```bash
ros2 launch agribot_navigation autonomous_mapping.launch.py
```

GUI 없이 빠르게 확인:

```bash
ros2 launch agribot_navigation autonomous_mapping.launch.py use_rviz:=false
```

### 추가 진행한 작업

- 자율 매핑 기본 동작을 greenhouse 전용 waypoint patrol에서 일반 frontier exploration으로 바꿨다.
- 원인을 LiDAR 범위 부족 하나로 단정하지 않고, 탐색 방식, Nav2 controller, costmap, SLAM, AMCL, 센서 유효 범위를 함께 조정했다.
- 그래서 이제 기본 `autonomous_mapping.launch.py`는 다른 맵으로 바꿔도 frontier 기반으로 자율 탐색/매핑을 시도할 수 있다.

### 이번에 추가로 수정한 파일

- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/agribot_navigation/frontier_explorer.py`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/frontier_explorer.yaml`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/test/test_frontier_explorer.py`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/launch/autonomous_mapping.launch.py`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/nav2_mapping_params.yaml`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/slam_mapping.yaml`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/amcl.yaml`
- `/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_description/models/agribot/model.sdf`

### 추가 핵심 변경 내용

- 기본 자율 매핑 launch
  - `use_frontier_explorer:=true`
  - `use_patrol:=false`
  - `use_boundary_map:=false`
- 새 frontier explorer
  - live `/map` 기준 frontier cluster 탐색
  - 실패 goal blacklist 후 재시도
  - 상태 topic: `/mapping_explorer/status`
  - 서비스: `/mapping_explorer/start`, `/mapping_explorer/stop`, `/mapping_explorer/resume`
- 센서/맵 관련 값
  - LiDAR samples `360 -> 720`
  - LiDAR max range `10.0 -> 25.0`
  - RGB-D far clip `10.0 -> 25.0`
  - SLAM / AMCL laser range도 `25.0` 기준으로 조정
- mapping 모드 Nav2
  - `use_rotate_to_heading: false`
  - lookahead / tolerance / local/global costmap 범위 재조정
  - 막힌 frontier에서 즉시 끝나지 않고 다른 frontier로 계속 진행

### 추가 검증 내용

- `python3 -m pytest src/agribot_navigation/test/test_harvest_routing.py src/agribot_navigation/test/test_frontier_explorer.py`
  - `6 passed`
- `colcon build --packages-select agribot_navigation agribot_description agribot_bringup --symlink-install`
  - 통과
- 실제 런타임 검증
  - `ros2 launch agribot_navigation autonomous_mapping.launch.py use_rviz:=false`
  - frontier goal 생성 확인
  - `/mapping_explorer/status`가 `exploring`으로 유지되는 것 확인
  - 최대 선속도 약 `1.5155 m/s`
  - planner에서 `Start occupied`가 떠도 blacklist 후 다음 frontier로 재시도하는 것 확인

### 추가 커밋

- 커밋 해시: `7185809`
- 메시지: `Add frontier-based autonomous mapping explorer`

### 지금 기준으로 다시 빌드 / 실행할 때 명령어

```bash
cd /home/ssafy/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --packages-select agribot_navigation agribot_description agribot_bringup --symlink-install
source install/setup.bash
```

기본 frontier 기반 자율 매핑:

```bash
ros2 launch agribot_navigation autonomous_mapping.launch.py use_rviz:=false
```

frontier explorer 수동 제어:

```bash
ros2 service call /mapping_explorer/start std_srvs/srv/Trigger "{}"
ros2 service call /mapping_explorer/stop std_srvs/srv/Trigger "{}"
ros2 service call /mapping_explorer/resume std_srvs/srv/Trigger "{}"
ros2 topic echo /mapping_explorer/status
```

기존 greenhouse patrol 방식으로 다시 실행하고 싶다면:

```bash
ros2 launch agribot_navigation autonomous_mapping.launch.py \
  use_patrol:=true \
  patrol_autostart:=true \
  use_frontier_explorer:=false \
  use_boundary_map:=true \
  use_rviz:=false
```
