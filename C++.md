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
下面将lambda的各个成分和类的各个成分对应起来就是如下的关系:

捕获列表，对应LambdaClass类的private成员。

参数列表，对应LambdaClass类的成员函数的operator()的形参列表

mutable，对应 LambdaClass类成员函数 operator() 的const属性 ，但是只有在捕获列表捕获的参数不含有引用捕获的情况下才会生效，因为捕获列表只要包含引用捕获，那operator()函数就一定是非const函数。

返回类型，对应 LambdaClass类成员函数 operator() 的返回类型

函数体，对应 LambdaClass类成员函数 operator() 的函数体。

引用捕获和值捕获不同的一点就是，对应的成员是否为引用类型。


这里可以看出，按值捕获时，会把变量作为类的数据成员，且生成的operator()是const类型，因此无法对成员进行修改。如果想要在函数内修改值，需要设置函数类型为mutable。

# C++ 11有哪些新特性
> https://www.jianshu.com/p/aaf789b64667

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
> https://zhuanlan.zhihu.com/p/596288935
C++支持两种多态性：编译时多态性、运行时多态性
+ 编译时多态性（静态多态）：通过重载函数、模版实现，在编译期确定
+ 运行时多态性（动态多态）：通过虚函数实现，在运行时才确定

## 静态多态
重载 允许多个同名函数，而这些函数的参数列表不同，具体的参数类型，在编译期就能确认。
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
g++ main.cc -o main.o && objdump -t main.o
```

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
其中，__Z是GCC的规定，4是函数名func的长度，d是函数参数double，f是函数参数float类型，i是函数参数int类型。经过name mangling后可以发现，函数重载与函数返回值类型无关，只与函数名和函数参数类型有关。

## 动态多态
- 动态：在编译期无法确认调用的是基类还是派生类的虚成员函数**vf**，需要等待执行期才能确定。而将vf设置为虚成员函数，只需要在vf的声明前，加上virtual关键字。
- 多态：由基类指针，调用基类和派生类的虚成员函数。
编译器如何实现动态多态？
编译器会为每个存在虚函数的类对象的内存中插入一个虚函数表指针（Virutal Function Pointer，vtpr），指向虚函数表(Virutal Function Table，vtbl)。虚表就是一个数组，槽位包含类的type_info，虚函数的地址。
动态多态是通过vtpr、vtbl实现的。下面我们进一步讨论下，动态多态从编译到运行，哪些任务是在编译期完成，哪些任务是在执行期决议的。
比如：
```
class Base {
public:
  Base() = default;
  virtual 
  void print()       { std::cout <<"Actual Type:  Base" << std::endl; }
  void PointerType() { std::cout <<"Pointer Type: Base" << std::endl;}
  virtual ~Base()    { std::cout <<"base-dtor"<< std::endl;}
};

class Derived : public Base{
public:
  Derived() = default;
  void print()       { std::cout <<"Actual Type:  Derived" << std::endl; }
  void PointerType() { std::cout <<"Pointer Type: Derived" << std::endl;}
  ~Derived()         { std::cout <<"derived-dtor ";}
private:
   int random_{0};
};

