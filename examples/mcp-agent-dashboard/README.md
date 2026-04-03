# MCP 기반 에이전트 대시보드 구축 예제

> MCP 서버들을 연동한 통합 에이전트 관리 대시보드 — Next.js + SSE 실시간 모니터링

## 이 예제에서 배울 수 있는 것

| 주제 | 설명 |
|------|------|
| MCP 클라이언트 구현 | TypeScript로 MCP 서버에 연결하고 도구 호출하는 클라이언트 |
| SSE 실시간 스트리밍 | Server-Sent Events로 에이전트 상태를 실시간 업데이트 |
| Next.js App Router | Server Components + Route Handlers 조합 패턴 |
| 대시보드 UI | shadcn/ui + Tailwind CSS로 모니터링 인터페이스 구축 |
| 에이전트 오케스트레이션 | 여러 MCP 서버를 하나의 UI에서 관리하는 패턴 |

## 아키텍처 개요

```
┌─────────────────────────────────────────────────┐
│                 Dashboard UI                     │
│  (Next.js App Router + shadcn/ui)               │
├─────────────────────────────────────────────────┤
│              API Route Handlers                  │
│  /api/agents   /api/tools   /api/stream         │
├─────────────────────────────────────────────────┤
│            MCP Client Manager                    │
│  (연결 풀, 자동 재연결, 상태 추적)                   │
├──────────┬──────────┬──────────┬────────────────┤
│ MCP 서버  │ MCP 서버  │ MCP 서버  │ MCP 서버       │
│ (파일)    │ (Git)    │ (DB)     │ (커스텀)        │
└──────────┴──────────┴──────────┴────────────────┘
```

## 프로젝트 구조

```
mcp-agent-dashboard/
├── src/
│   ├── app/
│   │   ├── layout.tsx              # 루트 레이아웃
│   │   ├── page.tsx                # 대시보드 메인
│   │   ├── agents/
│   │   │   └── page.tsx            # 에이전트 관리
│   │   ├── tools/
│   │   │   └── page.tsx            # 도구 브라우저
│   │   └── api/
│   │       ├── agents/route.ts     # 에이전트 CRUD
│   │       ├── tools/
│   │       │   ├── route.ts        # 도구 목록
│   │       │   └── invoke/route.ts # 도구 실행
│   │       └── stream/route.ts     # SSE 스트림
│   ├── lib/
│   │   ├── mcp-client.ts           # MCP 클라이언트 래퍼
│   │   ├── mcp-manager.ts          # 다중 서버 관리자
│   │   ├── event-emitter.ts        # SSE 이벤트 브로커
│   │   └── types.ts                # 공통 타입
│   └── components/
│       ├── agent-card.tsx          # 에이전트 상태 카드
│       ├── tool-browser.tsx        # 도구 탐색기
│       ├── tool-invoke-form.tsx    # 도구 실행 폼
│       ├── log-stream.tsx          # 실시간 로그 뷰어
│       └── server-status.tsx       # 서버 연결 상태
├── mcp-config.json                 # MCP 서버 설정
├── package.json
└── tsconfig.json
```

## 핵심 코드

### 1. MCP 클라이언트 매니저 (`src/lib/mcp-manager.ts`)

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js";
import { EventEmitter } from "events";

interface MCPServerConfig {
  id: string;
  name: string;
  transport: "stdio" | "sse";
  command?: string;
  args?: string[];
  url?: string;
  env?: Record<string, string>;
}

interface ServerState {
  id: string;
  name: string;
  status: "connecting" | "connected" | "error" | "disconnected";
  tools: ToolInfo[];
  lastPing: Date | null;
  error?: string;
}

interface ToolInfo {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
  serverId: string;
}

export class MCPManager extends EventEmitter {
  private clients = new Map<string, Client>();
  private states = new Map<string, ServerState>();
  private configs: MCPServerConfig[] = [];

  constructor(configs: MCPServerConfig[]) {
    super();
    this.configs = configs;
  }

  async connectAll(): Promise<void> {
    await Promise.allSettled(
      this.configs.map((config) => this.connectServer(config))
    );
  }

