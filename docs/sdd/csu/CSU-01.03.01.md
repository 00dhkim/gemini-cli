# CSU-01.03.01: 인증 흐름 관리

## 1. CSU 간단 설명

`ui/auth/` 디렉터리에 위치한 이 CSU는 사용자의 인증 과정을 처리하는 모든 UI와 관련 로직을 담당합니다. 사용자가 CLI를 처음 실행하거나 인증이 필요할 때, 이 CSU의 컴포넌트들이 화면에 표시되어 인증 방법을 선택하고 자격 증명을 확인하는 흐름을 안내합니다.

주요 기능으로는 인증 방법(Google 로그인, API 키 등)을 선택하는 다이얼로그 표시, 인증 진행 중 상태 표시, 그리고 실제 인증 로직을 처리하는 Hook이 포함됩니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 컴포넌트

| 컴포넌트명       | 설명                                                                                  |
| ---------------- | ------------------------------------------------------------------------------------- |
| `AuthDialog`     | 사용자에게 인증 방법을 선택하도록 제시하는 메인 다이얼로그 컴포넌트입니다.            |
| `AuthInProgress` | 인증이 진행되는 동안 "Waiting for auth..." 메시지와 스피너를 표시하는 컴포넌트입니다. |

### 주요 Hook

| Hook명           | 설명                                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------------------------- |
| `useAuthCommand` | 인증 상태를 관리하고, 선택된 인증 방법에 따라 실제 인증 절차를 트리거하는 핵심 로직을 담고 있습니다. |

## 3. 각 구성요소에 대한 입출력 및 함수 처리 설명

### 3.1 `AuthDialog.tsx`

- **역할**: 사용자가 여러 인증 방법 중 하나를 선택할 수 있는 UI를 제공합니다.
- **입력 (Props)**:
  - `settings`: `LoadedSettings` (현재 설정 값)
  - `setAuthState`: 인증 상태를 변경하는 함수.
  - `authError`: 표시할 인증 오류 메시지.
  - `onAuthError`: 오류 발생 시 호출될 콜백 함수.
- **처리 및 출력**:
  - `RadioButtonSelect` 컴포넌트를 사용하여 "Login with Google", "Use Gemini API Key" 등의 선택지를 나열합니다.
  - 환경 변수(`CLOUD_SHELL`)나 설정(`enforcedType`)에 따라 표시되는 선택지가 동적으로 변경될 수 있습니다.
  - 사용자가 특정 인증 방법을 선택(`onSelect`)하면, `settings.setValue`를 통해 선택된 방법을 설정 파일에 저장하고, `setAuthState`를 호출하여 `useAuthCommand` Hook이 다음 단계를 진행하도록 트리거합니다.
  - `authError` prop에 값이 있으면, 다이얼로그 하단에 오류 메시지를 표시합니다.

### 3.2 `AuthInProgress.tsx`

- **역할**: 백그라운드에서 인증이 진행되는 동안 사용자에게 대기 상태임을 알려주는 간단한 UI를 제공합니다.
- **입력 (Props)**:
  - `onTimeout`: 인증 시간이 초과되었을 때 호출될 콜백 함수.
- **처리 및 출력**:
  - `ink-spinner`를 사용하여 "Waiting for auth..." 메시지와 함께 로딩 스피너를 표시합니다.
  - `useEffect`를 사용하여 180초의 타임아웃 타이머를 설정합니다. 시간이 초과되면 `onTimeout` 콜백을 호출하고 "Authentication timed out" 메시지를 표시합니다.
  - 사용자가 ESC나 Ctrl+C를 누르면 `onTimeout`을 즉시 호출하여 인증을 취소할 수 있습니다.

### 3.3 `useAuth.ts` (`useAuthCommand` Hook)

- **역할**: 인증 상태(Unauthenticated, Authenticated 등)를 관리하고, 설정된 인증 방법에 따라 실제 인증 로직을 실행합니다.
- **관리하는 상태**:
  - `authState`: `AuthState` 열거형 (현재 인증 상태)
  - `authError`: `string | null` (인증 과정에서 발생한 오류 메시지)
- **반환하는 값 (인터페이스)**:
  - `authState`, `setAuthState`, `authError`, `onAuthError`
- **주요 로직 (`useEffect`)**:
  1.  `authState`가 `Unauthenticated`일 때 로직이 실행됩니다.
  2.  `settings`에서 `selectedType`(선택된 인증 방법)을 가져옵니다. 만약 설정된 값이 없으면 오류를 설정하고 UI를 업데이트합니다.
  3.  `validateAuthMethodWithSettings` 함수를 호출하여 선택된 인증 방법이 현재 설정(예: 환경 변수)과 충돌하지 않는지 확인합니다.
  4.  유효성 검사를 통과하면, `config.refreshAuth(authType)`를 비동기적으로 호출합니다. 이 함수는 `@google/gemini-cli-core` 패키지에 정의된 실제 인증 로직(OAuth 토큰 갱신, API 키 유효성 검사 등)을 수행합니다.
  5.  `refreshAuth`가 성공하면, `authState`를 `Authenticated`로 변경하고 오류를 초기화합니다.
  6.  `refreshAuth`가 실패하면, 예외를 캐치하여 `onAuthError`를 통해 사용자에게 오류 메시지를 표시합니다.

## 4. 기타 참고사항

- 이 세 구성요소는 긴밀하게 함께 동작합니다. `AuthDialog`에서 사용자가 인증 방법을 선택하면 `useAuthCommand`가 이를 감지하고 인증을 시도합니다. 인증이 진행되는 동안에는 `AuthInProgress`가 표시될 수 있으며, 최종 성공 또는 실패 결과는 다시 `useAuthCommand`에 의해 관리되고 UI에 반영됩니다.
