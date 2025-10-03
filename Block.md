
# 🧱 方块是 Minecraft 世界不可或缺的一部分

它们构成了所有地形、结构和机器。如果您有兴趣制作模组，那么您可能需要添加一些方块。本页面将指导您创建方块，以及使用方块的一些功能。

---

## 📋 创建方块注册表

我们得先创造一个方块注册表：

1. 
```java
public static final DeferredRegister<Block> BLOCKS = DeferredRegister.create(BuiltInRegistries.BLOCKS, ExampleMod.MOD_ID);
```

2. 
```java
public static final DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks("yourmodid");
// 注意，createblock和create的区别就是不需要手动选择方块类型注册表，createblock自动选择了，这是一个语法糖
```

这个 `BLOCKS` 就是表，接着我们造注册一个方块进表里。

这里有两种方法：

1.
```java
public static final DeferredHolder<Block, Block> BLOCKS.register("my_block", () -> new 									Block(BlockBehaviour.Properties.of())
);
```

2.
```java

public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.registerBlock(
    "example_block",
    Block::new, 
    BlockBehaviour.Properties.of()
);


⚠️ **需要注意的是：不要在注册之外调用 `new Block()`！一旦这样做，事情就可能崩溃：**

- 必须在注册表解冻期间创建区块。NeoForge 会先为您解冻注册表，然后再冻结它们，因此注册是您创建区块的时间窗口。
- 如果您在注册表再次冻结时尝试创建和/或注册一个块，游戏将崩溃并报告一个 null 块，这可能会非常令人困惑。
- 如果您仍然设法拥有一个悬垂的块实例，游戏将无法在同步和保存时识别它，并将其替换为空气。

这两种注册方法，第二种也是语法糖。第一种方法和第二种方法的属性和自定义的工厂（lambda表达式）分离了，增强了区分度。对于不需要特殊功能的简单方块（例如圆石、木板等），可以直接使用 `Block` 类。为此，在注册期间，使用 `BlockBehaviour.Properties` 参数实例化 `Block`。此 `BlockBehaviour.Properties` 参数可以使用 `BlockBehaviour.Properties.of` 创建。

---

## 🛠️ 如何创建自定义的块

直接使用 `Block` 只能实现非常基本的方块。如果要添加功能，例如玩家互动或不同的碰撞箱，则需要自定义一个继承自 `Block` 类。`Block` 有许多方法可以重写以执行不同的操作。

如果你想制作一个具有不同变体的方块（比如一个有底部、顶部和双层变体的台阶），你应该使用**方块状态**。最后，如果你想制作一个可以存储额外数据的方块（比如一个可以存储物品的箱子），你应该使用**方块实体**。这里的经验法则是，如果你的状态数量有限且相当少（最多几百个状态），就使用**方块状态**；如果你的状态数量无限或接近无限，就使用**方块实体**。你需要新建一个文件，用于定义新的块，然后在注册的时候，添加该块的工厂。

---

## 🔄 关于 Codec

在定义一个块之前我得说一下一个叫 **codec** 的东西，我要求每一个方块都必须写上 codec，这是必需的。

理论上来说：每一个 `Block` 的子类，都应该有一个属于自己的方块类型。**codec 是用来实现方块类型的**，方块类型（Block Type）其实就是一种用来保存和还原方块信息的工具。负责把方块对象序列化（存起来）和反序列化（读出来）。目前，这个东西只在生成方块列表报告的时候用到（比如调试或数据导出时）。

💡 **用一句话总结：方块类型（BlockType/MapCodec）= 一种“方块身份证”，用来保存/还原方块。** 每种方块子类都有一个自己的类型，简单的用 `simpleCodec`，复杂的就自己定义。

### 🔍 深入理解：什么是 Codec？

Codec 其实就是一个“翻译器”，它的作用是：把 Java 对象（Block 类实例）和数据（NBT/JSON/网络数据）互相转换。游戏要存档时，不能直接把 `new MyBlock(...)` 这个 Java 对象写进硬盘里。硬盘只认识文本或二进制。所以就得用 Codec 把方块翻译成数据（保存），下次加载时再用 Codec 把数据翻译回对象（读档）。

🚨 **如果你不写 codec**（不为你自己的方块写 codec），*这只是在你自定义的方块中有除属性以外的额外参数时或者有额外覆写逻辑函数*（重点，详见包含更多参数的方块定义），游戏在保存世界时，需要用 Codec 把方块翻译成数据。如果方块没有自己的 Codec，系统要么：

- 压根找不到 Codec → 存档报错，世界保存失败 / 崩溃。
- 强行用父类 Block 的 Codec → 只保存最基本的信息，忽略你自定义的额外参数。

**影响1**：游戏需要根据方块的“类型”来知道该用哪个逻辑。没有 Codec，你的方块在系统眼里就只是一个“普通 Block”。

**影响2**：服务器 → 客户端需要同步方块数据（尤其是有额外参数的方块）。没有 Codec，客户端根本不知道该怎么反序列化这块方块。联机玩时，方块会显示不同步：服务器上是“灯”，客户端可能看到是“空气”或“石头”。导致画面不一致，甚至直接掉线（因为数据不匹配）。客户端根本不知道该怎么反序列化这块方块。

**影响3**：其他 mod 无法正确识别或处理你的方块。可能在数据包 / 配方 / loot table 里无法引用你的方块。

⚠️ **注意我上面说的，再重申一遍**，如果你不写 codec 会强行加载父类 codec，强行加载父类的 codec 的结果就是和普通的 block 一样，在存档时：系统只知道这是一个 Block，并没有额外区分它是 MyBlock。→ 存档文件里只会写成“Block”，而不是“MyBlock”。在读档时：游戏会按照 `Block.CODEC` 去创建一个普通 Block 对象，而不是你的 MyBlock 类。结果就是，存档/读档还能跑，世界能加载。但你的方块行为退化成普通方块（比如你写的特殊 `onPlace`、`randomTick` 根本不会触发）。你注册的东西其实就变成了“披着你名字的石头”。更糟时 → 系统检测到不匹配，会把它替换成空气（相当于方块消失）。

💬 这时候你肯定会说：“如果我没有额外的参数，是不是就可以照搬原版 block 的 codec 了，或者直接不写，反正继承了 Block 还不用注册，这多简单啊”。我将会告诉你你这种想法是**完全错误的**，理论上确实可以用原版 Codec，因为它不需要保存额外的数据。存档时，它就会被当成一个“普通 Block”，不会报错。但是你自定义的，覆写的额外逻辑呢（比如你写的特殊 `onPlace`、`randomTick`）？他们也会随之消失，因为你用的原版 codec，变成原版石头类型了，当然也就和原版石头一样没有用处了。

📝 **说了这么多，到最后总结一下**，其实这个 codec 就是在创建完世界，退出，保存存档的时候会被调用，codec 会将你的石头信息保存起来，等你下次打开存档的时候再吐出来还给你。比如自定义一个 light 方块，有一个亮度等级 1 的属性，如果你用的是原版 block 的 codec 当你退出的时候，它用原版的 codec 将 light 块的信息保存，下次你再进存档的时候这个 light 就没有这个你定义的亮度等级了，你的灯全灭了 / 全统一成默认值（如果你定义的是一个箱子，里面存储的东西也将全部消失，甚至你不能打开这个箱子）。如果参数控制形状/高度，可能直接显示错乱。生成 codec 的函数叫 `simpleCodec`，函数内部是没有帮你翻译这亮度等级的功能的。

---

## ✅ 最后的最后总结一下

- **有额外逻辑但没参数**：必须写 Codec（不然逻辑根本不会生效），可以直接使用 `simpleCodec`（Mojang）生成一个 codec。因为和普通方块一样，只有功能，只需要创建一个新的方块类型，让存档能存储他的数据。
- **有额外参数但没逻辑**：也必须写 Codec（不然参数存不住，方块会丢配置）重写一个和 `simpleCodec` 一样的生成器，用于生成能存储自己定义的参数的 codec。
- **没参数也没逻辑（纯装饰砖头）**：可以用原版的 Codec，偷懒没问题。

🔮 **所以无论是哪种方块都写上 codec**，这样为未来添加其他功能做好铺垫。尽管块类型目前基本上没有使用，但随着 Mojang 继续朝着以编解码器为中心的结构发展，预计未来它将变得更加重要。

---

## 🧩 如果方块只有基础属性（注意这里没有逻辑函数）

```java
public class MyBlock extends Block {
    public static final MapCodec<MyBlock> CODEC = simpleCodec(MyBlock::new);

