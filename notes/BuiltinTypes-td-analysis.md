# BuiltinTypes.td 分析：以 IntegerType 为例

> 源文件：`third_party/llvm-project/mlir/include/mlir/IR/BuiltinTypes.td`

## IntegerType 的 TableGen 定义

```tablegen
def Builtin_Integer : Builtin_Type<"Integer", "integer"> {
```

- `def Builtin_Integer` — 定义一个名为 `Builtin_Integer` 的 TableGen 记录
- `Builtin_Type<"Integer", "integer">` — 基于 `Builtin_Type` 基类，C++ 类名 = `Integer`，MLIR 语法中的关键字 = `integer`

---

### 1. parameters — 存储参数（类型的唯一标识）

```tablegen
let parameters = (ins "unsigned":$width, "SignednessSemantics":$signedness);
```

定义该类型的**存储参数**：
- `width` — 无符号整数，表示位宽
- `signedness` — 有符号/无符号/无符号语义

这些参数**即唯一标识**。MLIR 的类型系统是**唯一化的**——相同参数只会有一个实例：

- `IntegerType::get(ctx, 32, Signless)` → `i32`
- `IntegerType::get(ctx, 8, Signed)` → `si8`

> `ctx` 即 `MLIRContext*`，MLIR 中所有 IR 对象（Type、Operation、Attribute 等）都绑定到一个 Context。类型实例在 Context 内是唯一化的，所以创建时始终需要传入 `ctx`。

`i32` 和 `si32` 是不同的类型，正是因为这个 `signedness` 参数不同。每次写 `i32`，TableGen 生成的代码会在 Context 中查找或创建唯一的 Type 实例，`$width` 和 `$signedness` 就是哈希/比较的依据。

---

### 2. builders — 构造器

```tablegen
let builders = [
  TypeBuilder<(ins "unsigned":$width,
                   CArg<"SignednessSemantics", "Signless">:$signedness)>
];
```

- `CArg<"Signless">` 表示 `signedness` 有默认值 `Signless`
- `IntegerType::get(ctx, 32)` → 等价于 `i32`（默认 Signless）
- `IntegerType::get(ctx, 32, Signed)` → 等价于 `si32`

---

### 3. extraClassDeclaration — 手写 C++ 嵌入

```tablegen
let extraClassDeclaration = [{
  enum SignednessSemantics : uint32_t { Signless, Signed, Unsigned };
  bool isSignless() const { return getSignedness() == Signless; }
  bool isSigned() const   { return getSignedness() == Signed; }
  bool isUnsigned() const { return getSignedness() == Unsigned; }
}];
```

这段会被直接粘贴到生成的 C++ 类体中，提供三个便捷判断方法：`isSignless()`、`isSigned()`、`isUnsigned()`。

---

## 对照：.td 定义 → 生成效果

| .td 中写 | 生成的 C++ 效果 |
|----------|----------------|
| `parameters (ins "unsigned":$width, ...)` | `unsigned getWidth() const;` 存储/比较/哈希自动生成 |
| `builders [TypeBuilder<...>]` | `static IntegerType get(MLIRContext*, unsigned, ...)` |
| `extraClassDeclaration [{...}]` | 直接粘贴到类体中 |
| `let typeName = "builtin.integer"` | MLIR 语法中写作 `i32`、`si8` 等时解析为该类型 |

## 一句话总结

`.td` 中 `parameters` 决定"这个类型的唯一标识是什么"，`builders` 决定"怎么创建它"，最终 TableGen 自动生成**存储、解析、打印、哈希比较**等全套样板代码。

---

## Q: Type 是绑定在 Dialect 上的吗？

**答：** 对。MLIR 中绝大部分 Type 都注册在某个 Dialect 上。比如 `i32` 属于 Builtin Dialect，`tensor<f32>` 属于 Tensor Dialect。

