# Skill：手机端"输入法形态"本地 AI 助手的部署与排障

## 1. 技能速览

- **一句话描述**：在一台 Android 手机上，用 Termux + llama.cpp 跑起"判断层 + 生成层"双模型，并让输入法 App 在零云端、零外部 Key 的条件下完成"读消息 → 判意图/风险 → 起草候选 → 手动插入"。
- **适用对象**：想把 AI 聊天助手塞进输入法、又不想把聊天内容发给云端的人；愿意在手机上敲命令的折腾型用户。
- **适用场景**：
  - 在微信/QQ 等任意 App 的输入框里，对"对方刚发来的话"做意图识别与风险打分；
  - 需要离线可用（飞行模式）的写作辅助；
  - 想复刻/魔改开源 IME 型 AI 项目（jev-simple 这类）。
- **不适用场景**：
  - 需要高质量长文生成的日常写作（手机上 4B 模型只能出短候选）；
  - 需要多轮对话记忆的场景（该链路是"单条消息 → 单次判断"）；
  - 需要"自动回复/自动发送"的场景（该 App 明确设计为永不自动发送）。
- **输入**：Android 手机（建议 12GB+ 内存、arm64）、Termux、两个 GGUF 模型文件、一个支持自定义端点的 IME App。
- **输出**：一套三进程本地服务（8080 生成 / 8082 判断后端 / 8081 协议兼容层）+ 可复现的启动脚本 + 一份排障阶梯。
- **来源**：与 trea 的聊天记录整理。
- **版本/日期**：v1 / 2026-10-22。

---

## 2. 触发条件

**应当使用**

- 用户说"想在手机上本地跑、不联网、不填 Key"。
- 用户已有某个"只认 OpenAI 风格接口"或"只认某私有协议"的 App，需要给它在本地造一个后端。
- 用户遇到"服务起了但请求没反应 / 一启动就 Exit 1 / App 里地址总被改回去"这类链路问题。
- 用户要把编译产物发给别人用（需要一条可点击的下载直链）。

**不要使用**

- 用户只想"装个 App 用用"、不打算开 Termux —— 这类需求直接推云端方案更省事。
- 目标是"高质量长文本/多轮对话" —— 手机端算力不匹配，先劝退或改云端。
- 涉及他人聊天记录的批量分析、自动化回复 —— 触及隐私与平台规则，不做。

依据：原话"零云端零 Key""能不能在手机微信上跑""嵌入到输入法里"。

---

## 3. 核心流程

### 步骤 1：先做可行性判定，别急着下载模型

- **目的**：避免把时间花在跑不动的模型上。
- **操作**：
  1. 记录手机三项硬指标：物理内存、CPU 大小核数量、是否 arm64。
  2. 用"模型文件大小 ÷ 可用内存"估算能否整模型驻留：占比 > 70% 基本会触发换页。
  3. 只把"能整块驻留"的模型列为候选。
- **判断标准**：能整块驻留 → 继续；需要换页 → 换更小的量化，或直接放弃该模型。
- **常见错误**：只看参数量不看文件大小；忽略 Android 后台本身要占 3~4GB。
- **依据**：用户给出 Redmi K60 Ultra（天玑 9200+ / 16GB）参数截图后确认可行；30B MoE 约 13.5GB 放入 16GB 机器后实测 1.0 t/s，判为"仅玩具"。

### 步骤 2：判断层与生成层分开选型

- **目的**：两层职责不同，选型标准也不同。
- **操作**：
  1. 判断层选"决策模型"（非自回归、只出概率），体积小、延迟低；本案例 591MB。
  2. 生成层选"小型指令模型"，本案例 Qwen3-4B Q4_K_M（2.49GB），实测 10.85 t/s。
  3. 生成层模型名必须从服务端 `/v1/models` 实际读取后回填，不要手猜。
- **判断标准**：判断层单次 < 5s、生成层首字 < 3s 为可用。
- **常见错误**：拿 MoE 大模型当生成层（1.0 t/s，等于不可用）；用生成模型兼任判断层（延迟翻倍且概率不可靠）。
- **依据**：原话"30B 太慢，用 4B"；`curl /v1/models` 返回 `qwen3-4b-q4_k_m.gguf` 后才回填。

### 步骤 3：给两个 llama-server 都加 API Key

