# INTERNAL_SHADER_PARAMETER_EXPLICIT

为什么了解这个宏，看另一篇 《UE5 MobileBasePassPixelShader.usf》



这里详细了解一下这个宏，不感兴趣的可以跳过，知道这个宏用于 <font color=red>**声明在统一缓冲区 (Uniform Buffer Object, UBO) 结构体成员时，自动生成和管理相关元数据** </font> 就行了。



```c++
/** Declares a member of a uniform buffer struct. */
#define INTERNAL_SHADER_PARAMETER_EXPLICIT(BaseType,TypeInfo,MemberType,MemberName,ArrayDecl,DefaultValue,Precision,OptionalShaderType,IsMemberStruct) \
		zzMemberId##MemberName; \
	public: \
		TypeInfo::TAlignedType MemberName DefaultValue; \
		// 静态断言：验证基础类型的有效性，确保 MemberType 是有效的 UBO 成员类型 
		static_assert(BaseType != UBMT_INVALID, "Invalid type " #MemberType " of member " #MemberName "."); \
	private: \
		struct zzNextMemberId##MemberName { enum { HasDeclaredResource = zzMemberId##MemberName::HasDeclaredResource || !TypeInfo::bIsStoredInConstantBuffer }; }; \
		static zzFuncPtr zzAppendMemberGetPrev(zzNextMemberId##MemberName, TArray<FShaderParametersMetadata::FMember>* Members) \
		{ \
			static_assert(TypeInfo::bIsStoredInConstantBuffer || TIsArrayOrRefOfTypeByPredicate<decltype(OptionalShaderType), TIsCharEncodingCompatibleWithTCHAR>::Value, "No shader type for " #MemberName "."); \
			static_assert(\
				(STRUCT_OFFSET(zzTThisStruct, MemberName) & (TypeInfo::Alignment - 1)) == 0, \
				"Misaligned uniform buffer struct member " #MemberName "."); \
			Members->Add(FShaderParametersMetadata::FMember( \
				TEXT(#MemberName), \
				(const TCHAR*)OptionalShaderType, \
				__LINE__, \
				STRUCT_OFFSET(zzTThisStruct,MemberName), \
				EUniformBufferBaseType(BaseType), \
				Precision, \
				TypeInfo::NumRows, \
				TypeInfo::NumColumns, \
				TypeInfo::NumElements, \
				TypeInfo::GetStructMetadata() \
				)); \
			zzFuncPtr(*PrevFunc)(zzMemberId##MemberName, TArray<FShaderParametersMetadata::FMember>*); \
			PrevFunc = zzAppendMemberGetPrev; \
			return (zzFuncPtr)PrevFunc; \
		} \
		typedef zzNextMemberId##MemberName
```

这个宏用于声明在统一缓冲区 (Uniform Buffer Object, UBO) 结构体成员时，自动生成和管理相关元数据。

**统一缓冲区是一种允许在 GPU 和 CPU 之间共享数据的方式，常用于存储不经常更改的渲染参数**

下面是宏定义各部分的详细解析：

- **宏名称： `INTERNAL_SHADER_PARAMETER_EXPLICIT`**
    - 目的：为 shader 参数在 UBO 结构体中声明成员，并同时生成用于反射（元数据描述）的代码
- **参数说明**
    - `BaseType`
        - 基础类型枚举，表示成员变量的基础数据类型（如 float, int 等）
    - `TypeInfo`
        - 一个类模板或结构，包含成员变量的类型信息，如对齐要求，是否存储在常量缓冲区等
    - `MemberType`
        - 成员变量的实际类型
    - `MemberName`
        - 成员变量名
    - `ArrayDecl`
        - decl 是 declaration 的缩写
        - 数组声明部分，如果成员是数组，则表示数组的维度
    - `DefaultValue`
        - 成员变量的默认值
    - `Precision`
        - 精渡标识，如浮点数的高精度、低精度等
    - `OptionalShaderType`
        - 可选的 shader 类型字符串，用于指定在 shader 语言中的对应类型
    - `IsMemberStruct`
        - 指示 MemberType 是否为结构体的布尔值



## 逐行解读