  private async connectServer(config: MCPServerConfig): Promise<void> {
    this.updateState(config.id, {
      id: config.id,
      name: config.name,
      status: "connecting",
      tools: [],
      lastPing: null,
    });

    try {
      const client = new Client(
        { name: "dashboard", version: "1.0.0" },
        { capabilities: {} }
      );

      let transport;
      if (config.transport === "stdio") {
        transport = new StdioClientTransport({
          command: config.command!,
          args: config.args || [],
          env: { ...process.env, ...config.env } as Record<string, string>,
        });
      } else {
        transport = new SSEClientTransport(new URL(config.url!));
      }

      await client.connect(transport);
      this.clients.set(config.id, client);

      // 도구 목록 가져오기
      const { tools } = await client.listTools();
      const toolInfos: ToolInfo[] = tools.map((t) => ({
        name: t.name,
        description: t.description || "",
        inputSchema: t.inputSchema as Record<string, unknown>,
        serverId: config.id,
      }));

      this.updateState(config.id, {
        id: config.id,
        name: config.name,
        status: "connected",
        tools: toolInfos,
        lastPing: new Date(),
      });

      this.emit("server:connected", { serverId: config.id, tools: toolInfos });
    } catch (error) {
      this.updateState(config.id, {
        id: config.id,
        name: config.name,
        status: "error",
        tools: [],
        lastPing: null,
        error: error instanceof Error ? error.message : "Unknown error",
      });
      this.emit("server:error", { serverId: config.id, error });
    }
  }

  async invokeTool(
    serverId: string,
    toolName: string,
    args: Record<string, unknown>
  ) {
    const client = this.clients.get(serverId);
    if (!client) throw new Error(`Server ${serverId} not connected`);

    this.emit("tool:invoke", { serverId, toolName, args });

    const result = await client.callTool({ name: toolName, arguments: args });

    this.emit("tool:result", { serverId, toolName, result });
    return result;
  }

  getAllTools(): ToolInfo[] {
    return Array.from(this.states.values()).flatMap((s) => s.tools);
  }

  getStates(): ServerState[] {
    return Array.from(this.states.values());
  }

  private updateState(id: string, state: ServerState) {
    this.states.set(id, state);
    this.emit("state:change", state);
  }

  async disconnect(): Promise<void> {
    for (const [id, client] of this.clients) {
      try {
        await client.close();
      } catch {
        // 무시
      }
      this.updateState(id, {
        ...this.states.get(id)!,
        status: "disconnected",
      });
    }
    this.clients.clear();
  }
}
```

### 2. SSE 스트림 엔드포인트 (`src/app/api/stream/route.ts`)

```typescript
import { getManager } from "@/lib/singleton";

