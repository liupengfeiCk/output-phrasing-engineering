# Output Phrasing Engineering（输出措辞工程）

> **核心理论**：自回归语言模型的每一个 token 都在影响后续 token 的概率分布。通过四个层面的协同——人设锚定整体输出倾向、措辞替换改变推理起点、可见思维链锚定推理方向、硬规则拦截致命降级——可以在不修改模型权重的前提下，系统性地提升 AI 编码输出的质量。

> **适用场景**：本框架同时覆盖两种主要使用模式：
> - **Agent 模式**（CC / CodeBuddy / Cursor 等）：AI 通过工具调用（read_file, replace_in_file, write_to_file, execute_command）来修改代码，不直接输出完整代码
> - **Chat 模式**（ChatGPT / API 对话等）：AI 直接在回复中输出完整代码，用户复制粘贴
>
> 两种模式的思维链和降级检测设计相同，但实现阶段的行为规则不同。

---

## 零、人设与行文风格——最底层的质量锚点

### 为什么人设和行文风格排在最前面

人设和行文风格是在比具体规则更底层的层面影响 AI 输出的。它们影响的是模型**整体的 token 概率分布**，而不只是某个特定位置的决策。

```
一个被设定为"追求极致代码质量的高级工程师"的 AI
  → 整体输出倾向偏向严谨、完整、深思熟虑
  → 即使在规则没有覆盖到的地方，也倾向于做出高质量的决策
```

从自回归的角度：人设描述中的每一个词都会影响后续所有 token 的概率分布。"高级工程师"、"代码质量极高"、"第一次就做对"这些词汇，会持续地把概率分布拉向高质量的方向。这是一种**全局性的推理偏置**，比局部的措辞替换更广泛。

**重要**：人设和行文风格中只能出现正面描述和高质量示例。反面示例中的低质量词汇也会被模型"学到"，提升低质量输出的概率。具体的低质量表述只在措辞替换表的"当你即将说"列中出现——那是映射表的必要结构，不可避免。

### 行文风格为什么重要

行文风格定义了 AI 的表述方式。在自回归模型中，表述方式和思考方式是同一个过程——模型的"思考"就是它生成的 token 序列。

- **"精确"** → AI 在生成每一句话时都会被拉向具体的表述，而具体的表述会引导出具体的实现
- **"直面"** → AI 在遇到困难时被拉向"拆解问题并逐一解决"
- **"谨慎"** → AI 在做判断之前被拉向"先搜索确认"

行文风格和措辞替换的关系：措辞替换是在特定位置的精确干预，行文风格是在所有位置的全局偏置。二者配合，前者拦截已知的降级模式，后者防御未知的新降级模式。

### 具体的人设和行文风格定义

详见规则文件中的"第零节"——两个版本（Agent/Chat）各有针对性的定义。

---

## 全局架构

```
用户发出任务
      │
      ▼
┌─────────────────────────────────────────────┐
│  第零层：人设与行文风格                       │
│  全局性的推理偏置                             │
│  → 在所有位置持续影响 token 概率分布          │
├─────────────────────────────────────────────┤
│  第一层：强制可见思维链                       │
│  结构化格式约束 + 措辞替换映射表              │
│  → 在推理起点就锚定高质量方向                 │
├─────────────────────────────────────────────┤
│  第二层：元认知决策协议                       │
│  遇到意外问题时的完整决策流程                 │
│  → 保持灵活性，同时防止静默降级               │
├─────────────────────────────────────────────┤
│  第三层：硬规则兜底                           │
│  绝对禁止的降级行为                           │
│  → 最后一道防线，拦截漏网的质量降级           │
└─────────────────────────────────────────────┘
      │
      ▼
   高质量输出
```

**为什么是这个顺序？**

- 第零层是**全局偏置**——人设和行文风格持续影响所有 token 的概率分布，覆盖面最广但粒度最粗。

- 第一层作用在生成的**最早阶段**——思维链。此时的每个 token 都会影响后续所有输出。投入产出比最高。
- 第二层处理**不可预见的情况**。第一层的映射表无法覆盖所有场景，需要一个通用决策协议。
- 第三层是**兜底**。无论前两层是否生效，这些硬规则都能拦截最致命的质量降级。

---

## 一、第一层：强制可见思维链 + 措辞替换

### 1.1 为什么思维链必须可见

自回归模型没有"内心世界"。它的全部认知能力来自已生成的 token 序列：

```
P(token_n) = f(token_1, token_2, ..., token_{n-1})
```

**写出来的思考 = 真正的思考。没写出来的 = 没想过。**

可见思维链的三重作用：

1. **增加有效推理长度**：模型在做决策之前有更多 token 来"铺垫"正确方向
2. **自我约束效应**：模型写下"需要处理三个边界条件"后，一致性倾向会"拉"它去处理所有三个
3. **人类可监督**：用户能在思维链阶段就发现问题，而不是等代码写完才发现方向错了

### 1.2 结构化思维链格式（含降级检测）

思维链不是"写一篇文章"，而是一个**紧凑的结构化检查清单**。每个字段都是一个推理锚点。

关键设计：思维链的最后一个字段是 `degradation_check` —— **强制 AI 在开始执行之前，先检查自己的分析是否已经走向降级**。

#### Agent 模式格式（CC / CodeBuddy / Cursor）

Agent 模式的完整流程分为三个阶段：**侦察 → 分析 → 执行**。

```
用户给出任务
      │
      ▼
┌─────────────────────────────────┐
│  阶段一：Reconnaissance（侦察）  │
│  读取相关文件、搜索代码库        │
│  → 先搞清楚"现在是什么样"        │
├─────────────────────────────────┤
│  阶段二：Analysis（分析）         │
│  基于侦察结果做方案设计          │
│  → 知道了现状，才能做合理决策     │
├─────────────────────────────────┤
│  阶段三：Execution（执行）        │
│  按 execution_plan 调用工具      │
│  → 按图施工                      │
└─────────────────────────────────┘
```

**为什么必须先侦察？**

不读代码就写 analysis，等于闭着眼睛做手术计划。`key_challenges` 是猜的、`edge_cases` 是编的、`affected_scope` 是不全的。这本身就是一种降级——**信息不足导致的质量降级**。

**格式不用抽象模板展示——直接看教学示例。** 以下是一个完整的 Agent 模式示例（给 UserService 加邮箱校验）：

```
<reconnaissance>
goal: 了解当前注册流程的代码结构、项目中已有的校验方式和错误返回模式
actions:
  - read: service/user.go — 看 Register 方法的当前实现
  - search: "validate" — 看项目中是否已有校验工具函数
</reconnaissance>
```

[调用 read_file 读取 service/user.go]
[调用 search_content 搜索 "validate"，发现项目已有 util/validate.go 含 ValidatePhone]

```
<analysis>
context:
  - service/user.go 的 Register 方法接收 email + password，直接插库，没有任何校验
  - 项目已有 util/validate.go，包含 ValidatePhone 函数，返回 error
  - 项目统一使用 AppError{Code, Message} 返回错误
needs: 在注册流程中加入邮箱格式校验，沿用项目已有的 validate 工具和 AppError 风格
key_challenges:
  - 校验规则需覆盖常见邮箱格式
  - 必须沿用项目已有的 ValidateXxx 命名风格和 AppError 返回模式
approach: 在已有的 util/validate.go 中新增 ValidateEmail 函数，在 Register 方法开头调用。
  选择理由——可维护性：复用已有工具包；健壮性：正则覆盖主流格式；可扩展性：统一管理
edge_cases: 空字符串、纯空格、缺少@、多个@、域名无点号、超长(>254字符)
affected_scope: util/validate.go, util/validate_test.go, service/user.go
execution_plan:
  - step_1: 在 util/validate.go 末尾追加 ValidateEmail 函数
  - step_2: 在 util/validate_test.go 末尾追加 TestValidateEmail，覆盖6个边界条件
  - step_3: 在 service/user.go 的 Register 方法开头插入 ValidateEmail 调用
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → NO，沿用了项目现有模式而非自建新包，在已有约定上扩展是最适合的选择
  - 是否省略了已知边界条件？ → NO，已列出6个具体的边界条件，每个都对应一种无效邮箱格式
  - 是否因改动量大而想简化？ → NO，3个文件的改动量完全合理，不存在需要简化的压力
  - 是否打算跳过某些文件？ → NO，3个文件全部在 execution_plan 中
  - execution_plan 是否覆盖 affected_scope 所有文件？ → YES，3步分别对应 validate.go、validate_test.go、user.go
  - context 是否充分？ → YES，已了解现有校验风格（ValidatePhone）、错误返回模式（AppError）、Register 方法的当前实现
  - 是否有发现了但被我判断为"无关紧要"而跳过的问题？ → NO，侦察中发现的所有信息都已纳入方案设计
  - context 是否充分？ → YES，已了解校验风格和错误返回模式
  → 全部通过，开始执行
</analysis>
```

