# CSU-02.04.02: MCP 토큰 저장소

## 1. CSU 간단 설명

`mcp/token-storage/` 디렉터리에 위치한 이 CSU는 MCP(Model Context Protocol) 서버 인증에 사용되는 OAuth 토큰을 사용자의 로컬 시스템에 안전하게 저장하고 관리하는 역할을 합니다. 보안을 최우선으로 하여, 운영체제의 안전한 자격증명 관리자(예: macOS의 Keychain, Windows의 Credential Manager)를 우선적으로 사용하며, 이것이 불가능할 경우 암호화된 파일에 저장하는 하이브리드(Hybrid) 방식을 채택하고 있습니다.

이 CSU는 토큰 저장 방식의 구체적인 구현을 추상화하여, 애플리케이션의 다른 부분에서는 일관된 인터페이스로 토큰을 저장하고 조회할 수 있도록 합니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 클래스

| 클래스명               | 파일                        | 설명                                                                                   |
| ---------------------- | --------------------------- | -------------------------------------------------------------------------------------- |
| `BaseTokenStorage`     | `base-token-storage.ts`     | 모든 토큰 저장소 클래스가 상속받아야 하는 추상 기본 클래스(Abstract Base Class)입니다. |
| `HybridTokenStorage`   | `hybrid-token-storage.ts`   | 키체인(Keychain)과 파일 저장을 결합한 기본 구현체입니다.                               |
| `KeychainTokenStorage` | `keychain-token-storage.ts` | 운영체제의 보안 키체인을 사용하여 토큰을 저장하는 구현체입니다.                        |
| `FileTokenStorage`     | `file-token-storage.ts`     | 암호화된 로컬 파일에 토큰을 저장하는 구현체입니다.                                     |

## 3. 각 클래스에 대한 입출력 및 함수 처리 설명

### 3.1 `BaseTokenStorage` 추상 클래스

- **역할**: 모든 토큰 저장소 구현체가 반드시 제공해야 하는 메서드들의 계약(contract)을 정의합니다.
- **주요 추상 메서드**:
  - `getCredentials(serverName)`: 특정 서버의 인증 정보(토큰 포함)를 가져옵니다.
  - `setCredentials(credentials)`: 인증 정보를 저장합니다.
  - `deleteCredentials(serverName)`: 특정 서버의 인증 정보를 삭제합니다.
  - `listServers()`: 저장된 모든 서버의 이름 목록을 반환합니다.
- **공통 유틸리티 메서드**:
  - `isTokenExpired()`: 토큰의 만료 시간을 확인하여 갱신이 필요한지 판단합니다.

### 3.2 `HybridTokenStorage` 클래스

- **역할**: 애플리케이션에서 기본적으로 사용되는 토큰 저장소. 보안과 호환성을 모두 고려하여 최적의 저장 방식을 동적으로 선택합니다.

#### `getStorage()` 메서드 (내부 로직)

- **입력**: 없음
- **출력**: `Promise<TokenStorage>` (실제로 사용할 저장소 인스턴스)
- **함수 처리 설명**:
  1.  **저장소 초기화 확인**: 내부에 `storage` 인스턴스가 이미 초기화되었는지 확인합니다. 이미 있다면 즉시 반환합니다.
  2.  **키체인 우선 시도**: `KeychainTokenStorage`를 동적으로 `import`하여 인스턴스를 생성합니다. `keytar` 라이브러리에 의존하므로, 런타임에 사용 가능 여부를 확인해야 합니다.
  3.  `keychainStorage.isAvailable()`을 호출하여 운영체제의 보안 키체인 사용이 가능한지 확인합니다.
      - **가능할 경우**: `storage`를 `KeychainTokenStorage` 인스턴스로 설정하고 반환합니다. 가장 안전한 방법이므로 여기서 과정을 마칩니다.
      - **불가능할 경우**: (예: `keytar` 설치 실패, GUI가 없는 서버 환경 등) 예외를 무시하고 다음 단계로 넘어갑니다.
  4.  **파일 저장소로 대체(Fallback)**: 키체인 사용이 불가능하면, `FileTokenStorage`를 인스턴스화하여 `storage`로 설정하고 반환합니다. 이는 더 넓은 호환성을 보장합니다.
  5.  이 모든 과정은 `storageInitPromise`를 통해 단 한 번만 실행되어, 여러 곳에서 동시에 호출해도 경쟁 상태(race condition)가 발생하지 않도록 보장합니다.

#### 공개 메서드 (예: `getCredentials`, `setCredentials`)

- **처리 설명**: 이 클래스의 모든 공개 메서드들은 먼저 `getStorage()`를 호출하여 실제 사용할 저장소(키체인 또는 파일)를 얻은 다음, 모든 작업을 해당 저장소 인스턴스에 위임(delegate)합니다. 예를 들어 `getCredentials`는 `(await this.getStorage()).getCredentials(...)`를 호출합니다.

## 4. 기타 참고사항

- **보안 우선 설계**: `HybridTokenStorage`는 가능한 한 가장 안전한 방법(키체인)을 먼저 시도하고, 여의치 않을 때만 차선책(암호화된 파일)을 사용하는 점진적 저하(graceful degradation) 방식을 채택하여 보안을 강화합니다.
- **`keytar` 의존성**: `KeychainTokenStorage`는 `keytar`라는 네이티브 Node.js 모듈에 의존합니다. 이 모듈은 동적으로 `import` 되는데, 이는 `keytar`가 설치되지 않았거나 실패한 환경에서도 `gemini-cli`가 비상 모드(파일 저장)로 동작할 수 있도록 하기 위함입니다.
- **환경 변수 재정의**: `GEMINI_FORCE_FILE_STORAGE=true` 환경 변수를 설정하면, 키체인 사용 가능 여부와 관계없이 강제로 `FileTokenStorage`를 사용하도록 할 수 있습니다. 이는 디버깅이나 특정 환경에서 유용합니다.
