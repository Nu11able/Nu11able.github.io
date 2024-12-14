---
title: Proxy
date: 2024-11-06
tags:
 - c++
 - source read
categories:
 - source read
---

# proxy
看一下官方例子
```cpp
PRO_DEF_MEM_DISPATCH(MemAt, at);

struct Dictionary : pro::facade_builder
    ::add_convention<MemAt, std::string(int)>
    ::build {};

// This is a function, rather than a function template
void PrintDictionary(pro::proxy<Dictionary> dictionary) {
  std::cout << dictionary->at(1) << "\n";
}

int main() {
  static std::map<int, std::string> container1{{1, "hello"}};
  auto container2 = std::make_shared<std::vector<const char*>>();
  container2->push_back("hello");
  container2->push_back("world");
  PrintDictionary(&container1);  // Prints: "hello"
  PrintDictionary(container2);  // Prints: "world"
}
```
首先提出几个问题，而后文章将围绕解答这几个问题展开
- 最终这个`Dictionary`在继承了这么一大串之后`build`得到的类型是什么样的？
- `container1`和`container2`在传递给`PrintDictionary`的时候是如何转换为`pro::proxy<Dictionary>`类型的？
- `pro::proxy<Dictionary>`是如何访问原始类型的at函数的

## PRO_DEF_MEM_DISPATCH(MemAt, at)展开
```cpp
struct MemAt
{
  template <class __T, class... __Args>
  decltype(auto) operator()(__T && __self, __Args &&...__args) noexcept(noexcept(::std::forward<__T>(__self).at(::std::forward<__Args>(__args)...)))
    requires(requires { ::std::forward<__T>(__self).at(::std::forward<__Args>(__args)...); })
  {
    return ::std::forward<__T>(__self).at(::std::forward<__Args>(__args)...);
  }
  template <class __F, class __C, class... __Os>
  struct __declspec(empty_bases) accessor
  {
    accessor() = delete;
  };
  template <class __F, class __C, class... __Os>
    requires(sizeof...(__Os) > 1u && (::std::is_trivial_v<accessor<__F, __C, __Os>> && ...))
  struct accessor<__F, __C, __Os...> : accessor<__F, __C, __Os>...
  {
    using accessor<__F, __C, __Os>::at...;
  };
  template <class __F, class __C, class __R, class... __Args>
  struct accessor<__F, __C, __R(__Args...)>
  {
    __R at(__Args... __args) { return ::pro::proxy_invoke<__C>(::pro::access_proxy<__F>(*this), ::std::forward<__Args>(__args)...); }
  };
// ...
};
```
展开后看到定义了一个MemAt结构体，是一个functor，里面有一堆特化的accessor模板，每个accessor里有一个宏参数传递进来的at函数，
而后又根据`__R(__Args...)`的类型定义了不同的accessor模板，看到这盲猜`__R(__Args...)`和`add_convention<MemAt, std::string(int)>`中的`std::string(int)`有点关系，
至于类型`__F`和`__C`是啥暂时不知道

