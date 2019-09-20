name: title-layout
layout: true
class: center, middle, title

---
name: basic-layout
layout: true
class: left, top

---
name: title
template: title-layout

# Reducing Template Compilation Overhead
.footnote[Jorg Brown, <jorg.brown@gmail.com>, [@jorgbrown](https://twitter.com/jorgbrown)]

???
Intro:
- I'm Jorg Brown
- I work with Chandler Carruth on Google's C++ PL platform, specializing in RISC-V
- Obsessed with performance and optimization.

I've wanted to have given a talk at CppCon for some time, and finally have something worthy of talking about.

There's a recurring theme at work: template compilation overhead.

Sometimes, it's a case of a team wondering why their builds are so slow.

A few months ago, we switched from libstdc++ to libc++, and while mostly an improvement, a handful of builds didn't work anymore.

---
name: Background
class: middle, center

![First Guess](templates_memory_bender_meme.jpg)

???
When I see the compiler running out of RAM, I can just blame templates without even looking.  Not much else can so easily run a compiler into the ground.  But before I do that, let me go over some background.

---
name: Background
class: left, middle

# Reducing Template Overhead
## First, some background
1. BubbleSort is better than you think... as long as N is tiny.
1. Hard to measure when things are going well
1. Compile-time calculation can be surprising

???
BubbleSort isn't just OK for low N; it's OPTIMAL up to N=3.

Even fairly bad template code does fine in small situations.

On the flip side, template code can be far worse than BubbleSort.

---
name: Background
class: left, middle

### It's hard to measure template overhead...

* Typically the overhead of starting up a compiler and reading in the standard
  headers dwarfs most template instantiations, so using the OS to measure total
  allocations is noisy
   * For clang, use -Xclang -print-stats to get fine detail; it even works in
     Compiler Explorer.
   * Dump all types: compile with "-g", then "dwarfdump -debug-types
     -debug-pubtypes -debug-gnu-pubtypes foo.o"

???
Well-designed O(n) template code is in the noise, as far as overhead goes.
So is O(n^2) overhead, typically.

---
name: Background
class: left, middle

### Compile-time basics: functions

```cpp
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  int fib = Fibonacci(15);
  return fib;
}
```

???
Inefficient at runtime, but compiles quickly.

---
name: Background
class: left, middle

### Compile-time basics: functions

```cpp
*constexpr
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  int fib = Fibonacci(15);
  return fib;
}
```

???
Marking the function constexpr doesn't mean that it gets executed at compile-time; just that it could.  You have to evaluate it in a constexpr context to force it to be evaluated at compile-time.

---
name: Background
class: left, middle

### Compile-time basics: functions

```cpp
constexpr
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  `constexpr` int fib = Fibonacci(15);
  return fib;
}
```

???
OK Great, now it's in a compile-time context.  But let's up that from 16 to 27...

---
name: Background
class: left, middle

### Compile-time basics: functions

```cpp
constexpr
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  // Fails to compile with clang due to "exceeding step limit"
  // Compiles with MSVC but takes 10 seconds; 28 will fail.
  constexpr int fib = Fibonacci(`27`);
  return fib;
}
```

Trouble: constexpr functions aren't necessarily memo-ized.

???
How do we fix this?  Well, let's do a template instead.

---
name: Background
class: left, middle

### Compile-time basics: templates

```cpp
template<int n>
struct Fibonacci {
  static constexpr int value = n <= 1 ? n : Fibonacci<n - 1>::value +
                                            Fibonacci<n - 2>::value;
};

int main() {
  constexpr int fib = Fibonacci<46>::value;
  return fib;
}
```

???
Note: Specializations for <0> and <1> were necessary, because lazy evaluation for
? : doesn't block template instantiation.

---
name: Background
class: left, middle

### Compile-time basics: templates

```cpp
template<int n>
struct Fibonacci {
  static constexpr int value = n <= 1 ? n : Fibonacci<n - 1>::value +
                                            Fibonacci<n - 2>::value;
};

*template<> struct Fibonacci<0> { static constexpr int value = 0; };
*template<> struct Fibonacci<1> { static constexpr int value = 1; };

int main() {
  constexpr int fib = Fibonacci<46>::value;
  return fib;
}
```

???
OK, but that's a bit ugly, and tedious.  It's weird to have the loop termination occur outside of the function. C++17 allows us to make this clearer; anything within a false "if constexpr" block won't be examined by the compiler, other than basic syntactic regularity like balanced parens.

---
name: Background
class: left, middle

### Compile-time basics: templates

```cpp
template<int n>
struct Fibonacci {
  static constexpr int value = [] {
*   if constexpr(n > 1) {
*     return Fibonacci<n - 1>::value + Fibonacci<n - 2>::value;
*   }
    return n;
  }();
};

int main() {
  constexpr int fib = Fibonacci<46>::value;
  return fib;
}
```

???
But is there maybe a way to make a function be evaluated at compile-time?

---
name: Background
class: left, middle

#### This one weird trick...

```cpp
template<int N>
constexpr auto Fibonacci() {
  if constexpr (N > 1) {
    return std::integral_constant<int, Fibonacci<N-1>() + Fibonacci<N-2>()>{};
  } else {
    return std::integral_constant<int, N>{};
  }
}

int main() {
  constexpr int fib = Fibonacci<46>();
  return fib;
}
```

???
By putting the calculation into a template instantiation, we force the compiler
to do the calculation at compile-time, even in debug mode.

By making the return value part of the returned type, we make it possible to
avoid calling the routine at all.

Speaking of auto return types, they have an extra benefit, and it has to do with
function signatures.

---
name: Background
class: left, middle

## The overhead of function signatures

```cpp
// Can you guess what the non-mangled signatures of these 3 functions are?

std::string add1(std::string a, std::string b);



template<typename T>
T add2(T a, T b);



template<typename T>
auto add3(T a, T b) { return a + b; }
```

---
name: Background
class: left, middle

## The overhead of function signatures

```cpp
// Did you guess what the non-mangled signatures of these 3 functions were?

std::string add1(std::string a, std::string b);

*add1(std::string, std::string)

template<typename T>
T add2(T a, T b);

*std::string add2<std::string>(std::string, std::string)

template<typename T>
auto add3(T a, T b) { return a + b; }

*auto add3<std::string>(std::string, std::string)
```

???
The first case is clear.

The second case is not so clear.  It turns out that the function's return type is included in its signature whenever the return type depends on a template parameter.  The reasons for this are a bit arcane; the obvious reason would be that sometimes you might want to have a function generate a string, and you want to template its return type.

---
name: Background
class: left, middle

## The overhead of function signatures

```cpp
template<typename T>
T process(T a);

*std::string process<std::string>(std::string)

template<typename T>
typename std::enable_if_t<!std::is_same<T, void>::value, T>
process(T a);

*What's your guess?
```

???
Give them a little while to think about it...

---
name: Background
class: left, middle

## The overhead of function signatures

```cpp
template<typename T>
T process(T a);

*std::string process<std::string>(std::string)

template<typename T>
typename std::enable_if_t<!std::is_same<T, void>::value, T>
process(T a);

*std::enable_if<!std::is_same<std::string, void>::value, std::string>::type
*            process<std::string>(std::string)

```

???
That's right, it's not the return type that gets encoded, it's the way the
return type was determined.

The moral of the story is: prefer to use an auto return type unless the enable_if
is being used to remove a funtion overload from consideration.

In particular, if you're just trying to prevent a routine from being called, it's
far more clear to provide a static_assert.  (Next slide)

---
name: Background
class: left, middle

## The overhead of function signatures

```cpp
template<typename T>
typename std::enable_if_t<!std::is_same<T, void>::value, T>
process(T a);

*std::enable_if<!std::is_same<std::string, void>::value, std::string>::type
*            process<std::string>(std::string)

template<typename T>
auto process(T a) {
  static_assert(!std::is_same<T, void>::value);
  // ...
}

*auto process<std::string>(std::string)
```

---
name: lessons
template: basic-layout

### Summary of background

1. '`inline`' functions aren't necessarily inlined.
   * '`inline`' is only guaranteed to affect linkage.
2. '`constexpr`' functions are not necessarily suitable for constexpr use, not
   necessarily inlined, and usually not memo-ized.
   * '`constexpr`' only guarantees that the result can be used in situations where
     only a constant is allowed, for at least one set of inputs.
3. '`template`' instantiations are guaranteed to be memo-ized
   * ... but their functions are not necessarily inlined.
4. '`if constexpr`' and '`auto`' return types go well with templated functions.
5. Results may vary by compiler.

???
So what's inline?
Any inline function is inline.
constexpr functions, like templated functions are implicitly inline.
Templated functions are implicitly inline, as are methods in templated classes.

But people are crazy about inline and often specify things as inline even when that's not feasible, so they're not necessarily inlined.
inline functions aren't necessarily inlined even if they always produce
the same result!

So what does it mean to be inline?
The linker is OK with dupes and will throw all but one away.
An inline function that isn't used won't generate code.

---
name: custom tuple
template: basic-layout

## Case study: A custom tuple type.

#### Or "How I learned to write an O(N-cubed) template"

```cpp
template<typename... T> class MyTuple;
```

???
Seems easy enough, right?  Examining the code of a simple tuple implementation taught me a lot about how bad template code happens.  Let's examine a naive implementation.

---
name: custom tuple
template: basic-layout

#### Case study: A custom tuple type.

```cpp
template<typename... T> class MyTuple;

template<typename T>
class MyTuple<T> {
  T first;
  template<size_t N, typename TN> friend struct Getter;
};

template<typename T, typename... More>
class MyTuple<T, More...> {
  T first;
  MyTuple<More...> more;

  template<size_t N, typename TN> friend struct Getter;
};
```

???
Note that the first case stops the recursion, and the second case instantiates another MyTuple instance as a member var.  The actual implementatio of Get is done inside the Getter routine... which together are too long to fit on one slide.

---
name: custom tuple
template: basic-layout

#### Case study: A custom tuple type.

```cpp
template<size_t N, typename TN> struct Getter {
  template<typename T0, typename... MoreN>
  static TN &get(MyTuple<T0, MoreN...> &tup) {
    return Getter<N - 1, TN>::get(tup.more);
  }
};

template<typename T0> struct Getter<0, T0> {
  template<typename... Ts>
  static T0 &get(MyTuple<Ts...> &tup) { return tup.first; }
};

template<size_t N, typename... Ts>
auto get(MyTuple<Ts...> &tup)
                -> typename std::tuple_element<N, std::tuple<Ts...>>::type& {
  return Getter<N, typename std::tuple_element<N, std::tuple<Ts...>>::type>::get(tup);
}
```

???
This helper class, Getter, is templated on the index of the value it should retrieve, and the type of that value.

Explain the general case, and then the specialilzation, within Getter.

Then show the implementation of get() which has to use std::tuple_element to figure out the return value.

So... O(N)?  O(N^2)?  Worse.

GodBolt: https://godbolt.org/z/75Qe2Z

---
name: custom tuple
template: basic-layout

#### Results: custom tuple type is O(n^3) in codegen

(As measured by clang -fno-inline and objdump --disassemble)

```cpp
using T = MyTuple<bool, char, short, int, double>;

bool tryit4T() {
  T tup;
  get<0>(tup) = true;
  get<1>(tup) = 1;
  get<2>(tup) = 1;
  get<3>(tup) = 1;
  get<4>(tup) = 1.0;
  return get<0>(tup);
}
```

???
GodBolt: https://godbolt.org/z/75Qe2Z

---
name: custom tuple
template: basic-layout

(As seen at https://godbolt.org/z/o2uJWt)

<pre style='font-size:12px'>
tuple_element&lt;0ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;0ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;1ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;1ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;2ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;2ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;3ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;3ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;4ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
bool& Getter&lt;0ul, bool>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
char& Getter&lt;1ul, char>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
char& Getter&lt;0ul, char>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
short& Getter&lt;2ul, short>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
short& Getter&lt;1ul, short>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
short& Getter&lt;0ul, short>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
int& Getter&lt;3ul, int>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
int& Getter&lt;2ul, int>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
int& Getter&lt;1ul, int>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
int& Getter&lt;0ul, int>::get&lt;int, double>(MyTuple&lt;int, double>&)
double& Getter&lt;4ul, double>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
double& Getter&lt;3ul, double>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
double& Getter&lt;2ul, double>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
double& Getter&lt;1ul, double>::get&lt;int, double>(MyTuple&lt;int, double>&)
double& Getter&lt;0ul, double>::get&lt;double>(MyTuple<double>&)
</pre>

???
These are the function signatures that were generated.

O(N^2) functions generated, each with O(N) signature size.

---
name: custom tuple
template: basic-layout

Just this one line:
```cpp
get<4>(tup) = 1.0;
```
Calls:
<pre style='font-size:20px'>
 tuple_element&lt;4ul, std::tuple&lt;bool, char, short, int, double> >::type&
     get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);

 // Which calls these, in order:
 double& Getter&lt;4ul, double>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
 double& Getter&lt;3ul, double>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
 double& Getter&lt;2ul, double>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
 double& Getter&lt;1ul, double>::get&lt;int, double>(MyTuple&lt;int, double>&)
 double& Getter&lt;0ul, double>::get&lt;double>(MyTuple<double>&)
</pre>

???
Explain why that is, then ask, well, what can we do to make this better?

---
name: custom tuple
template: basic-layout

1. Change the functions to return auto& to determine the type.

```cpp
template<size_t N, typename... Ts> auto get(MyTuple<Ts...> &tup)
                -> typename std::tuple_element<N, std::tuple<Ts...>>::type& {
  return Getter<N, typename std::tuple_element<N, std::tuple<Ts...>>::type>::get(tup);
```

Becomes this:
```cpp
template<size_t N, typename... Ts> auto& get(MyTuple<Ts...> &tup) {
  return Getter<N>::get(tup);
```
<hr />
<pre style='font-size:20px'>
 tuple_element&lt;4ul, std::tuple&lt;bool, char, short, int, double> >::type&
     get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);
</pre>
Becomes:
<pre style='font-size:20px'>
 auto& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);
