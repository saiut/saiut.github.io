---
title:  "Logic Apps ワークフローのアクション分岐をまとめる"
date:   2025-07-03 14:34:25
categories: アプリケーション
author_profile: false
tags:
  - Logic Apps
---

## こんなこと書いてます

* Logic Apps は既定でワークフローは順番で実行される
* 分岐をさせる方法をまとめました

<!--more-->

## 分岐方法のいくつか

Logic Apps を利用してワークフローを実行させる場合、通常であれば順に沿って実行されます。

ただし、条件を付けて「特定の曜日の場合は実行させない」などといったことを実施したいことはあると思います。

その場合にどのような方法があるかをまとめました。

以下のような方法があります。

* Conditional
* Switch 
* Brunches
* Loop
* Scopes

### Conditional

まずは Conditional、条件です。
その名の通り、条件に合致すればxxxする、合致しなければ△△△する、といった条件分岐です。

下記の例は「注文金額が10万円以上なら承認処理を呼び出す、10万円未満なら即時処理」です。
```json
"actions": {
  "CheckAmount": {
    "type": "If",
    "expression": {
      "greaterOrEquals": [
        "@int(body('GetOrder')?['amount'])",
        100000
      ]
    },
    "actions": {
      "True": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            /* 承認API接続情報 */
          }
        }
      },
      "False": {
        "type": "Compose",
        "inputs": "自動承認"
      }
    }
  }
}
 ```
- typeにIfを指定
- expressionで真偽を判定
- actions.True／actions.Falseに分岐先を記述

### Switch 

スイッチです。条件式に合致するかどうかを複数パターンに振り分けます。
下記の例はケースごとに新規通知・確認通知・その他通知をさせるような処理をしています。

```json
"actions": {
  "OrderStatusSwitch": {
    "type": "Switch",
    "expression": "@body('GetOrder')?['status']",
    "cases": {
      "New": {
        "actions": {
          "NotifyNew": {
            /* 新規通知処理 */
          }
        }
      },
      "Confirmed": {
        "actions": {
          "NotifyConfirmed": {
            /* 確認通知処理 */
          }
        }
      }
    },
    "default": {
      "actions": {
        "NotifyUnknown": {
          /* その他通知処理 */
        }
      }
    }
  }
}
 ```
- casesでキーごとに処理を定義
- defaultでどのケースにも該当しない値をハンドリング

### Brunches

いわゆるパラレル処理、処理を並列で走らせ、処理の高速化や非同期実行を実現させます。
下記の例は、メール送信とログ記録を同時に行わせています。

```json
"actions": {
  "ParallelBranches": {
    "type": "Parallel",
    "branches": {
      "Branch1": {
        "actions": {
          "SendEmail": {
            "type": "Compose",
            "inputs": "メール送信"
          }
        }
      },
      "Branch2": {
        "actions": {
          "LogApi": {
            "type": "Http",
            "inputs": {
              /* ログ記録API */  
            }
          }
        }
      }
    }
  }
}
 ```
- typeにParallelを指定
- branches内で複数のブランチを定義
- 必要に応じてrunAfterで実行タイミングを制御


### Loop
繰り返し処理です。Loop は For each と Until の 2 種類があります。

#### For each
配列内にある要素を処理します。 parallel で同時実行数を調整できます。
下記の例の場合は 5 回繰り返します。

```json
"actions": {
  "ForEachItem": {
    "type": "Foreach",
    "foreach": "@body('GetList')",
    "actions": {
      "ProcessItem": {
        "type": "Compose",
        "inputs": "@item()"
      }
    },
    "parallel": 5
  }
}
 ```

- foreachで対象の配列を指定
- parallelで同時実行数を設定


#### Until
条件が満たされるまで繰り返します。無限ループを防ぐためにも limit や timeout を設定するといいでしょう。

```json
"actions": {
  "UntilLoop": {
    "type": "Until",
    "expression": {
      "equals": [
        "@variables('done')",
        true
      ]
    },
    "actions": {
      "CheckStatus": {
        /* ステータス確認処理 */
      },
      "Delay": {
        "type": "Delay",
        "timeout": "PT10S"
      }
    },
    "limit": 20,
    "timeout": "PT5M"
  }
}
 ```
- limitで最大試行回数
- timeoutで全体の実行上限時間

### Scopes
複数アクションをまとめて、成功・しパ維持の皇族処理を一括で設定します。

```json
"actions": {
  "TryScope": {
    "type": "Scope",
    "actions": {
      "StepA": { /* 処理A */ },
      "StepB": { /* 処理B */ }
    }
  },
  "CatchScope": {
    "type": "Scope",
    "runAfter": { "TryScope": ["Failed"] },
    "actions": {
      "LogError": { /* エラーログ記録 */ },
      "NotifyAdmin": { /* 管理者通知 */ }
    }
  },
  "FinallyScope": {
    "type": "Scope",
    "runAfter": {
      "TryScope": ["Succeeded"],
      "CatchScope": ["Succeeded"]
    },
    "actions": {
      "Cleanup": { /* 後始末処理 */ }
    }
  }
}
 ```
- Scopeを3つ用意してTry/Catch/Finally構成を実現
- runAfterで実行条件をまとめて指定

## まとめ

改めて、それぞれの分岐方法の紹介です。

- Conditional：Ifタイプでシンプルに2択分岐
- Switch：複数ケースの振り分けに最適
- Branches：Parallelで並列実行を実現
- Loop：Foreach／Untilでバッチや条件付き繰り返し
- Scopes：Scope＋RunAfterでTry/Catch/Finallyを実装

ワークフローで適切な分岐を選択することで、拡張性の高い設計が可能となります。