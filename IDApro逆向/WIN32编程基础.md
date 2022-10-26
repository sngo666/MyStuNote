# WIN32编程基础

## 关于字符串

c++支持两种字符串，常规是ANSI编码和Unicode编码。

1. 普通字符串(多字节)类型 CHAR->char
2. 宽字符串(UNICODE)类型 WCHAR->wchar_t(长度为两个字节，故结束符'\0'也为两个字节)
3. 通用字符串类型TCHAR->位置类型(需要引用tchar.h)
TCHAR的长度由环境决定，当环境使用UNICODE类型时，TCHAR长度为两个字节，使用多字节类型（ANSI）时，为一个字节。

例:

> ```cpp
> WCHAR wchr[] = L"1234";  
> TCHAR tchr[] = _T("123456");  
> ```

字符串常用函数
长度函数: `strlen`, `wcslen`, `_tcslen`。

字符串转数字:
A:`atoi`, `strol`
W:`_wtoi`, `wcstol`
T:`ttoi`, `_tcsol`

