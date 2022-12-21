# Effective C++

## 一：C++基础

### C++很成熟，很NB

C++支持面向过程（procedural）、面向对象（object-orientend）、函数形式（functional）、泛型形式（generic）、元编程形式（metaprogramming）

其核心是四个部分

- C
  - 区块block
  - 语句statements
  - 预处理器preprocessor
  - 内置数据类型
  - 数组arrays
  - 指针pointers
- Object-Orientend C++
  - 类classes（构造函数，析构函数）
  - 封装encapsulation
  - 继承inheritance
  - 多态polymorphism
  - 虚函数virtual（动态绑定）
- Template C++
- STL

### 替换#define

使用编译器替代预处理器

尽量使用const、enum定义常量，使用inlines定义函数宏

#### const

```c++
#define PI 3.1415926
```

因为`#define`不是语言的一部分，在编译器开始工作前，`PI`就会被处理掉，所以一旦报错，你无法追踪到`PI`，只能看到`3.1415926`，这会**浪费你的时间**

应该改为

```c++
const double Pi 3.1415926;
```

值得注意的事

- 定义常量指针指向char*-based字符串

```c++
const char* const authorName = "Reuben";
```

- 作用域限制在class内的常量，需要让其成为类的一个成员，并且为了让常量至多有一份实体，必须让其成为一个静态成员

```c++
class GemePlayer{
private:
	static const int NumTurns = 5;	//这只是一个声明式
	int scores[NumTurns];
};
const int GamePlayer::NumTurns;		//这是定义式，因为在声明时已经赋值，所以这里就不赋值了
```

C++要求我们使用的所有东西都提供一个定义式，但如果不取其地址，可以只有声明式，不写定义式

从这里可以看出，**const可以封装**，而#define不行

#### enum

```c++
class GemePlayer{
private:
	enum { NumTurns = 5 };
	int scores[NumTurns];
};
```

### const指针

const在星号左边，被指物是常量

```c++
char greeting[] = "Hello";
const char* p = greeting;
```

const在星号右边，指针本身是常量

```c++
char greeting[] = "Hello";
char* const p = greeting;
```

const在星号两边，被指物和指针都是常量

```c++
char greeting[] = "Hello";
const char* const p = greeting;
```

### 确认对象在使用前已经被初始化

C++初始化顺序

- 基类比子类先初始化
- 成员变量根据其声明次序初始化

## 二：构造/析构/赋值

### 空类的默认函数

一个空类，编译器会给他声明一个copy构造函数，一个copy赋值操作符，一个析构函数

一个类，如果没有构造函数，也会自动声明一个default构造函数

这些函数都是public且inline的

### 禁用自动生成的函数

如果希望不自动生成coyy构造和copy赋值函数，但又不愿意自己定义相关函数，最好禁用掉（而且如果你自己定义了copy函数，那你的类就支持copy了，这可能不是你所希望的）

- 你可以把copy函数定义为private类型，并不实现他们（甚至参数不需要写参数名）

```c++
class HomeForSale{
private:
	HomeForSale(const HomeForSale&);
  HomeForSale& operator=(const HomeForSale&);
};
```

可以制作一个不可被copy的类，让子类继承

```c++
class Uncopyable{
protected:
	Uncopyable(){}
  ~Uncopyable(){}
private:
  Uncopyable(const Uncopyable&);
  Uncopyable& operator=(const Uncopyable&);
};

class HomeForSale: private Uncopyable{
  ...
};
```

### 为多态基类声明virtual析构函数

#### 一定要有一个virtual析构函数

- 如果这个类要成为一个基类，那么一定要有一个virtual析构函数

在工厂模式，我们使用工厂函数构造的对象需要适当的delete掉，但是，我们不能依赖客户去使用delete函数，因为他们可能会用错

```c++
class TimeKeeper{
public:
	...
};
class AtomicClock: public TimeKeeper {...};
class WaterClock: public TimeKeeper {...};
```

```c++
TimeKeeper* ptk = getTimeKeeper();		//创建一个动态分配对象
...
delete ptk;	//释放对象，避免资源泄漏
```

上面这个过程的问题其实出在`getTimeKeeper()`指向一个派生类（derived class）对象（比如`AtomicClock`），而这个对象却要经由一个基类（base class）指针删除（比如`TimeKeeper*`）

如果这个基类的析构函数不是virtual的，就会出现问题：

- 如果派生类有基类没有的成分（成员变量），这些新成分有可能不会被销毁，于产生了一个诡异的“局部销毁”对象

解决方法就是给基类一个virtual析构函数

```c++
class TimeKeeper{
public:
	TimeKeeper();
  virtual ~TimeKeeper();
  ...
};
```

#### 最好不要有virtual析构函数

- 如果这个类不可能成为基类，那么最好不要有virtual析构函数

为什么呢？因为要实现virtual函数，对象必须携带某种信息，用于在运行时确定该调用哪一个virtual函数，这个信息通常由**vptr（virtual table pointer）**指针携带，这个指针指向一个由函数指针构成的数组，称为**vtbl（virtual table）**，每一个带有virtual函数的类都有一个属于自己的vtbl

**这个vtbl会增大对象的体积**，一个指针要64bits（这个要取决于电脑系统的寻址范围，只不过现在电脑都是64位的？），对一些比较小的类来说，增加64bits可能会让容量翻倍

**这个vtbl会让代码失去兼容性**，因为其他语言没有vtpr，这就会导致C++的对象和其他语言（如C）结构不同，于是没法接受/传递给其他语言，如果你自己实现vptr，那将不再具备移植性

#### 请不要继承没有virtual析构函数的类

比如string、vector、list、set等等

而且 C++没有像 Java的`final classes`或者C#的`sealed classes`的禁止派生机制

### 不要在析构函数里抛出异常

当一个`vector v`容器被销毁时，其所包含的所有元素也需要被销毁。如果被销毁的元素的析构函数里可以抛异常，如果析构其中第一个元素的时候，抛了一个异常，如果后续的元素被析构时，也抛了异常，这样就会导致程序结束或者不确定性行为

有两个不怎么好的解决方法

- 遇到异常，直接`std::abort()`，即遇到异常，宁愿直接强制停止程序，也不要让异常传播
- 遇到异常，把异常记录下来，另程序继续运转，即**吞下异常**
  - 这会使得这个错误被”低估“，但仍然会比直接强杀程序/程序不确定性执行要好

