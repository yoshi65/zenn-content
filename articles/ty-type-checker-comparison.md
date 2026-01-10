---
title: "Python型チェッカー「ty」はただ爆速なのか？ ― mypy・pyright との比較検証"
emoji: "🔍"
type: "tech"
topics: ["Python", "ty", "mypy", "pyright", "型チェック"]
published: false
---

## TL;DR

Astral社（ruff 開発元）の新しい型チェッカー **ty** を mypy・pyright と比較検証しました。

### 結論

| 観点 | 結果 |
|------|------|
| 速度 | **ty が 11-17 倍高速** |
| 検出能力 | ty ≈ pyright > mypy |
| エラーメッセージ | ty が最も見やすい（Rust 風） |

### 誰におすすめか

| ユースケース | 推奨 |
|-------------|------|
| 新規プロジェクト | ✅ ty |
| CI/CD 高速化したい | ✅ ty |
| Pydantic/Django 使用 | △ 回避策あり、または 2026年の正式サポート待ち |
| 安定性重視 | ❌ mypy |

### 注意点

- **移行するとエラーが増える** - ty はデフォルトで mypy strict 相当（対応方法は後述）

---

## はじめに

2025年12月、ruff の開発元である Astral社が Python 型チェッカー **ty** のベータ版を公開しました。

本記事では、OSS プロジェクト（httpx, rich, requests, typer）を使って実際に検証した結果を報告します。

### ty とは

