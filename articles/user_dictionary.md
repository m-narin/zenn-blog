---
title: "【Mac】ユーザ辞書を一括登録する"
emoji: "💻"
type: "tech"
topics:
  - Mac
  - GoogleAppsScript
  - GoogleSpreadSheet
published: true
published_at: 2024-12-02 12:00
---

# はじめに

こちらの記事では、Macbook のユーザ辞書を一括登録する方法について書きます。
ユーザ辞書とは、長いフレーズやよく使うフレーズ、難しい人名や検索候補に出てきにくい専門用語などを、短い言葉でショートカット入力できる便利機能です。

具体的な方法としては、スプレッドシートに、ショートカット入力と変換語句を記載していき、GAS で専用のファイルを生成し、自分のPCにダウンロードします。それを、Macbook のユーザ辞書に登録します。

# スプシと GAS

まず、スプシの A 行にショートカット入力、B 行に変換後の語句の一覧を記載します。
![スプシ.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/65eefc82-091d-51c2-3a26-503cd89b7bf1.png)

そして、以下の GAS を書きます。`YOUR_FOLDER_ID`の部分だけ、ダウンロードされる先の Google Drive のフォルダー ID(URL の末尾)に書き換えます。

```js:GAS
function exportToPlist() {
  var YOUR_FOLDER_ID = "YOUR_FOLDER_ID" // 保存先のフォルダIDを指定
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getActiveSheet();
  var data = sheet.getDataRange().getValues();

  var plist = '<?xml version="1.0" encoding="UTF-8"?>\n';
  plist += '<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">\n';
  plist += '<plist version="1.0">\n';
  plist += '<array>\n';

  for (var i = 0; i < data.length; i++) {
    var phrase = data[i][1];
    var shortcut = data[i][0];
    plist += '    <dict>\n';
    plist += '        <key>phrase</key>\n';
    plist += '        <string>' + phrase + '</string>\n';
    plist += '        <key>shortcut</key>\n';
    plist += '        <string>' + shortcut + '</string>\n';
    plist += '    </dict>\n';
  }

  plist += '</array>\n';
  plist += '</plist>';

  var folder = DriveApp.getFolderById(YOUR_FOLDER_ID);
  var file = folder.createFile('ユーザ辞書.plist', plist, MimeType.PLAIN_TEXT);

  var downloadUrl = file.getDownloadUrl();
  Logger.log('plistファイルが作成されました: ' + downloadUrl);
}
```

あとは、この関数を実行するだけで、ユーザ辞書一括登録用のファイルを生成できます。このファイルを自分の PC にダウンロードして準備完了です。plist ファイルとは、Apple のソフトウェア環境で設定情報やデータを保存するためのフォーマットです。

ちなみに 上記の例で生成される plist ファイルの中身はこのようになっています。

```xml:ユーザ辞書.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<array>
    <dict>
        <key>phrase</key>
        <string>ありがとうございます</string>
        <key>shortcut</key>
        <string>あり</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>abc@gmail.com</string>
        <key>shortcut</key>
        <string>メール</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>森田一義</string>
        <key>shortcut</key>
        <string>タモ</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>👀</string>
        <key>shortcut</key>
        <string>えも</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>🙏</string>
        <key>shortcut</key>
        <string>えも</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>😊</string>
        <key>shortcut</key>
        <string>えも</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>😭</string>
        <key>shortcut</key>
        <string>えも</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>😢</string>
        <key>shortcut</key>
        <string>えも</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>😄</string>
        <key>shortcut</key>
        <string>えも</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>🤔</string>
        <key>shortcut</key>
        <string>えも</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>🧐</string>
        <key>shortcut</key>
        <string>えも</string>
    </dict>
    <dict>
        <key>phrase</key>
        <string>😖</string>
        <key>shortcut</key>
        <string>えも</string>
    </dict>
</array>
</plist>
```

# ユーザ辞書に登録する

あとは、ユーザ辞書に登録するだけです。Macbook のシステム設定を開き、「キーボード」→ 「ユーザ辞書」を開きます。

![ユーザ辞書.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/271091da-3f61-630b-63df-5170efae6dfc.png)

ここに、先ほどダウンロードした plist ファイルをドラッグ&ドロップするだけでユーザ辞書を一括登録できます。

![スクリーンショット 2024-06-28 2.29.25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/4b23db21-1d7b-4941-9af4-f457052e7acf.png)

# おわりに

MacBook のユーザ辞書に一括登録する方法を書きました。ユースケースとしては、例えば、会社の人の名前のひらがなと漢字の一覧をスプレッドシートに書いておき、ユーザ辞書に一括登録するといったものが考えられます。こうすることで、名前の入力ミスを防げますね！他にも筆者の場合は、よく使う絵文字をすぐに入力できるようにしたりしています。

こちらの記事が皆さんのお役に立てれば幸いです。