比较好的方法，这是一个控制数据库连接的函数，关闭连接时要析构相关内容

```c++
class DBConnevtion{
public:
    ...
    static DBConnevtion create();
    void close();     
};
class DBConn{
public:
    ...
    void close()	//封装给客户用的,关闭连接的函数
    {
        db.close();
        closed = true;
    }
    ~DBConn(){
        if(!closed){
            try{
                db.close();
            }
            catch(...){
                ...
                //强制关闭程序或者吞下异常
            }
        }
    }
private:
    DBConnection db;
    bool closed;
};
```

### 不要在构造和析构过程中调用virtual函数

在C++中（Java和C#没有这个烦恼），在构造和析构过程中调用virtual函数，有可能不会带来你所期望的结果

可以简单理解为**在C++中，基类构造期间，vritual函数不是vritual函数**

因为基类会先于派生类构造。基类构造时，构造的对象是基类类型，而非派生类类型，那么此时调用virtual函数，调用的是基类的版本，而非派生类的virtual函数

同理，进入派生类析构函数的对象，其派生出的成员变量就“消失”了，没法调用，进入基类析构函数的对象，就是一个基类对象，调用的virtual函数是基类版本的

### 令operator=返回一个对\*this的引用

连续赋值

```c++
x = y = z = 15;
//其实就等于x = (y = (z = 15))；
```

为了实现来连续赋值，赋值操作符必须返回一个reference，指向操作符左侧实参

```c++
class Widget{
public:
    Widget& operator=(const Widget& rhs)
    {
    	...
        return *this;
    }
    Widget& operator+=(const Widget& rhs){
        ...
        return *this;
    }
};
```

### 在operator=中处理自我赋值

如果对象自己赋给自己，我们称之为自我赋值

```c++
w = w;
a[i] = a[j]; //当i=j时，自我赋值
*px = *py;	//px和py指向同一个物体时，自我赋值
```

在赋值操作中：

1. 我们会先另左边的操作数先释放掉当前使用的数据
2. 令其使用右操作数的副本
3. 最后返回左操作数

