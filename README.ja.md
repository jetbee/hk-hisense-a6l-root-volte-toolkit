*(English version: [README.md](README.md))*

# Hisense A6L (HLTE730T) — root化・永続unlock・VoLTE有効化

Snapdragon 660 (SDM660) 搭載で、前面が通常のカラーLCD、背面がフルサイズのE Ink（電子ペーパー）というかなり特殊な両面ディスプレイ構成を持つ端末、**Hisense A6L (HLTE730T)** で、標準では一切対応していないVoLTE/IMSを、日本のMVNO/MNO SIM（mineo/KDDIおよび楽天モバイルで検証）向けに有効化するためのリバースエンジニアリング記録とツール一式。

初心者向けの整った手順書ではなく、「実際に何を試して、何が失敗して、何が効いたか」の記録です。次に同じことをする人（未来の自分を含む）が同じ回り道をしなくて済むように。指摘・修正歓迎。

**通常の免責事項:** この内容にはブートローダーの永続アンロック（不可逆）、パッチ済みboot imageの書き込み、Qualcomm診断ポート経由での稼働中モデムファームウェアの直接操作が含まれます。ブリックする可能性があります。始める前にEDLで`boot.img`、`vbmeta.img`、system/modemパーティション全体をバックアップしてください。HisenseやQualcommの公式サポート対象外の内容です。

## 目次

