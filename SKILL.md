---
name: wechat-extraction
version: 1.0.0
description: |
  从微信 PC 端数据库提取、解密和分析聊天记录。支持文本、语音、图片、
  链接、文件、合并转发等所有消息类型的完整提取与结构化分析。
  当用户说"分析微信""拉取微信消息""导出微信聊天记录""解密微信"
  "看看微信群里说了什么"或涉及微信消息整理时，使用此 skill。
  微信群产量日报校验（防漏报）、对话图片归档也走此 skill。
triggers:
  - 分析微信
  - 拉取微信消息
  - 导出微信聊天记录
  - 解密微信
  - 看看微信群里说了什么
  - 微信群里说了什么
  - 微信消息整理
  - 产量校验
  - 产量日报
  - 图片归档
  - 归档图片
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

# WeChat Message Extraction Skill

## 一、核心原理

微信 PC 端数据库是加密的 SQLite（`.db` 文件），存储在 `%USERPROFILE%\Documents\WeChat Files\wxid_xxx\Msg\`（旧版）或 `%USERPROFILE%\Documents\xwechat_files\wxid_xxx\`（微信 4.x）下。部分用户自定义数据目录可能位于其他盘符（如 `D:\RJ\Weixinjilu\xwechat_files\wxid_xxx_xxxx\db_storage`）。要读取消息，必须：

1. **从微信进程内存提取密钥**（需管理员权限，微信 4.0.x 为 raw key；微信 4.1.x 为 passphrase/派生 key）
2. **用密钥逐个解密数据库文件**
3. **连接解密后的数据库查询消息**

**关键技术栈**：MemoTrace (wxManager) + Python `pysilk`（语音解码）；微信 4.1.x 推荐使用 [TANGandXUE/wcdb-key-tool](https://github.com/TANGandXUE/wcdb-key-tool)。

---

## 一之一、微信 4.1.x 解密方案（2026-08-27 固化，2026-09-11 实战修正）

> ⚠️ **2026-09-11 实战固化（重要）**：`wxcli.py decrypt` 内置的 v4 密钥提取走 wxManager raw key 内存扫描，**对微信 4.1.x 直接失效**（报错"无法从微信进程提取 v4 密钥"）。**4.1.x 环境跳过 wxcli decrypt，直接用下方 wcdb-key-tool 两步法**。wxcli 的 export/analyze 仍可正常使用（消费解密缓存）。

### 1.1 版本差异

| 微信版本 | 密钥模式 | 内存中可提取 | 推荐工具 |
|---|---|---|---|
| 微信 4.0.x | raw key（64 hex key + 32 hex salt） | 完整 raw key | wxManager / MemoTrace |
| 微信 4.1.x（实测 4.1.13） | SQLCipher 4 passphrase | passphrase 或已派生的 enc_key | **wcdb-key-tool** |

微信 4.1.x 已切换到 SQLCipher 4 的 passphrase 模式：内存中只有 32 字节 passphrase，需要通过 `PBKDF2-HMAC-SHA512(passphrase, salt, 256000, 64)` 派生 `enc_key` / `hmac_key`。老版直接扫描 raw key 的工具（如 wechat-decrypt-clean 自带的 `find_all_keys.py`、wx_key）会失效。

### 1.2 推荐工具：wcdb-key-tool（4.1.x 标准流程，2026-09-11 实战固化）

[TANGandXUE/wcdb-key-tool](https://github.com/TANGandXUE/wcdb-key-tool) 的 Windows 版通过只读扫描进程内存中的 `com.Tencent.WCDB.Config.Cipher` 对象，可直接拿到已经派生好的 `enc_key`，无需自己处理 passphrase/PBKDF2。

**本机已 clone 位置**：`<你的项目路径>`（历史 keys 文件同目录）

**标准两步法（分步执行，勿用 `extract --decrypt` 一步走）**：

```bash
# 第 1 步：提取密钥（--output 是 keys 文件路径，不是目录！传目录报 PermissionError）
python wcdb_key_tool_windows.py extract \
  --db-dir "D:\RJ\Weixinjilu\xwechat_files\<wxid>\db_storage" \
  --output "<你的项目路径>"

# 第 2 步：解密数据库（--keys 密钥文件，--output 才是解密输出目录）
python wcdb_key_tool_windows.py decrypt \
  --db-dir "D:\RJ\Weixinjilu\xwechat_files\<wxid>\db_storage" \
  --keys "<你的项目路径>" \
  --output "<你的项目路径>"
```

解密输出目录直接对应 wxcli 的缓存约定（`<你的项目路径>`），第 2 步完成后即可用 `wxcli.py export` / `analyze` 消费。

**多账号机器**：`xwechat_files` 下可能有多个 `wxid_*` 账号（如本机 `wxid_example_ca93` + `wxid_example`），逐个 extract + decrypt，别漏。

**实测结果（微信 4.1.13，Windows 11）**：
- 成功提取 **21/22** 个数据库密钥；
- 唯一缺失：`migrate/unspportmsg.db`，不影响常规分析与监控；
- `session/session.db` 提取成功，是实时监控的核心。

### 1.3 备选方案：passphrase 派生

若 wcdb-key-tool 失效，可回归 passphrase 路线：
1. 用 DLL 注入/Hook 工具提取 64 位 hex passphrase；
2. 在 wechat-decrypt-clean 的 `config.json` 中加入 `"passphrase": "..."`；
3. 运行 `python main.py derive-keys` 派生 `all_keys.json`；
4. 运行 `python main.py web` 启动监控（已集成 passphrase 模式自动回退）。

### 1.4 路径说明

- 默认微信数据路径：`%USERPROFILE%\Documents\xwechat_files\wxid_xxx_xxxx\db_storage`
- 本机实测自定义路径：`D:\RJ\Weixinjilu\xwechat_files\wxid_example\db_storage`
- 解密后数据库默认缓存：`<你的本机路径>`

### 1.5 实时监控

**本机一键方案（2026-10-12 固化，多开版）——优先用这个，别再手动翻目录**：

```bash
python <你的项目路径> status     # 查看各账号监控状态
python <你的项目路径> install    # 注册开机自启（启动文件夹VBS无窗口方案）
python <你的项目路径> daemon     # 手动起守护（自启后不需要）
python <你的项目路径> watch      # 监听目标会话新消息（命中即退出）
```

- watchdog 机制：每 30 秒扫描数据根目录发现账号 → wcdb-key-tool extract 提密钥（**extract 成功=微信已登录该账号**）→ 每账号独立端口起 `monitor_web.py`（主号固定 5678，其余按 `port_map.json` 持久化分配）→ 微信退出自动清理。
- 支持多开：登录 2-3 个微信时自动逐个接入，各账号一个端口（`status` 子命令列出）。
- monitor 本体已迁出 08 临时调用到 `<你的项目路径>`（旧路径仅存档）；端口改造为 `WCM_PORT` 环境变量，多实例经 `WECHAT_DECRYPT_APP_DIR` 指各自 runtime 目录（config/decrypted/keys 全隔离）。
- 开机自启用启动文件夹 + VBS（`schtasks` 被组策略拒绝，Access denied，2026-10-12 实测）。

**底层原理（手动版，仅排障时用）**：

```bash
# 将 wcdb-key-tool 生成的 keys 放到 monitor 目录，启动单实例
cd <你的项目路径>
python monitor_web.py   # http://localhost:5678
```

**关键点**：
- `monitor_web.py`/`monitor.py` 只使用 `enc_key` 做 AES-CBC 解密，不校验 HMAC，因此 wcdb-key-tool 生成的 keys 可直接兼容；
- 微信 4.1.x 的 `Msg_*` 表通过 `real_sender_id` 区分消息方向，需在同库 `Name2Id` 表中动态查找当前用户 rowid；
- 当前用户 wxid 从 `db_dir` 路径自动推断，匹配 `Name2Id` 时需去掉 `_\d+` 后缀；
- **monitor 的捕获起点=启动时刻**：启动前的消息不在 messages_log（历史用解密缓存查，见陷阱 1/1.5）；
- 群历史未同步时 monitor 同样看不到 → 先按陷阱 1.5 打开窗口/重登微信，再监控。

### 1.5.1 直读解密库（2026-10-23 起首选，替代 `/api/history` 取数）

**为什么必须换**：运行时接口 `/api/history` 只吐最近活跃的 ~26 个会话，**新建的群完全看不见**。实测两账号 24 小时内有 53 个会话、1700+ 条消息，接口漏掉一半以上；用户截图里的「我晚上出个计划书哈」就在一个新建群里，接口取不到 → 待办漏抓。

**工具**：`<你的项目路径> AI Native/04-中台程序/wxreader.py`

```python
import wxreader as w
w.read_account("wxid_example", since_ts, limit_per_chat=200)  # 单账号全会话
w.search("杭师大", days=7)                                                  # 跨账号关键词搜
```
```bash
python <你的项目路径> AI Native/04-中台程序/wxreader.py 计划书      # CLI 关键词搜
python <你的项目路径> AI Native/04-中台程序/wxreader.py            # 24h 总量概览
```

返回字段：`acc / chat / chat_name / sender / is_self / type / ts / text`（text 已清洗）。

**读库四个必踩点**（`_clean()` 已全部处理，自己写代码时照抄）：
1. `Msg_<md5(会话名)>` 表名 → 会话名要反查 `Name2Id`，**每个分库的 Name2Id 独立**，不能用别库的映射；
2. `WCDB_CT_message_content=4` 表示 zstd 压缩，需 `zstandard.ZstdDecompressor().decompress()`；
3. 系统消息 `<sysmsg>` 不是对话，**且它常裹在 `"123@chatroom:"` 引用头后面**——必须剥完引用头再判类型，顺序反了就会把入群提示当正文；
4. 引用消息前缀可能是 `wxid_xxx:` / `昵称: 数字别名:` 多层嵌套，需循环剥（剥一次不够）。

**与运行时分工**：`monitor_web.py` 负责持续解密（保证库是新的），`wxreader` 负责取数。库文件停更 = 运行时挂了，不是读取问题；先 `wechat_monitor.py status` 再查读库。

**下游已接入**：
- AI Native 中台 `monitor.py`（10 分钟巡检，替代原接口版 `scan_wechat_live`）
- 参谋长 `15-实时微信与录音监控` 的日报自动发现（`active_chats()`）+ `report_period.py` 周报/月报

### 1.6 v4 消息导出的三个实战坑（2026-09-11 固化）

**① 分库 Name2Id 独立（最严重，会导致发送者大面积错位）**

微信 4.1.x 按会话分表（`Msg_<md5(对方wxid)>`），同一个会话的表可能横跨多个 `message_N.db`，而**每个分库的 `Name2Id` 映射互相独立**（rowid→wxid 各不相同）。`real_sender_id` 必须用**消息所在 db 的 Name2Id** 映射：

```python
def sender_name(dbf, rid):
    wx = db_name2id(dbf).get(rid)   # dbf = 该消息所在的 message_N.db
    if wx == me_wxid: return '我'
    return names.get(wx) or wx
