# 【個人開発】未来予想して無料で賭けをするSNS - べトーム - を作って技術的なやってよかったこと　Firebase Functions編

# まずは宣伝！！！

「今年のワールドカップの優勝国は？」などのスポーツベッティングや「来月のドル円はいくらになる？」などの経済の話題、「東京の明日の天気は？」などの軽いことまで、様々な話題を投稿してみんなで賭ける、SNS賭けアプリです！
賭けの成績が良い順にランクが決まります！あなたの未来予測力を試してみませんか？

詳細はこちら↓
https://nirai.io/betohm/ja/lp.html

ぜひダウンロードしてください！

宣伝はこのくらいにして、技術の話をします。

# Makefileを使ってよかった

これは「【個人開発】未来予想して無料で賭けをするSNS - べトーム - を作って技術的なやってよかったこと　Flutter編」で書いた内容と重複するので省略します。
端的に言うと「あのコマンドなんだっけ？」がなくなるのと、複数コマンドをまとめて実行できるので便利です！

# TypeScriptを採用して実装して良かった

特に説明不要だと思いますが、型チェックによるバグの早期発見や可読性が上がりるので嬉しいですね！

# 単体テストを書いて良かった

これも内容としては、「【個人開発】未来予想して無料で賭けをするSNS - べトーム - を作って技術的なやってよかったこと　Flutter編」の「統合テスト(Integration test)を実装して良かった」と大体同じなので省略します。
端的に言うと「手動テストって面倒だよね？」です！

以上、全体的な広い範囲話しましたが、一つだけこのコードを書いて良かったと言う狭い範囲の話もしたいと思います。

# firestoreのbatchの500件分割処理を楽にするコードを書いて良かった

```tsx
import { firestore } from "firebase-admin";

type StoreFunction = (batch: firestore.WriteBatch) => void;

export class FirestoreBatch {
    private batchArray: firestore.WriteBatch[] = [];
    private count = 0;

    constructor(private readonly db = firestore()) { }

    push(store: StoreFunction) {
        if (this.count % 499 == 0) {
            this.batchArray.push(this.db.batch())
        }
        store(this.batchArray[this.batchArray.length - 1])
        this.count++
    }

    async commit() {
        for (const batch of this.batchArray) {
            await batch.commit()
        }
    }
}
```

使用例

```tsx
const batch = new FirestoreBatch(this.db);
for (const user of users) {
    // create
    batch.push((batch) => {
        batch.create(this.db.collection("user").doc(user.uid), {
            xxx: user.xxx,
            yyy: user.yyy
        })
    })
    // update
    batch.push((batch) => {
        batch.update(this.db.collection("user").doc(user.uid), {
            xxx: user.xxx,
            yyy: user.yyy
        })
    })
    // delete
    batch.push((batch) => {
        batch.delete(this.db.collection("user").doc(user.uid))
    })
}
await batch.commit()
```

Firebase Functions編はこれまでとします。

次回は未定です。
反響があれば、「技術的なやって後悔したこと」を書いていこうかなと思います。


最後に...

アプリの詳細はこちら↓です！
https://nirai.io/betohm/ja/lp.html
ぜひダウンロードしてください！