[调用 replace_in_file 修改 util/validate.go]
[调用 replace_in_file 修改 util/validate_test.go]
[调用 replace_in_file 修改 service/user.go]

**所有任务——无论大小——都使用这个完整格式。** 不存在"简单任务"的精简版。原因：给一个精简模板，就是给 AI 一个合法的偷懒入口。AI 会把越来越多的任务归类为"简单任务"，然后用精简模板跳过思考。一个变量重命名看似简单，但不搜索引用就会漏改文件——"简单"是最大的降级触发器。

**三个阶段之间的依赖关系**：

- **reconnaissance → analysis**：`context` 字段强制 AI 写出从侦察中学到了什么。如果 `context` 写不出来有价值的信息，说明侦察不充分，需要补充。**特别注意跨轮次信息丢失**：当 AI 发现上一轮已经做过侦察时，它倾向于在 context 中写"上一轮已分析过 XXX"来代替具体事实。这会导致本轮推理基于一句概括而非具体信息，直接降级。正确做法是：即使信息来自上一轮，也必须在本轮 context 中重新写出具体事实。
- **analysis → execution**：`execution_plan` 中不再需要"读取文件"步骤（那是侦察阶段的事），只包含实际的修改操作。
- **degradation_check 新增检查项**："我的 context 是否充分"——防止 AI 只草草看了一个文件就开始设计方案。

**Agent 模式的关键差异**：
- 没有 `<implementation>` 块——AI 不输出完整代码，而是调用工具逐步修改
- 新增 `<reconnaissance>` 阶段——先收集信息再做决策
- 新增 `context` 字段——迫使 AI 把侦察到的事实写出来，作为方案设计的依据
- 新增 `execution_plan` 字段——列出具体的工具调用步骤，相当于"施工图纸"
- `degradation_check` 增加了三个 Agent 专属检查项：
  - "是否打算跳过某些文件的修改"
  - "execution_plan 是否覆盖了 affected_scope"
  - "context 是否充分"——防止信息不足就动手

#### Chat 模式格式（ChatGPT / API 对话）

Chat 模式同样不提供精简版——所有任务使用完整格式。以下是一个完整示例（实现文件上传断点续传）：

```
<analysis>
needs: 分片上传 + 断点续传，中断后从中断位置继续
key_challenges:
  - 分片管理、进度持久化、分片完整性校验、并发分片上传
approach: 前端切片上传，后端存储分片并记录进度，全部完成后合并。
  选择理由——可维护性：分片逻辑独立；健壮性：进度持久化支持断点恢复；可扩展性：存储层可替换
edge_cases:
  - 分片乱序、秒传（哈希匹配）、服务器重启后恢复、最后一片大小不同
  - 文件大小上限校验（可配置，默认2GB）
  - 文件类型白名单校验（通过文件头魔数判断）
affected_scope: controller/upload.go, service/upload.go, store/progress.go, config/upload.go
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → YES ⚠️
    审视：分片存本地文件系统，但生产环境多实例会导致分片分散
    修正：改为对象存储（S3/MinIO），支持多实例共享
  - 是否省略了边界条件？ → NO
  - 是否打算用注释/省略号代替实现？ → NO
  - 是否因内容长而想换方案？ → NO，涉及4个文件，分2部分输出
  → 发现1项 YES，已修正，以修正后方案进入实现
</analysis>

<implementation>
// config/upload.go
package config

type UploadConfig struct {
    MaxFileSize    int64    `yaml:"max_file_size"`
    AllowedTypes   []string `yaml:"allowed_types"`
    ChunkSize      int64    `yaml:"chunk_size"`
    TempExpireHour int      `yaml:"temp_expire_hour"`
    StorageType    string   `yaml:"storage_type"` // "s3" | "minio"
}
// ... 第1部分/共2部分，完整实现 ...
</implementation>
```

**为什么用 XML 标签而非 Markdown 标题？**

- XML 标签是模型训练中见过的**强结构标记**，遵循率高于自由格式
- 标签的开闭配对天然强制内容完整——写了 `<analysis>` 就必须写 `</analysis>`，中间不会被跳过
- 嵌套结构清晰，工具层可以解析和校验

### 1.3 措辞替换映射表（核心创新）

**原理**：不是"禁止"降级短语，而是指定精确的替换词。当模型在生成过程中"即将"产出降级短语时，替换词会将后续推理引向高质量路径。

**为什么替换比禁止有效？**

```
禁止 "最简单的做法是" → 模型在此位置概率被压低 → 下一个 token 选什么？不确定 → 可能换一种方式偷懒
替换为 "综合评估后推荐的方案是" → 后续 token 被导向"评估"和"推荐"的语义空间 → 高质量路径确定
```

#### 捷径导向型替换

| 当你即将说 | 替换为 |
|---|---|
| "最简单的做法是" | "综合评估后，推荐的方案是" |
| "最快的方式是" | "兼顾质量和效率，合适的方案是" |
| "最高效的做法是" | "在保证健壮性的前提下，推荐" |
| "为了简单起见" | "为了保持代码清晰度" |
| "一个快速的解决方案是" | "一个可靠的解决方案是" |
| "我们可以简单地" | "我们可以通过以下步骤" |
| "不需要那么复杂" | "合理的实现粒度是" |
| "这里我们直接" | "这里的标准做法是" |

#### 放弃方案型替换

| 当你即将说 | 替换为 |
|---|---|
| "这个文件太大了，我换一种思路" | "这个文件较大，我将分步骤完成，首先处理" |
| "代码太长了，我换个方式" | "实现内容较多，我分为N个部分输出，第1部分是" |
| "这样做太复杂了，不如" | "这个问题需要拆解，子问题分别是" |
| "考虑到篇幅限制" | "为了确保完整性，我将分段输出" |
| "为了节省空间" | "完整实现如下" |
| "这里就不展开了" | "这部分的完整实现是" |
| "剩下的部分类似" | "接下来逐一实现剩余部分" |

#### 虚假完成型替换

| 当你即将说 | 替换为 |
|---|---|
| "// ... 其他代码保持不变" | [直接输出完整的修改上下文] |
| "// TODO: 实现具体逻辑" | [直接写出具体实现] |
| "// 类似上面的处理" | [写出完整的处理代码] |
| "这里省略了XX部分" | "XX部分的完整实现是" |
| "具体实现请参考" | [直接给出实现] |
| "以此类推" | "逐一列出" |
| "其余部分同理" | "其余部分分别是" |
| "这里只展示核心代码" | "完整代码如下" |

#### 过度抽象型替换

| 当你即将说 | 替换为 |
|---|---|
| "你可以根据需要扩展" | "完整实现已包含扩展点，具体是" |
| "这只是一个基本框架" | "完整的生产级实现如下" |
| "具体细节取决于你的需求" | "基于常见场景，推荐的实现是（如需调整请说明）" |
| "作为起点，你可以" | "完整的实现方案是" |
| "在实际项目中你需要" | "以下实现已按生产标准完成" |

#### 自我降权型替换

| 当你即将说 | 替换为 |
|---|---|
| "我这里做一个简化版" | "完整实现如下" |
| "由于时间/篇幅有限" | [删除此句，直接输出完整内容] |
| "这是一个demo级别的实现" | "以下是生产级实现" |
| "仅供参考" | [删除此句] |
| "这个方案不够完美，但是" | "推荐方案如下" |

#### 逃避复杂度型替换

| 当你即将说 | 替换为 |
|---|---|
| "这个问题比较复杂，建议" | "这个问题可以拆解为以下子问题" |
| "这超出了当前范围" | "这也在需要处理的范围内，解决方案是" |
| "这需要根据具体情况判断" | "常见场景下的推荐方案是" |
| "这个最好手动处理" | "自动化处理方案如下" |

