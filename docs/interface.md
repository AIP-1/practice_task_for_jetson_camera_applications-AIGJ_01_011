# 결과 형식 문서 — team4 카메라 응용 서비스 (재실 인원 + 근접 경고)

다른 팀이 이 문서만 보고 구독·조회할 수 있도록 쓴다. 서비스 코드는 [`t6/service.py`](../t6/service.py), systemd 유닛은 [`t6/t6-vision.service`](../t6/t6-vision.service).

## 1. 무엇을 내놓나

| 기능 | 입력 | 출력 | 갱신 |
|---|---|---|---|
| A1 재실 인원 | IP 카메라 stream1 (2560x1440, 25 fps) → YOLO11n TensorRT FP16 960, 2x2 조각, 발 위치로 구역 판정 | 방 전체 인원 `count`, 구역별 인원 | 약 5회/초 추론, MQTT는 값이 바뀔 때 + 10초마다 |
| A4 근접 경고 | USB 웹캠 2대 스테레오(기준선 127 mm) → YOLO11n FP16 640 + SGBM | 가장 가까운 사람까지 거리(m)와 상태 | 약 8회/초, MQTT는 상태가 바뀔 때 + 60초마다 |

## 2. 접속

| 방법 | 주소 |
|---|---|
| MQTT | 팀 Jetson `1883` (익명, 발행은 서비스만) |
| REST | `http://<JETSON_IP>:8030` |

토픽 접두어 `<prefix>`는 서비스의 `--prefix` 값이다. 이 저장소의 기본값은 `team4/t6`이다. 실제 가동 중인 서비스의 접두어는 `/v1/health`의 로그나 `mosquitto_sub -t '#' -v -C 5`로 확인한다.

```bash
mosquitto_sub -h <JETSON_IP> -t 'team4/t6/#' -v          # 전부 구독 (retain 이라 접속 즉시 현재값을 받음)
curl -s http://<JETSON_IP>:8030/v1/occupancy
curl -s http://<JETSON_IP>:8030/v1/proximity
curl -s http://<JETSON_IP>:8030/v1/health
```

## 3. MQTT 토픽 (모두 JSON, UTF-8, QoS 1, retain)

| 토픽 | 언제 | 필드 |
|---|---|---|
| `<prefix>/occupancy` | 인원이 바뀔 때 + 10초마다 | `ts`(초, 유닉스 시각, 프레임을 받은 시각), `room`, `count`(명, 최근 10초 중앙값), `raw`(명, 이번 프레임), `zones`({구역: 명}, 이번 프레임), `capacity`(명), `window_s`(초) |
| `<prefix>/proximity` | 상태가 바뀔 때 + 60초마다 | `ts`, `state`, `nearest_m`(m, 측정 못 하면 null), `people`(검출된 사람 수), `near_m`, `far_m`(m, 기준값), `reason` |
| `<prefix>/status/<cam>` | 카메라가 끊기거나 돌아올 때 | `state`(`online`/`offline`), `ts` — `<cam>` = `ipcam`, `left`, `right` |
| `<prefix>/status/service` | 시작할 때 / 비정상 종료(LWT) | `state`(`online`/`offline`), `ts`, `ops`(운영 모드 여부) |

`state` 값

| 값 | 뜻 |
|---|---|
| `near` | 가장 가까운 사람 < 1.0 m |
| `mid` | 1.0 ~ 3.0 m |
| `far` | > 3.0 m |
| `none` | 사람 없음 |
| `unknown` | 사람은 있으나 거리 측정 불가(무늬 없음·유효 비율 30% 미만) 또는 웹캠 끊김 |
| `disabled` | A4를 끄고 실행함(`--no-stereo`) |

경계에서 흔들리지 않게 0.1 m 여유를 두고, 새 상태가 2초 이어져야 바꾼다. 웹캠이 끊기면 바로 `unknown`이 된다.

예시

```json
team4/t6/occupancy  {"ts": 1793390138.0, "room": "applab-classroom", "count": 5, "raw": 4, "zones": {"rear": 2, "middle": 2, "near": 0}, "capacity": 40, "window_s": 10}
team4/t6/proximity  {"ts": 1793345760.2, "state": "mid", "nearest_m": 2.84, "people": 1, "near_m": 1.0, "far_m": 3.0, "reason": ""}
team4/t6/status/left {"state": "offline", "ts": 1793345749.853}
```

## 4. REST (GET, JSON)

| 경로 | 내용 |
|---|---|
| `/v1/occupancy` | MQTT 필드 + `age_s`(값이 몇 초 전 프레임인지), `confidence`(셈에 들어간 사람들의 검출 신뢰도 평균, 0~1), `fps`, `infer_ms`, `camera`(온라인 여부·마지막 프레임 나이·재연결 횟수) |
| `/v1/proximity` | MQTT 필드 + `age_s`, `confidence`(가장 좋은 유효 비율), `people`(사람별 `id`, `z_m`, `valid`, `conf`), `skew_ms`(두 웹캠 프레임 시각 차), `cameras` |
| `/v1/health` | `status`(`ok` = 카메라 셋 다 온라인 / `degraded`), `uptime_s`, `mqtt`, `cameras`, 기능별 `age_s`·`fps` |

## 5. 쓰는 쪽이 지킬 것

- 제어(냉난방·조명)에는 `count`를 쓴다. `raw`는 한 프레임 값이라 가림 때문에 흔들린다. 원인 분석용으로 남겨 둔다.
- `age_s`가 10초를 넘거나 `status/<cam>`이 `offline`이면 값을 믿지 않는다.
- 측정 결과 `count`는 실제보다 **평균 1.1~1.6명 적게** 나온다(가림). 빈 방 판정에 쓸 때는 0이 5분 이상 이어질 때만 빈 방으로 본다.
- `unknown`은 "가깝지 않음"이 아니다. 경고 장치는 `unknown`을 별도로 처리한다.
- 운영 모드(`--ops`)에서는 영상·스냅샷·크롭을 저장하지 않고 화면 엔드포인트도 없다. 결과는 숫자뿐이다.
