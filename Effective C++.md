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











