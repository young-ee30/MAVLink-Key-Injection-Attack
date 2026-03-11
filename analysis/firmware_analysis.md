# 🔬 ArduCopter 펌웨어 리버싱 분석

## 분석 대상
- **파일**: `arducopter.elf`
- **버전**: ArduCopter V4.5.7 (2a3dc4b7)
- **도구**: IDA Pro

---

## 1. 서명 구조체 확인

펌웨어 내 `signing`, `key`, `secret` 등의 문자열을 검색하여 MAVLink2 표준 서명 구조체를 사용하고 있음을 확인했습니다.

### `handle_setup_signing` 함수 디컴파일 결과

```c
// handle_setup_signing 함수
if (*(_BYTE *)(v2 + 156))  // ARM 상태 체크
    return "ERROR: Won't setup signing when armed";

v4 = *(unsigned __int8 *)(a2 + 3);  // 메시지 길이
if (v4 >= 0x2A) v4 = 42;             // 최대 42바이트
memcpy(v10, a2 + 12, v4);            // offset +12부터 복사
```

### 핵심 발견
- **ARM 상태 체크**: DISARM 상태에서만 서명 키 설정 가능
- **버퍼 크기**: 최대 42바이트 (키 32B + 타임스탬프 8B + 여유 2B)
- **복사 시작 위치**: 메시지 offset +12

---

## 2. 키 구조 분석

| 구성 요소 | 크기 | 설명 |
|-----------|------|------|
| Secret Key | 32바이트 | SHA-256 서명에 사용되는 비밀키 |
| Timestamp | 8바이트 | 10μs 단위, MAVLink Epoch 기준 |

> **취약점**: 키 복잡도에 대한 제한이 없어 임의의 32바이트 값으로 설정 가능

---

## 3. 타임스탬프 처리

`load_signing_key` 함수 분석 결과:

- **기준**: MAVLink Epoch (2015-01-01 00:00:00 UTC = Unix 1420070400)
- **단위**: 10μs (마이크로초)
- **수식**: `timestamp = (현재_Unix_시간 - 1420070400) × 100,000`

### Anti-Replay 메커니즘
- FCU는 수신한 타임스탬프가 이전 값보다 **클 때만** 메시지를 수신
- 따라서 공격자는 현재 시간 기준으로 유효한 타임스탬프를 생성해야 함

---

## 4. 강제 DISARM 매직 넘버

`MAV_CMD_COMPONENT_ARM_DISARM` 처리 로직에서 발견:

```c
// param2 == 21196 이면 강제 DISARM (비행 중에도 가능)
if (param2 == 21196) {
    force_disarm = true;
}
```

- **일반 DISARM**: `param1=0, param2=0` → 비행 중 거부됨
- **강제 DISARM**: `param1=0, param2=21196` → 비행 중에도 즉시 정지

> ⚠️ 이 값은 ArduPilot 소스코드에서도 확인 가능한 공개 정보이지만, 
> 리버싱을 통해 펌웨어 수준에서 직접 확인한 점이 의미 있음

---

## 5. 파라미터 시스템 (AP_Param)

`AP_Param` 클래스를 통해 런타임에 드론 설정을 변경할 수 있음을 확인:

- `PARAM_SET` 메시지로 원격 변경 가능
- Safety, Arming Check 등 보안 관련 설정도 변경 가능
- 일부 파라미터는 재부팅 후 적용

### 보안 우회 가능한 파라미터
| 파라미터 | 기본값 | 공격 설정 | 효과 |
|----------|--------|-----------|------|
| `BRD_SAFETYENABLE` | 1 | 0 | 세이프티 스위치 비활성화 |
| `ARMING_CHECK` | 1 | 0 | Pre-Arm 체크 우회 |
| `BRD_SAFETY_MASK` | 0x3FFF | 0 | 세이프티 마스크 해제 |
