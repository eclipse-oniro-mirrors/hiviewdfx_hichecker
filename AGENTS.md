# AGENTS.md

本文件为 `hichecker` 仓库定义 Agent 工作规则。

## 1. 范围与优先级

- 本文件适用于 `OpenHarmony/base/hiviewdfx/hichecker`。
- 用户直接指令优先级高于本文件。
- 核心原则：**代码优先、证据优先、不臆造**。

## 2. 项目定位

HiChecker 是面向应用开发阶段的检测工具，用于检测代码运行中易忽略的问题（线程耗时调用、进程中元能力资源泄露、ArkUI 性能等），通过记录流水日志或触发进程 crash 两种告警方式帮助开发者定位并修改问题，提升应用体验。

HiChecker 是**纯库组件**（无 CLI 可执行程序、无独立 SystemAbility 服务进程），进程内被调用：

- **libhichecker.so**：C++ 内部接口（`inner_kits`），供子系统（ImagePacker、ability_runtime DFX FeatureAbility 等）在耗时点或泄露点调用。
- **@ohos.hichecker**：应用层 JS/ArkTS 规则 API，经 NAPI（`hichecker` 目标）与 ANI（`hichecker_ani` 目标）两套绑定，暴露同一 7 方法 + 6 规则常量。
- **jsleakwatcher**：独立的 JS 对象泄露检测模块（基于 `FinalizationRegistry` + 堆快照导出），与规则位掩码 API 无关，单独成库。
- 规则采用 `uint64_t` 位掩码，状态分**进程级**（`processRules_`）与**线程级**（`thread_local threadLocalRules_`）。

### 2.1 按任务类型定位代码

| 任务类型 | 首选目录 | 关键文件 |
|---|---|---|
| 修改规则常量 / HiChecker 静态 API | `interfaces/native/innerkits/include/` | `hichecker.h`（Rule 命名空间、HiChecker 类） |
| 修改规则引擎实现（AddRule/Notify*/HandleCaution/参数初始化） | `frameworks/native/` | `hichecker.cpp` |
| 修改告警载体 Caution | `interfaces/native/innerkits/include/` + `frameworks/native/` | `caution.h`、`caution.cpp` |
| 修改 C-ABI 包装（供 C/init 调用） | `interfaces/native/innerkits/include/` + `frameworks/native/` | `hichecker_wrapper.h`、`hichecker_wrapper.cpp` |
| 修改 NAPI（JS）绑定 | `interfaces/js/kits/napi/` | `include/napi_hichecker.h`、`src/napi_hichecker.cpp` |
| 修改 ANI（ArkTS）绑定 | `interfaces/ets/ani/hichecker/` | `src/ani_hichecker.cpp`、`ets/@ohos.hichecker.ets` |
| 修改 jsleakwatcher 泄露检测 | `interfaces/js/kits/napi/js_leak_watcher/` | `js_leak_watcher.ts`、`js_leak_watcher_napi.cpp` |
| 修改系统参数默认值 / DAC | `interfaces/native/innerkits/` | `hichecker.para`、`hichecker.para.dac` |
| 新增/修改单元测试 | `test/unittest/common/native/` | `hichecker_native_test.cpp` |
| 新增/修改 Fuzz 测试 | `test/fuzztest/hichecker_fuzzer/` | `hichecker_fuzzer.cpp` |
| Python 集成测试（不进 GN 构建） | `test/scripts/testModule/` | `test_hichecker.py`、`test_jsleakwatcher_auto.py` |
| 构建目标 / 组件依赖 | 根目录 | `bundle.json`、`hichecker.gni`、各 `BUILD.gn` |

### 2.2 嵌套指引

本仓库无目录级别的嵌套 Agent 指引。架构与依赖说明见 `README_zh.md` 与 `figures/HiChecker架构图.png`。

## 3. 构建与验证

构建命令从 OpenHarmony 源码根目录执行，不在本子目录执行。

