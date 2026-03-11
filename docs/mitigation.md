# 🛡️ 대응 방안 (Mitigation)

## 개요
MAVLink Key Injection 공격에 대한 방어 전략을 **즉시 적용**, **중기**, **장기** 단계로 분류합니다.

---

## 1. 즉시 적용 가능 (Quick Win)

### 1-1. MAVLink 서명 활성화
가장 직접적이고 효과적인 대응입니다.

```
# Mission Planner에서 서명 키 설정
1. Config → Full Parameter Tree
2. 서명 키 설정 (32바이트 랜덤 키 생성)
3. 서명 필수 적용 활성화
```

- **효과**: 서명이 없는 메시지를 거부하여 무단 접근 차단
- **주의**: GCS와 드론 양쪽 모두 같은 키를 설정해야 함

### 1-2. 안전 파라미터 활성화
```
BRD_SAFETYENABLE = 1    # 세이프티 스위치 활성화
ARMING_CHECK = 1        # Pre-Arm 안전 체크 활성화
BRD_SAFETY_DEFLT = 1    # 기본 세이프티 On
```

### 1-3. 물리적 보안
- 텔레메트리 포트에 물리적 접근 제한
- 비행 전 연결된 장치 확인

---

## 2. 중기 대응

### 2-1. SETUP_SIGNING 메시지 모니터링
- MAVLink 로그에서 `SETUP_SIGNING(#256)` 메시지 감시
- 예상치 못한 서명 키 변경 시 경고 발생

### 2-2. 통신 채널 분리
- 제어용 채널과 텔레메트리 채널 분리
- 제어 채널은 암호화된 통신 사용

### 2-3. 펌웨어 커스터마이징
```c
// handle_setup_signing 함수에 추가 검증
if (!physical_button_pressed()) {
    return "ERROR: Physical confirmation required";
}
```
- 키 설정 시 물리적 확인(버튼 누름 등) 요구

---

## 3. 장기 대응

### 3-1. End-to-End 암호화
- MAVLink 메시지에 TLS/DTLS 적용
- 인증서 기반 상호 인증

### 3-2. 안전한 키 교환 프로토콜
- Diffie-Hellman 등 키 교환 프로토콜 적용
- 하드코딩된 키 대신 동적 키 생성

### 3-3. 이상 탐지 시스템
- 비정상 MAVLink 트래픽 패턴 탐지
- 알 수 없는 시스템 ID의 메시지 차단
- 서명 키 변경 시도 로깅

### 3-4. MAVLink 프로토콜 개선 제안
- 서명을 **기본 활성화**(Default-On)로 변경
- `SETUP_SIGNING` 메시지에 인증 추가
- 키 설정 시 기존 키로 서명된 메시지 요구
