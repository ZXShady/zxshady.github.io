A while ago, I found myself wanting a C++ way to reflect over enums — converting them to strings and back, iterating over their values, and doing all of it at compile-time, then I found magic_enum which did it well.

magic_enum is amazing, however it has one main issue, it is slow to compile. I wanted something lighter and faster to compile. So I created enchantum, a header-only C++20 library for enum reflection. Here's how it works under the hood — no macros, no magic — just some carefully wielded compiler quirks and constexpr wizardry.

# Objective

```cpp
enum class Fruit { Apple = 1, Banana, Pear = 4 };

constexpr std::string_view name = enchantum::to_string(Fruit::Banana);
std::cout << name << " is yummy"; // Banana is yummy 
```


The Obvious (Bad) Solution:
```cpp
#define to_string(x) #x + sizeof("Fruit::")-1
```

Obviously... no.

# Compiler Signature Hacking

Just like magic_enum, enchantum relies on the same trick: compiler-specific builtins like `__PRETTY_FUNCTION__` and `__FUNCSIG__`. These include the full function signature which includes the template parameters at compile time, which we can abuse for our advantage.

```cpp
template <auto V>
constexpr std::string_view signature() {
#if defined(__clang__) || defined(__GNUC__)
    return __PRETTY_FUNCTION__;
#elif defined(_MSC_VER)
    return __FUNCSIG__;
#endif
}
```
The output however is compiler dependant I will be using clang as it is the simplest.

```cpp
std::cout << signature<0>();
std::cout << signature<Fruit::Banana>();
```

```
std::string_view signature() [V = 0]
std::string_view signature() [V = Fruit::Banana]
```

Hooray! we got our strings, what happens if we input an invalid enum member in `signature` template parameter?

```cpp
std::cout << signature<static_cast<Fruit>(5)>();
```

Outputs
```
std::string_view signature() [V = (Fruit)5]
```


The output shows a C-style cast (ew) — "Fruit::Banana" gives us a clean name, but `static_cast<Fruit>(5)` gives "(Fruit)5".

While quirky, this distinction is useful: if the compiler shows the value with a cast, we can treat it as not a valid enum member, this also tells us that LLVM codebase is full of C elitists that prefer C casts to C++ ones.

So how do we get the strings? well, we need to iterate over every single value possible in the enum which for `Fruit` with an underlying type of `int` that is wait let me get my calculator... it is 4,294,967,296, `2^32` compiling a template `2^32` times is sure a way to blow up your compiler, so sadly we have to restrict ourselves to a much finer range like `-5` to `5`

Using `std::index_sequence` we can implement it quite easily with C++20 explicit template lamdbas

```cpp
constexpr int Min = -5;
constexpr int Max = 5;

template<typename E>
constexpr auto string_table = []<std::size_t...Idx>(std::index_sequence<Idx...>)
{
    return std::array{signature<static_cast<E>(Idx+Min)...>()};
}(std::make_index_sequence<Max-Min+1>{});
```

This generates a compile-time array of the signatures for each possible enum value in a range. Most will be invalid, but the ones matching real enum members will have pretty names.


the table is like this if we print every element

```cpp
std::string_view signature() [V = (Fruit)-5]
std::string_view signature() [V = (Fruit)-4]
std::string_view signature() [V = (Fruit)-3]
std::string_view signature() [V = (Fruit)-2]
std::string_view signature() [V = (Fruit)-1]
std::string_view signature() [V = (Fruit)0]
std::string_view signature() [V = Fruit::Apple]
std::string_view signature() [V = Fruit::Banana]
std::string_view signature() [V = (Fruit)3]
std::string_view signature() [V = Fruit::Pear]
std::string_view signature() [V = (Fruit)5]
```

We just need to extract our strings it is pretty simple if the part after "V =" is '(' then it is not a valid enum otherwise it is valid we can modify our function `signature` to skip unnecessary parts

# Cleaning Up the Signature

Let’s update the signature function to clean the output:

```cpp
template <auto V>
constexpr std::string_view signature() {
    auto sig = std::string_view(__PRETTY_FUNCTION__ + std::string_view("std::string_view signature() [V = ").size());
    sig.remove_suffix(1); // remove ']'
    return sig;
}
```

now it prints

```
(Fruit)-5
(Fruit)-4
(Fruit)-3
(Fruit)-2
(Fruit)-1
(Fruit)0
Fruit::Apple
Fruit::Banana
(Fruit)3
Fruit::Pear
(Fruit)5
```

