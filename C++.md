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

## shared_ptr的构造函数与make_shared有啥区别
> https://www.cnblogs.com/chaohacker/p/14802112.html



## shared_ptr实现
> https://zhuanlan.zhihu.com/p/64543967


SharedPtr 类维护一个指向对象的指针 ptr 和一个指向引用计数器的指针 ref_count。在构造函数、拷贝构造函数和赋值操作符中，我们适当地增加或减少引用计数器的值。当引用计数器的值减少到 0 时，我们删除对象并删除引用计数器

```
#include <iostream>  
#include <memory> // For std::allocator  
#include <atomic> // For std::atomic  
  
template <typename T>  
class SharedPtr {  
private:  
    T* ptr;  
    std::atomic<long>* ref_count;  
  
public:  
    SharedPtr(T* p = nullptr) : ptr(p), ref_count(new std::atomic<long>(1)) {  
        if (p) {  
            std::cout << "SharedPtr constructor: Allocating new object " << p << std::endl;  
        }  
    }  
  
    SharedPtr(const SharedPtr& other) : ptr(other.ptr), ref_count(other.ref_count) {  
        if (ref_count) {  
            ++(*ref_count);  
            std::cout << "SharedPtr copy constructor: Incremented refcount to " << *ref_count << std::endl;  
        }  
    }  
  
    SharedPtr& operator=(const SharedPtr& other) {  
        if (this != &other) {  
            if (ref_count && --(*ref_count) == 0) {  
                delete ptr;  
                delete ref_count;  
                std::cout << "SharedPtr destructor: Deleting object " << ptr << " and refcount" << std::endl;  
            }  
  
            ptr = other.ptr;  
            ref_count = other.ref_count;  
            if (ref_count) {  
                ++(*ref_count);  
                std::cout << "SharedPtr assignment operator: Incremented refcount to " << *ref_count << std::endl;  
            }  
        }  
        return *this;  
    }  
  
    ~SharedPtr() {  
        if (ref_count && --(*ref_count) == 0) {  
            delete ptr;  
            delete ref_count;  
            std::cout << "SharedPtr destructor: Deleting object " << ptr << " and refcount" << std::endl;  
        }  
    }  
  
    T& operator*() const {  
        return *ptr;  
    }  
  
    T* operator->() const {  
        return ptr;  
    }  
  
    bool operator==(const SharedPtr& other) const {  
        return ptr == other.ptr;  
    }  
  
    bool operator!=(const SharedPtr& other) const {  
        return ptr != other.ptr;  
    }  
  
    long use_count() const {  
        return ref_count ? *ref_count : 0;  
    }  
};  
  
int main() {  
    SharedPtr<int> p1(new int(5));  
    SharedPtr<int> p2 = p1;  
    SharedPtr<int> p3;  
    p3 = p1;  
  
    std::cout << "p1 use_count: " << p1.use_count() << std::endl;  
    std::cout << "p2 use_count: " << p2.use_count() << std::endl;  
    std::cout << "p3 use_count: " << p3.use_count() << std::endl;  
  
    return 0;  
}
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
> 
> https://zhuanlan.zhihu.com/p/359466948
> 
> https://zhuanlan.zhihu.com/p/596288935
>


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

1. 声明方式：虚函数可以在类中声明，也可以在类的外部声明，编译器会自动将它们转换为虚函数；但是纯虚函数只能在类中声明，而不能在类的外部声明。
   
2. 实现方式：虚函数可以有实现，也可以没有实现；而纯虚函数没有实现，不可以有实现。
   
3. 覆盖方式：虚函数可以在子类中覆盖，也可以不被覆盖；而纯虚函数必须在子类中覆盖，否则编译器将报错。

4. 调用方式：虚函数可以被多态调用，也可以被静态调用；而纯虚函数只可以被多态调用，不可以被静态调用。

5. 用途：虚函数可以用来实现多态，可以根据调用对象的实际类型，而不是根据声明类型来调用适当的函数。这样可以有效地实现代码的重用，避免了重复编码。而纯虚函数可以用来实现抽象类，一个抽象类是指一个类中定义了至少一个纯虚函数的类。这样可以定义一个抽象的接口层，子类可以通过实现纯虚函数来实现抽象接口的不同功能。

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

# 指针和引用的区别？
> https://zhuanlan.zhihu.com/p/140966943

1. 基本概念：

指针：是一个变量，其值为另一个变量的地址。它有一个类型，这个类型决定了指针所指向的数据的类型。
引用：是一个已经存在的变量的别名，或者说是一个已经存在的变量的另一个名字。它与一个特定的变量关联，一旦一个引用被初始化，它就不能再被改变。
2. 初始化：

指针：需要使用new或nullptr进行初始化。例如：int* ptr = new int(10); 或 int* ptr = nullptr;
引用：必须在使用之前被初始化，并且一旦初始化后就不能改变。例如：int a = 10; int& ref = a;
3. 间接性：

指针：需要解引用，可以通过指针间接地访问和修改变量的值。例如：*ptr = 20;
引用：不需要解引用，是直接的，它并没有提供一种间接访问的机制。一旦一个变量被引用，就可以直接通过引用来修改变量的值。例如：ref = 20;
4. 空指针与未初始化的引用：

指针：空指针是一个合法的概念，表示这个指针不指向任何的内存地址。未初始化的指针则可能会导致未定义的行为。
引用：引用必须在声明时被初始化，而且一旦初始化后就不能改变。所以不存在未初始化的引用这个概念。
5. 使用场景：

指针：通常用于动态内存分配，如数组、链表等数据结构。
引用：通常用于函数参数传递、函数返回值等场景，以避免数据的复制，提高效率。
6. 安全性：

指针：使用不当可能会导致内存泄漏、野指针等问题，需要程序员小心处理。
引用：相对更安全一些，有类型安全检查，一旦一个变量被引用，就不能再被其他变量引用，且必须在使用前初始化。
7. 指向自身的指针和引用：

指针：可以指向自身，如一个结构体或类的成员函数可以返回一个指向自身的指针。
引用：不能直接引用自身，因为一个引用必须在声明时就被初始化，而且一旦初始化后就不能改变。但可以通过引用的方式间接地实现类似的效果。
8. 运算支持：
8.1 sizeof：
指针：sizeof指针得到的是本指针的大小，32位的机器就是4个字节。
引用：sizeof引用是引用所指向变量的大小。

8.2 ++：
指针：指针++，表示指向下一个对象的内存。比如：int类型，原指针int* p=0x00,p++后为0x04
引用：引用++，表示对引用对象++，比如：int类型，int& p = a; p++; 对a的数值+1

9. 多级：
指针：指针可以有多级，int**p = a;
引用：引用只能是一级, int&&p = a; 更具体的是说 左值引用只能是一级

# 指针和const的用法

以下是关于C++指针和const的一些基本用法：

1. 指向常量的指针（Pointer to Constant）
这种指针不能通过它来修改它所指向的数据，但指针本身可以改变，指向其他数据。
```
const int value = 10;  
const int* ptrToConst = &value; // 正确：指向常量的指针  
// *ptrToConst = 20; // 错误：不能通过指向常量的指针修改数据  
ptrToConst = nullptr; // 正确：可以改变指针本身的值
```
2. 常量指针（Constant Pointer）
这种指针的指向一旦确定后就不能改变，但是可以通过它来修改它所指向的数据（前提是数据本身不是const）。

```
int value = 10;  
int* const constPtr = &value; // 正确：常量指针  
// constPtr = nullptr; // 错误：常量指针不能改变指向  
*constPtr = 20; // 正确：可以通过常量指针修改数据
```
3. 指向常量的常量指针（Constant Pointer to Constant）
这种指针既不能改变它所指向的数据，也不能改变它自己的指向。

```
const int value = 10;  
const int* const constPtrToConst = &value; // 正确：指向常量的常量指针  
// *constPtrToConst = 20; // 错误：不能通过指向常量的常量指针修改数据  
// constPtrToConst = nullptr; // 错误：常量指针不能改变指向
```
4. 函数参数中的const指针
在函数参数中使用const指针可以防止函数内部意外修改指针指向的数据。

```
void func(const int* ptr) {  
    // *ptr = 20; // 错误：不能通过const指针修改数据  
}  
  