## 接下来逐步展开 Dictionary
### facade_builder
```cpp
  template <class Cs, class Rs, proxiable_ptr_constraints C>
  struct basic_facade_builder
  {
    template <class D, class... Os>
      requires(sizeof...(Os) > 0u &&
               (details::overload_traits<Os>::applicable && ...))
    using add_indirect_convention = basic_facade_builder<details::add_conv_t<
                                                             Cs, details::conv_impl<false, D, Os...>>,
                                                         Rs, C>;
    template <class D, class... Os>
      requires(sizeof...(Os) > 0u &&
               (details::overload_traits<Os>::applicable && ...))
    using add_convention = add_indirect_convention<D, Os...>;
    using build = details::facade_impl<Cs, Rs, details::normalize(C)>;
    // ...
  };

  using facade_builder = basic_facade_builder<std::tuple<>, std::tuple<>,
                                              proxiable_ptr_constraints{
                                                  .max_size = details::invalid_size,
                                                  .max_align = details::invalid_size,
                                                  .copyability = details::invalid_cl,
                                                  .relocatability = details::invalid_cl,
                                                  .destructibility = details::invalid_cl}>;
// ---------------------------------- 不算华丽的分割线 ----------------------------------
// 下面依次展开Dictionary的模版实例
struct Dictionary : pro::facade_builder
    ::add_convention<MemAt, std::string(int)>
    ::build {};

// Cs: std::tuple<>
// Rs: std::tuple<>
// C: ...
struct Dictionary : basic_facade_builder<
            std::tuple<>, 
            std::tuple<>,
            proxiable_ptr_constraints{
                .max_size = details::invalid_size,
                .max_align = details::invalid_size,
                .copyability = details::invalid_cl,
                .relocatability = details::invalid_cl,
                .destructibility = details::invalid_cl}>
    ::add_indirect_convention<MemAt, std::string(int)> // D: MemAt   Os: std::string(int)
    ::build {};


struct Dictionary :basic_facade_builder<
            details::add_conv_t<
                std::tuple<>, 
                details::conv_impl<false, MemAt, std::string(int)>>,
            std::tuple<>, 
            proxiable_ptr_constraints{
                .max_size = details::invalid_size,
                .max_align = details::invalid_size,
                .copyability = details::invalid_cl,
                .relocatability = details::invalid_cl,
                .destructibility = details::invalid_cl}>
    ::build {};
```

展开到这里还是一脸懵逼，下面展开说说`conv_impl`和`add_conv_t`
#### conv_impl
```cpp
template <bool IS_DIRECT, class D, class... Os>
struct conv_impl
{
    static constexpr bool is_direct = IS_DIRECT;
    using dispatch_type = D;
    using overload_types = std::tuple<Os...>;
    template <class F>
    using accessor = typename D::template accessor<F, conv_impl, Os...>;
};
/* 这时候就可以知道 dispatch_type=MemAt, overload_types=std::tuple<std::string(int)>
回到最初MemAt定义的accessor模板
  template <class __F, class __C, class __R, class... __Args>
  struct accessor<__F, __C, __R(__Args...)> {...}
*/
```
- `__C` = `conv_impl<false, MemAt, std::string(int)>`
- `__R(__Args...)` = `std::string(int)`
- `__F` = ? 暂时还不知道

#### add_conv_t
在展开之前需要先分别对这几个过一遍
- recursive_reduction
- add_tuple_reduction
- instantiated_t
完事后将学会
- 如何合并tuple类型并保持类型不重复

