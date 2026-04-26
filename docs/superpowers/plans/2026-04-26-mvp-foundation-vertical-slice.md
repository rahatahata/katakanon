# MVP 基盤縦スライス実装計画

> **エージェント作業者向け:** 必須サブスキル: この計画をタスク単位で実装する場合は、`superpowers:subagent-driven-development`（推奨）または `superpowers:executing-plans` を使うこと。進捗管理にはチェックボックス（`- [ ]`）を使う。

**目的:** 最初のフルスタック縦スライスとして、プロジェクト基盤、共有ドメイン契約、PostgreSQL/Prisma スキーマ、Fastify/Socket.IO バックエンド、ルーム作成・参加・ロビー表示・ゲーム開始・解答キュー・正解判定・スキップ・結果表示の基本 Next.js UI を作る。

**アーキテクチャ:** `apps/web`、`apps/server`、`packages/shared`、`packages/game-core` に分けた TypeScript モノレポ構成にする。サーバはゲーム状態をメモリ上で所有し、永続化すべきルーム・結果情報を Prisma 経由で保存し、参加者ごとの Socket.IO スナップショットを配信する。Web アプリはコマンド送信とスナップショット描画に集中し、ゲーム結果をローカルでは確定しない。

**技術スタック:** pnpm workspaces、TypeScript、Next.js、React、Fastify、Socket.IO、PostgreSQL、Prisma、Vitest、Docker Compose、GitHub Actions。

---

## スコープ確認

承認済みのアーキテクチャ spec は複数の実装領域を含む。この計画では、全レイヤーをまたいで実行・テストできる状態を最短で作るため、最初の縦スライスだけを対象にする。

この計画で扱う範囲:

- モノレポとツール設定
- 共有バリデーションと公開契約
- ロビー、開始、解答キュー、正解、不正解、スキップ、結果までの game-core 状態機械
- ルーム、参加者、ゲーム結果の Prisma スキーマ
- 作成、参加、再接続、ヘルスチェック用の Fastify REST エンドポイント
- ルームチャンネル参加、ゲーム開始、解答キュー登録、正誤判定、スキップ用の Socket.IO コマンド
- 作成、参加、ロビー、ゲーム、結果の基本 Next.js UI
- CI と README の土台

この計画では扱わない範囲:

- 手描きアバターエディタ
- 指摘、説明者の応答、投票フロー
- ホスト移譲
- 投票者のオフライン処理
- 本番デプロイ
- Redis adapter
- ユーザーアカウント認証

これらは、この縦スライスがエンドツーエンドで動いてから、それぞれ焦点を絞った後続計画として扱う。

## ファイル構成

- `package.json` - ルートのワークスペーススクリプト
- `pnpm-workspace.yaml` - ワークスペースパッケージ一覧
- `tsconfig.base.json` - 共通 TypeScript 設定
- `.editorconfig` - エディタ整形設定
- `.env.example` - 環境変数のサンプル
- `docker-compose.yml` - ローカル PostgreSQL
- `.github/workflows/ci.yml` - lint、typecheck、test、Prisma 検証
- `packages/shared` - アプリ横断のバリデーションと API/socket 契約型
- `packages/game-core` - 純粋なゲーム状態機械とテスト
- `apps/server` - Fastify、Socket.IO、Prisma、メモリ上のルーム実行時状態
- `apps/web` - Next.js UI と Socket.IO クライアント
- `README.md` - セットアップ、スクリプト、アーキテクチャメモ

---

### タスク 1: ワークスペース基盤

**対象ファイル:**

- 作成: `package.json`
- 作成: `pnpm-workspace.yaml`
- 作成: `tsconfig.base.json`
- 作成: `.editorconfig`
- 作成: `.env.example`
- 作成: `docker-compose.yml`
- 作成: `.gitignore` に不足項目がある場合は追記

- [ ] **ステップ 1: ルートのワークスペースファイルを作成する**

`package.json` を作成する:

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

`pnpm-workspace.yaml` を作成する:

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

`tsconfig.base.json` を作成する:

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

