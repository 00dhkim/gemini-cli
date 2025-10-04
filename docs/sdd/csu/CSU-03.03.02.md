# CSU-03.03.02: Google 웹 검색

## 1. CSU 간단 설명

`tools/web-search.ts`에 위치한 이 CSU는 모델(LLM)이 주어진 쿼리(query)로 Google 검색을 수행하고, 그 결과를 바탕으로 요약된 답변을 얻을 수 있도록 하는 `google_web_search` 도구를 구현합니다.

이 도구는 CLI 환경에서 직접 웹을 크롤링하는 것이 아니라, Gemini API에 내장된 Google 검색 연동 기능(`googleSearch: {}`)을 활용합니다. API는 검색을 수행하고 검색 결과를 바탕으로 답변을 생성하며, 이 도구는 API가 반환한 답변에 출처(Source) 및 인용(Citation) 정보를 추가하여 신뢰도를 높이는 후처리(post-processing) 역할을 담당합니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 클래스

| 클래스명                  | 설명                                                                                   |
| ------------------------- | -------------------------------------------------------------------------------------- |
| `WebSearchTool`           | `google_web_search` 도구의 정의(이름, 설명, 파라미터 스키마)를 담당하는 클래스입니다.  |
| `WebSearchToolInvocation` | `google_web_search` 도구가 실제로 호출되었을 때, 그 실행 로직을 담당하는 클래스입니다. |

## 3. 각 클래스에 대한 입출력 및 함수 처리 설명

### 3.1 `WebSearchTool` 클래스

- **역할**: 모델에게 `google_web_search` 도구의 사용법을 알려주는 명세(specification)를 정의합니다.
- **주요 로직**:
  - **`constructor()`**: 도구의 이름(`google_web_search`), 설명, 그리고 `query`라는 단일 파라미터의 JSON 스키마를 설정합니다.
  - **`validateToolParamValues(params)`**: 모델이 전달한 `query` 파라미터가 비어있지 않은지 검증합니다.
  - **`createInvocation(params)`**: 파라미터 유효성 검증이 끝나면, 실제 실행 로직을 담고 있는 `WebSearchToolInvocation` 인스턴스를 생성하여 반환합니다.

### 3.2 `WebSearchToolInvocation` 클래스

- **역할**: Gemini API의 검색 기능을 호출하고, 반환된 결과에 인용 및 출처 정보를 추가하여 최종 응답을 가공합니다.

#### `execute()` 메서드

- **입력**: 없음 (필요한 파라미터는 생성자에서 `this.params`로 전달받음)
- **출력**: `Promise<WebSearchToolResult>` (검색 결과 및 출처 정보를 포함한 객체)
- **함수 처리 설명**:
  1.  **API 호출**: `geminiClient.generateContent`를 호출합니다. 이때 `tools` 옵션에 `{ googleSearch: {} }`를 포함하여 전송합니다. 이 옵션은 Gemini API에게 `this.params.query`를 사용하여 Google 검색을 수행하고, 그 결과를 바탕으로 답변을 생성하라는 신호입니다.
  2.  **응답 파싱**: API 응답에는 모델이 생성한 요약 텍스트(`responseText`)와 함께, 답변의 근거가 된 웹 페이지 정보를 담은 `groundingMetadata`가 포함됩니다.
  3.  **인용(Citation) 처리**:
      - `groundingMetadata` 내의 `groundingSupports` 배열을 순회합니다. 이 배열에는 답변의 특정 문장(segment)이 어떤 출처(chunk)에서 왔는지를 매핑하는 정보가 들어있습니다.
      - 이 정보를 바탕으로, 모델이 생성한 `responseText`의 적절한 위치에 `[1]`, `[2]`와 같은 인용 마커를 삽입합니다. (API가 반환하는 인덱스는 UTF-8 바이트 기준이므로, `TextEncoder`를 사용하여 정확한 위치에 삽입합니다.)
  4.  **출처(Source) 목록 생성**:
      - `groundingMetadata` 내의 `groundingChunks` 배열(각 출처 웹 페이지의 제목과 URI 정보)을 순회하며, 사람이 읽기 좋은 형태의 출처 목록 문자열을 생성합니다. (예: `[1] Google - https://google.com`)
  5.  **최종 결과 조합**: 인용 마커가 삽입된 텍스트와, 그 끝에 추가된 출처 목록을 합쳐 최종 `llmContent`를 생성하고, 이를 `WebSearchToolResult` 객체에 담아 반환합니다.

## 4. 기타 참고사항

- **Grounded Generation**: 이 도구는 Gemini API의 Grounded Generation(근거 기반 생성) 기능을 활용하는 대표적인 예시입니다. 모델이 생성한 답변이 어떤 외부 정보에 기반했는지 명확히 보여줌으로써, 답변의 신뢰성과 검증 가능성을 크게 높입니다.
- **클라이언트의 역할**: 이 CSU의 코드는 직접 웹 검색을 수행하지 않습니다. 대신, API가 제공하는 풍부한 메타데이터를 가공하여 사용자에게 더 가치 있는 정보(인용, 출처)를 제공하는 클라이언트 측 후처리 로직에 집중합니다.