```c++
class Widget{
    ...
private:
    Bitmap *pb;
};
//!!!这个不安全
Widget& Widget::operator=(const Widget& rhs){
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

如果自我赋值，即rhs和pb指向同一个对象，那么`delete pb`后，这个对象就已经被销毁了，下面使用的`*rhs`就是一个已经被删除的对象

解决方法1：延后delete

```c++
Widget& Widget::operator=(const Widget& rhs){
    Bitmap* pOrig = pb;
    pb = new Bitmap(*rhs.pb);
    delete pOrig;
    return *this;
}
```

解决方法2：使用copy and swap技术

```c++
Widget& Widget::operator=(const Widget& rhs){
    Widget temp(rhs);
    swap(temp);		//令*this与temp交换
    return *this;
}
```

### 复制对象的一切

如果你对class的成员变量做修改（比如继承），一定要对copy函数也做修改，不然可能会出错，而且这个错编译器不会报错

派生类需要重载copy函数，但基类部分的copy要通过调用基类的copy函数（因为基类的许多成员变量可能是private的）

所以copy函数需要

- 复制所有local变量
- 调用所有基类中的适当的copy函数

## 三：资源管理

### 让对象管理资源

将资源（主要是heap- based类型的资源）放入对象内，一旦控制流离开对象，对象的析构函数就会自动释放那些资源

- 申请资源后立即将其放进对象中，**资源获取时机就是初始化时机（Resource Acquisition is Initialization，RAII）**
- 在对象的析构函数中释放资源

C++的`auto_ptr`是一个**类指针（pointer-like）对象**，也就是**智能指针**，其析构函数会自动delete掉其所指向的对象

注意：

- 不要让多个`auto_ptr`指向同一个对象，因为一个对象被多次删除就会导致“未定义行为”
- `auto_ptr`如果被复制，则原指针会指向null，新指针对获取对象的唯一拥有权

**RCSP（引用计数型智能指针）**也是一种智能指针（比如`tr1::shared_ptr`），会持续跟踪有多少对象指向某个资源，只有这个资源无人指向时，才会删除该资源

### 小心copy行为

大多数RAII对象的copy函数：

- 禁止复制
- 采用引用计数法（RCSP）
- 复制底部资源（深拷贝）
- 转移底层资源所有权（auto_ptr）

### 在资源管理类中提供对原始资源的访问

有的时候你需要操作原始资源，而不是智能指针类型。两个智能指针都有一个get函数，用于显示转换，获取原始指针类型（或者是复件）

### new与delete一个数组

一个指针指向一个数组，如果删除这个指针，是删掉这个指针？还是同时一起删掉这个数组呢？

- 如果new了一个数组，就delete一个数组

```c++
string* ptr1 = new string[100];
delete [] ptr1;
```

- 如果new了一个对象，就delete一个对象

```c++
string* ptr2 = new string;
delete ptr2;
```

很多时候很难确定当前这个对象是数组还是一个对象

```c++
typedef string AddressLines[4];
string* pal = new AddressLines;
delete [] pal;
```

最简单的方法是不用数组，使用STL里的容器，如`vector<string>`

### 以独立语句将newed对象置入智能指针

C++中调用一个函数，会先计算每一个传递进去的实参

如果按下面的写法，将newed对象置入智能指针中

```c++
分配函数(shared_ptr<Widget>(new Widget), 资源访问);		//不要这样写
```

需要执行一下函数

- 调用“资源访问”函数（A）
- 执行`new Widget`（B）
- 调用`shared_ptr`构造函数（C）

然而和C#等语言不同，C++里这几个函数执行顺序无法确定（其实只是A的顺序无法确定，B一定会比C先执行）

如果执行顺序是BAC，且A执行异常，那么B所返回的指针将会遗失，不会正常放入智能指针中，于是发生资源泄漏

所以简单的方法是分离语句

```c++
shared_ptr<Widget> pw(new Widget);
分配函数(pw, 资源访问);
```

## 四：设计与声明

### 让接口容易被正确使用

客户会犯错，所以接口要考虑各种错误，接口也不能要求用户记住要做某件事（因为他们可能会忘记）

#### 限制参数传递

这是一个日期类

```c++
class Date{
public:
  Date(int month, int day, int year);
  ...
};
...
Date d(4, 20, 2022);
```

客户很有可能填错顺序，也有可能填入一个无效的参数

可以使用**外覆类型（wrapper types）**，当然做出类会更好

```c++
struct Day{
  explict Day(int d) : val(d) {}
  int val;
};
struct Month{
  explict Month(int m) : val(m) {}
  int val;
};
struct Year{
  explict Year(int y) : val(y) {}
  int val;
};
class Date{
public:
  Date(const Mouth& month, const Day& day, const Year& year);
  ...
};
...
Date d(Month(4), Day(20), Year(2022));
```

#### 一致性

自定义的行为要与内置类型的行为一致，比如你不能把`operator*`重载成`operator+`

或则像STL中，容器的接口都很一致，比如`size`、`push_back`等等

### 设计class犹如设计type

- 对象要如何创建和销毁
- 对象的初始化和对象的赋值有什么差别（拷贝构造与赋值操作）
- 对象如果被值传递，意味着什么（深浅拷贝）
- 约束成员变量的合法值
- 是否可以/需要被继承
- 能否类型转换，如何类型转换
- 支持何种操作符
- 成员变量的访问修饰
- 成员函数的访问修饰
- 未声明接口（undecided interface）
- 是否需要定义模版
- 真的需要一个新类吗？

### 多用引用传递

C++默认以值传递的方式传递参数，值传递会创建值的副本（也就意味着会调用构造函数和析构函数），这对一些**很大的自定义类型**来说性能非常糟糕

使用const引用传递会好很多

- 不会创建新的对象
- 不会改变原有对象
- 可以避免**对象切割**问题
  - 对象切割：当派生类对象作为一个基类对象进行值传递时，调用的是基类的拷贝构造函数，于是这个派生类对象被切割了

只不过引用传递是大多是通过指针实现的，在处理一些**简单的内置类型**时（比如int），效率反而不如值传递（内置类型也可能是存放在栈里的？），此外STL容器和迭代器设计之初就是为了值传递，慎用引用传递

### 必须返回对象时，不要返回引用

如果必须返回对象，请不要返回引用（比如`operator*`，`operator==`），因为有可能会返回一个指向不存在的对象的引用，比如返回了一个**右值**的引用（只不过右值也不能被取地址），或者返回了一个local对象的引用

### 将成员变量隐藏

成员变量应该为private，而不是public

- 客户只能通过成员函数访问变量，于是可以统一接口（全都是函数）
- 分离读写权限（这一点C#做的更好？）
- 封装后便于后续更新（改变成员变量后，客户仍可以使用旧成员函数访问）
- 便于对成员变量进行约束（更不容易出现异常值）
- protected并不比public更具有封装性

### 使用非成员函数

- C#，java选手可以略过
- C++标准库就是这样写的

这里有一个类，其中有多个成员函数

```c++
class WebBrowser{
public:
	void doA();
  void doB();
  void doC();
  ...
};
```

现在需要令一个函数做ABC三件事，有两种写法

- 成员函数

```c++
class WebBrowser{
public:
	...
  void doEverything(){
    doA();
    doB();
    doC();
  }
  ...
};
```

- 非成员函数

```c++
void doEverything(WebBrowser& wb){
	wb.doA();
  wb.doB();
  wb.doC();
}
```

令人意外的是，第二种方法（使用非成员函数）更好

#### 什么是封装

一个东西被封装，那么将不再可见，越多东西被封装，能被看见的东西就越少，那么我们能改变的东西会越少，弹性越小，封装性越强

为什么推崇封装，是因为封装可以让我们在只影响有限客户的情况下改变事物

#### 为什么第二种比第一种封装性更强

因为第一种给用户两种调用方法，一个是调用成员函数`doEverything()`，一个是调用所有的do函数。两者功能完全一致，而且还增多了第一个类的成员函数，降低了封装性

- 注意第一种方法中，`doEverything()`和其他do函数访问权利一致，而且都是public，这个函数的作用仅仅是提供便利

那么第二种方式是不是不太符合面向对象呢？因为这个函数不在类里。解决方法也很简单，定义一个工具类，将这个函数定义为该工具类的**静态成员（static member）函数**即可

或者把相关的非成员函数写在一个namespace里（也可以定义在一个头文件中，这样其他namespace可以选择性使用）

```c++
namespace WebBrowserStuff{
	class WebBrowser{...};
	void doEverything(WebBrowser& wb){...}
}
```

- 可拓展性更强
  - 客户可以自行定义一个头文件，然后在头文件中自己定义一个提供便利的非成员函数（毕竟对客户而言，类是不可/不应该修改的）

- 可拆分
  - 用户可以把不同的非成员函数放在不同的头文件中，按需索取（而类必须完整定义，不可分割）

### 如果所有参数都需要进行类型转换，使用非成员函数

令类支持隐式类型转换是一个非常糟糕的主意，但如果每次都做显示转化又非常麻烦，尤其是当你在做一个数值运算的函数时

比如一个有理数乘法

```c++
class Rational{
public:
  //这个类没有自定义的explict构造函数
  const Rational opertaor* (const Rational& rhs) const;
  ...
};
```

```c++
Rational oneEighth(1,8);
Rational oneHalf(1,2);
Rational result = oneHalf * oneEighth;	//成功
result = result * oneEighth;	//成功
result = oneHalf * 2;		//成功，等价于 result = oneHalf.operator*(2)
result = 2 * oneHalf;		//失败，等价于 result = 2.operator*(oneHalf)
```

`result = oneHalf * 2;`为什么成功，因为这里发生了一次隐式转换，将`2`转化为了一个`Rational`类型

在编译器中可能等价于

```c++
const Rational temp(2);
result = oneHalf * temp;
```

- 如果这个类有自定义的explict构造函数，上面这几个运算都失败，因为无法将`2`转化为一个`Rational`类型

`result = 2 * oneHalf;	`为什么会失败，因为隐式转换只能转换**参数列（parameter list）**内的参数，不能转化成员函数所隶属的对象（即this对象），此时`2`就是一个int类型，没有我们所自定义的`operator*`函数，自然会失败

可以发现，想要实现混合运算，非常麻烦，但是如果把这个运算做成一个非成员函数，会好很多（因为所有的操作数都是参数，都在参数列中，都可以被隐式转换）

```c++
const Rational operator*(const Rational& lhs, const Rational& rhs){
  return Rational(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
}
```

- 此外要极力避免使用**友元（friend）函数**
- 成员函数的对立面是非成员函数，一个函数不方便做成成员函数，要先考虑做成非成员函数



### 写一个不抛异常的swap函数

swap函数原本是STL的一部分，后来称为了**异常安全性编程**的核心，以及成为用来处理自我赋值的常见机制。总之swap很重要

std是一个很特殊的命名空间，客户可以**全特化**（total template specialization)std里面的templates，但是不可以添加新的templates到std里面，std的内容完全由C++标准委员会决定

全特化：针对某个类做模板函数的特例，如对`std::swap`做一个针对`Widget`的特化

```c++
class WidgetImpl{...};		//这个类的对象中存储着真正的数据
class Widget{
public:
    Widget& operator=(const Widget& rhs){
        ...
        *pImpl = *(rhs.pImpl);
        ...
    }
    ...
    void swap(Widget& other){	//这个函数决对不可抛异常
        using std::swap;
        swap(pImpl, other.pImpl);//这是pimpl写法，交换两个对象只需要置换其pImpl指针
    }
    ...
private:
    WidgetImple* pImpl;		//这个类有一个指向资源对象的指针
};
namespace std{
	template<>
    void swap<Widget>(Widget& a, Widget& b){	//这个可以抛异常
        a.swap(b);	
    }
}
```

此外，C++的STL容器就是上面这种写法，提供了`public swap`成员函数和`std::swap`的特化版本

## 五：实现（Implementations）

- 随意定义变量可能会导致性能降低
- 过度使用转型（casts）可能会导致性能降低，难以维护，可读性低
- 返回对象的内部数据的handles，可能会破坏封装
- 未考虑异常可能会导致资源泄露和数据败坏
- 过度使用inline可能会导致包体膨胀
- 过度耦合（coupling）可能会增加构建时间（build times）

### 尽量延后变量定义式的出现时间

#### 避免未曾使用的变量

如果你定义了一个（类型中带有构造函数或析构函数的）变量，当程序的**控制流（control flow）**到达这个变量时，就会调用构造函数函数，当这个变量离开作用域时，就会调用析构函数，尽管这个变量没有被使用过

此外，如果一个函数的中间代码抛了异常，那么前面的变量就有可能未被使用，白构造了、

#### 避免无意义的默认构造函数

如果你提前定义一个变量，这个变量可能会用默认构造函数构造，最后再赋值。这样不如将变量定义延后，用拷贝构造函数构造，这样性能会更好

#### 循环

此外，如果变量只在循环内使用，是将其定义在循环内呢？还是循环外？

**循环内**

```c++
for(int i = 0; i < n; i++){
  Widget w(...);
  ...
}
```

- n个构造函数+n个析构函数

- 如果`Widget`是一个很敏感的类，这样会让其作用域更小，更容易理解和维护

**循环外**

```c++
Widget w;
for(int i = 0; i < n; i++){
  w = ...;
  ...
}
```

- 一个构造函数+一个析构函数+n个赋值操作
- 如果赋值成本比构造+析构要低，这样更好（尤其是n很大的时候）

### 少做转型

C++是强类型语言，设计目标应该是保证类型错误绝不发生，然而**类型转换**破坏了类型系统

Java、C#，这些语言的类型转换比频繁，而且相对安全，但C++极具风险

C++的类型转化

- 旧式转换
  - `(T)expression`
  - `T(expression)`
- 新式转换
  - `const_cast<T>(expression)`
    - 用于将对象的**常量性转除（cast away the constness）**
    - 比如将`const`转化为`non-const`
  - `dynamic_cast<T>(expression)`
    - 用来**安全向下转型**
    - 无法由旧式语句执行
    - 耗费巨大
  - `reinterpret_cast<T>(expression)`
    - 用于低级转型，实际操作取决于编译器，不可移植
    - 极其少用
  - `static_cast<T>(expression)`
    - 用于**强迫隐式转换（implicit conversions）**
    - 比如`non-const`转化为`const`，`int`转化为`double`，`void*`转化为`typed`，基类指针转化为派生类指针

避免C++类型转换出问题的核心是**避免使用基类的接口处理派生类**

#### 一个对象多个地址

C++很神奇，如果一个基类指针指向一个派生类对象，如

```C++
Dervied d;
Base* b = &d;
```

这可能会导致两个指针值不一样，即这个对象有两个地址，一个`Derivied*`指针一个`Base*`指针，这派生类指针上往往会有一个**偏移量（offset）**，通过这个偏移量，可以通过派生类指针找到基类指针

上面这种事在Java、C#、C中绝对不会发生，但是C++可以多继承，很容易出现这种情况（单继承时也会出现），所以**请不要假定对象在C++中如何布局**，更不应该基于这个假设对对象进行类型转换

如果你想让当前对象调用基类的函数，如果对`*this`做强制转化，转换为基类，`*this`其实是先前产生的`*this`对象的基类部分，这个部分的成员函数可能与当前对象不同，最后导致调用函数出现问题

```c++
class SpecialWindow: public Window{
public:
  virtual void onResize(){
    //static_cast<Window>(*this).onResize();	//这样不好
    Window::onResize();		//请用这种方式调用基类的onResize函数（作用到当前对象上）
    ...
  }
}
```

#### dynamic_cast

这东西执行起来特别慢，尤其是操作深度继承和多重继承的对象

什么时候使用这个东西？当**你想在**一个你认为是派生类对象的**对象上执行**派生类的操作**函数**，但你手里却只有一个指向基类的引用/指针时

解决方法：

- 使用类型安全容器（比如智能指针），存储指向派生类对象的指针，然后操作容器
- 在基类中提供virtual函数

### 避免返回指向对象内部成分的handles

前面也写过，我们可以把数据分离出来，对象中只存一个指向数据的指针/引用，这样复制起来会更方便

但是如果我们传递出指向对象的指针/引用，即使这个数据是private类型，但这个数据是可以会被修改的

```c++
class Point{
public:
  ...
  void setX(int val);
  ...
};
struct RectData{
  Point ulhc;		//upper left hand corner
  Point lrhc;		//lower right hand corner
};
class Rectangle{
public:
  ...
  Point& upperLeft() const { return pData->ulhc; }	//这样返回了引用，非常不好
  ...
private:
  std::tr1::shared_ptr<RectData> pData;
};
...
rec.upperLeft().setX(50);		//rec是一个Rectangle类型，我们发现这里居然实现了对Rectangle的修改
```

`upperLeft`函数本来只是为了提供给客户获得（get）`Rectangle`的一个坐标点，结果通过修改其指向/引用的对象。调用一个const函数，修改（set）了`Rectangle`本身，而且还是一个内部数据`RectData`

为什么会出现这种情况呢？是因为成员函数返回了一个handles（包括指针、引用、迭代器）

解决方法很简单，只要让handles不可以被修改，就可以了

```c++
class Rectangle{
public:
  ...
  const Point& upperLeft() const { return pData->ulhc; }	
  ...
};
```

但这样还是返回了指向对象内部的handles，如果所指向的东西不存在，就会导致**dangling handles（空悬的号码牌）**，比如返回了一个对local变量的引用，依然特别危险

当然，有的时候不得不返回handles，比如`operator[]`

### 异常安全性很重要

**异常安全性（Exception safety）**即当异常被抛出时，满足一下两个条件：

- 不泄漏任何资源
- 不允许数据败坏

不泄漏资源比较好解决，前面已经做过使用对象管理资源了，而解决数据败坏比较复杂

三个保证：

- 基本承诺：如果异常被抛出，程序中任何事物仍保持在有效状态下，没有对象/数据结构/约束被破坏
- 强烈保证：如果异常被抛出，程序状态不改变。即如果函数成功，则完全成功，如果函数失败，则返回调用函数前的状态（可以通过`copy-and-swap`实现）
- 不抛掷（nothrow）：绝对不会抛出异常，任何情况下都能完美完成承诺的任务，比如内置类型int、指针等（但是很难实现）

**异常安全码**必须提供上述三种保障之一，如果不能保障，则不具备异常安全性

### 了解inline函数

内联函数有很多优点，比如比宏好很多，也可以免除函数调用成本，此外编译器会对这部分代码做最优化

缺点也很明显，会让包体变大，会导致**换页行为（paging）**，会降低cache命中率（如果内联函数很大的话），所以只适用于小型、频繁调用的函数

内联函数一般放在头文件中，因为大多数建置环境，在编译时进行内联

### 降低文件间的编译依存

如果头文件内的代码被修改，所以使用这个头文件的文件也会被重新编译

为什么C++要让类的实现放在定义式之中？其中一个重要因素是编译器必须在编译期间知道对象的大小，而知道对象大小的方法就是去访问定义式（这也是C++为什么需要先定义，后使用）

这个问题在Java等语言中不存在，这些语言的实现类似于**pimpl（pointer to implementation）**写法，如果一个类中有一个自定义的数据类型，我们不需要知道这个类具体有多大，我们只需要分配一个指针大小的空间，让这个指针指向这个类

*话说应该不会有人不知道implementation是实现的意思吧*

```c++
class PersonImpl;	//pimpl写法，这是Person类的前置声明
class Data;				//Data的前置声明
class Address;		//Address的前置声明

class Persion{	//像这样使用pimpl的类，往往被称为Handle classes
public:
  ...
  std::string name() const;
  ...
private:
  std::tr1::shared_ptr<PersonImpl> pImpl;
};
```

在这种设计下，`Person`就与`Data`、`Address`以及`Persons`的实现分离了，改动这些类也不会导致使用`Person`的客户重新编译，客户无法看到`Person`的实现细节，真正实现**接口与实现分离**

这个操作的本质是用**声明的依赖性**替换**定义的依赖性**

此外最好为声明式和定义式提供不同的文件，比如把声明放在一个头文件中，引用这个头文件就可以快速引入多个声明

此外还有另一种制作`Handle class`的方法，就是令`Person`成为一个特殊的抽象基类，称为`Interface class`，这个类没有成员变量，没有构造函数，单纯描述了几个纯虚函数和一个virtual析构函数。从功能上很接近C#、Java的Interfaces，但是更有弹性（比如可以在其中实现成员变量和成员函数）

```c++
class Person{		//Interface class
public:
  virtual ~Person();
  virtual std::string name() const = 0;
  ...
};
class Person{		//具现化
public:
  static std::tr1::shared_ptr<Person> create(const std::string& name...);
  ...
};
...
//使用
std::tr1::shared_ptr<Person> pp(Person::create(name...));
std::cout << pp->name();
```

```c++
class RealPerson: public Person{	
public:
  RealPerson(const std::string& name, ...): theName(name), ...{}
  virtual ~RealPerson() {}
  std::string name() const;		
  ...
private:
	std::string theName;
};
std::string ReakPerson::name(){...}
std::tr1::shared_ptr<Person> Person::create(const std::string& name, ...){
  retrun std::tr1::shared_ptr<Person>(new RealPerson(name, ...));
}
```

## 六：继承与面向对象

- `is-a`：是一个
- `has-a`：有一个
- `is-implemented-in-terms-of`：根据xx实现出

### public继承是is-a关系

```c++
class Student: public Person{...};	//Student is a Person
```

每个学生都是人，但每个人不一定是学生，人这个概念更一般化，学生这个概念更特殊化

**public继承下，可以把子类当父类用**，毕竟需要人的地方绝对可以接受一个学生，父类的函数可以对子类使用

这就出现了一个问题，子类一定要`is a`父类，不然会出现问题

错误的继承：

- 企鹅是鸟（企鹅是鸟的派生类），鸟会飞（其实这句话是错的），所以企鹅会飞？
- 正方形是矩形的特例，矩形可以自由调整长宽，所以正方形也可以自由调整长宽？

### 避免遮掩父类成员

```c++
int x;
void Fun(){
    double x;
    ...
}
```

由于**作用域**的**名称遮掩规则**，函数内部的local变量x覆盖了全局变量x

#### 子类名称会遮掩父类名称，在public继承下是错误的

在OOP中，如果子类重载了父类的`non-virtual`函数，就意味着子类使用同名函数遮掩了父类函数，就意味着**这个父类函数没有被子类继承！**，那么在这种情况下，继承就不是`is-a`关系了

**在public继承下，子类继承了父类的一切**

```c++
class Base{
public:
    virtual void f1() = 0;
    virtual void f1(int);
    void f2();
    void f2(double);
    ...
private:
    int x;
};
class Derived: public Base{
public:
    virtual void f1();
    void f2();
};
...
Derived d;
int x;
d.f1();		//正确，调用Derived::f1
d.f1(x);	//错误，因为Derived::f1遮掩了Base::f1，而Derived::f1中没有f1(int)
d.f2();		//正确，调用Derived::f2
d.f2(x);	//错误，因为Derived::f2遮掩了Base::f2，而Derived::f2中没有f2(double)
```

#### 将被遮掩的名称重见天日

解决起来很简单，只需要让父类的函数在子类作用域内可见，可以**使用using关键字**

```c++
class Derived: public Base{
public:
    using Base::f1;
    using Base::f2;
    virtual void f1();
    void f2();
};
...
Derived d;
int x;
d.f1();		//正确，调用Derived::f1
d.f1(x);	//正确，调用Base::f1
d.f2();		//正确，调用Derived::f2
d.f2(x);	//正确，调用Base::f2
```

如果是private继承，子类只继承了父类的一部分，如果子类只想要父类的某一个函数，可以**使用转交函数**，这对象的作用就是不使用using关键字，实现让父类函数出现在子类作用域中

```c++
class Derived: private Base{
public:
    virtual void f1(){
        Base::f1();		//inline转交函数
    }
    ...
};
...
Derived d;
int x;
d.f1();		//正确，调用Derived::f1
d.f1(x);	//错误，因为Derived::f1遮掩了Base::f1
```

### 区分接口继承和实现继承

public继承分为两个部分

- 函数接口继承
- 函数实现继承

|                 | 接口继承 | 实现继承         |
| --------------- | -------- | ---------------- |
| 纯虚函数        | 具体指定 | 不继承           |
| 非纯虚函数      | 具体指定 | 继承一份缺省实现 |
| non-virtual函数 | 具体指定 | 继承一份强制实现 |

### 考虑使用virtual以外的选择

#### 基于NVI的Template Method模式

Non-Virtual Interface（NVI）流派主张virtual函数应该为private类型，让客户使用public non-virtual成员函数间接调用virtual函数

```c++
class GameCharacter{
public：
    int healthValue() const
	{
    	...
    	int retVal = doHealthValue();
    	...
    	return retVal;
	}
private：
    virtual int doHealthValue() const
    {
		...
    }
};
```

其中`healthValue()`被称为virtual函数的**外覆器（wrapper）**

#### 基于函数指针的Strategy模式

```c++
class GameCharacter;
int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter{
public:
    typedef int (*HealthCalcFunc)(const GameCharacter&);
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc): healthFunc(hcf){}
    int healthValue() const { return healthFunc(*this); }
    ...
private:
    HealthCalcFunc healthFunc;
};
```

在这种模式下，`defaultHealthCalc`函数不再是`GameCharacter`体系内的成员函数，通过修改函数指针，就可以让`GameCharacter`使用不同种类的计算函数，弹性更强，而且可以在运行时变更

此外`defaultHealthCalc`函数不需要/不能访问`GameCharacter`内的`non-public`部分，

####  基于tr1::function的Strategy模式

上面使用函数指针，是为了将函数变成某个类似于函数的东西，比如函数指针，比如`tr1::function`对象

```c++
class GameCharacter;
int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter{
public:
    typedef std::tr1::function<int (const GameCharacter&)> HealthCalcFunc;
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc): healthFunc(hcf){}
    int healthValue() const { return healthFunc(*this); }
    ...