int main(int argc, char const *argv[]) {
  Base* base = new Derived;  // base指向子类对象
  base->print();
  base->PointerType();
  delete base;
  return 0;
}
```
上面的代码中
```
Base* base = new Derived;  // base指向子类对象
base->print();
```
base->print()会被编译器大致转换为：
```
(*base->vtpr[1])(base)
```
编译期确认：
虚表的索引1是在编译期就能确认。即在编译期就能确定调用虚函数`print`在`vtpr`的位置，从而取出`print`的地址。而这个索引是按按照声明的顺序确认的，并且首位索引0是指向type_info，print是第一个声明的，那么其索引就是1。
执行期确认：
真正在执行时才能确认base指向的是基类对象还是派生类对象，从而确认不同的虚表，取出不同的虚函数。
By the way, 第二个base代表的是this指针
由于编译器对类的成员函数进行name mangling操作后将成员函数转换为非成员函数，会在参数列表首位安插一个this指针。

# 为什么析构函数一般写成虚函数？
由于类的多态性，基类指针可以指向派生类的对象，如果删除该基类的指针，就会调用该指针指向的派生类析构函数，而派生类的析构函数又自动调用基类的析构函数，这样整个派生类的对象完全被释放。

如果析构函数不被声明成虚函数，则编译器实施静态绑定，在删除基类指针时，只会调用基类的析构函数而不调用派生类析构函数，这样就会造成派生类对象析构不完全，造成内存泄漏。

所以将析构函数声明为虚函数是十分必要的。在实现多态时，当用基类操作派生类，在析构时防止只析构基类而不析构派生类的状况发生，要将基类的析构函数声明为虚函数。

# static的用法和作用？
> https://www.techxiaofei.com/post/cpp/static/
> https://zhuanlan.zhihu.com/p/399580918

static 是 C/C++ 中很常用的修饰符，它被用来控制变量的存储方式和可见性。而 static 在 C 语言和 C++ 语言中各有多个用途。
## C语言中static的三种用途
### 静态局部变量
1. 该变量在全局/静态存储区分配内存（局部变量在栈区分配内存）
2. 该变量在函数第一次调用时被首次初始化，后续的函数调用将不再进行初始化（局部变量每次函数调用都会初始化）
3. 该变量一般在声明处初始化，如果没有显式初始化，则会被程序自动初始化为0。（局部变量不会自动初始化）
4. 始终驻留在全局存储区，直到程序结束。但作用域是局部作用域，也就是不能在函数外使用它。（局部变量在栈区，在函数结束后立即释放内存）
举例：单例模式
```
Singleton &Singleton::GetInstance() {
    static Singleton signal;
    return signal;
}
```

### 静态全局变量
1. 该变量不能被其他文件所用（全局变量可以），除非include该文件;
2. 其他文件可以使用同名变量，不会发生冲突（全局变量不可以）;

### 静态函数
1. 静态函数不能被其它文件所用
2. 其它文件中可以定义相同名字的函数，不会发生冲突;

## C++语言中static的用法

### 静态数据成员
1. 静态数据成员被当作是类的成员，由该类型的所有对象共享访问；
2. 静态数据成员在全局数据区分配内存，程序开始运行时被分配空间，到程序结束之后才释放；
3. 静态数据成员必须初始化，而且只能在类体外进行；
4. 静态数据成员既可以通过对象名引用，也可以通过类名引用

### 静态成员函数
1. 静态成员函数不能访问非静态(包括成员函数和数据成员)，但是非静态可以访问静态。
2. 静态成员函数和静态数据成员一样，他们都属于类的静态成员，而不是对象成员。
3. 非静态成员函数在编译器的name mangling操作后会安插一个`this` 指针，而静态成员函数没有`this` 指针。

# 友元函数和友元类的基本情况
友元提供了不同类的成员函数之间、类的成员函数和一般函数之间进行数据共享的机制。通过友元，一个不同函数或者另一个类中的成员函数可以访问类中的私有成员和保护成员。友元的正确使用能提高程序的运行效率，但同时也破坏了类的封装性和数据的隐藏性，导致程序可维护性变差。

# 模版类的作用可以介绍下吗？
模板类的基本思想是，通过使用模板参数，将类型参数化。这样，就可以编写一个类或函数，该类或函数可以处理多种数据类型，而不仅仅是特定类型的对象。这使得代码更加通用和可重用，并且减少了代码的重复性。

模板类的主要优点包括：
+ 类型无关性：模板类是类型无关的，这意味着它可以处理任何数据类型。这使得模板类具有高度的可复用性，可以在不同的场景下使用。
+ 编译时类型检查：模板类的类型检查在编译时进行。这意味着在运行程序之前，可以发现并修复类型错误，从而提高程序的可靠性和安全性。
+ 提高代码复用性：通过使用模板类，可以将通用代码复用在不同的数据类型上，减少了代码的重复性。
+ 提高编程效率：使用模板类可以减少开发时间，因为程序员只需要编写一次代码，就可以在多种数据类型上使用。

# 基类的虚函数表存放在内存的什么区，虚表指针vptr的初始化时间
首先整理一下虚函数表的特征：

+ 虚函数表是全局共享的元素，即全局仅有一个，在编译时就构造完成
+ 虚函数表类似一个数组，类对象中存储vptr指针，指向虚函数表，即虚函数表不是函数，不是程序代码，不可能存储在代码段
+ 虚函数表存储虚函数的地址,即虚函数表的元素是指向类成员函数的指针,而类中虚函数的个数在编译时期可以确定，即虚函数表的大小可以确定,即大小是在编译时期确定的，不必动态分配内存空间存储虚函数表，所以不在堆中

根据以上特征，虚函数表类似于类中静态成员变量.静态成员变量也是全局共享，大小确定，因此最有可能存在全局数据区，测试结果显示：

虚函数表vtable在Linux/Unix中存放在可执行文件的只读数据段中(rodata)，这与微软的编译器将虚函数表存放在常量段存在一些差别

由于虚表指针vptr跟虚函数密不可分，对于有虚函数或者继承于拥有虚函数的基类，对该类进行实例化时，在构造函数执行时会对虚表指针进行初始化，并且存在对象内存布局的最前面。

一般分为五个区域：栈区、堆区、函数区（存放函数体等二进制代码）、全局静态区、常量区

C++中虚函数表位于只读数据段（.rodata），也就是C++内存模型中的常量区；而虚函数则位于代码段（.text），也就是C++内存模型中的代码区。

# push_back和emplace_back的区别？
> https://zhuanlan.zhihu.com/p/213853588

> https://zhuanlan.zhihu.com/p/183861524

+ 性能（开销更小）：
  + push_back会先调用构造函数来创建一个临时对象，再通过拷贝构造函数临时对象将这个临时对象放入容器中，最后销毁临时对象。
  + emplace_back仅会在容器中就地构造一个新对象减少临时对象拷贝、销毁的步骤，所以性能更高
+ 插入类的构造函数有多个参数时，push_back只能插入对象，emplace_back可以仅插入参数

## emplace_back/push_back源码剖析
> https://wenfh2020.com/2023/08/01/cpp-emplace-back/

> https://zhuanlan.zhihu.com/p/260508149

> https://blog.csdn.net/LIJIWEI0611/article/details/122014506

[emplace_back源码](https://gcc.gnu.org/onlinedocs/gcc-5.4.0/libstdc++/api/a01685_source.html#l00087) 
+ emplace_back的入参__args是**万能引用** ，入参__args**完美转发**给内部的::new进行对象的创建和就地构造，并将其追加到数组对应的位置。
```
/* /usr/include/c++/4.8.2/debug/vector */
template <typename _Tp, typename _Allocator = std::allocator<_Tp> >
class vector : public _GLIBCXX_STD_C::vector<_Tp, _Allocator>,
               public __gnu_debug::_Safe_sequence<vector<_Tp, _Allocator> > {
    ...
    // emplace_back 参数是万能引用。
    template <typename... _Args>
    void emplace_back(_Args&&... __args) {
        ...
        // 完美转发传递参数。
        _Base::emplace_back(std::forward<_Args>(__args)...);
        ...
    }
#endif
    ...
};
```
+ allocator的construct进一步将参数完美转发给::new
```
/* /usr/include/c++/4.8.2/bits/vector.tcc */
#if __cplusplus >= 201103L
template <typename _Tp, typename _Alloc>
template <typename... _Args>
void vector<_Tp, _Alloc>::emplace_back(_Args&&... __args) {
    if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage) {
        _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
                                 std::forward<_Args>(__args)...);
        ++this->_M_impl._M_finish;
    } else {
        _M_emplace_back_aux(std::forward<_Args>(__args)...);
    }
}
#endif