- ```c++
    zzMemberId##MemberName; \
    ```

    - **`##` C++ 预处理器中的 token pasting 令牌粘合操作符**

    这个操作符用于将宏定义中的两个相邻的标记 (tokens) 合并成为一个新的标记。在这个上下文中，`##` 的作用是将字符串字面量 `zzMemberId` 与实际传入的 `MemberName` 参数拼接在一起，形成一个新的标识符，如 `zzMemberIdMyVariable`。这种技术常用于生成唯一的标识符，以避免名字冲突，或是创建具有特定前缀或后缀的变量名。

    这样生成的标识符可能用于内部跟踪或者作为编译时期计算的一部分，以帮助管理成员变量的元数据或其他相关信息，而不直接暴露给用户代码

    总之，`##` 操作符在这里的作用就是动态生成具有特定模式的标识符名称，以支持宏展开过程中对特定成员的标识和处理。



### 1. 声明成员变量

- ```c++
    public: \
    		TypeInfo::TAlignedType MemberName DefaultValue; \
    ```

    - `TypeInfo:::TAlignedType`
        - 这部分表示成员变量的类型。`TypeInfo` 应该是一个模板类或者是某种类型信息结构，他有一个内部类型别名 `TAlignedType`，这个别名指向实际存储数据的类型，但可能包含了对齐信息，确保数据在内存中的正确对齐，这对于性能敏感的代码（特别是 GPU 编程）非常重要。
    - `MemberName`：这是成员变量的名称，它来自于宏参数。当宏展开时，`MemberName` 会被替换为实际的变量名，比如 `MyScalar`。
    - `DefaultValue`: 这部分表示该成员变量的初始值。在 C++ 中，你可以在声明时为 **非静态的非引用非 const 成员变量** 提供一个默认值，这里 `DefaultValue` 会根据宏调用时提供的参数来确定实际的默认值，比如 `1.0f` 对于一个 `float` 类型的成员。

    综上所述，这行代码实例化了一个类型为 `TypeInfo::TAlignedType` 的成员变量（其名称由宏参数确定），并将其初始化为 `DefaultValue` 指定的值。这是一种常见的做法，特别是在定义带有类型安全和对齐控制的统一缓冲区（UBO）或 shader 参数结构时。



### 2. 断言检查 BaseType

