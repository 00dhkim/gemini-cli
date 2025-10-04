# CSU-02.03.02: 원격 측정 서비스

## 1. CSU 간단 설명

`telemetry/` 디렉터리에 위치한 이 CSU는 애플리케이션의 사용 데이터를 수집하고 외부로 전송하는 원격 측정(Telemetry) 시스템 전체를 구현합니다. OpenTelemetry(OTLP) 표준을 기반으로 구축되었으며, 에이전트의 동작(API 호출, 도구 사용, 오류 등)을 구조화된 로그와 메트릭으로 기록하여 서비스 품질 분석 및 개선에 사용됩니다.

이 시스템은 설정에 따라 데이터를 Google Cloud로 직접 전송하거나, 로컬 파일에 저장하거나, 콘솔에 출력하는 등 다양한 대상으로 내보낼(export) 수 있습니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 파일 및 역할

| 파일명              | 설명                                                                                                                             |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `config.ts`         | 여러 소스(설정 파일, 환경 변수, CLI 인자)로부터 원격 측정 관련 설정을 수집하고 통합합니다.                                       |
| `sdk.ts`            | 수집된 설정을 바탕으로 OpenTelemetry SDK를 초기화하고, 데이터 처리 및 전송 파이프라인을 구성합니다.                              |
| `loggers.ts`        | `logUserPrompt`, `logToolCall` 등 애플리케이션의 다른 부분에서 이벤트를 쉽게 기록할 수 있도록 추상화된 로깅 함수들을 제공합니다. |
| `gcp-exporters.ts`  | 수집된 데이터를 Google Cloud로 전송하는 OTLP Exporter를 구현합니다.                                                              |
| `file-exporters.ts` | 수집된 데이터를 로컬 파일로 전송하는 OTLP Exporter를 구현합니다.                                                                 |

## 3. 각 구성요소에 대한 입출력 및 함수 처리 설명

### 3.1 `config.ts` (`resolveTelemetrySettings` 함수)

- **역할**: 원격 측정 시스템의 동작 방식을 결정하는 모든 설정을 한곳으로 모으고 해석합니다.
- **입력**: `argv` (CLI 인자), `env` (환경 변수), `settings` (`settings.json`의 내용)
- **출력**: `Promise<TelemetrySettings>` (통합된 원격 측정 설정 객체)
- **함수 처리 설명**:
  - **우선순위**: `CLI 인자 > 환경 변수 > settings.json` 순서의 우선순위를 적용하여 각 설정 값을 결정합니다.
  - **설정 항목**:
    - `enabled`: 원격 측정 활성화 여부.
    - `target`: 데이터 전송 대상 (`gcp` 또는 `local`).
    - `otlpEndpoint`: OTLP 수집기 엔드포인트 주소.
    - `logPrompts`: 사용자 프롬프트 내용을 로그에 포함할지 여부.
    - `outfile`: `local` 타겟일 경우, 로그를 저장할 파일 경로.
  - 유효하지 않은 설정 값(예: `target`이 `gcp`나 `local`이 아닌 경우)이 감지되면 오류를 발생시킵니다.

### 3.2 `sdk.ts` (`initializeTelemetry` 함수)

- **역할**: `resolveTelemetrySettings`를 통해 결정된 설정을 바탕으로 OpenTelemetry SDK의 전체 파이프라인을 구성하고 시작합니다.
- **입력**: `config: Config` (애플리케이션 전역 설정 객체)
- **출력**: `void`
- **함수 처리 설명**:
  1.  **초기화 방지**: 원격 측정이 비활성화되어 있거나 이미 초기화된 경우, 즉시 반환합니다.
  2.  **Exporter 선택**: 설정된 `telemetryTarget`과 `outfile` 값에 따라 데이터를 내보낼 Exporter를 선택합니다.
      - `target`이 `gcp`이면: `GcpLogExporter` 사용.
      - `outfile`이 지정되면: `FileLogExporter` 사용.
      - 둘 다 아니면: `ConsoleLogRecordExporter`를 기본값으로 사용.
  3.  **프로세서(Processor) 구성**: 선택된 Exporter를 `BatchLogRecordProcessor`에 연결합니다. 이 프로세서는 로그를 즉시 보내지 않고, 배치(batch)로 묶어 효율적으로 전송하는 역할을 합니다.
  4.  **SDK 초기화**: `NodeSDK` 인스턴스를 생성하고, 위에서 구성한 프로세서와 리소스 정보(서비스 이름, 세션 ID 등)를 전달합니다.
  5.  **SDK 시작**: `sdk.start()`를 호출하여 원격 측정 파이프라인을 활성화합니다. 이 시점부터 `loggers.ts`를 통해 기록되는 모든 이벤트가 처리되기 시작합니다.

### 3.3 `loggers.ts`

- **역할**: 애플리케이션의 각 부분에서 원격 측정 이벤트를 쉽게 기록할 수 있도록, 목적에 따라 특화된 여러 헬퍼 함수를 제공합니다.
- **주요 함수**:
  - `logUserPrompt(config, event)`: 사용자 프롬프트 정보를 기록합니다.
  - `logToolCall(config, event)`: 도구 호출의 시작, 종료, 성공/실패 여부, 소요 시간 등을 기록합니다.
  - `logApiError(config, event)`: Gemini API 호출 중 발생한 오류를 기록합니다.
  - `logApiResponse(config, event)`: API 응답에 대한 정보(토큰 사용량, 지연 시간 등)를 기록합니다.
- **함수 처리 설명 (예: `logUserPrompt`)**:
  1.  `isTelemetrySdkInitialized()`를 통해 SDK가 활성화 상태인지 확인합니다.
  2.  `logs.getLogger(SERVICE_NAME)`를 통해 OTLP 로거 인스턴스를 가져옵니다.
  3.  `getCommonAttributes(config)`를 호출하여 세션 ID와 같은 공통 속성을 가져옵니다.
  4.  입력받은 `event` 객체의 내용과 공통 속성을 합쳐 `LogRecord` 객체를 생성합니다.
  5.  `logger.emit(logRecord)`를 호출하여 생성된 로그 레코드를 OTLP 파이프라인으로 전송합니다. 이 로그는 `sdk.ts`에서 구성한 프로세서와 Exporter를 통해 최종 목적지로 보내집니다.

## 4. 아키텍처 및 기타 참고사항

- **데이터 흐름**: **`loggers.ts` (이벤트 발생 및 기록)** -> **`sdk.ts` (수집 및 처리)** -> **`*-exporters.ts` (외부 전송)** 의 명확한 파이프라인 구조를 가집니다.
- **추상화**: `loggers.ts`의 함수들은 복잡한 OpenTelemetry API를 추상화합니다. 애플리케이션의 다른 부분에서는 단순히 `logUserPrompt(...)`와 같이 의미 있는 이름의 함수를 호출하기만 하면 되므로, 원격 측정 시스템의 내부 구현을 알 필요가 없습니다.
- **ClearcutLogger**: OTLP와 별개로, Google 내부의 다른 로깅 시스템인 `Clearcut`에 로그를 보내기 위한 `ClearcutLogger`가 함께 사용됩니다. 각 로깅 함수는 OTLP와 Clearcut 양쪽으로 데이터를 보내는 이중 로깅 구조를 가집니다.
