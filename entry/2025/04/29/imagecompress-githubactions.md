---
Title: PNGファイル、JPEGファイルを圧縮するGitHub Actionsを作った話
Published: 2025/4/29 23:00:00
Tags:
  - "開発"
---

前回自宅キッチンのシンク下を修繕した話を投稿したのですが、このブログをホスティングしているAzure Static WebAppsにデプロイしようとしたところ、エラーが出てしまいました。  

エラーを確認したところ、デプロイ可能なサイズを超過しているとのこと。  
Azure Static WebAppsは250MBまでをサポートしており、写真を載せる際にウェブ用に圧縮せずにいたので超過した模様。  

とりあえず手作業で画像を圧縮することで取り急ぎブログ記事の公開をすることができたのですが、いちいち画像サイズを気にして置くのも面倒くさい。  

<!-- more -->

### GitHub Actionsで画像圧縮  

そんなわけで自動で画像を圧縮してくれる方法を探しました。  

最初は  

[oembed:"https://techracho.bpsinc.jp/hachi8833/2024_06_26/142464"]

こちらの記事を参考に `calibreapp/image-actions` を利用してみようと思ったのですが、リポジトリ全体の画像ファイルを圧縮しようとする模様。  
除外ディレクトリを設定するとかいろいろできるようですが、今回はプルリクエストに含まれている画像だけを対応するようにしたい。  

そんなわけで自分で作ることにしました。  

作るといってもわざわざプログラムを書くのも面倒なので、既存のツールの組み合わせで行くことに。  

そして作ったのが以下のリポジトリ。  

[oembed:"https://github.com/Ovis/image-compression-action"]

JPEG画像の圧縮率を指定できるほか、圧縮後のファイルサイズが指定の圧縮率以下である場合は圧縮をキャンセルするようにしました。そうしないとプルリクエストでプッシュするたびに画像が圧縮されてしまうので。  

内部的にはDocker上のDebian環境で `jpegoptim` 、 `optipng` を利用して圧縮をしているだけです。  
Docker内で処理をするようにしたことで、呼び出し側がWindowsランナーだろうがLinuxランナーだろうが動くようにしておきました。  

このワークフロー単体では圧縮だけを行い、ブランチへのコミットは行いません。そこはほかのActionsで十分に賄えるので。  

以下のような感じで呼び出して使えます。  

```yaml
name: Image Compression on PR

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: write

jobs:
  compress-images:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Compress Images
        id: compress
        uses: Ovis/image-compression-action@1.0.0
        with:
          jpeg_max_quality: "80"
          jpeg_compression_threshold: "5"
          png_compression_threshold: "40"

      - name: Commit compressed images
        uses: EndBug/add-and-commit@v4
        if: steps.compress.outputs.any_image_compressed == 'true'
        with:
          message: 'Compress images'
          default_author: github_actions
```

これでプルリクエストを作ったら勝手に画像圧縮されてコミットされるので、画像のサイズをいちいち気にしながら対応しなくても済むようになりました。  

今回のコードはChatGPTにいろいろ投げつけて吐き出されたものをもとにいじって作りました。この程度のものだと雑に指示してもある程度動作するものが吐き出されるのでいいですね。