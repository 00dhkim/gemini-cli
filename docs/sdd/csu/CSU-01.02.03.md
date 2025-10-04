# CSU-01.02.03: UI 상태 관리(Hooks)

## 1. CSU 간단 설명

`ui/hooks/` 디렉터리에 위치한 이 CSU는 Gemini CLI의 UI와 관련된 모든 비즈니스 로직과 상태 관리를 담당하는 React Hook의 집합입니다. React의 Hook 매커니즘을 사용하여, UI의 동적인 상태(메시지 목록, 로딩 상태, 사용자 입력 등)를 관리하고, 복잡한 로직을 재사용 가능한 함수로 캡슐화합니다.

이러한 접근 방식은 UI를 렌더링하는 컴포넌트(`CSU-01.02.02`)로부터 로직을 분리하여, 코드의 가독성, 테스트 용이성, 유지보수성을 크게 향상시킵니다.

## 2. Hook의 주요 분류

이 CSU는 수많은 커스텀 Hook을 포함하며, 기능에 따라 다음과 같이 분류할 수 있습니다.

- **코어 상태 관리**: 애플리케이션의 핵심적인 상태(대화 내역, API 스트림 등)를 관리합니다.
  - 예: `useHistoryManager.ts`, `useGeminiStream.ts`, `useMessageQueue.ts`
- **입력 및 자동완성**: 사용자 입력, 명령어 자동완성, 입력 내역 관리 로직을 처리합니다.
  - 예: `useInputHistory.ts`, `useCommandCompletion.tsx`, `useSlashCompletion.ts`
- **명령어 처리**: `@`, `/`, 셸 명령어 등 특수한 입력을 파싱하고 처리하는 로직을 담고 있습니다.
  - 예: `atCommandProcessor.ts`, `shellCommandProcessor.ts`, `slashCommandProcessor.ts`
- **UI 상태 및 효과**: 로딩 인디케이터, 터미널 크기 감지, 키 입력 처리 등 UI의 특정 상태나 효과를 관리합니다.
  - 예: `useLoadingIndicator.ts`, `useTerminalSize.ts`, `useKeypress.ts`
- **기능별 로직**: 폴더 신뢰, 테마, 에디터 설정 등 특정 기능에 종속된 상태와 로직을 관리합니다.
  - 예: `useFolderTrust.ts`, `useThemeCommand.ts`, `useEditorSettings.ts`

## 3. 주요 Hook 상세 설명

아래는 이 CSU를 구성하는 핵심 Hook 몇 가지에 대한 상세 설명입니다.

### 3.1 `useHistoryManager.ts`

- **역할**: 대화 내역(`history`) 상태를 관리하는 가장 기본적인 Hook입니다.
- **관리하는 상태**: `history: HistoryItem[]` (전체 대화 메시지 목록)
- **반환하는 값 (인터페이스)**:
  - `history`: 현재 대화 내역 배열.
  - `addItem`: 대화 내역에 새 항목을 추가하는 함수.
  - `updateItem`: 기존 항목을 수정하는 함수.
  - `clearItems`: 모든 대화 내역을 삭제하는 함수.
  - `loadHistory`: 기존 대화 내역을 불러오는 함수.
- **주요 로직**:
  - `addItem` 함수는 새로운 메시지 항목을 `history` 배열에 추가합니다. 이때 중복된 사용자 메시지가 연속으로 추가되는 것을 방지하는 로직이 포함되어 있습니다.
  - 각 메시지 항목에 고유 ID를 부여하기 위해 타임스탬프 기반의 카운터를 사용합니다.

### 3.2 `useGeminiStream.ts`

- **역할**: Gemini API와의 통신 및 스트리밍 응답 처리를 총괄하는 가장 복잡하고 핵심적인 Hook입니다.
- **관리하는 상태**:
  - `streamingState`: API 응답 상태 (IDLE, RESPONDING, WAITING_FOR_CONFIRMATION)
  - `pendingHistoryItem`: 현재 스트리밍 중인 메시지 항목.
  - `thought`: 모델의 "생각 중..." 메시지.
  - `toolCalls`: 모델이 요청한 도구 호출 목록 및 그 상태.
- **반환하는 값 (인터페이스)**:
  - `streamingState`, `pendingHistoryItems`, `thought` 등 현재 스트림 상태.
  - `submitQuery`: 사용자 입력을 받아 API로 전송을 시작하는 메인 함수.
  - `cancelOngoingRequest`: 진행 중인 API 요청을 중단하는 함수.
- **주요 로직**:
  1.  **`submitQuery`**: 사용자 입력을 받아 `@` 명령어(`handleAtCommand`), `/` 명령어(`handleSlashCommand`) 등을 전처리합니다.
  2.  `geminiClient.sendMessageStream`을 호출하여 API와 통신을 시작합니다.
  3.  `processGeminiStreamEvents` 함수 내에서 `for await...of` 루프를 통해 API가 보내는 스트림 이벤트를 실시간으로 처리합니다.
      - `Content` 이벤트: `useHistoryManager`의 `addItem`을 호출하여 화면에 텍스트를 점진적으로 표시합니다.
      - `ToolCallRequest` 이벤트: `useReactToolScheduler` Hook을 사용하여 도구 호출을 스케줄링합니다.
      - `Error`, `UserCancelled` 등 기타 이벤트를 처리합니다.
  4.  도구 실행이 완료되면, 그 결과를 다시 `submitQuery`에 전달하여 모델과의 대화를 이어갑니다.

### 3.3 `useLoadingIndicator.ts`

- **역할**: 모델이 응답 중일 때 표시되는 로딩 문구와 경과 시간을 관리합니다.
- **관리하는 상태**:
  - `elapsedTime`: 로딩 시작 후 경과된 시간.
  - `currentLoadingPhrase`: 주기적으로 변경되는 "생각 중..."과 같은 로딩 문구.
- **입력**: `streamingState` (현재 API 응답 상태)
- **주요 로직**:
  - `useTimer` Hook을 사용하여 `streamingState`가 `RESPONDING`일 때 시간을 측정합니다.
  - `usePhraseCycler` Hook을 사용하여 `RESPONDING` 상태일 때 미리 정의된 로딩 문구들을 순환하며 보여줍니다.
  - `streamingState`가 `WAITING_FOR_CONFIRMATION`(사용자 확인 대기)으로 변경되면 타이머를 일시 정지하고, 다시 `RESPONDING`으로 돌아오면 타이머를 리셋하는 등 상태 변화에 따라 경과 시간을 정확하게 관리합니다.

### 3.4 `useConsoleMessages.ts`

- **역할**: 디버그 콘솔에 표시될 메시지 목록을 관리합니다.
- **관리하는 상태**: `consoleMessages: ConsoleMessageItem[]` (콘솔 메시지 목록)
- **반환하는 값 (인터페이스)**:
  - `consoleMessages`: 현재 콘솔 메시지 배열.
  - `handleNewMessage`: 새 콘솔 메시지를 추가하는 함수.
  - `clearConsoleMessages`: 모든 콘솔 메시지를 삭제하는 함수.
- **주요 로직**:
  - `useReducer`를 사용하여 콘솔 메시지 상태를 관리합니다.
  - `handleNewMessage` 호출 시, 짧은 시간(16ms) 동안 들어오는 여러 메시지를 `setTimeout`을 이용해 일괄 처리(batching)하여 불필요한 렌더링을 줄이고 성능을 최적화합니다.
  - 동일한 메시지가 연속으로 들어올 경우, 새 항목을 추가하는 대신 기존 항목의 카운트를 증가시켜 UI를 깔끔하게 유지합니다.
