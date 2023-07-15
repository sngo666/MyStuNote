# 关于std::tuple

`​std::tuple`是C++11新标准引入的一个类模板，又称为元组，是一个固定大小的异构值集合，由`std::pair`泛化而来。pair可以看作是tuple的一种特殊情况，成员数目限定为两个。tuple可以有任意个成员数量，但是每个确定的tuple类型的成员数目是固定的。

引入:

```cpp
#include <tuple>
```

构造：

```cpp
constexpr tuple(); 
tuple(const Types&... args );
tuple(tuple&& tpl);

/*隐式类型转换构造函数*/
template<class... UTypes>
  tuple(const tuple<UTypes...>& other );
template<class... UTypes>
  tuple(tuple<UTypes...>&& other);

/*支持初始化列表的构造函数*/
explicit tuple (const Types&... elems);
template <class... UTypes>
    explicit tuple (UTypes&&... elems); 

/*将pair对象转换为tuple对象*/
template< class U1, class U2 >
  tuple(const std::pair<U1, U2>& p );
template< class U1, class U2 >
  tuple(std::pair<U1, U2>&& p );

```

make_tuple() 函数以模板的形式定义在头文件中，功能是创建一个tuple右值对象（或者临时对象）。

```cpp
std::tuple<std::string, double ,int> t1 = std::make_tuple("Jack", 66.6, 88);
auto t2 = std::make_tuple(1, "Lily", 'c');
```

## 获取tuple的值

* std::get(std::tuple)

```cpp
auto t1 = std::make_tuple(1, "PI", 3.14);
auto a = std::get<0>(t1);
auto b = std::get<1>(t1);
auto c = std::get<2>(t1);

```

由于get的特性，tuple不支持迭代，只能通过元素索引(或tie解包)进行获取元素的值。但是给定的索引必须是在编译器就已经给定，不能在运行期进行动态传递，否则将发生编译错误。

* std::tie()

解包时，我们如果只想解某个位置的值时，可以用std::ignore占位符来表示不解某个位置的值.

```cpp
auto t1 = std::make_tuple(1, "PI", 3.14);

int num = 0;
std::string name;
double value;
std::tie(num, name, value) = t1;
std::cout << '(' << num << ',' << name << ',' << value << ')' << '\n'; //(1,PI,3.14)
```

## 其他方法

* 获取元素个数

可以采用`std::tuple_size<T>::value`来获得，其中T必须要显式给出tuple的类型；

```cpp
auto t1 = std::make_tuple(2, "MAX", 999, 888, 65.6);

std::cout << "The t1 has elements: " << std::tuple_size<decltype(t1)>::value << '\n';
// output: The t1 has elements: 5
```

*　获取元素类型

可以直接采用`std::tuple<size_t i,decltype(tuple)>::type`来获取；

```cpp
auto t1 = std::make_tuple(2, "MAX", 999.9);
std::cout << "The t1 has elements: " << std::tuple_size<decltype(t1)>::value << '\n';
std::tuple_element<0, decltype(t1)>::type type0;
std::tuple_element<1, decltype(t1)>::type type1;
std::tuple_element<2, decltype(t1)>::type type2;
type0 = std::get<0>(t1);
...
```

* 使用tuple引用来改变tuple内元素的值

`std::tuple<T1&> t3(ref&)`

```cpp
auto t1 = std::make_tuple("test", 85.8, 185);
std::string name;
double weight;
int height;

auto t2 = std::make_tuple(std::ref(name), std::ref(weight),std::ref(height)) = t1;
name = "spark", weight = 95.6, height = 188;

std::cout << '(' << std::get<0>(t2) << ' ' << std::get<1>(t2) << ' ' << std::get<2>(t2) << ')' << '\n';
//(spark 95.6 188)
```