export async function GET() {
  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    start(controller) {
      const manager = getManager();

      const send = (event: string, data: unknown) => {
        controller.enqueue(
          encoder.encode(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`)
        );
      };

      // 초기 상태 전송
      send("init", {
        servers: manager.getStates(),
        tools: manager.getAllTools(),
      });

      // 실시간 이벤트 구독
      const onStateChange = (state: unknown) => send("state", state);
      const onToolInvoke = (data: unknown) => send("invoke", data);
      const onToolResult = (data: unknown) => send("result", data);

      manager.on("state:change", onStateChange);
      manager.on("tool:invoke", onToolInvoke);
      manager.on("tool:result", onToolResult);

      // 30초마다 heartbeat
      const heartbeat = setInterval(() => {
        send("heartbeat", { time: new Date().toISOString() });
      }, 30_000);

      // 클린업
      const cleanup = () => {
        manager.off("state:change", onStateChange);
        manager.off("tool:invoke", onToolInvoke);
        manager.off("tool:result", onToolResult);
        clearInterval(heartbeat);
      };

      // AbortSignal로 연결 종료 감지
      controller.close = () => cleanup();
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      Connection: "keep-alive",
    },
  });
}
```

### 3. 도구 실행 API (`src/app/api/tools/invoke/route.ts`)

```typescript
import { getManager } from "@/lib/singleton";
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  const { serverId, toolName, args } = await request.json();

  if (!serverId || !toolName) {
    return NextResponse.json(
      { error: "serverId and toolName required" },
      { status: 400 }
    );
  }

  try {
    const manager = getManager();
    const result = await manager.invokeTool(serverId, toolName, args || {});

    return NextResponse.json({
      success: true,
      result,
      timestamp: new Date().toISOString(),
    });
  } catch (error) {
    return NextResponse.json(
      {
        success: false,
        error: error instanceof Error ? error.message : "Invocation failed",
      },
      { status: 500 }
    );
  }
}
```

### 4. 에이전트 상태 카드 컴포넌트 (`src/components/agent-card.tsx`)

```tsx
"use client";

import { Badge } from "@/components/ui/badge";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

interface AgentCardProps {
  id: string;
  name: string;
  status: "connecting" | "connected" | "error" | "disconnected";
  toolCount: number;
  lastPing: string | null;
  error?: string;
}

const statusConfig = {
  connecting: { label: "연결 중", color: "bg-yellow-500", variant: "outline" },
  connected: { label: "정상", color: "bg-green-500", variant: "default" },
  error: { label: "오류", color: "bg-red-500", variant: "destructive" },
  disconnected: { label: "끊김", color: "bg-gray-500", variant: "secondary" },
} as const;

export function AgentCard({
  name,
  status,
  toolCount,
  lastPing,
  error,
}: AgentCardProps) {
  const config = statusConfig[status];

  return (
    <Card className="w-full">
      <CardHeader className="flex flex-row items-center justify-between pb-2">
        <CardTitle className="text-sm font-medium">{name}</CardTitle>
        <Badge variant={config.variant as any}>
          <span className={`mr-1 h-2 w-2 rounded-full ${config.color}`} />
          {config.label}
        </Badge>
      </CardHeader>
      <CardContent>
        <div className="text-2xl font-bold">{toolCount} tools</div>
        <p className="text-xs text-muted-foreground mt-1">
          {lastPing
            ? `마지막 응답: ${new Date(lastPing).toLocaleTimeString()}`
            : "응답 없음"}
        </p>
        {error && (
          <p className="text-xs text-red-500 mt-2 truncate">{error}</p>
        )}
      </CardContent>
    </Card>
  );
}
```

### 5. 실시간 로그 뷰어 (`src/components/log-stream.tsx`)

```tsx
"use client";

import { useEffect, useRef, useState } from "react";

interface LogEntry {
  id: string;
  type: "invoke" | "result" | "error" | "state";
  serverId: string;
  toolName?: string;
  message: string;
  timestamp: string;
}

export function LogStream() {
  const [logs, setLogs] = useState<LogEntry[]>([]);
  const [connected, setConnected] = useState(false);
  const bottomRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const es = new EventSource("/api/stream");

    es.onopen = () => setConnected(true);
    es.onerror = () => setConnected(false);

    es.addEventListener("invoke", (e) => {
      const data = JSON.parse(e.data);
      addLog({
        type: "invoke",
        serverId: data.serverId,
        toolName: data.toolName,
        message: `${data.toolName} 호출 → ${JSON.stringify(data.args).slice(0, 100)}`,
      });
    });

    es.addEventListener("result", (e) => {
      const data = JSON.parse(e.data);
      addLog({
        type: "result",
        serverId: data.serverId,
        toolName: data.toolName,
        message: `${data.toolName} 완료`,
      });
    });

    es.addEventListener("state", (e) => {
      const data = JSON.parse(e.data);
      addLog({
        type: "state",
        serverId: data.id,
        message: `${data.name}: ${data.status}`,
      });
    });

    return () => es.close();
  }, []);

  const addLog = (partial: Omit<LogEntry, "id" | "timestamp">) => {
    setLogs((prev) => [
      ...prev.slice(-199), // 최대 200개 유지
      {
        ...partial,
        id: crypto.randomUUID(),
        timestamp: new Date().toISOString(),
      },
    ]);
  };

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [logs]);

  const typeColors = {
    invoke: "text-blue-400",
    result: "text-green-400",
    error: "text-red-400",
    state: "text-yellow-400",
  };

  return (
    <div className="bg-gray-950 rounded-lg p-4 h-96 overflow-y-auto font-mono text-sm">
      <div className="flex items-center gap-2 mb-2 text-xs text-gray-500">
        <span
          className={`h-2 w-2 rounded-full ${connected ? "bg-green-500" : "bg-red-500"}`}
        />
        {connected ? "스트림 연결됨" : "연결 끊김"}
      </div>
      {logs.map((log) => (
        <div key={log.id} className="flex gap-2 py-0.5">
          <span className="text-gray-600 shrink-0">
            {new Date(log.timestamp).toLocaleTimeString()}
          </span>
          <span className={`shrink-0 ${typeColors[log.type]}`}>
            [{log.type.toUpperCase()}]
          </span>
          <span className="text-gray-300">{log.message}</span>
        </div>
      ))}
      <div ref={bottomRef} />
    </div>
  );
}
```

### 6. MCP 서버 설정 파일 (`mcp-config.json`)

```json
{
  "servers": [
    {
      "id": "filesystem",
      "name": "파일 시스템",
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    {
      "id": "git",
      "name": "Git 관리",
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-git", "--repository", "."]
    },
    {
      "id": "postgres",
      "name": "PostgreSQL",
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_URL": "postgresql://user:pass@localhost:5432/mydb"
      }
    },
    {
      "id": "custom-api",
      "name": "커스텀 API 서버",
      "transport": "sse",
      "url": "http://localhost:8080/sse"
    }
  ]
}
```

## 실행 방법

```bash
# 1. 프로젝트 생성
npx create-next-app@latest mcp-dashboard --typescript --tailwind --app
cd mcp-dashboard

# 2. 의존성 설치
npm install @modelcontextprotocol/sdk
npx shadcn@latest init
npx shadcn@latest add card badge button input tabs

# 3. MCP 서버 설정 (mcp-config.json 수정)

# 4. 개발 서버 실행
npm run dev
```

## AI 코딩 에이전트에게 요청하는 프롬프트

### 초기 스캐폴딩

```
Next.js App Router + shadcn/ui로 MCP 서버 관리 대시보드를 만들어줘.
- /api/stream: SSE로 실시간 이벤트 스트리밍
- /api/tools/invoke: POST로 MCP 도구 호출
- 메인 페이지: 연결된 서버 상태 카드 그리드
- /tools: 전체 도구 목록 + 검색 + 실행 폼
- 실시간 로그 뷰어 하단에 배치
MCP SDK는 @modelcontextprotocol/sdk 사용.
```

### 도구 실행 폼 자동 생성

```
MCP 도구의 inputSchema (JSON Schema)를 받아서
자동으로 폼 필드를 렌더링하는 React 컴포넌트를 만들어줘.
- string → Input
- number → Input type=number
- boolean → Switch
- enum → Select
- object → 중첩 폼
- required 필드 표시
shadcn/ui 컴포넌트 사용.
```

## 확장 아이디어

| 기능 | 설명 |
|------|------|
| 도구 사용 통계 | 도구별 호출 횟수, 평균 응답 시간 차트 (recharts) |
| 프롬프트 플레이그라운드 | 자연어 입력 → AI가 적절한 도구 선택 → 실행 |
| 서버 헬스 체크 | 주기적 ping + 자동 재연결 + 알림 |
| 권한 관리 | 도구별/서버별 접근 권한 설정 (NextAuth.js) |
| 도구 체인 빌더 | 여러 도구를 시각적으로 연결하는 파이프라인 에디터 |
| MCP Inspector 연동 | MCP Inspector 프로토콜 로그 시각화 |

## 핵심 포인트

1. **MCP SDK 직접 사용**: `@modelcontextprotocol/sdk`로 stdio/SSE 트랜스포트 모두 지원
2. **싱글턴 매니저**: Next.js 서버에서 MCP 연결 풀을 싱글턴으로 관리해 리소스 절약
3. **SSE 실시간 업데이트**: 폴링 없이 서버 상태 변화와 도구 실행 결과를 즉시 반영
4. **스키마 기반 UI**: MCP 도구의 JSON Schema에서 폼을 자동 생성해 모든 도구에 범용 대응
5. **shadcn/ui 패턴**: 컴포넌트를 프로젝트에 복사하는 방식이라 대시보드 커스터마이징에 최적

## 참고 자료

- [MCP 공식 TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector)
- [Next.js App Router 문서](https://nextjs.org/docs/app)
- [shadcn/ui 컴포넌트](https://ui.shadcn.com/)
- [Server-Sent Events MDN](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)

---

*이 예제는 텐빌더의 AI 코딩 도구 실전 시리즈의 일부입니다.*
*MCP 서버를 직접 만들고 싶다면 → [커스텀 MCP 서버 구축 워크플로우](../workflows/custom-mcp-server.md)*
*MCP 생태계 전체를 보고 싶다면 → [MCP 생태계 치트시트](../cheatsheets/mcp-ecosystem-cheatsheet.md)*
