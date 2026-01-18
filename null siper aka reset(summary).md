// 1. Phase 1: Null Cipher(암호화 미적용) 탐지 로직

sequenceDiagram
    autonumber
    participant RS as Reputation Server
    participant GNSS as GNSS (GPS)
    participant UE1 as UE (First Detector)
    participant FBS as Fake Base Station

    UE1->>FBS: 접속 시도 (Attach)
    FBS->>UE1: 보안 설정 요구 (EEA0/EIA0: Null Cipher)
    
    Note over UE1: [복합 검증]
    UE1->>GNSS: 현재 국가 코드 확인 (예: KR)
    Note right of UE1: 판단: 한국(KR)은 Null Cipher 금지 국가<br/>+ 단말은 EEA2 지원 가능<br/>→ "암호화 강제 해제 공격" 확정

    UE1->>RS: THREAT_REPORT (NULL_CIPHER, Score: 50)
    RS->>RS: 해당 Cell 상태 변경 (NORMAL → BARRING)
    UE1->>FBS: 보안 설정 거부 및 접속 종료



// 2. Phase 3: AKA 기반 강제 초기화

sequenceDiagram
    autonumber
    participant RS as Reputation Server
    participant UE1 as UE (Validator)
    participant Legit as Legitimate BS
    participant CN as Core Network

    Note over UE1,RS: 현재 서버 상태: BARRING (접속 금지)
    
    UE1->>Legit: [검증 모드] 강제 접속 시도
    Legit->>CN: 인증 요청
    CN->>UE1: AKA 인증 및 보안 명령 (정상 암호화)
    
    Note over UE1: [정상 기지국 검증 완료]
    Note right of UE1: 암호화 알고리즘 정상 적용 확인<br/>(FBS는 수행 불가능한 절차)

    UE1->>RS: AKA_SUCCESS_REPORT (Force Reset)
    RS->>RS: 해당 Cell 위협 기록 즉시 삭제
    Note right of RS: 상태 조기 복구 (BARRING → NORMAL)
    RS-->>UE1: ACK (시스템 정상화)
    