`.editorconfig` を作成する:

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

`.env.example` を作成する:

```dotenv
DATABASE_URL="postgresql://katakanon:katakanon@localhost:5432/katakanon?schema=public"
SERVER_PORT=4000
WEB_ORIGIN="http://localhost:3000"
NEXT_PUBLIC_SERVER_URL="http://localhost:4000"
```

`docker-compose.yml` を作成する:

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

- [ ] **ステップ 2: `.gitignore` が生成物を除外することを確認する**

`.gitignore` に次の項目が含まれていることを確認する:

```gitignore
node_modules/
.next/
dist/
coverage/
.env
.env.local
*.tsbuildinfo
```

- [ ] **ステップ 3: 依存関係をインストールする**

実行:

```powershell
corepack enable
pnpm install
```

期待結果: `pnpm-lock.yaml` が作成され、install が終了コード 0 で完了する。

- [ ] **ステップ 4: 子パッケージ未作成だけが理由で空ワークスペースのスクリプトが失敗することを確認する**

実行:

```powershell
pnpm typecheck
```

期待結果: pnpm が一致する workspace package なし、または package script なしを報告する。タスク 2 でパッケージを作成する前なので許容する。

- [ ] **ステップ 5: ワークスペース基盤をコミットする**

```powershell
git add package.json pnpm-workspace.yaml tsconfig.base.json .editorconfig .env.example docker-compose.yml .gitignore pnpm-lock.yaml
git commit -m "chore: add workspace foundation"
```

---

### タスク 2: 共有契約とプレイヤーネーム検証

**対象ファイル:**

- 作成: `packages/shared/package.json`
- 作成: `packages/shared/tsconfig.json`
- 作成: `packages/shared/src/displayName.test.ts`
- 作成: `packages/shared/src/displayName.ts`
- 作成: `packages/shared/src/contracts.ts`
- 作成: `packages/shared/src/index.ts`

- [ ] **ステップ 1: パッケージメタデータを作成する**

`packages/shared/package.json` を作成する:

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

`packages/shared/tsconfig.json` を作成する:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "noEmit": true
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **ステップ 2: 失敗するプレイヤーネーム検証テストを書く**

`packages/shared/src/displayName.test.ts` を作成する:

```ts
import { describe, expect, it } from "vitest";
import { validateDisplayName } from "./displayName";

describe("validateDisplayName", () => {
  it("trims leading and trailing spaces", () => {
    expect(validateDisplayName("  たろう  ")).toEqual({
      ok: true,
      value: "たろう",
    });
  });

  it("rejects blank names after trimming", () => {
    expect(validateDisplayName("　 ㅤ ")).toEqual({
      ok: false,
      error: "display_name_required",
    });
  });

  it("rejects names longer than 32 code points", () => {
    const value = "あ".repeat(33);
    expect(validateDisplayName(value)).toEqual({
      ok: false,
      error: "display_name_too_long",
    });
  });

  it("rejects control characters", () => {
    expect(validateDisplayName("a\nb")).toEqual({
      ok: false,
      error: "display_name_control_character",
    });
  });
});
```

- [ ] **ステップ 3: テストを実行して失敗を確認する**

実行:

```powershell
pnpm --filter @katakanon/shared test
```

期待結果: `./displayName` が存在しないため FAIL。

- [ ] **ステップ 4: プレイヤーネーム検証実装と契約型を追加する**

`packages/shared/src/displayName.ts` を作成する:

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

`packages/shared/src/contracts.ts` を作成する:

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

`packages/shared/src/index.ts` を作成する:

```ts
export * from "./contracts";
export * from "./displayName";
```

- [ ] **ステップ 5: shared のテストを実行する**

実行:

```powershell
pnpm --filter @katakanon/shared test
pnpm --filter @katakanon/shared typecheck
```

期待結果: 両方のコマンドが PASS。

- [ ] **ステップ 6: 共有契約をコミットする**

```powershell
git add packages/shared
git commit -m "feat: add shared contracts"
```

---

### タスク 3: game-core 状態機械

