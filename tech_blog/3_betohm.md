# 【個人開発】未来予想して無料で賭けをするSNS - べトーム - を作って技術的なやってよかったこと　Flutter編

# まずは宣伝！！！

「今年のワールドカップの優勝国は？」などのスポーツベッティングや「来月のドル円はいくらになる？」などの経済の話題、「東京の明日の天気は？」などの軽いことまで、様々な話題を投稿してみんなで賭ける、SNS賭けアプリです！

詳細はこちら↓
https://nirai.io/betohm/ja/lp.html

ぜひダウンロードしてください！

宣伝はこのくらいにして、技術の話をします。

# Makefileを使ってよかった

「コード生成のコマンドってなんだっけ？」って探す必要が無くなったり、「リリースビルドを作ったけど、間違えて向き先が開発環境になってたりしないよな？」という不安が大幅に減りました！

Makefileは、一連のコマンドをまとめて実行することができるので、freezedのコード生成コマンドも長いコードをいちいち書く必要がなくなり、リリースビルドを作る時も本番環境に接続しているかとか確認も組み込めるので、ヒューマンエラーを防ぐことが出来ます。（まだ入れてないですが、リリースのヒューマンエラーを防ぐ意味も込めて、これからCI/CDツールも入れる予定です。）

また、数年間、このプロジェクトを触っていなくて、久しぶりに見た時も「このコマンドでコード生成するんだな」「このコマンドでリリースするんだな」と未来の自分や他のプロジェクトメンバーのためにも有益です。

```makefile
# make update と実行すると flutter clean & flutter upgrade & flutter pub get　が実行されます。
.PHONY: update
update:
	flutter clean
	flutter upgrade
	flutter pub get

# make genfreezed と実行すると freezed のコード生成コマンドが実行されます。
.PHONY: genfreezed
genfreezed:
	flutter pub run build_runner build --delete-conflicting-outputs

# make checkpro と実行すると 本番環境であることの確認のshell（env_pro_check.sh）が実行されます。
.PHONY: checkpro
checkpro:
	./env_pro_check.sh

# make buildrelease と実行するとまず checkpro が動いて、本番環境であることを確認します、その後flutterのリリースビルドコマンドが実行されます。
.PHONY: buildrelease
buildrelease: checkpro
	flutter clean
	flutter build appbundle --obfuscate --split-debug-info=build/app/outputs/symbols
```

# 統合テスト(Integration test)を実装して良かった

手動でのテストって面倒じゃないですか？？

これは統合テストに限らず自動テストはやった方がいいです！！

本当は単体テストも書いたほうがいいですが、まずは統合テストから書きました。（これから単体テストも書いていく予定です。）

統合テストではMockは使ってなくて、データベースの更新も実際にfirebaseにリクエストしています。

リリースの不安を少なくするのは結構大事なことだと思うんですよね。

「このテストが動いたから大丈夫でしょう！」という安心感は、リリースへの抵抗を減らすことにも繋がりますし、何より、リリース時の手動テストの8割9割が無くます！

今後リリースは何十回もしますから、リリース時に必要なテストの時間を少なくするのは、投資効果が高いですよね！！！
あと、リファクタリングへの抵抗も減るので、コードの改善もやり易いです！


以上、全体的な広い範囲話しましたが、一つだけこのコードを書いて良かったと言う狭い範囲の話もしたいと思います。

# 無限スクロールのWidgetを作って良かった
```dart
import 'package:bet/common/constants.dart';
import 'package:bet/common/context_config.dart';
import 'package:flutter/material.dart';

class InfiniteScroll extends StatefulWidget {
  const InfiniteScroll({
    Key? key,
    required this.isLast,
    required this.onNextItems,
    required this.list,
    required this.item,
    this.emptyWidget,
  }) : super(key: key);

  final bool isLast;
  final Future<void> Function() onNextItems;
  final List list;
  final Widget Function(int) item;
  final Widget? emptyWidget;

  @override
  State<InfiniteScroll> createState() => _InfiniteScrollState();
}

class _InfiniteScrollState extends State<InfiniteScroll> {
  bool isLoading = false;

  @override
  Widget build(BuildContext context) {
    return widget.list.isEmpty
        ? empty()
        : ListView.builder(
            physics: const BouncingScrollPhysics(),
            padding: const EdgeInsets.only(top: ValueSize.double8),
            itemBuilder: (BuildContext context, int index) {
              if (index < widget.list.length) {
                return widget.item(index);
              }
              nextItems(widget.onNextItems);
              return lastItem();
            },
            itemCount: widget.list.length + 1,
          );
  }

  Widget empty() {
    return !widget.isLast
        ? const Center(
            child: Padding(
              padding: EdgeInsets.symmetric(vertical: ValueSize.double12),
              child: CircularProgressIndicator(),
            ),
          )
        : widget.emptyWidget ??
            Center(
              child: TweenAnimationBuilder(
                tween: Tween<double>(begin: 0, end: 1),
                duration: const Duration(milliseconds: 1000),
                builder: (context, double value, child) => Opacity(
                  opacity: value,
                  child: Text(ContextConfig.l10n.nullError),
                ),
              ),
            );
  }

  void nextItems(Future<void> Function() function) async {
    if (!widget.isLast && !isLoading) {
      isLoading = true;
      await function();
      isLoading = false;
    }
  }

  Widget lastItem() {
    if (widget.isLast) {
      return const Center();
    }
    return const Center(
      child: Padding(
        padding: EdgeInsets.symmetric(vertical: ValueSize.double12),
        child: CircularProgressIndicator(),
      ),
    );
  }
}
```
使用例
```
@override
Widget build(BuildContext context, WidgetRef ref) {
  final viewModel = ref.watch(followersProvider(user).notifier);
  final follows = ref.watch(followersProvider(user));
  return SafeArea(
    child: InfiniteScroll(
      isLast: viewModel.isLast,
      onNextItems: viewModel.nextItems,
      list: follows, 
      item: (index) {
        return UserCard(userId: follows[index].userId);
      },
    ),
  );
}
```


Flutter編はこれまでとします。

今回は全体に関わる話をしましたが、この実装をして良かったというもう少し狭い範囲の話も気が向いたら書こうと思います。

次回は「【個人開発】未来予想して無料で賭けをするSNS - べトーム - を作って技術的なやってよかったこと　Firebase編」です。


最後に...

アプリの詳細はこちら↓です！
https://nirai.io/betohm/ja/lp.html
ぜひダウンロードしてください！