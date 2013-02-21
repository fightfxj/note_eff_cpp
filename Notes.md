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

  析构函数抛出异常而没有正确结束可能造成内存泄漏。如果需要对造成的异常作出反应，应该提供一个普通函数，而不是在析构函数中操作。
  
## 条款9：绝不在构造和析构过程中调用virtual函数

  在derived class的构造过程中，base的构造函数会首先调用，在base构造函数执行过程中，derived class版本的virtual函数不会被调用。
  
## 条款10：令operator=返回一个reference to *this

  为了实现“连锁赋值”，赋值操作符必须返回一个reference指向操作符的左侧实参。
  
## 条款11：在operator=中处理“自我赋值”

  特别是赋值过程中需要申请新的空间的操作，特别要注意自我赋值的问题。否则会重复申请空间，且之前的空间没被释放。
  
  例如，用class保存一个指针指向动态分配的位图：
  
  ```
  class Bitmap { ... };
  Class Widget {
      ...
  private:
      Bitmap* bp;       // 指针，指向一个从heap分配得到的对象。
  };
  
  Widget&
  Widget::operator=(const Widget& rhs) {
      delete pb;
      pb = new Bitmap(*rhs.pb);
      return *this;
  }
  ```
  
  在自我赋值时，上述代码会将自己的pb先删掉，从而无法赋值。同时，该代码也不是“异常安全”的。如果new出问题，会有一个指针指向已经删除的空间。
  
  一种做法是“证同测试”（identity test），达到自我赋值检验的目的。另一种是copy-and-swap技术，可以提供异常安全性。
  
  ```
  Widget& Widget::operator=(const Widget& rhs) {
      Widget temp(rhs);   // 制作副本
      swap(temp)          // 将副本与*this的数据交换
      return *this;
  }
  ```
  
## 条款12：复制对象时勿忘其每一个成分

* 编写copying函数时，确保（1）复制所有local成员变量，（2）调用所有base classes内的适当copying函数。

* 不要让copy assignment操作符调用copy构造函数，或者反过来，应该建立一个新的private函数供copying函数共同调用。

## 条款13：以对象管理资源

* 使用shared_ptr和auto_ptr，它们在构造函数中获得资源，并在析构函数中释放。这被称为：Resource Acquisition is Initialization; RAII.

  例如：

  ```
  Investment* createInvestment(); // 返回指针，指向动态分配对象。
  void f() {
      std::auto_ptr<Investment> pInv(createInvestment());
  }

  ```

* auto_ptr若通过copying函数复制，会变成null，复制所得的指针获得资源唯一拥有权。

* shared_ptr复制时，复制前后的指针都指向同一个对象，销毁时同时销毁。

* auto_ptr和shared_ptr都在析构函数中用delete而不是delete []，所以不要在动态分配得到的array身上使用二者。

## 条款14：在资源管理类中小心copy行为

## 条款15：在资源管理类中提供对原始资源的访问

* 显示转换，shared_ptr和auto_ptr都提供一个get成员函数，返回智能指针内部的原始指针的附件。

* 隐式转换，用operatr->和operator*隐式转换至底部原始指针。

* 显示转换比较安全，但隐式转换对客户比较方便。

## 条款16：成对使用new和delete时要采取相同的形式

  new配对delete，new[]配对delte[]
  
## 条款17：以独立语句将newed对象置入只能指针

* 在“资源被创建”和“资源被转换为资源管理对象”两个时间点之间有可能发生异常。

  例如：
  
  ```
  int priority();
  void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
  
  // 如下写法不安全：
  processWidget(std::tr1::shared_ptr<Widget> (new Widget), priority())
  
  // priority()被调用的顺序由编译器确定，对同语句的执行顺序，编译器有控制的自由
  // 如果priority()的调用发生异常，可能new Widget得到的资源没放入智能指针
  
  // 正确写法:
  std::tr1::shared_ptr<Widget> pw(new Widget);
  processWidget(pw, priority());
  ```

## 条款18：让接口容易被正确使用，不易被误用

* 考虑用户可能做出什么样的错误

