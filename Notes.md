Notes for _Effective C++_
============================

## 导读
* 将class的构造函数声明为explicit，可以阻止它们被用来执行隐式类型转换。

  除非有一个好理由让构造函数被用于隐式类型转换，否则，应该声明为explicit

* pass-by-value意味“调用copy构造函数”

## 条款1：视C++为一个语言联邦
* 主要的C++ sublanguage包括：C，object-oriented C++，Template C++，STL

## 条款2：尽量以const，enum，inline替换#define
* 也就是“宁可让编译器替代预处理器”

* 一个属于枚举类型的数值，可以当作ints被使用 (the enum hack)

  ```
  class GamePlayer {
  Private
      enum { NumTruns = 5 }; // "the enum hack"
      int scores[NumTruns];
  }
  ```

* 取enum的地址不合法。

  如果你不想让别人获得一个pointer或reference指向某个整数常量，可以用enum。
  
* 即使为宏的参数都加上小括号，仍然可能出问题

  ```
  #define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
  int a = 5, b = 0;
  CALL_WITH_MAX(++a, b)       // a被累加1次
  CALL_WITH_MAX(++a, b + 10)  // a被累加2次
  ```

* templete inline函数可以获得宏的效率以及可预料行为和类型安全性

  上述宏可以改写为：

  ```
  templete<typename T>
  inline void callWithMax(const T& a, const T& b) {
      f(a > b ? a : b);
  }
  ```

## 条款3：尽可能使用const

* 指针和const。const出现在星号左边，被指物是常量；出现在星号右边，指针自身是常量。

  ```
  char greeting[] = "Hello";
  const char* p = greeting;         // const data
  char* const p = greeting;         // const pointer
  const char* const p = greeting;   // const pointer, const data
  ```

* 函数返回常量，避免对函数的结果进行赋值的误用。

  ```
  cosnt Rational operator *(const Rational& lhs, const Rational& rhs)
  if (a * b = c)
      ... // 编译器报错，因为a * b的结果是常量
  ```

* const成员函数

  让人容易理解那个函数会改动对象内容。使操作常量对象成为可能。
  
  如果两个函数只是常量性不同，可以被重载。
  
* C++的设计理念是保证bitwise constness，即成员函数在不更改任何成员变量（static除外时）才可以说是const

  为了使得常量性得到释放，在const成员函数中也能更改某些变量，可以定义变量为mutable。
  
  例如：
  
  ```
  class CTextBlock {
  public:
      ...
      std::size_t length() const;
  private:
      mutable std::size_t textLength;
  }
  
  std::size_t CTextBlock::length() const {
      textLength = ...;
      return textLength;
  }
  ```
  
* 常量性移除：const_cast<>

## 条款4：确定对象被使用前已先被初始化

* 使用member initialization list（成员初值列）对成员变量初始化。

  这样做效率较高。如果在构造函数中才进行复制，相当于先被默认值初始化，再被赋予新值。
  
* C++中有固定的“成员初始化顺序”：先base class再derived class；class成员变量按声明次序初始化。

## 条款5：了解C++默默编写并调用哪些函数

* 编译器给空类生成：copy构造函数、copy assignment操作符、析构函数、default构造函数

## 条款6：若不想使用编译器自动生成的函数，就应该明确拒绝它

* 具体做法是将相应的成员函数声明为private并且不予实现。
  例如，一个阻止copying动作的基类：
  
  ```
  class Uncopyable {
  protected:
      Uncopyable() {}
      ~Uncopyable() {}
  private:
      Uncopyable(const Uncopyable&);
      Uncopyable& operator=(const Uncopyable&);
  }
  ```
  
## 条款7：为多态基类声明virtual析构函数

* 当derived calss对象经由一个base class指针被删除，而该base class带有一个non-virtual析构函数，其结果未定义，实际执行的结果往往是derived成分没有销毁。

* 当类不作为基类，声明virtual析构函数会增加额外开销。

  对象体积增加：放入一个字长的vptr。对于本身成员变量很少的类来说，overhead巨大。
  
* 不要继承一个带有non-virtual析构函数的class，例如标准容器

* 有的class并不是被设计为“经由base class接口处理derived class对象”，或者说不是为了具备多态性，因此不需要virtual析构函数。

## 条款8：别让异常逃离析构函数