在 `initialize()` 里通过 `addTypes<MyCustomType>()` 注册。当一个 Type 被创建时，存储在所属 Dialect 的类型表里，可通过 `type.getDialect()` 查到来源。这就是为什么写 `.td` 时需要用 `TypeDef<MyDialect, ...>` 明确指定所属方言。

## Q: 不同 Dialect 定义了相同的 Type，会冲突吗？

**答：** 不会。MLIR 用**方言命名空间**隔离类型。完整标识是 `方言命名空间.类型助记符`，核心区分靠 **TypeID**（由 `MLIR_DEFINE_EXPLICIT_TYPE_ID` 生成）。

即使两个方言都定义 `"mytype"`：
- Dialect A → TypeID = 0x1234...
- Dialect B → TypeID = 0x5678...

TypeID 完全不同，cast/类型检查不会认错。IR 文本中也不混淆，因为带方言前缀：`"a.mytype"` vs `"b.mytype"`。

## Q: `IntegerType::get(ctx, 8, Signed)` 没有明确显示是哪个方言

**答：** IntegerType 属于 **Builtin Dialect**，而 Builtin 类型有特权——它们的 `mnemonic` 在 IR 语法中**不需要**带方言前缀。

```tablegen
let typeName = "builtin.integer";  // 内部完整名
let mnemonic = ?;                  // Builtin 类型不设助记符，走特化语法
```

Builtin 类型直接用缩写语法（`i32`、`f32`、`si8`），解析器/打印器对它有硬编码支持。自定义方言的类型则必须写成 `!mydialect.mytype<...>` 带前缀形式。

## Q: mnemonic 是什么？

**答：** 在 MLIR IR 文本语法中，助记符是类型或操作在方言命名空间下的**简短标识名**。

| | `.td` 定义 | IR 文本写法 |
|--|-----------|-------------|
| 自定义类型 | `let mnemonic = "mytype";` | `!mydialect.mytype` |
| 自定义操作 | `let mnemonic = "add";` | `mydialect.add` |
| Builtin Integer | `let mnemonic = ?;`（不设） | `i32`、`si8`（走特化语法） |

Builtin 设 `?` 是主动放弃 `!dialect.mnemonic` 标准格式，改由自己手写 parse/print 逻辑实现特化语法，因为 `i32`、`f32` 这种写法更简洁自然。

## Q: Builtin 是什么含义？

**答：** Builtin 是 MLIR 的**内置核心方言**，定义了 IR 的"基础词汇"——没有它，最基本的 IR 结构都无法表示。

包含 IR 骨架所需的一切：
- **Operation**：`module`、`func`、`unrealized_conversion_cast`
- **Type**：`i32`、`f64`、`index`、`none`、`memref<...>`、`tensor<...>` 等
- **Attribute**：`UnitAttr`、`IntegerAttr`、`DenseElementsAttr` 等

Builtin Dialect 是**始终加载**的，不需要声明依赖或手动加载。它相当于 MLIR 这门"语言"的内置关键字和基础类型系统。

## Q: builders 的作用是什么？

**答：** `builders` 定义的是**静态工厂方法**，即 `get(ctx, ...)` 的签名和实现。因为 MLIR 的类型/操作采用唯一化存储和值语义，不能用普通构造函数直接 `new`，必须通过工厂方法从 Context 中获取或创建。

- `parameters` 定义"存什么"
- `builders` 定义"怎么把外面传的参数转化为内部存储参数，再返回一个指向唯一实例的句柄"

## Q: 什么是静态工厂方法？

**答：** 一种创建对象的设计模式——不用 `new`/构造函数创建对象，而是通过类的静态方法返回实例。

```cpp
// 不用工厂：直接 new
IntegerType* t = new IntegerType(ctx, 32, Signless);  // MLIR 不允许这样做

// 静态工厂方法：
IntegerType t = IntegerType::get(ctx, 32, Signless);  // 返回唯一化实例
```

为什么 MLIR 用它而不是构造函数：