</pre>

???
Also, by using an auto return type, we don't need the type parameter for Getter.

So did it fix the problem?

---
name: custom tuple
template: basic-layout

Still O(N^2) to get a single element however...
```cpp
get<4>(tup) = 1.0;
```
Calls:
<pre style='font-size:20px'>
 auto& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);

 // Which calls these, in order:
 auto& Getter&lt;4ul>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
 auto& Getter&lt;3ul>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
 auto& Getter&lt;2ul>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
 auto& Getter&lt;1ul>::get&lt;int, double>(MyTuple&lt;int, double>&)
 auto& Getter&lt;0ul>::get&lt;double>(MyTuple<double>&)
</pre>

???
This is better but still O(N^2).  If only there were a way to avoid the intermediate get<>() calls...

Remember that one weird trick?

---
name: custom tuple
template: basic-layout

2 . Use function overrides to go directly to an inherited function of your choice

```cpp
template<typename... T> class MyTuple {
  // This definition is only used for MyTuple<>, the base instance.
  // Real uses are specialized below.
};

template<typename T, typename... More>
class MyTuple<T, More...> : MyTuple<More...> {
  T first;

 public:
  using MyTuple<More...>::get;
  auto& get(std::integral_constant<size_t, sizeof...(More)>) { return first; }
};

template<size_t N, typename... Ts>
auto& get(MyTuple<Ts...> &tup) {
  return tup.get(std::integral_constant<size_t, sizeof...(Ts) - 1 - N>{});
}
```

