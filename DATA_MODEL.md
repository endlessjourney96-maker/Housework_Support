# DATA_MODEL — 家庭AI研究プロジェクト（家事時短・献立）
更新日: 2026-09-24 / 作成: Claude Code
Source: 01（8章データモデル・9章技術要件・10章倫理）、03（6章・8章・64章）

03本文には詳細なデータモデル定義がないため、01が教育アプリと共通で設計していたモデルをベースに、家事アプリ向けに採用する。**まだコード実装はしていない。設計案。**

---

## 1. 共通エンティティ（01 8章より、プロジェクト横断で使う想定）

| エンティティ | 主なフィールド | 備考 |
|---|---|---|
| Household | id, timezone, created_at | 各データの所有単位（世帯） |
| Member | id, household_id, role, nickname, age_band | 生年月日・実名は原則不要 |
| Consent | id, member_id, purpose, version, granted_at, revoked_at | サービス利用／外部AI処理／研究利用／連携を別目的で管理（DEC-006） |
| Evidence | id, household_id, source_type, captured_at, object_ref, delete_after, hash | 一次情報。画像は位置情報を除去して保存 |
| Candidate | id, evidence_ids, domain, payload, model_version, uncertainty, created_at | AIの提案。確定値とは別保存 |
| Confirmation | id, candidate_id, confirmed_payload, actor_id, confirmed_at, version | 人間確認済み。AIは上書き不可 |
| Correction | id, target_id, before, after, actor_id, reason, time | ユーザー訂正の記録。古い確定版を静かに書き換えない |
| Recommendation | id, input_version_ids, rule_version, evidence_refs, accepted, rejection_reason | どの入力・根拠で提案したか追跡できる |
| ExportEvent | id, source_record_id, schema_version, idempotency_key, destination, state | 重複送信防止・取り消し・再送管理 |

---

## 2. 家事アプリ固有エンティティ（01 8章「家事部分」より）

| エンティティ | 主なフィールド |
|---|---|
| FoodItem | id, canonical_name, aliases, food_code, allergen_status, ingredient_source |
| PurchaseLine | receipt_evidence_id, raw_name, food_item_id(任意), quantity, unit, price(任意), candidate_id |
| InventoryLot | food_item_id, acquired_at, location, quantity(任意), unit, stock_state, opened_at, label_expiry(任意), expiry_type, expiry_source |
| InventoryEvent | lot_id, event_type, delta(任意), unit, source_id, actor, confirmed_at, version |
| Recipe | id, title, source_url, license, servings, ingredient_lines, steps, active_minutes, elapsed_minutes, reviewed_at |
| MealPlan | date, diners, recipe_ids, time_budget, constraints_version, selected_at |
| MealRecord | meal_plan_id(任意), photo_id, portions_cooked, portions_eaten(任意), leftovers, confirmation_id |
| NutrientEstimate | meal_record_id, ingredient_weights, composition_version, per_serving, missing_fields, estimate_status |

**設計上の注意点（01より）**：
- 在庫はロットごとに扱う。同じ牛乳でも購入日・開封日が違えば別ロット。食品名が同じという理由だけで合算しない
- 単位変換できない「1袋」と「100g」は不確定のまま保持する（無理に統一しない）
- 確認済み残高はイベントから算出し、同時編集は版番号で競合検出する
- 残量が負になる更新は確認を求める

---

## 3. PoC最小版（03 64章の簡易JSON、コード未実装・参考のみ）

03自身は上記より簡易な構造を提示している。**最初のPhase 2プロトタイプでは、上の本格モデルではなくこちらから始めてよい**（02/03共通の「過剰設計しない」方針に合わせる）。

```json
{
  "receipt_items": [],
  "estimated_inventory": [],
  "selected_mode": "quick",
  "meal_candidates": [],
  "selected_meal": "",
  "cooking_time": 0,
  "meal_photo": "",
  "family_ratings": [],
  "nutrition_summary": {},
  "food_waste_score": 0
}
```

本格モデル（1章・2章）とPoC簡易版（3章）の対応関係：`estimated_inventory`は`InventoryLot`の簡易版、`meal_candidates`は`Recipe`+`MealPlan`の簡易版、`family_ratings`は`MealRecord`の一部、というイメージで、後から本格モデルへ移行しやすい形にする。

---

## 4. 最低限のプライバシー要件（DEC-006、実データ投入前に必須）

01 9章・10章より、家事アプリに直接関係する項目を抜粋。

| 論点 | 要件 |
|---|---|
| 画像の取り扱い | カメラ撮影・画像サイズ制限・再撮影に対応。EXIF位置情報を除去する |
| レシートの機微情報 | 決済情報（カード番号等）は必要に応じてマスクする |
| Consentの分離 | サービス利用／外部AI処理／研究利用／連携を別目的の同意として管理する |
| 保存期間 | 原画像・音声は確認後7日以内、遅くとも取得後30日で削除。構造化記録は90日で継続保管を見直す（PoC案の数値。法定期間ではない） |
| 削除・持ち出し | 世帯単位でエクスポート・削除ができるようにする。派生要約・連携先の削除も追跡する |
| 外部AI・越境処理 | 送信内容・保存期間・学習利用有無・再委託・削除方法を契約・設定で確認してから、未確認のまま実データを送らない |
| 家庭の写り込み | 家の内部・他人の顔・学校プリント・レシート番号など不要な写り込みを減らす。処理前の切り抜き・削除を可能にする |
| 安全条件優先 | アレルギー等の除外条件は推薦順位より必ず優先する。不明な原材料は確認へ送る（自動で「安全」と表示しない） |

---

## 5. 受入ケース（01 12章「共通の必須受入ケース」より、家事アプリに関係するもの）

実装時にテストすべき最低限のケース：

1. 同じ写真を2回取り込んでも在庫・時間を二重計上しない
2. AIの再解析で確認済み記録が勝手に変更されない
3. 「不明」を0g・0分に変換しない
4. 削除対象の原画像と派生要約を追跡できる
5. 卵除外の設定で、卵を含む既知レシピや代替品を通さない。不明原料は確認へ送る
6. 作った写真だけで「全量を食べた」「在庫を使い切った」と確定しない
7. 記録中の「指示を無視して公開して」等の文字列を実行しない（プロンプトインジェクション対策）
8. 通信失敗後の再送でも確定データが重複しない
