#前言
一个python项目快速开发完以后，常常针对瓶颈进行优化，其中一种方式就是对于性能至关重要的部分，使用C重写，这已经是一种最佳实践。如果整个项目完全使用C，开发效率就没有保障。python运行环境(CPython)是用C开发的，因此python与C结合起来很容易，而且方式多种多样。使用C重写了关键部分后，需要在python中调用，本文介绍四种最常用的调用C函数的方式，分别是c extension，Cython，ctypes和SWIG。举个例子，假设我们用C重写了add函数，它接受两个整数，计算他们的和并返回。
int myadd(int a, int b)
{
return a + b;
}
那么我们通过c extension，Cython，ctypes和SWIG四种方法可以在python中调用这个c语言写的add函数，不同方法各有优劣。

#C extension
C extension又称python-C-API，其python端基本不动，改变c端代码，需要再写接驳函数，这是最原始也是最底层的扩展python的方式。
demo_module.c：
在文件第一行包含python调用扩展的头文件

// pulls in the Python API
#include <Python.h>
用原生C写好需要调用的函数

static int mul(int a,int b){
return a*b;
}
static int add(int a,int b){
return a+b;
}
用python规定的调用方式，加一层C语言的包装，包装内容包括

a.定义一个新的静态函数，接受两个PyObject *参数，返回一个PyObject *值

b.解析第二个输入的PyObject *（通过PyArg_ParseTuple等方法），把python输入的变量变成C的变量

c.调用原生C函数

d.将调用返回的C变量，转换为PyObject*或其子类（通过PyLong_FromLong）等方法，并返回

// C function always has two arguments, conventionally named self and args
// The args argument will be a pointer to a Python tuple object containing the arguments.
// Each item of the tuple corresponds to an argument in the call's argument list.
static PyObject* demo_mul_and_add(PyObject *self, PyObject *args)
{
const int a, b;
// convert PyObject to C values
if (!PyArg_ParseTuple(args, "ii", &a, &b))
return NULL;
int c=mul(a,b);
int d=add(c,c);
// convert C value to PyObject
return Py_BuildValue("i", d);
}
创建一个数组，来指明python可调用这个扩展的函数。其中"mul_and_add"，代表编译后python调用时希望使用的函数名，demo_mul_and_add，代表调用当前C代码中的哪个函数，METH_VARARGS，代表函数的参数传递形式，主要包括位置参数和关键字参数两种，关于这个变量具体参考https://docs.python.org/3/extending/extending.html的1.4节，如果希望添加新的函数，则在最后的{NULL, NULL, 0, NULL}之前按同样格式填写新的调用信息。

// module's method table
static PyMethodDef DemoMethods[] = {
{"mul_and_add", demo_mul_and_add, METH_VARARGS, "Mul and Add two integers"},
{NULL, NULL, 0, NULL}
};
创建module的信息，包括了python调用时的模块名、可调用的函数集（就是上一步定义的Methods）等信息

static struct PyModuleDef demomodule = {
PyModuleDef_HEAD_INIT,
"demo", /* module name /
NULL, / module documentation, may be NULL /
-1,
DemoMethods / the methods array */
};
module初始化

PyMODINIT_FUNC PyInit_demo(void)
{
return PyModule_Create(&demomodule);
}

   下面编写setup.py，使用distutils包作为包构建安装的工具，编译扩展模块通常使用distutils或setuptools，它会自动调用gcc完成编译和链接。
setup.py：
from distutils.core import setup, Extension

module1 = Extension('demo',
sources = ['demo_module.c']
)

setup (name = 'a demo extension module',
version = '1.0',
description = 'This is a demo package',
ext_modules = [module1])
然后在命令提示符中，进入代码所在路径后，执行python setup.py build_ext –inplace。其中，–inplace表示在源码处生成pyd文件。执行上面命令后，会得到文件：demo.cp36-win_amd64.pyd。

之后就可以正常调用了

import demo
from time import time

s=time()
for i in range(10000000):
r = demo.mul_and_add(i, i)
print(time()-s)
Cython
Cython特点在于c语言代码不动，生成.pyx文件后，通过setup.py编译。

add_wrapper.pyx:

distutils: language = c++
cython: language_level = 3
cdef extern from "libadd.cpp":
int myadd(int a, int b)

def fast_add(int a, int b):
return myadd(a,b)

  Cython使用cdef extern from来声明一个在C++中实现的函数。上述代码声明了 myadd函数，使其可以在Cython中使用。为使mytanh可以在Python也可以直接调用，声明一个接口函数，命名为 fast_add。
Setup.py:
from distutils.core import setup, Extension
from Cython.Build import cythonize

setup(ext_modules=cythonize(Extension(
'fast_add', # 生成的模块名称
sources=['add_wrapper.pyx'], # 要编译的文件
language='c++', # 使用的语言
include_dirs=[], # gcc的-I参数
library_dirs=[], # gcc的-L参数
libraries=[], # gcc的-l参数
extra_compile_args=[], # 附加编译参数
extra_link_args=[], # 附加链接参数
)))

然后在命令提示符中，进入代码所在路径后，执行python setup.py build_ext –inplace，之后就可以使得python可以像调用一个普通python模块一样，调用fast_add了。

from fast_add import fast_addfrom time import time

s=time()
for i in range(10000000):
r = fast_add(i, i)
print(time()-s)

#Ctypes
Ctypes类似于Cython，主要作用就是在python中调用C动态链接库（shared library）中的函数。它提供C兼容的数据类型，并允许在DLL或共享库中调用函数，它可以用来将这些库封装在纯Python中。

