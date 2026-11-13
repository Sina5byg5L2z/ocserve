# opencode 安卓装机指令（复制本文件全部内容，发给 opencode 执行）

> 使用方法：在手机 Termux 装好 opencode 后（自带免费模型，无需登录），把本文件**全文复制**粘贴进 opencode 对话框发送，它就会自动完成剩余配置。
> 项目主页：（发布后填你的 GitHub 仓库地址）
> 适用：Termux (aarch64) + opencode，版本不限；重复发送安全（幂等），不会破坏已有配置。

---

## 给 AI 助手（opencode）的装机任务书

你正在一台 **Android + Termux** 环境里为用户完成 opencode 后台服务的配置。严格按以下步骤执行。

### 铁律（必须遵守）
1. **只做**本任务书列出的事；不安装任何软件包、不使用 sudo、不删除/覆盖任务书之外的任何文件、不修改 opencode 自身配置
2. **幂等**：每步先检查是否已完成，已完成就跳过并说明
3. **失败即停**：任何一步失败，立即停止后续步骤，向用户报告失败步骤、错误原文、建议的解决方法
4. 每完成一步，用一行简短中文汇报该步结果
5. 全部完成后输出「验收报告」（格式见文末）

### 步骤

#### 步骤 1：环境自检
- 依次检查并记录：`uname -m`（应为 aarch64）、`opencode --version`（应能输出版本号）
- 若 `opencode` 命令不存在：停止，告诉用户"opencode 尚未安装，请先按教程完成安装"，不要尝试代装
- 检查手机存储链接：若 `~/storage/downloads` 不存在，执行 `mkdir -p ~/storage && ln -sfn /storage/emulated/0/Download ~/storage/downloads`；若访问 `/storage/emulated/0` 报 Permission denied，提示用户"去系统设置 → 应用 → Termux → 权限 → 文件和媒体 → 允许"，然后**本步标记为待定但继续后续步骤**（不阻塞）
- 创建推荐工作目录（幂等）：`mkdir -p ~/Desktop/workplace`，向用户说明这是装机者的个人习惯；跳过的话用户直接在 Home 路径启动 opencode（家目录即工地），不影响使用

#### 步骤 2：创建一键启动脚本
先检查：若 `$PREFIX/bin/ocserve` 已存在则跳过本步，汇报"启动脚本已存在"。
否则用 heredoc **原样**写入以下内容（注意保留引号防变量展开）：

```bash
cat > $PREFIX/bin/ocserve << 'OCEOF'
#!/data/data/com.termux/files/usr/bin/bash
# 一键启动 opencode 后台服务
if curl -s -m 2 -o /dev/null http://127.0.0.1:4096; then
  echo "已在运行"; exit 0
fi
termux-wake-lock
tmux new-session -d -s ocserver \
  "opencode serve --hostname 127.0.0.1 --port 4096"
for i in $(seq 1 15); do
  sleep 1
  curl -s -m 2 -o /dev/null http://127.0.0.1:4096 && \
    { echo "已启动: http://127.0.0.1:4096"; exit 0; }
done
echo "启动超时，重试 ocserve 即可"
OCEOF
```

#### 步骤 3：创建一键停止脚本
先检查：若 `$PREFIX/bin/ocstop` 已存在则跳过本步。
否则原样写入：

```bash
cat > $PREFIX/bin/ocstop << 'OCEOF'
#!/data/data/com.termux/files/usr/bin/bash
# 一键停止 opencode 后台服务
pkill -f "opencode serve" 2>/dev/null
sleep 1
pkill -9 -f "opencode serve" 2>/dev/null
termux-wake-unlock 2>/dev/null
curl -s -m 2 -o /dev/null http://127.0.0.1:4096 && echo "仍在运行" || echo "已停止"
OCEOF
```

#### 步骤 4：赋予执行权限
```bash
chmod +x $PREFIX/bin/ocserve $PREFIX/bin/ocstop
```

#### 步骤 5：启动并验证
- 运行 `ocserve`
- 用 `curl -s -m 5 -o /dev/null -w "%{http_code}" http://127.0.0.1:4096` 确认返回 `200`
- 注意：首次启动可能需要 10 秒以上，以 curl 结果为准；若脚本报告"启动超时"，等 10 秒再 curl 一次仍失败才算失败

#### 步骤 6：创建开机自启（可选，失败不阻塞）
- 若目录 `~/.termux/boot` 不存在则创建
- 写入 `~/.termux/boot/ocserve.sh`，内容为：

