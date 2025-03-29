---
title: python入门总结
date: 2022-7-30 17:00:00
---
主要来源于菜鸟教程

Python 是一种解释型语言： 这意味着开发过程中没有了编译这个环节。

Python 是交互式语言，面向对象语言

**修改编码格式，支持打印中文**

Python中默认的编码格式是 ASCII 格式，在没修改编码格式时无法正确打印汉字，所以在读取中文时会报错。解决方法为只要在python文件开头加入 **# -\*- coding: UTF-8 -\*-** 或者 **# coding=utf-8** 就行了

**注意：**Python3.X 源码文件默认使用utf-8编码，所以可以正常解析中文，无需指定 UTF-8 编码。

**注意：**如果你使用编辑器，同时需要设置 py 文件存储的格式为 UTF-8

以下划线开头的标识符是有特殊意义的。以单下划线开头 **_foo** 的代表不能直接访问的类属性，需通过类提供的接口进行访问，不能用 **from xxx import \*** 而导入。

以双下划线开头的 **__foo** 代表类的私有成员，以双下划线开头和结尾的 **__foo__** 代表 Python 里特殊方法专用的标识，如 **__init__()** 代表类的构造函数。

可以使用斜杠（ \）将一行的语句分为多行显示

```python
total = item_one + \
        item_two + \
        item_three
```

三引号可以由多行组成，编写多行文本的快捷语法，常用于文档字符串，在文件的特定地点，被当做注释。

```python
paragraph = """这是一个段落。
包含了多个语句"""
```

