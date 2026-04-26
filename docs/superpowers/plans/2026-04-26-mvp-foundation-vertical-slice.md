# MVP Foundation Vertical Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first full-stack vertical slice: project foundation, shared domain contracts, PostgreSQL/Prisma schema, Fastify/Socket.IO backend, and a basic Next.js UI for room creation, joining, lobby display, game start, answer enqueue, correct answer, skip, and result display.

**Architecture:** Use a TypeScript monorepo with separate `apps/web`, `apps/server`, `packages/shared`, and `packages/game-core` workspaces. The server owns game state in memory, persists durable room/result records through Prisma, and emits per-participant Socket.IO snapshots. The web app sends commands and renders server snapshots without deciding game outcomes locally.

**Tech Stack:** pnpm workspaces, TypeScript, Next.js, React, Fastify, Socket.IO, PostgreSQL, Prisma, Vitest, Docker Compose, GitHub Actions.

---

## Scope Check

The approved architecture spec covers multiple implementation areas. This plan deliberately covers the first vertical slice only, so the project becomes runnable and testable across all layers.

Covered in this plan:

- Monorepo and tooling setup
- Shared validation and public contracts
- Game-core state machine for lobby, start, answer queue, correct answer, incorrect answer, skip, and result
- Prisma schema for rooms, participants, and game results
- Fastify REST endpoints for create, join, resume, and health
- Socket.IO commands for joining a room channel, starting a game, enqueueing an answer, judging an answer, and skipping a turn
- Next.js basic UI for create/join/lobby/game/result
- CI and README foundation

Not covered in this plan:

- Hand-drawn avatar editor
- Accusation, speaker response, and vote flow
- Host transfer
- Offline voter handling
- Production deployment
- Redis adapter
- User account authentication

Those areas should each get a focused follow-up plan after this slice runs end to end.

## File Structure

- `package.json` - root workspace scripts
- `pnpm-workspace.yaml` - workspace package list
- `tsconfig.base.json` - shared TypeScript defaults
- `.editorconfig` - editor formatting defaults
- `.env.example` - documented environment variables
- `docker-compose.yml` - local PostgreSQL
- `.github/workflows/ci.yml` - lint, typecheck, test, Prisma validation
- `packages/shared` - cross-app validation and API/socket contract types
- `packages/game-core` - pure game state machine and tests
- `apps/server` - Fastify, Socket.IO, Prisma, in-memory room runtime
- `apps/web` - Next.js UI and Socket.IO client
- `README.md` - setup, scripts, architecture notes

---

### Task 1: Workspace Foundation

**Files:**
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `tsconfig.base.json`
- Create: `.editorconfig`
- Create: `.env.example`
- Create: `docker-compose.yml`
- Create: `.gitignore` additions if missing

- [ ] **Step 1: Create root workspace files**

Write `package.json`:

```json
{
  "name": "katakanon",
  "private": true,
  "packageManager": "pnpm@10.8.0",
  "scripts": {
    "dev": "pnpm --parallel --filter @katakanon/server --filter @katakanon/web dev",
    "dev:server": "pnpm --filter @katakanon/server dev",
    "dev:web": "pnpm --filter @katakanon/web dev",
    "lint": "pnpm -r lint",
    "typecheck": "pnpm -r typecheck",
    "test": "pnpm -r test",
    "prisma:validate": "pnpm --filter @katakanon/server prisma:validate",
    "prisma:generate": "pnpm --filter @katakanon/server prisma:generate",
    "prisma:migrate": "pnpm --filter @katakanon/server prisma:migrate"
  },
  "devDependencies": {
    "typescript": "^5.8.3"
  }
}
```

Write `pnpm-workspace.yaml`:

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

Write `tsconfig.base.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

Write `.editorconfig`:

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 2
trim_trailing_whitespace = true
```

Write `.env.example`:

```dotenv
DATABASE_URL="postgresql://katakanon:katakanon@localhost:5432/katakanon?schema=public"
SERVER_PORT=4000
WEB_ORIGIN="http://localhost:3000"
NEXT_PUBLIC_SERVER_URL="http://localhost:4000"
```

Write `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: katakanon
      POSTGRES_PASSWORD: katakanon
      POSTGRES_DB: katakanon
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U katakanon -d katakanon"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  postgres-data:
```

- [ ] **Step 2: Make sure `.gitignore` covers generated files**

Ensure `.gitignore` contains these entries:

```gitignore
node_modules/
.next/
dist/
coverage/
.env
.env.local
*.tsbuildinfo
```

- [ ] **Step 3: Install dependencies**

Run:

```powershell
corepack enable
pnpm install
```

Expected: `pnpm-lock.yaml` is created and the install exits with code 0.

- [ ] **Step 4: Verify empty workspace scripts fail only because child packages do not exist yet**

Run:

```powershell
pnpm typecheck
```

Expected: pnpm reports no matching workspace packages or no package scripts. This is acceptable before Task 2 creates packages.

- [ ] **Step 5: Commit workspace foundation**

```powershell
git add package.json pnpm-workspace.yaml tsconfig.base.json .editorconfig .env.example docker-compose.yml .gitignore pnpm-lock.yaml
git commit -m "chore: add workspace foundation"
```

---

### Task 2: Shared Contracts And Display Name Validation

**Files:**
- Create: `packages/shared/package.json`
- Create: `packages/shared/tsconfig.json`
- Create: `packages/shared/src/displayName.test.ts`
- Create: `packages/shared/src/displayName.ts`
- Create: `packages/shared/src/contracts.ts`
- Create: `packages/shared/src/index.ts`

- [ ] **Step 1: Create package metadata**

Write `packages/shared/package.json`:

```json
{
  "name": "@katakanon/shared",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "scripts": {
    "lint": "tsc --noEmit",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "devDependencies": {
    "vitest": "^3.1.2"
  }
}
```

