## [S14P-XX] [건드린 폴더 분야 (FE or BE or AI 등)] 이름 (지라 티켓으로도 활용 가능한 이름)

### 개요

---

### 배경

---

### 주요 변경 사항

---

### 검증 방법

---

### 검증 결과

---

### 확인된 이슈 및 주의사항

---

### 기대 효과

---

### 후속 작업 제안

---

### 변경 파일

---

### 관련 커밋

---

### 작업 브랜치

- ``

---

[S14P-31] 센서 출력 점검 및 Gazebo-ROS 브리지 안정화

## 개요

S14P-31 센서 출력 점검을 진행했습니다.

이번 MR의 목적은 다음 2가지입니다.

1. 현재 시뮬레이션에서 카메라, LiDAR, IMU, `/odom`이 실제로 어떤 상태로 출력되는지 확인한다.
2. 팀원이 같은 브랜치를 pull 받은 뒤 동일한 결과를 재현할수 있도록, 센서 브리지 구성을 런치 파일에 반영하고 점검 결과를 문서화한다.

점검 결과, 카메라 RGB/Depth/CameraInfo, IMU, `/odom`, `/clock`은 정상 수신되도록 정리했고, LiDAR `/agribot/lidar/scan`은 아직 미수신 상태로 남아 추가 원인 분석이 필요합니다.

---

### 4차 MR: 사각 농장 맵 자율 매핑의 좌표계와 row-end sweep를 저발열로 재정렬

### 배경

- 직전까지의 `farm_world` 매핑 순찰은 center aisle 진입 이후에는 어느 정도 전진했지만, `East Center South` 구간부터 `Start occupied`, `Failed to make progress`, `collision ahead`가 다시 반복됐다.
- 원인을 좁혀보니 단순 controller 튜닝 문제가 아니라, 다음 세 축이 겹쳐 있었다.
  - `odom` 기준 순찰 좌표와 실제 `farm_world`의 row-end connector 위치가 어긋나 있었다.
  - 회전 우선 추종 옵션 때문에 직선 구간에서도 제자리 heading 보정이 먼저 나오며 진전이 거의 없는 경우가 있었다.
  - LiDAR, SLAM, EKF, collision monitor가 현재 노트북 환경 기준으로는 다소 과하게 자주 갱신되어, 발열을 키우지 않으면서도 안정도를 높일 쪽으로 재정렬할 필요가 있었다.

### 이번 MR에서 반영한 내용

- `agribot_description/models/agribot/model.sdf`
  - mapping용 LiDAR를 `720 @ 10Hz`에서 `360 @ 8Hz`로 낮춰 저발열 방향으로 조정
  - diff-drive odom publish 주기를 `20Hz`로 맞춰 EKF 입력과의 과도한 불균형을 줄임
- `agribot_navigation/config/ekf_mapping.yaml`
  - EKF를 `20Hz`로 낮추고 `publish_tf: true`, `predict_to_current_time: true`를 명시
  - mapping 전용 상태추정이 현재 시각과 더 자연스럽게 맞물리도록 정리
- `agribot_navigation/launch/autonomous_mapping.launch.py`
  - spawn 쪽 `publish_odom_tf`를 끄고 EKF가 `odom -> base_link`를 일관되게 맡도록 정리
  - patrol node에 `max_batch_path_length_m: 6.0`, `max_lane_segment_length_m: 4.5`를 주입
- `agribot_navigation/launch/mapping.launch.py`
  - 수동 mapping launch도 동일하게 `publish_odom_tf: false`를 적용
- `agribot_navigation/agribot_navigation/patrol_node.py`
  - robot pose 기준 토픽을 `/odometry/filtered`로 전환
  - 첫 filtered pose를 받기 전에는 auto-start하지 않도록 보강
  - 긴 batch route를 끊는 `max_batch_path_length_m` 제약과 lane segment 분할 로직을 유지
- `agribot_navigation/config/slam_mapping.yaml`
  - `throttle_scans: 3`, `minimum_time_interval: 0.30`, `max_laser_range: 7.0` 등 실제 센서 사양 기준으로 정리
  - scan 처리량을 줄여 `slam_toolbox` 큐 적체와 발열을 줄이는 방향으로 조정
- `agribot_navigation/config/nav2_mapping_params.yaml`
  - `required_movement_radius: 0.12`, `movement_time_allowance: 18.0`으로 progress checker를 완화
  - `desired_linear_vel: 0.32`, `lookahead_dist: 0.70`, `xy_goal_tolerance: 0.50`으로 직선 추종을 좀 더 여유 있게 조정
  - `use_rotate_to_heading: false`로 바꿔 직선 세그먼트에서 제자리 회전부터 시도하던 패턴 제거
  - local/global inflation radius를 `0.36`으로 맞춰 row 근처에서 `Start occupied`가 과도하게 나는 상황을 완화
- `agribot_navigation/config/collision_monitor_mapping.yaml`
  - `FrontStopZone`을 더 좁고 앞쪽으로 재배치하고, `transform_tolerance`와 `source_timeout`을 보강
  - 너무 가까운 구간에서 즉시 멈추는 false positive를 줄이되 안전 여유는 유지
- `agribot_navigation/config/farm_mapping_patrol_waypoints.yaml`
  - 가장 중요한 수정으로, row 사이 차선 변경 좌표를 `odom x = ±5.4`가 아니라 실제 connector에 대응하는 `odom x = ±8.6` 근처로 재설정
  - `home`는 `(0.0, 0.0)` 유지, aisle sweep 순서는 기존 center-start 전략을 유지
  - 즉 lane change는 crop row 한가운데가 아니라 row-end connector에서만 일어나도록 geometry를 바로잡음
- 테스트
  - `test_mapping_patrol_plan.py`의 bounds와 대표 waypoint 기대값을 새 geometry에 맞게 수정
  - `test_patrol_batching.py`에 batch path length 제약과 segment 분할 회귀 검증을 유지

### 왜 이렇게 바꿨는가

- 현재 문제의 본질은 센서를 더 무겁게 쓰지 않아서가 아니라, geometry와 추종 전략이 실제 월드 구조와 정확히 맞지 않았던 데 있었다.
- `farm_world`는 직사각형 row 구조라서 차선 변경이 가능한 위치가 row 끝 connector로 사실상 제한된다.
- 그런데 waypoint가 중간 row 내부를 가리키면 planner는 경로를 짜더라도 controller가 계속 row/stop zone과 부딪혀 recovery 루프로 들어간다.
- 여기에 `rotate_to_heading`이 켜져 있으면 직선 우선 전략보다 heading 보정이 먼저 나와, 저속 플랫폼에서 진전이 더 느려진다.
- 따라서 이번 MR은 더 무거운 perception이나 더 빠른 control loop를 추가하는 대신, 좌표계-경로-센서 주기를 모두 현재 하드웨어에 맞게 정렬하는 쪽을 택했다.

### 검증

- `python3 -m py_compile`
  - `agribot_navigation/agribot_navigation/patrol_node.py`
- `pytest`
  - `test_patrol_batching.py`
  - `test_mapping_patrol_plan.py`
  - 총 `15 passed`
- 실제 `patrol_only` 기준 실주행 로그에서 다음을 확인했다.
  - `Home`는 tolerance 내로 바로 skip
  - `Center North`는 분할 주행으로 완료
  - `East Center North`도 완료
  - `East Center South`도 세그먼트 기반으로 실제 진입해 이전보다 깊게 진행
- 이후 `Start occupied`가 남던 말단 구간을 줄이기 위해 inflation과 segment 길이를 마지막으로 미세조정했고, 그 뒤에는 빌드와 테스트까지만 다시 확인했다.

### 기대되는 동작 변화

- row 중간에서 옆 aisle로 무리하게 건너가려던 잘못된 경로가 줄어든다.
- 직선 세그먼트에서 제자리 회전부터 반복하는 비효율이 줄어든다.
- collision monitor와 planner의 false positive가 줄어들어 recovery 연쇄 빈도가 낮아진다.
- LiDAR, SLAM, EKF 갱신량을 줄여 노트북 연산량과 발열을 크게 늘리지 않으면서도 mapping run 안정성이 올라간다.

### 남은 한계

- 이번 MR은 `farm_world`의 mapping patrol geometry와 저발열 추종 안정화에 초점을 맞춘 것이다.
- 마지막 미세조정 이후 전체 맵 완주를 끝까지 다시 확인한 장기 runtime은 아직 남아 있다.
- 필요하면 다음 단계에서 connector 주변 waypoint를 추가 세분화하거나, `FrontStopZone`과 footprint를 row 폭 기준으로 한 번 더 정교화해야 한다.

### 커밋

- `8db5f75` 사각 농장 맵 자율 매핑의 좌표계와 저발열 추종을 재정렬한다

---

## 2026-03-23 - sweep_hybrid 매핑 전략 도입 및 저발열 로보락 스타일 튜닝

### 배경

- 기존 자율 매핑은 `patrol-first` 또는 `frontier-only` 중 하나에 가깝게 사용됐고, 로보락류처럼 "쭉 전진하다가 마지막에 빈 곳만 메우는" 흐름과는 거리가 있었다.
- 특히 온실 통로 구조에서는 주행의 주권이 frontier/recovery 쪽으로 갈수록
  - 같은 위치 재방문
  - 큰 회전 recovery
  - stop-and-go
  - planner 재시도
  가 늘어 체감 발열과 움직임 품질 모두 불리했다.
- 이번 MR의 목표는 계산을 무겁게 만들지 않으면서, 로보락 스타일에 더 가까운 `sweep_hybrid`를 넣는 것이었다.

### 이번 MR의 설계 요점

#### 1. launch 레벨에 `mapping_strategy`를 도입했다

- 수정 파일: `agribot_ws/src/agribot_navigation/launch/autonomous_mapping.launch.py`
- 새 인자:
  - `mapping_strategy:=sweep_hybrid|patrol_only|frontier_only`
- 기본값:
  - `sweep_hybrid`
- 전략별 동작:
  - `sweep_hybrid`
    - `use_patrol=true`
    - `patrol_autostart=true`
    - `use_frontier_explorer=true`
    - `frontier_autostart=false`
    - `patrol_completion_action=start_frontier_explorer`
    - `use_boundary_map=true`
  - `patrol_only`
    - patrol만 자동 시작
    - frontier는 아예 launch하지 않음
  - `frontier_only`
    - frontier만 자동 시작
    - patrol은 launch하지 않음

이렇게 한 이유는, 같은 launch 안에서 전략을 명시적으로 고정하지 않으면 "무엇이 주행의 주권을 갖는지"가 다시 모호해지기 때문이다.

#### 2. patrol 완료 후 frontier hole-fill을 자동 요청하도록 연결했다

- 수정 파일: `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_node.py`
- 추가 파라미터:
  - `completion_action`
  - `completion_start_service`
  - `completion_service_wait_sec`
- `sweep_hybrid`에서는 patrol이 끝나면 `mapping_explorer/start` 서비스를 호출한다.

즉, 상태 흐름은 아래처럼 된다.

```text
STARTUP
  -> PATROL SWEEP
  -> PATROL COMPLETED
  -> FRONTIER HOLE-FILL START REQUEST
  -> DONE
```

이 방식은 frontier를 완전히 없애지 않으면서도, 기본 주행 동선을 patrol이 가져가도록 만든다.

#### 3. patrol의 stop-and-go를 줄이기 위해 batched goal을 넣었다

- 수정 파일:
  - `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_node.py`
  - `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py`
  - `agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml`
- waypoint 메타데이터 확장:
  - `lane_id`
  - `batchable`
  - `observe_here`
- 구현 내용:
  - 관찰이 필요 없는 연속 구간은 `NavigateThroughPoses`로 묶어서 전송
  - inspect waypoint는 단일 `NavigateToPose` + 짧은 dwell 유지

현재 실제로 배치되는 대표 구간은 아래다.

- `home -> front_connector -> left_entry`
- `left_turn -> rear_connector -> right_entry`
- `right_turn -> front_connector -> home`

즉, transfer / entry / turn 같은 구간은 부드럽게 이어지고, inspect만 짧게 멈춘다.

#### 4. 매핑 profile에서 unknown 공간으로의 부드러운 진입을 허용했다

- 수정 파일: `agribot_ws/src/agribot_navigation/config/nav2_mapping_params.yaml`
- 변경:
  - `planner_server.GridBased.allow_unknown: false -> true`

이 변경은 계산량을 거의 늘리지 않으면서, 지도 초기 단계에서 patrol path가 너무 쉽게 막히는 문제를 줄여준다.
안전성은 기존의 local obstacle / collision monitor / waypoint 설계와 frontier boundary map이 유지하는 구조다.

#### 5. recovery를 짧고 가볍게 줄였다

- 수정 파일:
  - `agribot_ws/src/agribot_navigation/behavior_trees/navigate_to_pose_w_backout_recovery.xml`
  - `agribot_ws/src/agribot_navigation/behavior_trees/navigate_through_poses_w_backout_recovery.xml`
- 변경:
  - top-level `number_of_retries: 10 -> 3`
  - `Spin spin_dist: 3.14159 -> 1.20`
  - `DriveOnHeading dist_to_travel: 0.60 -> 0.45`
  - `Wait wait_duration: 2.0 -> 1.0`

의도는 간단하다.

- 큰 회전으로 환경을 다시 "생각"하는 시간을 줄이고
- 짧은 local escape만 남긴다
- route-level 판단은 상위 전략이 맡게 한다

#### 6. SLAM 반응성을 조금 되살렸다

- 수정 파일: `agribot_ws/src/agribot_navigation/config/slam_mapping.yaml`
- 변경:
  - `map_update_interval: 4.0 -> 2.0`

이건 CPU를 크게 올리지 않는 선에서만 조정했다.
이미 patrol-first와 reduced recovery로 낭비성 움직임과 replanning이 줄기 때문에, 이 정도 반응성 복원은 전체 발열을 크게 밀어올리지 않는다고 판단했다.

### 기대 동작 변화

- 기본 실행만으로 `sweep_hybrid`가 동작한다.
- 초반 주행은 patrol sweep가 맡기 때문에 온실 통로를 따라 더 자연스럽게 "쭉 가는" 느낌이 나온다.
- transfer/turn 구간은 `NavigateThroughPoses` 배치 주행으로 이전보다 덜 끊긴다.
- inspect 지점은 멈추되, 그 외 구간의 stop-and-go는 줄어든다.
- patrol 완료 후 필요한 경우에만 frontier hole-fill이 시작되므로, frontier가 주력 엔진일 때보다 불필요한 재판단이 줄어든다.

### 발열/연산량 관점에서의 판단

이번 MR은 "알고리즘 추가"라기보다 "상위 전략 재배치"에 가깝다.

- 늘어나는 것:
  - patrol completion 후 frontier start 요청 로직
  - batch goal 판단 로직
  - `map_update_interval` 일부 회복
- 줄어드는 것:
  - 큰 recovery spin
  - 잦은 planner retry
  - stop-and-go
  - early frontier-led oscillation

즉, CPU를 무겁게 먹는 새로운 인식/최적화 모듈을 넣은 게 아니라, 기존 노드를 더 적절한 순서로 쓰게 만든 것이다.
그래서 체감 발열은 크게 늘리지 않으면서 동선 품질을 올리는 방향으로 설계했다.

### 검증

아래 항목을 직접 확인했다.

```bash
cd /home/ssafy/SSAFY/S14P21A602/agribot_ws

python3 -m py_compile \
  src/agribot_navigation/launch/autonomous_mapping.launch.py \
  src/agribot_navigation/agribot_navigation/patrol_node.py \
  src/agribot_navigation/agribot_navigation/patrol_config.py

python3 -m pytest \
  src/agribot_navigation/test/test_frontier_explorer.py \
  src/agribot_navigation/test/test_patrol_batching.py

source /opt/ros/jazzy/setup.bash
colcon build --packages-select agribot_navigation --symlink-install
```

결과:

- `py_compile` 통과
- `pytest` 기준 `17 passed`
- `agribot_navigation` 단독 빌드 통과

### 추가된 테스트

- `agribot_ws/src/agribot_navigation/test/test_patrol_batching.py`
  - non-observe 연속 구간이 batch goal로 묶이는지 확인
  - inspect waypoint에서는 batch가 끊기는지 확인

### 변경 파일

- `agribot_ws/src/agribot_navigation/launch/autonomous_mapping.launch.py`
- `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_node.py`
- `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py`
- `agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml`
- `agribot_ws/src/agribot_navigation/config/nav2_mapping_params.yaml`
- `agribot_ws/src/agribot_navigation/config/slam_mapping.yaml`
- `agribot_ws/src/agribot_navigation/behavior_trees/navigate_to_pose_w_backout_recovery.xml`
- `agribot_ws/src/agribot_navigation/behavior_trees/navigate_through_poses_w_backout_recovery.xml`
- `agribot_ws/src/agribot_navigation/test/test_patrol_batching.py`
- `docs/현 프로젝트 진행 상황 및 명령어 모음.md`

### 후속 작업 제안

- inspect waypoint 다발 구간을 lane driver 또는 관측 전용 task executor로 바꿔, 같은 레인의 inspect도 더 부드럽게 잇기
- Nav2 global costmap에 boundary map을 정식 layer/filter로 넣어 `allow_unknown=true`를 더 안전하게 쓰기
- patrol 완료 후 frontier start뿐 아니라 `known_ratio` 기준으로 hole-fill 필요 여부를 선판단하는 경량 gate 추가

---

## 배경

기존 상태에서는 `ros2 topic list` 상 토픽 이름은 보이지만, 실제 메시지가 안정적으로 수신되지 않는 문제가 있었습니다.

확인해보니 원인은 크게 2가지였습니다.

- Gazebo가 기본적으로 pause 상태로 시작되어 토픽만 생성되고 센서 메시지는 흐르지 않던 문제
- 카메라 관련 토픽을 단일 `ros_gz_bridgeparameter_bridge`에 한 번에 묶어둘 경우 ROS 측 수신이 불안정하던 문제

즉, 단순히 "토픽이 보인다" 수준이 아니라 실제 `echo--once`와 발행 주기까지 확인 가능한 상태로 런치를 보완할 필요가 있었습니다.

---

## 주요 변경 사항

### 1. Gazebo 자동 재생으로 변경
- `spawn_agribot.launch.py`에서 `gz_args`에 `-r`을 추가했습니다.
- 이제 `ros2 launch agribot_bringup simulation.launch.py`실행 시 Gazebo가 바로 재생 상태로 올라옵니다.
- 수동으로 GUI에서 Play 버튼을 누르지 않아도 센서 메시지가 흐르도록 수정했습니다.

### 2. 브리지 구성을 역할별로 분리
기존에는 여러 센서 토픽을 단일 `parameter_bridge`에 몰아서 연결하고 있었습니다.

이번 MR에서는 아래처럼 분리했습니다.

- 상태 브리지
- `/clock`
- `/agribot/imu`
- `/cmd_vel`
- `/odom`
- 카메라 정보 브리지
- `/agribot/camera/camera_info`
- LiDAR 브리지
- `/agribot/lidar/scan`
- 이미지 전용 브리지
- `/agribot/camera/image`
- `/agribot/camera/depth_image`

카메라 이미지 계열은 `ros_gz_bridge parameter_bridge` 대신 `ros_gz_image image_bridge`를 사용하도록 변경했습니다.

### 3. 패키지 의존성 추가
- `agribot_description/package.xml`에 `ros_gz_image` 실행 의존성을 추가했습니다.
- 팀원이 동일한 의존성을 기준으로 실행할 수 있도록 반영했습니다.

### 4. 센서 점검 결과 문서 추가
- 점검 결과를 [S14P-31_센서_출력_점검.md] 문서로 정리했습니다.
- 문서에는 다음 내용이 포함되어 있습니다.
- 실행 환경
- 실행/검증 명령
- 센서 점검표
- bridge 수정 포인트
- 미해결 이슈
- 후속 작업 제안

---

## 검증 방법

아래 순서로 직접 검증했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
source install/setup.bash
export HOME=/tmp/codex-home
export ROS_LOG_DIR=/tmp/ros_logs
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export DRI_PRIME=1

colcon build --symlink-install --packages-select agribot_description agribot_bringup
ros2 launch agribot_bringup simulation.launch.py
```

실행 후 아래 명령으로 확인했습니다.

```bash
ros2 topic list
ros2 topic echo --once --field header /agribot/camera/image
ros2 topic echo --once --field header /agribot/camera/depth_image
ros2 topic echo --once /agribot/camera/camera_info
ros2 topic echo --once /agribot/imu
ros2 topic echo --once /odom
ros2 topic echo --once /clock
ros2 topic echo --once --field header /agribot/lidar/scan

gz topic -e -n 3 -t /agribot/camera/camera_info
gz topic -e -n 3 -t /agribot/imu
gz topic -e -n 3 -t /odom

ros2 topic hz /agribot/imu
```

———

## 검증 결과

### 정상 수신 확인

- /agribot/camera/image
- /agribot/camera/depth_image
- /agribot/camera/camera_info
- /agribot/imu
- /odom
- /clock

### 미수신

- /agribot/lidar/scan

### 주기 확인 결과

- CameraInfo: GZ 헤더 기준 0.925s -> 0.991s, 약 15.15Hz
- IMU: GZ 헤더 기준 0.890s -> 0.900s -> 0.910s, 100Hz
- Odometry: GZ 헤더 기준 0.884s -> 0.918s -> 0.952s, 약 29.4Hz

즉, 카메라/IMU/Odometry는 설정된 센서 주기 기준으로는 정상 출력으로 판단했습니다.

———

## 확인된 이슈

### 1. LiDAR 미수신

- /agribot/lidar/scan은 메인 런치 기준에서도 미수신이었습니다.
- 별도 브리지로 분리해도 미수신이어서, 단순 bridge 설정 문제로 단정하기는 어렵습니다.
- 현재는 gpu_lidar 센서 자체, 렌더링 계층, 혹은 시뮬레이터 성능 저하 영향 가능성을 우선 의심하고 있습니다.

### 2. Gazebo 실시간 성능 저하

- ros2 topic hz /agribot/imu 기준 실제 벽시계 발행률은 약 0.635Hz 수준이었습니다.
- 반면 GZ 메시지 헤더 기준 시뮬레이션 내부 시간은 IMU 100Hz로 정상 증가했습니다.
- 즉, 센서 설정값이 틀린 것보다는 Gazebo가 매우 느리게 돌고 있는 상태입니다.

### 3. 물리 충돌 병목 의심

Gazebo 실행 중 아래 경고가 반복되었습니다.

- ODE Message 2: Trimesh-trimesh contact hash table bucket overflow

코드상 다음 모델들이 시각 메쉬를 그대로 collision mesh로 사용 중이었습니다.

- agribot_description/models/greenhouse/model.sdf
- agribot_description/models/tomato_plant/model.sdf
- agribot_description/models/tomato/model.sdf

월드에 이 메쉬 기반 모델이 여러 개 배치되어 있어 물리 계산 부하가 큰 상태로 보입니다.

———

## 기대 효과

- 팀원이 동일한 런치 명령만으로 카메라, IMU, /odom, /clock 출력 상태를 바로 재현할 수 있습니다.
- S14P-31 결과물인 센서 점검표와 bridge 수정 포인트가 문서로 남아 이후 티켓에서 참조 가능합니다.
- 다음 작업인 S14P-32 프레임 정합, S14P-33 주행/오도메트리 보정 전에 센서 입출력 기반이 정리되었습니다.

———

## 후속 작업 제안

- LiDAR 센서를 gpu_lidar 외 다른 센서 타입으로 바꿔 비교테스트
  헤드리스 실행 시 LiDAR 발행 여부 비교
- greenhouse, tomato_plant, tomato collision geometry를 단순화해서 실시간 성능 개선
- 이후 S14P-32에서 camera_link, imu_link, base_link,odom 프레임 정합 진행

## 변경 파일

- agribot_ws/src/agribot_description/launch/spawn_agribot.launch.py
- agribot_ws/src/agribot_description/package.xml
- S14P-31_센서_출력_점검.md


[S14P-32] 로봇 부품 좌표 정합 및 TF 트리 구성

## 개요

S14P-32 로봇 부품 좌표 정합을 진행했습니다.

이번 MR의 목적은 다음 3가지입니다.

1. `map`, `odom`, `base_link`, `camera_link`, `lidar_link`, `imu_link`의 관계를 ROS 2 `tf` 기준으로 고정한다.
2. Gazebo에서 이미 발행 중인 `/odom`을 RViz2와 `tf2`가 바로 사용할 수 있도록 `/tf`로 연결한다.
3. 다음 작업인 Nav2, localization, perception이 같은 프레임 기준선을 공유할 수 있도록 문서와 런치 설정을 함께 남긴다.

이번 작업으로 센서 링크의 정적 변환과 `odom -> base_link` 동적 변환을 ROS 쪽에서 확인 가능한 상태로 정리했고, 임시 `map -> odom` 기준점도 함께 추가했습니다.

---

## 배경

S14P-31까지 진행한 시점에는 카메라, IMU, `/odom` 토픽은 확인되었지만, TF 트리 기준으로는 아직 빈 부분이 있었습니다.

구체적으로는 아래 문제가 있었습니다.

- Gazebo DiffDrive가 `/odom` 메시지는 발행하지만 ROS `tf` 트리에서 `odom -> base_link`가 보장되지 않음
- `camera_link`, `lidar_link`, `imu_link`의 상대 위치가 SDF 안에는 정의되어 있지만 ROS `tf_static`으로는 노출되지 않음
- `map` 기준 프레임이 없어 이후 RViz2, Nav2, localization 연동 시 기준점이 일관되지 않을 수 있음

즉, 센서 토픽이 살아 있는 것과 별개로 "로봇 몸통과 센서가 서로 어디에 붙어 있는가"를 ROS 2 표준 방식으로 정리할 필요가 있었습니다.

---

## 주요 변경 사항

### 1. 정적 TF 체인 추가
- `spawn_agribot.launch.py`에 `static_transform_publisher`를 추가했습니다.
- 아래 4개 관계를 런치 시 즉시 발행하도록 구성했습니다.
- `map -> odom`
- `base_link -> camera_link`
- `base_link -> lidar_link`
- `base_link -> imu_link`

센서 링크 위치 값은 기존 `model.sdf`에 정의된 pose를 그대로 사용했습니다.

### 2. `/odom`을 `/tf`로 재발행하는 노드 추가
- `agribot_description/odom_tf_broadcaster.py`를 새로 추가했습니다.
- 이 노드는 `/odom`을 구독해 `odom -> base_link` 변환을 `tf2_ros.TransformBroadcaster`로 발행합니다.
- Gazebo 오도메트리와 ROS TF 소비자 사이를 연결하는 최소 브리지 역할입니다.

### 3. 런치 인자 추가
- `publish_map_to_odom_tf` 인자를 추가했습니다.
- 현재는 기본값 `true`로 두어 임시 항등 `map -> odom`을 발행합니다.
- 이후 SLAM 또는 localization이 실제 `map -> odom`을 발행할 때는 `false`로 끌 수 있게 했습니다.

### 4. 패키지 의존성과 엔트리포인트 보강
- `agribot_description/package.xml`에 `rclpy`, `tf2_ros`, `nav_msgs`, `geometry_msgs`, `launch_ros` 의존성을 추가했습니다.
- `setup.py`에 `odom_tf_broadcaster` 콘솔 엔트리포인트를 등록했습니다.

### 5. 좌표 정합 문서 추가
- 작업 결과를 `S14P-32_로봇_부품_좌표_정합.md`에 정리했습니다.
- 문서에는 프레임 구조, 좌표 기준, 적용 내용, 검증 명령, 현재 제한 사항을 남겼습니다.

---

## 검증 방법

아래 순서로 직접 검증했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --packages-select agribot_description
source install/setup.bash
export HOME=/tmp/codex-home
export ROS_LOG_DIR=/tmp/ros_logs

ros2 launch agribot_description spawn_agribot.launch.py
```

실행 후 아래 명령으로 확인했습니다.

```bash
ros2 topic list | rg "^/tf($|_static$)"
timeout 10s ros2 topic echo --once /tf_static
timeout 12s ros2 topic echo --once /tf
timeout 10s ros2 run tf2_ros tf2_echo base_link imu_link
```

---

## 검증 결과

### 정상 확인

- `/tf`
- `/tf_static`
- `base_link -> camera_link` 정적 변환 발행
- `base_link -> imu_link` 정적 변환 해석
- `odom -> base_link` 동적 변환 발행

### 확인 로그 요약

- `ros2 topic list`에서 `/tf`, `/tf_static` 생성 확인
- `/tf_static`에서 `base_link -> camera_link` 변환 확인
  - translation: `(0.2, 0.0, 0.1)`
- `/tf`에서 `odom -> base_link` 변환 확인
  - translation: `(약 0, 0, 0)`
- `tf2_echo base_link imu_link`에서 변환 확인
  - translation: `(0.0, 0.0, 0.05)`

즉, 최소 기준선인 `odom -> base_link -> sensor_links` 구조는 ROS 2 쪽에서 확인 가능한 상태로 정리했습니다.

---

## 확인된 이슈 및 주의사항

### 1. `map -> odom`은 임시 기준점

- 이번 MR의 `map -> odom`은 localization이 아직 없어서 넣은 임시 항등 변환입니다.
- 이후 Nav2 / SLAM / AMCL이 실제 `map -> odom`을 발행하면 반드시 중복 발행되지 않도록 꺼야 합니다.
- 런치 인자 `publish_map_to_odom_tf:=false`로 비활성화할 수 있도록 구성했습니다.

### 2. LiDAR 발행 문제는 별도 이슈

- 이번 MR은 좌표 정합 범위이며, `lidar_link` 프레임만 먼저 고정했습니다.
- `S14P-31`에서 확인된 `/agribot/lidar/scan` 미수신 문제는 여전히 남아 있습니다.

### 3. Gazebo 물리 경고는 그대로 존재

- 실행 중 `Trimesh-trimesh contact hash table bucket overflow` 경고는 계속 발생했습니다.
- 현재 TF 구성 검증과는 별개지만, Gazebo 실시간 성능 저하 원인 후보로 남아 있습니다.

---

## 기대 효과

- 이후 RViz2, tf2_tools, Nav2가 같은 프레임 이름과 센서 상대 위치를 기준으로 동작할 수 있습니다.
- 팀원이 동일한 런치만 실행해도 `base_link` 기준 센서 좌표와 `odom` 기준 본체 좌표를 재현할 수 있습니다.
- 다음 티켓에서 localization이나 SLAM을 붙일 때 기존 프레임 체인을 다시 정의하는 비용이 줄어듭니다.

---

## 후속 작업 제안

- `S14P-33`에서 바퀴 파라미터와 `/odom` 응답을 실제 주행 기준으로 보정
- localization 또는 SLAM 도입 시 `map -> odom` 임시 정적 TF 제거
- LiDAR 미발행 원인 분석과 충돌체 단순화 작업 병행

## 변경 파일

- agribot_ws/src/agribot_description/launch/spawn_agribot.launch.py
- agribot_ws/src/agribot_description/package.xml
- agribot_ws/src/agribot_description/setup.py
- agribot_ws/src/agribot_description/agribot_description/odom_tf_broadcaster.py
- S14P-32_로봇_부품_좌표_정합.md


[S14P-33] 주행·오도메트리 보정 및 calibration world 추가

## 개요

S14P-33 주행·오도메트리 보정을 진행했습니다.

이번 MR의 목적은 다음 4가지입니다.

1. `agribot`의 Differential Drive 파라미터를 실제 바퀴 치수 기준으로 재점검하고, teleop 직진/회전 응답을 과하지 않게 보정한다.
2. `/odom` 발행 주기를 상향해 이후 Nav2, localization, RViz 기준선으로 바로 쓸 수 있는 수준으로 정리한다.
3. `cmd_vel`이 끊긴 뒤에도 마지막 속도 명령이 남아 로봇이 계속 움직이는 문제를 watchdog으로 막는다.
4. 기본 `farm_world`의 무거운 삼각메시 충돌 병목과 분리된 전용 calibration world를 추가해 주행 검증을 재현 가능하게 만든다.

이번 작업으로 `/cmd_vel -> diff drive -> /odom -> /tf` 흐름을 안정화했고, 실제 직진 테스트를 반복할 수 있는 경량 검증 환경과 작업 문서까지 함께 남겼습니다.

---

## 배경

S14P-32까지 진행한 시점에는 TF 트리는 정리됐지만, 실제 주행 기준으로는 아직 두 가지 문제가 남아 있었습니다.

- 기본 `farm_world` 실행 시 `Trimesh-trimesh contact hash table bucket overflow` 경고가 반복되어 Gazebo 물리 성능이 크게 흔들림
- Gazebo DiffDrive가 마지막 속도 명령을 유지해, 짧게 준 `/cmd_vel` 이후에도 로봇이 계속 전진할 수 있음

즉, 바퀴 파라미터만 수치상 맞는지 보는 수준이 아니라 "직진 명령이 안정적으로 반영되고, `/odom`이 정상 증가하며, 명령이 끝나면 멈추는가"까지 같이 보정할 필요가 있었습니다.

---

## 주요 변경 사항

### 1. DiffDrive 파라미터 보정
- `agribot_ws/src/agribot_description/models/agribot/model.sdf`를 수정했습니다.
- 기존 wheel geometry를 다시 확인한 결과:
  - `wheel_separation = 0.35`
  - `wheel_radius = 0.06`
- 위 두 값은 실제 wheel pose / collision과 일치하므로 유지했습니다.
- 대신 동적 응답을 아래처럼 보수적으로 조정했습니다.
  - `max_linear_velocity = ±0.40`
  - `max_linear_acceleration = ±0.60`
  - `max_linear_jerk = ±1.50`
  - `max_angular_velocity = ±1.20`
  - `max_angular_acceleration = ±1.20`
  - `max_angular_jerk = ±2.50`
- `odom_publish_frequency`는 `30 -> 50 Hz`로 올렸습니다.

### 2. `cmd_vel` watchdog 추가
- `agribot_description/cmd_vel_watchdog.py`를 새로 추가했습니다.
- 입력은 `/cmd_vel`, 출력은 `/cmd_vel_safe`입니다.
- watchdog은 `20 Hz`로 명령을 재발행하고, `0.25초` 동안 새 명령이 없으면 zero twist를 보냅니다.
- DiffDrive 플러그인의 입력 토픽도 `cmd_vel_safe`로 바꿨습니다.

### 3. 런치 경로 수정
- `spawn_agribot.launch.py`에서 Gazebo 브리지를 `/cmd_vel_safe` 기준으로 연결했습니다.
- 같은 런치에서 `cmd_vel_watchdog` 노드를 함께 실행하도록 추가했습니다.
- 이로써 teleop와 이후 Nav2는 계속 `/cmd_vel`만 사용하고, Gazebo 쪽에는 timeout이 적용된 안전한 명령만 전달됩니다.

### 4. 오도메트리 전용 calibration world 추가
- `agribot_ws/src/agribot_description/worlds/odometry_calibration_world.sdf`를 새로 추가했습니다.
- 구성은 아래처럼 최소화했습니다.
  - ground plane
  - sun light
  - 중심선 확인용 마커 3개
  - `agribot` 단일 배치
- 목적은 온실/식물/토마토 mesh collision 부하와 분리된 상태에서 주행 기준선을 검증하는 것입니다.

### 5. 종료 안정성 및 문서 정리
- `odom_tf_broadcaster.py`는 `KeyboardInterrupt`와 중복 shutdown을 정상 처리하도록 정리했습니다.
- 작업 결과를 `S14P-33_주행_오도메트리_보정.md`에 문서화했습니다.
- 메인 가이드의 teleop 예시도 현재 구현 기준인 `/cmd_vel`로 맞췄습니다.

