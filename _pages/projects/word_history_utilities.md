---
layout: page
title: Word History Utilities
permalink: /word_history_utilities/
---

## 缘起

由于想体验下目前很🔥️的Flutter，看看这个平台的能力，就决定用Flutter把当时的一个小想法实现出来。因此就开了业余小项目：word_history_utilities，然后，Chrome浏览器市场又多了一个名为Google字典单词历史小工具的扩展应用。

## 小目标

Google字典单词历史小工具这个扩展是为使用谷歌字典[官方扩展](https://chrome.google.com/webstore/detail/mgijmajocgfcbeboacabfgobmjgjcoja)的小伙伴准备的。谷歌字典扩展有个记录查词历史的功能，默认是未启用的。启用该功能后，使用扩展查过的单词会被记录下来。记录下来的单词历史可以被保存为文件，同时提供了可以让其他扩展提取查词历史的开关。推测谷歌字典扩展的产品经理和我一样认为查词历史有进一步可挖掘的价值，才设计了这个功能，记录查词历史并暴露接口供进一步利用查词历史。

![google dictionary extension options]({{ site.baseurl }}/images/google_dictionary_extension_options.png)

Word history in Google Dictionary Extension

Google字典单词历史小工具这个扩展就是基于的查词历史，进一步挖掘查词历史价值的应用。

## 功能介绍

谷歌字典扩展的查词历史未提供浏览方法，并且由于使用谷歌字典划词查询时非精确操作，数据是含有网址，非单词，非生词等杂质的生数据。因此Google字典单词历史小工具首个版本提供了：同步查词历史，浏览查词历史，整理查词历史这三个基础功能。

![Untitled%20f02f208d64f948238c9f79457bf32e73/Screen_Shot_2020-07-17_at_07.24.54.png](/images/Screen_Shot_2020-07-17_at_07.24.54.png)

Navigation drawer

![Untitled%20f02f208d64f948238c9f79457bf32e73/Screen_Shot_2020-07-17_at_07.24.29.png](/images/Screen_Shot_2020-07-17_at_07.24.29.png)

Word history

![Untitled%20f02f208d64f948238c9f79457bf32e73/Screen_Shot_2020-07-17_at_07.29.10.png](/images/Screen_Shot_2020-07-17_at_07.29.10.png)

Organize

![Untitled%20f02f208d64f948238c9f79457bf32e73/Screen_Shot_2020-07-17_at_07.25.24.png](/images/Screen_Shot_2020-07-17_at_07.25.24.png)

Options

### 📕同步查词历史

每次点开扩展，可以自动或手动同步Google字典扩展的查词历史。即使Google词典扩展的查词历史被清空，也能合并保留之前的单词历史。

### 🔍浏览查词历史

通过表格的形式，直观的浏览通过Google字典扩展查询过的单词。

### 📖整理查词历史

对于误操作记录下来的网址，非生词，非单词等干扰信息，可选中并删除。清理过的单词历史可下载，或通过后续迭代功能回顾。

## 结语

很早就发现谷歌查词历史这个功能及接口，想进一步发挥查词历史的价值，但在很长一段时间都未找到有相关的应用。说明利用查词历史这个需求是个很小众的需求。不管怎样，趁着尝试Flutter平台这个机会，做出了这个扩展。至少满足一下自己的需求。有类似需求的其他朋友能注意到这个扩展更好。后面，根据自己的使用情况与朋友们的反馈，会继续进行功能迭代，充分挖掘查词历史的价值。