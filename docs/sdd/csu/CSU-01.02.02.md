# CSU-01.02.02: UI 컴포넌트

## 1. CSU 간단 설명

`ui/components/` 디렉터리에 위치한 이 CSU는 Gemini CLI의 사용자 인터페이스를 구성하는 모든 개별 UI 요소(컴포넌트)의 집합입니다. 이 컴포넌트들은 React를 기반으로 작성되었으며, 메시지 버블, 입력창, 다이얼로그, 상태 바 등 사용자가 화면에서 보는 거의 모든 시각적 요소를 정의합니다.

각 컴포넌트는 특정 기능을 수행하도록 설계된 재사용 가능한 부품이며, 이들이 `CSU-01.02.01: UI 레이아웃`에 의해 조합되어 전체 애플리케이션 화면을 형성합니다.

## 2. 컴포넌트의 주요 분류

이 CSU는 수많은 컴포넌트를 포함하며, 기능에 따라 다음과 같이 분류할 수 있습니다.

- **코어 인터랙션**: 사용자의 주된 상호작용을 담당하는 컴포넌트.
  - 예: `MainContent.tsx`, `Composer.tsx`, `InputPrompt.tsx`
- **메시지 표시**: 대화 내역에 표시되는 다양한 유형의 메시지 컴포넌트.
  - 예: `UserMessage.tsx`, `GeminiMessage.tsx`, `ToolMessage.tsx`, `ErrorMessage.tsx`
- **다이얼로그**: 사용자에게 추가 정보를 요청하거나 옵션을 제공하는 팝업 형태의 컴포넌트.
  - 예: `DialogManager.tsx`, `SettingsDialog.tsx`, `ShellConfirmationDialog.tsx`, `ThemeDialog.tsx`
- **상태 및 정보 표시**: 헤더, 푸터, 로딩 상태 등 각종 정보를 표시하는 컴포넌트.
  - 예: `Header.tsx`, `Footer.tsx`, `LoadingIndicator.tsx`, `StatsDisplay.tsx`
- **공용 컴포넌트**: 여러 곳에서 재사용되는 범용 컴포넌트.
  - 예: `shared/` 디렉터리 내의 컴포넌트들

## 3. 주요 컴포넌트 상세 설명

아래는 이 CSU를 구성하는 핵심 컴포넌트 몇 가지에 대한 상세 설명입니다.

### 3.1 `MainContent.tsx`

- **역할**: 대화 내역 전체(과거 및 현재 진행 중인 메시지)를 화면에 렌더링하는 주 영역입니다.
- **입력**: `useUIState` Hook을 통해 `history`(완료된 대화 목록)와 `pendingHistoryItems`(현재 처리 중인 대화) 상태를 입력받습니다.
- **처리 및 출력**:
  - Ink의 `<Static>` 컴포넌트를 사용하여 이미 완료된 `history` 항목들을 렌더링합니다. 이는 불필요한 재렌더링을 방지하여 성능을 최적화하는 역할을 합니다.
  - 각 대화 항목은 `HistoryItemDisplay` 컴포넌트를 통해 표시됩니다.
  - 현재 모델이 응답을 생성 중인 `pendingHistoryItems`는 `<Static>` 외부에 렌더링하여 실시간 스트리밍 업데이트가 가능하도록 합니다.
  - 최상단에는 `AppHeader` 컴포넌트를 포함하여 앱의 제목과 버전 정보를 표시합니다.

### 3.2 `Composer.tsx`

- **역할**: 사용자가 프롬프트를 입력하고 각종 상태 정보를 확인하는 UI 하단부의 복합 제어판입니다. 단순한 입력창 이상의 역할을 수행합니다.
- **입력**: `useUIState`, `useUIActions` 등 다수의 Context Hook을 통해 UI의 거의 모든 상태와 액션을 입력받습니다.
- **처리 및 출력**:
  - `uiState`의 상태에 따라 다양한 컴포넌트를 조건부로 렌더링합니다.
  - **`LoadingIndicator`**: 모델이 응답을 기다리거나 처리 중일 때 로딩 스피너와 "생각 중..." 메시지를 표시합니다.
  - **`ContextSummaryDisplay`**: 현재 대화의 컨텍스트에 포함된 파일(`@` 명령어)이나 도구 정보를 표시합니다.
  - **`InputPrompt`**: 사용자가 실제로 텍스트를 입력하는 핵심 컴포넌트입니다. 자동완성, 명령어 제안 등의 기능을 포함합니다.
  - **`DetailedMessagesDisplay`**: 디버그 콘솔이 활성화되었을 때 상세 오류 로그를 표시합니다.
  - **`Footer`**: 각종 단축키나 상태 정보를 보여주는 하단 바를 렌더링합니다.

### 3.3 `DialogManager.tsx`

- **역할**: 애플리케이션의 모든 다이얼로그(팝업)를 관리하고, 한 번에 하나의 다이얼로그만 표시되도록 제어하는 라우터(Router) 컴포넌트입니다.
- **입력**: `useUIState` Hook을 통해 `isSettingsDialogOpen`, `shellConfirmationRequest` 등 다이얼로그 표시 여부를 결정하는 다양한 boolean 상태 값을 입력받습니다.
- **처리 및 출력**:
  - 거대한 `if-else if` 체인으로 구성되어 있습니다.
  - `uiState`의 특정 상태값이 `true`이면, 그에 해당하는 다이얼로그 컴포넌트(예: `<SettingsDialog />`, `<ThemeDialog />`, `<ShellConfirmationDialog />` 등)를 렌더링하고 즉시 반환합니다.
  - 해당하는 상태값이 모두 `false`이면 `null`을 반환하여 아무 다이얼로그도 표시하지 않습니다.
  - 이를 통해 다이얼로그들이 서로 겹치지 않고 순차적으로 표시될 수 있도록 보장합니다.

### 3.4 `messages/GeminiMessage.tsx`

- **역할**: Gemini 모델이 생성한 응답 메시지를 화면에 표시하는 단순하고 재사용 가능한 컴포넌트입니다.
- **입력 (Props)**:
  - `text`: `string` (모델이 생성한 텍스트 내용)
  - `isPending`: `boolean` (응답이 아직 스트리밍 중인지 여부)
  - `terminalWidth`: `number` (터미널 너비)
- **처리 및 출력**:
  - 메시지 앞에 `✦` 접두사를 붙여 사용자의 메시지와 시각적으로 구분합니다.
  - 입력받은 `text`를 `MarkdownDisplay` 컴포넌트에 전달하여 마크다운 형식으로 렌더링합니다. 이를 통해 코드 블록, 목록, 강조 등 서식이 있는 텍스트를 터미널에 아름답게 표시할 수 있습니다.
  - 스크린 리더 사용자를 위해 `aria-label`을 제공하여 접근성을 준수합니다.
