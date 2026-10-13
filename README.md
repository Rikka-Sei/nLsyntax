# nLsyntax

一个控制自然语言输出风格的 skill：让回答用直白、具体的说法，少用需要额外解码的比喻和口号式词。

适用范围是所有写给人读的中文或英文自然语言：解释、建议、总结、报告、聊天、代码评审意见。

## 目录

```
.agent/skills/nLsyntax/
├── SKILL.md                      主文件：目标、判定标准、改写方法、约束表、维护说明
├── scripts/
│   ├── check.py                  检查脚本（128 条规则）
│   └── rules.json                规则数据：pattern / level / category / exclude / hint
├── references/
│   ├── constraint-table.md       由 check.py --table 生成的完整约束表
│   ├── experiment.md             对照实验记录
│   └── changelog.md              版本迭代记录
└── tests/
    ├── bad.txt bad2.txt bad3.txt   模型腔样例，预期全部命中
    ├── clean*.txt                  字面用法样例，预期 0 误报
    └── heldout.json                48 条留出评测词，不并入规则
```

## 三条判定标准

规则表外的词用这三问处理：

1. **字面问**：这个词的字面意思和想表达的意思是不是同一件事。不是 → 它是隐喻，看第 2 问。
2. **读者问**：不在这个行业、没读过英文技术博客的人，能不能一眼看懂它指什么。需要停下来想 → 改写。
3. **信息问**：换成直白说法会不会丢信息。不丢 → 改写。会丢 → 保留，第一次出现时用一句话说明。

成语和日常习惯说法（雪上加霜、双刃剑、冰山一角、走钢丝、天花板、卡脖子）通过这三问，直接保留。

## 规则规模

| 项 | 数量 |
|---|---|
| 规则总数 | 128 |
| 禁用级 | 74 |
| 慎用级 | 54 |
| 职场黑话 | 49 |
| 英译隐喻 | 36 |
| 体感隐喻 | 23 |
| 英文隐喻 | 14 |
| 伴随句式 | 3 |
| 咨询腔 | 3 |

`禁用` = 除例外情况都要改写。`慎用` = 先看上下文，字面用法或领域术语可以保留。

## 用法

普通聊天不需要运行脚本，按三问自检即可。处理较长的文字（超过约一千字的报告、文档）时用脚本先过一遍词表：

```bash
cd .agent/skills/nLsyntax
python3 scripts/check.py tests/bad.txt      # 检查文件
echo "文本" | python3 scripts/check.py -    # 从标准输入检查
python3 scripts/check.py --json tests/bad.txt   # JSON 输出
python3 scripts/check.py --table            # 打印完整约束表
```

退出码：`0` = 无禁用级命中；`1` = 有禁用级命中；`2` = 参数错误。

脚本没有命中不代表没问题，它只认识表里的词，表外的仍要走三问。

没有系统 Python 时（例如 macOS 上 `xcode-select` 未安装），用 nix：

```bash
nix-shell -p python3 --run 'python3 .agent/skills/nLsyntax/scripts/check.py tests/bad.txt'
```

## 测试结果

`nix-shell -p python3` 下运行六个样例，结果符合预期：

| 文件 | 禁用命中 | 慎用命中 |
|---|---|---|
| tests/bad.txt | 30 | 7 |
| tests/bad2.txt | 16 | 3 |
| tests/bad3.txt | 18 | 10 |
| tests/clean.txt | 0 | 0 |
| tests/clean2.txt | 0 | 0 |
| tests/clean3.txt | 0 | 0 |

bad 系列全部命中，clean 系列（含落地窗、右对齐、Unix 管道、承重墙等字面用法）0 误报。

## 维护

- 规则在 `scripts/rules.json`。每条包含：词、正则、级别、类别、字面语境排除词、改写提示。来自实验的规则标 `"source": "实验挖掘"`。
- 收新词：让模型在没加本 skill 的情况下回答一批真实问题，请不知情的评审标出需要解码的表达，把多次出现的收进规则。比凭空设想更准。
- `tests/heldout.json` 的 48 条不并入规则，用来检验对表外隐喻的处理能力。
- 新增规则后跑 `tests/`：`bad*.txt` 应全部命中，`clean*.txt` 应 0 命中。

  ```bash
  for f in tests/bad*.txt tests/clean*.txt; do echo "== $f"; python3 scripts/check.py "$f" | tail -1; done
  ```

- 排除词不能是被检词本身的一部分。例：给"温度"加排除词"度"，"度"包含在"温度"里，规则永远不触发。
- 修改规则后运行 `python3 scripts/check.py --table` 重新生成 `references/constraint-table.md`。
- 迭代记录见 `references/changelog.md`，实验记录见 `references/experiment.md`。