int main() {  
    int x = 10;  
    func(&x); // 调用函数，传递x的地址  
    return 0;  
}
```
5. 函数返回值中的const指针
当函数返回一个指针时，可以使用const来确保调用者不会通过这个指针修改返回的数据。

```
const int* func() {  
    static int value = 10; // 使用static确保在函数返回后数据仍然存在  
    return &value;  
}  
  
int main() {  
    const int* ptr = func();  
    // *ptr = 20; // 错误：不能通过返回的const指针修改数据  
    return 0;  
}
```
6. const和引用
const也可以与引用一起使用，用于保证引用的数据不会被修改。

```
void func(const int& ref) {  
    // ref = 20; // 错误：不能通过const引用修改数据  
}  
  
int main() {  
    int x = 10;  
    func(x); // 调用函数，传递x的引用  
    return 0;  
}
```
在C++中，正确地使用const和指针可以提高代码的安全性和可读性，同时也有助于避免潜在的错误。


# Python和C++有什么区别？
Python和C++是两种非常不同的编程语言，它们在语法、设计理念、应用领域等方面都有很大的差异。

语法: Python的语法更简洁、易读，对缩进要求严格，而C++的语法则更为复杂，需要更多的代码来实现相同的功能。
设计理念: Python的设计理念是“明确优于隐晦”，强调代码的可读性，而C++则更注重性能和底层操作。
应用领域: Python常用于数据分析、机器学习、Web开发等领域，而C++则广泛应用于系统软件、游戏、图形界面开发等领域。
性能: C++通常比Python运行得更快，因为C++是一种编译型语言，而Python是一种解释型语言。但是，通过使用像NumPy这样的库，Python在许多科学计算任务上的性能已经接近甚至超过了C++。
内存管理: Python有自动的垃圾收集和内存管理机制，而C++需要程序员手动管理内存，这要求更高的编程技能。
库和框架: Python有大量的第三方库和框架，如NumPy、Pandas、Django等，使得在各种领域都能快速开发应用。而C++也有很多库，如Boost、STL等，但相对来说学习曲线更陡峭。
学习曲线: Python对于初学者来说更友好，因为它的语法更简单，有大量的教程和社区支持。而C++则需要对编程有较深的理解才能有效地使用。

# C++与Java的区别

C++和Java在许多方面都存在显著的区别。以下是一些关键的差异：

编译与执行：C++是一种编译型语言，其源代码需要经过编译和链接后生成可执行的二进制文件。相对地，Java是一种解释型语言，其源代码首先被编译成字节码，然后由Java虚拟机（JVM）解释执行。这导致Java的执行速度通常较慢，但Java的优势在于其跨平台性，可以在任何安装了JVM的平台上运行。
面向对象：Java是一种纯面向对象的语言，所有的代码（包括函数和变量）必须在类中实现。此外，Java支持封装、继承和多态等面向对象特性。C++则兼具面向过程和面向对象编程的特点，可以定义全局变量和全局函数。
内存管理：Java提供了自动内存管理机制，所有对象都是通过new操作符在内存堆栈上创建的，并且有垃圾回收器自动回收无用的内存。C++则需要程序员手动管理内存的分配和释放，这增加了程序设计的工作量。
指针：指针是C++中的一大特点，也是其难点，但可能会引起系统问题。Java中没有指针的概念，这使得程序更为安全。
异常处理：Java有完善的异常处理机制，而C++也支持异常处理，但不如Java的异常处理系统强大和易用。
多重继承：C++支持多重继承，这可以带来很大的灵活性，但也可能导致复杂的继承关系和菱形问题。而Java只允许单一继承，但可以实现多个接口，以实现类似的功能。
预处理功能：Java不支持预处理功能，而C++在编译过程中有一个预处理阶段。
库和框架：C++和Java都有大量的第三方库和框架，但它们的应用领域和风格有所不同。例如，C++有大量的科学计算库，而Java在Web开发和企业级应用方面有很强的支持。

# 浅拷贝和深拷贝的区别

在C++中，浅拷贝（Shallow Copy）和深拷贝（Deep Copy）的区别主要在于对对象的复制方式。

浅拷贝是指创建一个新的对象，并将原对象的非静态成员变量进行复制。如果成员变量是值类型（如基本数据类型、结构体等），则进行直接的复制；如果成员变量是引用类型（如数组、字符串、类对象等），则复制引用地址，而不是实际的对象内容。这意味着新对象和原对象共享这些引用对象。因此，当修改新对象的非静态成员变量时，会同时影响原对象，反之亦然。

深拷贝则是指创建一个新的对象，并将原对象的所有成员变量，无论是值类型还是引用类型，都进行复制。对于值类型，直接复制值；对于引用类型，复制引用指向的对象，而不是引用本身。这样，新对象和原对象完全独立，没有任何关联。深拷贝可以递归地复制对象的所有成员变量，包括嵌套的对象或数组等复杂的数据结构。

在C++中，浅拷贝通常用于简单对象的复制，而对于复杂对象或需要完全独立副本的情况，应该使用深拷贝。需要注意的是，深拷贝通常需要更多的内存和时间开销，因此在性能和资源消耗方面需要考虑权衡。

```
#include <iostream>  
#include <string.h>
using namespace std;
 
