---
read_when:
    - デバイスのペアリングリクエストを承認しています
    - デバイストークンをローテーションまたは取り消す必要があります
summary: '`openclaw devices` の CLI リファレンス（デバイスのペアリングとトークンのローテーション／失効）'
title: デバイス
x-i18n:
    generated_at: "2026-07-26T09:29:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fb10f7a484fec06bfa5e53ae50181b12a9724746176bbace330ec468235494
    source_path: cli/devices.md
    workflow: 16
---

# `openclaw devices`

デバイスのペアリング要求とデバイススコープのトークンを管理します。

## 共通オプション

- `--url <url>`: Gateway WebSocket URL（設定時のデフォルトは `gateway.remote.url`）
- `--token <token>`: Gateway トークン（必要な場合）
- `--password <password>`: Gateway パスワード（パスワード認証）
- `--timeout <ms>`: RPC タイムアウト
- `--json`: JSON 出力（スクリプトでの使用を推奨）

<Warning>
`--url` を設定すると、CLI は設定または環境の認証情報にフォールバックしません。`--token` または `--password` を明示的に渡してください。そうしないと、コマンドはエラーになります。
</Warning>

## コマンド

### `openclaw devices list`

保留中のペアリング要求とペアリング済みデバイスを一覧表示します。

```bash
openclaw devices list
openclaw devices list --json
```

すでにペアリング済みのデバイスに保留中の要求がある場合、出力には要求されたアクセス権がデバイスの現在承認済みアクセス権と並んで表示されるため、スコープやロールのアップグレードがペアリングの消失に見えることなく確認できます。

ペアリング済みデバイスの表示名には、次の優先順位が適用されます。オペレーターラベル（`devices rename` の `operatorLabel`）、クライアントの `displayName`、`clientId`、`deviceId` の順です。

### `openclaw devices approve [requestId] [--latest]`

正確な `requestId` を指定して、保留中のペアリング要求を承認します。`requestId` を省略するか、`--latest` を渡した場合は、最新の保留中の要求をプレビューして終了するだけです（終了コード 1）。承認するには、正確な要求 ID を指定して再実行してください。

```bash
openclaw devices approve
openclaw devices approve <requestId>
openclaw devices approve --latest
```

<Note>
デバイスが認証情報（ロール、スコープ、または公開鍵）を変更してペアリングを再試行すると、OpenClaw は以前の保留中エントリを新しい `requestId` で置き換えます。現在の ID を取得するには、承認の直前に `openclaw devices list` を実行してください。
</Note>

承認の動作：

- デバイスがすでにペアリング済みで、より広いスコープまたはロールを要求した場合、OpenClaw は既存の承認を維持し、新しい保留中のアップグレード要求を作成します。承認する前に、`openclaw devices list` で `Requested` と `Approved` を比較するか、`--latest` でプレビューしてください。
- `node` ロールまたはその他の非オペレーターロールを承認するには、`operator.admin` が必要です。オペレーターデバイスの承認には `operator.pairing` で十分ですが、要求されたオペレータースコープが呼び出し元自身のスコープ内に収まる場合に限ります。[オペレータースコープ](/ja-JP/gateway/operator-scopes)を参照してください。
- `gateway.nodes.pairing.autoApproveCidrs` が設定されている場合、一致するクライアント IP からの初回の `role: node` 要求は、この一覧に表示される前に自動承認されることがあります。デフォルトでは無効で、オペレーター／ブラウザークライアントやアップグレード要求には適用されません。
- `gateway.nodes.pairing.sshVerify`（デフォルトで有効）は、Gateway が SSH 経由で Node ホストのデバイス鍵を検証した場合、初回の `role: node` 要求を自動承認します。そのため、要求は表示された直後に承認済みになることがあります。SSH 検証を無効にするには `sshVerify: false` を設定してください。これは `autoApproveCidrs` とは独立しているため、手動のみのペアリングにするには、そちらも設定解除してください。

