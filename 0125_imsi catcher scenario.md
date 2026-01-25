FBS의 근본적 한계 (3GPP TS 33.401 기반)
1. FBS가 할 수 없는 것

FBS는 KASME (NAS 보안키)가 없으므로:
- NAS Security Mode Command 생성 불가 (NAS-MAC 계산 불가)
- NAS 메시지 암호화/복호화 불가
- 정상적인 망 등록 절차 완료 불가
2. IMSI Catcher의 실제 공격 시나리오 (수정)
FBS는 의도적으로 절차를 실패시켜 UE를 RRC Idle로 되돌린 후, 자신의 Cell을 다시 선택하게 만들어 Identity Request (IMSI)만 획득하는 것이 목표입니다.

``` mermaid
sequenceDiagram
    participant UE
    participant FBS as FBS (IMSI Catcher)
    participant LegitBS as Legitimate eNB (450-05-eNB1234-Cell0)
    
    Note over UE,LegitBS: Phase 0: UE가 정상 기지국에 연결된 상태
    
    UE->>LegitBS: RRC Connected State
    Note over UE: TAI = 450-05-1234
    
    Note over FBS: ⚠️ FBS 활성화!<br/>높은 전력으로 신호 송출<br/>TAI = 450-05-9999 (의도적으로 다른 TAI)
    
    UE->>UE: Measurement<br/>FBS 신호 강도 증가<br/>RSRP: -70dBm (매우 강함!)
    
    UE->>LegitBS: Measurement Report<br/>(Event A3: Neighbor stronger)
    
    LegitBS->>UE: Handover Command<br/>Target: 450-05-eNB9999-Cell0 (FBS)
    
    Note over UE,FBS: Phase 1: Handover 시도 → 의도적 실패
    
    UE->>FBS: RACH Preamble (Msg1)
    Note over FBS: ⚠️ RAR을 의도적으로 보내지 않음!<br/>(NAS 처리 불가하므로 Handover 완료 불가)
    
    FBS--xUE: No RAR
    
    Note over UE: T304 Timer Expiry<br/>Handover Failure!
    
    Note over UE,FBS: Phase 2: RRC Reestablishment → 의도적 거부
    
    UE->>FBS: RRC Reestablishment Request
    Note over FBS: ⚠️ Reestablishment도 의도적으로 거부!<br/>(결국 NAS Security 필요하므로)
    
    FBS->>UE: RRC Reestablishment Reject
    
    Note over UE: ✅ FBS의 목표 달성!<br/>UE를 RRC Idle로 강제 전환
    
    Note over UE,FBS: Phase 3: Cell Reselection → FBS 다시 선택
    
    UE->>UE: Enter RRC Idle State<br/>Cell Reselection 수행
    
    Note over UE: Measurement 결과<br/>FBS: -70dBm (여전히 강함!)<br/>Legitimate BS: -85dBm (약함)
    
    UE->>FBS: Cell Selection → 450-05-eNB9999-Cell0
    FBS->>UE: SIB1 (Global Cell ID 획득)
    
    Note over UE,FBS: Phase 4: TAU 유도 → IMSI 획득
    
    UE->>FBS: RRC Connection Request
    FBS->>UE: RRC Connection Setup
    
    Note over UE: 이전 TAI: 450-05-1234<br/>현재 TAI: 450-05-9999 (SIB1)<br/>→ TAI 불일치 감지!
    
    UE->>FBS: TAU REQUEST (GUTI)
    Note over FBS: GUTI만으로는 사용자 식별 불가!<br/>IMSI 필요
    
    FBS->>UE: IDENTITY REQUEST (Type = IMSI)
    Note over FBS: ✅ IMSI 요구 성공!<br/>(UE가 응답하면 IMSI 탈취)
    
    Note over UE: ⚠️ IMSI Catcher 패턴 탐지!<br/>1. HO Failure (T304 Expiry)<br/>2. RRC Reest Reject<br/>3. RRC Idle 진입<br/>4. TAU → Identity Request (IMSI)
```
    
3. 핵심 포인트
3.1 FBS는 왜 Handover/Reestablishment를 실패시키는가?
3GPP TS 36.300 Section 10.1.2 (Handover):


Handover 완료 조건:
1. Target Cell로 RRC Connection Reconfiguration Complete 전송
2. MME로 Path Switch Request 전송
3. S1 Bearer 전환

→ 이 모든 과정에 NAS Security Context 필요
→ FBS는 KASME가 없어 불가능
→ 따라서 의도적으로 RAR을 보내지 않아 Handover 실패 유도
3GPP TS 36.331 Section 5.3.7 (RRC Reestablishment):


Reestablishment 완료 조건:
1. 이전 Security Context 복구
2. SRB1 재설정
3. NAS 메시지 전달 재개

→ 결국 NAS Security Context 필요
→ FBS는 처리 불가
→ 따라서 의도적으로 Reestablishment Reject 전송
3.2 FBS의 진짜 목표: RRC Idle + TAI 불일치
TS 23.401 Section 5.3.3.1 (TAU Trigger):


TAU가 트리거되는 조건:
1. UE가 새로운 TA (Tracking Area)로 이동
2. Periodic TAU Timer 만료
3. 기타 조건...

FBS 전략:
- SIB1에서 다른 TAI 방송 (예: 450-05-9999)
- UE는 TAI 변경 감지 → TAU REQUEST 전송
- FBS는 GUTI만으로 식별 불가 → IDENTITY REQUEST (IMSI) 전송
3.3 FBS는 IMSI 획득 후에도 망 등록 완료 불가

``` mermaid
sequenceDiagram
    participant UE
    participant FBS
    
    UE->>FBS: IDENTITY RESPONSE (IMSI)
    Note over FBS: ✅ IMSI 탈취 성공!
    
    Note over FBS: 이제 뭘 할까?<br/>망 등록을 완료시켜야 하는데...
    
    FBS->>UE: AUTHENTICATION REQUEST (RAND, AUTN)?
    Note over FBS: ❌ 문제!<br/>유효한 AUTN 생성 불가<br/>(K가 없음)
    
    Note over FBS: 설령 Real Network에서<br/>릴레이했다 해도...
    
    FBS->>UE: SECURITY MODE COMMAND?
    Note over FBS: ❌ 더 큰 문제!<br/>NAS-MAC 생성 불가<br/>(KASME 없음)
    
    Note over FBS: 결국 망 등록 완료 불가!<br/>IMSI만 얻고 끝

```