首先看一下定义
```cpp
template <template <class, class> class R, class O, class... Is>
struct recursive_reduction : std::type_identity<O> {};

template <template <class, class> class R, class O, class... Is>
using recursive_reduction_t = typename recursive_reduction<R, O, Is...>::type;

template <template <class, class> class R, class O, class I, class... Is>
struct recursive_reduction<R, O, I, Is...>
{
    using type = recursive_reduction_t<R, R<O, I>, Is...>;
};
/* 乍一看一脸懵逼，再仔细一看更懵逼了
尝试一下如果有一个类型recursive_reduction_t<A, B, C, D, E>展开
recursive_reduction<A, 
    A<B, C>, 
    D, 
    E>

recursive_reduction<A, 
    A<
        A<B, C>, 
        D>, 
    E>

recursive_reduction<A, 
    A<
        A<
            A<B, C>, 
            D>, 
        E>
    >

emm... 不知道干啥的，继续往下看吧...
*/

// ---------------------------------- 分割线 ----------------------------------
template <class O, class I>
struct add_tuple_reduction : std::type_identity<O> {};

template <class... Os, class I>
    requires(!std::is_same_v<I, Os> && ...)
struct add_tuple_reduction<std::tuple<Os...>, I>
    : std::type_identity<std::tuple<Os..., I>> {};
// 如果类型I不存在std::tuple<Os...>中那么将类型I添加到tuple中,最终得到一个新的类型std::tuple<Os..., I>，否则保持std::tuple<Os...>不变
// 保证tuple中的类型没有重复的

// ---------------------------------- 分割线 ----------------------------------
template <template <class...> class T, class TL, class Is, class... Args>
struct instantiated_traits;

template <template <class...> class T, class TL, std::size_t... Is, class... Args>
struct instantiated_traits<T, TL, std::index_sequence<Is...>, Args...> {
    using type = T<Args..., std::tuple_element_t<Is, TL>...>;
};
template <template <class...> class T, class TL, class... Args>
using instantiated_t = typename instantiated_traits<
    T, TL, std::make_index_sequence<std::tuple_size_v<TL>>, Args...>::type;
// 要求TL是一个tuple，尝试展开instantiated_t<A, tuple<B, C, D>, E, F, G>
// 得到类型 A<E, F, G, B, C, D>
// 这玩意有啥用呢？不知道...反正先过一边再说
```
到这里不出意外应该还是一脸懵逼的，上面一堆玩意儿干啥用的...我知道你很急但是你先别急，下面感觉慢慢来了
```cpp
template <class T, class U>
using add_tuple_t = typename add_tuple_reduction<T, U>::type;
// 将U添加进tuple类型T中,前提是tuple T中不存在类型U

template <class O, class... Is>
using merge_tuple_impl_t = recursive_reduction_t<add_tuple_t, O, Is...>;
/* 尝试展开merge_tuple_impl_t<tuple<int>, float, string, int, string>
recursive_reduction<add_tuple_t, tuple<int>, float, string, int, string>
recursive_reduction<
    add_tuple_t, 
    add_tuple_t<tuple<int>, float>, 
    string, int, string>

recursive_reduction<
    add_tuple_t, 
        add_tuple_t<
            add_tuple_t<tuple<int>, float>, 
            string
        >
    int, string>
也就是说递归的将类型float, string, int, string用add_tuple_t添加到tuple<int>中,最终得到tuple<int, float, string>
所以recursive_reduction<R, O, Is...>相当于是用R将每个Is...一个一个的添加到O里面去
兄弟萌感觉来了┗|｀O′|┛ 嗷~~
*/

template <class T, class U>
using merge_tuple_t = instantiated_t<merge_tuple_impl_t, U, T>;
/*
上面已经知道merge_tuple_impl_t的作用，这里又包了一层instantiated_t是干啥的呢, 假设U=tuple<int, string>
展开instantiated_t<merge_tuple_impl_t, tuple<int, string>, T>
得到merge_tuple_impl_t<T, int, string>
从上面的定义又知道T也得是一个tuple，所以merge_tuple_t相当于将两个tuple T U合并为一个tuple并且保持类型没有重复
我的天呐щ(ʘ╻ʘ)щ
*/
```
换个简单点例子感受一下`instantiated_t`是一个什么样的角色
```cpp
int operation(int(*op)(int, int), int a, int b) {
    return op(b, a)
}

int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

int main() {
    operation(add, 1, 2);
    operation(sub, 1, 2);
}
```
instantiated_t相当于operation，只不过operation是对值进行计算而instantiated_t是对类型进行计算，这个区别刚好也是普通编程和模板元编程的区别(不太确定这么区分有没有问题,大佬们轻点喷)。
OK，再来总结一遍
- recursive_reduction<R, O, Is...>相当于是用R将每个Is...一个一个的添加到O里面去
- instantiated_t<T, TL, Args...> TL是一个tuple，将TL每个类型展开组合成新的类型 T<Args..., TL...>


