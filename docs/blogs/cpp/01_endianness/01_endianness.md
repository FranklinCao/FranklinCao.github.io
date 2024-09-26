# 认识字节序转换问题

## 引言

计算机系统间通信时常常需要拟定通信协议，作为通信双方共同遵守的规则。协议中常常需要传递**数字**类型的信息，例如温度传感器上报当前温度（一个有符号的**小数**），设置工作模式（可能是一个无符号**整数**）等等。

发送数字信息，直接发送内存中的二进制表示是最直接的方式。这种方式速度快、占用带宽低，无需对数据做额外编码或解析。然而不同系统中对于同样一段内存的解释可能是不同的，所以虽然这种方式效率很高，但也带来了一个至关重要的问题：**大小端序（endianness）**。

字节序问题本质上是不同计算机体系结构在存储多字节数据时的顺序差异。这种差异会导致发送端和接收端对相同的数据字节有不同的解释，从而导致数据解析出错。因此跨平台通信时，必须正确处理字节序问题，才能确保数据的准确传输和解析。

在本文中，我们将深入探讨字节序问题的本质，分析它对二进制数据传输协议的具体影响，并介绍如何在实际应用中通过合适的技术手段正确处理和转换字节序。


## 什么是字节序


在二进制数据传输或存储中，数据的字节顺序决定了系统如何理解和解释多字节的数据值。解释字节的顺序可以分为**大端序（Big Endian）**和**小端序（Little Endian）**两种。

在大端序的系统中，高位字节（也就是权重高的字节）存储在内存的低地址，低位字节（也就是权重低的字节）存储在内存的高地址。大端序的排列方式类似于人们书写数字的顺序，即从左到右依次存储，比较符合人类的直觉。

数值 `BE BA FE CA`（十六进制）在大端序中会按如下方式存储：

```
地址：  01 02 03 04
内容：  BE BA FE CA
```

与大端序相反，小端序系统将低位字节存储在内存的低地址，而高位字节存储在内存的高地址。即从右到左的方式排列。

数值 `BE BA FE CA`（十六进制）在小端序中会按如下方式存储：

```
地址：  01 02 03 04
内容：  CA FE BA BE
```

所以如果发送端为大端系统，发送的数字为 `BE BA FE CA`，发送时直接将内存中的四个字节发出。小端系统收到后，如果不考虑大小端问题直接解析，就会认为第一个字节 `0xBE` 的权重最低，认为最后一个字节 `0xCA` 权重最高，但这完全不是发送方的本意，从而将这个数字解释为 `CA FE BA BE`，这样解析结果就是完全不同的数字了。


解决大小端问题最直接有效的方法就是统一字节序。可以认为地规定传输协议中所有数字均为大端序。在许多网络协议中，统一字节序的选择往往是**大端序（Big Endian）**，并被称为**网络字节序（Network Byte Order）**。使用大端序作为标准的原因之一是大端序的字节排列方式与人们书写数字的方式一致，相对直观且易于理解。

例如，在 TCP/IP 协议中无论是在数据报头中表示 IP 地址、端口号，还是传输协议中的其他数值字段，所有数据都按照大端序排列。该标准确保了数据在不同系统之间的兼容性与可移植性。

**然而**，几乎所有的计算机和嵌入式设备都采用小端序，只有网络字节序仍然采用大端序。

![alt text](01_endian_02.jpeg)


## 大小端转换的手动实现     


下面通过位运算完成几个常见数据类型的大小端转换

**uint8_t 类型**

`#!cpp uint8_t` 是单字节类型，不存在字节序的问题，因此无需转换。

**uint16_t 类型**

`#!cpp uint16_t` 占用 2 个字节，只需将第 1 个字节取出后右移 8 位，再将第 2 个字节取出左移 8 位，最后将两者通过或运算拼起来即可。

```cpp
uint16_t SwapUint16T(const uint16_t &value)
{
    return ((value & 0xff00) >> 8) | ((value & 0x00ff) << 8);
}
```

**uint32_t 类型**

`#!cpp uint32_t` 占用 4 个字节，和 `#!cpp uint16_t` 类型相似，但是分别要分别取出第 1、2、3、4 个字节，然后通过左移或右移将字节调换到与其原位置对称的位置，最后将所有调整好位置的字节通过或运算拼起来即可。

```cpp
uint32_t SwapUint32T(const uint32_t &value)
{
    return ((value & 0xff000000) >> 24) | ((value & 0x00ff0000) >> 8) | 
           ((value & 0x0000ff00) << 8)  | ((value & 0x000000ff) << 24);
}
```