Write `packages/shared/tsconfig.json`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "noEmit": true
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **Step 2: Write failing display name tests**

Write `packages/shared/src/displayName.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { validateDisplayName } from "./displayName";

describe("validateDisplayName", () => {
  it("trims leading and trailing spaces", () => {
    expect(validateDisplayName("  たろう  ")).toEqual({ ok: true, value: "たろう" });
  });

  it("rejects blank names after trimming", () => {
    expect(validateDisplayName("　 ㅤ ")).toEqual({
      ok: false,
      error: "display_name_required"
    });
  });

  it("rejects names longer than 32 code points", () => {
    const value = "あ".repeat(33);
    expect(validateDisplayName(value)).toEqual({
      ok: false,
      error: "display_name_too_long"
    });
  });

  it("rejects control characters", () => {
    expect(validateDisplayName("a\nb")).toEqual({
      ok: false,
      error: "display_name_control_character"
    });
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run:

```powershell
pnpm --filter @katakanon/shared test
```

Expected: FAIL because `./displayName` does not exist.

- [ ] **Step 4: Add display name implementation and contracts**

Write `packages/shared/src/displayName.ts`:

```ts
export type DisplayNameError =
  | "display_name_required"
  | "display_name_too_long"
  | "display_name_control_character";

export type DisplayNameResult =
  | { ok: true; value: string }
  | { ok: false; error: DisplayNameError };

const MAX_DISPLAY_NAME_CODE_POINTS = 32;
const CONTROL_CHARACTER_PATTERN = /[\u0000-\u001f\u007f]/u;

export function validateDisplayName(input: string): DisplayNameResult {
  const value = input.trim();

  if ([...value].length === 0) {
    return { ok: false, error: "display_name_required" };
  }

  if ([...value].length > MAX_DISPLAY_NAME_CODE_POINTS) {
    return { ok: false, error: "display_name_too_long" };
  }

  if (CONTROL_CHARACTER_PATTERN.test(value)) {
    return { ok: false, error: "display_name_control_character" };
  }

  return { ok: true, value };
}
```

Write `packages/shared/src/contracts.ts`:

```ts
export type ParticipantId = string;
export type RoomCode = string;

export type RoomPhase = "lobby" | "turn" | "finished";

export type PublicParticipant = {
  id: ParticipantId;
  displayName: string;
  score: number;
  isHost: boolean;
  isOnline: boolean;
};

export type RoomSnapshot = {
  roomCode: RoomCode;
  phase: RoomPhase;
  me: {
    participantId: ParticipantId;
    isHost: boolean;
    isSpeaker: boolean;
  };
  participants: PublicParticipant[];
  roundIndex: number | null;
  turnIndex: number | null;
  speakerId: ParticipantId | null;
  answerQueue: ParticipantId[];
  word: string | null;
  ranking: PublicParticipant[];
};

export type CreateRoomRequest = {
  displayName: string;
  rounds: number;
};

export type JoinRoomRequest = {
  displayName: string;
};

export type ParticipantCredentials = {
  participantId: ParticipantId;
  participantToken: string;
};
```

Write `packages/shared/src/index.ts`:

```ts
export * from "./contracts";
export * from "./displayName";
```

- [ ] **Step 5: Run shared tests**

Run:

```powershell
pnpm --filter @katakanon/shared test
pnpm --filter @katakanon/shared typecheck
```

Expected: both commands PASS.

- [ ] **Step 6: Commit shared contracts**

```powershell
git add packages/shared
git commit -m "feat: add shared contracts"
```

---

### Task 3: Game Core State Machine

**Files:**
- Create: `packages/game-core/package.json`
- Create: `packages/game-core/tsconfig.json`
- Create: `packages/game-core/src/game.test.ts`
- Create: `packages/game-core/src/game.ts`
- Create: `packages/game-core/src/index.ts`

- [ ] **Step 1: Create game-core package metadata**

Write `packages/game-core/package.json`:

```json
{
  "name": "@katakanon/game-core",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "scripts": {
    "lint": "tsc --noEmit",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "@katakanon/shared": "workspace:*"
  },
  "devDependencies": {
    "vitest": "^3.1.2"
  }
}
```

Write `packages/game-core/tsconfig.json`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "noEmit": true
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **Step 2: Write failing game flow tests**

Write `packages/game-core/src/game.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import {
  createRoom,
  enqueueAnswer,
  judgeAnswer,
  joinRoom,
  makeSnapshot,
  skipTurn,
  startGame
} from "./game";

const words = [
  { id: "w1", surface: "ラーメン" },
  { id: "w2", surface: "テレビ" },
  { id: "w3", surface: "コンビニ" }
];

function roomWithThreePlayers() {
  const room = createRoom({
    roomId: "room-1",
    roomCode: "ABCD12",
    hostParticipantId: "p1",
    hostDisplayName: "ホスト",
    rounds: 1,
    now: "2026-04-26T00:00:00.000Z"
  });

  const joined2 = joinRoom(room, {
    participantId: "p2",
    displayName: "参加者2",
    now: "2026-04-26T00:00:01.000Z"
  });

  return joinRoom(joined2, {
    participantId: "p3",
    displayName: "参加者3",
    now: "2026-04-26T00:00:02.000Z"
  });
}

