# 🔬 기술 상세 노트 (Technical Deep-Dive)

> 공격 코드의 각 함수를 구현하기 위해 필요한 기술 지식과 설계 근거를 정리한 문서입니다.

---

## 1. `ts_10us()` — 타임스탬프 생성

### 필요 지식
- **MAVLink Signing Epoch**: `2015-01-01 00:00:00 UTC`부터 시작
- 타임스탬프 단위: **10μs** (마이크로초의 10배)
- FCU와의 시간 오차 허용치: 수백 ms 마진 필요

### 설계 근거
```python
# Unix Epoch → MAVLink Epoch 변환
MAVLINK_EPOCH = datetime(2015, 1, 1, tzinfo=timezone.utc).timestamp()
timestamp = int((time.time() - MAVLINK_EPOCH) * 1e5)  # 10μs 단위
```

---

## 2. `wait_hb()` — Heartbeat 대기

### 필요 지식
- `HEARTBEAT` 메시지 주기: **1Hz** (약 1초 간격)
- 타임아웃: 합리적인 값 설정 필요 (보통 3~5초)
- 링크 초기에 GCS나 다른 노드가 모드 전환을 방해할 수 있음

---

## 3. `enable_signing()` — 로컬 서명 활성화

### 필요 지식
- **pymavlink**의 `MAVLinkSigning` 필드:
  - `secret_key`: 32바이트 서명 키
  - `link_id`: 링크 식별자 (각 연결마다 고유)
  - `timestamp`: 10μs 단위 타임스탬프
  - `sign_outgoing`: `True`로 설정 시 송신 메시지에 서명 추가

### 주의사항
- 송신자 `SYS/COMP` ID를 변경하면 GCS와 **ID 충돌** 발생 가능
- 공격자의 `link_id`와 `source_system/source_component` 정책을 신중히 결정해야 함

---

## 4. `probe_signed()` — 서명 통신 검증

### 필요 지식
- `PARAM_REQUEST_READ`에 **서명이 필수**인 상황에서 성공하면 "서명 유효"로 간주
- FCU 응답 타입: `PARAM_VALUE` 또는 `AUTOPILOT_VERSION`
- 타임아웃/재시도 전략 필요 (FCU가 바쁠 수 있음)

### 설계 근거
서명이 적용된 상태에서 파라미터 읽기 요청 → 응답 수신 시 양방향 서명 통신이 정상적으로 동작함을 확인

---

## 5. `reinit_signing_on_fcu()` — FCU 키 주입

### 필요 지식
- `setup_signing_send` 메서드: pymavlink 버전에 따라 **메서드 유무가 다름**
- FCU 측 서명 테이블이 적용되는 타이밍: 부팅 직후 안정화 시간 필요
- 키 주입은 **DISARM 상태에서만 가능** (펌웨어 리버싱에서 확인)

### 설계 근거
```python
m.mav.setup_signing_send(
    TARGET_SYSTEM, TARGET_COMPONENT,
    list(key),     # 32바이트 키
    ts_10us()      # 현재 타임스탬프
)
```
`SETUP_SIGNING` 메시지는 미서명 상태에서도 수신 가능 → 이것이 취약점의 핵심

---

## 6. `ensure_signed()` — 서명 통신 확립

### 필요 지식
- 서명 타임스탬프 **리드/래그 마진** (예: +0.2s, +0.3s)의 의미
  - FCU와 공격자 간 시계 오차를 보정
  - 너무 과거/미래 타임스탬프는 거부됨
- 실패 시 FCU 재초기화 루틴의 안전성 (연결 중 메시지 flush)

### 설계 근거
2회의 시도 루프:
1. 먼저 기존 키로 서명 통신 가능한지 확인 (probe)
2. 실패 시 → 키 주입 → 서명 활성화 → 재확인

---

## 7. `set_param()` / `get_param()` — 파라미터 읽기/쓰기

### 필요 지식
- 파라미터 **정확한 이름/타입**: UINT8, INT16, REAL32 등
- 일부 보드에서 `PARAM_VALUE.param_id`가 **패딩 포함**으로 돌아옴 → 문자열 비교 시 주의
- 쓰기 후 즉시 readback이 **지연**될 수 있음 (얕은 readback)

### 주요 파라미터 타입
```
MAV_PARAM_TYPE_UINT8  = 1
MAV_PARAM_TYPE_INT16  = 4
MAV_PARAM_TYPE_REAL32 = 9  (가장 일반적)
```

---

## 8. `reboot()` — FCU 원격 재부팅

### 필요 지식
- `COMMAND_LONG` 명령 ID 246 = `MAV_CMD_PREFLIGHT_REBOOT_SHUTDOWN`
  - `param1 = 1`: 오토파일럿 재부팅
  - `param1 = 3`: 보드 + 오토파일럿 재부팅
- 재부팅 대기 시간: 부팅 완료까지 **수 초** 필요
- 재부팅 후 서명 키는 유지되지만, 재연결 + 서명 재확립 필수

---

## 9. `set_mode_stabilize()` — Stabilize 모드 설정

