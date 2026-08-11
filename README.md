# dustalk-api — Dustalk 連携 API 仕様

PhoneAgent（電話）・DustalkChat（チャット）・ImageModel（画像認識）が **Dustalk 本体**や
相互に対して呼ぶ API 契約（OpenAPI）を**集約・公開**するリポジトリです。実装コードは含みません。

> **契約の正本は各 OpenAPI ファイル**（機械可読）。エンドポイント・認証は **暫定**
> （Dustalk バックエンド確定後に更新）。ImageModel は**実装済みの稼働サービス**。

## 公開ドキュメント（URL）

`main` へ push すると GitHub Actions が Redoc をビルドし GitHub Pages に公開します。

- 一覧: https://developerunion.github.io/dustalk-api/
- 電話: https://developerunion.github.io/dustalk-api/phone/
- チャット: https://developerunion.github.io/dustalk-api/chat/
- 画像認識: https://developerunion.github.io/dustalk-api/imagemodel/

> 初回のみ: **Settings → Pages → Source = GitHub Actions** を有効化。

## 構成

```
.
├── README.md                     ← このファイル（索引）
├── redocly.yaml                  ← 3 サービスの定義（lint / build-docs）
├── .github/workflows/pages.yml   ← Redoc ビルド → Pages 公開
├── phone/                        ← PhoneAgent（電話）→ Dustalk 本体（暫定・自動生成）
│   ├── openapi.yaml              ← OpenAPI 3.1（paths へ $ref）
│   ├── paths/{intakes,requests,customers}.yaml
│   └── components/schemas.yaml
├── chat/                         ← DustalkChat（チャット）→ Dustalk 本体（暫定）
│   └── openapi.chat.yaml         ← OpenAPI 3.0.3
└── imagemodel/                   ← DustalkChat → ImageModel（画像認識・実装済）
    └── openapi.yaml              ← OpenAPI 3.0.3
```

## サービス一覧

| 経路 | 仕様 | 状態 | 真実源 / 管理 |
|---|---|---|---|
| **PhoneAgent（電話）** → Dustalk 本体 | [`phone/openapi.yaml`](phone/openapi.yaml)（3.1） | 暫定 | `PhoneAgent/phone-agent/server/models.py`（Pydantic）から自動生成。`python -m scripts.gen_api_models`。手動編集不可 |
| **DustalkChat（チャット）** → Dustalk 本体 | [`chat/openapi.chat.yaml`](chat/openapi.chat.yaml)（3.0.3） | 暫定 | `DustalkChat/dustalk-chat/src/lib/slots/types.ts` の `Slots` 型を起点に手動 |
| **DustalkChat** → ImageModel（画像認識） | [`imagemodel/openapi.yaml`](imagemodel/openapi.yaml)（3.0.3） | 実装済 | `ImageModel/image-model` 実装を起点に手動 |

PhoneAgent 経路と DustalkChat 経路は同一ドメイン（回収申し込み）の別表現。チャット経路の方が
情報量が多い（品目ごとの依頼先・希望時間帯・構造化住所）。**将来は単一契約への収斂を想定**。

> 補足ドキュメント（API契約ではないため本リポには含めない）:
> 暫定 DB 設計（`db-schema-provisional.md`）は PhoneAgent リポの `docs/` にあります。
> モデルのフィールド表は別途生成せず、Redoc が `phone/components/schemas.yaml` を描画します。

### 電話経路の更新フロー

`phone/` は `PhoneAgent/phone-agent/server/models.py` から**直接生成**されます（コピーではない）。
モデルを変えたら PhoneAgent リポで生成器を実行 → このリポを commit & push してください。

```sh
# PhoneAgent リポにて（出力先は <work>/dustalk/Platform/docs/api/phone/）
.venv/bin/python -m scripts.gen_api_models
.venv/bin/python -m scripts.gen_api_models --check   # 差分（再生成漏れ）検出

# dustalk-api リポにて
git add phone && git commit -m "chore: regenerate phone OpenAPI" && git push
```

## 認証（暫定）

- PhoneAgent（`phone/openapi.yaml`）: API キー認証 `X-API-Key: <key>`（暫定・未確定）。
- DustalkChat → Dustalk 本体（`chat/openapi.chat.yaml`）: Bearer トークン（暫定・未確定）。
- DustalkChat → ImageModel（`imagemodel/openapi.yaml`）: **認証なし**（パブリック、CORS で Origin 制限）。

## エンドポイント一覧

**再掲しない。** 各 OpenAPI ファイルが正本なので、そちらを直接読むこと
（Redoc の公開 URL は上記「公開ドキュメント」）。