???
Rather than instantiate the rest of the tuple, we'll inherit from it.  That way we can access the other member functions directly.

This doesn't quite compile, though...

---
name: custom tuple
template: basic-layout

One tweak for the base class though:

```cpp
template<typename... T> class MyTuple {
*public:
* void get(...) = delete;
};

template<typename T, typename... More>
class MyTuple<T, More...> : MyTuple<More...> {
  T first;

 public:
  using MyTuple<More...>::get;
  auto& get(std::integral_constant<size_t, sizeof...(More)>) { return first; }
};

template<size_t N, typename... Ts>
auto& get(MyTuple<Ts...> &tup) {
  return tup.get(std::integral_constant<size_t, sizeof...(Ts) - 1 - N>{});
}
```

???

So, does this work?

---
name: custom tuple
template: basic-layout

And now...
```cpp
get<4>(tup) = 1.0;
```
Calls:
<pre style='font-size:20px'>
 auto& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);

 // Which calls:
 MyTuple&lt;double>::get(std::integral_constant&lt;unsigned long, 0ul>)
</pre>

???
So this solves the N^3 problem, right?

---
name: custom tuple
template: basic-layout

#### Still O(N^2) to instantiate MyTuple.

MyTuple&lt;A, B, C> inherits from
MyTuple&lt;B, C> which inherits from MyTuple&lt;C>.