### 3.1 构建

```bash
# 构建核心 inner_kits 库（C++ 接口）
./build.sh --product-name rk3568 --build-target //base/hiviewdfx/hichecker/interfaces/native/innerkits:libhichecker

# 构建 NAPI JS 模块
./build.sh --product-name rk3568 --build-target //base/hiviewdfx/hichecker/interfaces/js/kits/napi:hichecker

# 构建 ANI（ArkTS）模块
./build.sh --product-name rk3568 --build-target //base/hiviewdfx/hichecker/interfaces/ets/ani:ani_hichecker_package

# 构建 jsleakwatcher（TS + 原生）
./build.sh --product-name rk3568 --build-target //base/hiviewdfx/hichecker/interfaces/js/kits/napi/js_leak_watcher:jsleakwatcher
```

> 注：NAPI / ANI 目标受全局 `support_jsapi` 开关保护，未启用时构建会被跳过。`hichecker_support_asan`（`hichecker.gni:15`）为部件级 feature。

### 3.2 单元测试 / Fuzz 构建

```bash
# 单元测试
./build.sh --product-name rk3568 --build-target //base/hiviewdfx/hichecker/test:unittest

# Fuzz 测试
./build.sh --product-name rk3568 --build-target //base/hiviewdfx/hichecker/test:hichecker_fuzztest

# 单个测试二进制（ninja）
prebuilts/build-tools/linux-x86/bin/ninja -C out/rk3568 HiCheckerNativeTest
```

### 3.3 运行测试

```bash
# 单元测试（module_out_path = hichecker/hichecker）
./out/rk3568/tests/hichecker/hichecker/HiCheckerNativeTest

# 单个 gtest 用例
./out/rk3568/tests/hichecker/hichecker/HiCheckerNativeTest --gtest_filter=HiCheckerNativeTest.AddRuleTest001

# Fuzz 二进制
./out/rk3568/tests/hichecker/hichecker/HicheckerFuzzTest
```

### 3.4 完成标准

任务被认为完成，当且仅当：

1. **代码改动可编译** — 上述构建命令通过，无新增告警。
2. **相关测试通过** — `HiCheckerNativeTest` 通过（必要时加 `--gtest_filter`）；规则引擎改动须补 Fuzz 用例。
3. **板侧验证（如适用）** — 涉及 crash 规则、参数初始化、JS 绑定的改动需提供板侧证据（hilog 抓 `HICHECKER`/`HiChecker` tag、应用 crash 栈）。
4. **文档更新（如适用）** — 公共 API / 规则常量修改需同步 `README_zh.md`、`README.md` 及头文件注释。
5. **约束被遵守** — 第 7、8 节约束被尊重，尤其公共 API / ABI / JS 错误码 / 参数边界。

### 3.5 无法运行验证时

明确说明无法运行的原因，列出推荐验证步骤供人工执行，标记需要人工验证的部分。

### 3.6 完成报告格式

报告应包含：改动摘要（文件列表、改动点）、验证结果（构建/测试输出）、风险评估（API/ABI 兼容性、性能、crash 风险）、未完成事项。

## 4. 核心工作原则（必须遵守）

### 4.1 只以真实代码为准

- 任何代码级结论，必须先用搜索 / 读取工具读真实代码后再下结论。
- 给出结论时引用文件路径与行号（如 `hichecker.h:27`、`hichecker.cpp:198`）。
- 若代码缺失，明确说明「此代码在 hichecker 中未找到」。
- 不得将臆测的实现当作事实写出。

### 4.2 臆测管理

- 任何未经验证的陈述必须标注「推测」。
- 能验证的先验证；仅在确无验证途径时保留推测。

### 4.3 代码优先验证

- 用户提供的说法可能错误；先以源码核实再采信。
- 出现分歧时，以实现证据为准。

## 5. 项目地图

