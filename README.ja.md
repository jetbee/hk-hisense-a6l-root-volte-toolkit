*(English version: [README.md](README.md))*

# Hisense A6L (HLTE730T) — root化・永続unlock・VoLTE有効化

Snapdragon 660 (SDM660) 搭載の中国市場向け格安スマホ、**Hisense A6L (HLTE730T)** で、標準では一切対応していないVoLTE/IMSを、日本のMVNO/MNO SIM（mineo/KDDIおよび楽天モバイルで検証）向けに有効化するためのリバースエンジニアリング記録とツール一式。

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
- [オフラインで編集したmcfg_sw.mbnが拒否される理由](#オフラインで編集したmcfg_swmbnが拒否される理由)
- [このリポジトリに含まれるツール](#このリポジトリに含まれるツール)
- [未解決の課題](#未解決の課題)

## 端末の背景情報

- **機種:** Hisense A6L、型番`HLTE730T`（`HLTE730T.B1`、`.B6`表記も確認）
- **SoC:** Qualcomm Snapdragon 660 (SDM660)
- **モデムファームウェアベースライン:** `MPSS.AT.3.1-00819-SDM660_1.2`（QPSTで確認可能）、Policy Manager XMLのヘッダには`mmcp.mpss/8.1.1`と記載
- **Android:** 9.0、中国市場向けファームウェア、Googleアプリなし、モデムの実力とは無関係にAndroidフレームワーク層でVoLTEがゲートされている
- EDL用Firehoseローダー: `prog_emmc_ufs_firehose_Sdm660_ddr_30060000.elf`

## 永続root化・ブートローダーアンロック

1. OEMの`fastboot Hisense unlock`コマンドで一時アンロック。**素の platform-tools の`fastboot`は`Hisense`というOEMサブコマンドを認識しません** — これに対応したカスタム/パッチ済みfastbootバイナリが必要です（Hisense専用fastbootツールを検索してください。本リポジトリでは再配布していません）。
2. `fastboot erase avb_custom_key` — これが実際に不可逆な本アンロック処理です。この端末では、これ単体ではuserdataの消去も確認ダイアログの表示も**行われません**（他機種向けの一部ガイドとは異なる挙動）——自分で`fastboot erase userdata`も実行するか、結果として出る「Decryption Unsuccessful」のリカバリー画面から手動で初期化する覚悟をしてください。
3. Magiskでパッチした`boot.img`と、`--disable-verity --disable-verification`付きでフラッシュした`vbmeta.img`を書き込みます。
4. これで本当のコールドブートを経ても永続します（手順2を省いた一時アンロックのみだと、`fastboot continue`一回分しか有効にならず、本当の再起動で元に戻ります — ここの見極めにかなり時間を使いました）。

## ブリック復旧: EDLと開腹不要のトリガーケーブル

上記のどこかで何かがおかしくなった場合の最終手段がEmergency Download Mode (EDL)です。通常EDLへの突入には端末内部のテストポイントをショートさせる必要があります（分解が必要）。**「deep flash cable」「EDLケーブル」「test point cable」**などで検索してみてください——Snapdragon端末向けに、余ったピンに抵抗を仕込んで接続するだけでEDLに強制突入させるUSBケーブルが売られています。安価で、まさにこの用途のために広く流通しているので、必要になってからではなく始める前に買っておく価値があります。

EDLに入ったら、上記のFirehoseローダーを使って`edl.py`（またはQPST付属の`QSaharaServer`/`fh_loader`）で`boot`、`vbmeta`、`system`、モデムパーティションを生で読み書きします。このセッションでハマった制約:
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

かなりのパッチ作業を経た後、実際に楽天モバイルSIM（MCC/MNC `440-11`）でVoLTEと一般データの両方を解決したのは、拍子抜けするほどシンプルな方法でした: **無改造の別のキャリアプロファイルをそのままActivateして様子を見る。** これは他の人が別のSnapdragon端末向けに独立して書いているのと同じテクニックです（例えばあるXDAガイドは、プロファイル集を片っ端から試して「どのキャリアプロファイルで誰のVoLTEが有効になるか」を記録する、という同じ手法を推奨しています——MCFGキャリアポリシーは特定のオペレーターではなく**国（MCC）単位**でスコープされていることが多く、自分とは無関係な別キャリアのプロファイルが自分の国向けVoLTEを解除してしまうことが頻繁にある）。

具体的には、この端末に楽天SIMを挿した状態で:

| ロードしたプロファイル（すべて無改造） | VoLTE / IMS PDN | 一般データ（デフォルトAPN） |
|---|---|---|
| `apac/kddi` | 動く | **失敗**、`OEM_DCFAILCAUSE_4` |
| `apac/dcm`（NTT docomo） | 動く | **失敗**、同じ原因 |
| `common/row`（汎用ROW） | 動かない | 動く |
| **`apac/sbm`（SoftBank）** | **動く** | **動く** |

パターンとして: 試した2つの日本MNO専用プロファイル（KDDI、docomo）は、どちらも自社のPLMNをVoLTE適格リストとしてホワイトリスト化する`carrier_policy.xml`を持っており、かつ実測では、そのホワイトリスト外のSIMに対する一般データPDN確立もブロックしていました——これは端末側のバグというより、実際のMNOキャリアポリシーに意図的に組み込まれた不正/ローミング悪用防止的な挙動に見えます。汎用ROWにはそのようなホワイトリストがない（だから常にデータは通る）代わりに、国レベルのVoLTE有効化ディレクティブもありません。SoftBankのプロファイルにはどうやらどちらの制限もなく、楽天SIMに対してそのまま動作しました。**なぜ**そうなっているかまでは解析していません——純粋に実測で確認しただけです（`dumpsys telephony.registry`の`mVoLteServiceState`、`dumpsys connectivity`の`NetworkAgentInfo`が`extra: ims`と`extra: <apn>`の両方について`CONNECTED/CONNECTED`かつ`everValidated=true`になっていること、そして実際の`ping`疎通確認）。

**特定のSIM向けにVoLTEを追いかけていて、端末に複数の純正キャリアプロファイルが用意されているなら、パッチを書く前に全部試してみてください。** タダで、速くて、完全に元に戻せて（別のプロファイルで`MbnFileLoad`/`MbnFileActivate`をやり直すか、元のプロファイルに戻すだけ）、それだけで解決するかもしれません。

## オフラインで編集したmcfg_sw.mbnが拒否される理由

プロファイル総当たりで解決しなかった場合に備えて、実際に`carrier_policy.xml`を編集する必要が出たときに何と戦うことになるかを先に書いておきます。

`mcfg_sw.mbn`はELFでラップされたコンテナです（`readelf`/`pyelftools`で普通にパースできます）: セグメント0はハッシュテーブル/secure-boot的なセグメント、セグメント1には埋め込みのX.509的な証明書チェーン（OEMアテステーション用で、ペイロードとは無関係）、セグメント2（`PT_LOAD`）が実際のNVアイテムデータを`type,path-len,path,data-len,data`というTLVレコードの単純な列として保持しており、最後は他セグメントの**32バイトダイジェスト**（SHA-256）を含むトレーラーアイテムで終わります。

[`sbaresearch/mbn-mcfg-tools`](https://github.com/sbaresearch/mbn-mcfg-tools)（このハッシュを単にコピーするのではなく本当に再計算しているPythonツール——無変更でのextract→repackの往復が元ファイルとバイト単位で完全一致することで確認済み）を使って確認したところ、**内部ハッシュが正しく再計算されていても、この端末のモデムで改変済みファイルが受理されるには不十分**でした: どんな編集をしたファイルも、ハッシュの妥当性に関わらず、`MbnFileLoad`のQMIインポート経路から`error code:-1`が返ってきます。このツールの作者自身も、[`fenrir-naru/mbn_utils`](https://github.com/fenrir-naru/mbn_utils)（同じ目的のもっと未完成な先行ツール）も、独立に同じ壁にぶつかったと記録しており、ファイル外部に保存された参照ダイジェスト（`sbaresearch`のREADMEには`mcfg_sw_config_digest_version`がEFSに別途保存されているという記述がある）か、あるいはこれらのツールが偽造を試みていない本物の暗号署名のどちらかが疑われています。

結論: **`.mbn`を手で編集して`MbnFileLoad`で再インポートしようとするのはやめましょう。** 代わりに (a) 既にActivate済みのconfig内の、既に宣言済みのファイル/アイテムをEFS Explorerでライブ編集する（手法2）か、(b) やりたいことを既にやってくれている純正プロファイルを探す（手法3）のどちらかにしてください。

`patches/mbn-mcfg-tools-windows-path-fix.patch`は上記の話とは無関係で、このツールのWindows固有のバグに対する小さな修正です（`/policyman/carrier_policy.xml`のような先頭スラッシュ付きのアイテムパスがそのまま`pathlib.Path()`に渡されると、Windowsでは相対パスではなく現在のドライブのルートに解決されてしまい、展開されたファイルのほとんどが静かに失われる）。上記のハッシュの壁とは別問題として、Windowsでこのツールを使うなら有用です。

## このリポジトリに含まれるツール

- `patches/ModemTestMode-MbnFileActivate.smali.diff` — HWチェックのバイパス（手法1）。これは素のAPKを自分でbaksmaliした出力に対するdiffであり、再配布バイナリではありません——パッチ済みdexは自分の端末から取得したファームウェアを元に自分で生成してください。
- `patches/mbn-mcfg-tools-windows-path-fix.patch` — `sbaresearch/mbn-mcfg-tools`向けのWindowsパス処理修正。
- `scripts/efs-explorer-automation-helpers.ps1` — QPST EFS Explorerのダイアログを操作するPowerShellマウス/キーボード自動化（手法2）。

含まれていないもの: Hisense/Qualcommの著作権があるバイナリ全般（純正・パッチ済み問わずAPK、`.mbn`ファイル、boot image）。上記のパッチを使って、自分の端末のファームウェアから自分で再生成してください。

## 未解決の課題

- KDDI/docomoプロファイルが具体的になぜホーム以外のPLMNのデータPDNを拒否するのか（`OEM_DCFAILCAUSE_4`）——特定のNVアイテムに絞り込めていません。候補の一つ（`/nv/item_files/modem/mmode/is_plmn_block_req_in_lte_only_mode`）は試して実測で否定済みです。
- SoftBankのプロファイルが楽天SIMで動くのが「たまたま緩いポリシーだった」だけなのか、この2キャリアのMCC 440ポリシーの書かれ方に何か特有の事情があるのか——ブラックボックスな結果のまま未解析です。
- 5GHz WiFiテザリング: この端末にはフランス/日本的なDFSチャンネルの規制上のゲーティングが実在の理由で存在しており（単なる人為的制限ではない）、jar差し替えによる最初のパッチ試行はブートループしました（EDLで復旧済み）。ブートループの根本原因: SystemServerClasspathのjarは、ModemTestModeパッチと同じ作法（適切なdeodex/再エンコード）が必要で、素朴なdex差し替えでは済まない——同じ手法での再挑戦はまだ行っていません。
- あるKDDI Activate時に`subMask`が`3`（DSDS）ではなく`1`（シングルSIM相当）になった——HWチェックのバイパスの副作用と思われます。デュアルSIM動作への影響は未検証です。
