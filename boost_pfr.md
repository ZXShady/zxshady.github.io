[Boost::PFR](https://github.com/boostorg/pfr) is a header-only library that allows programmers to treat aggregate classes and structs as tuples. This means you can access members by index, iterate over fields, perform operations such as serialization (to json,binary),and apply a function for each member without explicitly defining these behaviors manually.

For example, given a simple structure:

```cpp
#include <iostream>
#include <boost/pfr.hpp>

struct Person {
    std::string name;
    int age;
};

int main() {
    Person p{"Bob", 30};
    std::cout << boost::pfr::get<0>(p) << '\n';
    std::cout << boost::pfr::get<1>(p) << '\n';
}
```

In this article I will show you how it works under the hood (for C++17 plus, C++14 impl is a bit different) because I could not find resources online, So I hope it serves as a useful reference.

# Goal

At a high level, Boost::PFR's job is just converting an aggregate type to a `std::tuple` of references.

```cpp
struct A {
    int x;
    float y;
};

A a{1, 2.0f};

std::tuple<int&, float&> tuple = as_tuple(a);
```

Once we can do that, many other features become trivial. Like

`boost::pfr::get<N>(obj)` is just an alias to `std::get<N>(as_tuple(obj))`.
Iterating over every member is just iterating over the tuple.
Serialization is just serializing the tuple.

We need a way to name members without knowing their names. something like `a.[0]`,`a.[1]` and structured bindings are exactly like that.

```cpp
auto [a,b] = A{1,2.0f};

// equal to 
auto _Element = A{1,2.0f};
auto&& a = _Element.x;
auto&& b = _Element.y;
```

With that info we can simply implement

```cpp
template<typename T>
auto as_tuple(T&& agg)
{
    if constexpr(requires {auto&& [a] = agg;})
    {
        auto&& [a] = agg;
        return std::tie(a);
    }
    else if constexpr(requires {auto&& [a,b] = agg;})
    {
        auto&& [a,b] = agg;
        return std::tie(a,b);
    }
    // continue to add more
}

auto tuple = as_tuple(aggregate);
```

Except you will get an error message that says `requires` clauses require expressions and not statements (which structured bindings are).

So we need to know the amount of members to select an appropriate length structured binding list.

After carefully studying the million ways of initializing C++ variables specificially aggregate init allows you to do exactly that

```cpp
// given a struct
struct X { 
    T m0, m1, mN...;
};

// you can construct it with an initializer list for each member
auto x = X{e0, e1, eN...};
```

Initialization is SFINAE friendly so we can test for it

```cpp
template<typename T>
constexpr size_t count_fields() 
{
    if constexpr(requires {T{0,0,0};}) return 3;
    else if constexpr(requires {T{0,0};}) return 2;
    else if constexpr(requires {T{0};}) return 1;
    return -1;
}

struct X {int a,b;};

static_assert(count_fields<X>() == 2);
```

We can use this now to select the structured binding declaration

```cpp
template <typename T>
auto as_tuple(T& agg) {
    constexpr size_t fields = count_fields<T>();
    static_assert(fields != -1,"Failed to count fields");
    if constexpr (fields == 1) {
        auto&& [a] = agg;
        return std::tie(a);
    }
    else if constexpr (fields == 2) {
        auto&& [a, b] = agg;
        return std::tie(a, b);
    }
    else if constexpr (fields == 3) {
        auto&& [a, b, c] = agg;
        return std::tie(a, b, c);
    }
    // add more
}
X x;
std::tuple<int&,int&> tuple = as_tuple(x);
```

We got a very basic Boost PFR implementation, but we can do better since it currently fails for any `struct` that doesn't contain types implicitly constructible from 0.

To fix this, we need a universal expression something that can satisfy any constructor or initialization.

In C++, we can exploit template conversion operators to create a type that can implicitly convert to anything.

```cpp
struct anything 
{
    template<typename T>
    operator T();
    // No implementation needed since it is only used in unevaluated contexts
};
```

Because `anything` claims it can turn into any type `T`, we can use instances of it as our universal expression. Let's rewrite the field counter

```cpp
template<typename T>
constexpr size_t count_fields() 
{
    if constexpr (requires { T{anything{}, anything{}, anything{}}; }) return 3;
    else if constexpr (requires { T{anything{}, anything{}}; }) return 2;
    else if constexpr (requires { T{anything{}}; }) return 1;
    return -1;
}
```

and now 

```cpp

struct Person 
{
    std::string name;
    int age;
};

Person p;
std::tuple<std::string&,int&> tuple = as_tuple(p); // works 
```


Hardcoding a sequential if constexpr ladder works fine , but in the real world structs might have alot more fields. Writing a linear if constexpr to count 100 fields is not practical. We can count the number of fields dynamically using `std::index_sequence`to generate the initialization expression to check for combined with template recursion


```cpp
struct anything 
{
    anything(size_t); // to allow simple pack expansion
    template<typename T>
    operator T();
}

template<typename T,size_t ... Is>
constexpr bool count_feidls_impl(std::index_sequence<Is...>) 
{
    return requires { T{anything(Is)...};};
}

template<typename T,size_t N = 256> // count down
constexpr size_t count_fields()
{
    if constexpr(count_fields_impl<T>(std::make_index_sequence<N>{}))
        return N;
    else if constexpr(N == 0)// if we cannot find anything
        return -1;
    else // otherwise recurse with trying 1 less 
        return count_fields<T,N-1>();
}

#define BIND_CASE(N, ...) \
    else if constexpr (fields == N) { \
        auto&& [__VA_ARGS__] = agg; \
        return std::tie(__VA_ARGS__); \
    }

template <typename T>
constexpr auto as_tuple(T& agg) {
    constexpr size_t fields = count_fields<T>();
    
    if constexpr (fields == 0) {
        return std::tuple<>{};
    }
    // use a python script 
    /*
    for i in range(1,256):
        list = []
        for j in range(1,i):
            list.append(f"n{j}")
        print(f"BIND_CASE({i},{list})")
    */
    BIND_CASE(1,v0)
    BIND_CASE(2,v0,v1)
    BIND_CASE(3,v0,v1,v2)
    ... 
    else {
        static_assert(sizeof(T) == 0, "Unsupported field count. max is 256");
    }
}
#undef BIND_CASE
```

The language specifies that a structured binding declaration must have a fixed, explicitly known number of identifiers.
Because of this limitation, the compiler has to know the exact number of elements before it can bind them. This leaves us forced to write a python script that hardcodes each structured binding list up to 256 elements.[^1] 


## Compile times
Lets test its compile times.

```cpp
struct Benchmark {
    int f1,f2,f3,f4,f5,f6,f7,f8,f9;
};

int main() {
    Benchmark b{};
    
    auto t = as_tuple(b);
}
```


Running this with `clang++ -ftime-trace test.cpp` gives me this flamegraph 

[Flame Graph showing count_fields taking ~750 ms of the compilation](./pics/boost_pfr_reimpl_flamegraph1.PNG)


Well like optimizing anything we should make some assumptions about data, Why do we start from the max (which is high) and then count down, most structs are less than 256 members we should instead count up.

```cpp
template<typename T, size_t N = 1>
constexpr size_t count_fields()
{
    if constexpr (count_fields_impl<T>(std::make_index_sequence<N>{}))
        // If it successfully initializes with N, we know there are at least N fields.
        // Try the next higher number.
        return count_fields<T, N + 1>();
    else
        // The first number that fails to initialize means the actual field count 
        // is exactly one step below it.
        return N - 1;
}
```

Let's trace how this changes the work the compiler has to do. For our 9 member struct, the compiler starts at N = 1 and counts up. It successfully instantates 1 through 9, and then stops at N = 10 since you can't initialize a 9 field struct with 10 expressions.

Comparing this to our previous count top to down implementation, which forced the compiler to recursively instantiate and discard more than 200 failed checks before hitting its first successful match the flame graph is much nicer

[Flame Graph showing count_fields taking ~4 ms of the compilation](./pics/boost_pfr_reimpl_flamegraph2.PNG)

`std::tuple` itself takes ~10 ms of the compilation, we could make it less this by making our own simple tuple since `std::tuple` has horrible compilation times since it uses recursion for implementation in some implemenations [^2], but I am not going to dive into that.

-----------------------

Okay we now extracted them into a tuple, how to get the member names now?

since C++98 you have `__FILE__` and `__LINE__` macro they expand to the current file and line number respectively there also exists `__func__` which expands to the current function name only. Compilers added more like `__PRETTY_FUNCTION__` for clang/gcc and `__FUNCSIG__` for msvc which expand to the function name with its parameters, return type and template arguments.

```cpp
1. int main() 
2. {
3.    __FILE__; // foo.cpp
4.    __LINE__;  // 4
5.    __func__; // "main"
6.    __PRETTY_FUNCTION__; // "int main()"
7. }
```


Let's try printing a auto template argument to see exactly how the compiler handle formatting

```cpp
template<auto V>
auto signature()
{
    #if defined(__clang__) || defined(__GNUC__)
    return __PRETTY_FUNCTION__;
    #else 
    return __FUNCSIG__;
    #endif
}

int main()
{
    std::cout << signature<1>() << '\n';
    std::cout << signature<nullptr>() << '\n';
}
```

Lets try using a member address. Since the address of a global variable is constant at compile time, it can be passed as a template argument.

```cpp
struct X {int member;};
X x;
int main()
{
    std::cout << signature<&x>() << '\n';
    std::cout << signature<&x.member>() << '\n';
}
```

See at [godbolt](https://godbolt.org/z/T3W9hPc8c)


| Thing | GCC | Clang | MSVC |
| --- | --- | --- | --- |
| `1` | `auto signature() [with auto V = 1]` | `auto signature() [V = 1]` | `auto __cdecl signature<0x1>(void)` |
| `nullptr` | `auto signature() [with auto V = nullptr]` | `auto signature() [V = nullptr]` | `auto __cdecl signature<nullptr>(void)` |
| `&x` | `auto signature() [with auto V = (& x)]` | `auto signature() [V = &x]` | `auto __cdecl signature<&x>(void)` |
| `&x.member` | `auto signature() [with auto V = (& x.X::member)]` | `auto signature() [V = &x.member]` | `auto __cdecl signature<&x.member>(void)` |


as you noticed GCC explicitly includes the fully qualified class name prefix `.X::member` when pointing to a field. Your parser will need to drop everything from the dot up to the double colon `.X::` to get the field name. This is more complex to parse since it requires getting the name of the class.

Clang and MSVC generate `&x.member`, meaning you can simply scan for the `.` character and copy everything until the closing delimiter `]` or `>` to retrieve the field the name.

I will continue using Clang as it is the simplest out of the three compilers in terms of format.


```cpp
template<auto M>
constexpr auto get_member_name_impl() {
    // the format is "auto get_member_name_impl() [M = &x.__MEMBER_NAME__]"
    constexpr auto sig = std::string_view(__PRETTY_FUNCTION__);
    auto sub = sig.substr(sig.rfind('.') + 1); // go past the dot
    sub.remove_suffix(1); // remove the `]`
    return sub;
}

template<typename T>
const T global{}; 

template<typename T,size_t Idx>
constexpr auto member_name()
{
    constexpr auto tuple = as_tuple(global<T>);
    return get_member_name_impl<&std::get<Idx>(tuple)>();
}
struct Person 
{
    std::string name;
    int age;
};

static_assert(member_name<Person,0>() == "name");
static_assert(member_name<Person,1>() == "age");
```

Thats great lets try another struct

```cpp
struct Entity
{
    std::reference_wrapper<Entity> target;
    int health;
    int speed;
};

static_assert(member_name<Entity,0>() == "target");
static_assert(member_name<Entity,1>() == "health");
static_assert(member_name<Entity,2>() == "speed");
```

```js
 error: no matching constructor for initialization of 'std::reference_wrapper<Entity>'
  320 | const T global{};
      |                ^
```

Well it does not work, since `Entity` has no default constructor since it contains `std::reference_wrapper` we cannot create `global` which is required for name extraction. We don't really need the `global` to exist though so we can instead mark it as extern and not ODR use it, which fixes it.

```cpp
// Disable annoying warnings about not intentionally defining the variable template 
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundefined-var-template"

template<typename T>
extern const T global; // we promise it to exist somewhere someday, we lie though :)

template<typename T,size_t Idx>
constexpr auto member_name()
{
    constexpr auto tuple = as_tuple(global<T>);
    return get_member_name_impl<&std::get<Idx>(tuple)>();
}
#pragma clang diagnostic pop

static_assert(member_name<Entity,0>() == "target"); // works
```


Let's try adding another member to our Entity class 

```cpp
struct Entity
{
    std::reference_wrapper<Entity> target;
    int health;
    std::reference_wrapper<Entity> teammate;
    int speed;
};
static_assert(member_name<Entity,0>() == "target"); // nope
```

We get this error
```js
error: static assertion failed due to requirement '_Always_false<std::integral_constant<unsigned long long, 0>>': tuple index out of bounds
  657 |     static_assert(_Always_false<integral_constant<size_t, _Index>>, "tuple index out of bounds");
      |                   ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Deciphering it it means that our `as_tuple` function returned an empty tuple (since how else would you be an out of index with index 0) which means that it failed to count the member fields.

Lets mentally backtrace `count_fields`. 

```cpp
template<typename T, size_t N = 1>
constexpr size_t count_fields()
{
    if constexpr (count_fields_impl<T>(std::make_index_sequence<N>{}))
        // If it successfully initializes with N, we know there are at least N fields.
        // Try the next higher number.
        return count_fields<T, N + 1>();
    else
        // The first number that fail to initialize means the actual field count 
        // is exactly one step below it.
        return N - 1;
}
```

Since the class `Entity` after adding a reference wrapper is not default constructible when you get to `count_fields_impl<T>(std::make_index_sequence<2>{})` it does `requires {T {anything{},anything{}}}` since field 3 (the ref wrapper) is not default constructible it returns false, exiting the loop earlier than it should, if we bring back the count down approach it works, but we don't want to pay the compile time cost for that unless we need it therefore we can conditionally dispatch it

```cpp
template<typename T, size_t N = 256>
constexpr size_t count_fields_count_down_impl()
{
    if constexpr(count_fields_impl<T>(std::make_index_sequence<N>{}))
        return N;
    else if constexpr(N == 0)// if we cannot find anything
        return -1;
    else // otherwise recurse with trying 1 less 
        return count_fields_count_down_impl<T,N-1>();
}

template<typename T, size_t N = 1>
constexpr size_t count_fields_count_up_impl()
{
    if constexpr (count_fields_impl<T>(std::make_index_sequence<N>{}))
        return count_fields_count_up_impl<T, N + 1>();
    else
        return N - 1;
}

template<typename T>
constexpr size_t count_fields()
{
    if constexpr (std::is_default_constructible_v<T>)
        return count_fields_count_up_impl<T>();
    else
        return count_fields_count_down_impl<T>();
}
```

[Final code for clang](https://godbolt.org/z/nPTjG91no)

## Conclusion 

It is fascinating how much can be achieved in C++ using relatively simple language tricks. Clever template metaprogramming tricks allow us to push the boundaries of what the compiler can do today.

## Self promotion

I made a Boost::PRF like library that compiles faster (especially in name reflection) here https://github.com/ZXShady/lahzam it uses more tricks to speed it up. It is not production ready however

## Footnotes

[^1]: C++26 lifts this restriction by introducing variadic structured bindings https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p1061r10.html which allow for `auto&& [...members] = aggregate` to work which makes this article alot simpler since we do not need to have `count_field`, but also C++26 introduces reflection so this is needed even less

[^2]: `std::tuple` implementations use self recursion to implement it which results in really atrocious compile times.

    MSVC https://github.com/microsoft/STL/blob/c430582bc1147f4c01973f6a051415110c4eb286/stl/inc/tuple#L307

    GCC https://github.com/gcc-mirror/gcc/blob/649b2a6d9f9f1a29f7f26199b7b3ebbaeef3c634/libstdc%2B%2B-v3/include/std/tuple#L236-L244

    Except Clang LibC++ amazingly, where it correctly uses a linear approach instead of a recursive one.

    https://github.com/llvm/llvm-project/blob/cefd20c496edd4df4ccd1faf399ee0d971b60116/libcxx/include/tuple#L490-L492