* 导入新类型，封装和预先定义所有合法的输入

  例如如下是一个安全有效的月份的定义：
  
  ```
  class Month {
  public:
      static Month Jan() { return Month(1); }
      ...
  private:
      explicit Month(int m)       // 阻止生成新的月份
  }
  Date d(Month::Jan(), Day(30), Year(1995));
  ```
  
* 除非有好理由，否则尽量令你的types的行为与内置types一致。**提供行为一致的接口**。

  例如客户知道int的行为，应该努力让你的types在合理的前提下有相同的表现。例如对a*b赋值不应该合法，所以operator*的返回值应该是const。
  
* 与其让客户将资源放入智能指针，不如只暴露智能指针给客户操作。消除客户的资源管理责任。

  客户有可能忘记。
  
## 条款19：设计class犹如设计type

  设计每个class都要面对的提问：
  
  * 新type的对象应该如何创建和销毁？
  * 对象的初始化和对象的赋值该有什么区别？
  * 新type的对象如果被passed by value，意味着什么？
  * 什么是新type的“合法值”
  * 新type需要配合某个继承图系吗？是则析构函数往往是virtual。
  * 新type需要怎么样的转换
  * 什么样的操作符和函数对此type是合理的？
  * 什么样的标准函数应该是private？
  * 谁该取用新tpye的成员？
  * 什么是新type的未声明接口？
  * 你的新type有多么一般化？是否应该用templete。
  * 你真的需要一个新type吗？
  
## 条款20：宁以pass-by-reference-to-const替换pass-by-value

* 缺省情况下C++以by value的方式传递对象至函数，参数和返回值都是一个以copy构造函数得到的副本。这可能导致过高的赋值开销。

* 以by reference方式传递参数可以避免slicing（对象切割）问题。

  当一个derived class对象以by value方式传递并被视为base class对象，base class的copy构造函数会被调用，而“草成此对象的行为像个derived class对象”的那些特化性质被完全切割掉了。
  
* 对STL迭代器和函数对象，以及内置类型，pass-by-value往往比较恰当

## 条款21：必须返回对象时，别妄想返回其reference

* reference只是个名称，代表某个既有的对象。

  任何时候看到一个reference声明式，都应该立刻问自己，它的另一个名称是什么。
  
* 如果返回reference指向一个local对象，函数返回时，local对象就被销毁。

## 条款22：将成员变量（data menbers）声明为priavate

* 如果以函数取得或者设定成员变量，就可以实现：只读、只写、读写、无读写等访问控制。

* **封装性**。日后可以改为用计算代替成员变量。

  **public意味着不封装，不封装意味着不可改变**。protected成员变量就像public成员一样缺乏封装性。其实只有两种访问权限：private（提供封装）和其他（不提供封装）。
  
## 条款23：宁以non-member、non-friend替换member函数

* 面向对象守则要求数据应尽可能封装。member函数带来的封装性比non-member函数低。

* 对类进行操作的函数，可以用non-member函数的形式写成许多便利函数的方式，放在同一个namesapce中。

  namespace可以跨越多个源码文件，而classes不能。
  这正是C++标准程序库的组织方式。并不是拥有单一、整体、庞大的一个头文件，并在其中包含std的每一样东西，而是有几十个头文件，每个头文件声明std的某些机能。
  
## 条款24：若所有参数皆需类型转换，请为此采用non-member函数

  例子：
  
  ```
  class Rational {
  public:
      Rational (int numerator = 0, int demominator = 1); // 刻意不是explicit，允许int-to-Rational隐式转换。
      int numerator() const;
      int denominator() const;
      
      cosnt Rational operator* (const Rational& rhs) const; // 成员函数版本
  private:
      ...
  }
  
  Rational onehalf(1, 2);
  result = oneHalf * 2;             // 正确
  result = 2 * oneHalf;             // 错误
  result = 2.operator*(oneHalf);    // 错误，与上一行的调用方式等价
  
  // non-member版本：
  const Rational operator* (const Rational& lhs, const Rational& rhs) {
      ...
  }
  ```
  
  只有当参数位于参数列表内，这个参数才有可能被进行隐式转换。在成员版本的operator*中，只有rhs的操作数在参数列表中。
  
