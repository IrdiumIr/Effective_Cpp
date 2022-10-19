# Section 1 让自己习惯C++

(Accustoming Yourself to C++)
---

***

## 01：视C++ 为一个语言联邦

* 最初的`C++`是`C`加上一些面向对象特性，即`C with Classes`
* 但是现代的`C++`已经是一个多重范式编程语言，包括了*过程形式、面向对象形式、函数形式、泛型形式、元编程形式*等多种范式的语言
* 因此可以将`C++`视为某几个相关语言组成的联邦而非单一语言；而在其中某个次级语言中，规则是相对简单的
  * 1.  以`C`为基础的语句、预处理器、内置类型、数组、指针等……
  * 2.  `Object-Oriented C++`增加了`C with Classes`的类、封装继承多态等特性、还有动态绑定等……
  * 3.  `Template C++`增加了面向各类类型无关的算法代码复用方式，其中也包括了全新的*模板元编程TMP*
  * 4.  `STL`是标准的特殊`template`程序库，包括了`C++`经常使用的容器、迭代器、算法等……
* 在不同语言中，相同的操作可能会带来不同的效率（例如*pass-by-value*和*pass-by-reference*）

### Conclusion：
* C++高效编程守则视状况而变化，取决于使用哪个子语言


## 02：尽量以const, enum, inline替换 #define

* 更直白的表述是，应该让编译器代替预处理器定义。因为预处理器定义的变量并没有进入到`symbol table`里面，导致编译器看不到预处理器定义
* 例如我们应该用
```cpp
    const double AspectRation = 1.653;
```
* 以替代
```cpp
    #define ASPECT_RATIO 1.653
```

***

* 除此之外，当定义一个常量指针（放在`.h`文件内），有必要将指针本身定义为`const`的：
```cpp
    const char* const authorName = "Scott Meyers";
```
* 当然，此时使用`std::string`更好一些

***

* 类内的成员常量，为了保证其只有一份实体，通常还需要定义为`static`的。但是其通常只是一个声明式：
```cpp
    class GamePlayer {
    private:
        static const int Numturns = 5;
        int scroes[NumTurns];
        ...
    };
```
* 但是`NumTurns`仅是一个声明式而非定义式（但当我们不取其地址时可以这样做）
* 除了上述特殊的情况，我们实际上需要提供一个成员常量的定义式，并将其放在实现文件而非头文件中，例如：
```cpp
    const int GamePlayer::NumTurns;
```
* 基于我们对`#define`的认知，其并不重视作用域，因此不能够提供类专属常量或者任何封装性
* 如果当编译器不支持此类常量在类内完成初值设定，可以改用`enum hack`补偿做法：
```cpp
    class GamePlayer {
    private:
        enum { NumTurns = 5 };
        int scroes[NumTurns];
        ...
    };
```
* `enum hack`和`#define`的行为较为接近，不能对其取地址，也不会造成非必要的内存分配

***

* 对于类似函数的宏，最好改用`inline`函数代替：
```cpp
    #define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))

    template<typename T>
    inline void callWithMax(const T& a, const T& b) {
        f(a > b ? a : b);
    }
```

### Conclusion：
* 对于单纯的常量，最好以`const`对象或`enums`替换`#define`
* 对于形似函数的宏(macros)，最好改用`inline`函数替换`#define`


## 03：尽可能使用const

* `const`可以使得编译器强制执行一个“*不应该被改动*”的语义约束
* `const`出现在`*`左侧，表示被指物是常量；如果出现在`*`右侧，表示指针本身是常量；如果同时出现在两侧，表示被指物和指针都是常量
* 当`const`在左侧时，指针指向的对象是一个常量，此时关键词`const`出现在类型之前还是类型之后的意义相同
```cpp
    void f1(const Widget* pw);
    void f2(Widget const * pw);
```
* 迭代器的`const`用法和一个形如`T*`的指针类似：声明一个为`const`的迭代器就好像是声明一个`T* const`的指针；而要表示一个常量指针，则应该使用`const_interator`
* `const`最强的用法是在**函数声明**时，如果将返回值/返回的函数指针设置成`const`，可以避免很多用户错误造成的意外：
```cpp
    class Rational {};
    const Rational operator* (...)

    Rational a, b, c;
    (a * b) = c;
```
* 形如这种错误对于内置类型显然是不合法的，而我们想要定义良好的自定义类型，通常希望其避免与内置类型产生不兼容的行为

***

* `const`成员函数能够有效的使成员函数作用于`const`对象身上；不仅使得类接口易于理解，同时还使得改善`C++`程序效率的一个好办法（以`pass by ref-to-const`的方式传递对象）得以实现
* 两个成员函数如果只是常量性质(*constness*)不同，是可以被重载的：例如封装了一个`string`的类返回其`pos[i]`的字符

* `bitwise constness`和`logical constness`之争：
  * `bitwise constness`要求对对象的任何成员不做任何更改(any bit)
    ```cpp
    class CTextBlock {
        public:
            char& operator[](std::size_t position) const {
                return pText[position];
            }
        private:
            char *pText;
    }
    const CTextBlock cctb("Hello");
    char *pc = &cctb[0];
    *pc = 'J'
    ```
    * 但是显然对于这样一个`const`成员函数，虽然没有更改指针本身，但是只有指针本身属于对象，其所指物依然可以在外部被改变
    
    ***

  * `logical constness`认可一个`const`成员函数可以修改对象内的某些部分，但是需要编译器不能识别出来
    ```cpp
    class cTextBlock {
        public:
            std::size_t length() const;
        private:
            char* pText;
            //  important
            mutable std::size_t textLength;
            mutable bool lengthIsValid;
    };
    std::size_t cTextBlock::length() const {
        if (!lengthIsValid) {
            textLength = std::strlen(pText);
            lengthIsValid = true;
        }
        return textLength;
    }
    ```
    * 使用一个`mutable`关键字，使得成员变量始终处于可变的状态，其解除了`bitwise constness`约束

***

* 但是为了解决`bitwise constness`问题，`mutable`并不能解决所有情况：很多时候往往会写出两个内容几乎一致但是只是`constness`有所不同的代码（特别是在单个函数内容很多的情况下）
* 如果实现`operator[]`的功能，并使一个函数*调用*另一个，则能够大大减少代码量：即使`non-const-operator[]`调用其`const`版本
```cpp
    class TextBlock {
        public:
            const char& operator[](std::size_t pos) const {}

            char& operator[](std::size_t pos) {
                return 
                    const_cast<char&>(
                        static_cast<const TextBlock&>(*this)
                            [pos]
                    );
            }
    };
```
* 上述的代码先将`non-const`的对象`*this`添加了`const`属性，使其能够调用`const`版本；并将结果返回的`const`属性去除
* 但是值得注意的是，反向做法——将`const`版本调用`non-const`版本是一个错误的做法：对象有可能会被改动（且你需要先将`*this`对象上的`const`属性去除，这是危险行为！）

### Conclusion：
* 将某些东西声明为`const`可以帮助编译器检查出错误
* 编译器强制实施`bitwise constness`，但是个人编写程序的时候应该考虑其概念上的常量性
* 当`const`和`non-const`版本有实质等价的实现时，令`non-const`版本调用`const`版本将避免代码重复


## 04：确定对象被使用前已先被初始化

* todo