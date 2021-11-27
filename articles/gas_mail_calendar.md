---
title: "gmailからカレンダーに予定を自動生成、削除する"
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gas, googlecalendar, gmail, javascript, automation]
published: true
---
# 何をしたのか
- ゴルフレッスンの予約及びキャンセルのメールからカレンダーの予定を自動修正

# はじまり
一人で打ちっぱなしに行ってる分には全然上達していないことに気づき、オープン記念で会費が下がっていた[アースゴルフアカデミー](https://earthgolf.jp/)に入会する。

月額制、通い放題のため、割と頻繁に通うものの、ゴルフレッスンの予約をカレンダーに手作業で入れるのが面倒になる。
要因としては、他の会員がキャンセルして空いた枠に変えたい等の理由でキャンセルしてから予約し直すことも多いことが挙げられる。

予約、キャンセルの通知メールはテンプレートなのだから、カレンダーに自動的に予定を作成してくれてもいいのに…と思ったものの、
終了時刻が記載されてないせいか自動的に作成はしてくれなかった。

ダメなら自分で書けばいいんじゃないのかと思って、調べてみたらできそうだったので、GASにはじめて触ることにした。

# メールからカレンダーに予定を作成し、削除する
GASを触ってみるにあたって、[GASを使ってGmailにきた予定をGoogle Calendarに自動で登録する](https://aakira.app/blog/2021/01/gas-gmail-to-calendar/)を参考にした。
スクリプトは[こちら](https://github.com/yoshi65/gas_utils/blob/main/mail2calendar/mail2calendar_several_mails_per_day.js)。
対象としているメールは以下。

```
xx xx 様

お申込みありがとうございます。
下記日程でレッスンの受付をいたしました。

日時：2021年xx月xx日(x) xx:00~

お問い合わせは下記までご連絡ください。
------------------------------------------------
アースゴルフxxxx
TEL : xx-xxxx-xxxx
E-MAIL : par@earthgolf.jp
------------------------------------------------
```

## GASを作成する
1. Google Driveを開く
1. 新規 > その他 > Google Apps Scriptを選ぶ

## 初回実行時に権限を許可する
1. `実行`をクリック
1. `権限を確認`をクリック
    ![](/images/gas_mail_calendar/gmc_confirmation.png)
1. 適切なアカウントを選択
1. `詳細`をクリック
    ![](/images/gas_mail_calendar/gmc_alert.png)
1. `<プロジェクト名>（安全でないページ）に移動`をクリック
1. `<プロジェクト名>が Google アカウントへのアクセスをリクエストしています`と表示されるので、`許可`をクリック

## 予約メールから自動的に予定を作成する
この部分は参考元ほぼそのままになっており、処理は以下のようになっている。
1. queryを指定して、スレッドを取得
    ```js
    const threads = GmailApp.search(query, 0, 5);
    ```
1. スレッドの各メッセージを確認
    ```js
    for (var i in threads) {
      const thread = threads[i];
      const messages = thread.getMessages()
      for (var j in messages) {
          // 処理
      }
    }
    ```
1. スターがついていない場合、以下の処理
    1. 予約情報をパース ([parseReserve](https://github.com/yoshi65/gas_utils/blob/4f76d1d4a1747a3e7206d7e4ae1f2272f5dd85f1/mail2calendar/mail2calendar_several_mails_per_day.js#L71-L92))
        1. メールに合わせて、パースの条件は書き換える必要あり
            1. `DATE_PREFIX` (i.e. `日時：`)
            1. [日時部分](https://github.com/yoshi65/gas_utils/blob/4f76d1d4a1747a3e7206d7e4ae1f2272f5dd85f1/mail2calendar/mail2calendar_several_mails_per_day.js#L6)
    1. 終了時刻を設定 (メール内に終了時刻の記載がないため)
    1. カレンダーに予定を作成 ([createEvent](https://github.com/yoshi65/gas_utils/blob/4f76d1d4a1747a3e7206d7e4ae1f2272f5dd85f1/mail2calendar/mail2calendar_several_mails_per_day.js#L36-L51))
    1. スレッドにラベルを追加
    1. メッセージにスターを追加
    ```js
    if (!messages[j].isStarred()) {
      callback(messages[j])
      label.addToThread(thread)
      messages[j].star()
    }
    ```

自動処理したらスターをつけると不便だなと思って試行錯誤したものの、
アーカイブやラベルはスレッドに対して付与するため、使用できず挫折。

1日に複数メールがこないことがわかっている場合、スレッドにならないので、ラベル付のみで対応できる。

## キャンセルメールから自動的に予定を削除する
予定を作成できるなら、削除もできるよな…と思って、取り組んだ。
処理は前項と同様であり、カレンダーに予定を作成の部分のみ異なる。

削除の処理は以下のようになっている。 ([deleteEvent](https://github.com/yoshi65/gas_utils/blob/4f76d1d4a1747a3e7206d7e4ae1f2272f5dd85f1/mail2calendar/mail2calendar_several_mails_per_day.js#L53-L69))
1. 指定した期間のイベントを取得
    ```js
    var calendarEvents = calendar.getEvents(startTime, endTime);
    ```
1. 各イベントのタイトルを確認
    ```js
    if (calendarEvent.getTitle() == TITLE) {
    ```
1. 該当タイトルであるイベントを削除
    ```js
    calendarEvent.deleteEvent()
    ```

## トリガーをセットする
実行する関数は`main`のままで、イベントのソースを`時間主導型`とする。

メールを受信するごとにトリガーできないため、このような対応となっている。

## `const`のサンプル
[const](https://github.com/yoshi65/gas_utils/blob/4f76d1d4a1747a3e7206d7e4ae1f2272f5dd85f1/mail2calendar/mail2calendar_several_mails_per_day.js#L1-L6)は対象のメールに合わせて設定する必要がある。
今回のアースゴルフアカデミーの場合、以下のようになる。
```js
const QUERY_RESERVE = 'subject:(レッスンの予約が完了しました。)'
const QUERY_CANCEL = 'subject:(レッスンの予約キャンセルが完了しました。)'
const TITLE = "ゴルフレッスン"
const LABEL = "アースゴルフアカデミー"
const DATE_PREFIX = "日時："
```

# 所感
- GASは難しくないし、便利。さっさとやっておけばよかった…
- メールは定形文が多いので、他にも転用していきたい。
    - スポーツジムのセントラルは入館手続き時にメールが来るので、それに合わせて[スクリプト](https://github.com/yoshi65/gas_utils/blob/main/mail2calendar/mail2calendar_one_mail_per_day.js)を作成した。
        - 時刻は[メール受信時刻](https://github.com/yoshi65/gas_utils/blob/4f76d1d4a1747a3e7206d7e4ae1f2272f5dd85f1/mail2calendar/mail2calendar_one_mail_per_day.js#L61-L71)
        - 滞在時間は[2時間](https://github.com/yoshi65/gas_utils/blob/4f76d1d4a1747a3e7206d7e4ae1f2272f5dd85f1/mail2calendar/mail2calendar_one_mail_per_day.js#L72)
        - 行ったことを忘れないという意味で快適 (あれ？前回いつ行ったっけってことはまあまあある)
- 継続できていることを実感することがモチベーションに繋がるので、回数カウントしてslackに通知するgasを書いた
    - 後日、記事にする予定

# Ref.
1. [GASを使ってGmailにきた予定をGoogle Calendarに自動で登録する](https://aakira.app/blog/2021/01/gas-gmail-to-calendar/)