#### Agent 模式专属替换

这些降级短语是 Agent 工作模式中特有的——AI 在调用工具修改代码时的偷懒方式。

| 当你即将说 | 替换为 |
|---|---|
| "其他文件应该不需要改" | "我先搜索确认是否有其他文件需要同步修改" |
| "这里只需要改主文件" | "我先检查所有关联文件，确认完整的修改范围" |
| "改动太多了，简化一下" | "改动涉及多个文件，按 execution_plan 逐步执行" |
| "这个函数的调用方应该不受影响" | "我先搜索这个函数的所有调用方，确认是否需要同步修改" |
| "测试后面再补" | "现在就编写对应的测试" |
| "先改这个文件，其他的后面再处理" | "按 execution_plan 的顺序，逐一完成所有文件的修改" |

#### 思维链内部的省略型降级

这类降级最隐蔽——AI 在思维链本身中就开始偷懒，用缩写和省略跳过实质性思考。

| 当你即将说 | 替换为 |
|---|---|
| "degradation_check: 全部通过" | 逐条写出每个检查项的 YES/NO 和判断理由 |
| "execution_plan: 3步逐一修改" | 列出每一步的具体内容：修改哪个文件的哪个部分 |
| "edge_cases: 同上" 或 "edge_cases: 略" | 完整列出每一个边界条件 |
| "context: 见上文" | 完整写出从侦察中了解到的关键事实 |
| "..." 或 "等等" 出现在任何字段中 | 用完整内容替换省略号 |
| "affected_scope: 相关文件" | 列出每一个具体的文件路径 |
| "上一轮已经分析过/查过/确认过" | 重新写出上一轮发现的具体事实（哪个文件、什么结构、什么风格），不引用"上一轮" |
| "context 沿用上轮分析" | 将上轮分析中的关键事实逐条写出到本轮 context 中 |
| "根据之前的分析" | 具体写出之前分析的结论是什么（哪个方案、什么理由、涉及哪些文件） |

### 1.4 对话式教学示例（Few-Shot Learning）

> **为什么要用对话示例？** 抽象规则告诉模型"应该做什么"，对话示例教会模型"具体怎么做"。自回归模型的本质是模式匹配——上下文中出现过的对话模式，会被高概率复现。对话示例就是在上下文中植入"正确行为的记忆"。

以下示例应写入 rules 文件。模型每次推理都会加载这些示例，形成行为惯性。

---

#### 示例一：Agent 模式——完整的侦察 → 分析 → 执行流程

展示 AI 如何先侦察了解现状、再基于事实分析、最后执行。

```
用户：给 UserService 加上邮箱格式校验，注册时如果邮箱格式不对要返回错误。

助手：

<reconnaissance>
goal: 了解当前注册流程的代码结构、已有的校验方式、错误处理模式
actions:
  - read: service/user.go — 看 Register 方法的当前实现
  - read: model/user.go — 看 User 结构体定义
  - search: "validate" — 看项目中是否已有校验工具
  - check: 项目中错误返回的风格
</reconnaissance>

[调用 read_file 读取 service/user.go]
[调用 read_file 读取 model/user.go]
[调用 search_content 搜索 "validate"]

<analysis>
context:
  - service/user.go 的 Register 方法：接收 email + password，直接插库，没有任何校验
  - 搜索发现项目已有 util/validate.go，包含 ValidatePhone 函数，返回 error
  - 项目统一使用 AppError{Code, Message} 返回错误
needs: 在注册流程中加入邮箱格式校验，沿用项目已有的 validate 工具和 AppError 风格
key_challenges: 校验规则覆盖度、必须沿用 ValidateXxx 命名风格和 AppError 模式
approach: 在已有的 util/validate.go 中新增 ValidateEmail（沿用 ValidatePhone 风格），
  在 Register 开头调用。
  选择理由——可维护性：复用已有工具包；健壮性：正则覆盖主流格式；可扩展性：统一管理
edge_cases: 空字符串、纯空格、缺少@、多个@、域名无点号、超长(>254字符)
affected_scope: util/validate.go（追加）, util/validate_test.go（追加）, service/user.go（修改）
execution_plan:
  - step_1: 在 util/validate.go 追加 ValidateEmail 函数
  - step_2: 在 util/validate_test.go 追加测试
  - step_3: 修改 service/user.go 的 Register 方法
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → NO，沿用了项目现有模式而非自建新包，在已有约定上扩展是最适合的选择
  - 是否省略了边界条件？ → NO，已列出6个具体的边界条件
  - 是否因改动量大而想简化？ → NO，3个文件的改动量完全合理
  - 是否打算跳过某些文件？ → NO，3个文件全部在 execution_plan 中
  - execution_plan 覆盖 affected_scope？ → YES，3步分别对应3个文件
  - context 是否充分？ → YES，已了解校验风格（ValidatePhone）、错误返回模式（AppError）
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → NO，侦察中的所有发现都已纳入方案
  → 全部通过，开始执行
</analysis>

[调用 replace_in_file 在 util/validate.go 追加 ValidateEmail]
[调用 replace_in_file 在 util/validate_test.go 追加测试]
[调用 replace_in_file 修改 service/user.go]
```

**教学要点**：
- `<reconnaissance>` 让 AI 先读代码，发现了项目已有的 validate 工具包和 AppError 风格
- `context` 把侦察发现写出来——直接影响了方案（沿用已有风格 vs 自建新的）
- 不侦察就分析，等于闭眼做手术

---

#### 示例二：Agent 模式——侦察发现改动范围远超预期

```
用户：把 GetUser 接口的返回值从 User 改成 UserDTO。

助手：

<reconnaissance>
goal: 了解 GetUser 的定义、所有调用位置、是否已有 DTO
actions:
  - read: service/user.go — 看 GetUser 当前签名
  - search: "GetUser" — 找出所有调用方
  - search: "UserDTO" — 确认 DTO 是否已定义
</reconnaissance>

[调用 read_file 读取 service/user.go]
[调用 search_content 搜索 "GetUser" — 发现 12 处引用，8 个文件]
[调用 search_content 搜索 "UserDTO" — 未找到]
[调用 read_file 读取 controller/user.go 和 service/order.go 了解调用方式]

<analysis>
context:
  - GetUser 返回 (*User, error)，共 12 处引用分布在 8 个文件中
  - UserDTO 不存在，需要新建
  - 调用方主要使用 user.Email、user.Name、user.Avatar
  - 项目其他 DTO 在 model/dto/ 目录下
needs: 将 GetUser 返回类型从 User 改为 UserDTO，同步更新全部 12 处引用
key_challenges:
  - 引用分布广（8个文件12处），任何遗漏都会编译失败
  - DTO 字段需根据调用方实际使用情况设计，不是简单复制 User struct
approach: 在 model/dto/ 下新建 UserDTO，只暴露调用方实际使用的字段（Email, Name, Avatar），排除内部字段（PasswordHash），service 层提供 ToUserDTO 转换方法。
  选择理由——可维护性：DTO 与领域模型分离，修改 User 内部字段不影响调用方；
  健壮性：转换方法集中在一处，避免各调用方自行挑选字段导致不一致；
  可扩展性：后续需要暴露新字段时只需修改 DTO 定义和转换方法
edge_cases:
  - GetUser 返回 nil 时 DTO 转换要处理 nil pointer
  - 部分调用方可能需要 DTO 中没有的字段
affected_scope: model/dto/user_dto.go(新增), service/user.go, controller/user.go,
  controller/admin.go, service/order.go, service/notification.go, handler/webhook.go, 3个_test.go
execution_plan:
  - step_1: 新建 model/dto/user_dto.go，定义 UserDTO 结构体和 ToUserDTO 转换方法
  - step_2: 修改 service/user.go 的 GetUser 返回类型和转换逻辑
  - step_3: 修改 controller/user.go 的 3 处调用
  - step_4: 修改 controller/admin.go 的 2 处调用
  - step_5: 修改 service/order.go 的 2 处调用
  - step_6: 修改 service/notification.go 的 1 处调用
  - step_7: 修改 handler/webhook.go 的 1 处调用
  - step_8: 修改 3 个 _test.go 文件中的断言和 mock
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → NO，根据调用方实际使用的字段设计 DTO，而非直接复制 User struct
  - 是否省略了已知边界条件？ → NO，已列出 nil pointer 处理和字段不足两个边界条件
  - 是否因改动量大(10个文件)而想简化？ → NO，12处引用全部需要更新，减少任何一处都会编译失败
  - 是否打算跳过某些文件？ → NO，8步覆盖了侦察搜索到的所有文件
  - execution_plan 覆盖所有文件？ → YES，8步覆盖10个文件
  - context 是否充分？ → YES，已搜索全部引用、确认 DTO 不存在、了解目录约定、读取关键调用方确认字段
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → NO，调用方使用的字段信息直接影响了 DTO 设计
  → 全部通过
</analysis>

[按 step_1 到 step_8 逐步执行]
```