```
/base/hiviewdfx/hichecker
├── frameworks/native/               # 引擎实现
│   ├── hichecker.cpp                # HiChecker 静态方法实现（规则增删/告警/参数初始化）
│   ├── hichecker_wrapper.cpp        # C-ABI 包装实现
│   └── caution.cpp                  # Caution 访问器实现
├── interfaces/
│   ├── native/innerkits/
│   │   ├── include/                 # 头文件：hichecker.h / caution.h / hichecker_wrapper.h
│   │   ├── hichecker.para           # 系统参数默认值：hiviewdfx.hichecker=0
│   │   └── hichecker.para.dac       # 参数 DAC：root:shell:0775
│   ├── js/kits/napi/
│   │   ├── include/、src/           # @ohos.hichecker NAPI 绑定
│   │   └── js_leak_watcher/         # 独立 JS 泄露检测模块（TS + 原生）
│   └── ets/ani/hichecker/
│       ├── include/、src/           # ArkTS ANI 绑定
│       └── ets/@ohos.hichecker.ets  # ArkTS 声明 + 规则常量
├── test/
│   ├── unittest/common/native/      # hichecker_native_test.cpp（438 行）
│   ├── fuzztest/hichecker_fuzzer/   # HicheckerFuzzTest
│   └── scripts/testModule/          # pytest 集成测试（未接入 GN 构建）
├── figures/                         # 架构图
├── bundle.json                       # 组件元数据（@ohos/hichecker, subsystem=hiviewdfx）
├── hichecker.gni                     # feature 开关 hichecker_support_asan
└── README.md / README_zh.md
```

### 5.1 经常改动的路径

- `frameworks/native/hichecker.cpp` — 规则引擎行为变更（最高频）。
- `interfaces/native/innerkits/include/hichecker.h` — 规则常量与 API 签名（高影响，ABI 相关）。
- `interfaces/js/kits/napi/src/napi_hichecker.cpp` 与 `interfaces/ets/ani/hichecker/src/ani_hichecker.cpp` — JS/ArkTS 绑定（须保持两套一致）。
- `interfaces/native/innerkits/hichecker.para` — 参数化规则默认值。

### 5.2 任务 → 路径速查

| 任务 | 起点 |
|---|---|
| 新增一条检测规则 | `hichecker.h` Rule 命名空间加位常量 + 更新 `ALL_RULES`/聚合掩码 + `hichecker.cpp` 路由到 thread/process + NAPI/ANI 枚举导出 + `.ets` 常量 |
| 修改耗时/泄露告警流程 | `frameworks/native/hichecker.cpp`（`NotifySlowProcess`/`NotifySlowEvent`/`NotifyAbilityConnectionLeak`/`HandleCaution`） |
| 修改告警输出（日志/crash） | `hichecker.cpp` `PrintLog`/`TriggerCrash` |
| 修改堆栈抓取 | `hichecker.cpp` `DumpStackTrace`（依赖 `faultloggerd:libbacktrace_local` 的 `GetBacktrace`） |
| 修改系统参数化规则 | `hichecker.cpp` `InitHicheckerParam` + `hichecker.para`/`.para.dac` |
| 修改 JS 错误码（401） | `napi_hichecker.cpp` `ThrowError` + `ani_hichecker.cpp` `ThrowError`（两套须同步） |
| 修改 jsleakwatcher 错误码（108010xx） | `js_leak_watcher.ts` |

## 6. 知识索引

稳定背景知识见 `README_zh.md` / `README.md` 与 `figures/HiChecker架构图.png`。本仓库无 `docs/kb/` 知识库，改动前请阅读：

- `README_zh.md` — 组件简介、架构与依赖（ImagePacker、ability_runtime DFX FeatureAbility）、接口说明表、支持特性（暂不支持 JS 回栈）。
- `bundle.json` — 组件依赖、inner_kits 头文件清单、构建目标。
- `hichecker.gni` — feature 开关。

### 6.1 开始编辑前

