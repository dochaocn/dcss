# maoxuan - 毛泽东思想 AI Skill

让 AI 学会用毛主席的思路来分析问题。

## 这是什么

这是一个为 AI 助手设计的思维框架 skill，提炼了《毛泽东选集》中的核心方法论，让 AI 在面对复杂问题时能够运用矛盾分析、实践论、实事求是等思想工具。

**不只是全文检索——更重要的是思维方法。**

## 六大核心框架

| 框架 | 来源 | 核心问题 |
|------|------|----------|
| 矛盾分析法 | 《矛盾论》 | 主要矛盾是什么？怎么转化？ |
| 实践论 | 《实践论》 | 事实是什么？怎么验证？ |
| 阶级分析法 | 《中国社会各阶级的分析》 | 各方的利益和立场是什么？ |
| 实事求是 | 《改造我们的学习》 | 有没有调查？有没有从实际出发？ |
| 群众路线 | 《关于领导方法的若干问题》 | 决策有没有脱离实际使用者？ |
| 战略战术 | 《论持久战》 | 怎么集中优势兵力、各个击破？ |

## 全文数据

包含《毛泽东选集》静火版 1-7 卷，共 242 篇文章，Markdown 格式，存储在 `references/maoxuan/volumes/` 下。

## 安装方式

将此仓库克隆到 Hermes Agent 的 skills 目录：

```bash
git clone https://github.com/Numberreport/maoxuan.git ~/.hermes/skills/maoxuan
```

## 使用示例

在对话中直接说：
- "用毛主席的思路帮我分析一下当前这个局面"
- "毛选里关于实践论怎么讲的？"
- "帮我用矛盾分析法梳理这个项目的问题"

AI 会自动加载 skill，按照矛盾分析→抓主要矛盾→看转化条件→具体问题具体分析的流程展开。

## 文件结构

```
maoxuan/
  SKILL.md              # skill 定义：思维框架 + 使用指南
  README.md             # 本文件
  references/
    index.json          # 242 篇文章的索引
    maoxuan/
      volumes/          # 7 卷全文，Markdown 格式
```

## 版本

静火版 1-7 卷。来源：[lindaye6/Selected-Works-of-Mao-Zedong-JingHuo-version](https://github.com/lindaye6/Selected-Works-of-Mao-Zedong-JingHuo-version)

## 许可

- `SKILL.md`（思维框架）采用 MIT 许可
- 毛选原文为静火版，来源：[lindaye6/Selected-Works-of-Mao-Zedong-JingHuo-version](https://github.com/lindaye6/Selected-Works-of-Mao-Zedong-JingHuo-version)
