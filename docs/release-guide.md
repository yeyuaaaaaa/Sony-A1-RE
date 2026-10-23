# GitHub 发布指南

## 发布前整理

在仓库根目录确认只保留文档和筛选后的证据：

```powershell
Get-ChildItem -Recurse -File docs, evidence
Get-ChildItem -Recurse -File -Include *.bin,*.img,*.DAT,*.dat,*.elf,*.so,*.key,*.pem
```

第二条命令如果列出设备固件、密钥或其他不应公开的文件，应先移出仓库范围。然后检查 Markdown 和 JSON：

```powershell
Get-Content -Raw evidence/round4_bootstrap_audit_evidence_20261024.json | ConvertFrom-Json | Out-Null
git diff --check
```

## 初始化并上传

如果目录还没有 Git 仓库：

```powershell
git init
git branch -M main
git add README.md LICENSE.recommendation.md .gitignore docs evidence
git diff --cached --stat
git commit -m "docs: publish A1 static analysis archive"
git remote add origin https://github.com/<ACCOUNT>/<REPOSITORY>.git
git push -u origin main
```

如果 GitHub 仓库已经存在，只需把 `git remote add origin` 换成实际远程地址；推送前检查 `git diff --cached --stat` 和 `git diff --cached --check`。

## About 建议

```text
Offline static analysis archive for Sony ILCE-1 (A1) firmware 4.00. Documents LUT, CP debug interfaces, updater boundaries, and hardware handoff points. No firmware patches or flashing tools.
```

## Release Note 建议

```text
## A1 4.00 static analysis archive

- Added the offline vulnerability theory memo for updater and Jiritsu paths.
- Added the bootstrap and hardware debug-interface audit.
- Added structured evidence for loader strings, CP UART/GPIO/SPI command tables, input hashes, and unresolved physical breakpoints.
- No firmware images, keys, patches, exploit scripts, or flashing packages are included.
```

## 发布后的维护

后续提交应继续保持“报告—证据—哈希—限制”四项对应关系。新增结论必须说明证据类型和反证条件；任何需要真机执行、写入或密钥的内容，都应停留在未决项中，不应直接提交到公开仓库。