```

用 message_0 的映射套所有分库，会出现"本人消息显示成陌生人"的大面积错标（2026-09-11 松慕导出 135 条错标，逐库映射后修复）。另注意部分分库 Name2Id 无 `user_name` 列（旧结构），取列名前先 `PRAGMA table_info`。

**② media_0.db 单独重解密必须保持目录结构**

wcdb-key-tool 批量解密可能让 `media_0.db` 输出 0 字节。单独重解时 `--db-dir` 必须保持原始相对结构（如 `tmp/message/media_0.db`），因为 keys 文件键名带路径前缀（`message\media_0.db`）；平铺目录会报"无密钥"SKIP。

**③ 批量回填语音转写时同分钟消息的顺序分配**

多条语音同一分钟内，按"MM-DD HH:MM"字符串匹配会反复命中第一条。正确做法：转写结果按时间戳排序，逐行匹配时维护 `used` 集合顺序分配。

**④ v4.1 消息内容 zstd 压缩（2026-09-24 尚都实战固化）**

微信 4.1.x 部分消息（图片/文件的 XML、大段文本）的 `message_content` 是 **zstandard 压缩**（魔数 `28 b5 2f fd`），直接 decode 得乱码。处理方式：`pip install zstandard`，读 bytes 后 `ZstdDecompressor().decompress(raw)`（注意加 `max_output_size` 上限防炸弹），再按 XML 解析。文本小消息多为明文，兼容两种即可。

**⑤ v4.1 图片 .dat 解密（2026-09-24 已攻克）**

`msg/attach/<md5(会话wxid)>/Img/*.dat` 文件头仍是 V2 标记（`0708 5632 0807`），但 4.1 把 MemoTrace 硬编码的 V2 固定密钥换成了**账号级派生密钥**：
- `code`：从 `%APPDATA%/Tencent/xwechat/net/kvcomm/key_<code>_*.statistic` 文件名取（**勿只取 mtime 最新者**，见下方 2026-10-20 修正；`xor_key = code & 0xFF` 与旧 `get_decode_code_v4` 输出一致可交叉验证）
- `aes_key = md5(f"{code}{wxid去_数字后缀}")[:16]`（16 个 ASCII 字符）
- 解密结构同 MemoTrace `decode_dat_v4`：0xf 头 + encrypt_length 指示的 AES-ECB-128 区（PKCS7 去垫）+ 尾段单字节 XOR（最后 1MB）
- **已验证**：输出与微信"另存为"原图字节级一致（md5 相同）
- 实现：`wechat_extraction/image_resolver.py` 的 `decode_dat_v41()`，`decode_dat_file(wxid=...)` 自动优先走此路径
- 已知限制：**wxgf 封装**（微信私有格式，头 `wxgf`）——**2026-10-07 已攻克**：wxgf = HEVC 视频编码存静态图（腾讯 WXAM 格式），非加密。结构：`wxgf` magic + headerLen(1B=0x13) + 变长头(ICC profile/参数) + **[4B BE length][Annex B HEVC 裸流]** + 20B 尾部。解码：V2 解密 → 从 0x255 读 4B BE 长度 → 提取 HEVC 裸流 → `ffmpeg -i x.hevc -vframes 1 -c:v mjpeg -q:v 3 out.jpg`。**注意 NAL 类型判读：HEVC 首字节 >>1&0x3F（VPS=32/SPS=33/PPS=34/IDR=19），勿用 H.264 的 0x67/0x68 找 SPS/PPS**。脚本 `<你的项目路径>`。实况照 `_t.dat` 缩略图走标准 V2 解密即得 JPEG（96×210），可作兜底。
- **2026-10-18 实战修正（念念作业批量解码 16/16）**：**勿硬编码 0x255/0x259 偏移**——wxgf 头变长，16 个样本中真实 HEVC 起始码出现在 0x2a/0x255/0x259 不等，且头部假起始码会抢匹配（如 SPS 型假码在 0x24f、真 VPS 在 0x259，按"首个匹配"会切错流致 ffmpeg 报 "received no packets"）。**稳健算法**：V2 解密后在 8KB 内扫全部 `00 00 00 01`，取 NAL 类型 ∈ {32,33,34,19,20} 且**长度前缀合理**（>0 且 off+len≤文件尾-20B尾）的候选，**优先 VPS(32)**，首个合理候选次之；ffmpeg  stderr 必须二进制捕获（GBK 解码会炸 reader 线程）。
- 图片定位三要素：会话目录 hash=md5(wxid)、_h/_t 后缀=高清/缩略、文件 mtime≈消息时间
- **2026-10-20 实战修正（科洋现场照 7 张全解）**：`image_resolver.get_account_v2_keys()` 按「mtime 最新的 key_ 文件」取 code，本次取到 `code=0`，解出头字节 `e7 14 e1 93…` 非任何图片魔数（落成 `.bin`）。**正确做法：枚举 kvcomm 下全部 `key_<code>_` 的 code，逐个派生 `md5(f"{code}{wxid去_数字后缀}")[:16]` 试解，用解出头字节是否为 `ff d8 ff e0`(JPEG)/`89 50 4e 47`(PNG)/`wxgf` 校验**。本次正确答案 `code=1161846061`（另一个候选 0 失败）。同一批 .dat 的 `_t` 缩略图解出 210×157 JPEG，原图解出 `wxgf` 后按上方 HEVC 算法出 1280×960 —— **缩略图太小无法辨认内容，PPT/文档照片必须解原图**。
- **微信 4.1 解密缓存必须与实时数据同源**：`<你的项目路径>` 缓存常是几天前的；查“昨天/今天”的消息前，用监控 runtime 里的最新 keys（`wechat-decrypt-clean/wechat-monitor-runtime/keys/keys_<wxid>.json`）重跑 `wcdb_key_tool_windows.py decrypt`，否则会话表存在但 `max(create_time)` 停在旧日期（本次 10-12，查 10-19 得 0 条）。

---

## 二、致命陷阱

### 陷阱0：重点内容判定 — 近7天用户主动发言聊天全量语义分析（2026-06-20 固化）

#### 问题现象
微信群聊数量多、消息量大，分析时容易淹没真正需要用户关注的内容。很多高优先级事务其实藏在用户自己曾经主动发言过的聊天中。仅靠关键词匹配会系统性遗漏表达不够"标准"的待办（如"男裤费用您直接转我就行了"）。

#### 判定标准
**近7天内，私聊或群聊中只要用户有主动发言，该聊天对象即列为重点内容，必须全部逐条语义分析，不依赖关键词预过滤。**

**主动发言定义**：
- 消息发送者为当前登录微信用户本人（非系统消息、非他人转发、非邀请入群等被动消息）
- 包含文本、语音、图片、文件、链接、引用回复等用户主动发出的任何消息类型
- 不包含：被@后回复、系统提示、群邀请、撤回提示

**为什么这是重点**：
- 用户主动说话的聊天，往往意味着用户已经在参与、决策、承诺、催促或处理
- 这些聊天中未完成的事项最容易被遗漏
- 比单纯按消息数量排序更可靠
- 比关键词匹配更不容易漏掉非标准表达

#### 必须执行的操作
1. 导出消息时，**标记每条消息是否为用户发送**（通过 `IsSender` 字段或发送者wxid与当前用户wxid比对）
2. 分析前，**先筛选出近7天用户主动发言过的全部聊天对象列表**
3. **对这些聊天不做关键词过滤，逐条语义判断**：
   - 是否涉及用户承诺/安排但未闭环？
   - 是否待对方回复？
   - 是否涉及商务/财务/收款/合同/合作/面试/招聘？
   - 是否涉及家庭/个人事务？
   - 是否需要用户盯一下或确认后续状态？
4. **输出报告时必须单独列出"近7天用户主动发言的重点聊天"清单**，并逐项给出需盯事项
5. 近8~30天内用户主动发言的聊天可作为补充扫描范围，用关键词+语义双轨制二次检查

> 核心原则：用户主动发过声的聊天，优先级高于用户只围观没说话的群；近7天全量语义分析优于关键词过滤。

---

### 陷阱1：旧缓存不会自动更新

#### 问题现象
"最近两三天的记录找不到""今天的记录没有"

#### 根本原因
`wechat_analyzer.py` 的 `find_decrypted_db()` 会自动复用已有的 `wxid_*/Msg` 目录。**第一次解密生成的缓存只包含到那天的数据，不会随微信新消息自动更新。**

> 案例：5月14日生成的 `wxid_example` 缓存，到5月18日查询时仍只包含到5月14日的数据，5月15日~18日的新消息全部缺失。

#### 必须执行的操作
每次需要最新数据时，**必须先删除旧缓存，重新解密**：

```bash
# 清理旧缓存（在 .claude/tools/ 目录下执行）
rm -rf wxid_*
# 然后重新运行解密脚本
```

---

### 陷阱1.5：PC 端数据库未同步最新消息（2026-08-01 固化）

#### 问题现象
清缓存、重新解密后，仍然查不到最近 1~2 天的消息；或者截图/用户明确知道存在的消息，在数据库中完全找不到。

> 案例：2026-08-01 查询"桐城嘉誉16栋业主交流群"撤回消息时，解密后数据库最新只到 7月30日 19:40，8月1日的消息全部缺失。

#### 根本原因
**微信 PC 端不会无条件把手机上所有最新消息同步到本地数据库。** 尤其是对于长时间未主动打开的群聊/私聊，PC 端可能只保留了较早的数据，新消息虽然能在手机端看到，但尚未写入 PC 端本地数据库。

这与"旧缓存陷阱"不同：
- 旧缓存陷阱：解密缓存是旧的，但 PC 端原始数据库已有新数据
- 本陷阱：PC 端原始数据库本身就没有新数据，重新解密也查不到

#### 必须执行的操作
1. 如果清缓存重新解密后，数据库 `MAX(CreateTime)` 明显晚于用户目标时间范围（比如差了半天以上），或目标聊天在数据库中完全不存在，**优先怀疑 PC 端未同步**
2. **提示用户退出微信 PC 端并重新登录**，重新登录会触发微信从服务器重新拉取同步
3. 登录后，**让用户点开目标聊天窗口**（群聊或私聊），滚动加载到目标时间段附近，确保消息写入本地数据库
4. 等待消息稳定加载（一般几十秒到几分钟）后，再次清缓存 → 重新解密 → 查询
5. 如果重新登录后仍然查不到，再排查：是否当前登录的微信号不对、截图是否来自另一台设备等

> 核心原则：解密只能读出 PC 端本地已有的数据；本地没有的数据，重新解密一百遍也没有。重新登录是让 PC 端补全数据的最有效手段。

### 陷阱2：按消息数量过滤联系人导致高价值私聊遗漏（2026-05-29 重大教训）

#### 问题现象
微信分析报告中遗漏了重要的商务私聊消息。案例中"君意 niki朋友 老胡"主动索要"大牌鞋服跟奢侈品的case"，但晨报完全未提及。

#### 根本原因
**分析脚本按消息数量排序，只截取了 Top N 聊天对象进行详细分析。** "君意_niki朋友_老胡"这个私聊总共只有 8 条消息，远低于主播通告群（508条）等群聊，被排序后直接过滤掉了。

这是系统性的筛选偏差：低消息量但高商务价值的私聊会被消息量过滤规则系统性遗漏。

#### 必须执行的操作
1. **遍历所有聊天对象**，不做消息数量过滤
2. **分析阶段用关键词/事务关联度排序**，而不是原始消息量排序
3. **高价值私聊识别（双轨制）**：
   - **关键词辅助扫描**：case/报价/合作/需要/安排/预算/方案/合同/签约等商务词汇出现 → 提高优先级
   - **AI自主判断**：遍历所有私聊时，主动判断内容是否涉及事务推进、商机线索、风险提醒、人员变动等。关键词只是辅助，**不能替代判断**。例如：对方主动索要过往案例、询问服务能力、提出见面/电话沟通，即使不含关键词也需强制纳入

> 核心原则：消息量 ≠ 价值。3 条商务私聊的价值可能远高于 500 条群聊闲聊。关键词是线索，不是门槛。

### 陷阱3：MSG 分片遗漏（2026-05-29 重大教训）

#### 问题现象
解密成功后查询数据库，发现最新消息只到几个月甚至几年前，"肯定是有最新的消息的，但查不到"。

#### 根本原因
**微信消息按时间分片存储在多个 MSG*.db 数据库中**：
- MSG0.db：最早的历史消息（如 2023年3月 ~ 2023年9月）
- MSG1.db：下一时间段（如 2023年9月 ~ 2024年3月）
- ...
- MSG6.db：最新的消息（如 2026年5月）

旧脚本 `extract_my_wechat.py` 只硬编码了 MSG0~MSG6 路径，但查询时可能因时间范围过滤只命中了早期的 MSG0，**导致 MSG6 中的上千条最新消息被完全遗漏**。

> 案例：2026-05-29 晨报初版因只检查 MSG0.db 的时间范围（2023年），漏掉了 MSG6.db 中 1681 条 5/28-5/29 的最新消息，导致多个 P0 事务（陈哥被骗款项、房东续租无回复等）未被识别。

#### 必须执行的操作
**任何微信数据查询必须遍历所有 MSG 分片**，不能只查询 MSG0 或假设数据在某个特定分片中：

```python
db_paths = [
    r'...\wxid_xxx\Msg\Multi\MSG0.db',
    r'...\wxid_xxx\Msg\Multi\MSG1.db',
    r'...\wxid_xxx\Msg\Multi\MSG2.db',
    r'...\wxid_xxx\Msg\Multi\MSG3.db',
    r'...\wxid_xxx\Msg\Multi\MSG4.db',
    r'...\wxid_xxx\Msg\Multi\MSG5.db',
    r'...\wxid_xxx\Msg\Multi\MSG6.db',
]

