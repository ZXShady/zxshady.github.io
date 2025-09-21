# Reimagining std::variant with Reflection in C++26

As you may have heard, [reflection](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r8.html#extractt) is coming to C++26.

> “Finally, I’ll have actual enum-to-string conversion!”
> — 90% of C++ programmers

> "Wow the comittee members spent 10 YEARS to implement another spelling for is_const this is stoopid blooooat”
> — 8% of C++ programmers

I was one in both groups, just waiting for the day I could easily convert enums to strings and cursing what is yet the `8`th spelling of `is_const` today. But reflection in C++26 opens the door to so much more than just that.

# A Better `std::variant`

One of the library features introduced in C++17  was `std::variant`: a type-safe union. It lets us hold one of several types in a single variable safely.

But its biggest downsides? The members of the variant are unnamed. Accessing them means using `std::get<T>` or `std::visit`, which can feel clunky and less intuitive

Why not both? Why do we have to sacrifice one (readability) for the other (type safety)?

Imagine combining the type safety of `std::variant` and the clarity of C unions named members:

The desired syntax will be in C++29 with [P07070](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p0707r5.pdf)

```cpp
class(variant) Events 
{
    MouseMoved mouse_moved;
    KeyPressed key_pressed;
};


Events events = getEvent();

if (auto* e = events.mouse_moved())
{
    // Handle mouse moved event
}
else if (auto* e = events.key_pressed())
{
    // Handle key pressed event
}
```


We don't have that currently in C++26 we can do this however as a workaround.

```cpp
namespace details {
struct Events
{
    MouseMoved mouse_moved;
    KeyPressed key_pressed;
};
}
using Events = magic_variant<details::Events>;
```

Let’s build a type-safe variant with named member access using C++26’s reflection features.

# Build the Variant Skeleton

A `std::variant` at it's core is just an index and a union of the types given to it.

```cpp
template<typename Blueprint>
class magic_variant {
    int discriminator;
    union {
        // ?? how do we store members?
    };
};
```
The challenge: we want to generate this union dynamically based on a `Blueprint` struct passed to `magic_variant`.

With C++26, we get powerful tools like:

* `^^Thing` and `[: Info :]` I like to think of them as & and * for pointers.

    * `&Thing` gives you a pointer to `Thing`.

    * `*Ptr` gives you back the pointed-to value.

    * `*&Thing` cancels each other.

    * `^^Thing` reflects the `Thing` giving you back a `std::meta::info` (a compile-time type-erased description of the reflected entity),

    * `[: Info :]` gives you back the value of the reflected entity.

    * `[: ^^Thing :]` cancels each other.

`std::meta::nonstatic_data_members_of()`

`std::meta::data_member_spec()`

`std::meta::define_aggregate()`: lets us define an incomplete type’s structure at compile time.

`consteval` blocks: execute code at compile time, useful with `std::meta` functions.

These allow us to generate a union from the members of a struct at compile-time.

```cpp
// this header gives access to the compile-time reflection API in the namespace std::meta.
#include <meta>

// shorter alias
namespace meta = std::meta;

// helper function for unchecked access to get all members
consteval std::vector<meta::info> get_members_of(meta::info type)
{
    // nonstatic_data_members_of(type, context)
    // unchecked access context to get all members including private ones
    return meta::nonstatic_data_members_of(type,meta::access_context::unchecked());
}


template<typename Blueprint>
class magic_variant {
    union impl; 
    consteval 
    {
        std::vector<meta::info> spec;
        for(meta::info member : get_members_of(^^Blueprint))
            spec.push_back(meta::data_member_spec(meta::type_of(member)));

        meta::define_aggregate(^^impl,spec);
    };
    impl mImpl; // now complete after executing the consteval block.
    int mDiscriminator;

    template<typename T>
    magic_variant(T&& t)
    {
        mDiscriminator = ??;
        mImpl = ??;
    }
};
```

Now we need a way to construct the appropaite member of `impl` and set the discriminator to the index.

To do that, we need a way to map a type to its index in the blueprint struct.

Before reflection this would be annoying you need to use typelists and meta functions with alot of `template` and `typename` keywords sprinkled around, alot of things but with reflection it is just normal C++ code.

```cpp
// Get the index of a type in a list of members
// Note the type has to match exactly (no stripping of cv ref qualifiers)
consteval std::ptrdiff_t get_member_index(meta::info type,std::span<const meta::info> members)
{
    auto it = std::ranges::find_if(members,[type](meta::info i)
    {
        return meta::type_of(i) == type;
    });

    auto index = it - members.begin();
    if(index == members.size())
        throw std::runtime_error("Did not find a member");
    return index;
}
```

Just normal C++ algorithms.

We can this use now to implement the constructor.

```cpp
template<typename Blueprint>
class magic_variant {
    ...
    
    // helper for concise usage
    static consteval auto members() { return get_members_of(^^Blueprint);}
    static consteval auto impl_members() { return get_members_of(^^impl);}

public:
    template<typename T>
    magic_variant(T&& t)
    {
        // splicing requires constant expression
        constexpr auto Index = get_member_index(meta::remove_cvref(^^T),members());

        mDiscriminator = int(Index);

        std::construct_at(&mImpl.[: impl_members()[Index] :],std::forward<T>(t));
    }

    std::size_t index() const 
    {
        return mDiscriminator;
    }

    template<std::size_t I,typename B>
    friend auto& get(magic_variant<B>& v);
    template<std::size_t I,typename B>
    friend const auto& get(magic_variant<B> const& v);
};

template<std::size_t I,typename B>
auto& get(magic_variant<B>& v)
{
    return v.mImpl.[: v.impl_members()[I] :];
}

template<std::size_t I,typename B>
const auto& get(magic_variant<B> const& v)
{
    return v.mImpl.[: v.impl_members()[I] :];
}

```

Now implementing the destructor it is simple by using a dispatch table to the destructors of each member.

```cpp
void destroy() { 
    constexpr static auto dtors =
    []<std::size_t... Is>(std::index_sequence<Is...>){
        return std::array<void(*)(magic_variant&),impl_members().size()>{
            [](magic_variant& self){ std::destroy_at(&self.mImpl.[: impl_members()[Is] :]);}...
        };
    }(std::make_index_sequence<impl_members().size()>{});
    // table dispatch
    dtors[index()](*this);
}
~magic_variant()
{
    destroy();
}
```

It works let try it

```cpp
struct Blueprint 
{
    int x;
    float y;
};

int main() 
{
    magic_variant<Blueprint> v = 42;

    std::cout << v.index(); // prints 0
    std::cout << get<0>(v); // prints 42
    v = 1.0f;

    std::cout << v.index();// prints 1
}
```


# Generating the getters.

we want to have this syntax

```cpp
magic_variant<Blueprint> v = 1;
v.i(); // getter to a pointer that may be null
v.i(1); // sets the members

if(auto* i = v.i())
{
    int val = *i;
    *i = val * 2; 
}
```

But here is the roadblock, `define_aggregate` does not allow for member functions to be defined. this would require C++**29** [token injection](https://github.com/cplusplus/papers/issues/1946).

So is this impossible? 
   well yesn't.

See, the syntax `a.f()` does not require `f()` to be a member function, it may as well be just a member that is a function pointer.

```cpp
template<typename Self>
struct Getter {
    Self* self;
    int operator()()
    {
        return self->x; 
    }
};

struct S {
    Getter<S> get_x{this};
    int x;
};

S s;
s.get_x();
```

So we can instead of generating *member functions*, we generate *members* that are *functions*.

```cpp
// Forward declaration to be able to friend
template<std::size_t I,typename Self>
struct magic_variant_function;


// Generates a class to be inherited from that contains a list of magic_variant_function's as members 
template<typename Self,typename Blueprint>
consteval meta::info get_magic_variant_functions() {
    struct impl;
    consteval {
        auto members = get_members_of(^^Blueprint);
        std::vector<meta::info> spec;
        for(std::size_t i =0;i<members.size();++i)
        {
            meta::info func = meta::substitute(
                ^^magic_variant_function,
                {meta::reflect_constant(i),^^Self}
            );
            spec.push_back(meta::data_member_spec(
                func,
                {
                    .name = meta::identifier_of(members[i])
                    .no_unique_address = true // to avoid unnecessary padding
                }
            ));
        }
        meta::define_aggregate(^^impl,spec);
    };
    return ^^impl;
}

template<typename Blueprint>
class magic_variant :
 public [: get_magic_variant_functions< magic_variant<Blueprint>,Blueprint>() :]
{
    template<std::size_t,typename>
    friend class magic_variant_function;
    ...
};

template<std::size_t I,typename Self>
struct magic_variant_function
{
    auto* operator()()
    {
        // a **hack** since the address of the this pointer is same as the magic_variant derived class.
        auto& self= *reinterpret_cast<const Self*>(this);
        return self.index() == I ? &get<I>(self) : nullptr;
    }
    const auto* operator()() const
    {
        // a **hack** since the address of the this pointer is same as the magic_variant derived class.
        auto& self= *reinterpret_cast<const Self*>(this);
        return self.index() == I ? &get<I>(self) : nullptr;
    }


    template<typename... Args>
    auto& operator()(Args&&... args)
    {
        // a **hack** since the address of the this pointer is same as the magic_variant derived class.
        auto& self= *reinterpret_cast<Self*>(this);
        self.destroy();
        self.mDiscriminator = I;
        return *std::construct_at(&get<I>(self),std::forward<Args>(args)...);
    }
}
```


Usage

```cpp
struct MouseMoved { int x, y; };
struct KeyPressed { char character; };

namespace details {
    struct Events {
        KeyPressed key_pressed;
        MouseMoved mouse_moved;
    };
}

using Events = magic_variant<details::Events>;

int main() {
    Events e = MouseMoved{500, 200};
    if (auto* m = e.mouse_moved()) {
        std::cout << m->x << ", " << m->y << "\n";
    }
    e.key_pressed('B');
    if (auto* k = e.key_pressed()) {
        std::cout << "Key pressed: " << k->character << "\n";
    }
}
```

# The end

Thanks all of the people who contributed to bringing reflection into C++26! It is such an awesome feature.
Reflection is not just about enum to strings or spelling `is_const` in a fancier way. it is much cooler than that.

[Full code here](https://godbolt.org/z/43GffWxbb)