按以下顺序确认：

1. 确认任务类别（见 2.1 表）。
2. 确认已阅读相关 README / 头文件（或明确声明「无相关文档」）。
3. 根据「项目约束」确认不违反任何约束。
4. 声明：「我将修改 X，已阅读 Y，遵循 Z 约束」。

## 7. 编码约定

### 7.1 命名空间与单例

- 全部代码位于 `namespace OHOS::HiviewDFX`；规则常量位于嵌套 `namespace Rule`（`hichecker.h:26`）。
- `HiChecker` 为**静态工具类**：构造删除（`hichecker.h:57-59`），无 `GetInstance`，所有 API 为静态方法。
- 静态状态成员（`hichecker.h:81-84`，定义于 `hichecker.cpp:44-47`）：`std::mutex mutexLock_`、`volatile bool checkMode_`、`volatile uint64_t processRules_`、`thread_local uint64_t threadLocalRules_`。

### 7.2 日志（直接用 HiLog 宏）

- 本仓库**不定义自定义日志宏**，直接使用标准 HiLog C/C++ 宏：包含 `"hilog/log_c.h"`、`"hilog/log_cpp.h"`。
- 每个文件内 `#undef` 后重定义 `LOG_DOMAIN`/`LOG_TAG`：
  - `hichecker.cpp`：`0xD002D0B` / `"HICHECKER"`（`hichecker.cpp:37-40`）。
  - `napi_hichecker.cpp`：`0xD002D0B` / `"HiChecker_NAPI"`。
  - `ani_hichecker.cpp`：`0xD002D0B` / `"HiChecker_ANI"`。
- 用法示例（`hichecker.cpp:186`）：
  ```cpp
  HILOG_INFO(LOG_CORE, "HiChecker caution with RULE_CAUTION_PRINT_LOG.\nCautionMsg:%{public}s\nStackTrace:\n%{public}s",
      cautionDetail.caution_.GetCautionMsg().c_str(), cautionDetail.caution_.GetStackTrace().c_str());
  ```
- 敏感信息不得明文打印；`%{public}s` 仅对非敏感字段。

### 7.3 规则位掩码（必须遵守）

- 规则为 `uint64_t` 位常量，定义于 `hichecker.h:27-32`：

  | 常量 | 值 | 作用域 |
  |---|---|---|
  | `RULE_CAUTION_PRINT_LOG` | `1ULL<<63` | 告警行为 |
  | `RULE_CAUTION_TRIGGER_CRASH` | `1ULL<<62` | 告警行为 |
  | `RULE_THREAD_CHECK_SLOW_PROCESS` | `1ULL<<0` | 线程级 |
  | `RULE_CHECK_SLOW_EVENT` | `1ULL<<32` | 进程级 |
  | `RULE_CHECK_ABILITY_CONNECTION_LEAK` | `1ULL<<33` | 进程级 |
  | `RULE_CHECK_ARKUI_PERFORMANCE` | `1ULL<<34` | 进程级 |

- 聚合掩码：`ALL_RULES`（`:33`）、`ALL_PROCESS_RULES`（`:35`）、`ALL_THREAD_RULES`（`:37`）、`ALL_CAUTION_RULES`（`:38`）。
- `AddRule` 按位路由：线程位 → `threadLocalRules_`，进程位 → `processRules_`（`hichecker.cpp:49`）。
- 合法性校验 `CheckRule`（`hichecker.cpp:218`）：`rule <= 0` 或非 `ALL_RULES` 子集即拒绝（静默 + `HILOG_INFO`）。
- 新增规则须同时更新聚合掩码与 NAPI/ANI 枚举导出 + `.ets` 常量，否则规则不可被应用使用。

### 7.4 告警流程

