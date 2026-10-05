# Lenovo Legion Y700 — DeviceInfo / rollbackindex 診断

[WildKernels/GKI_KernelSU_SUSFS](https://github.com/WildKernels/GKI_KernelSU_SUSFS) をフォークし、Lenovo Legion Y700 の ABL が扱う **DeviceInfo と32個の rollbackindex を読み取るための診断用カーネル**を追加したリポジトリです。

カーネル起動初期に DeviceInfo の候補を保存する EarlyCapture と、ROM の `abl.elf` を照合して候補を選ぶ解析ツールを使用します。

## 確認済みの端末

| 端末 | ROM | 診断用カーネル | 実機確認 |
|---|---|---|---|
| Legion Y700 Gen2 / TB320FC | ZUX OS 1.1.350 | Android12 / 5.10.209 / Wild KernelSU | 起動・root・DeviceInfo取得・非ゼロのrollbackindex照合 |
| Legion Y700 Gen3 / TB321FU | ZUX 1.5.10.184 | Android14 / 6.1.112 / Wild KernelSU | 起動・root・DeviceInfo取得・fastboot表示との32個の値の照合 |

上記はテストに使用した構成です。カーネル系列の Android12 / Android14 は、端末の現在の OS 表示そのものを表すものではありません。他の ROM・端末での互換性は未確認です。

## 診断用ビルド

| 対象 | GitHub Actions の名前 | ワークフローファイル |
|---|---|---|
| Gen2 | `Build Gen2 A12 5.10.209 EARLYCAPTURE KSU` | `.github/workflows/build-gen2-a12-510209-earlycapture.yml` |
| Gen3 | `Build A14 6.1.112 EARLYCAPTURE` | `.github/workflows/build-a14-6112-earlycapture.yml` |

診断用構成では Wild KernelSU と `/dev/mem` を有効化し、SUSFS と STRICT_DEVMEM を無効化します。EarlyCapture の出力先は `/proc/abl_early_capture` です。Gen2 の DXEv2 は DXE Heap を含む検索範囲へ拡張しています。

この構成はメモリー調査用です。Wi-Fi などの全機能や、日常利用向けの動作を保証するものではありません。

### ビルドとダウンロード

1. **Actions** を開き、対象端末の診断用ワークフローを選択します。
2. **Run workflow** を押します。確認済み構成をビルドする場合、入力は既定値のまま、`variant` と `ksu_commit` は空、`build_bypass` は false にします。
3. 成功した実行の **Summary → Artifacts** から AnyKernel3 の ZIP をダウンロードします。
4. Gen2 の現行 KSU 構成は、成果物名に `Gen2-DXEv2-KSU` が含まれることを確認してください。

ワークフローを修正した後は、**Run workflow で新しい実行を作成**してください。古い実行の Re-run jobs は、その実行時のワークフローを使います。

## boot イメージの準備

テストでは、**対象端末・対象 ROM の純正 `boot.img`** のカーネルを、ビルドした `Image` に置き換えて使用しました。Gen2 と Gen3 の boot イメージを相互に流用しないでください。

arm64 の `magiskboot` を使う場合の再構築例です。これは Android 側で実行するコマンドです。

```sh
./magiskboot unpack boot.img
cp Image kernel
./magiskboot repack boot.img
```

出力は `new-boot.img` です。`magiskboot` は再構築用のツールであり、Magisk を組み込む操作ではありません。純正 boot を元にすることで、元の ramdisk を保持します。

今回の Gen3 では unlocked 状態で `fastboot boot` による一時起動を確認しました。Gen2 は `fastboot boot` が `unknown command` となったため、診断用 boot を対象スロットへ書き込んでテストしています。端末のスロットと元の boot を確認してから扱ってください。診断カーネルの導入・復元は解析ツールの機能には含まれません。

## KernelSU Manager

Gen2・Gen3 の診断用カーネルでは、管理アプリに **KernelSU_Next_v3.0.0_32857-release.apk** を使用してください。

[KernelSU Manager をダウンロード](https://github.com/KernelSU-Next/KernelSU-Next/releases/download/v3.0.0/KernelSU_Next_v3.0.0_32857-release.apk)

インストール後、EarlyCapture ログを取得する際は adb shell の root 権限を許可してください。

## EarlyCapture ログの取得

診断カーネルで起動し、KernelSU の管理アプリで adb shell の root を許可した後、**Windows のコマンドプロンプト**で実行します。

```cmd
adb shell uname -r
adb exec-out su -c "cat /proc/abl_early_capture" > abl_early_capture.txt
```

`/proc/abl_early_capture` が存在することと、ログのプロファイルが対象端末のものであることを確認してください。通常のカーネルにはこのファイルはありません。

ログに出る `candidate_rollback_u64_le` は、まだ候補を機械的に解釈した値です。**候補ごとの値をそのまま実際の rollbackindex と決めつけず、以下のツールで対象 ABL と照合してください。**

## 解析ツール（Windows）

別途配布する `rollback-tool-distribution.zip` を展開して使用します。ZIPにはソース・起動ファイル・説明書のみを含め、実機ログや個人の解析結果は含めません。

### 必要なもの

- Python 3.8 以上。標準ライブラリーだけを使うため、`pip install` は不要です。
- Windows の `powershell.exe` と Windows Forms。通常の Windows 10 / 11 では標準環境で利用できます。
- 対象 ROM の `abl.elf` と、同じ端末・ROM の EarlyCapture ログ。

Codex は不要です。`start.cmd` は `py -3`、`python`、`python3` の順で、実行可能な Python を探します。WindowsApps 経由の Python も実行確認して使用します。

### ファイル選択で実行

`start.cmd` をダブルクリックし、まず `abl.elf` を選択します。取得ログも解析する場合は Yes を押し、次に `abl_early_capture.txt` を選択します。

### 引数で実行

```cmd
start.cmd "D:\ROM\image\abl.elf" "D:\capture\abl_early_capture.txt" "D:\analysis\deviceinfo"
```

| 引数 | 内容 |
|---|---|
| 第1引数 | 対象 ROM の `abl.elf` |
| 第2引数 | EarlyCapture の取得ログ。省略すると静的解析のみ |
| 第3引数 | 出力フォルダー。省略するとツール内の `results/日時` |

出力先を指定した場合は、そのフォルダーへ直接保存します。同じ出力先を再利用すると同名ファイルを上書きします。バッチから呼ぶ場合は `call start.cmd ...` を使い、画面を残したければ後に `pause` を記述してください。

Python から直接実行することもできます。

```cmd
python rollback_tool.py "D:\ROM\image\abl.elf" --capture "D:\capture\abl_early_capture.txt" --out "D:\analysis\deviceinfo"
```

### 出力ファイル

| ファイル | 内容 |
|---|---|
| `report.md` | 照合結果、配列位置、32個の値、DeviceInfo 全体 |
| `rollbackindex.csv` | index・オフセット・物理アドレス・10進数・16進数 |
| `result.json` | ABL の SHA-256、検出位置、照合結果 |
| `deviceinfo-list.txt` | 確認済み項目・ASCII文字列・全バイトの一覧 |
| `deviceinfo.bin` | 特定した DeviceInfo の生データ |
| `deviceinfo-all-bytes.csv` | 全バイトを1バイト1行で出力 |
| `deviceinfo-fields.csv` | 意味を確認できた項目 |
| `deviceinfo.json` | DeviceInfo の詳細データ |

DeviceInfo の出力は取得ログとの照合に成功した場合に生成します。ASCII 一覧は「文字として見えるバイトの列」を表示する機能です。短い文字列がバイナリ値から偶然抽出されることもあり、それだけで設定項目とは判断できません。

## 解析方法と制限

ツールは ABL の LZMA 展開、ARM64 PE の抽出、配列を読み書きする命令の解析を行います。取得候補と ABL 内の静的データを照合してロード位置を算出し、DeviceInfo の位置と一致する候補を選択します。

現在は **DeviceInfo 内 `+0x898` から32個の little-endian UINT64** という確認済み形式を対象にしています。ROM が変わっても位置を自動検出できる場合がありますが、すべての形式に対応するものではありません。取得範囲外・取得前に消去されたデータ、候補数上限による取りこぼし、未知の命令配置・構造変更などでは解析できない場合があります。

- 物理アドレスは取得時の値であり、OS・ABL・起動条件によって変わる可能性があります。
- 取得するのは ABL のメモリー上の情報です。eFuse や永続保存先を直接読んだ結果ではありません。
- unknown・不一致・曖昧な候補は拒否します。候補0件は「rollbackindex がすべて0」という意味ではありません。
- ツールは読み取り・解析のみです。flash、unlock、lock、rollbackindex の変更・消去は行いません。

## 配布について

ソース・README・起動ファイルのみを配布してください。`results`、実機のログ、`deviceinfo.bin`、`__pycache__` などは配布対象に含めないでください。DeviceInfo には未解析の情報も含まれます。

## 元プロジェクト

- [WildKernels/GKI_KernelSU_SUSFS](https://github.com/WildKernels/GKI_KernelSU_SUSFS)
- [WildKernels/Wild_KSU](https://github.com/WildKernels/Wild_KSU)
- [Android Verified Boot / AOSP](https://android.googlesource.com/platform/external/avb/)

この README は、このフォークで追加した診断用途を説明しています。元プロジェクトと各依存コンポーネントのライセンス・著作権表示は、既存の `LICENSE`、`THIRD_PARTY_NOTICES.md` および各配布元を参照してください。
