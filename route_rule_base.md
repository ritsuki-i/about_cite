# research-references-tool `route.ts` ルールベースまとめ

対象ファイル: `research-references-tool/src/app/api/process/route.ts`

この API は、Notion データベース内でチェック済みの BibTeX を読み取り、論文タイトル、参考文献表記、論文種別、URL、PDF ファイル情報をルールベースで生成して Notion ページへ書き戻す。

## 処理対象

- リクエスト JSON から `token` と `databaseId` を受け取る。
- Notion データベースを検索し、チェックボックスが `true` のページだけを処理対象にする。
- 各ページの `BibTeX` rich text を連結し、1 件の BibTeX として解析する。

## BibTeX 前処理

- BibTeX 全体を `trim()` する。
- 最終行直前の余分なカンマを削除する。
- `@type{key part,` のように citation key に空白が混入している場合、空白を詰める。
- BibTeX のエントリタイプを先頭の `@...{` から取得し、小文字化する。
- BibTeX のタグ名はすべて小文字キーに正規化する。

## 論文種別マッピング

BibTeX の種別は、Notion 用の種別ラベルに変換される。

| BibTeX 種別 | Notion 上の扱い |
| --- | --- |
| `article` | 雑誌論文 |
| `inproceedings` | 会議論文 |
| `conference` | 会議論文 |
| `misc` | arXiv 論文 |

さらに、`article` でも `journal` が `Proceedings of ...` で始まる、または `conference` / `symposium` / `workshop` / `meeting` を含む場合は、実質的に `inproceedings` として扱う。

## タイトル整形

`title` は `toTitleCase()` により整形される。

- `{}` を削除する。
- 単語単位で Title Case にする。
- `a`, `an`, `the`, `and`, `but`, `or`, `for`, `nor`, `of`, `in`, `to`, `at`, `by`, `up` は、先頭・末尾・コロン直後でなければ小文字にする。
- ハイフン区切り語も各要素ごとに整形する。
- 既に内部大文字や略語を含む語は変更しない。
- コロン `:` の直後の語は大文字化する。

## ページ表記

`pages` がある場合は `formatPages()` で整形する。

- ハイフン周辺の空白を除去し、連続ハイフンを `--` に統一する。
- 先頭に `pp. ` を付与する。

例: `12 - 14` -> `pp. 12--14`

## LaTeX アクセント処理

著者名の処理前に、簡易的に LaTeX アクセント表記を Unicode 風の文字へ置換する。

- 対象アクセント: `\'`, ``\` ``, `\^`, `\"`, `\~`
- 対象文字: 主に `a/e/i/o/u/n` とその大文字
- 最後に `{}` を削除する。

注意: 現在のソースでは一部の置換文字列やコメントが文字化けしている。

## 著者名処理

著者は `author` を `and` で分割する。

### 日本語判定

- 先頭著者に `\u3000-\u9fff` の範囲の文字が含まれる場合、日本語著者として扱う。

### 英語著者の解析

`parseAuthor()` は以下のルールで `{ family, initials }` を返す。

- `Family, Given Middle` 形式:
  - カンマ前を姓とする。
  - カンマ後を名のパーツとして、各パーツの頭文字を `X.` にする。
- `Given Middle Family` 形式:
  - 最後の空白区切り要素を姓とする。
  - それ以前を名のパーツとして、各パーツの頭文字を `X.` にする。
- `Bickford Smith` については特別扱いし、姓を `Bickford`、名側に `Smith` を入れる。
- 姓に含まれる `Van`, `Der`, `De`, `Den`, `Ten`, `Ter`, `Von` は小文字化する。

### 著者数によるスライド用表記

- 1 名: `Family, I.`
- 2 名: `Family1, I. and Family2, J.`
- 3 名以上: `Family1, I., et al.`

### 通常参考文献用の英語著者リスト

- 1 名: `A`
- 2 名: `A and B`
- 3 名以上: `A, B and C`

日本語著者の場合は、空白と `, ` を削除した著者名をカンマ区切りで並べる。

## 会議名処理

### 会議名の正規化

`booktitle` は `normalizeConferenceName()` で整形される。

- 先頭の `in proceedings of` / `proceedings of` / `the` を削除する。
- 先頭の年を削除する。
- 先頭の序数表現、例: `37th` を削除する。
- 連続空白を 1 つにする。

### 会議略称マッピング

主要会議は固定マップで略称化される。

| 会議名 | 略称 |
| --- | --- |
| International Conference on Machine Learning | ICML |
| International Conference on Learning Representations | ICLR |
| Neural Information Processing Systems | NeurIPS |
| Unifying Representations in Neural Models | UniReps |
| AAAI Conference on Artificial Intelligence | AAAI |
| International Joint Conference on Artificial Intelligence | IJCAI |
| European Conference on Machine Learning and Principles and Practice of Knowledge Discovery in Databases | ECML PKDD |
| Conference on Knowledge Discovery and Data Mining | KDD |
| International Conference on Artificial Intelligence and Statistics | AISTATS |
| Conference on Uncertainty in Artificial Intelligence | UAI |
| Conference on Computer Vision and Pattern Recognition | CVPR |
| IEEE/CVF International Conference on Computer Vision | ICCV |
| European Conference on Computer Vision | ECCV |
| International Conference on Data Mining | ICDM |
| International Conference on Pattern Recognition | ICPR |
| Annual Meeting of the Association for Computational Linguistics | ACL |
| Conference on Empirical Methods in Natural Language Processing | EMNLP |
| Conference on Learning Theory | COLT |
| International Conference on Autonomous Agents and Multiagent Systems | AAMAS |

### 会議略称の決定順