**uint64_t 类型**

`#!cpp uint64_t` 通常占用 8 个字节，和 `#!cpp uint32_t` 类型相似，依然是取出各个字节，调整位置后拼起来，只是共需要取出 8 次，稍显繁复。

```cpp
uint64_t SwapUint64T(const uint64_t &value)
{
    return ((value & 0xff00000000000000) >> 56) | ((value & 0x00ff000000000000) >> 40) |
           ((value & 0x0000ff0000000000) >> 24) | ((value & 0x000000ff00000000) >> 8)  |
           ((value & 0x00000000ff000000) << 8)  | ((value & 0x0000000000ff0000) << 24) |
           ((value & 0x000000000000ff00) << 40) | ((value & 0x00000000000000ff) << 56);
}
```

**float 类型**

`#!cpp float` 类型**通常**占用 4 个字节，但是由于 `#!cpp float` 类型无法直接进行与运算，也不能左移或右移，但是这里处理的是字节数据，所以可以将 float 内存复制到同样占用 4 个字节的 `#!cpp uint32_t` `#!cpp uint32_t` 的方法进行字节调换，最后再将调换完成的 `#!cpp uint32_t` 类型重新复制到 `#!cpp float` 数据中返回即可。

```cpp
float SwapFloat(const float &value)
{
    uint32_t originBit{};
    std::memcpy(&originBit, &value, sizeof(float));
    uint32_t swappedBit = SwapUint32T(originBit);

    float ret{};
    std::memcpy(&ret, &swappedBit, sizeof(float));
    return ret;
}
```

**double 类型**

`#!cpp double` 类型**通常** 8 个字节，和 `#!cpp float` 类型一样，也无法进行与和左右移运算，这里可以将 `#!cpp double` 所持有的内存复制到同样占用 8 个字节的 `#!cpp uint64_t` ，并按照 `#!cpp uint64_t` 处理，最后再重新复制到 `#!cpp double` 中返回即可。

```cpp
double SwapDouble(const double &value)
{
    uint64_t originBit{};
    std::memcpy(&originBit, &value, sizeof(double));
    uint64_t swappedBit = SwapUint64T(originBit);

    double ret{};
    std::memcpy(&ret, &swappedBit, sizeof(double));
    return ret;
}
```

下面编写一个函数用来测试 `#!cpp uint16_t` 的大小端转换函数：

```cpp hl_lines="3 4"
void TestShort()
{
    uint16_t value = 0x1234;
    uint16_t swapped = SwapUint16T(value);

    std::cout << "origin:  " << std::hex << std::showbase << value << std::endl;
    std::cout << "swapped: " << std::hex << std::showbase << swapped << std::endl;
}
```

输出：
```cpp
origin:  0x1234
swapped: 0x3412
```

这里选择使用的是人畜无害的无符号类型 `#!cpp uint16_t` 传入测试，被测试的函数 `SwapUint16T` 接受和返回的参数也都是无符号类型。

但是如果改传入有符号类型 `#!cpp int16_t` 呢，会有隐式类型转换问题吗。

是的，的确会发生 `#!cpp int16_t` 向 `#!cpp uint16_t` 的隐式类型转换，但很幸运不会有问题。

如果 `#!cpp int16_t` 类型的值为正数，那么相安无事，如果 `#!cpp int16_t` 类型的值为负数，根据类型转换规则，目标类型是无符号整数时，源整数会被转换为目标类型所能表示的最小的无符号整数。转换的规则是将源整数对 2^n （n 为该类型所占的位数）取模。

但是，**无论是以上两种中的哪种情况，都只是对于字节的内容的重新解释，从字节的角度看不会发生改变，因此不会影响字节级别的位置调换操作**。

C++ 标准草案中，也同样表述了有符号数向无符号数的转换规则，以及如果不发生截断，转换仅发生在语义层面，而不会改变二进制位层面的内容。

!!! quote "C++17 - N4713 - 7.8 Integral conversions"
    If the destination type is unsigned, the resulting value is the least unsigned integer congruent to the source
    integer (modulo 2^n where n is the number of bits used to represent the unsigned type). [ Note: In a two’s
    complement representation, this conversion is conceptual and there is no change in the bit pattern (if there is
    no truncation). — end note ]

**但是如果反过来传递呢**，如果 SwapUint16T 接受的是 `#!cpp int16_t`，但是将 `#!cpp uint16_t` 类型的数字传入，会发生隐式类型转换吗？

