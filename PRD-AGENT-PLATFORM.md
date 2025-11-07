# Product Requirements Document: AI Agent Execution Platform

**Version:** 1.0
**Date:** 2025-01-07
**Status:** Greenfield - Ready to Build

---

## Executive Summary

Build a production-ready AI agent execution platform similar to Manus AI/Claude Code/Suna, featuring:
- Real-time agent streaming with full visibility
- Persistent sandboxed code execution
- Interactive UI showing thoughts, tool calls, and live previews
- Multi-user support with background task processing

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    FRONTEND (TypeScript)                      │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐ │
│  │  Chat Pane  │  │ Browser View │  │   Sidebar           │ │
│  │             │  │   (iframe)   │  │  - Thoughts         │ │
│  │ - Messages  │  │              │  │  - Tool Calls       │ │
│  │ - Input     │  │ Live Preview │  │  - Files            │ │
│  │ - Status    │  │              │  │  - Context Stats    │ │
│  └──────┬──────┘  └──────────────┘  └─────────────────────┘ │
└─────────┼─────────────────────────────────────────────────────┘
          │ SSE (Server-Sent Events)
          ▼
┌──────────────────────────────────────────────────────────────┐
│              BACKEND SERVER (Node.js/TypeScript)              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ API Layer (Next.js App Router or Express)             │  │
│  │  - POST /api/agent/start                              │  │
│  │  - GET  /api/agent/stream/:taskId (SSE)               │  │
│  │  - POST /api/agent/message                            │  │
│  │  - GET  /api/agent/status/:taskId                     │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Agent Orchestration Layer                             │  │
│  │  - Claude Agent SDK                                   │  │
│  │  - Skills (.claude/skills/)                           │  │
│  │  - Event Streaming (Redis Pub/Sub)                    │  │
│  │  - Context Cache (Redis)                              │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Background Workers (BullMQ)                           │  │
│  │  - Agent execution tasks                              │  │
│  │  - Long-running operations                            │  │
│  │  - Concurrent task processing                         │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────┬──────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐     ┌──────────┐
    │PostgreSQL│      │  Redis   │     │ Daytona  │
    │          │      │          │     │ Sandboxes│
    │- Tasks   │      │- Pub/Sub │     │          │
    │- Messages│      │- Cache   │     │- Persist │
    │- Users   │      │- Queue   │     │- Execute │
    └──────────┘      └──────────┘     └──────────┘
                                             │
                                    ┌────────┴────────┐
                                    │   MCP Servers   │
                                    │                 │
                                    │ - GitHub        │
                                    │ - Slack         │
                                    │ - Custom Tools  │
                                    └─────────────────┘
```

---

## Tech Stack

### Frontend
- **Framework:** Next.js 14+ (App Router) with TypeScript
- **UI Library:** React 18+
- **Styling:** Tailwind CSS
- **State Management:** Zustand or React Context
- **Real-time:** EventSource API (SSE)
- **Code Display:** Monaco Editor or CodeMirror

### Backend
- **Runtime:** Node.js 20+
- **Framework:** Next.js API Routes (or Express)
- **Language:** TypeScript
- **Agent SDK:** `@anthropic-ai/agent-sdk`
- **Task Queue:** BullMQ
- **Real-time:** Redis Pub/Sub

### Infrastructure
- **Database:** PostgreSQL (Supabase or self-hosted)
- **Cache/Queue:** Redis
- **Sandboxes:** Daytona
- **ORM:** Prisma
- **Hosting:** Vercel (frontend) + Railway/Render (workers)

### Development Tools
- **Package Manager:** pnpm
- **Linting:** ESLint + Prettier
- **Type Checking:** TypeScript strict mode
- **Testing:** Vitest + Playwright

---

## Database Schema

```prisma
// prisma/schema.prisma

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  tasks     AgentTask[]
  sessions  AgentSession[]
}

model AgentSession {
  id              String   @id @default(cuid())
  userId          String
  user            User     @relation(fields: [userId], references: [id])

  daytonaWorkspaceId String?

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  lastActivityAt  DateTime @default(now())

  tasks           AgentTask[]

  @@index([userId])
}

model AgentTask {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  sessionId   String
  session     AgentSession @relation(fields: [sessionId], references: [id])

  description String   @db.Text
  status      String   // queued, running, completed, failed

  result      Json?
  error       String?  @db.Text

  metadata    Json?    // Additional context

  createdAt   DateTime @default(now())
  completedAt DateTime?

  messages    AgentMessage[]
  events      AgentEvent[]

  @@index([userId])
  @@index([sessionId])
  @@index([status])
}

model AgentMessage {
  id        String   @id @default(cuid())
  taskId    String
  task      AgentTask @relation(fields: [taskId], references: [id], onDelete: Cascade)

  role      String   // user, assistant, system
  content   String   @db.Text

  createdAt DateTime @default(now())

  @@index([taskId])
}

model AgentEvent {
  id        String   @id @default(cuid())
  taskId    String
  task      AgentTask @relation(fields: [taskId], references: [id], onDelete: Cascade)

  type      String   // thought, tool_call, tool_result, file_change, etc.
  data      Json

  timestamp BigInt   // Unix timestamp in ms
  createdAt DateTime @default(now())

  @@index([taskId])
  @@index([type])
}
```

---

## Core Components Implementation

### 1. Event Streaming System

```typescript
// lib/event-streamer.ts

import Redis from 'ioredis';

export type AgentEvent =
  | { type: 'thought'; content: string; timestamp: number }
  | { type: 'tool_call'; tool: string; args: any; timestamp: number }
  | { type: 'tool_result'; tool: string; result: any; timestamp: number }
  | { type: 'message'; role: 'user' | 'assistant'; content: string; timestamp: number }
  | { type: 'code_generation'; language: string; code: string; filePath: string; timestamp: number }
  | { type: 'file_change'; path: string; action: 'create' | 'edit' | 'delete'; timestamp: number }
  | { type: 'browser_update'; url: string; timestamp: number }
  | { type: 'status'; status: 'thinking' | 'executing' | 'waiting' | 'complete'; timestamp: number }
  | { type: 'error'; error: string; timestamp: number }
  | { type: 'context_update'; tokensUsed: number; cacheHits: number; timestamp: number };