```cpp

template <bool IS_DIRECT, class D>
struct merge_conv_traits {
    template <class... Os>
    using type = conv_impl<IS_DIRECT, D, Os...>;
};
template <class C0, class C1>
using merge_conv_t = instantiated_t<
    merge_conv_traits<C0::is_direct, typename C0::dispatch_type>::template type,
    merge_tuple_t<typename C0::overload_types, typename C1::overload_types>>; 
/* merge_tuple_t得到一个tuple，里面是去重后的函数类型
 也就是说merge_conv_t将两个conv_impl类型C1的overload_types合并到C0的overload_types中
 */

// 基础模板定义
template <class Cs0, class C1, class C>
struct add_conv_reduction;

// 第二个tuple模板参数不是空的，则递归的将其添加到第一个tuple中
template <class... Cs0, class C1, class... Cs2, class C>
struct add_conv_reduction<std::tuple<Cs0...>, std::tuple<C1, Cs2...>, C>
    : add_conv_reduction<std::tuple<Cs0..., C1>, std::tuple<Cs2...>, C>
{};
/*
为什么不直接这样呢一步到位而是一个个添加？
template <class... Cs0, class C1, class... Cs2, class C>
struct add_conv_reduction<std::tuple<Cs0...>, std::tuple<C1, Cs2...>, C>
    : add_conv_reduction<std::tuple<Cs0..., C1, Cs2...>, std::tuple<>, C> {};
*/

/* 如果C与C1的is_direct和dispatch_type相等，则合并它们两个，这里也就可以回答上面的问题了，
因为需要对各个conv_impl进行合并*/
template <class... Cs0, class C1, class... Cs2, class C>
    requires(C::is_direct == C1::is_direct && std::is_same_v<
                                                typename C::dispatch_type, typename C1::dispatch_type>)
struct add_conv_reduction<std::tuple<Cs0...>, std::tuple<C1, Cs2...>, C>
    : std::type_identity<std::tuple<Cs0..., merge_conv_t<C1, C>, Cs2...>>
{};
template <class... Cs, class C>
struct add_conv_reduction<std::tuple<Cs...>, std::tuple<>, C>
    : std::type_identity<std::tuple<Cs..., merge_conv_t<
                                                conv_impl<C::is_direct, typename C::dispatch_type>, C>>>
{};
// 最终add_conv_reduction得到的是一个tuple，里面的类型是按照is_direct和dispatch_type进行合并后的conv_impl

template <class Cs, class C>
using add_conv_t = typename add_conv_reduction<std::tuple<>, Cs, C>::type;
```
下面直接带入最开始的模板参数
```cpp
details::add_conv_t<
    std::tuple<>, // Cs = void
    details::conv_impl<false, MemAt, std::string(int)>> // C

add_conv_reduction<std::tuple<>, std::tuple<>, conv_impl<false, MemAt, std::string(int)>>::type;

add_conv_reduction<std::tuple<>, std::tuple<>, conv_impl<false, MemAt, std::string(int)>>
    : std::type_identity<
        std::tuple<
            merge_conv_t<
                conv_impl<C:false, MemAt>, // C0
                conv_impl<false, MemAt, std::string(int)> // C1
            >
        >
    >::type;

add_conv_reduction<std::tuple<>, std::tuple<>, conv_impl<false, MemAt, std::string(int)>>
    : std::type_identity<
        std::tuple<
            instantiated_t<
                merge_conv_traits<C:false, MemAt>::type,
                std::tuple<std::string(int)>>
            >
        >
    >::type;

//最终得到
tuple<conv_impl<false, MemAt, std::string(int)>>
// emm...绕了一大圈又回来了... 有一种脱裤子放屁的感觉
```
下面我们终于可以得到Dictionary的定义了

```cpp
template <class Cs, class Rs, proxiable_ptr_constraints C>
struct facade_impl
{
    using convention_types = Cs;
    using reflection_types = Rs;
    static constexpr proxiable_ptr_constraints constraints = C;
};

struct Dictionary :facade_impl<
            std::tuple<
                details::conv_impl<false, MemAt, std::string(int)>
            >, 
            std::tuple<>, 
            details::normalize(
                proxiable_ptr_constraints{
                .max_size = details::invalid_size,
                .max_align = details::invalid_size,
                .copyability = details::invalid_cl,
                .relocatability = details::invalid_cl,
                .destructibility = details::invalid_cl}
            )> {};

struct Dictionary :facade_impl<
            std::tuple<
                details::conv_impl<false, MemAt, std::string(int)>
            >, 
            std::tuple<>, 
            proxiable_ptr_constraints{
                .max_size = sizeof(void *[2]),
                .max_align = sizeof(void *[2]),
                .copyability = constraint_level::none,
                .relocatability = constraint_level::nothrow,
                .destructibility = constraint_level::nothrow}
            > {};
/* 也就是说facade_impl中
convention_types = std::tuple<details::conv_impl<false, MemAt, std::string(int)>>
reflection_types = std::tuple<>
constraints = proxiable_ptr_constraints{
                .max_size = sizeof(void *[2]),
                .max_align = sizeof(void *[2]),
                .copyability = constraint_level::none,
                .relocatability = constraint_level::nothrow,
                .destructibility = constraint_level::nothrow}
*/
```
## 接下来看看类型pro::proxy\<Dictionary\>
- pro::proxy\<Dictionary\>
- facade_traits
- facade_conv_traits_impl
- composite_accessor

