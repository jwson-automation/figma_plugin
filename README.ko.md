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

## 설치

### 1. 플러그인 빌드

```bash
npm install
npm run build
```

### 2. MCP 서버 설치

```bash
cd mcp-server
npm install
npm run build
```

### 3. Figma에 플러그인 등록

1. Figma Desktop 실행
2. **Plugins** → **Development** → **Import plugin from manifest...**
3. `manifest.json` 선택

### 4. Claude Code MCP 설정

프로젝트 루트의 `.mcp.json`이 자동 인식됩니다. 전역 등록은 `~/.claude/mcp.json`에 추가:

```json
{
  "mcpServers": {
    "figma-bridge": {
      "command": "node",
      "args": ["--import", "tsx/esm", "<경로>/mcp-server/src/index.ts"],
      "cwd": "<경로>/mcp-server",
      "env": {
        "FIGMA_API_TOKEN": "<your-figma-token>"
      }
    }
  }
}
```

> `cwd`는 필수. 없으면 `tsx` 모듈을 찾지 못합니다.
> `FIGMA_API_TOKEN`은 댓글 조회/답글 기능에만 필요합니다.

## 사용 방법

1. Figma Desktop에서 파일 열기
2. **Plugins** → **Development** → **MCP Bridge** 실행
3. 플러그인 패널에 **녹색 점 + "MCP 서버 연결됨"** 확인
4. Claude Code에서 도구 호출

> MCP 서버는 Claude Code 시작 시 자동 실행됩니다.

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

## 포트 변경

기본 WebSocket 포트는 `3055`. 변경 시 MCP 설정의 `env.MCP_BRIDGE_PORT`와 `ui.ts`의 `WS_URL`을 동일하게 수정 후 재빌드.
