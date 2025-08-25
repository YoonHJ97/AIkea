# AIkea
AMR with Robotic Arm for Item Delivery 
# Pinky AMR Delivery Demo (Kiosk → AMR, with MG400)

> AMR(Pinky Violet)와 로봇팔(MG400)을 소켓 서버로 연동하고, YOLO+호모그래피로 좌표를 정합하여 **픽앤플레이스**를 수행합니다.
>
> Connects an AMR (Pinky Violet) and a robotic arm (MG400) via a socket server; uses YOLO and homography-based coordinate mapping for **pick-and-place**.

---

## 1. 개요 (Overview)

* 키오스크(서버)에서 **품목·장소**를 입력 → AMR 클라이언트가 호출되어 **자율주행** 수행
* 로봇팔(MG400)은 YOLO 객체 인식 + **Homography**(픽셀→로봇 좌표 변환)로 **정밀 집기/배치**
* 본 문서는 **시연 주행 테스트(Standard Demo Run)** 절차를 정리합니다.

> **Note**
>
> * ROS 2 **Jazzy** + Ubuntu 24.04 기준
> * 워크스페이스: `~/nav_ws` (모노레포, `src/`만 Git 커밋 권장)
> * 기본 서버 포트: `9997` (변경 시 클라이언트 파라미터와 일치 필요)

---

## 2. 시연 주행 테스트 (Standard Demo Run)

### 2.1 터미널 레이아웃

* **총 8개 터미널**을 띄워 **좌/우 반반** 배치

  * **오른쪽: `pinky_1` (b42a)** — `ROS_DOMAIN_ID=69`
  * **왼쪽: `pinky_2` (b256 또는 a934)** — `ROS_DOMAIN_ID=17`

### 2.2 SSH 접속 (각 pinky에 접속)

1. 각 pinky의 IP로 SSH 접속

**SSH가 안 될 때 체크리스트**

1. Pinky 전원이 켜졌는지 확인
2. Pinky IP 확인 절차

   * PC의 Wi‑Fi를 확인하고자 하는 Pinky의 SSID로 연결
   * 브라우저에서 `http://192.168.4.1:8888` 접속 → **Terminal** 진입
   * `\sbin\ifconfig` 실행 후 IP 확인

> 필요 시 스크린샷을 `docs/images/`에 추가하고, 본 README의 이미지 경로를 갱신하세요.

### 2.3 ROS\_DOMAIN\_ID 설정 (매 터미널 필수)

> **중요**: 환경변수 이름은 **`ROS_DOMAIN_ID`** 입니다. (일부 자료의 `ROS_DOMAIN`는 오기)

```bash
# 오른쪽 그룹 (pinky_1: b42a)
export ROS_DOMAIN_ID=69

# 왼쪽 그룹 (pinky_2: b256 또는 a934)
export ROS_DOMAIN_ID=17
```

### 2.4 Kiosk 서버 실행

* VS Code에서 `~/nav_ws/src/nav_pkg/nav_pkg` 폴더를 열고 **kiosk 서버 스크립트** 실행

  * 파일명이 환경마다 다를 수 있습니다: `kiosk_2.py` 또는 `kiosk.py`
  * CLI 대안(패키징 상태에 따라):

    ```bash
    source ~/nav_ws/install/local_setup.bash
    # 예) 엔트리포인트가 등록되어 있는 경우
    ros2 run nav_pkg kiosk
    # 또는 파이썬 모듈 실행 형태
    python3 -m nav_pkg.kiosk
    ```
* 서버 포트(예: **9997**)를 확인하고, 클라이언트 파라미터와 일치시킵니다.

### 2.5 각 터미널에서 Launch/Node 실행

> 아래 명령은 각 robot 그룹(왼쪽 pinky\_2 / 오른쪽 pinky\_1)에서 **각각 4개 터미널**에서 수행하는 예시입니다.

#### (왼쪽) pinky\_2 — 주행 노드

```bash
source ~/nav_ws/install/local_setup.bash
ros2 run nav_pkg pinky_client_2 --ros-args -p server_port:=9997
```

#### (왼쪽) pinky\_2 — 네비게이션 RViz

```bash
source ~/pinky_violet/install/local_setup.bash
ros2 launch pinky_navigation nav2_view.launch.xml
```

#### (왼쪽) pinky\_2 — Bringup

```bash
ros2 launch pinky_bringup bringup.launch.xml
```

#### (왼쪽) pinky\_2 — Navigation (지도: `pltt_2.yaml`)

```bash
ros2 launch pinky_navigation bringup_launch.xml map:=pltt_2.yaml
```

---

#### (오른쪽) pinky\_1 — 주행 노드

```bash
source ~/nav_ws/install/local_setup.bash
ros2 run nav_pkg pinky_client_2 --ros-args -p server_port:=9997
```

#### (오른쪽) pinky\_1 — 네비게이션 RViz

```bash
source ~/pinky_violet/install/local_setup.bash
ros2 launch pinky_navigation nav2_view.launch.xml
```

#### (오른쪽) pinky\_1 — Bringup

```bash
ros2 launch pinky_bringup bringup.launch.xml
```

#### (오른쪽) pinky\_1 — Navigation (지도: `pltt_map.yaml`)

```bash
ros2 launch pinky_navigation bringup_launch.xml map:=pltt_map.yaml
```

> **Map 주의**: 환경에 따라 `pltt_2.yaml` / `pltt_map.yaml` 명칭이 다를 수 있습니다. 실제 보유한 맵 파일명으로 교체하세요.

### 2.6 클라이언트 노드에서 로봇 ID 입력

* 주행 노드(위의 주행 노드 터미널)에서 입력 대기 시 **각각** 입력

  * 왼쪽: `pinky_2`
  * 오른쪽: `pinky_1`
* 서버와의 연결 메시지가 정상 출력되는지 확인

### 2.7 서버에서 명령 내리기 `<목적지> <가구>`

* 예: `카페 의자`, `레스토랑 소파` 등
* **동작 정책**: 한 Pinky의 동작이 끝날 때까지 대기 → 이후 다른 Pinky 호출


