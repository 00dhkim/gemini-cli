# CSU-01.01.01: 메인 애플리케이션

## 1. CSU 간단 설명

`gemini.tsx`는 Gemini CLI 애플리케이션의 최상위 진입점(Entry Point) 및 메인 루프 역할을 수행하는 핵심 소프트웨어 단위입니다. 사용자가 CLI를 실행할 때 가장 먼저 실행되며, 전체 애플리케이션의 생명주기를 관리합니다.

이 CSU는 명령어 인자(arguments) 파싱, 설정 파일 로딩, 샌드박스 환경 설정, 인증 처리, 대화형(interactive) UI와 비대화형(non-interactive) 모드 분기 등 애플리케이션의 초기화와 실행에 필요한 모든 핵심 로직을 포함하고 있습니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 함수

| 함수명                             | 설명                                                                                  |
| ---------------------------------- | ------------------------------------------------------------------------------------- |
| `main()`                           | 애플리케이션의 주 진입점. 전체 실행 흐름을 총괄합니다.                                |
| `startInteractiveUI()`             | 대화형 모드의 React 기반 UI를 렌더링하고 실행합니다.                                  |
| `setupUnhandledRejectionHandler()` | 처리되지 않은 Promise 거부(rejection) 오류를 잡아내는 전역 오류 핸들러를 설정합니다.  |
| `validateDnsResolutionOrder()`     | 설정된 DNS 해석 순서 값의 유효성을 검사하고 기본값을 반환합니다.                      |
| `getNodeMemoryArgs()`              | 시스템 메모리에 기반하여 Node.js 힙 메모리(`--max-old-space-size`) 인자를 계산합니다. |
| `setWindowTitle()`                 | 터미널 창의 제목을 설정합니다.                                                        |

### 주요 컴포넌트

| 컴포넌트명   | 설명                                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `AppWrapper` | 대화형 UI의 최상위 React 래퍼(wrapper) 컴포넌트. 각종 Context Provider를 통해 하위 컴포넌트에 상태와 설정을 제공합니다. |

## 3. 각 함수에 대한 입출력 및 함수 처리 설명

### 3.1 `main()`

- **입력**: 없음 (프로세스 인자 `process.argv`와 표준 입력 `process.stdin`을 내부적으로 사용)
- **출력**: `Promise<void>`
- **함수 처리 설명**:
  1.  `setupUnhandledRejectionHandler()`를 호출하여 전역 오류 핸들러를 설정합니다.
  2.  `loadSettings()`를 호출하여 사용자 설정(`settings.json`)을 불러옵니다.
  3.  `parseArguments()`를 호출하여 사용자가 입력한 명령어 라인 인자(flags)를 파싱합니다.
  4.  디버그 모드 여부를 확인하고, `ConsolePatcher`를 통해 `console.log` 등의 동작을 재정의합니다.
  5.  샌드박스 실행이 필요한 경우(`start_sandbox`), 메모리 인자(`getNodeMemoryArgs`) 등을 계산하여 자식 프로세스로 애플리케이션을 재실행합니다.
  6.  `loadCliConfig()`를 통해 확장 기능, 인증 정보 등을 포함한 CLI의 전체 설정을 구성합니다.
  7.  사용자 입력과 설정에 따라 대화형 모드와 비대화형 모드를 판단합니다.
      - **대화형 모드**: `startInteractiveUI()`를 호출하여 UI를 렌더링합니다.
      - **비대화형 모드**: `runNonInteractive()`를 호출하여 프롬프트를 처리하고 결과를 출력한 뒤 종료합니다.
  8.  `runExitCleanup()`을 통해 애플리케이션 종료 시 필요한 정리 작업을 수행합니다.

### 3.2 `startInteractiveUI()`

- **입력**:
  - `config`: `Config` 객체 (CLI의 전체 설정)
  - `settings`: `LoadedSettings` 객체 (사용자 설정)
  - `startupWarnings`: `string[]` (시작 시 표시할 경고 메시지 목록)
  - `workspaceRoot`: `string` (현재 작업 디렉토리 경로)
  - `initializationResult`: `InitializationResult` (초기화 결과)
- **출력**: `Promise<void>`
- **함수 처리 설명**:
  1.  `setWindowTitle()`을 호출하여 터미널 창 제목을 설정합니다.
  2.  `AppWrapper` 내부 컴포넌트를 정의합니다. 이 컴포넌트는 React Context Provider들(`SettingsContext`, `KeypressProvider`, `SessionStatsProvider`, `VimModeProvider`)로 `AppContainer`를 감싸, UI 전반에 걸쳐 필요한 상태와 기능을 공유할 수 있도록 합니다.
  3.  Ink 라이브러리의 `render()` 함수를 호출하여 `AppWrapper` 컴포넌트를 터미널에 렌더링합니다.
  4.  `checkForUpdates()`를 비동기적으로 호출하여 CLI의 새로운 버전이 있는지 확인하고, 자동 업데이트 설정을 처리합니다.
  5.  `registerCleanup()`을 통해 애플리케이션 종료 시 `render` 인스턴스를 정리(`unmount`)하는 함수를 등록합니다.

### 3.3 `setupUnhandledRejectionHandler()`

- **입력**: 없음
- **출력**: `void`
- **함수 처리 설명**:
  - `process.on('unhandledRejection', ...)`을 사용하여 Node.js 프로세스에서 처리되지 않은 Promise 거부 이벤트를 감지하는 리스너를 등록합니다.
  - 오류 발생 시, `/bug` 도구를 사용하여 버그 리포트를 제출하도록 안내하는 비판적 오류 메시지를 `AppEvent.LogError` 이벤트를 통해 출력합니다.
  - 첫 오류 발생 시 `AppEvent.OpenDebugConsole` 이벤트를 발생시켜 디버그 콘솔을 엽니다.

## 4. 기타 참고사항

- 이 CSU는 애플리케이션의 다른 모든 부분(설정, UI, 코어 엔진 등)을 연결하고 초기화하는 중심 허브(central hub) 역할을 하므로, CLI의 동작 방식을 이해하는 데 가장 중요한 파일 중 하나입니다.
- 대화형 모드는 React와 Ink 라이브러리를 기반으로 동작하며, 비대화형 모드는 표준 입출력을 통해 간단한 프롬프트 처리 후 종료되는 방식으로 동작합니다.
- 샌드박스 기능은 보안을 위해 활성화될 수 있으며, 이 경우 CLI는 격리된 환경의 자식 프로세스 내에서 재실행됩니다.
