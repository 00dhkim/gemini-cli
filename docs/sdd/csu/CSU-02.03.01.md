# CSU-02.03.01: Gemini API 서비스

## 1. CSU 간단 설명

이 CSU는 Gemini API와의 모든 통신을 담당하는 핵심 서비스 계층입니다. `SDD.md`에 명시된 `services/gemini.ts` 파일은 현재 코드베이스에서 `core/baseLlmClient.ts`와 `core/geminiChat.ts` 두 파일로 나뉘어 구현되어 있습니다. 이들은 각각 저수준(low-level) API 호출과 고수준(high-level) 대화 상태 관리를 담당하며, 함께 Gemini API 클라이언트의 역할을 수행합니다.

- **`BaseLlmClient`**: 상태 없이(stateless) 재사용 가능한 유틸리티성 API 호출(JSON 생성, 임베딩 등)을 담당합니다.
- **`GeminiChat`**: 대화의 맥락(history)을 유지하며, 사용자와 모델 간의 스트리밍 대화를 관리합니다.

> _참고: SDD에 명시된 `services/gemini.ts` 파일은 `core/geminiChat.ts`와 `core/baseLlmClient.ts`로 리팩토링된 것으로 보입니다. 본 문서는 이 두 파일을 기준으로 작성되었습니다._

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 클래스

| 클래스명        | 파일               | 설명                                                                      |
| --------------- | ------------------ | ------------------------------------------------------------------------- |
| `BaseLlmClient` | `baseLlmClient.ts` | 상태 없는(stateless) 저수준 API 호출 클라이언트입니다.                    |
| `GeminiChat`    | `geminiChat.ts`    | 대화의 맥락(history)을 유지하는 고수준(high-level) 채팅 클라이언트입니다. |

## 3. 각 클래스에 대한 입출력 및 함수 처리 설명

### 3.1 `BaseLlmClient` 클래스

- **역할**: 특정 대화의 맥락에 얽매이지 않는, 일회성의 유틸리티성 API 호출을 수행합니다. 주로 내부적인 작업(예: 모델 라우팅 결정)에 사용됩니다.

#### `generateJson(options)` 메서드

- **입력**: `options: GenerateJsonOptions` (프롬프트 내용, JSON 스키마, 모델 이름, 취소 신호 등)
- **출력**: `Promise<Record<string, unknown>>` (모델이 생성한 JSON 객체)
- **함수 처리 설명**:
  1.  `responseMimeType`을 `application/json`으로 설정하고, 입력받은 `schema`를 `responseJsonSchema`로 설정하여 API에 JSON 출력을 강제합니다.
  2.  `contentGenerator.generateContent`를 호출하여 API 통신을 수행합니다.
  3.  API 응답이 비어있거나 유효한 JSON이 아닐 경우, `retryWithBackoff` 로직에 따라 최대 5회까지 재시도를 수행합니다.
  4.  모델이 응답 앞뒤에 ` ```json ... ``` `과 같은 마크다운 래퍼(wrapper)를 포함했을 경우, `cleanJsonResponse`를 통해 이를 제거하고 순수한 JSON 문자열만 파싱하여 반환합니다.
  5.  최종적으로 재시도에도 실패할 경우, 오류를 보고(`reportError`)하고 예외를 던집니다.

### 3.2 `GeminiChat` 클래스

- **역할**: 사용자와 모델 간의 대화 세션을 관리합니다. 대화 내역(`history`)을 내부에 저장하고, 이를 바탕으로 연속적인 대화를 가능하게 합니다.

#### `sendMessageStream(model, params, prompt_id)` 메서드

- **입력**:
  - `model`: 사용할 모델 이름
  - `params`: `SendMessageParameters` (전송할 메시지, 생성 설정 등)
  - `prompt_id`: 로깅을 위한 프롬프트 고유 ID
- **출력**: `Promise<AsyncGenerator<StreamEvent>>` (모델의 응답 스트림)
- **함수 처리 설명**:
  1.  **대화 내역 관리**: 사용자의 메시지(`params.message`)를 내부 `history` 배열에 추가합니다.
  2.  **API 호출 및 재시도**: `makeApiCallAndProcessStream`을 통해 실제 API 호출을 수행합니다. 이때 `retryWithBackoff` 로직을 사용하여 429(Too Many Requests)나 5xx 서버 오류 발생 시 자동으로 재시도합니다.
  3.  **스트림 처리**: `processStreamResponse` 메서드 내에서 `for await...of` 루프를 통해 API 응답 스트림을 실시간으로 처리합니다.
      - 각 `chunk`를 즉시 `yield`하여 UI가 실시간으로 응답을 표시할 수 있게 합니다.
      - 스트림이 끝날 때까지 모델이 생성한 모든 `Part`들을 `modelResponseParts` 배열에 수집합니다.
  4.  **응답 유효성 검사**: 스트림이 모두 끝난 후, 응답이 유효한지(예: `finishReason`이 있는지, 텍스트 내용이 비어있지 않은지) 검사합니다. 유효하지 않을 경우, `InvalidStreamError`를 발생시켜 재시도 로직을 트리거합니다.
  5.  **최종 대화 내역 추가**: 유효한 응답이 확인되면, 수집된 `modelResponseParts`를 `role: 'model'`로 하여 `history` 배열에 추가함으로써 대화의 한 턴(turn)을 완료합니다.

## 4. 아키텍처 및 기타 참고사항

- **2계층 구조**: 이 CSU는 저수준 통신을 담당하는 `BaseLlmClient`와 고수준 대화 상태를 관리하는 `GeminiChat`의 2계층 구조로 설계되어 있습니다. 이를 통해 역할과 책임을 명확히 분리합니다.
- **오류 처리 및 복원성**: `retryWithBackoff` 메커니즘을 적극적으로 사용하여 일시적인 네트워크 오류나 API의 비정상 응답에 대해 높은 복원성(resilience)을 가집니다.
- **상태 관리**: `GeminiChat`은 대화의 상태(history)를 클래스 내부에 유지하는 상태 저장(stateful) 클라이언트인 반면, `BaseLlmClient`는 상태를 저장하지 않고(stateless) 매 호출이 독립적인 유틸리티성 클라이언트입니다.
