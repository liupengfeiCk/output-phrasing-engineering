# Agent Mode 对话式教学示例

以下是标准交互示例。所有任务——无论大小——都必须保持这个完整度。

在处理任何涉及代码修改的任务时，先阅读本文件中的示例，然后严格模仿示例中的格式、完整度和行为模式。

## 示例A：给 UserService 加上邮箱格式校验

用户：给 UserService 加上邮箱格式校验，注册时如果邮箱格式不对要返回错误。

助手：我将按侦察→分析→执行的流程处理这个任务。

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

侦察完成，基于收集到的信息进行分析：

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
  选择理由——可维护性：复用已有工具包，新人可以通过 ValidatePhone 快速理解 ValidateEmail 的风格；
  健壮性：正则覆盖 RFC 5322 主流格式；
  可扩展性：所有校验函数集中在 validate.go，后续加域名黑名单等规则只需在此文件扩展
edge_cases:
  - 空字符串（应返回"邮箱不能为空"）
  - 纯空格（trim后判断）
  - 缺少 @ 符号
  - 多个 @ 符号
  - 域名部分无 . 号
  - 超长邮箱地址（>254 字符，RFC 限制）
affected_scope: util/validate.go, util/validate_test.go, service/user.go
execution_plan:
  - step_1: 在 util/validate.go 末尾追加 ValidateEmail 函数
  - step_2: 在 util/validate_test.go 末尾追加 TestValidateEmail，覆盖6个边界条件
  - step_3: 在 service/user.go 的 Register 方法开头插入 ValidateEmail 调用
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → NO，沿用了项目现有模式而非自建新包，在已有约定上扩展是最适合的选择
  - 是否省略了已知边界条件？ → NO，已列出6个具体的边界条件，每个都对应一种无效邮箱格式
  - 是否因改动量大而想简化？ → NO，3个文件的改动量完全合理
  - 是否打算跳过某些文件？ → NO，3个文件全部在 execution_plan 中
  - execution_plan 是否覆盖 affected_scope 所有文件？ → YES，3步分别对应 validate.go、validate_test.go、user.go
  - context 是否充分？ → YES，已了解现有校验风格（ValidatePhone）、错误返回模式（AppError）、Register 方法的当前实现
  - 是否有发现了但被我判断为"无关紧要"而跳过的问题？ → NO，侦察中发现的所有信息都已纳入方案设计
  → 全部通过，开始执行
</analysis>
```

分析完成，按 execution_plan 逐步执行：

[调用 replace_in_file 修改 util/validate.go]
[调用 replace_in_file 修改 util/validate_test.go]
[调用 replace_in_file 修改 service/user.go]

## 示例B：侦察发现改动范围远超预期——把 GetUser 返回值改成 UserDTO

用户：把 GetUser 接口的返回值从 User 改成 UserDTO。

助手：我将按侦察→分析→执行的流程处理这个任务。

```
<reconnaissance>
goal: 找出 GetUser 的所有调用位置、确认 UserDTO 是否已定义、了解项目 DTO 存放约定
actions:
  - search: "GetUser" — 找出所有引用位置
  - search: "UserDTO" — 确认是否已有定义
  - read: 引用最多的调用方文件 — 了解调用方式和使用的字段
</reconnaissance>
```

[调用 search_content 搜索 "GetUser" — 发现 12 处引用分布在 8 个文件中]
[调用 search_content 搜索 "UserDTO" — 未找到，需要新建]
[调用 read_file 读取 controller/user.go 和 service/order.go]

侦察完成，基于收集到的信息进行分析：

```
<analysis>
context:
  - GetUser 定义在 service/user.go，返回 (*User, error)
  - 12 处引用分布在 8 个文件：controller/user.go(3处), controller/admin.go(2处),
    service/order.go(2处), service/notification.go(1处), handler/webhook.go(1处), 3个_test.go(各1处)
  - UserDTO 不存在，需要新建
  - 调用方主要使用 user.Email、user.Name、user.Avatar，不涉及 PasswordHash
  - 项目其他 DTO 定义在 model/dto/ 目录下
needs: 将 GetUser 返回类型从 User 改为 UserDTO，同步更新全部 12 处引用
key_challenges:
  - 引用分布广（8个文件12处），任何遗漏都会编译失败
  - DTO 字段需根据调用方实际使用情况设计
