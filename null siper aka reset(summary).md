sequenceDiagram
    autonumber
    participant RS as Reputation Server
    participant GNSS as GNSS/GPS
    participant UE1 as UE_1
    participant UE2 as UE_2
    participant FBS as FBS
    participant LegitBS as Legitimate eNB
    participant CN as Core Network

    Note over RS,CN: Phase 1: NULL_CIPHER 탐지 (2시간 TTL)

    rect rgb(255, 220, 220)
    Note over UE1,FBS: T=0분 - Null Cipher 공격

    UE1->>FBS: ATTACH REQUEST
    FBS->>UE1: AUTHENTICATION REQUEST
    UE1->>FBS: AUTHENTICATION RESPONSE
    FBS->>UE1: SECURITY MODE COMMAND<br/>⚠️ EEA0/EIA0 (Null Cipher)

    Note over UE1: ⚠️ Security Mode Command에서<br/>Null Cipher (EEA0/EIA0) 탐지!

    UE1->>GNSS: Position Request<br/>(Country Code 확인 요청)
    GNSS->>GNSS: 위치 측위 (Position Measurement)<br/>+ Geocoding 작업
    GNSS->>UE1: Country Code 응답<br/>(예: KR - South Korea)

    Note over UE1: ⚠️ Scenario B 패턴 매칭!<br/>• Security Mode: EEA0/EIA0 (Null Cipher)<br/>• GNSS Country: KR (Null Cipher 금지)<br/>• UE Capability: EEA2 지원<br/>→ Downgrade Attack 확정!

    UE1->>RS: THREAT_REPORT<br/>{<br/>"cell_id": "450-05-eNB8888-Cell1",<br/>"threat_type": "NULL_CIPHER",<br/>"score": 50,<br/>"ttl_seconds": 7200<br/>}

    RS->>RS: 리포트 저장<br/>Current Reputation: 0 → 50<br/>Status: BARRING<br/>Expires: T+2시간

    RS->>UE1: ACK<br/>{<br/>"current_reputation": 50,<br/>"status": "BARRING",<br/>"expires_at": "12:00:00"<br/>}

    UE1->>FBS: SECURITY MODE REJECT
    UE1->>UE1: Cell Reselection
    end

    Note over RS,CN: Phase 2: UE_2 선제적 회피

    rect rgb(255, 245, 230)
    Note over UE2: T=10분 - Cell Selection

    UE2->>RS: GET /reputation/450-05-eNB8888-Cell1

    RS->>UE2: {<br/>"current_reputation": 50,<br/>"status": "BARRING"<br/>}

    UE2->>UE2: ⛔ BARRING 상태!<br/>Cell Selection에서 완전 제외

    UE2->>LegitBS: 다른 Cell 선택
    LegitBS->>UE2: RRC Connection Setup

    Note over UE2,LegitBS: ✅ NULL_CIPHER Cell 완전 차단
    end

    Note over RS,CN: Phase 3: FBS 제거 + AKA 기반 강제 초기화

    rect rgb(240, 255, 240)
    Note over FBS: T=30분 - FBS 제거됨 (공격자 이동)

    Note over LegitBS: 동일 Cell ID로<br/>정상 기지국 재가동

    Note over UE1: T=35분 - Cell Selection

    UE1->>RS: GET /reputation/450-05-eNB8888-Cell1

    RS->>RS: TTL 아직 남음 (1시간 30분)<br/>Current Reputation: 50<br/>Status: BARRING

    RS->>UE1: {<br/>"current_reputation": 50,<br/>"status": "BARRING"<br/>}

    UE1->>UE1: BARRING 상태 확인<br/>다른 Cell 선택 시도

    Note over UE1: 하지만 다른 Cell 없음<br/>긴급 상황

    UE1->>LegitBS: 강제 접속 시도 (Emergency)
    LegitBS->>UE1: RRC Connection Setup

    LegitBS->>CN: AUTHENTICATION REQUEST
    CN->>LegitBS: AUTHENTICATION REQUEST (Valid AUTN)
    LegitBS->>UE1: AUTHENTICATION REQUEST
    UE1->>UE1: AUTN 검증 성공

    UE1->>LegitBS: AUTHENTICATION RESPONSE
    Note over UE1: ⚠️ AUTH RESPONSE는<br/>Replay Attack 가능<br/>(아직 정상 기지국 확정 아님)

    LegitBS->>CN: Forward Response
    CN->>CN: RES = XRES ✅

    CN->>LegitBS: SECURITY MODE COMMAND (EEA2/EIA2)
    LegitBS->>UE1: SECURITY MODE COMMAND (EEA2/EIA2)
    Note over UE1: Security Mode Command 수신!<br/>• Country: KR (Null Cipher 금지)<br/>• Network: EEA2 선택<br/>→ FBS는 불가능한 절차

    UE1->>LegitBS: SECURITY MODE COMPLETE

    Note over UE1: ✅ AKA 성공 + 정상 기지국 확정!<br/>Security Mode Complete까지 완료

    UE1->>RS: AKA_SUCCESS_REPORT<br/>{<br/>"cell_id": "450-05-eNB8888-Cell1",<br/>"event_type": "AKA_SUCCESS",<br/>"score": 0,<br/>"force_reset": true<br/>}

    RS->>RS: 강제 초기화!<br/>모든 리포트 즉시 삭제<br/>Current Reputation: 50 → 0<br/>Status: NORMAL

    RS->>UE1: ACK<br/>{<br/>"current_reputation": 0,<br/>"status": "NORMAL",<br/>"reset_reason": "AKA_SUCCESS"<br/>}

    Note over RS: ✅ TTL 만료 전 강제 복구!<br/>(2시간 대기 불필요)
    end

    Note over RS,CN: Phase 4: 다른 UE들도 즉시 접속 가능

    rect rgb(240, 255, 240)
    Note over UE2: T=40분 - Cell Selection

    UE2->>RS: GET /reputation/450-05-eNB8888-Cell1

    RS->>UE2: {<br/>"current_reputation": 0,<br/>"status": "NORMAL"<br/>}

    UE2->>LegitBS: Cell Selection (정상 복구된 Cell)
    LegitBS->>UE2: RRC Connection Setup

    Note over UE2,LegitBS: ✅ 모든 UE 정상 서비스 재개
    end
