# CSU-02.02.02: 사용자 확인 버스

## 1. CSU 간단 설명

`confirmation-bus/` 디렉터리에 위치한 이 CSU는 민감한 도구(Tool) 실행 전, 사용자에게 확인을 요청하고 그 응답을 처리하기 위한 메시지 버스(Message Bus) 시스템을 구현합니다. Node.js의 `EventEmitter`를 기반으로 한 게시/구독(Publish/Subscribe) 패턴을 사용하여, 도구 실행 로직과 사용자 인터페이스(UI) 간의 의존성을 분리(decoupling)하는 역할을 합니다.

특히, 이 버스는 `PolicyEngine`과 통합되어, 모든 확인 요청을 사용자에게 전달하기 전에 먼저 정책을 확인합니다. 정책에 따라 요청이 자동 승인되거나 거부될 수 있으며, 정책이 "사용자에게 질문(ASK_USER)"으로 설정된 경우에만 UI로 확인 요청이 전달됩니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 클래스

| 클래스명     | 설명                                                                                                         |
| ------------ | ------------------------------------------------------------------------------------------------------------ |
| `MessageBus` | 메시지를 게시(publish)하고 구독(subscribe)하는 중앙 이벤트 버스 클래스입니다. `EventEmitter`를 상속받습니다. |

### 주요 타입 (`types.ts`)

| 타입/열거형                | 설명                                                                                                                                 |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `MessageBusType`           | 버스를 통해 전달될 수 있는 메시지의 종류를 정의한 열거형(Enum)입니다. (`TOOL_CONFIRMATION_REQUEST`, `TOOL_CONFIRMATION_RESPONSE` 등) |
| `ToolConfirmationRequest`  | 도구 실행 확인을 요청하는 메시지의 구조를 정의합니다. `correlationId`를 포함하여 요청과 응답을 짝지을 수 있습니다.                   |
| `ToolConfirmationResponse` | 확인 요청에 대한 응답(승인/거부) 메시지의 구조를 정의합니다.                                                                         |
| `Message`                  | 버스에서 사용되는 모든 메시지 타입의 유니온(Union) 타입입니다.                                                                       |

## 3. 각 구성요소에 대한 입출력 및 함수 처리 설명

### 3.1 `MessageBus` 클래스

- **역할**: 도구 실행 확인과 관련된 모든 메시지를 중개하는 중앙 허브입니다.

#### `publish(message: Message)` 메서드

- **입력**: `message: Message` (게시할 메시지 객체)
- **출력**: `void`
- **함수 처리 설명**:
  1.  **메시지 유효성 검사**: `isValidMessage`를 통해 메시지가 올바른 구조를 가졌는지 확인합니다.
  2.  **확인 요청 처리 (`TOOL_CONFIRMATION_REQUEST`)**:
      - 메시지 타입이 확인 요청일 경우, `policyEngine.check()`를 호출하여 해당 도구 호출(`message.toolCall`)에 대한 정책을 확인합니다.
      - **정책 결정에 따른 분기 처리**:
        - `PolicyDecision.ALLOW` (허용): 사용자에게 묻지 않고, 즉시 `confirmed: true`로 설정된 `TOOL_CONFIRMATION_RESPONSE` 메시지를 버스에 게시합니다.
        - `PolicyDecision.DENY` (거부): `TOOL_POLICY_REJECTION` 메시지와 `confirmed: false`로 설정된 `TOOL_CONFIRMATION_RESPONSE` 메시지를 모두 게시합니다.
        - `PolicyDecision.ASK_USER` (사용자에게 질문): 받은 `TOOL_CONFIRMATION_REQUEST` 메시지를 그대로 버스에 게시하여, UI와 같은 구독자(subscriber)가 처리하도록 합니다.
  3.  **기타 메시지 처리**: 확인 요청이 아닌 다른 모든 타입의 메시지는 즉시 버스에 게시됩니다.

#### `subscribe<T extends Message>(type, listener)` 메서드

- **입력**:
  - `type: T['type']` (구독할 메시지의 타입)
  - `listener: (message: T) => void` (메시지 수신 시 호출될 콜백 함수)
- **출력**: `void`
- **함수 처리 설명**: `EventEmitter`의 `on` 메서드를 사용하여 특정 타입의 메시지가 게시될 때마다 `listener` 함수가 실행되도록 등록합니다. UI는 이 메서드를 사용하여 `TOOL_CONFIRMATION_REQUEST`를 구독합니다.

### 3.2 사용자 확인 흐름 (Lifecycle of a Confirmation)

1.  **요청 (Request)**: 도구 실행기(`nonInteractiveToolExecutor.ts`)가 민감한 도구를 실행하기 전에, `correlationId`를 포함한 `ToolConfirmationRequest` 메시지를 생성하여 `MessageBus`에 `publish`합니다.
2.  **정책 확인 (Policy Check)**: `MessageBus`는 `PolicyEngine`을 통해 이 요청을 검사합니다. 정책이 `ALLOW` 또는 `DENY`이면, 버스는 즉시 해당 `ToolConfirmationResponse`를 게시하고 흐름이 4단계로 넘어갑니다.
3.  **UI 처리 (UI Handling)**: 정책이 `ASK_USER`이면, `TOOL_CONFIRMATION_REQUEST` 메시지가 버스에 게시됩니다. UI(예: `AppContainer.tsx`)는 이 메시지를 `subscribe`하고 있다가, 사용자에게 확인 다이얼로그(예: `ShellConfirmationDialog`)를 띄워줍니다. 사용자가 "Yes" 또는 "No"를 선택하면, UI는 해당 `correlationId`와 `confirmed` 값을 담은 `ToolConfirmationResponse` 메시지를 `publish`합니다.
4.  **응답 처리 (Response)**: 도구 실행기는 `TOOL_CONFIRMATION_RESPONSE` 메시지를 구독하고 있습니다. 자신의 `correlationId`와 일치하는 응답을 받으면, `confirmed` 값에 따라 도구를 실제로 실행하거나 실행을 취소합니다.

## 4. 기타 참고사항

- **디커플링(Decoupling)**: 이 메시지 버스는 도구 실행 로직이 UI의 존재나 구현 방식에 대해 전혀 알 필요가 없도록 만듭니다. 도구는 단지 확인을 요청하고 응답을 기다리기만 하면 됩니다. 이는 시스템의 모듈성을 크게 향상시킵니다.
- **정책 우선주의**: 모든 확인 요청은 사용자에게 도달하기 전에 반드시 정책 엔진을 통과합니다. 이를 통해 관리자는 특정 도구의 사용을 강제로 허용하거나 금지할 수 있습니다.
