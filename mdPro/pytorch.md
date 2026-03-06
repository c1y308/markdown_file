# 1.基本函数与数据类型
## 1.1 Tensor相关函数
### 1.1.1 Tensor形状与索引
张量表示一个**由数组组成的数组**。**每一行就是一个样本，每一列代表了不同样本在此特征下的值。**每个特征向量就是一个数组。**w的矩阵形状只与单个sample的feature有关！！！**

``` python
tensor_1([1, 2])  # 1*2的tensor

tensor_2([[1, 2, 3],
		  [4, 5, 6]])  # 2*3的tensor

tensor_2[0] # 对tensor进行索引
out: tensor([1, 2, 3])

tensor_3 = torch.randn(3, 4)  #创建一个3*4的tensor
tensor_4 = torch.arange(12， dtype=torch.float32).reshape((3,4))   #一个1*12的tensor(也即是一个有12个元素的数组),可以调用dtype直接指定类型或直接加上.reshape()立刻修改格式
tensor_4 = torch.zeros((2, 3, 4))  #创建一个2*3*4元素(3维)全为0的tensor
```
### 1.1.2 .shape与.reshape(-,-)
tensor.shape**返回值类型为列表[]**
``` python
tensor_1.shape
out: torch.size([2])  #若为1*n则只返回n

tensor_2.shape
out: torch.size([2,3])  #若为n*n则将其合并为数组输出

tensor_2.reshape(-1, 6) #某一维维度为-1可以自动计算
```
### 1.1.3 .size(-)
tensor.size()单独返回**行数**或者**列数**。**（返回值为实数）**
``` python
tensor_1.size(0)
out: 2  #若为1*n则只返回n

tensor_2.size(1)
out: 3  #若为n*n则支持索引

tensor_2.numel()  #访问元素总数
out: 6
```
### 1.1.4 张量运算
**所有运算都都有广播机制，对行与列进行复制填充！**

- torch.mul(-,-)：**两个张量对应的元素相乘**；也可以**直接**两个tensor**相乘，相加、相除同理**。**单个张量求和使用.sum()**
-  torch.cat((-,-), dim)：**将两个张量合并为一个，dim=0按行压缩合并；dim=1按列压缩合并。**
-  X.sum(dim, keepdim)：按照行或者列进行矩阵求和，第二个参数选择是否保留维度。

## 1.2 torch函数
### 1.2.1torch.normal( -,-,-)
生成服从正态分布的随机数；参数依次为：**均值，方差，输出形状**：可以是一个**整数**，用于**生成一个大小为(1,size)的一维张量**；也可以是一个**元组**，用于生成相应形状的**多维张量**。

``` python
X = torch.normal(0, 1, (num_examples, len(w)))  #
y = torch.matmul(X, w) + b
y += torch.normal(0, 0.01, y.shape)
```
# 2. 梯度
## 2.1 正反向传播

核心思想为**链式求导法则**，将梯度存作为分母的因子内。正向传播求出**每个当前因子关于上个因子的梯度**。反向传播利用正向传播求出的梯度（方程）求出**y关于各个因子**的梯度(来更新**此因子对应(相乘)的weight**)，**就可以知道每个weight关于loss的梯度，即使是多层网络**。针对**b**:输入为1，权重为**b**。

## 2.2 代码实现
默认情况下pytorch会**累积梯度**，因此再次计算前需要清零。
``` python
import torch
x = torch.arange(4.0)
x.requires_grad_(True)

y = 2 * torch.dot(x, x)
y.backward()  #调用反向传播函数计算y关于x的梯度
x.grad
out: tensor([0., 4., 8., 12.])

x.grad.zero_()  #清零x的梯度，下划线表示重写内容
```
# 3.线性神经网络
## 3.1数据集处理(mini-batch)
### 3.1.1数据集读取
针对`.csv\.csv.gz`文件可以使用`np.loadtxt(path, delimiter=``, dtype=np.float32)`函数读取

