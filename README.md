# 象信 AI Agent Skills

让编程智能体（Claude Code、Codex、Cursor 等）学会正确使用 [象信 AI 系统一 API](https://docs.xiangxinai.cn)：把「分类 / 打分 / 是非判断 / 路由」这类决策交给象信一号，一次调用拿到带校准概率的结构化答案。

## 安装

**Claude Code**

```bash
claude plugin marketplace add xiangxinai/skills
claude plugin install xiangxin@xiangxinai
```

**其他智能体**

```bash
npx skills add xiangxinai/skills --skill xiangxin
```

也可以直接阅读 [`skills/xiangxin/SKILL.md`](skills/xiangxin/SKILL.md)，或把它放进你智能体的技能目录。

## 相关链接

- 文档：https://docs.xiangxinai.cn
- 控制台（每月赠送 ¥5 额度）：https://console.xiangxinai.cn
- Python SDK：`pip install xiangxin-sdk` · [源码](https://github.com/xiangxinai/xiangxin-sdk-python)
- JavaScript SDK：`npm install @xiangxinai/sdk` · [源码](https://github.com/xiangxinai/xiangxin-sdk-js)

许可：Apache-2.0
