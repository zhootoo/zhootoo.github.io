---
title: AI Agent Introduction
date: 2026-08-10 11:53:36
categories: [AI-Agent]
tags:
  - LLM
  - AI-Agent
---

## Agent = LLM + 上下文 + 工具
- LLM是Agent的大脑
它是Agent的决策内核，理解意图，思考规划，做出判断。LLM的能力来源于两部分：预训练积累的世界知识和语言能力和后训练所固化的决策策略。

- 上下文是Agent的眼睛
上下文是Agent在每个决策点所能看到的全部信息，包括当前的环境信息，用户记忆，领域知识，自身状态和任务进展。Agent的上下文窗口就是它当下能看到的一切。

- 工具是Agent的四肢
工具是Agent所能做的所有事情的集合-从预定义的工具调用到按需加载的Skills，从委托子Agent协作到主动与用户沟通响应外部事件。