### `openclaw devices reject <requestId>`

保留中のデバイスペアリング要求を拒否します。

```bash
openclaw devices reject <requestId>
```

### `openclaw devices remove <deviceId>`

ペアリング済みデバイスのエントリを 1 件削除します。

```bash
openclaw devices remove <deviceId>
openclaw devices remove <deviceId> --json
```

ペアリング済みデバイストークンで認証された呼び出し元は、**自身の**デバイスエントリのみ削除できます。別のデバイスを削除するには `operator.admin` が必要です。

### `openclaw devices rename --device <id> --name <label>`

ペアリング済みデバイスにオペレーターラベルを割り当てます。ラベルは所有者側の状態です。ペアリングの修復やロールの再承認後も保持され、安定した `deviceId` は変更されません。

```bash
openclaw devices rename --device <deviceId> --name "Kitchen Mac"
openclaw devices rename --device <deviceId> --name "Kitchen Mac" --json
```

- `--name` は必須で、前後の空白が除去され、空ではなく、最大 64 文字に制限されます。
- 表示画面（CLI の一覧、Control UI のインベントリ）では、クライアントが報告した表示名よりオペレーターラベルが優先されます。
- 管理者ではないペアリング済みデバイスの呼び出し元は、**自身の**デバイスのみ名前を変更できます。別のデバイスの名前を変更するには `operator.admin` が必要です。

### `openclaw devices clear --yes [--pending]`

ペアリング済みデバイスを一括消去します。`--yes` によって保護されています。

```bash
openclaw devices clear --yes
openclaw devices clear --yes --pending
openclaw devices clear --yes --pending --json
```

`--pending` は、保留中のペアリング要求もすべて拒否します。

### `openclaw devices rotate --device <id> --role <role> [--scope <scope...>]`

ロールのデバイストークンをローテーションし、必要に応じてスコープを更新します。

```bash
openclaw devices rotate --device <deviceId> --role operator --scope operator.read --scope operator.write
```

- 対象ロールは、そのデバイスの承認済みペアリング契約にすでに存在している必要があります。ローテーションによって新しい未承認ロールを発行することはできません。
- `--scope` を省略すると、それ以降の再接続では保存済みトークンにキャッシュされた承認済みスコープが再利用されます。明示的な `--scope` 値を渡すと、以降のキャッシュ済みトークンによる再接続で使用される保存済みスコープセットが置き換えられます。
- 管理者ではないペアリング済みデバイスの呼び出し元は、**自身の**デバイストークンのみローテーションでき、対象スコープセットは呼び出し元自身のオペレータースコープ内に収まる必要があります。ローテーションによって、呼び出し元がすでに持つものより広いトークンを発行または維持することはできません。

ローテーションのメタデータを JSON として返します。呼び出し元がそのデバイストークンで認証中に自身のトークンをローテーションした場合、クライアントが再接続前に保存できるよう、応答に置換後のトークンが含まれます。共有／管理者によるローテーションでは、Bearer トークンが返されることはありません。

### `openclaw devices revoke --device <id> --role <role>`

ロールのデバイストークンを失効させます。

```bash
openclaw devices revoke --device <deviceId> --role node
```

管理者ではないペアリング済みデバイスの呼び出し元は、**自身の**デバイストークンのみ失効させることができます。別のデバイスのトークンを失効させるには `operator.admin` が必要です。また、対象スコープセットは呼び出し元自身のオペレータースコープ内に収まる必要があります。ペアリング専用の呼び出し元は、管理者／書き込みオペレータートークンを失効させることはできません。

## 注意事項