``` python
import numpy as np
xy = np.loadtxt('diabetes.csv.gz', delimiter=`,`, dtype=np.float32)
x_data = torch.from_numpy(xy[:, :-1])  #所有行以及除了最后一列
y_data = torhc.from_numpy(xy[:, [-1]])  #所有行以及最后一列 []确保y_data为矩阵，而不是向量
```
### 3.1.2 mini-batch
​    由于每一次更新参数需要遍历整个数据集，**算梯度很贵！**因此引入mini-batch进行数据集处理**使得使用每个mini-batch中的samples进行一次loss计算与更新参数**。loss。backward()首先计算出模型loss关于参数的数学表达式，然后输入此次训练计算得到的loss，以此得到此次训练的梯度！

  **参数的矩阵形状只与单个sample的feature数量与单个sample对应的label数量有关！**

  首先将原有数据集随机打乱，然后**将多个sample合并为一个mini-batch(有多个mini-batch）;每一次epoch都在所有mini-batch上面训练。**在每个epoch里，完整遍历一遍选取的mini-batch数据集， 不停地从中获取一个小批量的输入和相应的标签，**计算此mini-batch的平均损失以及一次模型参数的梯度，来更新模型参数。**

​	主要是基于Dataset类实现来读取.csv或.csv.gz文件数据集;然后使用DataLoader()函数获取mini-batch。

​	**Dataset本质上是一个抽象类，可以将数据封装为python可以识别的数据结构。**Dataset类不能实例化，所以在使用Dataset的时候，我们需要定义自己的数据集类，也是Dataset的子类，来继承Dataset类的属性和方法，必须重写`__getitem__`（）方法与`__len__`()方法。
``` python
import torch
from torch.utils.data import Dataset  #抽象类
from torch.utils.data import DataLoader
import numpy as np

class DiabetesDataset(Dataset):  #定义一个糖尿病数据集类，可继承Dataset中给的基本功能
	def __init__(self, filepath):
		xy = np.loadtxt(filepath, delimiter=`,`, dtype=np.float32)
        
        self.len = xy.shape[0]  #xy为N*9的矩阵，xy.shape为一个(N,9)的元组
		self.x_data = torch.from_numpy(xy[:, :-1])  #所有行以及除了最后一列
		self.y_data = torhc.from_numpy(xy[:, [-1]])  #所有行以及最后一列 []确保y_data为矩阵，而不是向量
	
	def __getitem__(self, index):  #使得支持索引操作
		return self.x_data[index], self.y_data[index]  #返回的为(x, y)形式的元组！！！
                                                       #同时确定对dataloader执行for循环返回的形式
		
	def __len__(self):  #使得支持读取数据长度
		return self.len
    
```
​	dataloader主要作用为构建mini-batch以及对数据的随机洗牌操作。**dataloader[0]是一个元组！：(张量A{包含batchsize个feature向量}， 张量B{包含batchsize个label向量})；也即是一个mini-batch。**对其进行for循环可分别读取样本feature与label。

``` python
DataLoader(Dataset dataset, batch_size=int, shuffle=True, num_workers=int)
# grac[0]:需要提取数据的数据集
# grac[1]:batch_size的大小
# grac[2]:进行新一轮epoch是否重新洗牌
# grac[3]:是否多进程
```
``` python
diaDataset = DiabetesDataset('diabetes.csv.gz')  #数据集是一个类
train_iter = DataLoader(dataset=diadataset, batch_size=32, shuffle=True, num_workers=2)
```
### 3.1.2The MNIST Dataset
数据集为手写的1-9数字。
Training set : 60,000 examples,
Test set : 10,000 examples.
Classes : 10

读取后train_set则为一个类似列表的数据结构，**每个样本(索引读取的)是一个元组,包含图像属性与label**。可以通过索引访问；图像属性是一个pytorch张量，label是一个整数。

``` python
import torchvision
trans = transforms.ToTensor()  #将像素数据转变为float，并除以255确保在0-1之间
train_set = torchvision.datasets.MNIST(root='../dataset/mnist',train=True, transform=trans,download=True)
test_set  = torchvision.datasets.MNIST(root='../dataset/mnist',train=False,transform=trans,download=True) 
```
### 3.1.3The CIFAR10 Dataset
数据集包含飞机，汽车，船，卡车，各种动物···
Training set : 50,000 examples,
Test set : 10,000 examples.
Classes : 10
``` python
import torchvision
trans = transforms.ToTensor()
train_set = torchvision.datasets.CIFAR10(root='../dataset/c',train=True, transform=trans,download=True)
test_set  = torchvision.datasets.CIFAR10(root='../dataset/c',train=False,transform=trans,download=True)
```
## 3.2 基础优化算法
最常规的为**随机(体现在mini-batch)梯度下降算法(SGD)**，梯度是增长率最快的方向，其反方向则降得最快。
### 3.2.1 定义优化器
传入需要计算得到的参数与学习率。
``` python
optimizer = torch.optim.SGD(torch.nn.module.parameters(), lr=0.03)
```
### 3.2.2 定义损失函数
#### 3.2.2.1 均方损失函数
``` python
criterion = torch.nn.MSELoss(size_aerage=False)
```
#### 3.2.2.2 交叉熵损失函数
softmax回归中针对每个输入的sample有一个**独热向量**来表示**此输入样本**归属于哪个类。
## 3.3 线性回归
注意：在线性模型中，一个sample的feature为行向量形式，即**行数表示sample数，列数表示有多少个feature数**。线性回归loss函数都是凸函数，无局部最优点。

对模型进行降维使用多重网络需要对其进行非线性变换，避免无效网络。

``` python
class LinearModel(torch.nn.Module)  #从torch.nn.Module父类中继承
	def __init__(self, in_units, units):  #构造函数(调用时初始化)
		super().__init__()  #继承父类
		self.linear = torch.nn.Linear(1,1)  #输入维度；输出维度(在此对linear进行初始化，本质是创建了个linear类)
		#self.linear是嵌套在类中的一个可调用子类
        self.weight = nn.Parameter(torch.randn(in_units, units))
        self.bias = nn.Parameter(torch.randn(units,))
	def forward(self, x):  #正向传播，重写forward函数
		y_pred = self.linear(x)  #这个类的__call__在self.linear中，调用forward(self, x)，做wx+b计算
		return y_pred

model = LinearModel(0, 0.01)  # 初始化参数

criterion = torch.nn.MSELoss(size_aerage=False)  #构建损失函数
optimizer = torch.optim.SGD(model.parameters(), lr=0.01)  #选择需要迭代的权重与学习率

x_data = torch.Tensor([[1.0],  #3*1的tensor
                       [2.0],
                       [3.0]])

y_data = torch.Tensor([[2.0],  #3*1的tensor
                       [4.0], 
                       [6.0]])

for eopch in range(100):
    	y_pred = model(x_data)  #计算预测值
        loss = criterion(y_pred, y_data)  #计算损失函数
        print(epoch, loss)
        
        optimizer.zero_grad()  #清零梯度
        loss.backward()  #反向传播
        optimizer.step()  #更新权重
        
print('w= ', model.linear.weight.item())
print('b= ', model.linear.bias.item())

x_test = torch.Tensor([[4.0]])
y_test = model(x_test)
print('y_pred= ', y_test.data)
```
## 3.4 Softmax回归
为了将输出o(i)转化为概率，引入softmax函数，使得输出预测概率满足非负、和为1。
#### 3.4.1 binary模型
例子:学习多长时间可以通过考试，满足哪些指标就是糖尿病。针对二值回归，损失函数更新为下面形式：

``` python
import torch.nn.functional as F
class LinearModel(torch.nn.Module)
	def __init__(self):
		super().__init__()
		self.linear = torch.nn.Linear(1,1)
        
	def forward(self, x):  #正向传播
		y_pred = F.sigmoid(self.linear(x))
		return y_pred

model = LinearModel()

criterion = torch.nn.BCELoss(size_aerage=False)  #构建损失函数,交叉熵
optimizer = torch.optim.SGD(model.parameters(), lr=0.01)

x_data = torch.Tensor([[1.0], 
                       [2.0], 
                       [3.0]])

y_data = torch.Tensor([[0], 
                       [0], 
                       [1]])
```
# 4.深度学习计算
## 4.1层与块
  `nn.Sequential()`构建模型，参数的传递顺序就是层的执行顺序。其是**特殊的Module**,即是一个表示块的类，**一个由module组成的有序列表。**
``` python
import torch
from torch import nn
from torch.nn import functional as F

net = nn.Sequential(nn.Linear(20, 256), nn.ReLU(), nn.Linear(256, 10))

X = torch.rand(2, 20)
net(X)
```
### 4.1.1自定义层
``` python
class MyLinear(nn.Module):
    def __init__(self, in_units, units):
        super().__init__()
        self.weight = nn.Parameter(torch.randn(in_units, units))
        self.bias = nn.Parameter(torch.randn(units,))
    def forward(self, X):
        linear = torch.matmul(X, self.weight.data) + self.bias.data
        return F.relu(linear)
```
### 4.1.2 自定义块
``` python
class MLP(nn.Module):
    def __init__(self):
        # 调用MLP的父类Module的构造函数来执行必要的初始化。
        # 这样，在类实例化时也可以指定其他函数参数，例如模型参数params
        super().__init__()
        self.hidden = nn.Linear(20, 256)  # 隐藏层
        self.out = nn.Linear(256, 10)  # 输出层
                                       # 同时可以将自定义层加入到此块里面
            
    # 定义模型的前向传播，即如何根据输入X返回所需的模型输出
    def forward(self, X):
        return self.out(F.relu(self.hidden(X)))
```
## 4.2 参数管理
### 4.2.1 参数的访问
  每个参数是一个**parameter数据结构，其包含.data(tensor类型)、.grad与require_grad，参数是复合的对象，包含值、梯度和额外信息。**

``` python
print(net[2].state_dict())  # 若第第三层为线性层，则打印出`weight`与`bias`
out:OrderedDict([('weight', tensor([[-0.0427,-0.2939,-0.1894,0.0220]])), ('bias', tensor([0.0887]))])

print(type(net[2].bias))
print(net[2].bias)
print(net[2].bias.data)
<class 'torch.nn.parameter.Parameter'>
Parameter containing:
tensor([0.0887], requires_grad=True)
tensor([0.0887])
```
### 4.2.2 参数初始化
  深度学习框架**提供默认随机初始化**， 也允许我们创建自定义初始化方法， 满足我们通过其他规则实现初始化权重，在`nn.init`里面。使用`net[].apply`来初始化参数。**注意稠密层的参数是互相绑定的。**
  **使用函数初始化：**

``` python
def init_normal(m):  # para: module 将所有权重参数初始化为标准差为0.01的高斯随机变量， 且将偏置参数设置为0。
    if type(m) == nn.Linear:
        nn.init.normal_(m.weight, mean=0, std=0.01)
        nn.init.zeros_(m.bias)
net.apply(init_normal)
net[0].weight.data[0], net[0].bias.data[0]

def init_constant(m):
    if type(m) == nn.Linear:
        nn.init.constant_(m.weight, 1)
        nn.init.zeros_(m.bias)

def init_42(m):
    if type(m) == nn.Linear:
        nn.init.constant_(m.weight, 42)

def init_xavier(m):
    if type(m) == nn.Linear:
        nn.init.xavier_uniform_(m.weight)

net[0].apply(init_xavier)
net[2].apply(init_42)
```
  **直接访问参数值修改：**
``` python
net[0].weight.data[:] += 1
net[0].weight.data[0, 0] = 42
net[0].weight.data[0]
```
## 4.3 读写文件
### 4.3.1 读写张量
``` python
import torch
x1 = torch.arange(4)

torch.save(x, 'x-file')
x2 = torch.load('x-file')
```
同时可以**将张量作为列表的元素来进行存储：**
``` python
x = torch.arange(4)
y = torch.zeros(4)
torch.save([x, y],'x-files')
x2, y2 = torch.load('x-files')

(x2, y2)
out:(tensor([0, 1, 2, 3]), tensor([0., 0., 0., 0.]))
```
将其作为**字典**进行存储：
``` python
mydict = {'x': x, 'y': y}
torch.save(mydict, 'mydict')
mydict2 = torch.load('mydict')

mydict2
out:{'x': tensor([0, 1, 2, 3]), 'y': tensor([0., 0., 0., 0.])}
```
### 4.3.2 保存模型参数与恢复
``` python
torch.save(net.state_dict(), 'mlp.params')

clone = MLP()
clone.load_state_dict(torch.load('mlp.params'))
clone.eval()
```
# 5.卷积神经网络
## 5.1 从全连接层到卷积
 **卷积核**将图像分割成多个区域，并为每个区域包含目标的可能性打分。全连接中**w**的形状为**feature个数×神经元个数。**以下以单输入通道多输出通道举例：

## 5.2 填充与步幅

## 5.3 多输入多输出通道

### 5.3.1 多输入通道

```  python
import torch
from torch import nn
from d2l import torch as d2l

def corr2d(X, K):  #@save
    """计算单通道二维互相关运算"""
    h, w = K.shape  # 首先读取卷积核的形状
    Y = torch.zeros((X.shape[0] - h + 1, X.shape[1] - w + 1))  # 计算出卷积后输出的形状
    for i in range(Y.shape[0]):
        for j in range(Y.shape[1]):  # 本质是在对一个二维数组进行处理
            Y[i, j] = (X[i:i + h, j:j + w] * K).sum()
    return Y

def corr2d_multi_in(X, K):
    # 先遍历“X”和“K”的第0个通道维度，再把它们加在一起
    return sum(d2l.corr2d(x, k) for x, k in zip(X, K))  # 每一个X,K就是一个矩阵。sum是每个下标对应元素相加
```
### 5.3.2 多输出通道
为了获得多个通道的输出，我们可以**为每个输出通道创建**一个卷积核张量。**有几个输入通道每堆卷积核就有几层；有几个输出通道就有几堆卷积核。**
``` python
def corr2d_multi_in_out(X, K):
    return torch.stack([corr2d_multi_in(X, k) for k in K], 0)  # 输出为多个二维矩阵

K = torch.tensor([[[0.0, 1.0], [2.0, 3.0]], [[1.0, 2.0], [3.0, 4.0]]])
K = torch.stack((K, K + 1, K + 2), 0)
#通过将核张量K与K+1（K中每个元素加1）和K+2连接起来，构造了一个具有个3输出通道的卷积核。
K.shape
out: torch.Size([3, 2, 2, 2])
```
## 5.4 汇聚层
双重目的：**降低卷积层对位置的敏感性(像素可以平移)**，同时**降低对空间降采样表示的敏感性。**
汇聚层是一个固定形状的矩阵，与卷积层相不同点：**汇聚层中不包含参数。**且运算具有确定性：通常**结果为汇聚窗口中所有元素的最大值或平均值。默认情况下汇聚层的步幅为汇聚层的大小（3， 3）。**

``` python
def pool2d(X, pool_size, mode='max'):
    p_h, p_w = pool_size  # 获取汇聚层的大小
    Y = torch.zeros((X.shape[0] - p_h + 1, X.shape[1] - p_w + 1))  # 计算输出矩阵的大小
    for i in range(Y.shape[0]):  # 本质是一个二维数组运算
        for j in range(Y.shape[1]):
            if mode == 'max':
                Y[i, j] = X[i: i + p_h, j: j + p_w].max()  # 得到二维数组的最大值
            elif mode == 'avg':
                Y[i, j] = X[i: i + p_h, j: j + p_w].mean()  # 得到二维数组的平均值
    return Y
```
### 5.4.1 汇聚层的步幅与填充

默认情况下汇聚层的步幅为汇聚层的大小（3， 3）。

``` python
pool2d = nn.MaxPool2d(3, padding=1, stride=2)
pool2d(X)

pool2d = nn.MaxPool2d((2, 3), stride=(2, 3), padding=(0, 1))
pool2d(X)
```
### 5.4.2 多通道汇聚层
在处理多通道输入数据时，**汇聚层在每个输入通道上单独运算，**而不是像卷积层合并每个通道的结果。 **汇聚层的输出通道数与输入通道数相同！**。 