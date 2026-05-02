# 💻 Developer — Godot Engine의 GDScript를 활용하여 '모듈형 시스템 인터페이스(Modular System Interface)'의 기본 뼈대를 구축할 것. 핵심은, 콘텐츠가 스토리라인에 따라 로드되는 것이 아니라, 표준 API (MCP/CSM)를 통해 외부 데이터 모듈이 삽입 및 교체될 수 있는 추상화 계층(Abstract Layer)을 설계하는 것입니다.

💻 Developer: 작업 시작합니다.

CEO의 지시와 Researcher가 제시한 '능동적 자아 재구성' 메커니즘(Echo Log, Trait Synthesis)을 실제로 구현하려면, 시스템이 고정된 스크립트 흐름에 의존해서는 안 됩니다. 외부에서 데이터를 주입하고 로직을 교체할 수 있는 **표준화된 아키텍처**가 필수적입니다.

GDScript 기반의 '모듈형 시스템 인터페이스' 뼈대를 구축합니다. 이 구조는 모든 콘텐츠 모듈이 따라야 할 계약(Contract)을 정의하고, 중앙 매니저가 이를 동적으로 로드/호출하는 원리입니다.

---

### 📂 프로젝트 구성 (Directory Structure)

```
res://
├── modules/               # [모든 외부 콘텐츠 모듈이 위치할 폴더]
│   ├── dialogue_module.gd  # 예시: 대화 시스템 모듈
│   └── trait_engine_module.gd # 예시: 트레이트 합성 엔진 모듈 (Researcher 구현용)
├── scripts/               # [핵심 로직 및 매니저 스크립트]
│   ├── GameManager.gd     # <<-- Singleton / Autoload (메인 제어기)
│   └── IModuleInterface.gd # <<-- Interface 정의 (모든 모듈의 계약서)
```

### 📜 핵심 코드 산출물

#### 1. `IModuleInterface.gd` (가상 인터페이스/계약서 정의)

이 스크립트는 모든 콘텐츠 모듈이 반드시 가져야 할 메서드 목록을 정의합니다. 실제 GDScript는 강력한 인터페이스를 지원하지 않으므로, 이 코드를 **모든 모듈의 부모 클래스**로 사용하고 `_init()` 함수에서 강제 로딩 및 체크를 수행하는 것이 가장 안전합니다.

```gdscript
# scripts/IModuleInterface.gd (Abstract Base Class 역할)
class_name IModuleInterface extends Node
extends Node

@export var module_name: String = "Unnamed Module" # 모듈 식별자 필수

# 이 함수는 모든 콘텐츠가 시작될 때 가장 먼저 호출되는 초기화 지점입니다.
# 자식 클래스(실제 모듈)는 반드시 오버라이드 해야 합니다.
func initialize(global_context: Dictionary):
    print("--- [{}]: 초기화 시작 ---".format([module_name]))
    if not global_context.has("player_data"):
        push_error("IModuleInterface 오류: Global Context에 'player_data'가 누락되었습니다.")
        return false

# 모듈이 특정 이벤트(예: 대화 종료, 아이템 획득)를 처리할 때 호출됩니다.
# 매니저가 이 함수를 통해 콘텐츠 로직을 분기합니다.
func handle_event(event_type: String, data: Dictionary = {}) -> bool:
    print("--- [{}]: 이벤트 수신 - {} ---".format([module_name, event_type]))
    return false # 기본값은 처리를 하지 않음

# 모듈이 게임 루프에서 주기적으로 업데이트해야 할 로직 (선택 사항)
func update(delta: float):
    pass

# 모듈을 메모리에서 안전하게 제거하는 정리 함수
func shutdown():
    print("--- [{}]: 종료 및 리소스 해제 완료 ---".format([module_name]))
```

#### 2. `GameManager.gd` (중앙 제어기 / Singleton)

이 스크립트는 **Autoload**로 설정되어야 하며, 모든 모듈을 관리하고 실행하는 유일한 출입구입니다.

```gdscript
# scripts/GameManager.gd (Singleton으로 등록할 파일)
extends Node

# 로드된 모든 활성 모듈 인스턴스를 저장합니다. (Key: Module Name)
var active_modules: Dictionary = {} 
const MODULE_PATH = "res://modules/" # 모듈 폴더 경로 지정

func _ready():
    print("=============================================")
    print("[GameManager] 시스템 부팅 완료. 모듈 로드 준비.")
    # 시작 시, 필수 모듈들을 강제로 초기화합니다.
    load_all_required_modules(["dialogue_module", "trait_engine_module"])

# -------------------------------------------
# [핵심 API: 모듈 관리]
# -------------------------------------------

# 외부 콘텐츠를 동적으로 로드하고 초기화하는 메인 함수 (API 호출 지점)
func load_all_required_modules(module_names: Array):
    var global_context = {"player_data": {} /* 실제 플레이어 데이터를 주입 */}
    for module_name in module_names:
        var path = MODULE_PATH + "%s.gd" % module_name
        var ModuleScene = load(path) # GDScript를 Resource로 로드 시도

        if ModuleScene is Node and ModuleScene.has_method("initialize"):
            var new_module = ModuleScene.instantiate() as IModuleInterface
            new_module.module_name = module_name
            
            # 1. 초기화 (계약서 강제 실행)
            if new_module.initialize(global_context):
                active_modules[module_name] = new_module
                add_child(new_module) # 씬 트리에 추가하여 메모리 유지

func execute_event(event_type: String, data: Dictionary = {}):
    """
    메인 스토리라인에서 이벤트가 발생했을 때 모든 활성 모듈에 이벤트를 전달합니다.
    각 모듈은 자신에게 필요한 로직만 처리하고 나머지는 무시합니다 (Decoupling).
    """
    print("\n[GameManager] ======================================")
    print("[GameManager] 시스템 이벤트 발생: {}".format([event_type]))
    var successful_handling = false
    for module_name in active_modules.keys():
        var module = active_modules[module_name]
        if module.has_method("handle_event"):
            # 모듈별로 이벤트를 전파하고, 성공 여부를 체크합니다.
            var success = module.handle_event(event_type, data)
            if success:
                successful_handling = true
    
    if not successful_handling:
        print("[GameManager] 경고: 어떤 모듈도 이 이벤트를 처리하지 않았습니다.")

# -------------------------------------------
# [게임 루프 연동]
# -------------------------------------------

func _process(delta):
    for module in active_modules.values():
        if module.has_method("update"):
            module.update(delta)
```

