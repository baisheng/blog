---
title: WSL IO 性能优化
date: 2018-12-07 14:00:00
tags: [windows,wsl]
top_img: https://i.imgur.com/oaax0wV.png
---

WSL 的 IO 性能一直为人所诟病, 常常一个非常小的 repo, 使用 git clone 只有几KiB/s的速度, 严重影响使用体验

## 关闭 Windows Defender

> 高安全性往往意味着低性能. 

Windows Defender 的实时保护, 对WSL的性能有巨大的影响, 所以需要关闭它. **注意: 关闭实时保护, 会增加电脑遭到恶意软件入侵的风险**. 不过就我个人而言, 保持良好的上网习惯, 一般情况不会受到影响, 重要软件也即及时使用 Git 和 Drive 进行备份, 所以, 牺牲一些安全带来性能的提升是值得的.

![img](https://i.imgur.com/KNPi7F5.png)

关闭的路径:  `Windows Settings -> Update & Security -> Windows Security -> Virus & thread protection -> Virus & thread protection(Manage settings) -> Real-time protection`

同时, 在这个页面滚到底部, 找到`Exclusions(Add or remove exclusions)`, 进入将 WSL 的安装目录填进去

![img](https://i.imgur.com/Pj1K7Ac.png)

参考上图的路径, 找你电脑上的路径

### 使用组策略来关闭

搜索`group policy`

![img](https://i.imgur.com/dTtYczb.png)

进入之后, 根据路径 `Computer Configuration -> AdaministrativeTemplates -> Windows Components -> Windows Defender Antivirus -> Real-time Protection`, 找到 `Turn off real-time protection`, 配置为`Enabled`即可

![img](https://i.imgur.com/EnIssqM.png)