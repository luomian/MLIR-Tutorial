# MemRefType 分析

源文件：`third_party/llvm-project/mlir/include/mlir/IR/BuiltinTypes.td`（第 425-699 行）

## 1. MemRefType 是什么

**对内存区域的引用**——可理解为"有形状的缓冲区指针"。

```mlir
memref<维度列表 x 元素类型 (, layout)? (, memory_space)?>
```

**核心四要素：**

```tablegen
let parameters = (ins
  ArrayRefParameter<"int64_t">:$shape,        // 形状
  "Type":$elementType,                         // 元素类型
  "MemRefLayoutAttrInterface":$layout,         // 内存布局
  "Attribute":$memorySpace                      // 地址空间
);
```

**与 RankedTensorType 的关键区别：**

| | RankedTensorType | MemRefType |
|------|-----------------|------------|
| 语义 | 值类型（不可变数值容器） | 引用类型（可读写内存区域） |
| 取数据指针 | **不可以** | **可以**（通过 `alloc`/`load`/`store` 操作） |
| 多出什么 | `encoding` 属性 | `layout` + `memorySpace` |
| 基类 | `TensorType` | `BaseMemRefType` |

---

## 2. 三个 builder 重载

```tablegen
let builders = [
    TypeBuilderWithInferredContext<(ins
      "ArrayRef<int64_t>":$shape, "Type":$elementType,
      CArg<"MemRefLayoutAttrInterface", "{}">:$layout,
      CArg<"Attribute", "{}">:$memorySpace)>,
    TypeBuilderWithInferredContext<(ins
      "ArrayRef<int64_t>":$shape, "Type":$elementType,
      CArg<"AffineMap">:$map,
      CArg<"Attribute", "{}">:$memorySpace)>,
    /// [deprecated]
    TypeBuilderWithInferredContext<(ins
      "ArrayRef<int64_t>":$shape, "Type":$elementType,
      "AffineMap":$map,
      "unsigned":$memorySpaceInd)>
  ];
```

**1. 现代版（推荐）** — layout 用属性接口，memory space 用 Attribute：
```cpp
static MemRefType get(ArrayRef<int64_t> shape, Type elementType,
                      MemRefLayoutAttrInterface layout = {},
                      Attribute memorySpace = {});
```
```cpp
MemRefType::get({4, 8}, f32);                                   // 最简
MemRefType::get({4, 8}, f32, StridedLayoutAttr::get(ctx, ...)); // 属性版
```

**2. AffineMap 便捷版** — 直接传 `AffineMap`，内部自动 wrap 为属性：
```cpp
MemRefType::get({4, 8}, f32, affineMap);
```

**3. 废弃版** — memory space 用 `unsigned` 整数，向后兼容旧 API。

---

## 3. 使用举例

```cpp
// 默认行优先，不指定 layout
auto memref = MemRefType::get({3, 4}, f32Type);
// IR: memref<3x4xf32>

// 用 AffineMap 指定列优先
auto colMap = AffineMap::get(2, 0, {getAffineDimExpr(1, ctx), getAffineDimExpr(0, ctx)});
auto colMemref = MemRefType::get({3, 4}, f32Type, colMap);
// IR: memref<3x4xf32, affine_map<(d0, d1) -> (d1, d0)>>

// stride 布局
auto layoutAttr = StridedLayoutAttr::get(ctx, /*offset=*/0, /*strides=*/{4, 1});
auto stridedMemref = MemRefType::get({3, 4}, f32Type, layoutAttr);
// IR: memref<3x4xf32, strided<[4, 1]>>

// 带 memory space（GPU 区分 global / shared memory）
auto gpuMemref = MemRefType::get({3, 4}, f32Type, {}, IntegerAttr::get(i32, 1));
// IR: memref<3x4xf32, 1>
```

---

## 4. AffineMap 是什么

**仿射映射**，描述"多维索引 → 一维内存地址"的线性变换关系。

格式：
```
affine_map<(d0, d1, ... dN)[s0, s1, ...] → (expr0, expr1, ... exprM)>
```

- **dN** — 维度变量（循环中的迭代变量，每次迭代会变）
- **sN** — 符号变量（运行时常量，本次调用中不变，编译时未知具体值）

```mlir
// 列优先：(i, j) → (j, i)  — 交换维度
affine_map<(d0, d1) -> (d1, d0)>

// 带偏移：(i, j)[offset] → (i + offset, j)  — 整体偏移 offset 行
affine_map<(d0, d1)[s0] -> (d0 + s0, d1)>
```

### 具体例子：列优先

```cpp
auto colMap = AffineMap::get(
    2,                    // 2 个维度变量: d0, d1
    0,                    // 0 个符号变量
    {
        getAffineDimExpr(1, ctx),  // 结果第 1 维 = d1
        getAffineDimExpr(0, ctx)   // 结果第 2 维 = d0
    }
);
// 生成: affine_map<(d0, d1) -> (d1, d0)>
// 效果: 访问 memref[a][b] 时实际按 [b][a] 走内存
```

### 具体例子：带符号偏移

```cpp
auto map = AffineMap::get(
    2,                    // 2 个维度变量: d0, d1
    1,                    // 1 个符号变量: s0
    {
        getAffineDimExpr(0, ctx) + getAffineSymbolExpr(0, ctx),
        //    ↑ d0                    ↑ s0
        //    结果第 1 维 = d0 + s0

        getAffineDimExpr(1, ctx),
        //    ↑ d1
        //    结果第 2 维 = d1
    }
);
// 生成: affine_map<(d0, d1)[s0] -> (d0 + s0, d1)>
// 效果: 逻辑上访问 (i, j)，底层内存中去 (i + s0, j) 找
```

| C++ 表达式 | IR 形式 | 含义 |
|-----------|---------|------|
| `getAffineDimExpr(0, ctx)` | `d0` | 输入的第 0 维（行号 i） |
| `getAffineDimExpr(1, ctx)` | `d1` | 输入的第 1 维（列号 j） |
| `getAffineSymbolExpr(0, ctx)` | `s0` | 运行时确定的偏移量 |
| `d0 + s0` | `d0 + s0` | 实际行地址 = 逻辑行号 + 偏移 |

使用 IR：
```mlir
// s0 = 5，实际访问从第 5 行开始，前面 5 行被跳过
%A = alloc()[5] : memref<16x32xf32, affine_map<(d0, d1)[s0] -> (d0 + s0, d1)>>
```