```cpp
template <class F>
class proxy : public details::facade_traits<F>::direct_accessor {
static_assert(facade<F>);
friend struct details::proxy_helper<F>;
using _Traits = details::facade_traits<F>;
public:
// ...
template <class P>
proxy(P &&ptr) noexcept(std::is_nothrow_constructible_v<std::decay_t<P>, P>)
    requires(proxiable<std::decay_t<P>, F> &&
            std::is_constructible_v<std::decay_t<P>, P>) {
    initialize<std::decay_t<P>>(std::forward<P>(ptr));
}
// ...
auto operator->() noexcept
    requires(_Traits::has_indirection) {
    return std::addressof(ia_);
}
private:
template <class P, class... Args>
P &initialize(Args &&...args)
{
    std::construct_at(reinterpret_cast<P *>(ptr_), std::forward<Args>(args)...);
    meta_ = details::meta_ptr<typename _Traits::meta>{std::in_place_type<P>};
    return *std::launder(reinterpret_cast<P *>(ptr_));
}

[[___PRO_NO_UNIQUE_ADDRESS_ATTRIBUTE]]
typename _Traits::indirect_accessor ia_;
details::meta_ptr<typename _Traits::meta> meta_;
alignas(F::constraints.max_align) std::byte ptr_[F::constraints.max_size]; // 一个P类型的指针
};

/* 当将一个std::map类型的container赋值给proxy<Dictionary>的时候，proxy的内部会保留原始值的指针，
当调用->运算符的时候返回的是facade_traits<Dictionary>::indirect_accessor类型的成员变量ia_*/

//下面一步步展开facade_traits<Dictionary>
struct facade_traits<F>
    : instantiated_t<facade_conv_traits_impl, typename F::convention_types, F>,
        instantiated_t<facade_refl_traits_impl, typename F::reflection_types, F> {/*...*/};

struct facade_traits<Dictionary>
    : facade_conv_traits_impl<Dictionary, details::conv_impl<false, MemAt, std::string(int)>>,
        facade_refl_traits_impl<Dictionary> {/*...*/};


struct facade_conv_traits_impl<
    Dictionary, 
    details::conv_impl<false, MemAt, std::string(int)>
> : applicable_traits {
    using conv_meta = composite_meta<typename conv_traits<Cs>::meta...>;
    using indirect_accessor = composite_accessor<false, Dictionary, conv_impl<false, MemAt, std::string(int)>>;
    using direct_accessor = composite_accessor<true, Dictionary, conv_impl<false, MemAt, std::string(int)>>;
    // ......
};



template <class... As>
class ___PRO_ENFORCE_EBO composite_accessor_impl : public As... {
    template <class>
    friend class pro::proxy;
    // ...
    // composite_accessor_impl 继承As... 没有多余的操作，那么As...又是一些什么东西呢？
};

template <template <class> class TA, class O, class I>
struct composite_accessor_reduction : std::type_identity<O> {}; // 有没有感觉和前面的recursive_reduction有点类似

template <template <class> class TA, class... As, class I>
    requires(requires { typename TA<I>; } && std::is_trivial_v<TA<I>> && !std::is_final_v<TA<I>>)
struct composite_accessor_reduction<TA, composite_accessor_impl<As...>, I> {
    using type = composite_accessor_impl<As..., TA<I>>;
};

template <bool IS_DIRECT, class F>
struct composite_accessor_helper {
    template <class C>
    requires(C::is_direct == IS_DIRECT)
    using single_accessor = typename C::template accessor<F>;
    template <class O, class I>
    using reduction_t =
        typename composite_accessor_reduction<single_accessor, O, I>::type;
    /* TA = single_accessor
     composite_accessor_reduction将single_accessor<I>添加到composite_accessor_impl的As模版本参数列表中
     推断出composite_accessor_impl是由一系列single_accessor<I>，到这里暂时还不知道I是什么类型
     */
};
template <bool IS_DIRECT, class F, class... Cs>
using composite_accessor = recursive_reduction_t<
    composite_accessor_helper<IS_DIRECT, F>::template reduction_t,
    composite_accessor_impl<>, Cs...>;

//带入实参
using composite_accessor = recursive_reduction_t<
    composite_accessor_helper<false, Dictionary>::template reduction_t, // IS_DIRECT=false  F=Dictionary
    composite_accessor_impl<> // O
    conv_impl<false, MemAt, std::string(int)> // I
>;
/* 推断出single_accessor<I> =
 conv_impl<false, MemAt, std::string(int)>::accessor<Dictionary> =
 MemAt::accessor<Dictionary, conv_impl<false, MemAt, std::string(int)>, std::string(int)>
*/
```
我们终于得到了成员变量ia_的类型_Traits::indirect_accessor =  MemAt::accessor<Dictionary, conv_impl<false, MemAt, std::string(int)>, std::string(int)>
**调用proxy<Dictionary>的at函数相当于调用MemAt的accessor模板中的at函数**
现在回到MemAt中的accessor模板,！！只差最后的proxy_invoke和access_proxy我们就通关了！！( •̀ ω •́ )✧
从名字大概就可以推断出access_proxy获取自身的proxy，拿到自身的proxy后通过ptr_指针调用原始类型的at函数(其实中间还要经过一层MemAt)
```cpp
template <class F>
struct proxy_helper {
    static inline const auto &get_meta(const proxy<F> &p) noexcept {
        return *p.meta_.operator->();
    }

    template <class C, qualifier_type Q, class... Args> // C=conv_impl<false, MemAt, std::string(int)>
    static decltype(auto) invoke(add_qualifier_t<proxy<F>, Q> p, Args &&...args) {
        using OverloadTraits = typename conv_traits<C>::template matched_overload_traits<Q, Args...>; // overload_traits_impl<qualifier_type::lv, false, std::string, int>
        auto dispatcher = p.meta_->template dispatcher_meta<typename OverloadTraits ::template meta_provider<C::is_direct, typename C::dispatch_type>>::dispatcher;
        /* dispatcher = p.meta_->template dispatcher_meta<
                overload_traits_impl<qualifier_type::lv, false, std::string, int>::meta_provider<false, MemAt>
            >::dispatcher
        */
        if constexpr (C::is_direct && OverloadTraits::qualifier == qualifier_type::rv) {
            meta_ptr_reset_guard guard{p.meta_};
            return dispatcher(std::forward<add_qualifier_t<std::byte, Q>>(*p.ptr_), std::forward<Args>(args)...); // 到这里基本可以猜到调用的就是MemAt的operator()了
        }
        else {
            return dispatcher(std::forward<add_qualifier_t<std::byte, Q>>(*p.ptr_), std::forward<Args>(args)...);
        }
    }

    template <class A, qualifier_type Q> // A = MemAt::accessor<Dictionary, conv_impl<false, MemAt, std::string(int)>, std::string(int)>
    static add_qualifier_t<proxy<F>, Q> access(add_qualifier_t<A, Q> a) { 
        if constexpr (std::is_base_of_v<A, proxy<F>>) { // proxy<F> 继承自A则直接将a向下转换得到proxy
            return static_cast<add_qualifier_t<proxy<F>, Q>>(
                std::forward<add_qualifier_t<A, Q>>(a));
        }
        else {
            return reinterpret_cast<add_qualifier_t<proxy<F>, Q>>(
                *(reinterpret_cast<add_qualifier_ptr_t<std::byte, Q>>(
                    static_cast<add_qualifier_ptr_t<
                        typename facade_traits<F>::indirect_accessor, Q>>(
                        std::addressof(a))) -
                offsetof(proxy<F>, ia_))); // 虽然看起来很复杂 其实只是通过a的地址减去ia_在proxy中的地址偏移来得到自身的proxy
        }
    }
};

template <class F, class A>
proxy<F> &access_proxy(A &a) noexcept {
    return details::proxy_helper<F>::template access<
        A, details::qualifier_type::lv>(a);
}

template <class C, class F, class... Args>
decltype(auto) proxy_invoke(proxy<F> &p, Args &&...args) {
    return details::proxy_helper<F>::template invoke<
        C, details::qualifier_type::lv>(p, std::forward<Args>(args)...);
}

```

