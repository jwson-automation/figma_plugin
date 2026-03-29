# Figma MCP Bridge

> **Language / 언어 / 言語:** [English](./README.md) | [日本語](./README.ja.md)

Figma 디자인 데이터를 Claude Code에서 직접 조회할 수 있는 MCP(Model Context Protocol) 브릿지.
Figma Desktop Plugin API를 통해 **실시간으로 열려 있는 파일**의 노드, 스타일, 댓글, 이미지를 추출합니다.

## 왜 필요한가

Figma REST API는 파일 전체를 가져오기 때문에 응답이 크고 느립니다.
이 플러그인은 **Figma Desktop Plugin API**를 사용하여:

- 특정 노드만 선택적으로 조회 (REST API 대비 응답 크기 90% 이상 절감)
- 기본값 자동 제거로 노이즈 없는 깔끔한 출력
- `detail` 레벨 선택으로 필요한 만큼만 데이터 수신
- 현재 열린 파일의 실시간 상태 반영 (저장 전 변경사항 포함)

## 아키텍처

```
Claude Code ──stdio──▶ MCP Server (Node.js, port 3055)
                            │
                        WebSocket
                            │
                       Figma Plugin UI (ui.ts)
                            │
                        postMessage
                            │
                       Figma Main Thread (code.ts) ──▶ Figma Plugin API
```

## 빠른 시작

```bash
# 1. 설치 & 빌드
npm install && npm run build
cd mcp-server && npm install && cd ..

# 2. Figma에 플러그인 등록
#    Plugins → Development → Import plugin from manifest... → manifest.json 선택

# 3. 에디터 자동 설정 (Claude Code 안에서 슬래시 커맨드로 실행)
/setup-claude-code                 # Claude Code (stdio, 권장)
/setup-vscode                      # VS Code (.vscode/mcp.json, SSE)
/setup-vscode --cursor             # Cursor (~/.cursor/mcp.json, SSE)
```

### 3. 에디터 설정

**방법 A — 스킬 사용 (자동, Claude Code 전용):**

이 저장소에는 `.claude/skills/`에 설정 스킬이 포함되어 있습니다. Claude Code에서 슬래시 커맨드로 실행:

```
/setup-claude-code                 # Claude Code (stdio, 권장)
/setup-vscode                      # VS Code (.vscode/mcp.json, SSE)
/setup-vscode --cursor             # Cursor (~/.cursor/mcp.json, SSE)
```

**방법 B — 수동 설정:**

