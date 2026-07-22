# NorthStar_Type 与 TypeDef 分析

源文件：`3-define_type/include/Dialect/NorthStar/IR/NorthStarTypes.td`

## 1. NorthStar_Type 逐行拆解

```tablegen
class NorthStar_Type<string name, string typeMnemonic, list<Trait> traits = [],
                   string baseCppClass = "::mlir::Type">
    : TypeDef<NorthStar_Dialect, name, traits, baseCppClass> {
  let mnemonic = typeMnemonic;
  let typeName = dialect.name # "." # typeMnemonic;
}
```

### 参数列表（即 TableGen 类的"构造参数"）

| 参数 | 类型 | 默认值 | 含义 |
|------|------|--------|------|
| `name` | `string` | 必选 | C++ 类名 |
| `typeMnemonic` | `string` | 必选 | IR 文本助记符 |
| `traits` | `list<Trait>` | `[]` | 附加能力标签，默认无 |
| `baseCppClass` | `string` | `"::mlir::Type"` | C++ 继承基类 |

### 每行的含义

```tablegen
: TypeDef<NorthStar_Dialect, name, traits, baseCppClass> {
```
将参数传给 `TypeDef`——TableGen 的核心代码生成引擎。固定死了 `NorthStar_Dialect`，所以后续类型自动属于该方言，无需每次手写。

```tablegen
let mnemonic = typeMnemonic;
```
设置 IR 文本助记符，使类型能以 `!north_star.foo` 格式被解析和打印。

```tablegen
let typeName = dialect.name # "." # typeMnemonic;
```
拼接内部唯一名 `"north_star.foo"`。`dialect.name` 来自 `NorthStar_Dialect` 定义，`#` 是 TableGen 字符串拼接符。

### "调用者"是谁

后续定义具体类型时的 `def` 语句：

```tablegen
def Foo : NorthStar_Type<"Foo", "foo"> { }
//                       ↑ "Foo" 是传给 name 的实参
//                             ↑ "foo" 是传给 typeMnemonic 的实参
```

### name vs dialect.name

| 出现位置 | 是谁的 name | 值 |
|---------|------------|-----|
| `string name` | 参数名，恰好叫 name | 调用者传的，如 `"Foo"` |
| `dialect.name` | 方言对象的字段 | 方言命名空间，固定 `"north_star"` |

两个 `name` 只是碰巧同名，通过词法作用域区分。

---

## 2. NorthStar_Type 的作用

一个**代码生成辅助基类**，做三件事：

1. **固定方言** — 所有 NorthStar 类型不用手写 `TypeDef<NorthStar_Dialect, ...>`
2. **统一命名规则** — `typeName` 自动拼接为 `"north_star.xxx"`
3. **减少参数** — 四参打包，后续只需 2 个必选 + 默认参数

```tablegen
// 不用基类（啰嗦）
def Foo : TypeDef<NorthStar_Dialect, "Foo", [], "::mlir::Type"> {
  let mnemonic = "foo";
  let typeName = "north_star.foo";
}

// 用基类（简洁）
def Foo : NorthStar_Type<"Foo", "foo"> { }
```

完全是给开发者省事的——把重复代码抽出成父类。

---

## 3. TypeDef 是什么

`TypeDef` 不是 C++ 的 `typedef`，是 TableGen 的**核心类**（`AttrTypeBase.td:288`）：

```tablegen
class TypeDef<Dialect dialect, string name, list<Trait> traits = [],
              string baseCppClass = "::mlir::Type">
```

负责把 `.td` 中的类型声明**翻译成 C++ 代码**。四个参数进去，TableGen 生成：

| 输入 | 输出 |
|------|------|
| `dialect` | 类型注册到哪个 dialect |
| `name` | C++ 类名 |
| `traits` | trait 方法的自动实现 |
| `baseCppClass` | C++ 继承链的父类 |

它是整个 MLIR 类型系统的代码生成引擎。类似的还有 `AttrDef`（定义 Attribute）和 `Op`（定义 Operation）。

---

## 4. baseCppClass 的作用

决定**生成的 C++ 类继承链**。

```tablegen
def Foo : NorthStar_Type<"Foo", "foo"> { }
// baseCppClass 默认 "::mlir::Type"
// 生成: class Foo : public ::mlir::Type { ... }
```

如果想共享手写父类：

```cpp
// 手写基类
class NorthStarValueType : public ::mlir::Type::TypeBase<...> {
public:
  int getBitWidth() const { ... }
};
```

```tablegen
def MyInt : NorthStar_Type<"MyInt", "my_int", [], "NorthStarValueType"> {
// 生成: class MyInt : public NorthStarValueType { ... }
```

这样 `MyInt`、`MyFloat` 都继承 `NorthStarValueType`，共享 `getBitWidth()` 等通用方法。