- **目的**：把"本地服务"变成"有鉴权边界的服务"，避免误连、误用。
- **操作**：
  1. 生成随机 Key：`python3 -c "import secrets;print('sk-'+secrets.token_hex(24))"`，写入 `.apikey` 文件。
  2. 两个 llama-server 启动时都带 `--api-key "$KEY"`。
  3. 兼容层通过 `LLAMA_KEY` 环境变量注入同一个 Key。
- **判断标准**：
  - 不带 Key 访问 `/v1/models` 返回 `401`；
  - 带 `Authorization: Bearer $KEY` 返回 `200` 和模型列表。
- **常见错误**：只给生成层加 Key、判断后端裸奔；把 Key 硬编码进脚本（应读 `.apikey`）。
- **依据**：原话"让本地生成个 apikey 吧，稳妥一点"；实测输出 `无 key 生成层: 401` / `有 key 生成层: 200`。

### 步骤 4：判断后端起不来时，先分辨"元数据缺失"还是"模板缺失"

- **目的**：判断 llama.cpp 原生决策端点能不能用，决定走"原生"还是"兼容层"。
- **操作**：
  1. 看退出日志的第一条 error；`failed to init decision model: ...` 是关键线索。
  2. 若报 `decision model has no "systemone" template` → 该 GGUF 只有 `tokenizer.chat_template`，没有 `tokenizer.chat_template.systemone`，**这是硬限制，补元数据无法绕过**。
  3. 若只是 `<arch>.decision.type` 缺失，理论上可补 KV；补之前必须先确认插入字节数是 `general.alignment`（默认 32）的整数倍，否则张量整体错位。
- **判断标准**：补完元数据后进程能稳定驻留且 `/v1/systemone` 返回 200 → 可继续用原生；否则立即转兼容层。
- **常见错误**：
  - 只补 `type` 不补 `systemone` 模板（必然 Exit 1）；
  - 插入 46 字节这种非 32 倍数的内容，解析器算出的数据起点与实际权重差 14 字节 —— 表现为"能加载但输出全错"；
  - 用 `/health` 判断服务可用。
- **依据**：沙箱复现得到 `E srv load_model: failed to init decision model: decision model has no "systemone" template`；原文件 header_end=5,131,969、数据起点 5,132,000（31 字节填充），插 46 字节后差 14 字节，改为 46+50=96 字节后张量字节完全一致。

### 步骤 5：写一个协议兼容层，而不是改 App

- **目的**：让只认私有协议的 App 直接对接本地 llama-server。
- **操作**：
  1. 用标准库起一个 HTTP 服务，暴露 `POST /v1/systemone`（外加 `/health`）。
  2. 内部只调 llama-server 的 `/completion`，取"1"和"0"两个 token 的 logprob，按 `P(yes) = sigmoid(l1 - l0)` 计算。
  3. 严格复刻原协议的 prompt 构造：
     - 二选：`<evidence>\n{state}\n</evidence>\n\nQuestion: {instruction}\nAnswer:`
     - 选择题：`{instructions}\nCandidates:\n- a\n- b\n...\n{phrasing}\nCandidate: {opt}`
     - 打分题：`{instructions}\nScale from lowest to highest: l0 < l1 < ...\n{phrasing}\nLevel: {lk}`
  4. 选择题对每个候选各问一次 yes/no，再用 `p_i / Σp` 归一化。
  5. 打分题对每个阈值问"是否 ≥ 该级别"，把通过数累加得到等级与分布。
- **判断标准**：同一句话在兼容层与原版二进制的输出接近（正向样本 0.977 vs 0.9728）。
- **常见错误**：
  - 用 `n_probs=300` 取负样本的"0" —— 负向时 "0" 的 logprob 排到第 79854 位，根本取不到；
  - 用 grammar 约束只出 "0"|"1" —— 返回空 token；
  - 自己发明 prompt 措辞（必须逐字复刻，否则概率全偏）。
- **依据**：官方 wiki 明确"llama-server 只给零件，不给分类器"；对比表：`My invoice... → 0.977 vs 0.9728`、`Could you send me... → 0.981 vs 0.9855`。

### 步骤 6：按"排障阶梯"定位，不要猜