1. **唯一化（intern）** — 相同参数只存在一份实例，`get()` 内部先查已有实例，有就返回，没有再创建
2. **隐藏实现** — 返回类型可以是父类引用（`Type`），内部具体是哪个子类对调用者透明
3. **有意义的命名** — 语义比构造参数更清晰

## Q: 那为什么不直接叫单例模式？

**答：** 不是单例。单例是**全局只有唯一一个实例**，而 MLIR 的类型是**同参数只有唯一实例**——这是**享元模式（Flyweight）**。

```cpp
auto a = IntegerType::get(ctx, 32, Signless);
auto b = IntegerType::get(ctx, 32, Signless);
auto c = IntegerType::get(ctx, 8, Signed);

// a == b   ✓  相同参数，同一个实例
// a != c   ✗  不同参数，不同实例
```

实现原理：`get()` 内部维护一个按参数值做 key 的哈希表：
1. 构造参数元组 → 查表
2. 命中 → 返回已有实例
3. 未命中 → 新建实例 → 插入表 → 返回

单例是"不管什么参数，永远同一个"，享元是"按参数去重，同参则同实例"。

## Q: 哪里表明用特化语法代替了标准格式？

**答：** 在 `third_party/llvm-project/mlir/lib/IR/AsmPrinter.cpp:2571-2686`，所有 Builtin 类型的打印都是**硬编码**的，根本没走 `!dialect.mnemonic` 的标准路径：

```cpp
// 打印 IntegerType 的特化逻辑（第 2591-2597 行）
.Case<IntegerType>([&](IntegerType integerTy) {
    if (integerTy.isSigned())
        os << 's';                          // 输出 "si32" 中的 "s"
    else if (integerTy.isUnsigned())
        os << 'u';
    os << 'i' << integerTy.getWidth();      // 输出 "i32"
})

// Float32Type（第 2587 行）— 直接输出 "f32"
.Case<Float32Type>([&](Type) { os << "f32"; })
```

标准路径在 **第 2748 行** `printDialectType`：
```cpp
// 非 Builtin 类型的打印：!dialect_namespace.助记符
printDialectSymbol(os, "!", dialect.getNamespace(), typeName);
```

Builtin 类型在 `TypeSwitch` 的各个 `Case` 分支中被提前拦截，根本不会落到 `Default`，所以 `.td` 中 `let mnemonic = ?;` 并不会被走到。

## Q: `genStorageClass = 0`, `skipDefaultBuilders = 1`, `genVerifyDecl = 1` 的含义？

源码注释（`AttrTypeBase.td:165-240`）：

| 标志 | 默认值 | 含义 |
|------|--------|------|
| `genStorageClass` | `1` | 是否自动生成存储类。`0` = 手工写存储，`1` = 自动生成 |
| `skipDefaultBuilders` | `0` | 是否跳过默认 builder。`1` = 只用自定义 builder（如带默认值） |
| `genVerifyDecl` | `0` | 是否生成验证方法。`1` = 生成 `verify()` 用于校验参数合法性 |

以 IntegerType 为例：
- `genStorageClass = 0`：width 和 signedness 被手工打包到单个 `uint32_t` 节省内存，默认生成的两字段结构不够高效
- `skipDefaultBuilders = 1`：默认 builder 的 signedness 无默认值，自定义提供 `= Signless`
- `genVerifyDecl = 1`：需要校验 width 不为 0 且不超过最大值

三者做同一件事：**"默认生成的不够用，我要手写/自定义"**。

## Q: `genStorageClass = 1` 会怎样？

**答：** 会自动生成存储类，按 `parameters` 字段逐个对应为成员变量，并自动实现构造函数、`==`、`hash` 等全套逻辑：

```cpp
// genStorageClass = 1 时自动生成
struct IntegerTypeStorage : public TypeStorage {
  unsigned width;
  SignednessSemantics signedness;  // 每个 parameter 一个字段
  // 自动生成：构造函数、operator==、hashKey 用于唯一化查表
};
```

绝大多数自定义类型用 `1`（默认值）就够了，不需操心。
