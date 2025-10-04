# CSU-01.01.04: 확장 명령어 핸들러

## 1. CSU 간단 설명

`extensions.tsx`는 `gemini extensions` 명령어를 위한 최상위 핸들러(라우터) 역할을 하는 소프트웨어 단위입니다. 이 CSU의 주된 목적은 확장 프로그램(extensions) 관리를 위한 다양한 하위 명령어들을 그룹화하고, 사용자의 입력을 해당 기능에 맞는 하위 명령어로 전달하는 것입니다.

`CSU-01.01.03: MCP 명령어 핸들러`와 유사하게, 이 파일은 자체적으로 복잡한 로직을 수행하지 않으며, `yargs` 라이브러리를 사용하여 `install`, `uninstall`, `list` 등 다수의 하위 명령어를 등록하고 관리하는 구조를 정의합니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 변수

| 변수명              | 설명                                                                                                              |
| ------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `extensionsCommand` | `yargs`의 `CommandModule` 인터페이스를 구현한 객체로, `gemini extensions` 명령어의 전체 구조와 동작을 정의합니다. |

## 3. 각 변수에 대한 상세 설명

### 3.1 `extensionsCommand`

`extensionsCommand` 객체는 다음과 같은 속성으로 구성됩니다.

- **`command: 'extensions <command>'`**: 이 모듈이 처리할 명령어의 이름을 'extensions'로 지정합니다. `<command>`는 하위 명령어가 필수적으로 와야 함을 나타냅니다.

- **`describe: 'Manage Gemini CLI extensions.'`**: CLI의 도움말(`--help`)에 표시될 명령어에 대한 간단한 설명입니다.

- **`builder: (yargs: Argv) => ...`**: `extensions` 명령어의 하위 명령어 구조를 정의하는 함수입니다.
  - **입력**: `yargs` 객체 (`Argv` 타입)
  - **출력**: 설정이 추가된 `yargs` 객체
  - **함수 처리 설명**:
    - `yargs.command(...)`를 연쇄적으로 호출하여 각 하위 명령어를 등록합니다.
    - 등록되는 하위 명령어는 다음과 같습니다:
      - `installCommand`: 확장 설치 (`gemini extensions install`)
      - `uninstallCommand`: 확장 제거 (`gemini extensions uninstall`)
      - `listCommand`: 설치된 확장 목록 표시 (`gemini extensions list`)
      - `updateCommand`: 확장 업데이트 (`gemini extensions update`)
      - `disableCommand`: 확장 비활성화 (`gemini extensions disable`)
      - `enableCommand`: 확장 활성화 (`gemini extensions enable`)
      - `linkCommand`: 로컬 개발용 확장 연결 (`gemini extensions link`)
      - `newCommand`: 새 확장 프로젝트 생성 (`gemini extensions new`)
    - `yargs.demandCommand(1, ...)`: `gemini extensions` 실행 시 위 목록의 하위 명령어 중 하나가 반드시 필요함을 명시합니다. 하위 명령어가 없으면 yargs가 자동으로 도움말 메시지를 출력합니다.

- **`handler: () => {}`**: `gemini extensions` 명령어 자체에 대한 핸들러입니다.
  - **함수 처리 설명**: 이 핸들러는 비어있습니다. `builder`에서 `demandCommand(1)`을 사용했기 때문에, 사용자가 하위 명령어를 지정하지 않으면 yargs가 자동으로 도움말을 보여주고 핸들러는 호출되지 않습니다. 모든 실제 작업은 각 하위 명령어의 핸들러에서 수행됩니다.

## 4. 기타 참고사항

- 이 CSU는 `gemini-cli`의 확장 프로그램 생태계를 관리하기 위한 모든 사용자 명령어의 진입점(entry point) 역할을 합니다.
- 전형적인 커맨드 라우터(Command Router) 패턴을 사용하여, 각 기능의 구현을 별도의 파일로 분리함으로써 코드의 모듈성과 확장성을 높입니다.
- 각 하위 명령어(`install`, `new` 등)의 구체적인 로직은 이 CSU에 포함되지 않으며, 각각 임포트된 모듈 내에 별도로 정의되어 있습니다.
