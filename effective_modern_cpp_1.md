# Item 6
* 警惕auto被类型推导为某个意想不到的proxy class（ie. vector<bool>::operator[]）;

# Item 7

* 大括号可以用来完成uniform initialization，且不会发生隐式narrowing conversion.

* 一般来说，`Object o{xxx};`会优先匹配参数是std::initializer_list的构造函数

* 类内非静态成员不能用()初始化

* * `Object o();` => Error
  * `Object o{};` => OK

* `auto t = {xxx};`, t类型被推断为std::initializer_list

* * `Object o{};`调用默认构造函数
  * `Object o{{}};`或者`Object o({})`调用参数为空的std::initializer_list的构造函数

# Item 8

* 0，NULL和nullptr三者，用nullptr表示空指针(之前单独总结过，不多说了)（逃

# Item 9

* With a alias template:

  ```cpp
  template<typename T>                          // MyAllocList<T>
  using MyAllocList = std::list<T, MyAlloc<T>>; // is synonym for
                                                // std::list<T,
                                                //     MyAlloc<T>>

  MyAllocList<Widget> lw;                       // client code

  ```

  ```cpp
  template<typename T>
  class Widget {
  private:
  MyAllocList<T> list;
  ...
  };

  ```

* And with a typedef:

  ```cpp
  template<typename T>
  // MyAllocList<T>::type
  struct MyAllocList {
  // is synonym for
  typedef std::list<T, MyAlloc<T>> type; // std::list<T,
  };
  //MyAlloc<T>>
  MyAllocList<Widget>::type lw;
  // client code

  ```

  ```cpp
  template<typename T>
  class Widget {
  // Widget<T> contains
  private:
  // a MyAllocList<T>
  typename MyAllocList<T>::type list;
  // as a data member
  ...
  };

  ```

* c++14运用alias template的一点改进
```cpp
std::remove_const<T>::type
std::remove_const_t<T>
// C++11: const T → T
// C++14 equivalent
std::remove_reference<T>::type
std::remove_reference_t<T>
// C++11: T&/T&& → T
// C++14 equivalent
std::add_lvalue_reference<T>::type
 // C++11: T → T&
std::add_lvalue_reference_t<T>
 // C++14 equivalent

```

# Item10

* 一般来说，用scoped enums代替unscoped enums。
  * scoped emums带来了namespace pollution。
  * unscoped enums的成员会隐式转化成整数类型。scoped enums则不会（但可以显式转化）。（这个带来了一点好处。见下 
* tuple和unscoped enum搭配更佳， 免去记住所有参数序列。否则就得...

```cpp
template<typename E>
constexpr auto toUType(E enumerator) noexecpt
{
  return static_cast<std::underlying_type_t<E>>(enumerator);
}

//...

auto val = std::get<toUType(UserInFoFields::uiEmail)>(uInfo);
```

# Item 11

* c++11前常常将成员函数声明放在private区域而不实现，以防止编译器自动生成。现在有了`=delete`

* `=delete`不光能用于成员函数。还能通过讲“重载函数”声明为`=delete`防止函数的参数被隐式转换。

* `=delete`还能用于阻止template instantiation。

# Item 12

* override虚函数的时候加上 `override`关键词。

* member function reference qualifiers能区分`*this`的类别（类似于以前的const

# Item 13

* 在c++98的时候，容器的非const对象无法直接获得const_iterator，一般通过强制转化，或者用绑定到一个reference-to-const。

* 在c++98，const_iterator不能用于在插入或者删除的函数指示位置。

* c++11提供了`begin`和`end`函数（包括许多容器的成员函数和非成员函数的通用版本）。但是c++14才提供了（这里指的是non-member版本）`cbegin`, `cend`, `rbegin`, `rend`和`crend`。有个用c++11实现`cbegin`的例子。
  ```cpp
  template <class C>
  auto cbegin(const C& container)->decltype(std::begin(container))
  {
      retrurn std::begin(container)
  }
  ```

# Item 14

* exception-neutral函数(指这类函数：它本身不抛出异常，但是它调用的函数可能会抛出异常)不能加noexcept

* 典型地，move operations，swap，memory deallocation函数， destructor都默认noexcept

# Item 15

* constexpr对象是编译期常量。(不过准确的说，应该`translation`期间的常量，包括编译`compile`和链接`link`)

* 对于constexpr函数来说，有两种情况。这意味着不需要对运行期和编译期执行的函数进行重载

    * 如果所有参数都是编译期常量，函数将在编译期就得到结果。

    * 当一个或多个参数不是编译期常量的时候，就和普通函数一样，在运行期执行。

* constexpr是函数接口的一部分，不能随意去除

# Item 16

* 对于某个需要同步的对象来说，用std::atomic就够了。但是对于多个对象作为一个整体需要同步的时候，往往需要用到mutex。

# Item 17

* c++11开始，特殊成员函数有
    * defualt constructor 默认构造函数
    * destructor 析构函数
    * copy constructor 拷贝构造函数
    * copy assignment operator 拷贝赋值函数
    * move constructor 移动构造函数
    * move assignment operator 移动赋值函数

* 对于后两个移动操作有关的函数(构造和赋值)来说，编译器默认生成的版本的行为是对所有非static成员调用执行对应(构造或赋值)的移动操作。对于带有继承的类来说，还会对基类进行对应的移动操作。但是，遇到不支持移动操作的对象，所以会用copy来替代move操作。

* 完成上述操作的核心是将std::move用于相应的对象(即非static成员或者基类部分)，然后根据函数重载规则，会调用相应的copy和move函数。

* 这两个移动操作不是相互独立的。如果声明了其中一个，编译器将不会自动生成另一个。

* 声明了拷贝操作(拷贝构造函数和拷贝赋值函数),那么编译器将不会生成移动操作的版本。反之亦然。

* c++98开始就有个`Rule of Three`，即析构函数，拷贝构造函数，拷贝赋值函数三者自定义了任意一个，那一般来说其他两个也得自定义，编译器生成的很可能是错误的行为。

* 总结：下面三个条件都成立，则移动操作将会被编译器生成：

    * 没有声明拷贝操作
    * 没有声明移动操作
    * 没有声明析构函数

* 想让编译器生成可以显示地用`=default`。尽量别依赖编译器隐式生成的规则，一是能规避复杂规则带来的bug，二是让代码意图明显。

* 成员函数模板不会影响编译器生成这些特殊成员函数。