    public MyBlock(Properties properties) { // 没有自定义任何属性
        super(properties);
    }

    @Override
    public @NotNull MapCodec<MyBlock> codec() {
        return CODEC;
    }

    // 在注册类里面，将该类型注册到 minecraft 内部注册表中
    public static final DeferredRegister<MapCodec<? extends Block>> BLOCK_TYPES =
            DeferredRegister.create(BuiltInRegistries.BLOCK_TYPE, "yourmodid");
                                                                                                                                                                                                                        
    public static final DeferredHolder<MapCodec<? extends Block>, MapCodec<MyBlock>> MYBLOCK_CODEC =
            BLOCK_TYPES.register("my_block", () -> MyBlock.CODEC);
}
```

---

## 💡 对于具有逻辑函数和额外参数的自定义方块

我们要重新写一个 codec 生成器。你可以查看 `simpleCodec` 的源码，它返回了一个 `MapCodec<T>` 的类型，所以我们直接编写返回的逻辑函数。`simpleCodec` 只是对这段逻辑函数进行了封装，它使用了 `RecordCodecBuilder.mapCodec` 来实现，我们复现它即可：

```java
public class LampBlock extends Block {

    private final int customLightLevel; // 我们定义的额外参数

    // 所以你可以理解为：
    // simpleCodec = Mojang 给的“快捷方式”
    // mapCodec = 你手动实现，更灵活，可以加自定义参数

