# ⚖️ NeoForge 教程：客户端 & 服务器（Sides）

## 概述

Minecraft 和很多程序一样，遵循 **客户端（Client）—服务器（Server）** 的模式：

- **客户端** 🎨 👉 负责显示游戏画面
- **服务器** ⚙️ 👉 负责更新游戏逻辑（比如方块状态、生物 AI、天气等）

> 🤔 听起来好像很直观？但其实……并不完全是这样。因为 Minecraft 有两种"端的概念"：
> - **物理端（Physical Side）** 🖥️
> - **逻辑端（Logical Side）** 🧩

## 🔌 物理端（Physical Side）

### 什么是客户端（ Client）
客户端就是你 运行游戏的那一边，负责：
- 画面渲染：你看到的方块、实体、GUI 界面。
- 声音播放：打怪声、放方块声。
- 玩家输入：键盘、鼠标（走动、点击）。
- 立即反馈：比如你右键时，手臂挥动的动作。

👉 可以理解成：客户端是 舞台前端，负责“看起来”和“操作”。

### 什么是服务端（ Server）
- 服务端是 决定游戏逻辑的权威，负责：
- 世界状态：方块到底放没放上去，怪物位置在哪里。
- 物品状态：你到底有没有这把剑，苹果是不是真的消耗掉了。
- 规则检查：你能不能打开某个箱子、能不能攻击这个实体。
- 多人同步：把每个玩家的操作传给其他玩家。

👉 可以理解成：服务端是 幕后导演，说了算。

> ⚠️ **警告**：如果你在物理服务器上调用客户端类，会直接报错 `ClassNotFound` 或者 `NoClassDefFoundError`，导致崩溃 💥。

## 🔍 两者的区别

看两个场景就懂了：

### 场景一：玩家进多人服务器
```
物理客户端 + 逻辑客户端（玩家电脑）
        ↓
连接到 物理服务器 + 逻辑服务器（远程服务器）
```

### 场景二：玩家开单人世界
- 玩家启动物理客户端
- 物理客户端自己开了一个逻辑服务器 🖥️
- 然后逻辑客户端连接到这个逻辑服务器（相当于本地 localhost）

### 💡 重要结论：
- **逻辑端能运行你的代码，不代表物理端一定能运行**
- 你必须在 **专用服务器（物理服务器）** 测试模组，否则可能会遇到各种崩溃

> ⚠️ **常见错误**：
> - 不区分客户端/服务器代码 → `ClassNotFoundException` ❌
> - 静态字段在逻辑两端同时用 → 数据不同步 ⚡

## 📡 端之间通信

如果你需要在两个端之间传数据：
👉 必须用 **数据包（packet）**。

在 NeoForge 里：
- **物理端** 用枚举 `Dist` 表示
- **逻辑端** 用枚举 `LogicalSide` 表示

## 🛠️ 怎么判断在哪个端？

### 1. `Level#isClientSide()` ✅ （推荐）

```java
if (level.isClientSide()) {
    // 当前是逻辑客户端 🎨
    // 执行客户端逻辑
} else {
    // 当前是逻辑服务器 ⚙️
    // 执行服务器逻辑
}
```

- `true` → 当前是逻辑客户端 🎨
- `false` → 当前是逻辑服务器 ⚙️

#### 📌 举例：
玩家点击方块，你想让它掉血/生成钻石机器。
👉 必须在逻辑服务器里执行（`isClientSide == false`），否则会造成数据不同步或崩溃。

> ⚠️ **注意**：`false` 不代表物理服务器！
> 在单人世界里，物理客户端也会包含逻辑服务器。

### 2. `FMLEnvironment.dist`

这是物理端的判断：

```java
import net.minecraftforge.fml.loading.FMLEnvironment;

if (FMLEnvironment.dist == Dist.CLIENT) {
    // 物理客户端
} else if (FMLEnvironment.dist == Dist.DEDICATED_SERVER) {
    // 物理服务器
}
```

- `Dist.CLIENT` → 物理客户端
- `Dist.DEDICATED_SERVER` → 物理服务器

## 🧭 用 `@Mod` 区分物理端代码

有些代码只在客户端能跑（比如渲染），就要写在专门的类里，并用 `@Mod` 注解限制加载位置：

### 主模组类（两端都加载）
```java
@Mod("examplemod")
public class ExampleMod {
    public ExampleMod(IEventBus modBus) {
        // 两端都要执行的逻辑
    }
}
```

### 客户端专用类
```java
@Mod(value = "examplemod", dist = Dist.CLIENT)
public class ExampleModClient {
    public ExampleModClient(IEventBus modBus) {
        // 只在物理客户端执行的逻辑
        // 比如：注册渲染器、键位绑定等
    }
}
```

### 服务器专用类
```java
@Mod(value = "examplemod", dist = Dist.DEDICATED_SERVER)
public class ExampleModDedicatedServer {
    public ExampleModDedicatedServer(IEventBus modBus) {
        // 只在物理服务器执行的逻辑
        // 比如：服务器专用的配置初始化
    }
}
```

## ✅ 总结

- Minecraft 有两种「端」：
  - **物理端**（程序层面）
  - **逻辑端**（游戏逻辑层面）

- **推荐用 `Level#isClientSide()`** 判断逻辑端

- **推荐用 `FMLEnvironment.dist`** 判断物理端

- **专用服务器测试必不可少** 🛠️

- 区分好逻辑端/物理端，是写模组时避免 90% 崩溃的关键

---
#在最后我要举个例子
##场景：
你放了一块石头 🪨，然后你右键这块石头，让它冒火焰粒子 🔥。

如果你在客户端直接写代码生成粒子
---
	level.addParticle(ParticleTypes.FLAME, x, y, z, 0, 0, 0);


- 结果：只有你自己能看到。
- 因为这是你本地机器调用的渲染方法，你好友的客户端根本不知道你加了火焰。
- 效果：你眼里石头冒火焰 → 好友眼里石头很普通。

👉 这是 客户端私有特效。

 如果你在服务端触发粒子
---
	((ServerLevel)level).sendParticles(ParticleTypes.FLAME, x, y, z, 10, 0, 0, 0, 0);


- 所有在范围内的客户端都会收到粒子消息。
- 你的好友都能看到火焰粒子。
- 端扮演“广播站”的角色，谁在附近就告诉谁要画粒子。
👉 这是 全员共享特效。

🎯 总结一句

- 端 addParticle → 只有自己能看到

- 端 sendParticles → 所有人能看到
⚡ 换句话说：
如果你希望 所有玩家都能看到效果，必须在 服务端生成粒子并同步。
如果你只是想做一个 本地小彩蛋（比如调试），客户端直接渲染就行。
*🎯 记住：正确理解客户端和服务器的工作方式，是开发稳定模组的基础！*