describe("game core", () => {
  it("starts a game with three online players and hides the word from non-speakers", () => {
    const room = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"]
    });

    expect(room.phase).toBe("turn");
    expect(room.currentTurn?.speakerId).toBe("p1");
    expect(makeSnapshot(room, "p1").word).toBe("ラーメン");
    expect(makeSnapshot(room, "p2").word).toBeNull();
  });

  it("rejects game start with fewer than three online players", () => {
    const room = createRoom({
      roomId: "room-1",
      roomCode: "ABCD12",
      hostParticipantId: "p1",
      hostDisplayName: "ホスト",
      rounds: 1,
      now: "2026-04-26T00:00:00.000Z"
    });

    expect(() => startGame(room, { words, orderedSpeakerIds: ["p1"] })).toThrow(
      "minimum_three_online_players_required"
    );
  });

  it("queues each answerer only once", () => {
    const room = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"]
    });

    const queued = enqueueAnswer(room, "p2");
    expect(enqueueAnswer(queued, "p2").currentTurn?.answerQueue).toEqual(["p2"]);
  });

  it("adds one point to answerer and speaker on correct answer", () => {
    const started = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"]
    });
    const queued = enqueueAnswer(started, "p2");
    const next = judgeAnswer(queued, {
      speakerId: "p1",
      answererId: "p2",
      result: "correct"
    });

    expect(next.participants.find((p) => p.id === "p1")?.score).toBe(1);
    expect(next.participants.find((p) => p.id === "p2")?.score).toBe(1);
    expect(next.currentTurn?.speakerId).toBe("p2");
    expect(next.currentTurn?.answerQueue).toEqual([]);
  });

  it("removes only the first answerer on incorrect answer", () => {
    const started = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"]
    });
    const queued = enqueueAnswer(enqueueAnswer(started, "p2"), "p3");
    const next = judgeAnswer(queued, {
      speakerId: "p1",
      answererId: "p2",
      result: "incorrect"
    });

    expect(next.currentTurn?.speakerId).toBe("p1");
    expect(next.currentTurn?.answerQueue).toEqual(["p3"]);
  });

  it("finishes after the final turn is skipped", () => {
    let room = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"]
    });

    room = skipTurn(room, "p1");
    room = skipTurn(room, "p2");
    room = skipTurn(room, "p3");

    expect(room.phase).toBe("finished");
    expect(room.currentTurn).toBeUndefined();
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run:

```powershell
pnpm --filter @katakanon/game-core test
```

Expected: FAIL because `./game` does not exist.

- [ ] **Step 4: Implement the game core**

Write `packages/game-core/src/game.ts`:

```ts
import type { ParticipantId, RoomCode, RoomSnapshot } from "@katakanon/shared";

export type Word = {
  id: string;
  surface: string;
};

export type Participant = {
  id: ParticipantId;
  displayName: string;
  score: number;
  isHost: boolean;
  isOnline: boolean;
  joinedAt: string;
};

export type GameRoom = {
  id: string;
  code: RoomCode;
  phase: "lobby" | "turn" | "finished";
  rounds: number;
  participants: Participant[];
  speakerOrder: ParticipantId[];
  wordDeck: Word[];
  usedWordIds: string[];
  currentTurn?: {
    roundIndex: number;
    turnIndex: number;
    speakerId: ParticipantId;
    word: Word;
    answerQueue: ParticipantId[];
  };
};

export function createRoom(input: {
  roomId: string;
  roomCode: RoomCode;
  hostParticipantId: ParticipantId;
  hostDisplayName: string;
  rounds: number;
  now: string;
}): GameRoom {
  return {
    id: input.roomId,
    code: input.roomCode,
    phase: "lobby",
    rounds: input.rounds,
    participants: [
      {
        id: input.hostParticipantId,
        displayName: input.hostDisplayName,
        score: 0,
        isHost: true,
        isOnline: true,
        joinedAt: input.now
      }
    ],
    speakerOrder: [],
    wordDeck: [],
    usedWordIds: []
  };
}

export function joinRoom(
  room: GameRoom,
  input: { participantId: ParticipantId; displayName: string; now: string }
): GameRoom {
  if (room.phase !== "lobby") {
    throw new Error("room_not_joinable");
  }

  return {
    ...room,
    participants: [
      ...room.participants,
      {
        id: input.participantId,
        displayName: input.displayName,
        score: 0,
        isHost: false,
        isOnline: true,
        joinedAt: input.now
      }
    ]
  };
}

export function startGame(
  room: GameRoom,
  input: { words: Word[]; orderedSpeakerIds: ParticipantId[] }
): GameRoom {
  const onlineParticipants = room.participants.filter((participant) => participant.isOnline);
  const requiredWordCount = onlineParticipants.length * room.rounds;

  if (onlineParticipants.length < 3) {
    throw new Error("minimum_three_online_players_required");
  }

  if (input.words.length < requiredWordCount) {
    throw new Error("not_enough_words");
  }

  const speakerOrder = input.orderedSpeakerIds;
  return beginTurn({
    ...room,
    phase: "turn",
    speakerOrder,
    wordDeck: input.words,
    usedWordIds: []
  }, 0, 0);
}

export function enqueueAnswer(room: GameRoom, participantId: ParticipantId): GameRoom {
  const turn = requireTurn(room);

  if (participantId === turn.speakerId) {
    throw new Error("speaker_cannot_answer");
  }

  if (turn.answerQueue.includes(participantId)) {
    return room;
  }

  return {
    ...room,
    currentTurn: {
      ...turn,
      answerQueue: [...turn.answerQueue, participantId]
    }
  };
}

export function judgeAnswer(
  room: GameRoom,
  input: { speakerId: ParticipantId; answererId: ParticipantId; result: "correct" | "incorrect" }
): GameRoom {
  const turn = requireTurn(room);

  if (turn.speakerId !== input.speakerId) {
    throw new Error("speaker_only_operation");
  }

  if (turn.answerQueue[0] !== input.answererId) {
    throw new Error("answerer_is_not_queue_head");
  }

  if (input.result === "incorrect") {
    return {
      ...room,
      currentTurn: {
        ...turn,
        answerQueue: turn.answerQueue.slice(1)
      }
    };
  }

  const participants = room.participants.map((participant) => {
    if (participant.id === input.speakerId || participant.id === input.answererId) {
      return { ...participant, score: participant.score + 1 };
    }
    return participant;
  });

  return advanceTurn({ ...room, participants });
}

export function skipTurn(room: GameRoom, speakerId: ParticipantId): GameRoom {
  const turn = requireTurn(room);

  if (turn.speakerId !== speakerId) {
    throw new Error("speaker_only_operation");
  }

  return advanceTurn(room);
}

export function makeSnapshot(room: GameRoom, viewerId: ParticipantId): RoomSnapshot {
  const turn = room.currentTurn;
  const speakerId = turn?.speakerId ?? null;
  const viewer = room.participants.find((participant) => participant.id === viewerId);

  if (!viewer) {
    throw new Error("viewer_not_found");
  }

  const participants = room.participants.map((participant) => ({
    id: participant.id,
    displayName: participant.displayName,
    score: participant.score,
    isHost: participant.isHost,
    isOnline: participant.isOnline
  }));

  return {
    roomCode: room.code,
    phase: room.phase,
    me: {
      participantId: viewerId,
      isHost: viewer.isHost,
      isSpeaker: viewerId === speakerId
    },
    participants,
    roundIndex: turn?.roundIndex ?? null,
    turnIndex: turn?.turnIndex ?? null,
    speakerId,
    answerQueue: turn?.answerQueue ?? [],
    word: viewerId === speakerId ? turn?.word.surface ?? null : null,
    ranking: [...participants].sort((a, b) => b.score - a.score)
  };
}

function advanceTurn(room: GameRoom): GameRoom {
  const turn = requireTurn(room);
  const nextTurnIndex = turn.turnIndex + 1;

  if (nextTurnIndex < room.speakerOrder.length) {
    return beginTurn(room, turn.roundIndex, nextTurnIndex);
  }

  const nextRoundIndex = turn.roundIndex + 1;
  if (nextRoundIndex < room.rounds) {
    return beginTurn(room, nextRoundIndex, 0);
  }

  return {
    ...room,
    phase: "finished",
    currentTurn: undefined
  };
}

function beginTurn(
  room: GameRoom,
  roundIndex: number,
  turnIndex: number
): GameRoom {
  const speakerId = room.speakerOrder[turnIndex];
  if (!speakerId) {
    throw new Error("speaker_not_found");
  }

  const nextWord = room.wordDeck.find((word) => !room.usedWordIds.includes(word.id));
  if (!nextWord) {
    throw new Error("not_enough_words");
  }

  return {
    ...room,
    currentTurn: {
      roundIndex,
      turnIndex,
      speakerId,
      word: nextWord,
      answerQueue: []
    },
    usedWordIds: [...room.usedWordIds, nextWord.id]
  };
}

function requireTurn(room: GameRoom): NonNullable<GameRoom["currentTurn"]> {
  if (room.phase !== "turn" || !room.currentTurn) {
    throw new Error("room_is_not_in_turn");
  }

  return room.currentTurn;
}
```

