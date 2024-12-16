---
title: defmt 一个相对优雅的字符串反序列化
date: 2024-11-16
tags:
 - c++
categories:
 - c++
---

# [defmt](git@github.com:Nu11able/defmt.git)
## [fmt](https://github.com/fmtlib/fmt.git) 
fmt是一个相当易用且高效的现代c++库，已被引入c++20标准，省去了繁琐低效的iostream和c风格的字符串格式化符号(%d， %s...)
用法类似于python的字符串格式化，以下是各个字符串序列化的一个例子：
```cpp
#include <iostream>
#include <sstream>
#include <string>
#include <vector>
#include <cstdio>
#include "fmt/format.h"
#include "fmt/ranges.h"

using namespace std;

struct CustomType {
    string name;
    int score;
};

template <> struct fmt::formatter<CustomType> : public fmt::formatter<string> {
  auto format(const CustomType& value, format_context& ctx) const {
    return fmt::format_to(ctx.out(), "{}:{}", value.name, value.score);
  }
};

int main() {
    string name{ "kevin" };
    int score = 99;
    
    string cstyle_str(50, '\0');
    snprintf(&cstyle_str[0], cstyle_str.size(), "%s:%d\n", name.c_str(), score); // c style
    cout << "cstyle_str:"<< cstyle_str;
    
    stringstream ss;
    ss << name << ":" << score << "\n"; // stringstream
    string ss_str = ss.str();
    cout << "ss_str:" << ss_str;

    string fmt_str = fmt::format("{}:{}\n", name, score); // fmt
    cout << "fmt_str:" << fmt_str;

    std::vector<CustomType> vec = {{"kevin", 99}, {"jane", 100}};
    cout << fmt::format("vec[0]->{}\n", vec[0]);
    cout << fmt::format("standard container: {}\n", vec);
    return 0;
}
/*
cstyle_str:kevin:99
ss_str:kevin:99
fmt_str:kevin:99
vec[0]->kevin:99
standard container: [kevin:99, jane:100]
*/
```

可见fmt相当的方便，那字符串的反序列化是否也能够如此方便呢

## [defmt](git@github.com:Nu11able/defmt.git)
在参考了fmt的源代码之后(具体分析可见[fmt源码分析]())。简单来说fmt内部维护了一个map，做了一个类型到与之对应的序列化函数的映射。

因此只需要参考fmt源码将参数format为字符串的形式，将字符串deformat到与之对应的参数中。相当于format的逆序。
尝试构建以下接口`deformat(std::string_view fmt, std::string_view str, Args&&... args)`，将`str`按照`fmt`的格式反序列化，`args`用于接收反序列化结果。
```cpp
template <typename... Args>
?? deformat(std::string_view fmt, std::string_view str, Args&&... args) {
  // static_assert((!std::is_lvalue_reference_v<Args> || ...), "deformat arguments must not be lvalues");
  // ......
}
```

易用的关键在于如何自动识别类型
```cpp
// 维护一个类型map
enum class type {
  none_type,
  int_type,
  // ...
  string_type,
};


struct undeformattable {};
template <typename T>
struct type_constant : std::integral_constant<type, type::none_type> {};

#define DEFMT_TYPE_CONSTANT(Type, constant) \
  template <> struct type_constant<Type>        \
      : std::integral_constant<type, type::constant> {}

DEFMT_TYPE_CONSTANT(int, int_type);
DEFMT_TYPE_CONSTANT(unsigned, uint_type);
```
以上通过`type_constant<type>::value`即可获得对应类型的enum值, 将类型值与参数地址绑定，方便后续获取解析结果
```cpp
using ptr_prototype = void*[2];

struct deformat_arg {
    type arg_type;
    std::byte arg_ptr[sizeof(ptr_prototype)]; // 存放参数地址，以便将结果反序列化到参数中

    template<typename T>
    T* get() {
        return *reinterpret_cast<T**>(arg_ptr);
    }

    template<typename T>
    deformat_arg(T&& arg) : arg_type(type_constant<decltype(arg_mapper().map(arg))>::value) {
        using arg_ptr_t = std::add_pointer_t<std::remove_reference_t<T>>;
        new (static_cast<void*>(arg_ptr)) arg_ptr_t(std::addressof(arg));
    }
};
```
将传入参数`args`做统一封装
```cpp
template<typename T>
deformat_arg make_deformat_arg(T&& arg) {
  return deformat_arg(arg);
}

template<size_t NUM_ARGS>
struct deformat_arg_store {
    deformat_arg de_args[NUM_ARGS];

    template<typename ...Args>
    deformat_arg_store(Args&&...args) : de_args{make_deformat_arg(std::forward<Args>(args))...} {}
};

template<typename... Args, size_t NUM_ARGS = sizeof...(Args)>
auto make_deformat_args(Args&&... args) -> deformat_arg_store<NUM_ARGS> {
    return deformat_arg_store<NUM_ARGS>(std::forward<Args>(args)...);
}
```
因此当调用`make_deformat_args(args...)`时，将args统一封装到`deformat_arg_store`中，内部含有`deformat_arg`数组。到此形式已经统一，已经知道了参数的类型和保存结果的地址，
接下来只需要解析字符串即可，借助[fast_float]()。
```cpp
fast_float::from_chars_result_t<char> deformat_parse(std::string_view view, deformat_arg& arg) {
  switch (arg.arg_type)
  {
  case type::int_type:
    return fast_float::from_chars(view.data(), view.data() + view.size(), *arg.get<int>());
  // ......
  case type::double_type:
    return fast_float::from_chars(view.data(), view.data() + view.size(), *arg.get<double>());
  case type::string_type:
    *arg.get<std::string>() = std::string(view);
    break;
  default:
    break;
  }
  return {};
}
```
最后就只剩下将`str`按照`fmt`的格式解析，然后依次调用`deformat_parse`将结果保存到`args`中
```cpp
size_t deformat_parse_fmt_view(std::string_view fmt, std::string_view str, std::vector<std::string_view>& views);


template <size_t N>
fast_float::from_chars_result_t<char> vdeformat(std::string_view fmt, std::string_view str, detail::deformat_arg_store<N> args) {
  std::vector<std::string_view> views;
  size_t num = detail::deformat_parse_fmt_view(fmt, str, views);
  for (size_t i = 0; i < num; ++i) {
    detail::deformat_parse(views[i], args.de_args[i]);
  }
  return {};
}


template <typename... Args>
fast_float::from_chars_result_t<char> deformat(std::string_view fmt, std::string_view str, Args&&... args) {
  return vdeformat(fmt, str, detail::make_deformat_args(std::forward<Args>(args)...));
}
```

## Example
```cpp
#include <iostream>
#include <string>
#include "defmt/defmt.hpp"

using namespace std;

int main() {
    string str{"hello kevin, your score is 99. Good job!"};
    std::string name;
    int score = 0;
    auto ret = defmt::deformat("hello {}, your score is {}. Good job!", str, name, score);
    if (ret.ec != std::errc())
        cout << "deformat failed!" << endl;
    else
        cout << name << ":" << score << endl;
    return 0;
}
```