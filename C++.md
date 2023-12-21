# 说一说你了解的关于lambda函数的全部知识
> https://c.biancheng.net/view/3741.html
> https://zhuanlan.zhihu.com/p/384314474

lambda表达式的语法形式：
[ capture ] ( params ) opt -> ret { body; };

其中：
capture表示捕获列表，params为参数列表，opt为函数类型，ret为返回值类型，body为函数体。

由于C++11的返回值类型后置特性，可省略返回值类型，表示为：
[ capture ] ( params ) opt { body; };

下面对捕获列表做详细的说明：
[]：不捕获任何外部变量
[=]：以复制的方式捕获外部作用域的所有变量，内部不可修改
[&]：以引用的方式捕获外部变量，内部可修改
[=, &a]：以复制的方式捕获外部变量，以引用方式捕获变量a
[a]：以复制的方式捕获变量a，不捕获其他变量
[this]：捕获当前类的this指针
要修改复制的变量，需要将opt函数类型设置为mutable

编译器如何看待lambda表达式（lambda的类型）
编译器会把lambda表达式翻译为重载了opeator()的类，即仿函数。
如
```
auto plus = [] (int a, int b) -> int { return a + b; }
int c = plus(1, 2);
```
会被编译器翻译为：
```
class foo {
  int operator() (int a, int b) const {
      return a + b;
  }
};

foo f;
int c = f(1, 2);
```
更复杂一些的lambda表达式：
```
int x = 1; int y = 2;
auto plus = [=] (int a, int b) -> int { return x + y + a + b; };
int c = plus(1, 2);
```
会被编译器翻译为：
```
class foo {
  foo (int xx, int yy) : x(xx), y(yy) {}

  int operator() (int a, int b) const {
      return x + y + a + b;
  }
private:
   int x;
   int y;
};

int x = 1; int y = 2;
foo f(x, y);
int c = f(1, 2);
```
这里可以看出，按值捕获时，会把变量作为类的数据成员，且生成的operator()是const类型，因此无法对成员进行修改。如果想要在函数内修改值，需要设置函数类型为mutable。

# C++ 11有哪些新特性

# 讲一下程序的内存分区/内存模型？
在C++程序中，内存可以分为如下区域：
1. 代码区：也称为文本区，用于存储程序的二进制代码。该区域是只读的，防止在程序运行过程中意外修改自身的代码。
2. 堆区：程序中动态分配内存的地方。如使用new操作符创建对象时，会在堆上分配内存。由于堆的内存分配和释放都需要手动管理，如果管理不当，可能会出现内存泄漏等问题。
3. 栈区：存储函数调用上下文和局部变量的内存区域。当函数发生调用时，函数的参数、返回地址、局部变量都会压入栈中。当函数返回时，这些参数也会从栈中弹出。栈的内存和分配是自动的，这使得它比堆更高效，但空间更有限。
4. 全局/静态存储区：存储全局/静态变量的内存区域。这些变量的生命周期是整个程序的运行时间，而不仅仅是在它们所在的函数/代码块。在C语言中，全局/静态变量区还分为已初始化和未初始化的，而在C++中，他们共用一个内存区域，若在该区域的变量没有初始化，则会被自动初始化，如int类型会被初始化为0。
5. 常量存储区：用于存储常量的内存区域。这里面存放的是常量，不允许修改。

和linux内核的内存布局的区别：
linux内核多了映射区（用于动态库，文件映射等），全局/静态存储区分为了 全局数据段（已初始化的）和BSS段（未初始化的）

# move 了解吗？有什么作用？
> https://www.cnblogs.com/S1mpleBug/p/16703328.html
> https://zhuanlan.zhihu.com/p/580797507
C++的move语义是一种高效的资源转移机制，允许我们把一个临时对象的资源“移动”到另一个对象，而不是复制整个对象。这种语义在处理大量数据或者动态分配的对象时，可以提高程序运行效率。

## move的实现
```
template<typename _Tp>
  constexpr typename std::remove_reference<_Tp>::type&&
  move(_Tp&& __t) noexcept
  { return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); }
```
move的实现非常简单，就是对传入的万能引用（无论你是左值引用还是右值引用）都强制转换为右值引用。
传递的是左值，推导为左值引用，仍旧static_cast转换为右值引用。
传递的是右值，推导为右值引用，仍旧static_cast转换为右值引用。

其中remove_reference的实现：
```
template <class T>
    struct remove_reference
    {
        typedef T type;
    };
    template <class T>
    struct remove_reference<T&>
    {
        typedef T type;
    };
    template <class T>
    struct remove_reference<T&&>
    {
        typedef T type;
    };
```
不管模版T被推导为type，type&, type&&，都可以提取出原本的type
前面提到右值引用，下面说明一下左、右值引用：
左值：表达式结束后仍存在的持久对象（代表一个在内存中占用确定位置的对象）
右值：表达式结束后不再存在的临时对象（不在内存中占有确定位置的对象）
左值引用：绑定在左值，对某个左值的引用，符号为&
右值引用：绑定在临时对象，对某个右值的引用，符号为&&

### 引用折叠
一个模版函数的入参和实参 引用的绑定关系，可以分为
T&-&：左值-左值
T&-&&：左值-右值
T&&-&：右值-左值
T&&-&&：右值-右值
C++不允许出现引用的引用，因此当出现引用的引用时，会折叠会一个引用。规则是：只要出现左值引用，就会折叠为左值引用。否则折叠为右值引用。
因此对于T&& 的模版参数，可以绑定为左值引用，也可以绑定为右值引用，被称为万能引用。
### 完美转发
对于万能引用T&&的模版函数，如果继续在内部调用其他函数（无法分化为左值/右值引用），对其他函数来说，T&& t变量是一个左值变量，因为是函数的局部变量，可以取地址。
使用std::forward 可以保持原来的值 属性不变。如果原来的参数是左值，经过forward处理后还是左值。如果原来的参数是右值，经过forward处理后就还是右值。
> https://zhuanlan.zhihu.com/p/369203981

在STL中，随处可见这种问题。比如C++11引入的emplace_back，它接受左值也接受右值作为参数，接着，它转调用了空间配置器的construct函数，而construct又转调用了placement new，placement new根据参数是左值还是右值，决定调用拷贝构造函数还是移动构造函数。

# 智能指针的原理、常用的智能指针及实现
