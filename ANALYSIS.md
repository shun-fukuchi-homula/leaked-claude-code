# Claude Code アーキテクチャ深層分析レポート

> リポジトリ: leaked-claude-code (約26,351行 TypeScript)
> 分析日: 2026-04-01
> 対象: 公式ドキュメントに記載されていないアーキテクチャ・仕組み・コンセプト・革新技術

---

## 目次

1. [全体アーキテクチャ](#1-全体アーキテクチャ)
2. [Bridge サブシステム — Remote Control 基盤](#2-bridge-サブシステム--remote-control-基盤)
3. [トランスポート層 — ハイブリッド通信戦略](#3-トランスポート層--ハイブリッド通信戦略)
4. [CCR v2 プロトコル — 次世代アーキテクチャ](#4-ccr-v2-プロトコル--次世代アーキテクチャ)
5. [セキュリティ・認証モデル](#5-セキュリティ認証モデル)
6. [メッセージングプロトコル — NDJSON・重複排除・バッチング](#6-メッセージングプロトコル--ndjson重複排除バッチング)
7. [パーミッションシステム](#7-パーミッションシステム)
8. [フィーチャーフラグシステム](#8-フィーチャーフラグシステム)
9. [Buddy システム — 隠し機能](#9-buddy-システム--隠し機能)
10. [プラグインシステム](#10-プラグインシステム)
11. [革新的実装パターン](#11-革新的実装パターン)
12. [内部API エンドポイント一覧](#12-内部api-エンドポイント一覧)
13. [定数・タイムアウト・閾値一覧](#13-定数タイムアウト閾値一覧)

---

## 1. 全体アーキテクチャ

### サブシステム構成

```
┌─────────────────────────────────────────────────┐
│                   Claude Code                    │
├──────────┬──────────┬──────────┬────────────────┤
│  Bridge  │   CLI    │  Buddy   │   Assistant    │
│ (48%)    │ (45%)    │ (5%)     │   (0.3%)       │
│ 12,613行 │ 11,883行 │ 1,298行  │   87行         │
├──────────┴──────────┴──────────┴────────────────┤
│             共通ユーティリティ層                   │
│  (analytics, auth, config, debug, permissions)   │
└─────────────────────────────────────────────────┘
```

### メッセージフロー全体像

```
claude.ai / VS Code / JetBrains
         │
         ▼
  ┌──────────────┐    ┌─────────────────┐
  │  CCR Server   │◀──│  Environments   │
  │  (v1 or v2)   │    │     API         │
  └──────┬───────┘    └────────┬────────┘
         │                     │
    SSE/WS/CCR            Poll/ACK
         │                     │
  ┌──────▼─────────────────────▼────────┐
  │         Bridge (replBridge)          │
  │  ┌──────────┐  ┌─────────────────┐  │
  │  │ Transport │  │ Message Router  │  │
  │  │ (v1/v2)  │  │ (echo dedup)    │  │
  │  └──────────┘  └─────────────────┘  │
  └──────────────────┬──────────────────┘
                     │ NDJSON (stdin/stdout)
           ┌─────────▼──────────┐
           │  Child CLI Process  │
           │  (StructuredIO)     │
           └────────────────────┘
```

---

## 2. Bridge サブシステム — Remote Control 基盤

### 2.1 セッションスポーンモード（3種）

`bridge/types.ts:63-69` で定義される3つのモード:

| モード | 説明 | 用途 |
|--------|------|------|
| `single-session` | 1セッション/cwd、終了時にBridge切断 | デフォルト |
| `worktree` | 永続サーバー、各セッションに独立gitワークツリー | 並行作業 |
| `same-dir` | 永続サーバー、全セッションがcwd共有 | 簡易マルチセッション |

### 2.2 Env-less Bridge（新世代パス）

`bridge/remoteBridgeCore.ts` で実装。従来のEnvironments APIをバイパスし、直接CCR v2に接続する低レイテンシパス。

**従来 (v1):**
```
OAuth → Register Environment → Poll for Work → Decode WorkSecret → WS/SSE
```

**新世代 (v2 env-less):**
```
OAuth → POST /v1/code/sessions → POST /v1/code/sessions/{id}/bridge → SSE + CCRClient
```

GrowthBookフラグ `tengu_bridge_repl_v2` でゲートされる。

### 2.3 BridgePointer — クラッシュリカバリ

`bridge/bridgePointer.ts` に実装。プロセスが異常終了した場合のセッション復旧機構:

- セッション開始時にポインタファイルを `~/.claude/projects/{sanitized_path}/bridge-pointer.json` に書き込み
- 定期的にmtimeを更新（TTL: 4時間 = `BRIDGE_POINTER_TTL_MS`）
- 次回起動時にポインタを検出 → `--session-id` で再接続を提案
- **ワークツリー横断検索**: `readBridgePointerAcrossWorktrees()` が `git worktree list` を実行し、全兄弟ワークツリーのポインタを並列スキャン（MAX_WORKTREE_FANOUT = 50）

### 2.4 FlushGate — メッセージ順序保証

`bridge/flushGate.ts` に実装。セッション開始時の履歴フラッシュ中に新メッセージがインターリーブするのを防ぐ状態マシン:

```
start() → enqueue() returns true (キューイング中)
end()   → 溜まったアイテムを返す → enqueue() returns false
drop()  → 破棄（永続的トランスポートclose）
deactivate() → アクティブフラグのみクリア（トランスポート交換時）
```

### 2.5 CapacityWake — ポーリングループ起床機構

`bridge/capacityWake.ts` に実装。At-capacity時のスリープを、セッション終了やトランスポートロストで即座に起床させる:

```typescript
// 外部シグナル（シャットダウン）とキャパシティ起床の2つをマージしたAbortSignalを生成
const merged = new AbortController()
outerSignal.addEventListener('abort', abort, { once: true })
wakeController.signal.addEventListener('abort', abort, { once: true })
```

---

## 3. トランスポート層 — ハイブリッド通信戦略

### 3.1 トランスポート階層

```
Transport (インターフェース)
├── WebSocketTransport (自動再接続、スリープ検出)
│   └── HybridTransport (WebSocket + SSE フォールバック)
├── SSETransport (HTTP長期ポーリング)
└── CCRClient (CCR v2 /worker/* エンドポイント)
```

### 3.2 WebSocket スリープ検出

`cli/transports/WebSocketTransport.ts:36`:

```typescript
const SLEEP_DETECTION_THRESHOLD_MS = 60_000 // 60秒

// 再接続試行間の空白が60秒を超えた場合、マシンがスリープしたと判定
// → 再接続バジェットをリセットして再試行
```

**定数:**
- 再接続バジェット: 10分 (`RECONNECT_GIVE_UP_MS = 600_000`)
- 基本再接続遅延: 1秒 (`BASE_RECONNECT_DELAY = 1000`)
- 最大再接続遅延: 30秒 (`MAX_RECONNECT_DELAY = 30000`)
- Keep-alive: 5分ごと (`KEEPALIVE_INTERVAL = 300_000`)
- Ping: 10秒ごと (`PING_INTERVAL = 10000`)
- 永続closeコード: `1002`(プロトコルエラー), `4001`(セッション期限切れ), `4003`(認証なし)

### 3.3 HybridTransport — 透過的フォールバック

WebSocketが使えない環境（企業プロキシ等）ではSSEに自動フォールバック。WebSocketの読み取り + HTTPのPOST書き込みを多重化。

### 3.4 CircularBuffer — バックプレッシャー管理

WebSocketTransportは `CircularBuffer` を使用してメッセージの背圧を管理。デフォルトバッファサイズ: 1000メッセージ。

---

## 4. CCR v2 プロトコル — 次世代アーキテクチャ

### 4.1 Worker Epoch メカニズム

`cli/transports/ccrClient.ts` に実装。サーバーがworkerの世代を管理:

- `POST /worker/register` → `worker_epoch` を取得
- 全リクエストに `worker_epoch` を含める
- サーバーが409を返す → より新しいworkerが存在 → `onEpochMismatch` コールバック
  - 子プロセスモード: `process.exit(1)` → 親が再スポーン
  - インプロセスモード（replBridge）: トランスポートclose → ポーリングループで回復

### 4.2 text_delta 合体（コアレシング）

`cli/transports/ccrClient.ts:141` の `accumulateStreamEvents()`:

```
100ms間隔でstream_eventをバッファ
→ 同一content_blockのtext_deltaチャンクを蓄積
→ フラッシュ時に「フルスナップショット」として1イベントに合体
→ 途中接続のクライアントも完全テキストを取得（断片ではなく）
```

**StreamAccumulatorState:**
- `byMessage`: API message ID → blocks[index] → chunk配列
- `scopeToMessage`: `{session_id}:{parent_tool_use_id}` → アクティブなmessage ID

### 4.3 SerialBatchEventUploader

イベントを逐次的にバッチ化してHTTP POSTで送信:
- `maxBatchSize`: 100
- `maxBatchBytes`: 10MB
- `maxQueueSize`: 100,000（stream_event用）/ 200（internal-event用）
- リトライ: 指数バックオフ（500ms〜30s）+ ジッター

### 4.4 Delivery Tracking

```
SSE受信 → reportDelivery('received')
処理開始 → reportDelivery('processing')
処理完了 → reportDelivery('processed')
```

POST `/worker/events/delivery` でバッチ送信。CCRの `processing_at`/`processed_at` カラムを更新。

---

## 5. セキュリティ・認証モデル

### 5.1 トラステッドデバイストークン

`bridge/trustedDevice.ts` に実装。Bridgeセッションは `SecurityTier=ELEVATED`:

**登録フロー:**
1. `/login` 直後に `POST /api/auth/trusted_devices` を呼び出し
2. `display_name: "Claude Code on {hostname} · {platform}"` を送信
3. サーバーが `device_token` を返却
4. macOS Keychain (`getSecureStorage()`) に永続化
5. 以降の全Bridge APIコールで `X-Trusted-Device-Token` ヘッダーに付与

**制約:**
- サーバー側で `account_session.created_at < 10min` をゲート → ログイン直後にしか登録できない
- 90日ローリング有効期限
- `CLAUDE_TRUSTED_DEVICE_TOKEN` 環境変数でオーバーライド可能（テスト/カナリア用）

### 5.2 JWT管理

`bridge/jwtUtils.ts` に実装:

- **先行リフレッシュ**: 期限の5分前にリフレッシュをスケジュール（`TOKEN_REFRESH_BUFFER_MS = 300_000`）
- **フォールバック**: JWT期限をデコードできない場合、30分間隔でリフレッシュ（`FALLBACK_REFRESH_INTERVAL_MS`）
- **最大連続失敗**: 3回（`MAX_REFRESH_FAILURES`）
- **世代カウンタ**: 非同期リフレッシュの競合を防止（`generations` Map）

```typescript
// JWTプレフィックスの除去
const jwt = token.startsWith('sk-ant-si-')
  ? token.slice('sk-ant-si-'.length)
  : token
```

### 5.3 WorkSecret エンコーディング

`bridge/workSecret.ts` — base64url エンコードされたJSON:

```typescript
type WorkSecret = {
  version: 1,
  session_ingress_token: string,  // セッション認証トークン
  api_base_url: string,           // APIベースURL
  sources: Array<{ type: string, git_info?: {...} }>,
  auth: Array<{ type: string, token: string }>,
  claude_code_args?: Record<string, string>,
  mcp_config?: unknown,
  environment_variables?: Record<string, string>,
  use_code_sessions?: boolean     // CCR v2セレクタ
}
```

### 5.4 パストラバーサル防止

`bridge/bridgeApi.ts:41-53`:

```typescript
const SAFE_ID_PATTERN = /^[a-zA-Z0-9_-]+$/
export function validateBridgeId(id: string, label: string): string {
  if (!id || !SAFE_ID_PATTERN.test(id)) {
    throw new Error(`Invalid ${label}: contains unsafe characters`)
  }
  return id
}
```

### 5.5 シークレット除去

`bridge/debugUtils.ts` — デバッグログから機密情報を自動除去:

```typescript
const SECRET_FIELD_NAMES = [
  'session_ingress_token', 'environment_secret',
  'access_token', 'secret', 'token'
]
// 16文字以上のトークンは先頭8文字...末尾4文字に短縮
```

---

## 6. メッセージングプロトコル — NDJSON・重複排除・バッチング

### 6.1 BoundedUUIDSet — O(1)メモリの重複排除

`bridge/bridgeMessaging.ts:429-461`:

FIFO制限付きセット。循環バッファで実装:

```typescript
class BoundedUUIDSet {
  private readonly ring: (string | undefined)[]
  private readonly set = new Set<string>()
  private writeIdx = 0

  add(uuid: string): void {
    if (this.set.has(uuid)) return
    const evicted = this.ring[this.writeIdx]
    if (evicted !== undefined) this.set.delete(evicted)
    this.ring[this.writeIdx] = uuid
    this.set.add(uuid)
    this.writeIdx = (this.writeIdx + 1) % this.capacity
  }
}
```

**用途:**
- `recentPostedUUIDs`: 自分が送信したメッセージのエコーを無視
- `recentInboundUUIDs`: サーバーからの再配信を無視（SSEのseq-num引き継ぎが失敗した場合のセーフティネット）

デフォルトバッファサイズ: 2000 (`uuid_dedup_buffer_size`)

### 6.2 メッセージタイプガード

```typescript
// SDKMessage: type フィールドの判別共用体
function isSDKMessage(value: unknown): value is SDKMessage
// control_response: パーミッション判定の応答
function isSDKControlResponse(value: unknown): value is SDKControlResponse
// control_request: サーバーからの制御要求
function isSDKControlRequest(value: unknown): value is SDKControlRequest
```

### 6.3 Bridgeメッセージフィルタリング

```typescript
function isEligibleBridgeMessage(m: Message): boolean {
  // 仮想メッセージ（REPL内部呼び出し）は除外
  if ((m.type === 'user' || m.type === 'assistant') && m.isVirtual) return false
  // user, assistant, system(local_command) のみサーバーに転送
  return m.type === 'user' || m.type === 'assistant' ||
    (m.type === 'system' && m.subtype === 'local_command')
}
```

### 6.4 タイトル抽出

`extractTitleText()` — セッションタイトル生成用のテキスト抽出:
- メタメッセージ（ナッジ）を除外
- ツール結果を除外
- コンパクトサマリーを除外
- 人間以外の起源（タスク通知、チャンネルメッセージ）を除外
- `<display_tag>` を除去

---

## 7. パーミッションシステム

### 7.1 StructuredIO のパーミッションフロー

`cli/structuredIO.ts` に実装:

```
ツール呼び出し
  │
  ▼
hasPermissionsToUseTool() → allow/deny → 即座に返却
  │
  ▼ (ask の場合)
Hook実行 と SDK権限プロンプト を並列で起動
  │
  ├── Hook が先に決定 → SDKプロンプトをAbort
  │
  └── SDKプロンプトが先に応答 → Hookの結果を無視
```

### 7.2 SandboxNetworkAccess — 合成ツール

```typescript
export const SANDBOX_NETWORK_ACCESS_TOOL_NAME = 'SandboxNetworkAccess'
```

サンドボックスのネットワーク許可リクエストを既存の `can_use_tool` プロトコルに乗せる。SDKホスト（VS Code, CCR等）が新プロトコルサブタイプなしでネットワークアクセス許可をプロンプトできる。

### 7.3 重複control_response防止

```typescript
private readonly resolvedToolUseIds = new Set<string>()
// MAX_RESOLVED_TOOL_USE_IDS = 1000
// 超過時はSetの挿入順で最古を削除
```

WebSocket再接続時に重複 `control_response` が到着するケースを防止。重複を処理すると `mutableMessages` に重複アシスタントメッセージが追加され、API 400エラー「tool_use ids must be unique」が発生する。

---

## 8. フィーチャーフラグシステム

### 8.1 コンパイル時フィーチャー（bun:bundle）

`feature()` 関数はBunのコンパイル時にインライン化される。外部ビルドではランタイムポリフィルで置換:

```typescript
import { feature } from 'bun:bundle'

// 正のパターン — docs/feature-gating.md 参照
// 負のパターン (if (!feature(...)) return) はインライン文字列リテラルを除去しない
return feature('BRIDGE_MODE')
  ? isClaudeAISubscriber() && getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
  : false
```

### 8.2 確認されたフィーチャーフラグ

コードから抽出されたフラグ:

| フラグ | 用途 |
|--------|------|
| `BRIDGE_MODE` | Remote Control機能全体 |
| `CCR_AUTO_CONNECT` | セッション開始時の自動CCR接続 |
| `CCR_MIRROR` | ミラーモード（ローカルセッションの転送） |
| `BASH_CLASSIFIER` | Bashコマンドの分類器 |
| `TRANSCRIPT_CLASSIFIER` | トランスクリプト分類器 |

### 8.3 GrowthBookフラグ

| フラグ名 | 用途 |
|----------|------|
| `tengu_ccr_bridge` | Bridge有効化 |
| `tengu_bridge_repl_v2` | Env-less Bridge有効化 |
| `tengu_bridge_repl_v2_cse_shim_enabled` | cse_→session_ IDリタグ |
| `tengu_bridge_poll_interval_config` | ポーリング間隔設定 |
| `tengu_bridge_repl_v2_config` | Env-less Bridge設定 |
| `tengu_bridge_min_version` | 最小バージョン要件 |
| `tengu_cobalt_harbor` | 自動CCR接続デフォルト |
| `tengu_ccr_mirror` | ミラーモード |
| `tengu_sessions_elevated_auth_enforcement` | トラステッドデバイス強制 |

---

## 9. Buddy システム — 隠し機能

### 9.1 決定論的コンパニオン生成

`buddy/companion.ts` — ユーザーID + ソルトから決定論的にペットを生成:

```typescript
const SALT = 'friend-2026-401'

// Mulberry32 PRNG（シード付き疑似乱数生成器）
function mulberry32(seed: number): () => number {
  let a = seed >>> 0
  return function() {
    a |= 0; a = (a + 0x6d2b79f5) | 0
    let t = Math.imul(a ^ (a >>> 15), 1 | a)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
```

### 9.2 種族とレアリティ

**18種族**: duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk

**種族名の難読化**: コードネームカナリアとの衝突を避けるため、種族名を `String.fromCharCode()` で構築:
```typescript
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
```

**レアリティ分布** (重み):
| レアリティ | 重み | 確率 | 帽子 |
|-----------|------|------|------|
| common | 60 | 60% | なし |
| uncommon | 25 | 25% | あり |
| rare | 10 | 10% | あり |
| epic | 4 | 4% | あり |
| legendary | 1 | 1% | あり |

**目**: `·`, `✦`, `×`, `◉`, `@`, `°`
**帽子**: none, crown, tophat, propeller, halo, wizard, beanie, tinyduck

### 9.3 ステータスシステム

5つのステータス: `DEBUGGING`, `PATIENCE`, `CHAOS`, `WISDOM`, `SNARK`

- 各レアリティにフロア値がある（common: 5, legendary: 50）
- 1つのピークステータス: `floor + 50 + random(30)`, 最大100
- 1つのダンプステータス: `max(1, floor - 10 + random(15))`
- 残り: `floor + random(40)`
- **シャイニー**: 1%確率 (`rng() < 0.01`)

### 9.4 Bones vs Soul

- **CompanionBones**: `hash(userId)` から毎回再生成（レアリティ、種族、目、帽子、シャイニー、ステータス）
- **CompanionSoul**: モデルが生成（名前、パーソナリティ）→ 設定ファイルに永続化
- Bonesは永続化しない → 種族名の変更やSPECIES配列の編集が既存コンパニオンを壊さない
- 設定ファイルの `companion` を編集してもレアリティを偽装できない

---

## 10. プラグインシステム

`cli/handlers/plugins.ts` に実装:

- マーケットプレースベースのプラグインインストール/アンインストール
- プラグイン検証（マニフェスト、コンテンツ）
- インストール数トラッキング
- MCPサーバー統合（`loadPluginMcpServers`）
- スコープ: installable と update で異なるスコープセット

---

## 11. 革新的実装パターン

### 11.1 セッションID互換性レイヤー

`bridge/sessionIdCompat.ts` + `bridge/workSecret.ts:62-73`:

CCR v2互換レイヤーが `session_*` と `cse_*` の2つのIDプレフィックスを使い分ける:
- Worker endpoints (`/worker/*`): `cse_*`
- Client-facing compat endpoints (`/v1/sessions/*`): `session_*`

```typescript
function sameSessionId(a: string, b: string): boolean {
  if (a === b) return true
  const aBody = a.slice(a.lastIndexOf('_') + 1)
  const bBody = b.slice(b.lastIndexOf('_') + 1)
  return aBody.length >= 4 && aBody === bBody
}
```

### 11.2 プロトコルバージョニング

```typescript
// localhost: /v2/ を使用（session-ingressに直接、Envoyリライトなし）
// 本番: /v1/ を使用（Envoyが /v1/ → /v2/ にリライト）
const version = isLocalhost ? 'v2' : 'v1'
```

### 11.3 Worker Epoch int64精度保護

```typescript
// protojsonはint64を文字列でシリアライズ（JS numberの精度損失を回避）
// Goエンコーダの設定によってはnumberの場合も
const raw = response.data?.worker_epoch
const epoch = typeof raw === 'string' ? Number(raw) : raw
if (!Number.isSafeInteger(epoch)) throw new Error(...)
```

### 11.4 フォルトインジェクション（テスト用）

`bridge/bridgeDebug.ts` — ant（内部ユーザー）専用のフォルトインジェクション:

```typescript
type BridgeFault = {
  method: 'pollForWork' | 'registerBridgeEnvironment' | 'reconnectSession' | 'heartbeatWork'
  kind: 'fatal' | 'transient'
  status: number
  errorType?: string
  count: number
}
```

`/bridge-kick` スラッシュコマンドからトリガー可能。`USER_TYPE === 'ant'` でガード。

### 11.5 画像ブロック正規化

`bridge/inboundMessages.ts` — iOSクライアントが `mediaType`（camelCase）を `media_type`（snake_case）の代わりに送る問題に対応:

```typescript
// Fast-path: 正規化不要なら元の配列参照を返す（ゼロアロケーション）
if (!blocks.some(isMalformedBase64Image)) return blocks
```

### 11.6 遅延インポートパターン

モジュール依存チェーンの爆発を防止:

```typescript
// utils/auth.ts は ~1300モジュールを推移的にプル
// (config → file → permissions → sessionStorage → commands)
// ダイナミックインポートで必要時のみロード
const { getClaudeAIOAuthTokens } = await import('../utils/auth.js')
```

### 11.7 OAuth 401 リトライ

`bridge/bridgeApi.ts:106-139`:

```typescript
async function withOAuthRetry<T>(
  fn: (accessToken: string) => Promise<{ status: number; data: T }>,
  context: string,
): Promise<{ status: number; data: T }> {
  const response = await fn(accessToken)
  if (response.status !== 401) return response
  // トークンリフレッシュ → 1回だけリトライ
  const refreshed = await deps.onAuth401(accessToken)
  if (refreshed) {
    const retryResponse = await fn(resolveAuth())
    if (retryResponse.status !== 401) return retryResponse
  }
  return response // リフレッシュ失敗時は元の401を返す
}
```

---

## 12. 内部API エンドポイント一覧

### Environments API (v1)

| メソッド | パス | 用途 |
|----------|------|------|
| POST | `/v1/environments/bridge` | 環境登録 |
| GET | `/v1/environments/{id}/work/poll` | ワークポーリング |
| POST | `/v1/environments/{id}/work/{id}/ack` | ワーク確認応答 |
| POST | `/v1/environments/{id}/work/{id}/stop` | ワーク停止 |
| POST | `/v1/environments/{id}/work/{id}/heartbeat` | ハートビート |
| DELETE | `/v1/environments/bridge/{id}` | 環境登録解除 |
| POST | `/v1/environments/{id}/bridge/reconnect` | セッション再接続 |

### Sessions API (v1)

| メソッド | パス | 用途 |
|----------|------|------|
| POST | `/v1/sessions` | セッション作成 |
| GET | `/v1/sessions/{id}` | セッション取得 |
| PATCH | `/v1/sessions/{id}` | タイトル更新 |
| POST | `/v1/sessions/{id}/archive` | セッションアーカイブ |
| POST | `/v1/sessions/{id}/events` | イベント送信 |

### Code Sessions API (v2)

| メソッド | パス | 用途 |
|----------|------|------|
| POST | `/v1/code/sessions` | セッション作成 |
| POST | `/v1/code/sessions/{id}/bridge` | Bridgeクレデンシャル取得 |
| POST | `/v1/code/sessions/{id}/worker/register` | Worker登録 |
| PUT | `/v1/code/sessions/{id}/worker` | Worker状態更新 |
| POST | `/v1/code/sessions/{id}/worker/heartbeat` | ハートビート |
| POST | `/v1/code/sessions/{id}/worker/events` | イベント送信 |
| POST | `/v1/code/sessions/{id}/worker/internal-events` | 内部イベント送信 |
| GET | `/v1/code/sessions/{id}/worker/events/stream` | SSEストリーム |
| POST | `/v1/code/sessions/{id}/worker/events/delivery` | 配信確認 |

### Session History API

| メソッド | パス | 用途 |
|----------|------|------|
| GET | `/v1/sessions/{id}/events?limit=N` | 最新イベント取得 |
| GET | `/v1/sessions/{id}/events?before=cursor&limit=N` | ページネーション |

### Auth API

| メソッド | パス | 用途 |
|----------|------|------|
| POST | `/api/auth/trusted_devices` | デバイス登録 |
| GET | `/api/oauth/files/{uuid}/content` | 添付ファイルダウンロード |

---

## 13. 定数・タイムアウト・閾値一覧

### セッション・ポーリング

| 定数 | 値 | 場所 |
|------|------|------|
| DEFAULT_SESSION_TIMEOUT_MS | 24h | types.ts |
| POLL_INTERVAL_NOT_AT_CAPACITY | 2s | pollConfigDefaults.ts |
| POLL_INTERVAL_AT_CAPACITY | 10min | pollConfigDefaults.ts |
| BRIDGE_POINTER_TTL_MS | 4h | bridgePointer.ts |
| MAX_WORKTREE_FANOUT | 50 | bridgePointer.ts |
| RECLAIM_OLDER_THAN_MS | 5s | pollConfigDefaults.ts |
| SESSION_KEEPALIVE_INTERVAL | 2min | pollConfigDefaults.ts |

### WebSocket

| 定数 | 値 | 場所 |
|------|------|------|
| RECONNECT_GIVE_UP_MS | 10min | WebSocketTransport.ts |
| BASE_RECONNECT_DELAY | 1s | WebSocketTransport.ts |
| MAX_RECONNECT_DELAY | 30s | WebSocketTransport.ts |
| SLEEP_DETECTION_THRESHOLD | 60s | WebSocketTransport.ts |
| KEEPALIVE_INTERVAL | 5min | WebSocketTransport.ts |
| PING_INTERVAL | 10s | WebSocketTransport.ts |
| MAX_BUFFER_SIZE | 1000 | WebSocketTransport.ts |

### CCR v2

| 定数 | 値 | 場所 |
|------|------|------|
| HEARTBEAT_INTERVAL | 20s | ccrClient.ts |
| STREAM_EVENT_FLUSH_INTERVAL | 100ms | ccrClient.ts |
| MAX_CONSECUTIVE_AUTH_FAILURES | 10 | ccrClient.ts |
| MAX_BATCH_SIZE | 100 | ccrClient.ts |
| MAX_BATCH_BYTES | 10MB | ccrClient.ts |
| MAX_QUEUE_SIZE (events) | 100,000 | ccrClient.ts |
| MAX_QUEUE_SIZE (internal) | 200 | ccrClient.ts |

### 認証

| 定数 | 値 | 場所 |
|------|------|------|
| TOKEN_REFRESH_BUFFER | 5min | jwtUtils.ts |
| FALLBACK_REFRESH_INTERVAL | 30min | jwtUtils.ts |
| MAX_REFRESH_FAILURES | 3 | jwtUtils.ts |
| REFRESH_RETRY_DELAY | 60s | jwtUtils.ts |
| TRUSTED_DEVICE_EXPIRY | 90d | trustedDevice.ts |

### Env-less Bridge

| 定数 | 値 | 場所 |
|------|------|------|
| INIT_RETRY_MAX_ATTEMPTS | 3 | envLessBridgeConfig.ts |
| INIT_RETRY_BASE_DELAY | 500ms | envLessBridgeConfig.ts |
| HTTP_TIMEOUT | 10s | envLessBridgeConfig.ts |
| UUID_DEDUP_BUFFER_SIZE | 2000 | envLessBridgeConfig.ts |
| HEARTBEAT_INTERVAL | 20s | envLessBridgeConfig.ts |
| HEARTBEAT_JITTER_FRACTION | 0.1 | envLessBridgeConfig.ts |
| TOKEN_REFRESH_BUFFER | 5min | envLessBridgeConfig.ts |
| TEARDOWN_ARCHIVE_TIMEOUT | 1.5s | envLessBridgeConfig.ts |
| CONNECT_TIMEOUT | 15s | envLessBridgeConfig.ts |

### パーミッション

| 定数 | 値 | 場所 |
|------|------|------|
| MAX_RESOLVED_TOOL_USE_IDS | 1000 | structuredIO.ts |
| MAX_ACTIVITIES | 10 | sessionRunner.ts |
| MAX_STDERR_LINES | 10 | sessionRunner.ts |

### デバッグ

| 定数 | 値 | 場所 |
|------|------|------|
| DEBUG_MSG_LIMIT | 2000 | debugUtils.ts |
| REDACT_MIN_LENGTH | 16 | debugUtils.ts |
| DOWNLOAD_TIMEOUT | 30s | inboundAttachments.ts |
| EMPTY_POLL_LOG_INTERVAL | 100 | bridgeApi.ts |

---

## 付録: 環境変数

| 変数名 | 用途 |
|--------|------|
| `CLAUDE_CODE_OAUTH_TOKEN` | OAuthトークン（Bridgeではストリップ） |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | 環境種別（`bridge`） |
| `CLAUDE_CODE_FORCE_SANDBOX` | サンドボックス強制 |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | セッションアクセストークン |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | v1 POSTモード |
| `CLAUDE_CODE_USE_CCR_V2` | CCR v2モード |
| `CLAUDE_CODE_WORKER_EPOCH` | Worker Epoch |
| `CLAUDE_TRUSTED_DEVICE_TOKEN` | トラステッドデバイストークン |
| `CLAUDE_CODE_CCR_MIRROR` | CCRミラーモード |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | カスタムOAuth URL |
| `USER_TYPE` | ユーザータイプ（`ant` = 内部） |