template <typename _Tp, typename... _Args>
static auto construct(_Alloc& __a, _Tp* __p, _Args&&... __args)
    -> decltype(_S_construct(__a, __p, std::forward<_Args>(__args)...)) {
    _S_construct(__a, __p, std::forward<_Args>(__args)...);
}

/* /usr/include/c++/4.8.2/ext/new_allocator.h */
template <typename _Tp>
class new_allocator {
#if __cplusplus >= 201103L
    template <typename _Up, typename... _Args>
    void construct(_Up* __p, _Args&&... __args) {
        // 新建构造对象，并通过完美转发给对象传递对应的参数。
        ::new ((void*)__p) _Up(std::forward<_Args>(__args)...);
    }
#endif
};
```
比如 `datas.emplace_back("ee")` 插入的参数是字符串常量引用（右值引用），它插入对象元素，并没有触发拷贝构造和移动构造。因为 emplace_back 接口传递的是字符串常量引用，而真正的对象创建和构造是在 std::vector 内部实现的：`::new ((void*)__p) _Up(std::forward<_Args>(__args)...)` ;，相当于 `new Data("ee")` ，在插入对象元素的整个过程中，并未产生须要拷贝和移动的 临时对象。

[push_back源码](https://link.zhihu.com/?target=https%3A//gcc.gnu.org/onlinedocs/gcc-5.4.0/libstdc%2B%2B/api/a01609_source.html%23l00913) 
```
void push_back(const value_type &__x) {
    if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage) {
        // 首先判断容器满没满，如果没满那么就构造新的元素，然后插入新的元素
        _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
                                 __x);
        ++this->_M_impl._M_finish; // 更新当前容器内元素数量
    } else
        // 如果满了，那么就重新申请空间，然后拷贝数据，接着插入新数据 __x
        _M_realloc_insert(end(), __x);
}

