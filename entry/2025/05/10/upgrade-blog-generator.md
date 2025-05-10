---
Title: お手製静的サイトジェネレーターを改修してついでにdotnet toolsとして公開した話
Published: 2025/5/10 23:00:00
Tags:
  - "開発"
---

以前このブログの生成のためにお手製の静的サイトジェネレーターを作ったという話をしました。  

[oembed:"https://blog.hitsujin.jp/entry/2025/01/14/bloggenerator.html"]

oEmbedにも対応させて最低限自分が欲しい機能を載せたものの、oEmbedの処理の都合上ページ単位で情報を取得する必要があるため、oEmbed処理するページが増えるほどHTML生成に時間がかかるというジレンマがありました。  
oEmbed処理がなければ数秒で生成されるだけにちょっとこの時間がもったいない。というわけでキャッシュ処理を追加したのですが、せっかくコードをいじるのだからということでリファクタリングすると同時に dotnet tools対応などを行ってみました。  

<!-- more -->

### oEmbedキャッシュ処理とコマンドライン引数の調整

このジェネレーターはMarkdownの処理にMarkdigというライブラリを利用しています。  

[oembed:"https://github.com/xoofx/markdig"]

このライブラリはExtensionsという形で拡張を入れられるようになっていて、oEmbed対応もこれで実装してます。  
その中で外部から渡されたJSONをキャッシュとして読み込み、該当のURLのキャッシュがあればoEmbedのデータ取得処理を行わずキャッシュで対応し、全MarkdownファイルのHTML化が終わった後でキャッシュをJSONにシリアライズして別途ファイルに保存するようにしました。  

この対応をする際、ついでにコマンドライン引数の渡し方もよりコマンドラインツールらしい構造に変更しました。  

これまでは

```bash
bloggenerator "/input" "/output" "/theme"
```

のような形で引数を渡していましたが、今回 `System.CommandLine` というコマンドラインパーサーを採用して、

```bash
bloggenerator --input "/input" --output "/output" --theme "/theme"
```

このようにオプションを渡すことができるようにしています。 

[oembed:"https://www.nuget.org/packages/System.CommandLine"]

このパーサーライブラリ、Microsoft謹製で結構いろいろ使われているようなんですが、2025年5月時点でいまだにプレビュー状態。いつになったら正式版がリリースされるんだ・・・って感じですが、開発自体は継続されており、以下のIssueを見るとそろそろ何かしら動きがある様子。  

[oembed:"https://github.com/dotnet/command-line-api/issues/2500"]

### dotnet tool化  

これまではGitHub Actionsで実行可能なファイルを生成してからサイト生成処理を走らせるという形だったのですが、逐一ビルドする意味があまりないので、あらかじめビルドしたものを用意し、それで生成させるようにしました。  

方法としてはいろいろあるのですが、今回はdotnet toolにすることに。  

[oembed:"https://learn.microsoft.com/ja-jp/dotnet/core/tools/global-tools-how-to-create"]

これならあらかじめビルドしてNuGetに配置するようにしておけば、今後 `dotnet tool install 〇〇` とコマンドを叩くだけでインストールができます。  
どうせ私しか使わないようなツールなんだからそこまでする必要もないと言えばないんですが、せっかくなので勉強がてら。  

dotnet toolの作成は割と簡単で、
```csproj
	<PropertyGroup>
		<PackAsTool>true</PackAsTool>
		<ToolCommandName>bloggen</ToolCommandName>
		<PackageId>eSheepDev.BlogGenerator</PackageId>
		<Version>0.0.1.0</Version>
		<PackageOutputPath>./nupkg</PackageOutputPath>
		<PackageReadmeFile>./README.md</PackageReadmeFile>
		<PackageLicenseExpression>MIT</PackageLicenseExpression>
		<RepositoryUrl></RepositoryUrl>
		<PackageProjectUrl>https://github.com/Ovis/BlogGenerator</PackageProjectUrl>
		<Copyright>© 2025 Ovis</Copyright>
	</PropertyGroup>
```

