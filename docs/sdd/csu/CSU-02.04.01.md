# CSU-02.04.01: MCP 클라이언트

## 1. CSU 간단 설명

이 CSU는 Model Context Protocol(MCP) 서버와 통신하여, 해당 서버가 제공하는 도구(Tool)와 프롬프트(Prompt)를 동적으로 발견하고 `gemini-cli`에 통합하는 클라이언트 로직을 담당합니다. `SDD.md`에 명시된 `mcp/mcp-client.ts` 파일은 현재 `tools/` 디렉터리 내의 `mcp-client.ts`와 `mcp-client-manager.ts` 두 파일로 구현되어 있습니다.

- **`mcp-client.ts`**: 단일 MCP 서버와의 연결, 통신, 도구/프롬프트 발견 등 저수준(low-level) 로직을 처리합니다.
- **`mcp-client-manager.ts`**: 설정된 모든 MCP 서버를 총괄하며, 각 서버에 대한 `McpClient` 인스턴스의 생명주기를 관리하고 전체 발견 프로세스를 오케스트레이션합니다.

> _참고: SDD에 명시된 파일 경로는 `tools/`로 변경되었습니다. 본 문서는 `mcp-client.ts`와 `mcp-client-manager.ts`를 기준으로 작성되었습니다._

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 클래스

| 클래스명           | 파일                    | 설명                                                         |
| ------------------ | ----------------------- | ------------------------------------------------------------ |
| `McpClientManager` | `mcp-client-manager.ts` | 여러 `McpClient` 인스턴스를 관리하는 상위 레벨 관리자입니다. |
| `McpClient`        | `mcp-client.ts`         | 단일 MCP 서버와의 통신을 담당하는 저수준 클라이언트입니다.   |

## 3. 각 클래스에 대한 입출력 및 함수 처리 설명

### 3.1 `McpClientManager` 클래스

- **역할**: 설정된 모든 MCP 서버에 대한 연결 및 도구 발견 프로세스를 총괄합니다.

#### `discoverAllMcpTools(cliConfig)` 메서드

- **입력**: `cliConfig: Config` (애플리케이션 전역 설정)
- **출력**: `Promise<void>`
- **함수 처리 설명**:
  1.  기존에 연결된 모든 클라이언트를 `stop()` 메서드를 통해 종료하고 초기화합니다.
  2.  `config`에서 MCP 서버 목록(`mcpServers`)을 가져옵니다.
  3.  각 서버 설정에 대해 `McpClient` 인스턴스를 생성합니다.
  4.  `Promise.all`을 사용하여 모든 서버에 대해 병렬로 다음 작업을 수행합니다:
      a. `client.connect()`: 서버에 연결을 시도합니다.
      b. `client.discover(cliConfig)`: 연결된 서버로부터 도구와 프롬프트를 가져옵니다.
  5.  발견된 도구와 프롬프트는 `McpClient` 내부에서 `ToolRegistry`와 `PromptRegistry`에 자동으로 등록됩니다.
  6.  하나의 서버에서 오류가 발생하더라도 다른 서버의 발견 프로세스는 중단되지 않습니다.

### 3.2 `McpClient` 클래스

- **역할**: 단일 MCP 서버와의 연결 설정, 도구/프롬프트 발견, 연결 해제 등 실질적인 통신을 담당합니다.

#### `connect()` 메서드

- **입력**: 없음
- **출력**: `Promise<void>`
- **함수 처리 설명**:
  1.  `createTransport()`를 호출하여 서버 설정(`mcpServerConfig`)에 맞는 통신 전송(Transport) 객체를 생성합니다. 서버 설정에 따라 `StdioClientTransport`(로컬 명령어 실행), `SSEClientTransport`(Server-Sent Events), `StreamableHTTPClientTransport`(HTTP) 중 하나가 선택됩니다.
  2.  `@modelcontextprotocol/sdk`의 `client.connect()` 메서드를 호출하여 생성된 전송 객체를 통해 실제 서버 연결을 수립합니다.
  3.  연결 상태(`status`)를 `CONNECTING`에서 `CONNECTED`로 업데이트합니다.

#### `discover(cliConfig)` 메서드

- **입력**: `cliConfig: Config`
- **출력**: `Promise<void>`
- **함수 처리 설명**:
  1.  `discoverPrompts()`와 `discoverTools()`를 호출하여 서버로부터 프롬프트와 도구 목록을 가져옵니다.
  2.  가져온 프롬프트는 `promptRegistry.registerPrompt()`를 통해 프롬프트 레지스트리에 등록합니다.
  3.  가져온 도구는 `toolRegistry.registerTool()`를 통해 도구 레지스트리에 등록합니다.
  4.  만약 발견된 프롬프트나 도구가 하나도 없으면 오류를 발생시킵니다.

#### `createTransport()` 메서드 (내부 로직)

- **함수 처리 설명**:
  - `mcpServerConfig`의 내용에 따라 다른 종류의 Transport 객체를 생성합니다.
  - `command` 속성이 있으면: 로컬 셸 명령어를 실행하는 `StdioClientTransport`를 생성합니다.
  - `url` 속성이 있으면: Server-Sent Events(SSE) 방식의 `SSEClientTransport`를 생성합니다.
  - `httpUrl` 속성이 있으면: 일반 HTTP 스트리밍 방식의 `StreamableHTTPClientTransport`를 생성합니다.
  - 인증(`authProviderType`, `oauth`) 설정이 있으면, 해당 인증 방식(Google Credentials, OAuth 등)을 처리하는 Provider를 생성하여 Transport에 포함시킵니다.

## 4. 기타 참고사항

- **프로토콜**: 이 CSU는 `@modelcontextprotocol/sdk` 라이브러리를 사용하여 Model Context Protocol을 구현합니다. 이를 통해 `gemini-cli`는 표준화된 방식으로 다양한 종류의 외부 도구 서버와 연동할 수 있습니다.
- **동적 도구 통합**: 이 클라이언트 덕분에 사용자는 `settings.json`에 서버 정보만 추가하면, 별도의 코드 수정 없이도 외부 서버가 제공하는 새로운 도구들을 `gemini-cli`에서 즉시 사용할 수 있습니다.
- **인증 처리**: `mcp-client.ts`는 Google 서비스 계정, OAuth 등 다양한 인증 방식을 처리하는 로직을 포함하여, 보안이 적용된 MCP 서버와도 안전하게 통신할 수 있습니다.