private:
    HealthCalcFunc healthFunc;
};
```

#### 古典的Strategy模式

```c++
class GameCharacter;
class HealthCalcFunc{
public:
    ...
    virtual int calc(const GameCharacter& gc) const{...}
    ...
};
HealthCalcFunc defaultHealthCalc;
class GameCharacter{
public:
    explicit GameCharacter(HealthCalcFunc* phcf = &defaultHealthCalc): pHealthFunc(phcf){}
    int healthValue() const { return phealthFunc->calc(*this); }
    ...
private:
    HealthCalcFunc* pHealthFunc;
};
```

### 绝不重新定义继承而来的non-virtual函数

- 静态绑定（staticcally bound）：non-virtual就是这种
- 动态绑定（dynamically bound）：virtual就是这种

```c++
class B{
public:
	void f();
	...
};
class D: public B{
public:
  void f();
  ...
};
...
D x;
B* pB = &x;
D* pD = &x;
pB->f();	//调用B::f
pD->f();	//调用D::f
```

### 绝对不重新定义继承而来的缺省参数值

**virtual函数是动态绑定的，缺省参数值是静态绑定的**

```c++
class Cricle: public Shape{...};
...
Shape* p1;	//p1的静态类型是Shape*，没有动态类型
Shape* p2 = new Circle;	//p2的静态类型是Shape*，动态类型是Circle*
```

- 静态类型
  - 指针的类型就是**静态类型**
- 动态类型
  - 所指向的对象的类型是**动态类型**
  - 动态类型可以通过赋值等操作改变

virtual函数也是动态绑定的，具体调用哪一个函数取决于发出调用的对象的动态类型，所以允许重载

但缺省参数值是静态绑定的，如果重载一个含有缺省参数值的virtual函数，有可能会导致使用父类的缺省参数值，调用子类的函数

### has-a和根据xx实现出

一个类中有多个小类，这种关系被称为**复合（composition）**，其中这些小类被称为**合成成分物（composed object）**

- 在应用域，复合意味着`has-a`
  - 人有名字（也不尽然）
- 在实现域，复合意味着`is-implemented-in-terms-of`
  - 队列是由数组实现的（有的队列中维护了一个数组，当然不是所有的队列都使用数组实现）

### 少用private继承

*经典 C++糟粕，请问 C#有这个吗？*

本质上是一种`is-implemented-in-terms-of`关系，父类和子类间并没有逻辑上的联系，仅仅是想用父类的某些特性来实现子类，这东西在设计层面完全没有意义，纯粹是一种实现技术

- private继承，编译器无法自动将子类对象转化为父类对象
- private继承，父类中所有属性变成private类型（比如父类中的public、protected类型）

**尽量使用复合来替代pirvate继承**，除非你想让子类可以访问父类protected成员，或者需要诚信定义virtual函数

此外private继承的对象有可能比复合的对象要小

### 少用多重继承

*经典 C++糟粕，请问 C#有这个吗？*

- 可能会导致歧义
  - 当然你可以在调用函数的时候指出是来自哪一个基类
- 可能会导致菱形继承
  - 菱形继承可能会导致变量重复

## 七：模版与泛型

模板（templates）是泛型编程（generic programming）的基础

模板机制是一个完整的图灵机（Turing-complete），引出了模板元编程（template metaprogramming， TMP），在编译时TMP从templates中具现出若干C++代码，这些代码会被编译期正常编译

### 评价

优点：

1. 模板编程能够实现非常灵活且类型安全的接口
2. 极好的性能（更小的文件、更短的运行期，更少的内存需求）
3. 可以将一些运行时才能侦测到的错误，在编译期找出来

缺点：

1. 难以编程和维护
2. 编译报错信息难以理解
3. 难以重构
4. 编译时间大幅变长

因此C++模板一般只用在少量高频使用的基础组件，不要写太复杂的，也不将模板暴露出去（用户不会用，就不给他们用），写好注释

### 隐式接口和编译期多态

- OOP中经常使用显式接口和运行时多态
- 泛型编程更多使用隐式接口和编译期多态

```C++
template<typename T>
void doProcessing(T& w)
{
    if(w.size() > 10 && w != someNastyWidget){
        T temp(w)
        temp.normalize();
        temp.swap(w);
    }
}
```

从这个例子来看，t的类型应该必须支持size、normalize、swap等函数，这些函数就是一组**隐式接口**

所有涉及t的函数调用，都有可能造成template的具现化（instantiated），使得这些调用能够成功

这种具现化行为出现在编译期，”以不同的template参数具现化function template“会导致调用不同的函数，这就是**编译期多态**

### Traits

一种约定俗成的技术方案，为同一类数据提供统一的操作函数

比如我们想实现一个通用的decode()，我们不可能每自定义一个类，就重载一次函数，我们可以使用模板来实现。

```C++
enum Type{
    TYPE_1;
    TYPE_2;
};
class FOO{
    Type type = Type::TYPE_1;
};
class Bar{
    Type type = Type::TYPE_2;
};
//统一的模板函数
template<typename T>
void decode(const T& data, char* buf){
    if(T::type == Type::TYPE_1){
        ...
    }
    else if(T::type == Type::TYPE_2){
        ...
    }
}
```

但是对于系统内置的变量，我们无法对其进行修改，于是我们引入了traits技术

```C++
enum Type{
    TYPE_1;
    TYPE_2;
};
class FOO{
    Type type = Type::TYPE_1;
};
class Bar{
    Type type = Type::TYPE_2;
};

