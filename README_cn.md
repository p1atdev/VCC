# VCC: View-oriented Conversation Compiler

（本文档由 Claude Code 翻译并经人工审核）

[English](README.md) | [简体中文](README_cn.md) | [日本語](README_jp.md)

"View-oriented Conversation Compiler for Agent Trace Analysis" 的官方实现 ([论文](https://lllyasviel.github.io/VCC-experiments/paper/vcc0331.pdf))

这个 repo 是日常使用的。要复现论文里的学术实验请看 [VCC-experiments](https://github.com/lllyasviel/VCC-experiments)。

VCC 是一个编译器，把你的对话日志 (Claude Code 的 JSONL) 编译成高效的、agent 友好的视图。装了之后你就再也不怕 Claude Code 的 `/compact` 了 - CC 终于能看到被 compact 掉的上下文的原始细节。也支持搜索你所有的 Claude Code 历史对话。

装完 VCC 之后一般会发生这些事情:

- 你发现如果早点装这个的话，可以省下好多因为跟 CC 的 `/compact` 较劲而浪费掉的钱。
- `/compact` + `/recall` 变成你最喜欢的组合技。
- 你开始想为什么 CC 不自带这个功能。
- 你删掉了你的多层 RAG 记忆系统、你的自进化 agent 技能记忆、还有另外 15 个 AGI 发明 - 如果你真有这些东西的话...

另外还有一篇[论文](https://lllyasviel.github.io/VCC-experiments/paper/vcc0331.pdf)证明 VCC 在某个学术 benchmark 上能提升 context learning 和 agent trace 分析和 harness 设计。

# 安装

目前只支持 Claude Code。Codex 和 OpenClaw 快了。

安装的话，把这段话复制粘贴到你的 Claude Code 里:

    Please help me install the skills from
    https://github.com/lllyasviel/VCC.git
    just clone it then follow the INSTALL.md

更新的话:

    Please help me update the skills from
    https://github.com/lllyasviel/VCC.git
    just clone it then follow the INSTALL.md

卸载的话:

    Please help me uninstall VCC skills by deleting
    `conversation-compiler`, `readchat`, `recall`, `searchchat`
    from my `.claude/skills`

# 基本用法

当 compact 触发的时候 (不管是自动还是手动)，你可以马上 `/recall`。

比如说，自动 compact 或者手动 `/compact` 之后，你会看到类似 `(... compacted)` 这样的东西。然后:

`/recall`

CC 会自动回忆所有内容，或者你可以

`/recall 我们回顾一下刚才的对话`

`/recall 刚才那六轮分析里面，哪次访问尝试导致了服务器三层日志的第二层崩溃？`

`/recall 把我刚才提到的关于用户调研的六个细节再过一遍`

`/recall 回顾一下我到目前为止的思路`

对于新对话，你可以用 `/searchchat` 或者 `/recall` (会自动 fallback 到搜索)。比如:

`/searchchat 上次我们讨论的那个浏览器监听系统里面验证码是怎么处理的？`

`/recall 上次讨论的那个 canvas 方案我们最后决定用 React 了吗？`

这些命令可以跨越几个月的对话历史找到并恢复原始聊天内容。

`/readchat` 是高级用法，看下面。

# 原理

一个 Claude Code 的 JSONL 长这样 (在你的 `~/.claude/projects` 里可以找到一堆这种东西):

```
{"type":"user","message":{"id":"msg_user1","content":"I have two pets.\nCan you write a P..."}}
{"type":"assistant","message":{"id":"msg_asst1","content":[{"type":"thinking","thinking":"The
user wants a pet tracking module.\nThey have a dog (Buddy) and a cat (Whiskers).\nLet me chec
k if there's an existing file first...
```

会被编译成这些视图:

## UI View

模拟用户在 CC 里实际看到的东西

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

像 `(example.txt:18-20,23-25)` 这样的指针可以跳到下面 full view 里对应的具体行。

## Full View

完整的转录。行号只分配一次，所有视图共享。

假设这是 `example.txt`:

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

这个视图可以适配搜索查询比如 `--grep dog`。只有包含匹配的 block 会显示，但是对话结构会保留。每个 block 都有指回 full view 的指针。

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

`Write:tu02` 有 22 行代码 (41-62)，但是只有第 50 行 (`'dog'`) 匹配了: Cat 类 (56-62) 不在里面。指针 `41-62` 告诉 agent 去哪里读完整的 block。

## Transposed View `--grep dog`

同样的匹配结果，但是是一个平铺的列表。每个条目标记了它是什么 (用户消息、thinking、tool call 之类的) 以及在 full view 里的位置:

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

Adaptive view 保持对话顺序所以适合理解匹配周围的上下文。Transposed view 是平铺列表所以适合快速扫一遍所有匹配。所有指针都指向 full view。

# Q&A

### "又一个 agent 记忆系统？"

不是。记忆系统存的是预计算好的东西比如 summary、embedding、graph 之类的。那些结构和层级一般是静态的。而且大部分都在调 LLM 做摘要什么的...

VCC 不存任何东西。视图是动态的。它们从原始 JSONL 实时计算出来，用完就扔。(我们管这个叫 "projection"。)

### "那这不就是 grep 吗？"

完全不是。Grep 给你匹配的行，但是你分不清一个匹配到底是用户在说话、agent 在思考、tool call、还是 tool result。VCC 有 **block range pointers** 和 **block roles**。

有些人可能会追问 *"那把聊天日志拆成一个个消息文件然后用文件系统 grep 呢？structured grep 呢？我的 XXXX 数据库系统呢？"*

假设你从 adaptive view 跳到 full view 的某个 block 行号，周围的上下文就在那里。要从 file-per-message 的拆分里得到同样的效果，你需要一棵树来追踪层级关系，然后用一个链表来追踪时间顺序，两个都要维护，而且你还要决定多少个 "时间上相邻的" 东西才是真正正确的上下文。等你终于终于搞定的时候你基本上重新实现了一遍 VCC..

### "那这不就是 pretty-print 吗？"

它们非常不一样。Pretty-print 只是重新排版文本。VCC 是一个真正的编译器，有 lex, parse, IR, lower, emit。举几个例子:

* Lexer 在 parse 之前就把 `queue-operation`, `progress`, `api_error` 这种垃圾记录丢掉了
* Parser 把 tool call 参数从转义的 JSON blob 变成可读的 YAML (多行用 block scalar)
* Parser 还会把 Read tool 返回结果里的 `数字→` 前缀去掉来还原原始源码，也会把 base64 图片解码成文件
* 到 IR 阶段，被 compaction 分裂的 assistant 消息 (同一个 ID 但是多条 JSONL 记录) 会被重组成一个 section
* Lowering 会去掉 harness 注入的 XML (`<system-reminder>`, `<ide_opened_file>` 之类的)，过滤内部工具 (`TodoWrite`, `ToolSearch`)，清理 ANSI 转义码，隐藏纯 markup 的用户 turn
* Emitter 产出三个视图，共享一套行号坐标系

IR 阶段一次性分配行号来保证一切是一致的。之后 lowering 只能选择、截断、标注。行号不能重排或重新编号。所以跨视图的指针永远是一致的。

# Cite

    @article{zhang2025vcc,
      title={View-oriented Conversation Compiler for Agent Trace Analysis},
      author={Lvmin Zhang and Maneesh Agrawala},
      year={2025},
      url={https://github.com/lllyasviel/VCC}
    }