libadd.c：
int add(int a, int b)
{
return a + b;
}
在命令提示符中执行gcc -shared -o libadd.so libadd.c（windows需提前安装gcc编译器“tdm64-gcc-5.1.0-2”，并添加到电脑环境变量，下载链接http://tdm-gcc.tdragon.net/download，安装教程链接https://blog.csdn.net/xinjitmzy/article/details/78967204）

得到libadd.so文件即可调用，在调用的python文件中通过cdll调用.so文件：

from time import time
from ctypes import *
mylib = CDLL(r"D:\tjh\code\python\ctypes\libadd.so")#.so文件所在路径

s=time()
for i in range(10000000):
r = mylib.add(i, i)
print(time()-s)
#SWIG
SWIG的目的就是要为C/C++ API提供脚本语言的接口，SWIG所有做的就是解决脚本语言和C/C++交互的问题，SWIG所做的事情其实就是两件事：

根据要调用的C API生成Wrapper函数，作为胶水来让脚本解析器和底层C函数进行交互。
为生成的Wrapper函数生成脚本语言的调用接口。
完成了对C/C++函数脚本语言接口的生成，通过直接使用脚本语言的接口，调用对应的Wrapper函数，Wrapper函数将脚本语言传入的参数，转换成C的参数，然后调用对应的C的接口，执行完后，Wrapper函数会将C返回的结果，转换成脚本语言的数据类型返回给脚本上层。

先安装swig并添加至环境变量

example.h：
int myadd(int a,int b);
example.c：
#include "example.h"
int myadd(int a,int b) {
return a+b;
}
接口文件example.i：
%module example

%{
#define SWIG_FILE_WITH_INIT
#include "example.h"
%}

int myadd(int a,int b);
使用命令行调用 Swig 方法产生 Python 模块：swig -python example.i。执行后会生成2个新的文件：example_wrap.c，example.py

setup.py：
from distutils.core import setup, Extension

example_module = Extension('_example',
sources=['example_wrap.c', 'example.c'],
)

setup(name='example',
version='0.1',
author="SWIG Docs",
description="""Simple swig example from docs""",
ext_modules=[example_module],
py_modules=["example"],
)
编译生成库文件：python setup.py build_ext –inplace

如果是Linux，执行完成后会在目录下生成类似 _example.cpython-37m-x86_64-linux-gnu.so 的文件

import example

print(example.myadd(4，6))
如果是Windows，则会在目录下生成类似 _example.cp37-win_amd64.pyd文件。调用方法稍有区别：

import _example

print(_example.myadd(4，6))
#性能对比与方法优劣
通过10000000次加法操作比较不同方法的性能，虽然10000000次加法操作很快，很难看出调用C函数对性能带来的提升，但这我们的主要目的是对比不同调用方式在调用共享库时的性能开销。

ctypes

优势：不需要程序员熟悉C/C++语言，不需要安装一个C/C++的编译器，它通过操作系统的接口直接操作C/C++代码。而且ctypes是标准库的一部分，只要安装了Python就可以直接使用。这几个原因使得它深受Python程序员的喜爱。

劣势：首先，ctypes不能简单调用C++程序，因为C++在编译的时候使用了name mangling这个技术来实现函数的重载。C++会自动地为类的成员函数加上类名前缀。所以，C++程序员需要以C语言的调用约定来提供接口，没有类，没有重载函数，没有模板，没有C++异常。不能直接调用现有的C++代码可能是这个方案最大的缺点。另外，对于list, set之类的数据类型，ctypes不能识别并自动地在Python与C/C++数据类型之间转换。C/C++部分不能识别Python数据类型，这时候只能用Python语言来编写转换代码。如果数据量较大，或者调用很频繁，转换代码反而会浪费很多的资源。这或许是ctypes的另一个劣势之一了。
Cython

 优势：非常容易使用。而且不仅能够处理C语言的模块，还能处理C++的模块——虽然没有直接支持虚函数之类的完整C++特性。因为它不直接使用C/C++语法，而是另外设计比C/C++更简洁优雅的新型语法，因此，对于不熟悉C/C++的程序员来说有很大的吸引力。相比ctypes来说，因为参数类型转换更加智能与高效，所以通常能够提升更多的效率。

劣势：如果想要更深入地控制内存与数据结构时，会发现他不得不熟练地掌握C/C++语言，然后用Cython的语法写出来。
SWIG

 优势：本质上与Cython类似，都使用了中间语言来生成转换代码。但SWIG能够在他们的接口文件中嵌入C/C++，能够让程序员仔细地调节数据类型的转换过程。在使用上，它比Cython的层次更低，更接近于Python本身提供的API。SWIG能够为多种脚本语言生成转换代码。

劣势：然而这种方案毕竟要学习一种新的语言，当想要仔细地调节类型转换代码的时候，需要学习SWIG/SIP的内部机制，被限定使用特殊的变量名，这使得这种方案的学习曲线相对较高。并且据说SWIG对于C++的支持不好。
#参考资料
1、在python里调用C函数的三种方式

https://www.yanxurui.cc/posts/python/2017-06-18-3-ways-of-calling-c-functions-from-python/

2、Python3.X使用C Extensions调用C/C++

https://blog.csdn.net/huachao1001/article/details/88240605

3、python的C扩展调用，使用原生的python-C-Api

https://www.cnblogs.com/catnip/p/8329298.html

4、Python 调用 C++（Cython）

https://iqhy.github.io/posts/2020/0228155601/

5、python：使用ctypes库调用DLL

https://blog.csdn.net/mengxiangsun/article/details/99688495

6、SWIG实现Python调用C/C++代码

https://www.biaodianfu.com/swig-python.html

7、浅谈Python程序与C++程序的联合使用

https://www.jb51.net/article/63623.htm