---

## 검증 방법

아래 순서로 직접 검증했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --packages-select agribot_description
source install/setup.bash
export HOME=/tmp/codex-home
export ROS_LOG_DIR=/tmp/ros_logs

ros2 launch agribot_bringup simulation.launch.py \
  world:=/home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_description/worlds/odometry_calibration_world.sdf
```

실행 후 아래 명령으로 확인했습니다.

```bash
timeout 10s ros2 topic hz /odom
timeout 8s ros2 topic echo --once --field transforms /tf
timeout 3s ros2 topic pub -r 5 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}, angular: {z: 0.0}}"
timeout 8s ros2 topic echo --once --field transforms /tf
```

---

## 검증 결과

### 정상 확인

- `/odom`이 약 `50 Hz`로 안정적으로 발행됨
- 직진 명령 후 `odom -> base_link`의 `x`가 정상 증가함
- `y` 편차와 yaw drift는 측정 오차 수준으로 작음
- 명령 입력이 끊긴 뒤 위치가 계속 누적되지 않아 watchdog 정지가 동작함
- calibration world에서는 기존 `farm_world`에서 보이던 ODE collision 경고가 발생하지 않음

### 확인 로그 요약

- `timeout 10s ros2 topic hz /odom`
  - 평균 약 `49.09 ~ 49.13 Hz`
- 직진 전 `odom -> base_link`
  - `x ≈ 0`
  - `y ≈ 0`
- `3초` 동안 `linear.x = 0.2` 명령 전송 후
  - `x = 0.5510845081793189`
  - `y = -1.145172846324149e-12`
- watchdog timeout 이후 재확인
  - `x = 0.5901500771292297`
  - 재재확인: `x = 0.5734698196138319`

즉, 전진 명령에 따라 `/odom`은 정상 증가했고, 명령 종료 뒤에는 로봇이 계속 가속되지 않는 수준으로 정리됐습니다.

---

## 확인된 이슈 및 주의사항

### 1. 기본 `farm_world`는 여전히 무거움

- 이번 MR은 주행 기준선 보정 범위이며, 기본 `farm_world`의 triangle-mesh collision 단순화까지는 포함하지 않았습니다.
- 따라서 실제 자율주행 전체 통합 검증은 이후 별도 성능 개선 작업이 필요합니다.

### 2. calibration world는 검증용 기준선

- 이번에 추가한 `odometry_calibration_world.sdf`는 S14P-33 검증용 최소 환경입니다.
- 실제 농장 월드에서의 최종 주행 안정화는 이후 환경 충돌체 정리와 함께 다시 확인해야 합니다.

### 3. watchdog은 의도적인 안전 장치

- 짧은 테스트 명령 뒤에도 로봇이 계속 움직이던 기존 거동을 막기 위해 넣은 안전 장치입니다.
- 이후 Nav2 연동 시에도 `/cmd_vel`만 유지하면 같은 구조를 그대로 사용할 수 있습니다.

---

## 기대 효과

- teleop 직진 테스트를 더 작은 편차로 반복할 수 있습니다.
- `/odom` 발행 주기가 50 Hz 수준으로 올라가 후속 localization/Nav2 기준선으로 쓰기 쉬워졌습니다.
- stale `cmd_vel`로 인한 계속 주행 문제가 사라져 검증과 디버깅이 안전해졌습니다.
- 무거운 농장 월드와 분리된 calibration world 덕분에 주행/오도메트리 이슈를 재현하고 비교하기 쉬워졌습니다.

---

## 후속 작업 제안

- `farm_world`의 `greenhouse`, `tomato_plant`, `tomato` collision 단순화
- calibration world 기준값을 바탕으로 실제 농장 월드에서도 직진 편차 재검증
- 이후 Nav2 연동 시 watchdog timeout과 local planner 응답의 궁합 확인

## 변경 파일

- agribot_ws/src/agribot_description/models/agribot/model.sdf
- agribot_ws/src/agribot_description/launch/spawn_agribot.launch.py
- agribot_ws/src/agribot_description/agribot_description/cmd_vel_watchdog.py
- agribot_ws/src/agribot_description/agribot_description/odom_tf_broadcaster.py
- agribot_ws/src/agribot_description/setup.py
- agribot_ws/src/agribot_description/worlds/odometry_calibration_world.sdf
- docs/S14P-33_주행_오도메트리_보정.md
- docs/수확해조_프로젝트_개발_계획_및_협업_가이드.md


[S14P-34] 비닐하우스 단일 구역 메타데이터 추가

## 변경 내용

- `agribot_navigation/config/zones.yaml` 파일 추가
- 현재 시뮬레이션의 비닐하우스 전체를 단일 구역 `greenhouse_01`로 정의
- 구역의 대표 위치와 기본 범위를 함께 명시해 후속 작업에서 공통 `zone_id`를 사용하도록 기준 마련

## 작업 이유

- 현재 비닐하우스 오브젝트는 1개뿐이지만, 프로젝트 문서와 ROS 인터페이스가 이미 `zone_id`를 전제로 설계되어 있음
- 이후 `plants.yaml`, `iot_devices.yaml`, 환경 센서 데이터, 백엔드 저장 구조에서 같은 `zone_id`를 재사용할 수 있도록 최소 메타데이터가 필요함
- 따라서 이번 작업은 "여러 구역 분할"이 아니라 "단일 구역 명시"에 초점을 맞춤

## 참고

- 현재는 운영 구역 1개만 정의한 최소 버전임
- 추후 실제 지도 생성 결과에 맞춰 bounds 값은 조정 가능

## 검증

- `zones.yaml` 파일이 작업 트리에 정상 추가된 것 확인
- 별도 빌드나 런타임 테스트는 수행하지 않음


[S14P-35] 식물·토마토 canonical ID 정의 및 CropStatus 식별 필드 확장

## 개요

S14P-35 식물·토마토 ID 정의를 진행했습니다.

이번 MR의 목적은 다음 3가지입니다.

1. `farm_world.sdf`에 배치된 개별 식물과 토마토 과실에 대해 외부 시스템에서 재사용 가능한 canonical ID를 정의한다.
2. Gazebo 모델 이름과 별개로 perception, control, IoT, backend가 공통으로 사용할 수 있는 ID 계약을 확정한다.
3. `CropStatus.msg`에 식별 필드를 반영해 이후 상태 발행 노드들이 같은 기준으로 `crop_id`, `plant_id`, `tomato_id`, `zone_id`를 실어 보낼 수 있게 한다.

이번 작업으로 시뮬레이션 월드 기준 식물 10개, 토마토 10개의 canonical ID를 단일 레지스트리로 정리했고, ROS 인터페이스까지 그 기준을 수용하도록 확장했습니다.

---

## 배경

S14P-34에서 `greenhouse_01` 단일 구역 메타데이터를 먼저 정의했지만, 그 위에서 실제 개별 작물 단위를 구분하는 규칙은 아직 비어 있었습니다.

구체적으로는 아래 문제가 있었습니다.

- Gazebo include name인 `tomato_plant_1`, `tomato_fruit_9`는 시뮬레이터 내부 이름일 뿐이라 외부 시스템 계약 ID로 그대로 쓰기 애매함
- perception, control, backend가 각자 다른 이름 규칙을 쓰기 시작하면 이후 수확 기록, 상태 이력, 알림 연계 시 식별자 정합이 깨질 수 있음
- 기존 `CropStatus.msg`는 작물 이름과 상태값만 담고 있어 개별 식물/과실 단위 추적이 어려움

즉, 단순히 문서에 이름만 적는 수준이 아니라 "월드 배치 객체 -> canonical ID -> ROS 메시지"까지 이어지는 공통 기준선을 먼저 고정할 필요가 있었습니다.

---

## 주요 변경 사항

### 1. 식물/토마토 개체 레지스트리 추가
- `agribot_ws/src/agribot_description/config/crop_instances.yaml` 파일을 새로 추가했습니다.
- 이 파일은 현재 시뮬레이션 기준 식물/토마토 ID의 single source of truth 역할을 합니다.
- 포함 내용은 아래와 같습니다.
  - `greenhouse_01 -> gh01` zone prefix 규칙
  - 식물 ID 패턴: `gh01_plant_<nn>`
  - 토마토 ID 패턴: `<plant_id>_tomato_<nn>`
  - 식물 10개와 토마토 10개의 canonical ID
  - 각 개체의 Gazebo 모델명
  - 각 개체의 pose 정보
  - 토마토 과실의 부모 식물 ID

### 2. Gazebo 이름과 외부 계약 ID를 분리
- `tomato_plant_1`, `tomato_fruit_9` 같은 월드 모델명은 그대로 보존하되, 외부 계약에서는 canonical ID를 사용하도록 원칙을 명시했습니다.
- 즉, 월드 모델명은 `source_model_name`으로만 유지하고, 서비스 간 데이터 연동은 아래 규칙을 따르도록 정리했습니다.
  - 식물 상태: `crop_id == plant_id`
  - 토마토 상태: `crop_id == tomato_id`

### 3. `CropStatus.msg` 확장
- `agribot_ws/src/agribot_interfaces/msg/CropStatus.msg`를 확장했습니다.
- 새로 추가한 필드는 아래와 같습니다.
  - `std_msgs/Header header`
  - `string crop_id`
  - `string zone_id`
  - `string plant_id`
  - `string tomato_id`
  - `string source_model_name`
  - `string crop_type`
- 기존 상태값 필드인 `crop_name`, `health_score`, `growth_stage`, `needs_water`, `ready_to_harvest`, `position`은 유지했습니다.
- 이 변경으로 이후 perception/control 노드가 식별자와 상태값을 함께 발행할 수 있는 기반을 만들었습니다.

### 4. 작업 문서 추가
- `docs/S14P-35_식물_토마토_ID_정의.md` 문서를 새로 추가했습니다.
- 문서에는 아래 내용을 정리했습니다.
  - ID 정의 원칙
  - 식물 ID 매핑표
  - 토마토 ID 매핑표
  - `CropStatus` 사용 규칙
  - 범위 제외 항목
  - 검증 방법
  - 현재 제한 사항

### 5. README 메시지 설명 갱신
- `README.md`의 `CropStatus.msg` 설명을 현재 인터페이스 기준으로 업데이트했습니다.
- 이제 문서상으로도 `crop_id`, `zone_id`, `plant_id`, `tomato_id`가 포함된 메시지임을 바로 확인할 수 있습니다.

---

## ID 규칙 요약

### 식물 ID
- 형식: `gh01_plant_<nn>`
- 예시: `gh01_plant_01`

### 토마토 ID
- 형식: `<plant_id>_tomato_<nn>`
- 예시: `gh01_plant_01_tomato_01`

### 사용 규칙
- 식물 상태 메시지에서는 `crop_id == plant_id`
- 토마토 상태 메시지에서는 `crop_id == tomato_id`
- 부모 식물 관계는 `plant_id` 필드로 유지
- Gazebo 월드 이름은 `source_model_name`으로만 유지

---

## 검증 방법

아래 순서로 직접 검증했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --packages-select agribot_interfaces agribot_description
```

추가로 아래 파일 내용을 직접 확인했습니다.

```bash
sed -n '1,220p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_description/config/crop_instances.yaml
sed -n '1,120p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_interfaces/msg/CropStatus.msg
sed -n '1,220p' ~/SSAFY/S14P21A602/docs/S14P-35_식물_토마토_ID_정의.md
```

---

## 검증 결과

### 정상 확인

- 식물 10개에 대한 canonical `plant_id` 정의 완료
- 토마토 10개에 대한 canonical `tomato_id` 정의 완료
- 각 토마토 과실에 대한 부모 `plant_id` 매핑 완료
- `CropStatus.msg`가 새 식별 필드 기준으로 정상 생성되도록 빌드 통과
- `agribot_description`, `agribot_interfaces` 패키지 빌드 통과

### 확인 로그 요약

- `colcon build --symlink-install --packages-select agribot_interfaces agribot_description` 실행 결과 최종 성공
- 빌드 중 `agribot_interfaces`의 예전 생성물이 symlink 단계와 충돌하는 이슈가 1회 있었음
- stale 생성물 경로를 정리한 뒤 재빌드해 정상 완료함

즉, 이번 MR은 문서 정의에 그치지 않고 실제 ROS 인터페이스 레벨에서 재사용 가능한 ID 계약으로 반영된 상태입니다.

---

## 확인된 이슈 및 주의사항

### 1. 현재 토마토는 식물당 1개 기준
- 현재 `farm_world.sdf`에는 식물당 토마토 과실 1개씩만 연결했습니다.
- 이후 과실 수가 늘어나면 같은 `plant_id` 아래 `tomato_02`, `tomato_03` 방식으로 확장하면 됩니다.

### 2. `unhealthy_leaf_1`은 이번 범위 제외
- 현재 월드에 있는 `unhealthy_leaf_1`은 식물/토마토 ID 정의 범위에 포함하지 않았습니다.
- 잎 개체 식별이 필요해지면 별도 `leaf_scan_id` 체계를 추가하는 편이 맞습니다.

### 3. perception 매핑 로직은 아직 없음
- 이번 MR은 ID 계약과 레지스트리 정의까지가 범위입니다.
- 실제 인식 결과를 nearest pose 또는 world model name 기준으로 `crop_id`에 연결하는 perception 로직은 후속 작업이 필요합니다.

---

## 기대 효과

- 이후 perception, mission manager, harvest 로직, backend 저장 구조가 같은 식별자를 기준으로 연결될 수 있습니다.
- 수확 기록이나 작물 상태 이력을 저장할 때 Gazebo 모델명 대신 안정적인 canonical ID를 사용할 수 있습니다.
- 현재 단일 구역 메타데이터(`greenhouse_01`)와 개별 작물 식별자가 결합돼 backend 스키마 확장 시 일관성을 유지하기 쉬워집니다.
- 향후 월드 객체가 늘어나더라도 같은 규칙으로 식물/과실 ID를 확장할 수 있습니다.

---

## 후속 작업 제안

- perception 노드에서 탐지 결과를 `crop_instances.yaml` 기준으로 `crop_id`에 매핑하는 로직 추가
- harvest / disease / history 저장 구조에서 `crop_id`, `plant_id`, `tomato_id`를 공통 키로 사용하도록 정리
- 잎 스캔 객체나 병해 샘플에 대한 별도 개체 ID 체계 필요 여부 검토

## 변경 파일

- agribot_ws/src/agribot_description/config/crop_instances.yaml
- agribot_ws/src/agribot_interfaces/msg/CropStatus.msg
- docs/S14P-35_식물_토마토_ID_정의.md
- README.md


[S14P-36] 지도 생성 설정

## 개요

S14P-36 지도 생성 설정을 진행했습니다.

이번 MR의 목적은 다음 3가지입니다.

1. AgriBot이 Gazebo 시뮬레이션 안에서 LiDAR를 기준으로 지도를 만들 수 있도록 `slam_toolbox` 기본 구성을 준비한다.
2. 팀원이 한 번의 launch 명령으로 시뮬레이션, SLAM, RViz를 함께 올릴 수 있도록 mapping 전용 런치를 추가한다.
3. 이후 티켓인 S14P-37 지도 저장, S14P-203 localization, S14P-204 Nav2 파라미터 조정이 바로 이어질 수 있도록 패키지 자원과 의존성을 정리한다.

이번 작업으로 `agribot_navigation` 패키지에 mapping 설정 파일, launch 파일, RViz 화면 설정, map 저장 디렉터리 안내를 추가했고, 패키지 빌드까지 통과하는 상태로 정리했습니다.

---

## 배경

기존 `agribot_navigation` 패키지는 `zones.yaml`만 있는 골격 상태였고, 실제로 지도 생성을 시작할 수 있는 실행 파일과 설정 파일이 없었습니다.

또한 현재 시뮬레이션 진입점인 `agribot_description/launch/spawn_agribot.launch.py`는 임시 `map -> odom` 정적 TF를 기본 발행하도록 되어 있어, SLAM이 실제 `map -> odom`을 발행하는 구성과는 충돌 가능성이 있었습니다.

즉, 이번 티켓에서는 단순히 YAML 하나를 추가하는 수준이 아니라 아래 기준을 먼저 갖출 필요가 있었습니다.

- 시뮬레이션 런치와 충돌 없이 SLAM이 TF를 소유할 수 있을 것
- LiDAR 토픽 `/agribot/lidar/scan`과 현재 프레임 체계 `map`, `odom`, `base_link`를 그대로 사용할 것
- 팀원이 바로 실행 가능한 RViz 화면과 map 저장 위치까지 함께 제공할 것

---

## 주요 변경 사항

### 1. SLAM Toolbox 매핑 파라미터 추가
- `agribot_ws/src/agribot_navigation/config/slam_mapping.yaml`을 새로 추가했습니다.
- 현재 로봇 기준에 맞춰 아래 항목을 연결했습니다.
- `map_frame: map`
- `odom_frame: odom`
- `base_frame: base_link`
- `scan_topic: /agribot/lidar/scan`
- LiDAR 최대 거리 10m와 비닐하우스 주행을 고려해 mapping update, travel threshold, loop closing 관련 기본값을 함께 조정했습니다.

### 2. mapping 통합 런치 추가
- `agribot_ws/src/agribot_navigation/launch/mapping.launch.py`를 새로 추가했습니다.
- 이 런치는 아래 구성을 한 번에 실행하도록 만들었습니다.
- `agribot_description`의 `spawn_agribot.launch.py`
- `slam_toolbox`의 `online_async_launch.py`
- mapping 전용 RViz2
- 여기서 `publish_map_to_odom_tf=false`를 내려 기존 임시 정적 TF를 끄고, SLAM이 `map -> odom` 변환을 담당하도록 구성했습니다.

### 3. mapping 전용 RViz 화면 추가
- `agribot_ws/src/agribot_navigation/rviz/mapping.rviz`를 새로 추가했습니다.
- 아래 항목이 바로 보이도록 기본 display를 구성했습니다.
- `/map`
- `/map_updates`
- `/agribot/lidar/scan`
- `map`, `odom`, `base_link`, `lidar_link` TF
- 팀원이 launch 직후 지도 생성 시작 여부를 바로 확인할 수 있도록 최소 구성으로 정리했습니다.

### 4. map 저장 디렉터리 스캐폴딩 추가
- `agribot_ws/src/agribot_navigation/maps/README.md`를 추가했습니다.
- 이후 S14P-37에서 `map_saver_cli`로 `greenhouse_map.pgm`, `greenhouse_map.yaml`을 저장할 위치와 명령 예시를 남겼습니다.

### 5. 패키지 설치 자원 및 의존성 보강
- `agribot_ws/src/agribot_navigation/setup.py`에서 `rviz`, `maps` 디렉터리도 함께 설치되도록 수정했습니다.
- `agribot_ws/src/agribot_navigation/package.xml`에 아래 실행 의존성을 추가했습니다.
- `agribot_description`
- `launch`
- `launch_ros`
- `ros2launch`
- `rviz2`
- 이를 통해 새 launch와 RViz 자원이 실제 설치 경로에서도 정상 동작하도록 정리했습니다.

---

## 검증 방법

아래 순서로 정적 검증과 패키지 빌드 검증을 진행했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation
python3 -m py_compile launch/mapping.launch.py

python3 -c "from pathlib import Path; import yaml; yaml.safe_load(Path('config/slam_mapping.yaml').read_text()); yaml.safe_load(Path('rviz/mapping.rviz').read_text()); print('yaml-ok')"

cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --packages-select agribot_navigation
```

추가로 아래 파일을 직접 확인했습니다.

```bash
sed -n '1,240p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/slam_mapping.yaml
sed -n '1,240p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/launch/mapping.launch.py
sed -n '1,220p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/package.xml
sed -n '1,220p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/setup.py
```

---

## 검증 결과

### 정상 확인

- `mapping.launch.py` Python 문법 검사 통과
- `slam_mapping.yaml` 파싱 통과
- `mapping.rviz` YAML 파싱 통과
- `agribot_navigation` 패키지 단독 빌드 통과
- launch, config, rviz, maps 자원이 설치 대상으로 정상 포함되도록 반영 완료

### 확인 로그 요약

- 첫 `colcon build`에서는 기존 `install/agribot_navigation`의 stale symlink 상태 때문에 `mapping.launch.py` 설치 단계가 1회 실패했습니다.
- 해당 패키지의 `build/agribot_navigation`, `install/agribot_navigation`만 정리한 뒤 재빌드해 정상 완료했습니다.

즉, 이번 MR은 "SLAM을 위한 설정 초안" 수준이 아니라 실제 워크스페이스에서 설치 가능한 mapping 패키지 상태까지 맞춘 작업입니다.

---

## 확인된 이슈 및 주의사항

### 1. 실제 map 파일은 아직 생성하지 않음
- 이번 티켓 범위는 S14P-36 설정 준비까지입니다.
- `greenhouse_map.pgm`, `greenhouse_map.yaml` 실제 생성은 다음 티켓인 S14P-37에서 수행하는 것이 맞습니다.

### 2. 런타임 SLAM 동작은 이번 MR에서 실주행 검증하지 않음
- 이번 작업에서는 문법, YAML, 패키지 설치/빌드 검증까지 수행했습니다.
- 실제 Gazebo를 띄운 뒤 로봇을 움직이며 `/map`이 증가하는지 확인하는 런타임 검증은 후속 실행 단계에서 진행하면 됩니다.

### 3. LiDAR 입력 품질은 기존 센서 상태에 의존
- `scan_topic`은 `/agribot/lidar/scan`으로 연결했지만, 실제 mapping 품질은 LiDAR 발행 안정성에 영향을 받습니다.
- 기존 센서 점검에서 LiDAR 미수신 이슈가 있었다면, mapping launch 실행 전 해당 부분을 함께 재확인할 필요가 있습니다.

---

## 기대 효과

- 팀원이 `ros2 launch agribot_navigation mapping.launch.py` 한 번으로 지도 생성 세션을 시작할 수 있습니다.
- 이후 S14P-37 map 저장, S14P-203 localization, S14P-204 Nav2 조정이 같은 패키지 구조 위에서 바로 이어질 수 있습니다.
- 임시 `map -> odom` 정적 TF와 SLAM TF가 충돌하지 않도록 미리 정리해 두어 후속 Nav2 연동 리스크를 줄였습니다.
- RViz 기본 화면이 함께 제공돼 SLAM 상태 확인과 디버깅 진입 장벽이 낮아집니다.

---

## 후속 작업 제안

- `mapping.launch.py` 실제 실행 후 teleop 기반으로 비닐하우스 전체를 주행하며 `/map` 증가 여부 확인
- `map_saver_cli`로 `greenhouse_map` 저장 후 S14P-37 결과물 생성
- 저장된 map을 기준으로 S14P-203 `amcl` localization launch 추가
- 이후 S14P-204에서 `nav2_params.yaml`과 costmap 설정 작성

## 변경 파일

- agribot_ws/src/agribot_navigation/config/slam_mapping.yaml
- agribot_ws/src/agribot_navigation/launch/mapping.launch.py
- agribot_ws/src/agribot_navigation/rviz/mapping.rviz
- agribot_ws/src/agribot_navigation/maps/README.md
- agribot_ws/src/agribot_navigation/package.xml
- agribot_ws/src/agribot_navigation/setup.py


[S14P-37] 비닐하우스 지도 생성·저장

## 개요

S14P-37 비닐하우스 지도 생성·저장을 진행했습니다.

이번 MR의 목적은 다음 3가지입니다.

1. 이후 localization과 Nav2가 바로 사용할 수 있는 저장 지도 산출물 `greenhouse_map.pgm`, `greenhouse_map.yaml`을 패키지에 추가한다.
2. 지도 파일을 한 번 생성하는 것으로 끝내지 않고, `map_server`로 다시 불러와 재사용 가능한 상태인지 검증한다.
3. 현재 `farm_world.sdf`의 물리 성능 제약이 있더라도, 팀원이 동일한 metadata를 기준으로 같은 기본 지도를 재생성할 수 있는 경로를 남긴다.

이번 작업으로 `agribot_navigation` 패키지에 baseline map generator, 저장 지도 preview launch, 실제 `greenhouse_map` 산출물을 추가했고, 설치된 패키지 경로에서 map load까지 검증했습니다.

---

## 배경

S14P-36까지 진행한 시점에는 `slam_toolbox` 설정, mapping launch, RViz 설정은 준비되어 있었지만 실제 저장 지도 파일은 아직 없었습니다.

또한 이번 turn에서 `mapping.launch.py`를 실제로 띄워 `farm_world.sdf` 기반 live mapping도 함께 확인해봤는데, 아래 문제가 바로 드러났습니다.

- Gazebo에서 `Trimesh-trimesh contact hash table bucket overflow` 경고가 지속적으로 반복됨
- `greenhouse`, `tomato_plant`, `tomato`가 triangle mesh collision을 그대로 사용해 물리 부하가 큼
- 시뮬레이션이 충분히 진행되지 않아 SLAM 기반 실주행 저장을 이번 상태 그대로는 재현성 있게 완료하기 어려움

즉, 이번 티켓에서는 "live SLAM 저장"만 바라보기보다, 우선 후속 티켓이 바로 쓸 수 있는 저장 map 산출물과 재로드 경로를 먼저 안정적으로 확보할 필요가 있었습니다.

---

## 주요 변경 사항

### 1. 저장 지도 산출물 추가
- `agribot_ws/src/agribot_navigation/maps/greenhouse_map.pgm`
- `agribot_ws/src/agribot_navigation/maps/greenhouse_map.yaml`
- 저장 map metadata는 아래 값으로 정리했습니다.
- `resolution: 0.1`
- `origin: [-35.0, -10.0, 0.0]`

### 2. baseline map generator 추가
- `agribot_ws/src/agribot_navigation/agribot_navigation/generate_static_map.py`를 새로 추가했습니다.
- 이 스크립트는 아래 shared metadata를 읽어 저장 map을 재생성합니다.
- `agribot_navigation/config/zones.yaml`
- `agribot_description/config/crop_instances.yaml`
- 비닐하우스 외벽, 출입구, 좌우 재배 구역, 개별 식물 footprint를 occupancy map으로 반영하도록 구성했습니다.

### 3. 저장 지도 preview launch 추가
- `agribot_ws/src/agribot_navigation/launch/map_preview.launch.py`를 새로 추가했습니다.
- 이 런치는 아래 구성을 한 번에 실행합니다.
- `nav2_map_server/map_server`
- `nav2_lifecycle_manager/lifecycle_manager`
- 선택적 `rviz2`
- 이를 통해 Gazebo 없이도 저장된 `greenhouse_map.yaml`을 다시 불러와 통로 구조가 보이는지 바로 확인할 수 있게 했습니다.

### 4. 패키지 실행 경로와 의존성 보강
- `agribot_ws/src/agribot_navigation/setup.py`에 `generate_static_map` console script를 추가했습니다.
- `agribot_ws/src/agribot_navigation/package.xml`에 아래 실행 의존성을 추가했습니다.
- `nav2_lifecycle_manager`
- `python3-yaml`
- `agribot_ws/src/agribot_navigation/maps/README.md`에는 재생성, preview, 추후 live SLAM 저장 명령까지 함께 정리했습니다.

---

## 검증 방법

아래 순서로 산출물 생성, 패키지 빌드, 저장 지도 재로드 검증을 진행했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
source install/setup.bash
export HOME=/tmp/codex-home
export ROS_LOG_DIR=/tmp/ros_logs

python3 src/agribot_navigation/agribot_navigation/generate_static_map.py
colcon build --symlink-install --packages-select agribot_navigation

ros2 run agribot_navigation generate_static_map --output-prefix /tmp/greenhouse_map_test
ros2 launch agribot_navigation map_preview.launch.py use_rviz:=false
```

추가로 아래 명령으로 산출물과 메타데이터를 직접 확인했습니다.

```bash
sed -n '1,120p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/maps/greenhouse_map.yaml
find ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/maps -maxdepth 1 -type f | sort
git status --short
```

---

## 검증 결과

### 정상 확인

- `greenhouse_map.pgm`, `greenhouse_map.yaml` 생성 완료
- `agribot_navigation` 패키지 단독 빌드 통과
- 설치된 패키지 기준 `ros2 run agribot_navigation generate_static_map` 실행 확인
- `map_preview.launch.py`에서 설치 경로의 저장 map load 성공

### 확인 로그 요약

- `map_server`가 `/home/ssafy/SSAFY/S14P21A602/agribot_ws/install/agribot_navigation/share/agribot_navigation/maps/greenhouse_map.yaml`을 정상 로드했습니다.
- 동일하게 `greenhouse_map.pgm`도 정상 로드했고, 로그 기준 map 크기는 `700 X 2450 @ 0.1 m/cell`로 확인했습니다.
- 별도로 `ros2 run agribot_navigation generate_static_map --output-prefix /tmp/greenhouse_map_test`도 성공해 설치된 entry point가 정상 동작함을 확인했습니다.

즉, 이번 MR은 단순히 저장 파일만 추가한 것이 아니라 "재생성 가능하고 다시 불러와 검증된 저장 지도"까지 포함한 작업입니다.

---

## 확인된 이슈 및 주의사항

### 1. `farm_world.sdf` 기반 live SLAM 저장은 아직 재현성이 낮음
- `mapping.launch.py` 실주행 시 Gazebo에서 `Trimesh-trimesh contact hash table bucket overflow` 경고가 대량으로 반복되었습니다.
- 현재 `greenhouse`, `tomato_plant`, `tomato` 모델이 mesh collision을 그대로 사용하고 있어 물리 병목이 큽니다.
- 따라서 이번 MR의 저장 지도는 "현재 협업용 기준선 map"으로 보는 것이 맞고, 추후 성능 개선 후 live SLAM 결과로 같은 파일명을 덮어쓰는 흐름이 적절합니다.

### 2. 이번 저장 map은 shared metadata 기반 baseline map임
- `zones.yaml`과 `crop_instances.yaml`을 기준으로 외벽, 통로, 재배 구역을 occupancy map으로 구성했습니다.
- 실제 센서 스캔을 누적한 SLAM 결과물과 완전히 동일하다고 보기는 어렵지만, localization/Nav2 연계를 시작할 기준 map으로는 바로 사용할 수 있습니다.

---

## 기대 효과

- 이후 S14P-203 localization이 바로 사용할 저장 map 산출물이 생겼습니다.
- 팀원이 `ros2 run agribot_navigation generate_static_map`으로 같은 기준 지도를 재생성할 수 있습니다.
- `map_preview.launch.py`로 Gazebo 없이도 map 파일 재로드를 바로 확인할 수 있어 디버깅 진입 비용이 낮아졌습니다.
- live SLAM 성능 문제가 남아 있더라도 후속 Nav2 설정 작업은 저장 map 기준으로 계속 진행할 수 있습니다.

---

## 후속 작업 제안

- `farm_world.sdf`의 mesh collision을 단순화해 Gazebo 실시간 성능 개선
- 성능 개선 후 `mapping.launch.py`로 비닐하우스 전체를 실제 주행하고 `map_saver_cli`로 `greenhouse_map` 재생성
- 저장 map 기준으로 S14P-203 `amcl` localization launch 추가
- 이후 S14P-204에서 `nav2_params.yaml`과 costmap 파라미터 조정

## 변경 파일

- agribot_ws/src/agribot_navigation/agribot_navigation/generate_static_map.py
- agribot_ws/src/agribot_navigation/launch/map_preview.launch.py
- agribot_ws/src/agribot_navigation/maps/greenhouse_map.pgm
- agribot_ws/src/agribot_navigation/maps/greenhouse_map.yaml
- agribot_ws/src/agribot_navigation/maps/README.md
- agribot_ws/src/agribot_navigation/package.xml
- agribot_ws/src/agribot_navigation/setup.py


[S14P-203] 지도 기반 위치 추정

## 개요

S14P-203 지도 기반 위치 추정을 진행했습니다.

이번 MR의 목적은 다음 3가지입니다.

1. 저장된 `greenhouse_map.yaml`을 기준으로 `map_server`와 `amcl`을 함께 실행할 수 있는 localization 경로를 추가한다.
2. Gazebo 시뮬레이션에서 임시 정적 `map -> odom` 대신 AMCL이 실제 `map -> odom`을 발행하도록 연결한다.
3. 팀원이 동일한 명령으로 localization 세션을 재현하고, RViz2에서 초기 위치를 지정해 바로 검증할 수 있도록 문서와 의존성을 함께 정리한다.

이번 작업으로 `agribot_navigation` 패키지에 AMCL 파라미터와 localization 전용 launch를 추가했고, 빌드와 launch argument 로딩까지 확인했습니다.

---

## 배경

S14P-202까지 진행한 시점에는 저장 지도 `greenhouse_map.pgm`, `greenhouse_map.yaml`은 이미 준비되어 있었고, `mapping.launch.py`와 `map_preview.launch.py`도 있었습니다.

하지만 실제 정적 지도 기반 운용 단계에서 필요한 localization 경로는 아직 비어 있었습니다.

구체적으로는 아래 상태였습니다.

- `spawn_agribot.launch.py`는 임시 항등 `map -> odom` 정적 TF를 발행하는 구조였음
- 저장 지도를 불러오는 `map_server`와 실제 위치 추정을 담당할 `amcl`을 함께 실행하는 launch가 없었음
- RViz2에서 `/initialpose`를 찍어 localization을 바로 확인하는 표준 실행 경로가 문서화되어 있지 않았음

즉, 지도는 이미 저장되어 있었지만 "저장 지도를 기준으로 로봇이 지금 어디 있는지 추정하는 단계"가 아직 연결되지 않은 상태였습니다.

---

## 주요 변경 사항

### 1. AMCL 파라미터 파일 추가
- `agribot_ws/src/agribot_navigation/config/amcl.yaml`을 새로 추가했습니다.
- 현재 로봇 프레임과 센서 토픽에 맞춰 아래 값을 반영했습니다.
- `base_frame_id: base_link`
- `odom_frame_id: odom`
- `global_frame_id: map`
- `scan_topic: /agribot/lidar/scan`
- LiDAR SDF 범위와 맞춘 `laser_min_range: 0.12`, `laser_max_range: 10.0`
- 이번 파일은 localization 시작선에 필요한 최소 설정으로 두었고, 세부 Nav2 주행 튜닝은 후속 티켓으로 분리했습니다.

### 2. localization 전용 launch 추가
- `agribot_ws/src/agribot_navigation/launch/localization.launch.py`를 새로 추가했습니다.
- 이 런치는 아래 구성을 한 번에 실행합니다.
- `agribot_description/spawn_agribot.launch.py`
- `nav2_map_server/map_server`
- `nav2_amcl/amcl`
- `nav2_lifecycle_manager/lifecycle_manager`
- 선택적 `rviz2`
- 특히 `publish_map_to_odom_tf:=false`를 넘겨, 기존 임시 정적 TF를 끄고 AMCL이 `map -> odom`을 맡도록 전환했습니다.

### 3. 실행 의존성 보강
- `agribot_ws/src/agribot_navigation/package.xml`에 아래 실행 의존성을 추가했습니다.
- `nav2_amcl`
- `nav2_map_server`
- 이를 통해 팀원이 패키지 의존성 기준만 맞추면 동일한 localization launch를 바로 실행할 수 있게 했습니다.

### 4. 사용 가이드와 작업 문서 추가
- `agribot_ws/src/agribot_navigation/maps/README.md`에 localization 실행 명령을 추가했습니다.
- `docs/S14P-203_지도_기반_위치_추정.md` 문서에 목표, 반영 내용, 실행 방법, 현재 제한 사항을 정리했습니다.
- RViz2에서 `2D Pose Estimate`로 `/initialpose`를 주는 절차까지 함께 남겼습니다.

---

## 검증 방법