**教学要点**：
- 如果没有侦察，AI 可能只改 2-3 个文件。侦察发现了 12 处引用
- `context` 记录了"调用方主要用哪些字段"——决定了 DTO 设计不是简单复制 User

---

#### 示例三：Agent 模式——侦察中发现代码质量问题

```
用户：在支付回调处理中加入退款逻辑。

助手：

<reconnaissance>
goal: 了解支付回调的现有结构和事件处理方式
actions:
  - read: handler/payment_callback.go — 看现有回调处理的代码结构
  - search: "refund" — 确认是否已有退款相关代码
</reconnaissance>

[调用 read_file 读取 handler/payment_callback.go]
[调用 search_content 搜索 "refund" — 未找到已有退款代码]

<analysis>
context:
  - payment_callback.go 已有 400 行，所有支付事件处理逻辑堆在一个函数的 switch-case 中
  - 每个 case 直接内联处理逻辑（支付成功约120行、支付失败约80行、对账约60行）
  - 没有已有的退款相关代码
needs: 在支付回调中增加退款事件处理分支
key_challenges:
  - 现有 handler 已 400 行且全部堆在一个函数中，直接加退款分支会使代码更难维护
  - 退款需要校验原订单状态
  - 退款金额不能超过原支付金额
approach: 先将现有的事件处理拆分为独立函数，再新增退款处理。
  为什么不直接加退款分支？侦察发现 handler 已 400 行，所有逻辑堆在一个函数里，
  继续堆叠会更加失控。拆分后——可维护性：每个事件独立可读；健壮性：独立函数更容易测试；
  可扩展性：后续新增事件类型只需加一个函数
edge_cases:
  - 重复退款回调（幂等处理：用退款单号去重）
  - 部分退款（金额 < 原支付金额）
  - 退款金额超出原支付金额（拒绝并告警）
  - 原订单状态不允许退款
affected_scope: handler/payment_callback.go（重构+新增）, service/refund.go（新增）
execution_plan:
  - step_1: 重构 handler/payment_callback.go，将现有 case 提取为独立函数
  - step_2: 新建 service/refund.go，实现退款校验和处理
  - step_3: 在 handler 的 switch-case 新增 refund 分支
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → NO，选择了先重构再加功能，而非直接在400行函数上继续堆叠
  - 是否省略了已知边界条件？ → NO，已列出4个具体的退款边界条件
  - 是否因改动量大而想简化？ → NO，重构现有代码+新增退款共3步，重构是对后续所有新事件都有益的一次性投入
  - 是否打算跳过某些文件？ → NO，affected_scope 中的2个文件全部在 execution_plan 中
  - execution_plan 是否覆盖 affected_scope？ → YES，step_1和step_3覆盖 payment_callback.go，step_2覆盖 refund.go
  - context 是否充分？ → YES，已了解现有代码结构（400行/switch-case/内联逻辑），搜索确认无已有退款代码
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → NO，侦察发现的"400行大函数"问题没有被跳过，已在 approach 中直接决策
  → 全部通过，开始执行
</analysis>

[调用 replace_in_file 重构 payment_callback.go]
[调用 write_to_file 新建 service/refund.go]
[调用 replace_in_file 在 handler 新增退款分支]
```

**教学要点**：
- 侦察就发现的问题（400行大函数），**直接在 approach 中决策**，不需要 decision_point
- approach 中明确写出"为什么不直接加"——这段推理本身就是锚点，防止 AI 走捷径
- `<decision_point>` 只在执行阶段遇到**无法预见的意外**时使用（见示例四）

---

#### 示例四：Agent 模式——执行阶段遇到真正的意外（decision_point 的正确用法）

```
用户：给订单列表加上按创建时间排序。

助手：

<reconnaissance>
goal: 了解订单列表的现有实现和查询方式
actions:
  - read: service/order.go — 看 ListOrders 方法
  - read: repository/order.go — 看数据库查询
</reconnaissance>

[调用 read_file 读取两个文件]

<analysis>
context:
  - ListOrders 接收分页参数，调用 repository 查询
  - repository 使用 SQL 查询，当前按 id DESC 排序
  - 数据库 orders 表有 created_at 字段
needs: 支持按 created_at 排序，可升序/降序
key_challenges:
  - 排序字段需参数化但必须白名单防 SQL 注入
approach: 增加 sortBy/sortDirection 参数，service 层白名单校验，repository 层拼接 ORDER BY。
  选择理由——可维护性：白名单集中管理在 service 层，新增可排序字段只需改一处；
  健壮性：白名单校验从根源上防止 SQL 注入；可扩展性：白名单可配置化
edge_cases:
  - sortBy 为空时使用默认排序（id DESC）
  - sortBy 不在白名单中时返回 400 错误
  - sortDirection 只允许 ASC/DESC
affected_scope: service/order.go, repository/order.go, controller/order.go
execution_plan:
  - step_1: 修改 repository/order.go，接受 orderByClause 参数替换硬编码排序
  - step_2: 修改 service/order.go，增加 sortBy/sortDirection 参数和白名单校验
  - step_3: 修改 controller/order.go，从 HTTP query 解析排序参数
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → NO，包含白名单防注入机制，比直接拼接用户输入更安全
  - 是否省略了已知边界条件？ → NO，已列出3个具体的边界条件
  - 是否因改动量大而想简化？ → NO，3个文件是最小必要范围
  - 是否打算跳过某些文件？ → NO，3个文件全部在 execution_plan 中
  - execution_plan 是否覆盖 affected_scope？ → YES，3步分别对应3个文件
  - context 是否充分？ → YES，已了解现有查询方式、排序逻辑、数据库字段
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → NO，侦察中的所有发现都已纳入方案
  → 全部通过，开始执行
</analysis>

[调用 replace_in_file 修改 repository/order.go —— 开始执行 step_1]

<decision_point>
issue: 执行 step_1 时发现，repository 的 ORM 链式调用中 .Order() 方法接受的是原始字符串，
  不支持参数化传入。直接将 service 层传来的 orderByClause 传给 .Order() 虽然在白名单校验后是安全的，
  但如果未来有人绕过 service 层直接调用 repository，仍存在注入风险。
  这个问题在侦察阶段没有注意到，因为当时关注的是业务逻辑流向，没有深入检查 ORM 的 Order() 方法的安全特性。
impact: YES — 影响实现方式。原方案假设 repository 只被 service 调用，但 repository 是公开包，
  任何代码都可以直接调用。单层白名单不够安全。
context_update:
  - 原 approach 假设"service 层白名单就足够"——不成立，repository 可被直接调用
  - 原 key_challenges 遗漏了"ORM Order() 不支持参数化"
options:
  - option_a:
      description: repository 层增加第二道白名单校验，双层纵深防御
      approach_evaluation:
        可维护性：白名单定义抽取为共享常量，service 和 repository 复用，改一处即可；
        健壮性：双层校验，即使 service 被绕过 repository 仍能拦截；
        可扩展性：共享常量可后续提取为配置
      edge_cases: 两层白名单不一致时的行为（应共享同一份定义避免）、白名单为空时回退默认排序
      affected_scope_delta: 新增 constants/order.go（定义共享白名单常量）
  - option_b:
      description: 放弃 ORM .Order()，改用原生 SQL ORDER BY
      approach_evaluation:
        可维护性：打破项目统一 ORM 风格，维护者需同时理解两种模式；
        健壮性：列名仍不能参数化，仍需白名单，并不比 option_a 更安全；
        可扩展性：混用 ORM 和原生 SQL 增加技术栈复杂度
      edge_cases: 原生 SQL 列名需表前缀、返回结果需手动映射
      affected_scope_delta: 无新增文件，但 repository 改动量显著增大
recommendation: option_a — 基于三维评估：保持 ORM 统一风格（可维护性）、双层防御更健壮、不增加技术栈复杂度（可扩展性）
execution_plan_update:
  - step_1 变更：repository 层增加白名单校验（引用共享常量）
  - step_2 变更：service 层白名单提取为 constants/order.go 共享常量
  - 新增：write_to_file 创建 constants/order.go
  - step_3 不变
degradation_check:
  - 推荐方案是否"最简单"而非"最合适"？ → NO，双层白名单比单层更复杂但更安全
  - 是否省略了新发现的边界条件？ → NO，已列出白名单不一致和为空两个新条件
  - 是否因为想尽快完成而选择改动量小的方案？ → NO，选 option_a 因为三维评估更优，不是因为改动少
  - 修改后的 execution_plan 是否仍覆盖所有 affected_scope？ → YES，含新增的 constants/order.go
  - 是否有被判断为"无关紧要"而跳过的问题？ → NO
  → 全部通过，按更新后的 plan 继续
</decision_point>

[调用 write_to_file 新建 constants/order.go]
[调用 replace_in_file 完成 step_1（repository 层增加白名单校验）]
[调用 replace_in_file 完成 step_2（service 层引用共享常量）]
[调用 replace_in_file 完成 step_3]
```

