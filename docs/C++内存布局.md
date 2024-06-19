Visual Studio：先选择左侧的C/C++->命令行，然后在其他选项这里写上/d1 reportAllClassLayout，它可以看到所有相关类的内存布局，如果写上/d1 reportSingleClassLayoutXXX（XXX为类名），则只会打出指定类XXX的内存布局
# 一、无继承

## 1.1 无虚函数
```
class Base
{
private:
	
	char c_a;
	
	static char c_b;
	
	int i_a;
	
	static int i_b;
	
	float f_a;
	
	static float f_b;
	
	double d_a;
	
public:
	
	void fun_1() {}

};
```

### 布局：
```
1>class Base size(24):

1> +---

1> 0 | c_a

1>   | <alignment member> (size=3)

1> 4 | i_a

1> 8 | f_a

1>   | <alignment member> (size=4)

1>16 | d_a

1> +---
```

* 普通的变量 ：是要占用内存的，但是要注意对齐原则

* static修饰的静态变量 ：不占用内容，原因是编译器将其放在全局变量区

* 成员函数不占用具体类对象内存空间，成员函数存在代码区

* 数据成员的访问级别并不影响其在内存的排布和大小，均是按照声明的顺序在内存中有序排布，并适当对齐

## 1.2 有虚函数
```
class Base

{
private:
	
	char c_a;
	
	static char c_b;
	
	int i_a;
	
	static int i_b;
	
	float f_a;
	
	static float f_b;
	
	double d_a;
	
public:
	
	void fun_1() {}
	
	virtual void vfun_1() {}
	
	virtual void vfun_2() {}

};
```


### 布局：
```
1>class Base size(32):

1> +---

1> 0 | {vfptr}

1> 8 | c_a

1>   | <alignment member> (size=3)

1>12 | i_a

1>16 | f_a

1>   | <alignment member> (size=4)

1>24 | d_a

1> +---

1>Base::$vftable@:

1> | &Base_meta

1> |  0

1> 0 | &Base::vfun_1

1> 1 | &Base::vfun_2

1>Base::vfun_1 this adjustor: 0

1>Base::vfun_2 this adjustor: 0

```


# 二、单一继承

## 2.1 无虚函数
```
class Base

{

private:
	
	char c_a;
	
	static char c_b;
	
	int i_a;
	
	static int i_b;
	
	float f_a;
	
	static float f_b;
	
	double d_a;
	
public:
	
	void fun_1() {}

};

//Derived1类
class Derived1 : public Base
{

	char c_b;

public:

	void fun_d1() {}

};

//Derived2类
class Derived2 : public Derived1
{

	double d_b;

public:

	void fun_d2() {}

};
```

### 布局：
```
1>class Base size(24):

1> +---

1> 0 | c_a

1>   | <alignment member> (size=3)

1> 4 | i_a

1> 8 | f_a

1>   | <alignment member> (size=4)

1>16 | d_a

1> +---

1>class Derived1 size(32):

1> +---

1> 0 | +--- (base class Base)

1> 0 | | c_a

1>   | | <alignment member> (size=3)

1> 4 | | i_a

1> 8 | | f_a

1>   | | <alignment member> (size=4)

1>16 | | d_a

1> | +---

1>24 | c_b

1>   | <alignment member> (size=7)

1> +---

1>class Derived2 size(40):

1> +---

1> 0 | +--- (base class Derived1)

1> 0 | | +--- (base class Base)

1> 0 | | | c_a

1>   | | | <alignment member> (size=3)

1> 4 | | | i_a

1> 8 | | | f_a

1>   | | | <alignment member> (size=4)

1>16 | | | d_a

1> | | +---

1>24 | | c_b

1>   | | <alignment member> (size=7)

1> | +---

1>32 | d_b

1> +---
```

* 每个派生类中起始位置都是**Base class subobjectj基类子对象**