all_messages = []
for db_path in db_paths:
    if not os.path.exists(db_path):
        continue
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    cursor.execute('SELECT ... FROM MSG WHERE CreateTime >= ? AND CreateTime <= ?', (start, end))
    all_messages.extend(cursor.fetchall())
    conn.close()
```

**数据新鲜度验证**：查询完成后，必须检查 `all_messages` 中 `MAX(CreateTime)` 是否覆盖目标查询范围。如未覆盖，说明还有更新的分片未被发现，需重新检查解密完整性。

## 二点五、新统一 CLI：wxcli.py（2026-08-19 新增）

为方便使用，已将核心能力封装为统一 CLI 入口 `<你的项目路径>`。

**推荐新流程**：

```bash
# 1. 解密（自动探测微信目录、自动清缓存、自动校验）
#    注意：仅对微信 4.0.x 有效；4.1.x 走 wcdb-key-tool 两步法（见"一之一"）
cd <你的项目路径>
python wxcli.py decrypt --auto-elevate

# 2. 导出近 2 天全部格式（json/html/excel）
python wxcli.py export --start 2026-08-18 --end 2026-08-19 --format json,html,excel --audio-mode none

# 3. 分析近 7 天用户主动发言的重点会话
python wxcli.py analyze --days 7 --current-user-wxid <你的wxid>

