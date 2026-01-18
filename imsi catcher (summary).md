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
