# I implemented UFCS inside clang 

I have implemented UFCS in Clang you can try it here in this [clang fork](https://github.com/ZXShady/llvm-project/tree/ufcs)

Uniform Function Call Syntax (UFCS) is one of the most discussed features in C++.

[](#)

# What is it?

In C++ we generally have two ways to perform an operation on an object

1. Member Functions: `object.method(args...)`
2. Non-member Free Functions: `function(object, args...)`

UFCS is a language feature that would allow these one syntax to be call  the other. If you wrote `x.f()`, the compiler would first look for a member function named `f`. If it didn't find one, it would look for a free function `f(x)`.

Example:

```cpp

class Foo {};
void do_thing(Foo f,int i);

Foo f;
do_thing(f,42);

// with UFCS
f.do_thing(42); 
```

# Why is it needed?

## Generic Programming

This is one of the biggest drivers. If you are writing a template, you often don't know if the user's type uses a member function or a free function for a specific task (like `begin()` and `end()`).

today without UFCS. You have to use wrappers to cover both cases and two step dances.

```cpp
namespace fallback {
    template<typename T,int N>
    T* begin(T(&a)[N]) { return &a[0];}
    template<typename T>
    auto begin(T& t)
    requires { t.begin();}
    {
        return t.begin();
    }
}

template<typename Container>
void algo(const Container& c)
{
    // not generic enough, misses C arrays.
    auto begin1 = c.begin(); 
    
    // not generic enough, C arrays for builtin types have no associated namespace.
    auto begin2 = begin(c); 

    // generic and inline but cumbersome to write and for the compiler to optimize
    auto begin3 = [&](){ 
        if constexpr(requires {c.begin()})
            return c.begin();
        else
            return begin(c);
    }(); 

    // generic but cumbersome needs to be done everywhere
    using fallback::begin;
    auto begin4 = begin(c);
}

```

With UFCS you just use member syntax and it works for both.

```cpp
template<typename Container>
void algo(const Container& c)
{
    // done!
    auto begin = c.begin(); 
    // if c.begin() is not valid then try begin(c)
}
```

## Discoverability and autocomplete

In IDEs, when typing `object.`. triggers a list of available methods. Free functions that operate on that object (like those in `<ranges>`) don't show up in that list. UFCS would allow "C# extension methods like" making library functions much easier to find and use. especially if those functions are chainable

## Minimal and lean interfaces

Herb Sutter suggests in his awesome Guru of the week series [#84](http://www.gotw.ca/gotw/084.htm) and that if a function can be implemented using only the public interface of a class, it should be a non-member function. 
- Increases encapsulation
- Allows generic algorithms to work on any object uniformly
- It keeps class definitions small and readable instead of being 4000 lines long.
- Less dependencies in the header file class leading to faster compilation times.

However, users prefer member syntax because it feels "connected" to the object and symmetric in usage. UFCS allows the internal purity of non-member functions with the ergonomic feel of member functions.



Let's imagine a simple file api
```cpp
// myfile.hpp
namespace filesys {
class File {
public:
    File(const char* path);
    size_t length() const;
    void read(void* buf,size_t len);
};
}
```

This header is compact and focused. It exposes only the fundamental operations, it has 0 dependencies.
But that public API is already enough to build higher-level utilities

```cpp
// myfile_utils.hpp
#include <optional> // dependency
namespace filesys {
template<typename T>
T read_object(File& file)
{
    T t;
    file.read(&t,sizeof(T));
    return t;
}

template<typename T>
std::optional<T> try_read(File& file)
{
    T t;
    // not enough bytes
    if(file.length() > sizeof(T))
        return {};
    file.read(&t,sizeof(T));
    return t;
}
}
```

With just the public API we created 2 much more convienent and powerful abstraction just but now the issue is user side.


```cpp
#include <myfile.hpp>
#include <myfile_utils.hpp> // for read_object,try_read

char buf[20];
// okay member syntax
file.read(buf,20); 

// free function syntax, have to specify namespace or rely on C++20 template ADL
Object obj = read_object<Object>(file);
std::optional<Object> obj = filesys::try_read<Object>(file); 

```


Now we broke the users flow, `read` gets the dot. `read_object` doesn't. The user deals with two different calling syntaxes for functions that conceptually belong to the same API, and must remember to qualify the namespace everytime to avoid Argument Dependant Lookup surprises. Why this distinction?

The temptation _and what many codebases actually and already do_ is to shove everything into the class.

```cpp
// myfile.hpp
#include <optional> // dependency
namespace filesys {
class File {
public:
    File(const char* path);
    size_t length() const;
    void read(void* buf,size_t len);
    template<typename T>
    T read_object();
    template<typename T>
    std::optional<T> try_read();
};
}
```

```cpp
#include <myfile.hpp>

char buf[20];
// uniform usage
file.read(buf,20); 
Object obj = file.read_object<Object>();
std::optional<Object> obj = file.try_read<Object>(); 
```

But this comes at a cost.

Including `<optional>` in `myfile.hpp` means every translation unit that includes this basic file header now pays for that dependency, even if it never uses `try_read` ever.

This is exactly the kind of header bloat that gives C++ its reputation for glacially slow compile times and it pushes classes toward monolithic designs.

UFCS would let us keep the clean separation:

- core operations in the class
- higher-level utilities as free functions

while still allowing users to write the operations with symmetric member function syntax.

## Arbitrary syntax tax.

In C++, the choice between member and non-member often affects syntax more than semantics.

A common defense of member functions is:

> I need access to privates.

But even that is not decisive, because a non-member can be made a friend.

```cpp
class Data {
    size_t size(){
        return m_size;
    }
    size_t m_size;
};
```

But you could just as well make it a free `friend` function, 

```cpp
class Data {
    friend size_t size(Data& self);
    size_t m_size;
};
size_t size(Data& self) 
{
    return self.m_size;
}

```

The real difference is not capability. It is syntax (and lookup rules).

C++ currently forces a choice between syntax and encapsulation. 
- If you want the dot, you must break encapsulation by putting the function inside the class. 
- If you want encapsulation, you must suffer the 'free function' API and non uniform usage.

UFCS decouples these two completely independent concerns giving library authors freedom to make the right engineering choice without imposing a syntax penalty on users.

## Enums/builtins with member functions

today `enum`s can't have member function.

```cpp
namespace myns {
enum class Colors {
    Green,Red,Blue
};
const char* to_string(Colors c);
}

Colors c = Colors::Red;

// unqualified opens the door to ADL surprises 
to_string(c);
// verbose
myns::to_string(c); 

// with ufcs
c.to_string(); 
// enums with members! 
// no namespace qualification needed
// IDE discoverable
```

Many APIs are object-oriented in spirit even when they are not expressed as class members (like C `FILE` API). UFCS would let the syntax reflect that.

```cpp
if (auto* f = "file.txt".std::fopen("wb"))
{
    f.fprintf("Python in my {}!".std::format("C++").c_str());
    f.fclose();
}
```

## Deduplication

Currently a significant amount class of member functions of many types are convenience functions that do not require access to private class state. UFCS provides a clear syntax for implementing such functions as non-members, reducing class API bloat and dramatically increasing reusability across types.

The classic example is the similarity between the purely const interfaces of `std::string` and `std::string_view`. `std::string_view` had to reimplement this entire interface as members, leading to significant duplication for operations that just act on *data* but their member-only implementation locks them to specific standard types.


Member Function       | Shared Overloads
----------------------|-----------------
`length()`              | 1
`max_size()`            | 1
`empty()`               | 1
`cbegin() / cend()`     | 1
`crbegin() / crend()`   | 1
`copy()`                | 1
`substr()`              | 1
`starts_with()`         | 4
`ends_with()`           | 4
`compare()`             | 9
`find()`                | 4
`rfind()`               | 4
`find_first_of()`       | 4
`find_last_of()`        | 4
`find_first_not_of()`   | 4
`find_last_not_of()`    | 4
Total                 | 46 * 2 == 92 OVERLOADS!

Thats 92 overloads! For what is essentialy algorithms already in the C++ standard library like `std::find`,`std::cbegin` and others. This would get worse as the Standard Library ends up with more types with similiar interfaces (e.g. `std::zstring_view`) more duplication will occur and this is bad for compile times and it bloats interfaces and debug builds. It also restricts usability why is `str.length()` fine but not `vector.length()`? why is `str.find('a')` fine, but not `vector.find(42)`? both are containers.

Other example is `value_or` method for `optional` and `expected` this algorithm is completely generic but it currently has to be written 4 times for each of those classes. while UFCS would allow 

```cpp
template<typename Opt,typename Def>
auto value_or(Opt&& opt,Def&& def)
{
    return opt ? *opt : def;
}
```

This covers everything pointer-like there is no reason why `value_or` should be a member function other than the ability to chain it being a member loses the ability for it to apply to anything *optional-like* like pointers, `unique_ptr`s,`shared_ptr`s and such.

## Less workarounds

Currently `std::ranges` require an insane amount of machinery to function with the `|` operator, which puts a burden on library devs and on the compiler which tries to optimize this machinery while UFCS would essentially be a builtin pipe operator.

Example taken from [cppreference](https://en.cppreference.com/w/cpp/ranges.html)
```cpp
auto const ints = {0, 1, 2, 3, 4, 5};
auto even = [](int i) { return 0 == i % 2; };
auto square = [](int i) { return i * i; };

// functional syntax, hard to read inside out reading
std::views::transform(std::views::filter(ints, even), square)

// English left-to-right reading but causes slower compilation and debug builds
ints | std::views::filter(even) | std::views::transform(square)

// English left-to-right reading but as fast as the functional syntax
ints.std::views::filter(even).std::views::transform(square)

```
# Why isn't it here yet?

For all its appeal, UFCS is not a trivial feature at all to add to C++.

The main objections are from what I see online are.

1. Unpredictable lookup complexity: C++ name lookup is already difficult
2. Overload resolution ambiguity: what if both a member and a free function exist?
3. API ownership confusion and unstable APIs: `x.f()` suggests `f` belongs to the type, even if it actually comes from some unrelated namespace.
4. Readability concerns: today, `x.f()` and `f(x)` may communicate different design intent which I think is bad design even without UFCS.
5. Typos with pointers `a.f()` and `a->f()` can mean different things, this is a weak point since this already happens with anything that has an overloaded `->` operator like `unique_ptr`,`optional` UFCS would just make it more uniform and allow pointers to have the same power.

Most is taken from online comments and from this [UFCS is a breaking change](https://isocpp.org/files/papers/P3027R0.html) paper.

These concerns are real. C++ is not a small language, and any feature that touches lookup and overload resolution is dangerous territory. C++ is not a playground

But I think there's a path forward.

# An idea to solve it

After thinking about the objections and reading the paper, I believe most of them stem from one design choice: making the fallback lookup too broad with ADL. If `a.foo(args...)` rewrites to `foo(a, args...)` using normal unqualified lookup scheme, you get chaos any `foo` visible at the call site could hijack the dot syntax.

So what if we constrain the lookup?

The rule: When `a.foo(args...)` fails to find an accessible (non private) member function, the compiler searches for free functions named `foo` only in the namespaces associated with the type of `a` and its base classes not at the call site, not in enclosing scopes, not via normal unqualified lookup.

Example

```cpp
namespace X {
    struct C { void member(long);};
    void member(C,int);
    void foo(const C&);
}

namespace Y {
    struct Tag {};
    void goo(X::C,Tag);
}

namespace X {
    void zoo(X::C,Y::Tag);
}
namespace Y {
    void zoo(X::C,Y::Tag);
}

void foo(X::C&);

int main() {
    X::C c;

    c.member(0); // calls X::C::member even if X::member is a better match

    Y::Tag tag;
    c.foo(); // calls X::foo, although foo(c) would find ::foo
    goo(c,tag); // finds Y::goo

    // fails, the namespaces it will find `goo`  in
    // is the associated namespaces of `c` which is X only which has no goo
    c.goo(tag);

    // okay explicit search.
    c.Y::goo(tag); 

    zoo(c,tag); // ambigious
    c.zoo(tag); // finds X::zoo
}
```
The lookup is simple, search the class for the member, then its bases for the member, then the namespaces of the class and its bases. Nothing else. No surprises. You control your type's namespace you know the functions inside it

This can be made even simpler by restricting it to the namespace of `a` only no base classes even, but I think considering base classes is logical since it is part of your API. 

This isn't a formal proposal. It's a sketch an idea that if you scope the lookup narrowly enough, most of the scary objections to UFCS dissolve and it still remains useful. The class author retains control of the API surface. The lookup is fast and predictable. Old code doesn't break. And the programmer finally gets to stop caring whether `begin` is a member or a free function and no dances anymore nor parties which if you think about it, it is not fun.

I implemented it here in this [clang fork](https://github.com/ZXShady/llvm-project/tree/ufcs) let me know what you think in the comments if you hate this or like it.


# How to use the fork

This is better in a readme but I added some compiler feature flags there are 3 "-fufcs=disabled" is the default, "-fufcs=herb" is a modified version of Herb Sutters proposal it only modifies member lookup and it has restricted name lookup search range, "-fufcs=extensions" is the same as herb except with a restriction that the function must be marked

```cpp
namespace X {
    class C{};
    void f(C&); // findable by herb
    void g(this C&); // findable by extensions, opt in
}

#if __has_feature(ufcs_all) // if herb or extensions
#if __has_feature(ufcs_herb)
#if __has_feature(ufcs_extensions)
```


## Implemenation 

The lookup scheme from [Barry revzin post](https://brevzin.github.io/c++/2019/04/13/ufcs-history/) for Candidiate Set is CS:MemberFindsFree and for Overlord Resolution it is OR:TwoRoundsMemberFirst


### Why clang?

Because this is my second time actually messing with it so I am a bit familiar and I am on windows not a linux distro so I don't want to face the pain that is MingW, and the GCC codebase in my eyes is less clean than Clang.


### Changes made

I used VsCodium with Clangd to edit the clang codebase because Visual Studio kept crashing when I opened the solution.

1. The flags

I have 3 flags as explained `-fufcs=disabled|herb|extensions` so I need to make the compiler recognize them so I searched for some flags by doing `clang --help` and taking a `-f` flag and searching for it in the codebase like `-fcxx-exceptions`  and I found `Options/Options.td` file which contains all the options so I went ahead copied a flag and edited it to have my new UFCS flag and I had to add an new `enum` to `Basic/LangOptions.h` 

```cpp
enum class UFCSModeKind : uint8_t {
    Disabled,
    Extensions,
    Herb
};
```


```hs
def fufcs_EQ : Joined<["-"], "fufcs=">, Group<f_Group>,
  Visibility<[ClangOption, CC1Option]>,
  HelpText<"Control UFCS behavior: 'extensions' enables C#-style explicit 'this' parameters; "
           "'herb' enables calling any free function with member syntax">,
  Values<"disabled,extensions,herb">,
  NormalizedValuesScope<"LangOptions">, 
  NormalizedValues<["UFCSModeKind::Disabled", "UFCSModeKind::Extensions", "UFCSModeKind::Herb"]>,
  MarshallingInfoEnum<LangOpts<"UFCSMode">, "UFCSModeKind::Disabled">;
```

I had to as well edit other files you can look at the commit history if you want.

but this was the simple part now we need to go to the harder parts.

2. Parser

So I needed  for when I have `-fufcs=extensions` to allow `this` qualifier on normal functions

```cpp
// error: an explicit object parameter cannot appear in a non-member function
void foo(this C&);
```

This was simple enough to implement I had to go into the parser and just put an if check that silences this when using extensions mode. but first I needed to know where this error is emitted so I went ahead and copied the error message and searched and found that the error `DiagID` is `err_explicit_object_parameter_nonmember` in `Basic/DiagnosticSemaKinds.td` so I went again and searched for this and found the file `SemaType.cpp` so I went ahead and put a simple if check to not emit it. now that was simple.

3. Sema

Now to the fun part actually implementing UFCS.

This is where things go from nice to welcome to hell. I had an insane amount of crashes while doing this part that drove me insane along with the insane compilation times that drove me even crazier eveyr little change took like 40 seconds to do.


The strat is

- Try normal member lookup `a.foo(args...)`
- If that fails (or is inaccessible), try UFCS: which does 
    - Rewrite to `foo(a, args...)`
    - Perform constrained lookup for `foo`
    - Run overload resolution again


After digging through the codebase, the key function responsible for this member function call expressions turned out to be `Sema::BuildCallToMemberFunction` in `SemaOverload.cpp`

First, I needed to construct the equivalent of `foo(a, args...)`

So I created a list. Then inserted the object.
```cpp
SmallVector<Expr *, 8> ArgsWithMember;
Expr *Base = MemExpr->getBase();
ArgsWithMember.push_back(Base);
ArgsWithMember.append(Args.begin(), Args.end());
```

Since my overload resolution is 2 step, try members then free I needed two overload candidate sets

```cpp
OverloadCandidateSet CandidateSet;      // normal members
OverloadCandidateSet UFCSCandidateSet;  // free functions
```

I then added functions to the UFCSCandidateSet `AddOverloadCandidate(FuncDecl, ..., ArgsWithMember, UFCSCandidateSet, ...)` and such. Notice the use of `ArgsWithMember`, not `Args`.

Now the logic is mostly copied from the existing code and slightly edited.


```cpp
switch (CandidateSet.BestViableFunction(...)) {
    case OR_Success: // #1
        ...
    case OR_No_Viable_Function: // #2
        ...
    case OR_Ambiguous:
        ...
    case OR_Deleted: // #3
        ...
}
```

For all of these 3 cases marked we need to perform overload resolution again, 

For the success case, this case is entered when overload resolution finds an accessible or inaccessible member function so yes a `private` uncallable function goes here, so I needed to detect the case when it is `private` and try the UFCS candidates.


```cpp
if (IsPrivate(ChosenCandidate) && TryUFCSFallback())
    Success = true; 
```

implementing `IsPrivate` was simple 

```cpp
const auto IsPrivate = [&](const DeclAccessPair &FoundDecl) -> bool {
    const SFINAETrap Trap(*this, true);
    return CheckUnresolvedMemberAccess(UnresExpr, FoundDecl) != AR_accessible;
};
```

The `SFINAETrap` is an RAII class that from my understanding captures all errors silences them and if any error is emitted it does a subsitutation failure.


Now to implement to `TryUFCSFallback`

```cpp
const auto TryUFCSFallback = [&]() -> bool {
      if (getLangOpts().getUFCSMode() == LangOptions::UFCSModeKind::Disabled)
        return false;
      // we don't do UFCS lookup for operators they already do something like it
      if (!UnresExpr->getMemberName().isIdentifier())
        return false;

      // if the member is explicitly qualified like a.B::foo no UFCS lookup is needed
      if (UnresExpr->getQualifier())
        return false;
      
      if (ArgsWithMember.empty())
        return false;

      // fill the UFCSCandidateSet
      // this is the same function as Sema::AddArgumentDependentLookupCandidates but slightly different in how it looks up functions
      UFCSADLOnly(*this, UnresExpr->getMemberName(), UnresExpr->getMemberLoc(),{ArgsWithMember[0]}, ArgsWithMember, TemplateArgs, UFCSCandidateSet);

      if (getLangOpts().getUFCSMode() == LangOptions::UFCSModeKind::Extensions) {
        for (auto &C : UFCSCandidateSet) {
          // Only bother checking candidates that are currently considered
          // "good"
          if (!C.Viable)
            continue;

          bool hasExplicitThis =
              (C.Function->getNumParams() > 0 &&
               C.Function->getParamDecl(0)->isExplicitObjectParameter());

          if (!hasExplicitThis) {
            C.Viable = false;
            C.FailureKind = ovl_fail_bad_target;
          }
        }
      }
      OverloadCandidateSet::iterator UFCSBest;
      if (UFCSCandidateSet.BestViableFunction(*this, UnresExpr->getBeginLoc(),UFCSBest) == OR_Success) {
        ChosenCandidate = cast<FunctionDecl>(UFCSBest->Function);
        return true;
      }
      return false;
    };
```

This is not the entire code needed to make it work  since I suck at explaining so you might want to look at the commits and ask me on reddit via DMs.





## The sad part

Although my lookup scheme is very conservative and predictable than other papers. despite that it STILL breaks old code and prevents it from compiling which made me so disappointed and made me stop even considering proposing this new UFCS lookup scheme which made me very sad

the code that broke was `std::begin`
the example is this

```cpp
namespace std {
    template<typename T,int N>
    T* begin(T(&arr)[N]);
    
    template<typename T>
    auto begin(T& cont) -> decltype(cont.begin());

    struct SomeStdType {};
}

int main() 
{
    std::SomeStdType arr[10];
    std::begin(arr);
/*
ufcs.cpp:6:42: fatal error: recursive template instantiation exceeded maximum depth of 3
    6 |     auto begin(T& cont) -> decltype(cont.begin());
      |                                          ^
ufcs.cpp:6:42: note: while substituting deduced template arguments into function template 'begin'
      [with T = std::SomeStdType[10]]
ufcs.cpp:6:42: note: while substituting deduced template arguments into function template 'begin'
      [with T = std::SomeStdType[10]]
ufcs.cpp:6:42: note: while substituting deduced template arguments into function template 'begin'
      [with T = std::SomeStdType[10]]
ufcs.cpp:14:5: note: while substituting deduced template arguments into function template 'begin'
      [with T = std::SomeStdType[10]]
   14 |     std::begin(arr); // infinite loop
      |     ^
ufcs.cpp:6:42: note: use -ftemplate-depth=N to increase recursive template instantiation depth
    6 |     auto begin(T& cont) -> decltype(cont.begin());
      |                                          ^
1 error generated.
*/
}
```

It seems unintuitive at first since the 1st one is just straight up best overload. but C++ still needs to instanstiate both overloads to see which could be better so it instanstiate it and recurses infinitly.

`std::begin` was written with mind that `cont.begin()` will **always** call a member function and never a free function so it does not just recurse forever. this could actually be fixed by doing `cont.T::begin()` which forces a member lookup only, but that is not the point. It broke code you are now forced to change what used to be perfectly fine code to adhere to this new change which is unacceptable. C++ is a massive language with many lines of code and forcing a massive breaking change isn't practical and will stop people from updating.

## The end

I thought that I finally cracked the code that I found a lookup scheme that does not destroy the universe I only realized that I failed when I finished my implementation, if you have any ideas to fix the above issue so it can actually be proposed please let me know in the comments but I don't have any solutions.

I think now that UFCS is bassicly impossible to get into C++ now it could have got into it from the start but it is too late now. I don't think extension methods can work either they would have the same issues.

The commitee has rejected alternatives like the [pizza operator](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2011r0.html) `|>` by Barry Revzin and Colby Pike, which thinking about it now seems to be the only practical alternative; the others breaks code.

Let this article be a reminder for anyone proposing an idea that an implementation for ideas is **always** needed nothing beats actually using your fork to compile code than just writing a paper you discover way more edge cases.


Special Thanks to [Vittorio Romeo](https://github.com/vittorioromeo), [Eczbek](https://github.com/Eczbek),[Terens](https://github.com/TerensTare) , and [QuStar](https://www.reddit.com/user/qustar_/) for keeping me motiviated making this fork.

Thanks for reading my article.

## Resources

Herb Sutter's UFCS Proposal [P3021R0](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p3021r0.pdf).

UFCS is a Breaking Change [P3027R0](https://isocpp.org/files/papers/P3027R0.html): The "rebuttal" paper that explains why the feature is so dangerous for existing codebases.

The Pipeline/Pizza Operator [P2011R0](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2011r0.html).

[Barry Revzin's History of UFCS](https://brevzin.github.io/c++/2019/04/13/ufcs-history/).

Herb Sutter's GotW [#84](http://www.gotw.ca/gotw/084.htm).

And the many reddit posts, google group discussions, stdproposals about UFCS.