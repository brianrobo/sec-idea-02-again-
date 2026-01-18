// **'비정상 동작'**과 **'판단'**에 집중합니다.

sequenceDiagram
    autonumber
    participant RS as Reputation Server
    participant UE1 as UE (First Detector)
    participant FBS as Fake Base Station
    participant Legit as Legitimate BS

    Note over UE1,Legit: [정상 연결 상태]
    UE1->>Legit: RRC Connected
    
    Note over FBS: [FBS 활성화] 강한 신호 송출
    UE1->>FBS: 비정상 핸드오버/접속 수행
    
    Note over UE1,FBS: [공격 패턴 탐지]
    FBS--xUE1: 접속 거부 및 IMSI(개인정보) 요구
    Note right of UE1: ⚠️ 패턴 매칭: 가짜 기지국 확정

    Note over UE1,RS: [위협 정보 공유]
    UE1->>RS: THREAT_REPORT (Cell ID, 위험도)
    RS->>RS: 평판 DB 업데이트 (SUSPICIOUS)
    RS-->>UE1: ACK (보호 조치 시작)




// 접속하기 전에 미리 서버에 물어보고 거른다"**는 Pre-connection Check 개념

sequenceDiagram
    autonumber
    participant RS as Reputation Server
    participant UE2 as UE (Target User)
    participant FBS as Fake Base Station
    participant Legit as Legitimate BS

    Note over UE2,FBS: [주변 기지국 탐색]
    UE2->>UE2: FBS 신호 감지 (접속 후보 등록)

    Note over UE2,RS: [Pre-connection Check]
    UE2->>RS: 해당 Cell ID 평판 조회
    RS-->>UE2: 위험 상태 알림 (SUSPICIOUS)

    Note over UE2: [선제적 회피 결정]
    Note right of UE2: 위험 기지국 블랙리스트 등록 및 제외

    Note over UE2,Legit: [안전한 연결]
    UE2->>Legit: 안전한 정상 기지국 선택 및 접속
    Note right of UE2: ✅ FBS 공격 원천 차단 성공




    // 자동 복구
    sequenceDiagram
    autonumber
    participant RS as Reputation Server
    participant UE3 as UE (New User)
    participant Legit as Legitimate BS

    Note over RS: [TTL 자동 만료]
    RS->>RS: 설정 시간(30분) 경과 확인
    RS->>RS: 해당 Cell 위협 기록 자동 삭제
    Note right of RS: 상태 복구 (SUSPICIOUS → NORMAL)

    Note over UE3,RS: [복구 후 재조회]
    UE3->>RS: 해당 Cell ID 평판 조회
    RS-->>UE3: 정상 상태 알림 (NORMAL)

    Note over UE3,Legit: [정상 접속 성공]
    UE3->>Legit: 동일 Cell ID로 안전하게 접속
    Note right of UE3: ✅ 시스템 신뢰성 유지


    
