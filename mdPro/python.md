# Python小技巧
---
## 1.变量与数据类型
### 1.1字符串的处理 
1.如果需要把字符串变量全部大写可以使用`name.title()`或者`name.upper()`实现；相应的小写则是为`name.lower()`。
2.合并两个字符串变量：`f"{first_name} {last_name}"`，其中f为format的缩写；`{}`内既可以为字符串变量也可以直接为字符串。 
3.在输出中添加`\t`来添加空白使得输出对齐；`\n`进行换行。可以调用形如`name.rstrip()`函数形式来删除字符串变量末尾的空白；及其
`name.lstrip()`函数删除开头空白与`name.strip()`同时删除前后的空白。 
### 1.2数的处理 
1.使用`num**num`来进行乘方运算。 
2.可以对很大的数采用下划线处理，如：`14_000_000_000`。  

---
## 2.列表 
### 2.1列表相关函数
```python
#列表元素添加
bicycles.append('honda')
bicycles.insert(,'honda')
#移除列表元素
del(bicycles[2]);
old_bicycles = bicycles.pop(2)
bicycles.remove('honda')
#列表排序
bicycles.sort()
bicycles.sorted()
bicycles.reverse()
#获取列表长度
len(bicycles)
#删除重复列表元素
set(bicycles)
```
1.使用`[]`表示列表，并且使用`,`分隔其中的元素。例如`bicycle = ['trek','redline','specialized']`。 
2.可以直接通过索引来访问列表的元素，最后一个为`-1`,从后往前依次递推，如:`bicycle[-1]`。 
3.可以使用`.append('')`函数在列表末尾添加元素；也可以使用`.insert(0,'')`来进行插入元素；以及`del(bicycle[1])`来进行删除元素。也可以使用`.pop(bicycle)`函数来删除元素并将其添加到另外的列表中去。若通过元素值进行删除则使用`.remove('')`来进行操作。 
4.如果列表元素全部为小写，可以使用`.sort()`来进行永久排序；以及`.sorted`来进行临时排序；使用`.reverse()`进行列表反转。 
5.使用`len()`进行列表长度获取。 
### 2.2列表基本操作
1.遍历列表(注意`:`)：
``` python
for bicycle in bicycles:
	print(bicycle)
```
2.如果需要生成连续数，可以使用`range(start,end-1)`来生成一系列数（如果没有指定起始值，默认从0开始）：
```python
for value in range(1,4):
	print(value)
```
3.使用`list()`函数直接生成列表：
```python
numbers = list(range(1,6))
```
 1vim hello.txtbash
```python
min(numbers)
max(numbers)
sum(numbers)
```
5.列表解析(用于简易生成数字列表，并对其进行操作)：
`squares = [value**2 for value in range(1,11)]`
6.处理列表的部分元素（切片操作）：指定要使用的第一个元素的索引和最后一个元素的索引+1（注意中间为`:`）如果没有指定就遍历完：
```python
numbers[0:3]#切片0，1，2号元素
numbers[2:]#从第二位元素开始切片所有
numbers[-3:0]#取列表后三个元素
numbers[:3]#取列表前三个元素
numbers[:]#切片所有元素（复制列表）
```
7.判断某字符串是否在某字符列表里面，使用`in`：
`'honda' in bicycles`
`True`
如果判断不在使用`not in `：
```python
if 'honda' not in bicycles:
	print("honda not in bicycles")
```
## 3.元组
3.1.元组使用`()`来进行定义与列表的`[]`进行区分。元组是不可更改的列表。如果要创建一个只包含一个元素的元组，则需要有`,`如`my_t(3,)`。
3.2.元组虽然不可以修改，但是可以重新进行定义。 
##4.if语句
1.if语句结尾需要添加`：`同理`else`也需要加`:`：