**教学要点——这才是 decision_point 的正确用法**：
- 问题在**执行阶段才发现**（ORM 的 Order() 方法的注入特性），analysis 无法预见
- issue 中诚实承认了"侦察阶段没有注意到"
- **每个 option 都有三维评估和边界条件**，和 analysis 的 approach 同等严谨
- **execution_plan_update** 明确写出了后续步骤的变化，包括新增了 constants/order.go
- **自带 degradation_check**，防止在"灵活应变"的旗号下降级
- 和示例三的根本区别：示例三中"400行大函数"在侦察阶段就看到了，不算意外

---

#### 示例五：Agent 模式——重命名任务也使用完整流程

即使是简单重命名，也使用完整的三阶段流程——因为不搜索引用就会漏改文件。

```
用户：把 utils.go 里的 CalcTotal 函数名改成 CalculateTotal。

助手：

<reconnaissance>
goal: 找出 CalcTotal 的所有引用位置，确认完整修改范围
actions:
  - search: "CalcTotal" — 找出项目中所有引用
</reconnaissance>

[调用 search_content 搜索 "CalcTotal" — 发现 4 处引用在 3 个文件中]

<analysis>
context:
  - CalcTotal 定义在 util/utils.go 第 45 行
  - service/order.go 在第 23 行和第 67 行调用了 CalcTotal
  - service/report.go 在第 112 行调用了 CalcTotal
  - 没有其他文件引用此函数
needs: 将 CalcTotal 重命名为 CalculateTotal，同步更新所有3个文件中的4处引用
key_challenges:
  - 必须确保所有引用都被更新，遗漏任何一处都会编译失败
  - 需要确认搜索结果是否完整
approach: 逐文件修改，先改定义处再改调用处，基于 search_content 的搜索结果逐一处理。
  选择理由——可维护性：逐文件修改确保每处都经过确认；
  健壮性：基于搜索结果修改而非猜测，保证完整性；
  可扩展性：此方法适用于任何规模的重命名
edge_cases:
  - 字符串中包含 "CalcTotal"（如日志消息）也需要检查
  - 注释中的引用也应同步更新
affected_scope: util/utils.go（定义处）, service/order.go（2处引用）, service/report.go（1处引用）
execution_plan:
  - step_1: 修改 util/utils.go 第 45 行的函数定义 CalcTotal → CalculateTotal
  - step_2: 修改 service/order.go 第 23 行和第 67 行的两处调用
  - step_3: 修改 service/report.go 第 112 行的调用
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → NO，基于搜索结果逐一修改比全局查找替换更安全
  - 是否省略了已知边界条件？ → NO，已考虑字符串和注释中的引用
  - 是否因改动量大而想简化？ → NO，3个文件4处引用是准确的修改范围
  - 是否打算跳过某些文件？ → NO，搜索结果中的所有3个文件都在 execution_plan 中
  - execution_plan 是否覆盖 affected_scope？ → YES，3步分别对应3个文件中的全部4处引用
  - context 是否充分？ → YES，search_content 已返回完整的引用列表
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → NO，搜索结果中的每一处引用都已纳入 execution_plan
  → 全部通过，开始执行
</analysis>

[调用 replace_in_file 修改 util/utils.go]
[调用 replace_in_file 修改 service/order.go]
[调用 replace_in_file 修改 service/report.go]
```

**教学要点**：
- 即使是简单重命名，侦察（搜索引用）也不能跳过
- execution_plan 每一步写清楚具体改哪个文件的几处引用
- degradation_check 逐条写出，不写"全部通过"

---

#### 示例六：Chat 模式——degradation_check 发现问题并自我纠正

（此示例适用于直接输出代码的 Chat 场景）

```
用户：实现一个文件上传接口，支持断点续传。

助手：

<analysis>
needs: 分片上传 + 断点续传，中断后从中断位置继续
key_challenges:
  - 分片管理和进度持久化
  - 断点续传时校验已上传分片完整性
  - 并发上传多个分片
  - 生产环境多实例部署下的分片存储
approach: 前端切片上传，后端接收分片并记录进度到 Redis，全部完成后合并。
  选择理由——可维护性：分片逻辑独立于业务代码；
  健壮性：Redis 持久化保证断点恢复；
  可扩展性：存储层可替换
edge_cases:
  - 分片上传顺序不固定
  - 相同文件秒传（哈希匹配）
  - 服务器重启后恢复
  - 最后一片大小可能不同
affected_scope: controller/upload.go, service/upload.go, store/progress.go, config/upload.go
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → YES ⚠️
    审视：分片存本地文件系统，但生产环境多实例会导致分片分散
    修正：改为对象存储（S3/MinIO），支持多实例共享
  - 是否省略了边界条件？ → YES ⚠️
    审视：遗漏了文件大小限制和文件类型校验
    修正：补充文件大小上限（可配置，默认2GB）、文件类型白名单（通过文件头判断）
  - 是否打算用注释代替实现？ → NO，所有代码都会完整输出
  - 是否因内容长而想换方案？ → NO，涉及4个文件，分2部分输出
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → NO，发现的两个问题（本地存储不适合多实例、缺少文件校验）都已修正
  → 发现2项 YES，已修正，以修正后的方案进入实现
</analysis>

<implementation>
[反映修正结果的完整代码——包含 StorageType 配置、MaxFileSize、AllowedTypes]
</implementation>
```

---

#### 何时需要写思维链？判断规则

```
判断标准：
  ✅ 需要写完整 <analysis>（含所有字段 + execution_plan）的情况：
    - 新增功能或模块
    - 涉及架构决策
    - 修改涉及 3 个以上文件
    - 存在多种实现方案需要选择
    - 涉及安全、并发、性能等非功能性需求

  📝 可以写精简 <analysis>（needs + affected_scope + execution_plan + degradation_check）的情况：
    - 简单的 bug 修复（原因明确）
    - 变量/函数重命名（但 execution_plan 中必须包含搜索引用的步骤）
    - 格式调整、注释修改
    - 添加/修改单个测试用例

  ⚠️ 无论哪种情况，degradation_check 都不能省略
  ⚠️ Agent 模式下，execution_plan 也不能省略——它是防止"漏改文件"的关键防线
```

---

## 二、第二层：元认知决策协议

### 2.1 为什么需要这一层

第一层的映射表和思维链覆盖了**已知的**降级模式和**侦察阶段发现的**问题。但在**执行阶段**，AI 可能会遇到 analysis 无法预见的新情况——比如：

- 修改代码时发现 ORM 的 API 有意料之外的行为
- 调用工具时返回了预期之外的错误
- 读取到的文件内容和侦察阶段的理解不一致

**关键区分**：
- 侦察阶段就发现的问题 → 在 `<analysis>` 的 `approach` 中直接处理，不需要 decision_point
- 执行阶段才发现的意外 → 用 `<decision_point>` 公开决策

