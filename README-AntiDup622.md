# Dominion-AntiDup622 (Dominion 4.8.4 末地传送门防复制补丁)

作者: SlothfulFaun622  
基于: Dominion 4.8.4 (LunaDeerMC)  
文件: Dominion-4.8.4-full-AntiDup622.jar (full 版, 27,914,985 字节)  
SHA-256: `6DF6FD1821C378528EF6D311D412506B9C0B614F37A84632CFA919973E1627A3`

## 一、修复的漏洞

Paper 服务端在玩家放置最后一颗末影之眼激活末地门时, 内部顺序是:

1. 先用末地传送门方块替换 3x3 中间的方块, 容器内容物此时已经掉落;
2. 然后才触发 `BlockMultiPlaceEvent`。

Dominion 的领地检查在第 2 步才运行, 取消事件会让 Paper 还原被替换的方块,  
于是地上留着一份掉落物, 容器里的物品也回来了 —— 物品复制完成。

Paper 官方 Issue 已确认: 没有任何 API 方法能在不触发复制的情况下阻止这些方块被覆盖,  
所以「在激活时拦截」这条路走不通。

## 二、补丁做法

改在 `BlockPlaceEvent` 阶段拦截, 而不是 `BlockMultiPlaceEvent`:

- **未激活的完整末地门**: 只要 12 个 `END_PORTAL_FRAME` 方向一致围成标准结构  
  (四角为空, 每块框架朝向中心), 中间 3x3 内禁止放置任何方块。  
  取消事件 -> Paper 不会替换方块 -> 不会掉落 -> 没有复制路径。
- **已激活的传送门**: 3x3 内存在 `END_PORTAL` 时直接放行, 刷沙机等红石机器不受影响。
- **物品不消耗**: Paper 取消 `BlockPlaceEvent` 不会消耗玩家手上的物品。
- **提示**: 拦截时给玩家发消息 (可在语言文件里改)。
- **频率限制**: 同一玩家 10 秒内被拦 3 次, 之后 10 秒内其在该末地门周围  
  (门框本身 + 紧贴门框的一圈, 即中心外 3 格以内、门框那一层和上面一层) 放不了方块,  
  防止「放 -> 拦 -> 放」循环。传送门一旦激活, 限制立刻解除。
- **BlockMultiPlaceEvent 豁免**: 涉及 `END_PORTAL` / `END_PORTAL_FRAME` 的多方块放置事件  
  不再被领地检查取消, 让传送门正常生成, 避开 Paper 的复制路径。
- **存量清理**: 插件启动时 (延迟 5 秒) 扫描已加载区块, 并在每次区块加载时扫描,  
  把未激活末地门 3x3 内已存在的方块移走 (容器内容物先掉落), 避免旧漏洞残留。

## 三、新增配置

配置文件: `plugins/Dominion/config.yml`

```yaml
end-portal-protection:
  # 总开关
  enable: true
  # 是否清理未激活末地门 3x3 内已存在的方块
  clean-existing-structures: true
  # 触发频率限制后, 禁止在该传送门周围放置方块的时长 (秒)
  cooldown-seconds: 10
```

语言文件新增 `end-portal-protection-handler-text` 段:  
`place-denied` / `cooldown-applied` / `cooldown-denied`  
(en_us / zh_cn / zh_tw / jp_jp 均已补齐)。

## 四、部署注意

1. 插件名是 `Dominion-AntiDup622` (看 `/plugins` 列表、控制台日志前缀),  
   但 **数据目录仍然是 `plugins/Dominion/`**。  
   插件启用时会把数据目录指回原路径, 老服务器直接覆盖 jar 即可,  
   配置、数据库、语言文件都不需要搬家。
2. 如果之前装过本补丁的早期构建 (那一版用过 `plugins/Dominion-AntiDup622/`),  
   首次启动会自动把这个目录改名回 `plugins/Dominion/` 并打一条日志。  
   如果两个目录同时存在, 插件优先使用 `plugins/Dominion/`,  
   旧的 `plugins/Dominion-AntiDup622/` 不会合并, 确认数据无误后可手动删除。
3. 主类不变 (`cn.lunadeer.dominion.Dominion`), 权限节点、命令、PlaceholderAPI 标识  
   (`dominion`) 全部不变, 其他插件不受影响。
4. `config.yml` 的版本号仍然是原版的 `3`, 不会触发整体重置:  
   老配置直接读取, 缺的 `end-portal-protection` 段会自动补上并写入文件,  
   数据库、语言等原有设置原样保留 (不会生成 `config.yml.bak`)。  
   例外: 如果用过本补丁之前的构建 (那里写的是 `version: 4`), 会重置一次并留下  
   `config.yml.bak` —— 那是配置版本号不同的正常行为, 改回默认值即可。
5. 建议先把 `plugins/Dominion/` 整个备份一份, 确认运行正常后再删备份。
6. full 版包含 1.21.9 模块 (Java 21 字节码), 请使用 **Java 21 或更高版本** 开服。  
   在 Java 17 上启用会报 `UnsupportedClassVersionError` —— 原版 `release-full.jar`  
   也一样, 不是本补丁引入的问题。

## 五、测试要点

- [x] 建一个带箱子/石头的未激活末地门 -> 启动扫描清空中间 3x3, 框架保持原样。
- [x] 已激活的传送门 -> 不判定为「未激活」, 放行。
- [x] 框架朝向不对 (不是标准结构) -> 不判定为传送门, 不拦截。
- [x] 数据目录为 `plugins/Dominion/`, 且不会额外生成 `plugins/Dominion-AntiDup622/`。
- [x] 旧目录 `plugins/Dominion-AntiDup622/` 自动迁移回 `plugins/Dominion/`。
- [x] 冷却范围判定: 门中心同一层 3 格内受限, 4 格外放行; 门框上一层受限, 两层以上和下方放行。
- [x] 老配置 (没有 `end-portal-protection` 段) 直接升级: 原有设置原样保留, 该段自动补上, 不生成 `.bak`。
- [ ] 真人放置测试: 在中间 3x3 放箱子/潜影盒 -> 应被拦截, 物品不消耗, 容器物品不复制。
- [ ] 放进最后一颗末影之眼激活传送门 -> 传送门正常生成, 不产生复制物品。
- [ ] 反复尝试放置 -> 触发频率限制, 提示临时禁止。
- [ ] 领地功能正常: 有权限的玩家照常放置, 无权限玩家在领地内仍被领地规则拦截。

前八项已在 Paper 1.20.1 (build 77) + Java 21 环境下实测通过。

## 六、构建说明

源码: Dominion 4.8.4, `api` 子模块按 4.8.4 标签锁定的提交 `f40955f0` 还原。

构建命令:

```
gradlew shadowJar -PBuildFull=true
```

产物名由 `build.gradle.kts` 固定为 `Dominion-<版本>-full-AntiDup622.jar`。

## 七、变更记录

- 2026-11-24 第一版: 末地传送门防复制补丁 + full 打包。
- 2026-11-24 第二版: 插件名改为 `Dominion-AntiDup622` 后, 数据目录仍保持 `plugins/Dominion/`,  
  并修掉存量扫描里 `ChunkSnapshot#isSectionEmpty` 的段落索引越界 (会导致区块加载报错)。
- 2026-11-24 第三版: 频率限制从「周围 6 格 / 60 秒」改为「门框及紧贴的一圈 / 10 秒」,  
  传送门激活后限制立即解除; 配置版本号回到原版的 3, 老配置升级不再整体重置。