- [端末の背景情報](#端末の背景情報)
- [永続root化・ブートローダーアンロック](#永続root化ブートローダーアンロック)
- [ブリック復旧: EDLと開腹不要のトリガーケーブル](#ブリック復旧-edlと開腹不要のトリガーケーブル)
- [VoLTE問題の本質](#volte問題の本質)
- [手法1: ModemTestModeをパッチしてQMIを直接叩く](#手法1-modemtestmodeをパッチしてqmiを直接叩く)
- [手法2: QPST経由でモデムEFSをライブ編集する](#手法2-qpst経由でモデムefsをライブ編集する)
- [手法3（楽天SIMで実際に効いたのはコレ）: 純正キャリアプロファイルを総当たりする](#手法3楽天simで実際に効いたのはコレ-純正キャリアプロファイルを総当たりする)
- [おまけ: 5GHz WiFiホットスポットの有効化](#おまけ-5ghz-wifiホットスポットの有効化)
- [オフラインで編集したmcfg_sw.mbnが拒否される理由](#オフラインで編集したmcfg_swmbnが拒否される理由)
- [このリポジトリに含まれるツール](#このリポジトリに含まれるツール)
- [参考リンク集](#参考リンク集)
- [未解決の課題](#未解決の課題)
- [付録: 動作確認済みの環境バージョン](#付録-動作確認済みの環境バージョン)

## 端末の背景情報

- **機種:** Hisense A6L、型番`HLTE730T`（`HLTE730T.B1`、`.B6`表記も確認）
- **筐体構成:** 両面ディスプレイ——前面は通常のカラーLCD、背面は独立して描画可能なフルサイズのE Ink（電子ペーパー）ディスプレイ。フレームワーク内部にもE Ink固有の仕組みが組み込まれており（`NetworkController.MobileSignalController`のログに現れる`mPreEInkStatus`フラグ、ロック画面上の歩数表示など）、普通の単一画面Android機とは前提が違う点に注意——一般的なAndroid/Qualcomm向けガイドがそのまま通用するとは限りません。
- **SoC:** Qualcomm Snapdragon 660 (SDM660)
- **モデムファームウェアベースライン:** `MPSS.AT.3.1-00819-SDM660_1.2`（QPSTで確認可能）、Policy Manager XMLのヘッダには`mmcp.mpss/8.1.1`と記載
- **Android:** 9.0、中国市場向けファームウェア、Googleアプリなし、モデムの実力とは無関係にAndroidフレームワーク層でVoLTEがゲートされている

## 永続root化・ブートローダーアンロック

### fastbootへの入り方

1. 端末を完全に電源OFFにする。
2. 普通のUSBケーブルを**端末側だけ**に挿す（EDLと違って、ここではトリガーケーブルや洗濯ばさみの技は不要）。
3. 電源ボタンは押さず、**音量上**だけを押した状態を保ち、そのままケーブルのもう一端を**PC側**に挿す。
4. ブルっと一回震えてfastbootモードで起動し、`START`や`Unlock status`などのfastbootステータス表示が、普通に大きく読みやすい文字で出るはず。これが成功。
5. 入ったら音量上ボタンは離してよい。

**ハマりどころ:** USBポートによっては、正常なfastboot画面が一瞬出てすぐ消え、暗くなってまた震える……を延々繰り返すことがある。うまいタイミングで止めようと頑張らないこと——うまく止められたとしても、LCD側の左上に虫眼鏡でしか読めないくらい小さい文字で`press any key to shutdown`と表示されており、それでループしているだけ。原因は、（Zadig等で誤って割り当てた）fastbootではないドライバが、OSがその接続をルーティングしているポートに紐付いてしまっていること。対処法: 間にUSB HUBを挟んで、**別の新しいポート**として認識させる——古いドライバの紐付けが無いポートに乗れば、期待通りきちんとfastboot画面で止まる。

### アンロック手順

1. OEMの`fastboot Hisense unlock`コマンドで一時アンロック。**素の platform-tools の`fastboot`は`Hisense`というOEMサブコマンドを認識しません** — これに対応したカスタム/パッチ済みfastbootバイナリが必要です。私たちが実際に使えたのは、[`aimindseye/hisense-a9`](https://github.com/aimindseye/hisense-a9)（別のHisense機種向けだが関連するリポジトリ）に貼られているGoogle Driveリンク経由のものでした——このパッチ済みfastbootバイナリ自体は機種固有ではなくOEM（Hisense）固有のものです。本リポジトリでは再配布していないので、自分でリンクを見つけて確認するか、リンクが切れていたら他のHisense専用fastbootツールを探してください。
2. `fastboot erase avb_custom_key` — これが実際に不可逆な本アンロック処理です。この端末では、これ単体ではuserdataの消去も確認ダイアログの表示も**行われません**（他機種向けの一部ガイドとは異なる挙動）——自分で`fastboot erase userdata`も実行するか、結果として出る「Decryption Unsuccessful」のリカバリー画面から手動で初期化する覚悟をしてください。
3. Magiskでパッチした`boot.img`と、`--disable-verity --disable-verification`付きでフラッシュした`vbmeta.img`を書き込みます。**パッチするのは、ネット上のファームウェア配布パッケージに入っている`boot.img`ではなく、自分の端末からEDLで吸い出した`boot.img`にしてください。** この端末にはリージョン/バージョン違いのファームウェアが複数出回っており、公開されている`boot.img`が自分の実機の中身と一致しない、ということが普通に起こります。違うものをMagiskパッチしてしまうと、下のリカバリー節のお世話になる羽目になりがちです。

   **ここでMagisk自身が勧めてくる「Recovery Mode」は無視してください。** この端末の`boot.img`は本当に`ramdisk_size=0`です（ファームウェアzip由来のものだけでなく、EDLでの実機ライブダンプでも確認済み）——ramdiskを一切持たないlegacy System-as-Root機です。Magiskアプリはこれを自動検出し（ホーム画面に「Ramdisk: No」と出る）、Installで**Recovery Mode**のチェックボックスを自動でONにして、`recovery.img`側をパッチしてそちらに書き込むよう誘導してきます。**その誘導には従わないでください——この端末で実際にroot化された通常起動を得るには行き止まりです。** Recovery Modeのチェックを外し、普通の`boot.img`を直接パッチしてください（「Install → Select and Patch a File」、チェックは外したまま）。Magiskのlegacy-SAR向けramdisk注入機能はramdiskの無いboot imageに対してもちゃんと機能し、出力ファイルの`ramdisk_size`は`0`からゼロでない実際の値になります——これを`recovery`ではなく`boot`に書き込みます。
4. これで本当のコールドブートを経ても永続します（手順2を省いた一時アンロックのみだと、`fastboot continue`一回分しか有効にならず、本当の再起動で元に戻ります — ここの見極めにかなり時間を使いました）。

## ブリック復旧: EDLと開腹不要のトリガーケーブル

**順調に進む場合でも、最初の1回だけはEDLを使います**——何もアンロックしていない時点ではrootもなくブートローダーもロックされたままなので、上の免責事項で推奨している通り純正の`boot`/`vbmeta`/`system`/モデムパーティションをバックアップするには、実質EDLでの生読み出し以外に現実的な方法がありません。この最初のバックアップさえ取ってしまえば、この節の残りは純粋に、何か失敗した時（書き込みミス、ブートループ等）の保険です——アンロック手順自体（すべて素の`fastboot`のみ）はその後二度とEDLに触れません。

Emergency Download Mode (EDL)への突入には、通常は端末内部のテストポイントをショートさせる必要があります（分解が必要）。**「deep flash cable」「EDLケーブル」「test point cable」**などで検索してみてください——Snapdragon端末向けに、余ったピンに抵抗を仕込んで接続するだけでEDLに強制突入させるUSBケーブルが売られています。安価で、まさにこの用途のために広く流通しているので、必要になってからではなく始める前に買っておく価値があります。実際に使ったもの: [zmart Xiaomi Deep Flash Cable「Open Port 9008」「Phone Model Free」](https://www.amazon.co.jp/dp/B06XYP1J7N) ——Xiaomi向けとして売られていますが、中身は汎用のSnapdragonトリガーケーブルで、このHisense端末でも問題なく使えました。

この端末でこのケーブルを使う場合、確実に成功する突入手順:

1. 端末を完全に電源OFFにする。
2. ケーブルのボタンを押し込んだ状態で固定し（洗濯ばさみ/ダブルクリップが便利）、ケーブルを**端末側**に挿す——**PC側はまだ挿さない**。
3. 左手を**音量下**と**電源**ボタンにスタンバイさせる（まだ押さない）。
4. 右手でUSB端を**PC側**に挿す——これが、左手で音量下＋電源を同時押しするタイミングの**わずかに先、もしくはほぼ同時**になるようにする。
5. 2〜3秒待つ。**何も振動せず、画面が真っ暗なままなら成功**（振動したら通常起動してしまったということなので、電源を切ってやり直す）。
6. すぐにボタンから指を離す。ケーブルのボタンに挟んだ洗濯ばさみ/クリップを外すのを忘れないこと。
7. これで端末がPCから認識・通信可能になっているはず（Sahara/Firehose）。注意: **ケーブルのボタンを押している間は端末は通信できない**——PC側のツールが実際に話せるようにするには、ボタンを離す必要がある。

EDLに列挙されるのはSaharaハンドシェイクまでで、それだけではまだ何も読み書きできません。そのためには、PC側から**Firehoseローダー**（小さな`.elf`形式のプログラマーイメージ）をSahara経由で端末のRAMにアップロードする必要があります——実際にFirehoseプロトコルを喋り、以降の生パーティション読み書きを行うのはこのローダーです。端末のPBLが受理するローダーが無ければSaharaで止まったままで、このセクションの残りは一切機能しません——事実上、そこから先の全部を開く「鍵」に相当します。

ここで実際に効いたもの: `prog_emmc_ufs_firehose_Sdm660_ddr_30060000.elf`。知っておくべき点:
- これは**汎用のSDM660用ローダー**であって、Hisense自身のファームウェアから抜き出したものではありません——この端末のPBLではHisenseに暗号的に紐付けられておらず、チップセットさえ合っていれば**別のSDM660端末**の純正フラッシュツールパッケージから取ったローダーで問題なく動きます。
- 一番手軽な入手元: [`bkerler/edl`](https://github.com/bkerler/edl)（オープンソースの`edl.py`/`qdl`プロジェクト）——SDM660を含む幅広いQualcommチップセット向けのローダーが同梱されています。他のSDM660機種向けのQPST/QFILパッケージも入手元の一つです。
- EDLには列挙される（Saharaは成功する）のに、あらゆる操作が失敗・タイムアウトする場合は、まずローダーを疑ってください——チップセットのバリアント違いか、PBLが本当に拒否しているローダーかのどちらかです。
- 実は最初に「素直な」方法として、**ダウンロードしたHisense A6L用ファームウェアパッケージ（本ドキュメント後半のROM配布サイト一覧参照）からFirehoseローダーを抜き出して**試しましたが、これは**動きませんでした**——Saharaが受け付けてくれませんでした。実際に動いたのは、この端末専用でも何でもない、上記の`bkerler/edl`の汎用SDM660ローダーです。根本原因までは特定できていません——ここで言えるのは、「この機種そのもの向けに見える純正っぽいファームウェアパッケージ内のローダーが安全に使える」という前提は成り立たない、ということです。最初からコミュニティの汎用ローダーを使った方が時間の節約になります。

Firehoseローダーが用意できたら、`edl.py`（またはQPST付属の`QSaharaServer`/`fh_loader`）で`boot`、`vbmeta`、`system`、モデムパーティションを生で読み書きします。このセッションでハマった制約:
- この端末では、EDLセッション1回につき読み書き1操作まで——操作の間に電源を入れ直してEDLに入り直す必要がある。
- 再試行の前に、ポートを掴んでいる残留Pythonプロセスをkillする。
- Windowsで`edl.py`を使う場合`PYTHONIOENCODING=utf-8` / `PYTHONUTF8=1`を設定する。
- 必要に応じてZadig経由でWinUSBドライバを入れる。

## VoLTE問題の本質

この端末は中国SKUのためVoLTE/IMSが一切入っていません。Android側のゲート（`ImsManager.isVolteEnabledByPlatform()`、`content://telephony/siminfo`のSIMごとの`volte_vt_enabled`カラム、中国キャリア限定デフォルトの`QtiVolteSwitchController`）はrootがあれば全部強制的に開けます:

```sh
setprop persist.dbg.volte_avail_ovr 1
# SIMごと、subIdが分かれば:
content update --uri content://telephony/siminfo --bind volte_vt_enabled:i:1 --where "_id=<subid>"
```

しかしこれだけでは足りません——**モデム自体のMCFG（Modem Config）キャリアポリシー**も、自分のネットワークのMCCに対してVoLTEを許可している必要があり、さらに別途、読み込んだキャリアプロファイルのホームPLMNと一致しないSIMに対して一般データPDNの確立をブロックしていないことも必要です。この両方を正しく揃えるのが本リポジトリの大部分の内容です。

## 手法1: ModemTestModeをパッチしてQMIを直接叩く

実行時に（起動時だけでなく）キャリアのMCFG設定を選択・有効化する、Qualcomm純正の正規の方法は`com.qualcomm.qti.modemtestmode`（パッケージ名「ModemTestMode」/「MBN Test」）で、`com.qualcomm.qti.qcrilhook.QcRilHook`経由でQMIでモデムを直接操作します。重要なActivityは2つ:

- **`MbnFileLoad`** — `/data/vendor/modem_config/mcfg_sw/generic/<リージョン>/<キャリア>/.../mcfg_sw.mbn`を参照し（このディレクトリには`oem_sw.txt`の制限と無関係に、制限のないキャリア全部が最初から入っています）、ファイルをモデムのEFSにロードします（`setupMbnConfig()` → `qcRilSetConfig()`）。
- **`MbnFileActivate`** — 現在EFSに登録されているconfig一覧を表示し、選択して有効化します（`selectConfig()` → `qcRilSelectConfig()`）。有効化すると約15秒後に自動的に再起動します。

この端末では、素の`MbnFileActivate`は「Device is not configured properly」というハードエラーで先に進めません。原因は`getMbnInfo()`が、HW側MBN設定の問い合わせがnullを返した時点（このモデムはHW設定を一度もアクティベートされたことがないので常にnull）で即座に処理を打ち切ってしまうためです——本来やりたいSW（キャリア）側の選択フローとは無関係なチェックに阻まれています。

**対処法:** `patches/ModemTestMode-MbnFileActivate.smali.diff` — `ModemTestMode.apk`をboot classpath全体を使ってbaksmaliでdeodexし、`return v1`という早期リターンを`goto :cond_83`（失敗したチェックを無視して処理続行）に変更し、再アセンブル・再パッケージします。

実際にハマった、時間を食ったポイント:
- 素のAPKには**classes.dexが最初から一切存在しません**——実コードはodex/vdexにしかありません。zipに`classes.dex`を「新規追加」する必要があります（「既存エントリを置き換える」ロジックのスクリプトだと、静かにdexなしAPKができあがります）。
- **Magiskモジュール**としてオーバーレイでデプロイする（`/data/adb/modules/<id>/system/app/ModemTestMode/ModemTestMode.apk`）——この端末はLegacy SAR（`/`自体が`mmcblk0p*`で、独立した`/system`マウントポイントがない）なので`mount -o rw,remount /`は失敗しますし、アプリレベルの変更のために生パーティションに触る必要もありません。
- **元のodex/vdexも隠す必要があります**（`mknod <module>/system/app/ModemTestMode/oat/arm64/ModemTestMode.{odex,vdex} c 0 0`でwhiteout化）、**さらに**別のARTキャッシュ`/data/dalvik-cache/arm64/system@app@ModemTestMode@ModemTestMode.apk@classes.{dex,vdex}`も削除する必要があります。この3箇所のどれか一つでも見落とすと古いコードが動き続け、「パッチが効いていない」と無駄にデバッグする羽目になります。
- `MbnFileLoad`のファイルブラウザはSELinuxをpermissiveにする必要があります（`setenforce 0`）——ポリシー上、`radio`ドメインには`vendor_mbn_data_file`コンテキスト（`/data/vendor/modem_config`）への`search`権限がなく、これがないとブラウザが黙って空リストを表示します。再起動でenforcingに戻るので、永続的なポリシー変更は不要ですが毎セッション実行し直す必要があります。

## 手法2: QPST経由でモデムEFSをライブ編集する

root化済み・diagポート使用可能な状態になったら、**QPSTのEFS Explorer**（`C:\Program Files (x86)\Qualcomm\QPST\bin\EFSExplorer.exe`）でモデム自身のEFSファイルシステムを直接閲覧・上書きできます——これは「新しいMCFGパッケージをインポートする」とは全く別のコードパスで、（少なくともこの端末では）下記の全体パッケージ用ダイジェスト検証の対象外です。

diagポートの有効化:
```sh
adb shell setprop persist.usb.eng 1
```
Windows側で`Diagnostics Interface (COMx)`というデバイスが列挙されるはずです（この端末では`USB\VID_109B&PID_90FE&MI_02\...`——いわゆる「9091」系Qualcomm診断コンポジット）。`QPSTConfig.exe`でポートを追加し、`EFSExplorer.exe`から接続します。

**うまくいったこと:**
- **既にActivate済みのキャリア設定**に対して`/policyman/carrier_policy.xml`を編集する（例: 既存の`<plmn_list name="unrestricted_operators">`に自分のSIMのPLMNを追加する）——これは`adb reboot`だけで反映されました。再Activateは不要。
- 同様に、既に宣言済みの個別NVアイテムを編集する（`Item File`型のエントリには右クリックメニューの「Copy Item File from PC...」を使う——上記XMLに使う「Copy Data File from PC」とは**別のメニュー項目**である点に注意。ローカルの元ファイルの**ファイル名**がEFS上のアイテム名と完全一致している必要があります）。

**うまくいかなかったこと:**
- キャリアの`/policyman/`ディレクトリに**元々存在しなかった新規ファイル**を書き込むこと（例: 元々`carrier_policy.xml`を一切持たない汎用`common/row`プロファイルに新規作成する）。書き込み自体は黙って成功しますが、再起動してもPolicy Managerが拾いません。推測: Policy Managerはインポート時点で構築されたconfigごとのアイテム一覧を消費しているだけで、ライブのディレクトリスキャンはしていない——つまり、後から中身を自由に書き換えられても、そもそも元のインポート済みパッケージの一部だったファイルでない限り一切読まれない、ということのようです。

作業全体を通して使ったWindows自動化ヘルパー（`scripts/efs-explorer-automation-helpers.ps1`）は、`SetCursorPos`/`mouse_event`/`SendKeys`ベースの小さな関数群で、まともなWin32オートメーションIDモデルを持たないEFS Explorer（古いMFCアプリで、標準のUI Automationからはリスト項目がほぼ見えない）のダイアログを操作するためのものです。ダイアログのスクリーンショットを撮り、**表示された画像**上のピクセル座標を読み取れば、このリポジトリの関数がそれを処理します。ファイルパスの入力には、`SendKeys`直接入力より**クリップボード貼り付け**（`Set-Clipboard` + `^v`）の方が確実でした——このシステムではアクティブなIMEの影響で`SendKeys`直接入力が時々文字化けしました。

## 手法3（楽天SIMで実際に効いたのはコレ）: 純正キャリアプロファイルを総当たりする

> **出典:** [XDA — \[Guide\] Enabling VoLTE/VoWiFi (deprecated)](https://xdaforums.com/rog-phone-2/how-to/guide-enabling-volte-vowifi-t4023529)（投稿者HomerSp、対象は全く別の機種のASUS ROG Phone 2、Snapdragon 855向け）。この手法全体があのスレッドのアイデアであって、私たちが考案したものではありません——ここではそれがこの端末にも当てはまることを確認しただけです。クレジットは全面的にあちらに帰属します。

あのスレッドの手法: **プロファイル集を無改造のまま片っ端から試して、どのキャリアのプロファイルで実際にVoLTEが有効になるかを記録する。** MCFGキャリアポリシーは特定のオペレーターではなく**国（MCC）単位**でスコープされていることが多く、自分とは無関係な別キャリアのプロファイルが自分の国向けVoLTEを解除してしまうことが頻繁にあります。

これをこの端末に適用してみると: かなりのパッチ作業を自分たちでも試した後、実際に楽天モバイルSIM（MCC/MNC `440-11`）でVoLTEと一般データの両方を解決したのは、拍子抜けするほどシンプルな方法でした——**無改造の別のキャリアプロファイルをそのままActivateして様子を見る。**

具体的には、この端末に楽天SIMを挿した状態で:

| ロードしたプロファイル（すべて無改造） | VoLTE / IMS PDN | 一般データ（デフォルトAPN） |
|---|---|---|
| `apac/kddi` | 動く | **失敗**、`OEM_DCFAILCAUSE_4` |
| `apac/dcm`（NTT docomo） | 動く | **失敗**、同じ原因 |
| `common/row`（汎用ROW） | 動かない | 動く |
| **`apac/sbm`（SoftBank）** | **動く** | **動く** |

パターンとして: 試した2つの日本MNO専用プロファイル（KDDI、docomo）は、どちらも自社のPLMNをVoLTE適格リストとしてホワイトリスト化する`carrier_policy.xml`を持っており、かつ実測では、そのホワイトリスト外のSIMに対する一般データPDN確立もブロックしていました——これは端末側のバグというより、実際のMNOキャリアポリシーに意図的に組み込まれた不正/ローミング悪用防止的な挙動に見えます。汎用ROWにはそのようなホワイトリストがない（だから常にデータは通る）代わりに、国レベルのVoLTE有効化ディレクティブもありません。SoftBankのプロファイルにはどうやらどちらの制限もなく、楽天SIMに対してそのまま動作しました。**なぜ**そうなっているかまでは解析していません——純粋に実測で確認しただけです（`dumpsys telephony.registry`の`mVoLteServiceState`、`dumpsys connectivity`の`NetworkAgentInfo`が`extra: ims`と`extra: <apn>`の両方について`CONNECTED/CONNECTED`かつ`everValidated=true`になっていること、そして実際の`ping`疎通確認）。

**特定のSIM向けにVoLTEを追いかけていて、端末に複数の純正キャリアプロファイルが用意されているなら、パッチを書く前に全部試してみてください。** タダで、速くて、完全に元に戻せて（別のプロファイルで`MbnFileLoad`/`MbnFileActivate`をやり直すか、元のプロファイルに戻すだけ）、それだけで解決するかもしれません。

## おまけ: 5GHz WiFiホットスポットの有効化

この端末はWiFi to WiFiテザリング（他所のWiFi——例えばホテルのWiFi——をこの端末経由でプロクシする使い方）に対応していますが、日本向けファームウェアでは5GHz帯のSoftApが常に`SAP_START_FAILURE_NO_CHANNEL`で起動失敗します。屋内でしか使わないのに5GHzが塞がれているのはもったいないので、有効化するパッチを作りました。

> **⚠️ 免責事項:** 日本国内で5GHz帯を屋外で使用することは電波法上できません。**必ず屋内でのみ使用してください。** このパッチは自己責任で使用してください。本節はあえて詳細な解説を省き、コマンドの列挙とパッチファイルのみとします。

パッチ内容: `patches/WifiInjector-makeSoftApManager.smali.diff`（`WifiInjector.smali`の`makeSoftApManager()`内、SoftApManagerに渡す国コードのみを`"US"`に固定——STA/通常WiFi接続には影響しない）。

```sh
# 1. スタブjar本体+実コード(odex/vdex)を取得
adb pull /system/framework/wifi-service.jar
adb pull /system/framework/oat/arm64/wifi-service.odex
adb pull /system/framework/oat/arm64/wifi-service.vdex

# 2. boot classpath全体を使ってdeodex（手法1のModemTestModeパッチと同じ流儀。boot.oatを含むディレクトリで実行すること）
java -jar baksmali.jar deodex -b boot.oat wifi-service.odex -o smali-out

# 3. patches/WifiInjector-makeSoftApManager.smali.diff を smali-out/com/android/server/wifi/WifiInjector.smali に適用

# 4. 再アセンブル
java -jar smali.jar assemble smali-out -o classes.dex --api 28

# 5. スタブjarに新規zipエントリとして追加（元jarにclasses.dexは無い）
cp wifi-service.jar wifi-service-patched.jar
jar uf wifi-service-patched.jar classes.dex

# 6. Magiskモジュールとしてデプロイ（systemパーティション本体には触らない）
adb shell su -c mkdir -p /data/adb/modules/<id>/system/framework/oat/arm64
adb push wifi-service-patched.jar /sdcard/
adb shell su -c cp /sdcard/wifi-service-patched.jar /data/adb/modules/<id>/system/framework/wifi-service.jar
adb shell su -c mknod /data/adb/modules/<id>/system/framework/oat/arm64/wifi-service.odex c 0 0
adb shell su -c mknod /data/adb/modules/<id>/system/framework/oat/arm64/wifi-service.vdex c 0 0
# module.prop を /data/adb/modules/<id>/module.prop に配置
adb shell su -c rm -f /data/dalvik-cache/arm64/system@framework@wifi-service.jar@classes.dex
adb shell su -c rm -f /data/dalvik-cache/arm64/system@framework@wifi-service.jar@classes.vdex

# 7. 再起動してから、設定 > パーソナルホットスポット > Set up super hotspot > Select AP Band > 5 GHz Band を選択してON
adb reboot
```

## オフラインで編集したmcfg_sw.mbnが拒否される理由

プロファイル総当たりで解決しなかった場合に備えて、実際に`carrier_policy.xml`を編集する必要が出たときに何と戦うことになるかを先に書いておきます。

`mcfg_sw.mbn`はELFでラップされたコンテナです（`readelf`/`pyelftools`で普通にパースできます）: セグメント0はハッシュテーブル/secure-boot的なセグメント、セグメント1には埋め込みのX.509的な証明書チェーン（OEMアテステーション用で、ペイロードとは無関係）、セグメント2（`PT_LOAD`）が実際のNVアイテムデータを`type,path-len,path,data-len,data`というTLVレコードの単純な列として保持しており、最後は他セグメントの**32バイトダイジェスト**（SHA-256）を含むトレーラーアイテムで終わります。

[`sbaresearch/mbn-mcfg-tools`](https://github.com/sbaresearch/mbn-mcfg-tools)（このハッシュを単にコピーするのではなく本当に再計算しているPythonツール——無変更でのextract→repackの往復が元ファイルとバイト単位で完全一致することで確認済み）を使って確認したところ、**内部ハッシュが正しく再計算されていても、この端末のモデムで改変済みファイルが受理されるには不十分**でした: どんな編集をしたファイルも、ハッシュの妥当性に関わらず、`MbnFileLoad`のQMIインポート経路から`error code:-1`が返ってきます。このツールの作者自身も、[`fenrir-naru/mbn_utils`](https://github.com/fenrir-naru/mbn_utils)（同じ目的のもっと未完成な先行ツール）も、独立に同じ壁にぶつかったと記録しており、ファイル外部に保存された参照ダイジェスト（`sbaresearch`のREADMEには`mcfg_sw_config_digest_version`がEFSに別途保存されているという記述がある）か、あるいはこれらのツールが偽造を試みていない本物の暗号署名のどちらかが疑われています。

結論: **`.mbn`を手で編集して`MbnFileLoad`で再インポートしようとするのはやめましょう。** 代わりに (a) 既にActivate済みのconfig内の、既に宣言済みのファイル/アイテムをEFS Explorerでライブ編集する（手法2）か、(b) やりたいことを既にやってくれている純正プロファイルを探す（手法3）のどちらかにしてください。

`patches/mbn-mcfg-tools-windows-path-fix.patch`は上記の話とは無関係で、このツールのWindows固有のバグに対する小さな修正です（`/policyman/carrier_policy.xml`のような先頭スラッシュ付きのアイテムパスがそのまま`pathlib.Path()`に渡されると、Windowsでは相対パスではなく現在のドライブのルートに解決されてしまい、展開されたファイルのほとんどが静かに失われる）。上記のハッシュの壁とは別問題として、Windowsでこのツールを使うなら有用です。

## このリポジトリに含まれるツール

- `patches/ModemTestMode-MbnFileActivate.smali.diff` — HWチェックのバイパス（手法1）。これは素のAPKを自分でbaksmaliした出力に対するdiffであり、再配布バイナリではありません——パッチ済みdexは自分の端末から取得したファームウェアを元に自分で生成してください。
- `patches/WifiInjector-makeSoftApManager.smali.diff` — 5GHz SoftApの国コード上書き（上記「おまけ」節参照）。同様に、自分のbaksmali出力に対するdiffであり、再配布バイナリではありません。
- `patches/mbn-mcfg-tools-windows-path-fix.patch` — `sbaresearch/mbn-mcfg-tools`向けのWindowsパス処理修正。
- `scripts/efs-explorer-automation-helpers.ps1` — QPST EFS Explorerのダイアログを操作するPowerShellマウス/キーボード自動化（手法2）。

含まれていないもの: Hisense/Qualcommの著作権があるバイナリ全般（純正・パッチ済み問わずAPK、`.mbn`ファイル、boot image）。上記のパッチを使って、自分の端末のファームウェアから自分で再生成してください。

## 参考リンク集

**この端末（HLTE730T）のファームウェア:**
- [Hisense A6L HLTE730T — Needrom](https://www.needrom.com/download/hisense-a6l-hlte730t/) — 日付違いの純正ファームウェア複数
- [Hisense A6L firmware support — RomProvider](https://romprovider.com/hisense-a6l-firmware-support/)
- [Hisense A6L HLTE730T — FindROM.info](https://www.findrom.info/hisense-a6l-hlte730t/)
- [fans.hisense.com 公式フォーラムスレッド](http://fans.hisense.com/thread-172687-1-1.html) — 一番権威のあるソース（Hisense自身のコミュニティサイト）だが、ダウンロードはフォーラムアカウントでの返信必須（「回复可见」）で塞がれている。可能ならやる価値あり——この端末のファームウェアのサードパーティミラーは他にあまり出回っていない。

**書き込みツール:**
- [`aimindseye/hisense-a9`](https://github.com/aimindseye/hisense-a9) — `Hisense`というOEMサブコマンドを実際に認識するパッチ済み`fastboot`バイナリの入手元（[アンロック手順](#アンロック手順)参照）。別のHisense機種向けだが、このバイナリ自体は機種固有ではなくOEM固有のもの。
- [`bkerler/edl`](https://github.com/bkerler/edl) — EDLで実際に動いた汎用SDM660用Firehoseローダーの入手元（[ブリック復旧](#ブリック復旧-edlと開腹不要のトリガーケーブル)節のローダーの話を参照）。
- [zmart Xiaomi Deep Flash Cable — Amazon.co.jp](https://www.amazon.co.jp/dp/B06XYP1J7N) —本リポジトリの作業全体を通して実際に使った、開腹不要のEDLトリガーケーブル。

**MCFG / `mcfg_sw.mbn` ツールとフォーマット関連の参考資料:**
- [`sbaresearch/mbn-mcfg-tools`](https://github.com/sbaresearch/mbn-mcfg-tools) — 本リポジトリ全体で使っているextract/repack/ハッシュ検証ツール（`patches/mbn-mcfg-tools-windows-path-fix.patch`参照）
- [`fenrir-naru/mbn_utils`](https://github.com/fenrir-naru/mbn_utils) — 同じフォーマット向けのより初期のシンプルなツール。READMEのダイジェスト/チェックサムに関する記述が、私たちがぶつかっていた「パッケージ丸ごとのインポート拒否」が既知の未解決の壁であることの最初の裏付けになった
- [`Biktorgj/mcfg_tools`](https://github.com/Biktorgj/mcfg_tools) — 同目的の別の独立実装。ここでは直接使っていないが知っておく価値はある
- [`JohnBel/QualcommMBNs`](https://github.com/JohnBel/QualcommMBNs) — 様々な機種のファームウェアから抽出された`mcfg_sw.mbn`キャリア設定の大規模コレクション。この端末/キャリア向けのものは無かったが、他の人の参考資料を探す場所として有用
- [`JohnBel/EfsTools`](https://github.com/JohnBel/EfsTools) — 下記の各種ガイドの多くが前提にしている、diagポート経由のWindows用EFS Explorerツールの元祖
- [`sm7150-mainline/firmware-xiaomi-courbet`](https://github.com/sm7150-mainline/firmware-xiaomi-courbet/tree/main/lib/firmware/qcom/sm7150/courbet/modem_pr/mcfg/configs/mcfg_sw/generic/apac/rakuten/commerci) — 別機種（Xiaomi、SM7150）のオープンなファームウェアツリーにある、本物の楽天モバイル用`mcfg_sw.mbn`。チップセットが違うのでこの端末には書き込めないが、実際にキャリアが発行した楽天ポリシーの中身がどうなっているかの参考として`carrier_policy.xml`の内容が役立った
- [Qualcomm Modem Configuration w/ Carrier Policy (XML) — tech.ssut.me](https://tech.ssut.me/qualcomm-modem-configuartion-mbn-with-carrier-policy-description/) — `carrier_policy.xml`の要素一般についての背景解説

**VoLTE/VoWiFi有効化ガイド（XDA他）:**
- [\[Guide\] Enabling VoLTE/VoWiFi (deprecated) — XDA](https://xdaforums.com/rog-phone-2/how-to/guide-enabling-volte-vowifi-t4023529) — ROG Phone 2向けだが、「プロファイル集を片っ端から試して、どれでVoLTEが効くか見る」というテクニックを記録している元スレッド。これがまさに本リポジトリの手法3で楽天の問題を実際に解決した方法
- [Attempting to Enable VoLTE — XDA](https://xdaforums.com/t/attempting-to-enable-volte.3979009/) — 機種固有の内容だが、`persist.vendor.dbg.*`プロパティの試行錯誤ログとして有用
- [Getting VoLTE and VoWiFi on unlisted carriers by flashing mbn file — XDA](https://xdaforums.com/t/getting-volte-and-vowifi-on-unlisted-carriers-by-flashing-mbn-file.4467745/) — `EfsTools.exe uploadDirectory` / `mcfg_autoselect_by_uim`の手順。本リポジトリの手法2と根本は同じ技法だが、QPST自身のEFS Explorerの代わりにEfsToolsを使うバージョン
- [How to Enable VoLTE and VoWiFi in Unsupported Country — GetDroidTips](https://www.getdroidtips.com/enable-volte-vowifi-unsupported-country/) — `setprop`のみの方法とQPST/MBNの方法、両方の一般的な解説。本リポジトリの手法2がコミュニティの標準的アプローチと一致していることの裏付けにもなった
- [OnePlus 7T Pro VoLTE — gaddet.com](https://gaddet.com/posts/oneplus-7t-pro-volte/) — 別機種向けのPDC/EfsToolsの`mcfg_autoselect_by_uim`アプローチに言及
- [楽天モバイル(楽天UN-LIMIT)対応、VoLTEなカスタムROMを作る — ポイドの忘備録](https://solarisintel.hateblo.jp/entry/2021/06/03/102528) — 別機種向けの、AOSPソースからのカスタムROMビルドによるアプローチ（APNテーブル、`CarrierConfig`オーバーレイ、`config_device_volte_available`）。著者自身も完全動作には至っていないが、`mcc=440,mnc=11`のAPN詳細や、（モデム側ではなく）Android側のゲートとしての`carrier_volte_available_bool`の存在は知っておく価値がある

## 未解決の課題

- KDDI/docomoプロファイルが具体的になぜホーム以外のPLMNのデータPDNを拒否するのか（`OEM_DCFAILCAUSE_4`）——特定のNVアイテムに絞り込めていません。候補の一つ（`/nv/item_files/modem/mmode/is_plmn_block_req_in_lte_only_mode`）は試して実測で否定済みです。
- SoftBankのプロファイルが楽天SIMで動くのが「たまたま緩いポリシーだった」だけなのか、この2キャリアのMCC 440ポリシーの書かれ方に何か特有の事情があるのか——ブラックボックスな結果のまま未解析です。
- あるKDDI Activate時に`subMask`が`3`（DSDS）ではなく`1`（シングルSIM相当）になった——HWチェックのバイパスの副作用と思われます。デュアルSIM動作への影響は未検証です。

## 付録: 動作確認済みの環境バージョン

本リポジトリの内容は**2026-09-24**時点でこの組み合わせで動作確認済みです。ソフトウェアは日々更新されるので、うまくいかない場合はゼロからデバッグし直す前に、まずこのリストと自分の環境を見比べてみてください。

| コンポーネント | バージョン |
|---|---|
| 端末ファームウェア | `L1632.6.01.04`（`ro.build.fingerprint`: `Hisense/HLTE730T/HLTE730T:9/PKQ1.190723.001/L1632.6.01.04:user/release-keys`） |
| Android | 9（PKQ1.190723.001） |
| モデムファームウェアベースライン | `MPSS.AT.3.1-00819-SDM660_1.2` |
| Magisk | 30.7 |
| QPST | 2.7（`2.7.496.1`） |
