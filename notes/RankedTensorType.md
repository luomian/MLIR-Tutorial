# ArrayRefParameter 与 Ref 理解

## 1. RankedTensorType 具体代码分析

源文件：`third_party/llvm-project/mlir/include/mlir/IR/BuiltinTypes.td`（第 775-880 行）

### .td 定义

```tablegen
def Builtin_RankedTensor : Builtin_Type<"RankedTensor", "tensor", [
    ShapedTypeInterface, ValueSemantics
  ], "TensorType"> {
```

- C++ 类名 `RankedTensor`，词根关键词 `tensor`
- 继承了 `ShapedTypeInterface`（多维形状能力）+ `ValueSemantics`（值语义 trait）
- 基类 `TensorType`（与 `UnrankedTensorType` 共享的抽象基类）

### parameters — 三个存储参数

```tablegen
let parameters = (ins
  ArrayRefParameter<"int64_t">:$shape,
  "Type":$elementType,
  "Attribute":$encoding
);
```

- `shape` — 维度数组，如 `{4, 8}` 表示 `4x8`，`{-1}` 表示动态 `?`
- `elementType` — 元素类型，如 `f32`
- `encoding` — 可选属性，如稀疏张量的编码

这三个值**共同决定类型唯一性**：`tensor<4x8xf32>` 和 `tensor<8x4xf32>` 是不同的类型。

### builders — 工厂方法

```tablegen
let builders = [
  TypeBuilderWithInferredContext<(ins
    "ArrayRef<int64_t>":$shape,
    "Type":$elementType,
    CArg<"Attribute", "{}">:$encoding
  ), [{
    return $_get(elementType.getContext(), shape, elementType, encoding);
  }]>
];
```

生成的工厂方法：
```cpp
static RankedTensorType get(ArrayRef<int64_t> shape,
                            Type elementType,
                            Attribute encoding = {});  // encoding 默认空
```

`TypeBuilderWithInferredContext` 表示 context 不需要显式传入——直接从 `elementType.getContext()` 推断。

### extraClassDeclaration — 手写方法

```tablegen
let extraClassDeclaration = [{
  RankedTensorType clone(::mlir::Type elementType) {
    return ::llvm::cast<RankedTensorType>(cloneWith(getShape(), elementType));
  }
}];
```

保持形状不变，更换元素类型：
```cpp
auto f32Tensor = RankedTensorType::get({4, 8}, f32Type);
auto i32Tensor = f32Tensor.clone(i32Type);  // tensor<4x8xi32>
```

### .td → 生成效果对照

| .td | 生成 C++ |
|-----|---------|
| `parameters shape/elementType/encoding` | `getShape()`, `getElementType()`, `getEncoding()` + 存储/哈希/判等 |
| `builders TypeBuilderWithInferredContext` | `static RankedTensorType get(shape, elementType, encoding={})` |
| `traits [ShapedTypeInterface, ValueSemantics]` | 获得 `getRank()`, `getDimSize()` 等接口全部实现 |
| `extraClassDeclaration clone` | 生成的类中直接多一个 `clone()` 方法 |

---

## 2. ArrayRefParameter 怎么理解

源文件：`third_party/llvm-project/mlir/include/mlir/IR/AttrTypeBase.td`（第 384-388 行）

```tablegen
class ArrayRefParameter<string arrayOf, string desc = ""> :
    AttrOrTypeParameter<"::llvm::ArrayRef<" # arrayOf # ">", desc> {
  let allocator = [{$_dst = $_allocator.copyInto($_self);}];
  let cppStorageType = "::llvm::SmallVector<" # arrayOf # ">";
}
```

`ArrayRefParameter` 是 TableGen 的参数包装类，解决一个矛盾：

| 层 | 类型 | 说明 |
|----|------|------|
| **用户侧**（工厂方法入参） | `llvm::ArrayRef<int64_t>` | 轻量级、非拥有的数组视图，方便调用者传参 |
| **存储侧**（实际存储） | `llvm::SmallVector<int64_t>` | 拥有数据所有权，保证类型实例生命周期内数据有效 |