[ty](https://github.com/astral-sh/ty) は Astral社（uv, ruff の開発元）が開発した Python 型チェッカーです。

| 項目 | 内容 |
|------|------|
| 実装言語 | Rust |
| 状態 | Beta（2025年12月公開） |

**開発原則（[公式ブログ](https://astral.sh/blog/ty)より）**:

1. **Performance** - インクリメンタルアーキテクチャにより、ファイル編集時に必要な計算のみを再実行。PyTorch のような大規模プロジェクトでも、編集後の診断再計算が 4.7ms で完了（Pyright は 386ms）
2. **Correctness and ergonomics** - intersection 型、高度な type narrowing、到達可能性分析など、型理論を活用しつつ false positive を抑制
3. **Open development** - MIT ライセンス、コミュニティとの協調開発

## 検証環境

- Python 3.13
- ty 0.0.10
- pyright 1.1.408
- mypy 1.19.1
- ベンチマークツール: [hyperfine](https://github.com/sharkdp/hyperfine)

### インストール

```bash
# ty
uv tool install ty

# pyright
uv tool install pyright

# hyperfine (ベンチマーク用)
brew install hyperfine
```

### 基本的な使い方

```bash
# 型チェック実行
ty check src/

# ファイル変更を監視して自動再チェック
ty check --watch src/
```

### pyproject.toml

```toml
[tool.ty.environment]
python-version = "3.13"
```

### GitHub Actions

`--output-format github` でアノテーション表示できます：

```bash
ty check --output-format github src/
```

出力例：
```
::error title=ty (invalid-assignment),file=example.py,line=16::...
```

※ mypy/pyright はネイティブ対応がなく、専用の Action（[mypy-action](https://github.com/marketplace/actions/mypy-action)、[pyright-action](https://github.com/marketplace/actions/run-pyright)）が必要です。

## 速度比較

### 検証方法

公平な比較のため、以下の条件で測定しました：

- **キャッシュなし**: `--prepare 'rm -rf .mypy_cache .pyright'`
- **複数回実行**: hyperfine で 5 回測定

```bash
hyperfine --warmup 1 -i --runs 5 \
  --prepare 'rm -rf .mypy_cache .pyright' \
  'ty check <path>' \
  'pyright <path>' \
  'uvx mypy <path>'
```

### 結果（OSS 4プロジェクト、キャッシュなし）

| Project | ty | mypy | pyright |
|---------|-----|------|---------|
| httpx (60 files) | **51ms** | 890ms (17x) | 863ms (17x) |
| rich (190 files) | **95ms** | 1.01s (11x) | 1.57s (17x) |
| requests (36 files) | **58ms** | 648ms (11x) | 781ms (14x) |
| typer (16 files) | **46ms** | 623ms (13x) | 757ms (16x) |

### キャッシュあり時の比較

mypy はキャッシュに依存すると聞くので、キャッシュあり時も測定しました:

| Project | ty | mypy (キャッシュあり) | 倍率 |
|---------|-----|---------------------|------|
| httpx | **51ms** | 177ms | 3.5x |
| rich | **82ms** | 217ms | 2.6x |
| requests | **72ms** | 163ms | 2.3x |
| typer | **44ms** | 150ms | 3.4x |

mypy はキャッシュで大幅に高速化しますが、それでも ty より 2-3 倍遅いです。

※ pyright はキャッシュの効果がほぼなかったため省略

### 速度に関する発見

1. **ty は常に最速** - キャッシュの有無に関わらず
2. **mypy はキャッシュに依存** - キャッシュありで 2 倍差、なしで 7-17 倍差
3. **CI/CD のクリーンビルドでは差が顕著** - ty の優位性が際立つ

## 検出能力比較

### 意図的な型エラーテスト

15カテゴリ・25件の意図的な型エラーを含むファイルで検証：

```python
# 基本的な型の不一致
x: int = "string"  # ERROR

# Union 型のナローイング不足
def union_not_narrowed(x: int | str) -> int:
    return x  # ERROR: str の可能性がある

# Sequence vs 具体的な型
def sequence_mismatch(seq: Sequence[int]) -> int:
    return sum(seq)

s = {1, 2, 3}
sequence_mismatch(s)  # ERROR: set は Sequence ではない
```

| ツール | 検出数 |
|--------|--------|
| ty | **25** |
| pyright | **25** |
| mypy (strict) | **25** |

基本的な型エラーでは **3ツールとも同等の検出能力** でした。

### OSS プロジェクトでの検出数

実際のプロジェクトでは差が出ました：

| Project | ty | pyright | mypy (通常) | mypy (strict) |
|---------|-----|---------|-------------|---------------|
| httpx | 57 | 38 | 21 | 45 |
| rich | 45 | 57 | 16 | 16 |
| requests | 105 | 120 | 40 | **625** |
| typer | 102 | 110 | 39 | 76 |

### 検出に関する発見

1. **ty は mypy (通常モード) より 2-3 倍多く検出**
2. **ty と pyright は同程度の検出数**
3. **mypy strict は過剰検出の傾向** - requests で 625 件（通常モードは 40 件）

:::message
mypy `--strict` は `--disallow-untyped-defs` を有効にするため、型注釈のない関数すべてをエラーにします。古いプロジェクトでは大量のエラーが出ます。
:::

### なぜ検出数に差が出るのか？

**具体例で確認してみましょう：**

```python
class Response:
    status_code: int
    text: str

def make_request() -> Response:
    return Response()

def process():  # 型注釈なし
    resp = make_request()
    return resp.nonexistent  # 存在しない属性
```

| ツール | 結果 |
|--------|------|
| ty | ✅ `Object of type 'Response' has no attribute 'nonexistent'` |
| mypy | ❌ `Success: no issues found` |

`process()` に `-> str` を追加すると、mypy も検出します。

**mypy が少ない理由（[gradual typing の設計](https://mypy.readthedocs.io/en/stable/command_line.html)）**:
- 型注釈のない関数の中身はデフォルトでスキップ（`--check-untyped-defs` で変更可能）
- 外部ライブラリは `Any` 型として扱い、エラーを出さない
- 段階的に型を導入できる設計思想

**ty が多い理由（[Unknown 型による推論](https://docs.astral.sh/ty/reference/typing-faq/)）**:
- 型注釈なしでも `Unknown` 型を使って推論・チェック
- 外部ライブラリも型推論を試みる
- gradual guarantee を維持しつつ、より多くのコードをチェック

つまり、mypy の「0件」は「問題がない」ではなく「チェック対象外」の可能性があります。これは mypy の欠点ではなく、段階的導入を重視した設計です。mypy で同様のチェックを行うには `--check-untyped-defs` オプションを使用します。

## mypy からの移行時の注意点

### mypy 互換モードはない

**重要**: ty には mypy 互換モードがありません。ty はデフォルトで strict なので、移行するとエラーが増えます。

### mypy と ty の主な違い

| 項目 | mypy | ty |
|------|------|-----|
| デフォルト動作 | 型注釈なしはスキップ | **全てチェック** |
| プラグイン | Pydantic, Django 等あり | **なし** |
| ファイル全体 ignore | `# mypy: ignore-errors` | **不可** |
| type narrowing | 独自アルゴリズム | 異なる場合あり |

### Pydantic / Django / SQLAlchemy を使っている場合

ty にはプラグインシステムがないため、これらのライブラリを使うと多数のエラーが発生します。

**発生する症状**:
- Pydantic: `BaseModel`, `Field`, `TypedDict` 関連でエラー
- Django/SQLAlchemy: メタプログラミング起因のエラー

**回避策**:
```toml
[tool.ty.rules]
invalid-argument-type = "ignore"
invalid-key = "ignore"
invalid-parameter-default = "ignore"
```

ただし、これらを ignore すると ty の利点が薄れます。Astral は 2026年の Stable リリースで Pydantic/Django のネイティブサポートを計画しているため、それまで待つのも選択肢です。

### 設定の対応表

```toml
# mypy
[tool.mypy]
ignore_missing_imports = true
python_version = "3.13"
strict = true

# ty で同等にする場合
[tool.ty.environment]
python-version = "3.13"

[tool.ty.rules]
unresolved-import = "ignore"  # ignore_missing_imports 相当
```

### エラー抑制の方法

**1. インラインコメント**（`--add-ignore` と同じ形式）
```python
value = some_func()  # ty:ignore[invalid-argument-type]

# 複数ルール
value = some_func()  # ty:ignore[rule1, rule2]
```

**2. type: ignore も使える**（ただし全違反を抑制）
```python
value = some_func()  # type: ignore
```

**3. 関数単位でスキップ**
```python
from typing import no_type_check

@no_type_check
def legacy_function():
    ...  # この関数内は全てスキップ
```

**4. 設定ファイルでルール調整**
```toml
[tool.ty.rules]
invalid-argument-type = "warn"   # エラー → 警告に下げる
unresolved-import = "ignore"     # 完全無効化
```

**5. 一括で ignore を追加**

エラーが多い場合、`--add-ignore` で自動的にコメントを追加できます：

```bash
ty check --add-ignore src/
```

実行後：
```python
return "string"  # ty:ignore[invalid-return-type]
```

### mypy にはなかった作業が発生する

mypy では外部ライブラリは `Any` 扱いでエラーにならなかったため、ライブラリを追加しても型チェック対応は不要でした。

ty ではライブラリ追加・更新のたびに:
1. ty でエラーが出るか確認
2. 必要なら `ty:ignore` を追加

この作業が継続的に発生します。速度のメリットとこの手間のトレードオフを考慮して導入を判断してください。

## エラーメッセージ比較

同じエラーに対する各ツールの出力を比較します。

### 例1: Union 型のナローイング不足

```python
def union_not_narrowed(x: int | str) -> int:
    return x  # ERROR
```

#### ty

```
45 | def union_not_narrowed(x: int | str) -> int:
   |                                         --- Expected `int` because of return type
47 |     return x
   |            ^ expected `int`, found `int | str`
```

#### pyright / mypy

```
error: Type "int | str" is not assignable to return type "int"
```

### 例2: 引数の型エラー

```python
def wrong_argument(x: int) -> int:
    return x * 2

wrong_argument("string")  # ERROR
```

#### ty

```
35 | def call_with_wrong_argument() -> None:
37 |     wrong_argument("string")  # ERROR
   |                    ^^^^^^^^ Expected `int`, found `Literal["string"]`
   |
info: Function defined here
31 | def wrong_argument(x: int) -> int:
   |     ^^^^^^^^^^^^^^ ------ Parameter declared here
```

#### pyright / mypy

```
error: Argument of type "Literal['string']" cannot be assigned to parameter "x" of type "int"
```

### 例3: 存在しない属性

```python
class Person:
    name: str
    age: int

p = Person("Alice", 30)
print(p.email)  # ERROR
```

#### ty

```
148 |     """存在しない属性へのアクセス"""
149 |     p = Person("Alice", 30)
150 |     print(p.email)  # ERROR
    |           ^^^^^^^ Object of type `Person` has no attribute `email`
```

#### pyright / mypy

```
error: "Person" has no attribute "email"
```

ty は Rust コンパイラのようなエラー表示で、コードの文脈・矢印での位置表示・関数定義元の表示があり見やすいです。

## 総合評価

| 観点 | 推奨ツール | 備考 |
|------|-----------|------|
| 速度 | **ty** | 11-17x 高速 |
| 検出能力 | ty ≈ pyright | mypy より多く検出 |
| エラーメッセージ | **ty** | Rust 風で見やすい |
| 安定性 | mypy | 長年の実績 |

## まとめ

ty は「**速くて厳密**」な型チェッカーです。

- 速度は圧倒的（11-17 倍高速）
- 検出能力も高い（mypy より多くの問題を発見）
- エラーメッセージも見やすい

ただし、mypy では不要だった ignore 対応がライブラリ追加・更新のたびに必要になる可能性があります。速度のメリットとこの手間を天秤にかけて、導入を検討してください。

## 参考リンク

- [ty 公式ブログ](https://astral.sh/blog/ty)
- [ty ドキュメント](https://docs.astral.sh/ty/)
- [ty GitHub](https://github.com/astral-sh/ty)
- [How to migrate from mypy to ty](https://pydevtools.com/handbook/how-to/how-to-migrate-from-mypy-to-ty/)
- [hyperfine](https://github.com/sharkdp/hyperfine)
