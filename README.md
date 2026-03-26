# TOOLS.md - Local Notes

Skills define _how_ tools work. This file is for _your_ specifics — the stuff that's unique to your setup.

## What Goes Here

Things like:

- Camera names and locations
- SSH hosts and aliases
- Preferred voices for TTS
- Speaker/room names
- Device nicknames
- Anything environment-specific

## Examples

```markdown
### Cameras

- living-room → Main area, 180° wide angle
- front-door → Entrance, motion-triggered

### SSH

- home-server → 192.168.1.100, user: admin

### TTS

- Preferred voice: "Nova" (warm, slightly British)
- Default speaker: Kitchen HomePod
```

## Why Separate?

Skills are shared. Your setup is yours. Keeping them apart means you can update skills without losing your notes, and share skills without leaking your infrastructure.

### 推送地址

- 企业微信 Webhook：`https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=e1808592-81bd-4310-9575-b39112a0e8c0`

### 青龙面板

- 地址：`http://81.70.1.192:13000/`
- 用户名：`ruaka`
- 密码：`124444`

### 自动化脚本部署流程

当用户提供 curl 命令和监控需求时：

1. **解析需求**：理解 curl 命令的 API 接口和用户的需求
2. **编写 Python 脚本**：
   - 使用 `requests` 库转换 curl 请求
   - 添加 `verify=False` 禁用 SSL 验证（避免证书问题）
   - 添加 `urllib3.disable_warnings()` 禁用警告
   - 设置 `sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')` 解决编码问题
   - 实现监控逻辑和推送（使用企业微信 Webhook）
3. **部署到青龙面板**：
   - 登录获取 token：`POST /api/user/login`
   - 上传脚本：`PUT /api/scripts`
   - 创建定时任务：`POST /api/crons`
   - 定时格式：`秒 分 时 日 月 周`（如 `0 * * * * ?` 表示每分钟）

**模板脚本位置**：`C:\Users\Ruaka\.qclaw\workspace\weidou_monitor.py`

---

Add whatever helps you do your job. This is your cheat sheet.