| 経路 | エンドポイントの定義 |
|---|---|
| PhoneAgent | [`phone/openapi.yaml`](phone/openapi.yaml) → `paths/{intakes,requests,customers}.yaml` |
| DustalkChat | [`chat/openapi.chat.yaml`](chat/openapi.chat.yaml) |
| ImageModel | [`imagemodel/openapi.yaml`](imagemodel/openapi.yaml) |

PhoneAgent 経路の各エンドポイントは `submit_intake` / `submit_request` / `look_up` /
`upsert_customer` の各エージェントツールに対応する（[specs/phone-agent.md](../specs/phone-agent.md)）。

エラーは各仕様の共通エラー（`ApiError` / `Error`、`code`/`message`/`detail`）。

## ローカルでプレビュー / 検証

```sh
npx @redocly/cli preview-docs phone@v1   # ブラウザ表示（phone/chat/imagemodel を指定可）
npx @redocly/cli lint                    # 3 サービスを検証
```

## 未実装要求リスト（本体フロー同期に伴うモデル改訂）

> **契約が正**（上の「契約の正本は各 OpenAPI ファイル」）。
> この節は「本体フローはこうなっているが、実装も契約もまだ追いついていない」項目の一覧であり、
> **現行の契約を上書きするものではない**。実装が変わったら契約を再生成／改訂し、
> ここから項目を消す。

依頼者フロー（Dustalk 本体 = Figma `dustalk_theguild_design`）の同期で、共通モデルに以下の改訂が必要。
`phone/` は **`PhoneAgent/phone-agent/server/models.py` から自動生成**のため、変更は
**models.py を一次編集 → 再生成**（`python -m scripts.gen_api_models`）で反映する（生成物を手編集しない）。
チャット経路は `chat/openapi.chat.yaml` / `Slots` 型も同方向で改訂。

| 対象モデル | 現状（契約・実装） | あるべき姿 |
|---|---|---|
| `WasteCategory` | 7値 | Dustalk 本体の排出区分（約19区分）に合わせ**約19値へ拡張**（旧7値は[移行対応表](../glossary.md)で吸収）。英語キー命名は需確認 |
| `Item` / `RecurringWasteItem` の数量 | `quantity` / `volume` 自由記述文字列 | **`{value, unit}` 構造**へ（単位は排出区分依存） |
| `RecurringPlan` | `frequency` + `weekday?`（単数） | **回収サイクル構造**（`cycle`: 毎週/隔週/毎月(日付指定)/毎月(曜日指定) + `weekdays[]` + `day_of_month`） |
| `BusinessForm` / `Applicant` | 旧4値（個人事業主/株式会社/有限会社/その他法人） | 事業者の**業態形態（個人/法人）の2値**で分岐。個人=屋号＋事業者名、法人=法人名＋代表者名、連絡先「同じ」フラグ |
| `IntakeAccepted` | `intake_id` / `status` / `estimated_cost` | **受付番号（例 `GHG295`）＋品目枝番（`GHG295-01`）**、ステータス、**見積り有効期限**、依頼詳細/キャンセルを表現 |
| `ProviderChoice` | 個人・事業者共通の5値（無料引取 / 自治体に依頼 / 訪問買取 / ネット買取 / 民間事業者に依頼） | **ユーザー種別で分ける**（個人5種 / 事業者: 民間事業者回収・**民間事業者持込**・無料引取・訪問買取）。**個別/一括**、**無料引取2モード**（自分で持込/回収を希望）を表現 |
| 持込先 | （未定義） | **処理業者 / 店舗**エンティティ（対応品目・料金/kg・営業時間・定休日・位置）。出所は需確認（[specs/scraping.md](../specs/scraping.md)） |
| JWNET | （未定義） | 事業者・民間事業者持込で **JWNET 登録有無・加入者番号・公開確認キー** |
| `Location` | `floor` / `dischargeMode` あり | 事業者固有の**ゴミの保管場所（店舗内/店舗外）**が無い |

> ImageModel 経路（`/api/detect`）は 2 モード（`general` / `industrial`）・bbox・型番属性・
> `retakeSuggestions`・`additional` による複数枚統合まで契約に反映済み
> （[specs/image-model.md](../specs/image-model.md)）。
> 残る乖離は `items[].name` が単数で「品目候補（複数）」を表現できない点。

## 未確定事項

- エンドポイント URL・バージョニング・採番方式（`GHG###` / 枝番 `-01` の採番規則含む）
- 認証方式の確定（経路間で統一するか）
- PhoneAgent 経路とチャット経路の契約を **単一契約に収斂** させるか
- `/customers/{phone}` で定期回収の進行中スケジュール（次回回収日・契約状態）を返すか
- `/intakes` で概算見積もり（`estimated_cost` / `proposed_dates`）を同期返却するか、非同期か