// 如果 C++ 版本为 C++11 及以上（也就是从 C++11 开始新加了这个方法），使用 emplace_back() 代替
#if __cplusplus >= 201103L
void push_back(value_type &&__x) {
    emplace_back(std::move(__x));
}
#endif
```

emplace_back() 和 push_back() 中区别最大的程序拎出来看：
```
_Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
                                 std::forward<_Args>(__args)...); // emplace_back()
_Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
                                 __x);                            // push_back()
```
参考https://zhuanlan.zhihu.com/p/183861524
+ 插入构造参数时
```
std::vector<A> a;
a.emplace_back(1);  
emplace_back会将参数1完美转发给construct，construct再完美转发给new，即new (finish) A(1)，仅调用A的构造函数

a.push_back(2);
push_back：
  1) 调用 有参构造函数 A (int x_arg) 创建临时对象；
  2）调用 移动构造函数 A (A &&rhs)   到vector中；
  3) 调用     析构函数               销毁临时对象；
```

+ 插入临时对象时，两者一样会调用移动构造函数
+ 插入对象实例时，两者一样会调用拷贝构造函数

# 为什么不能把所有的函数写成内联函数?/内联函数的缺点？
> https://www.cnblogs.com/yinbiao/p/11606554.html

> https://zhuanlan.zhihu.com/p/152055532

内联函数不是在调用时发生控制转移，而是在编译时将函数体嵌入每一个调用处，编译时，类似宏替换，使用函数体替换调用处的函数名，在代码中用inline关键字修饰，在类中声明的成员函数，自动转换为内联函数。
当编译器处理调用内联函数的语句时，不会将该语句编译成函数调用的指令，而是直接将整个函数体的代码插人调用语句处，就像整个函数体在调用处被重写了一遍一样，在执行时是顺序执行，而不会进行跳转。

## 内联函数的优点
1. 有参数类型检测，更加安全
2. 内联函数是在程序运行时展开，而且是进行参数传递
3. inline关键字只是对编译器的一个定义，如果函数本地不符合内联函数的标准，编译器就会将这个函数当作是普通函数

## 内联函数的缺点
因为内联函数是在调用处展开，所以会使代码边长，占用更多内存

## 内联函数和宏的区别
1. 宏是由预处理器对宏进行替换的，而内联函数是通过编译器控制实现的。
2. 内联函数在运行时可调试，而宏定义不可以
3. 编译器会对内联函数的参数类型做安全检查或自动类型转换，而宏定义则不会
4. 内联函数可以访问类的成员变量，而宏定义则不能

# 什么是纯虚函数，与虚函数的区别
虚函数是指在继承关系中的函数，可以用"virtual"关键字来声明。当调用此类函数时，编译器会根据调用对象的实际类型，而不是根据声明类型来调用适当的函数。该函数通常被称为“多态函数”。

纯虚函数则是在函数声明时，使用“virtual”和“= 0”关键字来声明的。纯虚函数是一个空函数，只提供函数声明，而不提供实际实现。由于编译器不能确定如何实现，因此需要在子类中实现它。纯虚函数也被称为抽象函数。

虚函数和纯虚函数的区别主要表现在以下几个方面：
声明方式：虚函数可以在类中声明，也可以在类的外部声明，编译器会自动将它们转换为虚函数；但是纯虚函数只能在类中声明，而不能在类的外部声明。
实现方式：虚函数可以有实现，也可以没有实现；而纯虚函数没有实现，不可以有实现。
覆盖方式：虚函数可以在子类中覆盖，也可以不被覆盖；而纯虚函数必须在子类中覆盖，否则编译器将报错。
调用方式：虚函数可以被多态调用，也可以被静态调用；而纯虚函数只可以被多态调用，不可以被静态调用。
用途：虚函数可以用来实现多态，可以根据调用对象的实际类型，而不是根据声明类型来调用适当的函数。这样可以有效地实现代码的重用，避免了重复编码。而纯虚函数可以用来实现抽象类，一个抽象类是指一个类中定义了至少一个纯虚函数的类。这样可以定义一个抽象的接口层，子类可以通过实现纯虚函数来实现抽象接口的不同功能。

# C++11中的auto是怎么实现识别自动类型的？模板是怎么实现转化成不同类型的？

## auto原理
> https://www.jianshu.com/p/aaf789b64667
> 
> https://www.zhihu.com/question/294048058/answer/489015726

auto使用的是模板实参推断（Template Argument Deduction）的机制。auto被一个虚构的模板类型参数T替代，然后进行推断，即相当于把变量设为一个函数参数，将其传递给模板并推断为实参，auto相当于利用了其中进行的实参推断，承担了模板参数T的作用。
```
int tmp = 1;
auto x = tmp
```
相当于
```
template<class T>
void deduce (T x);
deduce(tmp);
```

## 模版
模板是C++支持参数化多态（静态多态）的工具，使用模板可以使用户为类或者函数声明一种一般模式，使得类中的某些数据成员或者成员函数的参数、返回值取得任意类型。

# 创建三个线程，依次打印1到100
> https://developer.aliyun.com/article/1273406
> 
> https://zhuanlan.zhihu.com/p/512969481

## 使用互斥锁
```
#include <iostream>
#include <atomic>
#include <thread>  
#include <mutex>

