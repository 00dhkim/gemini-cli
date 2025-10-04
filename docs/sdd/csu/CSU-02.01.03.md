# CSU-02.01.03: 라우팅 전략

## 1. CSU 간단 설명

`routing/` 디렉터리에 위치한 이 CSU는 사용자의 질문 의도에 따라 최적의 Gemini 모델(예: 빠르고 간단한 `Flash` 모델 또는 강력하고 복잡한 `Pro` 모델)을 동적으로 선택하는 "모델 라우터"의 로직을 담당합니다. 이를 통해 간단한 작업은 더 빠르고 저렴한 모델로 처리하고, 복잡한 작업은 더 고성능의 모델로 처리하여 비용과 성능을 최적화합니다.

이 CSU는 Strategy 디자인 패턴을 기반으로 구현되어 있으며, 여러 라우팅 전략을 조합하여 최종적으로 사용할 모델을 결정합니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 파일 및 역할

| 파일명                  | 설명                                                                                                |
| ----------------------- | --------------------------------------------------------------------------------------------------- |
| `routingStrategy.ts`    | 모든 라우팅 전략이 준수해야 하는 `RoutingStrategy` 인터페이스를 정의합니다.                         |
| `modelRouterService.ts` | 여러 라우팅 전략을 조합하고 실행하여 최종 모델 결정을 내리는 메인 서비스 클래스입니다.              |
| `strategies/`           | `ClassifierStrategy`, `DefaultStrategy` 등 구체적인 라우팅 전략의 구현체를 포함하는 디렉터리입니다. |

## 3. 각 구성요소에 대한 입출력 및 함수 처리 설명

### 3.1 `routingStrategy.ts`

- **역할**: 모든 라우팅 전략의 기본 계약(Interface)을 정의합니다.
- **주요 인터페이스**: `RoutingStrategy`
  - **`name: string`**: 전략의 고유한 이름.
  - **`route(context, config, client)`**: 라우팅을 수행하는 핵심 메서드. `RoutingDecision`(모델 결정) 또는 `null`(결정 보류)을 반환합니다.
- **주요 타입**: `RoutingDecision`
  - **`model: string`**: 선택된 모델의 이름 (예: `gemini-1.5-flash-latest`).
  - **`metadata: object`**: 결정의 근거, 지연 시간 등 로깅을 위한 메타데이터.

### 3.2 `modelRouterService.ts`

- **역할**: 라우팅 전략들을 관리하고 실행하여 최종 모델을 결정하는 중앙 오케스트레이터입니다.
- **주요 클래스**: `ModelRouterService`
- **주요 메서드**: `route(context)`
- **함수 처리 설명**:
  1.  **전략 초기화**: `constructor`에서 `CompositeStrategy`를 사용하여 여러 전략을 우선순위 순서대로 조합합니다. 현재 우선순위는 `Fallback` -> `Override` -> `Classifier` -> `Default` 입니다.
  2.  **라우팅 실행**: `route` 메서드가 호출되면, `CompositeStrategy`는 등록된 전략들을 순서대로 실행합니다.
  3.  각 전략은 `route` 메서드를 통해 모델 결정을 시도합니다. 만약 특정 전략이 `null`이 아닌 `RoutingDecision`을 반환하면, `CompositeStrategy`는 그 결정을 즉시 채택하고 나머지 전략들은 실행하지 않습니다.
  4.  최종적으로 결정된 `RoutingDecision`을 반환합니다. 만약 라우팅 과정에서 오류가 발생하면, 오류를 로깅하고 예외를 다시 던집니다.

### 3.3 `strategies/classifierStrategy.ts`

- **역할**: 이 CSU의 핵심 두뇌. 경량 모델(`Flash-Lite`)을 사용하여 사용자의 프롬프트 복잡도를 "분류"하고, 그 결과에 따라 `Flash` 모델을 쓸지 `Pro` 모델을 쓸지 결정하는 가장 중요한 전략입니다.
- **주요 클래스**: `ClassifierStrategy`
- **`route()` 메서드 처리 설명**:
  1.  **분류용 프롬프트 생성**: 사용자의 최근 대화 내역과 현재 프롬프트를 조합합니다.
  2.  **경량 모델 호출**: `DEFAULT_GEMINI_FLASH_LITE_MODEL`을 사용하여, 미리 정의된 `CLASSIFIER_SYSTEM_PROMPT`와 함께 모델을 호출합니다. 이 시스템 프롬프트는 모델에게 사용자 요청의 복잡도를 분석하여 `flash` 또는 `pro` 중 하나를 선택하고, 그 이유를 JSON 형식으로 응답하도록 지시합니다.
  3.  **JSON 응답 파싱**: 경량 모델의 JSON 응답(`{ "reasoning": "...", "model_choice": "pro" }`)을 파싱합니다.
  4.  **결정 반환**: 파싱된 `model_choice` 값에 따라 `RoutingDecision` 객체를 생성하여 반환합니다.
      - `flash`일 경우: `DEFAULT_GEMINI_FLASH_MODEL`을 사용하도록 결정합니다.
      - `pro`일 경우: `DEFAULT_GEMINI_MODEL`을 사용하도록 결정합니다.
  5.  만약 이 과정에서 어떤 오류라도 발생하면, `null`을 반환하여 `ModelRouterService`가 다음 우선순위의 전략(예: `DefaultStrategy`)을 시도하도록 합니다.

## 4. 기타 참고사항

- **Strategy 패턴**: 이 CSU는 전형적인 Strategy 및 Chain of Responsibility 패턴의 조합을 보여줍니다. 각 전략은 독립적으로 라우팅 로직을 캡슐화하며, `ModelRouterService`는 이들을 체인처럼 연결하여 순차적으로 실행합니다.
- **메타인지(Meta-cognition)**: `ClassifierStrategy`는 더 작은 AI 모델을 사용해 "어떤 AI 모델을 사용할지 결정"하는 일종의 메타인지(meta-cognition) 또는 자기 성찰적 AI 아키텍처를 구현한 예시입니다.