**対象ファイル:**

- 作成: `packages/game-core/package.json`
- 作成: `packages/game-core/tsconfig.json`
- 作成: `packages/game-core/src/game.test.ts`
- 作成: `packages/game-core/src/game.ts`
- 作成: `packages/game-core/src/index.ts`

- [ ] **ステップ 1: game-core のパッケージメタデータを作成する**

`packages/game-core/package.json` を作成する:

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

`packages/game-core/tsconfig.json` を作成する:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "noEmit": true
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **ステップ 2: 失敗するゲーム進行テストを書く**

`packages/game-core/src/game.test.ts` を作成する:

```ts
import { describe, expect, it } from "vitest";
import {
  createRoom,
  enqueueAnswer,
  judgeAnswer,
  joinRoom,
  makeSnapshot,
  skipTurn,
  startGame,
} from "./game";

const words = [
  { id: "w1", surface: "ラーメン" },
  { id: "w2", surface: "テレビ" },
  { id: "w3", surface: "コンビニ" },
];

function roomWithThreePlayers() {
  const room = createRoom({
    roomId: "room-1",
    roomCode: "ABCD12",
    hostParticipantId: "p1",
    hostDisplayName: "ホスト",
    rounds: 1,
    now: "2026-04-26T00:00:00.000Z",
  });

  const joined2 = joinRoom(room, {
    participantId: "p2",
    displayName: "参加者2",
    now: "2026-04-26T00:00:01.000Z",
  });

  return joinRoom(joined2, {
    participantId: "p3",
    displayName: "参加者3",
    now: "2026-04-26T00:00:02.000Z",
  });
}

describe("game core", () => {
  it("starts a game with three online players and hides the word from non-speakers", () => {
    const room = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"],
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
      now: "2026-04-26T00:00:00.000Z",
    });

    expect(() => startGame(room, { words, orderedSpeakerIds: ["p1"] })).toThrow(
      "minimum_three_online_players_required",
    );
  });

  it("queues each answerer only once", () => {
    const room = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"],
    });

    const queued = enqueueAnswer(room, "p2");
    expect(enqueueAnswer(queued, "p2").currentTurn?.answerQueue).toEqual([
      "p2",
    ]);
  });

  it("adds one point to answerer and speaker on correct answer", () => {
    const started = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"],
    });
    const queued = enqueueAnswer(started, "p2");
    const next = judgeAnswer(queued, {
      speakerId: "p1",
      answererId: "p2",
      result: "correct",
    });

    expect(next.participants.find((p) => p.id === "p1")?.score).toBe(1);
    expect(next.participants.find((p) => p.id === "p2")?.score).toBe(1);
    expect(next.currentTurn?.speakerId).toBe("p2");
    expect(next.currentTurn?.answerQueue).toEqual([]);
  });

  it("removes only the first answerer on incorrect answer", () => {
    const started = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"],
    });
    const queued = enqueueAnswer(enqueueAnswer(started, "p2"), "p3");
    const next = judgeAnswer(queued, {
      speakerId: "p1",
      answererId: "p2",
      result: "incorrect",
    });

    expect(next.currentTurn?.speakerId).toBe("p1");
    expect(next.currentTurn?.answerQueue).toEqual(["p3"]);
  });

  it("finishes after the final turn is skipped", () => {
    let room = startGame(roomWithThreePlayers(), {
      words,
      orderedSpeakerIds: ["p1", "p2", "p3"],
    });

    room = skipTurn(room, "p1");
    room = skipTurn(room, "p2");
    room = skipTurn(room, "p3");

    expect(room.phase).toBe("finished");
    expect(room.currentTurn).toBeUndefined();
  });
});
```

- [ ] **ステップ 3: テストを実行して失敗を確認する**

実行:

```powershell
pnpm --filter @katakanon/game-core test
```

期待結果: `./game` が存在しないため FAIL。

- [ ] **ステップ 4: game-core を実装する**