class Student
{
private:
	int num;
	char *name;
public:
	Student(){
        name = new char(20);
		cout << "Student" << endl;
    };
	~Student(){
        cout << "~Student " << &name << endl;
        delete name;
        name = NULL;
    };
	Student(const Student &s){//拷贝构造函数
        //浅拷贝，当对象的name和传入对象的name指向相同的地址
        name = s.name;
        //深拷贝
        //name = new char(20);
        //memcpy(name, s.name, strlen(s.name));
        cout << "copy Student" << endl;
    };
};
 
int main()
{
	{// 花括号让s1和s2变成局部对象，方便测试
		Student s1;
		Student s2(s1);// 复制对象
	}
	system("pause");
	return 0;
}
//浅拷贝执行结果：
//Student
//copy Student
//~Student 0x7fffed0c3ec0
//~Student 0x7fffed0c3ed0
//*** Error in `/tmp/815453382/a.out': double free or corruption (fasttop): 0x0000000001c82c20 ***

//深拷贝执行结果：
//Student
//copy Student
//~Student 0x7fffebca9fb0
//~Student 0x7fffebca9fc0
```

# 如果同时有大量客户并发建立连接，服务器端有什么机制进行处理
两种方法：多线程同步阻塞和I/O多路复用socket的建立。
## 多线程同步阻塞
多线程同步阻塞是指在每个客户端连接到来时，都会创建一个新的线程来处理该连接，这样可以实现并发处理。
## I/O多路复用socket的建立
在一个线程中同时监听多个端口，当有新的连接请求到来时，该线程会将该连接请求分配给一个空闲的线程来处理。


# 引用是否能实现动态绑定，为什么可以实现？
可以。
引用在创建的时候必须初始化，在访问虚函数时，编译器会根据其所绑定的对象类型决定要调用哪个函数。注意只能调用虚函数。

# 知道C++中的组合吗？它与继承相比有什么优缺点吗？
组合（Composition）

组合是一种“has-a”关系，它允许一个类包含另一个类的对象作为其成员变量。这种关系实现了严格的封装，因为外部代码不能访问被包含对象的内部实现。

优点：

封装性：组合能更好地实现封装，因为它可以防止对内部对象的直接访问。
灵活性：组合允许你动态地改变对象的内部结构，因为你可以随时改变或添加成员变量。
减少耦合：组合减少了类之间的耦合，因为它允许你独立地修改和测试类。
缺点：

代码复杂性：使用组合可能会导致代码变得更复杂，因为你需要手动管理成员对象的生命周期和状态。
代码量增加：由于你需要创建和管理额外的对象，所以代码量可能会增加。
继承（Inheritance）

继承是一种“is-a”关系，它允许一个类（派生类）继承另一个类（基类）的属性和方法。继承允许你创建新的类，这些类可以重用现有类的代码。

优点：

代码重用：继承允许你重用基类的代码，从而减少代码量。
多态性：通过继承和虚函数，可以实现多态性，这是面向对象编程的四大基本特性之一。
易于理解：对于初学者来说，继承可能更容易理解，因为它与现实世界中的“子类”和“父类”概念相似。
缺点：

耦合性：继承增加了类之间的耦合，因为派生类依赖于基类的实现。如果基类发生更改，可能会影响派生类。
代码可维护性：由于继承的层次结构可能导致复杂的依赖关系，因此可能使代码更难维护和测试。
单一职责原则：过度使用继承可能会违反单一职责原则，因为一个类可能会继承多个不相关的功能。

# vector的扩容删除都是怎么做的？为什么是1.5或者是2倍？
vector是C++标准模板库中的一部分，类似于动态数组，当存储空间不足时可以自动扩容。其扩容和删除操作的具体方式如下：
vector内部有三个int*指针start、finish、end_of_storage，start和finish分别表示目前使用空间的头、尾，表示已经被使用的左闭右开[start, finish)范围。end_of_storage指向可用空间（包含备用空间）的尾部。
`push_back` 操作：
1. 首先判断finish是否等于end_of_storage，即是否还有备用空间，若有，则在finish位置构造元素，若没有了，进入步骤2；
2. 记录原来的大小为old_size，若原来的大小为0，则新的大小为1，否则为两倍的原来大小；
3. 调用内存分配器data_allocator的alloc分配新大小的空间
4. 拷贝原数组的值到新的内存空间，在新尾部new_finish构造元素
5. 调用析构函数destroy析构原来的对象
6. 调用deallocate释放原来的空间
`pop_back` 操作：
1. finish前移
2. 调用destroy析构掉finish位置的元素
`erase` 范围操作：
1. 调用copy将[last,finish)之间的元素拷贝到first处，返回new_finish
2. 调用destroy析构掉[new_finish, finish)中间的元素
3. 调整finish(finish - last + first)

`erase` 单个位置操作：
1. 调用copy将[pos + 1,finish)之间的元素拷贝到pos处
2. 调整finish - 1
3. 调用destroy析构掉finish位置的元素

扩容操作：当vector需要添加新元素但当前空间不足时，它会进行扩容。扩容策略通常是以原来的1.5倍或者2倍进行扩容，这样可以保证在平均情况下每次需要扩容的次数是最少的，从而减少了整体的内存开销。在扩容的同时，vector会重新分配内存，把原来的数据拷贝到新的内存空间中，并释放原来的内存空间，以避免内存浪费。

删除操作：vector中的元素可以通过多种方式删除，例如使用成员函数erase()或pop_back()等。这些操作会移除指定的元素并可能重新调整剩余元素的位置。

# Vector的resize和reserve有什么区别
> https://blog.csdn.net/JMW1407/article/details/108785448
1. resize(int n, T val = T()):
+ 当n小于当前容器的size时，则将将元素减少到前n个（调用析构函数析构多余的元素）；
+ 当n大于当前容器的size时，则在容器尾部finish追加元素，若val制定了，则追加val元素的拷贝，否则调用元素的默认构造函数；
+ 当n大于当前容器的capacity，内存会重新分配；

2. reserve(int n):
+ 当n大于当前容器的capacity，内存会重新分配，使capacity达到n；
+ 其他任何情况下内存都不会重新分配，且容器的capacity不变；
+ 不影响容器的size，也不会改变任何元素；

总结来说，以下几点：
1. vector的reserve增加了vector的capacity,但是它的size没有改变！而resize改变了vector的capacity同时也增加了它的size!
   
2. reserve是容器预留空间，但在空间内不真正创建元素对象，所以在没有添加新的对象之前，不能引用容器内的元素。加入新的元
素时，要调用push_back/insert函数。

3. rize是改变容器的大小，且会创建对象，因此，调用这个函数之后，就可以引用容器内的对象了，当加入新的元素时，用
operator[]操作符，或者用迭代器来引用元素对象。此时再调用push_back()函数，是加在这个新的空间后面的。

4. 两个函数的参数形式也有区别的，reserve函数只有一个参数，即需要预留的容器的空间；resize函数可以有两个参数，第一个参
数是容器新的大小，第二个参数是要加入容器中的新元素，如果这个参数被省略，那么就调用元素对象的默认构造函数。


# STL中unordered_map和map的区别和应用场景，set、unordered_set

>  https://zhuanlan.zhihu.com/p/439650164

### hashtable实现
`hashtable` 内部包含了3个函数指针hash（哈希函数），equals（判断key是否相同函数），get_key（从value获取key的方法），一个vector（所有的哈希桶所在），一个size_t 表示哈希表元素的数量，三个方法（函数或者仿函数）的大小是3。vector里有三个指针，大小是12。size_type大小是4。总大小为19，为了对齐系统会调整为20。
哈希表节点是链表节点的结构，包含了一个next指针和value
```
template <class Value>
struct _hashtable_node{
    _hashtable_node* next;
    Value val;
}
```

### 解决碰撞
解决哈希冲突的方法有：
1.线性探测

当使用hash function计算出位置后发现该位置已经没有空间时，会继续向下一一寻找可以放入元素的位置。如果到达尾部就再从头开始。

这个方法简单易懂，但也有一个很直观的问题：在很多空位都被占用后，就会出现一个新元素疯狂撞墙最后才插入成功的现象，增加了平均插入成本。

2.二次探测

二次探测说的是，既然一个一个的探测会出现问题，那么就以二次方的形式探测。原来是h+1,h+2...现在就变为h+（1的平方），h+（2的平方）...

很明显这样的探测法跳跃起来了，不会出现连续撞墙的现象，效果会好很多。但还是会出现在两个元素计算出的位置相同时，探测位置也相同而产生的效率浪费。

3.开链

开链是指在表格中单独维护一个list，让计算位置相同的元素形成一个链。这也是SGI STL中hash table的实现方法。

### 开链在hashtable中的运用
当元素的个数大于buckets的空间个数时，就会把buckets成2倍扩增，然后再调整为质数。这样重新计算后可以很好地把长链变短。
其实再STL中这些数是规定好的，我们可以看一下。
```
static const int _stl_num_primes=28;
static const unsigned long _stl_prime_list[_stl_num_primes]={
    53,97,193,389,769,1543,3079,6151,12289,24593,1572869,3145739,6291469...
}
```
比如第一次初始化时哈希表的buckets为53，当插入第54个元素会发生resize，将buckets增大为97

### hash function

我们都知道计算过程与是一个取模过程，但是也会有一些不同。比如遇到字符和字符串这种类型就不能进行单纯的计算了。

在STL包装了一个函数bkt_num()，在此函数中调用hash function。
```
size_type bkt_num_key(const key_type& key,size_t n)const
{
    return hash(key)%n;
}
```
针对char、int、long这些整数型别，hash functions 什么都没有做，就是原样返回。对于字符串，其偏特化哈希函数如下：
```
inline size_t _stl_hash_string(const char* s){
    unsigned long h=0;
    for(;*s;++s){
        h=5*h+*s;
        return size_t(h);
    }
}
```

### 插入操作insert与表格重整resize
插入操作分为不允许重复`insert_unique` 和允许重复值`insert_equal` 
具体来说，首先会调用`resize(num_elems + 1)`  判断是否需要重建表格，若需要，将表格大小扩大到下一个质数，并将所有节点重新哈希一遍，落到表格中。
再调用`insert_unique_noresize` 插入，用哈希函数计算出桶号，先看下桶内是否有重复元素，重复了就返回false（允许重复的话就直接插入），然后再安插到桶内链表头

STL中的unordered_map和map以及set和unordered_set都是关联容器，但它们之间的主要区别在于内部实现方式和性能特点。

unordered_map和unordered_set基于哈希表（Hash Table）实现，使用哈希函数将元素映射到桶（Bucket）中。这种实现方式使得元素的插入、删除和查找操作的平均时间复杂度为O(1)，但在最坏情况下可能达到O(n)。哈希表不保证元素的顺序，因此unordered_map和unordered_set中的元素顺序是随机的。由于不需要维护元素的顺序，unordered_map和unordered_set的存储空间开销相对较小。

相比之下，map和set基于红黑树（RB-Tree）实现，可以自动将元素按照键值排序。因此，元素的插入、删除和查找操作的时间复杂度均为O(log n)。由于需要维护元素的顺序，map和set的存储空间开销相对较大。同时，由于红黑树的特性，map和set中的元素是按照键值顺序排列的。

在应用场景上，unordered_map和unordered_set适用于那些对查找性能要求较高，而不太关心元素顺序的场合。例如，如果你需要在一堆数据中快速查找某个键对应的值，或者需要计算元素出现的频率，那么unordered_map和unordered_set是很好的选择。

而map和set则更适用于那些需要元素有序性的场合。例如，如果你需要根据键值的顺序对元素进行排序，或者需要利用元素的顺序进行某些操作，那么应该选择map和set。


# select/poll/epoll的区别
> https://xiaolincoding.com/os/8_network_system/selete_poll_epoll.html#%E6%9C%80%E5%9F%BA%E6%9C%AC%E7%9A%84-socket-%E6%A8%A1%E5%9E%8B

## select 的用法
```
fd_set readfds, tmpfds;
FD_ZERO(&readfds);  
FD_SET(STDIN_FILENO, &readfds); // 监视标准输入
while (1) {
tmpfds = readfds; // select前先拷贝，由于select会修改fds
retval = select(maxfd + 1, &tmpfds, NULL, NULL, &tv);
if (retval) {  
// select返回大于0，表示有文件描述符就绪  
if (FD_ISSET(STDIN_FILENO, &tmpfds)) {  
    // 标准输入可读，读取并处理输入  
    char buf[1024];  
    ssize_t n = read(STDIN_FILENO, buf, sizeof(buf) - 1);  
    if (n > 0) {  
	buf[n] = '\0'; // 添加字符串终止符  
	printf("Read from stdin: %s\n", buf);  
    }  
}
```
select的用法主要涉及到几个关键的参数和步骤。首先，你需要准备三个文件描述符集合：readfds、writefds和exceptfds，分别用于监视读事件、写事件和异常事件。然后，通过select()系统调用，将这三个集合以及超时时间作为参数传入。select()会阻塞进程，直到有文件描述符就绪或超时为止。当select()返回时，它会修改传入的文件描述符集合，以指示哪些文件描述符已就绪。你可以通过检查这些集合来确定哪些文件描述符可以进行读/写操作或出现了异常。

## select的缺点：

+ 文件描述符数量限制：select使用fd_set数据结构来存储文件描述符，这个数据结构的大小是固定的。在一些系统上，这个大小被限制为FD_SETSIZE（通常是1024），这意味着select最多只能监视1024个文件描述符。当需要监视的文件描述符数量超过这个限制时，select就无法满足需求。

+ 效率问题：每次调用select都需要将所有的文件描述符从用户空间复制到内核空间，这可能会导致性能问题。特别是当需要监视大量的文件描述符时，这种复制操作会带来显著的性能开销。

+ 不可扩展性：当需要监视的文件描述符数量增加时，select的性能可能会下降。这是因为它需要遍历整个fd_set来检查哪个文件描述符已就绪，这种遍历操作在文件描述符数量很多时是非常低效的。

+ 不支持超时精度：select的超时参数是一个struct timeval结构，它的精度只能到微秒级别。对于某些需要更高精度时间控制的应用来说，这可能不够精确。

+ 事件驱动的局限性：select是一个事件驱动的机制，但它只能监视读、写和错误三种类型的事件。对于更复杂的事件处理需求，select可能无法满足。

## poll的用法

```
struct pollfd fds[2]; // 定义要监视的文件描述符数组  
int numfds;  
int timeout = -1; // -1表示无限等待  
int ret;
// 设置第一个文件描述符，监视标准输入  
fds[0].fd = STDIN_FILENO;  
fds[0].events = POLLIN; // POLLIN表示读事件  
fds[0].revents = 0;
// 调用poll函数  
ret = poll(fds, 1, timeout);
if (ret > 0) {  
// 如果有文件描述符就绪  
for (numfds = 0; numfds < 1; ++numfds) {  
    if (fds[numfds].revents & POLLIN) {  
	if (fds[numfds].fd == STDIN_FILENO) {  
	    // 处理标准输入  
	    char buf[1024];  
	    ssize_t n = read(STDIN_FILENO, buf, sizeof(buf) - 1);  
	    if (n > 0) {  
		buf[n] = '\0';  
		printf("Read from stdin: %s\n", buf);  
	    }  
```

## poll的缺点

+ 效率问题：与select类似，poll在每次调用时都需要遍历整个文件描述符集合。当监视大量文件描述符时，这种遍历操作可能导致效率下降。

+ 没有文件描述符数量限制的直接提升：虽然poll使用链表来存储文件描述符，从而在理论上没有像select那样的文件描述符数量限制（通常为1024），但实际的限制还是取决于系统资源。当文件描述符数量非常大时，性能仍然可能受到影响。

+ 水平触发模式：poll和select都使用水平触发模式（level-triggered），这意味着只要条件满足（例如，数据可读），就会一直通知应用程序，即使应用程序还没有处理完上一次的通知。这可能导致应用程序需要不断处理这些事件，即使它们暂时无法处理更多数据。

+ 没有更好的事件通知机制：poll仅仅是select的一个改进版本，并没有提供比select更高级的事件通知机制。在需要更高效和灵活的事件通知时，可能需要使用更现代的机制，如epoll（在Linux中）或kqueue

## epoll的用法

```
int epollfd, nfds, n;  
struct epoll_event ev, events[MAX_EVENTS];  
int stdinfd = STDIN_FILENO;
epollfd = epoll_create1(0);  // 创建epoll实例，保存文件描述符的空间
ev.events = EPOLLIN; // 监视读事件  
ev.data.fd = stdinfd; // 设置与该事件关联的文件描述符  
epoll_ctl(epollfd, EPOLL_CTL_ADD, stdinfd, &ev); // 在epoll实例中注册该描述符，为了监视可读事件
// 等待事件发生  
for (;;) {  
nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
 // 遍历所有触发的事件  
for (n = 0; n < nfds; ++n) {  
    if (events[n].data.fd == stdinfd) {  
	// 处理标准输入  
	char buf[1024];  
	ssize_t count = read(stdinfd, buf, sizeof(buf) - 1);  
	if (count > 0) {  
	    buf[count] = '\0';  
	    printf("Read from stdin: %s\n", buf);  
	}  
    }
```

## epoll的缺点
+ 平台依赖性：epoll是Linux特有的机制，因此它不具有跨平台的兼容性。如果你的应用程序需要在非Linux系统上运行，那么你将无法使用epoll。

+ 学习曲线：相对于select和poll，epoll的API和概念可能更为复杂，需要一定的时间来学习和理解。

+ 边缘触发模式的问题：虽然epoll可以设置为边缘触发模式（EPOLLET），但这也意味着它需要更多的应用程序逻辑来正确处理事件，尤其是当多个事件同时到达时。

+ 资源消耗：虽然epoll在处理大量文件描述符时比select和poll更高效，但它仍然需要一定的系统资源来维护其内部数据结构。在高负载情况下，这些资源的消耗可能成为一个问题。

## select和poll
select 实现多路复用的方式是，将已连接的 Socket 都放到一个文件描述符集合，然后调用 select 函数将文件描述符集合拷贝到内核里，让内核来检查是否有网络事件产生，检查的方式很粗暴，就是通过遍历文件描述符集合的方式，当检查到有事件产生后，将此 Socket 标记为可读或可写， 接着再把整个文件描述符集合拷贝回用户态里，然后用户态还需要再通过遍历的方法找到可读或可写的 Socket，然后再对其处理。

所以，对于 select 这种方式，需要进行 2 次「遍历」文件描述符集合，一次是在内核态里，一个次是在用户态里 ，而且还会发生 2 次「拷贝」文件描述符集合，先从用户空间传入内核空间，由内核修改后，再传出到用户空间中。

select 使用固定长度的 BitsMap，表示文件描述符集合，而且所支持的文件描述符的个数是有限制的，在 Linux 系统中，由内核中的 FD_SETSIZE 限制， 默认最大值为 1024，只能监听 0~1023 的文件描述符。

poll 不再用 BitsMap 来存储所关注的文件描述符，取而代之用动态数组，以链表形式来组织，突破了 select 的文件描述符个数限制，当然还会受到系统文件描述符限制。

但是 poll 和 select 并没有太大的本质区别，都是使用「线性结构」存储进程关注的 Socket 集合，因此都需要遍历文件描述符集合来找到可读或可写的 Socket，时间复杂度为 O(n)，而且也需要在用户态与内核态之间拷贝文件描述符集合，这种方式随着并发数上来，性能的损耗会呈指数级增长。

## epoll
epoll 通过两个方面，很好解决了 select/poll 的问题。

第一点，epoll 在内核里使用红黑树来跟踪进程所有待检测的文件描述字，把需要监控的 socket 通过 epoll_ctl() 函数加入内核中的红黑树里，红黑树是个高效的数据结构，增删改一般时间复杂度是 O(logn)。而 select/poll 内核里没有类似 epoll 红黑树这种保存所有待检测的 socket 的数据结构，所以 select/poll 每次操作时都传入整个 socket 集合给内核，而 epoll 因为在内核维护了红黑树，可以保存所有待检测的 socket ，所以只需要传入一个待检测的 socket，减少了内核和用户空间大量的数据拷贝和内存分配。

第二点， epoll 使用事件驱动的机制，内核里维护了一个链表来记录就绪事件，当某个 socket 有事件发生时，通过回调函数内核会将其加入到这个就绪事件列表中，当用户调用 epoll_wait() 函数时，只会返回有事件发生的文件描述符的个数，不需要像 select/poll 那样轮询扫描整个 socket 集合，大大提高了检测的效率。

# new操作底层实现
> https://blog.csdn.net/weixin_43354152/article/details/115523575

new操作实际上相当于执行了如下三步操作：
1. 调用operator new分配内存，大小为Foo对象大小
2. 通过placement new调用Foo构造函数，初始化对象
3. 返回指向该对象的指针

## operator new

1. 只分配所要求的空间，不调用相关对象的构造函数。当无法满足所要求分配的空间时，
->如果有new_handler,则调用new_handler,否则
->如果没要求不抛出异常（以nothrow参数表达），则执行bad alloc异常，否则
>返回0
2. 可以被重载，并且一般只在类中重载
3. 重载时，返回类型必须声明为void*
5. 重载时，第一个参数类型必须为表达要求分配空间的大小（字节），类型为size_t
6. 重载时，可以带其它参数


# 程序编译过程

编译过程是将用高级编程语言（如C++）编写的源代码转换为计算机能够执行的机器语言代码的过程。对于C++程序，编译过程通常包括以下几个主要步骤：

1. 预编译（预处理）：
预处理器（例如GCC中的cpp）读取源代码文件，处理其中的预处理指令，如#include、#define等。
它会将头文件（header files）的内容包含到源文件中，替换宏定义（macro definitions）等。
经过预处理后，会生成一个新的文件，通常保留了原始文件的扩展名或添加了一个新的扩展名（如.i）。

2. 编译：
编译器（如GCC中的cc1）读取预处理后的文件，将其转换为汇编语言代码。
在这一过程中，编译器会进行词法分析、语法分析、语义分析等操作，以确保代码的正确性。
如果代码中存在语法错误或类型错误，编译器会生成错误信息并停止编译。
编译成功后，会生成一个或多个汇编语言文件（通常是.s文件）。

3. 汇编：
汇编器（如as）将编译器输出的汇编语言代码转换为目标文件（object file），即机器码的一种中间形式。
目标文件通常具有.o的扩展名，并包含了程序的各种段（如代码段、数据段等）。
汇编过程不会检查代码的逻辑错误或运行时错误，只关注语法和结构。

4. 链接：
链接器（如ld）负责将多个目标文件（可能是来自不同源文件编译得到的）以及必要的库文件（如标准库、其他第三方库等）组合成一个可执行文件。
在链接过程中，链接器会解决符号引用问题（例如函数和变量的地址），并生成最终的可执行文件（在Windows上通常是.exe文件）。
如果在链接阶段出现未解析的符号引用或库文件缺失等问题，链接器会生成错误信息。
经过以上步骤后，就得到了一个可执行的程序文件。用户可以通过运行这个文件来执行程序中定义的各项任务。

# 静态库和动态库的区别

静态库： 在C++中，静态库（Static Library）是一种包含一组预编译目标文件的库。它们在编译时被链接到应用程序中，以提供代码复用和模块化的好处。静态库文件通常以“.lib”（Windows）或“.a”（Unix/Linux/MacOS）扩展名为后缀。静态库是编译时链接的，也就是说，当你编译应用程序时，所有的静态库都会被复制到最终生成的可执行文件中，这意味着在运行时，不需要再加载这些库。静态库将所有的代码都打包到一个库文件中，因此静态库的文件大小较大，可能会占用较多的磁盘空间。如果静态库中的代码需要更新，那么需要重新编译整个应用程序。
动态库：
在C++中，动态库（Dynamic Library）是一种在运行时被加载的共享库，它在编译时被编译成目标文件，并在运行时通过动态链接器（Dynamic Linker）加载到内存中。动态库通常以“.dll”（Windows）或“.so”（Unix/Linux/MacOS）扩展名为后缀。内存节省：动态库只有在需要时才被加载到内存中，相比静态库可以节省内存空间。更新方便：如果动态库需要更新，只需要替换原有的库文件即可，不需要重新编译整个应用程序，可以提高开发效率。共享性：动态库可以被多个应用程序共享，可以减少代码的重复编译，提高代码的可重用性。动态性：动态库具有较强的动态性，可以在运行时加载、卸载，动态链接等，可以实现更加灵活的应用程序设计。
静态库和动态库的区别:
静态库是.o文件的集合，.o文件并没有执行过链接动作，.o文件中引用其他文件的符号并没有经过解引用操作，其他文件可以是同一个代码库下其他的.o文件，也可以是其他模块的静态库，还可以是其他模块或操作系统的动态库。静态库是按需链接，链接的粒度是.o文件，不是大家常在编译命令中看到的.a文件，只有被引用的符号所在的.o 文件才会被写入应用程序。如果代码中没有用静态库中的函数，即便在编译命令中指定了该库，连接器也会不会发生连接。
总的来说，一般来说，静态库适用于小型项目和需要独立部署的应用程序，动态库适用于大型项目和需要多个应用程序共享的代码库。

简而言之：
静态库是和你的程序直接“绑定”在一起的，它使得你的程序变得更大，但不需要依赖其他任何东西。
动态库则是和你的程序“分开”的，多个程序可以共享同一个动态库，使得你的程序更小、更灵活，但需要确保动态库是可用的。

# C++ 内存池实现

```
C++ 实现内存池通常涉及几个关键步骤：初始化内存池、分配内存块、回收内存块以及可能的内存池扩展。以下是一个相对简单但功能完备的内存池实现，并附有详细解释：

cpp
#include <iostream>  
#include <list>  
#include <cassert>  
#include <mutex>  
  
// 定义一个内存块结构体，用于记录每个内存块的元信息  
struct MemoryBlock {  
    void* ptr; // 内存块的地址  
    size_t size; // 内存块的大小  
};  
  
class MemoryPool {  
public:  
    // 构造函数，初始化内存池  
    MemoryPool(size_t chunkSize)  
        : chunkSize_(chunkSize) {  
        expandPool(); // 初始时创建一个内存池  
    }  
  
    // 析构函数，释放内存池中的所有内存块  
    ~MemoryPool() {  
        for (auto chunk : chunks_) {  
            delete[] reinterpret_cast<char*>(chunk.ptr);  
        }  
    }  
  
    // 分配内存块  
    void* alloc() {  
        std::lock_guard<std::mutex> lock(mutex_); // 锁定互斥量  
        if (freeChunks_.empty()) {  
            expandPool(); // 如果没有可用的内存块，则扩展内存池  
        }  
        MemoryBlock block = freeChunks_.back(); // 取出最后一个可用的内存块  
        freeChunks_.pop_back(); // 从可用列表中移除  
        return block.ptr; // 返回内存块的地址  
    }  
  
    // 回收内存块  
    void dealloc(void* ptr) {  
        std::lock_guard<std::mutex> lock(mutex_); // 锁定互斥量  
        for (auto it = freeChunks_.begin(); it != freeChunks_.end(); ++it) {  
            if (it->ptr == ptr) { // 如果找到对应的内存块  
                freeChunks_.erase(it); // 从可用列表中移除，因为不应该重复回收同一块内存  
                return;  
            }  
        }  
        assert(false && "试图释放不属于内存池的内存块"); // 如果没找到，则断言失败  
    }  
  
private:  
    // 扩展内存池  
    void expandPool() {  
        size_t newSize = chunks_.empty() ? chunkSize_ : chunks_.back().size * 2; // 新内存块的大小，首次为chunkSize_，之后翻倍  
        char* newBlock = new char[newSize]; // 分配新的内存块  
        MemoryBlock block = {newBlock, newSize}; // 创建内存块结构体  
        chunks_.push_back(block); // 将新内存块添加到内存块列表中  
          
        // 将新内存块分割成多个小块，并添加到可用列表中  
        for (size_t i = 0; i < newSize; i += chunkSize_) {  
            freeChunks_.push_back({newBlock + i, chunkSize_});  
        }  
    }  
  
    size_t chunkSize_; // 每个内存块的大小  
    std::list<MemoryBlock> chunks_; // 内存块列表  
    std::list<MemoryBlock> freeChunks_; // 可用的内存块列表  
    mutable std::mutex mutex_; // 互斥量，保护对共享资源的访问  
};  
  
int main() {  
    MemoryPool pool(1024); // 创建一个内存池，每个内存块的大小为1024字节  
  
    void* ptr1 = pool.alloc(); // 分配一个内存块  
    void* ptr2 = pool.alloc(); // 再分配一个内存块  
  
    // ... 使用ptr1和ptr2指向的内存 ...  
  
    pool.dealloc(ptr1); // 回收ptr1指向的内存块  
    pool.dealloc(ptr2); // 回收ptr2指向的内存块  
  
    return 0;  
}
解释：
结构体 MemoryBlock：定义了一个简单的结构体来记录每个内存块的地址和大小。

构造函数 MemoryPool(size_t chunkSize)：初始化内存池，设置每个内存块的大小，并调用 expandPool 来初始化第一个内存块。

析构函数 ~MemoryPool()：释放所有已分配的内存块。

alloc() 方法：用于分配内存块。首先，它检查是否有可用的内存块。如果没有，则调用 expandPool 来扩展内存池。然后，从 freeChunks_ 列表中取出一个内存块并返回其地址。

dealloc(void* ptr) 方法：用于回收内存块。它遍历 freeChunks_ 列表来查找要回收的内存块，并从列表中移除。
```


# 十大排序算法

> https://blog.csdn.net/qq_35344198/article/details/106857042

# C++对象模型

> https://www.cnblogs.com/QG-whz/p/4909359.html
