# Emergency Shelter：Forge 1.20.1 移植审计与实施计划

> 状态：初步审计计划（尚未完成逐文件 API 审计、编译或运行验证）
>
> 源代码基线：仓库 `main` 当前提交 `b46f17dba0a1eef879f795738a005d18c9c93dbd`，原始项目版本标记为 1.0.3，目标为 Minecraft 1.20.1 + Forge 47.x。
>
> 原则：先保证加载架构成立，再逐项移植功能；不把“已识别风险”误报为“已修复”，也不把成功编译误报为功能验证通过。

## 1. 当前仓库基线

- 现有源码是 NeoForge 1.21.1 项目，不是 Forge 1.20.1 项目。
- 构建脚本使用 `net.neoforged.moddev` 插件、Java 21 toolchain、NeoForge 21.1.251 和 Minecraft 1.21.1。
- Gradle Wrapper 当前指向 Gradle 9.2.1；这与目标 ForgeGradle 版本的兼容性必须核实，不应直接沿用。
- 项目有 `src/early` 与 `src/main` 两套源码。最终发布物不是简单的单一 mod JAR：`shelterJar` 将 early 输出放在外层，并将普通游戏内模组 JAR 嵌入 `META-INF/emergencyshelter/emergencyshelter-mod.jar`。
- 资源模板位于 `src/main/templates/META-INF/neoforge.mods.toml`，并由 Gradle 任务生成元数据。
- 主源码包含三个 Mixin 配置：`emergencyshelter.compat.mixins.json`、`emergencyshelter.guard.mixins.json`、`emergencyshelter.mixins.json`。
- 关键 early 类包括 `EarlyShelter`、`ShelterLocator`、`ShelterService`；它们参与模组扫描/启动阶段，不能当作普通游戏内模组类直接替换 import。

## 2. 初步更改矩阵

| 子系统 | 现状/风险 | 计划处理 | 验收条件 |
|---|---|---|---|
| 构建插件 | NeoForge ModDev 2.0.148 | 改用与 Minecraft 1.20.1、目标 Forge 47.x 相容的 ForgeGradle 配置；核对插件仓库与 Gradle Wrapper 版本 | 在 Java 17 环境完成依赖解析、编译、资源处理和打包 |
| Java toolchain | 当前 Java 21 | 目标 Forge 1.20.1 通常以 Java 17 为基线；统一编译与运行工具链 | Gradle 运行时和 toolchain 均符合目标环境要求 |
| Minecraft/loader 元数据 | 1.21.1、NeoForge 21.1.251、NeoForge TOML | 调整为 1.20.1、Forge 47.x 和 `META-INF/mods.toml`；版本范围由实际验证结果确定 | Forge 能发现模组，元数据版本范围与构建产物一致 |
| Mod 初始化 | 使用 NeoForge 的 `@Mod`、`IEventBus`、`ModContainer` 包 | 逐个转换到 Forge 1.20.1 的 `net.minecraftforge.*` API；核对构造函数、总线获取方式和初始化时机 | 客户端及专用服务器均可完成模组发现与初始化 |
| 注册与生命周期事件 | 依赖 NeoForge 事件包/事件类型 | 按 Forge 1.20.1 事件 API 逐类映射；确认 MOD bus 与 Forge bus，检查并行事件中的线程安全 | 注册项成功；事件只注册一次且在正确总线上触发 |
| 客户端初始化/渲染 | NeoForge 客户端事件和渲染注册 | 映射到 Forge 1.20.1 对应事件；客户端类不能被专用服务器意外加载 | 客户端启动、进入世界和退出世界无类加载错误 |
| Early 模组发现 | 使用 NeoForge SPI 的 `IModFileCandidateLocator`、`ILaunchContext`、`IDiscoveryPipeline` 等类型 | **高风险、优先调查**：核对 Forge 47.x 的 ModLauncher、FML/扫描接口和服务发现机制；决定适配或重构 | 在干净 Forge 实例中验证扫描顺序、候选模组处理和失败路径 |
| ModLauncher 服务 | `ShelterService` 实现 `ITransformationService`，并触发扫描后监控 | 对比目标 ModLauncher 版本的接口、服务注册文件和调用时序；不假设接口二进制兼容 | 服务被目标加载器实际发现并调用，且不会提前/重复执行 |
| 启动监控/故障分析 | early 代码分析启动失败、依赖与候选模组 | 保留纯 Java/文件系统逻辑；将所有 loader 专属信息访问隔离到适配层 | 对模拟日志/缺失文件/异常路径有测试，且分析失败不会额外阻断启动 |
| 配置保护与回滚 | early 层包含配置保护、备份/恢复逻辑 | 检查路径、原子写入、权限、损坏文件和重复恢复语义 | 配置不被静默覆盖；恢复动作有日志且可重复验证 |
| Mixin | 三份配置；注入点依赖 Minecraft 版本 | 检查 Mixin bootstrap、配置清单、目标类/方法/描述符和 refmap 生成方式 | 启动日志确认三份配置均按预期加载；注入无致命失败 |
| 嵌套 JAR 打包 | 外层 early JAR + 内嵌普通模组 JAR | 尽可能保留现有发行结构，但先证明 Forge 能发现外层服务并按设计加载内部模组 | 检查最终 JAR 文件清单、服务声明、嵌套路径及真实启动结果 |
| 本地测试模组 | 条件应用 `testmod.gradle` | 审计其与 ForgeGradle 的兼容性；保持可选，不让发布构建依赖本地测试文件 | 无测试文件时正常构建；存在时不污染正式发布产物 |
| 发布与兼容声明 | 当前元数据是 NeoForge 版本范围 | 最后再修改版本范围和发布标识，避免在技术可行性未确认前作出承诺 | 声明与实际测试过的 Minecraft/Forge 版本一致 |

