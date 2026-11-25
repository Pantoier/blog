---
title: "Chrome 网页翻译失败的排查与解决"
date: 2026-11-19T00:00:00-08:00
draft: false
tags: ["Chrome", "故障排查", "Google 翻译", "代理"]
categories: ["随笔"]
---

**时间：** 2026 年 11 月 19 日  
**环境：** macOS、Chrome、OpenClash、DMIT 代理节点

## 故障现象

Chrome 内置网页翻译突然失效，英文网页可以正常打开，但翻译时提示「无法翻译这个网页」，反复点击「再试一次」仍然失败。

![Chrome 网页翻译失败](/images/chrome-translate-troubleshooting/01-failure.jpg)

## 排查过程

1. Google 翻译网页版正常；无痕模式及其他英文网站的整页翻译均失败。
2. 打开 `chrome://translate-internals/#error-logs`，重新触发翻译，日志记录 `1: Network`。

   ![Chrome 翻译错误日志](/images/chrome-translate-troubleshooting/02-error-log.jpg)

3. 检查 OpenClash 连接记录，确认 `translate.googleapis.com:443` 命中代理规则，经 `DMIT-Direct-Reality` 传输数据。
4. 使用 curl 模拟翻译脚本请求，返回 `HTTP 302`，重定向至 `www.google.com/sorry/index`，表明该测试请求触发了 Google 反滥用检查。

   ![curl 返回 HTTP 302 及重定向地址](/images/chrome-translate-troubleshooting/03-curl.jpg)

## 解决方法

在 OpenClash 中临时切换至另一个代理出口，重新测试网页翻译，恢复正常。随后切回原 DMIT 节点，翻译仍然正常。

无需修改 DNS、代理规则或 Chrome 配置。

## 结论

故障表现为 Chrome 内置翻译网络请求异常，可能与 Google 临时反滥用检查或代理连接状态有关。curl 测试不能直接证明 Chrome 请求受到同样限制，具体根因未确认。

**再次遇到同类故障时，优先尝试切换代理出口并复测。**

<div id="tech-doc-chrome-translate" hidden></div>
<style>
/* 仅本篇技术文。1rem = html 的 --font-size-base，本站为 15px，不是浏览器默认 16px。 */
article:has(#tech-doc-chrome-translate) section.body {
  font-size: 1rem;
  line-height: 1.75;
}
article:has(#tech-doc-chrome-translate) section.body p,
article:has(#tech-doc-chrome-translate) section.body li {
  font-size: 1em;
  line-height: 1.75;
  margin: 0 0 0.8em 0;
}
article:has(#tech-doc-chrome-translate) section.body li {
  margin: 0 0 0.55em 0;
}
article:has(#tech-doc-chrome-translate) section.body li p {
  margin: 0 0 0.45em 0;
}
article:has(#tech-doc-chrome-translate) section.body li p:last-child {
  margin-bottom: 0;
}
article:has(#tech-doc-chrome-translate) section.body h2 {
  font-size: 1.2rem;
  line-height: 1.35;
  margin-top: 1.35em;
  margin-bottom: 0.5em;
}
article:has(#tech-doc-chrome-translate) section.body h2:first-of-type {
  margin-top: 1.1em;
}
article:has(#tech-doc-chrome-translate) section.body img {
  margin: 0.5em 0 0.7em;
}
article:has(#tech-doc-chrome-translate) section.body code {
  font-family: "Roboto Mono", "Courier New", monospace;
  font-size: 0.9em;
}
article:has(#tech-doc-chrome-translate) section.body pre {
  overflow-x: auto;
  max-width: 100%;
  line-height: 1.45;
}
article:has(#tech-doc-chrome-translate) section.body pre code {
  font-size: 100%;
}
</style>