`packages/game-core/src/game.ts` を作成する:

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
        joinedAt: input.now,
      },
    ],
    speakerOrder: [],
    wordDeck: [],
    usedWordIds: [],
  };
}

export function joinRoom(
  room: GameRoom,
  input: { participantId: ParticipantId; displayName: string; now: string },
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
        joinedAt: input.now,
      },
    ],
  };
}

export function startGame(
  room: GameRoom,
  input: { words: Word[]; orderedSpeakerIds: ParticipantId[] },
): GameRoom {
  const onlineParticipants = room.participants.filter(
    (participant) => participant.isOnline,
  );
  const requiredWordCount = onlineParticipants.length * room.rounds;

  if (onlineParticipants.length < 3) {
    throw new Error("minimum_three_online_players_required");
  }

  if (input.words.length < requiredWordCount) {
    throw new Error("not_enough_words");
  }

  const speakerOrder = input.orderedSpeakerIds;
  return beginTurn(
    {
      ...room,
      phase: "turn",
      speakerOrder,
      wordDeck: input.words,
      usedWordIds: [],
    },
    0,
    0,
  );
}

export function enqueueAnswer(
  room: GameRoom,
  participantId: ParticipantId,
): GameRoom {
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
      answerQueue: [...turn.answerQueue, participantId],
    },
  };
}

export function judgeAnswer(
  room: GameRoom,
  input: {
    speakerId: ParticipantId;
    answererId: ParticipantId;
    result: "correct" | "incorrect";
  },
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
        answerQueue: turn.answerQueue.slice(1),
      },
    };
  }

  const participants = room.participants.map((participant) => {
    if (
      participant.id === input.speakerId ||
      participant.id === input.answererId
    ) {
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

export function makeSnapshot(
  room: GameRoom,
  viewerId: ParticipantId,
): RoomSnapshot {
  const turn = room.currentTurn;
  const speakerId = turn?.speakerId ?? null;
  const viewer = room.participants.find(
    (participant) => participant.id === viewerId,
  );

  if (!viewer) {
    throw new Error("viewer_not_found");
  }

  const participants = room.participants.map((participant) => ({
    id: participant.id,
    displayName: participant.displayName,
    score: participant.score,
    isHost: participant.isHost,
    isOnline: participant.isOnline,
  }));

  return {
    roomCode: room.code,
    phase: room.phase,
    me: {
      participantId: viewerId,
      isHost: viewer.isHost,
      isSpeaker: viewerId === speakerId,
    },
    participants,
    roundIndex: turn?.roundIndex ?? null,
    turnIndex: turn?.turnIndex ?? null,
    speakerId,
    answerQueue: turn?.answerQueue ?? [],
    word: viewerId === speakerId ? (turn?.word.surface ?? null) : null,
    ranking: [...participants].sort((a, b) => b.score - a.score),
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
    currentTurn: undefined,
  };
}

function beginTurn(
  room: GameRoom,
  roundIndex: number,
  turnIndex: number,
): GameRoom {
  const speakerId = room.speakerOrder[turnIndex];
  if (!speakerId) {
    throw new Error("speaker_not_found");
  }

  const nextWord = room.wordDeck.find(
    (word) => !room.usedWordIds.includes(word.id),
  );
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
      answerQueue: [],
    },
    usedWordIds: [...room.usedWordIds, nextWord.id],
  };
}

function requireTurn(room: GameRoom): NonNullable<GameRoom["currentTurn"]> {
  if (room.phase !== "turn" || !room.currentTurn) {
    throw new Error("room_is_not_in_turn");
  }

  return room.currentTurn;
}
```

`packages/game-core/src/index.ts` を作成する:

```ts
export * from "./game";
```

- [ ] **ステップ 5: game-core のテストを実行する**

実行:

```powershell
pnpm --filter @katakanon/game-core test
```

期待結果: PASS。

- [ ] **ステップ 6: game-core の typecheck を実行する**

実行:

```powershell
pnpm --filter @katakanon/game-core typecheck
```

期待結果: PASS。

- [ ] **ステップ 7: game-core をコミットする**

```powershell
git add packages/game-core
git commit -m "feat: add game core vertical slice"
```

---

### タスク 4: サーバパッケージ、Prisma スキーマ、永続化境界

**対象ファイル:**

- 作成: `apps/server/package.json`
- 作成: `apps/server/tsconfig.json`
- 作成: `apps/server/prisma/schema.prisma`
- 作成: `apps/server/src/config.ts`
- 作成: `apps/server/src/db.ts`
- 作成: `apps/server/src/words.ts`

- [ ] **ステップ 1: サーバのパッケージメタデータを作成する**

`apps/server/package.json` を作成する:

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

`apps/server/tsconfig.json` を作成する:

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

- [ ] **ステップ 2: Prisma スキーマを追加する**

`apps/server/prisma/schema.prisma` を作成する:

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

- [ ] **ステップ 3: サーバ設定と固定お題リストを追加する**

`apps/server/src/config.ts` を作成する:

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
      "postgresql://katakanon:katakanon@localhost:5432/katakanon?schema=public",
  };
}
```

`apps/server/src/db.ts` を作成する:

```ts
import { PrismaClient } from "@prisma/client";

export const prisma = new PrismaClient();
```

`apps/server/src/words.ts` を作成する:

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
  { id: "w10", surface: "パソコン" },
];
```

- [ ] **ステップ 4: Prisma スキーマを検証する**

実行:

```powershell
pnpm --filter @katakanon/server prisma:validate
pnpm --filter @katakanon/server prisma:generate
```

期待結果: 両方のコマンドが PASS。

- [ ] **ステップ 5: サーバスキーマ基盤をコミットする**

```powershell
git add apps/server
git commit -m "feat: add server persistence schema"
```

---

### タスク 5: Fastify REST API と実行時ルームストア

**対象ファイル:**

- 作成: `apps/server/src/runtimeStore.ts`
- 作成: `apps/server/src/token.ts`
- 作成: `apps/server/src/http.test.ts`
- 作成: `apps/server/src/http.ts`

- [ ] **ステップ 1: 失敗する HTTP テストを書く**

`apps/server/src/http.test.ts` を作成する:

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
      payload: { displayName: "ホスト", rounds: 1 },
    });

    expect(response.statusCode).toBe(201);
    expect(response.json()).toMatchObject({
      roomCode: expect.any(String),
      participantId: expect.any(String),
      participantToken: expect.any(String),
    });
  });

  it("rejects invalid display names", async () => {
    const app = buildHttpServer({ store: createRuntimeStore() });
    const response = await app.inject({
      method: "POST",
      url: "/rooms",
      payload: { displayName: "", rounds: 1 },
    });

    expect(response.statusCode).toBe(400);
    expect(response.json()).toEqual({ error: "display_name_required" });
  });
});
```