* 内存空间会按照类的继承顺序(父类到子类)和字段的声明顺序布局
## 2.2 有虚函数
```
class Base
{
private:
	char c_a;
	static char c_b;
	int i_a;
	static int i_b;
	float f_a;
	static float f_b;
	double d_a;
public:
	virtual void fun_1() {}
	virtual void fun_2() {}
};

//Derived1类
class Derived1 : public Base
{
	char c_b;

public:
	virtual void fun_d1() {}
	void fun_2() {}
};

//Derived2类
class Derived2 : public Derived1
{
	double d_b;

public:
	virtual void fun_d2() {}
};
```
### 布局
```
1>class Base	size(32):
1>	+---
1> 0	| {vfptr}
1> 8	| c_a
1>  	| <alignment member> (size=3)
1>12	| i_a
1>16	| f_a
1>  	| <alignment member> (size=4)
1>24	| d_a
1>	+---
1>Base::$vftable@:
1>	| &Base_meta
1>	|  0
1> 0	| &Base::fun_1
1> 1	| &Base::fun_2
1>Base::fun_1 this adjustor: 0
1>Base::fun_2 this adjustor: 0
1>class Derived1	size(40):
1>	+---
1> 0	| +--- (base class Base)
1> 0	| | {vfptr}
1> 8	| | c_a
1>  	| | <alignment member> (size=3)
1>12	| | i_a
1>16	| | f_a
1>  	| | <alignment member> (size=4)
1>24	| | d_a
1>	| +---
1>32	| c_b
1>  	| <alignment member> (size=7)
1>	+---
1>Derived1::$vftable@:
1>	| &Derived1_meta
1>	|  0
1> 0	| &Base::fun_1
1> 1	| &Derived1::fun_2
1> 2	| &Derived1::fun_d1
1>Derived1::fun_d1 this adjustor: 0
1>Derived1::fun_2 this adjustor: 0
1>class Derived2	size(48):
1>	+---
1> 0	| +--- (base class Derived1)
1> 0	| | +--- (base class Base)
1> 0	| | | {vfptr}
1> 8	| | | c_a
1>  	| | | <alignment member> (size=3)
1>12	| | | i_a
1>16	| | | f_a
1>  	| | | <alignment member> (size=4)
1>24	| | | d_a
1>	| | +---
1>32	| | c_b
1>  	| | <alignment member> (size=7)
1>	| +---
1>40	| d_b
1>	+---
1>Derived2::$vftable@:
1>	| &Derived2_meta
1>	|  0
1> 0	| &Base::fun_1
1> 1	| &Derived1::fun_2
1> 2	| &Derived1::fun_d1
1> 3	| &Derived2::fun_d2
```
### 三、多重继承
```
class A
{
private:
	char c_a;
	static char c_b;
	int i_a;
	static int i_b;
	float f_a;
	static float f_b;
	double d_a;
public:
	void fun_A() {}
	virtual void vfun_A() {}
	virtual ~A() {}
};

class B
{
	char c_b;

public:
	void fun_B() {}
	virtual void vfun_B() {}
	virtual ~B() {}
};

class C : public A, public B
{
	double d_b;

public:
	void fun_C() {}
	virtual void vfun_C() {}
	virtual ~C() {}
};
```
## 布局
```
1>class A	size(32):
1>	+---
1> 0	| {vfptr}
1> 8	| c_a
1>  	| <alignment member> (size=3)
1>12	| i_a
1>16	| f_a
1>  	| <alignment member> (size=4)
1>24	| d_a
1>	+---
1>A::$vftable@:
1>	| &A_meta
1>	|  0
1> 0	| &A::vfun_A
1> 1	| &A::{dtor}
1>A::vfun_A this adjustor: 0
1>A::{dtor} this adjustor: 0
1>A::__delDtor this adjustor: 0
1>A::__vecDelDtor this adjustor: 0
1>class B	size(16):
1>	+---
1> 0	| {vfptr}
1> 8	| c_b
1>  	| <alignment member> (size=7)
1>	+---
1>B::$vftable@:
1>	| &B_meta
1>	|  0
1> 0	| &B::vfun_B
1> 1	| &B::{dtor}
1>B::vfun_B this adjustor: 0
1>B::{dtor} this adjustor: 0
1>B::__delDtor this adjustor: 0
1>B::__vecDelDtor this adjustor: 0
1>class C	size(56):
1>	+---
1> 0	| +--- (base class A)
1> 0	| | {vfptr}
1> 8	| | c_a
1>  	| | <alignment member> (size=3)
1>12	| | i_a
1>16	| | f_a
1>  	| | <alignment member> (size=4)
1>24	| | d_a
1>	| +---
1>32	| +--- (base class B)
1>32	| | {vfptr}
1>40	| | c_b
1>  	| | <alignment member> (size=7)
1>	| +---
1>48	| d_b
1>	+---
1>C::$vftable@A@:
1>	| &C_meta
1>	|  0
1> 0	| &A::vfun_A
1> 1	| &C::{dtor}
1> 2	| &C::vfun_C
1>C::$vftable@B@:
1>	| -32
1> 0	| &B::vfun_B
1> 1	| &thunk: this-=32; goto C::{dtor}
1>C::vfun_C this adjustor: 0
1>C::{dtor} this adjustor: 0
1>C::__delDtor this adjustor: 0
1>C::__vecDelDtor this adjustor: 0
```
* 每个包含虚函数的父类都会有自己的虚函数表，并且按照继承顺序布局(虚表指针+字段）；
* 如果子类重写父类虚函数，都会在每一个相应的虚函数表中更新相应地址；
* 如果子类有自己的新定义的虚函数或者非虚成员函数，也会加到第一个虚函数表的后面。
# 四、菱形继承

菱形继承：类B和C均继承自类A，类D同时继承类B和C
```
class A
{
private:
	char c_a;
	int i_a;
	float f_a;
	double d_a;
public:
	virtual ~A() {}
};

class B : public A
{
	int i_b;

public:
	virtual ~B() {}
};

class C : public A
{
	int i_c;

public:
	virtual ~C() {}
};

class D : public B, public C
{
	int i_d;

public:
	virtual ~D() {}
};
```
## 布局
```
1>class A	size(32):
1>	+---
1> 0	| {vfptr}
1> 8	| c_a
1>  	| <alignment member> (size=3)
1>12	| i_a
1>16	| f_a
1>  	| <alignment member> (size=4)
1>24	| d_a
1>	+---
1>A::$vftable@:
1>	| &A_meta
1>	|  0
1> 0	| &A::{dtor}
1>A::{dtor} this adjustor: 0
1>A::__delDtor this adjustor: 0
1>A::__vecDelDtor this adjustor: 0
1>class B	size(40):
1>	+---
1> 0	| +--- (base class A)
1> 0	| | {vfptr}
1> 8	| | c_a
1>  	| | <alignment member> (size=3)
1>12	| | i_a
1>16	| | f_a
1>  	| | <alignment member> (size=4)
1>24	| | d_a
1>	| +---
1>32	| i_b
1>  	| <alignment member> (size=4)
1>	+---
1>B::$vftable@:
1>	| &B_meta
1>	|  0
1> 0	| &B::{dtor}
1>B::{dtor} this adjustor: 0
1>B::__delDtor this adjustor: 0
1>B::__vecDelDtor this adjustor: 0
1>class C	size(40):
1>	+---
1> 0	| +--- (base class A)
1> 0	| | {vfptr}
1> 8	| | c_a
1>  	| | <alignment member> (size=3)
1>12	| | i_a
1>16	| | f_a
1>  	| | <alignment member> (size=4)
1>24	| | d_a
1>	| +---
1>32	| i_c
1>  	| <alignment member> (size=4)
1>	+---
1>C::$vftable@:
1>	| &C_meta
1>	|  0
1> 0	| &C::{dtor}
1>C::{dtor} this adjustor: 0
1>C::__delDtor this adjustor: 0
1>C::__vecDelDtor this adjustor: 0
1>class D	size(88):
1>	+---
1> 0	| +--- (base class B)
1> 0	| | +--- (base class A)
1> 0	| | | {vfptr}
1> 8	| | | c_a
1>  	| | | <alignment member> (size=3)
1>12	| | | i_a
1>16	| | | f_a
1>  	| | | <alignment member> (size=4)
1>24	| | | d_a
1>	| | +---
1>32	| | i_b
1>  	| | <alignment member> (size=4)
1>	| +---
1>40	| +--- (base class C)
1>40	| | +--- (base class A)
1>40	| | | {vfptr}
1>48	| | | c_a
1>  	| | | <alignment member> (size=3)
1>52	| | | i_a
1>56	| | | f_a
1>  	| | | <alignment member> (size=4)
1>64	| | | d_a
1>	| | +---
1>72	| | i_c
1>  	| | <alignment member> (size=4)
1>	| +---
1>80	| i_d
1>  	| <alignment member> (size=4)
1>	+---
1>D::$vftable@B@:
1>	| &D_meta
1>	|  0
1> 0	| &D::{dtor}
1>D::$vftable@C@:
1>	| -40
1> 0	| &thunk: this-=40; goto D::{dtor}
1>D::{dtor} this adjustor: 0
1>D::__delDtor this adjustor: 0
1>D::__vecDelDtor this adjustor: 0
```
如上class D，subobject A有两份，所以A的data member也存了两份，但实际上对于D而言，只需要有一份subobject A即够了。菱形继承不仅浪费存储空间，而且造成了数据访问的二义性。解决办法：虚继承
# 五、虚继承
```
class A
{
private:
	char c_a;
	int i_a;
	float f_a;
	double d_a;
public:
	virtual ~A() {}
};

class B : virtual public A
{
	int i_b;

public:
	virtual ~B() {}
};

class C : virtual public A
{
	int i_c;

public:
	virtual ~C() {}
};

class D : public B, public C
{
	int i_d;

public:
	virtual ~D() {}
};
```
## 布局
```
1>class A	size(32):
1>	+---
1> 0	| {vfptr}
1> 8	| c_a
1>  	| <alignment member> (size=3)
1>12	| i_a
1>16	| f_a
1>  	| <alignment member> (size=4)
1>24	| d_a
1>	+---
1>A::$vftable@:
1>	| &A_meta
1>	|  0
1> 0	| &A::{dtor}
1>A::{dtor} this adjustor: 0
1>A::__delDtor this adjustor: 0
1>A::__vecDelDtor this adjustor: 0
1>class B	size(48):
1>	+---
1> 0	| {vbptr}
1> 8	| i_b
1>  	| <alignment member> (size=4)
1>	+---
1>	+--- (virtual base A)
1>16	| {vfptr}
1>24	| c_a
1>  	| <alignment member> (size=3)
1>28	| i_a
1>32	| f_a
1>  	| <alignment member> (size=4)
1>40	| d_a
1>	+---
1>B::$vbtable@:
1> 0	| 0
1> 1	| 16 (Bd(B+0)A)
1>B::$vftable@:
1>	| -16
1> 0	| &B::{dtor}
1>B::{dtor} this adjustor: 16
1>B::__delDtor this adjustor: 16
1>B::__vecDelDtor this adjustor: 16
1>vbi:	   class  offset o.vbptr  o.vbte fVtorDisp
1>               A      16       0       4 0
1>class C	size(48):
1>	+---
1> 0	| {vbptr}
1> 8	| i_c
1>  	| <alignment member> (size=4)
1>	+---
1>	+--- (virtual base A)
1>16	| {vfptr}
1>24	| c_a
1>  	| <alignment member> (size=3)
1>28	| i_a
1>32	| f_a
1>  	| <alignment member> (size=4)
1>40	| d_a
1>	+---
1>C::$vbtable@:
1> 0	| 0
1> 1	| 16 (Cd(C+0)A)
1>C::$vftable@:
1>	| -16
1> 0	| &C::{dtor}
1>C::{dtor} this adjustor: 16
1>C::__delDtor this adjustor: 16
1>C::__vecDelDtor this adjustor: 16
1>vbi:	   class  offset o.vbptr  o.vbte fVtorDisp
1>               A      16       0       4 0
1>class D	size(72):
1>	+---
1> 0	| +--- (base class B)
1> 0	| | {vbptr}
1> 8	| | i_b
1>  	| | <alignment member> (size=4)
1>	| +---
1>16	| +--- (base class C)
1>16	| | {vbptr}
1>24	| | i_c
1>  	| | <alignment member> (size=4)
1>	| +---
1>32	| i_d
1>  	| <alignment member> (size=4)
1>	+---
1>	+--- (virtual base A)
1>40	| {vfptr}
1>48	| c_a
1>  	| <alignment member> (size=3)
1>52	| i_a
1>56	| f_a
1>  	| <alignment member> (size=4)
1>64	| d_a
1>	+---
1>D::$vbtable@B@:
1> 0	| 0
1> 1	| 40 (Dd(B+0)A)
1>D::$vbtable@C@:
1> 0	| 0
1> 1	| 24 (Dd(C+0)A)
1>D::$vftable@:
1>	| -40
1> 0	| &D::{dtor}
1>D::{dtor} this adjustor: 40
1>D::__delDtor this adjustor: 40
1>D::__vecDelDtor this adjustor: 40
1>vbi:	   class  offset o.vbptr  o.vbte fVtorDisp
1>               A      40       0       4 0
```
可以看到，class B中有两个虚指针： 第一个指向B自己的虚表（注意这里是vbptr而不是vfptr，是虚基类指针），第二个指向虚基类A的虚表 。而且， 从布局上看，class B的部分要放在前面，虚基类A的部分放在后面。在class B中虚基类A的成分相对内存起始处的偏移offset等于class B的大小（16字节）。C的内存布局和B类似。

Class如果内含一个或多个virtual base subobjects，将被分割成两部分：一个不变区域和一个共享区域。不变区域中的数据，不管后继如何衍化，总有固定的offset（从object的开头算起），所以这一部分可以直接存取。而共享区域所表现的就是virtual base class subobject。这部分数据的位置会因为每次的派生操作而发生变化，所以它们只可以被间接存取。

菱形/钻石继承，并采用了虚继承，则内存空间排列顺序为：各个父类(包含虚基类指针)、子类、公共基类(最上方的父类，包含虚函数指针)，并且各个父类不再拷贝公共基类中的数据成员。

虚继承的实现原理是，编译器在派生类的对象中添加一个指针vbptr。vbptr指的是虚基类表指针（virtual base table pointer），该指针指向了一个虚基类表（virtual table），虚基表中记录了虚基类与本类的偏移地址；通过偏移地址，这样就找到了虚基类成员，而虚继承也不用像普通多继承那样维持着公共基类（虚基类）的两份同样的拷贝，节省了存储空间。

仔细观察可以发现，`D::$vbtable@B@`中的偏移量是40，观察D的内存布局可以看到B类起始偏移量是0，而A类的偏移量是40，A相对D的偏移量是40；

同理观察 `D::$vbtable@C@`中的偏移量是24，可以看到D内存布局中C类偏移量是16，A相对C的便宜就是26。

综上验证了虚基表中记录了虚基类与本类的偏移地址