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

**RCSP（引用计数型智能指针）**也是一种智能指针（比如`trl::shared_ptr`），会持续跟踪有多少对象指向某个资源，只有这个资源无人指向时，才会删除该资源

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