# 4. 导出单个会话为 ChatLab 格式
python wxcli.py export --format chatlab --wxid <会话wxid> --start 2026-08-01 --end 2026-08-19

# 5. 查看配置
python wxcli.py config

# 6. 本地语音转写测试（需先下载 SenseVoice 模型）
python wxcli.py transcribe --sample C:\tmp\test.wav
```

**参数纠错（2026-09-11 实战）**：
- 校验开关是 **`--no-validate`**（默认自动校验），早期文档写的 `--validate` 不存在，会报 `unrecognized arguments`
- `--data-dir` 必须传**包含 wxid_* 子目录的父目录**（如 `D:\RJ\Weixinjilu\xwechat_files`）；直接传账号目录报"未找到微信账号目录"
- `wxcli.py decrypt` 的密钥提取对微信 4.1.x 失效（见"一之一"），4.1.x 用 wcdb-key-tool 解密后 wxcli 直接消费其输出目录

**核心优势**：
- 自动探测 `%USERPROFILE%\Documents\xwechat_files`（微信 4.x）和 `%USERPROFILE%\Documents\WeChat Files`（旧版）
- 自动识别 v3/v4 数据库结构
- 单线程解密（避免 ProcessPool 崩溃）
- 解密前自动清理旧缓存
- 解密后校验 MSG 分片数量与最新消息时间
- 支持 `--auto-elevate` 自动 UAC 提权

**配置文件**：`<你的项目路径>`

---

## 二点六、轻量替代方案：WeChatBridge / 微信流（2026-09-23 收录，[文档]未实测）

> 情报来源：苍何公众号文章拆解，归档于 `<你的项目路径> 学习资料与数据库/文章资料/20260923-WeChatBridge微信流-苍何/`
> 开源地址：https://github.com/freestylefly/WeChatBridge （基于开源项目 Dukou「渡口」二次开发）

**定位**：与本 skill「解密数据库 → 全量导出 → 分析」的重链路**互补不替代**。它站在微信分享菜单层，把选中的聊天记录片段一键转发给 Codex / Claude Code / WorkBuddy / 豆包 / 千问办公 / Obsidian / 剪贴板。

| | 本 skill（extraction） | WeChatBridge |
| --- | --- | --- |
| 路径 | 解密 PC 数据库 → 全量导出 | 微信「转发到其他应用」→ 直发 Agent |
| 粒度 | 群/私聊全量、跨天、结构化 | 选中的会话片段、即兴单发 |
| 权限成本 | 管理员解密，重 | 两个辅助权限，轻 |
| 适用场景 | 批量分析、产量校验、归档底稿 | 「这段聊天帮我看下」式即时需求 |

**特色机制（可借鉴）**：场景化预设——每个群/好友绑定自定义场景要求（如客户群 →「整理需求、风险和下一步」），转发时自动携带，免每次重复解释。

**⚠️ 风控提示**：属微信客户端注入/HOOK 类二开（Dukou 底座），作者上一个微信 CLI 已因此类原因停更。**主号封号风险评估为 0，禁止在生产主号上先行实测**；Windows 兼容性与微信 4.1.x 适配未验证。本 skill 默认仍走解密链路，此节仅作情报收录与备选 awareness。

---

## 三、权限问题：全自动 UAC 提权方案

### 问题现象
Claude Code 环境多次尝试 UAC 弹窗提权都被系统拦截，无法以管理员权限读取微信进程内存。

### 解决方案（已验证）：ShellExecuteW + 轮询等待

利用 `ConsentPromptBehaviorAdmin=0`（用户系统已配置"从不通知"），通过 `ShellExecuteW` 启动管理员子进程实现全自动提权，无需用户手动点击。

**核心脚本**：`<你的项目路径>`

**工作流程**：
1. **非管理员进程**（Claude Code 环境）调用 `ShellExecuteW(None, 'runas', ...)` 启动自身副本
2. **管理员子进程**执行解密逻辑（提取密钥 → 单线程解密所有 `.db` 文件）
3. 子进程完成后写入 `.decrypt_done` 标志文件
4. **主进程**轮询等待标志文件（最长5分钟），读取结果

**关键代码模式**：
```python
import ctypes, sys, os, time

def is_admin():
    try:
        return ctypes.windll.shell32.IsUserAnAdmin()
    except:
        return False

def main():
    if is_admin():
        # 管理员进程：执行解密
        from wxManager.decrypt import get_info_v3
        from wxManager.decrypt.decrypt_v3 import decrypt_db_file_v3
        # ... 解密逻辑 ...
        with open(DONE_FLAG, 'w') as f:
            f.write('OK: 成功X/总计Y')
    else:
        # 非管理员进程：启动管理员子进程，轮询等待
        ctypes.windll.shell32.ShellExecuteW(
            None, 'runas', sys.executable, f'"{__file__}"',
            WORK_DIR, 0  # SW_HIDE = 0
        )
        # 轮询等待 .decrypt_done 标志文件
        for i in range(300):  # 5分钟超时
            if os.path.exists(DONE_FLAG):
                result = open(DONE_FLAG).read()
                os.remove(DONE_FLAG)
                return result
            time.sleep(1)
