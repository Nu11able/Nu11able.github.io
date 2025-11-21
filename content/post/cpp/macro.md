# 宏的用法
文本记录一些使用过的宏的用法
## 基础用法
```cpp
#define MACRO_COUNT 30

#ifdef MACRO_COUNT
#elif (defined(LINUX) || defined(UNIX)) && (defined(HI) && HI==1)
#else
#endif
```
其中的`defined` 就可以组合多个条件和检查宏具体的值

## 字符串化 `#`
```cpp
#define STRINGIFY(x) #x
#define FOO BAR
#define BAR 100

STRINGIFY(FOO)     // 结果："FOO"（有#，不展开）
STRINGIFY(BAR)     // 结果："BAR"（有#，不展开）  
STRINGIFY(100)     // 结果："100"
```

## 宏拼接 `##`
最简单的宏拼接就是
```cpp
#define CONCAT(a, b) a##b

#define MACRO_1
#define MACRO_2
// 调用
#define MACRO CONCAT(MACRO, 1) // MACRO_1
```
但是上述的方法有个弊端，当需要拼接的是一个宏展开的结果时就会出现问题，比如：
```cpp
#define CONCAT(a, b) a##_##b

#define MACRO_1 macro_1
#define MACRO_2 macro_2
// 调用
#define MACRO CONCAT(MACRO_1, 1) // MACRO_1_1
//但其实实际想要的是 macro_1_1

// 更通用的做法是将实际拼接的宏在包一层,具体原因请看宏的展开规则
#define CONCAT_IMPL(a, b) a##_##b
#define CONCAT(a, b) CONCAT_IMPL(a,b)

#define MACRO_1 macro_1
#define MACRO_2 macro_2

// 调用
#define MACRO CONCAT(MACRO_1, 1) // macro_1_1
```
## 宏展开规则
[C语言 宏嵌套的展开规则](https://zhuanlan.zhihu.com/p/344240420)
有很多高级得完全看不懂的宏都是基于嵌套宏做的

- 一般的展开规律像函数的参数一样：先展开参数，再分析函数，即由内向外展开
- 当宏中有#运算符的时候，不展开参数
- 当宏中有##运算符的时候，先展开函数，再分析参数
- ##运算符用于将参数连接到一起，预处理过程把出现在##运算符两侧的参数合并成一个符号，注意不是字符串

总结下来就是`#`和`##`会打破宏的展开,因此需要借助间接展开的技巧,类似上面的`CONCAT_IMPL`

## 宏调用的识别 `标识符(宏名)后面紧跟左括号`
```cpp
#define MACRO(...) printf(__VA_ARGS__)
// 这些都等价：
MACRO ( arg )   // 空格在括号前
MACRO( arg )    // 空格在括号内  
MACRO(arg)      // 无空格
```

## 可变参数宏`__VA_ARGS__`

```cpp
#define debug(...) printf(__VA_ARGS__)

// 还可以给可变参数取名字__VA_ARGS__ => args
#define debug(format, args...) printf(format, args) // 当可变参数为空时展开为printf(format,) 多余的逗号导致报错
```
特殊处理`#define debug(format, args...) printf(format, ##args)`。其中`##args`在可变参数为空时去除前面多余的逗号

## 处理可变参数宏中的空参数情况 `__VA_OPT__`
`__VA_OPT__`出现是为了解决`__VA_ARGS__`中参数为空的情况
```cpp
#define MACRO(...) __VA_OPT__(内容) __VA_ARGS__
```
- 当 __VA_ARGS__ 非空时，__VA_OPT__(内容) 展开为 内容
- 当 __VA_ARGS__ 为空时，__VA_OPT__(内容) 展开为空

使用`##__VA_ARGS__`是 GCC 扩展，不是标准 C++
```cpp
#if defined(__GNUC__) && __GNUC__ >= 8
# define BOOST_DESCRIBE_PP_UNPACK(...) __VA_OPT__(,) __VA_ARGS__
#else
# define BOOST_DESCRIBE_PP_UNPACK(...) , ##__VA_ARGS__
#endif
```

## 统计宏中的参数个数
```cpp
#define MACRO_COUNT_IMPL(_1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19, _20, _21, _22, _23, \
                                          _24, _25, _26, _27, _28, _29, _30, N, ...) N
#define MACRO_COUNT(...)                                                                                                          \
    MACRO_COUNT_IMPL(__VA_ARGS__, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, \
                                      5, 4, 3, 2, 1, 0)
```
利用变参宏，当`MACRO_COUNT`传入参数，`MACRO_COUNT_IMPL`展开后`N`刚好对应的就是参数个数