아래 순서로 패키지 빌드와 localization launch 로딩을 검증했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --packages-select agribot_navigation
source install/setup.bash
export HOME=/tmp/codex-home
export ROS_LOG_DIR=/tmp/ros_logs

ros2 launch agribot_navigation localization.launch.py --show-args
```

추가로 아래 명령으로 변경 파일과 AMCL 설정을 직접 확인했습니다.

```bash
sed -n '1,220p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/launch/localization.launch.py
sed -n '1,220p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/amcl.yaml
git status --short
```

---

## 검증 결과

### 정상 확인

- `agribot_navigation` 패키지 단독 빌드 통과
- 설치된 패키지 기준 `localization.launch.py` 인자 로딩 성공
- launch 기본값으로 아래 경로가 연결됨
- `maps/greenhouse_map.yaml`
- `config/amcl.yaml`
- `rviz/mapping.rviz`

### 확인 로그 요약

- `ros2 launch agribot_navigation localization.launch.py --show-args` 기준 `use_sim_time`, `world`, `map`, `params_file`, `use_rviz`, `autostart` 인자가 정상 노출되었습니다.
- 설치 경로 기준 기본 map 파일은 `/home/ssafy/SSAFY/S14P21A602/agribot_ws/install/agribot_navigation/share/agribot_navigation/maps/greenhouse_map.yaml`으로 확인했습니다.
- 설치 경로 기준 AMCL 파라미터 파일은 `/home/ssafy/SSAFY/S14P21A602/agribot_ws/install/agribot_navigation/share/agribot_navigation/config/amcl.yaml`으로 확인했습니다.

즉, 이번 MR은 localization 설정 파일만 추가한 것이 아니라, 실제 패키지 설치 경로 기준으로 launch가 읽히는 상태까지 맞춘 작업입니다.

---

## 확인된 이슈 및 주의사항

### 1. 이번 MR은 localization 시작선까지 검증
- 이번 turn에서는 `ros2 launch ... --show-args`까지 확인했습니다.
- 실제 RViz2에서 초기 위치를 찍고 수렴 정도를 보는 런타임 검증은 Gazebo 세션을 길게 띄워 확인하는 후속 단계가 남아 있습니다.

### 2. localization 품질은 LiDAR 품질에 직접 영향받음
- `amcl`은 `/agribot/lidar/scan`을 사용하도록 연결했습니다.
- 따라서 scan 품질이나 시뮬레이션 성능 문제가 있으면 수렴 안정성도 함께 흔들릴 수 있습니다.

### 3. Nav2 주행 파라미터는 아직 범위 밖
- 이번 작업은 `map_server + amcl` 기반 위치 추정 경로를 만드는 범위입니다.
- costmap, planner, controller, recovery 동작은 후속 `S14P-204`에서 조정하는 편이 맞습니다.

---

## 기대 효과

- 저장된 `greenhouse_map` 기준 localization 실행 경로가 생겼습니다.
- 팀원이 `ros2 launch agribot_navigation localization.launch.py`만으로 같은 시작 구성을 재현할 수 있습니다.
- 기존 임시 `map -> odom` 정적 TF를 AMCL 기반 동적 추정으로 교체할 준비가 끝났습니다.
- 이후 S14P-204에서 Nav2 파라미터 조정을 바로 이어갈 수 있습니다.

---

## 후속 작업 제안

- 실제 Gazebo + RViz2 세션에서 `2D Pose Estimate` 후 `amcl_pose`, `particle_cloud`, `map -> base_link` 수렴 확인
- 저장 map 기준 `nav2_params.yaml` 작성 및 costmap/planner/controller 초기값 조정
- 시뮬레이션 성능과 LiDAR 품질을 함께 점검해 localization 흔들림 원인 분리

## 변경 파일

- agribot_ws/src/agribot_navigation/config/amcl.yaml
- agribot_ws/src/agribot_navigation/launch/localization.launch.py
- agribot_ws/src/agribot_navigation/maps/README.md
- agribot_ws/src/agribot_navigation/package.xml
- docs/S14P-203_지도_기반_위치_추정.md

[S14P-39] 자율주행 파라미터 조정 및 farm_world collision 경량화

## 개요

S14P-39 자율주행 파라미터 조정을 진행했습니다.

이번 MR의 목적은 다음 3가지입니다.

1. 저장된 정적 지도와 AMCL 기준선 위에서 Nav2 자율주행 스택을 실제로 기동 가능한 상태로 완성한다.
2. 비닐하우스 통로 환경에 맞게 planner, controller, costmap, recovery 파라미터를 조정해 RViz 목표점 이동 기준선을 만든다.
3. `farm_world`의 무거운 triangle-mesh collision 병목을 줄여, 시각 품질은 유지하면서 물리 계산과 TF 초기화를 더 안정적으로 만든다.

이번 작업으로 `navigation.launch.py`와 `nav2_params.yaml`을 추가해 정적 지도 기반 Nav2 스택을 바로 띄울 수 있게 했고, `farm_world`는 visual mesh는 유지한 채 collision만 단순화해 기존 ODE 충돌 병목을 줄였습니다.

---

## 배경

S14P-38까지 진행한 시점에는 저장 지도와 AMCL localization은 준비되어 있었지만, 실제 목표점 주행에 필요한 Nav2 서버 구성이 빠져 있었습니다.

구체적으로는 아래 문제가 있었습니다.

- `planner_server`, `controller_server`, `bt_navigator`, `behavior_server`, `waypoint_follower`를 한 번에 올리는 전용 launch가 없음
- 로봇 footprint, inflation, planner / controller 조합이 비닐하우스 통로 기준으로 아직 정리되지 않음
- 기본 `farm_world`는 `greenhouse`, `tomato_plant`, `tomato`, `leaf_scan`이 mesh collision을 그대로 사용하고 있어 Gazebo 물리 부하가 큼
- 실제로 Nav2 기동 시 `odom`과 `map` 관련 TF가 늦게 올라와 activation이 흔들리고, 이전 작업에서도 ODE `Trimesh-trimesh contact hash table bucket overflow` 경고가 반복됨

즉, 단순히 Nav2 YAML만 추가하는 수준이 아니라 "자율주행 파라미터 세팅"과 "월드 collision 경량화"를 같이 처리해야 실제 목표점 이동 기준선을 만들 수 있었습니다.

---

## 주요 변경 사항

### 1. Nav2 파라미터 파일 추가
- `agribot_ws/src/agribot_navigation/config/nav2_params.yaml`를 새로 추가했습니다.
- 이번 MR의 기본 조합은 아래와 같습니다.
  - Global Planner: `nav2_smac_planner::SmacPlanner2D`
  - Local Controller: `nav2_regulated_pure_pursuit_controller::RegulatedPurePursuitController`
  - Recovery: `spin`, `backup`, `wait`
- AgriBot 차체와 통로 여유폭을 기준으로 아래 값을 정리했습니다.
  - footprint: `[[0.24, 0.20], [0.24, -0.20], [-0.24, -0.20], [-0.24, 0.20]]`
  - local costmap: `4 x 4`, `0.05 m`
  - global costmap: `0.10 m`
  - inflation radius: `0.35`
  - desired linear velocity: `0.28 m/s`
  - goal tolerance: `0.15 m`, `0.15 rad`
- Nav2는 기본적으로 `Twist`를 사용하도록 두어 현재 `/cmd_vel -> /cmd_vel_safe` watchdog 구조와 바로 맞물리게 했습니다.

### 2. navigation 전용 launch 추가
- `agribot_ws/src/agribot_navigation/launch/navigation.launch.py`를 새로 추가했습니다.
- 이 launch는 기존 `localization.launch.py`를 그대로 포함하고, 그 위에 아래 Nav2 서버를 추가로 올립니다.
  - `controller_server`
  - `planner_server`
  - `smoother_server`
  - `behavior_server`
  - `bt_navigator`
  - `waypoint_follower`
  - `lifecycle_manager_navigation`
- 결과적으로 `ros2 launch agribot_navigation navigation.launch.py` 한 번으로 Gazebo + map_server + AMCL + Nav2 navigation stack까지 함께 검증할 수 있게 했습니다.

### 3. RViz 자율주행 검증 화면 보강
- `agribot_ws/src/agribot_navigation/rviz/mapping.rviz`를 보강했습니다.
- 아래 요소를 추가했습니다.
  - `nav2_rviz_plugins/GoalTool`
  - `/plan` global path 표시
  - `/local_plan` local path 표시
- 이제 RViz에서 `2D Pose Estimate` 후 `2D Goal Pose`로 바로 자율주행 검증이 가능합니다.

### 4. 패키지 의존성과 사용 가이드 정리
- `agribot_ws/src/agribot_navigation/package.xml`에 Nav2 planner / controller / smoother / behavior / BT / waypoint / RViz plugin 의존성을 추가했습니다.
- `agribot_ws/src/agribot_navigation/maps/README.md`에 full navigation launch 사용법과 검증 절차를 문서화했습니다.
- 작업 결과 문서는 `docs/S14P-39_자율주행_파라미터_조정.md`에 정리했습니다.

### 5. farm_world collision 경량화
- 시각 품질은 유지하고 collision만 단순화하는 방향으로 수정했습니다.
- `greenhouse`, `tomato_plant`, `leaf_scan` 모델에서는 mesh collision을 제거하고 visual mesh만 남겼습니다.
- `tomato`는 `static=true`로 바꿔 동적 mesh contact를 만들지 않게 했고, collision mesh도 제거했습니다.
- 대신 `agribot_ws/src/agribot_description/worlds/farm_world.sdf`에 보이지 않는 collision proxy를 추가했습니다.
  - 비닐하우스 외곽 벽: box collision
  - 작물 베드 영역 2개: box collision
- 즉, 화면에 보이는 모델은 그대로 두고 LiDAR / 물리 / costmap이 참조하는 충돌체만 단순화했습니다.

---

## 검증 방법

아래 순서로 직접 검증했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --packages-select agribot_description agribot_navigation
source install/setup.bash
export HOME=/tmp/codex-home
export ROS_LOG_DIR=/tmp/ros_logs

ros2 launch agribot_navigation navigation.launch.py
```

빠른 검증용으로는 아래 명령도 사용했습니다.

```bash
ros2 launch agribot_navigation navigation.launch.py --show-args
timeout 20s ros2 launch agribot_navigation navigation.launch.py use_rviz:=false
```

RViz에서는 아래 순서로 확인했습니다.

```bash
1. Fixed Frame이 `map`인지 확인
2. `2D Pose Estimate`로 초기 위치 지정
3. `2D Goal Pose`로 통로 안 목표 지정
4. `/plan`, `/local_plan`, `/map`, `/agribot/lidar/scan` 표시 확인
```

---

## 검증 결과

### 정상 확인

- `agribot_navigation` 빌드 성공
- `navigation.launch.py --show-args` 정상 출력
- `controller_server`, `planner_server`, `smoother_server`, `behavior_server`, `bt_navigator`, `waypoint_follower` configure 성공
- `SmacPlanner2D`, `RegulatedPurePursuitController`, `spin/backup/wait` recovery plugin 로딩 확인
- collision 경량화 이후 `controller_server`가 이전보다 더 빠르게 activation 단계까지 진행
- 수정 후 로그에서는 기존처럼 ODE `Trimesh-trimesh ... overflow` 문자열이 재발하지 않음

### 확인 로그 요약

- `ros2 launch agribot_navigation navigation.launch.py --show-args`
  - `map`, `localization_params_file`, `nav2_params_file`, `use_rviz`, `autostart` 등 주요 인자 노출 확인
- `timeout 20s ros2 launch agribot_navigation navigation.launch.py use_rviz:=false`
  - 초기에는 `odom -> base_link` 대기 후 `controller_server` activation 진입 확인
  - 이후 `map -> odom`은 AMCL 초기 위치가 없어서 늦게 형성되는 상태 확인

즉, Nav2 파라미터와 launch 자체는 정상적으로 붙었고, 월드 collision 경량화 이후에는 이전보다 기동 기준선이 분명히 좋아졌습니다.

---

## 확인된 이슈 및 주의사항

### 1. AMCL 초기 위치는 여전히 필요
- 이번 MR은 localization 위에 navigation stack을 얹은 작업입니다.
- 따라서 RViz에서 `2D Pose Estimate`를 주기 전까지는 `map -> odom`이 완전히 안정되지 않아 planner / controller가 map frame을 기다릴 수 있습니다.
- 이건 성능 문제라기보다 AMCL 동작 방식에 가까운 정상 흐름입니다.

### 2. collision은 단순화됐지만 visual은 유지됨
- 이번 변경은 visual mesh를 단순화한 것이 아닙니다.
- 화면에 보이는 온실 / 작물 / 열매 / 잎 메쉬는 유지했고, collision만 치환했습니다.
- 따라서 데모 품질을 떨어뜨리지 않으면서도 물리 부하를 줄이는 방향입니다.

### 3. tomato는 물리 객체가 아니게 됨
- `tomato`를 `static=true`로 바꾸고 collision을 제거했기 때문에, 이제 열매는 충돌 반응을 만드는 동적 물체가 아닙니다.
- 수확을 물리 접촉으로 구현할 계획이라면 이후 별도 구조를 다시 잡아야 합니다.

---

## 기대 효과

- 저장 지도 기반 `AMCL + Nav2` 검증이 한 번에 가능한 실행 경로가 생깁니다.
- RViz에서 목표점 주행 검증을 반복할 수 있어 이후 waypoint 순찰과 mission 로직 작업이 쉬워집니다.
- `farm_world`의 collision 병목이 줄어 Gazebo 물리 경고와 TF 초기화 지연을 완화할 수 있습니다.
- visual 퀄리티를 유지한 채 주행 성능만 개선하는 방향을 팀 기준선으로 남길 수 있습니다.

---

## 후속 작업 제안

- RViz에서 실제 `2D Pose Estimate -> 2D Goal Pose` 왕복 테스트를 여러 구간으로 반복해 goal tolerance 추가 보정
- `S14P-205` waypoint YAML 작성과 `S14P-41` 자동 순찰 노드 구현
- 필요하면 온실 내부 통로 collision proxy를 더 세분화해 costmap과 실제 주행 동선을 더 정밀하게 맞추기
- 수확 로직에서 토마토를 물리 객체로 다시 쓸 필요가 있다면, visual / interaction 전용 구조를 별도 분리

## 변경 파일

- agribot_ws/src/agribot_navigation/config/nav2_params.yaml
- agribot_ws/src/agribot_navigation/launch/navigation.launch.py
- agribot_ws/src/agribot_navigation/package.xml
- agribot_ws/src/agribot_navigation/maps/README.md
- agribot_ws/src/agribot_navigation/rviz/mapping.rviz
- docs/S14P-39_자율주행_파라미터_조정.md
- agribot_ws/src/agribot_description/worlds/farm_world.sdf
- agribot_ws/src/agribot_description/models/greenhouse/model.sdf
- agribot_ws/src/agribot_description/models/tomato_plant/model.sdf
- agribot_ws/src/agribot_description/models/tomato/model.sdf
- agribot_ws/src/agribot_description/models/leaf_scan/model.sdf


[S14P-205] 순찰 좌표 정의

## 개요

S14P-205 순찰 좌표 정의를 진행했습니다.

이번 MR의 목적은 다음 3가지입니다.

1. `greenhouse_01` 기준 순찰 좌표를 행(row) 단위로 정리해 이후 `patrol_node`가 바로 사용할 기준선을 만든다.
2. 각 좌표를 단순 숫자 목록이 아니라 `home`, `connector`, `entry`, `inspect`, `turn`, `exit` 의미를 가진 메타데이터로 구조화한다.
3. 좌표 파일이 깨지지 않았는지 패키지 내부에서 바로 검증할 수 있는 로더와 실행 명령을 함께 남긴다.

이번 작업으로 `agribot_navigation` 패키지에 순찰 좌표 파일 `patrol_waypoints.yaml`을 추가했고, 좌표 참조 무결성을 확인하는 validator와 실행 가이드까지 함께 정리했습니다.

---

## 배경

S14P-39까지 진행한 시점에는 정적 지도, AMCL, Nav2 목표점 이동 기준선은 준비되어 있었지만, 실제 자동 순찰이 사용할 waypoint 계약은 아직 비어 있었습니다.

구체적으로는 아래 문제가 있었습니다.

- 현재 문서 기준으로는 "행(row)과 구역(zone) 중심으로 waypoint를 설계하라"는 원칙은 있었지만, 실제 파일 산출물은 없었음
- 이후 `S14P-41` 자동 순찰 제어에서 사용할 좌표가 없어서, 노드 구현 시 좌표 정의와 제어 로직이 한 티켓에 섞일 위험이 있었음
- 개별 식물마다 목표점을 찍는 방식은 유지보수에 약하므로, 재배 베드 기준의 순찰 anchor를 먼저 고정할 필요가 있었음

즉, 이번 티켓에서는 순찰 제어 로직보다 먼저 "팀 전체가 공통으로 참조할 수 있는 waypoint 기준선"을 파일 형태로 확정하는 것이 핵심이었습니다.

---

## 주요 변경 사항

### 1. 순찰 좌표 메타데이터 파일 추가
- `agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml`을 새로 추가했습니다.
- 파일에는 아래 공통 메타데이터를 포함했습니다.
- `schema_version`
- `frame_id: map`
- `zone_id: greenhouse_01`
- `home_pose_id`
- 좌표 설계 근거(`coordinate_rationale`)
- 권장 주행/관찰 제약(`robot_constraints`)

### 2. 행(row) 단위 waypoint 구조 정의
- 순찰 좌표는 개별 식물 단위가 아니라 행 단위로 정의했습니다.
- 총 구성은 아래와 같습니다.
- `home` 1개
- `connector` 2개
- 좌측 베드용 `entry / inspect / turn`
- 우측 베드용 `entry / inspect / turn`
- 전체 기준으로는 `15개 waypoint`, `2개 route`, `17개 default patrol sequence`로 정리했습니다.
- 각 `inspect` waypoint에는 관찰 대상 `plant_id`, `tomato_id`를 함께 기록해 이후 perception / mission / harvest 로직이 같은 기준을 재사용할 수 있게 했습니다.

### 3. 좌우 베드 순찰 route 추가
- `routes` 항목에 아래 2개 경로를 정의했습니다.
- `greenhouse_01_left_bed_northbound`
- `greenhouse_01_right_bed_southbound`
- 각 route는 `entry -> inspect_pose_ids -> turn -> exit` 구조를 갖도록 만들었습니다.
- 이를 통해 이후 순찰 노드는 route 단위와 전체 기본 순찰 시퀀스 단위를 둘 다 사용할 수 있습니다.

### 4. 좌표 검증용 loader/validator 추가
- `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py`를 새로 추가했습니다.
- 이 모듈은 아래 검증을 수행합니다.
- waypoint / route 중복 ID 확인
- `home_pose_id` 존재 여부 확인
- `default_patrol_sequence` 참조 무결성 확인
- route가 참조하는 `entry`, `inspect`, `turn`, `exit` waypoint 존재 여부 확인
- 즉, 이번 MR은 좌표 YAML만 추가한 것이 아니라, 이후 작업에서 설정 파일이 깨졌을 때 바로 실패하도록 최소 검증 경로까지 포함합니다.

### 5. 실행 진입점과 가이드 추가
- `agribot_ws/src/agribot_navigation/setup.py`에 `validate_patrol_waypoints` 콘솔 엔트리포인트를 추가했습니다.
- `agribot_ws/src/agribot_navigation/maps/README.md`에 검증 명령을 문서화했습니다.
- 이제 팀원이 아래 명령으로 설치된 패키지 기준 검증을 바로 수행할 수 있습니다.

```bash
ros2 run agribot_navigation validate_patrol_waypoints
```

---

## 검증 방법

아래 순서로 빌드와 좌표 검증을 직접 확인했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --packages-select agribot_navigation
source install/setup.bash

ros2 run agribot_navigation validate_patrol_waypoints
```

추가로 아래 파일 내용을 직접 확인했습니다.

```bash
sed -n '1,260p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml
sed -n '1,260p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py
sed -n '1,220p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/maps/README.md
```

---

## 검증 결과

### 정상 확인

- `agribot_navigation` 패키지 단독 빌드 통과
- `validate_patrol_waypoints` 실행 성공
- `patrol_waypoints.yaml` 로드 성공
- route/sequence 참조 무결성 검증 성공

### 확인 로그 요약

- `ros2 run agribot_navigation validate_patrol_waypoints` 실행 결과 아래 내용이 출력됐습니다.
- `Loaded patrol plan for zone greenhouse_01`
- `15 waypoints, 2 routes, 17 sequence entries`
- `greenhouse_01_left_bed_northbound: 5 inspect poses, 7 plants`
- `greenhouse_01_right_bed_southbound: 3 inspect poses, 3 plants`

즉, 이번 MR은 단순 문서성 좌표 나열이 아니라 실제 패키지 설치 경로 기준으로 읽히고 검증까지 통과한 순찰 기준선입니다.

---

## 확인된 이슈 및 주의사항

### 1. 이번 MR은 좌표 정의와 검증 기준선까지가 범위
- 실제 자동 순찰 실행 노드인 `patrol_node.py`는 아직 포함하지 않았습니다.
- 이 부분은 후속 `S14P-41`에서 waypoint 소비 로직과 start/stop/resume 제어를 붙이는 편이 맞습니다.

### 2. 수확용 세부 접근 좌표는 아직 포함하지 않음
- 이번 파일은 순찰용 행 단위 anchor를 정의하는 범위입니다.
- 특정 토마토에 가까이 접근하는 `approach pose`는 후속 `S14P-42` 또는 수확 로직 단계에서 파생시키는 구조가 적절합니다.

### 3. 좌표는 현재 저장 지도와 월드 메타데이터 기준 baseline
- 이번 좌표는 `farm_world.sdf`, `greenhouse_map.yaml`, `crop_instances.yaml` 기준으로 잡은 기준선입니다.
- 실제 Gazebo / RViz 주행 결과에 따라 `x`, `y`, `yaw`는 후속 튜닝에서 일부 미세 조정될 수 있습니다.

---

## 기대 효과

- 이후 `S14P-41`에서 순찰 제어 로직만 구현하면 같은 좌표 파일을 바로 재사용할 수 있습니다.
- 팀원이 모두 같은 `waypoint_id`, `route_id`, `plant_id`, `tomato_id` 기준으로 순찰 경로를 해석할 수 있습니다.
- 개별 식물마다 goal을 찍는 방식보다 수정 범위가 작아 지도 변경이나 작물 추가에 더 강한 구조를 확보할 수 있습니다.
- perception, mission manager, harvest 접근 로직이 같은 순찰 anchor를 중심으로 연결되기 쉬워집니다.

---

## 후속 작업 제안

- `S14P-41`에서 `patrol_waypoints.yaml`을 읽는 `patrol_node.py` 구현
- RViz / Gazebo에서 기본 순찰 시퀀스 각 waypoint에 대한 실제 도달 가능성 확인
- 수확 이벤트 발생 시 inspect pose에서 파생되는 `approach pose` 계산 로직 추가

## 변경 파일

- agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml
- agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py
- agribot_ws/src/agribot_navigation/setup.py
- agribot_ws/src/agribot_navigation/maps/README.md


[S14P-41] 자동 순찰 제어

## 개요

S14P-41 자동 순찰 제어를 진행했습니다.

이번 MR의 목적은 다음 3가지입니다.

1. `S14P-205`에서 정의한 `patrol_waypoints.yaml`을 읽어 로봇이 waypoint를 순서대로 방문하는 실행 노드를 추가한다.
2. 순찰을 한 번에 끝까지 돌리는 것뿐 아니라, 중간에 멈추고 같은 지점부터 다시 이어서 갈 수 있도록 `start / stop / resume` 제어를 붙인다.
3. Nav2 실행 런치, 패키지 엔트리포인트, README까지 함께 연결해 팀원이 바로 실행하고 검증할 수 있게 만든다.

이번 작업으로 `agribot_navigation` 패키지에 `patrol_node.py`를 추가했고, `/patrol/start`, `/patrol/stop`, `/patrol/resume` 서비스와 `/patrol/status` 상태 토픽을 통해 자동 순찰을 제어할 수 있게 했습니다.

---

## 배경

`S14P-205`까지 진행한 시점에는 순찰 좌표와 route 메타데이터는 정리되어 있었지만, 실제로 그 waypoint를 소비해서 움직이는 실행 노드는 없었습니다.

구체적으로는 아래 문제가 있었습니다.

- `patrol_waypoints.yaml`은 존재하지만 이를 읽어 Nav2 goal로 보내는 순찰 실행 주체가 없었음
- 후속 티켓 요구사항인 "중간 정지 후 재개"를 고려하면 단순 목표점 이동보다 waypoint 진행 상태를 노드 내부에서 직접 관리할 필요가 있었음
- 설치된 패키지 경로 기준으로 설정 파일을 읽고, 서비스 이름까지 고정해 두지 않으면 팀원이 같은 방식으로 재현하기 어려웠음

즉, 이번 티켓의 핵심은 좌표 정의 이후의 다음 단계로서, `S14P-205` 산출물을 실제 주행 제어로 연결하는 최소 실행 경로를 만드는 것이었습니다.

---

## 주요 변경 사항

### 1. 자동 순찰 실행 노드 추가
- `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_node.py`를 새로 추가했습니다.
- 이 노드는 `patrol_waypoints.yaml`의 `default_patrol_sequence`를 읽고 waypoint를 순서대로 방문합니다.
- 내부 구현은 `nav2_waypoint_follower`에 전부 위임하지 않고, `NavigateToPose` 액션을 waypoint 단위로 순차 호출하는 방식으로 구성했습니다.
- 이렇게 구성한 이유는 정지 시점과 재개 지점을 노드 내부 인덱스로 직접 관리하는 편이 `S14P-41` 요구사항에 더 단순하고 명확하기 때문입니다.

### 2. 순찰 상태 머신과 제어 서비스 추가
- 노드 내부에 아래 상태를 두었습니다.
- `idle`
- `starting`
- `running`
- `observing`
- `stopping`
- `stopped`
- `completed`
- `error`
- 외부 제어용 서비스도 함께 추가했습니다.
- `/patrol/start`
- `/patrol/stop`
- `/patrol/resume`
- 순찰 중지 시에는 현재 goal을 cancel하고, 재개 시에는 중단된 waypoint 인덱스부터 다시 시작하도록 만들었습니다.

### 3. inspect waypoint 관찰 대기 로직 추가
- `inspect` 목적의 waypoint에 도달하면 바로 다음 지점으로 넘어가지 않고 잠시 대기하도록 구현했습니다.
- 대기 시간은 `patrol_waypoints.yaml`의 `robot_constraints.recommended_observation_dwell_sec` 값을 재사용합니다.
- 이를 통해 문서에 정의한 "도착 후 잠시 멈추고 관찰" 요구사항이 실제 실행 노드에도 반영됩니다.

### 4. patrol 설정 로더 보강
- 기존 `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py`를 확장했습니다.
- 설치된 패키지 기준 기본 `patrol_waypoints.yaml` 경로를 계산하는 helper를 추가했습니다.
- `recommended_observation_dwell_sec`를 파싱하고 음수 여부를 검증하도록 보강했습니다.
- validator 출력에도 inspect dwell 정보를 포함해, 설치 환경 기준 설정 요약을 더 바로 확인할 수 있게 했습니다.

### 5. navigation.launch.py에 patrol 노드 연결
- `agribot_ws/src/agribot_navigation/launch/navigation.launch.py`에 patrol 관련 launch argument를 추가했습니다.
- `use_patrol`
- `patrol_waypoints_file`
- `patrol_autostart`
- 기본 `navigation.launch.py` 실행 시 patrol 노드도 함께 띄울 수 있고, 필요하면 자동 시작 여부를 launch argument로 조절할 수 있게 했습니다.

### 6. 엔트리포인트, 의존성, 실행 가이드 정리
- `agribot_ws/src/agribot_navigation/setup.py`에 `patrol_node` 콘솔 엔트리포인트를 추가했습니다.
- `agribot_ws/src/agribot_navigation/package.xml`에 `nav2_msgs`, `action_msgs`, `std_srvs`, `std_msgs`, `ament_index_python` 등 필요한 런타임 의존성을 추가했습니다.
- `agribot_ws/src/agribot_navigation/maps/README.md`에 아래 명령을 문서화했습니다.

```bash
ros2 service call /patrol/start std_srvs/srv/Trigger "{}"
ros2 service call /patrol/stop std_srvs/srv/Trigger "{}"
ros2 service call /patrol/resume std_srvs/srv/Trigger "{}"
ros2 topic echo /patrol/status
```

---

## 검증 방법

아래 순서로 문법 검사, 빌드, validator, patrol 노드 기동을 직접 확인했습니다.

```bash
cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --packages-select agribot_navigation
source install/setup.bash

ros2 run agribot_navigation validate_patrol_waypoints
timeout 5s ros2 run agribot_navigation patrol_node
ros2 service list | rg '^/patrol/(start|stop|resume)$'
```

추가로 변경 파일들의 문법과 내용도 직접 확인했습니다.

```bash
python3 -m py_compile \
  ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py \
  ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/agribot_navigation/patrol_node.py \
  ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/launch/navigation.launch.py
```

---

## 검증 결과

### 정상 확인

- `agribot_navigation` 패키지 단독 빌드 통과
- `validate_patrol_waypoints` 실행 성공
- `patrol_node` 설치 경로 기준 기동 성공
- `/patrol/start`, `/patrol/stop`, `/patrol/resume` 서비스 등록 확인

### 확인 로그 요약

- `ros2 run agribot_navigation validate_patrol_waypoints` 실행 결과 아래 내용이 출력됐습니다.
- `Loaded patrol plan for zone greenhouse_01`
- `15 waypoints, 2 routes, 17 sequence entries, 3.0s inspect dwell`
- `greenhouse_01_left_bed_northbound: 5 inspect poses, 7 plants`
- `greenhouse_01_right_bed_southbound: 3 inspect poses, 3 plants`
- `timeout 5s ros2 run agribot_navigation patrol_node` 실행 시 patrol plan 로드 로그가 정상 출력됐습니다.
- `ros2 service list`에서 `/patrol/start`, `/patrol/stop`, `/patrol/resume` 3개 서비스가 확인됐습니다.

즉, 이번 MR은 문서상 계획 수준이 아니라 실제 설치된 ROS 2 패키지에서 patrol 노드가 기동되고 제어 서비스가 올라오는 수준까지 연결한 작업입니다.

---

## 확인된 이슈 및 주의사항

### 1. 이번 MR은 순찰 실행과 제어 기준선까지가 범위
- waypoint 순차 방문, 관찰 대기, 중지/재개는 포함했습니다.
- 하지만 perception 이벤트에 따라 수확 위치로 분기하거나, 작업 후 원래 순찰 지점이나 home으로 복귀하는 로직은 아직 포함하지 않았습니다.
- 이 부분은 후속 `S14P-42`, mission manager 티켓과 자연스럽게 이어지는 범위입니다.

### 2. 상태 토픽은 현재 `std_msgs/String` JSON 형태
- `/patrol/status`는 현재 빠른 연결을 위해 JSON 문자열을 발행합니다.
- 문서에서 제안한 `MissionStatus.msg` 또는 별도 인터페이스가 준비되면 그쪽으로 승격하는 것이 더 적절합니다.

### 3. 이번 턴에서는 Gazebo 실제 순찰 주행까지는 수행하지 않음
- 노드 기동, 서비스 등록, 설정 로딩, 빌드 검증까지는 확인했습니다.
- 다만 실제 Gazebo + AMCL + Nav2 + map 환경에서 17개 waypoint 전체를 완주하는 end-to-end 검증은 아직 별도로 진행해야 합니다.

---

## 기대 효과

- 이제 `agribot_navigation` 패키지 기준으로 자동 순찰의 최소 실행 경로가 생겼습니다.
- 팀원이 `navigation.launch.py`를 띄운 뒤 서비스 호출만으로 순찰을 시작, 중지, 재개할 수 있습니다.
- `S14P-205`에서 정의한 waypoint 메타데이터가 실제 제어 로직에서 재사용되기 시작했기 때문에, 이후 perception / mission / harvest / dashboard 연결이 쉬워집니다.
- 향후 `RunPatrol.action`이나 `MissionStatus.msg`를 도입하더라도, 현재 노드 구조를 기반으로 단계적으로 확장할 수 있습니다.

---

## 후속 작업 제안

- Gazebo + RViz 환경에서 17개 기본 순찰 시퀀스 전체 완주 검증
- `MissionStatus.msg` 또는 `RunPatrol.action`으로 인터페이스 승격
- perception 이벤트 발생 시 inspect waypoint에서 접근 pose로 분기하는 mission 연동 추가
- 수확 또는 예외 처리 후 `home_pose` 복귀 및 원래 순찰 지점 재개 로직 연결

## 변경 파일

- agribot_ws/src/agribot_navigation/agribot_navigation/patrol_node.py
- agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py
- agribot_ws/src/agribot_navigation/launch/navigation.launch.py
- agribot_ws/src/agribot_navigation/setup.py
- agribot_ws/src/agribot_navigation/package.xml
- agribot_ws/src/agribot_navigation/maps/README.md


[S14P-42] 접근·복귀 경로 로직 개선

## 개요

S14P-42 접근·복귀 경로 로직을 진행했습니다.

이번 MR의 목적은 다음 3가지입니다.

1. `S14P-205`, `S14P-41`에서 정리한 순찰 inspect waypoint와 작물 메타데이터를 이용해, 특정 토마토에 대한 접근 위치를 계산한다.
2. 수확 작업이 끝난 뒤 로봇이 원래 순찰 지점으로 복귀하거나, 필요하면 `home_pose`로 복귀할 수 있는 정책을 명확히 만든다.
3. patrol 노드와 Nav2 사이에 harvest 전용 조정 노드를 추가해 `정지 -> 접근 -> 작업 대기 -> 복귀 -> 순찰 재개` 흐름을 실제 ROS 2 노드 기준으로 연결한다.

이번 작업으로 `agribot_navigation` 패키지에 접근 pose 계산 유틸리티와 복귀 조정 노드를 추가했고, 설치된 패키지 기준 CLI 계산, 단위 테스트, 빌드까지 통과하는 상태로 정리했습니다.

---

## 배경

`S14P-41`까지 진행한 시점에는 순찰 waypoint를 따라 이동하고 중간 정지/재개까지는 가능했지만, 특정 토마토를 발견했을 때 "어디로 짧게 파고들어 접근할지"와 "작업 후 어디로 복귀할지"를 계산하는 로직은 아직 없었습니다.

구체적으로는 아래 문제가 있었습니다.

- `patrol_waypoints.yaml`에는 inspect anchor까지만 정의되어 있고, 토마토 수확용 세부 접근 좌표는 비어 있었음
- `crop_instances.yaml`에는 식물/토마토 개체 pose가 정리되어 있지만, 이를 순찰 route와 연결해 접근 pose로 바꾸는 코드가 없었음
- 기존 `patrol_node`만으로는 수확 시점에 순찰을 잠시 멈추고, 작업 후 원래 자리나 `home_pose`로 복귀한 뒤 다시 순찰을 이어가는 흐름을 표현하기 어려웠음

즉, 이번 티켓의 핵심은 "토마토를 찾으면 inspect 지점에서 어디까지 접근할지 계산하고, 작업 후 어디로 복귀할지 상태 전환까지 묶는 것"이었습니다.

---

## 주요 변경 사항

### 1. harvest routing 설정과 validator 확장
- `agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml`에 `harvest_routing` 섹션을 추가했습니다.
- 아래 정책을 설정값으로 분리했습니다.
  - 재배 베드 경계 기준 접근 마진
  - inspect 지점에서 허용할 최대 측면 오프셋
  - 기본 복귀 모드
  - fallback 복귀 모드
- `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py`도 함께 확장해, `coordinate_rationale.source_bounds`와 `harvest_routing` 값을 로드하고 무결성을 검증하도록 만들었습니다.

### 2. 토마토 기준 접근 pose 계산 유틸리티 추가
- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_routing.py`를 새로 추가했습니다.
- 이 모듈은 아래 3개 데이터를 함께 읽습니다.
  - `patrol_waypoints.yaml`
  - `crop_instances.yaml`
  - route / inspect waypoint 메타데이터
