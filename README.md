# 🧠 [내일배움캠프 언리얼엔진] AI 비헤이비어 트리 : 근접 AI의 두뇌

> ## **학습 키워드**
>
> *   `Behavior Tree (행동 트리)`: AI의 의사결정 과정을 시각적으로 설계하는 계층적 구조.
> *   `Blackboard (블랙보드)`: AI가 기억해야 할 데이터(타겟, 상태 등)를 저장하는 메모리.
> *   `Selector`: 자식 노드를 왼쪽부터 차례로 실행하며, 하나라도 성공하면 즉시 멈추는 제어 노드. (OR 연산)
> *   `Sequence`: 자식 노드를 왼쪽부터 차례로 실행하며, 하나라도 실패하면 즉시 멈추는 제어 노드. (AND 연산)
> *   `Decorator`: 특정 노드의 실행 여부를 결정하는 '조건문'.
> *   `Service`: 특정 노드가 활성화되어 있는 동안 주기적으로 로직을 실행하는 '백그라운드 작업'.
> *   `Task`: AI가 수행하는 실제 행동 단위 (이동, 공격, 대기 등).

<br>


---

## **1. AI의 전체적인 행동 패턴 요약**

---
<img width="1625" height="699" alt="Image" src="https://github.com/user-attachments/assets/1c5e6839-a93b-4eb1-bbaf-ace8aca97653" />

이 AI는 **근접 공격형(Melee)** 적 캐릭터로 설계되었습니다. 

AI는 뚜렷한 우선순위에 따라 다음과 같은 행동 패턴을 보입니다.

> 1.  **공격 (Attack)**: 플레이어가 공격 범위 내에 있으면 최우선으로 공격합니다.
> 2.  **추격 (Chase)**: 플레이어를 볼 수 있지만 공격 범위 밖이라면, 플레이어를 향해 달려갑니다.
> 3.  **수색 (Investigate)**: 추격하던 플레이어를 놓치면, 마지막으로 목격했던 장소로 달려가 잠시 주변을 살핍니다.
> 4.  **순찰 (Patrol)**: 아무런 위협이 없으면, 주변의 임의의 지점을 정해 걸어서 순찰합니다.

이러한 행동 우선순위는 트리의 최상위에 위치한 **`Selector`** 노드에 의해 결정됩니다. 

`Selector`는 왼쪽 자식부터 차례대로 조건을 검사하여, 실행 가능한 첫 번째 행동을 즉시 수행합니다.

<br>

---

## **2. 비헤이비어 트리 구조 심층 분석**

---

### 1️⃣ **루트와 최상위 셀렉터: AI의 의사결정 허브**

트리의 가장 상위에는 **`Selector`** 노드가 자리 잡고 있습니다. 이 노드는 AI의 모든 행동 가지(Branch)를 연결하는 중심축입니다.

*   **Service: `BTService_CheckAttackRange`**
    > 이 `Selector`가 활성화되어 있는 동안, 이 서비스는 주기적으로 실행됩니다. 이 서비스의 역할은 플레이어와의 거리를 계산하여 "공격 가능한 거리인가?"를 판단하고,
    >
    > 그 결과를 블랙보드의 **`IsPlayerNear`** 키(bool)에 저장합니다. 즉, AI가 다른 행동을 하는 중에도 계속해서 공격 범위를 체크하는 백그라운드 작업입니다.

### 2️⃣ **1순위: 공격 (Attack Sequence)**

가장 왼쪽에 위치하여 최우선 순위를 가지는 행동입니다.

*   **Decorator: `IsPlayerNear is Set`**
    > 이 시퀀스는 블랙보드의 **`IsPlayerNear`** 값이 `true`일 때만 실행됩니다. 위에서 설명한 서비스가 이 값을 갱신해주므로,
    >
    >  플레이어가 공격 범위에 들어오는 순간 AI는 다른 모든 행동을 멈추고 이 공격 시퀀스를 시도합니다.

*   **Tasks:**
    1.  `BTTask_TestAILogic (StateToSet = Attacking)`: AI의 내부 상태를 '공격 중'으로 변경합니다. (애니메이션 제어 등에 사용될 예정입니다.)
    2.  `BTTask_PlayAttack`: 실제 공격 애니메이션이나 로직을 실행하는 커스텀 태스크입니다.
    3.  `BTTask_Wait (3.0s)`: 공격 후 3초 동안 대기합니다. 이는 공격 쿨타임 역할을 하여, AI가 무한정 공격을 난사하는 것을 방지합니다.

### 3️⃣ **2순위: 추격 (Chase Target Sequence)**

플레이어가 공격 범위 밖이지만, 시야 내에 있을 때의 행동입니다.