- これらのコマンドには、`operator.pairing`（または `operator.admin`）スコープが必要です。オペレーター以外のデバイスロールには常に `operator.admin` が必要です。[オペレータースコープ](/ja-JP/gateway/operator-scopes)を参照してください。
- トークンのローテーションと失効は、デバイスの承認済みペアリングロールセットおよびスコープの基準内に限定されます。孤立したキャッシュ済みトークンエントリによって、トークン管理の対象権限が付与されることはありません。
- ペアリング済みデバイストークンのセッションでは、デバイスをまたぐ管理（`remove`、`rename`、`rotate`、`revoke`）は、呼び出し元が `operator.admin` を持たない限り、自身のデバイスのみに制限されます。
- トークンのローテーションでは新しいトークン（機密情報）が返されます。シークレットとして扱ってください。
- local loopback でペアリングスコープが利用できず、明示的な `--url` が渡されていない場合、`list`/`approve` はローカルのペアリング状態にフォールバックできます。

## トークンの不整合からの復旧チェックリスト

Control UI またはその他のクライアントで `AUTH_TOKEN_MISMATCH`、`AUTH_DEVICE_TOKEN_MISMATCH`、または `AUTH_SCOPE_MISMATCH` による失敗が続く場合に使用してください。

1. 現在の Gateway トークンの取得元を確認します。

   ```bash
   openclaw config get gateway.auth.token
   ```

2. ペアリング済みデバイスを一覧表示し、影響を受けるデバイス ID を特定します。

   ```bash
   openclaw devices list
   ```

3. 影響を受けるデバイスのオペレータートークンをローテーションします。

   ```bash
   openclaw devices rotate --device <deviceId> --role operator
   ```

4. ローテーションで解決しない場合は、古いペアリングを削除して再度承認します。

   ```bash
   openclaw devices remove <deviceId>
   openclaw devices list
   openclaw devices approve <requestId>
   ```

5. 現在の共有トークン／パスワードを使用して、クライアント接続を再試行します。

注意事項：

- 通常の再接続時の認証優先順位は、明示的な共有トークン／パスワード、明示的な `deviceToken`、保存済みデバイストークン、ブートストラップトークンの順です。
- 信頼済みの `AUTH_TOKEN_MISMATCH` 復旧では、1 回に限定された再試行のため、共有トークンと保存済みデバイストークンの両方を一時的に送信できます。
- `AUTH_SCOPE_MISMATCH` は、デバイストークンは認識されたものの、要求されたスコープセットを保持していないことを意味します。共有 Gateway 認証を変更する前に、ペアリング／スコープ承認の契約を修正してください。

関連項目：

- [ダッシュボード認証のトラブルシューティング](/ja-JP/web/dashboard#if-you-see-unauthorized-1008)
- [Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting#dashboard-control-ui-connectivity)

## Paperclip / `openclaw_gateway` の初回実行時の承認

`openclaw_gateway` アダプターを介して接続する Paperclip エージェントには、他の新規クライアントと同じ初回実行時のデバイスペアリング承認が適用されます。Paperclip が `openclaw_gateway_pairing_required` を報告した場合は、保留中のデバイスを承認して再試行してください。

```bash
openclaw devices approve --latest
```

プレビューには正確な `openclaw devices approve <requestId>` コマンドが表示されます。詳細を確認してから、要求 ID を指定してそのコマンドを再実行し、承認してください。リモート Gateway または明示的な認証情報を使用する場合は、プレビュー時と承認時に同じオプションを渡します。

```bash
openclaw devices approve --latest --url <gateway-ws-url> --token <gateway-token>
```

再起動のたびに再承認する必要がないようにするには、実行ごとに新しい一時的なデバイス ID を生成させる代わりに、Paperclip で永続的な `adapterConfig.devicePrivateKeyPem` を設定します。

```json
{
  "adapterConfig": {
    "devicePrivateKeyPem": "<ed25519-private-key-pkcs8-pem>"
  }
}
```

承認に失敗し続ける場合は、まず `openclaw devices list` を実行して、保留中の要求が存在することを確認してください。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Node](/ja-JP/nodes)