- 내부 동작은 아래 순서로 구성했습니다.
  - `tomato_id`에 해당하는 부모 `plant_id` 확인
  - 해당 토마토를 관찰하는 inspect waypoint와 route 선택
  - 좌우 베드 방향에 따라 재배 베드 경계 안쪽으로 접근 pose 투영
  - `resume_patrol` 또는 `home` 기준 return target 계산
  - fallback return target 계산
- 즉, 접근 좌표를 하드코딩하지 않고 `토마토 ID -> 순찰 route -> 접근 pose -> 복귀 waypoint`로 파생시키는 구조를 만들었습니다.

### 3. patrol 정지/접근/복귀 조정 노드 추가
- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_route_node.py`를 새로 추가했습니다.
- 이 노드는 아래 입력을 받습니다.
  - `CropStatus`의 `ready_to_harvest=true`
  - 수동 테스트용 `std_msgs/String` 기반 `harvest/request`
  - `patrol/status`
- 내부 흐름은 아래 상태로 구성했습니다.
  - `waiting_for_patrol_pause`
  - `approaching`
  - `harvesting`
  - `returning`
  - `resuming_patrol`
  - `completed`
  - `error`
- 순찰이 돌고 있으면 먼저 `/patrol/stop`을 호출하고, 정지 확인 후 접근 goal을 보냅니다.
- 접근 성공 후에는 짧은 dwell로 수확 작업 대기 시간을 시뮬레이션하고, 이후 복귀 goal을 전송합니다.
- 복귀 성공 시 `return_mode=resume_patrol`이면 `/patrol/resume`을 호출하고, 실패하면 `fallback_return_waypoint_id`로 한 번 더 복귀를 시도합니다.

### 4. navigation launch와 엔트리포인트 연결
- `agribot_ws/src/agribot_navigation/launch/navigation.launch.py`에 아래 launch argument를 추가했습니다.
  - `use_harvest_route`
  - `crop_instances_file`
  - `harvest_return_mode`
- 기본 launch에서 `harvest_route_node`를 함께 띄울 수 있게 연결했습니다.
- `agribot_ws/src/agribot_navigation/setup.py`에는 아래 console script를 추가했습니다.
  - `plan_harvest_route`
  - `harvest_route_node`
- `agribot_ws/src/agribot_navigation/package.xml`에는 `agribot_interfaces` 의존성을 추가했습니다.

### 5. 단위 테스트 추가
- `agribot_ws/src/agribot_navigation/test/test_harvest_routing.py`를 새로 추가했습니다.
- 아래 경우를 테스트했습니다.
  - 우측 베드 토마토에 대해 lane 방향에 맞는 접근 pose 계산
  - `home` 강제 복귀 모드 계산
  - `resume_patrol` 시 preferred return waypoint 반영
- 이 테스트로 접근/복귀 계산 로직을 Gazebo 없이도 빠르게 검증할 수 있게 했습니다.

---

## 검증 방법

아래 순서로 문법 검사, 단위 테스트, 빌드, 설치된 entry point 실행을 직접 확인했습니다.

```bash
cd ~/SSAFY/S14P21A602

python3 -m py_compile \
  agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py \
  agribot_ws/src/agribot_navigation/agribot_navigation/harvest_routing.py \
  agribot_ws/src/agribot_navigation/agribot_navigation/harvest_route_node.py \
  agribot_ws/src/agribot_navigation/test/test_harvest_routing.py

python3 -m pytest \
  agribot_ws/src/agribot_navigation/test/test_harvest_routing.py

cd ~/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --packages-select agribot_navigation
source install/setup.bash

ros2 run agribot_navigation plan_harvest_route --tomato-id gh01_plant_09_tomato_01
```

추가로 아래 파일 내용을 직접 확인했습니다.

```bash
sed -n '1,260p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/agribot_navigation/harvest_routing.py
sed -n '1,320p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/agribot_navigation/harvest_route_node.py
sed -n '1,120p' ~/SSAFY/S14P21A602/agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml
```

---

## 검증 결과

### 정상 확인

- `py_compile` 문법 검사 통과
- `pytest` 기준 `3 passed`
- `agribot_navigation` 패키지 단독 빌드 통과
- 설치된 패키지 기준 `plan_harvest_route` 실행 성공

### 확인 로그 요약

- `python3 -m pytest .../test_harvest_routing.py`
  - `3 passed in 0.08s`
- `ros2 run agribot_navigation plan_harvest_route --tomato-id gh01_plant_09_tomato_01`
  - `route_id = greenhouse_01_right_bed_southbound`
  - `inspect_waypoint_id = greenhouse_01_right_rear_inspect`
  - `approach_pose = (15.3179, 166.456497, yaw 0.0)`
  - `return_mode = resume_patrol`
  - `fallback_return_waypoint_id = greenhouse_01_home`

즉, 이번 MR은 수확 접근·복귀 아이디어 수준이 아니라, 실제 설치된 패키지 기준으로 접근 좌표와 복귀 목표를 계산하고 재사용할 수 있는 상태까지 연결한 작업입니다.

---

## 확인된 이슈 및 주의사항

### 1. 이번 MR은 접근·복귀 경로 기준선까지가 범위
- 접근 pose 계산과 복귀 상태 전환은 포함했습니다.
- 하지만 실제 토마토를 제거하거나 바구니 적재를 기록하는 수확 액션 서버는 아직 포함하지 않았습니다.
- 이 부분은 후속 `S14P-453`, `S14P-454`, `S14P-455`와 자연스럽게 이어지는 범위입니다.

### 2. 수확 작업은 현재 dwell 기반 시뮬레이션
- `harvest_route_node`는 접근 후 `harvest_dwell_sec`만큼 대기한 뒤 복귀하도록 구성했습니다.
- 즉, 현재는 "수확 시퀀스 placeholder" 역할이며, 실제 manipulator/harvest action이 붙으면 이 부분을 교체하는 구조가 맞습니다.

### 3. Gazebo 기반 end-to-end 주행은 아직 별도 검증 필요
- 이번 turn에서는 계산 로직, 단위 테스트, build, 설치된 CLI 실행까지 확인했습니다.
- 실제 Gazebo + AMCL + Nav2 + patrol node가 함께 떠 있는 상태에서 접근 후 복귀까지 완주하는 검증은 후속 통합 세션에서 진행하면 됩니다.

---

## 기대 효과

- perception 또는 mission 쪽에서 `tomato_id`만 넘기면 접근 pose와 복귀 target을 같은 기준으로 계산할 수 있습니다.
- `patrol_node`와 수확 로직 사이에 명확한 연결 지점이 생겨, 이후 수확 액션 서버를 붙이기 쉬워집니다.
- 복귀 정책을 `resume_patrol`과 `home`으로 분리해, 데모 상황에 따라 안전한 fallback을 유지할 수 있습니다.
- 순찰 anchor와 작물 canonical ID가 실제 harvest routing까지 이어지기 시작했기 때문에, 팀원 간 데이터 계약이 더 단단해집니다.

---

## 후속 작업 제안

- `S14P-453` 수확 액션 서버와 연결해 dwell 대신 실제 수확 결과를 받아 복귀 시작
- `S14P-455`에서 `RETURN_HOME -> RESUME` 미션 상태를 별도 인터페이스로 승격
- Gazebo 통합 세션에서 `harvest/request` 발행 후 접근/복귀 완주 검증
- 필요하면 접근 pose 계산 시 토마토 높이, 정면 정렬, 그리퍼 진입 방향까지 반영하도록 고도화

## 변경 파일

- agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py
- agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml
- agribot_ws/src/agribot_navigation/agribot_navigation/harvest_routing.py
- agribot_ws/src/agribot_navigation/agribot_navigation/harvest_route_node.py
- agribot_ws/src/agribot_navigation/launch/navigation.launch.py
- agribot_ws/src/agribot_navigation/setup.py
- agribot_ws/src/agribot_navigation/package.xml
- agribot_ws/src/agribot_navigation/test/test_harvest_routing.py

---

# MR: 자율 매핑 순찰 경로를 저발열 기준으로 안정화하고 완주까지 검증

## 배경

이번 작업의 목표는 `autonomous_mapping.launch.py` 기반 자율 매핑에서 반복적으로 발생하던 다음 문제를, 노트북 연산량과 발열을 과하게 늘리지 않는 방향으로 해결하는 것이었습니다.

- 시작 직후 `Start Coordinates ... was outside bounds`
- `Home Pose`/`Left Front-Mid Inspection`에서의 `Start occupied`, `Goal outside bounds`
- left lane 진행 중 `RegulatedPurePursuitController detected collision ahead!`
- `FrontStopZone polygon` 반복 정지
- recovery 연쇄 실패 후 `error_code=104`, `error_code=205`
- 긴 순찰 구간에서 거의 도착했는데도 subgoal 완료 처리 전에 abort되는 현상

핵심 방향은 "새로운 무거운 알고리즘을 얹는 것"이 아니라, 기존 patrol 중심 매핑 경로를 더 가볍고 구조적으로 안정하게 만드는 것이었습니다.

---

## 변경 요약

### 1. 매핑용 Nav2를 rolling odom 기반으로 정리

`nav2_mapping_params.yaml`을 손봐서 매핑용 planner/controller가 SLAM 초기 불안정성에 덜 민감하게 만들었습니다.

- 매핑용 전역 프레임을 `map`이 아니라 `odom` 기준 rolling window로 사용
- 글로벌 코스트맵에 불필요한 `static_layer`를 빼고 obstacle/inflation만 사용
- global/local costmap origin을 명시해 초기 `outside bounds`를 줄임

이 변경은 계산량을 늘리지 않고, 오히려 정적 맵 의존과 초기 경계 문제를 줄이는 효과가 있습니다.

### 2. patrol batching과 장거리 레인 이동을 현실적인 단위로 분할

`patrol_node.py`를 중심으로 순찰 목표 발행 로직을 재구성했습니다.

- `batching`은 같은 `lane_id` 내부에서만 허용
- 긴 레인 이동은 synthetic segment로 분할
- segment 길이는 매핑 launch에서 `10m`로 낮춰 과도한 장거리 목표를 피함
- inspect waypoint는 lane heading 기준 yaw를 유지해, observation 뒤 불필요한 측면 회전을 줄임

초기에는 `24m` segment였는데, 실제 Gazebo 주행에서 `Left Front-Mid` 구간이 자주 꼬여 `10m`까지 낮췄습니다.

### 3. 이미 가까운 목표는 억지로 다시 계획하지 않도록 보정

`patrol_node.py`에 두 종류의 경량 보정을 넣었습니다.

- waypoint/segment가 이미 현재 odom pose 근처면 바로 통과
- goal이 충분히 가까워졌는데 Nav2가 abort하는 경우, patrol 레벨에서 `soft completion`으로 성공 처리

이 로직은 CPU/GPU 부담을 늘리지 않고, 오히려 불필요한 재계획과 recovery 반복을 줄입니다.

### 4. 실제 로봇 크기에 맞춰 footprint와 stop zone을 축소

기존 매핑용 footprint는 실제 차체보다 크게 잡혀 있었습니다.

- 실제 chassis collision: `0.4 x 0.3`
- 이전 Nav2 footprint: `0.48 x 0.40`

이를 실제 모델 크기에 가깝게 줄였습니다.

- local/global footprint를 `[[0.20, 0.15], ...]`로 수정
- footprint padding 제거
- `FrontStopZone`도 실제 차체 앞 기준으로 더 타이트하게 축소

이 수정은 안전을 크게 해치지 않으면서, 좁은 레인에서의 과민한 정지를 줄이는 데 직접적으로 기여했습니다.

### 5. controller를 공격적 추종에서 저발열 안정 추종으로 완화

`RegulatedPurePursuit` 파라미터를 다음 방향으로 조정했습니다.

- `desired_linear_vel` 감소
- `lookahead_dist` / `max_lookahead_dist` 감소
- `max_allowed_time_to_collision_up_to_carrot` 감소
- `xy_goal_tolerance`를 매핑용으로 완화

즉, 더 빨리/더 멀리 보는 공격적인 추종 대신, 짧게 보고 천천히 가는 sweep 성향으로 바꿨습니다. 이건 노트북 발열을 늘리는 방향이 아니라 오히려 줄이는 방향입니다.

### 6. 실제 레인 좌표를 통로 중앙 쪽으로 재배치

`patrol_waypoints.yaml`의 좌우 레인을 더 안전한 쪽으로 옮겼습니다.

- left lane: `x = -10.0 -> -8.5`
- right lane: `x = 13.0 -> 11.8`

이후 실제 Gazebo 주행 로그에서 left lane 실패 지점이 크게 뒤로 밀렸고, 최종적으로 전체 순찰 완주까지 이어졌습니다.

---

## 수정 파일

- `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_node.py`
- `agribot_ws/src/agribot_navigation/config/nav2_mapping_params.yaml`
- `agribot_ws/src/agribot_navigation/config/collision_monitor_mapping.yaml`
- `agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml`
- `agribot_ws/src/agribot_navigation/launch/autonomous_mapping.launch.py`
- `agribot_ws/src/agribot_navigation/test/test_patrol_batching.py`
- `docs/현 프로젝트 진행 상황 및 명령어 모음.md`

---

## 검증 과정

### 정적 검증

- `python3 -m py_compile` 통과
- `pytest agribot_ws/src/agribot_navigation/test/test_patrol_batching.py -q`
  - 최종 `10 passed`
- `colcon build --packages-select agribot_navigation --symlink-install` 반복 통과
- YAML 파싱 검증 통과

### 실제 런타임 검증

초기에는 `sweep_hybrid` 기준으로 직접 재주행하며 에러를 좁혔고, 최종 완주 검증은 가장 가벼운 조건인 아래 명령으로 진행했습니다.

```bash
cd /home/ssafy/SSAFY/S14P21A602/agribot_ws
source /opt/ros/jazzy/setup.bash
source install/setup.bash
ros2 launch agribot_navigation autonomous_mapping.launch.py \
  mapping_strategy:=patrol_only \
  use_rviz:=false
