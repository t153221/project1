# Design

## Mystery Survival

**22112116, 강민호 (MinHo Kang), 22112116@yu.ac.kr**

---

## Revision History

| Revision date | Version # | Description | Author |
|---|---|---|---|
| 06/07/2026 | 1.00 | Initial design document | 강민호 |

---

## Contents

1. [Introduction](#1-introduction)
2. [Class Diagram](#2-class-diagram)
3. [Sequence Diagram](#3-sequence-diagram)
4. [State Machine Diagram](#4-state-machine-diagram)
5. [Implementation Requirements](#5-implementation-requirements)
6. [Glossary](#6-glossary)
7. [References](#7-references)

---

## 1. Introduction

본 문서는 언리얼 엔진 기반 청각 중심 공포 생존 게임 "Mystery Survival"의 설계 내용을 기술한다. Conceptualization 및 Analysis 단계에서 도출된 유스케이스와 도메인 분석 결과를 바탕으로, 시스템의 구조적·동적 설계를 다이어그램과 상세 설명을 통해 명확하게 제시한다.

**설계의 주요 포인트**는 다음과 같다.

첫째, 게임의 핵심 메커니즘인 청각 기반 인터랙션을 EscapeZone 클래스를 중심으로 설계하였다. 언리얼 엔진의 Sound Cue, Attenuation Settings, Resonance Audio Plugin을 활용하여 탈출 지점에서 발생하는 환경음이 플레이어에게 방향 정보를 제공한다.

둘째, 적 AI의 추적 로직을 EnemyController가 담당하며, AI MoveTo를 이용하여 매 프레임 플레이어 위치를 목표로 이동한다. 플레이어와 접촉 시 즉시 게임 오버 조건이 발생한다.

셋째, UIManager가 플레이어, 적, 탈출 지점의 월드 좌표를 화면 좌표로 변환하여 HUD 아이콘 위치를 매 프레임 갱신한다.

---

## 2. Class Diagram

### 2.1 Class Diagram

```mermaid
classDiagram
    class PlayerController {
        -Vector3D position
        -float moveSpeed
        -Vector3D direction
        +Move(direction: Vector3D) void
        +GetPosition() Vector3D
    }

    class EnemyController {
        -float moveSpeed
        -PlayerController targetActor
        +ChaseTarget(targetPos: Vector3D) void
        +GetCurrentPosition() Vector3D
    }

    class EscapeZone {
        -Vector3D position
        -bool isActive
        +OnPlayerEnter(player: PlayerController) void
        +IsActive() bool
        +GetPosition() Vector3D
    }

    class UIManager {
        -Vector2D playerIconPos
        -Vector2D enemyIconPos
        -Vector2D destIconPos
        +UpdateIconPositions() void
        +ShowGameOverScreen() void
        +ShowClearScreen() void
        +ShowMainMenu() void
    }

    EnemyController "*" --> "1" PlayerController : chases
    PlayerController ..> UIManager : notifies
    EscapeZone ..> UIManager : notifies
    UIManager ..> PlayerController : observes
    UIManager ..> EnemyController : observes
    UIManager ..> EscapeZone : observes
```

---

### 2.2 Class 상세 설명

#### PlayerController

| 항목 | 내용 |
|---|---|
| **역할** | 플레이어 캐릭터의 이동과 방향 전환을 담당한다. |
| **position** | 플레이어의 현재 3D 월드 좌표 |
| **moveSpeed** | 이동 속도 |
| **direction** | 현재 이동 방향 벡터 |
| **Move()** | WASD 입력에 따라 이동 처리. 발걸음 소리를 재생한다. |
| **GetPosition()** | 현재 위치를 반환한다. UIManager와 EnemyController가 참조한다. |

---

#### EnemyController

| 항목 | 내용 |
|---|---|
| **역할** | AI가 제어하는 적의 추적 행동을 담당한다. |
| **moveSpeed** | 적의 이동 속도 |
| **targetActor** | 추적 대상(플레이어) 참조 |
| **ChaseTarget()** | NavMesh를 이용해 플레이어 위치로 이동한다. 경로 탐색 실패 시 직선 이동을 시도한다. |
| **GetCurrentPosition()** | 현재 위치를 반환한다. UIManager가 아이콘 표시에 사용한다. |

---

#### EscapeZone

| 항목 | 내용 |
|---|---|
| **역할** | 플레이어가 도달 시 클리어 조건을 GameManager에 전달하고, 공간 음향으로 위치를 알린다. |
| **position** | 탈출 지점의 월드 좌표 |
| **isActive** | 탈출 지점 활성화 여부 |
| **OnPlayerEnter()** | 플레이어 진입 감지 시 GameManager의 EndGame(Clear)를 호출한다. |

---

#### UIManager

| 항목 | 내용 |
|---|---|
| **역할** | 게임 내 모든 HUD 및 화면 UI를 관리한다. |
| **playerIconPos** | 화면에 표시되는 플레이어 아이콘의 현재 위치 |
| **enemyIconPos** | 화면에 표시되는 적 아이콘의 현재 위치 |
| **destIconPos** | 화면에 표시되는 목적지 아이콘의 현재 위치 |
| **UpdateIconPositions()** | 매 프레임 각 액터의 월드 좌표를 화면 좌표로 변환하여 아이콘 위치를 갱신한다. |
| **ShowGameOverScreen()** | 게임 오버 화면을 표시한다. |
| **ShowClearScreen()** | 클리어 화면을 표시한다. |
| **ShowMainMenu()** | 메인 메뉴 화면을 표시한다. |

---

## 3. Sequence Diagram

### 3.1 UC#1: 게임 시작

```mermaid
sequenceDiagram
    actor Player
    participant WBP_MainMenu
    participant PlayerController
    participant EscapeZone
    participant UIManager

    Player->>WBP_MainMenu: START 버튼 선택
    WBP_MainMenu->>WBP_MainMenu: Open Level (게임 씬 로드, < 5초)
    alt 씬 로드 성공
        PlayerController->>UIManager: WBP_HUD 생성 및 표시
        EscapeZone->>EscapeZone: 환경음 재생 시작 (Play Sound at Location)
    else 씬 로드 실패
        WBP_MainMenu->>Player: 오류 메시지 표시 후 메인 메뉴 유지
    end
```

**설명**: 플레이어가 메인 메뉴(WBP_MainMenu)에서 START 버튼을 누르면 Open Level 노드로 게임 씬을 로드한다. 씬 로드 후 BP_Player의 Begin Play에서 WBP_HUD가 생성되고, BP_EscapeZone의 Begin Play에서 공간 음향이 재생된다.

---

### 3.2 UC#2 + UC#4: 환경 탐색 및 이동 조작

```mermaid
sequenceDiagram
    actor Player
    participant PlayerController
    participant UIManager

    Player->>PlayerController: 이동 입력 (WASD)
    PlayerController->>PlayerController: 충돌 감지 확인
    alt 장애물 없음
        PlayerController->>PlayerController: 캐릭터 이동
        PlayerController->>PlayerController: 마우스 방향으로 캐릭터 회전
        PlayerController->>PlayerController: 발걸음 소리 재생 (Play Sound at Location)
        UIManager->>UIManager: 플레이어 아이콘 위치 갱신
    else 장애물 충돌
        PlayerController->>PlayerController: 해당 방향 이동 중단
    end

    Player->>PlayerController: 이동 입력 중단
    PlayerController->>PlayerController: 이동 정지
```

**설명**: 플레이어의 이동 입력에 따라 캐릭터가 이동하며, BP_Player 내부에서 Play Sound at Location 노드로 발걸음 소리를 재생한다. UIManager(WBP_HUD)는 플레이어 아이콘 위치를 매 프레임 갱신한다. 마우스 커서 방향으로 캐릭터가 회전한다.

---

### 3.3 UC#3: 소리 감지

```mermaid
sequenceDiagram
    participant EscapeZone
    actor Player

    EscapeZone->>EscapeZone: Begin Play → Play Sound at Location (모노 파일, NewSoundAttenuation)
    EscapeZone->>EscapeZone: Attenuation 적용 (거리에 따른 음량 감쇠)
    EscapeZone->>EscapeZone: Spatialization 적용 (방향 정보 좌우/전후 채널 분리)

    alt 헤드폰 출력 환경
        EscapeZone->>EscapeZone: Resonance Audio Plugin → HRTF 기반 입체 음향 적용
    end

    EscapeZone->>Player: 방향/거리 정보가 반영된 소리 전달
    Player->>Player: 소리의 방향과 거리를 청각으로 인지
```

**설명**: 탈출 지점(BP_EscapeZone)의 Begin Play에서 Play Sound at Location 노드로 모노 파일을 재생한다. NewSoundAttenuation 설정과 Resonance Audio Plugin을 통해 거리 감쇠 및 방향성이 적용되어 플레이어에게 전달된다.

---

### 3.4 UC#5 + UC#6: 적 회피 및 적 반응

```mermaid
sequenceDiagram
    actor Player
    participant PlayerController
    participant EnemyController
    participant UIManager

    EnemyController->>EnemyController: 플레이어 위치 확인 (매 프레임)
    EnemyController->>EnemyController: AI MoveTo(플레이어 위치)로 이동

    UIManager->>UIManager: 적 아이콘 위치 갱신
    Player->>Player: HUD 아이콘으로 적 위치 파악

    alt 플레이어가 적을 회피
        Player->>PlayerController: 이동 입력으로 적 회피
        Player->>Player: 안전하게 이동 완료
    else 플레이어가 적과 접촉
        EnemyController->>PlayerController: 접촉(Overlap) 감지
        EnemyController->>UIManager: 게임 오버 화면 표시 요청
    end
```

**설명**: 적은 매 프레임 AI MoveTo를 통해 플레이어를 추적한다. UIManager가 HUD에 적 아이콘 위치를 갱신하며, 플레이어는 이를 참고해 회피 경로를 결정한다. 플레이어와 접촉하면 즉시 게임 오버가 발생한다.

---

### 3.5 UC#7: 게임 오버

```mermaid
sequenceDiagram
    participant EnemyController
    participant PlayerController
    participant UIManager
    actor Player

    EnemyController->>PlayerController: 접촉(ActorBeginOverlap) 감지
    PlayerController->>PlayerController: Cast to BP_Enemy 성공
    PlayerController->>UIManager: WBP_GameOver 생성 및 Add to Viewport
    UIManager->>Player: 재시작(RESTART) / 종료(QUIT) 옵션 제공

    alt 재시작 선택
        Player->>UIManager: RESTART 버튼 클릭
        UIManager->>UIManager: Open Level "Untitled"
    else 종료 선택
        Player->>UIManager: QUIT 버튼 클릭
        UIManager->>UIManager: Quit Game
    end
```

**설명**: BP_Enemy가 BP_Player와 Overlap되면 BP_Player의 ActorBeginOverlap 이벤트에서 Cast to BP_Enemy 후 WBP_GameOver 위젯을 생성하여 화면에 표시한다. 플레이어는 재시작 또는 종료를 선택할 수 있다.

---

### 3.6 UC#8 + UC#9: 목표 달성 및 게임 종료

```mermaid
sequenceDiagram
    actor Player
    participant PlayerController
    participant EscapeZone
    participant UIManager

    Player->>PlayerController: 청각적 단서(환경음)로 탈출 지점 방향 파악
    PlayerController->>EscapeZone: 탈출 지점 도달 (ActorBeginOverlap)
    EscapeZone->>EscapeZone: Cast to BP_Player 성공
    EscapeZone->>UIManager: WBP_Clear 생성 및 Add to Viewport
    UIManager->>Player: 재시작(RESTART) / 종료(QUIT) 옵션 제공

    alt 재시작 선택
        Player->>UIManager: RESTART 버튼 클릭
        UIManager->>UIManager: Open Level "Untitled"
    else 종료 선택
        Player->>UIManager: QUIT 버튼 클릭
        UIManager->>UIManager: Quit Game
    end
```

**설명**: 플레이어가 청각적 단서(환경음)를 통해 탈출 지점 방향을 파악하고 BP_EscapeZone에 도달하면 ActorBeginOverlap 이벤트에서 Cast to BP_Player 후 WBP_Clear 위젯을 생성하여 화면에 표시한다.

---

## 4. State Machine Diagram

### 4.1 게임 전체 상태 머신

```mermaid
stateDiagram-v2
    [*] --> MainMenu : 앱 실행

    MainMenu --> Playing : 게임 시작 버튼 선택
    MainMenu --> [*] : 종료 버튼 선택

    Playing --> GameOver : Enemy 접촉
    Playing --> Clear : Escape_Zone 도달
    Playing --> MainMenu : ESC → 나가기 선택

    GameOver --> Playing : 재시작 선택
    GameOver --> [*] : 종료 선택

    Clear --> Playing : 재시작 선택
    Clear --> [*] : 종료 선택
```

**설명**: 게임은 MainMenu, Playing, GameOver, Clear의 4가지 상태로 구성된다. 게임 시작 시 Playing 상태로 전환되며, 적과 접촉하면 GameOver, 탈출 지점에 도달하면 Clear 상태로 전환된다. 결과 화면에서 플레이어는 재시작 또는 종료를 선택할 수 있다.

---

### 4.2 플레이어 상태 머신

```mermaid
stateDiagram-v2
    [*] --> Idle : 게임 시작

    Idle --> Walking : 이동 입력 (WASD)
    Walking --> Idle : 이동 입력 중단

    Idle --> Dead : 적 접촉
    Walking --> Dead : 적 접촉

    Dead --> [*] : 게임 오버 처리
```

**설명**: 플레이어는 Idle(정지), Walking(걷기), Dead(사망)의 3가지 상태를 가진다. 이동 입력 여부에 따라 Idle/Walking 상태가 전환되며, 적과 접촉하면 Dead 상태로 전환된다.

---

### 4.3 적(Enemy) 상태 머신

```mermaid
stateDiagram-v2
    [*] --> Chase : 게임 시작 / 배치

    Chase --> Attack : 플레이어 위치 도달 (Overlap)

    Attack --> [*] : 플레이어 게임 오버
```

**설명**: 적은 게임 시작과 동시에 Chase(추적) 상태로 진입하여 AI MoveTo를 통해 플레이어를 지속적으로 추적한다. 플레이어 위치에 도달하여 Overlap이 발생하면 Attack 상태로 전환되고 게임 오버가 발생한다.

---

## 5. Implementation Requirements

### 5.1 개발 환경

| 항목 | 내용 |
|---|---|
| **게임 엔진** | Unreal Engine 5.6 |
| **스크립팅** | Blueprint Visual Scripting |
| **오디오 시스템** | Unreal Engine Sound Cue + Resonance Audio Plugin |
| **AI 시스템** | Unreal Engine AI MoveTo + NavMesh |
| **버전 관리** | Git |

---

### 5.2 하드웨어 요구 사항

| 항목 | 최소 사양 | 권장 사양 |
|---|---|---|
| **OS** | Windows 10 64-bit | Windows 11 64-bit |
| **CPU** | Intel Core i5-8400 | Intel Core i7-12700 이상 |
| **RAM** | 8 GB | 16 GB 이상 |
| **GPU** | NVIDIA GTX 1060 6GB | NVIDIA RTX 3060 이상 |
| **저장 공간** | 10 GB 이상 | 20 GB 이상 (SSD 권장) |
| **오디오** | 스테레오 스피커 | 스테레오 헤드폰 (HRTF 활용) |

---

### 5.3 소프트웨어 요구 사항

| 항목 | 내용 |
|---|---|
| **Unreal Engine** | 5.6 |
| **Visual Studio** | 2022 이상 (C++ 빌드 환경) |
| **DirectX** | DirectX 12 이상 |
| **오디오 드라이버** | WASAPI 지원 드라이버 |

---

### 5.4 성능 요구 사항

| 항목 | 요구 사항 |
|---|---|
| **프레임 레이트** | 최소 30 FPS 이상 유지 |
| **입력 반응 시간** | < 50ms (이동, 소리 감지) |
| **게임 로드 시간** | < 5초 (씬 로드, 게임 종료 화면) |
| **AI 반응 시간** | < 100ms (적 추적 경로 갱신) |
| **사운드 지연** | < 50ms (Spatial Audio 처리) |

---

### 5.5 오디오 구현 요구 사항

| 항목 | 내용 |
|---|---|
| **Spatial Audio** | 언리얼 엔진 Attenuation Settings 및 Resonance Audio Plugin 활용 |
| **HRTF** | 헤드폰 환경에서 Head-Related Transfer Function 적용 |
| **Sound Cue** | 발걸음, 탈출 지점 환경음 등 개별 Sound Cue 구성 |
| **채널** | 모노(탈출 지점 환경음) — 스테레오 파일은 공간 음향 미지원 |

---

## 6. Glossary

| 용어 | 정의 |
|---|---|
| **Mystery Survival** | 본 프로젝트의 게임 명칭. 청각 기반 탑다운 공포 생존 게임. |
| **Player** | 게임을 조작하는 사용자 및 플레이어 캐릭터 |
| **Enemy / Enemy_AI** | 플레이어를 추적하는 적 AI 개체 |
| **PlayerController** | 플레이어의 이동, 방향 전환을 관리하는 클래스 |
| **EnemyController** | 적 AI의 추적 행동을 담당하는 클래스 |
| **EscapeZone** | 플레이어 도달 시 클리어 조건을 전달하는 탈출 지점 클래스 |
| **UIManager** | 플레이어/적/목적지 아이콘 HUD UI를 관리하는 클래스 |
| **Spatial Audio** | 소리의 방향과 거리를 입체적으로 표현하는 3D 오디오 기술 |
| **Attenuation** | 소리 발생 위치로부터 거리가 멀어질수록 음량이 감소하는 효과 |
| **HRTF** | Head-Related Transfer Function. 헤드폰 환경에서 소리의 3D 방향감을 구현하는 기술 |
| **Sound Cue** | 언리얼 엔진에서 여러 사운드를 조합하고 제어하는 오디오 구성 요소 |
| **Blueprint** | 언리얼 엔진의 노드 기반 시각적 스크립팅 시스템 |
| **NavMesh** | Navigation Mesh. 적 AI가 장애물을 우회하여 경로를 탐색하는 데 사용하는 탐색 가능 영역 메시 |
| **AI MoveTo** | 언리얼 엔진에서 AI가 지정된 목표 위치로 NavMesh 경로를 통해 이동하도록 하는 기능 |
| **Post Processing** | 렌더링된 화면에 밝기 조정 등의 시각 효과를 추가하는 처리 과정 |
| **HUD** | Heads-Up Display. 게임 화면에 오버레이로 표시되는 아이콘 등의 UI 요소 |

---

## 7. References

1. Epic Games. *Unreal Engine Documentation*. https://docs.unrealengine.com/
2. Epic Games. *Unreal Engine Audio System Overview*. https://docs.unrealengine.com/audio/
3. Freesound. *Audio Resources*. https://freesound.org/
4. INCOSE. *Systems Engineering Handbook*, 4th Edition.
5. Mavin, A. et al. "EARS: The Easy Approach to Requirements Syntax." *18th IEEE International Requirements Engineering Conference*, 2009.
6. Larman, C. *Applying UML and Patterns: An Introduction to Object-Oriented Analysis and Design and Iterative Development*, 3rd Edition. Prentice Hall, 2004.