template<typename T>
struct type_traits{
    Type type = T::type;
}
//为内置数据类型特化为独有的 type_traits
template<typename int>
struct type_traits{
    Type type = Type::TYPE_1;
}
//统一的模板函数
template<typename T>
void decode(const T& data, char* buf){
    if(type_traits<T>::type == Type::TYPE_1){
        ...
    }
    else if(type_traits<T>::type == Type::TYPE_2){
        ...
    }
}
```

该技术使得类型测试在编译期可用，将类型测试放在编译期，可以使得测试代码不进入可执行文件中，这也就是将类型测试的工作量从运行时转到编译期，这也就是为什么TMP能以牺牲编译时长为代价，提高代码运行效率的原因

### 模板元编程

TMP是一个函数式语言，这类语言经常使用递归。函数式语言的递归不涉及函数调用，而是递归模板具现化

如果一门语言具备以下功能，则称为图灵完全

1. 数值运算和符号运算
2. 判断
3. 递归

#### 数值运算+递归

```C++
//一个TMP计算阶乘，而且阶乘的技术发生在编译期
template<unsigned n>
struct Factorial
{
    enum { value = n * Factorial<n-1>::value };
};
template<>
struct Factorial<0>
{
    enum { value = 1 };
};

int main()
{
    std::cout << Factorial<5>::value;
}
```

C++11TMP这种函数式编程得到了加强，上文也可以这样写

```C++
template<unsigned n>
struct Factorial
{
    constexpr static auto value{ n * Factorial<n - 1>::value };
};
template<>
struct Factorial<0>
{
    constexpr static auto value = 1;
};
```

#### 判断

```C++
template<bool Value>
struct if_constexpr
{
    constexpr static auto value = 1;
};

