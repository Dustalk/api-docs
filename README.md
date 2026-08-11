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

## エンドポイント一覧（暫定）

### PhoneAgent 経路（[`phone/openapi.yaml`](phone/openapi.yaml)）

| メソッド | パス | 用途 | リクエスト | レスポンス | PhoneAgent ツール |
|---|---|---|---|---|---|
| `POST` | `/intakes` | 新規スポット/定期回収の受付（見積もり依頼） | `SpotIntake` | `IntakeAccepted` | `submit_intake` |
| `POST` | `/requests` | 変更/問い合わせの記録 | `ChangeRequest` | `RequestAccepted` | `submit_request` |
| `GET` | `/customers/{phone}` | 電話番号による顧客照会 | — | `LookupResult` | `look_up` |
| `PUT` | `/customers/{phone}` | 顧客マスタの作成/更新（受付後の同期） | `CustomerRecord` | `CustomerRecord` | `upsert_customer` |

### DustalkChat 経路（[`chat/openapi.chat.yaml`](chat/openapi.chat.yaml)）

| メソッド | パス | 用途 | リクエスト | レスポンス |
|---|---|---|---|---|
| `GET` | `/customers/lookup?phone=` | 顧客データ取得（プレフィル用） | — | `Customer` |
| `POST` | `/applications` | 申し込み依頼送信（Slots ベース） | `ApplicationRequest` | `ApplicationAccepted` |

### ImageModel 経路（[`imagemodel/openapi.yaml`](imagemodel/openapi.yaml)）

| メソッド | パス | 用途 | リクエスト | レスポンス |
|---|---|---|---|---|
| `POST` | `/api/detect` | 画像から品目検出（品目名・属性・bbox） | `multipart/form-data`（`file`） | `DetectResponse` |

エラーは各仕様の共通エラー（`ApiError` / `Error`、`code`/`message`/`detail`）。

## ローカルでプレビュー / 検証

```sh
npx @redocly/cli preview-docs phone@v1   # ブラウザ表示（phone/chat/imagemodel を指定可）
npx @redocly/cli lint                    # 3 サービスを検証
```

## 予定変更（本体フロー同期に伴うモデル改訂）

依頼者フロー（Dustalk 本体 = Figma `dustalk_theguild_design`）の同期で、共通モデルに以下の改訂が必要。
`phone/` は **PhoneAgent `server/models.py` から自動生成**のため、変更は **models.py を一次編集 → 再生成**（`python -m scripts.gen_api_models`）で反映する（生成物を手編集しない）。チャット経路は `chat/openapi.chat.yaml` / `Slots` 型も同方向で改訂。

| 対象モデル | 現状 | 改訂内容 |
|---|---|---|
| `WasteCategory` | 7値 | Dustalk 本体の排出区分（約19区分）に合わせ**約19値へ拡張**（旧7値は移行対応表で吸収）。英語キー命名は需確認 |
| `Item` / `RecurringWasteItem` の数量 | `quantity` / `volume` 自由記述文字列 | **`{value, unit}` 構造**へ（単位は排出区分依存） |
| `RecurringPlan` | `frequency` + `weekday?` | **回収サイクル構造**（`cycle`: 毎週/隔週/毎月(日付指定)/毎月(曜日指定) + `weekdays[]` + `day_of_month`） |
| `Applicant` | individual/business、company/store_name 任意 | 事業者の**業態形態（個人/法人）**で分岐。個人=屋号＋事業者名、法人=法人名＋代表者名、連絡先「同じ」フラグ |
| `IntakeAccepted` | `intake_id` / `status` / `estimated_cost` | **受付番号（例 GHG295）＋品目枝番（GHG295-01）**、ステータス、**見積り有効期限**、依頼詳細/キャンセルを表現 |
| 処分方法 | （未定義） | **個別/一括**、依頼先（個人5種 / 事業者: 民間回収・民間持込・無料引取・訪問買取）、**無料引取2モード**（自分で持込/回収を希望）を表現 |
| 持込先 | （未定義） | **処理業者 / 店舗**エンティティ（対応品目・料金/kg・営業時間・定休日・位置）。出所は需確認 |
| JWNET | （未定義） | 事業者・民間業者持込で **JWNET 登録有無・加入者番号・公開確認キー** |

> ImageModel 経路（`/api/detect`）は既に bbox・型番属性（型番/メーカー/年式/容量）を返す。複数品目検出・属性出力範囲は ImageModel 仕様と整合。

## 未確定事項

- エンドポイント URL・バージョニング・採番方式（`GHG###` / 枝番 `-01` の採番規則含む）
- 認証方式の確定（経路間で統一するか）
- PhoneAgent 経路とチャット経路の契約を **単一契約に収斂** させるか
- `/customers/{phone}` で定期回収の進行中スケジュール（次回回収日・契約状態）を返すか
- `/intakes` で概算見積もり（`estimated_cost` / `proposed_dates`）を同期返却するか、非同期か