- [ ] **ステップ 2: テストを実行して失敗を確認する**

実行:

```powershell
pnpm --filter @katakanon/server test
```

期待結果: `./http` と `./runtimeStore` が存在しないため FAIL。

- [ ] **ステップ 3: 実行時ストアとトークンヘルパーを実装する**

`apps/server/src/runtimeStore.ts` を作成する:

```ts
import type { GameRoom } from "@katakanon/game-core";

export type RuntimeStore = {
  roomsByCode: Map<string, GameRoom>;
  participantTokens: Map<string, { participantId: string; roomCode: string }>;
};

export function createRuntimeStore(): RuntimeStore {
  return {
    roomsByCode: new Map(),
    participantTokens: new Map(),
  };
}
```

`apps/server/src/token.ts` を作成する:

```ts
import { createHash, randomUUID } from "node:crypto";

export function createParticipantToken(): string {
  return randomUUID();
}

export function hashParticipantToken(token: string): string {
  return createHash("sha256").update(token).digest("hex");
}
```

- [ ] **ステップ 4: Fastify ルートを実装する**

`apps/server/src/http.ts` を作成する:

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
      now,
    });

    input.store.roomsByCode.set(roomCode, room);
    input.store.participantTokens.set(token, { participantId, roomCode });

    return reply.status(201).send({
      roomCode,
      participantId,
      participantToken: token,
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
      now: new Date().toISOString(),
    });

    input.store.roomsByCode.set(room.code, nextRoom);
    input.store.participantTokens.set(token, {
      participantId,
      roomCode: room.code,
    });

    return reply.status(201).send({
      roomCode: room.code,
      participantId,
      participantToken: token,
    });
  });

  app.post<{
    Params: { code: string };
    Body: { participantToken: string };
  }>("/rooms/:code/resume", async (request, reply) => {
    const session = input.store.participantTokens.get(
      request.body.participantToken,
    );
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

- [ ] **ステップ 5: HTTP テストを実行する**

実行:

```powershell
pnpm --filter @katakanon/server test
pnpm --filter @katakanon/server typecheck
```

期待結果: 両方のコマンドが PASS。

- [ ] **ステップ 6: HTTP API をコミットする**

```powershell
git add apps/server/src
git commit -m "feat: add room HTTP API"
```

---

### タスク 6: Socket.IO 縦スライス

**対象ファイル:**

- 作成: `apps/server/src/socket.ts`
- 作成: `apps/server/src/main.ts`

- [ ] **ステップ 1: Socket.IO コマンドを実装する**

`apps/server/src/socket.ts` を作成する:

```ts
import type { Server as HttpServer } from "node:http";
import type { GameRoom } from "@katakanon/game-core";
import {
  enqueueAnswer,
  judgeAnswer,
  makeSnapshot,
  skipTurn,
  startGame,
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
      origin: input.webOrigin,
    },
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
      const actor = room?.participants.find(
        (participant) => participant.id === participantId,
      );

      if (!room || !actor?.isHost) {
        socket.emit("command:error", { error: "host_only_operation" });
        return;
      }

      const nextRoom = startGame(room, {
        words: fixedWords,
        orderedSpeakerIds: room.participants.map(
          (participant) => participant.id,
        ),
      });

      input.store.roomsByCode.set(roomCode, nextRoom);
      emitRoomState(io, input.store, roomCode);
    });

    socket.on("answer:enqueue", () => {
      updateRoom(socket, input.store, (room, participantId) =>
        enqueueAnswer(room, participantId),
      );
      emitRoomState(io, input.store, socket.data.roomCode);
    });

    socket.on("answer:judge", ({ answererId, result }) => {
      updateRoom(socket, input.store, (room, participantId) =>
        judgeAnswer(room, { speakerId: participantId, answererId, result }),
      );
      emitRoomState(io, input.store, socket.data.roomCode);
    });

    socket.on("answer:skip", () => {
      updateRoom(socket, input.store, (room, participantId) =>
        skipTurn(room, participantId),
      );
      emitRoomState(io, input.store, socket.data.roomCode);
    });
  });

  return io;
}

function updateRoom(
  socket: Socket,
  store: RuntimeStore,
  updater: (room: GameRoom, participantId: string) => GameRoom,
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
    socket.emit("command:error", {
      error: error instanceof Error ? error.message : "unknown_error",
    });
  }
}

function emitRoomState(io: Server, store: RuntimeStore, roomCode: string) {
  const room = store.roomsByCode.get(roomCode);
  if (!room) {
    return;
  }

  for (const participant of room.participants) {
    io.to(participantRoom(roomCode, participant.id)).emit(
      "state:full",
      makeSnapshot(room, participant.id),
    );
  }
}

function participantRoom(roomCode: string, participantId: string): string {
  return `${roomCode}:participant:${participantId}`;
}
```

`apps/server/src/main.ts` を作成する:

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
  origin: config.webOrigin,
});

