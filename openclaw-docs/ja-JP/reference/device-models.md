---
read_when:
    - デバイスモデル識別子のマッピングまたは NOTICE/ライセンスファイルの更新
    - Instances UI でのデバイス名の表示方法を変更する
summary: OpenClaw が macOS アプリでわかりやすい名前を表示するために Apple デバイスのモデル識別子をベンダリングする仕組み。
title: デバイスモデルデータベース
x-i18n:
    generated_at: "2026-07-26T09:16:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 930cd330594072d9c986b8c85c5a68e02dd096e5f0c015e3ee86b767073b93e6
    source_path: reference/device-models.md
    workflow: 16
---

macOS コンパニオンアプリの **Instances** UI は、Apple のモデル識別子をわかりやすい名前に対応付けます（`iPad16,6` -> 「iPad Pro 13-inch (M4)」、`Mac16,6` -> 「MacBook Pro (14-inch, 2024)」）。`DeviceModelCatalog` も識別子のプレフィックス（該当しない場合はデバイスファミリー）を使用して、デバイスごとに SF Symbol を選択します。

`apps/macos/Sources/OpenClaw/Resources/DeviceModels/` 内のファイル：

| ファイル                                   | 用途                               |
| -------------------------------------- | ------------------------------------- |
| `ios-device-identifiers.json`          | iOS/iPadOS 識別子 -> 名前のマッピング |
| `mac-device-identifiers.json`          | Mac 識別子 -> 名前のマッピング        |
| `NOTICE.md`                            | 固定されたアップストリームのコミット SHA           |
| `LICENSE.apple-device-identifiers.txt` | アップストリームの MIT ライセンス                  |

## データソース

MIT ライセンスの `kyle-seongwoo-jun/apple-device-identifiers` GitHub リポジトリからベンダリングされています。ビルドの再現性を保つため、JSON ファイルは `NOTICE.md` に記録されたコミット SHA に固定されています。

## データベースの更新

1. 固定するアップストリームのコミット SHA を選択します（iOS 用と macOS 用に 1 つずつ）。
2. `apps/macos/Sources/OpenClaw/Resources/DeviceModels/NOTICE.md` を新しい SHA で更新します。
3. それらのコミットに固定された JSON ファイルを再ダウンロードします：

```bash
IOS_COMMIT="<commit sha for ios-device-identifiers.json>"
MAC_COMMIT="<commit sha for mac-device-identifiers.json>"

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${IOS_COMMIT}/ios-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/ios-device-identifiers.json

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${MAC_COMMIT}/mac-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/mac-device-identifiers.json
```

4. `LICENSE.apple-device-identifiers.txt` が引き続きアップストリームと一致することを確認します。アップストリームのライセンスが変更された場合は置き換えます。
5. macOS アプリがエラーなくビルドされることを確認します：

```bash
swift build --package-path apps/macos
```

## 関連項目

- [Node](/ja-JP/nodes)
- [Node のトラブルシューティング](/ja-JP/nodes/troubleshooting)
