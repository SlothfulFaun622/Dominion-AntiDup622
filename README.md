# Dominion-AntiDup622

Dominion 4.8.4 的一个分叉，补掉了 Paper 服务端上末地传送门复制物品的漏洞。

我没打算把它做成什么新插件，就是个补丁：插件名改成 `Dominion-AntiDup622` 方便跟原版区分，数据目录还是 `plugins/Dominion/`，配置也沿用原来的，老服直接换 jar 就行。

## 这个漏洞是怎么回事

Paper 在玩家插上最后一颗末影之眼的时候，顺序是这样的：

1. 先把传送门方块塞进中间 3x3，箱子里掉出来的东西这时候已经在地上了；
2. 然后才发 `BlockMultiPlaceEvent`。

Dominion 的领地检查是在第 2 步才跑。只要它取消掉这个事件，Paper 就会把方块还原回原样 —— 地上那份掉落物还在，箱子里的东西也回来了。放个潜影盒进去，插个眼，东西就多一份。

这个顺序 Paper 那边确认过是有意为之，也没有 API 能让你在不触发复制的情况下把方块挡回去。所以「在激活那一刻拦截」这条路走不通。

## 补丁的做法

把拦截点挪到 `BlockPlaceEvent`：

- 一个 12 个框、朝向都对、但还没激活的末地门，中间 3x3 一律不让放方块。事件取消之后 Paper 根本不会替换方块，也就没有掉落，复制链断了。
- 已经激活的门（3x3 里有末地传送门方块）直接放行，刷沙机这类机器照常用。
- 取消放置不消耗玩家手上的物品，只弹一句提示。
- `BlockMultiPlaceEvent` 那边对末地门相关的事件直接豁免，不再掺和，让传送门正常生成。
- 同一个玩家 10 秒内被拦 3 次，接下来 10 秒里他在这个门周围（门框本身加上紧贴的一圈）放不了东西，省得有人搁那儿一直点。
- 插件启动 5 秒后扫一遍已加载的区块，之后每个区块加载时也扫一次，把老漏洞留下的、卡在没激活的门中间的方块清掉（容器里的东西先掉出来，再清方块）。

## 和原版的区别

- 插件名 `Dominion-AntiDup622`；主类、命令、权限节点、PlaceholderAPI 占位符都没变。
- 数据目录仍然是 `plugins/Dominion/`。插件启用时会把数据目录指回这个路径，老服务器的配置和数据库不用搬。
- 配置多了一段 `end-portal-protection`。版本号保持原版的 `3`，老配置读进来只会补上缺的段，不会整体重置。
- 其余代码跟上游 4.8.4 一致。

## 配置

`plugins/Dominion/config.yml`：

```yaml
end-portal-protection:
  # 总开关
  enable: true
  # 是否清理未激活末地门 3x3 里遗留的方块
  clean-existing-structures: true
  # 触发频率限制后，禁止在门周围放置方块的时长（秒）
  cooldown-seconds: 10
```

提示语在语言文件的 `end-portal-protection-handler-text` 段，四种语言（en_us / zh_cn / zh_tw / jp_jp）都补了。

## 安装

1. 先把 `plugins/Dominion/` 整个备份一份；
2. 删掉原来的 Dominion jar，把这个放进去；
3. 用 Java 21 或更高版本开服。full 版里有 1.21.9 的模块，Java 17 上会报 `UnsupportedClassVersionError` —— 原版 full 版一样起不来，不是这个补丁的问题。

## 自己编译

`api/` 目录里放的是 DominionAPI 4.8.4（提交 `f40955f0`）的快照，不用再拉 submodule。

```
gradlew shadowJar -PBuildFull=true
```

需要 JDK 17 和 JDK 25（`v26` 那个模块要 25）。Gradle 找不到的话，把 `gradle.properties` 里的 `org.gradle.java.installations.paths` 指过去。

## 已知限制

- 只认标准结构：12 个框、朝向都对、四角空着。结构不对的末地门它抓不到 —— 那种门一般也激活不了。
- 只在 Paper 1.20.1 上实测过。更新的版本理论上是同一套事件顺序，但我没环境试。
- 真人客户端上的手感（提示语、连续被拦之后的冷却）没法自动化测试，我只验证了服务端的判定逻辑。遇到问题直接开 issue，说清楚服务端版本和复现步骤就行。

## 许可

MIT，跟上游一致。原版 Dominion 是 LunaDeerMC 的项目，这个 fork 只改了上面提到的这些地方。