### 필요 지식
- ArduCopter **custom mode 번호**: `Stabilize = 0`
  - 펌웨어 버전에 따라 확인 필요
  - stabilize(0), acro(1), alt_hold(2), auto(3), guided(4), loiter(5)...
- `MAV_MODE_FLAG_CUSTOM_MODE_ENABLED` 플래그 사용 규칙
  - `SET_MODE` 메시지의 `base_mode`에 이 플래그를 포함해야 custom mode가 적용됨

---

## 10. `arm_cmd()` — ARM/DISARM 명령

### 필요 지식
- `MAV_CMD_COMPONENT_ARM_DISARM` 파라미터:
  - `param1 = 1`: ARM (시동)
  - `param1 = 0`: DISARM (정지)
  - `param2 = 21196`: **강제 DISARM 매직 넘버** (모든 체크 우회)
- `COMMAND_ACK` 응답 확인:
  - `result = 0 (MAV_RESULT_ACCEPTED)` → 성공
  - `result = 4 (MAV_RESULT_FAILED)` → 실패 (PreArm 체크 미통과 등)
- `STATUSTEXT` 해석: PreArm 실패 원인 코드 파악 가능
- DISARM 시 ACK가 **오지 않을 수도 있음** → 별도 상태 확인 필요

---

## 11. `print_diag()` — 진단 출력

### 필요 지식
- 점검할 파라미터들의 **의미/상호작용**:
  - `BRD_SAFETYENABLE` ↔ `BRD_SAFETY_DEFLT` ↔ `BRD_SAFETYOPTION`
  - `SERVO*_FUNCTION` ↔ `MOT_PWM_TYPE` ↔ `FRAME_CLASS`
- 출력만으로 위험 변경이 없도록 **읽기/쓰기 분리** 설계

---

## 12. `harden_outputs_and_safety()` — 공격 환경 설정

### 필요 지식
- **벤치 전용** 설정: Safety off, PreArm 우회(`ARMING_CHECK=0`), 모터 매핑
- ESC 규격에 따라 `MOT_PWM_TYPE`과 `BRD_PWM_COUNT`가 보드/펌웨어와 일치해야 함:
  - `MOT_PWM_TYPE = 0`: Normal PWM
  - `MOT_PWM_TYPE = 1`: OneShot
  - `MOT_PWM_TYPE = 4/5`: DShot150/DShot300

---

## 13. `rc_throttle()` / `rc_override_all_zero()` — RC 오버라이드

### 필요 지식
- `RC_CHANNELS_OVERRIDE`의 **채널 순서**: CH3 = 스로틀
- **수신기/FMU 믹스 우선순위**: RC Override가 수신기 입력보다 우선
- **PWM 스케일**: 1000μs(최소) ~ 2000μs(최대)
  - 1000μs = 모터 정지
  - 1500μs = 중간 출력
  - 2000μs = 최대 출력

### 주의사항
- ARM 상태가 아니면 스로틀 오버라이드가 무시됨
- 모드/스로틀 인터락: Stabilize 모드에서만 직접 스로틀 제어 가능

---

## 14. `request_rcou()` — SERVO 출력 모니터링 요청

### 필요 지식
- 구형 방식: `REQUEST_DATA_STREAM` (Hz 단위)
- 신형 방식: `SET_MESSAGE_INTERVAL` (μs 단위)
- `SERVO_OUTPUT_RAW` 가용성: 20Hz = 50,000μs 간격 권장
- 실제 모터 출력을 확인하여 공격 성공 여부를 모니터링

---

## 15. `print_rcou_once()` — SERVO 값 1회 출력

### 필요 지식
- `SERVO_OUTPUT_RAW.servo1_raw` 등 필드명
- 값 단위: **마이크로초 (μs)**
- 타임아웃과 **최근 프레임 캐치** 타이밍 고려

---

## 16. `force_disarm()` — 강제 DISARM

### 필요 지식
- 매직 넘버 **21196**: 일부 빌드/세이프티 옵션에서 제한 가능
  - 대부분의 ArduPilot 빌드에서는 허용됨
- 반복 전송 간격: 120ms (메시지 충돌 방지)
- 3회 반복 전송으로 신뢰성 확보

---

## 17. `confirm_disarmed()` — DISARM 상태 확인

### 필요 지식
- `HEARTBEAT.base_mode`의 **ARMED 비트**: `0x80` (128)
  - `base_mode & 0x80 != 0` → ARM 상태
  - `base_mode & 0x80 == 0` → DISARM 상태
- Heartbeat 수신 주기: ~1Hz (타임아웃 3~5초)

---

## 18. `main()` — 전체 공격 시퀀스

### 필요 지식
- `force_mavlink2=True` 설정: 텔레메트리 라디오와 FCU 모두 MAVLink2 지원 필수
- **재부팅 → 재연결 → 서명 재확립** 순서: GCS 동시 연결 시 충돌 가능
- 스핀 루프에서 **중간 DISARM 감지** 처리: 외부 요인으로 DISARM 시 모터 정지/종료
- **쿨다운/정지 시퀀스**: 저스로틀 펄스 → override 해제 → 강제 DISARM → 상태 확인
