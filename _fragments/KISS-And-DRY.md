---
layout: fragment
title: KISS And DRY
tags: [编程原则]
description: 
keywords: 
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

<p><span style="color: #007bff;"><strong>把一件事变得复杂很容易，把一件事变得简单很困难。</strong></span></p>

## KISS

**KISS: Keep It Simple and Stupid**

- 解释：保持简单，避免不必要的复杂设计。

- 思想：简单就是美。越简单的代码，越容易理解、维护和调试。

- 实践建议：
  
  - 避免过度设计：只实现当前需要的功能，不要为未来可能用到的功能编写过多的代码。
  
  - 优先考虑清晰度：代码应该易于阅读和理解，即使是其他人也能很快掌握。
  
  - 分解问题：将复杂的问题分解成更小的、更容易解决的子问题。

## DRY

**DRY: Don't Repeat Yourself**

- 解释：避免代码中的重复，不要在代码中重复编写相同的逻辑。

- 思想：任何一块逻辑都应该拥有单一的、明确的表示。

- 实践建议：

  - 提取公共代码：将重复的代码提取成函数或模块。
  
  - 使用配置文件：将重复的数据存储在配置文件中。
  
  - 利用模版和模式：使用设计模式和模版来避免代码重复。

KISS 和 DRY 这两个原则相辅相成。**KISS 强调代码的简单性，DRY 强调代码的复用性**。通过遵循这两个原则，可以编写出更加高效、可维护的代码。