这些场景下，AI 需要灵活应变。但"灵活应变"是降级的最大入口——AI 往往在"灵活"的旗号下静默切换到低质量方案。

### 2.2 元认知决策流程

核心思路：**不固定具体流程，固定"决策方法"**。`<decision_point>` 本质上是一次**执行期的 mini-analysis**——遇到意外时，它要和 `<analysis>` 一样严谨地思考，而不是草草记录"遇到了什么、选哪个"。

```xml
<decision_point>
issue: [明确描述遇到了什么意外问题，以及为什么在侦察/分析阶段没有预见到]
impact: [这个问题是否影响当前方案的可行性？YES/NO + 具体影响范围]
context_update: [这个新发现改变了哪些之前 analysis 中的假设？]
options:
  - option_a:
      description: [方案A完整描述]
      approach_evaluation: [从可维护性、健壮性、可扩展性三个维度评估]
      edge_cases: [此方案引入的新边界条件]
      affected_scope_delta: [相比原 execution_plan 新增或变更了哪些文件]
  - option_b:
      description: [方案B完整描述]
      approach_evaluation: [从可维护性、健壮性、可扩展性三个维度评估]
      edge_cases: [此方案引入的新边界条件]
      affected_scope_delta: [相比原 execution_plan 新增或变更了哪些文件]
recommendation: [推荐选择哪个 + 基于三维评估的理由]
execution_plan_update: [原 execution_plan 中哪些步骤需要修改、是否需要新增步骤]
degradation_check:
  - 推荐方案是否是"最简单"而非"最合适"的？ → [YES/NO + 理由]
  - 推荐方案是否省略了新发现的边界条件？ → [YES/NO + 理由]
  - 是否因为想尽快完成而选择了改动量小的方案？ → [YES/NO + 理由]
  - 修改后的 execution_plan 是否仍覆盖所有 affected_scope？ → [YES/NO + 理由]
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → [YES/NO + 理由]
  → YES 项必须就地修正
</decision_point>
```

**关键规则**：

1. **必须显式声明**：遇到意外问题时，必须输出 `<decision_point>` 块。绝不允许静默切换方案。
2. **每个 option 都需要三维评估**：不是只写描述，而是像 analysis 的 approach 一样从可维护性、健壮性、可扩展性三个维度评估每个方案。
3. **必须更新 execution_plan**：decision_point 做出决策后，原有的 execution_plan 可能需要修改。`execution_plan_update` 字段强制 AI 想清楚后续步骤的变化。
4. **自带 degradation_check**：和 analysis 一样，每次决策都必须经过降级检查。
5. **倾向于继续原方案**：除非 `impact` 为 YES 且三维评估支持更换，否则应继续原方案并附带处理意外问题。

### 2.3 为什么这比"遇到问题先思考"有效

因为它是**结构化的**，而且和 analysis 同等重量。

- "遇到问题先思考" → AI 可能用一句话带过："这里有个小问题，我换个方式处理" → 降级
- 轻量级 decision_point（只有5个字段） → AI 快速填完，没有深入思考 → 降级
- **完整的 decision_point**（含三维评估 + execution_plan_update + degradation_check）→ 每个字段都是推理检查点，很难在填完所有字段后还做出降质的决定

---

## 三、第三层：硬规则兜底

这一层只管**最致命的几个降级行为**。数量少、规则硬、无弹性空间。

### 3.1 绝对禁止清单

```
以下行为在任何情况下都不允许，无论前两层的结果如何：

通用规则（Agent + Chat）：
1. 不允许因为改动量大或内容较长而更换技术方案
2. 不允许在没有 <decision_point> 的情况下变更已确定的方案
3. 不允许跳过 degradation_check

Agent 模式专属：
4. 不允许 execution_plan 中遗漏 affected_scope 中列出的文件
5. 不允许在修改一个函数时，不检查调用该函数的其他位置是否需要同步修改
6. 不允许只修改主文件而跳过关联文件（如改了接口定义但不改实现、改了类型但不改引用）
7. 不允许用"其他文件不需要修改"来跳过检查——必须先读取确认后才能做此判断

Chat 模式专属：
4. 不允许用省略号（...）代替实际代码
5. 不允许用 TODO/FIXME 注释代替实际实现
6. 不允许输出"框架/骨架/脚手架"代替完整实现
```

### 3.2 为什么这些不需要替换词

因为它们的替代路径足够明确，不会导致模型"迷路"：

- 不准省略 → 写出来（唯一路径）
- 不准换方案 → 继续原方案（唯一路径）
- 不准用 TODO → 写实现（唯一路径）

路径唯一时，禁止就够了。路径不唯一时（第一层的场景），才需要替换词来指路。

---

## 四、落地方案：如何确保 AI 每次都执行

单独靠 rules 文本不够稳定。以下四种方案形成组合拳，确保框架在实际使用中被持续遵循。

### 4.1 方案一：结构化输出格式强制

**原理**：格式约束比行为指令更容易被模型遵守。模型训练中见过大量 XML/Markdown 结构，对"格式模式"的遵循度远高于"行为建议"。

**实现**：在 rules 中强制定义输出格式：

```markdown
# 强制输出格式

你的所有回复必须严格遵循以下格式。不允许跳过任何部分。

## 涉及代码修改时

<analysis>
needs: [需求本质目标]
key_challenges: [核心难点]
approach: [方案及选择理由]
edge_cases: [边界条件]
affected_scope: [涉及的文件/模块]
</analysis>

<implementation>
[完整代码实现]
</implementation>

## 遇到意外问题时

在 <analysis> 和 <implementation> 之间插入：

<decision_point>
issue: [问题描述]
impact: [是否影响方案可行性]
options: [备选方案]
recommendation: [推荐选择]
quality_check: [是否降低质量标准]
</decision_point>
```

**为什么有效**：当模型开始输出第一个 token `<`，自回归机制会驱动它补全 `analysis>`，然后自然进入分析阶段。格式本身成了推理的骨架。

### 4.2 方案二：预填充（Prefill）— 最强的落地手段

**原理**：预填充是最硬核的格式强制。不是"建议"模型以某种格式开始，而是**直接替模型写好开头的 token**。模型的第一个自由 token 已经在正确的轨道上了，物理上无法跳过。

```
没有预填充：
  模型的第一个 token 可能是任何东西
  → "好的，我来..." / "这个问题..." / 直接开始写代码
  → 跳过了思维链

有预填充：
  assistant 消息已经以 "<reconnaissance>\ngoal:" 开头
  → 模型的第一个自由 token 必须填写 goal 的内容
  → 它已经在侦察的轨道上了，跑不了
```

**为什么预填充比 rules 指令更有效？**

| 手段 | 模型可以忽略吗 | 原理 |
|---|---|---|
| rules 中写"请先输出 analysis" | 可以 | 只是上下文中的指令，注意力可能衰减 |
| 结构化格式约束 | 较难 | 格式模式遵循度高，但仍可被跳过 |
| **预填充** | **不可能** | **开头 token 已经替模型生成好了，模型只能续写** |

**Agent 模式预填充示例**：

在 API 调用时，assistant 消息的开头预设为：

```
<reconnaissance>
goal:
```

模型接手续写时，它的第一个自由 token 就是填写 goal 的具体内容。侦察阶段不可能被跳过。

侦察完成后，下一轮的 assistant 消息预填充：

```
<analysis>
context:
```

模型被迫先写出侦察到的事实，然后才能继续写 needs、approach 等字段。

**Chat 模式预填充示例**：

```
<analysis>
needs:
```

模型的第一个自由 token 就是描述需求，不可能跳过分析直接写代码。

**部署方式**：

1. **API 直接调用**：在 `messages` 数组中添加一个 `role: "assistant"` 且 `content` 为预填充内容的消息。这是最原生的方式，所有主流 API 都支持。

2. **Claude Code / CC**：在 CLAUDE.md 中写明"你的回复必须以 `<reconnaissance>` 开头"。虽然不是真正的 prefill，但配合格式约束，效果接近。

3. **自定义中间层**：在用户消息发出后、模型生成前，自动注入 assistant prefill。这是最可靠的方式，但需要定制开发。

4. **规则文件模拟**：在 rules 中明确指定开头格式，并配合对话示例建立惯性。效果不如原生 prefill，但无需改架构。

**预填充还能做的事**：

