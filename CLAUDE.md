# JEB MCP — Android APK 분석 환경

## 프로젝트 개요

JEB Pro 디컴파일러를 MCP(Model Context Protocol)로 연결하여 Claude가 APK를 분석하는 환경.

## 아키텍처

```
Claude ──stdio──> MCP Server (Python 3) ──HTTP:18700──> JEB Bridge (Jython) ──> JEB API
                  mcp_server/jeb_mcp_server.py          coreplugins/scripts/McpBridgeAutoStart.py
```

- **JEB Bridge**: JEB 내부에서 실행되는 Jython 엔진 플러그인. HTTP 서버(포트 18700)로 코드를 받아 실행
- **MCP Server**: Python 3 stdio MCP 서버. Claude의 도구 호출을 Jython 코드 템플릿으로 변환하여 Bridge에 전송
- **포트 설정**: 환경변수 `JEB_MCP_PORT`로 변경 가능 (기본 18700)
- **자동 시작**: `bin/jeb-engines.cfg`에 `.LoadPythonPlugins = true` 설정으로 JEB 시작 시 Bridge 자동 실행

## MCP 도구 목록 (24개)

### 기본 탐색
| 도구 | 설명 |
|------|------|
| `get_jeb_status` | JEB 연결 상태 및 프로젝트 정보 확인 |
| `execute_script` | 임의 Jython 코드 실행 (고급 사용) |
| `list_units` | 분석 유닛 목록 (DEX, Manifest 등) |
| `list_classes` | 클래스 목록. `filter`로 패키지 필터링 |
| `list_methods` | 클래스의 메서드 목록 |
| `list_fields` | 클래스의 필드 목록 |

### 디컴파일
| 도구 | 설명 |
|------|------|
| `decompile_class` | 클래스 전체 디컴파일 (모든 메서드 포함) |
| `decompile_method` | 특정 메서드 디컴파일 |

### 분석
| 도구 | 설명 |
|------|------|
| `get_xrefs` | 크로스 레퍼런스 조회 (callers/callees) |
| `search_strings` | DEX 문자열 검색 |
| `search_code` | 디컴파일된 소스 코드에서 패턴 검색. `class_filter`로 범위 제한 권장 |
| `get_callgraph` | 메서드의 호출 그래프 (callers + callees) |
| `get_class_hierarchy` | 클래스 상속 관계 (superclass, interfaces, subclasses) |

### 리네이밍 & 리팩토링
| 도구 | 설명 |
|------|------|
| `rename_class` | 클래스 이름 변경 |
| `rename_method` | 메서드 이름 변경 |
| `rename_field` | 필드 이름 변경 |
| `rename_variable` | 로컬 변수 이름 변경 |
| `bulk_rename` | 여러 항목 일괄 이름 변경 |
| `move_class_to_package` | 클래스를 다른 패키지로 이동 |
| `add_comment` | 메서드/클래스에 주석 추가 |

### Android 전용
| 도구 | 설명 |
|------|------|
| `get_apk_manifest` | AndroidManifest.xml 조회 |
| `get_apk_certificate` | APK 서명 인증서 정보 |
| `get_apk_resource` | APK 리소스 파일 조회 (strings.xml 등) |
| `list_components` | 매니페스트의 Activity/Service/Receiver/Provider 목록 |

## 시그니처 형식

JEB 도구에 전달하는 시그니처는 Dalvik/Smali 형식:
- **클래스**: `Lcom/example/MyClass;` (L 접두사, / 구분, ; 접미사)
- **메서드**: `Lcom/example/MyClass;->methodName(Ljava/lang/String;I)V`
- **필드**: `Lcom/example/MyClass;->fieldName:Ljava/lang/String;`
- **타입**: `I`=int, `Z`=boolean, `B`=byte, `[`=array, `V`=void

## 분석 워크플로우 가이드

### 일반적인 분석 순서
1. `get_jeb_status` — 연결 확인, 열린 프로젝트 확인
2. `get_apk_manifest` — 매니페스트에서 컴포넌트, 권한, intent-filter 파악
3. `list_components` — exported 컴포넌트 빠르게 확인
4. `list_classes` + filter — 관심 패키지의 클래스 탐색
5. `decompile_class` 또는 `decompile_method` — 코드 확인
6. `search_code` — 위험 API 호출 패턴 검색
7. `get_xrefs` / `get_callgraph` — 호출 관계 추적

### 취약점 분석 시
- `search_code`의 `class_filter`를 적극 활용하여 exported 컴포넌트 범위로 검색 제한
- 검색 → 디컴파일 → 데이터 흐름 추적 (source → sink) 패턴
- `/find-intent-redirect` — Intent Redirect 취약점 자동 탐지
- `/find-deeplink-vuln` — Deeplink 원클릭 공격 취약점 자동 탐지

### 리버싱/리네이밍 시
- 난독화된 코드를 분석하면서 `rename_class`, `rename_method` 등으로 의미있는 이름 부여
- `bulk_rename`으로 일괄 처리
- `add_comment`로 분석 메모 기록

## 주의사항

- **Jython 2.7 환경**: JEB Bridge는 Jython(Python 2.7)에서 실행됨. 유니코드 출력 시 `JString(text)` 사용 필수
- **내부 클래스**: `decompileToUnit`으로 내부 클래스(e.g. `$3`)를 요청하면 외부 클래스의 소스 유닛이 반환됨. 내부 클래스 메서드는 AST에서 찾을 수 없음
- **search_code 성능**: `class_filter` 없이 전체 검색하면 느림. 가능하면 패키지명으로 범위 제한
- **decompile_class**: 내부적으로 모든 메서드를 개별 디컴파일한 후 전체 텍스트를 반환함 (stub만 나오는 문제 해결됨)

## 파일 구조

```
├── CLAUDE.md                          # 이 파일
├── bin/
│   ├── jeb-engines.cfg                # .LoadPythonPlugins = true
│   └── jeb-client.cfg                 # JEB 클라이언트 설정
├── coreplugins/scripts/
│   └── McpBridgeAutoStart.py          # 엔진 플러그인 (JEB 시작 시 자동 실행)
├── mcp_server/
│   └── jeb_mcp_server.py              # MCP 서버 (Python 3)
└── .claude/
    └── skills/
        ├── find-intent-redirect/      # Intent Redirect 취약점 분석 스킬
        └── find-deeplink-vuln/        # Deeplink 원클릭 취약점 분석 스킬
```