- **目的**：把"没反应"拆成可判定的层次。
- **操作**（从上往下逐层验证）：

  | 现象 | 含义 | 下一步 |
  |---|---|---|
  | `curl -s` 无任何输出 | 端口上没有服务（`-s` 会吞掉连接拒绝） | 看进程是否存活、看启动日志 |
  | `404 {"detail":"Not Found"}` | 打到了兼容层但路径不对（例如把生成请求发到 8081） | 检查端口分配 |
  | `501 This model is not a decision model` | 打到 llama.cpp 原生决策端点但模型无决策元数据 | 改走兼容层 |
  | `401` | Key 缺失或不一致 | 比对 `.apikey` |
  | `Exit 1` 且日志无端口信息 | 端口被旧进程占用 | 先 kill 再起 |
  | `/health` 返回 ready 但业务请求失败 | 上游 `/health` 不校验 Key，假通过 | 用真实业务请求验证 |

- **判断标准**：每一层都用"能返回 200 + 可解析 JSON"的真实请求确认，而不是用 `/health`。
- **常见错误**：把 `/health` 200 当成"整条链路通"。
- **依据**：用户先看到 `8081 -> {"status":"ready"}` 但业务失败；日志显示旧版兼容层未带 `LLAMA_KEY`，而 `/health` 不校验鉴权。

### 步骤 7：配置 App 时，Key 是最关键的开关

- **目的**：避免 App 偷偷回落到内置云端中转。
- **操作**：
  1. 判断层：地址 `http://127.0.0.1:8081/v1`，模型 `jev-latest`，Key 填本地生成的那串。
  2. 生成层：地址 `http://127.0.0.1:8080/v1`，模型取 `/v1/models` 的 id，Key 填同一串。
  3. 若要纯本地：**Key 必须非空**；Key 为空会被替换成内置中转地址。
  4. 生成层留空 + 判断层配好 = "仅判断模式"。
- **判断标准**：页面底部状态行不应再出现内置中转域名或云端域名；点"测试"能返回真实意图与风险。
- **常见错误**：
  - 只改地址不填 Key（地址被丢弃）；
  - 两个层都填 8081（生成层 404）；
  - 只看下拉框标签，不看底部状态行。
- **依据**：页面原文"不填 Key 时自动走内置中转"；用户截图底部 `当前: openai · glm-4-flash · Key 内置中转（免填）`；以及 `HTTP 404: {"detail": "Not Found"}` 说明请求打到了兼容层。

### 步骤 8：用"飞行模式 + 剪贴板"做最终验收

- **目的**：证明整链路不依赖网络。
- **操作**：
  1. 开飞行模式。
  2. 复制一条对方发来的消息 → 切到该输入法 → 点"分析剪贴板"。
  3. 确认出现：意图徽章 + 风险等级 + 风险文案 + 建议（仅判断模式下无候选）。
  4. 点一条候选/建议，确认内容**插入输入框**而非自动发送。
- **判断标准**：飞行模式下仍出结果 = 全本地。
- **常见错误**：点成"分析输入框文字"（分析的是自己打的字，必然偏"闲聊"）。
- **依据**：原话"开飞行模式再点试一试"；App 设计"永不自动发送"。

### 步骤 9：分发时先解决签名，再解决下载

- **目的**：让别人能装上。
- **操作**：
  1. 自行编译出的包与上游签名不同 → **必须先卸载旧版**。
  2. 上传为 GitHub Release asset（公开仓库才有免登录直链），上传后用 `curl -I` 验证 `200` 与 `application/vnd.android.package-archive`。
  3. 国内网络卡顿时把 `github.com` 换 gh-proxy 前缀。
- **判断标准**：直链返回 200 且 content-length 与本地字节数一致。
- **常见错误**：直接覆盖安装（签名冲突 → "应用未安装"）；把 APK 塞进 git 仓库而不是 Release（下载体验差、仓库膨胀）。
- **依据**：实际验证 `200` / `content-type: application/vnd.android.package-archive` / 5,768,197 字节。

---

## 4. 规则与偏好

### 硬规则

1. **不确认端口上的服务身份，就不要下结论。** `curl -s` 无输出 ≠ 服务正常。
2. **不能用 `/health` 判断链路可用**，必须用真实业务请求（`/v1/systemone`、`/v1/models`）验证。
3. **补 GGUF 元数据时，插入字节数必须是 `general.alignment` 的整数倍。**
4. **App 里生成层 Key 留空会被替换成内置云端中转**，要纯本地就必须填非空 Key。
5. **生成层地址不能填 8081**（那是判断层兼容层）。
6. **自行编译的 APK 与上游签名不同，安装前必须卸载旧版。**
7. **启动脚本里必须 `pkill` 旧进程**，否则新进程 `Exit 1`（端口占用）。
8. **不改提示词措辞就不换模型**；判断不准时优先调 criteria 文案。

