# VCC: View-oriented Conversation Compiler

（本ドキュメントは Claude Code による翻訳を人間がレビューしたものです）

[English](README.md) | [简体中文](README_cn.md) | [日本語](README_jp.md)

"View-oriented Conversation Compiler for Agent Trace Analysis" の公式実装 ([論文](https://lllyasviel.github.io/VCC-experiments/paper/vcc0331.pdf))

このリポジトリは普段使い用です。論文の学術実験を再現するには [VCC-experiments](https://github.com/lllyasviel/VCC-experiments) を参照してください。

VCC は会話ログ (Claude Code の JSONL) を効率的で agent フレンドリーなビューにコンパイルするコンパイラです。これで Claude Code の `/compact` はもう怖くない - CC がやっと compact されたコンテキストの元の詳細を見れるようになります。全ての Claude Code 会話を横断検索する機能もあります。

VCC をインストールすると大体こうなります:

- 早くインストールしていたらCCの `/compact` と格闘するために無駄にしたお金が節約できたことに気づく。
- `/compact` + `/recall` がお気に入りのコンボになる。
- なんで CC に最初から入ってないのか不思議に思い始める。
- 自分のマルチレイヤー RAG メモリシステム、自己進化 agent スキルメモリ、その他15個の AGI 発明を削除する - 本当に持ってたらの話だけど...

[論文](https://lllyasviel.github.io/VCC-experiments/paper/vcc0331.pdf)もあって、VCC がとある学術ベンチマークで context learning と agent trace 分析と harness 設計を改善することを示しています。

# インストール

現在 Claude Code のみ対応。Codex と OpenClaw は近日対応予定。

インストールするには、これを Claude Code にコピペしてください:

    Please help me install the skills from
    https://github.com/lllyasviel/VCC.git
    just clone it then follow the INSTALL.md

アップデート:

    Please help me update the skills from
    https://github.com/lllyasviel/VCC.git
    just clone it then follow the INSTALL.md

アンインストール:

    Please help me uninstall VCC skills by deleting
    `conversation-compiler`, `readchat`, `recall`, `searchchat`
    from my `.claude/skills`

# 基本的な使い方

Compact がトリガーされたら (自動でも手動でも)、すぐに `/recall` できます。

例えば、自動 compact や手動 `/compact` の後に `(... compacted)` みたいなのが表示されます。そしたら:

`/recall`

で CC が全部自動で思い出してくれます。もしくは

`/recall let's review our conversation`

`/recall in those six rounds of analysis we just did, which access attempt causes the second layer of the server's three-layer log to crash?`

`/recall go through all six details I mentioned about the user survey again`

`/recall review my thought process so far`

新しい会話では `/searchchat` か `/recall` (検索にフォールバックします) が使えます。例えば:

`/searchchat how did we handle the captcha in that browser listener system we discussed last time?`

`/recall did we decide to use React for that canvas solution we discussed last time?`

これらのコマンドで数ヶ月分の会話履歴から元のチャットを見つけて復元できます。

`/readchat` は上級者向けです、下記参照。

# 仕組み

Claude Code の JSONL はこんな感じです (`~/.claude/projects` に沢山あります):

```
{"type":"user","message":{"id":"msg_user1","content":"I have two pets.\nCan you write a P..."}}
{"type":"assistant","message":{"id":"msg_asst1","content":[{"type":"thinking","thinking":"The
user wants a pet tracking module.\nThey have a dog (Buddy) and a cat (Whiskers).\nLet me chec
k if there's an existing file first...
```

これが以下のビューにコンパイルされます:

## UI View

ユーザーが CC で実際に見たものをシミュレートします

```
[user]

I have two pets.
Can you write a Python module for tracking them?
One is a dog named Buddy and a cat named Whiskers.

[assistant]

Sure! Let me check if there's an existing file.

* Read "src/pets.py" (example.txt:18-20,23-25)

No existing file.
I'll create a module with Dog and Cat classes.

* Write "src/pets.py" (example.txt:40-63,66-68)

Created src/pets.py.
```

`(example.txt:18-20,23-25)` のようなポインタで下の full view の該当行にジャンプできます。

## Full View

完全なトランスクリプト。行番号は一度だけ割り当てられて、全ビューで共有されます。

これが `example.txt` だとします:

```
  1  [user]
  2
  3  I have two pets.
  4  Can you write a Python module for tracking them?
  5  One is a dog named Buddy and a cat named Whiskers.
  6
  7  ══════════════════════════════
  8  [assistant]
  9
 10  >>>thinking
 11  The user wants a pet tracking module.
 12  They have a dog (Buddy) and a cat (Whiskers).
 13  Let me check if there's an existing file first.
 14  <<<thinking
 15
 16  Sure! Let me check if there's an existing file.
 17
 18  >>>tool_call Read:tu01
 19  file_path: src/pets.py
 20  <<<tool_call
 21
 22  ══════════════════════════════
 23  [tool_error] Read:tu01
 24
 25  Error: file not found src/pets.py
 26
 27  ══════════════════════════════
 28  [assistant]
 29
 30  >>>thinking
 31  No existing file.
 32  I'll create Dog and Cat subclasses.
 33  The dog needs walk tracking.
 34  The cat needs nap tracking.
 35  <<<thinking
 36
 37  No existing file.
 38  I'll create a module with Dog and Cat classes.
 39
 40  >>>tool_call Write:tu02
 41  file_path: src/pets.py
 42  content: |
 43    class Pet:
 44        def __init__(self, name, species):
 45            self.name = name
 46            self.species = species
 47
 48    class Dog(Pet):
 49        def __init__(self, name):
 50            super().__init__(name, 'dog')
 51            self.walks = []
 52
 53        def log_walk(self, duration, distance):
 54            self.walks.append({'min': duration, 'km': distance})
 55
 56    class Cat(Pet):
 57        def __init__(self, name):
 58            super().__init__(name, 'cat')
 59            self.naps = []
 60
 61        def log_nap(self, duration, spot):
 62            self.naps.append({'min': duration, 'spot': spot})
 63  <<<tool_call
 64
 65  ══════════════════════════════
 66  [tool] Write:tu02
 67
 68  File written successfully.
 69
 70  ══════════════════════════════
 71  [assistant]
 72
 73  Created src/pets.py.
```

## Adaptive View `--grep dog`

ビューは `--grep dog` のような検索クエリに適応できます。マッチを含むブロックだけが表示されますが、会話の構造は保持されます。各ブロックには full view へのポインタがあります。

```
[user]

(example.txt:3-5)
  5: One is a dog named Buddy and a cat named Whiskers.

══════════════════════════════
[assistant]

>>>thinking
(example.txt:11-13)
  12: They have a dog (Buddy) and a cat (Whiskers).
<<<thinking

══════════════════════════════
[assistant]

>>>thinking
(example.txt:31-34)
  33: The dog needs walk tracking.
<<<thinking

>>>tool_call Write:tu02
(example.txt:41-62)
  50:           super().__init__(name, 'dog')
<<<tool_call
```

`Write:tu02` は 22 行のコード (41-62) ですが、マッチしたのは 50 行目 (`'dog'`) だけ: Cat クラス (56-62) は含まれていません。ポインタ `41-62` が agent にフルブロックの場所を教えます。

## Transposed View `--grep dog`

同じマッチ結果ですが、フラットなリストとして。各エントリにはそれが何か (ユーザーメッセージ、thinking、tool call など) と full view での場所がタグ付けされています:

```
(example.txt:3-5) [user]
  5: One is a dog named Buddy and a cat named Whiskers.

(example.txt:11-13) [thinking]
  12: They have a dog (Buddy) and a cat (Whiskers).

(example.txt:31-34) [thinking]
  33: The dog needs walk tracking.

(example.txt:41-62) [tool_call]
  50:           super().__init__(name, 'dog')
```

Adaptive view は会話順を保つのでマッチ周辺のコンテキストの理解に向いています。Transposed view はフラットリストなので全マッチを一気にスキャンするのに向いています。全ポインタは full view を指します。

# Q&A

### "また agent メモリシステム？"

違います。メモリシステムは事前計算されたもの、例えば summary、embedding、graph とかを保存します。それらの構造やレイヤーは普通は静的です。しかも大体 LLM を呼んで要約とかさせてます...

VCC は何も保存しません。ビューは動的です。元の JSONL からその場で計算されて、使ったら捨てられます。(これを "projection" と呼んでいます。)

### "それってただの grep では？"

全然違います。Grep はマッチした行を返しますが、そのマッチがユーザーの発言なのか、agent の思考なのか、tool call なのか、tool result なのか区別がつきません。VCC には **block range pointers** と **block roles** があります。

*「じゃあチャットログをメッセージごとにファイル分割してファイルシステム grep は？structured grep は？俺の XXXX データベースシステムは？」* と聞き続ける人もいるかもしれませんが、

Adaptive view から full view にブロック行番号でジャンプすると、周囲のコンテキストはそこにあります。file-per-message の分割から同じことをやろうとすると、階層を追跡するためのツリー的なものと時間順を追跡するためのリンクリストが必要で、両方メンテする必要があって、さらに "時間的に隣接する" もののうちどれだけが本当に正しい周辺コンテキストなのかも判断しないといけません。やっとやっと動くようにした頃には基本的に VCC を再実装しています..

### "でもこれ pretty-print でしょ？"

全く違います。Pretty-print はテキストを整形するだけです。VCC は本物のコンパイラで、lex, parse, IR, lower, emit があります。いくつか例を挙げると:

* Lexer が parse の前に `queue-operation`, `progress`, `api_error` などのゴミレコードを落とす
* Parser が tool call パラメータをエスケープされた JSON blob から読みやすい YAML (block scalar 付き) に変換する
* Parser は Read tool の結果から `数字→` プレフィックスを除去して元のソースを復元し、base64 画像もファイルにデコードする
* IR 段階で、compaction により分裂した assistant メッセージ (同じ ID だけど複数 JSONL レコード) が一つの section に再組立てされる
* Lowering が harness の XML (`<system-reminder>`, `<ide_opened_file>` など) を除去し、内部ツール (`TodoWrite`, `ToolSearch`) をフィルタし、ANSI エスケープコードをクリーンアップし、マークアップだけのユーザー turn を隠す
* Emitter が一つの行番号座標系を共有する三つのビューを出力する

IR 段階で行番号を一度だけ割り当てて全ての一貫性を保証します。その後 lowering は選択、切り詰め、注釈しかできません。行番号の並べ替えや再割り当てはできません。なのでビュー間のポインタは常に一貫しています。

# Cite

    @article{zhang2025vcc,
      title={View-oriented Conversation Compiler for Agent Trace Analysis},
      author={Lvmin Zhang and Maneesh Agrawala},
      year={2025},
      url={https://github.com/lllyasviel/VCC}
    }