| 에디터 | 방법 |
|--------|------|
| **Claude Code** | 프로젝트 루트의 `.mcp.json`이 자동 인식됩니다. 이 프로젝트를 Claude Code에서 열고 재시작하면 바로 사용 가능. 다른 프로젝트에서도 전역으로 사용하려면 [4단계](#4-claude-code-연결-stdio-모드--권장)를 참고하세요. |
| **VS Code / Cursor** | SSE 모드로 서버를 실행한 뒤, 에디터에서 `http://localhost:3100/sse`로 연결. [4-1단계](#4-1-sse-모드-다른-mcp-클라이언트용)를 참고하세요. |

## 설치

### 1. 플러그인 빌드

```bash
npm install
npm run build
```

### 2. MCP 서버 의존성 설치

```bash
cd mcp-server
npm install
```

### 3. Figma에 플러그인 등록

1. Figma Desktop 실행
2. **Plugins** → **Development** → **Import plugin from manifest...**
3. `manifest.json` 선택

### 4. Claude Code 연결 (stdio 모드 — 권장)

> **동작 원리:** Claude Code는 이미 실행 중인 서버에 접속하는 게 아닙니다. Claude Code가 **MCP 서버 프로세스를 직접 생성**하고 stdin/stdout(stdio)으로 통신합니다. 서버를 수동으로 실행할 필요가 없습니다 — 설정만 추가하고 Claude Code를 재시작하세요.

**방법 A — 프로젝트 레벨 (클론 후 바로 사용):**

이 저장소에는 프로젝트 루트에 상대 경로로 된 `.mcp.json`이 이미 포함되어 있습니다. Claude Code에서 이 프로젝트 디렉토리를 열면 MCP 서버가 자동 인식됩니다. **추가 설정 불필요.**

```
your-project/
├── .mcp.json          ← Claude Code가 자동 인식
├── mcp-server/
│   └── src/index.ts
```

`cd mcp-server && npm install`만 먼저 실행하세요.

**방법 B — 전역 등록 (다른 프로젝트에서도 사용):**

`~/.claude/mcp.json`에 **절대 경로**로 추가:

```json
{
  "mcpServers": {
    "figma-bridge": {
      "command": "node",
      "args": ["--import", "tsx/dist/esm", "<절대경로>/mcp-server/src/index.ts"],
      "cwd": "<절대경로>/mcp-server",
      "env": {
        "FIGMA_API_TOKEN": "<your-figma-token>"
      }
    }
  }
}
```

`<절대경로>`를 실제 경로로 변경하세요 (예: `/Users/you/figma-mcp-bridge` 또는 `C:/Users/you/figma-mcp-bridge`).

> `cwd`는 필수. 없으면 `tsx` 모듈을 찾지 못합니다.
> `FIGMA_API_TOKEN`은 댓글 조회/답글 기능에만 필요합니다.

### 4-1. SSE 모드 (다른 MCP 클라이언트용)

일부 MCP 클라이언트(Cursor, 특정 IDE 확장 등)는 stdio를 지원하지 않고 HTTP 서버 접속이 필요합니다. 이 경우 **SSE(Server-Sent Events) 모드**를 사용하세요.

> **핵심 차이:** stdio 모드에서는 Claude Code가 서버를 자동 실행합니다. SSE 모드에서는 **서버를 직접 실행**한 뒤 클라이언트를 연결해야 합니다.

**Step 1 — 서버 실행:**

```bash
cd mcp-server

# Linux / macOS
MCP_TRANSPORT=sse npm start

# Windows (PowerShell)
$env:MCP_TRANSPORT="sse"; npm start

# Windows (cmd)
set MCP_TRANSPORT=sse && npm start
```

서버가 `http://localhost:3100/sse`에서 대기합니다. 포트 변경은 `MCP_SSE_PORT` 환경변수.

**Step 2 — MCP 클라이언트 설정:**

```json
{
  "mcpServers": {
    "figma-bridge": {
      "url": "http://localhost:3100/sse"
    }
  }
}
```

> Figma 플러그인용 WebSocket 브릿지(포트 3055)는 두 모드 모두에서 동작합니다.

## 사용 방법

1. Figma Desktop에서 파일 열기
2. **Plugins** → **Development** → **MCP Bridge** 실행
3. 플러그인 패널에 **녹색 점 + "MCP 서버 연결됨"** 확인
4. Claude Code(또는 MCP 클라이언트)에서 도구 호출

> stdio 모드: Claude Code 시작 시 MCP 서버가 자동 실행됩니다.
> SSE 모드: 서버가 이미 실행 중인지 확인하세요.

## MCP 도구

### `figma_get_node`

특정 노드의 디자인 스펙을 가져옵니다.

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `nodeId` | string | **필수** | Figma 노드 ID (URL의 `node-id` 값, `730-16041` → `730:16041`) |
| `maxDepth` | number | `5` | 자식 노드 탐색 깊이 |
| `detail` | `minimal` \| `standard` \| `full` | `standard` | 직렬화 상세 수준 |

### `figma_get_selection`

현재 Figma에서 선택된 노드들의 스펙을 가져옵니다. 파라미터는 `figma_get_node`와 동일 (`nodeId` 제외).

### `figma_get_page_nodes`

현재 페이지의 최상위 노드 목록. 30개 초과 시 경고 메시지를 반환하며, `detail:"minimal"`로 변경하거나 특정 노드를 `figma_get_node`로 직접 조회하여 범위를 줄여주세요.

### `figma_get_file_info`

파일명, 페이지 목록, 현재 페이지, 선택 노드 수 반환. 파라미터 없음.

### `figma_get_comments`

Figma REST API를 통해 댓글을 가져옵니다. `FIGMA_API_TOKEN` 환경변수 필요.

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `fileKey` | string | 자동 감지 | Figma URL의 파일 키 |
| `unresolvedOnly` | boolean | `false` | 미해결 댓글만 반환 |

### `figma_reply_comment`

댓글에 답글을 답니다. 수정 완료 내역 등을 남길 때 사용합니다.

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `fileKey` | string | Figma 파일 키 |
| `commentId` | string | 답글 대상 댓글 ID |
| `message` | string | 답글 내용 |

### `figma_export_node`

노드를 이미지로 내보냅니다 (base64).

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `nodeId` | string | **필수** | 노드 ID |
| `format` | `PNG` \| `SVG` \| `JPG` | `PNG` | 포맷 |
| `scale` | number | `2` | 배율 |

### `figma_analyze_comments`

미해결 댓글을 조회하고, 각 댓글이 위치한 노드의 디자인 스펙을 함께 반환합니다. 한 번의 호출로 "어디에 어떤 문제가 있는지" 파악할 수 있습니다.

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `fileKey` | string | 자동 감지 | Figma 파일 키 |
| `detail` | `minimal` \| `standard` | `minimal` | 노드 정보 상세 수준 |

**반환 값 (각 댓글):**

| 필드 | 설명 |
|------|------|
| `commentId` | 댓글 ID |
| `author` | 작성자 |
| `message` | 댓글 내용 |
| `nodeId` | 댓글이 위치한 노드 ID |
| `position` | 노드 내 오프셋 좌표 |
| `replies` | 해당 댓글의 답글 목록 |
| `node` | 노드 디자인 스펙 (detail 수준에 따라) |

**동작 방식:**
1. Figma REST API로 미해결 댓글 전체 조회
2. 댓글을 스레드별로 그룹핑 (원댓글 + 답글)
3. 각 댓글이 위치한 고유 노드 ID 수집
4. Plugin API로 각 노드의 디자인 스펙 조회 (중복 노드는 1회만)
5. 댓글 + 노드 정보를 합쳐서 반환

> 댓글이 20개를 초과하면 경고 메시지가 포함됩니다. 특정 노드를 상세히 보려면 `figma_get_node(nodeId, detail:"standard")`로 재조회하세요.

### `figma_get_styles`

로컬 디자인 시스템 스타일 (색상, 텍스트, 이펙트). 파라미터 없음.

## Detail Level

`detail` 파라미터로 응답 상세도를 조절합니다. 실제 응답 예시는 [`docs/`](./docs/) 참고.

### `minimal`

노드 식별 정보만 반환. 대량 노드 목록 파악에 적합.

```json
{
  "id": "673:2483",
  "name": "type=job, context=featured",
  "type": "COMPONENT",
  "width": 320,
  "height": 121,
  "childCount": 2
}
```

**포함:** id, name, type, width, height, childCount
**용도:** 페이지 구조 파악, 특정 노드 ID 탐색

### `standard` (기본값)

디자인 구현에 필요한 핵심 정보. 기본값(visible:true, opacity:1 등)은 자동 제거.

**포함:** minimal + fills, strokes, cornerRadius, layout(flexbox/padding/gap), typography(font/size/lineHeight/letterSpacing/color), componentId/Name, 자식 노드 재귀
**용도:** UI 컴포넌트 구현, 디자인-코드 비교, 스타일 추출

### `full`

디자인 토큰, 컴포넌트 variant, 오버라이드 등 전체 속성.

**포함:** standard + absoluteBoundingBox, boundVariables(디자인 토큰 바인딩), textTruncation/maxLines, textStyleId, componentProperties, overrides, variantGroupProperties, clipsContent, counterAxisSpacing
**용도:** 디자인 토큰 매핑, variant 분석, 디자인 시스템 문서화

### 자동 축소

대량 노드 시 응답 크기를 제어하기 위해 자동으로 detail이 낮아집니다:

- 페이지 노드 30개 초과 → 경고 메시지 반환 (자동 전환 없음, 사용자 판단)
- 깊은 자식 노드에서 형제 50개 초과 + depth ≥ 2 → `minimal`
- 깊은 자식 노드에서 형제 20개 초과 + depth ≥ 3 + `full` 요청 → `standard`
- depth ≥ 5 → `minimal`

## 응답 최적화

REST API 응답과 비교하여:

| 항목 | REST API | 이 플러그인 (standard) |
|------|----------|------------------------|
| 기본값 (visible, opacity 등) | 모두 포함 | 자동 제거 |
| Paint 속성 | `type, visible, opacity, color` 4개 | `type, color` 2개 |
| 빈 배열 (`dashPattern: []`) | 포함 | 제거 |
| boundVariables | ID만 (변수명 없음) | standard에서 생략, full에서 포함 |
| 조회 범위 | 파일 전체 | 특정 노드만 |

## 구현 가능한 워크플로우

### 디자인 → 코드 자동 구현

```
1. figma_get_node(nodeId, detail: "standard")  → 레이아웃, 색상, 타이포 추출
2. 추출된 스펙으로 컴포넌트 코드 생성
3. figma_export_node(nodeId)  → 디자인 스크린샷으로 결과 비교
```

### 디자인 토큰 추출

```
1. figma_get_node(nodeId, detail: "full")  → boundVariables에서 토큰 ID 확인
2. figma_get_styles()  → 색상/텍스트/이펙트 스타일 전체 추출
3. 디자인 토큰 파일(JSON/CSS Variables) 생성
```

### 디자인 QA (코드 vs 디자인 비교)

```
1. figma_get_selection()  → 현재 선택된 컴포넌트 스펙
2. 코드의 스타일 값과 1:1 비교 (font-size, color, padding, gap 등)
3. 차이점 리포트 생성
```

### 댓글 기반 수정 자동화

```
1. figma_analyze_comments()  → 미해결 댓글 + 각 위치 노드 스펙 한번에 조회
2. 각 댓글의 node 정보로 문제 파악
3. 상세 스펙 필요 시 figma_get_node(nodeId, detail:"standard")  → 재조회
4. 코드 수정
5. figma_reply_comment(commentId, "수정 완료")  → 답글 남기기
```

### 페이지 구조 파악

```
1. figma_get_file_info()  → 파일명, 페이지 목록
2. figma_get_page_nodes(detail: "minimal")  → 최상위 프레임 목록
3. figma_get_node(targetNodeId, detail: "standard")  → 특정 프레임 상세 조회
```

## 개발

```bash
# 플러그인 watch 모드
npm run watch

# MCP 서버 빌드
cd mcp-server && npm run build
```

Figma에서 **Plugins** → **Development** → **Reload plugin**으로 반영.

> MCP 서버 코드(`index.ts`) 수정 후 반드시 `npm run build` 실행.
> 이후 Claude Code에서 `/mcp` 명령으로 재접속하면 새 도구 스키마가 반영됩니다.

## 문제 해결

| 증상 | 해결 |
|------|------|
| 플러그인 패널에 빨간 점 | Claude Code 재시작 (MCP 서버 자동 실행) |
| "플러그인이 연결되어 있지 않습니다" | Figma에서 MCP Bridge 플러그인 재실행 |
| MCP 도구에 새 파라미터가 안 보임 | `cd mcp-server && npm run build` 후 `/mcp` 재접속 |
| 댓글 조회 실패 | `FIGMA_API_TOKEN` 환경변수 확인 |
| "MCP 서버에 연결할 수 없음" | 서버를 직접 실행하지 마세요. `mcp.json`에 설정을 추가하고 Claude Code를 재시작하면 stdio로 자동 실행됩니다. stdio를 지원하지 않는 클라이언트는 SSE 모드를 사용하세요 (4-1 참고). |

## 포트 변경

기본 WebSocket 포트는 `3055`. 변경 시 MCP 설정의 `env.MCP_BRIDGE_PORT`와 `ui.ts`의 `WS_URL`을 동일하게 수정 후 재빌드.