### 软偏好

- 所有服务绑 `127.0.0.1`，不暴露到局域网。
- 服务统一用 `--api-key` + 本地随机 Key，即使本机不校验。
- 用 `setsid` / `nohup` + `termux-wake-lock` 维持后台存活。
- 先用 curl 证明链路，再动 App 配置；App 只作为最后一层验证。
- 结论先行，命令可直接整段粘贴，避免用户按顺序拼命令出错。

### 禁忌

- 不要用"看起来像正常退出"的 503/401 当成成功（应读 `%{http_code}`）。
- 不要用大模型"兼任"决策层来替代专用决策模型。
- 不要把用户聊天内容打到任何公网端点（含"临时调试"）。
- 不要在日志/文档里留明文 Key。
- 不要承诺手机端能"自动回复"。

### 冲突与待确认

- **"0" token 读取方案存在两个候选但未定论**：全词表 `n_probs=130560` 可稳定取到但响应体巨大；`logit_bias` 禁用除 37/38 外全部 token 理论上可行，聊天记录中仅停留在"准备验证"，未见成功结果。
- **短句判"闲聊"的边界不明**：实测"在吗"判成"问进度 0.33 / 闲聊 0.28"，与"用户反馈全是闲聊"矛盾，最终未拿到结果卡片截图确认，判为未闭环。
- **上游项目的分发许可未确认**：聊天中只讨论了"能否公开上传构建产物"，未核实上游 LICENSE。
- **30B 模型是否可用 Vulkan/多线程救活**：只提出选项，未实测。

---

## 5. 可复用模板/话术/命令

### 5.1 一键启动脚本（三进程 + 健康检查）

```bash
#!/data/data/com.termux/files/usr/bin/bash
cd ~/llama.cpp
[ -f .apikey ] || python3 -c "import secrets;print('sk-'+secrets.token_hex(24))" > .apikey
KEY=$(cat .apikey)

pkill -f "llama-server"; pkill -f "jevjudge.py"; sleep 1
termux-wake-lock

nohup ./build/bin/llama-server -m <生成模型>.gguf -c 4096 -t 6 \
  --host 127.0.0.1 --port 8080 --api-key "$KEY" > gen.log 2>&1 &
nohup ./build/bin/llama-server -m <判断模型>.gguf -c 2048 -t 4 \
  --host 127.0.0.1 --port 8082 --api-key "$KEY" > judge.log 2>&1 &
LLAMA_URL=http://127.0.0.1:8082 LLAMA_KEY="$KEY" PORT=8081 \
  nohup python3 jevjudge.py > jevjudge.log 2>&1 &

sleep 12
for p in 8080 8082 8081; do
  echo -n "$p -> "
  curl -s -H "Authorization: Bearer $KEY" http://127.0.0.1:$p/health; echo
done
```

### 5.2 鉴权双向验证

```bash
KEY=$(cat ~/llama.cpp/.apikey)
echo -n "无 key: "; curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/v1/models
echo -n "有 key: "; curl -s -H "Authorization: Bearer $KEY" \
  http://127.0.0.1:8080/v1/models | head -c 60; echo
```

预期：`401` / `{"models":[...]}`。

### 5.3 判断层真实业务验证（不要用 /health）

```bash
KEY=$(cat ~/llama.cpp/.apikey)
curl -s http://127.0.0.1:8081/v1/systemone \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $KEY" \
  -d '{"model":"jev-latest","state":"<待判文本>",
  "questions":{"intent":{"type":"choice","instructions":"这句话的真实意图是什么？",
  "criteria":{"派活":"对方要我做一件事或接一个任务","问进度":"对方在询问进展",
  "闲聊":"对方只是在聊天、分享或表达感受，没有具体要求"}}}}'; echo
```

### 5.4 批量判别力回归（4 条基线句）