Write `packages/game-core/src/index.ts`:

```ts
export * from "./game";
```

- [ ] **Step 5: Run game-core tests**

Run:

```powershell
pnpm --filter @katakanon/game-core test
```

Expected: PASS.

- [ ] **Step 6: Run game-core typecheck**

Run:

```powershell
pnpm --filter @katakanon/game-core typecheck
```

Expected: PASS.

- [ ] **Step 7: Commit game core**

```powershell
git add packages/game-core
git commit -m "feat: add game core vertical slice"
```

---

### Task 4: Server Package, Prisma Schema, And Persistence Boundary

**Files:**
- Create: `apps/server/package.json`
- Create: `apps/server/tsconfig.json`
- Create: `apps/server/prisma/schema.prisma`
- Create: `apps/server/src/config.ts`
- Create: `apps/server/src/db.ts`
- Create: `apps/server/src/words.ts`

- [ ] **Step 1: Create server package metadata**

Write `apps/server/package.json`:

```json
{
  "name": "@katakanon/server",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/main.ts",
    "lint": "tsc --noEmit",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "prisma:validate": "prisma validate",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev"
  },
  "dependencies": {
    "@fastify/cors": "^11.0.1",
    "@katakanon/game-core": "workspace:*",
    "@katakanon/shared": "workspace:*",
    "@prisma/client": "^6.6.0",
    "fastify": "^5.3.2",
    "socket.io": "^4.8.1"
  },
  "devDependencies": {
    "@types/node": "^22.15.2",
    "prisma": "^6.6.0",
    "tsx": "^4.19.3",
    "vitest": "^3.1.2"
  }
}
```

Write `apps/server/tsconfig.json`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "noEmit": true,
    "types": ["node"]
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **Step 2: Add Prisma schema**

Write `apps/server/prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Room {
  id                String        @id
  code              String        @unique
  status            RoomStatus
  rounds            Int
  hostParticipantId String
  createdAt         DateTime      @default(now())
  startedAt         DateTime?
  finishedAt        DateTime?
  participants      Participant[]
  result            GameResult?
}

model Participant {
  id                   String   @id
  roomId               String
  displayName          String
  participantTokenHash String
  score                Int      @default(0)
  isHost               Boolean  @default(false)
  joinedAt             DateTime @default(now())
  room                 Room     @relation(fields: [roomId], references: [id], onDelete: Cascade)

  @@index([roomId])
}

model GameResult {
  id         String   @id
  roomId     String   @unique
  scoresJson Json
  createdAt  DateTime @default(now())
  room       Room     @relation(fields: [roomId], references: [id], onDelete: Cascade)
}

enum RoomStatus {
  LOBBY
  IN_GAME
  FINISHED
}
```

- [ ] **Step 3: Add server config and fixed word list**

Write `apps/server/src/config.ts`:

```ts
export type ServerConfig = {
  port: number;
  webOrigin: string;
  databaseUrl: string;
};

export function readConfig(env: NodeJS.ProcessEnv): ServerConfig {
  return {
    port: Number(env.SERVER_PORT ?? 4000),
    webOrigin: env.WEB_ORIGIN ?? "http://localhost:3000",
    databaseUrl:
      env.DATABASE_URL ??
      "postgresql://katakanon:katakanon@localhost:5432/katakanon?schema=public"
  };
}
```

Write `apps/server/src/db.ts`:

```ts
import { PrismaClient } from "@prisma/client";

export const prisma = new PrismaClient();
```

Write `apps/server/src/words.ts`:

```ts
import type { Word } from "@katakanon/game-core";

export const fixedWords: Word[] = [
  { id: "w1", surface: "ラーメン" },
  { id: "w2", surface: "テレビ" },
  { id: "w3", surface: "コンビニ" },
  { id: "w4", surface: "スマホ" },
  { id: "w5", surface: "カメラ" },
  { id: "w6", surface: "アイス" },
  { id: "w7", surface: "コーヒー" },
  { id: "w8", surface: "ホテル" },
  { id: "w9", surface: "タクシー" },
  { id: "w10", surface: "パソコン" }
];
```

- [ ] **Step 4: Validate Prisma schema**

Run:

```powershell
pnpm --filter @katakanon/server prisma:validate
pnpm --filter @katakanon/server prisma:generate
```

Expected: both commands PASS.

- [ ] **Step 5: Commit server schema foundation**

```powershell
git add apps/server
git commit -m "feat: add server persistence schema"
```

---

### Task 5: Fastify REST API And Runtime Room Store

**Files:**
- Create: `apps/server/src/runtimeStore.ts`
- Create: `apps/server/src/token.ts`
- Create: `apps/server/src/http.test.ts`
- Create: `apps/server/src/http.ts`

- [ ] **Step 1: Write failing HTTP tests**

Write `apps/server/src/http.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { buildHttpServer } from "./http";
import { createRuntimeStore } from "./runtimeStore";

describe("HTTP API", () => {
  it("creates a room and returns participant credentials", async () => {
    const app = buildHttpServer({ store: createRuntimeStore() });
    const response = await app.inject({
      method: "POST",
      url: "/rooms",
      payload: { displayName: "ホスト", rounds: 1 }
    });

    expect(response.statusCode).toBe(201);
    expect(response.json()).toMatchObject({
      roomCode: expect.any(String),
      participantId: expect.any(String),
      participantToken: expect.any(String)
    });
  });

  it("rejects invalid display names", async () => {
    const app = buildHttpServer({ store: createRuntimeStore() });
    const response = await app.inject({
      method: "POST",
      url: "/rooms",
      payload: { displayName: "", rounds: 1 }
    });

    expect(response.statusCode).toBe(400);
    expect(response.json()).toEqual({ error: "display_name_required" });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```powershell
pnpm --filter @katakanon/server test
```

Expected: FAIL because `./http` and `./runtimeStore` do not exist.

- [ ] **Step 3: Implement runtime store and token helpers**

Write `apps/server/src/runtimeStore.ts`:

```ts
import type { GameRoom } from "@katakanon/game-core";

export type RuntimeStore = {
  roomsByCode: Map<string, GameRoom>;
  participantTokens: Map<string, { participantId: string; roomCode: string }>;
};

export function createRuntimeStore(): RuntimeStore {
  return {
    roomsByCode: new Map(),
    participantTokens: new Map()
  };
}
```

Write `apps/server/src/token.ts`:

```ts
import { createHash, randomUUID } from "node:crypto";

export function createParticipantToken(): string {
  return randomUUID();
}

export function hashParticipantToken(token: string): string {
  return createHash("sha256").update(token).digest("hex");
}
```

- [ ] **Step 4: Implement Fastify routes**

Write `apps/server/src/http.ts`:

```ts
import { randomUUID } from "node:crypto";
import Fastify from "fastify";
import { createRoom, joinRoom } from "@katakanon/game-core";
import { validateDisplayName } from "@katakanon/shared";
import { createParticipantToken } from "./token";
import type { RuntimeStore } from "./runtimeStore";

export function buildHttpServer(input: { store: RuntimeStore }) {
  const app = Fastify({ logger: false });

  app.get("/health", async () => ({ ok: true }));

  app.post<{
    Body: { displayName: string; rounds: number };
  }>("/rooms", async (request, reply) => {
    const validation = validateDisplayName(request.body.displayName);
    if (!validation.ok) {
      return reply.status(400).send({ error: validation.error });
    }

    const roomId = randomUUID();
    const participantId = randomUUID();
    const roomCode = randomRoomCode();
    const token = createParticipantToken();
    const now = new Date().toISOString();
    const room = createRoom({
      roomId,
      roomCode,
      hostParticipantId: participantId,
      hostDisplayName: validation.value,
      rounds: request.body.rounds,
      now
    });

    input.store.roomsByCode.set(roomCode, room);
    input.store.participantTokens.set(token, { participantId, roomCode });

    return reply.status(201).send({
      roomCode,
      participantId,
      participantToken: token
    });
  });

  app.post<{
    Params: { code: string };
    Body: { displayName: string };
  }>("/rooms/:code/join", async (request, reply) => {
    const room = input.store.roomsByCode.get(request.params.code);
    if (!room) {
      return reply.status(404).send({ error: "room_not_found" });
    }

    const validation = validateDisplayName(request.body.displayName);
    if (!validation.ok) {
      return reply.status(400).send({ error: validation.error });
    }

    const participantId = randomUUID();
    const token = createParticipantToken();
    const nextRoom = joinRoom(room, {
      participantId,
      displayName: validation.value,
      now: new Date().toISOString()
    });

    input.store.roomsByCode.set(room.code, nextRoom);
    input.store.participantTokens.set(token, { participantId, roomCode: room.code });

    return reply.status(201).send({
      roomCode: room.code,
      participantId,
      participantToken: token
    });
  });

  app.post<{
    Params: { code: string };
    Body: { participantToken: string };
  }>("/rooms/:code/resume", async (request, reply) => {
    const session = input.store.participantTokens.get(request.body.participantToken);
    if (!session || session.roomCode !== request.params.code) {
      return reply.status(401).send({ error: "invalid_participant_token" });
    }

    return reply.send(session);
  });

  return app;
}

