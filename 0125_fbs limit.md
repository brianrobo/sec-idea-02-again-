3G 이후 상호 인증 도입의 의미

1. 2G (GSM)의 취약점

2G에서는 단방향 인증만 존재 (TS 43.020):

``` mermaid
sequenceDiagram
    participant UE as UE (SIM)
    participant BSS as BTS/BSC
    participant NSS as MSC/VLR
    
    UE->>BSS: Location Update Request (IMSI)
    BSS->>NSS: Location Update Request
    
    NSS->>NSS: Generate RAND using Ki
    NSS->>BSS: AUTHENTICATION REQUEST (RAND)
    BSS->>UE: AUTHENTICATION REQUEST (RAND)
    
    Note over UE: UE만 인증됨!<br/>UE → Network 인증<br/>Network → UE 인증 없음!
    
    UE->>UE: SRES = A3(Ki, RAND)
    UE->>BSS: AUTHENTICATION RESPONSE (SRES)
    BSS->>NSS: AUTHENTICATION RESPONSE
    
    NSS->>NSS: SRES 검증
    
    Note over NSS: ✅ UE 인증 성공<br/>❌ Network는 인증 안됨!
    
    NSS->>BSS: LOCATION UPDATE ACCEPT
    BSS->>UE: LOCATION UPDATE ACCEPT
    
    Note over UE,NSS: ⚠️ 2G의 문제점:<br/>• Network 인증 없음<br/>• FBS가 자유롭게 망 등록 완료 가능<br/>• IMSI 탈취 후 정상 동작 가능

```

2G FBS 공격 성공 시나리오:


1. FBS → UE: AUTHENTICATION REQUEST (임의의 RAND)
2. UE → FBS: AUTHENTICATION RESPONSE (SRES)
3. FBS: SRES 무시하고 LOCATION UPDATE ACCEPT 전송
4. ✅ 망 등록 완료! (UE는 FBS를 정상 망으로 인식)
5. FBS는 이후 모든 통신 도청/조작 가능
2. 3G (UMTS)부터 상호 인증 도입
3GPP TS 33.102 Section 6.3.3 - Mutual Authentication:

``` mermaid
sequenceDiagram
    participant UE as UE (USIM)
    participant RNC as RNC
    participant CN as SGSN/MSC
    participant HLR as HLR/AuC
    
    Note over UE,HLR: Phase 1: AV (Authentication Vector) 생성
    
    CN->>HLR: Send Authentication Info Request
    HLR->>HLR: Generate AV using Ki<br/>RAND, AUTN, XRES, CK, IK
    HLR->>CN: Send Authentication Info Response (AV)
    
    Note over UE,CN: Phase 2: 상호 인증 시작
    
    CN->>RNC: AUTHENTICATION REQUEST (RAND, AUTN)
    RNC->>UE: AUTHENTICATION REQUEST (RAND, AUTN)
    
    Note over UE: ✅ Network 인증!<br/>AUTN 검증 (f1 함수)<br/>MAC = f1(Ki, SQN, RAND, AMF)<br/>계산된 MAC == AUTN의 MAC?
    
    alt AUTN 검증 실패
        UE->>RNC: AUTHENTICATION FAILURE<br/>(MAC failure or Synch failure)
        Note over UE: ❌ FBS 탐지!<br/>Network 인증 실패
    else AUTN 검증 성공
        UE->>UE: RES = f2(Ki, RAND)<br/>CK = f3(Ki, RAND)<br/>IK = f4(Ki, RAND)
        UE->>RNC: AUTHENTICATION RESPONSE (RES)
        RNC->>CN: AUTHENTICATION RESPONSE
        
        Note over CN: ✅ UE 인증!<br/>RES == XRES?
        
        CN->>RNC: SECURITY MODE COMMAND (CK, IK)
        RNC->>UE: SECURITY MODE COMMAND
        
        Note over UE: CK/IK 사용 가능<br/>(동일하게 계산했으므로)
        
        UE->>RNC: SECURITY MODE COMPLETE
        
        Note over UE,CN: ✅ 상호 인증 완료!<br/>• Network → UE (AUTN 검증)<br/>• UE → Network (RES 검증)
    end
```

3. FBS가 3G 이후에 망 등록 공격을 못하는 이유
3GPP TS 33.102 기반 분석:

시나리오 1: FBS가 임의의 AUTN 생성 시도

# FBS의 시도
RAND = random_128bit()  # 임의 생성 가능
AUTN = random_128bit()  # 임의 생성 (하지만 무효)

# AUTN 구조 (TS 33.102 Section 6.3.3)
AUTN = SQN ⊕ AK || AMF || MAC

# 여기서:
MAC = f1_Ki(SQN || RAND || AMF)  # ← Ki 필요!
AK = f5_Ki(RAND)                 # ← Ki 필요!

# FBS는 Ki가 없으므로:
# → 유효한 MAC 생성 불가
# → UE의 AUTN 검증 실패
결과:

``` mermaid
sequenceDiagram
    participant UE
    participant FBS as FBS (3G Fake NodeB)
    
    FBS->>UE: AUTHENTICATION REQUEST<br/>(RAND, Invalid_AUTN)
    
    Note over UE: AUTN 검증<br/>계산: MAC' = f1(Ki, SQN, RAND, AMF)<br/>비교: MAC' == AUTN.MAC?
    
    Note over UE: ❌ MAC 불일치!<br/>또는 SQN out of range!
    
    UE->>FBS: AUTHENTICATION FAILURE<br/>Cause: MAC failure
    
    Note over FBS: ❌ 공격 실패!<br/>망 등록 진행 불가
```

시나리오 2: FBS가 Real Network에서 AUTN 릴레이

``` mermaid
sequenceDiagram
    participant UE
    participant FBS as FBS (Relay)
    participant Real as Real Network
    
    UE->>FBS: Location Update Request (IMSI)
    FBS->>Real: Location Update Request (릴레이)
    
    Real->>FBS: AUTHENTICATION REQUEST (RAND, Valid_AUTN)
    FBS->>UE: AUTHENTICATION REQUEST (릴레이)
    
    Note over UE: AUTN 검증 성공<br/>(진짜 망의 AUTN)
    
    UE->>FBS: AUTHENTICATION RESPONSE (RES)
    FBS->>Real: AUTHENTICATION RESPONSE (릴레이)
    
    Note over Real: RES == XRES ✅
    
    Real->>FBS: SECURITY MODE COMMAND (with CK/IK based MAC)
    FBS->>UE: SECURITY MODE COMMAND (릴레이)
    
    Note over UE: Security Mode Command 검증<br/>MAC = f(CK/IK, message)
    
    UE->>FBS: SECURITY MODE COMPLETE (암호화됨!)
    
    Note over FBS: ⚠️ 문제 발생!<br/>• Security Mode Complete는 CK로 암호화됨<br/>• FBS는 CK가 없어 복호화 불가<br/>• 이후 모든 메시지 암호화됨
    
    Note over FBS: ❌ 릴레이 공격의 한계!<br/>• 망 등록은 완료될 수 있으나<br/>• 암호화된 통신 내용 확인 불가<br/>• 실질적인 도청/조작 불가

```



4. 4G (LTE)에서의 강화
3GPP TS 33.401 - 추가 보안 강화:


3G → 4G 변경사항:

1. AKA 알고리즘 강화:
   - KASME 도입 (Key Hierarchy 확장)
   - NAS Security와 AS Security 분리

2. Integrity Protection 필수화:
   - 3G: Ciphering만 필수, Integrity는 선택
   - 4G: Integrity Protection 필수

3. NAS Security Mode Command:
   - NAS-MAC 검증 추가
   - KASME 없이는 NAS-MAC 생성 불가
4G에서 FBS의 한계:


# FBS가 4G 망 등록을 완료하려면:

# 1. AUTN 생성 (3G와 동일)
AUTN = f(Ki, RAND, SQN)  # ← Ki 없어 불가

# 2. NAS Security Mode Command 생성
NAS_MAC = CMAC(KNASint, message)
# 여기서:
KNASint = KDF(KASME, ...)
KASME = KDF(CK || IK, SQN ⊕ AK, ...)
CK = f3(Ki, RAND)  # ← Ki 없어 불가
IK = f4(Ki, RAND)  # ← Ki 없어 불가

# → NAS-MAC 생성 불가
# → Security Mode Command 생성 불가
# → 망 등록 완료 불가
5. 결론: 3G 이후 FBS의 전략 변화
세대	FBS 망 등록 가능?	FBS의 목표	주요 공격 방식
2G (GSM)	✅ 가능	망 등록 완료 후 도청/조작	단방향 인증 악용
3G (UMTS)	❌ 불가	IMSI 탈취만	HO Fail → Reest Reject → IMSI Req
4G (LTE)	❌ 불가	IMSI 탈취만	동일 (NAS Security로 더 강화됨)
5G	❌ 불가	SUPI 탈취도 어려움	SUCI 암호화로 차단
3G/4G FBS의 현실적 공격 시나리오 (본 특허 문서의 Scenario A):


목표: 망 등록 완료 (불가능) → IMSI 탈취로 전환

공격 절차:
1. 강한 신호로 Handover 유도
2. Handover/Reestablishment 의도적 실패
   (이유: AUTN 생성 불가, Security Mode 처리 불가)
3. UE를 RRC Idle로 강제 전환
4. TAU 유도 (다른 TAI 방송)
5. IDENTITY REQUEST (IMSI) 전송
6. IMSI 획득 후 종료
   (이유: 더 이상 진행 불가능)
따라서 본 특허의 **"선제적 회피 (Preemptive Avoidance)"**는 3G/4G 환경에서 특히 중요합니다:

FBS는 망 등록을 완료할 수 없음
하지만 IMSI 탈취는 여전히 가능
Identity Request 받기 전에 회피하는 것이 유일한 완벽한 방어책