是的，会发生由 `#!cpp uint16_t` 向 `#!cpp int16_t` 的隐式类型转换，但是就没有刚才那么幸运了。

!!! Danger "无符号类型向有符号类型的隐式转换是实现定义行为"
    一旦 `#!cpp uint16_t` 所表示的内容超出了 `#!cpp int16_t` 表示的范围（很可能发生），这种转换将是**实现定义行为**，也就是依赖于编译器的具体实现，不同编译器可能有不同的处理方式。可能是报错，可能是警告，可能是你不知道的其他结果。程序设计时应该竭尽全力避免这些不确定的行为。

C++ 17 标准草案中中也提到，这是实现定义的。

!!! quote "C++17 - N4713 - 7.8 Integral conversions"
    If the destination type is signed, the value is unchanged if it can be represented in the destination type; otherwise, the value is implementation-defined.

**所以如果要做位操作，尽可能统一使用无符号类型操作。**


此前，将 `#!cpp float` 数据按照 `#!cpp uint32_t` 数据处理前有一步预处理操作：将 `#!cpp float` 数据中的内存逐个字节拷贝到了另一个 `#!cpp uint32_t` 数据中。返回时，再将调整好字节顺序的 `#!cpp uint32_t` 数据重新拷贝到 `#!cpp float` 数据中用于返回。

```cpp hl_lines="4 8"
float SwapFloat(const float &value)
{
    uint32_t originBit{};
    std::memcpy(&originBit, &value, sizeof(float));
    uint32_t swappedBit = SwapUint32T(originBit);

    float ret{};
    std::memcpy(&ret, &swappedBit, sizeof(float));
    return ret;
}
```

这样内存搬移是否有必要，有人提出一种更为简洁的方法：

可以直接将传入的参数的地址强转为 `#!cpp uint32_t` 类型的指针，然后解引用。相当于将 `#!cpp value` 中所占的内存按照 `#!cpp uint32_t` 方式重新解释。返回时用相似的方法再将其所占的内存重新解释为 `#!cpp float` 数据并返回。（使用 C++ 风格的 `#!cpp reinterpret_cast<T>` 也是相同的含义）。

```cpp 
float SwapFloat(const float &value)
{
    uint32_t swappedBit = SwapUint32T(*(uint32_t *)&value);
    return *(float *)&swappedBit;
}
```

这种方式使用了**类型双关 (Type Punning)**，意在通过重新解释数据的内存表示，将数据从一种类型视为另一种类型。然而，此处的使用方式会违反**严格别名规则 (Strict Aliasing Rule)**。

!!! note "严格别名规则 (Strict Aliasing Rule)"
    该规则会帮助编译器优化程序，其假设不同类型的指针不会指向相同的内存地址（即不同类型的别名不能指向相同的对象）。如果打破了这个假设，编译器可能会生成不符合预期的代码，虽然有时会产生符合预期的结果，但是记住，不要把赌注押在未定义行为上。

    严格别名规则中有一种**例外情况**，就是 `#!cpp char*、unsigned char*、signed char *`，C++标准允许通过这些指针访问任何对象的原始字节序列。这是因为C++标准中的字节操作正是通过这三种类型定义的。例如以下操作是完全合法的。

    ```cpp
    int i = 0;
    char *buf = (char *)&i; 
    buf[0] = 1;
    ```

除了使用 `#!cpp std::memcpy`，C++20 在 `#!cpp <bit>` 头文件中还引入了 `#!cpp std::bit_cast<T>` 模板函数，专门用于处理这类将一种类型的数据重新解释为另一种类型的问题：

```cpp
float SwapFloat(const float &value)
{
    uint32_t originBit = std::bit_cast<uint32_t>(value);
    uint32_t swappedBit = SwapUint32T(originBit);
    return std::bit_cast<float>(swappedBit);
}
```

## 大小端转换的第三方库实现

