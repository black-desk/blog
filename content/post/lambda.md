---
title: lambda
date: 2022-07-10T02:23:58+08:00
draft: false
---
这里整理一些关于`c++`中的匿名函数的知识.

《C++ primer》上有的内容就不在重述了.这里重点讲一些primer上**可能**没有的东西.

## 捕获的时机

primer 向我们介绍到: 有两种捕获变量的方式, 值捕获和引用捕获. 其中:

> ...
> 
> 与参数不同,被捕获的变量的值是在lambda创建时拷贝,而不是调用时拷贝
> 
> ...
> 
> 如果我们采用引用方式捕获一个变量,就必须确保被引用的对象在lambda执行的时候是存
> 在的.lambda捕获的都是局部变量,这些变量在函数结束后就不复存在了.
> 
> ...

我们看一段代码:

``` cpp
auto funtionTimesMod1(int mod) {
  int variableA = mod;
  auto f = [variableA](int a, int b) { return a % variableA * b % variableA; };
  return f;
}
void test1() {
  cout << "---test1---" << endl;
  int a = 10, b = 20, c = 7;
  auto times = funtionTimesMod1(c);
  cout << times(a, b) << endl;
}
```

这里`funtionTimesMod1(int mod)`返回一个签名为`int(int, int)`的函数,这个函数计算
两个参数相乘对`mod`取模的结果.

当然可以正常运行,结果为`4`,如果我们将它换成引用捕获:

``` cpp
auto funtionTimesMod2(int mod) {
  int variableA = mod;
  auto f = [&variableA](int a, int b) { return a % variableA * b % variableA; };
  return f;
}
void test2() {
  cout << "---test2---" << endl;
  int a = 10, b = 20, c = 7;
  auto times = funtionTimesMod2(c);
  cout << times(a, b) << endl;
}
```

这段代码就已经不能正常工作了,它的输出是 `0`.