function randomRoomCode(): string {
  return Math.random().toString(36).slice(2, 8).toUpperCase();
}
```

- [ ] **Step 5: Run HTTP tests**

Run:

```powershell
pnpm --filter @katakanon/server test
pnpm --filter @katakanon/server typecheck
```

Expected: both commands PASS.

- [ ] **Step 6: Commit HTTP API**

```powershell
git add apps/server/src
git commit -m "feat: add room HTTP API"
```

---

### Task 6: Socket.IO Vertical Slice

**Files:**
- Create: `apps/server/src/socket.ts`
- Create: `apps/server/src/main.ts`

- [ ] **Step 1: Implement Socket.IO commands**

Write `apps/server/src/socket.ts`:

```ts
import type { Server as HttpServer } from "node:http";
import type { GameRoom } from "@katakanon/game-core";
import {
  enqueueAnswer,
  judgeAnswer,
  makeSnapshot,
  skipTurn,
  startGame
} from "@katakanon/game-core";
import { Server, type Socket } from "socket.io";
import type { RuntimeStore } from "./runtimeStore";
import { fixedWords } from "./words";

export function attachSocketServer(input: {
  httpServer: HttpServer;
  store: RuntimeStore;
  webOrigin: string;
}) {
  const io = new Server(input.httpServer, {
    cors: {
      origin: input.webOrigin
    }
  });

  io.on("connection", (socket) => {
    socket.on("room:join", ({ roomCode, participantToken }) => {
      const session = input.store.participantTokens.get(participantToken);
      const room = input.store.roomsByCode.get(roomCode);

      if (!session || !room || session.roomCode !== roomCode) {
        socket.emit("command:error", { error: "invalid_room_session" });
        return;
      }

      socket.data.roomCode = roomCode;
      socket.data.participantId = session.participantId;
      socket.join(roomCode);
      socket.join(participantRoom(roomCode, session.participantId));
      emitRoomState(io, input.store, roomCode);
    });

    socket.on("game:start", () => {
      const roomCode = socket.data.roomCode;
      const participantId = socket.data.participantId;
      const room = input.store.roomsByCode.get(roomCode);
      const actor = room?.participants.find((participant) => participant.id === participantId);

      if (!room || !actor?.isHost) {
        socket.emit("command:error", { error: "host_only_operation" });
        return;
      }

      const nextRoom = startGame(room, {
        words: fixedWords,
        orderedSpeakerIds: room.participants.map((participant) => participant.id)
      });

      input.store.roomsByCode.set(roomCode, nextRoom);
      emitRoomState(io, input.store, roomCode);
    });

    socket.on("answer:enqueue", () => {
      updateRoom(socket, input.store, (room, participantId) => enqueueAnswer(room, participantId));
      emitRoomState(io, input.store, socket.data.roomCode);
    });

    socket.on("answer:judge", ({ answererId, result }) => {
      updateRoom(socket, input.store, (room, participantId) =>
        judgeAnswer(room, { speakerId: participantId, answererId, result })
      );
      emitRoomState(io, input.store, socket.data.roomCode);
    });

    socket.on("answer:skip", () => {
      updateRoom(socket, input.store, (room, participantId) => skipTurn(room, participantId));
      emitRoomState(io, input.store, socket.data.roomCode);
    });
  });

  return io;
}

function updateRoom(
  socket: Socket,
  store: RuntimeStore,
  updater: (room: GameRoom, participantId: string) => GameRoom
) {
  const roomCode = socket.data.roomCode;
  const participantId = socket.data.participantId;
  const room = store.roomsByCode.get(roomCode);

  if (!room || !participantId) {
    socket.emit("command:error", { error: "invalid_room_session" });
    return;
  }

  try {
    store.roomsByCode.set(roomCode, updater(room, participantId));
  } catch (error) {
    socket.emit("command:error", { error: error instanceof Error ? error.message : "unknown_error" });
  }
}

function emitRoomState(io: Server, store: RuntimeStore, roomCode: string) {
  const room = store.roomsByCode.get(roomCode);
  if (!room) {
    return;
  }

  for (const participant of room.participants) {
    io.to(participantRoom(roomCode, participant.id)).emit("state:full", makeSnapshot(room, participant.id));
  }
}

function participantRoom(roomCode: string, participantId: string): string {
  return `${roomCode}:participant:${participantId}`;
}
```

Write `apps/server/src/main.ts`:

```ts
import cors from "@fastify/cors";
import { buildHttpServer } from "./http";
import { readConfig } from "./config";
import { createRuntimeStore } from "./runtimeStore";
import { attachSocketServer } from "./socket";

const config = readConfig(process.env);
const store = createRuntimeStore();
const app = buildHttpServer({ store });

await app.register(cors, {
  origin: config.webOrigin
});

attachSocketServer({
  httpServer: app.server,
  store,
  webOrigin: config.webOrigin
});