1. `booktitle` と固定マップのキーを小文字で比較し、どちらかがもう一方を含めばマッチとする。
2. マッチした場合は固定マップの略称を使う。
3. マッチしないが会議名に括弧 `(...)` がある場合、括弧内を略称として使う。
4. それ以外は以下の単語だけ短縮する。
   - `International` -> `Int.`
   - `Conference` -> `Conf.`
   - `Recognition` -> `Recognit.`
5. スライド用の会議略称が空の場合は、会議名の各単語の頭文字から initialism を作る。

### 会議名表示

- 会議名末尾に同じ略称が括弧付きで重複している場合は削除する。
- `booktitle` がない場合は、欠落メッセージを使う。

## 雑誌名処理

雑誌名は固定マップで略称を補完する。

| 雑誌名 | 略称 |
| --- | --- |
| Transactions on Machine Learning Research | TMLR |
| TMLR | TMLR |

`journal` が固定マップの正式名または略称に一致した場合、表示名は `正式名 (略称)` になる。

`article` で `journal` がない場合は、欠落メッセージを使う。

## arXiv 検索と補完

### URL 生成

arXiv の abs URL から以下を生成する。

- abs URL: `https://arxiv.org/abs/<id>`
- PDF URL: `https://arxiv.org/pdf/<id>.pdf`
- Notion の URL プロパティには、arXiv が見つかった場合 `https://www.alphaxiv.org/abs/<id>` を優先して入れる。

### 検索順

1. BibTeX に `eprint` がある場合、arXiv API の `id_list` で検索する。
2. `eprint` で見つからない場合、タイトル完全句検索 `ti:"<title>"` で最大 5 件検索する。
3. タイトル検索では、タイトルを小文字化し、英数字以外を空白に正規化して比較する。
4. 正規化タイトルが完全一致、または片方が片方を含む場合にマッチとする。

### arXiv メタデータ補完

`journal` がなく `eprint` がある場合は、arXiv API から追加情報を取得する。

- `arxiv:comment` があれば `journal` に入れる。
- なければ `arXiv preprint arXiv:<eprint>` を `journal` に入れる。
- `published` の先頭 4 桁を `year` に入れる。

## 参考文献表記

このコードでは 2 種類の参考文献表記を生成する。

- スライド用参考文献
- 通常参考文献

### 共通要素

- 年の短縮表記は `year` の下 2 桁を使う。
- 文献先頭のラベルはおおむね `[Surname, 'YY]` 形式。
- タイトルは Title Case 済みのものを使う。
- 巻号ページは `Vol.`, `No.`, `pp.` の順で付与する。

### arXiv 論文

スライド用:

```text
[Surname, 'YY] Author: Title, arXiv preprint arXiv:<eprint> (Year).
```

通常:

```text
Authors: Title, arXiv preprint arXiv:<eprint> (Year).
```

### 会議論文

スライド用:

```text
[Surname, 'YY] Author: Title, In Proceedings of Conference Name (Abbr), Vol. X, No. Y, pp. A--B (Year).
```

その後、スライド用では `Proceedings of ...` 部分を短い会議略称へ置換する追加処理がある。

通常:

```text
Authors: Title, In Proceedings of the Conference Name (Abbr), Vol. X, No. Y, pp. A--B (Year).
```

### 日本語論文

スライド用:

```text
[Surname, 'YY] Surname ほか: Title, Journal, Vol. X, No. Y, pp. A--B (Year).
```

通常:

```text
Authors: Title, Journal, Vol. X, No. Y, pp. A--B (Year).
```

### 英語雑誌論文

スライド用:

```text
[Surname, 'YY] Author: Title, Journal, Vol. X, No. Y, pp. A--B (Year).
```

通常:

```text
Authors: Title, Journal, Vol. X, No. Y, pp. A--B (Year).
```

`volume` がない `article` の場合:

```text
Authors: Title, Journal (Year).
```

### publisher がある文献

通常参考文献では、上記のどれにも当てはまらず `publisher` がある場合:

```text
Authors: Title, Publisher (Year).
```

### その他

どの詳細情報もない場合:

```text
Authors: Title (Year).
```

## 論文種別説明

Notion の種別欄には、基本種別ラベルに会場情報を補足する。

- `article`: `雑誌論文 (Journal Display Name)`
- `inproceedings`: `会議論文 (Conference Name)`
- その他: 種別マップの値、または BibTeX 種別文字列

## Notion への書き戻し

各ページに以下のプロパティを書き戻す。

- 論文名: 整形済みタイトル
- スライド用参考文献: `slideRef`
- 通常参考文献: `normalRef`
- 種別: `typeDesc`
- チェックボックス: `false`
- URL:
  - arXiv が見つかった場合は alphaXiv URL
  - それ以外で BibTeX に `url` があればその URL
  - どちらもなければ更新しない
- 論文 PDF:
  - arXiv PDF URL が見つかった場合、Notion の external file として追加
  - ファイル名はタイトルから Windows 禁止文字を除去し、最大 100 文字以内の `<title>.pdf`

## エラー処理

- arXiv API の HTTP エラー、XML でないレスポンス、XML パースエラーは `console.error` に詳細を出して処理を継続する。
- 全体処理で例外が出た場合は、HTTP 500 とエラーメッセージを JSON で返す。
- 正常終了時は成功メッセージを JSON で返す。

## 注意点

- ソース内のコメント、Notion プロパティ名、一部の文字列は文字化けしている。
- `author`, `title`, `year` など必須想定の BibTeX フィールドが欠けた場合の防御は限定的。
- `number` は会議論文のスライド用表記で `volume` がある場合だけ取得される箇所がある。
- arXiv タイトル検索は正規化後の包含一致なので、似たタイトルに誤マッチする可能性がある。
