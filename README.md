# fairdeckit

FairPlay付きIPAと16バイトのcredentialから、復号済みIPAまたはarm64バイナリを返すSwift Packageです。iOS/macOSアプリに組み込めます。

`decrypt.py`、`core.py`、`transform.c`の復号処理をSwiftに移植しています。Python・C実行ファイル・外部プロセス・ネットワーク接続は実行時に使いません。AES-CBCはCommonCrypto、MD5はCryptoKit、ZIP処理は[ZIPFoundation](https://github.com/weichsel/ZIPFoundation) 0.9.20以降を使います。変換テーブルはSPMリソースに含まれます。

credentialの取得は既存の[Pythonスクリプト](/Users/yyyywaiwai/fairplay-decrypt/credential-retrieval/README.md)で行います。

## アプリから使う

Xcodeで`FairDecKit`ディレクトリをローカルSwift Packageとして追加し、`FairDecKit` productをアプリのtargetに組み込みます。Swift tools 5.9以降、iOS 15以降、macOS 12以降が対象です。

### IPAを復号する

```swift
import Foundation
import FairDecKit

func decryptIPA(input: URL, output: URL, credentialFile: URL, mode: FairDecKit.Mode = .lowMemory) async throws -> URL {
    let credential = try Data(contentsOf: credentialFile)
    return try await Task.detached(priority: .userInitiated) {
        try FairDecKit.decryptIPA(at: input, to: output, credential: credential, mode: mode)
    }.value
}
```

`input`、`output`にはローカルのファイルURLを渡します。出力先にはハードリンクを使えるファイルシステムが必要です。出力先が既にある場合はエラーになり、入力IPAと既存の出力ファイルは上書きしません。新しい出力先ディレクトリは復号に失敗しても残ることがあります。途中で失敗した場合は一時IPAを削除し、出力先には公開しません。

ZIPはエントリ単位で再構築し、展開後の内容・権限・圧縮方式と、有効なZIP更新日時を引き継ぎます。無効な日付は正規化されます。追加フィールド・コメント・エントリ順は保持しません。APIは同期処理です。アプリでは上の例のようにメインアクターの外で呼び出してください。

### 処理モードを選ぶ

`decryptIPA`と`decryptBinaries`は`mode: FairDecKit.Mode`を受け取ります。省略時は`.lowMemory`です。

| モード | 処理方法 | 向いている用途 |
| --- | --- | --- |
| `.lowMemory` | 一時ファイルへ分割展開し、4KBずつ復号 | iOSや大きいIPAなど、メモリを抑えたい場合 |
| `.fast` | エントリ全体をメモリに読み、エントリ用の一時ファイルを使わず処理 | メモリに余裕があり、ディスクI/Oを減らしたい場合 |

```swift
try FairDecKit.decryptIPA(at: inputURL, to: outputURL, credential: credential, mode: .fast)
```

両モードとも、入力・既存出力の保護、CRC/サイズ検証、出力権限、復号する内容は同じです。`.fast`でもIPAの出力は非公開の一時IPAへ書き、成功後に公開します。大きなバイナリでは暗号化データと復号結果の両方を保持するため、数百MB以上を使います。

### IPAからバイナリだけを取り出す

```swift
let binaries: [String: Data] = try FairDecKit.decryptBinaries(in: inputURL, credential: credential, mode: .lowMemory)
```

辞書のキーはIPA内のパス、値は復号済みMach-Oです。本体・拡張機能・Frameworkをまとめて返し、IPAファイルは書き出しません。`.lowMemory`では復号中の暗号化バイナリを全体バッファに保持しませんが、両モードとも返すバイナリはすべて`Data`としてメモリに保持します。

### Mach-Oを直接渡す

```swift
import Foundation
import FairDecKit

func decryptBinary(binary: Data, sinf: Data, supf: Data, credential: Data) throws -> Data {
    try FairDecKit.decryptBinary(binary, sinf: sinf, supf: supf, credential: credential)
}
```

このAPIは既にメモリ上にある`Data`を受け取るため、モード指定はありません。

`binary`はMach-O全体、`sinf`と`supf`は対応する`SC_Info`ファイルの内容です。IPA APIは`.v5.supf`を優先し、対応する`.v4.sinf`、通常の`.sinf`、親appのSINFの順でこれらのレコードを自動で選択します。`cryptid == 0`のバイナリはIPA内ではそのまま残し、バイナリ単体APIでは`FairDecKitError.notEncrypted`になります。

## 対応形式と制限

- thin／FAT32／FAT64 Mach-Oのarm64スライス。arm64スライスは1つ、暗号化ページは4,096バイト、`cryptid == 1`が対象です。他アーキテクチャのスライスは変更しません。
- version 3／5のSUPF（識別子`507`、`510`、`612`）。arm64 subtype 0のレコードを扱います。
- 448バイト、または末尾8バイトがゼロの440バイトSINF `priv`レコード。
- credentialはそのIPAに対応した16バイトが必要です。別アカウント等のcredentialはSINF検証エラーになります。

`.lowMemory`のIPA出力では各エントリを非公開の一時ファイルへ64KBずつ展開し、暗号領域を4KBずつ復号してZIPへ戻します。バイナリ全体の入力・出力バッファは保持しません。SINF/SUPFとZIPのメタデータはメモリに保持するため、使用量はこれらのサイズにも依存します。最大エントリ程度の追加一時ディスク領域が必要です。シンボリックリンクの内容は64KBまでです。通常のファイルと異なりZIPFoundationが一括で読むため、この上限を設けています。

再署名・インストールは行わず、バイナリ変更後のコード署名も再利用できません。復号範囲はPython版と同じです。ページ暗号は認証付き暗号ではないため、暗号化前の原本との一致を証明するAPIではありません。

## モード別の測定例

macOS arm64のReleaseビルドで、同じIPAの復号と全12,946エントリの照合を行った1回ずつの測定です。コンパイラを含めず、XCTestプロセスを`/usr/bin/time -l`で測っています。

| 指標 | `.lowMemory` | `.fast` |
| --- | ---: | ---: |
| 最大RSS | 51.5MiB | 437.0MiB |
| 復号・照合の経過時間 | 14.46秒 | 10.12秒 |

この入力では`.lowMemory`はメモリを抑え、`.fast`は一時ファイルのI/Oを減らして短時間で完了しました。使用量と速度は入力・端末・ストレージに依存します。`decryptBinaries`や`decryptBinary`が返す`Data`の使用量まで固定するものではありません。

## テスト

以降のコマンドは`/Users/yyyywaiwai/FairDecKit`で実行してください。別のディレクトリからは`swift test --package-path /Users/yyyywaiwai/FairDecKit`でも実行できます。

```sh
swift test
```

合成credentialだけを使い、Python/Cから得た既知の変換値、thin／FAT32／FAT64、SINF／SUPFの対応形式、IPA内のレコード選択、破損データ・上書き防止・失敗時の後片付けを確認します。実際のcredentialやIPAはテストリソースに含めません。

実IPAとPython版の出力を照合するテストは、次の4つの環境変数を指定した場合だけ実行します。

```sh
FAIRDECKIT_IPA=/path/to/input.ipa \
FAIRDECKIT_CREDENTIAL=/path/to/credential.bin \
FAIRDECKIT_REFERENCE=/path/to/python-output.ipa \
FAIRDECKIT_OUTPUT=/path/to/new-swift-output.ipa \
FAIRDECKIT_MODE=fast \
swift test -c release --filter testDownloadedIPAAgainstPython
```

`FAIRDECKIT_MODE`は`lowMemory`または`fast`を指定し、省略時は`lowMemory`になります。`FAIRDECKIT_OUTPUT`は存在しないパスにしてください。ZIP内の全エントリについて、展開後のSHA-256・CRC32・サイズ・権限をPython版の出力と照合します。
