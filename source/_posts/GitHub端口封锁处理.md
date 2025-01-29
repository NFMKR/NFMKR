---
title: GitHub端口封锁处理
date: 2025-01-29 11:42:50
categories:
  - 文章
---

## Git推送报错

#### P：fatal: unable to access 'https://github.com/example/example.git/': Failed to connect to github.com port 443 after 21044 ms: Could not connect to server
#### A: 
    - 检查网络是否正常：ping github.com
    - 更换github的端口：ssh -T -p 443 git@ssh.github.com    |   git remote set-url origin ssh://git@ssh.github.com:443/example/example.git
    - git push origin main