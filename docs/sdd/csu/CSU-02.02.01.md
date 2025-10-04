# CSU-02.02.01: 도구 레지스트리

## 1. CSU 간단 설명

`tools/tool-registry.ts`에 위치한 이 CSU는 Gemini CLI 에이전트가 사용할 수 있는 모든 도구(Tool)를 등록, 발견, 관리하는 중앙 저장소 역할을 합니다. `ToolRegistry` 클래스는 내장된 핵심 도구뿐만 아니라, 외부 명령어 또는 MCP(Model Context Protocol) 서버를 통해 동적으로 발견되는 도구들까지 모두 포함하여 관리합니다.

모델에게 어떤 도구를 사용할 수 있는지 알려주기 위한 도구 스키마(JSON Schema) 목록을 제공하고, 모델이 도구 사용을 요청했을 때 해당 도구를 찾아 실행할 수 있도록 하는 핵심적인 역할을 수행합니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 클래스

| 클래스명         | 설명                                                                           |
| ---------------- | ------------------------------------------------------------------------------ |
| `ToolRegistry`   | 모든 도구를 등록하고 관리하는 중앙 저장소 클래스입니다.                        |
| `DiscoveredTool` | 외부 명령어를 통해 동적으로 발견된 도구를 나타내는 래퍼(wrapper) 클래스입니다. |

### `ToolRegistry`의 주요 메서드

| 메서드명                                | 설명                                                                                               |
| --------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `registerTool()`                        | 주어진 도구를 레지스트리에 수동으로 등록합니다.                                                    |
| `discoverAllTools()`                    | 설정된 모든 소스(외부 명령어, MCP 서버)로부터 도구를 자동으로 발견하여 등록합니다.                 |
| `discoverAndRegisterToolsFromCommand()` | 외부 셸 명령어를 실행하여 도구를 발견하고 등록합니다.                                              |
| `getFunctionDeclarations()`             | 등록된 모든 도구의 스키마(`FunctionDeclaration`) 목록을 반환합니다. 이 목록은 모델에게 전송됩니다. |
| `getTool()`                             | 이름으로 특정 도구 객체를 조회합니다.                                                              |
| `getAllTools()`                         | 등록된 모든 도구 객체의 배열을 반환합니다.                                                         |

## 3. 각 함수에 대한 입출력 및 함수 처리 설명

### 3.1 `ToolRegistry.registerTool()`

- **입력**: `tool: AnyDeclarativeTool` (등록할 도구 객체)
- **출력**: `void`
- **함수 처리 설명**:
  - 내부 `tools` 맵(`Map<string, AnyDeclarativeTool>`)에 도구의 이름을 키(key)로 하여 도구 객체를 저장합니다.
  - 만약 동일한 이름의 도구가 이미 존재할 경우, 경고 메시지를 출력하고 기존 도구를 덮어씁니다.

### 3.2 `ToolRegistry.discoverAllTools()`

- **입력**: 없음
- **출력**: `Promise<void>`
- **함수 처리 설명**:
  1.  기존에 동적으로 발견되었던 도구들을 모두 삭제하여 레지스트리를 초기화합니다.
  2.  `discoverAndRegisterToolsFromCommand()`를 호출하여 외부 명령어를 통해 도구를 발견합니다.
  3.  `mcpClientManager.discoverAllMcpTools()`를 호출하여 설정된 모든 MCP 서버에 연결하고 도구를 발견합니다.
  - 이 메서드는 애플리케이션 시작 시 또는 도구 목록을 새로고침해야 할 때 호출됩니다.

### 3.3 `ToolRegistry.discoverAndRegisterToolsFromCommand()`

- **입력**: 없음
- **출력**: `Promise<void>`
- **함수 처리 설명**:
  1.  `config` 객체에서 `toolDiscoveryCommand` (예: `"my-script --list-tools"`) 설정을 가져옵니다.
  2.  해당 명령어를 자식 프로세스(child process)로 실행합니다.
  3.  명령어의 표준 출력(stdout)으로 출력된 JSON 문자열을 파싱합니다. 이 JSON은 `FunctionDeclaration` 객체의 배열이어야 합니다.
  4.  파싱된 각 `FunctionDeclaration`에 대해 `DiscoveredTool` 클래스의 인스턴스를 생성합니다.
  5.  생성된 `DiscoveredTool` 인스턴스를 `registerTool()`을 통해 레지스트리에 등록합니다.

### 3.4 `ToolRegistry.getFunctionDeclarations()`

- **입력**: 없음
- **출력**: `FunctionDeclaration[]` (도구 스키마의 배열)
- **함수 처리 설명**:
  - 레지스트리에 등록된 모든 도구 객체(`tools` 맵의 값)를 순회합니다.
  - 각 도구 객체의 `schema` 속성(도구의 이름, 설명, 파라미터 등을 기술한 JSON 스키마)을 추출하여 배열에 담아 반환합니다.
  - 이 배열은 Gemini 모델에 전송되어, 모델이 어떤 도구를 어떤 파라미터로 호출할 수 있는지 알게 하는 역할을 합니다.

### 3.5 `ToolRegistry.getTool()`

- **입력**: `name: string` (찾고자 하는 도구의 이름)
- **출력**: `AnyDeclarativeTool | undefined`
- **함수 처리 설명**:
  - `tools` 맵에서 주어진 `name`을 키로 사용하여 해당하는 도구 객체를 찾아 반환합니다.
  - 에이전트 실행기(`CSU-02.01.01`)는 모델이 도구 호출을 요청했을 때 이 메서드를 사용하여 실행할 도구의 실제 객체를 얻습니다.

## 4. 기타 참고사항

- **도구의 3가지 종류**: 이 레지스트리는 세 가지 종류의 도구를 관리합니다.
  1.  **내장 도구(Built-in)**: `ls`, `read_file` 등 소스 코드에 클래스로 구현되어 직접 `registerTool`로 등록되는 도구.
  2.  **명령어 발견 도구(Command-Discovered)**: 사용자가 설정한 셸 명령어를 통해 동적으로 발견되는 도구. `DiscoveredTool` 클래스로 래핑됩니다.
  3.  **MCP 발견 도구(MCP-Discovered)**: 원격 MCP 서버로부터 발견되는 도구. `DiscoveredMCPTool` 클래스로 래핑됩니다.
- `ToolRegistry`는 에이전트가 다양한 소스의 도구들을 일관된 방식으로 사용하고 관리할 수 있도록 하는 추상화 계층을 제공합니다.