using namespace std;
static int counter = 0;
static mutex mtx;

void print_num(int thread_id) {
    while (counter < 10) {
        //unique_lock<mutex> ul(mtx);
        mtx.lock();
        if (counter % 3 == thread_id) {
            cout << "counter: " << counter << " thread_id: " << thread_id << endl;
            counter++;
        }
        mtx.unlock();
    }
}

int main() {
    std::thread t1(print_num, 0);
    std::thread t2(print_num, 1);
    std::thread t3(print_num, 2);
    t1.join();
    t2.join();
    t3.join();
    return 0;
}
```

## 使用信号量
```
#include <iostream>
#include <atomic>
#include <thread>  
#include <mutex>
#include <semaphore.h> 

using namespace std;
static int counter = 0;
//static mutex mtx;
static sem_t semaphore; // 定义信号量  

void print_num(int thread_id) {
    while (counter < 10) {
        sem_wait(&semaphore); // 信号量-1
        if (counter % 3 == thread_id) {
            cout << "counter: " << counter << " thread_id: " << thread_id << endl;
            counter++;
        }
        sem_post(&semaphore); // 信号量+1
    }
}

int main() {
    std::thread t1(print_num, 0);
    std::thread t2(print_num, 1);
    std::thread t3(print_num, 2);
    t1.join();
    t2.join();
    t3.join();
    return 0;
}
```
