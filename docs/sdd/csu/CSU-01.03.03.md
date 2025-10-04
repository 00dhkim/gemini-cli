# CSU-01.03.03: 테마 관리

## 1. CSU 간단 설명

`ui/themes/` 디렉터리에 위치한 이 CSU는 Gemini CLI의 전체적인 색상과 스타일(테마)을 관리하는 시스템입니다. 사용자가 선택한 테마에 따라 UI의 모든 시각적 요소(텍스트 색상, 배경, 테두리 등)를 일관되게 적용하는 역할을 합니다.

이 CSU는 개별 테마 파일, 테마의 데이터 구조를 정의하는 파일, 모든 테마를 총괄하는 관리자(Manager), 그리고 현재 테마 색상을 UI 컴포넌트에 제공하는 프록시(Proxy)로 구성됩니다.

## 2. CSU를 구성하는 변수와 함수 목록

### 주요 파일 및 역할

| 파일명               | 설명                                                                                                                         |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `theme.ts`           | 모든 테마가 준수해야 하는 `Theme` 인터페이스와 `ColorsTheme` 같은 데이터 구조를 정의합니다.                                  |
| `[테마명].ts`        | `dracula.ts`, `github-dark.ts` 등 개별 테마의 색상 값을 정의하는 파일들입니다.                                               |
| `theme-manager.ts`   | 내장 테마와 커스텀 테마를 모두 등록하고, 현재 활성화된 테마를 설정하고 조회하는 `ThemeManager` 클래스를 제공합니다.          |
| `semantic-colors.ts` | 현재 활성화된 테마의 색상 객체를 export하여, UI 컴포넌트들이 쉽게 테마 색상을 가져다 쓸 수 있도록 하는 프록시 역할을 합니다. |

## 3. 각 구성요소에 대한 입출력 및 함수 처리 설명

### 3.1 `theme.ts`

- **역할**: 테마의 데이터 구조(Data Structure)를 정의합니다.
- **주요 인터페이스**:
  - `ColorsTheme`: 테마를 구성하는 기본 색상(배경, 전경, 강조색 등)의 집합을 정의합니다.
  - `CustomTheme`: 사용자가 `settings.json`에 직접 정의할 수 있는 커스텀 테마의 구조를 정의합니다.
  - `Theme`: `react-syntax-highlighter`와 호환되는 코드 하이라이팅 스타일과 `semanticColors`를 포함하는 최종 테마 클래스의 구조를 정의합니다.
- **처리 및 출력**: 이 파일은 주로 타입 정의(interface, type)로 구성되어 있으며, 다른 파일들이 일관된 테마 객체를 생성하고 사용할 수 있도록 하는 기반을 제공합니다.

### 3.2 `theme-manager.ts`

- **역할**: 애플리케이션의 모든 테마를 총괄하는 중앙 관리자(Singleton)입니다.
- **주요 클래스**: `ThemeManager`
- **주요 메서드**:
  - `constructor()`: `dracula`, `github-dark` 등 모든 내장 테마를 `availableThemes` 배열에 등록하며 초기화합니다.
  - `loadCustomThemes(settings)`: `settings.json`에 정의된 커스텀 테마를 읽어와 유효성을 검사하고, `customThemes` 맵에 추가합니다.
  - `setActiveTheme(themeName)`: 인자로 받은 이름의 테마를 찾아 활성 테마로 설정합니다. 성공 시 `true`를 반환합니다.
  - `getActiveTheme()`: 현재 활성화된 `Theme` 객체를 반환합니다. `NO_COLOR` 환경 변수가 설정된 경우 색상이 없는 테마를 반환하는 등 예외 처리도 포함합니다.
  - `getSemanticColors()`: 현재 활성 테마의 시맨틱 색상(`SemanticColors`) 객체를 반환합니다.
  - `getAvailableThemes()`: UI(테마 선택 다이얼로그)에 표시할 수 있는 모든 테마(내장+커스텀)의 목록을 반환합니다.
- **인스턴스**: `export const themeManager = new ThemeManager();`를 통해 싱글턴 인스턴스를 생성하고 export하여 애플리케이션 전역에서 동일한 테마 관리자를 사용하도록 합니다.

### 3.3 `semantic-colors.ts`

- **역할**: UI 컴포넌트와 `ThemeManager` 사이의 중개자(프록시) 역할을 합니다.
- **처리 및 출력**:
  - `themeManager` 인스턴스를 임포트합니다.
  - `themeManager.getSemanticColors()` 메서드를 호출하여 현재 활성화된 테마의 시맨틱 색상 객체를 가져옵니다.
  - 가져온 색상 객체를 `theme`이라는 상수로 export 합니다.
- **사용 예시**:
  - UI 컴포넌트에서는 `import { theme } from '../semantic-colors.js';` 구문을 통해 현재 테마의 색상 값을 쉽게 가져올 수 있습니다.
  - 예: `<Text color={theme.text.primary}>...</Text>`
  - 이를 통해 UI 컴포넌트는 현재 어떤 테마가 활성화되어 있는지 알 필요 없이, `theme` 객체를 통해 추상화된 색상 이름(`primary`, `accent` 등)을 일관되게 사용할 수 있습니다.

## 4. 아키텍처 및 기타 참고사항

- **테마 적용 흐름**: `settings.json`에 `ui.theme` 값이 설정되면, 애플리케이션 시작 시(`gemini.tsx`) `themeManager.setActiveTheme()`이 호출됩니다. 이후 각 UI 컴포넌트는 `semantic-colors.ts`를 통해 활성화된 테마의 색상 값을 참조하여 렌더링됩니다.
- **시맨틱 색상**: 이 시스템은 "파란색", "빨간색"과 같은 구체적인 색상 이름 대신 "주요 텍스트 색상(`text.primary`)", "오류 상태 색상(`status.error`)"과 같은 의미론적(semantic)인 이름을 사용합니다. 이를 통해 테마가 변경되어도 코드 수정 없이 UI의 일관성을 유지할 수 있습니다.
- **확장성**: 새로운 내장 테마를 추가하려면 단순히 테마 파일을 생성하고 `theme-manager.ts`의 `availableThemes` 배열에 추가하기만 하면 됩니다. 사용자는 `settings.json`을 통해 자신만의 커스텀 테마를 쉽게 추가할 수 있습니다.
