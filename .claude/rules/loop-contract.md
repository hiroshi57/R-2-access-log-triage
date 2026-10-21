# Loop Contract ルール（Lead が Worker に出す次の一手）

> **適用対象**: Lead（breezing mode で Worker を導く役）。solo / codex mode では、自分で次の回を決めるときに同じ形式を使ってよい（SHOULD）
> **起源**: 提案 `harness-proposals/2026-09-30-loop-contract-advance-verify-stop.md`（提案 A・B）／根拠 arXiv:2608.28281（§2.2・§5.2・付録 C.1）
> **承認**: 2026-09-30 人間承認済み（P-1〜P-4 は既定案で確定）

## ルール1: 各回の後に action を1つ選ぶ（MUST）

Worker の各回（`worker-report.<n>.json` と `review.<n>.json` がそろった時点）の後、Lead は次の4つから1つを選ぶ。

| action | 選ぶ条件 | Worker にさせること |
|---|---|---|
| `advance` | 実装が足りない、または REQUEST_CHANGES の欠陥（`named_defects`）を直す | 次の実装の1区切り |
| `verify` | 実装は済んでいそうだが、判断に要る証拠が足りない | 検証コマンドの実行と出力の提出だけ。実装ファイルは変えない |
| `stop` | DoD の全項目に外部証跡（`evidence_ref`）があり、verdict が APPROVE | なし（終了） |
| `escalate` | 安全に進める方法がない、同じ原因の失敗が続く（decision-boundaries ルール1）、`route: human` | なし（人間へ） |

- 迷ったら、足りない情報で判断が変わるかを問う。変わるなら `verify` を選ぶ。
- 証拠がないことを、成功とも失敗とも決めつけない。
- `open_issues`（worker-report の未解決項目、review の `uncertain: true`）が DoD に関わるなら、`stop` を選ばない。
  前回の指示で扱わなかったことを理由に無視しない。
- 元のタスク・DoD にない要件を、次の指示で足さない。
- `verify` の回は、decision-boundaries ルール1の「同じ原因の自動修正（最大3回）」に数えない。
  ただし `verify` が2回続いても新しい証拠が出なければ、次は `escalate` を選ぶ（P-1）。

## ルール2: 判断を `loop-contract.<n>.json` に残す（MUST）

`<n>` は判断の対象になった `worker-report.<n>.json` と同じ番号。判断のたびに上書きせず残す。

```json
{
  "schema_version": "loop-contract.v1",
  "task_id": "43.3.1",
  "after_round": 1,
  "action": "advance | verify | stop | escalate",
  "rationale": "判断の理由。worker-report / review / commands.stdout.log の該当箇所を指して書く",
  "evidence_refs": ["review.1.json#scores[3]", "commands.stdout.log:L120-L134"],
  "worker_instruction": {
    "goal": "次の1回で達成する単一の成果",
    "context": "背景と、どこまでが確認済みか",
    "required_outcomes": ["この回で得るべき結果・証拠"],
    "prohibited_actions": ["この回でやってはいけないこと"],
    "completion_condition": "いつ手を止めて報告するか"
  },
  "protected_invariants": ["この回の間ずっと守るべき振る舞い・制約"],
  "verification_acceptance_condition": "この回の goal を達成したと言える外部証跡"
}
```

- `stop` / `escalate` のときは `worker_instruction`・`protected_invariants`・`verification_acceptance_condition` を書かない。
- `rationale` は証拠の場所を指して書く。「問題なさそう」「おそらく十分」だけの理由は書かない（decision-boundaries ルール5）。
- `review.json` の `verdict` は2値のまま変えない。`loop-contract.<n>.json` は verdict の後の「次の一手」を残す別ファイル。

## ルール3: 7項目の書き方（MUST）

| 項目 | 書くこと | 書かないこと |
|---|---|---|
| `goal` | 次の1回で達成する**1つ**の成果 | タスク全体の目標の言い直し |
| `context` | 背景、確認済みのこと、まだ確認していないこと | 「たぶん大丈夫」などの結論 |
| `required_outcomes` | この回で得るべき結果・証拠（出力を提出すべき検証コマンドを含む） | やり方の好み |
| `prohibited_actions` | この回でやってはいけないこと（否定形だけ） | やるべきこと・報告の仕方（それは `required_outcomes` へ） |
| `completion_condition` | 手を止めて Lead に返す条件（観測できる形で） | 「完成したら」だけ |
| `protected_invariants` | この回の間ずっと成り立つべき振る舞い（既存テストが通る、API の形を変えない など） | この回で新しく作る成果 |
| `verification_acceptance_condition` | goal を達成したと言える外部証跡（どのコマンドのどの結果か） | Worker の `self_review.verified` だけ（judge-rubric ルール4） |

- `goal`・`completion_condition`・`verification_acceptance_condition` は空にせず、同じ1つの仕事について書く。
- `verify` の回は、`prohibited_actions` に「`files` のうちテスト以外のファイルを変更しない」を必ず入れる。
- sprint-contract がある場合、`verification_acceptance_condition` は `definition_of_done` の項目から選び、`prohibited_actions` には `non_goals` のうちこの回に関わるものを入れる。

## ルール4: Worker への渡し方（MUST）

- Worker 入力の `task` は元のタスクのまま変えない。`context` の先頭に、7項目を次の見出しで書く。

  ```text
  ## 今回の指示（Loop Contract, after_round=<n>, action=<advance|verify>）
  - 目標: ...
  - 背景: ...
  - 必ず得る結果: ...
  - やってはいけないこと: ...
  - 止めて報告する条件: ...
  - 守り続けること: ...
  - 達成の証拠: ...
  ```

- `files` は goal に要るものだけに絞る。`verify` の回は、検証コマンドの出力を保存する先以外を増やさない。
- `context` に `purpose`（validation / delivery）を入れない（judge-rubric「検証タスクの書き方」）。
- `context` に秘密・個人情報・クライアント生データを入れない（secret-isolation）。

## ルール5: 最初の回（SHOULD）

- 最初の Worker の回（`after_round: 0`）は、sprint-contract と task から同じ7項目で書いてよい。必須ではない。
- 2回目以降（差し戻し・verify を含む）はルール1〜4を必須とする。
