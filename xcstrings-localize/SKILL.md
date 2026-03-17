---
name: xcstrings-localize
description: >
  iOS アプリの xcstrings (String Catalog) を起点としたローカライズスキル。
  xcodebuild でエクスポートした xcloc パッケージ内の XLIFF を解析し、
  未翻訳の trans-unit を対象言語に翻訳してインポート可能な状態にする。
  プロジェクトが対応する全言語を自動検出して翻訳する。
  ユーザーが「ローカライズ」「翻訳」「多言語対応」「xcstrings」「XLIFF」
  「xcloc」「○○語対応」に言及した場合に使用する。
metadata:
  author: Kenji Wada
  version: 1.0.0
  category: localization
  tags: [ios, xcstrings, translation, localization, swiftui]
---

# iOS 多言語ローカライズワークフロー

xcstrings (String Catalog) を起点とした iOS アプリの多言語対応を自動化する。
xcodebuild によるエクスポート/インポートと、XLIFF ファイルの AI 翻訳を組み合わせる。

## 全体フロー

```
xcstrings → xcodebuild -exportLocalizations → xcloc (XLIFF) → AI翻訳 → バリデーション → xcodebuild -importLocalizations → xcstrings
```

---

## ステップ 0: 対象言語の特定

翻訳対象の言語はプロジェクトの設定から動的に取得する。ハードコードしない。

### 方法 A: xcstrings から検出（推奨）

プロジェクト内の `.xcstrings` ファイルは JSON 形式で、`sourceLanguage` と各言語の翻訳情報を保持している。

```bash
# xcstrings ファイルを探す
find . -name "*.xcstrings" -not -path "*/.*"

# 対応言語の一覧を取得（sourceLanguage 以外のキーが翻訳対象）
python3 -c "
import json, sys
with open(sys.argv[1]) as f:
    data = json.load(f)
src = data.get('sourceLanguage', 'ja')
langs = set()
for key, val in data.get('strings', {}).items():
    langs.update(val.get('localizations', {}).keys())
langs.discard(src)
print(f'開発言語: {src}')
print(f'翻訳対象: {\" \".join(sorted(langs))}')
" Localizable.xcstrings
```

### 方法 B: Xcode プロジェクト設定から検出

```bash
# .xcodeproj 内の knownRegions を取得
grep -A 20 'knownRegions' *.xcodeproj/project.pbxproj | grep -oE '"[a-z]{2}(-[A-Za-z]+)?"'
```

### 方法 C: ユーザー指定

ユーザーが「英語とフランス語に翻訳して」のように明示した場合は、その言語のみを対象とする。

### 言語コードの対応表（参考）

よく使われる言語コード：
- `en` (英語), `zh-Hans` (中国語簡体字), `zh-Hant` (中国語繁体字)
- `ko` (韓国語), `fr` (フランス語), `de` (ドイツ語)
- `es` (スペイン語), `it` (イタリア語), `pt-BR` (ポルトガル語・ブラジル)
- `th` (タイ語), `vi` (ベトナム語), `id` (インドネシア語)
- `ar` (アラビア語), `hi` (ヒンディー語)

---

## ステップ 1: xcloc パッケージのエクスポート

ステップ 0 で特定した言語を `-exportLanguage` に指定してエクスポートする。

```bash
# 例: LANGUAGES 変数に検出結果を格納してエクスポート
xcodebuild -exportLocalizations \
  -project <プロジェクト名>.xcodeproj \
  -localizationPath ./Localizations \
  $(printf -- '-exportLanguage %s ' ${LANGUAGES})
```

### 注意事項
- プロジェクトの場合は `-project`、ワークスペースの場合は `-workspace` と `-scheme` を指定する
- `-exportLanguage` を省略すると開発言語のみエクスポートされる
- コマンドは全ターゲットをビルドするため、コンパイルエラーがあると失敗する
- 出力先に `Localizations/{lang}.xcloc/` が言語ごとに生成される

---

## ステップ 2: XLIFF ファイルの翻訳

### 2-1. 対象ファイルの特定

xcloc パッケージ内の XLIFF ファイルパスは以下の規則に従う：

```
{言語コード}.xcloc/Localized Contents/{言語コード}.xliff
```

### 2-2. 翻訳対象の判定

XLIFF ファイル内で以下の条件に合致する `<trans-unit>` を翻訳対象とする：

- `<target>` 要素が空、または `state="new"` である
- `<source>` 要素にテキストが存在する

```xml
<!-- 翻訳が必要な例 -->
<trans-unit id="greeting_message" xml:space="preserve">
  <source>こんにちは</source>
  <target state="new"/>
  <note>ホーム画面の挨拶メッセージ</note>
</trans-unit>

<!-- 翻訳済みの例（スキップ） -->
<trans-unit id="app_name" xml:space="preserve">
  <source>ヘルステイクアウト</source>
  <target state="translated">HealthTakeout</target>
</trans-unit>
```

### 2-3. 翻訳ルール

#### 絶対に守るべきルール
1. **`<source>` 要素を変更しない** — Xcode はインポート時に source でマッチングする
2. **`id` 属性・`original` 属性を変更しない**
3. **フォーマット指定子を正確に保持する**：`%@`, `%d`, `%lld`, `%1$@`, `%2$@`
4. **stringsdict の変数キーをそのまま保持する**：`%#@variable@`
5. **翻訳した `<target>` に `state="translated"` を設定する**
6. **`xml:space="preserve"` を維持する**

#### 翻訳品質のルール
1. `<note>` 要素がある場合、コンテキストとして翻訳に活用する
2. UI 要素のラベル（ボタン、タブ等）は簡潔に翻訳する
3. アプリ名・ブランド名は翻訳しない（references/glossary.yaml を参照）
4. 位置指定子（`%1$@`, `%2$@`）は語順に合わせて並べ替え可能だがインデックスは維持する