在此选择一个简 (wǒ) 洁 (néng) 明 (kàn) 了 (dǒng) 的第三方大小端转换库 [Giant](https://github.com/r-lyeh-archived/giant/tree/master)，来学习别人的实现方式。

首先该库探查了一系列宏定义用于推测系统字节序。这些宏有的直接指名大小端类型，有的代表的是系统架构。

```cpp
#if defined(_LITTLE_ENDIAN) || (defined(BYTE_ORDER) && defined(LITTLE_ENDIAN) && BYTE_ORDER == LITTLE_ENDIAN) ||       \
    (defined(_BYTE_ORDER) && defined(_LITTLE_ENDIAN) && _BYTE_ORDER == _LITTLE_ENDIAN) ||                              \
    (defined(__BYTE_ORDER) && defined(__LITTLE_ENDIAN) && __BYTE_ORDER == __LITTLE_ENDIAN) || defined(__i386__) ||     \
    defined(__alpha__) || defined(__ia64) || defined(__ia64__) || defined(_M_IX86) || defined(_M_IA64) ||              \
    defined(_M_ALPHA) || defined(__amd64) || defined(__amd64__) || defined(_M_AMD64) || defined(__x86_64) ||           \
    defined(__x86_64__) || defined(_M_X64)
enum
{
    xinu_type = 0,
    unix_type = 1,
    nuxi_type = 2,
    type = xinu_type,
    is_little = 1,
    is_big = 0
};
#elif defined(_BIG_ENDIAN) || (defined(BYTE_ORDER) && defined(BIG_ENDIAN) && BYTE_ORDER == BIG_ENDIAN) ||              \
    (defined(_BYTE_ORDER) && defined(_BIG_ENDIAN) && _BYTE_ORDER == _BIG_ENDIAN) ||                                    \
    (defined(__BYTE_ORDER) && defined(__BIG_ENDIAN) && __BYTE_ORDER == __BIG_ENDIAN) || defined(__sparc) ||            \
    defined(__sparc__) || defined(_POWER) || defined(__powerpc__) || defined(__ppc__) || defined(__hpux) ||            \
    defined(_MIPSEB) || defined(_POWER) || defined(__s390__)
enum
{
    xinu_type = 0,
    unix_type = 1,
    nuxi_type = 2,
    type = unix_type,
    is_little = 0,
    is_big = 1
};
#else
#error <giant/giant.hpp> says: Middle endian/NUXI order is not supported
enum
{
    xinu_type = 0,
    unix_type = 1,
    nuxi_type = 2,
    type = nuxi_type,
    is_little = 0,
    is_big = 0
};
#endif
``` 

以小端部分的判断为例，宏定义分为以下两种类型。

和字节序有关的的宏：
```shell
BYTE_ORDER
__BYTE_ORDER
LITTLE_ENDIAN
__LITTLE_ENDIAN
```

和处理器架构有关的编译器预定义宏：
```shell
__i386__    # x86 架构（32位）

__amd64     # x86_64 架构（64位）
__amd64__   # x86_64 架构（64位）

__x86_64    # x86_64 架构（64位）
__x86_64__  # x86_64 架构（64位）

__alpha__   # Alpha 处理器架构  
_M_ALPHA    # Alpha 处理器架构
 
__ia64      # Itanium 64 位架构
__ia64__    # Itanium 64 位架构
_M_IA64     # Itanium 64 位架构
 
_M_IX86     # MSVC 编译器下的 x86 宏，表示 32 位 x86 处理器。
 
_M_AMD64    # MSVC 编译器下的 64 位处理器宏
_M_X64      # MSVC 编译器下的 64 位处理器宏
```

其中关于处理器架构的宏都是由编译器预定义的。和字节序有关的宏均来自头文件 `<endian.h>`。

!!! note "编译器预定义宏" 
    编译器的预定义宏主要用于帮助程序根据编译时环境来进行条件编译，从而编写跨平台、跨架构或与特定编译器功能兼容的代码。  

    可以通过如下指令查看编译器预定义的宏：

    ```shell
    touch foo.h
    cpp -dM -E foo.h
    ``` 

    其中 **-E** 意为执行到预处理完成结束，**-dM** 意为显示所有的宏定义（包括预定义宏）。    

    gcc 手册中也对 **-dM** 选项显示所有宏定义的功能做出了解释

    ```
    -dM

    Instead of the normal output, generate a list of ‘#define’ directives for 
    all the macros defined during the execution of the preprocessor, including predefined macros. 
    This gives you a way of finding out what is predefined in your version of the preprocessor. 
    
    Assuming you have no file foo.h, the command
    touch foo.h; cpp -dM foo.h
    shows all the predefined macros.
    ```

    其中 **-d** 意为启用特定编译器阶段的 **转存储（dump）** 功能，后面的所接的字母意为转存储的具体内容，**M** 表示 **输出宏定义（macro）**，包括编译器预定义的和用户定义的，并且是直接在预处理阶段完成的。实际上，**-dM** 选项必须配合 **-E** 使用，以确保它只进行预处理，而不进入后续的编译阶段。


我所使用的 gcc version 14.2.1 20240805 编译器预定义了以下两个宏，但是该库并没有利用这两个宏作为判断依据。
```cpp
#define __ORDER_LITTLE_ENDIAN__ 1234
#define __BYTE_ORDER__ __ORDER_LITTLE_ENDIAN__
```


适配多种类型的大小端互转模板函数 `#!cpp T swap(T out)` 是该库的核心。在该函数的开端，作者定义了一个静态联合体（union），利用联合体和静态局部变量的特性，实现了在运行时验证字节序推测结论是否正确。

```cpp hl_lines="4 5"
template <typename T> T swap(T out)
{
    static union autodetect {
        int word;
        char byte[sizeof(int)];
        autodetect() : word(1)
        {
            assert
            (
                (
                    "<giant/giant.hpp> says: wrong endianness detected!", 
                    (!byte[0] && is_big) || (byte[0] && is_little)
                )
            );
        }
    } _;

    // ...
}
```

联合体中有一个整型变量，和一个字符数组，数组大小为 `#!cpp sizeof(int)`，用于按字节查看整型变量。 `#!py static` 关键字表示联合体的实例是静态局部变量，只会在程序的生命周期内被初始化一次，并在多次调用 `#!cpp T swap(T out)` 函数时共享。

联合体的构造函数中利用初始化列表将 `#!py word` 变量初始化为 `1`。对于一个int类型的值 `1`，它的二进制表示是 `#!py 0x 00 00 00 01`。

随后在构造函数中通过断言验证系统字节序，这里发生了两件值得关注的事情。

```cpp hl_lines="4"
assert
(
    (
        "<giant/giant.hpp> says: wrong endianness detected!", 
        (!byte[0] && is_big) || (byte[0] && is_little)
    )
);
```


- `#!py assert` 宏实际上只接受一个参数，但是作者利用逗号运算符只返回最后一个逗号后面的运算结果的特性，使得如果断言触发，既能够打印逗号前的出错误提示信息，又能够判断逗号后的断言触发条件。


!!! note "逗号运算符"
    逗号运算符（,）用于顺序执行多个表达式，并返回最后一个表达式的值。它可以在许多场景下使用，比如在循环中，或在声明和初始化多个变量时。
    当使用逗号运算符时，多个表达式从左到右依次执行，但仅返回最后一个表达式的值。
    ```cpp
    int a = 1, b = 2;
    int result = (a += 2, b += 3);
    // a = 3, b = 5, result = 5
    ```
    在这个例子中：a += 2 先执行，将 a 的值变为 3。然后执行 b += 3，将 b 的值变为 5。最后，返回 b += 3 的结果，即 5，并将其赋值给 result

!!! note "assert 与 NDEBUG 宏"
    由于 `#!py assert` 断言仅在 `#!py NDEBUG` 没有定义时有效，当代码以 `#!py CMAKE_BUILD_TYPE` 为 `#!py Release` 时编译时，不会定义 `#!py NDEBUG` 宏，也就是说 `#!py Release` 模式下所有 `#!py assert` 断言会失效。


    为了解决这个问题，通常有以下几种方法可供参考：

    方法一：直接使用 `#!py if` 进行显式检查
    ```cpp
    if ((!byte[0] && is_big) || (byte[0] && is_little)) 
    {
        // 字节序正确
    } 
    else 
    {
        throw std::runtime_error("<giant/giant.hpp> says: wrong endianness detected!");
    }
    ```
    这种处理方式没有宏和条件编译相关的复杂性，代码更加清晰可控。但是如果有大量的断言，所有断言都转换成 if 语句会增加代码冗余，并且显得冗长。


    方法二：使用自定义的 `#!py MyAssert` 函数
    ```cpp
    template<typename T>
    void MyAssert(bool condition, T message) {
        if (!condition) {
            std::cerr << message << std::endl;
            std::abort();
        }
    }
    ```

    方法三：在 `#!py Release` 模式下仍然开启 `#!py NDEBUG` 宏

    ```shell
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -UNDEBUG")
    ```

```cpp hl_lines="5"
assert
(
    (
        "<giant/giant.hpp> says: wrong endianness detected!", 
        (!byte[0] && is_big) || (byte[0] && is_little)
    )
);
```

- 断言的后半部分是首先判断了 `#!py byte[0]`（`#!py word` 的第一个字节）。如果系统是小端序，那么 `#!py 1` 的最低有效字节会存储在最低地址，所以 `#!py byte[0] == 1`。如果是大端序，最高有效字节会在最低地址，所以 `#!py byte[0] == 0`。再通过和最前面通过宏定义得到的结论对比，就能验证此前大小端结论推测是否正确。一旦验证失败，就会触发断言。

!!! note "联合体中访问非活动成员的例外情况"
    C++ 标准中，联合体中的**活动成员**是最近一次写入的成员。当联合体的某个成员被赋值时，这个成员就成为**活动成员**。如果在某一时刻访问了不是活动成员的其他成员，将是未定义行为。但是这里构造函数只指定了成员 `#!py word` 的值，但是却访问了非活动成员 `#!py char` 数组，这是未定义行为吗？
    
    虽然 C++ 对联合体跨成员访问要求严格，但是有一个**例外情况**即当其他成员是**布局兼容类型（layout-compatible types）**时允许访问非活动成员，常见的布局兼容情况包含同一基本类型的不同别名、字节数组与任何类型、标准布局类型。在这里属于第二类情况，即字节数组与任何类型：`#!py char[]` 或 `#!py unsigned char[]` 可以安全地读取和写入任何类型的内存。这是因为 C++ 标准允许通过 `#!py char` 或 `#!py unsigned char` 访问任何对象的底层字节表示。这同样也是一种**类型双关**。

最后进入正题：该库的大小端互转模板函数

```cpp
template <typename T> T swap(T out)
{
    static union autodetect {
        int word;
        char byte[sizeof(int)];
        autodetect() : word(1)
        {
            assert(
                ("<giant/giant.hpp> says: wrong endianness detected!", (!byte[0] && is_big) || (byte[0] && is_little)));
        }
    } _;

    if (!std::is_pod<T>::value)
    {
        return out;
    }

    char *ptr;

    switch (sizeof(T))
    {
    case 0:
    case 1:
        break;
    case 2:
        ptr = reinterpret_cast<char *>(&out);
        std::swap(ptr[0], ptr[1]);
        break;
    case 4:
        ptr = reinterpret_cast<char *>(&out);
        std::swap(ptr[0], ptr[3]);
        std::swap(ptr[1], ptr[2]);
        break;
    case 8:
        ptr = reinterpret_cast<char *>(&out);
        std::swap(ptr[0], ptr[7]);
        std::swap(ptr[1], ptr[6]);
        std::swap(ptr[2], ptr[5]);
        std::swap(ptr[3], ptr[4]);
        break;
    case 16:
        ptr = reinterpret_cast<char *>(&out);
        std::swap(ptr[0], ptr[15]);
        std::swap(ptr[1], ptr[14]);
        std::swap(ptr[2], ptr[13]);
        std::swap(ptr[3], ptr[12]);
        std::swap(ptr[4], ptr[11]);
        std::swap(ptr[5], ptr[10]);
        std::swap(ptr[6], ptr[9]);
        std::swap(ptr[7], ptr[8]);
        break;
    default:
        assert(!"<giant/giant.hpp> says: POD type bigger than 256 bits (?)");
        break;
    }

    return out;
}
```

该库中没有为各类字节长度不同的类型分别编写函数，而是利用模板函数统一处理，再利用 `#!cpp sizeof` 操作符判断长度并处理。

在判断传入的类型占用的字节数后，作者使用了上文中提到的类型双关中不违反严格别名规则的例外情况：使用 `#!cpp char *` 类型重新解释，然后使用 `#!cpp std::swap()` 完成字节序列交换。


## 参考

- [Draft C++17 standard is freely available here](https://github.com/cplusplus/draft/tree/main/papers "draft/papers at main · cplusplus/draft")

- [Latest Draft C++ Standard](http://eel.is/c++draft/ "Draft C++ Standard: Contents")

- [What is Strict Aliasing and Why do we Care?](https://gist.github.com/shafik/848ae25ee209f698763cffee272a58f8 "What is Strict Aliasing and Why do we Care?")

- [Preprocessor Options (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc-14.2.0/gcc/Preprocessor-Options.html "Preprocessor Options (Using the GNU Compiler Collection (GCC))")

- [Giant: A tiny C++11 library to handle little/big endianness.](https://github.com/r-lyeh-archived/giant "r-lyeh-archived/giant: :moyai: Giant is a tiny C++11 library to handle little/big endianness.")

---
创建于：`2024-09-23`

编辑于：`2024-09-23`