template<>
struct if_constexpr<false> {
    constexpr static auto value = 2;
};

int main()
{
    std::cout << if_constexpr<true>::value << std::endl;
    std::cout << if_constexpr<false>::value << std::endl;
}
```

### typedef

在泛型编程中，typedef也很常用，他的作用很多，其中有一个是为复杂声明定义一个简单的别名

下面是一个函数指针的示例

```C++
void add(int x, int y) {
    std::cout << "x+y=" << x + y << std::endl;
}
void dec(int x, int y) {
    std::cout << "x-y=" << x - y << std::endl;
}
void mul(int x, int y) {
    std::cout << "x*y=" << x*y << std::endl;
}

void (*op[3])(int, int) = { add, dec, mul };

int main()
{
    for (int i = 0; i < 3; ++i) {
        （*op[i])(4, 3);
    }
}
```

如果使用typedef

```C++
typedef void (*Func[3])(int, int);
Func f = { add, dec, mul };

int main()
{
    for (int i = 0; i < 3; ++i) {
        f[i](4, 3);
    }
}
```

## 八：定制new和delete

Java和C#等语言有自己内置的GC，但C++必须手动管理内存，这虽然麻烦，但是值得，尤其是一些设计苛刻的项目

### new-handler

当`operator new`无法分配内存时，会抛异常。在其抛异常前，会调用一个错误处理函数来处理内存不足的问题，即`new-handler`

使用`set_new_handler`来指定`new-handler`

```c++
void outOfMem(){
  std::cerr << "内存不足\n";
  std::abort();
}
int main(){
  std::set_new_handler(outOfMem);		//该函数的参数是一个函数指针
  int* array = new int[10000000L];
  ...
}
```

当`operator new`无法满足内存申请时，会不断调用`new-handler`函数，直到找到足够的内存，所以`new-handler`函数应该满足

- 让更多的内存可被使用
  - 实现方法是程序开始时就分配一大块内存，每次调用`new-handler`时就释放一点点
- 安装另一个`new-handler`
  - 如果现在这个`new-handler`无法获取更多内存，需要知道哪一个`new-handler`具备增大内存的实力，然后使用`set_new_handler`来替换自己
- 卸除`new-handler`
  - 通过`set_new_handler`赋值`null`，将`new-handler`卸载，使得在内存分配不足时，会抛异常
- 抛出`bad_alloc`异常
  - 这种异常不会被`operator new`捕获，会被传播至内存索求处
- 不反回
  - 调用`abort`或者`exit`

```c++
class NewHandlerHolder{
public:
  explicit NewHandlerHolder(std::new_handler nh): handler(nh) {}	//获取当前的new_handler
  ~NewHandlerHolder() { std::set_new_handler(handler); }	
private:
  std::new_handler handler;		//用于记录当前的new_handler
};
void* Widget::operator new(std::size_t size) throw(std::bad_alloc){
  NewHandlerHolder h(std::set_new_handler(currentHandler));	//安装Widget的new-handler
  return ::operator new(size);	//分配对象或者抛异常
}
//离开这个函数的声明周期时，NewHandlerHolder被析构，new-handler恢复之前的值
```

```c++
void outOfMem();
Widget::set_new_handler(outOfMem);
Widget* pwl = new Widget;	//内存不足时会调用outOfMem
```

mixin风格的写法

```c++
template<typename T>
class NewHandlerSupport{
public:
  static std::new_handler_set set_new_handler(std::new_handler p) throw();
  static void* operator new(std::size_t size) throw(std::bad_alloc);
  ...
private:
  static std::new_handler currentHandler;
};