## 3. Early 启动架构是主要技术门槛

普通游戏内模组 API 的转换通常是逐类适配问题；本项目的 early 部分则运行在模组发现/启动阶段，并直接依赖 NeoForge 的候选模组定位与发现管线。Forge 47.x 的内部实现和依赖版本不应被假定与 NeoForge 21.x 相同。

在修改 early 源码之前，必须完成以下核对：

1. 从目标 Forge 47.x 的实际依赖图中确定 Forge、FML、ModLauncher 的版本。
2. 确认目标环境可用的服务接口、服务注册路径、类加载器和启动时序。
3. 对比候选模组发现接口；确定是否存在可用的 Forge 对应接口，以及是否允许把内部 JAR 加入扫描管线。
4. 确认故障处理逻辑不会在 Forge 自己生成崩溃报告前破坏原始日志、配置或模组目录。
5. 如果无法在 Forge 47.x 的受支持 API 上复现相同的早期发现行为，应明确记录功能降级方案，而不是用反射/内部类猜测来伪装兼容。

## 4. 目标版本与最低依赖策略

- 目标 Minecraft：1.20.1。
- 目标加载器：Minecraft Forge 47.x。
- Java：以 Java 17 为目标基线。
- **最低 Forge 构建版本尚未确定。** 需要先选择一个候选 47.x 版本完成 Gradle 解析、编译、启动和功能验证，再按所使用 API 的最低可用版本确定下限。不能仅凭“47.x”推断所有小版本都兼容。
- 构建工具版本需与所选 ForgeGradle 版本匹配；不要把当前 Gradle 9.2.1 当作既定目标。

## 5. 实施顺序

