
What is a today's article without a dumb awful ai generated image?

![Very Angry man for waiting too long to compile his files due to slow enum reflection](./dumbpics/angry_man_for_slow_cpp_compiling_due_to_enum_reflection.jpg)


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

Just like magic_enum, enchantum relies on the same trick: compiler-specific builtins like `__PRETTY_FUNCTION__` for gcc and clang and `__FUNCSIG__` for msvc. These include the full function signature which includes the template parameters at compile time, which we can abuse for our advantage.

```cpp
template <auto V>
constexpr auto signature() {
#if defined(__clang__) || defined(__GNUC__)
    return __PRETTY_FUNCTION__;
#elif defined(_MSC_VER)
    return __FUNCSIG__;
#endif
}
```
The output however is compiler dependant.

```cpp
std::cout << signature<0>();
std::cout << signature<Fruit::Banana>();
```

| Compiler | `signature<0>`               | `signature<Fruit::Banana>`              |
|----------|------------------------------|-----------------------------------------|
|  Clang   | "auto signature() [V = 0]" | auto signature() [V = Fruits::Banana]   |
|  GCC     | "constexpr auto signature() [with auto V = 0]" | "constexpr auto signature() [with auto V = Fruits::Banana]"  |
|  MSVC    | "auto __cdecl signature\<0x0\>(void)" | "auto __cdecl signature\<Fruits::Banana\>(void)"   |



I will be using clang as it is the simplest.

As you can see from the compiler output every time an enum is placed in the template parameter you get a the enumerator name.

What happens if we input an invalid enum member in `signature` template parameter?

```cpp
std::cout << signature<static_cast<Fruit>(5)>();
```

Outputs
```
auto signature() [V = (Fruit)5]
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
constexpr auto signature() {
    auto sig = std::string_view(__PRETTY_FUNCTION__ + std::string_view("auto signature() [V = ").size());
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
            enum_value = static_cast<E>(i + Min); // adjust the range from [0,size()] to [Min,size()+Min]
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
constexpr auto signature() {
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
constexpr auto signature() {
    auto sig = std::string_view(__PRETTY_FUNCTION__ + std::string_view("auto signature() [Vs = <").size());
    sig.remove_suffix(2); // remove '>]'
    return sig;
}
```

Lets change our `string_table` to instead be a single `std::string_view`

```cpp
template<typename E>
constexpr auto string_table = []<std::size_t...Idx>(std::index_sequence<Idx...>)
{
    return signature<static_cast<E>(Idx+Min)...>();
}(std::make_index_sequence<Max-Min+1>{});
```


Now lets try to parse this it is more complicated due to the fact we don't know the end of the string it is more convulated.

Printing the single string out gives us.

`(Fruit)-5, (Fruit)-4, (Fruit)-3, (Fruit)-2, (Fruit)-1, (Fruit)0, Fruit::Apple, Fruit::Banana, (Fruit)3, Fruit::Pear, (Fruit)5`

Let's try to figure out a way to parse it.
By observing it: the values are seperated by commas followed by a space we can use that as the end of the enum name.


```cpp
// Constexpr std::strlen used to remove magic numbers
template<std::size_t N>
constexpr std::size_t c_strlen(const char(&)[N]) {
    return N-1;
}

template<typename E>
constexpr auto entries = [](){
    
    // first get the minimal size we need to store all the entries
    constexpr auto minimal_size = [](){
        std::size_t count = 0;
         std::string_view s = string_table<E>;
         while (!s.empty()) {
             const auto pos = s.find(',');
             if(s.front() != '(') // if it is not a cast
                 ++count;
             if(pos == s.npos) // if we did not find a comma it is the last element
                 break;
             s.remove_prefix(pos + c_strlen(", ")); // move past ','
         }
         return count;
    }();

    // second use it to be the array length
    std::array<std::pair<E,std::string_view>,minimal_size> entries_return{};
    std::size_t entry_index =0;
    std::string_view s = string_table<E>;
    for (int curr_enum_val = Min;!s.empty();++curr_enum_val) {
        const auto commapos = s.find(',');
        // if it is not a cast
        if(s.front() != '(')
        {
           auto& [enum_value,string] = entries_return[entry_index];
           enum_value = static_cast<E>(curr_enum_val);
           std::string_view name = s.substr(s.find("::") + c_strlen("::"));
           string = name.substr(0,name.find(',')); //if there is no comma found then we take the rest of the string.
           ++entry_index;
        }
        if(commapos == s.npos) // if we did not find a comma it is the last element
            break;
        s.remove_prefix(commapos + c_strlen(", ")); // move past ','
    }
    return entries_return;
}();
```

