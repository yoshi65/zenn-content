---
title: "自作キーボードのデバッグにClaude Codeを使う"
emoji: "⌨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [自作キーボード, keyboard, claudecode, キーマップ, zmk]
published: true
---

## はじめに

自作キーボードをKeyball39からroBaに移行する際に発生したキーマップの問題と、トラックボールのcoropit換装で発生した問題を、Claude Codeを使ってトラブルシューティングした記録です。

:::message
本記事は「[Claude Code Advent Calendar 2025](https://qiita.com/advent-calendar/2025/claude-code)」13日目の記事です。
:::

### きっかけ

ふと思い立って購入してから、かれこれ3年近く[Keyball39](https://shirogane-lab.net/items/64b8f8693ee3fd0045280190)を愛用していました。
在宅勤務メインの中、有線接続に強めのテンティングで、QMKファームウェアでキーマップも自分好みにカスタマイズして快適に使っていました。

しかし、たまの外での作業や移動先での利用を考えるとさすがに「無線で使いたい」という欲求が高まり、[roBa](https://booth.pm/ja/items/6010869)を購入。ZMKファームウェア・Bluetooth対応のトラックボール付きキーボードです。

組み立ては完了したものの、キーマップの移行は後回しにしていました。
ファームウェアが違うため、容易に同じ使用感に辿り着けなかったこともあり…

**ところが、突如出張が決まりました。**

さすがに長時間ラップトップのキーボードはつら過ぎる…無線のroBaを持っていきたい。となると、キーマップを完成させる必要があります。
急いで移行作業を始めたのですが、思った通りに動かない箇所が続出。そこでClaude Codeが活躍しました。

### 直面した問題

QMKとZMKは似ているようで違う。ドキュメントをじっくり読み込まずに「なんとかなるだろう」と移行を始めたところ、問題が続出しました。

**キーマップの問題（QMK → ZMKの壁）**
- cmd+spaceで言語切り替えが効かない（早く押すとただのスペースになる）
- Shift+数字で記号（@や!など）が出ない
- Ctrl+Aが効かない
- そもそもビルドが通らない

**トラックボール換装の問題**
- 持ち運びを考慮して[coropit](https://booth.pm/ja/items/6830658)の25mmに換装したら、カーソルがゆらゆらするし遅い

経験者からすれば「それはそう」という内容かもしれませんが、初心者がハマりがちなポイントだと思います。

一つずつ手動で調べるのは大変です。ドキュメントを読み比べて、設定ファイルを見て、試行錯誤して...。

**ここでClaude Codeを使ってみることにしました。**

## Claude Codeによるトラブルシューティング

### キーマップの問題

#### cmd+spaceで言語切り替えが効かない

#### 症状を伝える

```
> cmd+spaceに言語交換を当てているが、早すぎるとただスペースが
> 入力されてしまい困っている
> これって設定の問題？それとも仕様としての限界？
```

#### Claude Codeの調査

Claude Codeは設定ファイルを読み込み、問題箇所を特定してくれました。

```
⏺ 原因がわかりました！

  問題の核心

  SPACEキーの設定が原因です：

  // SPACEキーの設定
  &lt_to_layer_0 2 SPACE

  // このマクロが実行される
  to_layer_0: {
      bindings = <&to 0 &macro_param_1to1 &kp MACRO_PLACEHOLDER>;
      //         ^^^^^ これが問題！
  }

  何が起きているか：

  1. CMDキーを押す（修飾キーとして認識される）
  2. SPACEキーを素早く押す（200ms以内にリリース→タップ判定）
  3. to_layer_0マクロが実行される
  4. &to 0がLayer 0に強制移動 → この時点でCMDの修飾キー状態がリセットされる
  5. その後SPACEが入力される
  6. 結果：cmd+spaceではなく、ただのspaceだけが入力される
```

#### Keyball39との比較

以前使っていたKeyball39のPDFチートシートを渡すと、比較してくれました。

```
⏺ 重要な発見

  Keyball39（QMK・有線）の実装

  LT(1, KC_SPC)  // シンプル！
  - タップ → ただSPACEを送信
  - ホールド → Layer 1を有効化
  - レイヤー切り替えマクロは使わない

  roBa（ZMK・無線）の現状

  &lt_to_layer_0 2 SPACE  // 複雑！
  - タップ → to_layer_0マクロ実行（&to 0 + SPACE）
  - ホールド → Layer 2を有効化
  - &to 0がCMD状態を壊している可能性大

  結論：
  roBaのSpaceキーを&lt 2 SPACE（シンプルなlayer-tap）に変更すれば、
  keyball39と同じ動作になり、cmd+spaceが正常に動作するはずです。
```

#### 解決策

```diff
- &lt_to_layer_0 2 SPACE  // カスタムマクロ使用
+ &lt 2 SPACE              // シンプルなlayer-tap
```

#### Shift+数字で記号が出ない

#### 症状を伝える

```
> ホールドSPACEでホールドZして、2から@打とうとしたら
> 2しかでないんだけど、これは当てている2が悪いのか、
> ホールドSPACEでレイヤー変えたらholdZのshiftが消えているのかどっち？
```

#### Claude Codeの回答

```
⏺ 原因がわかりました！

  問題2の答え：当てている2が悪い

  問題の原因：
  // Layer 2の設定（103行目付近）
  &kp KP_NUMBER_2  // ← テンキーの2を使っている！

  何が起きているか：
  - KP_NUMBER_2 = テンキーの2（Shiftが効かない）
  - Zのhold（Shift）はちゃんと押されている
  - でもテンキーの2はShiftを無視する仕様
  - だから2だけが出力される

  つまり：
  - ❌ Shiftの状態が消えている（これではない）
  - ✅ 2キーの設定が悪い（これが原因！）
```

#### 解決策

```diff
- &kp KP_NUMBER_2  // テンキーコード
+ &kp N2           // 通常の数字キー
```

Layer 2の全数字キー（0-9）を通常キーに変更しました。

#### Ctrl+Aが効かない

#### 症状を伝える

```
> ctr + Aが聞かなくて困っている
> keyballの時はhold ;をctrにして、動作していたはずが
> roBaにすると正常に動作していない
```

#### Claude Codeの調査

```
⏺ 問題を発見しました！

  現在の;キーの設定（84行目）：
  &mt LG(RIGHT_CONTROL) SEMICOLON

  これはCMD+Ctrlになっています！

  keyball39の設定：
  PDFでは;キーは単純に*Ctrl（mod-tap）でした。

  正しい設定：
  &mt RIGHT_CONTROL SEMICOLON

  LG(RIGHT_CONTROL)のLG（Left GUI = CMD）が余計です。
```

#### 解決策

```diff
- &mt LG(RIGHT_CONTROL) SEMICOLON  // LG（CMD）が余計
+ &mt RIGHT_CONTROL SEMICOLON      // Ctrlのみ
```

#### ビルドエラーの解決

キーマップを修正してプッシュしたら、GitHub Actionsでビルドが失敗しました。

#### エラー内容

```
error: too few arguments to function 'zmk_keymap_layer_activate'
    zmk_keymap_layer_activate(AUTOMOUSE_LAYER);
```

#### Claude Codeの調査

```
⏺ エラーが見つかりました！

  問題：
  - これはキーマップの修正とは無関係です
  - zmk-pmw3610-driver（トラックボールドライバー）が古いZMK APIを使っている
  - ZMKの最新バージョンでzmk_keymap_layer_activate()関数の引数が変わった
```

さらに調査を進めると、PR #2717まで特定してくれました。

```
⏺ 見つかりました！

  API変更の詳細

  PR #2717: "Allow layer behaviors to lock layers on"

  新しいAPI：
  int zmk_keymap_layer_activate(zmk_keymap_layer_id_t layer, bool locking);

  第2引数bool lockingが追加されました（レイヤーロック機能用）。

  API変更日：2025年11月14日（PR #2717マージ）
```

#### 解決策

```yaml
# config/west.yml
- name: zmk
  remote: zmkfirmware
  revision: f9be3ed  # API変更直前のコミット
```

一度失敗しましたが、正確なコミットを特定して再試行。

```
⏺ 🎉 ビルド成功！

  おめでとうございます！ファームウェアのビルドが完了しました。
```

### トラックボールの最適化

持ち運びを考えると少しでもコンパクトにしたい。roBa用に[coropit](https://booth.pm/ja/items/6830658)の25mmを購入しました。

#### 発生した問題

- カーソルがゆらゆらする
- 細かい動きが過剰に拾われてしまう
- 意図しないカーソル移動が発生

#### Claude Codeとの試行錯誤

#### 1回目の調整

```
- CPI: 400 → 280（感度を下げる）
- scroll tick: 16 → 22
- movement threshold: 2（微小な動きを無視）
```

→ **結果**: 遅すぎた

#### 2回目の調整

```
- CPI: 280 → 600
```

→ 25mmは物理的な移動量が小さいため、より高いCPIが必要だった

#### 3回目の調整（最終）

```
- CPI: 600 → 900
- movement threshold: 2 → 3
```

→ **結果**: 快適な速度と安定性を両立 ✅

#### 学び

- 34mm → 25mmへの換装は単純な置き換えではない
- ボールサイズが小さくなる = 同じ指の動きでの回転量が増える
- CPIを上げつつ、thresholdで微細なブレを除去するバランスが重要

## Claude Codeが活躍したポイント

### なぜClaude Codeが有効だったか

1. **設定ファイルの読み込みと解析**
   - keymapファイル、west.yml、ドライバーのソースコードまで横断的に調査
   - 問題の根本原因を特定

2. **QMKとZMKの違いを把握**
   - Keyball39のPDFチートシートを読み込んで比較
   - 「keyballの設定に近づければいけそう」という方針を実現

3. **GitHub連携**
   - `gh run list`、`gh run view --log`でビルドログを確認
   - PRの変更内容まで調査してAPI変更を特定

4. **試行錯誤のサイクル高速化**
   - 「これが原因では？」→「違った」→「じゃあこれは？」のループが早い
   - ドキュメントを読み込む時間を短縮

### 自作キーボード × Claude Code

ドキュメントを全部読み込まなくても、「動かない」→「Claude Codeに聞く」→「修正」のサイクルで問題を解決できました。ファームウェアの知識が浅くても、エラーの原因を特定して対処法まで導いてくれるのは心強いです。

## まとめ

### 解決した問題一覧

| 問題 | 原因 | 解決策 |
|------|------|--------|
| cmd+spaceが効かない | `&to 0`がCMD状態をリセット | `&lt 2 SPACE`に変更 |
| Shift+数字で記号が出ない | テンキーコード使用 | 通常の数字キーに変更 |
| Ctrl+Aが効かない | `LG(RC)`でCMDが混入 | `RC`のみに変更 |
| ビルドエラー | ZMK APIの変更 | バージョン固定 |
| トラックボールがゆらゆら | CPI/thresholdの調整不足 | CPI 900、threshold 3 |

### 感想

長期出張前の限られた時間で、Claude Codeのおかげで効率的にデバッグできました。

特に印象的だったのは、「キーマップの問題」と「ビルド環境の問題」を切り分けて、それぞれ適切な解決策を提示してくれたことです。GitHub ActionsのログからPRまで追跡してAPI変更を特定したのは、自分一人では時間がかかっていたと思います。

自作キーボードに興味はあるけどファームウェアで躊躇している人も、Claude Codeと一緒なら意外となんとかなります。ぜひ沼へどうぞ。

## 参考リンク

- [Keyball39](https://shirogane-lab.net/items/64b8f8693ee3fd0045280190)
- [roBa](https://booth.pm/ja/items/6010869)
- [ZMK Firmware Documentation](https://zmk.dev/docs)
- [coropit](https://booth.pm/ja/items/6830658)