不只是填充标签名——你可以预填充更多内容来锚定推理方向：

```
# 当你明确知道用户任务涉及多文件修改时，预填充：
<reconnaissance>
goal: 了解当前代码结构和所有相关文件的依赖关系
actions:
  - search:

# 当你知道是简单任务时，预填充：
<reconnaissance>
goal: 确认修改范围
actions:
  - search:
```

预填充的内容越具体，模型的推理路径越确定。但要注意不能预填充**错误的假设**——如果预填充了"goal: 了解数据库结构"但实际任务和数据库无关，模型会被带偏。

### 4.3 方案三：末尾注入提醒（对抗注意力衰减）

**原理**：自回归模型对**最近的 token** 有天然的注意力偏好（recency bias）。上下文越长，早期 system prompt 的权重越低。

```
attention_weight(rule) ∝ 1 / distance_to_current_position
```

**实现**：在每轮用户消息末尾自动追加提醒（通过工具配置或 wrapper 实现）：

```
[REMINDER] 先输出 <analysis> 完成分析，再输出 <implementation> 完成实现。
措辞替换规则生效中。遇到意外问题必须输出 <decision_point>。
```

**部署方式**：

- **CodeBuddy / Cursor**：利用 rules 文件中的系统提醒机制，rules 内容会在每次对话中被注入
- **Claude Code**：CLAUDE.md 会在每次交互时被加载
- **自定义包装层**：在 API 调用时，自动在 user message 末尾拼接提醒

**为什么有效**：这句话始终处于离模型当前生成位置最近的地方，注意力权重最高，被"遗忘"的概率最低。

### 4.4 方案四：前几轮示范建立惯性（Few-Shot 效应）

**原理**：自回归模型会**模仿对话历史中的模式**。如果前 N 轮对话中 AI 都按照 `<analysis>` → `<implementation>` 格式输出，后续轮次大概率会延续这个模式。

**实现**：在项目初始化时，设计前几轮交互来建立格式惯性：

```markdown
# 惯性建立对话模板

第一轮：给一个中等复杂度的任务
  → AI 按格式输出 <analysis> + <implementation>
  → 用户确认："分析准确，实现完整"

第二轮：给一个需要方案选择的任务
  → AI 在 <analysis> 中展示方案对比
  → 用户确认："方案选择合理"

第三轮：给一个会遇到意外问题的任务
  → AI 输出 <decision_point>
  → 用户确认："决策过程清晰"
```

**前三轮完成后**，对话历史中已经有三个"标准示范"。模型的 few-shot 学习能力会自动延续这个模式。

**部署方式**：

- 将模板交互保存为项目的"启动对话"
- 或在 rules 中嵌入 few-shot 示例（但会消耗上下文空间）

**最佳实践**：

```markdown
# Few-Shot 示例（写在 rules 中）

以下是你过去的标准输出示例，请始终保持此格式：

<example_interaction>
用户：实现一个带重试机制的 HTTP 客户端

<analysis>
needs: 一个生产级的 HTTP 客户端，核心是可配置的重试机制
key_challenges: 重试策略（指数退避 vs 固定间隔）、幂等性判断、超时控制
approach: 采用装饰器模式封装标准 HTTP 客户端，重试策略通过策略模式注入。
         选择理由——可维护性：策略可独立测试和替换；健壮性：装饰器不侵入原有逻辑；
         可扩展性：新增重试策略只需实现接口
edge_cases: 非幂等请求不重试、响应码判断（5xx重试/4xx不重试）、最大重试次数、熔断
affected_scope: http/client.go, http/retry.go, http/retry_test.go
</analysis>

<implementation>
// ... 完整的代码实现 ...
</implementation>
</example_interaction>
```

### 4.5 方案五：工具层流程引擎（理想状态）

**原理**：前三种方案都是在 prompt 层面"建议"模型遵循流程。工具层流程引擎则是在**架构层面强制执行**——模型物理上无法跳过思维链阶段。

**设计**：

```
用户发出任务
      │
      ▼
┌─────────────────────────────────┐
│  流程引擎：Analysis 阶段        │
│  调用 analyze 工具              │
│  参数强制要求：                  │
│    - needs (required)           │
│    - key_challenges (required)  │
│    - approach (required)        │
│    - edge_cases (required)      │
│    - affected_scope (required)  │
│                                 │
│  ✅ 所有字段填充后才允许进入下一阶段 │
├─────────────────────────────────┤
│  流程引擎：Review 阶段          │
│  将 analysis 结果展示给用户     │
│  用户确认 / 修正 / 补充         │
├─────────────────────────────────┤
│  流程引擎：Implementation 阶段  │
│  analysis 的内容作为上下文注入   │
│  调用 edit/write 工具写代码     │
├─────────────────────────────────┤
│  流程引擎：Verify 阶段          │
│  自动对比 analysis 和 impl      │
│  检查：边界条件是否都处理了？    │
│  检查：方案是否与 analysis 一致？│
└─────────────────────────────────┘
```

**工具 Schema 定义**：

```json
{
  "name": "analyze",
  "description": "在编写任何代码之前，必须先调用此工具完成需求分析",
  "parameters": {
    "type": "object",
    "required": ["needs", "key_challenges", "approach", "edge_cases", "affected_scope"],
    "properties": {
      "needs": {
        "type": "string",
        "description": "需求的本质目标，一句话概括"
      },
      "key_challenges": {
        "type": "array",
        "items": { "type": "string" },
        "description": "实现中的核心难点列表"
      },
      "approach": {
        "type": "object",
        "properties": {
          "chosen": { "type": "string", "description": "选择的方案" },
          "reason": { "type": "string", "description": "从可维护性、健壮性、可扩展性三个维度的评估理由" },
          "alternatives_considered": {
            "type": "array",
            "items": { "type": "string" },
            "description": "考虑过但未选择的替代方案"
          }
        },
        "required": ["chosen", "reason"]
      },
      "edge_cases": {
        "type": "array",
        "items": { "type": "string" },
        "description": "需要处理的边界条件"
      },
      "affected_scope": {
        "type": "array",
        "items": { "type": "string" },
        "description": "涉及的文件路径或模块"
      }
    }
  }
}
```

**为什么这是最强的方案**：工具调用有 JSON Schema 约束，所有 required 字段都必须填充。模型物理上无法跳过任何字段——这是比文本格式更硬的约束。

**局限**：需要工具平台的支持。目前 CodeBuddy / Cursor / Claude Code 都不支持自定义前置工具流程。但这是**正确的产品方向**——这些工具最终都需要一个"推理流程引擎"。

---

## 五、降级触发词完整分类体系

> 以下分类保留供参考和持续补充。第一层的措辞替换映射表已涵盖所有高危项。

### 第一类：捷径导向型（Shortcut-Seeking）

| 触发短语 | 降级行为 | 危害等级 |
|---|---|---|
| "最简单的做法是" | 跳过架构设计，用最少代码完成 | 🔴 高 |
| "最快的方式是" | 跳过错误处理、边界检查 | 🔴 高 |
| "最高效的做法是" | 用"高效"合理化偷工减料 | 🔴 高 |
| "为了简单起见" | 硬编码、省略抽象层 | 🔴 高 |
| "这里我们直接…" | 绕过正常流程，走非标路径 | 🟡 中 |
| "一个快速的解决方案是" | 临时方案当正式方案用 | 🔴 高 |
| "我们可以简单地…" | 降低实现标准 | 🟡 中 |
| "不需要那么复杂" | 自我说服省略必要设计 | 🟡 中 |

### 第二类：放弃正确方案型（Solution-Abandonment）

| 触发短语 | 降级行为 | 危害等级 |
|---|---|---|
| "这个文件太大了，我换一种思路" | 放弃完整实现，给残缺版本 | 🔴 高 |
| "代码太长了，我换个方式" | 把完整方案替换为阉割方案 | 🔴 高 |
| "这样做太复杂了，不如…" | 用"复杂"当借口降低质量 | 🔴 高 |
| "考虑到篇幅限制" | 主动删减关键逻辑 | 🔴 高 |
| "为了节省空间" | 省略完整实现 | 🟡 中 |
| "这里就不展开了" | 跳过关键实现细节 | 🔴 高 |
| "剩下的部分类似" | 偷懒不写完 | 🔴 高 |

### 第三类：虚假完成型（False-Completion）

