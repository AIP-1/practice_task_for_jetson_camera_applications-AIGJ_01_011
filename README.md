# practice_task_for_jetson_camera_applications — team4 (AIGJ_01_011)

AIP-1 「Jetson 카메라 응용 실습」 결과.

팀 과제 (팀 Jetson 1대 · 웹캠 2대 · IP 카메라).

이 저장소는 팀 Jetson 작업 폴더 사본에서 사람이 찍힌 사진·영상과 대용량 파일을 뺀 것이다.

MQTT 토픽 접두어와 홈 경로에 있던 계정 이름은 `team4`, `<user>`로 바꿔 적었다.

## 폴더

| 폴더 | 과제 | 주요 결과 |
|---|---|---|
| `P_check.md` | 준비 확인 P1–P8 | GPU·키트 상태 진단 |
| `t0/` | 1 카메라 연동 | `camlib.py`(최신 1장 리더, 재연결), `probe.py`, `view.py`, `latency.py`, `reconn.py` → [runs/test_log.md](runs/test_log.md) |
| `t2/` | 2 물체 인식 | 벤치 [t2_bench.md](runs/t2_bench.md), 조명×거리 [t2_conf.md](runs/t2_conf.md), 선 통과 [t2_line.md](runs/t2_line.md), 개방 어휘 [t2_items.md](runs/t2_items.md) |
| `t3/` | 3 재실 인원 | `occupancy.py`, 구역 `zones/room.json`, 평가 스크립트, 결과 [t3_eval_final_output.md](runs/t3_eval_final_output.md) |
| `t4/` | 4 행동 패턴 | `pose_app.py` → [t4_actions.md](runs/t4_actions.md), [t4_handshake.md](runs/t4_handshake.md), [t4_behavior.md](runs/t4_behavior.md), [t4_joints.md](runs/t4_joints.md) |
| `t5/` | 5 거리 측정 | 보정 [t5_calib_report.md](runs/t5_calib_report.md) + `runs/t5_calib_files/`, 실측 [t5_dist_report.md](runs/t5_dist_report.md), 실패 조건 [t5_fail_report.md](runs/t5_fail_report.md), 사람 거리 [t5_person_report.md](runs/t5_person_report.md), 가상 영상 `runs/t5_synth*/result.json` |
| `t6/` | 6 응용 서비스 | `service.py`(재실 인원 A1 + 근접 경고 A4), `t6-vision.service`, [t6/README.md](t6/README.md), [t6/PLAN.md](t6/PLAN.md), 12시간 [t6_12h_summary.md](runs/t6_12h_summary.md) |
| `b1/`, `b2/` | 심화 | NVDEC vs CPU 복원 [b1_decode.md](runs/b1_decode.md), RTSP 송출 → 검출 [b2_rtsp.md](runs/b2_rtsp.md) |

## 확인 기준 점검 (README 기준, 기록으로 확인한 것만)