こんな感じでプロジェクトファイルに必要な情報を記載し、 `dotnet pack` コマンドを実行するだけ。  
NuGetに公開することを前提に、パッケージIDなど公開に必要な情報も記載しています。  
パッケージIDは一意なものになるようにしないと、NuGet側で公開しようとしたときにほかのパッケージと重複しているとしてはじかれます。  

今回は パッケージIDを `eSheepDev.BlogGenerator` にしたので

```shell
dotnet tool install -g eSheepDev.BlogGenerator
```

これでインストール可能。  

[oembed:"https://www.nuget.org/packages/eSheepDev.BlogGenerator"]

使い方は README.md に詳しく書いたので、もしこれを使ってみようという奇特な方がいらっしゃったらそちらをご覧ください。  

[oembed:"https://github.com/Ovis/BlogGenerator"]


### GitHub Actionsでアーティファクトにファイルを格納し、次回実行時にファイルを取得する

さて、oEmbedキャッシュ処理を入れたことで、実行後にoEmbedのキャッシュをJSONファイルとして吐き出すようにしました。  
ローカル環境であればそのままファイルを次回使うようにするだけで済みますが、GitHub Actionsの場合は単純にファイルを生成しただけでは次回に使いまわすことはできません。  

そこで、Artifactを利用してファイルを使いまわすようにします。  
本来Artifactはジョブ間でファイルをやり取りするものらしいですが、ワークフロー間でファイルをやり取りすることもできます。  

以下は今回実際に使っているワークフローのうち、必要な個所を切り出したものです。  

```yaml
jobs:
  build_and_deploy_job:
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - name: Prepare artifact dir
        run: mkdir -p ./Artifact

      - name: Resolve workflow ID
        id: wf
        uses: actions/github-script@v6
        with:
          script: |
            const workflows = await github.rest.actions.listRepoWorkflows({
              owner: context.repo.owner,
              repo: context.repo.repo
            });
            const wfInfo = workflows.data.workflows.find(w => w.path === '.github/workflows/deploy-azure-static-web-apps.yml');
            core.setOutput('id', wfInfo.id);
            console.log(`🔍 Resolved workflow ID: ${wfInfo.id}`);
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Find last successful staging run
        id: find_run
        uses: actions/github-script@v6
        with:
          script: |
            const runResponse = await github.rest.actions.listWorkflowRuns({
              owner: context.repo.owner,
              repo: context.repo.repo,
              workflow_id: ${{ steps.wf.outputs.id }},
              status: 'success',
              per_page: 1
            });
            const lastRun = runResponse.data.workflow_runs[0];
            const runId = lastRun ? lastRun.id : '';
            core.setOutput('run_id', runId);
            console.log(`🔎 Last successful staging run ID: ${runId || '(none)'}`);

      - name: Download oembed.json artifact
        if: ${{ steps.find_run.outputs.run_id }}
        uses: actions/download-artifact@v4
        with:
          name: oembed
          path: ./
          run-id: ${{ steps.find_run.outputs.run_id }}
          repository: Ovis/blog.hitsujin.jp
          github-token: ${{ secrets.GITHUB_TOKEN }}
        continue-on-error: true

      - name: Run BlogGenerator
        run: |
          bloggen --input ./input --output ./output --theme ./templates --oembed ./oembed.json --config ./blogconfig.json

      - name: Upload oembed.json artifact
        uses: actions/upload-artifact@v4
        with:
          name: oembed
          path: ./oembed.json
          retention-days: 90
```

`Resolve workflow ID` の箇所で、指定したワークフローのIDを取得し、 `Find last successful staging run` の処理で該当ワークフローのうち前回成功した実行IDを取得するようにしています。  

後は `actions/download-artifact` で `run-id` に取得した実行IDを渡せばそのワークフローのアーティファクトがダウンロードできます。  

Artifactは最大90日まで保持できるので、90日以内にブログを更新すれば前回実行時のoEmbedのキャッシュデータを取得して処理できます。  
私は90日以上ブログを更新しないこともざらなので、30日に一度程度ワークフローを自動実行するようにでもしておこうかと。  

[amazon:B0D3V9D8ZF]

[amazon:B0D4DBYJJ9]