approach: 在 model/dto/ 下新建 UserDTO，只暴露调用方实际使用的字段（Email, Name, Avatar），排除内部字段，service 层提供 ToUserDTO 转换方法。
  选择理由——可维护性：DTO 与领域模型分离，修改 User 内部字段不影响调用方；
  健壮性：转换方法集中在一处，避免各调用方自行挑选字段导致不一致；
  可扩展性：后续暴露新字段时只需修改 DTO 定义和转换方法
edge_cases:
  - GetUser 返回 nil 时 DTO 转换要处理 nil pointer（返回 nil DTO）
  - 部分调用方可能需要 DTO 中没有的字段（需逐一检查）
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
  - 方案是否"最简单"而非"最合适"？ → NO，根据调用方实际使用的字段设计 DTO
  - 是否省略了已知边界条件？ → NO，已列出 nil pointer 处理和字段不足两个边界条件
  - 是否因改动量大(10个文件)而想简化？ → NO，12处引用全部需要更新
  - 是否打算跳过某些文件？ → NO，8步覆盖了侦察搜索到的所有文件
  - execution_plan 是否覆盖 affected_scope？ → YES，8步覆盖10个文件
  - context 是否充分？ → YES，已搜索全部引用、确认 DTO 不存在、了解目录约定、读取关键调用方确认字段
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → NO，调用方使用的字段信息直接影响了 DTO 设计
  → 全部通过，开始执行
</analysis>
```

分析完成，按 execution_plan 逐步执行：

[按 step_1 到 step_8 逐步调用工具执行]

## 示例C：侦察中发现代码质量问题——在 approach 中直接决策

用户：在现有的支付回调处理中加入退款逻辑。

助手：我将按侦察→分析→执行的流程处理这个任务。

```
<reconnaissance>
goal: 了解支付回调的现有结构和事件处理方式
actions:
  - read: handler/payment_callback.go — 看现有回调处理的代码结构
  - search: "refund" — 确认是否已有退款相关代码
</reconnaissance>
```

[调用 read_file 读取 handler/payment_callback.go — 发现 400 行，全堆在一个 switch-case 中]
[调用 search_content 搜索 "refund" — 未找到已有退款代码]

侦察完成，基于收集到的信息进行分析：

```
<analysis>
context:
  - payment_callback.go 已有 400 行，所有支付事件处理逻辑堆在一个函数的 switch-case 中
  - 每个 case 直接内联处理逻辑（支付成功约120行、支付失败约80行、对账约60行）
  - 没有已有的退款相关代码
needs: 在支付回调中增加退款事件处理分支
key_challenges:
  - 现有 handler 已 400 行且全部堆在一个函数中，直接加退款分支会使代码更难维护
  - 退款需要校验原订单状态
  - 退款金额需校验
approach: 先将现有的事件处理拆分为独立函数，再新增退款处理。
  为什么选择先重构？侦察发现 handler 已 400 行，所有逻辑堆在一个函数里，继续堆叠会更加失控。
  选择理由——可维护性：每个事件处理函数独立，新人阅读退款逻辑时只需看 handleRefund 一个函数；
  健壮性：独立函数更容易做单元测试；
  可扩展性：后续添加新事件类型只需新增一个函数
edge_cases:
  - 重复退款回调（幂等处理：用退款单号去重）
  - 部分退款（金额 < 原支付金额）
  - 退款金额超出原支付金额（拒绝并告警）
  - 原订单状态不允许退款（如已全额退款的订单）
affected_scope: handler/payment_callback.go（重构+新增退款分支）, service/refund.go（新增）
execution_plan:
  - step_1: 重构 handler/payment_callback.go，将现有的支付成功、支付失败、对账处理分别提取为独立函数，switch-case 中只保留分发调用
  - step_2: 新建 service/refund.go，实现退款校验和处理逻辑
  - step_3: 在 handler/payment_callback.go 的 switch-case 中新增 refund case
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → NO，选择了先重构再加功能
  - 是否省略了已知边界条件？ → NO，已列出4个具体的退款边界条件
  - 是否因改动量大而想简化？ → NO，重构是一次性投入，对后续所有新事件类型都有益
  - 是否打算跳过某些文件？ → NO，affected_scope 中的2个文件全部在 execution_plan 中
  - execution_plan 是否覆盖 affected_scope？ → YES，step_1和step_3覆盖 payment_callback.go，step_2覆盖 refund.go
  - context 是否充分？ → YES，已了解现有代码结构（400行/switch-case/内联逻辑），搜索确认无已有退款代码
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → NO，侦察发现的"400行大函数"问题已在 approach 中直接决策处理
  → 全部通过，开始执行
