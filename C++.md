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
> https://www.cnblogs.com/xumaomao/p/15175448.html
> https://zhuanlan.zhihu.com/p/64543967
## 智能指针原理
**智能指针主要用于管理堆区分配的内存，它将普通的指针封装为一个栈区的对象。在栈对象的生命周期结束后，会在析构函数中自动释放堆区的申请的内存，从而防止内存泄漏。**
简要的说，智能指针使用了C++的RAII机制来GC，在智能指针对象作用域结束后，会自动做内存释放的相关动作，不需要我们再手动去操作内存。

RAII是C++的发明者Bjarne Stroustrup提出的概念，RAII全称是“Resource Acquisition is Initialization”，直译过来是“资源获取即初始化”，也就是说在构造函数中申请分配资源，在析构函数中释放资源。因为C++的语言机制保证了，当一个对象创建的时候，自动调用构造函数，当对象超出作用域的时候会自动调用析构函数。所以，在RAII的指导下，我们应该使用类来管理资源，将资源和对象的生命周期绑定。
C++ 中有四种智能指针：auto_pt、unique_ptr、shared_ptr、weak_ptr 其中后三个是 C++11 支持，第一个已经被 C++11 弃用且被 unique_prt 代替，不推荐使用。下文将对其逐个说明。
## 常见的智能指针
1. std::auto_ptr: auto_ptr是拷贝语义，在拷贝后原对象失效（指向NULL），可能引发很严重的问题。
2. std::unique_ptr: 作为auto_ptr的改进，unique_ptr 对其持有的堆内存具有**唯一拥有权**，也就是 std::unique_ptr 不可以拷贝或赋值给其他对象。unique_ptr 类的拷贝构造函数和赋值运算符（operator =）被标记为 delete。
既然 std::unique_ptr 不能复制，那么如何将一个 std::unique_ptr 对象持有的堆内存转移给另外一个呢？答案是使用移动构造
3. std::shared_ptr: unique_ptr对其持有的资源具有独占性，而shared_ptr对持有的堆内存具有**共享所有权**。多个shared_ptr对象可共享占用同一块堆内存，当最后一个shared_ptr对象析构时，自动释放其持有的内存。
4. std::weak_ptr: weak_ptr是对shared_ptr管理的堆内存具有**弱引用**的智能指针，它不控制所指向对象生存期，不会改变shared_ptr的引用计数。一旦最后一个所指向对象的shared_ptr被销毁，所指向的对象就会被释放，即使此时有weak_ptr指向该对象，所指向的对象依然被释放。
weak_ptr用于解决shared_ptr循环引用时，引用计数无法降为0导致的内存泄漏问题。
> https://blog.csdn.net/Xiejingfa/article/details/50772571
## shared_ptr实现
> https://zhuanlan.zhihu.com/p/64543967
```
template<typename T>
class sharedPtr {
private:
    T* _ptr;
    int* _count;
public:
    // 构造函数
    sharedPtr(T* ptr) : _ptr(ptr) {
        _count = new int(1);
    }
    // 拷贝构造函数
    sharedPtr(const sharedPtr& rhs): _ptr(rhs._ptr), _count(rhs._count){
        *_count++;
    }
    // 赋值运算符
    sharedPtr& operator= (const sharedPtr& rhs) {
        if (this == &rhs) {
            return *this;
        }
        reset(); // 减少左边引用计数
        _ptr = rhs._ptr;
        _count = rhs._count;
        *_count++;
        return *this;
    }

    // 析构函数
    ~sharedPtr() {
        reset();
    }
    // 获取原始指针
    T* get() {
        return _ptr;
    }
    // 获取引用计数
    int count() {
        return _count ? *_count ? 0;
    }
    void reset() {
        if (_count) {
            --*_count;
            if (*_count == 0) {
                delete _ptr;
                delete _count;
            }
        }
    }
};
```

# 说一说你理解的内存对齐以及原因
内存对齐是一种编程优化技术，用于提高计算机硬件的性能和效率。内存对齐是指将数据按照特定的对齐边界进行存储，通常是为了使数据能够更快地被访问和操作。
内存对齐的原因有以下几点：
1. 性能优化：许多处理器访问对齐数据的方式比访问未对齐数据更快。由于CPU通常一次从内存中读取固定大小的数据块（称为字或字节），因此将数据对齐可以减少CPU需要读取的数据量，从而提高性能。
2. 硬件效率：某些硬件指令只能操作对齐的数据。如果数据没有对齐，硬件需要额外的步骤来处理，这可能会降低硬件的执行效率。
3. 减少内存碎片：对齐的数据可以更有效地利用内存空间，因为它们可以连续存储，而不是分散在内存中。这有助于减少内存碎片，提高内存利用率。
4. 兼容性：某些硬件平台和操作系统可能要求或优待对齐的数据。因此，遵循内存对齐规则可以帮助编写跨平台的可移植代码。
5. 避免异常：在某些编程语言中，未对齐的数据可能会导致运行时异常或错误。例如，某些处理器会抛出异常或错误，如果试图访问未对齐的内存地址。
在C++中，可以通过特定的编译器指令或结构体布局控制来指定内存对齐规则。例如，可以使用#pragma pack指令或结构体内部的alignas关键字来控制内存对齐。

# C++的多态是如何实现的？
> https://byteage.com/59.html
> https://zhuanlan.zhihu.com/p/359466948
C++支持两种多态性：编译时多态性、运行时多态性
+ 编译时多态性（静态多态）：通过重载函数实现，在编译期确定
+ 运行时多态性（动态多态）：通过虚函数实现，在运行时才确定
C++函数重载是基于编译器的name mangling机制。
编译器需要为C++中所有的函数，在符号表中生成唯一的标识符，来区分不同的函数。对于同名不同参的函数，编译器在进行name mangling操作时，通过函数名和参数类型，生成唯一的标识符，来支持函数重载。
注意：name mangling 后得到的函数标识符与返回值类型是无关的，因此函数重载与返回值类型无关。
比如，下面的几个同名的重载函数：
```
int func(int a) {
    return a;
}
double func(double b) {
    return b;
}
float func(float c) {
    return c;
}
```
在经过编译的name mangling操作后，得到的符号表中与func有关的：
```

main.o: file format mach-o 64-bit x86-64

SYMBOL TABLE:
0000000100003f80 g     F __TEXT,__text __Z4funcd
0000000100003f90 g     F __TEXT,__text __Z4funcf
0000000100003f70 g     F __TEXT,__text __Z4funci
0000000100000000 g     F __TEXT,__text __mh_execute_header
0000000100003fa0 g     F __TEXT,__text _main
0000000000000000         *UND* dyld_stub_binder
```
