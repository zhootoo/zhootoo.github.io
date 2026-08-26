---
title: PyTorch的个人理解
date: 2025-08-26 07:46:36
categories: [深度学习]
tags:
  - PyTorch
  - 深度学习
  - LLM
---

接触 PyTorch 也有一段时间了，从最初的“照着教程敲代码”，到后来能比较顺畅地搭模型、调训练循环。这篇文章不是官方文档的翻译，而是我站在使用者角度，对 PyTorch 底层设计的一些个人理解，希望能帮到同样在入门的朋友。

## 一句话概括 PyTorch

> **PyTorch 就是一个带自动微分的 NumPy，外加一个模块化的神经网络组件库。**

如果你能理解这句话，PyTorch 的大半就掌握了。剩下的是各种 API 的熟练度问题。

## 核心概念一：张量（Tensor）

Tensor 是PyTorch中一切高层概念的基石，是建设高层建筑所需要的基础材料，理解Tensor对于熟练使用PyTorch至关重要。在PyTorch中，模型的输入输出和模型的权重参数都是用Tensor来表示的。可以认为Tensor 本质就是 n 维数组，和 NumPy 的 `ndarray` 几乎一一对应，但它多了两个关键能力：**GPU 加速**和**自动微分**。

### Tensor初始化

```python
import torch

# 支持从数据中直接创建
data = [[1, 2, 3], [4, 5, 6]]
x_data = torch.tensor(data)


# 和 NumPy 几乎一样的创建方式
x = torch.zeros(3, 4)          # 全 0
y = torch.randn(3, 4)          # 标准正态分布随机数
z = torch.arange(10)           # 0~9

# 与 NumPy 互转
import numpy as np
arr = x.numpy()                # Tensor -> ndarray
x2 = torch.from_numpy(arr)     # ndarray -> Tensor

# 从另外一个张量中创建
data_ones = torch.ones_like(x_data)    # 参照已有张量的形状和类型创建一个全 1 的张量，保留x_data的属性不变
data_rand = torch.rand_like(x_data, dtype=torch.float)  #参照已有张量的形状和类型创建一个随机数张量，保留x_data的属性不变

# 值得注意的是requires_grad不会被自动继承过来
```

如果想生成固定shape的tensor，有一些函数可以直接指定想要生成的张量的维度
```python
x = torch.zeros(3, 4)          # 全 0
y = torch.randn(3, 4)          # 标准正态分布随机数
z = torch.arange(10)           # 0~9
```

### Tensor的属性

```python
x = torch.tensor([1, 2, 3])
x.dtype      # 描述存储的数据类型
x.device     # 描述存储在哪个设备上，默认是CPU，如果在GPU上，则为cuda:0
x.shape      # 描述张量的形状
x.requires_grad # 描述是否需要梯度追踪，默认是False
```

### Tensor的操作

PyTorch支持约100多种不同类型的tensor操作：转置，索引，切片，常规的数学运算，线性代数领域的运算，随机采样等。每种运算都支持在GPU上进行加速运算，这是PyTorch的框架所支持的。

```python
tensor = torch.rand(3, 4)
tensor.T  # 转置
tensor[1, 2]  # 索引
tensor[:2, 1:]  # 切片
tensor + 1  # 常规的数学运算
tensor.mul(tensor)  # 常规的数学运算
tensor.matmul(tensor.T)  # 线性代数领域的运算
tensor @ tensor.T  # 等价于matmul
tensor.mean()  # 随机采样
torch.cat((tensor, tensor), dim=0)  # 沿着某个维度拼接
```

#### 就地操作

```python
tensor = torch.rand(3, 4)
tensor.add_(1)  # 就地操作
tensor += 1  # 等价于add_
# 就地操作节约了一些内存，但会改变原始的tensor，所以不推荐在循环中使用
```

### Tensor与Numpy进行桥接

存储在CPU上的Tensor可以与Numpy进行桥接，他们共享相同的内存，所以对其中一个进行修改，另一个也会被修改。

```python
import numpy as np
tensor = torch.rand(3, 4)
print(tensor)
# tensor([[0.2071, 0.2361, 0.2611, 0.2911],
#         [0.3211, 0.3411, 0.3611, 0.3811],
#         [0.4011, 0.4211, 0.4411, 0.4611]])

np_array = tensor.numpy()
print(np_array)
# [[0.20710754 0.2361021  0.26110649 0.29110145]
#  [0.32110214 0.3411021  0.36110649 0.38110145]
#  [0.40110214 0.4211021  0.44110649 0.46110145]]

tensor.add_(1)
print(np_array)   # 共享的底层内存改变
# [[1.20710754 1.2361021  1.26110649 1.29110145]
#  [1.32110214 1.3411021  1.36110649 1.38110145]
#  [1.40110214 1.4211021  1.44110649 1.46110145]]

n = np.array([1, 2, 3])
tensor = torch.from_numpy(n)
np.add(n,1,out=n)
print(tensor) # 共享的底层内存改变
# tensor([2, 3, 4])
```