```

완주 확인은 `/mapping_patrol/status`로 했습니다.

최종 확인 결과:

- `state: completed`
- `message: Patrol completed the configured waypoint sequence.`
- `next_waypoint_index: 17`
- `error: null`

즉, 이번 수정은 "로그가 덜 나오는 것" 수준이 아니라, 실제 순찰 기반 자율 매핑 주행이 시작부터 마지막 `Home Pose` 복귀까지 끝나는 상태로 바꾼 작업입니다.

---

## 런타임에서 실제로 해결된 문제

### 해결된 항목

- 시작 직후 `outside bounds`
- 시작점 `Home Pose` 재계획 실패
- `Left Lane Entry -> Left Front Inspection -> Left Front-Mid Inspection` 연속 실패
- 장거리 lane sweep 중 recovery 연쇄 실패
- 거의 도착했는데 subgoal 완료 전 abort되던 현상
- 최종 우측 레인 종료 후 복귀 경로 불안정

### 남긴 의도적 선택

- 검증은 `patrol_only`로 완주 확인
  - 가장 가볍고, 문제였던 순찰 주행 경로만 순수하게 검증 가능
- `sweep_hybrid`는 같은 patrol 경로를 공유하므로, 순찰 구간 자체는 같은 수정 효과를 받음
- frontier hole-fill 전체는 이번 MR의 핵심 검증 범위에서 제외

---

## 발열/연산량 관점에서의 판단

이번 MR은 "성능을 더 쓰는 대신 성공시키는" 방향이 아니라, 아래처럼 가볍게 만드는 쪽을 유지했습니다.

- `RViz` 없이 검증
- frontier 없이 `patrol_only` 기준 검증
- controller 속도와 lookahead를 낮춤
- segment를 짧게 쪼개되, 고주파 planner/controller 주파수는 올리지 않음
- footprint/stop zone 과대평가를 줄여 불필요한 recovery 연쇄를 감소시킴

결과적으로 벽시계 시간은 늘 수 있지만, 노트북을 더 뜨겁게 만드는 식의 무거운 연산 증가는 피했습니다.

---

## 후속 제안

- `sweep_hybrid` 기본 모드에서도 frontier 프로세스가 불필요하게 살아나는지 launch 조건 재확인
- right lane 마지막 turn/home 복귀도 patrol 전용 lane driver로 바꾸면 더 로보락 스타일에 가까워질 수 있음
- 추후 frontier hole-fill 검증은 별도 세션에서 진행하되, 현재 patrol 경로는 기준선으로 유지

---

## feature/S14P21A602-97

### 1차 MR: 새로운 직사각형 farm 맵 기준 매핑/자율주행 기본값 재정비

### 배경

- 기존 `greenhouse_01` 기준 설정은 좌표계와 순찰 경로가 새 직사각형 `farm_world`와 맞지 않았다.
- 매핑용 기본 경로가 여전히 예전 온실형 구조를 가정하고 있어서, 새 맵에서는 직진 위주 sweep의 장점을 살리기 어려웠다.
- 목표는 새 맵에서 불필요한 재방문과 장거리 점프를 줄이면서, 기존 저발열 하이브리드 전략을 유지하는 것이었다.

### 이번 MR에서 반영한 내용

- `autonomous_mapping.launch.py`
  - 기본 boundary map을 `farm_exploration_boundary.yaml`로 교체
  - 기본 매핑 순찰 파일을 `farm_mapping_patrol_waypoints.yaml`로 교체
  - 매핑 순찰 segment 길이와 근접 완료 허용치를 새 맵에 맞게 조정
- `farm_mapping_patrol_waypoints.yaml`
  - 새 직사각형 농장 맵을 9개 aisle로 훑는 serpentine sweep 경로 추가
  - `farm_01_home` 기준으로 시작해 좌우 레인을 반복 방문하는 mapping 전용 경로 정의
- `frontier_explorer.yaml`
  - 가까운 직진형 goal을 더 선호하도록 거리/전방 bias 관련 값 조정
  - 이미 일부 알려진 맵에서는 bootstrap을 길게 끌지 않도록 known ratio 기준 추가
  - hole-fill 성격에 맞게 최대 goal 거리, recent goal 억제, coverage lane spacing을 새 맵에 맞게 튜닝
- `frontier_explorer.py`
  - 멀리 있는 큰 frontier를 과도하게 우대하던 scoring을 수정
  - 부분적으로 맵이 채워져 있으면 bootstrap 대신 frontier 단계로 바로 전환되도록 로직 보강
- `nav2_mapping_params.yaml`
  - 직선 통로에서 불필요한 흔들림을 줄이도록 lookahead와 추종 속도 재조정
  - 전역 rolling costmap 크기를 `24 x 24`로 줄여 불필요한 연산량 증가를 억제
- `navigation.launch.py`, `localization.launch.py`
  - 기본 정적 맵을 `farm_map.yaml`로 전환
- 테스트
  - `test_frontier_explorer.py`에 거리 보상 검증 추가
  - `test_mapping_patrol_plan.py`를 추가해 새 farm mapping patrol plan 로드와 경로 개수를 검증

### 왜 이렇게 바꿨는가

- 새 맵은 직사각형 레인 구조라서 frontier 중심보다 patrol-first/hole-fill 보완 구조가 더 자연스럽다.
- 정형 레인에서는 먼 frontier 점프보다 가까운 연속 전진 목표를 우대하는 편이 실제 주행 안정성이 높다.
- costmap과 planner 범위를 필요 이상으로 크게 가져가면 노트북 발열과 연산량이 올라가므로, 새 맵 크기에 맞게 줄이는 편이 합리적이었다.

### 검증

- `python3 -m py_compile`로 수정한 Python/launch 파일 확인
- `pytest`로 `test_frontier_explorer.py`, `test_patrol_batching.py`, `test_mapping_patrol_plan.py` 통과
- `colcon build --packages-select agribot_navigation --symlink-install` 통과
- `autonomous_mapping.launch.py mapping_strategy:=patrol_only use_rviz:=false` 30초 smoke test에서
  - 새 `farm_mapping_patrol_waypoints.yaml`가 로드되는 것
  - `farm_01_home`로 첫 goal이 전송되는 것
  - 실행 종료 후 `/clock`과 관련 프로세스가 남지 않는 것 확인

### 한계와 다음 단계

- 이번 1차 MR은 매핑/자율주행 기본값만 새 `farm_world`에 맞췄다.
- 수확용 `crop_instances`, inspect waypoint와의 관측 계약, `harvest_routing`은 아직 예전 `greenhouse_01` 전제를 그대로 갖고 있다.
- 다음 단계에서 수확 메타데이터와 수확 경로를 새 맵 기준으로 같이 일반화해야 한다.

### 커밋

- `35c27f2` 직사각형 농장 맵에 맞춰 매핑 자율주행 기본값을 재정비한다

---

### 2차 MR: 새 farm 맵 기준 수확 메타데이터와 수확 경로 일반화

### 배경

- 1차 작업으로 매핑/자율주행 기본값은 새 `farm_world`에 맞췄지만, 수확용 메타데이터는 여전히 예전 `greenhouse_01` 계약을 그대로 쓰고 있었다.
- 특히 아래 세 가지가 서로 어긋나 있었다.
  - `crop_instances.yaml`의 plant / tomato ID와 좌표
  - `patrol_waypoints.yaml`의 inspect waypoint 관측 대상 연결
  - `harvest_routing.py`의 좌우 2베드 전용 접근 자세 계산
- 이 상태에선 새 직사각형 맵에서 수확 route planning이 실제 배치와 맞지 않았다.

### 이번 MR에서 반영한 내용

- `agribot_description/config/crop_instances.yaml`
  - `farm_world.sdf`의 `tomato_0 ~ tomato_71` plant grid를 기준으로 crop catalog를 전면 재생성
  - `farm_01` / `farm01_plant_<nn>` / `farm01_plant_<nn>_tomato_01` 규칙으로 canonical ID를 재정의
  - 총 `72`개 plant, `72`개 tomato fruit 메타데이터를 일관되게 반영
- `agribot_navigation/config/patrol_waypoints.yaml`
  - 예전 온실형 좌표를 제거하고, 새 직사각형 맵의 9개 aisle 기준 harvest patrol metadata로 재작성
  - 각 aisle에 대해 짧은 inspect stop 5개씩을 두고, 인접 row group을 관측 대상으로 연결
  - 기본 수확 patrol sequence, route observed list, home pose를 모두 `farm_01` 기준으로 교체
- `agribot_navigation/agribot_navigation/harvest_routing.py`
  - `lane_side == left/right` 가정과 bed-edge 기반 X 계산을 제거
  - inspect waypoint에서 target tomato 쪽으로 짧게 이동한 뒤 standoff를 남기는 generic approach pose 계산으로 일반화
  - 기존 Y축만 보던 후보 선택을 inspect waypoint와 tomato의 2D 거리 기준으로 개선
  - 현재 patrol inspect waypoint가 이미 target을 보고 있으면 그 waypoint를 우선 사용하는 bias를 추가
- `agribot_navigation/test/test_harvest_routing.py`
  - 새 `farm_01` 계약 기준으로 테스트를 전면 갱신
  - 72개 crop catalog와 patrol 관측 범위가 전체 tomato ID를 모두 덮는지 검증
  - generic approach pose, home return, preferred inspect waypoint 선택을 검증
- `docs/현 프로젝트 진행 상황 및 명령어 모음.md`
  - 수동 저장 맵 기본 경로를 `greenhouse_map`에서 `farm_map`으로 수정
  - 새 `farm_01` crop ID와 `plan_harvest_route` 예시 명령을 추가

### 왜 이렇게 바꿨는가

- 새 맵은 직사각형 grid라서 예전 좌우 2베드 전용 수확 접근식이 더 이상 맞지 않는다.
- 수확 route planning은 특정 맵 형상을 하드코딩하기보다, 현재 inspect waypoint와 target tomato 사이의 실제 기하 관계로 계산하는 편이 훨씬 범용적이다.
- 현재 관측 중인 inspect waypoint를 우선하면 이미 지나간 aisle을 다시 크게 되돌아가는 일을 줄일 수 있다.
- harvest metadata도 `farm_world.sdf`의 실제 배치와 같은 기준으로 맞춰야 perception / mission / UI가 모두 같은 plant / tomato ID를 사용할 수 있다.

### 검증

- `python3 -m py_compile`로 `harvest_routing.py`, `harvest_route_node.py` 확인
- `ros2 run agribot_navigation validate_patrol_waypoints --patrol-waypoints .../patrol_waypoints.yaml`
  - `farm_01` 기준으로 64 waypoints, 9 routes 로드 확인
  - 각 harvest lane의 inspect pose 수와 observed plant 수 확인
- `pytest`
  - `test_harvest_routing.py`
  - `test_frontier_explorer.py`
  - `test_mapping_patrol_plan.py`
  - `test_patrol_batching.py`
  - 총 `32 passed`
- `colcon build --packages-select agribot_navigation agribot_description --symlink-install` 통과
- `ros2 run agribot_navigation plan_harvest_route --tomato-id farm01_plant_37_tomato_01`
  - 새 crop catalog와 patrol metadata를 기준으로 approach / return pose가 출력되는 것 확인

### 기대되는 동작 변화

- 새 `farm_world` 기준으로 perception 또는 수동 요청이 `farm01_plant_<nn>_tomato_01` ID를 넘기면, 수확 route planning이 실제 patrol inspect 지점과 일치하는 route를 계산한다.
- 직사각형 맵에서 특정 tomato를 수확하러 갈 때, 예전처럼 좌우 bed edge를 하드코딩해 옆으로 튀는 대신 inspect stop에서 target 쪽으로 짧게 접근한다.
- 현재 inspect 중인 waypoint가 이미 그 tomato를 관측 중이라면, route selection이 그 위치를 우선 사용해 불필요한 재방문을 줄인다.

### 남은 한계

- 이번 MR은 수확 메타데이터와 수확 접근 경로까지 정렬한 것이다.
- 실제 perception 노드가 `farm01_*` tomato ID를 어떤 기준으로 publish할지는 별도 perception 런타임 작업이 필요하다.
- 실제 수확 매니퓰레이터 접근성은 아직 Gazebo / 실제 로봇에서 별도 검증이 필요하다.

### 커밋

- `736bfe1` 직사각형 농장 맵 기준으로 수확 메타데이터와 경로를 일반화한다

---

### 3차 MR: 사각 농장 맵 매핑 순찰의 시작 경로와 진행 실패를 저발열로 안정화

### 배경

- 새 `farm_world`에 맞춘 mapping patrol을 실제로 돌려보니, 다음 두 문제가 겹쳐 초기 구간에서 반복 정지와 recovery 루프가 발생했다.
  - `NavigateThroughPoses`가 같은 lane 안의 긴 구간까지 한 번에 묶으면서, 첫 sweep가 지나치게 긴 batch route로 시작됨
  - mapping patrol의 `home`과 default sequence가 실제 Gazebo spawn과 맞지 않아, 시작부터 불필요한 장거리 이동과 progress failure가 발생함
- 로그상으로는 `Failed to make progress`, `Controller patience exceeded`, `FrontStopZone polygon`, `spin failed`, `backup failed`가 반복됐다.

### 이번 MR에서 반영한 내용

- `agribot_navigation/agribot_navigation/patrol_node.py`
  - `collect_batch_goal_end_index()`에 `max_batch_path_length_m` 제한을 추가해, 같은 lane 안이라도 지나치게 긴 batch route는 더 이상 묶지 않도록 변경
  - `PatrolNode`에 `max_batch_path_length_m` 파라미터를 추가하고 launch에서 주입하도록 연결
  - auto-start가 첫 `/odom`을 받기 전에 시작되지 않도록 수정해서, spawn과 같은 `home` goal을 괜히 보내지 않도록 보강
- `agribot_navigation/launch/autonomous_mapping.launch.py`
  - mapping patrol에 `max_batch_path_length_m: 6.0`, `max_lane_segment_length_m: 6.0`을 적용
  - 긴 lane은 batch가 아니라 짧은 segment 기반으로 쪼개서 진행하게 조정
- `agribot_navigation/config/farm_mapping_patrol_waypoints.yaml`
  - `farm_01_home`를 `(0.0, 0.0)`으로 바꿔 실제 Gazebo spawn과 정렬
  - default sequence를 `west outer` 시작이 아니라 `center-start` sweep로 재배치
  - 즉 `home -> center north -> center south -> east block -> west block` 순으로 바꿔 첫 구간을 직진 우선으로 정리
  - lane 끝점의 `y`를 `±8.9`에서 `±7.6`으로 줄여, 북/남 벽과 crop row 끝단 근처에서 collision monitor가 과민하게 반응하던 구간을 줄임
  - route metadata도 새 center-start sweep 흐름에 맞게 함께 갱신
- `agribot_navigation/config/nav2_mapping_params.yaml`
  - progress checker를 `required_movement_radius: 0.15`, `movement_time_allowance: 10.0`으로 소폭 완화
  - 실제로 조금씩 전진 중인 goal을 너무 빨리 `progress failure`로 취급하던 false positive를 줄이는 목적
- 테스트
  - `test_patrol_batching.py`에 batch path length 상한 검증 추가
  - `test_mapping_patrol_plan.py`에 `home == spawn`, `center-start` sequence 검증 추가

### 왜 이렇게 바꿨는가

- 직사각형 레인 구조에서 제일 자연스러운 시작점은 실제 spawn과 맞닿아 있는 center aisle이다.
- 처음부터 서쪽 끝 레인으로 크게 꺾어 들어가면, 전진 위주 sweep보다 오히려 장거리 재계획과 recovery가 늘어난다.
- 긴 lane을 batch로 한 번에 보내면 실제 하드웨어 제약과 costmap 갱신 오차가 누적돼 progress failure가 잘 난다.
- 반대로 짧은 segment와 center-start sweep는 계산량을 거의 늘리지 않으면서도 더 예측 가능한 주행을 만든다.
- lane 끝점을 안쪽으로 당긴 것은 충돌 안전 여유를 확보하기 위한 것이고, 남는 가장자리 strip은 이후 hole-fill/frontier가 메우도록 두는 쪽이 더 안정적이다.

### 검증

- `python3 -m py_compile`
  - `patrol_node.py`
- `pytest`
  - `test_patrol_batching.py`
  - `test_mapping_patrol_plan.py`
  - 총 `13 passed`
- `colcon build --packages-select agribot_navigation --symlink-install` 통과
- 실제 launch 로그 기준으로는 다음을 확인했다.
  - `farm_01_home` goal이 실제 spawn과 같은 좌표 `(0.0, 0.0)`로 전송되는 것
  - 기존처럼 `west outer` batch route가 아니라 `Center North (farm_01_lane_05_north)`로 첫 실주행이 넘어가는 것
- 다만 최종 runtime 재검증 중 Gazebo `/clock` time jump가 다시 발생해, 마지막 endpoint 축소 후의 전체 완주 검증은 이번 MR 범위에서 끝까지 확정하지 못했다.

### 기대되는 동작 변화

- mapping patrol이 spawn과 맞지 않는 첫 goal 때문에 제자리에서 낭비하는 일이 줄어든다.
- 첫 sweep가 center aisle 직진으로 시작하므로, 불필요한 초기 대각 이동과 batch abort가 줄어든다.
- 같은 lane 안의 긴 batch navigation 대신 segment 기반 진행으로 바뀌어 `Failed to make progress`와 recovery 연쇄 발생 빈도가 낮아진다.
- lane 끝점이 벽과 너무 가까운 문제를 줄여, `FrontStopZone polygon`으로 인한 말단 구간 정지가 덜해진다.

### 남은 한계

- 이번 MR은 mapping patrol과 progress false positive를 줄이는 쪽에 집중한 것이다.
- Gazebo 시간 점프가 재현 환경에서 반복되기 때문에, 완전한 end-to-end 완주 여부는 시뮬레이션 clock 안정화와 함께 다시 확인할 필요가 있다.
- 필요하면 다음 단계에서 `collision_monitor_mapping.yaml`의 stop zone과 `farm_world` spawn/physics 초기화도 같이 손봐야 한다.

### 커밋

- `55707df` 사각 농장 맵 순찰 시작 경로와 진행 실패를 저발열로 안정화한다

---

[feature/S14P21A602-99] 가제보 종료시 잔여 프로세스 정리를 자동화

### 배경

- Gazebo를 여러 번 켰다 끄는 동안 `gz sim`, `ros_gz_state_bridge`, `rviz2`, `slam_toolbox`, Nav2 lifecycle 노드가 완전히 내려가지 않고 남는 경우가 잦았다.
- 특히 `Ctrl+C`나 터미널 종료로 `ros2 launch`를 끊을 때, 부모 launch 프로세스는 종료됐는데 하위 프로세스가 남아 `/clock`이나 TF를 계속 점유하는 패턴이 반복됐다.
- 사용자는 실행 전마다 긴 `pkill`과 좀비 부모 정리 명령을 직접 쳐야 했고, 이 상태가 누적되면 다음 실행에서 `time jump`, `TF buffer clearing`, `clock publisher 잔존` 같은 2차 문제가 이어졌다.

### 왜 이런 일이 자주 생겼는가

- `true zombie`는 이미 끝난 자식을 부모가 회수하지 못할 때 생기므로, zombie 자체를 직접 kill할 수 없다.
- Gazebo GUI, `ros_gz_bridge`, RViz, Nav2처럼 서로 다른 프로세스 그룹과 GUI 이벤트 루프를 갖는 프로그램은 `ros2 launch` 하나만 죽인다고 항상 같이 정리되지 않는다.
- 따라서 프로젝트 코드만으로 “좀비를 절대 안 남기게” 만드는 것은 현실적으로 어렵고, 실행 전/종료 후 정리 루틴을 자동화하는 쪽이 가장 안전하다.

### 이번 MR에서 반영한 내용

- `scripts/cleanup_sim_processes.sh`
  - 기존 수동 `pkill` 명령을 스크립트로 승격
  - 기존 목록에 없던 잔여 프로세스 패턴을 함께 추가
  - 추가된 주요 대상
    - `smoother_server`, `behavior_server`, `waypoint_follower`
    - `collision_monitor`, `mapping_collision_monitor`
    - `frontier_explorer`, `mapping_patrol_node`
    - `parameter_bridge`, `ros_gz_state_bridge`, `ros_gz_lidar_bridge`
    - `ros_gz_camera_info_bridge`, `ros_gz_camera_image_bridge`, `ros_gz_camera_depth_bridge`, `image_bridge`
    - `robot_state_publisher`, `ekf_node`, `mapping_ekf_filter`, `odom_tf_broadcaster`, `cmd_vel_watchdog`
    - `component_container_mt`, `nav2_container`, `static_transform_publisher`
    - `ros2 launch .*simulation.launch.py`, `.*mapping.launch.py`, `.*autonomous_mapping.launch.py`, `.*navigation.launch.py`
  - 마지막에는 `Z` 상태 항목의 부모 PID를 찾아 `kill -KILL` 하도록 포함
- `scripts/agribot_launch.sh`
  - `ros2 launch` 전용 래퍼 추가
  - 실행 직전에 cleanup 스크립트를 먼저 실행
  - `Ctrl+C`, `TERM`, 정상 종료 모두에 대해 trap을 걸어 launch 자식 프로세스 그룹을 먼저 내리고 cleanup 스크립트를 다시 실행
  - 즉 사용자는 앞으로 raw `ros2 launch` 대신 이 래퍼를 사용하면 된다.
- `docs/현 프로젝트 진행 상황 및 명령어 모음.md`
  - 상단 잔여 프로세스 정리 섹션을 새 스크립트 기준으로 갱신
  - raw fallback `pkill` 패턴도 스크립트와 동일한 확장 목록으로 교체
  - 권장 실행 명령 전부를 `ros2 launch ...` 대신 `/home/ssafy/SSAFY/S14P21A602/scripts/agribot_launch.sh ...` 형식으로 변경
  - `Ctrl+C` 시에도 cleanup이 자동으로 돈다는 사용법을 함께 문서화

### 왜 이렇게 바꿨는가

- Gazebo 잔여 프로세스 문제는 한두 노드만 더 죽인다고 해결되지 않고, 실행 루틴 자체를 통제해야 재발 빈도를 줄일 수 있다.
- 그래서 이번에는 기존 launch 파일의 주행 로직을 건드리지 않고, 실행 래퍼와 cleanup 스크립트 계층을 추가하는 방식으로 분리했다.
- 이 방식이면 mapping, navigation, simulation 명령을 공통으로 안전하게 감쌀 수 있고, 기존 코드 훼손도 최소화된다.

### 검증

- `bash -n`
  - `scripts/cleanup_sim_processes.sh`
  - `scripts/agribot_launch.sh`
- `cleanup_sim_processes.sh` 단독 실행 정상 확인
- `agribot_launch.sh` 무인자 실행 시 usage 출력 확인
- 문서 내 권장 실행 명령이 새 래퍼 경로로 모두 치환되었는지 확인

### 기대되는 동작 변화

- 사용자가 실행 전마다 긴 `pkill` 블록을 직접 붙여넣을 필요가 줄어든다.
- `Ctrl+C`로 종료해도 launch 자식 프로세스 그룹과 잔여 Gazebo/ROS 프로세스를 자동으로 한 번 더 정리한다.
- `/clock` publisher, bridge, Nav2 lifecycle 노드가 애매하게 남아 다음 세션을 오염시키는 빈도가 줄어든다.

### 남은 한계

- 외부 GUI/시뮬레이터 스택 특성상 `true zombie`를 프로젝트 코드만으로 100% 근본 제거하는 것은 어렵다.
- 이번 변경은 그 한계를 인정하고, 실사용에서 문제를 재현하기 어렵게 만드는 자동 정리 계층을 추가한 것이다.
- 만약 이후에도 특정 프로세스가 반복적으로 남는다면, 그 이름만 cleanup 스크립트 패턴에 추가하면 된다.

### 커밋

- `36f2278` 가제보 종료 잔여 프로세스 정리를 자동화한다

---

[feature/S14P21A602-50] 노드 인터페이스 정의

### 배경

- `수확해조_프로젝트_개발_계획_및_협업_가이드.md`의 `S14P-401`은 관측 결과, 로봇 상태, 수확 기록, 미션 상태, 장치 상태, 장치 실행 명령을 노드 간 계약으로 정리하는 티켓이다.
- 현재 저장소의 `agribot_interfaces`에는 `CropStatus`, `EnvironmentData`, `IoTCommand`, `SetHumidity` 정도만 있었고, 미션/수확/관측용 타입은 비어 있었다.
- 문서상 권장안에는 `PlantObservation`, `RobotStatus`, `HarvestEvent`, `MissionStatus`, `IoTDeviceState`, `RunPatrol.action`, `HarvestTomato.action`, `ExecuteActuation.srv`가 명시돼 있었지만 실제 패키지에는 아직 반영되지 않았다.
- 원격 `ai/` 브랜치도 같이 확인했지만, 관련 코드는 거의 없고 문서상 인터페이스 이름만 제안된 수준이었다. 그래서 이번 MR은 문서 기준 계약을 인터페이스 패키지에 실제 타입으로 옮기는 작업으로 진행했다.

### 이번 MR에서 반영한 내용

- `agribot_ws/src/agribot_interfaces/msg/PlantObservation.msg`
  - 관측 ID, mission ID, zone/plant/fruit ID, class, confidence, health score, pose, image path를 담는 관측 이벤트 메시지 추가
  - `S14P-307`의 위치 추정/사진 저장 결과가 backend와 mission manager로 넘어갈 수 있는 최소 계약을 마련
- `agribot_ws/src/agribot_interfaces/msg/RobotStatus.msg`
  - 현재 모드, 상태, zone, mission, pose, 선속도/각속도, 오류 상태를 담는 로봇 상태 메시지 추가
- `agribot_ws/src/agribot_interfaces/msg/HarvestEvent.msg`
  - 수확 이벤트 ID, plant/fruit ID, success, failure reason, basket count를 담는 수확 기록 메시지 추가
- `agribot_ws/src/agribot_interfaces/msg/MissionStatus.msg`
  - mission id, mission type, state, current phase, progress, retry count, detail message를 담는 미션 상태 메시지 추가
- `agribot_ws/src/agribot_interfaces/msg/IoTDeviceState.msg`
  - 장치 종류, 구역, 현재 상태, opening ratio, speed level, current value, 가용 여부를 담는 장치 상태 메시지 추가
- `agribot_ws/src/agribot_interfaces/action/RunPatrol.action`
  - 장시간 순찰 제어를 위한 goal/result/feedback 골격 추가
- `agribot_ws/src/agribot_interfaces/action/HarvestTomato.action`
  - 접근-정렬-수확-적재 시퀀스를 위한 action 골격 추가
- `agribot_ws/src/agribot_interfaces/srv/ExecuteActuation.srv`
  - 급수/커튼/환기/영양제 실행 요청용 서비스 추가
- `agribot_ws/src/agribot_interfaces/msg/IoTCommand.msg`
  - 기존 단순 필드에서 벗어나 `header`, `command_id`, `zone_id`, `device_type`, `requires_approval`, `auto_execute`, `requested_by`, `reason` 등을 포함하도록 확장
  - 문서의 “추천 후 승인 실행” 흐름과 이후 IoT 브릿지 연결을 고려한 형태로 보강
- `agribot_ws/src/agribot_interfaces/CMakeLists.txt`
  - 새 msg/action/srv를 전부 `rosidl_generate_interfaces()`에 등록
  - `action_msgs` 의존성을 추가
- `agribot_ws/src/agribot_interfaces/package.xml`
  - `action_msgs` 의존성 추가
- `README.md`
  - 인터페이스 패키지 표를 새 계약 기준으로 갱신

### 왜 이렇게 바꿨는가

- `S14P-401`은 이후 `S14P-402`, `S14P-405`, `S14P-410`, `S14P-307` 같은 티켓들의 공통 선행조건이다.
- 지금 단계에서 가장 중요한 것은 런타임 노드를 먼저 대충 만드는 것이 아니라, 팀 전체가 동일한 타입 이름과 필드 계약을 공유하는 것이다.
- 특히 perception, mission manager, IoT, backend가 동시에 연결될 예정이라, 뒤늦게 메시지 이름이 바뀌면 영향 범위가 훨씬 커진다.
- 그래서 이번 MR은 실행 로직보다 먼저 `interface-first`로 기준선을 고정하는 쪽을 선택했다.

### 검증

- `colcon build --packages-select agribot_interfaces --symlink-install` 통과
- `ros2 interface list | rg agribot_interfaces` 확인
  - 새 msg 5종, action 2종, srv 1종이 모두 생성됨
- `ros2 interface show`
  - `agribot_interfaces/msg/PlantObservation`
  - `agribot_interfaces/action/HarvestTomato`
  - `agribot_interfaces/srv/ExecuteActuation`
  - 대표 타입 3종이 기대한 필드 구조로 생성되는 것 확인

### 기대되는 동작 변화

- perception 팀은 `PlantObservation`과 기존 `CropStatus`를 용도에 맞게 나눠 쓸 수 있다.
- mission/control 팀은 `RobotStatus`, `MissionStatus`, `RunPatrol.action`, `HarvestTomato.action`을 바로 기준 타입으로 삼을 수 있다.
- IoT 팀은 `IoTCommand`, `IoTDeviceState`, `ExecuteActuation.srv`를 기준으로 추천/승인/실행 흐름을 연결할 수 있다.
- backend/frontend 팀은 어떤 상태와 이벤트가 ROS에서 올라올지 더 명확하게 맞출 수 있다.

### 남은 한계

- 이번 MR은 인터페이스 정의까지만 다룬다. 실제 publisher/subscriber/server/client 구현은 아직 없다.
- `ai/` 브랜치에는 이 인터페이스를 실제로 소비하는 perception 런타임 코드는 아직 없어서, 다음 단계에서 `detector_node`와 MQTT/mission 연동을 이어야 한다.
- `CropStatus`와 `PlantObservation`의 역할 분리는 문서 기준으로 정리했지만, 실제 운용 시 어떤 노드가 어느 메시지를 주권적으로 발행할지는 다음 티켓에서 확정해야 한다.

### 커밋

- `603f923` 노드 인터페이스 확장으로 미션과 관측 계약을 정리한다

---

### [feature/S14P21A602-51] 미션 상태머신 뼈대 구현

### 배경

- `수확해조_프로젝트_개발_계획_및_협업_가이드.md`의 `S14P-402`는 로봇의 "머리" 역할을 하는 미션 상태머신의 기본 코드를 만드는 티켓이다.
- 현재 저장소의 `agribot_control` 패키지는 사실상 빈 골격 상태였고, `mission_manager.py`, 상태 전이 규칙, 상태 발행 코드가 전혀 없었다.
- 직전 `S14P-401`에서 `MissionStatus`, `RobotStatus`, `RunPatrol.action`, `HarvestTomato.action` 같은 인터페이스는 준비했기 때문에, 이제 최소 동작 가능한 상태머신 뼈대를 붙일 수 있는 조건이 갖춰졌다.
- `ai/` 브랜치도 같이 확인했지만, mission manager 실구현은 없고 문서상 요구와 인터페이스 명칭만 제안된 수준이었다. 따라서 이번 MR은 `agribot_control`에 직접 상태머신 초안을 넣는 방식으로 진행했다.

### 이번 MR에서 반영한 내용

- `agribot_control/agribot_control/mission_manager.py`
  - `MissionStateMachine` 순수 Python 클래스 추가
  - 로봇 모드 enum
    - `IDLE`, `PATROL`, `OBSERVE`, `HARVEST`, `RETURN_HOME`, `IOT_ACTION`, `ERROR`, `STOPPED`
  - 미션 상태 enum
    - `PENDING`, `RUNNING`, `PAUSED`, `COMPLETED`, `FAILED`, `CANCELED`
  - 미션 타입 enum
    - `PATROL`, `OBSERVE`, `HARVEST`, `RETURN_HOME`, `IOT_ACTION`
  - 시작, phase 변경, progress 갱신, pause, resume, complete, cancel, fail, reset 전이를 구현
  - ROS 노드 `MissionManagerNode` 추가
    - `/mission/command` 문자열 명령 구독
    - `/mission/status` 로 `MissionStatus` 발행
    - `/robot/status` 로 `RobotStatus` 발행
    - `/odometry/filtered`를 받아 현재 pose / 속도 반영
- `agribot_control/config/mission_manager.yaml`
  - `robot_id`, `zone_id`, `command_topic`, `mission_status_topic`, `robot_status_topic`, `odometry_topic`, `status_publish_hz` 기본값 정의
- `agribot_control/config/mission_state_diagram.md`
  - Mermaid 기반 상태 다이어그램 추가
  - `IDLE -> PATROL/OBSERVE/HARVEST/RETURN_HOME/IOT_ACTION`, `pause/resume`, `fail/reset` 흐름을 문서화
- `agribot_control/launch/mission_manager.launch.py`
  - `mission_manager` 실행용 launch 추가
  - `use_sim_time`, `params_file` 인자를 받을 수 있게 구성
- `agribot_control/test/test_mission_manager.py`
  - `start_mission`, `pause/resume`, `fail` 전이 단위 테스트 추가
- `agribot_control/package.xml`
  - `agribot_interfaces`, `nav_msgs`, `launch`, `launch_ros` 의존성 추가
- `agribot_control/setup.py`
  - console script entry point `mission_manager` 추가

### 왜 이렇게 바꿨는가

- `S14P-402`는 완성형 미션 매니저가 아니라, 이후 티켓들이 기대는 기준 상태머신 골격을 먼저 세우는 작업이다.
- 그래서 이번 MR은 실제 순찰 액션 서버나 수확 액션 서버를 다 붙이지 않고, 상태 전이와 상태 발행 구조를 먼저 고정하는 데 집중했다.
- 특히 `S14P-403` 순찰 시작/중지/재개, `S14P-405` 로봇·미션 상태 발행, `S14P-404` 감지 우선순위 규칙은 모두 이 상태머신 뼈대를 기반으로 확장하는 것이 자연스럽다.

### 구현 의도

- 문자열 명령 기반 `/mission/command`를 먼저 둔 이유는, 프론트엔드/CLI/API 연동 전에도 토픽 하나로 상태 전이를 빠르게 검증할 수 있게 하기 위해서다.
- `MissionStateMachine`을 ROS 노드와 분리한 이유는, 전이 로직을 unit test로 검증하기 쉽게 만들기 위해서다.
- `RobotStatus`는 현재 pose와 속도를 odometry에서 직접 채우고, `MissionStatus`는 상태머신 snapshot을 그대로 발행하도록 나눠서 역할을 분리했다.

### 검증

- `python3 -m py_compile`
  - `agribot_control/agribot_control/mission_manager.py`
- `pytest`
  - `agribot_control/test/test_mission_manager.py`
  - 총 `3 passed`
- `colcon build --packages-select agribot_control --symlink-install` 통과
- `ros2 run agribot_control mission_manager --ros-args -p use_sim_time:=false`
  - 노드 기동 로그 확인
  - `Mission manager skeleton ready...` 출력 확인

### 기대되는 동작 변화

- 이제 `agribot_control`에 미션 상태 전이의 기준점이 생겼다.
- 운영자는 `/mission/command`에 문자열 명령을 넣어 상태 변화를 바로 확인할 수 있다.
- backend/frontend는 `MissionStatus`, `RobotStatus`를 받을 실제 발행 지점을 갖게 된다.
- 다음 단계에서 순찰 액션, 수확 액션, IoT 실행 연결을 상태머신 위에 덧붙일 수 있다.

### 남은 한계

- 이번 MR은 “뼈대” 수준이라 실제 `RunPatrol.action` 서버나 `HarvestTomato.action` 서버와는 아직 연결되지 않았다.
- `S14P-403`에서 start/stop/resume 제어 명령을 더 정교하게 바꾸고, `S14P-404`에서 관측 이벤트 우선순위 규칙을 붙여야 한다.
- 현재 `/mission/command`는 `std_msgs/String` 기반이므로, 장기적으로는 명시적 command msg 또는 action/service로 승격하는 것이 더 좋다.

### 커밋

- `14a1878` 미션 상태머신 뼈대를 추가하고 상태 발행을 연결한다

---

### [feature/S14P21A602-52] 순찰 시작·중지·재개 제어 구현

### 배경

- `S14P-403`의 목표는 프론트엔드나 CLI가 순찰 시작, 중지, 재개 명령을 보냈을 때 실제 로봇 순찰 주행에 반영되게 만드는 것이다.
- 이미 `agribot_navigation/patrol_node.py`에는 `/patrol/start`, `/patrol/stop`, `/patrol/resume` 서비스가 있었고, 직전 티켓 `S14P-402`에서 만든 `mission_manager.py`는 상태만 바꾸는 뼈대 수준이었다.
- 따라서 이번 작업은 새 순찰 노드를 만드는 것이 아니라, `mission_manager`를 실제 순찰 제어면과 연결하는 쪽으로 잡는 것이 맞았다.
- `ai/` 브랜치도 확인했지만, 관련 구현은 문서/계획 수준이었고 실제 제어 코드는 없어서 현재 브랜치 기준으로 직접 연결했다.

### 이번 MR에서 반영한 내용

- `agribot_control/agribot_control/mission_manager.py`
  - `/patrol/start`, `/patrol/stop`, `/patrol/resume`용 `Trigger` service client 추가
  - `/patrol/status` 구독 추가
  - `/mission/command`에서 아래 명령을 실제 순찰 제어로 연결
    - `patrol_start`, `start_patrol`
    - `patrol_stop`, `stop_patrol`
    - `patrol_resume`, `resume_patrol`
    - `pause`, `resume`
      - 현재 미션이 `PATROL`이면 순찰 서비스 호출로 우회
      - 그 외 미션이면 기존 상태머신 동작 유지
  - 순찰 상태 JSON 파서 `parse_patrol_status()` 추가
  - 순찰 상태를 미션 상태머신에 반영하는 `apply_patrol_status_snapshot()` helper 추가
    - `running/starting/observing` -> `PATROL` 상태 유지 및 진행률 반영
    - `stopped` -> `PAUSED`
    - `completed` -> `COMPLETED`
    - `error` -> `FAILED`
  - 서비스 응답 성공/실패에 따라 `MissionStatus` detail message가 같이 갱신되도록 처리
- `agribot_control/config/mission_manager.yaml`
  - `patrol_status_topic`
  - `patrol_start_service`
  - `patrol_stop_service`
  - `patrol_resume_service`
  - `patrol_service_wait_sec`
  기본 파라미터 추가
- `agribot_control/config/mission_state_diagram.md`
  - `S14P-403` 이후에는 `/mission/command`가 실제 patrol service 호출로 이어진다는 점 반영
  - `patrol/status`를 통해 상태/진행률을 동기화한다는 설명 추가
- `agribot_control/package.xml`
  - `std_srvs` 실행 의존성 추가
- `agribot_control/test/test_mission_manager.py`
  - patrol status parsing test 추가
  - stopped -> running 재개 동기화 test 추가
  - patrol completed 동기화 test 추가

### 왜 이렇게 바꿨는가

- 기존 `mission_manager`는 명령을 받아도 내부 state만 바꿨기 때문에, 문서의 “실제로 반영되도록 연결” 요구를 만족하지 못했다.
- 반면 `patrol_node`는 이미 제어 서비스를 갖고 있었으므로, 그 서비스들을 재사용하는 것이 가장 안전하고 변경 범위도 작았다.
- 이 접근은 기존 순찰 구현을 훼손하지 않으면서도, 프론트엔드/CLI/백엔드가 `mission_manager` 하나를 통해 순찰 제어를 걸 수 있게 만든다.

### 구현 의도

- `mission_manager`는 상태의 기준점이면서 동시에 제어 명령 핸들러 역할을 맡는다.
- `patrol_node`는 여전히 실제 waypoint 순찰 실행을 담당하고, `mission_manager`는 제어 명령과 상태 발행을 담당한다.
- 즉 “실행 엔진”과 “오케스트레이션/상태 관리”를 분리해, 이후 수확/관찰/IoT 제어를 같은 패턴으로 확장할 수 있게 했다.

### 검증

- `python3 -m py_compile`
  - `agribot_control/agribot_control/mission_manager.py`
- `pytest`
  - `agribot_control/test/test_mission_manager.py`
  - 총 `6 passed`
- `colcon build --packages-select agribot_control --symlink-install` 통과
- `ros2 run agribot_control mission_manager --ros-args -p use_sim_time:=false`
  - 노드 기동 확인
  - `patrol_status_topic=/patrol/status` 포함 startup log 확인

### 기대되는 동작 변화

- 이제 `/mission/command`만으로 순찰 시작, 중지, 재개 명령을 실제 patrol service 호출에 연결할 수 있다.
- `MissionStatus`와 `RobotStatus`는 순찰 노드의 진행 상태를 따라가면서 더 현실적인 상태를 발행한다.
- 순찰을 중지하면 `PAUSED`, 다시 재개하면 `RUNNING/PATROL`로 자연스럽게 복귀한다.
- 순찰 완료나 오류도 patrol status를 통해 mission manager가 자동 반영한다.

### 남은 한계

- 이번 MR은 `Trigger` 서비스 기반 연결이다. `RunPatrol.action`을 직접 쓰는 장시간 제어 구조로 승격하려면 다음 단계가 필요하다.
- 아직 frontend/backend API와 직접 연결한 것은 아니고, `mission_manager`의 `/mission/command`까지를 기준점으로 세웠다.
- 수확/관찰 중 순찰을 선점하거나 복귀 후 순찰을 이어 붙이는 규칙은 `S14P-404`, 후속 harvest orchestration 티켓에서 정교화해야 한다.

### 커밋

- `250c088` 순찰 시작 중지 재개 제어를 미션 매니저에 연결한다

### [feature/S14P21A602-53] 감지 이벤트 우선순위 규칙 구현

### 배경

- `S14P-404`의 목표는 병든 잎과 익은 토마토가 거의 동시에 관측될 때, 무엇을 먼저 처리할지 명확한 규칙을 만들고 코드로 옮기는 것이다.
- 문서 요구사항에는 “같은 대상을 여러 번 본 경우 중복 처리하지 않게 규칙 만들기”도 포함되어 있다.
- `ai/` 브랜치를 다시 확인했지만, 관련 내용은 perception dataset/codebook과 문서 수준이었고, 미션 레벨 우선순위 arbiter 구현은 없었다.
- 따라서 이번 작업은 기존 `mission_manager` 위에 `PlantObservation` 기반 우선순위/중복 제거 계층을 추가하는 방식으로 진행했다.

### 이번 MR에서 반영한 내용

- `agribot_control/agribot_control/observation_priority.py`
  - 순수 Python helper `ObservationPriorityArbiter` 추가
  - `ObservationInput`, `ObservationTaskCandidate`, `ObservationSelection` dataclass 추가
  - 규칙 구현
    - `diseased_leaf` > `ripe_tomato` > `generic_observation`
    - 같은 priority면 confidence가 높은 후보 우선
    - 둘 다 같으면 먼저 들어온 후보 유지
  - dedup key 규칙
    - `zone_id + plant_id + fruit_id + event_kind`
  - duplicate window 규칙
    - 기본 `120초`
    - active target duplicate 무시
    - cooldown 중 duplicate 무시
    - pending duplicate는 더 높은 priority/confidence일 때만 갱신
- `agribot_control/agribot_control/mission_manager.py`
  - `PlantObservation` 구독 추가
  - observation candidate를 pending/active로 관리
  - patrol 중 higher-priority observation이 들어오면 patrol stop 요청
  - stop 성공 직후 선택된 observation을 `OBSERVE` 또는 `HARVEST` 미션으로 승격
  - active observation 미션 완료/취소 후 다음 pending candidate 자동 승격
  - reset 시 arbiter/pending 상태도 같이 초기화
- `agribot_control/config/mission_manager.yaml`
  - `plant_observation_topic`
  - `observation_duplicate_window_sec`
  - `max_pending_observations`
  - `interrupt_patrol_on_observation`
  기본값 추가
- `agribot_control/config/observation_priority_rules.md`
  - 우선순위 표
  - dedup 규칙
  - tie-break 규칙
  - 현재 구현 범위 문서화
- `agribot_control/config/mission_state_diagram.md`
  - `PlantObservation`이 들어와 우선순위 규칙으로 정렬된다는 메모 추가
- `agribot_control/test/test_observation_priority.py`
  - 병든 잎이 익은 토마토보다 우선되는지 검증
  - cooldown duplicate가 무시되는지 검증
  - active 완료 후 다음 pending candidate가 승격되는지 검증

### 왜 이렇게 바꿨는가

- perception 노드가 아무리 관측을 잘 보내더라도, 미션 레벨에서 “무엇을 먼저 처리할지”를 정해 주지 않으면 이벤트가 쌓일수록 상태가 꼬인다.
- 이 규칙은 ROS callback 안에서 ad-hoc으로 처리하기보다, 테스트 가능한 pure-Python helper로 분리하는 편이 안전하다.
- `mission_manager`는 이미 `S14P-402`, `S14P-403`에서 상태 전이와 patrol 제어의 기준점이 되었기 때문에, 감지 우선순위도 그 위에 얹는 것이 가장 자연스럽다.

### 구현 의도

- 이번 MR은 “실제 수확 동작”이나 “질병 후속 조치”를 완성하는 티켓이 아니다.
- 대신 다음 단계들이 기대는 공통 규칙층을 먼저 만든다.
  - 동일 대상 중복 이벤트 무시
  - 병든 잎 우선
  - active/pending 이벤트 분리
  - patrol 중 들어온 이벤트를 미션 레벨에서 pause-and-takeover 할 수 있는 기반
- 즉 앞으로 `S14P-405`, harvest orchestration, disease alert 연결이 이 규칙을 그대로 재사용할 수 있게 만드는 것이 핵심이다.

### 검증

- `python3 -m py_compile`
  - `agribot_control/agribot_control/mission_manager.py`
  - `agribot_control/agribot_control/observation_priority.py`
- `pytest`
  - `agribot_control/test/test_mission_manager.py`
  - `agribot_control/test/test_observation_priority.py`
  - 총 `9 passed`
- `colcon build --packages-select agribot_control --symlink-install` 통과
- `ros2 run agribot_control mission_manager --ros-args -p use_sim_time:=false`
  - startup log에 `plant_observation_topic=/plant_observation` 확인

### 기대되는 동작 변화

- perception 쪽에서 `PlantObservation`이 들어오면, mission manager가 우선순위와 dedup 규칙을 적용한 뒤 다음 처리 대상을 고른다.
- 같은 병든 잎/같은 익은 토마토를 짧은 시간 내 여러 번 봐도 중복 이벤트로 미션이 계속 새로 열리지 않는다.
- patrol 중 higher-priority event가 들어오면 patrol을 멈추고 해당 event 미션으로 넘길 수 있는 기반이 생긴다.
- active event가 끝나면 다음 pending event를 자연스럽게 이어받을 수 있다.

### 남은 한계

- 실제 `OBSERVE`, `HARVEST` 액션 서버와 연결된 것은 아니므로, 지금은 미션 상태 전환과 우선순위 결정까지가 구현 범위다.
- disease alert를 backend/MQTT로 보내는 후속 연동은 별도 티켓에서 붙여야 한다.
- 현재 priority 분류는 `class_name` / `ready_to_harvest` 규칙을 사용하므로, AI 팀의 실제 추론 라벨 명명 규칙이 바뀌면 매핑도 같이 조정해야 한다.

### 커밋

- `dbb645c` 감지 이벤트 우선순위 규칙을 미션 매니저에 반영한다

## [feature/S14P21A602-54] 로봇·미션 상태 발행 구현

### 배경

- `S14P-405` 문서 요구는 프론트엔드와 백엔드가 로봇의 현재 상태를 그대로 받을 수 있게 `RobotStatus`, `MissionStatus`를 계속 발행하는 것이다.
- 기존 `mission_manager`도 두 메시지를 퍼블리시하고 있었지만, 값이 상태머신 내부의 raw phase를 거의 그대로 내보내는 수준이라 실제 화면 표시나 서버 소비용 정보로는 부족했다.
- 특히 순찰 waypoint 진행, 관측 대기열, 복귀 여부, 오류 코드 같은 운영 정보가 한눈에 드러나지 않았다.

### 무엇을 바꿨는가

- `agribot_ws/src/agribot_control/agribot_control/mission_manager.py`
  - `StatusTelemetry` dataclass를 추가했다.
  - `build_status_telemetry()` helper를 도입해 상태머신 snapshot, patrol status, active observation, pending observation 개수를 한 번에 조합하도록 정리했다.
  - `MissionStatus.current_phase`를 raw mode 대신 운영 의미가 드러나는 phase로 확장했다.
    - `PATROL_NAVIGATING`
    - `PATROL_OBSERVING`
    - `PATROL_PAUSED`
    - `PATROL_COMPLETED`
    - `PATROL_ERROR`
    - `RETURN_HOME_NAVIGATING`
    - `OBSERVE_ACTIVE`
    - `HARVEST_ACTIVE`
    - `IOT_ACTION_ACTIVE`
  - `target_id`는 mission manager 내부 target보다 실제 patrol의 `next_waypoint_id`와 active observation target을 우선 반영하도록 바꿨다.
  - `detail_message`는 단순 한 줄 텍스트 대신 아래 운영 정보를 합성하도록 바꿨다.
    - patrol status message
    - `current=...`, `next=...`
    - `active_observation=...`
    - `pending_observations=...`
    - `waiting_for_patrol_stop=true`
  - `RobotStatus`에도 동일 telemetry를 사용해 아래 값이 실제 미션 문맥을 따라가게 정리했다.
    - `mode`
    - `state`
    - `is_returning_home`
    - `has_error`
    - `error_code`
    - `error_message`
  - patrol status 콜백에서 가장 최근 snapshot을 보관해 상태 발행 주기와 patrol 상태 수신 시점이 어긋나도 화면에 최신 진행 상황이 반영되도록 했다.
- `agribot_ws/src/agribot_control/test/test_mission_manager.py`
  - patrol running 상태에서 phase/target/progress/detail이 기대대로 확장되는지 검증을 추가했다.
  - return-home 실패 시 `RETURN_HOME_ERROR`로 매핑되는지 검증을 추가했다.
  - active observation이 상태 발행 target/detail에 반영되는지 검증을 추가했다.

### 왜 이렇게 바꿨는가

- `S14P-402`, `S14P-403`, `S14P-404`까지 구현되면서 mission manager는 이미 상태 전이와 순찰 제어, 감지 우선순위의 중심축이 됐다.
- 그런데 상태 발행이 raw snapshot 수준에 머물면 프론트엔드와 백엔드가 다시 해석 로직을 각자 가져야 한다.
- 이 MR은 그 해석 책임을 mission manager 안으로 당겨서, ROS 밖 소비자들이 “현재 무엇을 하고 있는지”를 바로 사용할 수 있게 만드는 작업이다.
- `ai/` 브랜치도 다시 확인했지만, 관련 코드는 문서/모델 자산 수준이고 실제 상태 발행 구현은 없어서 현재 브랜치에서 직접 정리하는 편이 맞았다.

### 검증

- `python3 -m py_compile`
  - `agribot_control/agribot_control/mission_manager.py`
  - `agribot_control/test/test_mission_manager.py`
- `pytest -q`
  - `agribot_control/test/test_mission_manager.py`
  - `agribot_control/test/test_observation_priority.py`
  - 총 `12 passed`
- `colcon build --packages-select agribot_control --symlink-install` 통과
- `timeout 3 ros2 run agribot_control mission_manager --ros-args -p use_sim_time:=false`
  - startup log 정상 확인

### 기대되는 동작 변화

- backend/frontend는 `/mission/status`, `/robot/status`만 구독해도 현재 미션 phase와 목표 지점, 복귀 여부, 오류 여부를 즉시 표시할 수 있다.
- patrol 중에는 현재 waypoint와 다음 waypoint가 detail에 노출돼 진행 상황을 사람이 추적하기 쉬워진다.
- observation 우선순위 큐가 쌓여 있을 때도 현재 active target과 pending 개수가 상태 메시지에 함께 반영된다.
- 복귀 실패, 순찰 실패 등은 더 이상 단순 `MISSION_ERROR` 하나로만 보이지 않고 미션 종류에 맞는 오류 코드로 구분된다.

### 남은 한계

- 아직 backend websocket/MQTT 브리지까지 직접 연결한 것은 아니므로, 이번 MR 범위는 ROS 토픽 발행 정리까지다.
- status detail은 문자열 조합 방식이라, 차후 프론트 요구가 더 구체화되면 세분화된 필드로 분리할 여지가 있다.
- 실제 `HARVEST`, `RETURN_HOME`, `IOT_ACTION` 서버가 붙으면 phase 세분화 규칙은 한 번 더 정교화할 수 있다.

### 커밋

- `736ebc7` 로봇과 미션 상태 발행 정보를 실시간 표시용으로 정리한다

## [feature/S14P21A602-55] 환경·질병 규칙표 구현

### 배경

- `S14P-406`의 목표는 환경 센서값과 병해 관측 결과를 보고 어떤 장치를 움직일지 팀 전체가 같은 기준으로 판단할 수 있게 만드는 것이다.
- 문서 요구만 보면 규칙표 정리 티켓이지만, 바로 다음 `S14P-407`, `S14P-408`이 이 기준을 그대로 가져가게 될 가능성이 높다.
- 그래서 이번 작업은 “문서만 있는 규칙표” 대신, 사람이 읽는 규칙표와 코드에서 재사용 가능한 rulebook을 같이 만들었다.

### 현재 코드베이스 확인 결과

- backend에는 `/api/v1/actuations/watering`, `/curtain`, `/fan`, `/nutrients` 라우터 골격이 이미 있다.
- `agribot_interfaces/msg/EnvironmentData.msg`에는 온도, 습도, 토양 수분, 조도, CO2 필드가 이미 정의되어 있다.
- `ai/` 브랜치에는 병해 코드북과 라벨 자산은 있지만, 환경·질병 조합으로 장치 추천을 내리는 규칙 코드는 없었다.
- 즉 이번 티켓은 현재 브랜치에서 기준선을 직접 만드는 편이 맞았다.

### 무엇을 구현했는가

- `agribot_ws/src/agribot_control/agribot_control/environment_disease_rules.py`
  - `EnvironmentSnapshot`, `DiseaseSignal`, `RuleDecision` dataclass를 추가했다.
  - `evaluate_environment_disease_rules()` pure-Python helper를 만들었다.
  - 출력은 다음 장치 타입 기준으로 정리된다.
    - `watering`
    - `curtain`
    - `fan`
    - `nutrient`
    - `hold_watering`
  - 주요 규칙은 아래와 같다.
    - `soil_moisture < 22` 이면 급수 `900 ml` 자동 실행 후보
    - `22 <= soil_moisture < 30` 이면 급수 `600 ml` 승인 후 실행 후보
    - `temperature >= 32` and `light_level >= 35000` 이면 커튼 `60%` 닫힘 자동 실행 후보
    - `temperature >= 29` and `light_level >= 25000` 이면 커튼 `35%` 닫힘 자동 실행 후보
    - `humidity >= 75` and `temperature >= 30` 이면 환기팬 `level 2`
    - `humidity >= 80` 이고 곰팡이 계열 질병이 `repeat_count >= 2` 이면 급수 보류 + 환기팬 `level 3`
    - 착화/과실기 + `blossom_end_rot` / `calcium_deficiency` 계열이면 칼슘 보강제 `250 ml` 승인 후 실행 후보
  - 결과는 priority 기준으로 정렬되도록 했다.
- `agribot_ws/src/agribot_control/config/environment_disease_rules.md`
  - 규칙표 문서
  - 곰팡이 계열 라벨 기준
  - 영양제 추천 보수 정책
  - 예시 입력/결과 표를 정리했다.
- `agribot_ws/src/agribot_control/test/test_environment_disease_rules.py`
  - 매우 건조한 토양에서 자동 급수 후보가 나오는지 검증
  - 고온/고조도에서 커튼 추천이 나오는지 검증
  - 고습도 곰팡이 반복 상황에서 급수 보류 + 환기팬 강풍이 우선되는지 검증
  - 착화/과실기 칼슘 결핍 의심 상황에서 영양제 승인 후 실행 후보가 나오는지 검증

### 왜 이렇게 구현했는가

- `S14P-406`은 다음 티켓들의 기준표 역할을 하므로, 수치와 해석 기준이 테스트 가능한 형태로 남아 있어야 한다.
- backend나 IoT 실행 노드에 바로 결합하면 이후 수치 조정이 어려워진다.
- 그래서 이번 MR은 “판단 기준만” 순수 함수로 분리해 두고, 실제 명령 발행은 후속 티켓에서 연결하기 쉽게 만들었다.
- 또한 AI 브랜치의 병해 라벨 자산을 고려해 `tomato_powdery_mildew`, `tomato_gray_mold`, `blossom_end_rot` 같은 라벨을 규칙에 반영했다.

### 검증

- `python3 -m py_compile`
  - `agribot_control/agribot_control/environment_disease_rules.py`
  - `agribot_control/test/test_environment_disease_rules.py`
- `pytest -q`
  - `agribot_control/test/test_environment_disease_rules.py`
  - `agribot_control/test/test_mission_manager.py`
  - `agribot_control/test/test_observation_priority.py`
  - 총 `13 passed`
- `colcon build --packages-select agribot_control --symlink-install` 통과

### 기대되는 다음 단계

- `S14P-407`은 이번 rulebook에서 `watering` 관련 decision만 연결하면 된다.
- `S14P-408`은 `curtain`, `fan` decision을 그대로 가져가면 된다.
- `S14P-410`에서는 `RuleDecision`을 `IoTCommand`나 backend actuation request로 매핑하는 층만 추가하면 된다.

### 한계

- 지금은 규칙 추천과 보류 결정까지만 구현했고, 실제 장치 명령 발행은 하지 않는다.
- 영양제 규칙은 칼슘 보강 baseline 한 종류만 포함한다.
- 환경값 임계치는 프로젝트 baseline이므로, 실제 하드웨어 실험 후 조정이 필요할 수 있다.

### 커밋

- `7a03b04` 환경 질병 규칙표를 코드와 문서 기준으로 정리한다

## [feature/S14P21A602-56] 급수 실행 판단 로직 구현

### 배경

- `S14P-407`의 목표는 급수 조건을 코드로 옮기고, 결과를 `자동 실행`, `승인 필요`, `실행 보류`로 나누는 것이다.
- 직전 티켓 `S14P-406`에서 환경·질병 rulebook을 만들었기 때문에, 이번 티켓은 그중 `watering` 관련 규칙만 재사용하는 전용 판단층으로 구현하는 것이 가장 자연스럽다.
- 또한 결과물 요구에 `판단 결과 로그`가 포함되어 있어, helper만 만드는 대신 저부하 ROS 노드까지 함께 추가했다.

### 무엇을 구현했는가

- `agribot_ws/src/agribot_control/agribot_control/watering_decision.py`
  - `WateringDecisionState`
    - `AUTO_EXECUTE`
    - `REQUIRES_APPROVAL`
    - `HOLD`
    - `NO_ACTION`
  - `WateringDecision` dataclass
  - `evaluate_watering_decision()` helper
    - `evaluate_environment_disease_rules()` 결과에서 `watering` 관련 후보만 추려서 최종 급수 판단으로 축약한다.
    - 보류 규칙이 있으면 급수 추천보다 항상 우선한다.
    - 추천 규칙은 `auto_execute` 여부에 따라 자동 실행과 승인 필요로 구분한다.
    - 급수 후보가 없으면 `NO_ACTION`으로 명시한다.
  - `format_watering_decision_log()` helper
    - zone, 상태, command, target, soil_moisture, humidity, disease context, rule, reason을 한 줄 로그로 정리한다.
- `agribot_ws/src/agribot_control/agribot_control/watering_decision_node.py`
  - `/environment_data`, `/plant_observation` 구독
  - zone별 최신 병해 관측과 반복 횟수 저장
  - 환경값이 들어올 때만 급수 판단 수행
  - 결정이 바뀔 때만 로그 출력해서 불필요한 로그 폭증과 CPU 낭비를 막음
  - `HOLD`는 warning, 나머지는 info로 출력
- `agribot_ws/src/agribot_control/config/watering_decision.yaml`
  - topic, zone filter, log-on-change 파라미터 추가
- `agribot_ws/src/agribot_control/launch/watering_decision.launch.py`
  - 단독 기동용 launch 추가
- `agribot_ws/src/agribot_control/test/test_watering_decision.py`
  - 매우 건조한 토양 -> 자동 실행 검증
  - 중간 수준 건조 -> 승인 필요 검증
  - 고습도 곰팡이 반복 -> 급수 보류 검증
  - 충분한 토양 수분 -> 무동작 검증
  - 로그 문자열에 이유와 질병 context가 들어가는지 검증
- `agribot_ws/src/agribot_control/setup.py`
  - `watering_decision_node` console script 추가

### 왜 이렇게 구현했는가

- 이번 티켓은 실제 IoT 명령 발행이 아니라 판단 규칙 구현과 로그 검증이 범위이므로, 실행 요청은 후속 티켓에 남겨두고 판단 책임만 분리했다.
- 환경값이 들어올 때만 평가하고, 변경 시점에만 로그를 남기게 해서 노트북 연산량과 발열 증가를 최소화했다.
- `ai/` 브랜치는 병해 코드북과 데이터셋 도구만 있고, 급수 실행 판단 로직은 없어서 현재 브랜치에서 직접 구현하는 편이 맞았다.

### 판단 기준

- `soil_moisture < 22` -> `AUTO_EXECUTE` / `900 ml`
- `22 <= soil_moisture < 30` -> `REQUIRES_APPROVAL` / `600 ml`
- `humidity >= 80` and 곰팡이 계열 질병 반복 `repeat_count >= 2` -> `HOLD`
- 그 외 -> `NO_ACTION`

### 검증

- `python3 -m py_compile`
  - `agribot_control/agribot_control/watering_decision.py`
  - `agribot_control/agribot_control/watering_decision_node.py`
  - `agribot_control/test/test_watering_decision.py`
- `pytest -q`
  - `agribot_control/test/test_watering_decision.py`
  - `agribot_control/test/test_environment_disease_rules.py`
  - `agribot_control/test/test_mission_manager.py`
  - `agribot_control/test/test_observation_priority.py`
  - 총 `18 passed`
- `colcon build --packages-select agribot_control --symlink-install` 통과
- `timeout 3 ros2 run agribot_control watering_decision_node --ros-args -p use_sim_time:=false`
  - startup log 정상 확인

### 기대되는 동작 변화

- 토양 수분이 낮으면 급수 후보가 안정적으로 `자동 실행` 또는 `승인 필요`로 구분된다.
- 고습도 곰팡이 반복 상황에서는 기존 급수 후보보다 `보류`가 우선된다.
- 운영 중에는 환경 토픽만 들어와도 현재 급수 판단 상태가 로그로 남아 디버깅하기 쉬워진다.

### 한계

- 실제 `IoTCommand` 발행이나 backend actuation API 호출은 아직 하지 않는다.
- 질병 반복 판단은 현재 zone별 최근 class 이름 반복 횟수 기반이라, 더 정교한 plant/fruit 단위 누적은 후속 티켓에서 보강할 수 있다.
- 급수량은 baseline 수치라 하드웨어 실측 후 조정이 필요할 수 있다.

### 커밋

- `2cd5613` 급수 실행 판단 로직을 자동 승인 보류 단계로 구현한다

## [feature/S14P21A602-57] 커튼·환기 실행 판단 로직 구현

### 배경

- `S14P-408`의 목표는 온도/조도 조건에 따른 커튼 제어와, 습도/병해 조건에 따른 환기팬 제어를 코드로 만드는 것이다.
- `S14P-406`에서 이미 환경·질병 rulebook을 만들었고, `S14P-407`에서 급수 전용 판단층을 분리했기 때문에 이번 티켓도 같은 구조로 묶는 것이 자연스럽다.
- 결과물에 `판단 결과 로그`가 포함되어 있으므로, helper뿐 아니라 저부하 ROS 노드까지 함께 구현했다.

### 무엇을 구현했는가

- `agribot_ws/src/agribot_control/agribot_control/climate_decision.py`
  - `DeviceDecisionState`
    - `AUTO_EXECUTE`
    - `REQUIRES_APPROVAL`
    - `NO_ACTION`
  - `DeviceDecision` dataclass
  - `evaluate_curtain_decision()`
    - `temperature >= 32` and `light_level >= 35000` 이면 커튼 `60%` 닫힘
    - `temperature >= 29` and `light_level >= 25000` 이면 커튼 `35%` 닫힘
    - 그 외는 `NO_ACTION`
  - `evaluate_fan_decision()`
    - `humidity >= 75` and `temperature >= 30` 이면 환기팬 `level 2`
    - `humidity >= 80` 이고 곰팡이 계열 질병 반복 `repeat_count >= 2` 이면 환기팬 `level 3`
    - 곰팡이 반복 조건은 일반 고온다습 조건보다 우선한다.
    - 그 외는 `NO_ACTION`
  - `format_device_decision_log()` helper
    - zone, state, command, target, temperature, humidity, light, disease, rule, reason을 한 줄 로그로 정리한다.
- `agribot_ws/src/agribot_control/agribot_control/climate_decision_node.py`
  - `/environment_data`, `/plant_observation` 구독
  - zone별 최신 병해 class와 반복 횟수 저장
  - 환경값 수신 시 커튼/환기 두 장치 판단을 같이 수행
  - `log_only_on_change=true` 기본값으로 판단이 바뀔 때만 로그를 출력해 부하를 줄임
- `agribot_ws/src/agribot_control/config/climate_decision.yaml`
  - topic, zone filter, log-on-change 파라미터 추가
- `agribot_ws/src/agribot_control/launch/climate_decision.launch.py`
  - 단독 기동용 launch 추가
- `agribot_ws/src/agribot_control/test/test_climate_decision.py`
  - 강한 고온/고조도 -> 커튼 60% 닫힘 검증
  - 중간 고온/고조도 -> 커튼 35% 닫힘 검증
  - 정상 환경 -> 커튼 무동작 검증
  - 고온다습 -> 환기팬 level 2 검증
  - 고습도 곰팡이 반복 -> 환기팬 level 3 검증
  - 정상 환경 -> 환기팬 무동작 검증
  - 로그 문자열에 rule과 disease context 포함 검증
- `agribot_ws/src/agribot_control/setup.py`
  - `climate_decision_node` console script 추가

### 왜 이렇게 구현했는가

- 커튼과 환기는 입력 환경값과 병해 문맥을 공유하므로 한 노드에서 같이 판단하는 편이 중복 구독과 중복 상태관리를 줄일 수 있다.
- 실제 장치 제어는 후속 티켓에서 연결할 예정이므로, 이번 티켓은 판단 책임과 로그만 확실히 분리했다.
- `ai/` 브랜치에는 병해 코드북과 환경 필드 정의는 있지만, 커튼/환기 실행 판단 코드는 없어서 현재 브랜치에서 직접 구현하는 편이 맞았다.

### 판단 기준

- 커튼
  - `temperature >= 32` and `light_level >= 35000` -> `AUTO_EXECUTE` / `60%`
  - `temperature >= 29` and `light_level >= 25000` -> `AUTO_EXECUTE` / `35%`
  - 그 외 -> `NO_ACTION`
- 환기팬
  - `humidity >= 75` and `temperature >= 30` -> `AUTO_EXECUTE` / `level 2`
  - `humidity >= 80` and 곰팡이 계열 질병 반복 `repeat_count >= 2` -> `AUTO_EXECUTE` / `level 3`
  - 그 외 -> `NO_ACTION`

### 검증

- `python3 -m py_compile`
  - `agribot_control/agribot_control/climate_decision.py`
  - `agribot_control/agribot_control/climate_decision_node.py`
  - `agribot_control/test/test_climate_decision.py`
- `pytest -q`
  - `agribot_control/test/test_climate_decision.py`
  - `agribot_control/test/test_watering_decision.py`
  - `agribot_control/test/test_environment_disease_rules.py`
  - `agribot_control/test/test_mission_manager.py`
  - `agribot_control/test/test_observation_priority.py`
  - 총 `25 passed`
- `colcon build --packages-select agribot_control --symlink-install` 통과
- `timeout 3 ros2 run agribot_control climate_decision_node --ros-args -p use_sim_time:=false`
  - startup log 정상 확인

### 기대되는 동작 변화

- 온도와 조도만 바뀌어도 커튼 제어 후보가 안정적으로 선택된다.
- 습도와 곰팡이 반복 관측이 들어오면 일반 환기보다 강한 환기팬 단계가 선택된다.
- 운영 중에는 환경 토픽 변화만으로 커튼/환기 판단 로그를 바로 확인할 수 있어 디버깅이 쉬워진다.

### 한계

- 실제 `IoTCommand` 발행이나 backend actuation API 호출은 아직 하지 않는다.
- 환기팬은 현재 승인 단계 없이 자동 실행 후보만 다루고 있다.
- 병해 반복은 zone 기준 최근 class 반복 횟수만 보므로, 더 정교한 plant 단위 누적은 후속 티켓에서 보강할 수 있다.

### 커밋

- `e9b1373` 커튼과 환기 실행 판단 로직을 환경 규칙으로 구현한다

## [feature/S14P21A602-58] 수동 제어 우선·안전 잠금 구현

### 배경

- `S14P-409`의 목표는 자동 제어와 수동 제어가 같은 장치에 동시에 들어올 때, 명령이 꼬이지 않게 중간에서 정리하는 것이다.
- 문서 요구사항은 네 가지였다.
  - 수동 명령 우선
  - 중복 명령 방지
  - 짧은 시간 내 반복 실행 방지
  - 동시에 충돌하는 장치 실행 차단
- 현재 `agribot_control`에는 급수 판단과 커튼/환기 판단은 있지만, 그 결과를 실제 장치 실행 직전에서 중재하는 계층은 없었다.
- 원격 `ai/` 브랜치도 확인했는데 `origin/ai/illness_model_data/S14P21A602-28-aihub153`, `origin/ai/model_function_check/S14P21A602-13` 수준이며, 이번 작업과 직접 겹치는 수동 제어 arbitration 코드는 없었다.

### 무엇을 구현했는가

- `agribot_ws/src/agribot_control/agribot_control/manual_actuation_safety.py`
  - `CommandSource`
    - `AUTO`
    - `MANUAL`
  - `ActuationCommand` dataclass
  - `ArbitrationResult` dataclass
  - `ManualPrioritySafetyLock`
    - 같은 장치 lock key 계산
    - 충돌 그룹 key 계산
    - duplicate signature 추적
    - manual override window, device execution lock, conflict group lock 관리
- `agribot_ws/src/agribot_control/agribot_control/manual_actuation_guard_node.py`
  - `/iot/commands/manual`, `/iot/commands/auto` 구독
  - 통과한 명령만 `/iot/commands/dispatch`로 전달
  - 거부된 명령은 rule code와 사유를 로그로 남김
- `agribot_ws/src/agribot_control/config/manual_actuation_guard.yaml`
  - topic 경로
  - `manual_override_seconds`
  - `duplicate_window_seconds`
  - `execution_lock_seconds`
  - `conflict_groups`
- `agribot_ws/src/agribot_control/launch/manual_actuation_guard.launch.py`
  - 단독 기동용 launch 추가
- `agribot_ws/src/agribot_control/test/test_manual_actuation_safety.py`
  - 수동 우선
  - duplicate 거부
  - 같은 장치 짧은 실행 잠금
  - 충돌 장치 그룹 잠금
  - 잠금 만료 후 자동 명령 재허용
- `agribot_ws/src/agribot_control/setup.py`
  - `manual_actuation_guard_node` console script 등록

### 왜 이렇게 구현했는가

- `S14P-409`는 실제 장치 제어보다 앞단의 arbitration 책임이 핵심이라, ROS node 안에 조건문을 흩뿌리기보다 순수 helper로 분리하는 편이 테스트와 재사용에 유리하다.
- 지금 단계에서는 backend나 MQTT payload 세부가 완전히 고정되지 않았으므로, `IoTCommand`를 그대로 받아서 통과/차단만 판단하는 중간 계층이 가장 작고 안정적이다.
- 충돌 그룹은 하드코딩 로직이 아니라 파라미터로 둬서, 현재는 `watering,nutrient`만 묶고 이후 장치가 늘어도 launch/config만으로 확장할 수 있게 했다.

### 안전 규칙

- 수동 우선
  - 같은 장치에 수동 명령이 수락되면 `manual_override_seconds` 동안 자동 명령을 보류한다.
- 중복 방지
  - 같은 장치, 같은 command, 같은 target, 같은 source가 `duplicate_window_seconds` 안에 반복되면 차단한다.
- 짧은 실행 잠금
  - 명령이 한 번 수락되면 같은 장치에 대해 `execution_lock_seconds` 동안 연속 실행을 차단한다.
- 충돌 그룹 잠금
  - 같은 zone 안에서 같은 conflict group에 속한 장치가 실행 중이면 동시 명령을 차단한다.
  - 현재 기본 그룹은 `watering`과 `nutrient`를 같은 irrigation 그룹으로 묶는다.

### 검증

- `python3 -m py_compile`
  - `agribot_control/agribot_control/manual_actuation_safety.py`
  - `agribot_control/agribot_control/manual_actuation_guard_node.py`
  - `agribot_control/test/test_manual_actuation_safety.py`
- `pytest -q`
  - `agribot_control/test/test_manual_actuation_safety.py`
  - `agribot_control/test/test_climate_decision.py`
  - `agribot_control/test/test_watering_decision.py`
  - `agribot_control/test/test_environment_disease_rules.py`
  - `agribot_control/test/test_mission_manager.py`
  - `agribot_control/test/test_observation_priority.py`
  - 총 `30 passed`
- `colcon build --packages-select agribot_control --symlink-install` 통과
- `timeout 3 ros2 run agribot_control manual_actuation_guard_node --ros-args -p use_sim_time:=false`
  - startup log 정상 확인

### 기대되는 동작 변화

- 사용자가 같은 장치를 연속으로 눌러도 중복 명령이 그대로 누적되지 않는다.
- 수동 조작 직후에는 같은 장치에 대한 자동 제어가 짧게 양보하므로, UI 조작과 자동 판단이 서로 덮어쓰는 일이 줄어든다.
- 급수와 영양제처럼 동시에 돌리기 껄끄러운 장치는 같은 그룹으로 묶어 동시 실행을 차단할 수 있다.

### 한계

- 이번 MR은 아직 `/iot/commands/review`를 다루지 않는다. 현재는 수동 명령과 자동 명령을 직접 guard하는 계층만 구현했다.
- 실제 IoT executor나 backend 기록 저장은 후속 티켓 범위다. 이번 작업은 dispatch 이전 중재까지만 맡는다.
- 충돌 그룹은 현재 기본값이 `watering,nutrient` 하나라서, 커튼/환기 등 추가 장치 정책은 실제 운영 기준이 확정되면 더 넣어야 한다.

### 커밋

- `수동 제어 우선·안전 잠금을 구현한다`

## [feature/S14P21A602-59] IoT 실행 요청 발행 구현

### 배경

- `S14P-410`의 목표는 환경 판단 결과가 실제 IoT 실행 요청으로 이어지도록 연결하는 것이다.
- 현재 `agribot_control`에는 `S14P-407`, `S14P-408`, `S14P-409`까지의 기반이 이미 있었다.
  - 급수 판단
  - 커튼/환기 판단
  - 수동 우선·안전 잠금
- 그래서 이번 티켓은 판단 로직 자체를 다시 만드는 것이 아니라, 그 판단을 `IoTCommand` 발행으로 바꾸는 publisher 계층을 추가하는 작업으로 잡는 것이 가장 자연스럽다.
- 원격 `ai/` 브랜치도 확인했는데 `origin/ai/illness_model_data/S14P21A602-28-aihub153`, `origin/ai/model_function_check/S14P21A602-13` 수준이었고, 이번처럼 실행 요청을 `IoTCommand`로 발행하는 control 레이어 코드는 직접 겹치지 않았다.

### 무엇을 구현했는가

- `agribot_ws/src/agribot_control/agribot_control/actuation_request_planner.py`
  - `RequestRoute`
    - `AUTO`
    - `REVIEW`
  - `ActuationRequestPlan` dataclass
  - `plan_actuation_requests()`
    - `environment_disease_rules`의 `RECOMMEND` 결과만 실행 요청 후보로 변환
    - `auto_execute=true`이면 `/iot/commands/auto`
    - 승인 필요 후보면 `/iot/commands/review`
  - `request_signature()`
    - zone, device, command, target, route, rule 기반 중복 발행 억제용 signature 제공
  - `infer_growth_stage()`
    - 현재 `PlantObservation.msg`에 `growth_stage`가 없어서
    - `ready_to_harvest=true` 또는 `blossom_end_rot`, `calcium_deficiency` 계열 class일 때만 보수적으로 `flowering_fruiting_stage`를 추론
- `agribot_ws/src/agribot_control/agribot_control/actuation_request_node.py`
  - `/environment_data`, `/plant_observation` 구독
  - zone별 최근 관측 class 반복 횟수와 confidence를 저장
  - 환경값 수신 시 `plan_actuation_requests()`를 호출해 `IoTCommand`를 생성
  - `AUTO`는 `/iot/commands/auto`, `REVIEW`는 `/iot/commands/review`로 발행
  - `log_only_on_change=true` 기본값으로 동일한 요청 반복 발행을 줄임
- `agribot_ws/src/agribot_control/config/actuation_request.yaml`
  - environment topic
  - plant observation topic
  - zone filter
  - auto/review topic
  - `requested_by`
- `agribot_ws/src/agribot_control/launch/actuation_request.launch.py`
  - 단독 기동용 launch 추가
- `agribot_ws/src/agribot_control/test/test_actuation_request_planner.py`
  - 저수분 auto 급수 요청
  - 중간 수분 review 급수 요청
  - 고습 곰팡이 반복 시 fan만 남고 watering이 빠지는지
  - 칼슘 결핍 계열 review nutrient 요청
  - 현재 PlantObservation 계약 기준 growth stage 추론 helper
  - route 차이에 따라 request signature가 달라지는지
- `agribot_ws/src/agribot_control/setup.py`
  - `actuation_request_node` console script 등록

### 왜 이렇게 구현했는가

- 문서에는 “미션 매니저가 실행 요청을 보낸다”라고 적혀 있지만, 현재 코드 구조에서 `mission_manager`는 상태머신/상태 발행 책임이 중심이고 환경 규칙 기반 actuation 판단은 별도 계층으로 분리되어 있다.
- 그래서 `mission_manager`에 device별 조건문을 직접 넣기보다, 현재 규칙 엔진 결과를 `IoTCommand`로 바꾸는 request publisher 노드를 따로 두는 편이 책임 분리가 더 선명하다.
- 현재 인터페이스의 가장 큰 빈칸은 `PlantObservation.growth_stage` 부재인데, 이걸 무시하면 영양제 요청이 영원히 나오지 않으므로 최소한의 보수적 추론 helper를 넣었다.
- 다만 이 추론은 명시적 성장단계 필드보다 약하므로, 추후 perception/interface가 정리되면 그때 실제 필드로 대체하는 것이 맞다.

### 요청 발행 규칙

- auto route
  - `auto_execute=true`
  - topic: `/iot/commands/auto`
- review route
  - `requires_approval=true`
  - topic: `/iot/commands/review`
- 발행 대상
  - 급수
  - 커튼
  - 환기팬
  - 영양제
- 중복 억제
  - 같은 zone/device 요청 signature가 바뀌지 않으면 기본적으로 재발행하지 않는다.

### 현재 인터페이스 기준의 보수적 추론

- `PlantObservation.msg`에는 `growth_stage`가 없다.
- 그래서 nutrient 요청이 필요한 칼슘 결핍 계열은 아래 경우에만 fruiting stage로 본다.
  - `ready_to_harvest=true`
  - `class_name`에 `blossom_end_rot`, `calcium_deficiency`, `배꼽썩음`, `칼슘결핍`이 포함됨
- 즉 이번 MR은 “현재 메시지 계약으로 가능한 최대치”까지만 연결한 것이다.

### 검증

- `python3 -m py_compile`
  - `agribot_control/agribot_control/actuation_request_planner.py`
  - `agribot_control/agribot_control/actuation_request_node.py`
  - `agribot_control/test/test_actuation_request_planner.py`
- `pytest -q`
  - `agribot_control/test/test_actuation_request_planner.py`
  - `agribot_control/test/test_manual_actuation_safety.py`
  - `agribot_control/test/test_climate_decision.py`
  - `agribot_control/test/test_watering_decision.py`
  - `agribot_control/test/test_environment_disease_rules.py`
  - `agribot_control/test/test_mission_manager.py`
  - `agribot_control/test/test_observation_priority.py`
  - 총 `36 passed`
- `colcon build --packages-select agribot_control --symlink-install` 통과
- `timeout 3 ros2 run agribot_control actuation_request_node --ros-args -p use_sim_time:=false`
  - startup log 정상 확인

### 기대되는 동작 변화

- 환경값과 관측값이 들어오면 실행 가능한 IoT 후보가 실제 `IoTCommand` 메시지로 발행된다.
- 자동 실행 후보와 승인 필요 후보가 topic 단계에서 나뉘므로, 이후 `manual_actuation_guard`와 backend 연동이 쉬워진다.
- 같은 요청이 계속 반복될 때 로그와 발행이 과도하게 중복되지 않는다.

### 한계

- 이번 MR은 여전히 `mission_manager` 내부에서 직접 발행하지는 않는다. 현재 구조에서는 별도 request node가 더 자연스럽다고 판단했다.
- `PlantObservation`에 성장 단계가 없어 nutrient 요청은 보수적 추론에 의존한다.
- `/iot/commands/review`를 승인/실행으로 넘기는 후속 consumer는 아직 없다.
- 실제 IoT executor나 MQTT bridge는 `S14P-503` 이후 범위다.

### 커밋

- `IoT 실행 요청 발행을 구현한다`

## [feature/S14P21A602-60] 수확 팔·바구니 모델 추가 구현

### 배경

- `S14P-451`의 목표는 토마토를 딸 손 역할을 하는 팔과, 수확물을 담을 바구니를 로봇 모델에 붙여 Gazebo에서 보이게 만드는 것이다.
- 문서 요구사항도 "팔이나 간단한 대체 장치"를 허용하고 있어서, 이번 단계에서 꼭 최종 외형용 3D 메쉬가 필요하지는 않다.
- 현재 `agribot` 모델은 주행 본체와 센서만 있는 최소 구조였고, harvest 관련 frame이나 visual은 없었다.
- 그래서 이번 작업은 나중에 `S14P-452` 자세 정렬과 수확 액션이 붙을 수 있도록, 팔/그리퍼/바구니의 위치를 먼저 고정 frame으로 확보하는 데 초점을 맞췄다.

### 무엇을 구현했는가

- `agribot_ws/src/agribot_description/models/agribot/model.sdf`
  - `harvest_arm_link` 추가
    - 세로 column
    - 전방 reach bar
    - wrist block
    - 좌우 gripper finger
  - `harvest_basket_link` 추가
    - 바닥판
    - 좌우 벽
    - 앞뒤 벽
  - 두 링크 모두 `base_link` 기준 fixed joint로 연결
  - visual과 collision을 모두 box 기반 placeholder geometry로 구성해 Gazebo 물리 부하를 과하게 늘리지 않도록 정리
- `agribot_ws/src/agribot_description/urdf/agribot.urdf`
  - RViz와 `robot_state_publisher`에서도 같은 팔/바구니가 보이도록 동일한 link/joint 구조 추가
  - 색상도 구분되게 arm은 주황/회색, basket은 갈색 계열로 지정

### 왜 이렇게 구현했는가

- 아직 마음에 드는 로봇팔 메쉬가 없더라도, 이번 티켓 자체는 충분히 진행 가능하다.
- 실제로 나중에 중요한 것은 "팔이 어디에 달려 있고, 그리퍼 끝이 대략 어디를 향하며, 바구니가 어느 위치에 있는지"이지, 지금 단계에서의 외형 디테일이 아니다.
- 그래서 link/joint/frame을 먼저 고정하고, 외형은 단순 box primitive로 두는 편이 후속 작업에 더 안전하다.
- 이 방식이면 나중에 원하는 3D 모델을 찾았을 때 visual만 mesh로 교체하고, collision은 지금처럼 단순 box로 유지할 수 있다.

### 배치 기준

- harvest arm
  - `base_link` 기준 약간 위쪽과 전방으로 reach가 나가도록 배치
  - 이후 자세 정렬 단계에서 토마토 앞 접근 기준 frame으로 쓰기 쉬운 위치
- harvest basket
  - 본체 뒤쪽 상단에 적재함처럼 배치
  - 수확 이벤트나 basket count 개념이 붙었을 때 시각적으로도 역할이 바로 보이게 구성

### 왜 placeholder여도 괜찮은가

- 이번 티켓의 문서 요구사항은 "시뮬레이션에서 보이고 충돌 없이 스폰됨" 수준이다.
- 따라서 지금은 메쉬 품질보다 아래가 더 중요하다.
  - 로봇에 실제로 harvest 관련 부위가 보일 것
  - 후속 task가 참조할 수 있는 frame이 생길 것
  - 너무 무거운 mesh collision을 넣지 않을 것
- 현재는 이 기준을 충족하도록 단순 geometry를 선택했다.

### 나중에 외형만 바꿀 수 있는가

- 가능하다.
- 이번처럼 `harvest_arm_link`, `harvest_basket_link`의 위치와 joint를 먼저 고정해 두면, 이후에는 아래 방식으로 외형만 바꾸는 것이 가능하다.
  - `visual` box를 mesh `<uri>`로 교체
  - collision은 기존 단순 box 유지
  - 필요하면 scale과 origin만 미세조정
- 다만 메쉬 크기나 중심이 크게 달라지면 아래는 다시 봐야 한다.
  - visual origin
  - collision 범위
  - inertial 값

### 로봇팔 3D 모델을 찾을 때 무엇을 봐야 하는가

- 꼭 특정 브랜드/정교한 산업용 팔일 필요는 없다.
- 대신 아래 조건은 맞는 편이 좋다.
  - low-poly 또는 Gazebo에서 감당 가능한 무게
  - `glb`, `obj`, `dae` 등 가져오기 쉬운 형식
  - forward reach가 현재 로봇 크기와 너무 동떨어지지 않을 것
  - 그리퍼 끝 방향이 토마토 쪽으로 해석하기 쉬울 것
  - 바구니와 함께 붙였을 때 본체보다 과도하게 크지 않을 것
- 즉 "정확한 최종 산업용 팔"보다 "현재 로봇에 자연스럽게 얹히고, later swap이 쉬운 메쉬"가 더 중요하다.

### 검증

- `xmllint --noout`
  - `agribot_description/models/agribot/model.sdf`
  - `agribot_description/urdf/agribot.urdf`
- `colcon build --packages-select agribot_description --symlink-install` 통과
- `ros2 launch agribot_description spawn_agribot.launch.py ...`
  - launch 초기화와 `robot_state_publisher` 기동은 정상 확인
  - 다만 현재 환경에서는 Gazebo GUI의 offscreen GLX 렌더링 문제로 시각 확인은 끝까지 못 봤다.
- `gz sim -s -r farm_world.sdf`
  - `GZ_SIM_RESOURCE_PATH`를 준 상태에서 server-only 구동 시 새 팔/바구니 모델 자체의 SDF 파싱 오류는 없음을 확인

### 한계

- 이번 MR은 fixed placeholder geometry만 넣었고, 실제 관절 구동이나 grasp 애니메이션은 없다.
- Gazebo GUI 렌더링 문제 때문에 "눈으로 본 최종 스폰 화면"까지는 현재 환경에서 확인하지 못했다.
- 이후 실제 메쉬 교체 시 scale/origin만 맞추면 되도록 만들었지만, 완전히 다른 길이의 팔을 쓰면 collision과 inertial은 같이 조정해야 한다.

### 커밋

- `수확 팔과 바구니 모델을 추가한다`

# [feature/S14P21A602-61] 수확 전 자세 정렬 구현

## MR 개요

- 수확 경로 로직에 `align_pose`를 추가해, 토마토 근처에 먼저 접근한 뒤 실제 수확 직전에는 작물 정면에 가깝게 다시 자세를 맞추도록 바꿨다.
- `harvest_route_node`의 상태 전이를 `approach -> align -> aligned hold -> harvest dwell -> return`으로 확장했다.
- 현재 로봇팔은 `S14P-451`에서 넣은 fixed placeholder geometry이므로, 이번 티켓에서는 팔 관절 제어 대신 "고정형 수확부가 토마토를 바라보는 베이스 정렬"을 수확 전 자세 정렬의 기준으로 잡았다.

## 관련 문서 기준

- 티켓: `S14P-452` 수확 전 자세 정렬
- 요구사항
  - 수확 지점에 도착한 뒤 너무 비뚤지 않게 천천히 자세를 맞출 것
  - 팔이 토마토를 집기 좋은 각도로 준비되게 만들 것
  - 토마토 앞에서 정면에 가깝게 멈추는 것이 확인 가능할 것

## 왜 진행 가능하다고 판단했는가

- 이미 `S14P-451`에서 harvest arm/basket의 위치가 `base_link` 기준으로 고정돼 있었고, arm reach도 전방 기준으로 배치돼 있었다.
- 기존 `agribot_navigation`에는
  - 토마토별 `approach_pose`를 계산하는 `harvest_routing.py`
  - 접근/복귀 시퀀스를 실행하는 `harvest_route_node.py`
  가 있어, 이번 티켓은 별도 새 패키지 없이 기존 harvest flow 위에 한 단계만 더 얹는 형태로 구현할 수 있었다.
- `/ai/`로 시작하는 원격 브랜치도 확인했지만 이번 작업과 직접 겹치는 수확 정렬/매니퓰레이터 arbitration 코드는 보이지 않았다.
  - `origin/ai/illness_model_data/S14P21A602-28-aihub153`
  - `origin/ai/model_function_check/S14P21A602-13`

## 핵심 변경 사항

### 1. harvest routing에 `align_pose` 추가

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_routing.py`
  - `HarvestRoutePlan`에 `align_pose`를 추가했다.
  - 기존 `approach_pose`는 inspect 지점에서 토마토 쪽으로 짧게 들어가는 generic 접근용으로 유지했다.
  - 새 `align_pose`는 route의 실제 lane 방향을 기준으로, 작물 row에 수직인 방향으로 정렬되도록 계산했다.
  - 즉 "토마토 근처에 먼저 접근"과 "토마토를 정면으로 보며 멈추는 최종 자세"를 분리했다.

