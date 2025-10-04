# CSU-01.01.02: 비대화형 CLI 처리

## 1. CSU 간단 설명

`nonInteractiveCli.ts`는 Gemini CLI가 비대화형(non-interactive) 모드로 실행될 때의 핵심 로직을 담당하는 소프트웨어 단위입니다. 사용자가 `-p` 플래그나 표준 입력(stdin)을 통해 프롬프트를 직접 제공하면, 이 CSU가 해당 입력을 받아 Gemini 모델과 상호작용한 후, 결과를 표준 출력(stdout)으로 내보내고 애플리케이션을 종료합니다.

주요 역할은 사용자 입력 처리, 모델 호출, 도구(Tool) 실행, 결과 출력 및 오류 관리를 포함하며, 대화형 세션 없이 단일 요청-응답 사이클을 처리하는 데 특화되어 있습니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 함수

| 함수명                | 설명                                                       |
| --------------------- | ---------------------------------------------------------- |
| `runNonInteractive()` | 비대화형 모드의 전체 실행 흐름을 관리하는 메인 함수입니다. |

## 3. 각 함수에 대한 입출력 및 함수 처리 설명

### 3.1 `runNonInteractive()`

- **입력**:
  - `config`: `Config` 객체 (CLI의 전체 설정)
  - `settings`: `LoadedSettings` 객체 (사용자 설정)
  - `input`: `string` (사용자가 제공한 프롬프트 문자열)
  - `prompt_id`: `string` (해당 요청을 식별하기 위한 고유 ID)
- **출력**: `Promise<void>`
- **함수 처리 설명**:
  이 함수는 단일 프롬프트 처리를 위한 루프(loop) 기반의 알고리즘을 사용합니다.
  1.  **초기화**:
      - `promptIdContext.run()`을 통해 모든 로직을 특정 프롬프트 ID의 컨텍스트 내에서 실행합니다.
      - `ConsolePatcher`를 활성화하여 콘솔 출력을 관리합니다.

  2.  **입력 처리**:
      - `isSlashCommand()`로 입력이 `/help`와 같은 슬래시 명령어인지 확인하고, 맞다면 `handleSlashCommand()`로 처리합니다.
      - 슬래시 명령어가 아니라면, `handleAtCommand()`를 통해 `@file.txt`와 같이 파일 내용을 프롬프트에 포함시키는 `@` 명령어를 처리합니다.
      - 처리 과정에서 오류가 발생하면 `FatalInputError`를 발생시켜 즉시 종료합니다.

  3.  **메인 실행 루프 (`while (true)`)**:
      - 이 루프는 모델이 도구 호출(Tool Call)을 요청할 때마다 반복 실행됩니다.
      - **턴(Turn) 제한**: `config.getMaxSessionTurns()` 설정에 따라 최대 실행 횟수를 초과하면 오류를 발생시키고 종료합니다.
      - **모델 호출**: `geminiClient.sendMessageStream()`을 호출하여 현재까지의 메시지(`currentMessages`)를 Gemini API로 전송하고, 응답을 스트림으로 수신합니다.
      - **응답 스트림 처리**:
        - **콘텐츠 (`GeminiEventType.Content`)**: 모델이 생성하는 텍스트 응답입니다. 이 내용을 `process.stdout`을 통해 실시간으로 출력합니다. (JSON 출력 모드일 경우, 텍스트를 변수에 누적합니다.)
        - **도구 호출 요청 (`GeminiEventType.ToolCallRequest`)**: 모델이 특정 도구를 사용해야겠다고 판단하면, 해당 요청 정보를 `toolCallRequests` 배열에 수집합니다.

  4.  **도구 실행 및 루프 제어**:
      - **도구 호출이 없을 경우**: 모델의 응답이 완료된 것으로 간주합니다. 최종 결과를 포맷(JSON 또는 일반 텍스트)에 맞춰 출력하고, `return`을 통해 함수와 루프를 모두 종료합니다.
      - **도구 호출이 있을 경우**:
        - `toolCallRequests` 배열을 순회하며 `executeToolCall()` 함수를 호출하여 각 도구를 실행합니다.
        - 도구 실행 중 오류가 발생하면 `handleToolError()`로 처리합니다.
        - 모든 도구의 실행 결과를 `toolResponseParts`에 수집한 후, 이 결과를 새로운 사용자 메시지로 구성하여 `currentMessages`를 업데이트합니다. 루프의 다음 반복(iteration)으로 돌아가 업데이트된 메시지를 모델에 다시 전송합니다.

  5.  **오류 처리 및 정리**:
      - `try...catch` 블록을 통해 실행 중 발생하는 모든 예외를 `handleError()` 함수로 전달하여 일관되게 처리합니다.
      - `finally` 블록에서 `consolePatcher.cleanup()`과 `shutdownTelemetry()`를 호출하여 사용된 리소스를 안전하게 정리하고 애플리케이션을 종료합니다.

## 4. 기타 참고사항

- 이 CSU는 사용자와의 지속적인 상호작용 없이 단발성 작업을 자동화하는 스크립트나 파이프라인에 `gemini-cli`를 연동할 때 핵심적인 역할을 합니다.
- 모델이 여러 차례 도구를 호출해야 하는 복잡한 작업(e.g., 파일 읽기 -> 내용 분석 -> 파일 쓰기)도 메인 실행 루프를 통해 처리할 수 있습니다.