| 과제 | 기준 | 기록 | 판정 |
|---|---|---|---|
| 1 | T0-1~T0-6 합격 | test_log.md: T0-1·2·3·4·5·6 모두 합격 | ✅ |
| 1 | (a)/(b) 2분 뒤 지연 차이와 원인 | 웹캠 약 0.8 s, IP 카메라 약 103 s, 수신 버퍼 차이로 설명 | ✅ |
| 1 | 재연결 30 s 안, 재시작 없이 | 웹캠 33.0 s, IP 카메라 3 s 타임아웃 적용 후 약 32 s (프로세스 유지) | ⚠️ 30 s를 약간 넘음 |
| 2 | FP16 ≤ 10 ms, 카메라 FPS 이상 | 4.0 ms, 전체 64 fps | ✅ |
| 2 | 선 통과 10회 오차 ≤ 1 | IN 5 / OUT 5 그대로, 오차 0 | ✅ |
| 2 | 벤치 4행 + INT8 mAP 하락 | 4행 완성, INT8 −0.023 (−4.5%) | ✅ |
| 2 | 개방 어휘 5종 중 4종 | YOLOE-11l 4/5 (11s는 2/5) | ✅ |
| 3 | MAE ≤ 1.0, ±1 ≥ 80%, 빈 방 오판 ≤ 10% | 최선 30s p75 MAE 1.40, ±1 47%. 빈 방 정답 장면 없음 | ❌ |
| 3 | 편향 원인 사례 설명 | 과소 계수(bias −1.1~−1.6)만 확인, 사례별 분류 문서 없음 | ❌ |
| 3 | 출입 계수 vs 직접 계수 30분 동시 운영 | `t3_door_1030*` 기록은 있으나 비교 문서 없음 | ❌ |
| 4 | 안정화 10회 중 9회, 깜빡임 ≤ 1 | 만세·한 팔 10/10, 깜빡임 0 (손 흔들기 깜빡임 9 — 원인 설명 있음) | ✅ |
| 4 | 손 흔들기 vs 한 팔 정지 (오판 ≤ 1) | 오검출 0/10 | ✅ |
| 4 | 행동 패턴 12회 중 10회 | 10/12 | ✅ |
| 4 | 혼동 사례 5건 이상 + 원인 | t4_behavior.md "혼동 사례", t4_actions/handshake | ✅ |
| 5 | 가상 영상 세 지점 ≤ 5% | joint 보정 −0.16 / −0.16 / +0.56 % (기본 보정은 4 m −8.6 %) | ✅ |
| 5 | 재투영 ≤ 0.5 px, 기준선 ±3 mm, 행 차이 ≤ 0.5 px | 0.144 px, 127.1 vs 127 mm, 평균 0.093 px | ✅ |
| 5 | 실측 1~2 m ≤ 5%, 3 m ≤ 8%, 4 m ≤ 12% + 예상 오차와 비교 | 6개 거리 모두 합격, 예측 ΔZ 열 있음 | ✅ |
| 5 | 측정 불가 조건 4가지와 대응 | 무늬 없음·역광·근접·빠른 움직임 | ✅ |
| 6 | 12시간 무중단 기록 | 12.00 h, 재시작 0, 재연결 0, GPU 평균 41%, RAM 3.1 GB, 9.9 W | ✅ |
| 6 | 카메라 1대 분리 시 나머지 정상, 복귀 시 재연결 | `t6_unplug_test.csv`, `t6_unplug_mqtt.log` 기록은 있으나 판정 문서 없음 | ⚠️ |
| 6 | 재부팅 후 3분 안 복귀 | 기록 없음 | ❌ |
| 6 | 결과 형식 문서(토픽·필드·단위·예시·주기) | t6/README.md·PLAN.md에 토픽만 있음, 1~2쪽 형식 문서 없음 | ❌ |
| 6 | 운영 모드 영상 저장 0건 | 12시간 새 이미지 0개, 저장 경로 목록 t6/README.md | ✅ |

최종 제출물 중 **기능 시험 기록(`runs/test_log.md`)**은 있다. **결과 보고서(과제 1~6 묶음), 확인 문제 답안, 결과 형식 문서, 발표 자료**는 아직 없다.

## 공개 저장소에서 뺀 것

- 사람·얼굴이 찍힌 사진과 스냅샷: `runs/t5_person/`, `runs/t5_fail/`, `runs/t3_snaps_*/`(정답 CSV만 `runs/t3_gt/`로 옮김), `runs/t2_conf_*.jpg`, `runs/ipcam_angle_latest.jpg`, `t0/runs/`, 보정 촬영 사진 `runs/t5_calib*/` 이미지, 그 밖의 모든 jpg·png
- 시험 영상(`b1/*.mkv`), 모델·엔진(`*.pt`, `*.onnx`, `*.engine`, `mobileclip_blt.ts`), MediaMTX 바이너리·TLS 키, 18 MB 수신 로그
- IP 카메라 계정(`~/.config/eg2/ipcam.env`) — `t0/camlib.py`가 이 파일 또는 환경변수 `IPCAM_USER`, `IPCAM_PW`에서 읽는다
