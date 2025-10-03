# 📦 NeoForge 注册表

## 什么是注册？

在 NeoForge 里，**注册（Registration）** 就是把模组里的东西（物品 Items、方块 Blocks、生物 Entities 等）告诉游戏。

> ⚠️ **重要提醒**：如果你不注册，游戏就完全不知道它们的存在，结果就是各种离谱的 bug 🐛 甚至闪退 💥。

## 🗂️ 什么是注册表（Registry）？

可以把**注册表**理解成一个"字典/映射表"：

- **key（键）** 👉 注册名（registry name）
- **value（值）** 👉 注册进去的对象（注册项 registry entry）

### 🔑 规则：

- 注册名在**同一个注册表里**必须唯一
- **不同注册表**可以有相同名字

### 📌 例子：

- `BLOCKS` 注册表里有 `minecraft:dirt`（泥土方块 🧱）
- `ITEMS` 注册表里也有一个 `minecraft:dirt`（泥土方块的物品形态）

## 🏷️ 注册名（Registry Name）

每个注册的对象都有唯一的名字，叫**注册名**，格式是：

```
namespace:path
```

👉 用 `ResourceLocation` 表示。

### 📌 举例：

- 泥土方块：`minecraft:dirt`
- 僵尸实体：`minecraft:zombie` 🧟

> ⚠️ **注意**：模组加的内容不会用 `minecraft` 命名空间，而是用你的 `modid`。

例如模组 `coolmod` 注册一个魔法石：
```
coolmod:magic_stone
```

## 🏗️ Vanilla vs. Modded

要理解 NeoForge 的设计，先看看 Minecraft 本身是怎么做的：

- 注册表里的东西都是**单例（singleton）**
- 👉 游戏里所有的石头其实都是同一个 `Block` 实例，只是显示了很多次

Minecraft 把所有方块都注册在 `Blocks` 类里，通过 `Registry#register()` 把方块放进 `BuiltInRegistries.BLOCK`。

注册完后，游戏会做一些检查（比如方块有没有模型）。

**为什么能这样？** 因为 `Blocks` 这个类会被游戏很早加载。
而模组不会自动被加载，所以需要 NeoForge 提供额外的工具来帮你注册。

## 🛠️ 注册对象的方法

NeoForge 提供了两种注册方式：

1. **DeferredRegister** ✅（推荐）
2. **RegisterEvent**

> ⚠️ `DeferredRegister` 本质上就是对 `RegisterEvent` 的封装，但更不容易出错。

---

### 1️⃣ DeferredRegister（推荐）

#### ✨ 创建 DeferredRegister

```java
public static final DeferredRegister<Block> BLOCKS = DeferredRegister.create(
    // 要用的注册表
    BuiltInRegistries.BLOCKS,
    // 我们模组的 modid
    ExampleMod.MOD_ID
);
```

#### ➕ 注册对象

```java
public static final DeferredHolder<Block, Block> EXAMPLE_BLOCK_1 = BLOCKS.register(
    "example_block", // 注册名
    () -> new Block(...) // 注册对象的构造方法
);

public static final DeferredHolder<Block, SlabBlock> EXAMPLE_BLOCK_2 = BLOCKS.register(
    "example_block_2", // 注册名
    registryName -> new SlabBlock(...) // 带注册名的构造方法
);
```

#### 📝 说明：

- `DeferredHolder<R, T extends R>` 保存了注册对象
- `R` 👉 注册表的类型（比如 `Block`）
- `T` 👉 具体对象类型（比如 `SlabBlock` 是 `Block` 的子类）

#### 🔄 DeferredHolder vs Supplier

`DeferredHolder` 继承了 `Supplier`，所以你也可以直接写成 `Supplier`：

```java
public static final Supplier<Block> EXAMPLE_BLOCK_1 = BLOCKS.register(
    "example_block",
    () -> new Block(...)
);

public static final Supplier<SlabBlock> EXAMPLE_BLOCK_2 = BLOCKS.register(
    "example_block_2",
    registryName -> new SlabBlock(...)
);
```

> ⚠️ **注意**：有些地方必须要用 `Holder` 或 `DeferredHolder`，不能只用 `Supplier`。遇到这种情况，就把类型改回来。

#### 🔗 注册到事件总线

最后别忘了要把 `DeferredRegister` 挂到事件总线（mod bus）：

```java
public ExampleMod(IEventBus modBus) {
    ExampleBlocksClass.BLOCKS.register(modBus);
    // 其他初始化
}
```

#### 💡 提示：

NeoForge 还提供了专门的版本：

- `DeferredRegister.Blocks`
- `DeferredRegister.Items` 
- `DeferredRegister.Entities`
- `DeferredRegister.DataComponents`

这些都有额外的便捷方法。

---

### 2️⃣ RegisterEvent

另一种方式是直接监听 `RegisterEvent`。

#### ⚙️ 触发时机：

这个事件会在：
- 模组构造函数之后触发（`DeferredRegister` 就是靠这个）
- 配置文件加载之前触发

事件挂在 **mod bus** 上。

#### 📌 示例代码

```java
@SubscribeEvent // 在 mod event bus 上
public static void register(RegisterEvent event) {
    event.register(
        // 注册表 key
        BuiltInRegistries.BLOCKS,
        // 注册对象
        registry -> {
            registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_1"), new Block(...));
            registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_2"), new Block(...));
            registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_3"), new Block(...));
        }
    );
}
```

---

## ✅ 总结

- **必须注册**：不注册 = 游戏完全不知道你的东西
- **注册表就是字典**：名字 → 对象
- **推荐用 DeferredRegister**：写法更清晰，避免出错
- **RegisterEvent 更灵活**：但一般只有在特殊情况下才直接用

---

*📝 文档整理完成！希望这份教程能帮助你更好地理解 NeoForge 的注册机制。*