#### 3. `dialogue_module.gd` (샘플 모듈 - 대화 시스템)

실제 콘텐츠를 담는 외부 데이터 모듈의 예시입니다. 이 코드는 **절대 메인 스토리라인 스크립트와 직접 연결되지 않습니다.** 오직 `GameManager`가 호출합니다.

```gdscript
# modules/dialogue_module.gd 
extends Node
class_name DialogueModule # 클래스명을 명시하여 독립적임을 강조

var current_scene_id: String = ""

func initialize(global_context: Dictionary):
    if not super.initialize(global_context): return false
    print("[Dialogue Module]: 대화 데이터베이스를 로드했습니다.")
    # TODO: 외부 JSON/CSV 파일에서 대화 트리를 불러오는 로직 구현 (API 통합 지점)
    return true

func handle_event(event_type: String, data: Dictionary = {}) -> bool:
    if event_type == "SCENE_LOADED":
        current_scene_id = data.get("scene_id", "")
        print("[Dialogue Module]: 현재 씬 ID에 맞춰 대화 초기 세트를 설정합니다.")
        return true # 이 이벤트는 제가 처리했습니다!

    elif event_type == "PLAYER_CHOICE_MADE":
        var choice_text = data.get("choice")
        if choice_text:
            print("[Dialogue Module]: 선택된 내용 '{}'에 따라 다음 대화 흐름을 계산합니다.".format([choice_text]))
            # 💡 여기에서 Echo Log 데이터를 참조하여 다음 대화를 변경할 수 있습니다.
            return true

    return false # 처리하지 않은 이벤트는 false 반환 (다른 모듈에게 기회 제공)

func update(delta: float):
    pass
```

---

### ✨ 사용 방법 및 통합 원리 설명

1.  **Godot 설정:** `GameManager.gd`를 **Autoload**로 등록합니다. (`Project Settings > Autoload`).
2.  **실행 로직 (가상의 Main Scene 스크립트):** 메인 씬의 스크립트는 콘텐츠에 대한 지식을 갖지 않습니다. 오직 이벤트 발생 시점을 감지하고 `GameManager`에게 메시지를 던집니다.

    ```gdscript
    # ExampleMainScene.gd (메인 오브젝트 스크립트)
    func _on_player_enters_village():
        # 1. 모듈 로드 필요 시점: 매니저가 이미 초기화해줬지만, 상황 변화에 따라 재설정 호출 가능
        GameManager.execute_event("SCENE_LOADED", {"scene_id": "VillageA"})

    func _on_player_selects_lie():
        # 2. 선택 이벤트 발생 시: GameManager를 통해 모든 모듈에게 알림
        var choice_data = {"choice": "거짓말하기"}
        GameManager.execute_event("PLAYER_CHOICE_MADE", choice_data)

    func _on_system_timer_tick():
        # 3. 주기적 시스템 로직: GameManager를 통해 모든 모듈의 업데이트 함수 호출
        GameManager._process(0.1) # 실제로 매니저가 이 역할을 하므로, 여기서는 개념만 설명합니다.
    ```

**[핵심 원리 요약]**
*   **분리 (Decoupling):** 메인 씬 스크립트는 `dialogue_module`의 존재를 몰라도 됩니다. 그저 "선택이 일어났다"는 **이벤트 이름**만 알고 `GameManager`에게 전달하면 됩니다.
*   **확장성:** 새로운 기능(예: 인벤토리 모듈, 날씨 시스템 모듈)이 필요할 때, `IModuleInterface`를 따르는 새 스크립트만 만들고 `GameManager`의 로드 목록에 추가하기만 하면 됩니다. 기존 코드는 수정할 필요가 없습니다.

📊 평가: 완료 — 표준화된 인터페이스와 중앙 매니저 패턴을 성공적으로 구축하여 콘텐츠 모듈화를 위한 견고한 기반이 마련되었습니다.
📝 다음 단계: Researcher에게 전달받은 'Echo Log'의 구체적인 데이터 구조(JSON/Resource)를 바탕으로, `trait_engine_module.gd`가 실제로 해당 데이터를 읽어와 로직을 수행하도록 구현해야 합니다.