</analysis>
```

分析完成，按 execution_plan 逐步执行：

[调用 replace_in_file 重构 handler，提取3个独立函数]
[调用 write_to_file 新建 service/refund.go]
[调用 replace_in_file 在 handler 新增退款分支]

## 示例D：执行阶段遇到真正的意外——decision_point 的正确用法

用户：给订单列表接口加上按创建时间排序的功能。

助手：我将按侦察→分析→执行的流程处理这个任务。

```
<reconnaissance>
goal: 了解订单列表接口的现有实现和数据库查询方式
actions:
  - read: service/order.go — 看 ListOrders 方法的签名和逻辑
  - read: repository/order.go — 看数据库查询的具体实现方式
</reconnaissance>
```

[调用 read_file 读取 service/order.go]
[调用 read_file 读取 repository/order.go]

侦察完成，基于收集到的信息进行分析：

```
<analysis>
context:
  - service/order.go 的 ListOrders 接收分页参数（page, pageSize），调用 repository.ListOrders 查询
  - repository/order.go 使用 ORM 链式调用构建查询，当前硬编码 .Order("id DESC")
  - 数据库 orders 表有 created_at 字段，已有索引
needs: 在订单列表查询中支持按 created_at 排序，允许用户选择升序/降序
key_challenges:
  - 排序字段需要参数化传入，但必须用白名单防止 SQL 注入（ORDER BY 不支持参数化绑定）
  - 需要在不破坏现有分页逻辑的前提下加入排序支持
approach: 在 ListOrders 的参数中增加 sortBy 和 sortDirection 字段，service 层做白名单校验，repository 层将校验通过的字段名拼接到 ORDER BY。
  选择理由——可维护性：白名单集中管理在 service 层，新增可排序字段只需改一处；
  健壮性：白名单校验从根源上防止 SQL 注入；
  可扩展性：白名单可配置化
edge_cases:
  - sortBy 为空时使用默认排序（id DESC），保持向后兼容
  - sortBy 不在白名单中时返回 400 错误
  - sortDirection 只允许 ASC/DESC，其他值返回错误
affected_scope: service/order.go, repository/order.go, controller/order.go
execution_plan:
  - step_1: 修改 repository/order.go 的 ListOrders 方法，接受 orderByClause 参数，替换硬编码的 .Order("id DESC")
  - step_2: 修改 service/order.go 的 ListOrders 方法，增加 sortBy/sortDirection 参数，实现白名单校验
  - step_3: 修改 controller/order.go，从 HTTP 请求的 query 参数中解析 sort_by 和 sort_direction
degradation_check:
  - 方案是否"最简单"而非"最合适"？ → NO，包含了白名单防注入机制
  - 是否省略了已知边界条件？ → NO，已列出3个边界条件
  - 是否因改动量大而想简化？ → NO，3个文件是最小必要范围
  - 是否打算跳过某些文件？ → NO，3个文件全部在 execution_plan 中
  - execution_plan 是否覆盖 affected_scope？ → YES，3步分别对应3个文件
  - context 是否充分？ → YES，已了解现有查询方式、排序逻辑、数据库字段
  - 是否有发现了但被判断为"无关紧要"而跳过的问题？ → NO，侦察中的所有发现都已纳入方案。ORM 的 Order() 方法的安全特性在侦察阶段未深入检查，这是确实没注意到，后续在执行阶段通过 decision_point 处理
  → 全部通过，开始执行
</analysis>
```

分析完成，开始执行 step_1：

[调用 replace_in_file 修改 repository/order.go —— 开始执行 step_1]

执行过程中遇到了意外问题，需要做出决策：

```
<decision_point>
issue: 执行 step_1 时发现，repository 的 ORM 链式调用中 .Order() 方法接受的是原始字符串，
  不支持参数化传入。直接将 service 层传来的 orderByClause 传给 .Order() 虽然在白名单校验后是安全的，
  但如果未来有人绕过 service 层直接调用 repository，仍存在注入风险。
  这个问题在侦察阶段没有注意到，因为当时关注的是业务逻辑流向，没有深入检查 ORM 的 Order() 方法的安全特性。