```

**前提条件**：用户系统注册表 `ConsentPromptBehaviorAdmin=0`（从不通知）。如不是此设置，仍会弹出UAC确认框。

**备用方案（手动执行）**：
如自动提权失败，仍可在管理员终端中手动执行：
```bash
cd <你的项目路径>
python decrypt_wechat_only.py
```

## 四、多进程解密崩溃问题

### 问题现象
`wechat_analyzer.py` 的 `decrypt_wechat()` 调用 `decrypt_v3.decrypt_db_files()`，内部使用 `ProcessPoolExecutor(max_workers=16)` 并行解密。在管理员权限运行时，子进程频繁崩溃：`BrokenProcessPool: A child process terminated abruptly`。

### 解决方案
**改用单线程逐个解密。** 参考 `decrypt_wechat_only.py`：

```python
from wxManager.decrypt.decrypt_v3 import decrypt_db_file_v3

for root, dirs, files in os.walk(wx_dir):
    for file in files:
        if file.endswith(".db"):
            src = os.path.join(root, file)
            dest = os.path.join(output_dir, os.path.relpath(root, wx_dir), file)
            os.makedirs(os.path.dirname(dest), exist_ok=True)
            ok, result = decrypt_db_file_v3(key, src, dest)
```

## 五、标准操作流程（SOP）

### Step 1 — 判断数据新鲜度
- 检查已有的 `wxid_*/Msg` 目录是否存在
- 如果存在，判断其数据时间是否覆盖目标查询范围
- **如果不覆盖 → 必须重新解密**

### Step 2 — 清理旧缓存
```bash
cd <你的项目路径>
rm -rf wxid_*
```

### Step 3 — 管理员权限单线程解密

**方式A：仅解密（推荐，最稳定）**
```bash
# 在管理员 PowerShell 中执行
cd <你的项目路径>
python decrypt_wechat_only.py
```
输出目录：`wxid_xxx/`（位于 `.claude/tools/` 下）

**方式B：解密+导出特定联系人（私聊）**
```bash
python extract_private_chat.py
```

### Step 4 — 连接数据库（在 Claude Code 环境内执行）

```python
from wxManager import DatabaseConnection

msg_dir = r'<你的项目路径>'
conn = DatabaseConnection(msg_dir, 3)
database = conn.get_interface()  # 正确
# 错误：conn.init_database() — 不存在此方法
```

### Step 5 — 查询并导出消息

```python
import datetime

# 时间范围用 Unix timestamp
start = int(datetime.datetime(2026, 5, 15).timestamp())
end = int(datetime.datetime(2026, 5, 19).timestamp())

# 群聊
msgs = database.get_messages(wxid, time_range=(start, end))

# 私聊
contacts = database.get_contacts()
for c in contacts:
    if not c.is_chatroom:
        name = getattr(c, 'remark', '') or getattr(c, 'nick_name', '')
        wxid = getattr(c, 'wxid', '')
        msgs = database.get_messages(wxid, time_range=(start, end))