We now can check whether the first character is a '(' to construct a pair of enum and string array.
```cpp

template<typename E>
constexpr auto entries = [](){
    constexpr auto& table = string_table<E>;
    std::array<std::pair<E,std::string_view>,table.size()> entries_return{};
    for(std::size_t i=0;i< table.size();++i){
        if(table[i].front() != '(')
        {
            // Fruit::Apple]
            std::size_t pos = table[i].find("::") + 2; // skip the colons
            auto& [enum_value,string] = entries_return[i]; // structured bindings for readability
            enum_value = static_cast<E>(i + Min);
            string = table[i].substr(pos);
        }
    }
    return entries_return;
}();
```


Printing the it using
```cpp
for(auto [e,s] :entries<Fruit>)
{
    if(!s.empty()) {
        std::cout << int(e) << " maps to " << s << '\n';
    }
}
```

We get
```cpp
1 maps to Apple
2 maps to Banana
4 maps to Pear
```


we can implement `to_string` really simply now

```cpp
template<typename E>
constexpr std::string_view to_string(E value) {
    for(auto [e,s] : entries<E>)
        if(!s.empty() && e == value)
            return s;
    return "";
}
```

It works! 

But... we’re storing a lot of empty string_views for invalid entries. That bloats our binary and requires extra checks for empty strings.
Let’s trim those out by computing a minimal size we need for the `entries` array:
```cpp
template<typename E>
constexpr auto entries = [](){
    
    // first get the minimal size we need to store all the strings
    constexpr auto minimal_size = [](){
        std::size_t count=0;
        for(const std::string_view s: string_table<E>)
            if(s.front() != '(')
                ++count;
        return count;
    }();

    // second use it to be the array length
    std::array<std::pair<E,std::string_view>,minimal_size> entries_return{};
    std::size_t entry_index =0;
        constexpr auto& table = string_table<E>;
    for(std::size_t i=0;i< table.size();++i){
        if(table[i].front() != '(')
        {
            // Fruit::Apple]
            std::size_t pos = table[i].find("::") + 2; // skip the colons
            auto& [enum_value,string] = entries_return[entry_index]; // structured bindings for readability
            enum_value = static_cast<E>(i + Min);
            string = table[i].substr(pos);
            ++entry_index;
        }
    }
    return entries_return;
}();
```



Now, all entries are valid, and we can simplify to_string:

```cpp
template<typename E>
constexpr std::string_view to_string(E value) {
    for (auto [e, s] : entries<E>)
        if (e == value)
            return s;
    return "";
}
```
But we run into an issue what if our enum is not inside the range -5 to 5 we specified?

```cpp
enum class Animals {Giraffe=5,Lion,Elephant};

int main()
{
    for(auto [e,s] : entries<Animals>)
    {
        std::cout << s << '\n';
    }
}
```

We surprisngly got only.

```
Giraffe
```

this is because our `Min` and `Max` values don't cover the range of this enum, this is bad the fix is to make them larger

```cpp
constexpr int Min = -100;
constexpr int Max = 100;
```

We now correcly get printed

```
Giraffe
Lion
Elephant
```


Congrats! with some tricks we got basic enum reflection but wait there is more to cover.

You might be tempted to increase the `Min` and `Max` ranges so you don't have to worry about missing entries, but this comes with a cost,compile times will explode.

How can we fix this? Well lets see the core issue we have, the expensive part is in `string_table` it requires `Max-Min+1` individual template instantations so with -100,100 range we get 201 instantations per enum type per translation unit this can make builds really slow so you have to balance between correctness (get every enum reflected) and compile times.

Can we reduce the required instantations? Yes we can why don't we try to print multiple template parameters?

```cpp

template <auto... Vs> // now variadic
constexpr std::string_view signature() {
    return __PRETTY_FUNCTION__;
}
```

```cpp
std::cout << signature<0,1,2,3,4,5,6,7>();
```

gives us 
```
std::string_view signature() [Vs = <0, 1, 2, 3, 4, 5, 6, 7>]
```



The main advantage is that we *only* instantaited a single template this is great for compile time speed

Lets prettify it for easier handling.

```cpp
template <auto... Vs> // now variadic
constexpr std::string_view signature() {
    auto sig = std::string_view(__PRETTY_FUNCTION__ + std::string_view("std::string_view signature() [Vs = <").size());
    sig.remove_suffix(2); // remove '>]'
    return sig;
}
```








`<(Fruit)-5, (Fruit)-4, (Fruit)-3, (Fruit)-2, (Fruit)-1, (Fruit)0, Fruit::Apple, Fruit::Banana, (Fruit)3, Fruit::Pear, (Fruit)5>]`