python 中多行注释使用三个单引号 **'''** 或三个双引号 **"""**。

Python可以在同一行中使用多条语句，语句之间使用分号(;)分割，**注意这里也设计到了python的包引用：**

```python
import sys; x = 'runoob'; sys.stdout.write(x + '\n')
```

> python2的打印方法是print x，python3的打印方法是print(x)

获取当前时间：

```python
import time
 
localtime = time.localtime(time.time())
print "本地时间为 :", localtime
localtime = time.asctime( time.localtime(time.time()) ) # 这是格式化的时间
print "本地时间为 :", localtime
print time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()) 
```

## python命令行参数

[参考](https://www.runoob.com/python/python-command-line-arguments.html)

Python 中可以使用 **sys** 的 **sys.argv** 来获取命令行参数：sys.argv[0] 表示脚本名。使用格式：`python test.py arg1 arg2 arg3`

也可以使用getopt模块。` opts, args = getopt.getopt(argv,"hi:o:")`，h后面没有冒号表示不需要参数，`test.py -h`即可，而i和o后面有冒号，表示需要参数，即`test.py -i inputfile -o outputfile`

```python
#!/usr/local/bin/python3
import sys
import getopt


def main(argv):
    inputfile = ''
    outputfile = ''
    try:
        opts, args = getopt.getopt(argv, "hi:o:")
        print(opts)
        print(args)
    except getopt.GetoptError:
        print('test.py -i <inputfile> -o <outputfile>')
        sys.exit(2)
    for opt, arg in opts:
        if opt == '-h':
            print('test.py -i <inputfile> -o <outputfile>')
            sys.exit()
        elif opt in ("-i", "--ifile"):
            inputfile = arg
        elif opt in ("-o", "--ofile"):
            outputfile = arg
    print('输入的文件为：', inputfile)
    print('输出的文件为：', outputfile)


if __name__ == "__main__":
    main(sys.argv[1:])
```

-----

## 变量类型

Python 中的变量赋值不需要类型声明。给变量赋值的时候可以一次性赋值多个：`a, b, c = 1, 2, "john"`

变量可以理解为是对对象（比如列表等）的引用（一个指针）。

删除对象的引用：`del var1,var2`，del之后，使用这个变量会直接报错：'var1' is not defined

Python有五个标准的数据类型：

- Numbers（数字）
- String（字符串）
- List（列表）：是有序的对象集合。列表用 **[ ]** 标识，是 python 最通用的复合数据类型。可以对列表进行+或者*操作，操作完了会改变list的长度，加号 **+** 是列表连接运算符，星号 ***** 是重复操作。可以用append操作，`list.append('Google')`这样。`del list[index]`删除某个元素。*
- *Tuple（元组）：元组是另一个数据类型，类似于 List（列表）。元组用 **()** 标识。但是元组不能二次赋值，相当于只读列表。但是可以进行进行+或者*操作，`列表和元组不能放在一起进行+或者*`。元组中只包含一个元素时，需要在元素后面添加逗号：`tup1 = (50,)`。`del l`删除整个元组。 
- Dictionary（字典）：类似map，字典用"{ }"标识。字典由索引(key)和它对应的值value组成。可以调用`map.keys()`和`.values()`来获取所有key和value。`d.clear()`清除字典的所有条目。`del d`删除字典。键必须不可变，所以可以用数字，字符串或元组充当，所以用列表就不行。`dict.has_key(key)`查看key是否存在。``str(dict)`输出字典可打印的字符串表示。

Python 列表截取可以接收第三个参数，参数作用是截取的步长，比如`letter[1:2:4]`

**运算符**

/是除号，不是取整。//是取整操作。Python2.x 里，/操作，只能得出整数。如果要得到小数部分，把其中一个数改成浮点数即可。

**表示多少次幂，比如`x**y	`

&按位与运算符，|按位或运算符，^ 按位异或，~按位取反，<<左移，>>右移

and与操作，布尔值和整型等的与操作，or和not同理

成员运算符：in和not in，用于看在指定的序列中是否存在某个成员。

身份运算符：is 判断两个标识符是不是引用自一个对象，相反的是is not，相同值的整型is为true，好像类似go的==，**python也有==操作**

## 语句

 python 并不支持 switch 语句，可用多个elif判断

```python
if 判断条件：
    执行语句……
else：
    执行语句……
 
if 判断条件1:
    执行语句1……
elif 判断条件2:
    执行语句2……
elif 判断条件3:
    执行语句3……
else:
    执行语句4……

if ( var  == 100 ) : print "变量 var 的值为100" 
```

也可在循环中加入 continue，break语句

```python
while 判断条件(condition)：
    执行语句(statements)……

# while … else 在循环条件为 false 时执行 else 语句块：
while count < 5:
   print count, " is  less than 5"
   count = count + 1
else:
   print count, " is not less than 5"

while (flag): print 'Given flag is really true!'
```

```python
for iterating_var in sequence:
   statements(s)
# for … else 表示这样的意思，for 中的语句和普通的没有区别，else 中的语句会在循环正常执行完（即 for 不是通过 break 跳出而中断的）的情况下执行，while … else 也是一样。
for num in range(10,20):  # 迭代 10 到 20 之间的数字
   for i in range(2,num): # 根据因子迭代
      if num%i == 0:      # 确定第一个因子
         j=num/i          # 计算第二个因子
         print ('%d 等于 %d * %d' % (num,i,j))
         break            # 跳出当前循环
   else:                  # 循环的 else 部分
      print ('%d 是一个质数' % num)
# ....
fruits = ['banana', 'apple',  'mango']
for index in range(len(fruits)):
   print ('当前水果 : %s' % fruits[index])
for letter in 'Python':     # 第一个实例
   print("当前字母: %s" % letter)
```

**pass** 不做任何事情，一般用做占位语句。

```python
for letter in 'Python':
   if letter == 'h':
      pass
      print '这是 pass 块'
   print '当前字母 :', letter
```

格式化打印

```python
print "My name is %s and weight is %d kg!" % ('Zara', 21)
```

三引号，支持换行，所见即所得

```python
 hi = '''hi 