attachSocketServer({
  httpServer: app.server,
  store,
  webOrigin: config.webOrigin,
});

await app.listen({ port: config.port, host: "0.0.0.0" });
```

- [ ] **ステップ 2: サーバの typecheck を実行する**

実行:

```powershell
pnpm --filter @katakanon/server typecheck
```

期待結果: PASS。

- [ ] **ステップ 3: Socket.IO サーバスライスをコミットする**

```powershell
git add apps/server/src/socket.ts apps/server/src/main.ts
git commit -m "feat: add socket vertical slice"
```

---

### タスク 7: Next.js Web 縦スライス

**対象ファイル:**

- 作成: `apps/web/package.json`
- 作成: `apps/web/tsconfig.json`
- 作成: `apps/web/next.config.mjs`
- 作成: `apps/web/src/app/layout.tsx`
- 作成: `apps/web/src/app/page.tsx`
- 作成: `apps/web/src/app/rooms/[code]/page.tsx`
- 作成: `apps/web/src/app/globals.css`

- [ ] **ステップ 1: web のパッケージメタデータを作成する**

`apps/web/package.json` を作成する:

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

`apps/web/tsconfig.json` を作成する:

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

`apps/web/next.config.mjs` を作成する:

```js
/** @type {import("next").NextConfig} */
const nextConfig = {};

