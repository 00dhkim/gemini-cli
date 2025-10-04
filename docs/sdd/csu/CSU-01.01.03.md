# CSU-01.01.03: MCP 명령어 핸들러

## 1. CSU 간단 설명

`mcp.ts`는 `gemini mcp` 명령어를 위한 최상위 핸들러(또는 라우터) 역할을 하는 소프트웨어 단위입니다. 이 CSU는 자체적으로 복잡한 로직을 수행하지 않으며, 대신 `mcp` 명령어에 대한 하위 명령어들(sub-commands)을 정의하고, 사용자의 입력을 해당 하위 명령어로 전달하는 역할을 합니다.

`yargs` 라이브러리의 `CommandModule`을 사용하여 `add`, `remove`, `list`와 같은 하위 명령어들을 등록하고 관리합니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 변수

| 변수명       | 설명                                                                                                  |
| ------------ | ----------------------------------------------------------------------------------------------------- |
| `mcpCommand` | `yargs`의 `CommandModule` 인터페이스를 구현한 객체로, `gemini mcp` 명령어의 구조와 동작을 정의합니다. |

## 3. 각 변수에 대한 상세 설명

### 3.1 `mcpCommand`

`mcpCommand` 객체는 다음과 같은 속성으로 구성됩니다.

- **`command: 'mcp'`**: 이 모듈이 처리할 명령어의 이름을 'mcp'로 지정합니다.

- **`describe: 'Manage MCP servers'`**: CLI의 도움말(`--help`)에 표시될 명령어에 대한 간단한 설명입니다.

- **`builder: (yargs: Argv) => ...`**: `mcp` 명령어의 하위 명령어 구조를 정의하는 함수입니다.
  - **입력**: `yargs` 객체 (`Argv` 타입)
  - **출력**: 설정이 추가된 `yargs` 객체
  - **함수 처리 설명**:
    1. `yargs.command(addCommand)`: `./mcp/add.js` 파일에서 가져온 `addCommand` 모듈을 하위 명령어로 등록합니다. 이를 통해 `gemini mcp add` 명령어를 처리할 수 있게 됩니다.
    2. `yargs.command(removeCommand)`: `./mcp/remove.js` 파일에서 가져온 `removeCommand` 모듈을 하위 명령어로 등록합니다. 이를 통해 `gemini mcp remove` 명령어를 처리할 수 있게 됩니다.
    3. `yargs.command(listCommand)`: `./mcp/list.js` 파일에서 가져온 `listCommand` 모듈을 하위 명령어로 등록합니다. 이를 통해 `gemini mcp list` 명령어를 처리할 수 있게 됩니다.
    4. `yargs.demandCommand(1, ...)`: `gemini mcp` 실행 시 `add`, `remove`, `list` 중 하나 이상의 하위 명령어가 반드시 필요함을 명시합니다. 만약 하위 명령어가 없으면, yargs가 자동으로 도움말 메시지를 출력합니다.

- **`handler: () => {}`**: `gemini mcp` 명령어 자체에 대한 핸들러입니다.
  - **함수 처리 설명**: 이 핸들러는 비어있습니다. `builder`에서 `demandCommand(1)`을 사용했기 때문에, 사용자가 하위 명령어를 지정하지 않으면 yargs가 자동으로 도움말을 보여주고 핸들러는 호출되지 않습니다. 모든 실제 작업은 각 하위 명령어(`add`, `remove`, `list`)의 핸들러에서 수행됩니다.

## 4. 기타 참고사항

- 이 CSU는 전형적인 커맨드 라우터(Command Router) 패턴을 보여줍니다. 메인 명령어는 하위 기능들을 그룹화하고, 실제 구현은 각 기능별 모듈로 분리하여 코드의 구조를 명확하고 유지보수하기 쉽게 만듭니다.
- `add`, `remove`, `list` 명령어의 실제 로직은 이 CSU에 포함되지 않으며, 각각 임포트된 모듈 내에 정의되어 있습니다.