```bash
ask() {
  curl -s http://127.0.0.1:8081/v1/systemone -H 'Content-Type: application/json' \
    -H "Authorization: Bearer $KEY" -d "{\"model\":\"jev-latest\",\"state\":\"$1\",
    \"questions\":{\"intent\":{\"type\":\"choice\",\"instructions\":\"这句话的真实意图是什么？\",
    \"criteria\":{ ...八类... }}}}" \
    | python3 -c "import json,sys;d=json.load(sys.stdin)['answers']['intent'];print('$1 →',d['choice'],d['confidence'])"
}
ask "这个需求你今天跟一下"        # 期望 问进度
ask "明天下午三点开个会，会议室A"  # 期望 约会议
ask "为什么这个数据又错了？"       # 期望 批评 / 要解释
ask "你这版做得不错，辛苦了"       # 期望 夸奖
```

### 5.5 兼容层核心公式与 prompt

```
state_text = "<evidence>\n" + <str 或 json dumps> + "\n</evidence>"
prompt     = state_text + "\n\nQuestion: " + instruction + "\nAnswer:"
P(yes)     = 1 / (1 + exp(clamp(l0 - l1, -30, 30)))
```

### 5.6 GGUF 元数据补丁（仅在确认有 systemone 模板时使用）

```
{ head -c <header_end> 原文件
  printf '<KV 区>'
  tail -c +<header_end+1> 原文件
} > 新文件

# 自检：新文件字节数 - 原文件字节数 == 插入字节数
# 且 插入字节数 % 32 == 0
# 且 用解析器读出的第一个张量偏移 == 原偏移 + 插入字节数
```

### 5.7 发布到 GitHub Release

```bash
gh repo create <仓库名> --public --source=. --push
gh release create v0.2 <本地APK> --title "<标题>" --notes "<变更说明>"
curl -sIL https://github.com/<用户名>/<仓库名>/releases/download/v0.2/<文件名>.apk \
  | grep -E '^HTTP|content-length|content-type' | tail -6
```

---

## 6. 示例

### 成功示例 1：用兼容层替代原生决策端点

- **场景**：判断模型 GGUF 缺 `tokenizer.chat_template.systemone`，llama.cpp 原生 `/v1/systemone` 必然 Exit 1。
- **做法**：不改模型、不改 App，另起一个 8081 进程，暴露同形状的 `POST /v1/systemone`，内部只调 8082 的 `/completion` 读两个 token 的 logprob。
- **结果**：App 判断层测试返回 `意图「问进度」59%，风险 2.4/9`；四句基线分别命中问进度/约会议/批评/夸奖。
- **可复用点**：**"协议不兼容"优先在中间加一层翻译，而不是去改模型或改客户端。**

### 成功示例 2：从现象到根因的排障

- **场景**：`nohup llama-server ...` 后立刻 `[2]+ Exit 1`，`curl /health` 无输出。
- **做法**：逐层验证 —— 端口有无监听 → 启动日志第一条 error → 发现 `decision model has no "systemone" template`。
- **结果**：排除 Key、地址、补丁字节数等猜测，直接判定为"硬限制，必须改架构"。
- **可复用点**：**先读第一条 error，再谈假设。**

### 失败示例 1：46 字节补丁导致张量错位

- **场景**：想给 GGUF 补 `decision.type`，直接在 header 末尾插入 46 字节。
- **做法**：`head -c / printf / tail -c +` 拼接。
- **结果**：文件能加载，但解析器算出数据起点 5,132,032，真实权重在 5,132,046，差 14 字节 —— 权重整体错位，输出不可信。
- **可复用点**：**任何 header 插入都必须凑到对齐数（默认 32）的整数倍，并用"偏移差 == 插入量"自检。**

### 失败示例 2：`/health` 假通过

- **场景**：8081 返回 `{"status":"ready"}`，用户以为链路已通。
- **做法**：只 curl `/health`。
- **结果**：实际跑的是旧版兼容层、未带上游 Key；上游 `/health` 不校验鉴权所以仍返回 ok，业务请求才是 401。
- **可复用点**：**健康检查必须打业务路径。**

### 失败示例 3：只改地址不填 Key

- **场景**：生成层地址已改成 `127.0.0.1:8080/v1`，但 Key 框空着。
- **做法**：直接点测试。
- **结果**：请求仍发往内置云端中转，底部显示 `Key 内置中转（免填）`；用户误以为"App 不听话"。
- **可复用点**：**先看状态行里的三个字段（协议名 / 模型 / Key 来源），再看报错。**

---

## 7. 检查清单

**执行前**

- [ ] 确认物理内存与模型文件大小的比例（>70% 直接换方案）
- [ ] 确认手机是 arm64、Termux 已装 `python3`
- [ ] 确认两个模型文件都存在且字节数对得上
- [ ] 确认 8080 / 8081 / 8082 无旧进程占用

