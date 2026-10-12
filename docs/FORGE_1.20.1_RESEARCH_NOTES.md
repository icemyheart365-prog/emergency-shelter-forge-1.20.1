# Forge 1.20.1 移植研究记录：Early 加载与构建基线

> 更新日期：2026-10-12
> 状态：资料核对阶段；尚未修改构建脚本或 Java 源码，也尚未运行 Forge 构建。
> 目标：Emergency Shelter 从 NeoForge 1.21.1 移植到 Minecraft 1.20.1 + Forge 47.x。

## 1. 当前结论

这不是单纯将 NeoForge import 批量替换为 Forge import 的移植。普通游戏内 API 可以逐项映射，但项目的 early 层直接介入 ModLauncher / FML 的启动与模组发现，必须先确定 Forge 47.x 的真实 SPI。

Forge 1.20.1 的模组发现 API 使用 Forge SPI 的 `net.minecraftforge.forgespi.locating.IModLocator` / `IDependencyLocator`。Forge 47.2.17 的 `ModDiscoverer` API 文档说明其通过 `ServiceLoader<IModLocator>` 和 `ServiceLoader<IDependencyLocator>` 加载 locator。这与当前代码实现的 NeoForge `net.neoforged.neoforgespi.locating.IModFileCandidateLocator` 不是同一个接口。不能只改包名后就假定行为相同。

参考：
- Forge 47.2.17 ModDiscoverer API 文档：https://mcstreetguy.github.io/ForgeJavaDocs/1.20.1-latest/net/minecraftforge/fml/loading/moddiscovery/ModDiscoverer.html
- Forge 47.2.17 ModFile API 文档：https://mcstreetguy.github.io/ForgeJavaDocs/1.20.1-latest/net/minecraftforge/fml/loading/moddiscovery/ModFile.html
- Forge 1.20.x 官方文档入口：https://docs.minecraftforge.net/en/1.20.x/
- ForgeGradle 6 官方文档：https://docs.minecraftforge.net/en/fg-6.x/

## 2. 构建工具基线

Forge 官方 ForgeGradle 6 文档给出的插件写法为：

```groovy
// settings.gradle: pluginManagement.repositories 中需要 Forge Maven
maven { url = 'https://maven.minecraftforge.net/' }

// build.gradle
plugins {
    id 'net.minecraftforge.gradle' version '[6.0,6.2)'
}
```

ForgeGradle 6 使用 Gradle 8。Minecraft 1.20.1 的目标 Java 基线是 Java 17。当前仓库使用 NeoForge ModDev 2.0.148、Java 21、NeoForge 21.1.251 和 Gradle Wrapper 9.2.1，不能直接视作 Forge 1.20.1 的有效构建组合。

下一步构建试验应选定一个具体 Forge 47.x MDK 版本，并对齐该 MDK 的：
- Gradle Wrapper 版本；
- ForgeGradle 插件版本及 Maven 仓库；
- Java 17 toolchain；
- `minecraft { mappings ...; runs ...; mods ... }` 配置；
- `META-INF/mods.toml` 资源处理。

Forge 47.x 最低依赖版本仍未确定，需等编译和启动测试之后再声明。

## 3. Early SPI 差异

### 当前 NeoForge 实现

- `ShelterLocator` 实现 `net.neoforged.neoforgespi.locating.IModFileCandidateLocator`。
- 其入口接收 `ILaunchContext` 和 `IDiscoveryPipeline`，并调用 `EarlyShelter.locate`。
- 服务文件当前为 `META-INF/services/net.neoforged.neoforgespi.locating.IModFileCandidateLocator`。
- `ShelterService` 实现 `cpw.mods.modlauncher.api.ITransformationService`，在 `transformers()` 中触发 `LaunchMonitor.afterScan()`。

### Forge 1.20.1 需核实/适配

- Forge locator SPI 名称为 `net.minecraftforge.forgespi.locating.IModLocator`；方法签名和返回类型应从所选 Forge MDK 实际解析出的 ForgeSPI JAR 核实。
- 服务注册名必须与实现的目标接口完全一致；旧的 NeoForge 服务文件不能留作 Forge locator 注册。
- Forge 的扫描器还存在 dependency locator 概念；不得将其和 mod locator 混为一谈。
- `ITransformationService` 属于 ModLauncher API。虽然包名相同，也必须核对目标 ModLauncher 版本、服务加载时序、方法签名和是否可在目标 Forge 运行环境中被发现。
- Forge 的内部扫描流程并非稳定的普通模组 API。若要跳过候选模组或将内部 JAR 动态加入发现流程，应优先用目标 Forge 47.x 实际接口验证；避免未经验证地依赖私有实现或反射。

## 4. Voxy 等多版本项目能证明什么

公开项目确实存在针对 Minecraft 1.20.1 Forge 与 1.21.1 NeoForge 分别构建的版本。例如 Neo Voxy 项目列出 1.20.1 Forge（Java 17）和 1.21.1 NeoForge（Java 21）产物；Voxy Server Side 的发布页也同时列出这两个平台的独立构建。

参考：
- Neo Voxy 多版本仓库：https://github.com/StrangeSmell/Voxy
- Voxy Server Side Forge/NeoForge 发布页：https://www.curseforge.com/minecraft/mc-mods/voxy-server-side-forge-neoforge/files/all

这些例子说明跨版本/跨加载器移植是可行的，但不能推导出所有项目都容易移植。Emergency Shelter 的 early 模组发现、错误恢复、Mixin 保护和嵌套 JAR 打包使其复杂度明显高于普通内容模组。可借鉴构建方法，不能照搬其兼容结论。

## 5. 目前不能声称已完成的事项

- 未确认 Forge 47.x 的 early locator 是否能完整实现当前 NeoForge pipeline 的功能。
- 未确认嵌套 `emergencyshelter-mod.jar` 的当前打包方式能否被 Forge 发现。
- 未完成所有 Mixin 目标类、方法描述符和注入点的 1.20.1 对照。
- 未完成所有 Java 文件的逐文件 API 映射。
- 未运行 `./gradlew clean build`，未运行客户端或专用服务器。

## 6. 后续执行顺序

1. 选择并记录一个明确的 Forge 47.x 候选版本。
2. 从 Forge 47.x 实际源码/JAR 核实 `IModLocator`、服务发现、ModLauncher 版本与嵌套 JAR 处理。
3. 先做最小 early-loader 原型，验证 Forge 是否会发现服务及其 locator；原型不能破坏现有主源码。
4. 建立 ForgeGradle + Java 17 构建骨架并跑首次编译，按错误清单迭代。
5. 逐类迁移主源码 API，再逐个验证 Mixin。
6. 最终做客户端、专用服务器、缺失模组恢复和备份/回滚回归测试。