await app.listen({ port: config.port, host: "0.0.0.0" });
```

- [ ] **Step 2: Typecheck server**

Run:

```powershell
pnpm --filter @katakanon/server typecheck
```

Expected: PASS.

- [ ] **Step 3: Commit Socket.IO server slice**

```powershell
git add apps/server/src/socket.ts apps/server/src/main.ts
git commit -m "feat: add socket vertical slice"
```

---

### Task 7: Next.js Web Vertical Slice

**Files:**
- Create: `apps/web/package.json`
- Create: `apps/web/tsconfig.json`
- Create: `apps/web/next.config.mjs`
- Create: `apps/web/src/app/layout.tsx`
- Create: `apps/web/src/app/page.tsx`
- Create: `apps/web/src/app/rooms/[code]/page.tsx`
- Create: `apps/web/src/app/globals.css`

- [ ] **Step 1: Create web package metadata**

Write `apps/web/package.json`:

```json
{
  "name": "@katakanon/web",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "next dev",
    "lint": "tsc --noEmit",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "@katakanon/shared": "workspace:*",
    "next": "^15.3.1",
    "react": "^19.1.0",
    "react-dom": "^19.1.0",
    "socket.io-client": "^4.8.1"
  },
  "devDependencies": {
    "@types/node": "^22.15.2",
    "@types/react": "^19.1.2",
    "@types/react-dom": "^19.1.2",
    "vitest": "^3.1.2"
  }
}
```

Write `apps/web/tsconfig.json`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "jsx": "preserve",
    "allowJs": true,
    "noEmit": true,
    "incremental": true,
    "plugins": [{ "name": "next" }]
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

Write `apps/web/next.config.mjs`:

```js
/** @type {import("next").NextConfig} */
const nextConfig = {};

export default nextConfig;
```

- [ ] **Step 2: Add app shell and basic styling**

Write `apps/web/src/app/layout.tsx`:

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "カタカナ禁止ゲーム",
  description: "リアルタイムで遊ぶカタカナ禁止ゲーム"
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>{children}</body>
    </html>
  );
}
```

Write `apps/web/src/app/globals.css`:

```css
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: system-ui, sans-serif;
  background: #f8fafc;
  color: #111827;
}

button,
input,
select {
  font: inherit;
}

.shell {
  width: min(960px, calc(100% - 32px));
  margin: 0 auto;
  padding: 32px 0;
}

.panel {
  border: 1px solid #d1d5db;
  border-radius: 8px;
  background: #ffffff;
  padding: 20px;
}

.stack {
  display: grid;
  gap: 16px;
}

.row {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
}

.primary {
  border: 0;
  border-radius: 6px;
  background: #2563eb;
  color: #ffffff;
  padding: 10px 14px;
  cursor: pointer;
}

.secondary {
  border: 1px solid #9ca3af;
  border-radius: 6px;
  background: #ffffff;
  color: #111827;
  padding: 10px 14px;
  cursor: pointer;
}
```

- [ ] **Step 3: Add create/join page**

Write `apps/web/src/app/page.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";

const serverUrl = process.env.NEXT_PUBLIC_SERVER_URL ?? "http://localhost:4000";

export default function HomePage() {
  const router = useRouter();
  const [displayName, setDisplayName] = useState("");
  const [roomCode, setRoomCode] = useState("");
  const [error, setError] = useState<string | null>(null);

  async function createRoom() {
    setError(null);
    const response = await fetch(`${serverUrl}/rooms`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ displayName, rounds: 1 })
    });
    const payload = await response.json();
    if (!response.ok) {
      setError(payload.error);
      return;
    }
    localStorage.setItem(`katakanon:${payload.roomCode}:token`, payload.participantToken);
    router.push(`/rooms/${payload.roomCode}`);
  }

  async function joinRoom() {
    setError(null);
    const code = roomCode.trim().toUpperCase();
    const response = await fetch(`${serverUrl}/rooms/${code}/join`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ displayName })
    });
    const payload = await response.json();
    if (!response.ok) {
      setError(payload.error);
      return;
    }
    localStorage.setItem(`katakanon:${payload.roomCode}:token`, payload.participantToken);
    router.push(`/rooms/${payload.roomCode}`);
  }

  return (
    <main className="shell stack">
      <h1>カタカナ禁止ゲーム</h1>
      <section className="panel stack">
        <label className="stack">
          プレイヤーネーム
          <input value={displayName} onChange={(event) => setDisplayName(event.target.value)} />
        </label>
        <div className="row">
          <button className="primary" onClick={createRoom}>ルーム作成</button>
        </div>
      </section>
      <section className="panel stack">
        <label className="stack">
          ルームコード
          <input value={roomCode} onChange={(event) => setRoomCode(event.target.value)} />
        </label>
        <button className="secondary" onClick={joinRoom}>参加</button>
      </section>
      {error ? <p role="alert">{error}</p> : null}
    </main>
  );
}
```

- [ ] **Step 4: Add room page**

Write `apps/web/src/app/rooms/[code]/page.tsx`:

```tsx
"use client";

import type { RoomSnapshot } from "@katakanon/shared";
import { useParams } from "next/navigation";
import { useEffect, useMemo, useState } from "react";
import { io } from "socket.io-client";

const serverUrl = process.env.NEXT_PUBLIC_SERVER_URL ?? "http://localhost:4000";

export default function RoomPage() {
  const params = useParams<{ code: string }>();
  const roomCode = params.code.toUpperCase();
  const [snapshot, setSnapshot] = useState<RoomSnapshot | null>(null);
  const [error, setError] = useState<string | null>(null);

  const socket = useMemo(() => io(serverUrl, { autoConnect: false }), []);

  useEffect(() => {
    const token = localStorage.getItem(`katakanon:${roomCode}:token`);
    if (!token) {
      setError("participant_token_missing");
      return;
    }

    socket.connect();
    socket.emit("room:join", { roomCode, participantToken: token });
    socket.on("command:error", (payload: { error: string }) => setError(payload.error));
    socket.on("state:full", setSnapshot);

    return () => {
      socket.off("state:full", setSnapshot);
      socket.disconnect();
    };
  }, [roomCode, socket]);

  return (
    <main className="shell stack">
      <h1>ルーム {roomCode}</h1>
      {error ? <p role="alert">{error}</p> : null}
      {snapshot ? (
        <section className="panel stack">
          <p>フェーズ: {snapshot.phase}</p>
          <p>説明者: {snapshot.speakerId ?? "未開始"}</p>
          {snapshot.word ? <p>お題: {snapshot.word}</p> : null}
          <div className="row">
            {snapshot.me.isHost && snapshot.phase === "lobby" ? (
              <button className="primary" onClick={() => socket.emit("game:start")}>開始</button>
            ) : null}
            {snapshot.phase === "turn" && !snapshot.me.isSpeaker ? (
              <button className="secondary" onClick={() => socket.emit("answer:enqueue")}>解答</button>
            ) : null}
            {snapshot.phase === "turn" && snapshot.me.isSpeaker ? (
              <>
                <button
                  className="primary"
                  disabled={snapshot.answerQueue.length === 0}
                  onClick={() =>
                    socket.emit("answer:judge", {
                      answererId: snapshot.answerQueue[0],
                      result: "correct"
                    })
                  }
                >
                  正解
                </button>
                <button
                  className="secondary"
                  disabled={snapshot.answerQueue.length === 0}
                  onClick={() =>
                    socket.emit("answer:judge", {
                      answererId: snapshot.answerQueue[0],
                      result: "incorrect"
                    })
                  }
                >
                  不正解
                </button>
                <button className="secondary" onClick={() => socket.emit("answer:skip")}>スキップ</button>
              </>
            ) : null}
          </div>
          <h2>参加者</h2>
          <ul>
            {snapshot.participants.map((participant) => (
              <li key={participant.id}>
                {participant.displayName} / {participant.score} pt
              </li>
            ))}
          </ul>
          <h2>解答キュー</h2>
          <ol>
            {snapshot.answerQueue.map((participantId) => (
              <li key={participantId}>{participantId}</li>
            ))}
          </ol>
        </section>
      ) : (
        <p>接続中...</p>
      )}
    </main>
  );
}
```

- [ ] **Step 5: Run web typecheck**

Run:

```powershell
pnpm --filter @katakanon/web typecheck
```

Expected: PASS.

- [ ] **Step 6: Commit web vertical slice**

```powershell
git add apps/web
git commit -m "feat: add web vertical slice"
```

---

### Task 8: CI And README

**Files:**
- Create: `.github/workflows/ci.yml`
- Create or modify: `README.md`

- [ ] **Step 1: Add CI workflow**

Write `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  checks:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: katakanon
          POSTGRES_PASSWORD: katakanon
          POSTGRES_DB: katakanon
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U katakanon -d katakanon"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      DATABASE_URL: postgresql://katakanon:katakanon@localhost:5432/katakanon?schema=public
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 10.8.0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm prisma:validate
      - run: pnpm prisma:generate
      - run: pnpm typecheck
      - run: pnpm test
```

- [ ] **Step 2: Add README foundation**

Write `README.md`:

```md
# カタカナ禁止ゲーム

Web 系転職ポートフォリオとして作る、リアルタイム参加型のカタカナ禁止ゲームです。

## Tech Stack

- Frontend: Next.js, React, TypeScript
- Backend: Fastify, Socket.IO, TypeScript
- Database: PostgreSQL
- ORM: Prisma
- Test: Vitest
- Local infra: Docker Compose
- CI: GitHub Actions

## Architecture

ゲーム判定と状態遷移はサーバ側の状態機械に集約します。クライアントはコマンドを送信し、サーバから配信されるスナップショットを表示します。

MVP のゲーム中状態はサーバメモリに保持し、ルーム、参加者、結果の永続化は PostgreSQL と Prisma で扱います。

## Setup

```powershell
corepack enable
pnpm install
Copy-Item .env.example .env
docker compose up -d
pnpm prisma:validate
pnpm prisma:generate
pnpm dev
```

## Checks

```powershell
pnpm typecheck
pnpm test
pnpm prisma:validate
pnpm prisma:generate
```

## MVP Vertical Slice

この段階でできること:

- ルーム作成
- ルーム参加
- ロビーの参加者表示
- ホストによるゲーム開始
- 説明者だけへのお題表示
- 解答キュー登録
- 正解、誤答、スキップ
- 最終結果表示の土台

## Design Docs

- `docs/01_requirements.md`
- `docs/superpowers/specs/2026-04-26-portfolio-architecture-design.md`
- `docs/superpowers/plans/2026-04-26-mvp-foundation-vertical-slice.md`
```

- [ ] **Step 3: Run all checks**

Run:

```powershell
pnpm prisma:validate
pnpm prisma:generate
pnpm typecheck
pnpm test
```

Expected: all commands PASS.

- [ ] **Step 4: Commit CI and README**

```powershell
git add .github/workflows/ci.yml README.md
git commit -m "docs: add project README and CI"
```

---

## Self-Review

Spec coverage:

- The plan covers the agreed stack: Next.js, Fastify, Socket.IO, PostgreSQL, Prisma, Docker Compose, and CI.
- The plan covers server-owned game state for the first gameplay slice.
- The plan covers participant tokens for anonymous identity and reconnect-ready sessions.
- The plan covers per-participant snapshots and hides the word from non-speakers.
- The plan covers DB schema foundation, but the first runtime store is memory-only.
- The plan does not cover avatar drawing, accusation/vote, host transfer, and production deployment; those are intentionally separate follow-up plans.

Placeholder scan:

- No task contains unresolved placeholder markers or open-ended implementation text.
- Every code-writing step includes concrete file contents or concrete replacement instructions.
- Each task has explicit commands and expected outcomes.

Type consistency:

- Shared `RoomSnapshot`, `ParticipantId`, and `RoomCode` are consumed by game-core, server, and web.
- Game-core command names match Socket.IO command handlers.
- `participantToken` naming is consistent across HTTP, Socket.IO, web localStorage, and README.
