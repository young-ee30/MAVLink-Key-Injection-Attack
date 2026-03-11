# 📄 드론 파라미터 설정 분석

## 분석 대상
- **파일**: `param.param` (Mission Planner에서 추출)
- **드론**: Holybro X500 V2 (Pixhawk 기반)

---

## 1. 시스템 식별

| 파라미터 | 값 | 의미 |
|----------|-----|------|
| `SYSID_THISMAV` | 1 | 드론의 MAVLink 시스템 ID |

→ 공격 시 `TARGET_SYSTEM=1`로 설정

---

## 2. 안전 설정 (Security Settings)

| 파라미터 | 값 | 의미 | 취약성 |
|----------|-----|------|--------|
| `BRD_SAFETYENABLE` | 0 | 세이프티 스위치 비활성화 | ⚠️ 물리적 안전장치 없음 |
| `BRD_SAFETY_DEFLT` | 0 | 기본 세이프티 상태 | ⚠️ 기본적으로 해제 |  
| `BRD_SAFETY_MASK` | 0 | 세이프티 적용 채널 마스크 | ⚠️ 모든 채널 보호 없음 |
| `BRD_SAFETYOPTION` | 0 | 세이프티 옵션 | ⚠️ 추가 옵션 없음 |
| `ARMING_CHECK` | 0 | Pre-Arm 체크 | ⚠️ 안전 체크 우회 상태 |

> **분석 결과**: 안전 관련 설정이 모두 비활성화되어 있어  
> 별도의 파라미터 조작 없이도 ARM이 가능한 상태

---

## 3. 프레임 설정 (Frame Configuration)

| 파라미터 | 값 | 의미 |
|----------|-----|------|
| `FRAME_CLASS` | 1 | Quad (4모터) |
| `FRAME_TYPE` | 1 | X형 배치 |

### X형 모터 배치도
```
       앞
    3     1
      \ /
       X
      / \
    4     2
```

---

## 4. 모터 매핑 (Motor Mapping)

| 파라미터 | 값 | 모터 | 위치 |
|----------|-----|------|------|
| `SERVO1_FUNCTION` | 33 | Motor1 | 앞-오른쪽 |
| `SERVO2_FUNCTION` | 34 | Motor2 | 뒤-왼쪽 |
| `SERVO3_FUNCTION` | 35 | Motor3 | 앞-왼쪽 |
| `SERVO4_FUNCTION` | 36 | Motor4 | 뒤-오른쪽 |

---

## 5. PWM 설정

| 파라미터 | 값 | 의미 |
|----------|-----|------|
| `BRD_PWM_COUNT` | 4 | MAIN 출력 채널 수 |
| `MOT_PWM_TYPE` | 0 | Normal PWM |
| `MOT_SPIN_ARM` | 0.10 | ARM 시 모터 최소 출력 (10%) |
| `DISARM_DELAY` | 60 | 자동 DISARM 지연 (60초) |

---

## 6. 통신 설정

| 파라미터 | 값 | 의미 |
|----------|-----|------|
| 시리얼 포트 | UART | 텔레메트리 연결 방식 |
| Baud Rate | 57600 | 통신 속도 |
| 프로토콜 | MAVLink2 | 서명 지원 버전 |

---

## 7. 취약점 관련 설정 요약

### ❌ 서명 미적용
- MAVLink 서명 관련 설정이 **존재하지 않음**
- 이는 서명이 **활성화되지 않았음**을 의미
- 누구나 MAVLink 메시지를 보내 드론 제어 가능

### ❌ Safety 비활성화  
- 세이프티 관련 모든 파라미터가 0
- 물리적 안전장치가 작동하지 않는 상태

### ❌ Pre-Arm 체크 비활성화
- `ARMING_CHECK=0`으로 설정되어 있어
- GPS, 센서 등의 상태와 무관하게 ARM 가능
