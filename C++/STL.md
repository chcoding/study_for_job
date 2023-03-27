# STL

[API总结](http://vernlium.github.io/2019/12/29/C-STL%E5%B8%B8%E7%94%A8%E5%AE%B9%E5%99%A8API%E6%80%BB%E7%BB%93/)



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

