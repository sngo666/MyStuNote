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