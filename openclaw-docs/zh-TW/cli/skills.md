---
read_when:
    - 你想查看有哪些 Skills 可用且已準備好執行
    - 你想要搜尋 ClawHub，或從 ClawHub、Git 或本機目錄安裝 Skills
    - 你想使用 ClawHub 驗證 ClawHub Skill
    - 你想要偵錯 Skills 缺少的二進位檔、環境變數或設定
summary: '`openclaw skills` 的命令列介面參考（搜尋/安裝/更新/驗證/列出/資訊/檢查/工作坊）'
title: Skills
x-i18n:
    generated_at: "2026-07-26T07:47:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3eafd40704b666e6be185aa8148b60613c861a2899fb9b0cc3353212e8e4d678
    source_path: cli/skills.md
    workflow: 16
---

# `openclaw skills`

檢查本機 Skills、搜尋 ClawHub、從 ClawHub/Git/本機目錄安裝 Skills、驗證 ClawHub Skills，以及更新由 ClawHub 追蹤的安裝項目。

相關內容：

- Skills 系統：[Skills](/zh-TW/tools/skills)
- Skill Workshop：[Skill Workshop](/zh-TW/tools/skill-workshop)
- Skills 設定：[Skills 設定](/zh-TW/tools/skills-config)
- ClawHub 安裝：[ClawHub](/zh-TW/clawhub/cli)

## 命令

```bash
openclaw skills search "calendar"
openclaw skills search --limit 20 --json
openclaw skills install @owner/<slug>
openclaw skills install @owner/<slug> --version <version>
openclaw skills install git:owner/repo
openclaw skills install git:owner/repo@main
openclaw skills install ./path/to/skill --as custom-name
openclaw skills install @owner/<slug> --force
openclaw skills install @owner/<slug> --force-install
openclaw skills install @owner/<slug> --acknowledge-clawhub-risk
openclaw skills install @owner/<slug> --agent <id>
openclaw skills install @owner/<slug> --global
openclaw skills update @owner/<slug>
openclaw skills update @owner/<slug> --force-install
openclaw skills update @owner/<slug> --acknowledge-clawhub-risk
openclaw skills update @owner/<slug> --global
openclaw skills update --all
openclaw skills update --all --agent <id>
openclaw skills update --all --global
openclaw skills verify @owner/<slug>
openclaw skills verify @owner/<slug> --version <version>
openclaw skills verify @owner/<slug> --tag <tag>
openclaw skills verify @owner/<slug> --card
openclaw skills verify @owner/<slug> --global
openclaw skills list
openclaw skills list --eligible
openclaw skills list --json
openclaw skills list --verbose
openclaw skills list --agent <id>
openclaw skills info <name>
openclaw skills info <name> --json
openclaw skills info <name> --agent <id>
openclaw skills check
openclaw skills check --agent <id>
openclaw skills check --json
openclaw skills workshop propose-create --name "qa-check" --description "QA checklist" --proposal ./PROPOSAL.md
openclaw skills workshop propose-update qa-check --proposal ./PROPOSAL.md
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "Not reusable"
openclaw skills workshop quarantine <proposal-id> --reason "Needs security review"
```

`search`、`update` 和 `verify` 會直接使用 ClawHub。`install @owner/<slug>` 會安裝 ClawHub Skill，`install git:owner/repo[@ref]` 會複製 Git Skill，而 `install ./path` 會複製本機 Skill 目錄。依預設，`install`、`update` 和 `verify` 會以作用中工作區的 `skills/` 目錄為目標；搭配 `--global` 時，則會以共用的受管理 Skills 目錄為目標。`list`/`info`/`check` 仍會檢查目前工作區和設定可見的本機 Skills。
以工作區為基礎的命令會先從 `--agent <id>` 解析目標工作區，接著在目前工作目錄位於已設定的代理程式工作區內時使用該目錄，最後才使用預設代理程式。

Git 和本機目錄安裝要求來源根目錄包含 `SKILL.md`。若 `SKILL.md` frontmatter 中的 `name` 有效，安裝代稱會優先取自該值，其次使用來源目錄或儲存庫名稱；可使用 `--as <slug>` 覆寫。`--version` 僅適用於 ClawHub。Skill 安裝不支援 npm 套件規格或 zip/封存檔路徑，而 `openclaw skills update` 僅會更新由 ClawHub 追蹤的安裝項目。

從新手引導或 Skills 設定觸發、由閘道支援的 Skill 相依性安裝，會改用獨立的 `skills.install` 請求路徑。

注意事項：