export default nextConfig;
```

- [ ] **ステップ 2: アプリシェルと基本スタイルを追加する**

`apps/web/src/app/layout.tsx` を作成する:

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "カタカナ禁止ゲーム",
  description: "リアルタイムで遊ぶカタカナ禁止ゲーム",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja">
      <body>{children}</body>
    </html>
  );
}
```

`apps/web/src/app/globals.css` を作成する:

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

- [ ] **ステップ 3: 作成/参加ページを追加する**

`apps/web/src/app/page.tsx` を作成する:

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
      body: JSON.stringify({ displayName, rounds: 1 }),
    });
    const payload = await response.json();
    if (!response.ok) {
      setError(payload.error);
      return;
    }
    localStorage.setItem(
      `katakanon:${payload.roomCode}:token`,
      payload.participantToken,
    );
    router.push(`/rooms/${payload.roomCode}`);
  }

  async function joinRoom() {
    setError(null);
    const code = roomCode.trim().toUpperCase();
    const response = await fetch(`${serverUrl}/rooms/${code}/join`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ displayName }),
    });
    const payload = await response.json();
    if (!response.ok) {
      setError(payload.error);
      return;
    }
    localStorage.setItem(
      `katakanon:${payload.roomCode}:token`,
      payload.participantToken,
    );
    router.push(`/rooms/${payload.roomCode}`);
  }

  return (
    <main className="shell stack">
      <h1>カタカナ禁止ゲーム</h1>
      <section className="panel stack">
        <label className="stack">
          プレイヤーネーム
          <input
            value={displayName}
            onChange={(event) => setDisplayName(event.target.value)}
          />
        </label>
        <div className="row">
          <button className="primary" onClick={createRoom}>
            ルーム作成
          </button>
        </div>
      </section>
      <section className="panel stack">
        <label className="stack">
          ルームコード
          <input
            value={roomCode}
            onChange={(event) => setRoomCode(event.target.value)}
          />
        </label>
        <button className="secondary" onClick={joinRoom}>
          参加
        </button>
      </section>
      {error ? <p role="alert">{error}</p> : null}
    </main>
  );
}
```

- [ ] **ステップ 4: ルームページを追加する**

`apps/web/src/app/rooms/[code]/page.tsx` を作成する:

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
    socket.on("command:error", (payload: { error: string }) =>
      setError(payload.error),
    );
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
              <button
                className="primary"
                onClick={() => socket.emit("game:start")}
              >
                開始
              </button>
            ) : null}
            {snapshot.phase === "turn" && !snapshot.me.isSpeaker ? (
              <button
                className="secondary"
                onClick={() => socket.emit("answer:enqueue")}
              >
                解答
              </button>
            ) : null}
            {snapshot.phase === "turn" && snapshot.me.isSpeaker ? (
              <>
                <button
                  className="primary"
                  disabled={snapshot.answerQueue.length === 0}
                  onClick={() =>
                    socket.emit("answer:judge", {
                      answererId: snapshot.answerQueue[0],
                      result: "correct",
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
                      result: "incorrect",
                    })
                  }
                >
                  不正解
                </button>
                <button
                  className="secondary"
                  onClick={() => socket.emit("answer:skip")}
                >
                  スキップ
                </button>
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

- [ ] **ステップ 5: web の typecheck を実行する**

実行:

```powershell
pnpm --filter @katakanon/web typecheck
```

期待結果: PASS。

- [ ] **ステップ 6: web 縦スライスをコミットする**

```powershell
git add apps/web
git commit -m "feat: add web vertical slice"
```

---

### タスク 8: CI と README

**対象ファイル:**

- 作成: `.github/workflows/ci.yml`
- 作成または変更: `README.md`

- [ ] **ステップ 1: CI workflow を追加する**

`.github/workflows/ci.yml` を作成する:

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

- [ ] **ステップ 2: README の土台を追加する**

`README.md` を作成する:

````md
# カタカナ禁止ゲーム

Web 系転職ポートフォリオとして作る、リアルタイム参加型のカタカナ禁止ゲームです。

## 技術スタック

- Frontend: Next.js、React、TypeScript
- Backend: Fastify、Socket.IO、TypeScript
- Database: PostgreSQL
- ORM: Prisma
- Test: Vitest
- Local infra: Docker Compose
- CI: GitHub Actions

## アーキテクチャ

ゲーム判定と状態遷移はサーバ側の状態機械に集約します。クライアントはコマンドを送信し、サーバから配信されるスナップショットを表示します。

MVP のゲーム中状態はサーバメモリに保持し、ルーム、参加者、結果の永続化は PostgreSQL と Prisma で扱います。

## セットアップ

```powershell
corepack enable
pnpm install
Copy-Item .env.example .env
docker compose up -d
pnpm prisma:validate
pnpm prisma:generate
pnpm dev
```

## チェック

```powershell
pnpm typecheck
pnpm test
pnpm prisma:validate
pnpm prisma:generate
```

## MVP 縦スライス

この段階でできること:

- ルーム作成
- ルーム参加
- ロビーの参加者表示
- ホストによるゲーム開始
- 説明者だけへのお題表示
- 解答キュー登録
- 正解、誤答、スキップ
- 最終結果表示の土台

## 設計ドキュメント

- `docs/01_requirements.md`
- `docs/superpowers/specs/2026-04-26-portfolio-architecture-design.md`
- `docs/superpowers/plans/2026-04-26-mvp-foundation-vertical-slice.md`

````

- [ ] **ステップ 3: 全チェックを実行する**

実行:

```powershell
pnpm prisma:validate
pnpm prisma:generate
pnpm typecheck
pnpm test
```

期待結果: すべてのコマンドが PASS。

- [ ] **ステップ 4: CI と README をコミットする**

```powershell
git add .github/workflows/ci.yml README.md
git commit -m "docs: add project README and CI"
```

---

## 自己レビュー

Spec 対応:

- 合意した技術スタック（Next.js、Fastify、Socket.IO、PostgreSQL、Prisma、Docker Compose、CI）を扱っている。
- 最初のゲームプレイスライスについて、サーバ所有のゲーム状態を扱っている。
- 匿名参加と再接続準備のための `participantToken` を扱っている。
- 参加者ごとのスナップショット配信と、説明者以外へのお題秘匿を扱っている。
- DB スキーマ基盤を扱っている。ただし最初の実行時ストアはメモリのみとする。
- アバター描画、指摘・投票、ホスト移譲、本番デプロイは扱わない。これらは意図的に後続計画へ分離する。

プレースホルダー確認:

- 未解決のプレースホルダーや、曖昧な実装指示は残していない。
- コードを書くステップには、具体的なファイル内容または具体的な置換指示を含めている。
- 各タスクには、実行コマンドと期待結果を明記している。

型の整合性:

- 共有の `RoomSnapshot`、`ParticipantId`、`RoomCode` は game-core、server、web で一貫して使う。
- game-core のコマンド名と Socket.IO の command handler が対応している。
- `participantToken` の命名は HTTP、Socket.IO、web の localStorage、README で一貫している。
