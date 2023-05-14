---
Title: Proxmoxで仮想環境をお手軽構築する話 インストールと初期設定
Published: 2023/05/14 23:00:00
Tags:
  - "環境構築"
---

これまでVMWare ESXiを利用して仮想環境を構築していたんですが、
- ESXi 7.0からRealtekのNICドライバがインストールできない
  - 正確にはこれまでコミュニティが提供していたが、最新のドライバ形式に対応できない
- 無償ESXiはTPM対応ができないか、手段が複雑
 
という問題があり、また今後も継続的に無償で使うことができない可能性も否定できないこともあり、そろそろ別の仮想化ソリューションを検討したいなぁと。  

そんなわけでいろいろ調べたところ Proxmoxというソリューションが大変よさげだったので導入しました。  

なお、今回は一旦ESXi上で作っていた仮想化環境をすべてふっ飛ばして一から作ることにしたので移行作業はなし。  

### Proxmoxの特徴
- VMware ESXiはベースがDebianなのでESXiと違ってドライバーがインストールしやすい
- ハードウェアの対応が幅広い
  - CPUサポートはLinuxカーネルのサポートに準ずるのでESXiよりサポートが長くなる
- KVMによる仮想マシンとLinux Container（LXC）が利用可能
- VMあたりのコア数制限なし
- ウェブコンソール対応  

標準で日本語に対応しており、GPUパススルーなどの設定も割と簡単。  
いくらか使った感想として、ESXiと比べて遜色ないか下手したらProxmoxのほうが使い勝手が良いなぁという感触。  


### Proxmoxインストール

[公式サイト](https://proxmox.com/en/downloads)からProxmox VEのISOファイルをダウンロード。  

![](proxmoxdownload.png)

ダウンロードしたISOイメージをDVDに焼くか、USBメモリーでブートできる形で書き込み。  
私はIODDというイメージブート可能なストレージを持っているのでこれを利用。  

<?# AmazonAffiliate B0B3HQMV5T /?>

インストールするマシンのUEFIからCPU Virtualizationのサポートを有効化しておいたうえでブート。  

![](proxmoxinstall1.png)

起動するとこの画面になるので `Install Proxmox VE` を選択している状態でEnter。  

![](proxmoxinstall2.png)
EULA画面になるのでチェックしてから `I agree` を押下。  

![](proxmoxinstall3.png)
インストール先のストレージ選択。  

![](proxmoxinstall4.png)
Optionsからファイルシステムの選択などが可能。  

![](proxmoxinstall5.png)
タイムゾーン、キーボードレイアウトの指定。  

![](proxmoxinstall6.png)
rootユーザーのパスワード指定。  

![](proxmoxinstall7.png)
サーバーのネットワーク設定。  

![](proxmoxinstall8.png)
設定した内容が表示されるので確認し、 `Install` ボタンを押下。  
あとはインストールが完了してProxmoxが立ち上がるのを待つだけ。  

![](proxmoxinstall9.png)
ベースがDebianなので起動するとコンソールが表示。  
通常はコンソールで作業することはなく、表示されてるURLをブラウザで開いてWebインターフェースを使って作業する。  

### Webインターフェースへログイン

コンソールに表示されていたURLをブラウザで開くと下記のようにログイン画面が表示される。（SSLの自己証明書を利用してる関係で警告が出るけども無視）  

![](web_login.png)

初期表示時点では英語表記。  
LanguageのところをJapaneseにすれば日本語UIに変更可能。  
Languageを変更するとUsername、Passwordの入力内容がリセットされるので、言語を変えるなら最初に変えること。  

![](web_logined.png)

Proxmoxはサブスクリプションサポートを採用してるので、サブスクリプション契約してないとログインするたびに `有効なサブスクリプションがありません` とアラートが表示される。  
サブスクリプション契約がなくとも普通に使えるので気にしない。  

### 無償版のリポジトリを設定  

初期インストール時点ではパッケージ管理システムのリポジトリに有償版のものしか登録されてないためProxmoxのアップデートができない状態。  
サブスクリプション契約しないのであれば無償版のリポジトリを設定する必要がある。  

DebianのAPTパッケージマネージャを使っているので、コンソールから無償版のリポジトリを登録してもよいけども、Webインターフェースからも設定が可能。  

まずリポジトリメニューを開く。  
左側のサーバーツリーから、 `データセンター > (インストール時指定したホスト名) ` を選び、`アップデート > リポジトリ` を選択。  

![](repository1.png)

`APT リポジトリ` の追加ボタンを押すとリポジトリ追加ウィンドウが表示されるので、リポジトリリストから`No-Subscription`を選択し追加。  

![](repository2.png)

これでサブスクリプション無しの無償版リポジトリが追加されたので、サブスクリプション契約が必要なエンタープライズリポジトリを無効化。  
`pve-enterprise` の項目を選択してから`Disable` を押せばOK。  
![](repository3.png)

これができたら一旦最新状態に更新するために、アップデートメニューを開き、 `再表示`ボタン(英語UIだと`Refresh`、多分誤訳)を押して最新のリポジトリ情報を取得。  

![](repository4.png)

`再表示` ボタンを押すとログイン時と同じく有効なサブスクリプション契約がない胸のメッセージが表示されるものの無視してOKボタンを押す。すると `apt-get update` 処理がログ画面に表示される。  
`TASK OK`と表示されたら 右上の×ボタンを押して閉じる。  
![](repository5.png)

`アップグレード` ボタンを押すと別ウィンドウが開き、コンソールで `apt-get dist-upgrade` 処理が走り、続行してよいか聞かれるので `y` ボタンを押下。  

![](repository6.png)

アップグレードが完了しても自動的にウィンドウは閉じないので自分で閉じる。今回の場合カーネル更新があり再起動が必要なので、リブートコマンドをたたくなり上部メニューの`再起動`ボタンを押下して再起動。  


![](repository7.png)

### ストレージの追加  
Proxmoxをインストールしたマシンにストレージが一つだけならここまででいいんですけども、他にもストレージを追加する場合はディスク追加を行う必要があります。  

![](adddisk1.png)

ホスト選択状態でディスクメニューを開き、追加したいディスクを選んで `GPTでディスクを初期化` を選択。  

終わったらディスクメニュー配下にある `LVM` `LVM-Thin` `ZFS` のうち利用するファイルシステムを選択。今回はZFS。    

![](adddisk2.png)

接続されているストレージがリストに表示されているので、対象のディスクを選択、名前も後で把握できる名前で。  
RAIDを組みたい場合はRAIDレベルを変更。  

![](adddisk3.png)

必要な設定が終わったら `作成` ボタンを押下すればストレージが追加される。  

![](adddisk4.png)




これでProxmoxの最低限の導入は完了。  
次からVMのインストールへ。  
