---
Title: 自家製静的サイトジェネレータに移行した話
Published: 2025/1/14 22:00:00
Tags:
  - "雑記"
---

元々はWordPressで運用していたこのブログ。  
その後ブログだけのためにVPSを運用するコストが高すぎてはてなブログに移行し、そして静的サイトジェネレータを利用したブログ構築が流行り始めたのでその波に乗ったのが3年ちょっと前。  

[oembed:"https://blog.hitsujin.jp/entry/2021/10/26/223803"]

以降ブログ更新が捗るか・・・といえばそうでもなく、中途半端にOEmbed対応した結果、ブログのビルドに10分近くかかってしまう（並列処理できてなかった）問題がありました。  
後で直そうと思っていたもののずっと放置していたのですが、久しぶりに（1年ぶりに）ブログを書こうかと思ってみたら、エントリをGitHubにプッシュした後に動作するGitHub Actionsでエラーが。  


ずっと放置していた関係で.NET SDKのバージョン指定が古く（.NET5だった）、処理不能になっていたのが原因だったんですが、それだけでなく指定されたURL遷移先がエラーだった場合のOEmbedの処理がよろしくなくて動作不全を起こしていることも発覚。  

仕方がないので重い腰を上げて書き換えることに。  


<!-- more -->


最初はStatiqをそのまま使って何とか動かすだけでいいかなぁと思っていたんですが、Statiqの最後のリリースから1年経過しており、またずっとベータ版状態が続いている中でこのまま使い続ける意義があまりないことからStatiqを利用しない方法に切り替えることにしました。  

時間が有り余っているならせっかくなので勉強がてらReactを使うなどしてシステム構築したかったんですが、いつ完成するかわからないままブログ更新ができなくなるのも嫌だったので、書きなれたC# で書くことに。  

参考にしたのはのいえさんの Blog2。  

[oembed:"https://github.com/neuecc/Blog2/"]  

のいえさんのシステムを丸パクリしていては何の技術向上もないので、Statiqと同じようにRazor構文をサポートしつつ、OEmbed対応など自分が使いたい機能の追加をすることに。  

で、いったんブログが出力できるところまで実装したのが下記のプロジェクト。  

[oembed:"https://github.com/Ovis/BlogGenerator"]

自作とはいえ便利なNuGetパッケージは利用することにして、 Razor構文サポートに RazorLight、Markdownファイルの処理に MarkDig などを利用しています。  

OEmbedの処理は以前作っていたジェネレーターからほぼ丸っと持ってきたうえで少し調整を加えました。  

RazorLightは本当にRazorが対応してる部分だけしか機能を持っておらず、 ASP.NET MVC が実装している `RenderSection` などがないので地味に使いづらいのですが、まぁ基本的に自分以外の人が使うことなんてないと思うので、自分がわかればいいだろうのスタンスで実装。  

並列処理で処理できるようになったので、以前のジェネレーターではOEmbedの処理を含めて10分以上かかっていた処理が、ローカル環境で40秒、GitHub Actionsでも2分半～3分くらいで生成できるようになった模様。  

OEmbedの処理がなければ2秒くらいなんですが、こればかりは仕方がない。  

今のGitHub Actionsは以下のような感じ。  
以前のジェネレーターでの設定をほぼ流用していて冗長な処理も多いので今後削りたい。  

[oembed:"https://github.com/Ovis/blog.hitsujin.jp/blob/2a634a0835b67875fddd219374ceb8c3786047dd/.github/workflows/azure-static-web-apps-calm-smoke-0628df600.yml"]


今後はGitHub ActionsでPagefindの定義ファイルを生成してサイト内検索を動くようにしたり予約投稿できるように調整を予定。  