export class AgentEventStreamer {
  private redis: Redis;
  private publisher: Redis;

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');
    this.publisher = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');
  }

  async publishEvent(taskId: string, event: AgentEvent) {
    // Persist to Redis Stream (max 1000 events per task)
    await this.redis.xadd(
      `task:${taskId}:events`,
      'MAXLEN', '~', '1000',
      '*',
      'event', JSON.stringify(event)
    );

    // Publish for real-time subscribers (SSE)
    await this.publisher.publish(
      `task:${taskId}`,
      JSON.stringify(event)
    );

    // Also save to database for permanent storage
    await prisma.agentEvent.create({
      data: {
        taskId,
        type: event.type,
        data: event as any,
        timestamp: BigInt(event.timestamp),
      },
    });
  }

  async subscribe(taskId: string, callback: (event: AgentEvent) => void) {
    const subscriber = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');

    await subscriber.subscribe(`task:${taskId}`);

    subscriber.on('message', (channel, message) => {
      if (channel === `task:${taskId}`) {
        callback(JSON.parse(message));
      }
    });

    return () => subscriber.quit();
  }

  async getEventHistory(taskId: string, count = 100) {
    const events = await this.redis.xrevrange(
      `task:${taskId}:events`,
      '+',
      '-',
      'COUNT',
      count
    );

    return events.reverse().map(([id, fields]) => ({
      id,
      event: JSON.parse(fields[1]) as AgentEvent,
    }));
  }
}

export const eventStreamer = new AgentEventStreamer();
```

### 2. Agent Runner with Hooks

```typescript
// lib/agent-runner.ts

import { query, ClaudeAgentOptions } from '@anthropic-ai/agent-sdk';
import { eventStreamer } from './event-streamer';
import { Daytona } from '@daytonaio/sdk';
import { prisma } from './db';

export async function runStreamingAgent(
  taskId: string,
  userId: string,
  sessionId: string,
  task: string
) {
  let totalTokens = 0;
  let cacheHits = 0;

  // Get or create Daytona workspace
  const session = await prisma.agentSession.findUnique({
    where: { id: sessionId },
  });

  let workspaceId = session?.daytonaWorkspaceId;

  if (!workspaceId) {
    const daytona = new Daytona({ apiKey: process.env.DAYTONA_API_KEY! });
    const sandbox = await daytona.create({
      name: `agent-${userId}-${sessionId}`,
      image: process.env.DAYTONA_IMAGE || 'your-registry/agent-sandbox:latest',
      persistent: true,
      autoArchiveInterval: 10080, // 7 days
      ports: [3000, 8080, 5173],
    });

    workspaceId = sandbox.id;

    // Update session with workspace ID
    await prisma.agentSession.update({
      where: { id: sessionId },
      data: { daytonaWorkspaceId: workspaceId },
    });

    // Bootstrap workspace
    await sandbox.process.code_run('/usr/local/bin/bootstrap.sh');
  }

  const options: ClaudeAgentOptions = {
    systemPrompt: `You are an autonomous agent with real-time UI feedback.

      IMPORTANT:
      - Use <thinking> tags to explain your reasoning
      - Users can see your thoughts in real-time
      - Explain why you're using each tool
      - When generating code, explain what it does

      Your workspace: ${workspaceId}
      Files in /workspace/ persist across sessions.`,

    allowedTools: ['bash', 'files', 'browser', 'webSearch', 'skills'],
    skillsPath: '.claude/skills',

    mcpServers: {
      daytona: {
        command: 'daytona',
        args: ['mcp'],
        env: {
          DAYTONA_API_KEY: process.env.DAYTONA_API_KEY!,
          DAYTONA_WORKSPACE_ID: workspaceId,
        },
      },
    },

    maxTurns: 50,
    permissionMode: 'acceptEdits',

    hooks: {
      beforeTurn: async () => {
        await eventStreamer.publishEvent(taskId, {
          type: 'status',
          status: 'thinking',
          timestamp: Date.now(),
        });
      },

      afterResponse: async (response: any) => {
        // Extract thinking blocks
        const thinkingMatch = response.content?.match(/<thinking>(.*?)<\/thinking>/s);
        if (thinkingMatch) {
          await eventStreamer.publishEvent(taskId, {
            type: 'thought',
            content: thinkingMatch[1].trim(),
            timestamp: Date.now(),
          });
        }

        // Track token usage
        if (response.usage) {
          totalTokens += response.usage.input_tokens + response.usage.output_tokens;
          cacheHits += response.usage.cache_read_input_tokens || 0;

          await eventStreamer.publishEvent(taskId, {
            type: 'context_update',
            tokensUsed: totalTokens,
            cacheHits,
            timestamp: Date.now(),
          });
        }
      },

      beforeToolCall: async (toolName: string, args: any) => {
        await eventStreamer.publishEvent(taskId, {
          type: 'tool_call',
          tool: toolName,
          args,
          timestamp: Date.now(),
        });

        await eventStreamer.publishEvent(taskId, {
          type: 'status',
          status: 'executing',
          timestamp: Date.now(),
        });
      },

      afterToolCall: async (toolName: string, args: any, result: any) => {
        await eventStreamer.publishEvent(taskId, {
          type: 'tool_result',
          tool: toolName,
          result,
          timestamp: Date.now(),
        });

        // File change tracking
        if (toolName === 'edit_file' || toolName === 'write_file') {
          await eventStreamer.publishEvent(taskId, {
            type: 'file_change',
            path: args.path || result.path,
            action: toolName === 'write_file' ? 'create' : 'edit',
            timestamp: Date.now(),
          });

          // Code generation event
          if (args.path?.match(/\.(ts|js|py|jsx|tsx)$/)) {
            await eventStreamer.publishEvent(taskId, {
              type: 'code_generation',
              language: args.path.split('.').pop()!,
              code: args.content || result.content,
              filePath: args.path,
              timestamp: Date.now(),
            });
          }
        }

        // Browser preview detection
        if (result?.stdout?.match(/https?:\/\/localhost:\d+/)) {
          const urlMatch = result.stdout.match(/https?:\/\/localhost:(\d+)/);
          if (urlMatch) {
            // Map to Daytona preview URL
            const previewUrl = `https://${workspaceId}-${urlMatch[1]}.daytona.app`;
            await eventStreamer.publishEvent(taskId, {
              type: 'browser_update',
              url: previewUrl,
              timestamp: Date.now(),
            });
          }
        }
      },

      onError: async (error: Error) => {
        await eventStreamer.publishEvent(taskId, {
          type: 'error',
          error: error.message,
          timestamp: Date.now(),
        });
      },
    },
  };

  try {
    // Update task status
    await prisma.agentTask.update({
      where: { id: taskId },
      data: { status: 'running' },
    });

    // Run agent
    for await (const message of query(task, options)) {
      if (message.type === 'text') {
        await eventStreamer.publishEvent(taskId, {
          type: 'message',
          role: 'assistant',
          content: message.content,
          timestamp: Date.now(),
        });

        // Save to database
        await prisma.agentMessage.create({
          data: {
            taskId,
            role: 'assistant',
            content: message.content,
          },
        });
      }
    }

    // Mark complete
    await eventStreamer.publishEvent(taskId, {
      type: 'status',
      status: 'complete',
      timestamp: Date.now(),
    });

    await prisma.agentTask.update({
      where: { id: taskId },
      data: {
        status: 'completed',
        completedAt: new Date(),
      },
    });
  } catch (error: any) {
    await eventStreamer.publishEvent(taskId, {
      type: 'error',
      error: error.message,
      timestamp: Date.now(),
    });

    await prisma.agentTask.update({
      where: { id: taskId },
      data: {
        status: 'failed',
        error: error.message,
        completedAt: new Date(),
      },
    });

    throw error;
  }
}
```

### 3. Background Worker (BullMQ)

```typescript
// workers/agent-worker.ts