### 2. patrol metadata에 정렬 거리 설정 추가

- `agribot_ws/src/agribot_navigation/agribot_navigation/patrol_config.py`
  - `HarvestRoutingConfig`에 `align_standoff_from_crop_m`를 추가했다.
  - 값 검증도 함께 추가했다.
- `agribot_ws/src/agribot_navigation/config/patrol_waypoints.yaml`
  - `harvest_routing.align_standoff_from_crop_m: 0.65`를 추가했다.
  - note에도 "fixed harvest arm이 토마토를 향하도록 row 정렬" 의도를 적어 두었다.

### 3. harvest route node 상태 전이 확장

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_route_node.py`
  - `approaching` 성공 후 바로 `harvesting`으로 가지 않고, 먼저 `aligning` 단계를 탄다.
  - `aligning` 성공 후에는 짧은 `aligned` hold를 둔 다음 `harvesting`으로 넘어가게 했다.
  - 정렬이 실제로 의미 있는 경우에만 align 단계가 실행되도록, approach/align pose 차이를 보고 skip 여부도 판단한다.
  - 상태 payload에 `navigation_phase`를 추가해 외부에서 현재 진행 단계가 `approaching`, `aligning`, `returning` 중 무엇인지 구분할 수 있게 했다.

## 구현 의도

- 현재 수확팔은 관절식 매니퓰레이터가 아니라 fixed geometry라서, 이번 단계에서 "팔 자체의 각도 제어"를 넣는 것은 오히려 과한 구현이다.
- 대신 아래를 만족시키는 것이 이번 티켓의 현실적인 최소 구현이라고 봤다.
  - 로봇이 작물 앞에서 비스듬히 멈추지 않을 것
  - 수확부가 전방을 향한 상태로 토마토 정면에 서도록 만들 것
  - 후속 `S14P-453` 수확 액션 서버에서 `align` 단계를 재사용할 수 있을 것
- 그래서 이번에는 manipulator joint control이 아니라, harvest pre-alignment를 내비게이션 단계에서 책임지도록 정리했다.

## 예시 결과

- `farm01_plant_01_tomato_01`
  - `approach_pose`: `(-8.3182, -7.6818, yaw=-0.7854)`
  - `align_pose`: `(-8.65, -8.0, yaw=0.0)`
  - 즉, 토마토 근처 대각선 접근 후 최종적으로는 토마토 정면에서 동쪽을 향해 정렬한다.
- `farm01_plant_01_tomato_01`을 lane 02 쪽에서 접근할 때
  - `align_pose`: `(-7.35, -8.0, yaw=pi)`
  - 즉, 같은 토마토라도 접근 aisle 반대편이면 반대 방향에서 정면 정렬한다.

## 검증

- `pytest agribot_ws/src/agribot_navigation/test/test_harvest_routing.py`
  - 4개 테스트 통과
  - `align_pose` 계산값과 기존 `approach_pose` 회귀를 함께 확인
- `python3 -m py_compile`
  - `harvest_route_node.py`
  - `harvest_routing.py`
  - `patrol_config.py`
- `colcon build --packages-select agribot_navigation --symlink-install` 통과
- `ros2 run agribot_navigation plan_harvest_route --tomato-id farm01_plant_01_tomato_01`
  - `approach_pose`와 `align_pose`가 함께 출력되는 것 확인
- `timeout 3 ros2 run agribot_navigation harvest_route_node --ros-args -p use_sim_time:=false`
  - 노드 초기화와 metadata 로딩 정상 확인

## 한계와 후속 작업

- 이번 MR은 "수확 전 정렬"까지만 구현했다.
- 실제 집기 동작이나 grasp 성공 판정은 아직 없고, 그 부분은 `S14P-453` 액션 서버에서 이어서 다루는 게 맞다.
- 현재 정렬은 Nav2 `NavigateToPose`를 한 번 더 보내는 방식이라, 진짜 저속 미세 조향 제어까지는 아니다.
- 필요하면 다음 단계에서 아래를 붙일 수 있다.
  - 정렬 전용 속도 프로파일
  - TF 기반 end-effector frame 도입
  - 수확 성공/실패 feedback과 연동한 action server

## 커밋

- `수확 전 자세 정렬을 구현한다`

# [feature/S14P21A602-62] 수확 액션 서버 구현

## MR 개요

- `S14P-453` 요구사항에 맞춰 `HarvestTomato.action` 서버를 실제로 동작하는 노드로 구현했다.
- 이번 MR의 핵심은 `접근 -> 자세 정렬 -> 집기 -> 성공 확인 -> 바구니 적재`를 하나의 action goal로 묶고, 각 단계의 progress feedback과 최종 success/failure result를 반환하도록 만든 것이다.
- 기존 `S14P-452`에서 만든 `approach_pose`, `align_pose`를 그대로 재사용하되, 아직 실제 매니퓰레이터 관절 제어는 없으므로 pick/verify/stow는 시간 기반 시뮬레이션 단계로 연결했다.

## 왜 진행 가능하다고 판단했는가

- `agribot_interfaces/action/HarvestTomato.action` 정의가 이미 저장소에 존재했다.
- `agribot_navigation`에는 이미 아래 자산이 준비돼 있었다.
  - `compute_harvest_route()`로 fruit별 접근/정렬 pose를 계산하는 로직
  - `S14P-452`에서 추가한 align 단계
  - Nav2 `NavigateToPose` action client 기반의 수확 접근 시퀀스 구현 경험
- 따라서 이번 티켓은 "새 action 계약을 정의"하는 작업이 아니라, 기존 계약을 실제 서버 노드로 실체화하는 작업으로 판단했다.
- `/ai/` 원격 브랜치도 확인했지만, 이번 harvest action orchestration과 직접 겹치는 수확 액션 서버 구현은 없었다.
  - `origin/ai/illness_model_data/S14P21A602-28-aihub153`
  - `origin/ai/model_function_check/S14P21A602-13`

## 주요 변경 사항

### 1. HarvestTomato 액션 서버 구현

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_server.py`
  - `ActionServer(HarvestTomato)`를 추가했다.
  - goal 하나로 아래 순서를 수행한다.
    - planning
    - patrol pause 대기
    - approach navigation
    - align navigation
    - picking
    - verifying
    - basket stow
  - 각 단계마다 feedback의
    - `current_phase`
    - `progress_pct`
    - `aligned_to_target`
    - `gripper_engaged`
    를 갱신하도록 했다.
  - Nav2 서버가 없거나 patrol stop service가 없으면 action을 abort하고, 원인 메시지를 result로 돌려준다.
  - cancel 요청이 오면 현재 대기/내비게이션 단계를 중단하고 canceled result로 정리한다.

