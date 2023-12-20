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

# move 了解吗？有什么作用？
> https://www.cnblogs.com/S1mpleBug/p/16703328.html
> https://zhuanlan.zhihu.com/p/580797507