import { Worker, Queue } from 'bullmq';
import Redis from 'ioredis';
import { runStreamingAgent } from '../lib/agent-runner';

const connection = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');

export const agentQueue = new Queue('agent-tasks', { connection });

export const agentWorker = new Worker(
  'agent-tasks',
  async (job) => {
    const { taskId, userId, sessionId, task } = job.data;

    console.log(`[Worker] Starting agent task ${taskId}`);

    await runStreamingAgent(taskId, userId, sessionId, task);

    console.log(`[Worker] Completed agent task ${taskId}`);
  },
  {
    connection,
    concurrency: 5, // Process 5 tasks concurrently
    limiter: {
      max: 10,
      duration: 1000, // Max 10 jobs per second
    },
  }
);

agentWorker.on('completed', (job) => {
  console.log(`✅ Job ${job.id} completed`);
});

agentWorker.on('failed', (job, err) => {
  console.error(`❌ Job ${job?.id} failed:`, err.message);
});
```

### 4. API Routes (Next.js)

```typescript
// app/api/agent/start/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/db';
import { agentQueue } from '@/workers/agent-worker';
import { z } from 'zod';

const startAgentSchema = z.object({
  task: z.string().min(1),
  userId: z.string(),
  sessionId: z.string().optional(),
});

export async function POST(req: NextRequest) {
  try {
    const body = await req.json();
    const { task, userId, sessionId: providedSessionId } = startAgentSchema.parse(body);

    // Get or create session
    let session;
    if (providedSessionId) {
      session = await prisma.agentSession.findUnique({
        where: { id: providedSessionId },
      });
    }

    if (!session) {
      session = await prisma.agentSession.create({
        data: {
          userId,
        },
      });
    }

    // Create task
    const agentTask = await prisma.agentTask.create({
      data: {
        userId,
        sessionId: session.id,
        description: task,
        status: 'queued',
      },
    });

    // Create user message
    await prisma.agentMessage.create({
      data: {
        taskId: agentTask.id,
        role: 'user',
        content: task,
      },
    });

    // Queue the job
    await agentQueue.add('agent-task', {
      taskId: agentTask.id,
      userId,
      sessionId: session.id,
      task,
    });

    return NextResponse.json({
      taskId: agentTask.id,
      sessionId: session.id,
      status: 'queued',
      streamUrl: `/api/agent/stream/${agentTask.id}`,
    });
  } catch (error: any) {
    console.error('Error starting agent:', error);
    return NextResponse.json(
      { error: error.message },
      { status: 400 }
    );
  }
}
```

```typescript
// app/api/agent/stream/[taskId]/route.ts

import { NextRequest } from 'next/server';
import { eventStreamer } from '@/lib/event-streamer';