---

## 5. traits 是什么

附加在类型上的**能力标签**。声明"这个类型具备某种行为"：

```tablegen
def Foo : NorthStar_Type<"Foo", "foo"> { }                           // 无 trait，普通类型
def Bar : NorthStar_Type<"Bar", "bar", [ShapedTypeInterface]> { }    // 有 trait，获得 getRank() 等方法
```

| Trait | 效果 |
|-------|------|
| `ShapedTypeInterface` | 自动获得 `getRank()`、`getDimSize()`、`getNumElements()` 等 |
| `ValueSemantics` | 标记为值语义（tensor vs memref 的关键区别） |

TableGen 会自动为带 trait 的类型生成对应接口方法实现。类似 Java 的 `implements`。

|---

## 6. `let typeName = dialect.name # "." # typeMnemonic;` 详细解释

```tablegen
let typeName = dialect.name # "." # typeMnemonic;
//  ↓            ↓              ↓        ↓
//  字段名     方言对象的      拼接符   方法参数
//  (设为)      name字段    (拼接字符串)  (助记符)
```

`#` 是 TableGen 的**字符串拼接运算符**。

各部分实际值（以 `def Foo : NorthStar_Type<"Foo", "foo">` 为例）：

| 部分 | 来源 | 值 |
|------|------|-----|
| `dialect.name` | `NorthStar_Dialect` 对象内置字段 `name` | `"north_star"` |
| `"."` | 字符串字面量 | `"."` |
| `typeMnemonic` | 调用者传入的参数 | `"foo"` |

拼接结果：`"north_star" # "." # "foo"` → `"north_star.foo"`

**`typeName` 的作用** — MLIR 类型的全局唯一标识字符串：

1. **IR 文本解析** — 遇到 `!north_star.foo` 时，用 `typeName` 定位正确的类型定义
2. **工厂查询** — `Dialect::lookupType("north_star.foo")` 用这个字符串查表
3. **错误信息** — 类型不匹配时报错显示

在生成的 `.h.inc` 中展开为：
```cpp
static constexpr StringLiteral name = "north_star.foo";
```

## 7. typeName / traits 在声明和实现中都会体现吗？

**是的。** 以第 3 章实际生成的 `NSTensorType` 为例：

**.h.inc（声明）：**
```cpp
class NSTensorType : public ::mlir::Type::TypeBase<...> {
  static constexpr StringLiteral name = "north_star.ns_tensor";  // ← typeName
  static constexpr StringLiteral dialectName = "north_star";
  static NSTensorType get(MLIRContext*, ...);      // ← builders → 工厂方法声明
  ArrayRef<int64_t> getShape() const;              // ← parameters → getter 声明
};
```

**.cpp.inc（实现）：**
```cpp
struct NSTensorTypeStorage : public TypeStorage { ... };  // ← 存储类定义

ArrayRef<int64_t> shape;   // ← parameters → 存储成员
Type elementType;
int64_t device_id;

NSTensorType NSTensorType::get(MLIRContext*, ...) { ... }  // ← builders → 工厂方法实现

generatedTypeParser(...);   // ← typeName → 用 mnemonic "ns_tensor" 匹配
generatedTypePrinter(...);  // ← typeName → 用 mnemonic "ns_tensor" 打印
```

| 生成内容 | .h.inc（声明） | .cpp.inc（实现） |
|---------|:--:|:--:|
| `typeName` → `name` 常量 | ✓ | — |
| `mnemonic` → `getMnemonic()` | ✓ | — |
| `parameters` → getter 声明 / 存储成员 | ✓ getter | ✓ 存储结构体 |
| `builders` → 工厂方法 | ✓ 声明 | ✓ 实现 |
| 解析/打印 | ✓ `parse`/`print` 声明 | ✓ dispatch 函数 |
| traits → `using` 声明 | ✓ | ✓（如需要） |

## 8. 常见标志位速查

| 标志 | 默认值 | 含义 |
|------|--------|------|
| `hasCustomAssemblyFormat` | `0` | `1` = 手写 parse/print，不用自动生成的泛型格式 |
| `hasStorageCustomConstructor` | `0` | `1` = 手写存储类构造函数，不自动生成 |
| `skipDefaultBuilders` | `0` | `1` = 不生成默认 `get`/`getChecked`，用自己的 builder 列表 |
| `genStorageClass` | `1` | `0` = 不自动生成存储类（手写） |
| `genVerifyDecl` | `0` | `1` = 生成 `verify()` 校验方法声明 |
| `genAccessors` | `1` | `0` = 不生成参数 getter 方法 |

### `hasCustomAssemblyFormat = 1`

告诉 TableGen：这个类型的 IR 文本格式我来手写。设 `1` 后生成 `.h.inc` 中包含 `parse`/`print` 方法声明，开发者在自己的 `.cpp` 中实现：

