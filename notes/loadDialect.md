# loadDialect 理解笔记

## 代码

```cpp
getContext()->loadDialect<::mlir::tensor::TensorDialect>();
```

## Q: 这行代码应该怎么理解？

**答：** 在当前 `MLIRContext` 中加载（注册）`TensorDialect`，确保 NorthStar 初始化时它的依赖方言已经可用。

## Q: 不太理解上面这句话

**答（类比理解）：**

- **`MLIRContext`** 就像一个仓库/字典，存储了"这门语言有哪些合法词汇"——有哪些数据类型（Type）、有哪些操作（Operation）、有哪些方言（Dialect）。

- **`loadDialect<TensorDialect>()`** 就是在说："把 Tensor 方言定义的词汇（比如 `tensor<...>` 类型）全部登入到这个仓库里。"

- **为什么需要这行代码：** NorthStar 方言依赖于 Tensor 方言（例如它的操作可能用 `tensor<f32>` 作为参数类型）。如果在初始化 NorthStar 时 Tensor 还没有被注册，MLIR 就不知道 `tensor<f32>` 是什么，就会报错。

- **一句话总结：** 在 NorthStar 初始化之前，保证 Tensor 方言的"词汇表"已经被放进 Context 了。

## 相关知识

- `getContext()` — 返回当前方言关联的 `MLIRContext*`
- `loadDialect<T>()` — 向该 Context 请求加载指定方言（若已加载则跳过，线程安全）
- 这行代码由 TableGen 根据 `.td` 文件中的 `let dependentDialects = ["tensor"];` 自动生成
- `initialize()` 在构造函数末尾被调用，是构造流程的一部分；NorthStar 自己的 Operation/Type/Attribute 在 `initialize()` 中注册

## Q: `getContext` 是哪里来的？

**答：** 是继承来的。`NorthStarDialect` 继承自 `mlir::Dialect`，`getContext()` 定义在基类中：

`third_party/llvm-project/mlir/include/mlir/IR/Dialect.h:52`

```cpp
MLIRContext *getContext() const { return context; }
```

它返回构造时传入的那个 `MLIRContext*`。

## Q: `auto dialect = context.getOrLoadDialect<NorthStarDialect>();` 实际调用了构造函数吗？

**答：** 是的（首次调用时）。`getOrLoadDialect` 内部逻辑：

- **已加载** → 直接返回已有实例
- **未加载** → 调用 `NorthStarDialect` 的构造函数（即 TableGen 自动生成的那个），进而触发：
  1. 父类 `Dialect` 初始化（命名空间、TypeID）
  2. `TensorDialect` 的加载
  3. `initialize()` — 注册 Operation/Type/Attribute

所以**首次调用** `getOrLoadDialect` = 构造对象 → `initialize()` → 注册进 Context → 返回指针。