In order to do MyTuple in O(N), we need a way to instantiate each individual type that doesn't involve the others.  So let's try inheriting from Leaf&lt;A> and Leaf&lt;B> and Leaf&lt;C> etc.

???
Next slide.

---
name: custom tuple
template: basic-layout

So let's try inheriting from Leaf&lt;A> and Leaf&lt;B> and Leaf&lt;C> etc.

```cpp
template <typename T> struct MyTupleLeaf {
  T value_;
  template<typename... Arg>
  constexpr MyTupleLeaf(Arg&&... arg) : value_(std::forward<Arg>(arg)...) {}
};

template<typename... T> class MyTuple : MyTupleLeaf<T>... {
public:
  template<typename... Args>
  explicit constexpr MyTuple(Args&&... args)
    : MyTupleLeaf<T>(std::forward<Args>(args))... {}

 template<size_t I>
 auto& get() {  /* ?? */ }
};

MyTuple<char, short, int> tuple{'c', 1, 2};
```

???

Mention the constructor API and the fact that future slides won't have it.

But how does get() work?  More importantly, doesn't this have problems if a T is mentioned
twice?

---
name: custom tuple
template: basic-layout

So let's bake a selector value into the types we derive from...

```cpp
template <`std::size_t I,` typename T>
struct MyTupleLeaf {
  T value_;
};

template<`size_t... Index,` typename... T> class MyTuple : MyTupleLeaf<`Index,` T>... {
public:
 template<size_t I>
 auto& get() {  /* ?? */ }
};

MyTuple<`0, 1, 2,` char, short, int> tuple;
```