**执行中**

- [ ] 两个 llama-server 都带 `--api-key`
- [ ] 兼容层进程带上了 `LLAMA_KEY`
- [ ] 用 `401` / `200` 双向验证 Key 生效
- [ ] 用真实 `/v1/systemone` 验证，而不是 `/health`
- [ ] 跑 4 条基线句确认判别力没有退化
- [ ] App 里两层地址分别指向 8081 / 8080
- [ ] App 里两层 Key 都非空
- [ ] 页面底部状态行无云端域名、无"内置中转"

**执行后**

- [ ] 飞行模式下仍能出意图 + 风险
- [ ] 复制 → 分析剪贴板 → 插入输入框全程可用
- [ ] 确认不会自动发送
- [ ] 把 `start.sh` 加入 shell 启动项
- [ ] 分发前确认已卸载旧版、直链 200、字节数一致

---

## 8. 待确认问题

1. 负向样本读 `"0"` 的稳定实现：全词表 `n_probs` 与 `logit_bias` 禁 token 两条路，哪条在真机上延迟可接受？
2. 键盘"分析剪贴板"实际传给判断层的 `state` 是什么（纯文本？含上下文？含发送者/时间？）—— 决定了"为何短句被判闲聊"的解释。
3. 判"闲聊"的边界值：多短、多客套的句子应当归入闲聊？是否需要用 criteria 文案调优？
4. jevos 系模型对中文的判别质量上限，是否需要换更强的中文决策模型？
5. 自行编译的 APK 公开分发是否合规？上游 LICENSE 是什么？
6. 4B 生成层在输入法内端到端耗时（判断 + 起草 + 渲染）实测值是多少？
7. 30B MoE 能否通过 Vulkan 后端或多线程参数从 1.0 t/s 提到可用区间？
8. HyperOS 后台查杀下，长时间不使用后三进程是否仍存活？

---

## 9. 附录：原始依据摘录

| 主题 | 原话摘录（已脱敏） |
|---|---|
| 目标 | "零云端零 Key""能不能在手机微信上跑""嵌入到输入法里" |
| 硬件 | "Redmi K60 Ultra，24GB+1TB"（截图参数：天玑 9200+ / 16GB 物理内存） |
| 大模型判定 | "30B 只有 1.0 t/s"→ 判为"仅玩具，不可作后端" |
| 模型并存 | "我要 4b 和 30b 都同时存在" |
| 端口分配 | "8080 生成 / 8082 判断后端 / 8081 兼容层" |
| 启动失败 | "`[2]+ Exit 1`，随后 curl `/v1/systemone` 无输出" |
| 真实报错 | "`E srv load_model: failed to init decision model: decision model has no \"systemone\" template`" |
| 补丁字节 | "原文件 619,289,312 字节，补后 619,289,408 字节（+96）" |
| 读概率 | "`P(yes) = sigmoid(l1 - l0)`，读 answer token '1' 和 '0' 的 logits" |
| 负样本难题 | "rank of 0: 79854 / rank of 1: 130559" |
| 官方对比 | "`My invoice shows a charge I do not recognize.` 官方 0.9728 vs 本地 0.977" |
| 判别力基线 | "这个需求你今天跟一下 → 问进度 0.6245""明天下午三点开个会 → 约会议 0.8562" |
| Key 需求 | "让本地生成个 apikey 吧，稳妥一点" |
| Key 效果 | "无 key 生成层: 401""有 key 生成层: 200 + 模型列表" |
| 被顶掉 | "不填 Key 时自动走内置中转（http://<已脱敏>:11111/v1 · glm-4-flash）" |
| 状态行 | "当前: openai · glm-4-flash · Key 内置中转（免填）" |
| 误配 | "`HTTP 404: {\"detail\": \"Not Found\"}`" —— 生成层填成了 8081 |
| 控件 bug | "每次我点模型他就给我调回去" |
| 键盘切换 | "切换键盘那个按键他也不灵" |
| 判断反馈 | "为什么这个模型的判断层每个都是轻松回应" |
| 行动建议映射 | "闲聊 → 轻松回应 / 可以互动 / 不用当真" |
| 分发 | "把这个下载链接放 github 上" |
| 安装约束 | "签名不同，必须先卸载旧版" |