there'''
```

unicode字符串，引号前小写的"u"表示这里创建的是一个 Unicode 字符串。：

```python
print(u'Hello\u0020World !)
```

字符串内建函数：`string.capitalize()`等

## 函数

定义的格式：

```python
def functionname( parameters ):
   "函数_文档字符串"
   function_suite
   return [expression]
```

比如：

```python
def printme( str ):
   "打印传入的字符串到标准显示设备上"
   print str
   return
printme( str = "My string") # 一种调用方法，通过关键字参数
printme("My string") # 另一种一种调用方法
```

函数的参数传递：

- **不可变类型：**类似 c++ 的值传递，如 整数、字符串、元组。如fun（a），传递的只是a的值，没有影响a对象本身。比如在 fun（a）内部修改 a 的值，只是修改另一个复制的对象，不会影响 a 本身。
- **可变类型：**类似 c++ 的引用传递，如 **列表，字典**。如 fun（la），则是将 la 真正的传过去，修改后fun外部的la也会受影响

默认参数

```python
def printinfo( name, age = 35 ):
   "打印任何传入的字符串"
   print "Name: ", name
   print "Age ", age
   return
 
#调用printinfo函数
printinfo( age=50, name="miki" )
printinfo( name="miki" )
```

不定长参数，加了星号（*）的变量名会存放所有未命名的变量参数。：

```python
def printinfo( arg1, *vartuple ):
   print arg1
   for var in vartuple:
      print var
   return
 
printinfo( 10 )
printinfo( 70, 60, 50 )
```

**匿名函数：**

通过lambda创建。语法：

```python
lambda [arg1 [,arg2,.....argn]]:expression
```

比如

```python
sum = lambda arg1, arg2: arg1 + arg2
print "相加后的值为 : ", sum( 10, 20 )
```

返回值，不需要定义返回的参数有哪些，接收处直接使用即可：

```python
def sum( arg1, arg2 ):
   return arg1 + arg2
 
# 调用sum函数
total = sum( 10, 20 )
```

定义在函数内部的变量拥有一个局部作用域，定义在函数外的拥有全局作用域。

```python
total = 0 # 这是一个全局变量
def sum( arg1, arg2 ):
   # global total
   # 如果在使用前指定 global total，那么就会用全局的total变量操作
   total = arg1 + arg2 # total在这里是局部变量，如果一个局部变量和一个全局变量重名，则局部变量会覆盖全局变量。
```

## 模块

Python 模块(Module)，是一个 Python 文件，以 .py 结尾，包含了 Python 对象定义和Python语句。

模块的引入：`import module1[, module2[,... moduleN]]`

调用模块的函数的语法为`模块名.函数名`，同级目录下可以直接引入模块，import xxx或者`import .xxx`即可。不在同级目录，则搜索在 shell 变量 PYTHONPATH 下的每个目录。如果都找不到，Python会察看默认路径。UNIX下，默认路径一般为/usr/local/lib/python/。

xxx.py的文件，xx就是模块名。

也可以指定引入模块的某个函数：`from modname import name1[, name2[, ... nameN]]`，然后可以直接调用name1(xxx)来调用函数而不用指定模块名，也可以导入模块的所有内容`from modname import *`

**包的概念**

包下面必须有__init__.py，__init__.py可以为空，也可以写：

 ```python
if __name__ == '__main__':
    print '作为主程序运行'
else:
    print 'package_runoob 初始化'
 ```

被调用时`__name__`是模块名，如果直接运行init.py，那么就会打印出`作为主程序运行`，`__name__`是`__main__`

## 错误

异常即是一个事件，捕捉异常可以使用try/except语句。

```python
try:
<语句>        #运行别的代码
except <名字>：
<语句>        #如果在try部份引发了'name'异常
except <名字>，<数据>:
<语句>        #如果引发了'name'异常，获得附加的数据
else:
<语句>        #如果没有异常发生
```

```python
try:
    正常的操作
   ......................
except(Exception1[, Exception2[,...ExceptionN]]):
   发生以上多个异常中的一个，执行这块代码
   ......................
else:
    如果没有异常执行这块代码
```

```python
try:
<语句>
finally:
<语句>    #退出try时总会执行，总是会执行
```

可以使用raise语句自己触发异常，触发异常后，后面的代码就不会再执行

```python
raise [Exception [, args [, traceback]]]
```

## 面向对象

类的self代表类的实例，self 在定义类的方法时是必须有的，虽然在调用时不必传入相应的参数。**self代表类的实例，而非类**。**self.__class__**指向类而不是类的实例。

**类变量：**类变量在整个实例化的对象中是公用的。

定义类的函数可以指定self参数，self指的是类的实例。

```python
class Employee:
   empCount = 0
   def __init__(self, name, salary):
      self.name = name
      self.salary = salary
      Employee.empCount += 1
   def displayCount(self):
     print "Total Employee %d" % Employee.empCount
 
emp1 = Employee("Zara", 2000)
emp1.displayEmployee()
```

可以添加，删除（通过del e.xxx)，修改类的属性

也可以使用以下函数的方式来访问属性：

- getattr(obj, name[, default]) : 访问对象的属性。
- hasattr(obj,name) : 检查是否存在一个属性。
- setattr(obj,name,value) : 设置一个属性。如果属性不存在，会创建一个新属性。
- delattr(obj, name) : 删除属性。

**python对象销毁(垃圾回收)**

Python 使用了引用计数这一简单技术来跟踪和回收垃圾。

Python 的垃圾收集器实际上是一个引用计数器和一个循环垃圾收集器。作为引用计数的补充， 垃圾收集器也会留心被分配的总量很大（即未通过引用计数销毁的那些）的对象。 在这种情况下， 解释器会暂停下来， 试图清理所有未引用的循环。

```python
a = 40      # 创建对象  <40>
b = a       # 增加引用， <40> 的计数
c = [b]     # 增加引用.  <40> 的计数

del a       # 减少引用 <40> 的计数
b = 100     # 减少引用 <40> 的计数
c[0] = -1   # 减少引用 <40> 的计数
# 4494605328 4494605328 4494605328
# Point 销毁
```

析构函数 __del__ ，__del__在对象销毁的时候被调用，当对象（比如一个类）不再被使用时，__del__方法运行：

**类的继承**

通过继承创建的新类称为**子类**或**派生类**，被继承的类称为**基类**、**父类**或**超类**。

如果在子类中需要父类的构造方法就需要显式地调用父类的构造方法，或者不重写父类的构造方法。

子类不重写 **__init__**的话，实例化子类时，会自动调用父类定义的 **__init__**。如果重写了**__init__** 时，实例化子类，就不会调用父类已经定义的 **__init__**

如果重写了**__init__** 时，要继承父类的构造方法，可以使用 **super** 关键字：

```python
class Father(object):
    def __init__(self, name):
        self.name=name
        print ( "name: %s" %( self.name))
    def getName(self):
        return 'Father ' + self.name
 
class Son(Father):
    def __init__(self, name):
        super(Son, self).__init__(name)
        print ("hi")
        self.name =  name
    def getName(self):
        return 'Son '+self.name
 
if __name__=='__main__':
    son=Son('runoob')
    print ( son.getName() )
```

派生类的声明，与他们的父类类似，继承的基类列表跟在类名之后，如下所示，这是继承了多个类：

```python
class SubClassName (ParentClass1[, ParentClass2, ...]):
```

Python同样支持运算符重载

```python
class Vector:
    def __init__(self, a, b):
        self.a = a
        self.b = b

    def __str__(self):
        return 'Vector (%d, %d)' % (self.a, self.b)

    def __add__(self, other):
        return Vector(self.a + other.a, self.b + other.b)


v1 = Vector(2, 10)
v2 = Vector(5, -2)
print(v1, v2)
print(v1 + v2)
# Vector (2, 10) Vector (5, -2)
# Vector (7, 8)
```

**私有属性**：两个下划线开头，则声明该属性为私有，不能在类的外部被使用或直接访问。Python不允许实例化的类访问私有数据，但你可以使用 **object._className__attrName**（ **对象名._类名__私有属性名** ）访问属性。

```python
class Runoob:
    __site = "www.runoob.com"

runoob = Runoob()
print runoob._Runoob__site
```

**类内定义的方法第一个参数必须是self。**

单下划线、双下划线、头尾双下划线说明：

- **__foo__**: 定义的是特殊方法，一般是系统定义名字 ，类似 **__init__()** 之类的。
- **_foo**: 以单下划线开头的表示的是 protected 类型的变量，即保护类型只能允许其本身与子类进行访问，不能用于 **from module import \***
- **__foo**: 双下划线的表示的是私有类型(private)的变量, 只能是允许这个类本身进行访问了。