???

But you can only have one variadic in a template specialization, so we'll need to encapsulate them in a type.
---
name: custom tuple
template: basic-layout

Encapsulate the indices into a type.

```cpp
template <std::size_t I, typename T>
struct MyTupleLeaf {
  T value_;
};

*template<typename... T> class MyTuple;

template<size_t... Index, typename... T>
class MyTuple<`std::index_sequence<Index...>,` T...> : MyTupleLeaf<Index, T>... {
public:
 template<size_t I>
 auto& get() {  /* ?? */ }
};

MyTuple<`std::index_sequence<0, 1, 2>,` char, short, int> tuple;
```

???
And now we can implement get!  Remember std::integral_constant, the one weird trick?
---
name: custom tuple
template: basic-layout

Now, implement get()

```cpp
template <std::size_t I, typename T>
struct MyTupleLeaf {
  T value_;
* T& get(std::integral_constant<size_t, I>) { return value_; }
};

template<typename... T> class MyTuple;

template<size_t... Index, typename... T>
class MyTuple<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
* using MyTupleLeaf<Index, T>::get...;
public:
  template<size_t I>
* auto& get() { return get(std::integral_constant<size_t, I>{}); }
};

MyTuple<std::index_sequence<0, 1, 2>, char, short, int> tuple;
*short& get1 = tuple.get<1>();
```
???
But what if you don't have C++17?  The technique is applicable elsewhere, so...

---
name: custom tuple
template: basic-layout

Make MyTuple generate the index sequence!

```cpp
template <std::size_t I, typename T>
struct MyTupleLeaf {
  T value_;
  T& get(std::integral_constant<size_t, I>) { return value_; }
};

*template<typename... T>
*class MyTuple : MyTuple<std::make_index_sequence<sizeof...(T)>, T...> {
*};
template<size_t... Index, typename... T>
class MyTuple<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
  using MyTupleLeaf<Index, T>::get...;
public:
  template<size_t I>
  auto& get() { return get(std::integral_constant<size_t, I>{}); }
};

*MyTuple<char, short, int> tuple;
```
???
But what if you want a tuple with a first type that is an index sequence?

---
name: custom tuple
template: basic-layout

Make MyTuple a using statement, to separate it from its implementation.

```cpp
template <std::size_t I, typename T>
struct MyTupleLeaf {
  T value_;
  T& get(std::integral_constant<size_t, I>) { return value_; }
};

template<typename... T> class MyTuple`Impl`;
template<size_t... Index, typename... T>
class MyTuple`Impl`<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
  using MyTupleLeaf<Index, T>::get...;
public:
  template<size_t I>
  auto& get() { return get(std::integral_constant<size_t, I>{}); }
};

template<typename... T>
 `using` MyTuple = MyTuple`Impl`<std::make_index_sequence<sizeof...(T)>, T...>;

MyTuple<char, short, int> tuple;
```

