---
title: "Z-Library 下载报错 Error 1034：故障排查与处理"
date: 2026-11-24T18:20:00-08:00
draft: false
tags: ["Chrome", "故障排查", "Cloudflare", "代理"]
categories: ["随笔"]
---

## 一、故障现象

在 macOS 环境下使用 Chrome 访问 Z-Library，可以正常登录、搜索图书及查看图书详情，但点击 EPUB 下载按钮后，请求跳转至下载域名 `dln1.ncdn.ec`，页面返回 Cloudflare 错误：**Error 1034 — Edge IP Restricted**。

![图 1：Chrome 下载请求返回 Error 1034（原始截图局部）](/images/zlibrary-download-error-1034/01-chrome-1034.jpg)

根据 [Cloudflare Error 1034 官方说明](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1034/)，该错误与目标 IP 地址的使用限制有关。仅凭错误页面，不能判定客户端公网出口 IP 被封禁。

## 二、运行环境

| 项目 | 配置 |
|---|---|
| 操作系统 | macOS |
| 故障浏览器 | Google Chrome |
| 对照浏览器 | Safari |
| 对照设备 | Mac、iPhone |
| 网络代理 | OpenClash，Fake-IP 模式 |
| 代理节点 | DMIT |
| Chrome 安全 DNS | 已启用，系统默认供应商 |
| 故障域名 | `dln1.ncdn.ec` |

## 三、故障排查

### 3.1 替代下载渠道测试

使用 Z-Library Mac 客户端进行下载测试时，客户端显示 **Suspected Phishing**（疑似钓鱼）安全警告。未选择 `Ignore & Proceed`，终止该路径测试。

![图 2：Mac 客户端显示疑似钓鱼警告（原始截图局部）](/images/zlibrary-download-error-1034/02-app-warning.jpg)