## 条款25：考虑写出一个不抛异常的swap函数

* std中默认的swap会进行三次复制

* std命名空间的东西通常不允许改变，我们定制的swap可以实现为标准templates的特化版本

  例如：
  
  ```
  class Widget {
  public:
      ...
      void swap(Widget& other) {
          using std::swap;
          swap(pImpl, other.pImpl)
      }
  }
  
  namespace std {
      template<>            // std::swap的特化版本
      void swap<Widget>(Widget&a, Widget&b) {
          a.swap(b);
      }
  }
  ```
  
  由于可能触及Widget的私有成员变量，特化版本中调用了Widget中提供的swap函数。同时在Widget中，声明using，表示使用std中的特化版本。
  
## 条款26：尽可能延后变量定义式的出现时间

* 只有定义但是没有实参动作，将会调用default构造函数，造成额外开销。

* 应该尽量延后定义直到能够给他初值实参为止，“以具有明显意义之初值”将变量初始化，还可以附带说明变量的目的。

## 条款27：尽量少做转型动作

* 两种风格的转型动作：

  * C风格：(T)expression
  * 函数风格：T(expression)

* 四种新式转型：

  * const_cast<T>(expression)：移除对象的常量性
  * dynamic_cast<T>(expression)：安全向下转型，决定某对象是否归属继承体系中的某个类型。唯一无法由旧式语法执行的转型动作。
  * reinterpret_cast<T>(expression)：实际结果取决于编译器，因此不可移植。
  * static_cast<T>(expression)：强迫隐式转换
  
* 尽量避免dynamic_cast，试着开发无需转型的替代设计。尽量使用新式的转型。

## 条款28：避免返回handles指向对象内部成分

  这样做可以帮助可封装性，帮助const成员函数的行为像个const
  
## 条款29：努力做到“异常安全”

* 异常被抛出时，带有异常安全行的函数会：

  * 不泄漏任何资源，比如lock等不会被违法占据
  * 不允许数据败坏，数据的逻辑性不会被破坏，例如表示对象数目的累加器逻辑仍然正确

* 异常安全的三个保证层次

  * 基本承诺：任何事物仍然在有效状态，没有任何对象或数据结构因此败坏。
  * 强烈保证：程序状态不改变，如果函数失败，程序会回到“调用之前”的状态。
  * 不抛异常：承诺不抛出异常，承诺总是能够完成指定的功能。

* 往往用copy-and-swap实现强烈保证

  关键在于“修改对象数据的副本，然后在一个不抛异常的函数中将修改后的数据与原件置换”。由于要制作副本，会有额外开销。

* 一个软件系统要不就具备异常安全行，要不就全然否定

## 条款30：透彻了解inlining的里里外外

* inline函数背后的整体观念是，将“对此函数的每一个调用”都以函数本体替换之

* 两种定义方式：
  
  * 显示指出inline
  * 隐式：将函数定义在class的定义式中
  
* 过度inline可以造成严重的代码膨胀。将大多数inline限制在小型、被频繁调用的函数身上。

## 条款31：将文件间的编译依存关系降至最低

* 如果使用object references或object pointers可以完成任务，就不要使用objects

* 如果可以，尽量以class声明式替换class定义式

  例如：
  
  ```
  class Date;                       // class声明式
  Date today();
  void clearAppointments(Date d);   // Date的定义式
  ```
  
  声明上述两个函数并不需要知道Date的定义。
  
* 为声明式和定义式提供不同的头文件。使得编译相依于声明式，不要相依于定义式。

  程序头文件应该以“完全且仅有声明式”的形式存在。这种做法无论是否涉及templates都适用。
  
## 条款32：确定你的public继承塑模出is-a关系

* 如果让D public继承B，则意味着：每一个类型为D的对象同时也是一个类型为B的对象，反之不成立。同时，B对象可以派上用场的任何地方，D对象一样可以派上用场。

## 条款33：避免掩盖继承而来的名称