???
But what about C++14?

C++14 has no variadic using declaration...
---
name: custom tuple in C++14
template: basic-layout

But before C++17, you can't use variadic using statements.  What about C++14?

```cpp
template<typename... T> class MyTupleImpl;
template<size_t... Index, typename... T>
class MyTupleImpl<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
* using MyTupleLeaf<Index, T>::get...;
public:
  template<size_t I>
  auto& get() { return get(std::integral_constant<size_t, I>{}); }
};
```

---
name: custom tuple in C++14
template: basic-layout

In C++14, we could do it with an extra template parameter...

```cpp
template<typename... T> class MyTupleImpl;
template<size_t... Index, typename... T>
class MyTupleImpl<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
//using MyTupleLeaf<Index, T>::get...;
public:
//template<size_t I>
//auto& get() { return get(std::integral_constant<size_t, I>{}); }
  template<size_t I`, typename TType`>
  auto& get() {
    `MyTupleLeaf<I, TType> &leaf = *this;`
    return `leaf.`get(std::integral_constant<size_t, I>{});
  }
};

MyTuple<char, short, int> tuple;
//short& get1 = tuple.get<1>();
short& get1 = tuple.get<1`, short`>();
```

???
But no one wants an extra template parameter for get.  Can we somehow deduce it?

When you pass an argument to a function, the compiler can deduce its type.  It can even deduce specific parts of the type.  For example:
---
name: custom tuple in C++14
template: basic-layout

Better yet, if we pass the tuple as a parameter, we can deduce the type:

```cpp
template<typename... T> class MyTupleImpl;
template<size_t... Index, typename... T>
class MyTupleImpl<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
//using MyTupleLeaf<Index, T>::get...;
public:
//template<size_t I>
//auto& get() { return get(std::integral_constant<size_t, I>{}); }
* template<size_t I, typename TType>
* auto& get_leaf(MyTupleLeaf<I, TType> *leaf) { return *leaf; }
  template<size_t I>
  auto& get() {
*   auto &leaf = get_leaf<I>(this);
    return leaf.get(std::integral_constant<size_t, I>{});
  }
};

MyTuple<char, short, int> tuple;
short& get1 = tuple.get<1>();
```

???
Of course, no reason to call the get<> function anymore.
---
name: custom tuple in C++14
template: basic-layout

At this point, we don't need the get<> function in MyTupleLeaf.

```cpp
template <std::size_t I, typename T> struct MyTupleLeaf { T value_; };

template<typename... T> class MyTupleImpl;
template<size_t... Index, typename... T>
class MyTupleImpl<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
public:
  template<size_t I, typename TType>
  auto& get_leaf(MyTupleLeaf<I, TType> *leaf) { return *leaf; }
  template<size_t I>
  auto& get() {
*   return get_leaf<I>(this).value_;
  }
};

MyTuple<char, short, int> tuple;
short& get1 = tuple.get<1>();
```

???
But what about C++11?  What if we can't deduce the function return type?
---
name: custom tuple in C++11
template: basic-layout

In C++11, we can't deduce return types, so let's fix that, too...

```cpp
template <std::size_t I, typename T> struct MyTupleLeaf { T value_; };

template<typename... T> class MyTupleImpl;
template<size_t... Index, typename... T>
class MyTupleImpl<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
public:
  template<size_t I, typename TType>
  `MyTupleLeaf<I, TType>`& get_leaf(MyTupleLeaf<I, TType> *leaf) { return *leaf; }

  template<size_t I>
  auto get() `-> decltype(get_leaf<I>(this).value_)&` {
    return get_leaf<I>(this).value_;
  }
};

MyTuple<char, short, int> tuple;
short& get1 = tuple.get<1>();
```
C++11 has no make_index_sequence; Abseil (among others) does.

???
Now let's move on to case study #2.

These arose when I was looking at the "-g" output from clang, which was causing some builds to fail after switching from libc++ to libstdc++.

---
name: std library is_same
template: basic-layout

## Case study #2: libc++ and its debug symbols

Consider std::is_same:

```cpp
template<class T, class U>
struct is_same : std::false_type {};

template<class T>
struct is_same<T, T> : std::true_type {};
```

