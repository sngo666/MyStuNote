# std::reference_wrapper<>

```cpp
template <class T> class reference_wrapper;
```

该模板类仿真一个`T`类型对象的引用。但行为与普通的引用不太一样。

用类型T实例化的`reference_wrapper`类能包装该类型的一个对象，并产生一个`reference_wrapper<T>`类型的对象，该对象就像是一个引用，与普通引用最大的不同是：该引用可以考贝或赋值。

通过初始化构造或者赋值运算符进行赋值，但是只能接受同类型的对象，也就是说如果T类型是内置类型，使用常量进行拷贝赋值是不允许的。

## 通过get成员获取到reference_wrapper所引用的对象

```cpp
int a = 10;
reference_wrapper<int> r1 = a;
cout << r1 << endl;
cout << r1.get() << endl;
```

## 其他

* 普通的引用不是对象，所以无法存入任何标准库容器。`reference_wrapper`包装的引用就可以被存入容器中。
* 一个`reference_wrapper<T>`类型的对象，可以作为实参，传入形参类型为T的函数中。
* `std::ref`和`std::cref`通常用来产生一个reference_wrapper对象
* 若`reference_wrapper`包裹的引用是可以调用的，则`reference_wrapper`对象也是可调用的
* `reference_wrapper`常通过引用传递对象给`std::bind`函数或者`std::thread`构造函数
