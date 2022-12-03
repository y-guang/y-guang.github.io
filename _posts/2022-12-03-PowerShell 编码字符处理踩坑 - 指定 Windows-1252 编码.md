---
layout: post
title:  "PowerShell 编码/字符处理踩坑 - 指定 Windows-1252 编码"
date:   2022-12-03 12:47:00 +0800
categories: [computing]
tag: [powershell,chinese]
comments: true
---

## 要点摘要

PowerShell 默认 ASCII 编码方案为 Ascii (7-bit) character set. 如需 ASCII 扩展方案如 Windows-1252, 我们需要使用 

````powershell
[System.Text.Encoding]::GetEncoding("windows-1252").GetString($bytes)
````

指定具体编码.

## 背景及问题

需求是以 PowerShell 脚本解析二进制文件中以 Windows-1252 (ASCII 的一个特定扩展) 编码的 string. 使用默认 ASCII 解析, 发现无法正常解析扩展字符. 

例如, `0x92`, 即符号 `’` 不属于基本ASCII字符, 而属于 Windows-1252 扩展中

```powershell
> [System.Text.Encoding]::ASCII.GetString([byte]0x92)
?
> $a = [System.Text.Encoding]::ASCII.GetString([byte]0x92)
> [System.Text.Encoding]::ASCII.GetBytes($a)
63
```

可以发现, `ASCII.GetBytes` 方法将其解析为`?`, 即未知字符.

## 解决

最初想到的是查看 `[System.Text.Encoding]::ASCII` 的对应版本. 

```powershell
> [System.Text.Encoding]::ASCII

IsSingleByte      : True
BodyName          : us-ascii
EncodingName      : US-ASCII
HeaderName        : us-ascii
WebName           : us-ascii
WindowsCodePage   : 1252
IsBrowserDisplay  : False
IsBrowserSave     : False
IsMailNewsDisplay : True
IsMailNewsSave    : True
EncoderFallback   : System.Text.EncoderReplacementFallback
DecoderFallback   : System.Text.DecoderReplacementFallback
IsReadOnly        : True
CodePage          : 20127
```

然而 `WindowsCodePage: 1252` 指定了 `ASCII` 确实是 `Windows-1252`. 这在一开始给我造成了很大的困惑.

进一步查阅 [about_Character_Encoding](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_character_encoding?view=powershell-7.3) 的文档.

> the **Encoding** parameter supports the following values:
>
> - `Ascii` Uses Ascii (7-bit) character set.

默认的 ASCII 编码使用了 7-bit 的基本编码, 而不使用扩展编码.

### 指定特定编码

为了正确解码, 我们需要指定具体的 ASCII 扩展编码. 但进一步查阅文档发现. 在 `[System.Text.Encoding]` 中, 没有 `Windows-1252` 作为可选项.

经过查询, 发现 `GetEncoding(String)` 方法可以获得任意 code page 对应的 `encoding`. 

因此, 正确的解码方式为

```powershell
[System.Text.Encoding]::GetEncoding("windows-1252").GetString($bytes)
```

例如, 

```powershell
> [System.Text.Encoding]::GetEncoding("windows-1252").GetString([byte]0x92)
’
```

### 存储

另一方面, 存储时也应指定 encoding 为能包含对应字符的编码方案. 例如, `Export-Csv -Encoding UTF8`.