1. **只读审计**：完成所有 early/main Java 源文件、Mixin 配置、服务注册资源、依赖和打包任务的逐项清单。
2. **构建骨架**：迁移 Gradle/ForgeGradle、Java 17、资源生成与 Forge 元数据；暂不宣称功能完成。
3. **加载器可发现性原型**：优先验证 early 服务和内嵌 JAR 在 Forge 47.x 中是否能按预期被发现。这是继续全面移植前的关键门槛。
4. **普通模组 API 迁移**：迁移 mod 初始化、注册、事件总线、客户端事件和资源。
5. **Mixin 适配**：逐个验证配置和注入点，不以“无编译错误”作为唯一标准。
6. **构建与运行测试**：运行 `./gradlew clean build`，检查最终 JAR 内容，并分别测试客户端与专用服务器启动。
7. **版本声明收尾**：仅在最低可用 Forge 版本得到实际证据后，更新 metadata、README 和发布说明。

## 6. 验收清单

- [ ] Java 17 环境可完成 Gradle 配置和依赖解析。
- [ ] Forge 47.x 下主源码编译通过。
- [ ] early 源码编译通过，且服务注册机制已在目标加载器验证。
- [ ] 最终 JAR 的外层/内嵌结构与预期一致。
- [ ] Forge 能发现模组及其内部游戏模组部分。
- [ ] 三份 Mixin 配置在启动日志中按预期加载。
- [ ] 客户端可启动并进入测试世界。
- [ ] 专用服务器不会因客户端类引用而崩溃。
- [ ] 故障检测、配置回滚和依赖分析分别有可复现的测试场景。
- [ ] README、元数据和实际测试的兼容范围一致。

## 7. 当前结论

目前只能确认这是一个包含自定义 early 启动层的 NeoForge 1.21.1 项目，并且构建系统、模组元数据、事件 API 和 early 扫描机制都需要审计或迁移。尚未完成 Forge 1.20.1 的构建或运行验证。

**下一项工作应是逐文件 API/资源审计，优先厘清 `ShelterLocator`、`ShelterService`、`EarlyShelter` 的 Forge 47.x 可行性。** 只有这个门槛明确后，才值得大规模修改其余 96 个主源码文件。


## 8. README 功能范围与代码审计补充（2026-10-12）

已重新读取当前 `README.md`、递归源码树，以及部分关键实现和 Mixin 配置。README 所列功能是项目目标/说明，不等同于目标 Forge 环境下已经验证的行为。

### 8.1 README 中必须纳入回归测试的功能

| 功能域 | README 声明的行为 | 移植时必须验证 |
|---|---|---|
| 启动前保护 | 缺少前置、错误版本、重复/损坏模组可被临时跳过；提供启动报告 | Forge 是否允许在扫描/依赖解析阶段实现相同行为；错误候选是否会被正确隔离 |
| 占位数据 | 缺失模组的物品、方块、生物及玩家附加数据可保留，模组回来后恢复 | 注册表读取、NBT/codec、实体/物品/方块序列化与反序列化兼容 |
| 避险箱 | 占位物品从玩家背包/末影箱移入玩家专属容器 | 容器注册、菜单同步、权限和服务器端权威逻辑 |
| 运行时隔离 | 方块实体、生物、物品、区块数据、配方等异常时隔离问题部分 | Mixin 注入点、异常传播边界、原版回退语义和数据不丢失 |
| 启动恢复 | 上次启动失败时先回滚配置，再尝试跳过问题模组 | early 与游戏内层共享状态文件的格式、写入时序和恢复幂等性 |
| 存档/配置保护 | 备份、恢复损坏文件、减少模组前备份存档 | 路径定位、文件原子替换、备份空间与错误处理 |
| 渲染保护 | 隔离渲染异常；必要时关闭问题光影/资源包 | 客户端专属类隔离、渲染目标方法与 Forge/Minecraft 1.20.1 对应实现 |
| 世界生成/脚本/卡死/数据膨胀 | 隔离出错的局部操作，提供安全模式 | 注入点准确性、线程/超时处理、是否会掩盖可恢复以外的严重错误 |
| 管理与诊断 | 命令、报告、隔离对象重试、配置恢复 | 命令注册 API、权限检查、网络同步和服务器/客户端一致性 |