```

## 六、脚本工具清单

| 脚本 | 用途 | 执行环境 | 备注 |
|------|------|----------|------|
| **`wxcli.py`** | **新统一 CLI**（解密/导出/转写/配置） | Claude 环境 / 管理员终端 | **推荐入口**，自动探测、v3/v4 兼容、自动提权 |
| `decrypt_wechat_only.py` | **仅解密**，清理旧缓存+单线程解密 | 管理员终端 | 旧入口，功能已被 `wxcli.py decrypt` 覆盖 |
| `auto_decrypt_wechat.py` | **全自动解密**（ShellExecuteW提权+单线程+轮询等待） | Claude 环境 | 旧入口，功能已被 `wxcli.py decrypt --auto-elevate` 覆盖 |
| `export_wechat_recent_fast.py` | **快速导出近N天全部有消息联系人** | Claude 环境 | **性能优化版**：先查MSG分片定位有消息wxid，再详细导出，避免遍历全部联系人 |
| `run_admin.py` | 解密+查找特定联系人+导出 | 管理员终端 | 针对"金浙东"场景定制 |
| `extract_private_chat.py` | 解密+查找私聊联系人+导出近30天 | 管理员终端 | 通用私聊导出 |
| `extract_group_chat.py` | 连接已有缓存，导出群聊 | Claude 环境 | 不复用，仅演示 |
| `simple_decrypt_export.py` | 单线程解密+导出群聊 | 管理员终端 | 早期版本，含 `init_database()` 错误 |
| `export_only.py` | 仅导出（依赖已有缓存） | Claude 环境 | 当缓存已是最新时使用 |
| `wechat_analyzer.py` | 一键分析（含自动解密） | Claude 环境 | **有缺陷**：旧缓存陷阱+多进程崩溃 |

## 七、常见错误与解决

| 错误 | 现象 | 解决 |
|------|------|------|
| **旧缓存陷阱** | 查询不到最近几天的消息 | 删除 `wxid_*` 目录，重新解密 |
| **权限不足** | `psutil.AccessDenied: (pid=xxx)` | 必须以管理员权限运行 Python 脚本 |
| **ProcessPool 崩溃** | `BrokenProcessPool: A child process terminated abruptly` | 改用单线程 `decrypt_db_file_v3()` |
| **GBK 编码报错** | `UnicodeEncodeError: 'gbk' codec can't encode character` | `sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')` |
| **数据库对象为空** | `'NoneType' object has no attribute 'get_contacts'` | 用 `DatabaseConnection` + `get_interface()` 重新连接 |
| **数据库初始化失败** | `数据库初始化失败, 请检查路径或数据库版本是否正确` | 缓存损坏/不完整 → 删除重新解密 |
| **Voice 属性不存在** | `type object 'MessageType' has no attribute 'Voice'` | 使用 `MessageType.Audio` |
| **新群未匹配** | 被拉进的新群未被关键词覆盖 | 用 `get_contacts()` 遍历所有群，用 `in` 匹配 |
| **MSG 分片遗漏** | 解密成功但查不到最新消息，数据库中最晚时间远早于预期 | 微信消息按时间分片存储在 MSG0~MSG6 中，必须遍历所有分片查询，不能只查 MSG0 |
| **UAC 提权失败** | 弹窗被系统拦截 | 使用 `auto_decrypt_wechat.py` 自动提权，或手动在管理员终端执行 |

## 八、消息类型处理规范

| 微信消息类型 | `MessageType` 属性 | 处理要求 |
|---|---|---|
| 文本 | `MessageType.Text` | 全部遍历提取 |
| 图片 | `MessageType.Image` | OCR 提取或标注 |
| 语音 | `MessageType.Audio` | **不是 Voice**；优先用微信自带 `voicetrans`，无则调用 Kimi API |
| 视频 | `MessageType.Video` | 记录并判断是否需要查看 |
| 系统消息 | `MessageType.System` | 如邀请入群、撤回消息 |
| 聊天记录分享 | `MessageType.MergedMessages` | 被邀请时对方分享的近期聊天记录 |
| 引用回复 | `MessageType.Quote` | — |

### 语音转文字规则
1. **第一优先**：从消息 XML 的 `<voicetrans transtext="...">` 提取微信自带转文字
2. **第二优先**：本地 SenseVoiceSmall ASR（离线，无需联网）
3. **第三优先**：调用 Kimi (Moonshot) API（需 `KIMI_API_KEY` 环境变量）
4. **兜底**：标注"[语音-未转文字]"

**⚠️ 2026-10-07 两次实测，先错后对（必读）**：

- **错判版（同日已作废）**：曾仅凭"消息 XML 无 transtext + 附件 Rec 目录为空"断言 4.1 语音未点播放即不可提取。**错因：只查了消息库和附件目录，没查媒体库。**
- **正确结论（实测 27/27 条转写成功）**：规则 1 的 transtext 通道在 4.1 确实已断（1968 条语音/视频消息全量 0 命中，此半句仍有效）。**但 4.1 把 SILK 音频直接入库 `media_0.db` 的 `VoiceInfo` 表**（列：chat_name_id / create_time / voice_data BLOB / data_index），与是否点播放、附件目录是否落盘**完全无关**。`msg/attach/*/Rec/` 为空 ≠ 语音不存在。

**4.1 语音标准提取链路（2026-10-07 实测跑通）**：
1. `SELECT voice_data FROM VoiceInfo WHERE chat_name_id=<id> AND create_time=<ts>`（chat_name_id 用 VoiceInfo **自带** Name2Id 表映射，与消息库 rowid 不同体系；voice_data 字节数 = 消息 XML 的 `length` 属性，可交叉验证配对）
2. voice_data 去首字节 `\x02` → `#!SILK_V3` 头
3. `pysilk.decode(BytesIO(blob), out, 24000)` → **裸 s16le PCM，无 RIFF 头**（勿用 `wave` 模块读，直接 `array.array('h', data)`）
4. SenseVoice 转写：sherpa_onnx 1.13.7 下旧封装 asr.py 已不兼容（OfflineSenseVoiceModelConfig 不再收 tokens 参数），直接用 `sherpa_onnx.OfflineRecognizer.from_sense_voice(model=..., tokens=..., use_itn=True)` + `create_stream`/`accept_waveform(24000, samples)`/`decode_stream`
5. 参考实现：`wechat_extraction/_build_scripts/extract_voice.py`（取数）+ `<你的项目路径>`（全链路）+ `<你的项目路径>`（转写）

**视频同理待验**：视频二进制大概率在 media 库其他表（VideoInfo 类），需要时同思路枚举 media_0.db 表名。

**本地 ASR 模型**：`<你的项目路径>`
- 首次使用自动从 ModelScope 下载（约 245MB）
- 依赖 `sherpa-onnx` Python 包

### 强制检查清单（输出报告前逐项勾选）

```
□ 文本消息 — 已全部遍历，关键内容已提取
□ 语音消息 — 已转文字或标注"未识别"（报告标注：总数/已转/未识别）
□ 图片消息 — 已OCR提取文字或描述关键内容
□ 视频消息 — 已记录并判断是否需要查看
□ 文件/文档 — 已记录文件名并判断是否需要处理
□ 链接/文章 — 已抓取标题+摘要，核心观点已提取
□ 位置分享 — 已记录地点和时间
□ 合并转发 — 已分析内部内容
□ 私聊消息 — 已覆盖（不是只分析群聊）
```

## 九、分析报告固定结构

每份分析报告必须包含以下板块：

### 9.1 核心内容
- 今日/近期要点摘要
- 重要决策与待办事项
- 人员动态（请假、入职、任务分配等）

### 9.2 附件与分享汇总（强制板块）

| 类型 | 必须记录 |
|---|---|
| **语音** | 总数 X 条，已转 Y 条，未识别 Z 条（列时间和发送者） |
| **文件/文档** | 文件名列表，标注是否需后续处理 |
| **链接/文章** | 标题 + 摘要 + 核心观点 + 关联性判断 |
| **短视频** | 平台 + UP主 + 核心观点 + 可落地性判断 |
| **商品/购物** | 商品名 + 价格 + 品牌 + 是否关联当前采购/销售 |
| **图片** | OCR 文字内容或关键内容描述 |
| **位置** | 地点 + 时间 + 关联性判断 |

## 十、严重教训

### 教训1：私聊必须检查（2026-05-18）

只分析群聊会导致严重误判。案例中吴欣怡"忘记写日报"的判断是错误的——她已通过私聊向老板请假并获批准。**微信分析必须同时覆盖群聊和私聊。**

触发检查私聊的场景：
- 用户说"某人跟我请假了""某人跟我说了..."
- 员工缺勤/缺报日报时
- 用户明确要求"看看私聊"或"看看某人跟我说了什么"

### 教训2：同一天不要堆叠太多P0事项（2026-05-25）
5月25日同时处理面试+运营周会+系统测试+合同邮寄，多线程密度导致认知负荷过载。这种密度会反映在沟通语气和决策质量上。

## 十一、迭代记录

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-10-07 | v14 | ~~语音转写规则 4.1 实测修正~~：**当日已被 v22 推翻**。transtext 0 命中的实测保留有效，但"未点播放即不可提取"的结论错误 |
| 2026-10-07 | v23 | **wxgf 图片完整攻克（念念项目）**：推翻"体区加密"误判——wxgf=腾讯 WXAM 格式（HEVC 编码存图，**非加密**）。完整链：V2 解密 .dat → headerLen=raw[4](0x13) → 0x255 处 4B BE 长度 → 提取 Annex B HEVC 裸流 → `ffmpeg -i x.hevc -vframes 1 -c:v mjpeg -q:v 3 out.jpg`（7/7 成功）。脚本 `<你的项目路径>`。**防再犯三查**：① NAL 判读先定编码——HEVC 首字节 `>>1&0x3F`（VPS=32/SPS=33/PPS=34/IDR=19），H.264 是 `&0x1F`（SPS=7/PPS=8），找不到 0x67/0x68 ≠ 加密，先怀疑是不是 HEVC；② 数据熵≈8.00 bit/byte 是压缩格式的正常特征，不能作为"加密"判据；③ 附件目录/已知格式查不到 ≠ 数据不存在，必先回权威源（本 skill + [[wechat-41x-msg-decode-pipeline]] + 配方脚本目录）。完整战斗记录：`<你的项目路径> 参谋长/08-家族/念念/01-沟通记录/沟通记录-2026-10-07.md`（语音 VoiceInfo/视频 hardlink/wxgf 三仗全记录）。 |
| 2026-10-07 | v22 | **语音提取正确链路实测固化（推翻 v14 错判）**：4.1 语音 SILK 音频入库 `media_0.db` 的 `VoiceInfo` 表（voice_data BLOB，与点播放/附件目录无关）；去 \x02 前缀→pysilk 解码 24kHz 裸 s16le PCM（无 RIFF 头）→from_sense_voice 转写；27/27 条实测成功。**教训：附件目录为空 ≠ 数据不在库，媒体类数据必先枚举 media_0.db 全表** |
| 2026-05-14 | v1 | `wechat_analyzer.py` 初始版本，第一次解密生成 `wxid_example` 缓存 |
| 2026-05-18 | v2 | 发现旧缓存陷阱 + 多进程崩溃问题，编写 `simple_decrypt_export.py` |
| 2026-05-18 | v3 | 编写 `decrypt_export_admin.py` / `decrypt_export_admin_v2.py`，修复数据库连接 |
| 2026-05-18 | v4 | 编写 `export_only.py`，支持仅导出（缓存已解密时） |
| 2026-05-26 | v5 | 编写 `decrypt_wechat_only.py`（最简洁的单线程解密脚本） |
| 2026-05-26 | v6 | 编写 `extract_private_chat.py` / `extract_group_chat.py` / `run_admin.py` |
| 2026-05-27 | v7 | 编写 `auto_decrypt_wechat.py`，实现全自动UAC提权+单线程解密+轮询等待，无需手动操作 |
| 2026-05-29 | v8 | **重大教训**：发现 MSG 分片遗漏问题。微信消息按时间分片存储在 MSG0~MSG6 中，旧脚本只查 MSG0 导致漏掉 1681 条最新消息。新增"致命陷阱2：MSG 分片遗漏"章节，强制要求遍历所有分片查询。 |
| 2026-05-29 | v9 | **重大教训**：发现按消息数量过滤联系人导致高价值私聊遗漏。"君意 niki朋友 老胡"索要case的3条消息被系统性过滤。新增"致命陷阱3：按消息数量过滤联系人"章节，强制要求遍历所有聊天对象、用商务信号词扫描替代消息量排序。 |
| 2026-06-20 | v10 | **性能优化**：新增 `export_wechat_recent_fast.py`。导出大数据量时，先直接查询MSG分片获取近N天有消息的wxid列表，再调用wxManager详细导出，避免遍历全部联系人，速度从约16分钟降至2-3分钟。 |
| 2026-06-20 | v11 | **重点内容判定规则**：新增"陷阱0"。近30天内，私聊或群聊中只要用户有主动发言，即列为重点内容，必须单独分析并输出"需盯事项"清单。 |
| 2026-06-20 | v12 | **重点内容判定规则升级**：将"陷阱0"升级为"近7天用户主动发言聊天全量语义分析"。不再依赖关键词预过滤，对近7天内用户主动发言的全部聊天逐条语义判断，避免遗漏"男裤费用您直接转我就行了"等非标准表达。近8~30天作为补充扫描范围。 |
| 2026-08-01 | v13 | **新增"陷阱1.5：PC 端数据库未同步最新消息"**。2026-08-01 查询"桐城嘉誉16栋业主交流群"撤回消息时，发现清缓存重新解密后数据库仍缺少最近两天消息，原因是 PC 端未从服务器同步。新增规则：解密后若数据明显缺失，应提示用户退出微信 PC 端重新登录并打开目标聊天窗口触发同步。 |
| 2026-08-19 | v14 | **新增统一 CLI `wxcli.py`**，封装解密、导出、转写、配置；新增 `wechat_extraction` Python 包（config/paths/decryptor/db/reader/audio/asr/image_resolver/analyzer）；自动探测 xwechat_files / WeChat Files；自动识别 v3/v4；新增本地 SenseVoiceSmall ASR、图片解密、HTML/Excel/ChatLab 导出、重点会话分析、pytest 回归测试。旧脚本保留，新入口优先。 |
| 2026-08-27 | v15 | **新增微信 4.1.x 解密方案**：微信 4.1.13 实测使用 passphrase/SQLCipher 4 模式，老版 raw key 扫描失效；推荐使用 wcdb-key-tool 提取 Config.Cipher 派生密钥；补充 passphrase 派生备选路线与实时监控说明；更新微信数据路径示例。 |
| 2026-09-11 | v16 | **实战修正（微信 4.1.x 全链路）**：① wxcli.py decrypt 的 v4 密钥提取（wxManager raw key 扫描）对 4.1.x 失效，4.1.x 直接用 wcdb-key-tool 两步法：extract（--output 传 keys 文件路径）→ decrypt（--output 传目录）；② wxcli 参数纠错：--validate 实为 --no-validate；--data-dir 必须传含 wxid_* 的父目录；③ 多账号机器（xwechat_files 下多个 wxid_*）需逐个 extract+decrypt；④ 本机 wcdb-key-tool 已 clone 至 <你的项目路径> xuanyunz ECS 用专用密钥 ~/.ssh/id_ed25519_caa_oa。 |
| 2026-09-11 | v17 | **新增 1.6「v4 消息导出三个实战坑」**：① 分库 Name2Id 独立——real_sender_id 必须按消息所在 message_N.db 映射，跨库复用 message_0 映射会大面积错标发送者（松慕实战 135 条错标）；② media_0.db 单独重解密必须保持目录结构（keys 键带路径前缀，平铺报"无密钥"SKIP）；③ 同分钟多条语音回填转写需按时间戳排序 + used 集合顺序分配。 |
| 2026-09-23 | v18 | **收录轻量替代方案 WeChatBridge（[文档]未实测）**：分享菜单直发 Agent 形态（GitHub: freestylefly/WeChatBridge，Dukou 二开），与本 skill 解密全量链路互补；含主号风控红线提示。详见「二点六」。 |

## 十二、环境信息

- **MemoTrace 路径**：`<你的项目路径>`
- **新统一 CLI**：`<你的项目路径>`
- **新包路径**：`<你的项目路径>`
- **新配置文件**：`<你的项目路径>`
- **本地 ASR 模型路径**：`<你的项目路径>`
- **解密缓存路径**：`<你的项目路径>`
- **微信数据路径**：`%USERPROFILE%\Documents\WeChat Files\wxid_xxx\` 或 `%USERPROFILE%\Documents\xwechat_files\wxid_xxx\`
- **Python 版本**：3.10+
- **必要依赖**：`PyYAML`、`psutil`
- **语音依赖**：`pysilk`（SILK 语音解码）、可选 `sherpa-onnx`（本地 ASR）
- **Kimi API Key**：环境变量 `KIMI_API_KEY`（语音转文字备用）

---

## 十三、群产量校验（2026-09-24 自 rules/wechat.md 并入）

> 适用：从微信群提取生产/产量日报写入 OA/SQLite，防止整人、整天、整行漏报。

### 13.1 核心流程

1. 删除旧缓存：`rm -rf <你的项目路径>`
2. 管理员权限重新解密
3. 导出目标群消息（含前后各 1 天，防跨天补报）
4. 按发送者+日期统计微信产量行数
5. 与 OA 数据库同维度记录数对比
6. 有缺失 → 人工核对 `messages.md` → 脚本补录并备份；无缺失 → 生成日报/AI 报告

### 13.2 差异校验

```bash
python <你的项目路径> \
  --messages messages.json \
  --db /opt/songmu-oa/backend/data/company.db \
  --reporter-map '{"生产车间：赵小英":"赵小英","夏小英":"赵小英","夏末":"夏末","L":"L","高清伟":"高清伟"}'
```

**判断标准**：微信某天某发送者有行数但 OA 为 0 → 高度疑似整人漏报；微信多于 OA → 核对说明/投入人数是否误计；OA 多于微信 → 多为行拆分差异或历史数据，暂不处理。

### 13.3 人工核对要点

- 消息日期与产量日期是否一致（常见笔误：7-25 消息标题写"7.26 产量"）。
- 区分真实产量、说明、投入人数、备注。
- 截图/图片中的产量需 OCR 或用户转发文字版。

### 13.4 补录要求

写入前自动备份 `company.db`；按 `日期+报工人+产品+工序` 去重；`source` 统一标 `wechat_group`；更新或新增对应日期 `daily_reports`。

## 十四、对话图片归档（2026-09-24 自 rules/wechat.md 并入）

对话中上传的图片（命盘截图、名片、合同照片等）需归档时，优先从本地会话 JSONL 提取。

- **会话文件**：`<你的项目路径>`（项目目录名如 `d--Claude---`；会话 UUID 可在 `Claude VSCode.log` 搜 `sessionId`，或取最新 jsonl）
- **提取命令**：`python <你的项目路径> <jsonl_path> <output_dir>` 或 `--latest <项目目录名> <output_dir>`
- **归档后**：读图确认 → 按项目规范重命名（如 `YYYYMMDD-命盘截图-01-基础信息.png`）→ 放入档案 `原始资料/` → 记录中标「已归档」→ 删临时文件 `extracted_*.png/jpg`

## 十五、底线禁令（2026-09-24 自 rules/wechat.md 并入）

- ❌ 不忽略语音（必须转文字或标记"未识别"）、文件/链接/位置消息。
- ❌ 不只分析群聊不查私聊。
- ❌ 不删旧缓存直接导出；只导出目标当天而忽略跨天补报。
- ❌ 仅凭自动解析写入 OA 不做数据库对比；发现差异不人工核对 messages.md 直接按行数补录。
- ❌ 手动直连生产数据库改数，不写脚本、不备份。

| 2026-09-24 | v19 | **自 rules/wechat.md 并入**「群产量校验」「对话图片归档」「底线禁令」三章（原 rules 常驻自动加载，随全局规则 skill 化迁移至此，触发词不变）。 |
| 2026-09-24 | v20 | **1.6 新增坑④⑤（尚都实战固化）**：④ v4.1 message_content 需 zstd 解压（魔数 28b52ffd）再解析 XML；⑤ v4.1 图片 .dat 的 V2 AES 密钥已换，MemoTrace 未跟进，解密失效——按"会话hash=md5(wxid)+文件mtime+XML length/hdlength"三要素定位 .dat 后，兜底走用户微信PC端右键另存原图。 |
| 2026-09-24 | v21 | **坑⑤攻克（尚都实战）**：4.1 图片 .dat 并非换密钥而是换密钥机制——V2 头下改账号级派生 `md5(f"{code}{wxid}")[:16]`，code 取自 kvcomm 文件名，xor_key=code&0xFF。实现已入 image_resolver.decode_dat_v41，字节级验证通过；遗留 wxgf 动图封装未破（兜底另存为）。 |
| 2026-10-08 | v24 | **坑⑥⑦（禾煦提取实战固化，坑⑥曾致"陷阱1.5"误诊）**：⑥ 会话表名 `Msg_<md5(wxid)>` 的 md5 必须用**内部 wxid**（contact.db `username` 列，形如 wxid_xxx），**绝不能用展示微信号**（contact.db `alias` 列，如 Ftxiaoxu7687）——两者 md5 不同，用 alias 算表名查无此表，且极易误诊为"陷阱1.5 PC 未同步"（10-08 实战：清缓存重解两次均"无会话"，实为查错对象）。定位内部 wxid 权威路径：`contact.db` 按 alias/nick_name/remark/description 任一击中 → 取 username；⑦ 4.1 `message_content` 列 SQLite 按列类型返回 **str（TEXT 列）或 bytes（BLOB 列）**，提取代码必须 isinstance 双分支兼容——str 走 `.encode()` 直接用，bytes 才 decode/zstd 解压；否则 str 调 `.decode` 抛 AttributeError 被外层 except 吞掉，大面积文本消息变 binary 静默丢失（306 条中 202 条中招）。 |
| 2026-10-20 | v25 | **图片 code 取法修正 + 缓存同源（科洋现场照实战）**：`get_account_v2_keys()` 按 mtime 取 code 可能取到 `0` 而解出乱码（头 `e714e193`），必须**枚举全部候选 code 逐个试解并以图片魔数校验**（本次正确值 1161846061）；`_t` 缩略图仅 210×157 无法辨认 PPT 内容，需解 wxgf 原图；查询「昨天/今天」消息前须用监控 runtime 最新 keys 重解密，旧缓存会让会话表存在但 `max(create_time)` 停在旧日期。 |
| 2026-10-27 | v26 | **新增「十六、客户端界面操作」**：微信 4.x 丢弃一切合成输入，只有 Interception 驱动能点动界面；工具沉淀到 `tools/wechat-ui-driver/wx_ui.py`（当日用它完成未下载 PPTX 的下载与视频号录屏）。 |
| 2026-11-05 | v27 | **新增「16.1 发文件给指定联系人」**：wx_ui.py 只映射了翻页方向键，发文件还差 Ctrl+V 与回车，补 `tools/wechat-ui-driver/wx_type.py`（组合键 + 单进程置前点击）。固化四个实战坑：前台被抢（激活与点击跨进程）、贴靠助手（重复最大化）、中文名搜索命中的是备注而非本人、中文不能用键盘打只能剪贴板。 |

## 十六、客户端界面操作：必须走 Interception 驱动

**结论先行**：需要让微信客户端「做动作」（点会话、点下载、翻页、播放）时，**不要试 SendInput / mouse_event / PostMessage / SendKeys / windows-mcp Click，直接用 Interception 驱动**。这些上层方式微信一律丢弃，点前后截图像素差恒为 0，提权到管理员也一样（不是 UIPI 问题）。UIA 树里微信只有 `mmui::MainWindow` → `MMUIRenderSubWindowHW` 一层自绘窗口，没有可 Invoke 的元素。

**工具**：`<你的项目路径>`（详见同目录 README）

```bash
python wx_ui.py click 941 1011 --probe          # 点一下并验证是否点中
python wx_ui.py shot --tag a                    # 窗口截图，打印窗口左上角 origin
python wx_ui.py key PAGEUP 3 --click 1900 900   # 点空白处聚焦后翻页
```

**四条铁律**：

1. **必须先 DPI 感知**（工具内已做）。150% 缩放下不设的话 `GetWindowRect` 返回逻辑坐标，坐标与截图全都偏 1.5 倍。
2. **滚轮无效**。微信不响应注入的 wheel，聊天记录翻页只能用 PageUp / PageDown / 方向键。
3. **会话列表会随新消息重排**。点行之前先截图确认位置，按旧顺序盲点必然点错（10-27 实战点成了隔壁群）。
4. **验证靠像素差，不靠肉眼**。`--probe` 输出 `changed_samples`，几百以上是点中了，0 就是没进去。

**典型链路（取微信里「未下载」的附件）**：截图量会话行位置 → 点开（probe 验证）→ PageUp 翻到目标消息 → 点文件卡右侧格式图标触发下载 → 到 `msg/file/YYYY-MM/` 取文件并与消息 XML 的 `totallen` 比对字节数。

**什么时候仍然让用户动手**：扫码/短信登录、需用户本人确认的敏感操作。除此之外，「你自己点一下」不再作为默认答案。

### 16.1 发文件给指定联系人（2026-11-05 实战固化）

**发之前先过出站扫描**：任何离开本机的文件，先跑
`python <你的项目路径> <文件>`，通过再往下走。

**工具**：`<你的项目路径>`

```bash
python wx_type.py front                 # 置前微信（已最大化就不再最大化）
python wx_type.py click 258 84          # 置前并点击，同一个进程内完成
python wx_type.py paste                 # Ctrl+V
python wx_type.py tap ENTER             # 回车发送
python wx_type.py combo CTRL A          # 任意组合键
```

**完整链路**：

1. 出站扫描通过。
2. **搜微信号，不搜中文名。** 搜「张三」命中的往往是「备注里提到张三」的人。2026-11-05 实测搜「胡娟」，结果前三名是她的朋友、同学、大舅，她本人不在其中；改用通讯录里的微信号（`contact.db` 的 `alias` 或 wxid）精确搜，只出一个联系人结果。
3. 点开这条结果，**先核对聊天内容**再继续——用对话里能确认身份的上下文兜底，别只看头像。
4. 文件送进剪贴板：
   `powershell -NoProfile -Command "Set-Clipboard -Path '<绝对路径>'"`
   再 `Get-Clipboard -Format FileDropList` 回读确认路径对。
5. `wx_type.py click <输入框坐标>` → `wx_type.py paste`。文件会作为**待发附件**出现在输入框里，这时还没发出去。
6. 截图确认附件在、收件人对，再 `wx_type.py tap ENTER`。
7. 发送后截图核对：自己发的文件卡在对话**右侧**，看文件名与大小（微信显示 KB，应与本地字节数换算一致）。

**四个坑（全部实测踩过）**：

- **前台被抢**：把「激活」和「点击」拆成两个进程跑，进程启动的几百毫秒里前台会被别的窗口抢走（实测被 Edge 抢），点击直接落到别人窗口上，改的是用户在用的软件。必须走 `wx_type.py click`，它在同一进程里先置前再点击，并复查前台。
- **贴靠助手**：对已经最大化的窗口再次 `ShowWindow(SW_MAXIMIZE)`，会弹出系统「贴靠助手」抢走前台。`force_front()` 已改成 `IsZoomed()` 为假时才最大化一次。
- **输入法不可控**：不要用键盘打中文，输入法中英文状态未知，打出来是拼音串或候选词。中文一律走剪贴板。
- **要还原用户的窗口**：为抢前台可能最小化过用户正在用的窗口（实测把 VS Code 最小化了），发完必须 `ShowWindow(hwnd, 3)` 还原成最大化。

**算完成的标准**：文件卡在对话右侧 + 文件名正确 + 大小与本地字节数对得上 + 收件人已核对。四项缺一不算发成功，不能只说「已发送」。