trying it out on [godbolt](https://godbolt.org/z/o6GeK51af) correctly works but however there is an issue if we look at the binary.

```cpp
.L.str.1:
  .asciz "std::string_view signature() [Vs = <(Fruits)-5, (Fruits)-4, (Fruits)-3, (Fruits)-2, (Fruits)-1, Fruits::Banana, Fruits::Pear, (Fruits)2, (Fruits)3, Fruits::Apple, (Fruits)5>]"
```

although we only need to store `"Banana"`,`"Pear"`,`"Apple"` we get the entire string stored this is bad for binary sizes and is bloating our binary, how can we fix this?

# Trimming strings

We can store the required strings in a global static storage constexpr variable and thanks to CNNTP (class non type template parameters)
it makes it really easy to do so.


```cpp
template<auto Value>
constexpr auto static_storage_for = Value;

template<typename E>
constexpr auto entries_optimized = [](){
    // first get the required length for storing all strings
    constexpr auto total_string_length = [](){
        std::size_t count = 0;
        for(auto [e,s] : entries<E>)
            count += s.size();
        return count;
    }();
    // second group the strings into a single array
    constexpr auto grouped_strings = [total_string_length](){
        std::array<char,total_string_length> strings{};
        for(std::size_t i =0;auto [e,s] : entries<E>) {
            s.copy(strings.data()+i,s.size());
            i += s.size();
        }
        return strings;
    }();
    // third make grouped_strings have static storage by making a constexpr global variable
    // copy its value since we don't have static constexpr local variables in constexpr functions until C++26
    constexpr auto& storage = static_storage_for<grouped_strings>;

    // fourth make the new entries point to that storage we created
    auto fixed_entries = entries<E>;
    for (std::size_t offset = 0;auto& [e, s] : fixed_entries) {
        s = {storage.data() + offset, s.size()};
        offset += s.size();
    }
    return fixed_entries;
}();
```

Now looking at the binary we get this instead

```cpp
static_storage_for<std::array<char, 15ul>{"BananaPearApple"}>:
  .ascii "BananaPearApple"
```

much much better and more optimized. only 15 bytes needed instead of whatever previously was needed

Let's assume that for some reason your `i` key on your keyboard is broken that means sadly that you can't spell out `.size()` member function so you have to resort to `std::strlen` which does not contain `i` 

```cpp
for(auto [e,s] : entries<E>)
{
   std::cout << std::strlen(s.data());
}
```

surprisingly it does not work, because our strings are not null terminated the fix is simple.

# Null Termination

We modify total_string_length and other local functions to accomdate for the space the null terminator needs 

```cpp
template<typename E>
constexpr auto entries_optimized = [](){
    // first get the required length for storing all strings
    constexpr auto total_string_length = [](){
        std::size_t count = 0;
        for(auto [e,s] : entries<E>)
            count += s.size();
        // each string required a null terminator
        return count + entries<E>.size();
    }();
    // second group the strings into a single array
    constexpr auto grouped_strings = [total_string_length](){
        // The array is initialized to zero (the null terminator)
        // the gaps we skip is filled with them automaticly.
        std::array<char,total_string_length> strings{};
        for(std::size_t i =0;auto [e,s] : entries<E>) {
            s.copy(strings.data()+i,s.size());
            
            i += s.size() + c_strlen("\0"); // past null terminator
        }
        return strings;
    }();
    // third make grouped_strings have static storage by making a constexpr global variable
    // copy its value since we don't have static constexpr local variables in constexpr functions until C++26
    constexpr auto& storage = static_storage_for<grouped_strings>;

    // fourth make the new entries point to that storage we created
    auto fixed_entries = entries<E>;
    for (std::size_t offset = 0;auto& [e, s] : fixed_entries) {
        s = {storage.data() + offset, s.size()};
        offset += s.size() + 1;
    }
    return fixed_entries;
}();
```

Looking at the assembely we see

```cpp
static_storage_for<std::array<char, 18ul>{"Banana\0Pear\0""Apple"}>:
  .asciz "Banana\000Pear\000Apple"
```

much better and yes the "Apple" is null terminated but it does not show up here as you can see from the array size being 18 it means 3 null terminators were added.

# End

This is a simplified implementation of enchantum logic, this implementation does not currently handle

1. Namespaced enums or enums in classes
2. C style enums

Here is final code

[godbolt](https://godbolt.org/z/cxjdTdzaj)
