# JEB MCP — Claude Code로 APK 분석하기

JEB Pro 디컴파일러를 MCP(Model Context Protocol)로 연결하여 Claude Code에서 APK를 분석하는 도구입니다.

## 요구사항

- **JEB Pro** 5.x
- **Python 3.8+** (MCP 서버용, 외부 패키지 불필요)
- **Claude Code** CLI

## 설치

### 1. 파일 배치

JEB 설치 디렉토리에 다음 파일들을 배치합니다.

```
<JEB_DIR>/
├── bin/jeb-engines.cfg                        # Step 2에서 수정
├── coreplugins/scripts/McpBridgeAutoStart.py  # JEB Bridge (자동 시작)
└── mcp_server/jeb_mcp_server.py               # MCP 서버
```

### 2. Python 플러그인 활성화

`bin/jeb-engines.cfg` 파일에 다음 내용을 추가합니다:

```
.LoadPythonPlugins = true
```

이 설정으로 JEB 시작 시 `coreplugins/scripts/McpBridgeAutoStart.py`가 자동으로 로드되어 MCP Bridge HTTP 서버가 시작됩니다.

### 3. Claude Code MCP 설정

Claude Code 설정 파일(`~/.claude/settings.json` 또는 프로젝트의 `.mcp.json`)에 MCP 서버를 등록합니다.

**.mcp.json** (프로젝트 루트에 생성):
```json
{
  "mcpServers": {
    "jeb": {
      "command": "python",
      "args": ["mcp_server/jeb_mcp_server.py"],
      "cwd": "<JEB_DIR>"
    }
  }
}
```

또는 Claude Code에서 직접 추가:
```bash
claude mcp add jeb -- python <JEB_DIR>/mcp_server/jeb_mcp_server.py
```

### 4. 포트 변경 (선택)

기본 포트는 `18700`입니다. 변경이 필요하면 환경변수 `JEB_MCP_PORT`를 설정합니다.

```json
{
  "mcpServers": {
    "jeb": {
      "command": "python",
      "args": ["mcp_server/jeb_mcp_server.py"],
      "cwd": "<JEB_DIR>",
      "env": {
        "JEB_MCP_PORT": "19000"
      }
    }
  }
}
```

JEB 측에서도 동일한 환경변수를 읽으므로, JEB 실행 전에 시스템 환경변수로 설정하거나 JEB 실행 스크립트에 추가해야 합니다.

## 사용법

### 기본 흐름

```
1. JEB 실행 → APK 열기 (Bridge 자동 시작)
2. Claude Code 실행 (MCP 서버 자동 연결)
3. 분석 시작
```

### 분석 예시

```
# 기본 분석
> 이 APK의 매니페스트를 분석해줘
> exported 컴포넌트 목록 보여줘
> MainActivity 디컴파일해줘

# 취약점 분석 (내장 스킬)
> /find-intent-redirect
> /find-deeplink-vuln

# 코드 검색
> loadUrl을 호출하는 코드를 찾아줘
> com.example 패키지에서 startActivity 패턴 검색해줘

# 리버싱
> 난독화된 클래스 a.b.c를 분석하고 의미있는 이름으로 리네이밍해줘
```

## MCP 도구 (24개)

| 카테고리 | 도구 | 설명 |
|----------|------|------|
| 탐색 | `get_jeb_status` | 연결 상태 확인 |
| | `execute_script` | 임의 Jython 코드 실행 |
| | `list_units` | 분석 유닛 목록 |
| | `list_classes` | 클래스 목록 (필터 가능) |
| | `list_methods` | 메서드 목록 |
| | `list_fields` | 필드 목록 |
| 디컴파일 | `decompile_class` | 클래스 전체 디컴파일 |
| | `decompile_method` | 메서드 디컴파일 |
| 분석 | `get_xrefs` | 크로스 레퍼런스 |
| | `search_strings` | 문자열 검색 |
| | `search_code` | 소스 코드 패턴 검색 |
| | `get_callgraph` | 호출 그래프 |
| | `get_class_hierarchy` | 클래스 상속 관계 |
| 리네이밍 | `rename_class` | 클래스 이름 변경 |
| | `rename_method` | 메서드 이름 변경 |
| | `rename_field` | 필드 이름 변경 |
| | `rename_variable` | 변수 이름 변경 |
| | `bulk_rename` | 일괄 이름 변경 |
| | `move_class_to_package` | 패키지 이동 |
| | `add_comment` | 주석 추가 |
| Android | `get_apk_manifest` | AndroidManifest.xml |
| | `get_apk_certificate` | 서명 인증서 정보 |
| | `get_apk_resource` | 리소스 파일 조회 |
| | `list_components` | 컴포넌트 목록 |

## 아키텍처

```
Claude Code ──stdio──> MCP Server (Python 3) ──HTTP──> JEB Bridge (Jython) ──> JEB API
```

1. Claude Code가 도구를 호출하면 MCP Server가 Jython 코드 템플릿을 생성
2. 템플릿을 HTTP POST로 JEB Bridge에 전송
3. JEB Bridge가 JEB 내부에서 Jython 코드를 실행
4. 결과를 JSON으로 반환

## 트러블슈팅

### Bridge가 시작되지 않음
- `bin/jeb-engines.cfg`에 `.LoadPythonPlugins = true`가 있는지 확인
- JEB 콘솔에서 `[MCP Bridge]` 로그 확인
- 포트 충돌 시 `JEB_MCP_PORT` 환경변수로 변경

### MCP 서버 연결 실패
- JEB가 실행 중이고 APK가 열려 있는지 확인
- `curl http://localhost:18700/status`로 Bridge 상태 확인
- 포트가 MCP 서버와 Bridge 양쪽에서 일치하는지 확인

### search_code가 느림
- `class_filter` 파라미터로 검색 범위를 패키지명으로 제한

### 디컴파일 결과가 stub만 나옴
- `decompile_class`는 내부적으로 모든 메서드를 개별 디컴파일하므로 정상 동작
- `decompile_method`로 개별 메서드를 확인

## 라이선스

JEB Pro 라이선스가 필요합니다. MCP 서버와 Bridge 스크립트는 자유롭게 사용할 수 있습니다.