*   **Decorator: `CanSeeTarget is Set`**
    > AI의 '인지 시스템(Perception System)'이 플레이어를 감지하면 블랙보드의 **`CanSeeTarget`** 키가 `true`로 설정됩니다. 이 조건이 만족되면 AI는 플레이어를 추격합니다.

*   **Tasks:**
    1.  `BTTask_TestAILogic (StateToSet = Chasing)`: AI 상태를 '추격 중'으로 변경합니다.
    2.  `BTTask_SetMovementSpeed (bIsRunning = True)`: 이동 속도를 '달리기' 속도로 변경합니다.
    3.  `BTTask_MoveTo (BlackboardKey = TargetActor)`: 블랙보드에 저장된 **`TargetActor`**(플레이어)를 향해 이동합니다.

### 4️⃣ **3순위: 수색 (Investigate Sequence)**

추격하던 플레이어를 놓쳤을 때의 행동입니다.

*   **Decorator: `IsInvestigating is Set`**
    > 플레이어를 놓치는 순간(예: 인지 시스템에서 시야가 끊길 때), AI 컨트롤러가 블랙보드의 **`IsInvestigating`** 키를 `true`로 설정하고
    >
    > **`TargetLastKnownLocation`** 에 마지막 위치를 저장하는 로직이 있습니다. 이 데코레이터는 그 상태를 감지합니다.

*   **Tasks:**
    1.  `BTTask_TestAILogic (StateToSet = Investigating)`: AI 상태를 '수색 중'으로 변경합니다.
    2.  `BTTask_SetMovementSpeed (bIsRunning = True)`: 마지막 위치까지 빠르게 달려갑니다.
    3.  `BTTask_MoveTo (BlackboardKey = TargetLastKnownLocation)`: 마지막으로 플레이어를 목격했던 위치로 이동합니다.
    4.  `BTTask_SetMovementSpeed (bIsRunning = False)`: 목적지에 도착하면 '걷기' 속도로 감속합니다.
    5.  `BTTask_Wait (2.0s)`: 2초간 주변을 둘러보며 대기합니다.
    6.  `BTTask_SetKeyValueBool (Key = IsInvestigating, Value = false)`: 수색을 마치고, **`IsInvestigating`** 상태를 해제합니다. 이로써 AI는 다시 평범한 순찰 상태로 돌아갑니다.

### 5️⃣ **4순위: 순찰 (Patrol Sequence)**

위의 어떤 조건에도 해당하지 않을 때 수행하는 기본 행동입니다.

*   **Decorator: `IsInvestigating is Not Set`**
    > 이 데코레이터는 수색 중이 아닐 때를 조건으로 하지만, `Selector`의 구조상 앞선 모든 조건(공격, 추격, 수색)이 거짓일 때만 실행되므로 사실상 '기본 상태'를 의미합니다.

*   **Tasks:**
    1.  `BTTask_TestAILogic`: AI 상태를 '순찰 중'으로 변경합니다.
    2.  `BTTask_SetMovementSpeed (bIsRunning = False)`: '걷기' 속도로 이동합니다.
    3.  `BTTask_FindRandomLocation`: 주변의 이동 가능한 임의의 위치를 찾아 블랙보드의 **`RandomLocation`** 키에 저장하는 커스텀 태스크입니다.
    4.  `BTTask_MoveTo (BlackboardKey = RandomLocation)`: 방금 찾은 임의의 위치로 이동합니다.

<br>

---

## **3. AI의 기억: 블랙보드 키 분석**

---

이 AI의 모든 행동은 블랙보드에 저장된 데이터를 기반으로 결정됩니다.

*   **`TargetActor` (Object)**: 현재 AI가 인지하고 있는 타겟(플레이어) 객체입니다.
*   **`CanSeeTarget` (Bool)**: 타겟을 현재 볼 수 있는지 여부입니다.
*   **`IsPlayerNear` (Bool)**: 타겟이 공격 범위 내에 있는지 여부입니다. (`BTService_CheckAttackRange`가 갱신)
*   **`IsInvestigating` (Bool)**: 타겟을 놓쳐서 마지막 위치를 수색 중인지 여부입니다.
*   **`TargetLastKnownLocation` (Vector)**: 타겟을 놓쳤을 때, 마지막으로 목격했던 위치 좌표입니다.
*   **`RandomLocation` (Vector)**: 순찰할 다음 목적지의 임의 좌표입니다.

이처럼 언리얼 엔진의 비헤이비어 트리는 블랙보드라는 '메모리'와 

`Selector`, `Sequence` 같은 '논리 회로', 그리고 `Task`라는 '행동'이 결합하여 복잡하면서도 체계적인 AI를 만들어냅니다.

---

#UnrealEngine #언리얼엔진 #AI #BehaviorTree #비헤이비어트리 #내일배움캠프