- ```c++
    static_asssert(BaseType != UBMY_INVALID, "Invalid type " #MemberType " of member " #MemberName ".");
    ```

    - 这行代码使用 C++ 的 `static_assert` 关键字进行 **编译时** 的检查。它确保了一个条件表达式在编译时期为真，如果不满足条件，则会生成编译错误，并显示提供的错误信息。

    - 详细解释：

        `static_assert`

        这是一个编译时 **断言** ，意味着它会在程序编译时期执行检查，而不是在运行时。如果断言失败（即条件为假），编译器会产生一个错误并停止编译。

        `BaseType != UBMT_INVALID`

        这是静态断言中的条件表达式。它比较 `BaseType` （基础类型）与 `UBMT_INVALID` （一个假设表示无效的常量或枚举值，看下面代码，在 `RHIDefinitions.h` 中，确认是 `EUniformBufferBaseType` 中的枚举类型）

        ```c++
        /** The base type of a value in a shader parameter structure. */
        enum EUniformBufferBaseType : uint8
        {
        	UBMT_INVALID,
        
        	// Invalid type when trying to use bool, to have explicit error message to programmer on why
        	// they shouldn't use bool in shader parameter structures.
        	UBMT_BOOL,
        
        	// Parameter types.
        	UBMT_INT32,
        	UBMT_UINT32,
        	UBMT_FLOAT32,
        
        	// RHI resources not tracked by render graph.
        	UBMT_TEXTURE,
        	UBMT_SRV,
        	UBMT_UAV,
        	UBMT_SAMPLER,
        
        	// Resources tracked by render graph.
        	UBMT_RDG_TEXTURE,
        	UBMT_RDG_TEXTURE_ACCESS,
        	UBMT_RDG_TEXTURE_ACCESS_ARRAY,
        	UBMT_RDG_TEXTURE_SRV,
        	UBMT_RDG_TEXTURE_UAV,
        	UBMT_RDG_BUFFER_ACCESS,
        	UBMT_RDG_BUFFER_ACCESS_ARRAY,
        	UBMT_RDG_BUFFER_SRV,
        	UBMT_RDG_BUFFER_UAV,
        	UBMT_RDG_UNIFORM_BUFFER,
        
        	// Nested structure.
        	UBMT_NESTED_STRUCT,
        
        	// Structure that is nested on C++ side, but included on shader side.
        	UBMT_INCLUDED_STRUCT,
        
        	// GPU Indirection reference of struct, like is currently named Uniform buffer.
        	UBMT_REFERENCED_STRUCT,
        
        	// Structure dedicated to setup render targets for a rasterizer pass.
        	UBMT_RENDER_TARGET_BINDING_SLOTS,
        
        	EUniformBufferBaseType_Num,
        	EUniformBufferBaseType_NumBits = 5,
        };
        ```

        > UBMT 可能是 Uniform Buffer Member Type 的缩写

        这段代码定义了一个枚举类型 `EUniformBufferBaseType` ，用于表示正在 shader 参数结构中使用的各种基础类型。每个枚举成员代表了一种不同的数据类型或资源类型，这些类型可以被存储在 `UniformBufferObject ` (UBO，统一缓冲区，GPU 和 CPU 之间共享数据的方式，常用于存储不经常更改的渲染参数) 中，用于向 shader 传递数据。

        <font color=red>**在 C++ 枚举定义中，如果枚举成员没有显式地赋予一个值，编译器会自动为他们分配数值**</font>。默认情况下，枚举的第一个成员（在这个例子中是 `UBMT_INVALID`）会被分配数值 0，后的每个枚举成员会自动递增1。

        > - `UBMT_INVALID` -> 0
        > - `UBMT_BOOL` -> 1
        > - `UBMT_INT32` -> 2
        > - `UBMT_UINT32` -> 3
        > - ...
        > - `UBMT_RENDER_TARGET_BINDING_SLOTS` -> 27

        最后两个成员稍微特殊一些：

        - `EUniformBufferBaseType_Num` 通常用于表示枚举类型的成员数量，这里没有显示赋值，按自动递增规则应为28。
        - `EUniformBufferBaseType_NumBits` 显然是为了表示枚举类型所需的位数，这里赋值为5，意味着设计者认为最多需要使用到5位来唯一地表示所有的枚举值（2^5 = 32，但实际上只用了28个，可能是预留或是出于对齐、未来扩展的考虑）。

    - `"Invalid type " #MemberType " of member " #MemberName "." `

        - 这是错误信息的格式化字符串，其中包含了两个 **预处理器的字符串化操作符 `#`**。
        - 当 `static_assert` 失败时，这个字符串会被转化为实际的错误消息。`#MemberType` 和 `#MemberName` 会被替换为实际上 `MemberType` 和 `MemberName` 的文本值。
        - C++ 中 `#` 是一个预处理符号，称为 **字符串化操作符 (Stringification Operator)**。在宏定义中使用时，它会把紧跟在后面的宏参数转换成一个包含该参数内容的字符串字面量。



### 3. 结构体 zzNextMemberId##MemberName

- ```c++
    private: \
    		struct zzNextMemberId##MemberName { enum { HasDeclaredResource = zzMemberId##MemberName::HasDeclaredResource || !TypeInfo::bIsStoredInConstantBuffer }; }; \
    ```

    - 为下一个成员变量生成一个标识符，并定义一个嵌套的私有结构体，用于跟踪是否已经声明了资源



### 4. 函数 zzAppendMemberGetPrev

- ```c++
    static zzFuncPtr zzAppendMemberGetPrev(zzNextMemberId##MemberName, TArray<FShaderParametersMetadata::FMember>* Members) \
    ```

    - 这里始终不理解为什么 `zzNextMemberId##MemberName`  不是 `float32 a` 这种写法
    
    - 简单说，如果 MemberName 是 Fog，那么替换完毕之后就是 
    
        ```c++
        static zzFuncPtr zzAppendMemberGetPrev(zzNextMemberIdFog, TArray<FShaderParametersMetadata::FMember>* Members) \
        ```
    
        > 以下说法来自 ChatGPT，不确定是否正确
    
        这种写法不常见，但是它的意图在于：
    
        1. **类型定义和静态断言**
    
            通过明确第一个参数的类型为 `zzNextMemberIdFog`，可以在函数中实现 **静态断言** 和 **类型检查**
    
        2. **避免参数未使用的警告**
    
            如果不打算在函数实现中使用第一个参数，只是为了进行静态检查或类型定义，这样的写法可以避免编译器警告未使用参数
    
        
    
        包括前面的 `zzMemberId##MemberName;` 展开后是 `zzMemberIdFog;`
    
        这个标识符的作用在宏的定义中看似没有直接使用，但是实际上可能是为了生成唯一的类型名或者变量名，以便在后续代码或其它宏展开中使用。