- 触发链：`NotifySlowProcess/NotifySlowEvent/NotifyAbilityConnectionLeak/NotifyCaution` → 构建 `Caution`（含 `GetBacktrace` 堆栈） → `HandleCaution` → `OnThreadCautionFound`/`OnProcessCautionFound`。
- 告警规则缺省为 `RULE_CAUTION_PRINT_LOG`（仅记日志）；`RULE_CAUTION_TRIGGER_CRASH` 走 `kill(getpid(), SIGABRT)`（`hichecker.cpp:198`）。
- `NeedCheckSlowEvent()` 返回 `checkMode_`，供高频路径前置短路（避免无规则时构造 Caution 的开销）。

### 7.5 错误码约定

- **C++ 层**：无异常 / 无错误码，非法规则静默忽略（仅日志）；返回 `void`/`uint64_t`/`bool`。
- **JS/ArkTS 层**：单一错误码 `401`（`ERR_PARAM`），参数须为且仅为 1 个可无损转 `uint64_t` 的 `BigInt`，否则抛 `BusinessError`（`napi_hichecker.cpp:150`、`ani_hichecker.cpp:96`）。
- **jsleakwatcher**：自定义 `108010xx` 错误码（`js_leak_watcher.ts:26`），与规则 API 错误码分离。

### 7.6 C-ABI 包装

- `InitHicheckerParamWrapper`（`hichecker_wrapper.h:22`，`extern "C"`）仅转调 `HiChecker::InitHicheckerParam`，供 C-only 调用方（如 init/bootstrap）初始化参数化规则，不拉入 C++ 符号。NAPI/ANI 不经此包装。

## 8. 项目约束

### 8.1 性能约束

- `NotifySlowProcess` 等检测点常位于子系统热路径（如图片编解码），无规则时必须经 `NeedCheckSlowEvent()`/位检测短路，**不得**在无规则时构造 `Caution` 或抓栈。
- 规则增删经 `std::mutex` 保护，单次调用须 <1ms（单测 `AddRulePerfTest001`/`RemoveRulePerfTest001` 有此断言）。
- 不要在告警热路径做冗余字符串格式化或 INFO 之外的高频日志。

### 8.2 架构约束

- 规则按作用域分离：线程位只能进 `thread_local threadLocalRules_`，进程位进 `processRules_`，不要混用。
- HiChecker 为静态工具类，不要改成实例单例（破坏既有 ABI/调用约定）。
- NAPI 与 ANI 是同一 API 的两套绑定，行为必须**完全一致**（方法集、错误码、规则常量数值），改一处须同步另一处 + `.ets`。
- jsleakwatcher 是独立模块，规则位掩码 API 不得依赖其内部实现。

### 8.3 公共 API / ABI 约束

**禁止（未经确认不得）：**

- 修改 `hichecker.h` 已有规则常量的**数值**（`1ULL<<63` 等）或新增/删除常量时不更新 `ALL_RULES` 与聚合掩码——破坏应用侧 BigInt 比对与向后兼容。
- 修改 `HiChecker` 静态方法签名、返回类型、参数顺序。
- 修改 `caution.h` 的 `Caution` 公有方法签名。
- 修改 `hichecker_wrapper.h` 的 `extern "C"` 签名（ABI 承诺）。
- 修改 JS 错误码 `401` 的语义或抛错条件。
- 修改 `@ohos.hichecker.ets` 已导出规则常量的 BigInt 字面量数值，除非对应 `hichecker.h` 同步变更。

**修改前必须确认：**

- 新增规则常量：须 `ALL_RULES` / 聚合掩码 / NAPI 枚举导出 / ANI 绑定 / `.ets` / README 接口表同步。
- 新增 C++ 静态方法：评估是否需 NAPI/ANI 暴露与 inner_kits 头文件登记（`bundle.json` `inner_kits`）。
- `innerapi_tags = ["platformsdk"]`（`libhichecker`）变更：影响 SDK 归属。

### 8.4 系统参数与 DAC 边界

**禁止：**