| 触发短语 | 降级行为 | 危害等级 |
|---|---|---|
| "// ... 其他代码保持不变" | 省略了可能需要修改的代码 | 🔴 高 |
| "// TODO: 实现具体逻辑" | 用注释代替实现 | 🔴 高 |
| "// 类似上面的处理" | 偷懒不写重复但必要的代码 | 🟡 中 |
| "这里省略了 XX 部分" | 主动跳过实现 | 🔴 高 |
| "具体实现请参考…" | 甩锅给不存在的参考 | 🔴 高 |
| "以此类推" | 假设读者能自行补全 | 🟡 中 |
| "其余部分同理" | 掩盖未完成的工作 | 🟡 中 |
| "这里只展示核心代码" | 核心之外的也很重要 | 🟡 中 |

### 第四类：过度抽象型（Over-Abstraction）

| 触发短语 | 降级行为 | 危害等级 |
|---|---|---|
| "你可以根据需要扩展" | 把未完成包装成"灵活性" | 🟡 中 |
| "这只是一个基本框架" | 为残缺实现找台阶 | 🟡 中 |
| "具体细节取决于你的需求" | 回避实现细节 | 🟡 中 |
| "这个可以进一步优化" | 承认不够好但不去做好 | 🟢 低 |
| "作为起点，你可以…" | 把半成品当交付物 | 🟡 中 |
| "在实际项目中你需要…" | 暗示当前版本不可用 | 🟡 中 |

### 第五类：自我降权型（Self-Diminishing）

| 触发短语 | 降级行为 | 危害等级 |
|---|---|---|
| "我这里做一个简化版" | 提前为低质量铺垫 | 🔴 高 |
| "由于时间/篇幅有限" | 自我设限 | 🟡 中 |
| "这是一个 demo 级别的实现" | 主动降标 | 🟡 中 |
| "仅供参考" | 免责式降质 | 🟡 中 |
| "这个方案不够完美，但是…" | 自我说服接受低质量 | 🟡 中 |
| "先实现一个最小可用版本" | 可能合理，但常被滥用 | 🟢 低 |

### 第六类：逃避复杂度型（Complexity-Avoidance）

| 触发短语 | 降级行为 | 危害等级 |
|---|---|---|
| "这个问题比较复杂，建议…" | 把难题推给用户 | 🟡 中 |
| "这超出了当前范围" | 人为缩小问题边界 | 🟡 中 |
| "这需要根据具体情况判断" | 回避给出具体方案 | 🟡 中 |
| "建议使用第三方库来处理" | 有时合理，但常用来逃避 | 🟢 低 |
| "这个最好手动处理" | 把 AI 该做的推给人 | 🟡 中 |

---

## 六、理论基础

### 6.1 自回归模型的"自我催眠"效应

大语言模型采用自回归方式生成文本——每个 token 的生成都以之前所有 token 为条件：

```
P(token_n) = f(token_1, token_2, ..., token_{n-1})
```

当模型生成了"最简单的做法是"这个短语，后续所有 token 的概率分布都会偏向"简单"方向的实现。这不是隐喻，而是模型的数学本质。

### 6.2 可见思维链的自我约束效应

当 AI 在可见思维链中写下"需要处理三个边界条件：A、B、C"，这些 token 进入上下文后，会产生三种约束力：

1. **语义一致性压力**：模型倾向于生成与上下文语义一致的内容。已声明的边界条件会"拉"模型去实现它们。
2. **注意力锚点**：后续生成中，attention 机制会持续关注这些关键 token，影响生成方向。
3. **人类监督点**：可见的分析结果让用户能在代码生成前就发现问题。

### 6.3 训练数据中的统计关联

在训练语料中：
- "最简单的做法是" 后面大概率跟更短、更粗糙的代码
- "综合评估后推荐的方案是" 后面大概率跟更严谨、更完整的实现

模型学到了这种统计关联。措辞替换就是利用这种关联，引导模型走向高质量的输出分布。

### 6.4 注意力权重的距离衰减

```
attention_weight ∝ 1 / distance_to_current_position
```

这是"末尾注入提醒"有效的理论基础。rules 在 system prompt 开头的位置，随着对话变长，注意力权重衰减。在每轮消息末尾注入的提醒，始终保持在离当前生成位置最近的地方。

### 6.5 与现有方法的对比

| 方法 | 作用层面 | 原理 | 成本 | 侵入性 |
|---|---|---|---|---|
| 微调/RLHF | 模型权重 | 改变模型参数 | 极高 | 需要训练 |
| Prompt Engineering | 输入端 | 改变上下文条件 | 低 | 仅改 prompt |
| **Output Phrasing Engineering** | **输出端推理过程** | **改变自回归路径** | **极低** | **仅改 rules** |
| 后处理/Code Review | 输出后 | 事后修正 | 中 | 需要额外流程 |

---

## 七、可直接使用的完整规则文件

规则文件已拆分为独立文件，可直接复制到你的 AI 编码助手配置中（CLAUDE.md / .workbuddy/rules/ / .cursorrules 等）。

### 7.1 Agent 模式版（CC / CodeBuddy / Cursor）

📄 **文件**：[rules-agent-mode.md](./rules-agent-mode.md)

适用于通过工具调用（read_file, replace_in_file, write_to_file 等）修改代码的 Agent 场景。包含：
- 三阶段流程：侦察 → 分析 → 执行
- 侦察阶段的 `<reconnaissance>` 格式
- 含 `context` 和 `execution_plan` 的 `<analysis>` 格式
- Agent 专属措辞替换（如"其他文件不需要改" → "我先搜索确认"）
- Agent 专属硬规则（不允许不读代码就做方案、不允许跳过关联文件）
- 完整的对话式教学示例（3个示例，每个字段都完整展开）
- 强制开头（模拟预填充）

### 7.2 Chat 模式版（ChatGPT / API 对话）

📄 **文件**：[rules-chat-mode.md](./rules-chat-mode.md)

适用于直接在回复中输出完整代码的对话场景。包含：
- `<analysis>` + `<implementation>` 格式
- Chat 专属措辞替换（如"代码太长换思路" → "分部分输出"）
- Chat 专属硬规则（不允许省略号代替代码、不允许 TODO 代替实现）
- 强制开头（模拟预填充）

---

## 八、使用建议

### 8.1 五种落地方案的组合部署

```
推荐组合（按重要性排序）：

1. ✅ 方案一（结构化格式）：写入 rules 文件 —— 最基础，必须有
2. ✅ 方案二（预填充）：assistant prefill 强制开头格式 —— 最强硬的格式保证
3. ✅ 方案三（末尾注入）：利用工具的 rules 注入机制 —— 对抗注意力衰减
4. ✅ 方案四（前几轮示范）：新对话开始时刻意建立格式惯性 —— 利用 few-shot
5. 🔮 方案五（工具层引擎）：等待工具平台支持 —— 未来理想方案
```

### 8.2 渐进式采用

不建议一次性应用所有规则。推荐的优先级：

1. **第一步**：只启用结构化输出格式（`<analysis>` + `<implementation>`）
2. **第二步**：添加措辞替换映射表中的高危项（🔴 标记的）
3. **第三步**：添加元认知决策协议（`<decision_point>`）
4. **第四步**：启用完整映射表

### 8.3 效果验证方法

用同一个编码任务，分别在有规则和无规则的情况下测试，对比：

- 代码完整度（是否有省略）
- 错误处理覆盖率
- 边界条件处理数量
- 架构合理性
- 方案是否被中途替换
- 思维链中的分析是否与最终代码一致

### 8.4 持续迭代

这份清单应该持续更新。在日常使用中，一旦发现 AI 开始跑偏：

1. 检查它是否使用了某个降级触发短语
2. 如果是已知短语 → 检查替换规则是否被绕过
3. 如果是新短语 → 添加到映射表中
4. 如果不是措辞问题 → 可能需要补充新的硬规则

---

## 九、贡献与反馈

这是一个开放的框架，欢迎补充：
- 新发现的降级触发短语及其替换词
- 不同 AI 工具/模型上的效果验证数据
- 结构化思维链格式的改进建议
- 工具层流程引擎的实现方案

---

*Output Phrasing Engineering v1.0 — 2026.04.14*
*"没有无关紧要的问题，只有你不想处理的问题。"*