### 2. 액션 서버용 보조 로직 분리

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_support.py`
  - goal 정규화와 검증을 분리했다.
  - 특히 `ros2 action send_goal`에서 빈 문자열이 `None` 또는 문자열 `"None"`으로 들어오는 케이스를 안전하게 처리하도록 만들었다.
  - runtime 중 이미 수확한 fruit를 다시 요청하면 거절할 수 있도록 duplicate check helper도 넣었다.
  - feedback/result/event 생성도 helper로 뽑아 테스트 가능하게 정리했다.

### 3. launch / config 추가

- `agribot_ws/src/agribot_navigation/config/harvest_action_server.yaml`
  - action 이름, Nav2 action 이름, patrol stop 관련 timeout, picking/verify/stow duration을 파라미터화했다.
- `agribot_ws/src/agribot_navigation/launch/harvest_action_server.launch.py`
  - 단독으로 액션 서버를 띄울 수 있는 launch를 추가했다.
- `agribot_ws/src/agribot_navigation/setup.py`
  - `harvest_action_server` console entry point를 등록했다.

## 설계 의도

- 이번 티켓은 문서상으로 "수확이라는 긴 작업을 한 번의 명령으로 실행"하는 것이 핵심이다.
- 아직 `S14P-454`, `S14P-455`가 남아 있어서 아래는 일부러 최소 구현으로 뒀다.
  - 월드에서 토마토 모델 제거
  - 바구니 실적 영속화
  - 수확 후 복귀
- 대신 이번 단계에서는 후속 확장이 쉬운 액션 orchestration 뼈대를 우선 만들었다.
  - 이미 있는 route planner를 재사용
  - pick / verify / stow는 phase만 먼저 연결
  - `HarvestEvent`를 발행할 자리도 마련

## 결과적으로 무엇이 가능해졌는가

- 이제 외부에서 `HarvestTomato` goal 하나만 보내면 서버가 수확 시퀀스를 실행한다.
- 진행률은 phase 기반 feedback으로 확인할 수 있다.
- 성공/실패는 result로 받는다.
- 실행 결과는 `HarvestEvent` 토픽으로도 남길 수 있다.
- 즉, `S14P-452`의 정렬 로직이 더 이상 단순 utility가 아니라, action workflow 안에서 재사용 가능한 단계가 되었다.

## 검증

- `pytest agribot_ws/src/agribot_navigation/test/test_harvest_action_support.py agribot_ws/src/agribot_navigation/test/test_harvest_routing.py`
  - 총 12개 테스트 통과
  - goal 정규화, duplicate reject, feedback/result 생성, align 필요 여부, 기존 routing 회귀를 함께 확인
- `python3 -m py_compile`
  - `harvest_action_server.py`
  - `harvest_action_support.py`
- `colcon build --packages-select agribot_navigation --symlink-install` 통과
- `ros2 run agribot_navigation harvest_action_server --ros-args -p use_sim_time:=false`
  - action server 초기화 정상 확인
- `ros2 action send_goal /harvest_tomato ...`
  - 현재 환경에는 `navigate_to_pose` action server가 없어서 success path까지는 못 갔지만,
  - goal accepted 이후 `NavigateToPose action server not available on navigate_to_pose.` 메시지로 abort result를 반환하는 것은 확인했다.
  - 즉, action contract 자체와 실패 처리 경로는 정상 동작함을 확인했다.

## 한계와 후속 작업

- 이번 MR은 집기/검증/바구니 적재를 시간 기반 시뮬레이션으로 연결한 것이다.
- 아직 실제로 아래는 하지 않는다.
  - end-effector 움직임 제어
  - 실제 grasp 성공 판정
  - 월드에서 fruit 제거
  - 수확 후 홈/순찰 지점 복귀
- 따라서 다음 티켓에서 자연스럽게 이어질 부분은 아래다.
  - `S14P-454`: basket count 영속화, HarvestEvent 고도화, fruit 재수확 방지 강화
  - `S14P-455`: return-home 시퀀스 연결
  - `S14P-456`: retry / fail-safe 정책

## 커밋

- `수확 액션 서버를 구현한다`

# [feature/S14P21A602-63] 바구니 적재·수확 기록 구현

## MR 개요

- `S14P-454` 요구사항에 맞춰 수확 성공 시 바구니 적재 수량과 수확 이력을 함께 남기는 상태 발행 흐름을 추가했다.
- 이번 MR의 핵심은 `HarvestTomato` action server가 성공한 fruit를 단순히 이벤트 1건으로 끝내지 않고, 누적 바구니 상태와 재수확 방지 기준을 함께 유지하도록 만든 것이다.
- 문서에는 "시뮬레이터에서 사라지게 하거나 바구니 개수가 늘어나게"라고 돼 있는데, 현재 저장소 구조에서는 Gazebo 엔티티 제거보다 `바구니 적재 수량 증가 + 수확 기록 상태 발행`이 더 작고 안정적인 구현 단위라고 판단했다.

## 왜 진행 가능하다고 판단했는가

- `S14P-453`에서 이미 `harvest_action_server.py`가 아래 동작을 일부 수행하고 있었다.
  - 수확 성공 시 `HarvestEvent` 발행
  - 런타임 중 이미 수확한 fruit ID 재요청 차단
  - basket count 내부 변수 증가
- 따라서 이번 티켓은 새로운 수확 파이프라인을 만드는 작업이 아니라, 이미 있는 성공 처리 흐름을 `외부에서 관측 가능한 상태`로 끌어올리는 작업으로 해석할 수 있었다.
- `/ai/` 원격 브랜치도 다시 확인했지만, 이번 basket state / harvest record state 확장과 직접 충돌하는 구현은 없었다.
  - `origin/ai/illness_model_data/S14P21A602-28-aihub153`
  - `origin/ai/model_function_check/S14P21A602-13`

## 주요 변경 사항

### 1. HarvestBasketState 메시지 추가

- `agribot_ws/src/agribot_interfaces/msg/HarvestBasketState.msg`
  - 바구니 적재 상태를 한 번에 볼 수 있는 메시지를 새로 추가했다.
  - 포함 필드는 아래다.
    - `zone_id`
    - `basket_count`
    - `harvested_count`
    - `remaining_ready_count`
    - `last_event_id`
    - `last_harvested_fruit_id`
    - `loaded_fruit_ids`
- `agribot_ws/src/agribot_interfaces/CMakeLists.txt`
  - 새 메시지가 정상 생성되도록 인터페이스 빌드 목록에 등록했다.

### 2. 수확 액션 서버의 상태 누적 확장

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_server.py`
  - 기존 `HarvestEvent` publisher 외에 `harvest/basket_state` publisher를 추가했다.
  - 이 publisher는 `TRANSIENT_LOCAL + RELIABLE` QoS로 만들어서 늦게 붙는 subscriber도 마지막 바구니 상태를 받을 수 있게 했다.
  - 서버 시작 시 초기 상태를 1회 publish한다.
  - 수확 성공 시 아래 상태를 함께 갱신한다.
    - `_basket_count`
    - `_harvested_tomato_ids`
    - `_loaded_tomato_ids`
    - `_last_success_event_id`
    - `_last_harvested_tomato_id`
    - `_remaining_ready_count`
  - 성공 이벤트 발행 직후 바구니 상태도 다시 publish하도록 연결했다.

### 3. 상태/이벤트 조립 helper 분리

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_support.py`
  - `build_basket_state()` helper를 추가해서 message 조립을 테스트 가능한 순수 함수로 뺐다.
  - 기존 수확 이벤트 helper들과 같은 위치에 두어, 수확 성공 결과가 어떤 필드로 외부에 노출되는지 한 파일에서 볼 수 있게 했다.

### 4. 설정과 회귀 테스트 보강

- `agribot_ws/src/agribot_navigation/config/harvest_action_server.yaml`
  - `basket_state_topic` 파라미터를 추가했다.
- `agribot_ws/src/agribot_navigation/test/test_harvest_action_support.py`
  - 바구니 상태 메시지 생성 테스트를 추가했다.
  - loaded fruit 목록, 누적 count, remaining ready count, 마지막 이벤트/fruit ID가 기대값대로 채워지는지 확인한다.

## 설계 의도

- 이 티켓의 문서 요구사항은 크게 두 가지다.
  - 수확 성공을 기록으로 남길 것
  - 같은 fruit를 다시 수확하지 못하게 할 것
- 현재 액션 서버는 이미 runtime duplicate 차단 로직을 갖고 있었으므로, 이번 단계에서는 그 기준을 더 명시적인 상태 토픽으로 드러내는 쪽이 효과가 크다고 봤다.
- 반대로 "월드에서 fruit를 실제로 삭제"하는 작업은 Gazebo entity lifecycle과 모델 네이밍 계약까지 얽혀서 이 티켓 하나로 끝내기 어렵다.
- 그래서 이번 MR은 아래 원칙으로 범위를 정했다.
  - P0 데모에 직접 필요한 관측 가능한 상태 먼저
  - 재수확 방지 기준은 유지
  - 백엔드/대시보드가 나중에 붙을 수 있는 메시지 계약 우선

## 결과적으로 무엇이 가능해졌는가

- 수확 액션 서버가 시작되면 현재 바구니/수확 상태를 즉시 topic으로 노출한다.
- fruit 하나를 성공적으로 수확하면 `HarvestEvent`와 `HarvestBasketState`가 함께 갱신된다.
- 외부 노드는 이제 아래를 별도 추론 없이 바로 사용할 수 있다.
  - 현재 바구니 개수
  - 지금까지 총 수확 개수
  - 마지막으로 적재한 fruit
  - 아직 남은 ready fruit 개수
  - 이미 적재된 fruit ID 목록
- 따라서 이후 백엔드 DB 적재나 대시보드 통계 화면은 action result만 기다리지 않고 topic 구독만으로도 현재 상태를 동기화할 수 있다.

## 검증

- `python3 -m py_compile`
  - `harvest_action_server.py`
  - `harvest_action_support.py`
- `colcon build --packages-select agribot_interfaces agribot_navigation --symlink-install` 통과
- `pytest agribot_ws/src/agribot_navigation/test/test_harvest_action_support.py agribot_ws/src/agribot_navigation/test/test_harvest_routing.py`
  - 총 13개 테스트 통과
- `ros2 run agribot_navigation harvest_action_server --ros-args -p use_sim_time:=false -p nav_server_wait_sec:=0.5`
  - 액션 서버와 basket state publisher 초기화 확인
- `ros2 topic info /harvest/basket_state -v`
  - `HarvestBasketState` publisher 존재와 `TRANSIENT_LOCAL` QoS 확인
- `ros2 topic echo --once /harvest/basket_state --qos-durability transient_local --qos-reliability reliable`
  - 초기 latched 상태 확인
  - `basket_count=0`, `harvested_count=0`, `remaining_ready_count=72`, `loaded_fruit_ids=[]`

## 한계와 후속 작업

- 이번 MR은 basket 상태를 메모리와 topic 기준으로 관리하는 단계다.
- 아직 하지 않은 것은 아래다.
  - Gazebo 상 fruit 모델 제거
  - ROS 재시작 이후 basket state 영속 복구
  - backend DB로 `HarvestEvent` 자동 저장
  - basket full 정책과 unload 시나리오
- 따라서 다음 단계에서 자연스럽게 이어질 작업은 아래다.
  - `S14P-455`: 복귀 시퀀스와 basket 상태 연동
  - backend 실시간 수집기에서 `HarvestEvent` / `HarvestBasketState` 저장
  - 동일 fruit 재수확 방지 기준을 world entity 상태와도 연결

## 커밋

- `바구니 적재와 수확 기록을 구현한다`

# [feature/S14P21A602-64] 수확 후 홈 복귀 구현

## MR 개요

- `S14P-455` 요구사항에 맞춰 `HarvestTomato` action server의 성공 경로 뒤에 복귀 시퀀스를 추가했다.
- 이번 MR의 핵심은 수확이 끝난 뒤 액션이 바로 끝나는 것이 아니라, `RETURN_HOME` 단계로 지정된 복귀 지점까지 이동하고 필요하면 `RESUME` 단계로 순찰 재개까지 이어지도록 만든 것이다.
- 별도 임시 문자열 토픽을 새로 만드는 대신, 이미 저장소에 있던 `MissionStatus.msg` 계약을 그대로 재사용해서 복귀 상태를 남기도록 정리했다.

## 왜 진행 가능하다고 판단했는가

- `agribot_navigation/harvest_routing.py`에는 이미 아래 정보가 준비돼 있었다.
  - `return_mode`
  - `return_waypoint_id`
  - `fallback_return_waypoint_id`
- `agribot_navigation/harvest_route_node.py`에는 구형 구현이지만 아래 복귀 흐름이 이미 있었다.
  - return navigation
  - fallback return target
  - patrol resume service 호출
- 따라서 이번 티켓은 "복귀 개념을 새로 설계"하는 작업이 아니라, 기존 route metadata와 old coordinator 로직을 `HarvestTomato` 액션 서버 쪽으로 수렴시키는 작업으로 볼 수 있었다.
- `/ai/` 원격 브랜치도 다시 확인했지만, 이번 return-home orchestration과 직접 겹치는 코드는 없었다.
  - `origin/ai/illness_model_data/S14P21A602-28-aihub153`
  - `origin/ai/model_function_check/S14P21A602-13`

## 주요 변경 사항

### 1. 수확 액션 서버에 복귀 단계 추가

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_server.py`
  - 수확 성공 후 아래 순서가 이어지도록 확장했다.
    - `STOWING`
    - `RETURN_HOME`
    - `RESUME`
    - `COMPLETED`
  - 복귀 목표는 기존 patrol metadata의 `return_waypoint_id`를 사용한다.
  - 1차 복귀 목표가 실패하면 `fallback_return_waypoint_id`로 한 번 더 시도한다.
  - `return_mode_override` 파라미터가 비어 있으면 기존 waypoint metadata의 기본값을 따르고, 필요하면 `home`으로 강제할 수 있게 했다.

### 2. patrol 재개 서비스 연결

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_server.py`
  - `patrol_resume_service` client를 추가했다.
  - return mode가 `resume_patrol`이고 `auto_resume_patrol=true`이면 복귀 후 `patrol/resume`를 호출한다.
  - resume 실패는 action 전체 실패로 처리하되, 이미 수확이 성공한 경우에는 성공 `HarvestEvent`를 덮어쓰지 않도록 분리했다.

### 3. MissionStatus 기반 복귀 상태 발행

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_support.py`
  - `build_mission_status()` helper를 추가했다.
  - `RETURN_HOME`, `RESUME` phase progress를 `PHASE_PROGRESS_PCT`에 정의했다.
- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_server.py`
  - `harvest/mission_status` publisher를 `TRANSIENT_LOCAL + RELIABLE` QoS로 추가했다.
  - action 실행 중 주요 단계마다 mission status를 같이 publish하도록 연결했다.
  - 최종 성공 시에는 `current_phase=RESUME`, `state=COMPLETED`로 마무리되도록 했다.

### 4. 설정과 테스트 보강

- `agribot_ws/src/agribot_navigation/config/harvest_action_server.yaml`
  - `patrol_resume_service`
  - `mission_status_topic`
  - `return_mode_override`
  - `auto_resume_patrol`
    를 파라미터로 추가했다.
- `agribot_ws/src/agribot_navigation/test/test_harvest_action_support.py`
  - `MissionStatus` helper 테스트를 추가해서 복귀 단계 상태가 계약대로 조립되는지 확인했다.

## 설계 의도

- 문서 요구사항은 "수확 후 home 또는 이전 순찰 지점으로 돌아가고, 복귀 완료 상태를 남길 것"이다.
- 지금 저장소에는 복귀 경로를 계산하는 자산과 순찰 resume 서비스 계약이 이미 있었으므로, 새로운 액션이나 새로운 route planner를 만들 필요는 없었다.
- 대신 이번 MR은 아래 원칙으로 범위를 정했다.
  - 복귀는 기존 `HarvestTomato` action 안에서 끝낼 것
  - 이미 성공한 수확 이벤트와 복귀 실패를 구분할 것
  - 후속 client/backend가 재사용할 수 있게 `MissionStatus` 계약을 우선할 것

## 결과적으로 무엇이 가능해졌는가

- 이제 `HarvestTomato`는 단순 수확 액션이 아니라 수확 완료 후 복귀까지 포함한 시퀀스가 된다.
- 외부에서는 `harvest/mission_status`만 구독해도 아래 흐름을 읽을 수 있다.
  - `PLANNING`
  - `APPROACHING`
  - `ALIGNING`
  - `PICKING`
  - `VERIFYING`
  - `STOWING`
  - `RETURN_HOME`
  - `RESUME`
- 복귀 목표는 현재 waypoint metadata를 그대로 따르므로, 순찰 재개와 home 복귀를 같은 액션 서버 안에서 파라미터로 제어할 수 있게 됐다.

## 검증

- `python3 -m py_compile`
  - `harvest_action_server.py`
  - `harvest_action_support.py`
- `colcon build --packages-select agribot_navigation --symlink-install` 통과
- `pytest agribot_ws/src/agribot_navigation/test/test_harvest_action_support.py agribot_ws/src/agribot_navigation/test/test_harvest_routing.py`
  - 총 14개 테스트 통과
- `ros2 run agribot_navigation harvest_action_server --ros-args -p use_sim_time:=false -p nav_server_wait_sec:=0.5`
  - 액션 서버와 `harvest/mission_status` publisher 초기화 확인
- `ros2 action send_goal /harvest_tomato ...`
  - 현재 환경에서는 `APPROACHING` 단계 내비게이션이 성공하지 않아 전체 성공 경로까지는 못 갔다.
  - 대신 goal accepted, 실패 result 반환, `harvest/mission_status` latched 상태 발행은 확인했다.
- `ros2 topic echo --once /harvest/mission_status --qos-durability transient_local --qos-reliability reliable`
  - 실행 중 `APPROACHING`
  - 실패 후 `state=FAILED`, `current_phase=APPROACHING`
    상태가 남는 것을 확인했다.

## 한계와 후속 작업

- 이번 MR은 return-home orchestration을 액션 서버 안에 넣은 단계다.
- 아직 현재 환경에서는 아래를 end-to-end로 확인하지 못했다.
  - 수확 성공 후 실제 `RETURN_HOME` navigation 완료
  - `patrol/resume` 성공까지 포함한 완전 성공 경로
- 따라서 다음 단계에서 보면 좋은 것은 아래다.
  - Nav2가 정상 기동된 시뮬레이션에서 성공 경로 회귀 테스트
  - `home` 강제 복귀와 `resume_patrol` 복귀를 둘 다 시연 스크립트로 검증
  - 필요하면 `RobotStatus`까지 연결해서 `is_returning_home` 상태도 함께 표시

## 커밋

- `수확 후 홈 복귀를 구현한다`

# [feature/S14P21A602-65]  수확 실패 재시도·안전 정지 구현

## MR 개요

- `S14P-456` 요구사항에 맞춰 `HarvestTomato` action server에 실패 재시도 규칙과 안전 정지 흐름을 추가했다.
- 이번 MR의 핵심은 수확 실패가 발생했을 때 즉시 끝내는 대신, retry 가능한 phase만 제한적으로 다시 시도하고, retry limit을 넘기면 `SAFETY_STOP` 상태와 failure alert를 남기도록 만든 것이다.
- 기존 `HarvestEvent`는 유지하고, 실패 정책 메타데이터는 `MissionStatus`와 alert topic으로 분리해서 후속 backend/client가 그대로 구독할 수 있게 정리했다.

## 왜 진행 가능하다고 판단했는가

- 현재 `harvest_action_server.py`는 이미 아래 기반을 갖고 있었다.
  - phase별 실행 분리
  - `MissionStatus` 발행
  - 수확 성공/실패 `HarvestEvent`
  - `patrol/stop`, `patrol/resume` 서비스 연결
- 따라서 이번 티켓은 새 액션 서버를 만드는 작업이 아니라, 기존 phase 실행 위에 failure policy를 얹는 작업으로 해석할 수 있었다.
- `/ai/` 원격 브랜치도 다시 확인했지만, 이번 retry/safety-stop 정책과 직접 겹치는 코드는 없었다.
  - `origin/ai/illness_model_data/S14P21A602-28-aihub153`
  - `origin/ai/model_function_check/S14P21A602-13`

## 주요 변경 사항

### 1. retry 정책 helper 추가

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_support.py`
  - `should_retry_phase()` helper를 추가했다.
  - retryable phase allowlist와 retry limit을 기준으로 재시도 가능 여부를 결정한다.
  - `PHASE_PROGRESS_PCT`에 `SAFETY_STOP` phase를 추가했다.