Do you see why this was a huge burden on debug symbols?

???
Pause to give people time to think...

---
name: std library is_same
template: basic-layout

### Case study #2: libc++ and its debug symbols

Consider debug symbols derived from std::is_same:

```cpp
std::is_same<char, char>    std::is_same<float, char>
std::is_same<char, int>     std::is_same<float, int>
std::is_same<char, float>   std::is_same<float, float>
std::is_same<char, double>  std::is_same<float, double>
std::is_same<int, char>     std::is_same<double, char>
std::is_same<int, int>      std::is_same<double, int>
std::is_same<int, float>    std::is_same<double, float>
std::is_same<int, double>   std::is_same<double, double>

... and on and on.  literally N^2 classes instantiated.
```

???
So, how can we make this O(N) instead of O(N^2)?

---
name: std library is_same
template: basic-layout

### Case study #2: libc++ and its debug symbols

Better (in some ways) would be:
```cpp
template<typename T>
struct Wrapper {
  static constexpr bool IsSame(Wrapper<T>*) { return true; }
  static constexpr bool IsSame(void *)      { return false; }
};

// Old code:
  if (std::is_same<Type1, Type2>::value) ...

// New code:
  if (Wrapper<Type1>::IsSame<(Wrapper<Type2>*)(nullptr)) ...
```

???
Explain that the latter version only ever instantiates two functions per type.

---
name: std library is_same
template: basic-layout

### Case study #2: libc++ and its debug symbols

With the wrapper version, we have:

```cpp
Wrapper<char>
Wrapper<char>::IsSame(Wrapper<char>*)
Wrapper<char>::IsSame(void *);
Wrapper<int>
Wrapper<int>::IsSame(Wrapper<int>*)
Wrapper<int>::IsSame(void *);
Wrapper<float>
Wrapper<float>::IsSame(Wrapper<float>*)
Wrapper<float>::IsSame(void *);
Wrapper<double>
Wrapper<double>::IsSame(Wrapper<double>*)
Wrapper<double>::IsSame(void *);
... and on and on.  literally N classes instantiated, and 2*N functions.
```

???
So, not a win for code clarity, and really not much of a win if you're not comparing a lot of types.  We can do better.

---
name: std library is_same
template: basic-layout

### Case study #2: libc++ and its debug symbols

```cpp
// Two more tweaks:
struct WrapperBase {
  static constexpr bool IsSame(void *)      { return false; }
};
 
template<typename T>
struct Wrapper : WrapperBase {
  using WrapperBase::IsSame;
  static constexpr bool IsSame(Wrapper<T>*) { return true; }
};

template<typename Type1, typename Type2> using is_same =
  std::integral_constant<bool, Wrapper<Type1>::IsSame((Wrapper<Type2>*)(nullptr))>;

// Code is same either way:
  if (std::is_same<Type1, Type2>::value) ...
```

???
So, how does this do?