template<typename T>
std::new_handler NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw(){
  std::new_handler oldHandler = currentHandler;
  currentHandler = p;
  return oldHandler;
}

template<typename T>
void* NewHandlerSupport<T>::operator new(std::size_t size) throw(std::bad_alloc){
  NewHandlerHolder h(std::set_new_handler(currentHandler));
  return ::operator new(size);
}
```

```c++
class Widget: public NewHandlerSupport<Widget>{
	...
};
```

像这样，一个类继承于一个模版基类，而且这个模版基类以这个类作为类型参数，被称为**怪异的循环模版模式（curiously recurring template pattern，CRTP）**

### 替换new和delete的时机

C++中所有的news返回的指针都必须要**地址对齐**，int要4对齐，double要8对齐

写一个好的new很难，只有当你想改善效能、对heap运行作物进行调试、收集heap使用信息等时才对其进行替换

### 编写new和delete的规则

如果你真的需要自己写一个new/delete，那就写吧，只不过要符合一些规则

- new
  - 应该内含一个无穷循环，在其中尝试分配内存，点那个无法满足内存需求时，调用`new-handler`
  - 有能力处理0 bytes申请（比如将0 bytes申请视为1 bytes申请）
  - new可能会被继承，而派生类的大小可能会比基类大，需要对其做处理（比如改用标准new），即处理**比正确大小更大的（错误）申请**

- delete
  - 收到null指针时不做任何事
  - 处理**比正确大小更大的（错误）申请**

### 编写new时也要写对应的delete

```c++
Widget* pw = new Widget;
```

在这里调用了两个函数，一个时用以分配内存的`operator new`，一个是`Widget`的构造函数

如果构造函数调用异常，pw将不会被赋值，客户手中将不会有指针指向之前分配的内存。但如果不释放那个内存，就会导致内存泄漏。所以释放内存是交给C++运行时系统的

运行时系统会调用`operator new`所对应的`operator delete`来释放地址，对于拥有正常签名式的new和delete来说不成问题

```c++
void* operator new(std::size_t) throw(std::bad_alloc);	//普通的new
void operator delete(void* rawMemory) throw();	//global中的普通的new
void operator delete(void* rawMemory, std::size_t size) throw();	//class中的new
```

但当你自定义了一个new，却同时写了一个普通形式的delete，就会出现问题

```c++
void* operator new(std::size_t, void* pMemory) throw();	//placement new，比普通new多带一个参数