### 8.2 源码结构与已确认的技术事实

- `src/early` 当前包含 21 个 Java 文件；`src/main/java` 当前包含 96 个 Java 文件。
- Early 层的服务声明文件包括：
  - `META-INF/services/cpw.mods.modlauncher.api.ITransformationService`
  - `META-INF/services/net.neoforged.neoforgespi.locating.IModFileCandidateLocator`
- `ShelterLocator` 实现 NeoForge 的 `IModFileCandidateLocator`，并通过 `ILaunchContext` / `IDiscoveryPipeline` 调用 `EarlyShelter.locate`。
- `ShelterService` 实现 ModLauncher 的 `ITransformationService`，在 `transformers()` 时调用 `LaunchMonitor.afterScan()`。
- `EarlyShelter` 的入口先运行启动分析，再尝试将 `META-INF/emergencyshelter/emergencyshelter-mod.jar` 加入发现流程；两条路径都捕获 `Throwable` 并记录错误，以避免本模组自己的早期处理阻断启动。
- `Defense` 定义了防御性回退：自己的处理逻辑失败时，尽可能记录错误并保留原始异常/原版行为。移植不能把这些保护语义简化成无条件吞异常。
- `WorldGuard` 使用 Forge/NeoForge 事件风格的服务端生命周期订阅，并处理世界会话标记、隔离日志与备份；其具体事件 API 需要逐项映射。
- 三份 Mixin 配置的 `compatibilityLevel` 当前均为 `JAVA_21`；目标 Forge 1.20.1 的 Java 17 基线要求重新审视所有编译目标、Mixin 版本与注入目标，不能只修改 JSON 字符串。
- 当前已读取的只是关键样本与完整文件路径清单，不应称为 117 个 Java 文件已完成逐行审计。

### 8.3 构建工具的官方基线参考

Forge 官方 ForgeGradle 6 文档使用 `net.minecraftforge.gradle` 的 6.0–6.2 版本范围，并明确 ForgeGradle 6 使用 Gradle 8。Minecraft Forge 1.20.1 的官方下载页提供 47.3.x MDK；项目应从选定的 MDK/版本配置开始对齐，而不是沿用当前 Gradle 9.2.1。参考：<https://docs.minecraftforge.net/en/fg-6.x/>、<https://files.minecraftforge.net/net/minecraftforge/forge/index_1.20.1.html>。

因此，计划中的构建骨架阶段应优先选定一个明确的 Forge 1.20.1 版本（例如官方 47.3.x 系列中的候选版本），对照其 MDK 的 Gradle Wrapper、ForgeGradle、Java toolchain 和 mappings 配置。候选版本仍须经过实际解析、编译和启动验证后，才可作为最低兼容版本声明。

## 9. 审计状态与下一步门槛

| 项目 | 当前状态 |
|---|---|
| README 功能范围整理 | 已整理；尚未逐项做运行回归 |
| 仓库源码路径清点 | 已完成递归路径清点 |
| Early 启动关键类样本阅读 | 已读取 `EarlyShelter`、`ShelterLocator`、`ShelterService` 等关键类的样本 |
| 主源码 API 全量映射 | 未完成 |
| 所有 Mixin 目标/描述符审计 | 未完成；目前只核对了配置清单与部分样本 |
| Forge 47.x early 服务可行性 | 未验证，是首要门槛 |
| ForgeGradle 构建与编译 | 未运行 |
| 客户端/专用服务器运行测试 | 未运行 |

**下一步顺序保持不变：先研究 Forge 47.x 的启动服务、候选模组扫描和嵌套 JAR 发现能力；并行继续逐个检查 early 文件与 Mixin 目标。只有找到可验证的加载器方案后，才进入大规模主源码 API 迁移。**