```bash
#!/data/data/com.termux/files/usr/bin/sh
sleep 10
/data/data/com.termux/files/usr/bin/ocserve
```

- `chmod +x ~/.termux/boot/ocserve.sh`
- 汇报时注明：需安装 Termux:Boot 插件并打开一次才生效；若本步失败，提示用户可跳过

#### 步骤 7：在验收报告末尾附一条建议（提示性质，禁止追问）
写入任何规则之前，只在验收报告末尾附上以下建议文字（原话即可）。**注意：这只是提示，不是提问**——不要使用提问/确认交互，不要停下来等待用户答复，报告输出完任务即结束；用户不回应就到此为止，不要二次提醒。

```
【一条建议】默认情况下，opencode 生成的成品文件都在 Termux 内部，手机文件管理器看不到。
如果想以后让成品自动存到 手机Download/opencode（图形界面直接看），随时对我说：
- 「要」→ 我写入规则
- 「目录改为 某路径」→ 按你指定的路径写入
- 「不用」→ 保持现状
```

#### 步骤 8：仅当用户主动响应建议时才执行
- 用户在**之后的对话**中说「要」：将下方规则块写入 `~/.config/opencode/AGENTS.md`（文件已存在则检查未包含时追加，严禁覆盖已有内容）
- 用户说「目录改为 X」：同上写入，但把规则里的路径换成用户指定的路径
- 用户说「不用」：回复"好的，保持现状"，不写入任何内容
- 没有明确响应建议 → 不执行、不追问、不再次提醒

执行写入完毕后，输出以下简短报告（一屏内，不重复装机报告）：

```
【规则已生效】
- 成品目录：~/storage/downloads/opencode（手机 文件管理器 → Download/opencode）
- 写入位置：~/.config/opencode/AGENTS.md（追加规则块，未改动原有内容）
- 生效范围：以后所有 opencode 会话
- 目录验证：可写 ✓
以后想换目录说「目录改为 X」，想撤销说「删掉成品目录规则」，随时可改。
```

（若写入失败或目录不可写，如实报告失败原因和修复建议，不谎报成功）

写入的规则块内容：

```markdown
# 全局规则（所有会话自动加载）

## 交流
- 用中文交流

## 成品文件落盘（图形界面可见）
- 生成的成品文件（文档、脚本、图片、音视频、APP 包等）统一放到 `~/storage/downloads/opencode/`——它就是手机「文件管理器 → Download/opencode」，用户可直接查看、分享
- 读取用户手机里的文件用 `~/storage/` 链接（downloads/dcim/pictures/documents 等）
- 临时文件放 `$TMPDIR`（Termux 下没有 /tmp），任务结束记得清理

## 命令踩坑记录
- 运行有风险的命令（删目录、后台进程、进程探测、装包、存储授权）之前，先读 `~/.config/opencode/命令踩坑记录.md` 避免重蹈覆辙
- 踩了新坑要回写该文档（文件不存在就创建）
```

### 验收报告（最终输出格式）

```
【装机验收报告】
1. 环境自检：aarch64 ✓ / opencode x.y.z ✓ / 手机存储链接 ✓(或待定+原因)
2. ocserve：新建 ✓ / 已存在跳过
3. ocstop：新建 ✓ / 已存在跳过
4. 执行权限：✓
5. 服务启动：HTTP 200 ✓（http://127.0.0.1:4096）
6. 开机自启：已建 ✓ / 未建（原因）

【你还需要做的】
1. 手机浏览器打开 https://github.com/crim50n/oc-remote/releases/latest 下载 APK 安装
2. 打开 OC Remote → 点 + → 地址填 http://127.0.0.1:4096 → Connect
3. 新建会话时，项目建议选 workplace（~/Desktop/workplace）——装机者个人习惯；跳过它 = 选 home / 直接在 Home 路径启动，同样能用
4. 日常启停：Termux 里敲 ocserve（开）/ ocstop（停）

【一条建议】
（步骤 7 的建议文字放这里，输出后任务即结束，不等待回复）
```

---

## 给人类用户（看不懂上面内容也没关系）

- 你**不需要**看懂这份任务书，只需：装好 Termux → 装好 opencode（发句"你好"确认能用）→ 把本文件全文复制发给 opencode → 等它汇报"验收报告"
- 报告全 ✓ 后，按报告末尾"你还需要做的"两步操作即可
- 任何一步失败：把 opencode 的报错原样截图求助，或把报错原样发给 opencode 让它自查
