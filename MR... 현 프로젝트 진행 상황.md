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