( My use only: https://godbolt.org/z/oa8TSJ )

---
name: std library is_same
template: basic-layout

### Case study #2: libc++ and its debug symbols

With the wrapper version, we have:

```cpp
WrapperBase::IsSame(void *);
Wrapper<char>
Wrapper<char>::IsSame(Wrapper<char>*)
Wrapper<int>
Wrapper<int>::IsSame(Wrapper<int>*)
Wrapper<float>
Wrapper<float>::IsSame(Wrapper<float>*)
Wrapper<double>
Wrapper<double>::IsSame(Wrapper<double>*)
... and on and on.  literally N classes instantiated, and N functions.
```

???
So, now it's more clearly a win.

A better example would be std::conditional

---
name: std library conditional
template: basic-layout

## Case study #2: libc++ and its debug symbols

Consider std::conditional:

```cpp
template<bool B, class T, class F>
struct conditional { typedef T type; };

template<class T, class F>
struct conditional<false, T, F> { typedef F type; };
```

Do you see why this was a huge burden on debug symbols?

???
Pause to give people time to think...

---
name: std library conditional
template: basic-layout

### Case study #2: libc++ and its debug symbols

Consider std::conditional:

```cpp
std::conditional<true, char, void>
std::conditional<true, const char (&)[22], void>
std::conditional<true, double, void>
std::conditional<true, float, void>
std::conditional<true, int, void>
std::conditional<true, std::deque<int>, void>
std::conditional<true, std::string, void>
std::conditional<true, std::string_view, void>
std::conditional<true, std::vector<std::string>&, void>
std::conditional<true, void (*)(), void>
... and on and on
```

???
How can we fix this?

Pause to give people time to think...

---
name: std library conditional
template: basic-layout

### Case study #2: libc++ and its debug symbols

```cpp
template<bool B, class T, class F> struct conditional { typedef T type; }; 
template<class T, class F> struct conditional<false, T, F> { typedef F type; };
```
Better would be:
```cpp
template<bool B> struct conditional {
  template <class T, class F> using type = T;
};

template<> struct conditional<false> {
  template <class T, class F> using type = F;
};
```

This was checked into libc++ a few months ago...

???
Explain that the latter version only ever instantiates two types.

OK, next example: add_const.

---
name: std library add_const
template: basic-layout

## Case study #2: libc++ and its debug symbols

Consider std::add_const:

```cpp
template< class T>
struct add_const {
  using type = const T;
};

template <class T>
using add_const_t = typename add_const<T>::type;
```

Since this only takes one type, this wasn't a debug symbol burden...

???
But what about the obvious improvement?

---
name: std library add_const
template: basic-layout

### Case study #2: libc++ and its debug symbols

Consider this improvement to std::add_const:

```cpp
template <class T>
struct add_const {
  using type = const T;
};

template <class T>
//using add_const_t = typename add_const<T>::type;
using add_const_t = const T;
```

How much better will this be?

???
Pause to give people time to think...

---
name: Hyrum STOP
class: middle, center

![First Guess](HyrumStop.png)

???
Hyrum's Law states: "With a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviors of your system will be depended on by somebody."

---
name: std library add_const
template: basic-layout

### Case study #2: libc++ and its debug symbols

```cpp
template <class T>
struct add_const {
  using type = const T;
};

template <class T>
using add_const_t = const T;

template<typename T>
T nop(add_const_t<T>);

int main() {
  int i = 0;
  return nop(i);
}
```


???
Explain that this code compiles, but it wouldn't if nop had been declared using the real std::add_const_t, because the real one blocks template type deduction.

Talk is basically over, time for one more weird trick...

---
name: one more weird trick
template: basic-layout

## One more thing^W weird trick

```cpp
// Using C++17 fold expressions to check if any of these types is void
template<typename... T>
constexpr bool are_any_void() {
    return (std::is_void<T>::value || ...);
}
```

But how would you do this in C++11?

---
name: one more weird trick
template: basic-layout

### One more weird trick : std::integral_constant again!

```cpp
constexpr bool Or(std::initializer_list<std::integral_constant<bool, false>>) {
  return false;
}
constexpr bool Or(std::initializer_list<bool>) {
  return true;
}
```

The first overload is chosen only if all the types are integral_constant<bool, false>

The second overload gets the rest, since integral_constant<bool, x> implicitly converts to bool.

---
name: one more weird trick
template: basic-layout

### One more weird trick : std::integral_constant again!

```cpp
constexpr bool Or(std::initializer_list<std::integral_constant<bool, false>>) {
  return false;
}
constexpr bool Or(std::initializer_list<bool>) {
  return true;
}
// Using C++11 to check if any of these types is void
template<typename... T>
constexpr bool are_any_void() {
  return Or( {typename std::is_void<T>::type()...} );
}

}
```

Once again, integral_constant and function overloads make a great team.

---
name: Disclaimer
template: title-layout

# Disclaimer: compilers vary and evolve

---
name: questions
template: title-layout

# Questions?

Reference1: http://talesofcpp.fusionfenix.com/post-22/true-story-efficient-packing

Reference2: https://ldionne.com/2015/11/29/efficient-parameter-pack-indexing/

libc++ change https://github.com/llvm-mirror/libcxx/commit/80f67f31b4de5df3de185040a388d57e356c230f

???

---
name: custom tuple
template: basic-layout

LAST SLIDE
LAST SLIDE
LAST SLIDE
LAST SLIDE
LAST SLIDE


???
Older code: https://godbolt.org/z/5RZkb5
Newest code: https://godbolt.org/z/vgGT2I

---
name: conclusion
template: title-layout

## Hopefully this at least serves a basis for understanding C++ templates!