我们来看看为啥:

    Process 15820 stopped
    * thread #1, name = 'tmp', stop reason = step in
        frame #0: 0x000000000040096b tmp`funtionTimesMod2(mod=7) at tmp.cpp:17
       14   }
       15
       16   auto funtionTimesMod2(int mod) {
    -> 17     int variableA = mod;
       18     auto f = [&variableA](int a, int b) { return a % variableA * b % variableA; };
       19     return f;
       20   }
    (lldb) p &variableA
    (int *) $1 = 0x00007fffffffe120
    
    Process 15820 stopped
    * thread #1, name = 'tmp', stop reason = step over
        frame #0: 0x00000000004009dc tmp`test2() at tmp.cpp:25
       22     cout << "---test2---" << endl;
       23     int a = 10, b = 20, c = 7;
       24     auto times = funtionTimesMod2(c);
    -> 25     cout << times(a, b) << endl;
       26   }
       27   auto funtionTimesMod3(int mod) {
       28     int variableA = mod;
    (lldb) p times
    ((anonymous class)) $2 = {
      variableA = 0x00007fffffffe120
    }

可以看到`times`这个对象中保存下来的`variableA`只是一个指针,它指向我们之前创建的
局部变量`variableA`,这个地址在栈上,这意味着当我们真的调用`times`的时候,局部变
量`variableA`所在的那片内存已经被使用过了.所以会返回错误的结果.接下来,我们可以
看到实际上调用的时候`variableA`里面是`20`,这是因为刚好参数`b`被放置在
了`variableA`之前的位置上.

    Process 21271 stopped
    * thread #1, name = 'tmp', stop reason = step in
        frame #0: 0x0000000000400a32 tmp`funtionTimesMod2(this=0x00007fffffffe158, a=10, b=20)::$_1::operator()(int, int) const at tmp.cpp:18
       15
       16   auto funtionTimesMod2(int mod) {
       17     int variableA = mod;
    -> 18     auto f = [&variableA](int a, int b) { return a % variableA * b % variableA; };
       19     return f;
       20   }
       21   void test2() {
    (lldb) p variableA
    (int) $3 = 20
    (lldb) p &b
    (int *) $4 = 0x00007fffffffe120

## lambda 的实现

根据《C++ Primer》在10.3以及14.8.1的介绍,我们知道一个lambda表达式是由编译器负责
翻译成一个**没有类名,重载过调用运算符**的对象的.

这解释了为什么我们只能用`auto`来定义一个lambda类型的变量,因为这个类是没有名字的,
或者说它的名字是编译器自己生成的,我们并不能知道它叫什么.

这听起来很美好,我们也确实可以写出代码,用一个自己创建的,重载过调用运算符的对象,来
模拟一个lambda:

``` cpp
class INT {
public:
  int num;
  INT(const INT &i) : num(i.num) { cout << "Copy constructor" << endl; }
  INT() = default;
};

auto funtionTimesMod3(int mod) {
  INT variableA;
  cout << "----A" << endl;
  variableA.num = mod;
  cout << "----B" << endl;
  auto f = [variableA](int a, int b) {
    cout << "----C" << endl;
    return a % variableA.num * b % variableA.num;
  };
  cout << "----D" << endl;
  return f;
}

void test3() {
  cout << "---test3---" << endl;
  int a = 10, b = 20, c = 7;
  cout << "----E" << endl;
  auto times = funtionTimesMod3(c);
  cout << "----F" << endl;
  cout << times(a, b) << endl;
  cout << "----G" << endl;
}

auto funtionTimesMod4(int mod) {
  INT variableA;
  cout << "----A" << endl;
  variableA.num = mod;
  cout << "----B" << endl;
  class function {
    const INT variableA;

  public:
    function(const INT &variableA) : variableA(variableA) {}
    int operator()(int a, int b) {
      cout << "----C" << endl;
      return a % variableA.num * b % variableA.num;
    }
  } f(variableA);
  cout << "----D" << endl;
  return f;
}

void test4() {
  cout << "---test4---" << endl;
  int a = 10, b = 20, c = 7;
  cout << "----E" << endl;
  auto times = funtionTimesMod4(c);
  cout << "----F" << endl;
  cout << times(a, b) << endl;
  cout << "----G" << endl;
}

```

为了观察发生的值拷贝的时机,以确定我们自己写的这个类的行为和lambda真的完全一致,我
们可以自定义一个class,并且让他发生拷贝构造的时候打印一些信息.

运行结果如下:

    ---test3---
    ----E
    ----A
    ----B
    Copy constructor
    ----D
    ----F
    ----C
    4
    ----G
    ---test4---
    ----E
    ----A
    ----B
    Copy constructor
    ----D
    ----F
    ----C
    4
    ----G

这又一次说明了说明捕获引起的值拷贝发生在lambda被初始化的时候.

看起来这两个东西的行为几乎一致, 但是真的是这样吗?

## 编译器的优化

编译器会对我们写的代码做出一些优化,以减少复制对象的次数.如果不了解这点可以看看[
这篇](https://www.cnblogs.com/kekec/p/11303391.html)博客.

如果我们关闭返回值优化,那么运行的结果是这样的:

    ---test3---
    ----E
    ----A
    ----B
    Copy constructor
    Copy constructor
    ----D
    Copy constructor
    Copy constructor
    ----F
    ----C
    4
    ----G
    ---test4---
    ----E
    ----A
    ----B
    Copy constructor
    ----D
    Copy constructor
    Copy constructor
    ----F
    ----C
    4
    ----G

可以看到我们自己写的类,少了一次拷贝构造.

我们先来解释一下lambda为什么会发生这么多次拷贝构造.

``` cpp
auto funtionTimesMod3(int mod) {
  INT variableA; // 5
  cout << "----A" << endl; // 6 
  variableA.num = mod; // 7
  cout << "----B" << endl; // 8
  auto f = [variableA](int a, int b) { 
    cout << "----C" << endl; // 14
    return a % variableA.num * b % variableA.num; //15
  }; // 9
  cout << "----D" << endl; // 10
  return f; // 11
}

void test3() {
  cout << "---test3---" << endl; // 1
  int a = 10, b = 20, c = 7; // 2
  cout << "----E" << endl; // 3
  auto times = funtionTimesMod3(c); // 4
  cout << "----F" << endl; // 12
  cout << times(a, b) << endl; // 13
  cout << "----G" << endl; //16
}
```

首先我们的程序是按照如上的顺序运行的.

可以看到前两个拷贝构造发生在9,而第3、4次发生在11.

如果完全按照语义来看的话,9这句话可以有两种理解方式:

1. 创建一个lambda对象,对象名字叫f,这个对象的内容就是后面那个lambda表达式.
2. 创建一个lambda表达式,然后将其作为参数,调用同类型对象f的拷贝构造函数.

如果按照第一种方法来理解,那么第9行发生两次拷贝构造就不是很能理解了.

所以应该是第二种.

在10之后也发生了两次拷贝调用.应该是先建立了一个变量用来做返回值,比如说叫r,然后
将f赋值给返回值变量r,然后返回值变量r在被赋值给test3()中的times,这样发生的两次拷
贝构造.

``` cpp
auto funtionTimesMod4(int mod) {
  INT variableA;
  cout << "----A" << endl;
  variableA.num = mod;
  cout << "----B" << endl;
  class function {
    const INT variableA;

  public:
    function(const INT &variableA) : variableA(variableA) {}
    int operator()(int a, int b) {
      cout << "----C" << endl;
      return a % variableA.num * b % variableA.num;
    }
  } f(variableA); // <- 这里
  cout << "----D" << endl;
  return f;
}

void test4() {
  cout << "---test4---" << endl;
  int a = 10, b = 20, c = 7;
  cout << "----E" << endl;
  auto times = funtionTimesMod4(c);
  cout << "----F" << endl;
  cout << times(a, b) << endl;
  cout << "----G" << endl;
}
```

之前自己模拟lambda的这个代码,我们在注释标注的那个位置的实现和lambda中,"先建一个
右值,然后拷贝构造出f"的行为不太一样.导致这里少了一次拷贝构造.所以实际上应该是这
样:

``` cpp
auto funtionTimesMod5(int mod) {
  INT variableA;
  cout << "----A" << endl;
  variableA.num = mod;
  cout << "----B" << endl;
  class function {
    const INT variableA;

  public:
    function(const INT &variableA) : variableA(variableA) {}
    int operator()(int a, int b) {
      cout << "----C" << endl;
      return a % variableA.num * b % variableA.num;
    }
  };
  function f = function(variableA);
  cout << "----D" << endl;
  return f;
}

void test5() {
  cout << "---test5---" << endl;
  int a = 10, b = 20, c = 7;
  cout << "----E" << endl;
  auto times = funtionTimesMod5(c);
  cout << "----F" << endl;
  cout << times(a, b) << endl;
  cout << "----G" << endl;
}
```

实际上没必要纠结那么多,正常编译的话,其实是不会去先建一个右值的对象的.

## 代码

``` cpp
#include <bits/stdc++.h>
using namespace std;

auto funtionTimesMod1(int mod) {
  int variableA = mod;
  auto f = [variableA](int a, int b) { return a % variableA * b % variableA; };
  return f;
}
void test1() {
  cout << "---test1---" << endl;
  int a = 10, b = 20, c = 7;
  auto times = funtionTimesMod1(c);
  cout << times(a, b) << endl;
}

auto funtionTimesMod2(int mod) {
  int variableA = mod;
  auto f = [&variableA](int a, int b) { return a % variableA * b % variableA; };
  return f;
}
void test2() {
  cout << "---test2---" << endl;
  int a = 10, b = 20, c = 7;
  auto times = funtionTimesMod2(c);
  cout << times(a, b) << endl;
}

class INT {
public:
  int num;
  INT(const INT &i) : num(i.num) { cout << "Copy constructor" << endl; }
  INT() = default;
};

auto funtionTimesMod3(int mod) {
  INT variableA;
  cout << "----A" << endl;
  variableA.num = mod;
  cout << "----B" << endl;
  auto f = [variableA](int a, int b) {
    cout << "----C" << endl;
    return a % variableA.num * b % variableA.num;
  };
  cout << "----D" << endl;
  return f;
}

void test3() {
  cout << "---test3---" << endl;
  int a = 10, b = 20, c = 7;
  cout << "----E" << endl;
  auto times = funtionTimesMod3(c);
  cout << "----F" << endl;
  cout << times(a, b) << endl;
  cout << "----G" << endl;
}

auto funtionTimesMod4(int mod) {
  INT variableA;
  cout << "----A" << endl;
  variableA.num = mod;
  cout << "----B" << endl;
  class function {
    const INT variableA;

  public:
    function(const INT &variableA) : variableA(variableA) {}
    int operator()(int a, int b) {
      cout << "----C" << endl;
      return a % variableA.num * b % variableA.num;
    }
  } f(variableA);
  cout << "----D" << endl;
  return f;
}

void test4() {
  cout << "---test4---" << endl;
  int a = 10, b = 20, c = 7;
  cout << "----E" << endl;
  auto times = funtionTimesMod4(c);
  cout << "----F" << endl;
  cout << times(a, b) << endl;
  cout << "----G" << endl;
}

auto funtionTimesMod5(int mod) {
  INT variableA;
  cout << "----A" << endl;
  variableA.num = mod;
  cout << "----B" << endl;
  class function {
    const INT variableA;

  public:
    function(const INT &variableA) : variableA(variableA) {}
    int operator()(int a, int b) {
      cout << "----C" << endl;
      return a % variableA.num * b % variableA.num;
    }
  };
  function f = function(variableA);
  cout << "----D" << endl;
  return f;
}

void test5() {
  cout << "---test5---" << endl;
  int a = 10, b = 20, c = 7;
  cout << "----E" << endl;
  auto times = funtionTimesMod5(c);
  cout << "----F" << endl;
  cout << times(a, b) << endl;
  cout << "----G" << endl;
}

int main() {
  test1();
  test2();
  test3();
  test4();
  test5();
}
```