## 核心概念二：自动微分（Autograd）

这是 PyTorch 的灵魂。自动微分的意思是：**你只需要定义前向计算过程，PyTorch 会自动帮你把梯度算出来。**

```python
x = torch.tensor([2.0], requires_grad=True)  # 开启梯度追踪
y = x ** 2 + 3 * x                           # y = x^2 + 3x
y.backward()                                 # 反向传播
print(x.grad)                                # tensor([7.])，即 2x + 3 = 7
```

关键在于 `requires_grad=True`。设置之后，PyTorch 会在**前向计算的过程中偷偷记录计算图**，等 `backward()` 时再从后往前把每个中间变量的梯度算出来。

我最初一直困惑的是：梯度到底是怎么"自动"算出来的？后来想明白了，其实就是**链式法则**。PyTorch 把 `y = x ** 2 + 3 * x` 拆成一张图：

```
x -> x**2 -> + -> y
  -> 3*x  ->
```

`backward()` 就是沿着这张图反向走一遍，把 `dy/dx` 逐层传回去。所谓的"深度学习框架"，本质就是替你维护了这张图和链式求导。

### 计算图的释放

有一个细节值得注意：**每次 `backward()` 之后，计算图会被释放**，所以 `loss.backward()` 只能在训练循环里调用一次。如果需要多次反向（比如某些 GAN 的写法），要传 `retain_graph=True`。这个细节常常是新手报错 `RuntimeError: Trying to backward through the graph a second time` 的根源。

## 核心概念三：动态计算图

这是 PyTorch 区别于老式框架（如 TensorFlow 1.x 的静态图）最重要的设计。

- **静态图**：先定义好完整的计算图，再喂数据执行。优点是可以预先优化，缺点是图的定义和调试都很痛苦，Python 的 `if`、`for` 都得想办法塞进图 API 里。
- **动态图**：**执行即构建**。每跑一次前向，图就现场搭一次。写模型就跟写普通 Python 代码一样，`if`、`for`、函数调用随手就用。

```python
def forward(self, x, use_dropout=True):
    x = self.fc1(x)
    x = torch.relu(x)
    if use_dropout:            # 动态图：Python 控制流直接可用
        x = torch.dropout(x, p=0.5, train=self.training)
    return self.fc2(x)
```

我的个人理解：动态图的本质是**让"搭网络"这件事回归到"写程序"**，框架不再强迫你用一套声明式 DSL。这对研究、调试、快速迭代极其友好——想加一个分支？写个 `if` 就完了。

## 核心概念四：nn.Module 模块化

`nn.Module` 是搭模型的"积木"基类。任何网络都可以看作模块的嵌套组合。

```python
import torch.nn as nn

class MLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(784, 128)
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
        x = torch.relu(self.fc1(x))
        return self.fc2(x)
```

我理解 `nn.Module` 真正做的事有三件：
1. **自动注册子模块和参数**：`self.fc1 = nn.Linear(...)` 一旦赋值，`model.parameters()` 就会自动收集到 `fc1` 的权重，不用手动管理。
2. **统一前向接口**：`forward` 定义后，`model(x)` 自动帮你处理一些细节（比如切换训练/评估模式）。
3. **管理状态**：`.train()` / `.eval()`、`.to(device)`、`.state_dict()` 等一整套生命周期操作。

## 一个完整的训练循环

训练的本质其实非常简单，就五步，每步一行：

```python
optimizer = torch.optim.SGD(model.parameters(), lr=0.01)
loss_fn = nn.CrossEntropyLoss()

for data, target in dataloader:
    optimizer.zero_grad()     # 1. 清空上一步的梯度
    output = model(data)      # 2. 前向传播
    loss = loss_fn(output, target)  # 3. 计算损失
    loss.backward()           # 4. 反向传播，算出梯度
    optimizer.step()          # 5. 更新参数
```

这五行代码我建议**手写十遍**。深度学习看似高大上，落到训练这一层，就是"前向算损失，反向算梯度，沿着梯度反方向走一步"。

其中 `optimizer.zero_grad()` 是最容易忽略的一步——PyTorch 的梯度是**累加**的，不清零的话每轮梯度会叠加，训练直接崩掉。这也是个人踩过的坑。

## 个人总结

最后说说我对 PyTorch 的整体观感：

1. **它首先是个科学的玩具，其次才是工业工具。** 动态图 + Python 生态让它成为研究首选，部署上虽有 TorchScript 等方案，但历史包袱比某些框架重。
2. **理解自动微分 > 记住 API。** 只要想清楚"计算图 + 链式法则"，遇到任何自定义损失、自定义层的需求，心里就有底。
3. **先会套模板，再拆模板。** 训练循环、`nn.Module` 的写法都是高度模板化的，先跑通，再去抠每一行的原理，是效率最高的学习路径。

一句话收尾：**PyTorch 让"从想法到训练出模型"的路径变得足够短，剩下的，就是你的数据和想法了。**