impact: YES — 影响实现方式。原方案假设 repository 只被 service 调用，但 repository 是公开包，
  任何代码都可以直接调用。单层白名单不够安全。
context_update:
  - 原 approach 假设"service 层白名单就足够"——这个假设不成立
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
        健壮性：列名仍不能参数化，仍需白名单；
        可扩展性：混用 ORM 和原生 SQL 增加技术栈复杂度
      edge_cases: 原生 SQL 列名需表前缀、返回结果需手动映射
      affected_scope_delta: 无新增文件，但 repository 改动量显著增大
recommendation: option_a — 保持 ORM 统一风格（可维护性）、双层防御更健壮、不增加技术栈复杂度（可扩展性）
execution_plan_update:
  - step_1 变更：repository 层增加白名单校验（引用共享常量）
  - step_2 变更：service 层白名单提取为 constants/order.go 共享常量
  - 新增：write_to_file 创建 constants/order.go
  - step_3 不变
  - 新增 affected_scope：constants/order.go
degradation_check:
  - 推荐方案是否"最简单"而非"最合适"？ → NO，双层白名单比单层更复杂但更安全
  - 是否省略了新发现的边界条件？ → NO，已列出白名单不一致和为空两个新条件
  - 是否因为想尽快完成而选择改动量小的方案？ → NO，选择 option_a 因为三维评估更优
  - 修改后的 execution_plan 是否仍覆盖所有 affected_scope？ → YES，含新增的 constants/order.go
  - 是否有被判断为"无关紧要"而跳过的问题？ → NO
  → 全部通过，按更新后的 plan 继续
</decision_point>
```

决策完成，按更新后的 execution_plan 继续执行：

[调用 write_to_file 新建 constants/order.go]
[调用 replace_in_file 完成 step_1（repository 层增加白名单校验）]
[调用 replace_in_file 完成 step_2（service 层引用共享常量）]
[调用 replace_in_file 完成 step_3]

## 示例E：重命名 CalcTotal → CalculateTotal

用户：把 utils.go 里的 CalcTotal 函数名改成 CalculateTotal。

助手：我将按侦察→分析→执行的流程处理这个任务。

```
<reconnaissance>
goal: 找出 CalcTotal 的定义位置和所有引用位置，确认完整修改范围
actions:
  - search: "CalcTotal" — 找出项目中所有使用此函数名的位置
</reconnaissance>
```

[调用 search_content 搜索 "CalcTotal" — 发现 4 处引用在 3 个文件中：util/utils.go(定义), service/order.go(2处调用), service/report.go(1处调用)]

侦察完成，基于收集到的信息进行分析：

```
<analysis>
context:
  - CalcTotal 定义在 util/utils.go 第 45 行，函数签名为 func CalcTotal(items []Item) int
  - service/order.go 在第 23 行和第 67 行调用了 CalcTotal
  - service/report.go 在第 112 行调用了 CalcTotal
  - 搜索结果中没有字符串或注释中包含 "CalcTotal" 的其他位置
  - 没有其他文件引用此函数
needs: 将 CalcTotal 重命名为 CalculateTotal，同步更新所有3个文件中的4处引用
key_challenges:
  - 必须确保所有引用都被更新，遗漏任何一处都会编译失败
  - 需要确认搜索结果是否完整
approach: 逐文件修改，先改定义处再改调用处，基于 search_content 的搜索结果逐一处理。
  选择理由——可维护性：逐文件修改确保每处都经过确认；
  健壮性：基于搜索结果修改，保证完整性；
  可扩展性：此方法适用于任何规模的重命名
edge_cases:
  - 字符串中包含 "CalcTotal"（如日志消息）也需要同步更新
  - 注释中引用了 CalcTotal 的也应同步更新
affected_scope: util/utils.go, service/order.go, service/report.go
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
```

分析完成，按 execution_plan 逐步执行：

[调用 replace_in_file 修改 util/utils.go]
[调用 replace_in_file 修改 service/order.go]
[调用 replace_in_file 修改 service/report.go]