#### 言語固有のガイダンス

翻訳前に `references/language-guide.yaml` を読み込み、対象言語の以下の情報を確認する：
- **CLDR 複数形カテゴリ**: 言語によって必要なカテゴリ数が異なる
- **敬語・丁寧さのレベル**: デフォルトのフォーマリティ設定
- **テキスト長の傾向**: 日本語と比較した文字数の増減
- **特記事項**: RTL（右から左）レイアウト、文字体系固有の注意点等

`language-guide.yaml` に未掲載の言語が対象の場合は、CLDR の複数形ルールを調べて適用する。

### 2-4. 複数形・デバイスバリエーションの扱い

trans-unit の id に `|==|` セパレータがある場合、複数形またはデバイスバリエーションを示す：

```xml
<!-- 複数形の例 -->
<trans-unit id="item_count_%lld|==|plural.one" xml:space="preserve">
  <source>%lld件</source>
  <target state="new"/>
</trans-unit>
<trans-unit id="item_count_%lld|==|plural.other" xml:space="preserve">
  <source>%lld件</source>
  <target state="new"/>
</trans-unit>
```

**重要**: 複数形カテゴリはターゲット言語の CLDR ルールに従う。
ソース言語（日本語）には `other` のみだが、ターゲット言語によっては
`zero`, `one`, `two`, `few`, `many`, `other` が必要になる。
XLIFF エクスポート時に Xcode が必要なカテゴリの trans-unit を生成するため、
生成されたすべての trans-unit を漏れなく翻訳すること。

同一キーのデバイスバリエーション（`device.iphone`, `device.mac` 等）も全て翻訳する。

---

## ステップ 3: 翻訳後のバリデーション

翻訳完了後、**各言語の XLIFF ファイルに対して以下のチェックリストをすべて検証する**。
1件でも問題があれば、該当箇所を修正してからステップ 4 に進むこと。

### チェックリスト

#### 3-1. XML の整合性
- XLIFF ファイルが well-formed な XML であること
- 開きタグ・閉じタグの対応が正しいこと
- `&`, `<`, `>` 等の特殊文字が適切にエスケープされていること

#### 3-2. 未翻訳の残存
- `state="new"` の `<trans-unit>` が残っていないこと
- `<target>` が空のまま `state="translated"` になっていないこと

#### 3-3. プレースホルダーの一致
- 各 `<trans-unit>` の `<source>` と `<target>` で、以下が完全に一致すること：
  - フォーマット指定子の種類と数（`%@`, `%d`, `%lld` 等）
  - 位置指定子のインデックス（`%1$@`, `%2$@` 等）
  - stringsdict 変数キー（`%#@variable@`）
- **不一致がある場合**: 該当 trans-unit の id を報告し、target を修正する

#### 3-4. 翻訳禁止用語
- `references/glossary.yaml` の `do_not_translate` リストの用語が翻訳されていないこと
- アプリ名・ブランド名がそのまま保持されていること

#### 3-5. state 属性
- 翻訳済みの全 `<target>` に `state="translated"` が設定されていること
- 元から翻訳済み（スキップした）trans-unit の state が変更されていないこと

### バリデーション結果の報告

問題が見つかった場合は、以下の形式で報告してから修正する：

```
### バリデーション結果: {言語コード}

- ✅ XML 整合性: OK
- ❌ 未翻訳残存: 2件（greeting_message, settings_title）
- ⚠️ プレースホルダー不一致: 1件（item_count_%lld — %lld が target に欠落）
- ✅ 翻訳禁止用語: OK
- ✅ state 属性: OK

→ 修正後、再検証する。
```

---

## ステップ 4: xcloc パッケージのインポート

翻訳済みの xcloc パッケージを Xcode プロジェクトにインポートする。
**言語ごとに個別に実行する**（一括インポートは不可）。

```bash
for lang in ${LANGUAGES}; do
  xcodebuild -importLocalizations \
    -project <プロジェクト名>.xcodeproj \
    -localizationPath "./Localizations/${lang}.xcloc"
done
```

### 注意事項
- 問題のある翻訳はインポート時に警告が出るがスキップされる
- String Catalogs プロジェクトでは xcstrings の該当言語部分のみ更新される
- インポート後、Xcode で String Catalog を開いて結果を確認する

---

## 実行例

### 例 1: 全言語を自動翻訳

ユーザーが「ローカライズして」と言った場合：
1. xcstrings ファイルから対応言語を自動検出
2. 全言語をエクスポート → 翻訳 → バリデーション → インポート
3. 翻訳結果のサマリーを言語別に報告

### 例 2: 特定言語のみ翻訳

ユーザーが「フランス語とイタリア語だけ翻訳して」と言った場合：
1. 指定された `fr`, `it` のみをエクスポート
2. 2言語のみ翻訳 → バリデーション → インポート

### 例 3: 差分翻訳

ユーザーが「新しく追加した文字列だけ翻訳して」と言った場合：
1. 全言語をエクスポート
2. `state="new"` の trans-unit のみ翻訳（既存翻訳はスキップ）
3. バリデーション → インポート

### 報告フォーマット

翻訳完了後、以下の形式でサマリーを報告する：

```
## ローカライズ結果

| 言語 | 翻訳数 | スキップ | 警告 | ステータス |
|------|--------|---------|------|-----------|
| en   | 42     | 5       | 0    | ✅ 完了    |
| fr   | 42     | 5       | 1    | ⚠️ 要確認  |
| it   | 42     | 5       | 0    | ✅ 完了    |
```