```cpp
Type NSTensorType::parse(AsmParser &parser) {
  // 自定义解析：!north_star.ns_tensor<4x8xf32, device:0>
}
void NSTensorType::print(AsmPrinter &printer) const {
  // 自定义打印
}
```

如果设 `0`（默认），MLIR 用它自动生成的泛型格式，直接把参数按顺序列出，没有任何可读性优化：

```
!north_star.ns_tensor<4x8xf32, f32, 42>  // 自动格式，生硬
```

而自定义后可写成：

```
!north_star.ns_tensor<4x8xf32, device:42>  // 更可读
```

### `hasStorageCustomConstructor = 0`

默认值就是 `0`，TableGen 自动根据 `parameters` 生成存储构造函数：

```cpp
// 自动生成：
NSTensorTypeStorage(ArrayRef<int64_t> shape, Type elementType, int64_t device_id)
    : shape(std::move(shape)), elementType(std::move(elementType)), device_id(std::move(device_id)) {}
```

若设为 `1`，告诉 TableGen 这个构造函数你要手写。

### `skipDefaultBuilders`

默认 `0` 时，TableGen 自动生成把所有 `parameters` 都作为参数的标准 `get` 方法。设为 `1` 表示只用 `builders` 列表里手写的工厂方法。

典型场景：IntegerType 设为 `1`，因为默认 builder 的 `signedness` 无默认值，需要自定义 `signedness = Signless` 的版本。

## 9. CRTP 模式详解

生成的代码中：
```cpp
class NSTensorType : public ::mlir::Type::TypeBase<
    NSTensorType,                       // CRTP 自身类名 (来自 name)
    ::mlir::Type,                       // 基类 (来自 baseCppClass)
    detail::NSTensorTypeStorage         // 存储类
> { ... };
```

CRTP（Curiously Recurring Template Pattern）即**子类把自己作为模板参数传给父类**。

### 为什么需要？简单例子

```cpp
// 父类模板
template <typename T>
class Animal {
public:
    void speak() {
        static_cast<T*>(this)->sound();  // 把自己转成子类，调子类方法
    }
};

// 子类把自己 T=Dog 塞回父类
class Dog : public Animal<Dog> {
public:
    void sound() { cout << "汪汪" << endl; }
};

class Cat : public Animal<Cat> {
public:
    void sound() { cout << "喵喵" << endl; }
};

// 使用
Dog d;
d.speak();  // "汪汪" — speak() 是父类写的，但最终调了 Dog::sound()
Cat c;
c.speak();  // "喵喵"
```

### MLIR 为什么用它

父类 `TypeBase` 中的工厂方法需要返回具体子类，不是基类：

```cpp
// TypeBase 内部：
template <typename ConcreteType, typename BaseType, typename StorageType>
class TypeBase : public BaseType {
public:
    static ConcreteType get(MLIRContext *ctx, ...) {
        //            ↑ ConcreteType = NSTensorType，编译期已知
        return BaseType::get(ctx, ...);
    }
};
```

如果不用 CRTP（普通虚函数），调用者每次都要手动 cast：

```cpp
Type t = get(ctx, ...);           // 返回基类
auto ns = cast<NSTensorType>(t);  // 被迫写 cast
```

用了 CRTP：

```cpp
NSTensorType t = NSTensorType::get(ctx, ...);  // 直接返回子类，无需 cast
```

| | 虚函数（动态多态） | CRTP（静态多态） |
|---|---|---|
| 返回类型 | 永远返回基类 | 返回具体子类 |
| 调用开销 | 虚表跳转 | 编译期内联，零开销 |
| 类型安全 | 需要手动 cast | 编译期强制检查 |

MLIR 选 CRTP 因为编译器 IR 操作极其频繁——每个 get/create 如果多一次虚表跳转，整个编译流程会显著变慢。

## 10. `detail` 命名空间

`AttrTypeBase.td:169`：
```tablegen
string storageNamespace = "detail";
```

`detail` 是 LLVM/MLIR 通用 C++ 惯例，把实现细节藏起来：

```cpp
namespace detail {
  struct NSTensorTypeStorage : public TypeStorage { ... };  // 内部实现，别碰
}
```

为什么必须暴露又必须藏？因为 C++ 模板编译需要 `Storage` 的完整定义（`sizeof` 等），不能是前向声明。于是用 `detail` 作为社交契约：

- **技术上**：你包含了这个头文件，编译器需要看 `Storage`
- **约定上**：不要在你的代码里写 `detail::NSTensorTypeStorage`，升级时这东西随时变

和 Java 的 `private` 内部类不同——C++ 做不到编译器层面彻底隐藏模板参数，所以用命名空间约定替代。