```python
if car == 'bwm':
	print(car.upper())
```
2.Python中检查是否相等对于字符串需要区分大小写，若为且条件使用`and`语句；若为且条件则使用`or`。
3.如果检查超过两个情形需要使用`if-elif-else`语句。
## 4.字典
```python
#定义字典
alien_0 = {
           'color':'green',
           'point':10,
          }

color_0 = alien_0['color']#读取字典键对应的值
color_0 = alien_0.get('point','no point value')

del alien_0['color']#删除键值
#遍历键值：
for key,value in alien_0.items（）:
for key in alien_0.keys()：
for value in aliens.values():
#嵌套操作，字典列表：
for i in range(30):
	new_alien = {'color':'green','points':5,'speed':'slow'}
	aliens.append(new_alien)
```
1.字典可以类比为C语言中的结构体，其键为字符串形式；值可以为字符串也可以为数值也可以为列表也可以为字典（嵌套操作）。其定义为`{}`，与列表一样访问同样通过`[]`。 
2.访问字典中的值可以通过读取其对应的键来进行：`print('alien_0['color']')`。如果访问的键可能不存在，可以使用`.get（）`方法 
如：`alien_0.get('point','no point value')`。 
3.删除字典中的键对值与列表的操作一样，使用`del`函数，比如：`del alien_0['point']`
4.遍历字典的键对值,使用`.item（）`函数： 
```python
for key,value in alien_0.item（）:
	print(f"\nKey:{key}")
	print(f"\nValue:{value}")
```
如果只需要键，则使用`.keys()`函数：`for key in alien_0.keys()：`。相应的，遍历值：`for value in aliens.values():`。 
5.字典可以直接通过`print函数进行打印`。如进行嵌套操作，创建字典列表。 
## 5.用户输入以及循环
### 5.1 input输入
1.input输入默认为字符串，若需要转化为数值则需要使用`int()`函数。
### 5.2 while循环 
1.可以使用`break`语句来推出while循环；如果希望满足某种条件后不执行接下来的代码，但需要继续执行循环，使用`continue`语句。 
2.while循环可以在遍历列表的同时修改列表的元素！！！
```python
while 'cat' in pets:
	pets.remove('cat')
```
3.while通常配合标志位一起使用。 
### 5.3 for循环
1.zip函数：将两个列表内的元素合并为一个由*元组*构成的列表。合并后可通过for循环一次获取多个可迭代对象。

```python
x_data = [1,2,3,4]
y_data = [2,4,6,8]
z_data = zip(x_data, y_data)

for x_val, y_val in zip(x_data, y_data):
	#process x_val, y_val	
```
## 6.函数
1.函数的形参可以给默认值，因此调用函数时可以不给此形参传递实参。
2.形参可以传递列表，函数内部可以通过循环进行处理。
3.函数可以返回字符串，字典。 
4.传递任意数量的实参：`*username`：原理为创建一个名为`username`的元组，并将所有传递进来的实参收集进这个元组内。`**username`创建字典。 
5.使用`import`来代替`include`。

## 7.类

### 7.1类

1.类就是结构体。但函数可以为类中的内容,。在python中，首写字母大写的就是类。
2.类中的内容也可以为类(嵌套！)如：`self.battery = Battery()`
3.若为可调用类，需定义`__call__(self, *argc, **argv)函数`
4.*所有`self`都指的是定义的类！*
5.创建一个实例时，默认调用__init__方法，即传入的参数传入__init__。
```python
class Dogs:
	def __init__(self,name,age):
		self.name = name
		self.age = age
    def __call__(self, *args, **kwargs):  #前者将所有不定长数字合并为一个元组args，后者将所有赋值合并为一个字典输出
        print("hello" + str(args[0]))
	def sit(self):
		print(f"{self.name} is now sitting")
	def roll_over(self):
		print(f"{self.name} rolled over")
```
### 7.2子类
定义子类：
```python
class ElectricCar(Car):#父类
	__init__(self,make,model,year):
		super.__init__(make,model,year)
```
1.通过`super.__init__()`的方式进行调用父类的方法(函数)。
2.如果父类中的方法子类不需要，可以重写这个方法对其进行重定义。 





 