    public static final MapCodec<LampBlock> CODEC = RecordCodecBuilder.mapCodec(instance -> instance.group(
        // 继承 Block 必须的 properties
        BlockBehaviour.Properties.CODEC.fieldOf("properties").forGetter(block -> block.properties),    
        // 新增的亮度参数 
        Codec.INT.fieldOf("custom_light_level").forGetter(block -> block.customLightLevel) // codec.int 实际上是将 int 翻译为 minecraft 里的数据结构
    ).apply(instance, LampBlock::new)); // 生成 codec

    // 其实这个 RecordCodecBuilder 的返回值就是 simplecodec 函数的返回值，我们将仿照 simplecodec 重新写了一个 codec 生成器

    public LampBlock(BlockBehaviour.Properties properties, int customLightLevel) {
        super(properties); 
        this.customLightLevel = customLightLevel;
    }

    public int getCustomLightLevel() {
        return customLightLevel;
    }

    @Override
    public @NotNull MapCodec<LampBlock> codec() {
        return CODEC;
    }

    // 最后我们把写好的 codec 注册到 minecraft 内部注册表中，让 minecraft 知道我们添加了一个新的 codec
    public static final DeferredRegister<MapCodec<? extends Block>> REGISTRAR = DeferredRegister.create(BuiltInRegistries.BLOCK_TYPE, "yourmodid");

    public static final DeferredHolder<MapCodec<? extends Block>, MapCodec<LampBlock>> LAMPBLOCK_CODEC =
            REGISTRAR.register("lamp_block", () -> LampBlock.CODEC);
}
```

---

## 🎯 最后在注册方块的时候

```java
public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.registerBlock(
    "example_block",
    Block::new, // The factory that the properties will be passed into. 填入你的工厂
    BlockBehaviour.Properties.of() // The properties to use.
);
```

💡 **提示**：注册可以和方块自定义写一起。

---

## ⏳ 最后了解一下方块的生命周期吧（选择性观看）

### 1. 资源文件

你注册了方块，但是游戏里看不到贴图 → 因为方块的外观是通过资源系统控制的（模型 + blockstate + 纹理）。

想让方块有样子，就得提供：

- 方块模型（告诉游戏它长啥样，立方体还是别的）
- blockstate 文件（决定不同状态下用哪个模型）
- 材质贴图（png 图片）

👉 如果只注册代码，不写资源文件 → 游戏里就是紫黑格子。

### 2. 方块本质 = BlockStates

- 方块类（Block）只定义行为，不是直接用来放在世界里的。
- 游戏里真正放下去的，是 BlockState（方块的状态）。
- 比如草方块可以有雪覆盖、可以朝不同方向，就是状态不同。
- 你操作方块（放置、破坏、更新）时，底层都是操作 BlockState。

### 3. 方块放置流程

当你右键一个方块（拿着方块物品时），流程是这样的：

1. 检查能不能放（不是旁观模式、不出界等）。
2. 调用 `BlockBehaviour#canBeReplaced` → 判断当前位置能不能被替换（比如草丛可以被覆盖）。
3. 调用 `Block#getStateForPlacement` → 决定放下去的方块状态（比如朝向、半砖上下）。
4. 调用 `BlockBehaviour#canSurvive` → 看方块能不能存在（比如沙子要有方块支撑）。
5. 真正调用 `Level#setBlock` 把方块放进世界。
6. 触发 `onPlace`、`setPlacedBy` 等钩子。

👉 这些方法你都可以在自定义方块里覆盖，来改放置逻辑。