Widget* pw = new (std::cerr) Widget;	//调用operator new，并以cerr作为其实参
```

当内存分配成功，而构造函数出现异常时，运行时系统有责任取消内存分配，并恢复旧观，但现在运行时系统无法知道真正被调用的`operator new`时如何运作的，所以运行时系统会去寻找**参数个数与类型**都与`operator new`相同的某个`operator delete`

```c++
void operator delete(void*, std::ostream&) throw();	//palcement delete
```

```c++
class Widget{
public:
  static void* operator new(std::size_t size, std::ostream& logStream) throw(std::bad_alloc);
  static void operator delete(void* pMemory) throw();
  static void operator delete(void* pMemoty, std::ostream& logStream) throw();
  ...
};
```

如果此时调用`delete pw`，只会调用普通的`delete`，因为只有在构造时发生异常时，运行时系统才会调用placement delete

最简单的方式是建立一个base class，令其包含所有正常形式的new和delete，然后继承这个基类，使用using表达式，再扩充new和delete

## 九：杂项

### 不要忽视编译器警告

很多人忽视警告，毕竟一个问题如果真的很严重，应该报错

比如下面这个错误，虽然只会报一个警告，但会导致错误的程序行为

```c++
class B{
public:
  virtual void f() const;
};
class D: public B{
  virtual void f();
};
```

报警告

```c++
warning: D::f() hides virtual B::f()
```

原本的目的是为了在D中重新定义virtual函数`f()`，但由于B中`f()`是const，在D中不是，此时B中的`f()`并没有在D中重新被声明，而是被整个遮掩了

### 去熟悉标准程序库

尤其是TR1

#### C++98有什么

- STL、容器（container）、迭代器（iterator）、算法（algorithm）、函数对象（function object）、容器适配器、函数对象适配器
- Iostream
- 国际化支持
- 数值处理，包括复数（complex）和纯数值数组（valarray）
- 异常阶层体系
- C89标准程序库

#### TR1有什么（全在`std::tr1`中）

- 智能指针`tr1::shared_ptr`和`tr1::weak_ptr`
- `tr1::function`
- `tr1::bind`

和（彼此无关的独立组件）

- 哈希表
- 正则表达式
- Tuple变量组
- `tr1::array`
- `tr1::mem_fn`
- `tr1::reference_wrapper`
- 随机数生成工具
- 数学特殊函数
- C99兼容

和（基于template）

- Type traits
- `tr1::result_of`

### 熟悉Boost









