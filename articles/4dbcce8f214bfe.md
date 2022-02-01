---
title: "Goでコマンドライン引数を受け取る"
emoji: "🐹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# はじめに

Go で `go run main.go` するときの引数を扱いたかったので、その実装の過程の備忘録です。
.env ファイルまたは環境変数の定義がされていればそれも使用し、優先度順ではコマンドライン引数 > .env > 環境変数の順にしたかったのでそのような実装を目指しました。

# 実装過程

## コマンドライン引数を取得する

Go でコマンドライン引数を取得する方法は `os.Args` と `flag` パッケージの 2 通りがあるそうです。
簡単かつできることも多そうなので `flag` を選択しました！

参考記事:
https://qiita.com/nakaryooo/items/2d0befa2c1cf347800c3

使い方としては、

- `flag.String()`などに引数名、初期値、説明の順に渡して flag を定義
- `flag.Parse()`で解析
- 変数にはポインタが入ってるので、実体を取り出す

といった感じになります。`flag.Args()`で名前がついてない引数も受け取れますし、[公式ドキュメント](https://pkg.go.dev/flag)を見るといろんなユースケースに対応してそうです。

```go
func init() {
    var name = flag.String("name","John","User name.")
    flag.Parse()
    fmt.Println(*name) // -> Johnまたは実行時に-name=で渡した引数
}
```

コマンドは

```sh
$ go run main.go -name=Bob
```

のようにします。ハイフンは 1 つでも 2 つでもよく、boolean でないフラグなら=もなくていいみたいです。

重要なのは flag を定義したあとに flag.Parse()を呼び出すことで、コード内では問題なくても、上の例のように`init()`関数で flag の解析を行っている場合、`go test`したときにテストフラグの処理がテスト用に生成された main 関数で行われるため、それより先に呼び出される`init()`内の flag.Parse()が呼ばれた時点でテストフラグの解析ができなくなってしまい、テストが通りません。
この問題に関しては`init()`関数の先頭で明示的に`testing.Init()`を呼び出すことで解消できます。

参考記事:
https://syfm.hatenablog.com/entry/2019/10/01/063436

## .env ファイルで定義した環境変数を受け取る