### 4. 方块破坏流程

分三步：

1. **启动阶段**（点一下左键）：触发事件、调用 attack。
2. **挖掘阶段**（长按左键）：每 tick 调用 `getDestroyProgress`，更新破坏进度（裂纹动画）。
3. **实际破坏阶段**：
   - 检查能不能掉落东西（比如正确工具）。
   - 调用 `playerWillDestroy`、`onDestroyedByPlayer`、`dropResources` 等。
   - 把方块替换成空气。
   - 触发掉落、经验、事件。

👉 如果你想做特殊掉落逻辑，就要改 `dropResources` 或 `playerDestroy`。

### 5. 方块的 Tick（更新机制）

方块不是静态的，可以有“时间逻辑”。主要有几种：

- **服务器 Tick**
  - 调用 `BlockBehaviour#tick`，你可以自己调度（比如红石更新、植物生长）。
- **客户端 Tick**
  - 调用 `Block#animateTick`，只在客户端执行（比如火把冒烟粒子）。
- **天气 Tick**
  - 调用 `handlePrecipitation`，下雨/下雪时触发（比如炼药锅接水）。
- **随机 Tick**
  - 开启 `randomTicks()` 属性后，每 tick 游戏会随机挑一些方块调用 `randomTick`。
  - 应用：植物生长、冰融化、铜氧化。
  - 这就是植物/矿石之类“自己变化”的机制。

### 6. 挖掘速度（矿物难度）

游戏算挖掘速度时，会考虑：

- 方块硬度（destroyTime）。
- 工具是否合适（镐/斧/铲）。
- 附魔（效率、海豚、急迫）。
- 状态（潜水、飞行会减速）。
- 玩家属性。
- 事件 BreakSpeed（模组可改）。

👉 你可以在自定义方块里覆盖 `getDestroyProgress` 来改破坏速度。

---

## ⚡ 总结要点

- 方块类 ≠ 方块 → 世界里的是 BlockState。
- 放置流程：检查能放 → 决定状态 → 放置 → 触发事件。
- 破坏流程：启动 → 挖掘 → 破坏 → 掉落。
- Tick 类型多 → 随机 Tick（植物生长）、客户端 Tick（粒子）、天气 Tick（雨水）等。
- 挖掘速度受工具、属性、状态影响。
- 资源系统必不可少，负责外观。

---

## 🧭 BlockStates

**什么是 BlockState？**

- Block 是“方块的类型定义”。
- BlockState 是“方块的具体状态”。
- 举例：小麦 = 一个 Block；小麦 1 阶段、2 阶段、成熟 = 不同的 BlockState。
- 👉 世界里放置的不是 Block，而是 BlockState。

之前说过了，如果状态比较少可以用 blockstate，如果特别多，例如可以交互，产生其他效果，这可以使用方块实体。

我们来看一段举例代码就知道了：

```java
public class EndPortalFrameBlock extends Block { 

    public static final EnumProperty<Direction> FACING = BlockStateProperties.FACING; // 这里使用做好了的朝向类

    public static final BooleanProperty EYE = BlockStateProperties.EYE; // 这个也是内置的方块状态属性，可以尝试自己构造属性，但太过于复杂，我建议还是循序渐进

    public EndPortalFrameBlock(BlockBehaviour.Properties pProperties) {
        super(pProperties);
                
        this.registerDefaultState(stateDefinition.any()
                .setValue(FACING, Direction.NORTH) // 设置方块的初始朝向
                .setValue(EYE, false)
        );
    }

	 // 注册属性到 BlockState 系统
    @Override
    protected void createBlockStateDefinition(StateDefinition.Builder<Block, BlockState> pBuilder) {
        
        pBuilder.add(FACING, EYE);
    }

    @Override
    @Nullable
    public BlockState getStateForPlacement(BlockPlaceContext pContext) {
        // 这里可以设置方块朝向规则，比如朝着玩家方向
    }
}
```

### 🔧 使用 BlockState

- 获取默认状态：`BlockState state = myBlock.defaultBlockState();`
- 获取属性值：`Direction dir = state.getValue(FACING);`
- 修改属性（返回新的 BlockState）：`state = state.setValue(FACING, Direction.SOUTH);`

⚠️ **注意：BlockState 是不可变的！所以必须保存返回的新值。**

- 在世界中获取状态：`BlockState state = level.getBlockState(pos);`
- 在世界中设置状态：`level.setBlock(pos, state, Block.UPDATE_ALL);`

---

🧱 **方块的知识结束了，进入下一个章节吧！**
```