# CSU-02.01.02: 프롬프트 관리자

## 1. CSU 간단 설명

`prompts/` 디렉터리에 위치한 이 CSU는 애플리케이션에서 사용되는 다양한 프롬프트 템플릿을 체계적으로 관리하기 위한 저장소(Registry) 역할을 합니다. 특히 외부 MCP(Model Context Protocol) 서버로부터 동적으로 추가되는 프롬프트들을 등록, 조회, 관리하는 기능을 제공합니다.

`PromptRegistry` 클래스를 통해 프롬프트들을 이름 또는 서버별로 관리하며, 이를 통해 프롬프트의 재사용성을 높이고 코드의 다른 부분에서 쉽게 가져다 쓸 수 있도록 합니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 클래스

| 클래스명         | 설명                                                     |
| ---------------- | -------------------------------------------------------- |
| `PromptRegistry` | 프롬프트들을 등록하고 관리하는 중앙 저장소 클래스입니다. |

### `PromptRegistry`의 주요 메서드

| 메서드명                  | 설명                                             |
| ------------------------- | ------------------------------------------------ |
| `registerPrompt()`        | 새로운 프롬프트 정의를 레지스트리에 등록합니다.  |
| `getAllPrompts()`         | 등록된 모든 프롬프트의 목록을 반환합니다.        |
| `getPrompt()`             | 이름으로 특정 프롬프트를 조회합니다.             |
| `getPromptsByServer()`    | 특정 MCP 서버에 속한 모든 프롬프트를 조회합니다. |
| `clear()`                 | 레지스트리의 모든 프롬프트를 삭제합니다.         |
| `removePromptsByServer()` | 특정 MCP 서버에 속한 모든 프롬프트를 삭제합니다. |

### 주요 유틸리티 함수 (`mcp-prompts.ts`)

| 함수명                  | 설명                                                                                           |
| ----------------------- | ---------------------------------------------------------------------------------------------- |
| `getMCPServerPrompts()` | `Config` 객체에 포함된 `PromptRegistry`를 사용하여 특정 MCP 서버의 프롬프트 목록을 가져옵니다. |

## 3. 각 함수에 대한 입출력 및 함수 처리 설명

### 3.1 `PromptRegistry.registerPrompt()`

- **입력**: `prompt: DiscoveredMCPPrompt` (MCP 서버로부터 발견된 프롬프트 객체)
- **출력**: `void`
- **함수 처리 설명**:
  - `prompts` 맵(`Map<string, DiscoveredMCPPrompt>`)에 새로운 프롬프트를 추가합니다.
  - 만약 등록하려는 프롬프트와 동일한 이름(`prompt.name`)이 이미 맵에 존재할 경우, 이름 충돌을 피하기 위해 `서버명_프롬프트명` 형식의 새로운 이름으로 변경하여 등록하고 경고 메시지를 출력합니다.
  - 중복되지 않는 이름일 경우, 그대로 맵에 저장합니다.

### 3.2 `PromptRegistry.getPrompt()`

- **입력**: `name: string` (조회할 프롬프트의 이름)
- **출력**: `DiscoveredMCPPrompt | undefined` (발견된 프롬프트 객체 또는 `undefined`)
- **함수 처리 설명**:
  - `prompts` 맵에서 주어진 `name`을 키(key)로 사용하여 해당하는 프롬프트 객체를 찾아 반환합니다.

### 3.3 `PromptRegistry.getPromptsByServer()`

- **입력**: `serverName: string` (조회할 MCP 서버의 이름)
- **출력**: `DiscoveredMCPPrompt[]` (해당 서버에 속한 프롬프트 객체의 배열)
- **함수 처리 설명**:
  - `prompts` 맵의 모든 값을 순회하면서, 각 프롬프트의 `serverName` 속성이 입력된 `serverName`과 일치하는지 확인합니다.
  - 일치하는 프롬프트들만 수집하여 배열로 만든 후, 이름순으로 정렬하여 반환합니다.

### 3.4 `getMCPServerPrompts()`

- **입력**:
  - `config: Config` (애플리케이션의 전역 설정 객체)
  - `serverName: string` (조회할 MCP 서버의 이름)
- **출력**: `DiscoveredMCPPrompt[]` (해당 서버에 속한 프롬프트 객체의 배열)
- **함수 처리 설명**:
  - `config` 객체에서 `getPromptRegistry()` 메서드를 통해 `PromptRegistry` 인스턴스를 가져옵니다.
  - 가져온 레지스트리 인스턴스의 `getPromptsByServer()` 메서드를 호출하여 결과를 반환합니다. 이 함수는 다른 모듈에서 `PromptRegistry`에 쉽게 접근할 수 있도록 하는 래퍼(wrapper) 역할을 합니다.

## 4. 기타 참고사항

- 이 CSU는 SDD의 설명("프롬프트를 조합하여 최종 프롬프트를 생성한다")과는 다소 차이가 있습니다. 실제 코드는 프롬프트를 동적으로 "조합"하거나 "생성"하는 로직보다는, 사전에 정의된 프롬프트 템플릿들을 "등록"하고 "관리"하는 저장소(Registry) 패턴에 가깝습니다.
- 실제 프롬프트 조합 및 생성 로직은 `CSU-02.01.01`의 `AgentExecutor` 내 `buildSystemPrompt`와 같은 메서드에서 이루어집니다. 이 CSU는 그 과정에 필요한 재료(프롬프트 템플릿)를 제공하는 역할을 합니다.
