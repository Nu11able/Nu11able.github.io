---
title: defmt 一个相对优雅的字符串反序列化
date: 2024-11-06
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

因此只需要参考fmt源码将参数format为字符串的形式，将字符串deformat到与之对应的参数中。相当于format的逆序。易用的关键在于如何自动识别类型
```cpp
维护一个类型map
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
以上通过`type_constant<type>::value`即可获得对应类型的enum值


```cpp
#ifndef DEFMT__BASE__H__
#define DEFMT__BASE__H__

#include <vector>
#include <string_view>
#include <memory>

#ifdef DEFMT_DEBUG
#include <iostream>
#include <typeinfo>
#endif

#include "fast_float/fast_float.h"

namespace defmt {

namespace detail {



// Maps core type T to the corresponding type enum constant.


DEFMT_TYPE_CONSTANT(long long, long_long_type);
DEFMT_TYPE_CONSTANT(unsigned long long, ulong_long_type);
DEFMT_TYPE_CONSTANT(bool, bool_type);
DEFMT_TYPE_CONSTANT(float, float_type);
DEFMT_TYPE_CONSTANT(double, double_type);
DEFMT_TYPE_CONSTANT(std::string, string_type);

// Maps formatting arguments to core types.
// arg_mapper reports errors by returning unformattable instead of using
// static_assert because it's used in the is_formattable trait.
struct arg_mapper {
  auto map(signed char val) -> int { return val; }
  auto map(unsigned char val) -> unsigned { return val; }
  auto map(short val) -> int { return val; }
  auto map(unsigned short val) -> unsigned { return val; }
  auto map(int val) -> int { return val; }
  auto map(unsigned val) -> unsigned { return val; }
  auto map(long val) -> long { return val; }
  auto map(unsigned long val) -> unsigned long { return val; }
  auto map(long long val) -> long long { return val; }
  auto map(unsigned long long val) -> unsigned long long { return val; }
  auto map(bool val) -> bool { return val; }
  auto map(float val) -> float { return val; }
  auto map(double val) -> double { return val; }
  auto map(long double val) -> long double { return val; }
  auto map(std::string val) -> std::string { return val; }

  auto map(...) -> undeformattable { return {}; }
};

using ptr_prototype = void*[2];

struct deformat_arg {
    type arg_type;
    std::byte arg_ptr[sizeof(ptr_prototype)];

    template<typename T>
    T* get() {
        return *reinterpret_cast<T**>(arg_ptr);
    }

    template<typename T>
    deformat_arg(T&& arg) : arg_type(type_constant<decltype(arg_mapper().map(arg))>::value) {
        using arg_ptr_t = std::add_pointer_t<std::remove_reference_t<T>>;
        new (static_cast<void*>(arg_ptr)) std::add_pointer_t<std::remove_reference_t<T>>(std::addressof(arg));
        #ifdef DEFMT_DEBUG
        std::cout << std::addressof(arg) << " " << *reinterpret_cast<arg_ptr_t*>(arg_ptr) << " "
         << typeid(arg).name() << " " << typeid(arg_ptr_t).name() << " " << std::endl;
        #endif
    }
};

fast_float::from_chars_result_t<char> deformat_parse(std::string_view view, deformat_arg& arg) {
  #ifdef DEFMT_DEBUG
    std::cout << "type:" << static_cast<int>(arg.arg_type) << std::endl;
  #endif
  switch (arg.arg_type)
  {
  case type::int_type:
  #ifdef DEFMT_DEBUG
    std::cout << arg.get<int>() << std::endl;
  #endif
    return fast_float::from_chars(view.data(), view.data() + view.size(), *arg.get<int>());
  case type::uint_type:
    return fast_float::from_chars(view.data(), view.data() + view.size(), *arg.get<unsigned>());
  case type::long_long_type:
    return fast_float::from_chars(view.data(), view.data() + view.size(), *arg.get<long long>());
  case type::ulong_long_type:
    return fast_float::from_chars(view.data(), view.data() + view.size(), *arg.get<unsigned long long>());
  case type::float_type:
    return fast_float::from_chars(view.data(), view.data() + view.size(), *arg.get<float>());
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

size_t deformat_parse_fmt_view(std::string_view fmt, std::string_view str, std::vector<std::string_view>& views) {
  size_t nums = 0;
  size_t fmt_len = fmt.size();
  size_t str_len = str.size();

  for (size_t i = 0, j = 0; i < fmt_len && j < str_len; ++i, ++j) {
    if (fmt[i] == '{') {
      size_t start = j;
      while (i < fmt_len && fmt[i] != '}') {
        ++i;
      }
      ++i; // Skip the closing '}'
      if (i == fmt_len) {
        views.push_back(str.substr(start));
        ++nums;
        break;
      }
      while (j < str_len && str[j] != fmt[i]) {
        ++j;
      }
      views.push_back(str.substr(start, j - start));
      ++nums;
    } else if (fmt[i] != str[j]) {
        break; // If characters do not match, break the loop
    }
  }

  return nums;
}

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
}


}


#endif
```