| 旗標/行為                    | 說明                                                                                                                                                                                                                                                                       |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `search [query...]`              | 選用查詢；省略即可瀏覽預設的 ClawHub 搜尋摘要。                                                                                                                                                                                                                |
| `search --limit <n>`             | 限制傳回的結果數量。                                                                                                                                                                                                                                                            |
| `install git:owner/repo[@ref]`   | 安裝 Git Skill。分支參照可以包含斜線，例如 `git:owner/repo@feature/foo`。                                                                                                                                                                                      |
| `install ./path/to/skill`        | 安裝根目錄包含 `SKILL.md` 的本機目錄。                                                                                                                                                                                                                        |
| `install --as <slug>`            | 覆寫為 Git 和本機目錄安裝推斷出的代稱。                                                                                                                                                                                                                 |
| `install --version <version>`    | 僅套用至 ClawHub Skill 參照。                                                                                                                                                                                                                                               |
| `install --force`                | 覆寫相同代稱的現有工作區 Skill 資料夾。                                                                                                                                                                                                                  |
| `install/update --force-install` | 在 ClawHub 掃描完成前，安裝待處理且以 GitHub 為基礎的 ClawHub Skill。                                                                                                                                                                                                   |
| `--global`                       | 以共用的受管理 Skills 目錄為目標；無法與 `--agent <id>` 搭配使用。                                                                                                                                                                                                  |
| `--agent <id>`                   | 以一個已設定的代理程式工作區為目標；覆寫目前工作目錄的推斷結果。                                                                                                                                                                                            |
| `update @owner/<slug>`           | 更新單一受追蹤的 Skill。加入 `--global`，可改以共用的受管理 Skills 目錄為目標，而非工作區。                                                                                                                                                            |
| `update --all`                   | 更新所選工作區中受追蹤的 ClawHub 安裝項目；搭配 `--global` 時，則更新共用的受管理 Skills 目錄。                                                                                                                                                               |
| `verify @owner/<slug>`           | 預設印出 ClawHub 的 `clawhub.skill.verify.v1` JSON 封套。由於 JSON 已是預設格式，因此沒有 `--json` 旗標。當 Skill 已安裝或不存在歧義時，為了相容性也接受不含擁有者的代稱；包含擁有者的完整參照可避免發布者歧義。 |
| `verify` 來源資訊              | 當 ClawHub 傳回由伺服器解析的來源資訊時，驗證 JSON 也會包含鎖定至提交版本的 `openclaw.verifiedSourceUrl`。無法取得或自行宣告的來源 URL 只會保留在原始來源資訊封套中，不會提升使用。                                           |
| `verify` 版本選擇器        | 對於已安裝的 ClawHub Skills，`verify` 會使用 `.clawhub/origin.json`，因此會根據其來源登錄檔驗證已安裝版本。`--version` 和 `--tag` 會覆寫版本選擇器，但存在來源中繼資料時仍會保留該已安裝項目所用的登錄檔。                    |
| `verify --card`                  | 印出產生的 Skill Card Markdown，而非 JSON。當 ClawHub 傳回 `ok: false` 或 `decision: "fail"` 時，會以非零狀態結束；除非 ClawHub 政策變更，否則未簽署的簽章僅供參考。                                                                             |
| Skill Card 指紋           | 已安裝的 ClawHub 套件組合可包含產生的 `skill-card.md`。OpenClaw 將驗證視為 ClawHub 伺服器的決策，不會只因產生的卡片變更了套件組合指紋，就拒絕已安裝的 Skill。                                              |
| `check --agent <id>`             | 檢查所選代理程式的工作區，並回報哪些已就緒的 Skills 實際可由該代理程式的提示詞或命令介面使用。                                                                                                                                              |
| `list`                           | 未提供子命令時的預設動作。                                                                                                                                                                                                                                    |
| `list`/`info`/`check` 輸出     | 轉譯後的輸出會送至 stdout。搭配 `--json` 時，機器可讀取的承載內容仍會保留在 stdout，以供管線和指令碼使用。                                                                                                                                                                |

社群 ClawHub Skill 的安裝與更新會在下載前檢查信任狀態。具版本的社群封存發行版會使用精確發行版的信任中繼資料。由解析器支援、以 GitHub 為基礎的 Skills，會依賴 ClawHub 的安裝解析器，在傳回鎖定的提交版本前強制執行掃描和強制安裝政策；若要在掃描完成前安裝待處理且以 GitHub 為基礎的 Skill，請使用 `--force-install`。惡意或遭封鎖的社群發行版會被拒絕。有風險的社群發行版需要經過審查；若非互動式命令要在審查後繼續執行，還需要 `--acknowledge-clawhub-risk`。ClawHub 官方 Skill 發布者及 OpenClaw 內建 Skill 來源會略過此發行版信任提示。

## Skill Workshop

`openclaw skills workshop` 管理所選工作區中待處理的 skill 提案。
提案在套用前不會成為啟用中的 skill。如需瞭解提案儲存、支援檔案保護措施、閘道方法與核准政策，請參閱
[Skill 工作坊](/zh-TW/tools/skill-workshop)。

```bash
openclaw skills workshop propose-create \
  --name "qa-check" \
  --description "可重複使用的 QA 檢查清單" \
  --proposal ./PROPOSAL.md
openclaw skills workshop propose-create \
  --name "qa-check" \
  --description "可重複使用的 QA 檢查清單" \
  --proposal-dir ./qa-check-proposal
openclaw skills workshop propose-update qa-check --proposal ./PROPOSAL.md
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "重複"
openclaw skills workshop quarantine <proposal-id> --reason "需要安全性審查"
```

`propose-create`、`propose-update` 和 `revise` 也接受 `--goal <text>`
與 `--evidence <text>`，用於在 `--proposal`/`--proposal-dir` 內容旁記錄提案的動機與佐證
備註。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [Skills](/zh-TW/tools/skills)
