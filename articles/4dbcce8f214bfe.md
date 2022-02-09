---
title: "Goでコマンドライン引数を受け取る"
emoji: "🐹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

# はじめに

Go で `go run main.go` するときの引数を扱いたかったので、その実装の過程の備忘録です。
.env ファイルまたは環境変数の定義がされていればそれも使用し、優先度順ではコマンドライン引数 > .env > 環境変数の順にしたかったのでそのような実装を目指しました。

# 実装過程

## コマンドライン引数を受け取る

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

[godotenv](https://github.com/joho/godotenv)を使い、.envファイルを読み込むことができます。

ただし、通常通り`godotenv.Load()`で読み込むと元から存在する環境変数が優先されるため、今回の場合`godotenv.Overload()`を使用する必要があります。

```go
// デフォルトではプロジェクトルートの.envが読まれる。
// 引数にファイル名を渡せば.env.developmentなどの使い分けも可能。
godotenv.Overload()
```

## 環境変数を受け取る

ビルトインの`os`パッケージを使って環境変数を読み込むことができます。

```go
// 環境変数 HOGE を読み込む
hoge := os.Getenv("HOGE")
```

また、環境変数のセットや、存在チェックも行うことができます。

```go
// 環境変数 HOGE に "hoge" をセット
os.Setenv("HOGE","hoge")

// 環境変数 HOGE に既に値がセットされていれば hoge に代入され、 ok に true が入る
// セットされていなければhogeには空文字、okにfalseが入る
hoge,ok := os.LookupEnv("HOGE")
```

## コマンドライン引数 > .env > 環境変数の優先順位で読み込む

これらの方法を駆使して、こんな感じで実装してみました！

```go
func init(){
    const (
        hogeKey = "HOGE"
    )
    var (
        hogeValue = flag.String(hogeKey, "", "hoge.")
    )

    flag.Parse()
    godotenv.Overload()

    m := map[string]string{
        hogeKey: *hogeValue,
    }

    for k, v := range m {
        if err := overrideEnv(k, v); err != nil {
            panic(err)
        }
    }
}

// 与えられたキーの環境変数を与えられた文字列で上書きします。
// 元の環境変数と上書きする環境変数どちらも存在しない場合、errorを返します。
func overrideEnv(key, value string) error {
	if value != "" {
		os.Setenv(key, value)
		return nil
	} else if _, ok := os.LookupEnv(key); !ok {
		return fmt.Errorf("%sを指定してください。(-%s=<VALUE>)", key, key)
	}
	return nil
}
```

# まとめ

`flag`に関しては提供されている関数が多く、他のユースケースでも試してみたいなと思いました。

`godotenv`に関しても.env.developmentみたいな実践的なパターンでも使ってみたいです！

ではーーー