export async function GET(
  req: NextRequest,
  { params }: { params: { taskId: string } }
) {
  const { taskId } = params;

  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    async start(controller) {
      // Send event history first
      const history = await eventStreamer.getEventHistory(taskId);
      for (const { event } of history) {
        controller.enqueue(
          encoder.encode(`data: ${JSON.stringify(event)}\n\n`)
        );
      }

      // Subscribe to new events
      const unsubscribe = await eventStreamer.subscribe(taskId, (event) => {
        controller.enqueue(
          encoder.encode(`data: ${JSON.stringify(event)}\n\n`)
        );
      });

      // Cleanup on close
      req.signal.addEventListener('abort', () => {
        unsubscribe();
        controller.close();
      });
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

```typescript
// app/api/agent/message/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/db';
import { eventStreamer } from '@/lib/event-streamer';
import { z } from 'zod';

const messageSchema = z.object({
  taskId: z.string(),
  message: z.string(),
});

export async function POST(req: NextRequest) {
  try {
    const { taskId, message } = messageSchema.parse(await req.json());

    // Save user message
    await prisma.agentMessage.create({
      data: {
        taskId,
        role: 'user',
        content: message,
      },
    });

    // Publish event
    await eventStreamer.publishEvent(taskId, {
      type: 'message',
      role: 'user',
      content: message,
      timestamp: Date.now(),
    });

    // TODO: Resume agent with new message
    // This requires agent pause/resume functionality

    return NextResponse.json({ success: true });
  } catch (error: any) {
    return NextResponse.json({ error: error.message }, { status: 400 });
  }
}
```

### 5. Frontend Hook

```typescript
// hooks/useAgentStream.ts

import { useEffect, useState } from 'react';
import { AgentEvent } from '@/types';

export function useAgentStream(taskId: string | null) {
  const [events, setEvents] = useState<AgentEvent[]>([]);
  const [status, setStatus] = useState<string>('waiting');
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    if (!taskId) return;

    const eventSource = new EventSource(`/api/agent/stream/${taskId}`);

    eventSource.onopen = () => {
      setIsConnected(true);
    };

    eventSource.onmessage = (event) => {
      const agentEvent: AgentEvent = JSON.parse(event.data);

      setEvents((prev) => [...prev, agentEvent]);

      if (agentEvent.type === 'status') {
        setStatus(agentEvent.status);
      }
    };

    eventSource.onerror = () => {
      setIsConnected(false);
    };

    return () => {
      eventSource.close();
    };
  }, [taskId]);

  return { events, status, isConnected };
}
```

### 6. Main UI Component

```typescript
// components/AgentViewer.tsx

'use client';

import { useAgentStream } from '@/hooks/useAgentStream';
import { useState } from 'react';
import { AgentEvent } from '@/types';

export function AgentViewer({ taskId }: { taskId: string }) {
  const { events, status, isConnected } = useAgentStream(taskId);
  const [selectedTab, setSelectedTab] = useState<'chat' | 'thoughts' | 'tools' | 'files'>('chat');

  const thoughts = events.filter(e => e.type === 'thought');
  const toolCalls = events.filter(e => e.type === 'tool_call' || e.type === 'tool_result');
  const fileChanges = events.filter(e => e.type === 'file_change');
  const messages = events.filter(e => e.type === 'message');
  const browserUpdate = events.filter(e => e.type === 'browser_update').pop();
  const contextUpdate = events.filter(e => e.type === 'context_update').pop();

  return (
    <div className="flex h-screen bg-gray-50">
      {/* Left Panel */}
      <div className="flex-1 flex flex-col bg-white">
        {/* Header */}
        <div className="border-b px-6 py-4 bg-gray-50">
          <div className="flex items-center justify-between">
            <div className="flex items-center gap-3">
              <StatusIndicator status={status} connected={isConnected} />
              <div>
                <h2 className="font-semibold text-gray-900">Agent Status</h2>
                <p className="text-sm text-gray-500">{status}</p>
              </div>
            </div>

            {contextUpdate && (
              <div className="text-right text-sm text-gray-600">
                <div>Tokens: {contextUpdate.tokensUsed.toLocaleString()}</div>
                <div className="text-xs text-gray-500">
                  Cache hits: {contextUpdate.cacheHits.toLocaleString()}
                </div>
              </div>
            )}
          </div>
        </div>

        {/* Tabs */}
        <div className="border-b flex bg-white">
          <Tab active={selectedTab === 'chat'} onClick={() => setSelectedTab('chat')}>
            💬 Chat ({messages.length})
          </Tab>
          <Tab active={selectedTab === 'thoughts'} onClick={() => setSelectedTab('thoughts')}>
            🧠 Thoughts ({thoughts.length})
          </Tab>
          <Tab active={selectedTab === 'tools'} onClick={() => setSelectedTab('tools')}>
            🔧 Tools ({toolCalls.length})
          </Tab>
          <Tab active={selectedTab === 'files'} onClick={() => setSelectedTab('files')}>
            📁 Files ({fileChanges.length})
          </Tab>
        </div>

        {/* Content */}
        <div className="flex-1 overflow-auto p-6">
          {selectedTab === 'chat' && <ChatView messages={messages} />}
          {selectedTab === 'thoughts' && <ThoughtsView thoughts={thoughts} />}
          {selectedTab === 'tools' && <ToolsView toolCalls={toolCalls} />}
          {selectedTab === 'files' && <FilesView fileChanges={fileChanges} />}
        </div>
      </div>

      {/* Right Panel: Browser Preview */}
      <div className="w-1/2 border-l flex flex-col bg-white">
        <div className="border-b px-4 py-3 bg-gray-50 flex items-center gap-2">
          <span className="text-sm font-semibold text-gray-700">🌐 Live Preview</span>
          {browserUpdate && (
            <span className="text-xs text-gray-500 truncate">{browserUpdate.url}</span>
          )}
        </div>

        <div className="flex-1">
          {browserUpdate ? (
            <iframe
              src={browserUpdate.url}
              className="w-full h-full border-0"
              sandbox="allow-scripts allow-same-origin allow-forms"
            />
          ) : (
            <div className="flex items-center justify-center h-full">
              <div className="text-center text-gray-400">
                <div className="text-4xl mb-2">🌐</div>
                <p>No preview available yet</p>
              </div>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}

function StatusIndicator({ status, connected }: { status: string; connected: boolean }) {
  const colors = {
    thinking: 'bg-purple-500 animate-pulse',
    executing: 'bg-yellow-500 animate-pulse',
    waiting: 'bg-gray-400',
    complete: 'bg-green-500',
  };

  return (
    <div className="relative">
      <div className={`w-3 h-3 rounded-full ${colors[status as keyof typeof colors] || 'bg-gray-400'}`} />
      {!connected && (
        <div className="absolute -top-1 -right-1 w-2 h-2 bg-red-500 rounded-full" />
      )}
    </div>
  );
}

function Tab({ active, onClick, children }: any) {
  return (
    <button
      onClick={onClick}
      className={`px-4 py-3 text-sm font-medium transition-colors ${
        active
          ? 'border-b-2 border-blue-500 text-blue-600'
          : 'text-gray-600 hover:text-gray-900'
      }`}
    >
      {children}
    </button>
  );
}

function ChatView({ messages }: { messages: any[] }) {
  return (
    <div className="space-y-4">
      {messages.map((msg, i) => (
        <div key={i} className={`rounded-lg p-4 ${
          msg.role === 'user'
            ? 'bg-blue-50 border border-blue-200'
            : 'bg-gray-50 border border-gray-200'
        }`}>
          <div className="text-xs text-gray-500 mb-2 flex items-center gap-2">
            <span className="font-semibold">{msg.role}</span>
            <span>•</span>
            <span>{new Date(msg.timestamp).toLocaleTimeString()}</span>
          </div>
          <div className="prose prose-sm max-w-none">
            {msg.content}
          </div>
        </div>
      ))}
    </div>
  );
}

function ThoughtsView({ thoughts }: { thoughts: any[] }) {
  return (
    <div className="space-y-3">
      {thoughts.map((thought, i) => (
        <div key={i} className="bg-purple-50 border-l-4 border-purple-400 rounded p-4">
          <div className="text-xs text-purple-600 mb-2">
            💭 {new Date(thought.timestamp).toLocaleTimeString()}
          </div>
          <div className="text-sm whitespace-pre-wrap font-mono text-gray-700">
            {thought.content}
          </div>
        </div>
      ))}
    </div>
  );
}

function ToolsView({ toolCalls }: { toolCalls: any[] }) {
  return (
    <div className="space-y-3">
      {toolCalls.map((event, i) => (
        <div key={i} className={`rounded border-l-4 p-4 ${
          event.type === 'tool_call'
            ? 'bg-yellow-50 border-yellow-400'
            : 'bg-green-50 border-green-400'
        }`}>
          <div className="text-xs mb-2 flex items-center gap-2">
            <span className="font-semibold">
              {event.type === 'tool_call' ? '🔧 Calling' : '✅ Result'}
            </span>
            <span>•</span>
            <span className="font-mono">{event.tool}</span>
            <span>•</span>
            <span>{new Date(event.timestamp).toLocaleTimeString()}</span>
          </div>
          <pre className="text-xs bg-white p-3 rounded overflow-auto border">
            {JSON.stringify(
              event.type === 'tool_call' ? event.args : event.result,
              null,
              2
            )}
          </pre>
        </div>
      ))}
    </div>
  );
}

function FilesView({ fileChanges }: { fileChanges: any[] }) {
  return (
    <div className="space-y-2">
      {fileChanges.map((change, i) => (
        <div key={i} className="bg-blue-50 border border-blue-200 rounded p-4 flex items-center gap-3">
          <span className="text-2xl">
            {change.action === 'create' ? '➕' : change.action === 'edit' ? '✏️' : '🗑️'}
          </span>
          <div className="flex-1">
            <div className="font-mono text-sm font-semibold">{change.path}</div>
            <div className="text-xs text-gray-500 mt-1">
              {change.action} • {new Date(change.timestamp).toLocaleTimeString()}
            </div>
          </div>
        </div>
      ))}
    </div>
  );
}
```

---

## Custom Docker Image Setup

### Dockerfile

```dockerfile
# Dockerfile.agent-sandbox

FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    build-essential \
    postgresql-client \
    redis-tools \
    nodejs \
    npm \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js 20
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
    apt-get install -y nodejs

# Install Python packages
RUN pip install --no-cache-dir \
    pandas \
    numpy \
    matplotlib \
    seaborn \
    scipy \
    scikit-learn \
    requests \
    beautifulsoup4 \
    sqlalchemy \
    psycopg2-binary \
    redis \
    httpx \
    pydantic \
    pytest \
    jupyterlab

# Create directories
WORKDIR /workspace
RUN mkdir -p /templates/skill-scripts /usr/local/skill-scripts

# Copy bootstrap script
COPY bootstrap.sh /usr/local/bin/bootstrap.sh
RUN chmod +x /usr/local/bin/bootstrap.sh

# Copy skill script templates
COPY skill-scripts/ /templates/skill-scripts/
RUN chmod +x /templates/skill-scripts/*.sh
RUN chmod +x /templates/skill-scripts/*.py

# Set environment
ENV PYTHONUNBUFFERED=1
ENV WORKSPACE=/workspace
ENV PATH="/workspace/skill-scripts:${PATH}"

CMD ["/bin/bash"]
```

### Bootstrap Script

```bash
#!/bin/bash
# /usr/local/bin/bootstrap.sh

WORKSPACE_SCRIPTS="/workspace/skill-scripts"
TEMPLATE_SCRIPTS="/templates/skill-scripts"
INIT_FLAG="/workspace/.initialized"

if [ -f "$INIT_FLAG" ]; then
    echo "Workspace already initialized"
    exit 0
fi

echo "Initializing workspace..."

# Copy skill scripts to workspace
mkdir -p "$WORKSPACE_SCRIPTS"
cp -r "$TEMPLATE_SCRIPTS"/* "$WORKSPACE_SCRIPTS"/
chmod +x "$WORKSPACE_SCRIPTS"/*.sh
chmod +x "$WORKSPACE_SCRIPTS"/*.py

# Create directory structure
mkdir -p /workspace/{data,results,temp,notebooks}

# Add to PATH
echo 'export PATH="/workspace/skill-scripts:$PATH"' >> /workspace/.bashrc

# Create README
cat > /workspace/README.md << 'EOF'
# Agent Workspace

This workspace persists across sessions.

## Directories
- `/workspace/skill-scripts/` - Editable scripts
- `/workspace/data/` - Input data
- `/workspace/results/` - Outputs
- `/workspace/temp/` - Temporary files
- `/workspace/notebooks/` - Jupyter notebooks

## Scripts
All scripts in `/workspace/skill-scripts/` are editable.
Templates in `/templates/skill-scripts/` (read-only).

## Reset
To restore scripts:
```bash
cp /templates/skill-scripts/* /workspace/skill-scripts/
```
EOF

touch "$INIT_FLAG"
echo "✅ Workspace initialized"
```

---

## Skills Setup

### Data Analysis Skill

```markdown
<!-- .claude/skills/data-analysis/SKILL.md -->
---
name: Data Analysis
description: Analyze CSV/Excel files, create visualizations, statistical analysis. Use when user asks to analyze data, create charts, or examine datasets.
---

# Data Analysis Skill

Pre-installed data science tools in your Daytona workspace.

## Pre-installed Scripts

### analyze_csv.py

Location: `/workspace/skill-scripts/analyze_csv.py`

Quick CSV analysis with statistics.

**Usage:**
```bash
python /workspace/skill-scripts/analyze_csv.py /workspace/data.csv
```

**Output:** JSON with shape, columns, dtypes, missing values, statistics

**Editable:** YES - You can improve this script!

## Workflow

1. Upload data to `/workspace/data/`
2. Run `analyze_csv.py` for quick overview
3. Write custom analysis code if needed
4. Save results to `/workspace/results/`

## Pre-installed Packages

- pandas, numpy, scipy
- matplotlib, seaborn
- scikit-learn
- statsmodels

## Examples

See `examples.md` for common patterns.
```

### Web Scraping Skill

```markdown
<!-- .claude/skills/web-scraping/SKILL.md -->
---
name: Web Scraping
description: Extract data from websites, parse HTML, scrape tables. Use when user asks to get data from web pages.
---

# Web Scraping Skill

## Pre-installed Scripts

### web_scraper.py

Location: `/workspace/skill-scripts/web_scraper.py`

**Usage:**
```bash
python /workspace/skill-scripts/web_scraper.py "https://example.com" ".product-title"
```

## Pre-installed Libraries

- requests, httpx
- beautifulsoup4
- scrapy (for advanced scraping)

## Best Practices

1. Respect robots.txt
2. Add delays between requests
3. Set proper User-Agent
4. Handle errors gracefully

See `examples.md` for patterns.
```

---

## Environment Variables

```bash
# .env

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/agent_platform

# Redis
REDIS_URL=redis://localhost:6379

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...

# Daytona
DAYTONA_API_KEY=...
DAYTONA_API_URL=https://api.daytona.io
DAYTONA_IMAGE=your-registry/agent-sandbox:latest

# Next.js
NEXT_PUBLIC_API_URL=http://localhost:3000
```

---

## Project Structure

```
agent-platform/
├── app/                          # Next.js App Router
│   ├── api/
│   │   └── agent/
│   │       ├── start/route.ts
│   │       ├── stream/[taskId]/route.ts
│   │       └── message/route.ts
│   ├── dashboard/
│   │   └── page.tsx
│   └── agent/[taskId]/
│       └── page.tsx
├── components/
│   ├── AgentViewer.tsx
│   ├── ChatView.tsx
│   ├── ThoughtsView.tsx
│   └── ToolsView.tsx
├── hooks/
│   └── useAgentStream.ts
├── lib/
│   ├── agent-runner.ts
│   ├── event-streamer.ts
│   ├── db.ts
│   └── context-cache.ts
├── workers/
│   └── agent-worker.ts
├── prisma/
│   └── schema.prisma
├── .claude/
│   └── skills/
│       ├── data-analysis/
│       │   ├── SKILL.md
│       │   └── examples.md
│       └── web-scraping/
│           ├── SKILL.md
│           └── examples.md
├── docker/
│   ├── Dockerfile.agent-sandbox
│   ├── bootstrap.sh
│   └── skill-scripts/
│       ├── analyze_csv.py
│       └── web_scraper.py
├── package.json
├── tsconfig.json
├── next.config.js
└── .env
```

---

## Implementation Roadmap

### Phase 1: Foundation (Week 1)
**Total Tasks: 12**

#### Database & Infrastructure (4 tasks)
- [ ] **Task 1.1:** Set up PostgreSQL database (local or Supabase)
  - *Deliverable:* Database running and accessible
  - *Time:* 2 hours

- [ ] **Task 1.2:** Set up Redis instance (local or cloud)
  - *Deliverable:* Redis running and accessible
  - *Time:* 1 hour

- [ ] **Task 1.3:** Initialize Prisma with schema
  - *Deliverable:* `prisma/schema.prisma` created, migrations run
  - *Time:* 2 hours

- [ ] **Task 1.4:** Set up Next.js project with TypeScript
  - *Deliverable:* Next.js app running on localhost:3000
  - *Time:* 1 hour

#### Event Streaming System (4 tasks)
- [ ] **Task 1.5:** Implement `AgentEventStreamer` class
  - *Deliverable:* `lib/event-streamer.ts` with Redis pub/sub
  - *Time:* 3 hours

- [ ] **Task 1.6:** Create SSE endpoint `/api/agent/stream/[taskId]`
  - *Deliverable:* Working SSE endpoint
  - *Time:* 2 hours

- [ ] **Task 1.7:** Build `useAgentStream` hook
  - *Deliverable:* React hook consuming SSE
  - *Time:* 2 hours

- [ ] **Task 1.8:** Test event streaming end-to-end
  - *Deliverable:* Mock events flowing to frontend
  - *Time:* 2 hours

#### Background Workers (4 tasks)
- [ ] **Task 1.9:** Set up BullMQ with Redis
  - *Deliverable:* Queue and worker configured
  - *Time:* 2 hours

- [ ] **Task 1.10:** Create agent worker stub
  - *Deliverable:* `workers/agent-worker.ts` processing mock jobs
  - *Time:* 2 hours

- [ ] **Task 1.11:** Implement `/api/agent/start` endpoint
  - *Deliverable:* Endpoint queuing tasks to BullMQ
  - *Time:* 2 hours

- [ ] **Task 1.12:** Test job queuing and processing
  - *Deliverable:* Jobs queued, picked up by workers
  - *Time:* 2 hours

### Phase 2: Agent Integration (Week 2)
**Total Tasks: 10**

#### Claude Agent SDK Setup (3 tasks)
- [ ] **Task 2.1:** Install Claude Agent SDK and dependencies
  - *Deliverable:* `@anthropic-ai/agent-sdk` installed
  - *Time:* 1 hour

- [ ] **Task 2.2:** Create basic agent runner without hooks
  - *Deliverable:* Simple query execution working
  - *Time:* 3 hours

- [ ] **Task 2.3:** Add API key configuration and validation
  - *Deliverable:* Environment variables set up
  - *Time:* 1 hour

#### Agent Hooks Implementation (4 tasks)
- [ ] **Task 2.4:** Implement `beforeTurn` and `afterResponse` hooks
  - *Deliverable:* Thinking blocks extracted and streamed
  - *Time:* 3 hours

- [ ] **Task 2.5:** Implement `beforeToolCall` and `afterToolCall` hooks
  - *Deliverable:* Tool calls/results streamed
  - *Time:* 3 hours

- [ ] **Task 2.6:** Add token usage tracking
  - *Deliverable:* Context updates streamed
  - *Time:* 2 hours

- [ ] **Task 2.7:** Add error handling hook
  - *Deliverable:* Errors streamed to frontend
  - *Time:* 2 hours

#### Integration Testing (3 tasks)
- [ ] **Task 2.8:** Test full agent execution flow
  - *Deliverable:* Agent task completes with all events
  - *Time:* 3 hours

- [ ] **Task 2.9:** Test concurrent task execution
  - *Deliverable:* Multiple agents running simultaneously
  - *Time:* 2 hours

- [ ] **Task 2.10:** Add comprehensive error handling
  - *Deliverable:* Graceful failure modes
  - *Time:* 2 hours

### Phase 3: Daytona Integration (Week 3)
**Total Tasks: 12**

#### Docker Image Creation (4 tasks)
- [ ] **Task 3.1:** Write `Dockerfile.agent-sandbox`
  - *Deliverable:* Dockerfile with all dependencies
  - *Time:* 3 hours

- [ ] **Task 3.2:** Create bootstrap script
  - *Deliverable:* `bootstrap.sh` initializing workspace
  - *Time:* 2 hours

- [ ] **Task 3.3:** Write skill scripts (analyze_csv.py, web_scraper.py)
  - *Deliverable:* 2-3 utility scripts
  - *Time:* 4 hours

- [ ] **Task 3.4:** Build and push Docker image
  - *Deliverable:* Image in registry
  - *Time:* 2 hours

#### Daytona SDK Integration (4 tasks)
- [ ] **Task 3.5:** Install Daytona SDK
  - *Deliverable:* `@daytonaio/sdk` installed
  - *Time:* 1 hour

- [ ] **Task 3.6:** Implement workspace creation/reuse logic
  - *Deliverable:* Persistent workspaces per session
  - *Time:* 4 hours

- [ ] **Task 3.7:** Set up Daytona MCP server integration
  - *Deliverable:* MCP server configured in agent options
  - *Time:* 3 hours

- [ ] **Task 3.8:** Test sandbox execution
  - *Deliverable:* Code running in Daytona sandbox
  - *Time:* 2 hours

#### Browser Preview (4 tasks)
- [ ] **Task 3.9:** Detect dev server URLs from tool results
  - *Deliverable:* Browser update events emitted
  - *Time:* 2 hours

- [ ] **Task 3.10:** Map to Daytona preview URLs
  - *Deliverable:* Correct preview URL generated
  - *Time:* 2 hours

- [ ] **Task 3.11:** Add iframe component to UI
  - *Deliverable:* Live preview displaying
  - *Time:* 2 hours

- [ ] **Task 3.12:** Test end-to-end preview flow
  - *Deliverable:* Create app → see live preview
  - *Time:* 2 hours

### Phase 4: Skills System (Week 4)
**Total Tasks: 8**

#### Skills Infrastructure (3 tasks)
- [ ] **Task 4.1:** Create `.claude/skills/` directory structure
  - *Deliverable:* Directory created with subdirectories
  - *Time:* 1 hour

- [ ] **Task 4.2:** Write data-analysis skill
  - *Deliverable:* `SKILL.md` + `examples.md`
  - *Time:* 3 hours

- [ ] **Task 4.3:** Write web-scraping skill
  - *Deliverable:* `SKILL.md` + `examples.md`
  - *Time:* 3 hours

#### Skills Testing (3 tasks)
- [ ] **Task 4.4:** Enable skills in agent options
  - *Deliverable:* Skills loading correctly
  - *Time:* 1 hour

- [ ] **Task 4.5:** Test skill activation
  - *Deliverable:* Agent uses skills appropriately
  - *Time:* 2 hours

- [ ] **Task 4.6:** Test script execution in sandbox
  - *Deliverable:* Skill scripts run successfully
  - *Time:* 2 hours

#### Documentation (2 tasks)
- [ ] **Task 4.7:** Document skill authoring process
  - *Deliverable:* `SKILLS.md` guide
  - *Time:* 2 hours

- [ ] **Task 4.8:** Create skill templates
  - *Deliverable:* Template files for new skills
  - *Time:* 2 hours

### Phase 5: Frontend UI (Week 5)
**Total Tasks: 14**

#### Layout & Navigation (3 tasks)
- [ ] **Task 5.1:** Build main layout with split panes
  - *Deliverable:* Responsive layout component
  - *Time:* 3 hours

- [ ] **Task 5.2:** Create tab navigation (Chat/Thoughts/Tools/Files)
  - *Deliverable:* Tab component with routing
  - *Time:* 2 hours

- [ ] **Task 5.3:** Add status indicator and header
  - *Deliverable:* Header with connection status
  - *Time:* 2 hours

#### Event Views (5 tasks)
- [ ] **Task 5.4:** Build ChatView component
  - *Deliverable:* Message list with styling
  - *Time:* 3 hours

- [ ] **Task 5.5:** Build ThoughtsView component
  - *Deliverable:* Thought bubbles with timestamps
  - *Time:* 2 hours

- [ ] **Task 5.6:** Build ToolsView component
  - *Deliverable:* Tool call/result cards
  - *Time:* 3 hours

- [ ] **Task 5.7:** Build FilesView component
  - *Deliverable:* File change list
  - *Time:* 2 hours

- [ ] **Task 5.8:** Add syntax highlighting for code
  - *Deliverable:* Monaco/CodeMirror integrated
  - *Time:* 3 hours

#### Interactivity (4 tasks)
- [ ] **Task 5.9:** Add message input component
  - *Deliverable:* Input with send button
  - *Time:* 2 hours

- [ ] **Task 5.10:** Implement auto-scroll on new events
  - *Deliverable:* Smooth scrolling behavior
  - *Time:* 2 hours

- [ ] **Task 5.11:** Add loading states and skeletons
  - *Deliverable:* Loading indicators
  - *Time:* 2 hours

- [ ] **Task 5.12:** Add error boundaries
  - *Deliverable:* Graceful error handling
  - *Time:* 2 hours

#### Polish (2 tasks)
- [ ] **Task 5.13:** Style with Tailwind CSS
  - *Deliverable:* Consistent design system
  - *Time:* 4 hours

- [ ] **Task 5.14:** Add animations and transitions
  - *Deliverable:* Smooth UI transitions
  - *Time:* 3 hours

### Phase 6: Production Readiness (Week 6)
**Total Tasks: 10**

#### Performance (3 tasks)
- [ ] **Task 6.1:** Implement context caching in Redis
  - *Deliverable:* `context-cache.ts` with LRU cache
  - *Time:* 3 hours

- [ ] **Task 6.2:** Add event pagination/windowing
  - *Deliverable:* Only load recent 100 events
  - *Time:* 3 hours

- [ ] **Task 6.3:** Optimize database queries
  - *Deliverable:* Indexes, eager loading
  - *Time:* 2 hours

#### Security (3 tasks)
- [ ] **Task 6.4:** Add authentication (NextAuth.js)
  - *Deliverable:* User login/signup
  - *Time:* 4 hours

- [ ] **Task 6.5:** Implement authorization middleware
  - *Deliverable:* User can only access own tasks
  - *Time:* 3 hours

- [ ] **Task 6.6:** Add rate limiting
  - *Deliverable:* Rate limits per user
  - *Time:* 2 hours

#### Monitoring (2 tasks)
- [ ] **Task 6.7:** Add logging (Winston or Pino)
  - *Deliverable:* Structured logging
  - *Time:* 2 hours

- [ ] **Task 6.8:** Add error tracking (Sentry)
  - *Deliverable:* Error monitoring configured
  - *Time:* 2 hours

#### Deployment (2 tasks)
- [ ] **Task 6.9:** Set up CI/CD pipeline
  - *Deliverable:* GitHub Actions workflow
  - *Time:* 3 hours

- [ ] **Task 6.10:** Deploy to production
  - *Deliverable:* App live on Vercel + Railway
  - *Time:* 4 hours

---

## Total Project Summary

**Total Tasks:** 66 tasks
**Estimated Time:** 170 hours (≈6 weeks at 30 hrs/week)

**Breakdown by Phase:**
- Phase 1 (Foundation): 12 tasks, 24 hours
- Phase 2 (Agent Integration): 10 tasks, 22 hours
- Phase 3 (Daytona): 12 tasks, 31 hours
- Phase 4 (Skills): 8 tasks, 16 hours
- Phase 5 (Frontend): 14 tasks, 36 hours
- Phase 6 (Production): 10 tasks, 25 hours

---

## Missing Components & Decisions Needed

### 1. Authentication Strategy
**Options:**
- [ ] NextAuth.js with email/password
- [ ] Auth0 or Clerk
- [ ] Supabase Auth
- [ ] Custom JWT implementation

**Decision needed:** Which auth provider?

### 2. File Upload/Download
**Missing:**
- [ ] File upload endpoint for user data
- [ ] File browser UI component
- [ ] Direct file access from Daytona workspace
- [ ] File size limits and validation

**Decision needed:** Max file size? Storage location (S3, local, Daytona)?

### 3. Agent Pause/Resume
**Missing:**
- [ ] Pause agent mid-execution
- [ ] Resume from paused state
- [ ] User interruption handling
- [ ] Checkpoint/restore mechanism

**Decision needed:** Is this critical for v1?

### 4. Multi-Agent Orchestration
**Missing:**
- [ ] Multiple agents per session
- [ ] Agent-to-agent communication
- [ ] Supervisor agent pattern
- [ ] Agent delegation

**Decision needed:** Support multiple agents in v1?

### 5. Cost Management
**Missing:**
- [ ] Token usage billing
- [ ] User quotas/limits
- [ ] Cost tracking dashboard
- [ ] Budget alerts

**Decision needed:** Billing model? Free tier limits?

### 6. Collaboration Features
**Missing:**
- [ ] Share agent sessions
- [ ] Team workspaces
- [ ] Role-based access
- [ ] Session forking

**Decision needed:** Support teams in v1?

### 7. Observability
**Missing:**
- [ ] LangSmith/Langfuse integration
- [ ] Trace viewing UI
- [ ] Performance metrics dashboard
- [ ] Agent quality scoring

**Decision needed:** Which observability tool?

### 8. MCP Server Management
**Missing:**
- [ ] UI to add/remove MCP servers
- [ ] MCP server discovery/marketplace
- [ ] Custom MCP server builder
- [ ] OAuth flow for MCP integrations

**Decision needed:** How do users configure MCP servers?

### 9. Workspace Management
**Missing:**
- [ ] List all workspaces per user
- [ ] Delete/archive workspaces
- [ ] Export workspace contents
- [ ] Workspace templates

**Decision needed:** Workspace lifecycle management?

### 10. Testing Strategy
**Missing:**
- [ ] Unit tests for core functions
- [ ] Integration tests for API
- [ ] E2E tests with Playwright
- [ ] Agent behavior tests

**Decision needed:** Testing coverage requirements?

### 11. Documentation
**Missing:**
- [ ] API documentation (OpenAPI/Swagger)
- [ ] User guide
- [ ] Developer docs
- [ ] Video tutorials

**Decision needed:** Documentation priorities?

### 12. Deployment Infrastructure
**Questions:**
- [ ] Where to host workers? (Railway, Render, EC2)
- [ ] Database hosting? (Supabase, Neon, RDS)
- [ ] Redis hosting? (Upstash, ElastiCache)
- [ ] Docker registry? (Docker Hub, ECR, GCR)

**Decision needed:** Hosting providers for each service?

---

## Quick Start Commands

```bash
# 1. Clone and setup
git clone <your-repo>
cd agent-platform
pnpm install

# 2. Setup environment
cp .env.example .env
# Edit .env with your keys

# 3. Setup database
pnpm prisma generate
pnpm prisma db push

# 4. Start Redis (local)
docker run -d -p 6379:6379 redis:7

# 5. Start development
pnpm dev          # Next.js (localhost:3000)
pnpm worker       # BullMQ worker

# 6. Build Docker image
cd docker
docker build -f Dockerfile.agent-sandbox -t agent-sandbox:latest .

# 7. Push to Daytona (if using Daytona cloud)
daytona image push agent-sandbox:latest
```

---

## Success Metrics

**Week 1-2:**
- [ ] Agent executes simple tasks
- [ ] Events stream to frontend
- [ ] Basic UI shows messages

**Week 3-4:**
- [ ] Code executes in Daytona sandbox
- [ ] Skills activated correctly
- [ ] Browser preview works

**Week 5-6:**
- [ ] Full UI with all tabs functional
- [ ] Production deployment complete
- [ ] 5+ test users onboarded

---

## Next Steps

1. **Review this PRD** and confirm architecture
2. **Make decisions** on missing components
3. **Set up development environment**
4. **Start Phase 1, Task 1.1**

Ready to build? 🚀
