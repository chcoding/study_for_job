# STL

[API总结](http://vernlium.github.io/2019/12/29/C-STL%E5%B8%B8%E7%94%A8%E5%AE%B9%E5%99%A8API%E6%80%BB%E7%BB%93/)



## 顺序容器

常用的顺序容器：

1. vector：可变大小数组，支持快速随机访问。
2. deque：双端队列，支持快速随机访问。
3. list：双向链表，只支持双向顺序访问。
4. forward_list：单向链表，只支持单向随机访问。
5. array：固定大小数组，支持快速随机访问，不能添加删除元素。
6. string：与vector相似的容器，但专门用于保存字符，随机访问快。



### 容器使用

#### 定义和初始化

1. 当创建一个容器为另外一个容器的拷贝时，要求**容器类型和元素类型**一样。
2. 用时初始化列表，要求元素类型一样。
3. 使用迭代器拷贝一个范围时，不要求容器类型相同，新容器和原来容器的元素类型也可以不同，只要元素能转换为被初始化容器的元素即可。（例如const char *和string之间相互转换）
4. 内置数组（`int a[10]`）不能进行拷贝或者对象赋值操作，但是array容器（`array<int, 10>`）支持，但是要求元素类型和**大小**相同。

```c++
1. 用其他容器初始化
C c1(c2);
C c1 = c2;

2.初始化列表
C c{i, j, k...};
    
3.迭代器
C c{b, e};	//b和e为另外一个容器的迭代器
```



#### 赋值和swap

​	赋值要求左右两边对象具有相同的类型（**容器和元素都相同**），将右边对象的元素拷贝到左边。

​	swap用于交换两个相同类型容器的内容，

1. string容器的迭代器、引用、指针在swap后会失效
2. array容器进行swap操作会真正交换它们的元素，执行时间与元素数目成正比，**迭代器指向不变，指向的元素值发生改变**。
3. 其他容器进行swap交换，只是交换了两个数据结构，不会对任何元素进行拷贝、删除，**常数时间**内就能完成。交换前后，迭代器指向的元素值不变。



#### 关系运算符

​	除了无序关联容器外，所有容器都支持关系运算符（>,>=,<,<=），要求运算符两边的运算对象必须是相同类型的容器，且必须保存相同类型的元素。比较规则和string容易一致。



#### 操作

##### 添加元素

1. 除了forward_list（单向链表）、array（大小不可变）外，其他顺序容器都支持**push_back**。
2. list、deque和forward_list还支持**push_front**，vector和string不支持。
3. vector、string、list、deque都支持insert，insert函数总是将元素插入到迭代器所指向的**位置之前**。
4. forward_list有自己特殊版本的insert和emplace。

​	

​	**向一个vector、deque、string插入元素会使得所有指向容器的迭代器、引用和指针失效，添加元素可能会导致元素移动。**

​	通过insert函数可以为不支持push_front的容器提供相当的功能，但是**效率不高**，例如vector。

```c++
c.push_back(t);
c.emplace_back(t);
c.push_front(t);
c.emplace_front(t);
p1 = c.insert(p, t);		往迭代器p指向的元素前面插入元素t，并返回指向新添加第一个元素的迭代器
 	c.insert(p, n, t);	往迭代器p指向的元素前面插入n个元素t，并返回指向新添加第一个元素的迭代器
       c.insrt(p, b, e)		往迭代器p指向的元素前面插入迭代器b和e之间的元素
  
```



###### **push_back和emplace_back的区别？**

​		push_back和push_front这些操作都是拷贝元素，创建一个局部变量，并压入容器中。而emplace_back则是在容器管理的内存直接构建元素，因此效率更高。



##### 访问元素

​	在容器中访问元素的成员函数（即front、back、下标和at）返回的都是引用（可以直接当做左值使用），例如`v.back() = 42`。

1. at和下标操作只适用于string、vector、deque和array
2. back不适用于forward_list
3. **使用下标越界是不安全的，使用at发生越界，会抛出异常。**

```c++
c.back();
c.front();
c.at(n);
c[n]
```



##### 删除元素

​	vector和string不支持pop_front，forward_list有特殊版本的erase。

**删除元素会导致迭代器失效：**

1. 删除deque中除首尾位置之外的任何元素会导致所有的迭代器、引用和指针失效。
2. 指向vector和string删除点之后位置的迭代器、引用和指针都会失效。

```C++
c.pop_back();
c.pop_front();
c.erase(p);	删除迭代器p的元素
c.erase(b,e);	删除[b, e)之间的元素
```



##### 改变容器大小

​	array不支持resize()，resize函数用来增大或者缩小容器。如果当前大小所要求的的大小，容器后部的元素会被删除；小于新大小，会将新元素添加到容器后部。

​	**resize操作缩小容器，会导致指向删除元素的迭代器、引用和指针失效。对vector、string和deque进行resize操作可能导致迭代器、引用和指针失效。**

```c++
c.resize(n);
c.resize(n, t);	//t用来初始化添加的元素
```



### 迭代器失效

向容器添加元素

1. **vector和string：**存储空间被重新分配了，则指向容器的迭代器、指针和引用都会失效。如果空间没有被重新分配，插入位置之前的元素迭代器、指针和引用等不失效，插入之后元素的迭代器、指针和引用失效。
2. **deque：**插入到首尾之外的任何位置，都会导致迭代器、指针和引用失效；**如果在首尾位置插入，迭代器会失效，但是指针和引用不会失效**。
3. **list和forward_list：**迭代器、指针和引用都不会失效。



从容器删除元素

1. **vector和string：**指向被删除元素之前的迭代器、引用和指针仍有效。（**尾后迭代器总是会失效。**）
2. **deque：**首尾之外的任何位置删除元素，都会导致迭代器、指针和引用失效；**如果删除尾元素，尾后迭代器会失效，其他迭代器、引用和指针不受影响；删除首元素，不受影响。**
3. **list和forward_list：**迭代器、指针和引用都不会失效。



## 关联容器

​	标准库提供8个关联容器，这些容器不同体现在3个维度：

1. map、set
2. 是否重复
3. 有序还是无序

```c++
//有序数组
map
set
multimap
multiset

//无序
unordered_map
unordered_set
unordered_multimap
unordered_multiset
```

​	对于有序容器map、multimap、set和multiset，关键字类型必须定义元素比较的方法。可以通过在定义容器时往关键字添加自己定义的比较函数，具体参考c++ prime379页。



### map

​	map的关键字是const的

#### pair

​	map容器中的元素是一个pair类型的对象，pair是一个模板类型，保存两个名为first和second的数据成员。map使用pair的first保存关键字，使用second保存对应的值。

```c++
pair<T1, T2> p;
pair<T1, T2> p(v1, v2);
pair<T1, T2> p = {v1, v2};
make_pair(v1, v2);
```



#### 添加元素

	1. 对于map、set等不包含重复关键字的容器，**添加单一元素的insert和emplace返回一个pair**，告知我们插入操作是否成功。pair的first是指向指定关键字的迭代器，second是一个bool类型，成功插入，second为true，已经存在则返回false。
 	2. 对于multimap、multiset等允许包含重复关键字的容器，总会插入元素，只返回一个指向关键字的迭代器。

```c++
c.insert(v);
c.emplace(args);
c.insert(b, e);
```



#### 删除元素

​	关联容器相较于顺序容器额外支持了一个版本的erase函数，接收一个key_type。此版本删除所有匹配给定关键字的元素，并返实际删除的元素数量。对于不重复的容器，返回值为0或者1。

```c++
c.erase(k);	//删除指定关键字
c.erase(b, e);	//删除迭代器之间的元素
c.erase(p);	//删除指定迭代器的元素
```



#### 下标操作

​	map和unordered_map支持下标操作，下标接收是关键值key，返回对应的值；set和unordered_map则不支持。

**注意：对map使用下标操作，如果关键字不存在，会在容器中创建该关键字，其关联值默认初始化。**

```c++
c[key];
c.at(key);
```





#### 访问元素

​	判断一个关键字是否存在于容器中，使用find函数，而不要下标（map支持，set不支持）。下标操作在关键字不存在的时候，会插入。

重复容器中查找所有的元素：

	1. 对于允许重复的容器，**相同关键字的元素是相邻存储的（无论是unordered还是有序的）**。因此，可以先通过count计算出数量，接着find找到第一个值，然后通过自增迭代器找到所有的值
 	2. 通过lower_bound和upper_bound找到指定关键字的区间
 	3. 通过equal_range返回一个范围

```c++
c.find(k);
c.count(k);
c.lower_bound(k);	//找到第一个>= k的元素
c.upper_bound(k);	//找到第一个 >k的元素
c.equal_range(k);	//返回一个迭代器的pair，[)左开右闭，第一个满足和最后一个满足的下一个元素
```





## string（字符串对象）

一些有用的操作

​	以下的初始化，当从const char*创建string时，数组必须以空字符'\0'结尾，**拷贝操作遇到空字符结束**。如果还传递了一个长度pos2，就不必遇到空字符结束。

```C++
初始化，接收对象可以是string或者const char*
string s(cp, n);	s是cp的前n个字符
string s(s2, pos2);		s是s2从下标pos2开始的拷贝
string s(s2, pos2, len2);	s是s2从下标pos2开始的len2个字符，pos2不能超过s2的大小，否则抛出异常。

末尾插入： s.append(s);
删除s中范围的字符，替换为args字符（range可以是一个下标和长度，或者一对s的迭代器）：	s.repalce(range, args);
例如：s.replace(10, 3, "hello");	把s中下标为10开始的3个字符替换为hello
```



## 单向队列（queue）

只能访问queue容器适配器的第一个和最后一个元素。只能在容器的末尾添加新元素，只能从头部移除元素。

#### queue操作

- front()：返回queue中第一个元素的引用。如果queue是常量，就返回一个常引用，如果queue为空，返回值是未定义的。
- back()：返回queue中最后个元素引用
- push(const T &obj)：在queue的尾部添加一个元素的副本。
- pop()：删除queue中的第一个元素
- size()：返回queue中元素的个数
- empty()：如果queue中没有元素返回true。
- emplace()：用传给emplace()的参数调用T的构造函数，在queue的尾部生成对象。



## 双端队列（deque)

deque（双端队列）是由一段一段的定量连续空间构成，可以向两端发展，因此不论在尾部或头部安插元素都十分迅速。 在中间部分安插元素则比较费时，因为必须移动其它元素。

#### 头文件

```c++
#include <deque>
```

#### 定义

```c++
deque<int> a; // 定义一个int类型的双端队列a
deque<int> a(10); // 定义一个int类型的双端队列a，并设置初始大小为10
deque<int> a(10, 1); // 定义一个int类型的双端队列a，并设置初始大小为10且初始值都为1
deque<int> b(a); // 定义并用双端队列a初始化双端队列b
deque<int> b(a.begin(), a.begin()+3); // 将双端队列a中从第0个到第2个(共3个)作为双端队列b的初始值
```

#### 常用函数

```C++
//容量
容器大小：deq.size();
容器最大容量：deq.max_size();
更改容器大小：deq.resize();
容器判空：deq.empty();

//插入
头部添加元素：deq.push_front(const T& x);
末尾添加元素：deq.push_back(const T& x);
任意位置插入一个元素：deq.insert(iterator it, const T& x);
任意位置插入 n 个相同元素：deq.insert(iterator it, int n, const T& x);
插入另一个向量的 [forst,last] 间的数据：deq.insert(iterator it, iterator first, iterator last);

//删除
头部删除元素：deq.pop_front();
末尾删除元素：deq.pop_back();
任意位置删除一个元素：deq.erase(iterator it);
删除 [first,last] 之间的元素：deq.erase(iterator first, iterator last);
清空所有元素：deq.clear();

//取元素
下标访问：deq[1]; // 并不会检查是否越界
at 方法访问：deq.at(1); // 以上两者的区别就是 at 会检查是否越界，是则抛出 out of range 异常
访问第一个元素：deq.front();
访问最后一个元素：deq.back();
```



## 双向链表（list）

list 由双向链表（doubly linked list）实现而成，元素也存放在堆中，每个元素都是放在一块内存中，他的内存空间可以是不连续的，通过指针来进行数据的访问，这个特点使得它的随机存取变得非常没有效率，因此它没有提供 [] 操作符的重载。但是由于链表的特点，它可以很有效率的支持任意地方的插入和删除操作。

#### 头文件

```c++
#include <list>
```

####  定义

```c++
list<int> a; // 定义一个int类型的列表a
list<int> a(10); // 定义一个int类型的列表a，并设置初始大小为10
list<int> a(10, 1); // 定义一个int类型的列表a，并设置初始大小为10且初始值都为1
list<int> b(a); // 定义并用列表a初始化列表b
deque<int> b(a.begin(), ++a.end()); // 将列表a中的第1个元素作为列表b的初始值

int n[] = { 1, 2, 3, 4, 5 };
list<int> a(n, n + 5); // 将数组n的前5个元素作为列表a的初值
```

#### 基础函数

```c++
容器大小：lst.size();
容器最大容量：lst.max_size();
更改容器大小：lst.resize();
容器判空：lst.empty();

头部添加元素：lst.push_front(const T& x);
末尾添加元素：lst.push_back(const T& x);
任意位置插入一个元素：lst.insert(iterator it, const T& x);
任意位置插入 n 个相同元素：lst.insert(iterator it, int n, const T& x);
插入另一个向量的 [forst,last] 间的数据：lst.insert(iterator it, iterator first, iterator last);

头部删除元素：lst.pop_front();
末尾删除元素：lst.pop_back();
任意位置删除一个元素：lst.erase(iterator it);
删除 [first,last] 之间的元素：lst.erase(iterator first, iterator last);
清空所有元素：lst.clear();

//返回的是元素，不是指针
访问第一个元素：lst.front();
访问最后一个元素：lst.back(); 
```



## array

​	数组，容器大小固定，定义需要指定一个常量作为数组的大小（const表示只读语意，constexpr则是常量语意），这是因为编译器就需要知道分配多大的内存空间。

[具体分析参考](https://blog.csdn.net/wzz953200463/article/details/116176071)

```cpp
#include <iostream>
#include <array>
using namespace std;
 
constexpr int sqr1(int arg)
{
    return arg*arg;
}
 
const int sqr2(int arg)
{
    return arg*arg;
}
 
int main()
{
    array<int,sqr1(10)> mylist1;//可以，因为sqr1时constexpr函数
    array<int,sqr2(10)> mylist1;//不可以，因为sqr2不是constexpr函数
    return 0;
}
```



## map

​	**如果关键字不存在于map中，对map取下标操作，会添加一个新元素。**