### 2. failure alert payload 추가

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_support.py`
  - `build_failure_alert_payload()` helper를 추가했다.
  - 아래 정보를 JSON으로 묶어서 alert topic에 publish하도록 만들었다.
    - `alert_type`
    - `severity`
    - `mission_id`
    - `zone_id`
    - `fruit_id`
    - `current_phase`
    - `retry_count`
    - `safety_stop_requested`
    - `safety_stop_completed`
    - `harvest_completed`
    - `failure_reason`

### 3. 액션 서버에 retry / safety-stop orchestration 추가

- `agribot_ws/src/agribot_navigation/agribot_navigation/harvest_action_server.py`
  - `phase_retry_limit`, `phase_retry_backoff_sec`, `retryable_phases`, `safety_stop_on_failure`, `failure_alert_topic` 파라미터를 추가했다.
  - `_run_phase_with_retry()` wrapper를 추가해서 아래 phase를 bounded retry로 감쌌다.
    - `WAITING_FOR_PATROL_PAUSE`
    - `APPROACHING`
    - `ALIGNING`
    - `RETURN_HOME`
    - `RESUME`
  - retry limit을 넘기면 `_perform_safety_stop()`을 호출해 patrol stop service를 다시 요청하고, 결과를 `SAFETY_STOP` 상태와 alert로 남긴다.
  - terminal failure 시 `MissionStatus.retry_count`가 실제 retry 횟수를 반영하도록 연결했다.

### 4. 설정과 테스트 보강

- `agribot_ws/src/agribot_navigation/config/harvest_action_server.yaml`
  - retry limit, backoff, retryable phase 목록, safety stop 사용 여부, failure alert topic을 추가했다.
- `agribot_ws/src/agribot_navigation/test/test_harvest_action_support.py`
  - retry helper 테스트
  - failure alert payload 테스트
    를 추가해서 policy가 계약대로 조립되는지 검증했다.

## 설계 의도

- 문서 요구사항의 핵심은 아래 세 가지였다.
  - 무한 반복하지 않을 것
  - 실패 이유를 남길 것
  - 필요하면 오류 상태와 사용자 알림을 보낼 것
- 그래서 이번 MR은 아래 원칙으로 범위를 정했다.
  - retry는 phase별 allowlist로 제한
  - retry 횟수는 파라미터로 관리
  - terminal failure는 `SAFETY_STOP`으로 수렴
  - alert는 backend 전용 인터페이스를 새로 만들지 않고 JSON `String` topic으로 남김

## 결과적으로 무엇이 가능해졌는가

- 이제 수확 액션이 실패하더라도 바로 끝나지 않고, 허용된 phase는 정해진 횟수만큼 다시 시도한다.
- retry limit을 넘기면 `harvest/mission_status`는 `state=FAILED`, `current_phase=SAFETY_STOP`으로 남는다.
- 동시에 `harvest/alerts`에 terminal failure alert가 publish된다.
- 즉, 외부에서는 아래 두 경로로 실패를 추적할 수 있다.
  - `MissionStatus`로 현재 실패 상태와 retry count 확인
  - `harvest/alerts`로 사용자 알림/로그 저장용 payload 수집

## 검증

- `python3 -m py_compile`
  - `harvest_action_server.py`
  - `harvest_action_support.py`
- `colcon build --packages-select agribot_navigation --symlink-install` 통과
- `pytest agribot_ws/src/agribot_navigation/test/test_harvest_action_support.py agribot_ws/src/agribot_navigation/test/test_harvest_routing.py`
  - 총 16개 테스트 통과
- `ros2 run agribot_navigation harvest_action_server --ros-args -p use_sim_time:=false -p nav_server_wait_sec:=0.5 -p phase_retry_backoff_sec:=0.1`
  - 액션 서버와 failure alert topic 초기화 확인
- `ros2 action send_goal /harvest_tomato ...`
  - 현재 환경에서는 Nav2 서버가 없어 `APPROACHING`이 실패했다.
  - 대신 아래 흐름은 확인했다.
    - `APPROACHING` retry 경고 로그
    - retry limit 초과 후 safety stop 로그
    - failure alert publish
    - aborted result 반환
- `ros2 topic echo --once /harvest/mission_status --qos-durability transient_local --qos-reliability reliable`
  - 최종 상태가 `state=FAILED`, `current_phase=SAFETY_STOP`, `retry_count=1`로 남는 것 확인
- `ros2 topic echo --once /harvest/alerts`
  - `HARVEST_FAILURE` alert payload 수신 확인

## 한계와 후속 작업

- 이번 MR은 failure policy와 안전 정지 orchestration을 넣은 단계다.
- 아직 현재 환경에서는 아래를 성공 경로와 함께 검증하지 못했다.
  - recoverable failure 뒤 retry 성공
  - 실제 return-home 단계에서 retry 후 복구
  - patrol stop service가 active patrol 상태에서 safety stop으로 수렴하는지
- 따라서 다음 단계에서 보면 좋은 것은 아래다.
  - Nav2/순찰 노드가 같이 살아 있는 시뮬레이션에서 retry 성공/실패 회귀 테스트
  - backend 실시간 수집기에서 `harvest/alerts` 저장
  - 필요하면 alert를 `Alert` 테이블 계약과도 맞추기

## 커밋

- `수확 실패 재시도와 안전 정지를 구현한다`

# [feature/S14P21A602-70] 천장 커튼 제어 노드 구현

## 작업 브랜치

- `feature/S14P21A602-70`

## 관련 티켓

- `S14P-505` 천장 커튼 제어 노드

## 요구사항 다시 정리

- 천장 커튼을 열고 닫는 역할의 노드를 만든다.
- `open_curtain`, `close_curtain`, 열림 비율 변경 명령을 처리한다.
- 현재 커튼 상태를 계속 발행한다.
- 장치 상태와 실행 결과 로그를 확인할 수 있어야 한다.

## 왜 진행 가능하다고 판단했는가

- 직전 작업으로 `agribot_iot`에는 이미 아래 기반이 있었다.
  - `iot_devices.yaml` 기반 장치 매핑
  - `IoTCommand` 수신 구조
  - `IoTDeviceState` 발행 구조
  - JSON 실행 결과 발행 구조
  - 급수 제어 노드 패턴
- 따라서 이번 티켓은 새 아키텍처를 만드는 일이 아니라, 기존 IoT 실행 패턴을 커튼 장치에 맞게 확장하는 작업으로 해석할 수 있었다.
- `/ai/` 원격 브랜치도 다시 확인했지만 이번 작업 경로와 직접 겹치는 변경은 없었다.
  - `origin/ai/illness_model_data/S14P21A602-28-aihub153`
  - `origin/ai/model_function_check/S14P21A602-13`

## 주요 변경 사항

### 1. 커튼 명령 해석 로직 추가

- `agribot_ws/src/agribot_iot/agribot_iot/curtain_controller_logic.py`
  - `open_curtain`
  - `close_curtain`
  - `set_curtain_position`
    세 가지 명령을 표준화해서 실행 계획으로 변환하도록 만들었다.
  - `percent_open`, `percent`, `percent_closed` 단위를 해석한다.
  - control 레이어가 보내는 `set_curtain_position + percent_closed`를 실제 커튼 열림 비율로 변환한다.
  - 현재 위치와 목표 위치 차이, 전환 속도를 기준으로 예상 실행 시간을 계산한다.
  - `IoTDeviceState`와 JSON 실행 결과 payload 조립 helper도 같이 넣었다.

### 2. 커튼 제어 노드 추가

- `agribot_ws/src/agribot_iot/agribot_iot/curtain_controller_node.py`
  - `/iot/commands/dispatch`에서 `device_type=curtain` 명령만 소비하도록 만들었다.
  - 구역/장치 매핑은 `iot_devices.yaml`을 그대로 재사용한다.
  - 초기 상태는 열림 100%로 두고, 주기적으로 현재 상태를 `/iot/device_state`에 발행한다.
  - 명령이 들어오면 목표 열림 비율까지 서서히 이동하면서 상태를 `OPENING`, `CLOSING`, `PARTIAL_OPEN`, `OPEN`, `CLOSED`로 갱신한다.
  - 이동 완료 시 `/iot/command_result`에 JSON 결과를 publish한다.
  - 이동 중 새 명령이 오면 이전 명령은 `INTERRUPTED` 결과로 정리하도록 넣었다.

### 3. 장치 매핑과 실행 설정 보강

- `agribot_ws/src/agribot_iot/config/iot_devices.yaml`
  - `farm_01_curtain` 장치에 아래 지원 명령을 명시했다.
    - `open_curtain`
    - `close_curtain`
    - `set_curtain_position`
  - 커튼 기본 단위를 `percent_open`으로 정리했다.
- `agribot_ws/src/agribot_iot/config/curtain_controller.yaml`
  - 명령/상태/결과 토픽
  - 초기 열림 비율
  - 상태 발행 주기
  - 전환 속도
    를 파라미터로 분리했다.
- `agribot_ws/src/agribot_iot/launch/curtain_controller.launch.py`
  - 단독 실행용 launch 파일을 추가했다.
- `agribot_ws/src/agribot_iot/setup.py`
  - `curtain_controller_node` entry point를 등록했다.

### 4. MQTT 계약 문서와 테스트 추가

- `docs/iot_mqtt_contract.md`
  - 문서 범위를 `S14P-505`까지 확장했다.
  - 커튼 제어 노드가 처리하는 명령과 단위 규약을 적었다.
- `agribot_ws/src/agribot_iot/test/test_curtain_controller_logic.py`
  - `percent_closed -> opening_ratio` 변환
  - 이미 열린 상태에서 `open_curtain`
  - `close_curtain` 종료 상태
    를 검증하는 테스트를 추가했다.

## 설계 의도

- 기존 급수 노드와 최대한 같은 계약을 유지했다.
  - 입력: `IoTCommand`
  - 상태: `IoTDeviceState`
  - 결과: JSON `String`
- control 레이어는 이미 `set_curtain_position`과 `percent_closed`를 사용하고 있었기 때문에, IoT 레이어에서 그 의미를 해석하는 쪽으로 두었다.
- 상태 토픽은 계속 발행하되, 로그는 변화가 있을 때만 남겨서 런타임 노이즈를 줄였다.

## 결과적으로 무엇이 가능해졌는가

- 이제 `farm_01_curtain` 장치에 대해 아래 흐름이 닫힌다.
  - 수동/자동 제어 명령 수신
  - 열림 비율 해석
  - 상태 전이 발행
  - 실행 완료 결과 발행
- `close_curtain`, `open_curtain`, `set_curtain_position` 중 어느 명령을 보내도 IoT 장치 레이어에서 일관된 형식으로 응답할 수 있게 됐다.

## 검증

- `python3 -m py_compile`
  - `curtain_controller_logic.py`
  - `curtain_controller_node.py`
- `pytest agribot_ws/src/agribot_iot/test`
  - 총 10개 테스트 통과
- `colcon build --packages-select agribot_iot --symlink-install` 통과
- `timeout 3 ros2 run agribot_iot curtain_controller_node --ros-args -p use_sim_time:=false -p state_publish_hz:=2.0`
  - 노드 기동과 초기 상태 발행 확인
- `ros2 topic pub --once /iot/commands/dispatch ... command_type=set_curtain_position target_value=60.0 unit=percent_closed`
  - `100% open -> 40% open`으로 해석되는지 확인
- `ros2 topic echo --once /iot/device_state`
  - `state=CLOSING` 및 상태 메시지 확인
- `ros2 topic echo --once /iot/command_result`
  - 실행 완료 JSON 결과 수신 확인
- 실제 노드 로그에서도 아래를 확인했다.
  - `Curtain motion started`
  - 중간 `CLOSING`
  - 최종 `PARTIAL_OPEN`
  - result publish 로그

## 한계와 후속 작업

- 이번 MR은 커튼 단독 제어 노드까지 닫은 단계다.
- 아직 아래는 후속 티켓에서 연결하는 것이 맞다.
  - `S14P-506` 환기팬 제어 노드
  - `S14P-507` 영양제 장치 제어 노드
  - `S14P-508` 여러 장치 결과를 backend로 모으는 통합 경로
- 현재는 단일 구역 `farm_01` 기준으로 검증했고, 다구역 확장은 `iot_devices.yaml`에 같은 패턴으로 확장하면 된다.

## 커밋

- `S14P21A602-70 천장 커튼 제어 노드를 구현한다`

# [feature/S14P21A602-71] 환기팬 제어 노드 구현

## 작업 브랜치

- `feature/S14P21A602-71`

## 관련 티켓

- `S14P-506` 환기팬 제어 노드

## 요구사항 다시 정리

- 환기팬을 켜고 끄는 노드를 만든다.
- 속도 단계 변경을 처리한다.
- 실행 시간을 기록하고 상태로 볼 수 있어야 한다.
- 명령을 보내면 상태가 바뀌고 결과 로그가 남아야 한다.

## 왜 진행 가능하다고 판단했는가

- 환기팬은 `IoTCommand -> IoTDeviceState -> JSON result` 구조만 있으면 현재 코드베이스에 자연스럽게 들어갈 수 있다.
- control 레이어도 이미 아래 계약을 갖고 있었다.
  - `device_type=fan`
  - `command_type=set_fan_level`
  - `unit=level`
- 다만 현재 브랜치가 `develop` 기준선이라 `agribot_iot` 선행 기반이 비어 있었다.
- 그래서 먼저 아래 기존 feature 변경을 현재 브랜치로 안전하게 가져온 뒤, 그 위에 `S14P-506`을 구현했다.
  - `S14P-501~504` IoT 기반
  - `S14P-505` 커튼 제어 노드
- `/ai/` 원격 브랜치도 다시 확인했지만 이번 IoT/fan 범위와 직접 겹치는 파일 변경은 없었다.
  - `origin/ai/illness_model_data/S14P21A602-28-aihub153`
  - `origin/ai/model_function_check/S14P21A602-13`

## 주요 변경 사항

### 1. fan 명령 해석 로직 추가

- `agribot_ws/src/agribot_iot/agribot_iot/fan_controller_logic.py`
  - 아래 명령을 처리하도록 만들었다.
    - `turn_on_fan`
    - `turn_off_fan`
    - `set_fan_level`
    - `set_fan_speed`
  - `set_fan_speed`는 기존 매핑 호환용 alias로 받고, control 레이어 표준은 `set_fan_level`로 맞췄다.
  - 현재 속도와 목표 속도를 기준으로 실행 계획을 만들고, fan 상태 메시지와 JSON 결과 payload helper를 같이 넣었다.

### 2. 환기팬 제어 노드 추가

- `agribot_ws/src/agribot_iot/agribot_iot/fan_controller_node.py`
  - `/iot/commands/dispatch`에서 `device_type=fan` 명령만 소비하도록 만들었다.
  - fan 속도는 즉시 바뀌게 두고, `/iot/device_state`는 주기적으로 계속 발행한다.
  - `speed_level`에는 현재 속도 단계를 넣고, `current_value`에는 실행 시간을 초 단위로 넣었다.
  - 즉, 외부에서는 아래처럼 읽을 수 있다.
    - 현재 속도: `speed_level`
    - 누적 실행 시간: `current_value`, `unit=sec`
  - 실행 중 새 명령이 들어오면 이전 실행은 `INTERRUPTED` 결과로 정리한다.
  - `turn_off_fan`이 들어오면 이전 실행 시간까지 포함해 `STOPPED` 결과를 남긴다.

### 3. 장치 매핑과 실행 설정 보강

- `agribot_ws/src/agribot_iot/agribot_iot/device_mapping.py`
  - fan capability를 읽기 위한 `default_speed_level`, `max_speed_level` 필드를 추가했다.
- `agribot_ws/src/agribot_iot/config/iot_devices.yaml`
  - `farm_01_fan`에 지원 명령 목록을 보강했다.
  - 기본 속도 단계와 최대 속도 단계를 설정했다.
- `agribot_ws/src/agribot_iot/config/fan_controller.yaml`
  - command/state/result topic과 상태 발행 주기를 파라미터로 분리했다.
- `agribot_ws/src/agribot_iot/launch/fan_controller.launch.py`
  - 단독 fan 제어 노드 실행용 launch를 추가했다.
- `agribot_ws/src/agribot_iot/setup.py`
  - `fan_controller_node` entry point를 등록했다.

### 4. MQTT 계약 문서와 테스트 보강

- `docs/iot_mqtt_contract.md`
  - 문서 범위를 `S14P-506`까지 확장했다.
  - fan 명령 종류와 fan 상태에서 `speed_level/current_value(unit=sec)` 의미를 명시했다.
- `agribot_ws/src/agribot_iot/test/test_fan_controller_logic.py`
  - `set_fan_level`
  - `turn_on_fan`
  - `turn_off_fan`
    세 흐름의 계약을 검증하는 테스트를 추가했다.

## 설계 의도

- fan은 커튼처럼 점진적 위치 이동이 있는 장치가 아니라, 속도 단계가 즉시 바뀌는 장치로 모델링했다.
- 대신 티켓 핵심인 `실행 시간 기록`을 살리기 위해 상태 메시지에 run time을 넣었다.
- 그래서 fan 상태는 아래 두 축으로 분리했다.
  - 상태 값 `ON/OFF`
  - 속도 값 `speed_level`
- 로그는 상태 변화가 있을 때만 남기고, 토픽 발행은 계속 유지하도록 해서 런타임 노이즈를 줄였다.

## 결과적으로 무엇이 가능해졌는가

- 이제 `farm_01_fan` 장치에 대해 아래가 닫힌다.
  - 수동/자동 fan 명령 수신
  - 속도 단계 적용
  - 실행 중 시간 누적
  - 정지/중단 결과 발행
- control 레이어가 보내는 `set_fan_level` 명령도 그대로 IoT 실행 레이어에서 소비할 수 있게 됐다.

## 검증

- `python3 -m py_compile`
  - `device_mapping.py`
  - `fan_controller_logic.py`
  - `fan_controller_node.py`
- `pytest agribot_ws/src/agribot_iot/test`
  - 총 13개 테스트 통과
- `colcon build --packages-select agribot_iot --symlink-install` 통과
- `ros2 run agribot_iot fan_controller_node --ros-args -p use_sim_time:=false -p state_publish_hz:=2.0`
  - 노드 기동과 초기 `OFF` 상태 확인
- `ros2 topic pub --once /iot/commands/dispatch ... command_type=set_fan_level target_value=3 unit=level`
  - `/iot/device_state`에서 `state=ON`, `speed_level=3`, `current_value≈run_seconds` 확인
- `ros2 topic pub --once /iot/commands/dispatch ... command_type=turn_off_fan`
  - `/iot/device_state`에서 `state=OFF`, `current_value=누적 실행 시간`, `detail_message=Fan stopped after ...` 확인
- `ros2 topic echo --once /iot/command_result`
  - 비래치 토픽이라 구독 시점에 따라 놓칠 수 있었고,
  - 대신 fan 노드 로그에서 아래를 확인했다.
    - `Fan result published: command_id=fan-test-01, state=RUNNING`
    - `Fan result published: command_id=fan-test-02, state=STOPPED`

## 한계와 후속 작업

- 이번 MR은 환기팬 단독 제어 노드까지 닫은 단계다.
- 아직 다음은 후속 티켓에서 이어 붙이는 것이 맞다.
  - `S14P-507` 영양제 장치 제어 노드
  - `S14P-508` 장치 상태/실행 결과 통합 발행 경로 정리
  - backend에서 fan 실행 이력을 저장하는 경로
- 현재는 `farm_01` 단일 fan 장치 기준으로 검증했고, 다구역 fan 확장은 `iot_devices.yaml`에 같은 패턴으로 장치를 추가하면 된다.

## 커밋

- `S14P21A602-71 환기팬 제어 노드를 구현한다`

---

[feature/S14P21A602-72] 영양제 장치 제어 노드 구현

## 개요

`S14P-507` 요구사항에 맞춰 `agribot_iot` 패키지에 영양제 장치 제어 노드를 추가했습니다.

이번 MR의 목표는 다음 3가지입니다.

1. `IoTCommand(device_type=nutrient)`를 실제 실행 노드가 받을 수 있게 한다.
2. 영양제 종류, 양, 실행 요청자를 실행 계획과 결과 payload에 남긴다.
3. 기존 급수/커튼/환기팬 패턴과 같은 방식으로 상태 토픽과 실행 결과를 발행한다.

현재 control 레이어는 영양제 추천 시 `apply_nutrient_recipe` 명령과 `payload=nutrient_type=...` 형식의 reason 문자열을 이미 만들고 있으므로, 이번 MR은 그 계약을 IoT 실행 레이어에서 실제로 소비하는 데 초점을 맞췄습니다.

## 이번 MR에서 반영한 내용

### 1. 영양제 실행 계획 로직 추가

- `agribot_ws/src/agribot_iot/agribot_iot/nutrient_controller_logic.py`
  - `NutrientExecutionPlan`을 정의했습니다.
  - `apply_nutrient_recipe` 명령만 허용하도록 했습니다.
  - `reason` 문자열에서 `payload=nutrient_type=calcium_boost` 형태를 파싱해 `nutrient_type`을 추출하도록 했습니다.
  - `requested_by`를 실행 계획에 그대로 보존하도록 했습니다.
  - `target_value <= 0`인 경우는 거부하도록 했습니다.
  - `iot_devices.yaml`의 `flow_rate_per_sec`를 사용해 투여 시간을 계산하도록 했습니다.
  - 결과 JSON payload에 아래 필드를 포함하도록 했습니다.
    - `requested_by`
    - `nutrient_type`
    - `planned_duration_sec`
    - `executed_duration_sec`

### 2. 영양제 제어 노드 추가

- `agribot_ws/src/agribot_iot/agribot_iot/nutrient_controller_node.py`
  - `/iot/commands/dispatch`에서 `nutrient` 명령을 구독합니다.
  - 명령 승인 후 장치 상태를 `DISPENSING -> IDLE`로 전환합니다.
  - 실행 중 새 영양제 명령이 들어오면 이전 실행은 `INTERRUPTED` 결과로 정리합니다.
  - 실행 상태는 `IoTDeviceState`, 결과는 기존 패턴대로 JSON `std_msgs/String`으로 발행합니다.
  - 로그에 `nutrient_type`, `requested_by`, `target_value`, `duration`을 남기도록 했습니다.

### 3. 패키징/설정/런치 추가

- `agribot_ws/src/agribot_iot/setup.py`
  - `nutrient_controller_node` entry point를 등록했습니다.
- `agribot_ws/src/agribot_iot/config/nutrient_controller.yaml`
  - command/state/result topic과 zone filter 파라미터를 분리했습니다.
- `agribot_ws/src/agribot_iot/launch/nutrient_controller.launch.py`
  - 단독 실행 가능한 launch를 추가했습니다.
- `agribot_ws/src/agribot_iot/config/iot_devices.yaml`
  - `farm_01_nutrient` 장치에 `flow_rate_per_sec: 90.0`을 추가해 amount 기반 duration 계산이 가능하도록 했습니다.

### 4. 테스트 추가

- `agribot_ws/src/agribot_iot/test/test_nutrient_controller_logic.py`
  - `apply_nutrient_recipe` 명령이 payload에서 `nutrient_type`을 추출하는지 검증했습니다.
  - amount와 유량으로 실행 시간이 계산되는지 검증했습니다.
  - `requested_by`와 `nutrient_type`이 결과 payload에 포함되는지 검증했습니다.
  - 0 ml 요청은 거부되는지 검증했습니다.

## 설계 의도

- 영양제는 문서 기준으로도 `승인 후 실행`이 원칙이라, 자동 실행 분기보다 `실행 노드가 승인된 명령을 정확히 소비하고 로그를 남기는 것`이 우선이라고 판단했습니다.
- `IoTCommand`에 별도 payload 필드가 아직 없기 때문에, control에서 이미 사용 중인 `reason + payload=...` 규칙을 그대로 받아들이는 쪽이 현재 저장소와 가장 호환성이 높았습니다.
- 급수 노드와 같은 amount 기반 장치이므로, `flow_rate_per_sec`를 재사용해 투여 시간을 계산하도록 했습니다. 이렇게 하면 장치 상태와 결과 메시지의 의미가 더 분명해집니다.

## 결과적으로 가능해진 것

- 이제 `farm_01_nutrient` 장치에 대해 아래 흐름이 닫힙니다.
  - 승인된 영양제 명령 수신
  - 영양제 종류 파싱
  - 양 기준 실행 시간 계산
  - `DISPENSING` 상태 발행
  - `COMPLETED` 또는 `INTERRUPTED` 결과 발행
- control 레이어가 만드는 `apply_nutrient_recipe` 요청을 IoT 레이어가 실제 장치 실행 이벤트로 바꿀 수 있게 됐습니다.

## 검증

- `python3 -m py_compile`
  - `agribot_ws/src/agribot_iot/agribot_iot/nutrient_controller_logic.py`
  - `agribot_ws/src/agribot_iot/agribot_iot/nutrient_controller_node.py`
  - `agribot_ws/src/agribot_iot/launch/nutrient_controller.launch.py`
- `pytest`
  - `agribot_ws/src/agribot_iot/test/test_nutrient_controller_logic.py`
  - `agribot_ws/src/agribot_iot/test/test_watering_controller_logic.py`
  - `agribot_ws/src/agribot_iot/test/test_curtain_controller_logic.py`
  - `agribot_ws/src/agribot_iot/test/test_fan_controller_logic.py`
  - `agribot_ws/src/agribot_control/test/test_environment_disease_rules.py`
  - `agribot_ws/src/agribot_control/test/test_actuation_request_planner.py`
  - `agribot_ws/src/agribot_control/test/test_manual_actuation_safety.py`
  - 총 `25 passed`
- `colcon build --packages-select agribot_iot --symlink-install`
  - 빌드 성공

## 한계와 후속 작업

- 현재 `nutrient_type`은 `IoTCommand.reason`의 `payload=` 구문에서 파싱합니다. 장기적으로는 별도 메시지 필드나 typed result 메시지로 승격하는 편이 맞습니다.
- 이번 MR은 영양제 실행 노드까지만 닫았습니다.
  - review queue를 `/dispatch`로 넘기는 승인 브리지
  - backend의 영양제 실행 이력 저장
  - dashboard 승인/실행 UI
  는 후속 작업으로 남아 있습니다.

## 커밋

- `731b6f7` S14P21A602-72 영양제 장치 제어 노드를 구현한다

---

[feature/S14P21A602-73] 장치 상태·실행 결과 발행 구현

## 개요

`S14P-508` 요구사항에 맞춰, 각 IoT 장치가 발행하는 상태와 실행 결과를 한 번에 올리고 검증할 수 있는 통합 경로를 추가했습니다.

이번 MR의 목표는 다음 3가지입니다.

1. 급수, 커튼, 환기팬, 영양제 장치의 상태/결과 발행 노드를 하나의 launch로 함께 띄울 수 있게 한다.
2. MQTT bridge가 장치 상태는 retain, 실행 결과는 raw JSON으로 내보내는 계약을 테스트로 고정한다.
3. 기본 시뮬레이션 bringup에서도 IoT status/result publishing stack을 선택적으로 함께 올릴 수 있게 한다.

기존 `S14P-504`부터 `S14P-507`까지에서 각 장치 노드는 이미 `IoTDeviceState`와 `/iot/command_result`를 발행하고 있었습니다.  
이번 MR은 그 개별 구현을 `S14P-508` 관점에서 "통합 실행 + MQTT 전달 + 검증 가능" 상태로 정리하는 데 초점을 맞췄습니다.

## 이번 MR에서 반영한 내용

### 1. IoT status pipeline 통합 launch 추가

- `agribot_ws/src/agribot_iot/launch/iot_status_pipeline.launch.py`
  - 다음 launch를 한 번에 포함하는 통합 entrypoint를 추가했습니다.
    - `environment_sensor.launch.py`
    - `watering_controller.launch.py`
    - `curtain_controller.launch.py`
    - `fan_controller.launch.py`
    - `nutrient_controller.launch.py`
    - `mqtt_bridge.launch.py`
  - 외부에서 공통으로 넘길 수 있는 인자는 아래 2개로 제한했습니다.
    - `use_sim_time`
    - `force_log_only`

즉, 장치 상태와 실행 결과를 보는 데 필요한 최소 IoT 런타임을 이제 한 launch로 올릴 수 있습니다.

### 2. MQTT bridge launch 재사용성 보강

- `agribot_ws/src/agribot_iot/launch/mqtt_bridge.launch.py`
  - 기존에는 `force_log_only`를 launch 레벨에서 제어할 수 없었습니다.
  - 이번 MR에서 `force_log_only` launch argument를 추가해,
    - 브로커가 없는 개발 환경
    - 시뮬레이션 상태 점검 환경
    - 실제 MQTT 연결 환경
    를 같은 launch 파일로 다룰 수 있게 했습니다.

### 3. 기본 simulation bringup에 IoT stack 포함

- `agribot_ws/src/agribot_bringup/launch/simulation.launch.py`
  - 기존 TODO였던 IoT launch 부분을 실제 포함하도록 변경했습니다.
  - 새 인자:
    - `use_iot`
    - `mqtt_force_log_only`
  - `use_iot=true`이면 `agribot_iot/launch/iot_status_pipeline.launch.py`를 포함해, 기본 시뮬레이션에서도 IoT 상태/결과 발행 경로를 함께 올릴 수 있게 했습니다.

이 변경으로 `simulation.launch.py`는 더 이상 "Nav2까지만"이 아니라, 장치 상태/실행 결과 발행 stack까지 포함할 수 있는 상위 진입점이 되었습니다.

### 4. MQTT 계약 테스트 보강

- `agribot_ws/src/agribot_iot/test/test_mqtt_contract.py`
  - `/iot/device_state` route가 retain publish인지 검증하도록 추가했습니다.
  - `/iot/command_result` route가 `raw_json` serializer를 사용하는지 검증하도록 추가했습니다.
  - nutrient 실행 결과 예시 payload를 raw JSON serializer가 그대로 유지하는지 검증하도록 추가했습니다.

이 테스트는 backend가 이후 `agribot/iot/device_state`와 `agribot/iot/command_result`를 구독할 때 기대할 수 있는 최소 계약을 고정하는 역할을 합니다.

### 5. launch smoke test 추가

- `agribot_ws/src/agribot_iot/test/test_iot_status_pipeline_launch.py`
  - IoT status pipeline launch가 실제로 6개의 하위 launch를 포함하는지 검증합니다.
- `agribot_ws/src/agribot_bringup/test/test_simulation_launch.py`
  - simulation launch가 IoT 관련 인자를 선언하고 IoT pipeline을 포함하는지 검증합니다.

## 설계 의도

- `S14P-508`의 핵심은 새 장치 로직을 하나 더 만드는 것이 아니라, 이미 구현된 장치 노드들의 상태/결과 발행을 외부에서 안정적으로 소비할 수 있게 만드는 데 있습니다.
- 그래서 이번 MR은 per-device controller 로직을 다시 건드리기보다,
  - launch 진입점
  - MQTT bridge 실행 옵션
  - 계약 테스트
  를 정리하는 쪽으로 범위를 잡았습니다.
- 특히 `force_log_only`를 launch argument로 노출한 이유는, 로컬에서 MQTT broker가 없더라도 같은 실행 경로를 유지한 채 상태/결과 발행을 점검할 수 있게 하려는 목적입니다.

## 결과적으로 가능해진 것

- 이제 IoT status/result stack은 아래 흐름으로 한 번에 올릴 수 있습니다.
  - 환경 센서
  - 급수/커튼/환기팬/영양제 제어 노드
  - MQTT bridge
- `simulation.launch.py`에서도 옵션 하나로 이 stack을 포함할 수 있습니다.
- MQTT 계약상 아래 2개가 테스트로 고정됐습니다.
  - 장치 상태: retain publish
  - 실행 결과: raw JSON publish

## 검증

- `python3 -m py_compile`
  - `agribot_ws/src/agribot_iot/launch/mqtt_bridge.launch.py`
  - `agribot_ws/src/agribot_iot/launch/iot_status_pipeline.launch.py`
  - `agribot_ws/src/agribot_bringup/launch/simulation.launch.py`
  - `agribot_ws/src/agribot_iot/test/test_mqtt_contract.py`
  - `agribot_ws/src/agribot_iot/test/test_iot_status_pipeline_launch.py`
  - `agribot_ws/src/agribot_bringup/test/test_simulation_launch.py`
- `pytest`
  - `test_device_mapping.py`
  - `test_environment_sensor_profile.py`
  - `test_mqtt_contract.py`
  - `test_watering_controller_logic.py`
  - `test_curtain_controller_logic.py`
  - `test_fan_controller_logic.py`
  - `test_nutrient_controller_logic.py`
  - `test_iot_status_pipeline_launch.py`
  - `test_simulation_launch.py`
  - 총 `19 passed`
- `colcon build --packages-select agribot_iot agribot_bringup --symlink-install`
  - 빌드 성공

## 한계와 후속 작업

- 이번 MR은 장치 상태·실행 결과를 "발행하고 전달하는 경로"를 닫은 단계입니다.
- backend가 실제로 `agribot/iot/device_state`, `agribot/iot/command_result`를 저장하고 WebSocket으로 fan-out하는 부분은 후속 작업입니다.
- 현재 command result는 `std_msgs/String + JSON` 형식을 유지하고 있습니다. 장기적으로는 typed result message로 분리하는 편이 더 안전합니다.

## 커밋

- `4bc8e24` S14P21A602-73 장치 상태·실행 결과 발행을 구현한다

---

## [S14P21A602-100] 프론트엔드 UI '동물의 숲' 스타일 개편

### MR 제목

프론트엔드 UI '동물의 숲' 스타일 개편

### 작업 브랜치

- `S14P21A602-100`

### 이번 작업에서 해결하려고 한 문제

- 기존 프론트는 기본 화면은 갖춰져 있었지만 시각 톤이 분산돼 있어 하나의 제품처럼 보이기 어려웠고, 발표나 시연 기준으로는 기술 데모 느낌이 더 강했습니다.
- 문서와 API 초안에서 이미 전제한 로봇 제어, 구역 요약, 작물 확인, 수확 요청, 설비 추천, 알림 확인 같은 운영 흐름이 화면에서 충분히 드러나지 않았고, 어떤 패널이 실연동인지 어떤 패널이 fallback 또는 placeholder인지 설명하기도 어려웠습니다.
- 실행 문서 역시 개인 절대 경로와 과거 실행 순서를 기준으로 남아 있어, 팀원이 같은 방식으로 화면과 시뮬레이션을 재현하기 어렵고 발표 준비 시 화면, 문서, 실제 런치 구조가 서로 어긋나는 문제가 있었습니다.

### 왜 이렇게 정리했는가

- 이번 MR의 목표는 단순한 스타일 변경이 아니라, 발표용 완성도와 운영 화면으로서의 설명력을 함께 끌어올리는 것이었습니다.
- 그래서 시각적으로는 화이트, 그린, 세이지, 토마토 포인트를 중심으로 한 '동물의 숲' 감성의 친환경적이고 부드러운 톤으로 UI를 재구성하되, 카드 중심 레이아웃과 정보 밀도는 유지해 관제 화면의 가독성을 잃지 않도록 했습니다.
- 기능적으로는 문서와 API 명세를 기준으로 빠져 있던 운영 패널과 액션을 보강하고, 백엔드가 아직 비어 있는 구간도 fallback/adapter 계층으로 먼저 연결해 화면 구조를 미리 안정화했습니다.
- 발표 환경에서는 같은 화면을 그대로 쓰면서 구현 상태를 투명하게 설명할 수 있어야 했기 때문에, 개발자 오버레이를 추가하고 이후에는 엔드포인트 계약과 응답 상태를 바탕으로 자동 판별하도록 리팩토링했습니다.
- 실행 문서도 저장소 루트 상대 경로와 현재 `simulation.launch.py` 중심의 통합 실행 흐름으로 정리해, 화면, 문서, 시연 절차가 같은 기준을 보도록 맞췄습니다.

### 주요 변경 내용

- `frontend`
  - 전체 관제 화면을 '동물의 숲' 감성의 화이트·그린 중심 톤으로 재정리하고, 카드, 배지, 버튼, 헤더, 사이드바, 모바일 도크를 일관된 디자인 시스템으로 통합
  - 대시보드, 로봇, 작물, 설비, 수확, 알림 화면에 운영 흐름상 필요한 패널과 액션을 보강
  - `zones`, `alerts`, `robot/commands`, `missions/harvest`, `actuations/history`, `actuations/recommendations` 등을 읽고 호출할 수 있도록 fallback/adapter 계층 확장
  - 병해, 수확, 카메라 프리뷰, 온실 전경 등은 발표용 샘플 자산으로 보강해 시각 설득력 강화
- 개발자 오버레이
  - 패널별 상태를 `실연동`, `샘플 표시`, `혼합 상태`, `계약 확인`, `미구현` 관점에서 설명하는 오버레이 추가
  - 이후 상태 텍스트 하드코딩을 줄이기 위해 OpenAPI 계약, 실제 응답 source, 액션 성공 여부를 기준으로 자동 판별하도록 리팩토링
  - 같은 화면 위에서 발표용 UI와 개발자 설명 레이어를 함께 운영할 수 있도록 정리
- 실행 문서
  - `/home/ssafy/...` 형태의 개인 절대 경로를 제거하고 `git rev-parse --show-toplevel` 기반 저장소 루트 상대 경로로 통일
  - 기본 실행 절차를 현재 `simulation.launch.py` 중심의 통합 런치 흐름에 맞게 재정리
  - 실제 참조되는 패키지에 맞춰 빌드 범위를 조정하고, 종료 절차도 IoT 관련 프로세스까지 포함하도록 보완

### 기대 효과

- 프론트 화면이 더 친근하고 일관된 인상을 주면서도, 발표와 시연에서 실제 운영 서비스처럼 보이는 완성도를 확보할 수 있습니다.
- 사용자는 대시보드에서 상황을 보고, 로봇 제어, 작물 확인, 수확 요청, 설비 승인, 알림 확인으로 이어지는 운영 흐름을 더 자연스럽게 이해할 수 있습니다.
- 백엔드가 아직 부분 구현 상태여도 fallback 데이터와 개발자 오버레이 덕분에 구조와 구현 현황을 함께 설명할 수 있습니다.
- 팀원은 clone 위치와 상관없이 같은 실행 문서를 따라 현재 코드 기준의 런치 흐름을 재현할 수 있습니다.

### 검증 내용

- `cd /home/ssafy/SSAFY/S14P21A602/frontend && npm run build`
  - TypeScript + Vite 빌드 통과
- 실행 문서와 실제 런치/패키지 정의 정적 대조
  - `agribot_ws/docs/실행명령.md`
  - `docs/현 프로젝트 진행 상황 및 명령어 모음.md`
  - `agribot_bringup/launch/simulation.launch.py`
  - `agribot_iot/launch/iot_status_pipeline.launch.py`
- 개발 모드 기준 오버레이 설계 검토
  - 패널 계약과 응답 source, 액션 성공 여부를 조합해 상태가 자동 계산되도록 확인

### 아직 남아 있는 리스크

- 현재 백엔드 라우터의 상당수는 아직 placeholder 응답이라, 일부 패널은 실제 운영 데이터가 아니라 fallback 데이터로 보입니다.
- 개발자 오버레이의 자동 판별은 계약과 응답 구조를 잘 보여주지만, 비즈니스 로직의 완성도까지 100% 보장하지는 않습니다.
- 발표용 샘플 이미지는 실제 카메라, 비전, 실시간 스트림이 연결되면 교체 또는 제거가 필요합니다.

---

## [S14P21A602-101] 로봇 관제 지도에 실제 맵, 위치, semantic 레이어를 연결

### MR 제목

로봇 관제 지도에 실제 맵, 위치, semantic 레이어를 연결

### 작업 브랜치

- `S14P21A602-101`

### 이번 작업에서 해결하려고 한 문제

- 기존 `/robot` 페이지 지도 영역은 실제 농장 맵이 아니라 가짜 SVG 경로와 고정 마커를 보여주는 수준이라, 현재 로봇이 어디에 있는지 믿고 보기 어려웠고 온실 구조나 운영 대상을 이해하기도 어려웠습니다.
- 백엔드에는 정적 occupancy map과 현재 pose를 프론트가 읽을 수 있는 API가 없었고, 프론트 역시 semantic layer 계약이 없어 식물, 급수 포인트, 현재 목표 같은 운영 정보를 지도 위에 함께 보여주지 못했습니다.
- 발표와 시연 관점에서도 RViz 화면을 그대로 노출하는 것보다, 실제 서비스형 관제 UI처럼 정적 맵 위에 현재 위치와 운영 대상 레이어를 얹는 구조가 더 자연스럽고 설명력이 높았습니다.

### 왜 이렇게 정리했는가

- 이 프로젝트의 관제 화면은 RViz를 프론트에 그대로 옮기기보다, 정적 map 이미지 위에 robot pose와 의미 있는 운영 자산을 얹는 서비스형 UI 구조가 더 적합했습니다.
- 그래서 이번 MR은 백엔드, ROS, 프론트를 한 번에 묶어 정적 맵 API, pose 스냅샷, PGM 렌더링, semantic 레이어를 연결하는 방향으로 정리했습니다.
- 현재 백엔드에 semantic layer API가 아직 없기 때문에, 프론트에서는 world/config 기반 fallback adapter를 먼저 두고 이후 실제 API가 생기면 데이터만 교체할 수 있게 설계했습니다.
- 위치 갱신은 우선 1초 polling으로 시작하되, 이후 WebSocket이나 MQTT 브리지를 붙여도 구조를 크게 바꾸지 않도록 기본 계약을 먼저 만드는 데 집중했습니다.

### 주요 변경 내용

- `backend`
  - `backend/routers/robots.py`에 `/api/v1/robot/map`, `/api/v1/robot/map/raw`, `/api/v1/robot/pose`, 구조화된 `/api/v1/robot/status`를 추가
  - `/api/v1/zones`도 현재 프론트 동선에 맞는 구역 목록으로 정리
  - `backend/main.py`에서 프론트 직접 호출을 위한 CORS를 허용하고, DB가 아직 준비되지 않은 환경에서도 로봇 관제 API가 기동되도록 초기화 실패를 비치명적으로 처리
- `agribot_ws`
  - `robot_pose_snapshot.py` 노드를 추가해 `map -> base_link` 또는 `odom` 기준 pose와 속도, timestamp를 스냅샷 파일로 기록
  - `simulation.launch.py`에 해당 노드를 포함시켜 기존 실행 흐름을 바꾸지 않아도 관제용 좌표가 자동 기록되도록 구성
- `frontend`
  - `frontend/src/lib/api/agribot.ts`에 `RobotMapData`, `RobotPoseData`, `/robot/map`, `/robot/pose` query와 API URL resolver 추가
  - `frontend/src/lib/maps/pgm.ts`에 PGM 파서를 추가해 occupancy map 원본을 브라우저에서 직접 렌더링 가능하게 구성
  - `frontend/src/components/robot-map-canvas.tsx`에서 정적 맵, 로봇 heading, live/fallback 상태, 마지막 갱신 시각을 표시
  - `frontend/src/lib/robot-map/farm-semantic-map.ts`, `frontend/src/components/robot-facility-map.tsx`를 통해 식물, 급수 포인트, guide, 선택 패널, 레이어 토글을 포함한 semantic map 레이어 구성
  - `frontend/src/pages/map-control-page.tsx`에서 기존 가짜 오버레이를 제거하고 실제 map query, pose query, semantic layer, zoom control, 선택 패널을 연결
  - `frontend/src/index.css`에서 지도 캔버스, marker, 범례, notice, semantic 레이어 UI 스타일 정리

### 기대 효과

- Gazebo에서 로봇이 움직이면 `/robot` 페이지 지도 위 marker도 함께 움직이는 구조가 갖춰지고, 관제 화면이 RViz 복사본이 아니라 실제 서비스형 UI에 가까운 형태로 바뀝니다.
- 온실 구조, 현재 로봇 위치, 목표 식물, 급수 포인트를 한 화면에서 함께 보여줄 수 있어 발표와 운영 설명의 설득력이 높아집니다.
- 프론트 구조상 이미 semantic layer renderer가 준비되므로, 이후 backend가 semantic map API를 제공하면 화면을 다시 갈아엎지 않고 데이터만 교체할 수 있습니다.
- 현재는 1초 polling이지만, 이후 WebSocket이나 MQTT 기반 실시간 갱신으로 확장하기 쉬운 기본 계약이 마련됩니다.

### 검증 내용

- `python3 -m compileall /home/ssafy/SSAFY/S14P21A602/backend /home/ssafy/SSAFY/S14P21A602/agribot_ws/src/agribot_bringup`
  - 통과
- `cd /home/ssafy/SSAFY/S14P21A602/agribot_ws && source /opt/ros/jazzy/setup.bash && colcon build --packages-select agribot_bringup --symlink-install`
  - 통과
- `cd /home/ssafy/SSAFY/S14P21A602/frontend && npm run build`
  - TypeScript + Vite 빌드 통과
- 런타임 API 호출 검증
  - 현재 셸 환경에는 `fastapi` 패키지가 설치돼 있지 않아 `TestClient` 기반 실제 응답 호출 검증은 수행하지 못했고, 우선 Python 문법, ROS 패키지 빌드, 프론트 빌드 기준으로 구조적 오류를 점검

### 아직 남아 있는 리스크

- 현재 위치 갱신은 1초 polling 방식이라, 추후 WebSocket이나 MQTT push 방식에 비해 반응성이 아주 높지는 않습니다.
- `map -> base_link` TF가 아직 안정되지 않은 초반 구간에는 fallback 좌표가 유지될 수 있어, localization 초기 몇 초 동안은 실제 위치 반영이 늦을 수 있습니다.
- semantic 레이어는 아직 frontend fallback adapter 기반이라 실시간 운영 데이터와 완전히 동기화된 상태는 아닙니다.
- 물리 설비 관점에서는 `farm_01_watering` 같은 논리 장치와 지도에 그린 개별 sprinkler 포인트 사이의 모델링 차이를 이후 backend/agribot_ws 계약에서 정리할 필요가 있습니다.