关键在 `allocator` 那一行：当创建存储实例时，自动生成代码把外部传入的 `ArrayRef` 拷贝到内部的 `SmallVector` 中。调用者传临时 `{4, 8}` 也无所谓——数据已被内部持有，不会悬空。

这也解释了为什么 `getShape()` 返回的是 `ArrayRef<int64_t>` 而不是 `SmallVector`——存储侧持有数据，暴露侧只是指向它的轻量视图。

---

## 3. Ref 是什么含义

`ref` 是 **reference** 的缩写，在 LLVM 语境中特指一种**不拥有数据的轻量级视图**。

| 类型 | 含义 | 拥有数据？ |
|------|------|-----------|
| `llvm::ArrayRef<T>` | 数组的只读视图（指针+长度） | 否 |
| `llvm::StringRef` | 字符串的只读视图（指针+长度） | 否 |
| `llvm::SmallVector<T>` | 小容量优化的可增长数组 | **是** |

以 `ArrayRef<int64_t>` 为例：
```cpp
// ArrayRef 只是一个 {指针, 长度} 对，不持有数据
ArrayRef<int64_t> shape = tensor.getShape();  // shape 指向内部存储的数据，不拷贝
// 等价于手工构造
auto shape = ArrayRef<int64_t>(dataPtr, size);
```

与 C++ 标准库对比：
- `ArrayRef` ≈ 不可变的 `std::span`（C++20）
- `StringRef` ≈ `std::string_view`

**为什么叫 ref？** 因为它只是"引用/指向"别人已有的数据，自己不分配、不释放。一旦被引用的原始数据销毁，它就悬空了。这就是为什么存储侧必须用 `SmallVector` 真正持有数据——外面的 `ArrayRef` 只是个借用。

---

## 4. encoding 参数是什么

`"Attribute":$encoding` — `encoding` 的类型是 `Attribute`，在 `ArrayRefParameter` 中是一个参数名。

为什么 tensor 需要一个 attribute？举例：

```mlir
// 普通密集存储（无 encoding）
tensor<100x100xf64>

// 稀疏存储（encoding 描述压缩格式）
tensor<100x100xf64, #sparse_tensor.encoding<{
  dimLevelType = [ "compressed", "compressed" ]
}>
```

形状和元素类型完全相同（都是 100x100 的 f64），但一个对应 10000 个浮点数的连续数组，另一个可能只存 5 个非零元素。**`encoding` 属性就是用来区分这种"同形同元素但不同存储"的情况。** encoding 不同，类型就不同。

---

## 5. builder 与 TypeBuilderWithInferredContext

```tablegen
let builders = [
    TypeBuilderWithInferredContext<(ins
      "ArrayRef<int64_t>":$shape,              // 必选参数 #1
      "Type":$elementType,                     // 必选参数 #2
      CArg<"Attribute", "{}">:$encoding        // 可选参数 #3，默认 {}
  ), [{
      return $_get(elementType.getContext(),    // context 从 elementType 推断
                  shape, elementType, encoding);
  }]>
];
```

与 IntegerType `TypeBuilder` 的对比：

| | IntegerType | RankedTensorType |
|------|-------------|------------------|
| 基类 | `TypeBuilder` | `TypeBuilderWithInferredContext` |
| context 传法 | 显式第一参数 `ctx` | **不传**，从 `elementType.getContext()` 自动取 |
| 生成签名 | `get(ctx, width, signedness=Signless)` | `get(shape, elementType, encoding={})` |

为什么能推断？因为 MLIR 的所有 Type/Attribute 都绑定在一个 Context 下——给了 `elementType`，就能通过它拿到属于哪个 Context，省得调用者每次写 `ctx`。

最终调用：
```cpp
auto f32 = Float32Type::get(ctx);
auto t = RankedTensorType::get({4, 8}, f32);           // encoding 默认 {}
auto s = RankedTensorType::get({100, 100}, f32, enc);  // 显式传入稀疏编码
```
