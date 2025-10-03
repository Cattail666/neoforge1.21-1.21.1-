
# 📖 NeoForge 事件系统

## 🔌 事件总线（Event Bus）

事件都会丢到一个"总线"上：

- **游戏总线（Game Bus）** 👉 `NeoForge.EVENT_BUS` （最重要，几乎所有事件都在这上面）
- **模组总线（Mod Bus）** 👉 每个模组启动时会生成一个专属的总线，并传进构造函数

**区别是：**

- 游戏总线的事件都在同一个线程执行 🧵
- 模组总线有些事件会并行执行 ⚡，这样启动速度更快

## 🛠️ 注册事件监听器（Event Handler）

事件监听器就是你写的响应函数。要求是：

- 方法参数只有一个，就是事件对象
- 方法返回值必须是 `void`

### 方式一：用 IEventBus#addListener ✨（最简单）

```java
@Mod("yourmodid")
public class YourMod {
    public YourMod(IEventBus modBus) {
        NeoForge.EVENT_BUS.addListener(YourMod::onLivingJump);
    }

    // 实现：每次跳跃时回血半格 ❤️
    private static void onLivingJump(LivingEvent.LivingJumpEvent event) {
        LivingEntity entity = event.getEntity();
        if (!entity.level().isClientSide()) {
            entity.heal(1);
        }
    }
}
```

### 方式二：用 @SubscribeEvent 注解 🏷️

```java
public class EventHandler {
    @SubscribeEvent
    public void onLivingJump(LivingEvent.LivingJumpEvent event) {
        LivingEntity entity = event.getEntity();
        if (!entity.level().isClientSide()) {
            entity.heal(1);
        }
    }
}

@Mod("yourmodid")
public class YourMod {
    public YourMod(IEventBus modBus) {
        NeoForge.EVENT_BUS.register(new EventHandler());
    }
}
```

### 方式三：静态方法（更省事） 🗂️

```java
public class EventHandler {
    @SubscribeEvent
    public static void onLivingJump(LivingEvent.LivingJumpEvent event) {
        LivingEntity entity = event.getEntity();
        if (!entity.level().isClientSide()) {
            entity.heal(1);
        }
    }
}

@Mod("yourmodid")
public class YourMod {
    public YourMod(IEventBus modBus) {
        NeoForge.EVENT_BUS.register(EventHandler.class);
    }
}
```

### 方式四：@EventBusSubscriber 🚀（完全自动化）

```java
@EventBusSubscriber(modid = "yourmodid")
public class EventHandler {
    @SubscribeEvent
    public static void onLivingJump(LivingEvent.LivingJumpEvent event) {
        LivingEntity entity = event.getEntity();
        if (!entity.level().isClientSide()) {
            entity.heal(1);
        }
    }
}
```

✅ NeoForge 会自动扫描这个类，你不用在构造函数里手动注册。

⚠️ 注意：所有事件方法必须是 `static`。

## 🎯 事件的属性

### 字段和方法

事件对象里一般带上下文信息，比如：

- 谁触发了事件（实体）
- 事件在哪个世界发生

### 继承层次 🧬

事件类不是都直接继承 Event，有层次关系：

- `BlockEvent` 👉 方块相关
- `EntityEvent` 👉 实体相关  
- `LivingEvent` 👉 生物相关
- `PlayerEvent` 👉 玩家相关

⚠️ 千万不要监听抽象父类事件（比如 `EntityEvent`），否则游戏会崩掉。必须监听具体子类。

## ⛔ 可取消事件（Cancellable Events）

有些事件可以取消（实现了 `ICancellableEvent` 接口）：

- 用 `setCanceled(true)` 取消事件
- 取消后其他监听器就不会收到这个事件
- 游戏对应的行为也会被阻止

**例子：** 取消 `LivingChangeTargetEvent`，生物就不会更换攻击目标。

💡 如果你还想监听那些"已经取消"的事件，可以在注册时加 `receiveCanceled = true`。

## ⚖️ 三态值（TriState）和结果（Result）

有些事件不止能取消，还能决定"怎么处理"：

- ✅ `TriState.TRUE` → 强制执行
- ❌ `TriState.FALSE` → 强制不执行  
- 🪢 `TriState.DEFAULT` → 按默认逻辑

**例子：**

```java
@SubscribeEvent
public static void renderNameTag(RenderNameTagEvent.CanRender event) {
    event.setCanRender(TriState.FALSE); // 阻止显示名字
}

@SubscribeEvent
public static void mobDespawn(MobDespawnEvent event) {
    event.setResult(MobDespawnEvent.Result.DENY); // 禁止生物消失
}
```

## 🥇 事件优先级（Priority）

事件监听器可以有优先级：

- 🔥 `HIGHEST`
- 🔼 `HIGH` 
- ⚖️ `NORMAL`（默认）
- 🔽 `LOW`
- 🧊 `LOWEST`

执行顺序是 **从高到低**。
同优先级的，顺序和模组加载顺序有关。

## 🖥️ 单边事件（Sided Events）

有些事件只在一边触发：

- 渲染相关事件只在客户端触发 🎨

如果用 `IEventBus#addListener()`，要手动判断 `FMLEnvironment.dist`。
如果用 `@EventBusSubscriber`，可以直接写：

```java
@EventBusSubscriber(value = Dist.CLIENT, modid = "yourmodid")
```

## 🔄 事件总线分类

- 大多数事件走 **游戏总线** 👉 `NeoForge.EVENT_BUS`
- 有些走 **模组总线（mod bus）** 👉 实现了 `IModBusEvent` 的事件

如果你用 `@EventBusSubscriber`，NeoForge 会自动帮你注册到对应总线。

## 📅 模组生命周期（The Mod Lifecycle）

模组在启动时，会按顺序触发一系列生命周期事件：

1. 🚀 调用模组构造函数（这里通常注册事件）
2. ⚡ 执行所有 `@EventBusSubscriber`
3. 🏗️ 触发 `FMLConstructModEvent`
4. 📦 各种注册事件（`NewRegistryEvent`、`RegisterEvent` 等）
5. 🔧 触发 `FMLCommonSetupEvent`（初始化）
6. 🎨 客户端事件：`FMLClientSetupEvent` / 🌐 服务端事件：`FMLDedicatedServerSetupEvent`
7. 🔄 模组间通信（`InterModComms`）
8. ✅ 最后触发 `FMLLoadCompleteEvent`

## 🔗 模组间通信（InterModComms）

`InterModComms` 系统让模组之间能发消息，提高兼容性。

- 在 `InterModEnqueueEvent` 👉 用 `InterModComms#sendTo` 发消息给其他模组
- 在 `InterModProcessEvent` 👉 用 `InterModComms#getMessages` 收到别的模组的消息

每条消息包含：
- 📤 发送者
- 📥 接收者  
- 📝 数据 key
- 📦 数据本身

## 🧩 其他模组总线事件

除了生命周期事件，还有一些"杂项事件"在 mod bus 上触发，比如：

- `RegisterColorHandlersEvent.Block`
- `ModelEvent.BakingCompleted` 
- `TextureAtlasStitchedEvent`

⚠️ 注意：大多数这类事件未来会被迁移到游戏总线上。

---