- ```c++
    static_assert(
        TypeInfo::bIsStoredInConstantBuffer 
        || 
        TIsArrayOrRefOfTypeByPredicate<decltype(OptionalShaderType), TIsCharEncodingCompatibleWithTCHAR>::Value,
        "No shader type for " #MemberName ".");
    ```

    - 检查成员是否存储在常量缓冲区中，或者是否与 `TCHAR` 兼容



- ```c++
    static_assert(\
    				(STRUCT_OFFSET(zzTThisStruct, MemberName) & (TypeInfo::Alignment - 1)) == 0, \
    				"Misaligned uniform buffer struct member " #MemberName "."); \
    ```

    - 确保类型成员对齐



- ```c++
    Members->Add(FShaderParametersMetadata::FMember( \
    				TEXT(#MemberName), \
    				(const TCHAR*)OptionalShaderType, \
    				__LINE__, \
    				STRUCT_OFFSET(zzTThisStruct,MemberName), \
    				EUniformBufferBaseType(BaseType), \
    				Precision, \
    				TypeInfo::NumRows, \
    				TypeInfo::NumColumns, \
    				TypeInfo::NumElements, \
    				TypeInfo::GetStructMetadata() \
    				)); \
    ```

    - 将成员信息添加到 Members 数组中，包括 成员名，可选 Shader 类型，行号，结构体偏移，统一缓存基础类型，精度，行数，列数，元素数，结构体元数据（我瞎翻译的）



### 5. 函数指针 PrevFunc

- ```c++
    zzFuncPtr(*PrevFunc)(zzMemberId##MemberName, TArray<FShaderParametersMetadata::FMember>*); \
    			PrevFunc = zzAppendMemberGetPrev; \
    ```

    这两行的目的在于通过宏定义的展开，生成唯一的 <font color=red>**函数指针**</font> 类型，并将当前函数指针 `zzAppendMemberGetPrev` 赋值给 `PrevFunc` 。这种模式通常用于构建 **链式调用** 或 **函数指针列表**。

    在这段代码中，我们通过宏展开实现了 **类型安全** 和 **功能扩展**。



#### 详细解读

**宏展开后的代码**

假设 `MemberName` 是 `Fog`，宏展开后的代码如下：

```c++
zzFuncPtr(*PrevFunc)(zzMemberIdFog, TArray<FShaderParametersMetadata::FMember>*);
PrevFunc = zzAppendMemberGetPrev;
```



#### 第一句：声明函数指针

```c++
zzFuncPtr(*PrevFunc)(zzMemberIdFog, TArray<FShaderParameterMetadata::FMember>*);
```

这里声明一个函数指针 `PrevFunc`

1. `zzFuncPtr` 

    假设这是一个预定义的函数指针类型，表示返回值类型。通常它会是某种类型的别名，比如 `typedef void (*zzFuncPtr)(...);`
    
2. `(*PrevFunc)`

    这部分声明了一个名为 `PrevFunc` 的函数指针

3. `(zzMemberFog, TArray<FShaderParameterMetadata::FMember>*)`

    这是函数指针 `PrevFunc` 所指向的函数的参数列表。这里，`zzMemberIdFog` 是一个类型，而 `TArray<FShaderParametersMetadata::FMember>*` 是一个指向 `TArray<FShaderParametersMetadata::FMember>` 的指针。

这个声明表示 `PrevFunc` 是一个指向返回类型为 `zzFuncPtr`，参数为 `zzMemberIdFog` 和 `TArray<FShaderParametersMetadata::FMember>` 的函数指针



#### 第二句：函数指针赋值

```c++
PrevFunc = zzAppendMemberGetPrev;
```

这行代码将 `zzAppendMemberGetPrev` 函数的地址赋值给 `PrevFunc`.



#### 背景和用途

这两行代码在更大背景下用于构建链式调用或函数指针列表，尤其是在使用 **宏定义** 和 **模板元编程** 的复杂场景中

1. **构建链式调用**

    通过宏展开，将多个函数指针链接起来，形成链式调用。这有助于动态扩展功能和行为

2. **类型安全**

    通过生成唯一的类型和函数指针，确保在编译期进行类型检查，避免类型冲突和错误。



### 6. 类型别名

- ```c++
    typedef zzNextMemberId##MemberName
    ```

定义一个类型别名，用于链式调用

假设 `MemberName` 是 `Fog` ，则展开如下 `typedef zzNextMemberIdFog`

在宏定义中，typedef 后面紧跟的代码可能会在其他地方补全

