- 修改 `hichecker.para.dac` 的 DAC 权限（`root:shell:0775`）除非经安全评审。
- 扩大 `InitHicheckerParam` 的允许规则集（当前仅 `RULE_CHECK_ARKUI_PERFORMANCE`，`hichecker.cpp:42` `ALLOWED_RULE`）——参数化规则是安全敏感的远程开关，不得随意放宽。

**修改前必须确认：**

- 新增参数化规则支持：确认安全影响（系统参数可被具备写权限的进程设置，存在被滥用触发 crash/日志风暴风险）。
- 修改 `hichecker.para` 默认值：默认 `0` 表示无规则，不得默认开启 crash 类规则。

### 8.5 安全与权限边界

**禁止：**

- 在未校验规则合法性的路径直接将外部输入位掩码传入 `AddRule`（须经 `CheckRule`）。
- 将堆栈信息（`Caution::GetStackTrace()`）写入非安全日志或回传应用层非预期通道（README 明确暂不支持 JS 回栈）。
- 绕过 `RULE_CAUTION_TRIGGER_CRASH` 的 `SIGABRT` 语义，改用其他退出方式。

**修改前必须确认：**

- 涉及 `faultloggerd`（`libbacktrace_local`）抓栈能力的改动。
- 涉及 `init` 系统参数读写（`GetParameter`）的改动。

### 8.6 crash 行为约束

- `RULE_CAUTION_TRIGGER_CRASH` 必须 `kill(getpid(), SIGABRT)`（`hichecker.cpp:198`），不得改成静默或异步退出——这是开发者定位问题的契约。
- Fuzz 用例对含 crash 规则的路径须跳过 crash 分支（见 `hichecker_fuzzer.cpp` 对 `NotifyCautionFuzz` 的 guard），否则会杀掉 fuzzer 进程。

### 8.7 构建 / 生成代码边界

- `.abc`（`hichecker.abc`）由 `generate_static_abc` 从 `@ohos.hichecker.ets` 生成，**不要手改**生成产物；改 `.ets` 源。
- `jsleak_watcher.abc` 由 `es2abc_gen_abc` 从 `js_leak_watcher.ts` 生成，同理。
- `branch_protector_ret = "pac_ret"`、`-fstack-protector-strong` 等加固项不要移除。

## 9. 常见陷阱

- **新增规则只改了 `hichecker.h`，漏改 NAPI 枚举导出 / ANI 绑定 / `.ets`** — 应用侧拿不到新常量，规则无法被启用。
- **NAPI 与 ANI 行为不一致** — 同一方法在 JS 与 ArkTS 下错误码/返回类型不同，须两套同步。
- **把线程位规则塞进 `processRules_`** — 跨线程污染，检测错位；必须按 `ALL_THREAD_RULES`/`ALL_PROCESS_RULES` 分流。
- **无规则时仍构造 Caution + 抓栈** — 热路径性能退化；须先 `NeedCheckSlowEvent()`/位检测短路。
- **放宽 `ALLOWED_RULE`** — 系统参数可远程开启 crash 规则，存在被滥用导致进程被杀风险。
- **修改规则常量数值** — 破坏已发布应用对 BigInt 的依赖与向后兼容。
- **改 C++ API 不登记 inner_kits 头文件** — `bundle.json` `inner_kits` 未同步，外部无法引用。
- **手改 `.abc` 生成产物** — 下次重新生成被覆盖；应改 `.ets`/`.ts` 源。
- **Fuzz 用例触发 crash 规则未加 guard** — `SIGABRT` 杀掉 fuzzer，须对 `NotifyCaution`/`NotifyAbilityConnectionLeak` 路径跳过 crash 位。
- **把 jsleakwatcher 错误码（108010xx）与规则 API 错误码（401）混用** — 二者属不同模块，错误码空间分离。
- **在告警日志中明文打印堆栈到非安全通道** — `GetStackTrace()` 含敏感调用信息，须走 HiLog 且 `%{public}s` 仅限非敏感字段。