在 [LibTalk 论坛](https://lib-talk.io/viewtopic.php?t=9&start=110) 中发现其他用户报告了相同下载域名的 1034 错误，并提供了 Telegram 机器人作为替代渠道。

通过 Z-Library 图书详情页的「发送到 → Telegram」功能，将目标图书发送至已绑定的机器人，成功下载约 1.5 MB 的 EPUB 文件。

该测试验证了替代下载渠道可用，但未解决 Chrome 网页下载故障。

### 3.2 浏览器与设备对照测试

保持家庭网络环境不变，对不同设备及浏览器进行访问测试。为减少账号、图书及完整下载链接等变量，进一步直接访问 `https://dln1.ncdn.ec/`。

| 测试环境 | 测试内容 | 结果 |
|---|---|---|
| iPhone，同一 Wi-Fi | 网页下载 | 成功 |
| Mac，Safari | 网页下载 | 成功 |
| Mac，Safari | 直接访问下载域名 | 显示 `Welcome to nginx!` |
| Mac，Chrome | 直接访问下载域名 | Error 1034 |
| Mac，Chrome 无痕窗口 | 直接访问下载域名 | Error 1034 |

测试结果表明，同一台 Mac 在相同网络环境下，Safari 与 Chrome 的访问结果存在差异，且 Chrome 无痕模式无法消除故障。

因此，后续排查重点转向 Chrome 的 DNS 解析状态及网络连接状态。上述结果不能完全排除代理分流、DNS 处理或目标服务侧的影响，但不支持将故障简单归因于 DMIT 节点整体不可用。

## 四、故障处理

处理期间保持 OpenClash 配置及 DMIT 节点不变，按以下顺序操作。

### 4.1 临时关闭 Chrome 安全 DNS

1. 在 Chrome 地址栏输入 `chrome://settings/security`。
2. 记录「使用安全 DNS」的原始配置，临时关闭该功能。
3. 重新访问下载域名。

**验证结果：** 仍返回 Error 1034，故障未恢复。

### 4.2 清除 Chrome DNS 缓存

1. 在 Chrome 地址栏输入 `chrome://net-internals/#dns`。
2. 单击 **Clear host cache**，清除浏览器主机解析缓存。
3. 使用 **Lookup** 查询 `dln1.ncdn.ec`，检查解析结果。
4. 重新访问下载域名。

**验证结果：** 域名查询结果由此前的公网 IP 地址变为 OpenClash Fake-IP 地址 `198.18.2.251`，但页面仍返回 Error 1034。

### 4.3 清理 Chrome Socket 连接池

1. 在 Chrome 地址栏输入 `chrome://net-internals/#sockets`。
2. 单击 **Flush socket pools**，清理浏览器 Socket 连接池。
3. 重新访问 `https://dln1.ncdn.ec/`。

**验证结果：** 页面正常显示 `Welcome to nginx!`，原有 1034 错误不再出现。

### 4.4 恢复安全 DNS 配置

1. 返回 `chrome://settings/security`。
2. 将安全 DNS 恢复至原始配置，即开启并使用系统默认供应商。
3. 再次访问下载域名，确认页面正常。
4. 返回 Z-Library 图书详情页，重新执行 EPUB 下载。

**验证结果：** 下载域名访问正常，《蹒跚前行：1870—2010年全球经济史》的 EPUB 文件成功下载至本地。

## 五、原因分析

本次排查确认了以下现象：

- 临时关闭安全 DNS 后，故障未恢复。
- 清除 DNS 缓存后，域名查询结果发生变化，但故障仍然存在。
- 清理 Socket 连接池后，下载域名恢复正常访问。
- 恢复安全 DNS 原始配置后，实际下载功能仍然正常。

DNS 解析与网络连接建立是两个不同的过程。DNS 查询结果发生变化，并不意味着浏览器已有的网络连接会立即失效。若旧连接仍被复用，新的请求可能不会立即使用更新后的解析结果。

据此推断，本次故障可能与 Chrome 原有网络连接状态及 DNS 解析状态之间的不一致有关。清理 Socket 连接池是故障恢复前最后一个有效操作，但由于此前已调整安全 DNS 并清除 DNS 缓存，尚不能确认单独清理连接池是否足以解决问题。

本次未采集 Chrome NetLog、网络数据包或完整 DNS 请求记录，无法确认最初触发 Cloudflare 1034 的具体连接路径及根本原因。

## 六、后续处理建议

如果再次出现相同故障，且 Safari 可以正常访问下载域名，建议优先执行以下操作：

1. 打开 `chrome://net-internals/#sockets`，单击 **Flush socket pools**。
2. 重新执行下载，验证故障是否恢复。
3. 若故障仍然存在，按照第四节的完整顺序进行处理，并在每次操作后独立验证。
4. 如需进一步定位根本原因，应在清理缓存和连接池之前采集相关网络诊断记录。

注意：优先清理连接池属于基于本次排查结果提出的简化方案，尚未经过独立验证。清理连接池可能影响浏览器当前网络连接。

## 七、处理结论

Chrome 网页下载功能已恢复，并通过实际 EPUB 文件下载完成验证。DMIT 节点及 OpenClash 配置保持不变，Chrome 安全 DNS 已恢复原始配置。

Z-Library Mac 客户端的疑似钓鱼警告属于另一项未解决的安全问题，本次未对该警告进行根因分析，也未绕过安全限制。

<div id="tech-doc-zlibrary-1034" hidden></div>
<style>
/* 仅本篇技术文。1rem = html 的 --font-size-base，本站为 15px，不是浏览器默认 16px。 */
article:has(#tech-doc-zlibrary-1034) section.body {
  font-size: 1rem;
  line-height: 1.75;
}
article:has(#tech-doc-zlibrary-1034) section.body p,
article:has(#tech-doc-zlibrary-1034) section.body li {
  font-size: 1em;
  line-height: 1.75;
  margin: 0 0 0.8em 0;
}
article:has(#tech-doc-zlibrary-1034) section.body li {
  margin: 0 0 0.55em 0;
}
article:has(#tech-doc-zlibrary-1034) section.body li p {
  margin: 0 0 0.45em 0;
}
article:has(#tech-doc-zlibrary-1034) section.body li p:last-child {
  margin-bottom: 0;
}
article:has(#tech-doc-zlibrary-1034) section.body h2 {
  font-size: 1.2rem;
  line-height: 1.35;
  margin-top: 1.35em;
  margin-bottom: 0.5em;
}
article:has(#tech-doc-zlibrary-1034) section.body h2:first-of-type {
  margin-top: 1.1em;
}
article:has(#tech-doc-zlibrary-1034) section.body h3 {
  font-size: 1.08rem;
  line-height: 1.4;
  margin-top: 1.15em;
  margin-bottom: 0.45em;
}
article:has(#tech-doc-zlibrary-1034) section.body table {
  font-size: 1em;
  line-height: 1.5;
  margin: 0.4em 0 0.9em;
  width: 100%;
  border-collapse: collapse;
}
article:has(#tech-doc-zlibrary-1034) section.body th,
article:has(#tech-doc-zlibrary-1034) section.body td {
  font-size: 1em;
  line-height: 1.5;
  padding: 0.4em 0.55em;
  vertical-align: top;
}
article:has(#tech-doc-zlibrary-1034) section.body img {
  margin: 0.5em 0 0.7em;
}
article:has(#tech-doc-zlibrary-1034) section.body code {
  font-family: "Roboto Mono", "Courier New", monospace;
  font-size: 0.9em;
}
article:has(#tech-doc-zlibrary-1034) section.body pre {
  overflow-x: auto;
  max-width: 100%;
  line-height: 1.45;
}
article:has(#tech-doc-zlibrary-1034) section.body pre code {
  font-size: 100%;
}
</style>
