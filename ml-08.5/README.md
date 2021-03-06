# 写给程序员的机器学习入门 (八 补充) - 使用 GPU 训练模型

在之前的文章中我训练模型都是使用的 CPU，因为家中黄脸婆不允许我浪费钱买电脑😭。终于的，附近一个废品回收站的朋友转让给我一台破烂旧电脑，所以我现在可以体验使用 GPU 训练模型了🥳。

## 显卡要求

pytorch, tensorflow 等主流的框架的 GPU 支持都基于 CUDA 框架，而目前提供 CUDA 支持的显卡只有 nvidia，这次我捡到的破烂是 GTX 1650 4GB 所以满足最低要求了。简单描述下目前各种显卡的支持程度：

- Intel 核显：死心叭
- APU：没法用
- Nvidia Geforce
    - 2GB 可以用来跑一些入门例子
    - 4GB 可以跑一些简单模型
    - 6GB 可以跑一些中级模型
    - 8GB 可以跑一些高级模型
    - 10GB以上 可以跑最前沿的模型
- Radeon：要折腾，试试 [ROCm](https://github.com/RadeonOpenCompute/ROCm)

如果真的要玩机器学习推荐购买 RTX 系列，因为有 tensor 核心和 16 位浮点数支持，训练速度会快很多，并且使用 16 位浮点数可以让显存占用少一半。虽然在过几个星期就可以看到 3000 系列的显卡了，可惜没钱买🤒。此外，明年如果出支持机器学习的民用国产显卡必定会大力支持😡。

## 安装显卡驱动

Windows 的话会通过 Windows Update 自动安装，pytorch 会自动检测出显卡，不需要做任何工作。Linux 需要安装 Nvidia 官方的闭源驱动 (开源的 Nouveau 驱动不支持 CUDA)，如果是 Ubuntu 那么在安装系统的时候打个勾就可以自动安装，如果没打可以参考[这篇文章](https://linoxide.com/ubuntu-how-to/how-to-install-nvidia-driver-on-ubuntu/)，其他 Linux 系统如果源没有提供可以去 Nvidia [官方下载驱动](https://www.nvidia.com/en-us/drivers/unix/)。

安装以后可以执行以下代码看看 pytorch 是否可以检测出显卡：

``` python
>>> import torch

# 判读是否有 GPU 支持
>>> torch.cuda.is_available()
True

# 判断插了几张可用的显卡
>>> torch.cuda.device_count()
1

# 获取第一张显卡的名称
>>> torch.cuda.get_device_name(0)
'GeForce GTX 1650'
```

如果输出类似以上的结果，那么就代表没有问题了。

## 在 pytorch 中使用 GPU

pytorch 默认会把 tensor 对象的数据保存在内存上，计算会由 CPU 执行，如果我们想使用 GPU，可以调用 tensor 对象的 `cuda` 方法把对象的数据复制到显存上，复制以后的 tensor 对象运算会使用 GPU。注意在内存上的 tensor 对象和在显存上的 tensor 对象之间无法进行运算。

``` python
# 创建一个 tensor，默认会保存在内存上，由 CPU 进行计算
>>> a = torch.tensor([1,2,3])
>>> a
tensor([1, 2, 3])

# 把 tensor 复制到显存上，针对此 tensor 的计算将会使用 GPU
>>> b = a.cuda()
>>> b
tensor([1, 2, 3], device='cuda:0')
```

如果你想编写同时兼容 GPU 和 CPU 的代码可以使用以下写法，如果有支持的 GPU 则会使用 GPU，如果没有则会使用 CPU：

``` python
# 创建一个 device 对象，如果显卡可用则指向显卡，否则指向 CPU
>>> device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# 创建一个 tensor 并复制到指定 device
>>> a = torch.tensor([1,2,3])
>>> b = a.to(device)
>>> a
tensor([1, 2, 3])
>>> b
tensor([1, 2, 3], device='cuda:0')
```

如果你插了多张显卡，以上的写法只会使用第一张，你可以通过 "cuda:序号" 来指定不同的显卡来实现分布式计算。

``` python
>>> device1 = torch.device("cuda:0")
>>> device1
device(type='cuda', index=0)

>>> device2 = torch.device("cuda:1")
>>> device2
device(type='cuda', index=1)
```

## 使用 GPU 训练识别验证码的模型

这里我拿前一篇文章的代码来展示怎样实际使用 GPU 训练识别验证码的模型，以下是修改后完整的代码：

如何生成训练数据和如何使用这份代码的说明请参考前一篇文章。

``` python
import os
import sys
import torch
import gzip
import itertools
import random
import numpy
import json
from PIL import Image
from torch import nn
from matplotlib import pyplot

# 分析目标的图片大小，全部图片都会先缩放到这个大小
# 验证码原图是 120x50
IMAGE_SIZE = (56, 24)
# 分析目标的图片所在的文件夹
IMAGE_DIR = "./generate-captcha/output/"
# 字母数字列表
ALPHA_NUMS = "abcdefghijklmnopqrstuvwxyz0123456789"
ALPHA_NUMS_MAP = { c: index for index, c in enumerate(ALPHA_NUMS) }
# 验证码位数
DIGITS = 4
# 标签数量，字母数字混合*位数
NUM_LABELS = len(ALPHA_NUMS)*DIGITS

# 用于启用 GPU 支持
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

class BasicBlock(nn.Module):
    """ResNet 使用的基础块"""
    expansion = 1 # 定义这个块的实际出通道是 channels_out 的几倍，这里的实现固定是一倍
    def __init__(self, channels_in, channels_out, stride):
        super().__init__()
        # 生成 3x3 的卷积层
        # 处理间隔 stride = 1 时，输出的长宽会等于输入的长宽，例如 (32-3+2)//1+1 == 32
        # 处理间隔 stride = 2 时，输出的长宽会等于输入的长宽的一半，例如 (32-3+2)//2+1 == 16
        # 此外 resnet 的 3x3 卷积层不使用偏移值 bias
        self.conv1 = nn.Sequential(
            nn.Conv2d(channels_in, channels_out, kernel_size=3, stride=stride, padding=1, bias=False),
            nn.BatchNorm2d(channels_out))
        # 再定义一个让输出和输入维度相同的 3x3 卷积层
        self.conv2 = nn.Sequential(
            nn.Conv2d(channels_out, channels_out, kernel_size=3, stride=1, padding=1, bias=False),
            nn.BatchNorm2d(channels_out))
        # 让原始输入和输出相加的时候，需要维度一致，如果维度不一致则需要整合
        self.identity = nn.Sequential()
        if stride != 1 or channels_in != channels_out * self.expansion:
            self.identity = nn.Sequential(
                nn.Conv2d(channels_in, channels_out * self.expansion, kernel_size=1, stride=stride, bias=False),
                nn.BatchNorm2d(channels_out * self.expansion))

    def forward(self, x):
        # x => conv1 => relu => conv2 => + => relu
        # |                              ^
        # |==============================|
        tmp = self.conv1(x)
        tmp = nn.functional.relu(tmp)
        tmp = self.conv2(tmp)
        tmp += self.identity(x)
        y = nn.functional.relu(tmp)
        return y

class MyModel(nn.Module):
    """识别验证码 (ResNet-18)"""
    def __init__(self, block_type = BasicBlock):
        super().__init__()
        # 记录上一层的出通道数量
        self.previous_channels_out = 64
        # 把 3 通道转换到 64 通道，长宽不变
        self.conv1 = nn.Sequential(
            nn.Conv2d(3, self.previous_channels_out, kernel_size=3, stride=1, padding=1, bias=False),
            nn.BatchNorm2d(self.previous_channels_out))
        # ResNet 使用的各个层
        self.layer1 = self._make_layer(block_type, channels_out=64, num_blocks=2, stride=1)
        self.layer2 = self._make_layer(block_type, channels_out=128, num_blocks=2, stride=2)
        self.layer3 = self._make_layer(block_type, channels_out=256, num_blocks=2, stride=2)
        self.layer4 = self._make_layer(block_type, channels_out=512, num_blocks=2, stride=2)
        # 把最后一层的长宽转换为 1x1 的池化层，Adaptive 表示会自动检测原有长宽
        # 例如 B,512,4,4 的矩阵会转换为 B,512,1,1，每个通道的单个值会是原有 16 个值的平均
        self.avgPool = nn.AdaptiveAvgPool2d((1, 1))
        # 全连接层，只使用单层线性模型
        self.fc_model = nn.Linear(512 * block_type.expansion, NUM_LABELS)
        # 控制输出在 0 ~ 1 之间，BCELoss 需要
        # 因为每组只应该有一个值为真，使用 softmax 效果会比 sigmoid 好
        self.softmax = nn.Softmax(dim=2)

    def _make_layer(self, block_type, channels_out, num_blocks, stride):
        blocks = []
        # 添加第一个块
        blocks.append(block_type(self.previous_channels_out, channels_out, stride))
        self.previous_channels_out = channels_out * block_type.expansion
        # 添加剩余的块，剩余的块固定处理间隔为 1，不会改变长宽
        for _ in range(num_blocks-1):
            blocks.append(block_type(self.previous_channels_out, self.previous_channels_out, 1))
            self.previous_channels_out *= block_type.expansion
        return nn.Sequential(*blocks)

    def forward(self, x):
        # 转换出通道到 64
        tmp = self.conv1(x)
        tmp = nn.functional.relu(tmp)
        # 应用 ResNet 的各个层
        tmp = self.layer1(tmp)
        tmp = self.layer2(tmp)
        tmp = self.layer3(tmp)
        tmp = self.layer4(tmp)
        # 转换长宽到 1x1
        tmp = self.avgPool(tmp)
        # 扁平化，维度会变为 B,512
        tmp = tmp.view(tmp.shape[0], -1)
        # 应用全连接层
        tmp = self.fc_model(tmp)
        # 划分每个字符对应的组，之后维度为 batch_size, digits, alpha_nums
        tmp = tmp.reshape(tmp.shape[0], DIGITS, len(ALPHA_NUMS))
        # 应用 softmax 到每一组
        tmp = self.softmax(tmp)
        # 重新扁平化，之后维度为 batch_size, num_labels
        y = tmp.reshape(tmp.shape[0], NUM_LABELS)
        return y

def save_tensor(tensor, path):
    """保存 tensor 对象到文件"""
    torch.save(tensor, gzip.GzipFile(path, "wb"))

def load_tensor(path):
    """从文件读取 tensor 对象"""
    return torch.load(gzip.GzipFile(path, "rb"))

def image_to_tensor(img):
    """转换图片对象到 tensor 对象"""
    in_img = img.resize(IMAGE_SIZE)
    in_img = in_img.convert("RGB") # 转换图片模式到 RGB
    arr = numpy.asarray(in_img)
    t = torch.from_numpy(arr)
    t = t.transpose(0, 2) # 转换维度 H,W,C 到 C,W,H
    t = t / 255.0 # 正规化数值使得范围在 0 ~ 1
    return t

def code_to_tensor(code):
    """转换验证码到 tensor 对象，使用 onehot 编码"""
    t = torch.zeros((NUM_LABELS,))
    code = code.lower() # 验证码不分大小写
    for index, c in enumerate(code):
        p = ALPHA_NUMS_MAP[c]
        t[index*len(ALPHA_NUMS)+p] = 1
    return t

def tensor_to_code(tensor):
    """转换 tensor 对象到验证码"""
    tensor = tensor.reshape(DIGITS, len(ALPHA_NUMS))
    indices = tensor.max(dim=1).indices
    code = "".join(ALPHA_NUMS[index] for index in indices)
    return code

def prepare_save_batch(batch, tensor_in, tensor_out):
    """准备训练 - 保存单个批次的数据"""
    # 切分训练集 (80%)，验证集 (10%) 和测试集 (10%)
    random_indices = torch.randperm(tensor_in.shape[0])
    training_indices = random_indices[:int(len(random_indices)*0.8)]
    validating_indices = random_indices[int(len(random_indices)*0.8):int(len(random_indices)*0.9):]
    testing_indices = random_indices[int(len(random_indices)*0.9):]
    training_set = (tensor_in[training_indices], tensor_out[training_indices])
    validating_set = (tensor_in[validating_indices], tensor_out[validating_indices])
    testing_set = (tensor_in[testing_indices], tensor_out[testing_indices])

    # 保存到硬盘
    save_tensor(training_set, f"data/training_set.{batch}.pt")
    save_tensor(validating_set, f"data/validating_set.{batch}.pt")
    save_tensor(testing_set, f"data/testing_set.{batch}.pt")
    print(f"batch {batch} saved")

def prepare():
    """准备训练"""
    # 数据集转换到 tensor 以后会保存在 data 文件夹下
    if not os.path.isdir("data"):
        os.makedirs("data")

    # 查找所有图片
    image_paths = []
    for root, dirs, files in os.walk(IMAGE_DIR):
        for filename in files:
            path = os.path.join(root, filename)
            if not path.endswith(".png"):
                continue
            # 验证码在文件名中，例如
            # 00000-R865.png => R865
            code = filename.split(".")[0].split("-")[1]
            image_paths.append((path, code))

    # 打乱图片顺序
    random.shuffle(image_paths)

    # 分批读取和保存图片
    batch_size = 1000
    for batch in range(0, len(image_paths) // batch_size):
        image_tensors = []
        image_labels = []
        for path, code in image_paths[batch*batch_size:(batch+1)*batch_size]:
            with Image.open(path) as img:
                image_tensors.append(image_to_tensor(img))
            image_labels.append(code_to_tensor(code))
        tensor_in = torch.stack(image_tensors) # 维度: B,C,W,H
        tensor_out = torch.stack(image_labels) # 维度: B,N
        prepare_save_batch(batch, tensor_in, tensor_out)

def train():
    """开始训练"""
    # 创建模型实例
    model = MyModel().to(device)

    # 创建损失计算器
    # 计算多分类输出最好使用 BCELoss
    loss_function = torch.nn.BCELoss()

    # 创建参数调整器
    optimizer = torch.optim.Adam(model.parameters())

    # 记录训练集和验证集的正确率变化
    training_accuracy_history = []
    validating_accuracy_history = []

    # 记录最高的验证集正确率
    validating_accuracy_highest = -1
    validating_accuracy_highest_epoch = 0

    # 读取批次的工具函数
    def read_batches(base_path):
        for batch in itertools.count():
            path = f"{base_path}.{batch}.pt"
            if not os.path.isfile(path):
                break
            yield [ t.to(device) for t in load_tensor(path) ]

    # 计算正确率的工具函数
    def calc_accuracy(actual, predicted):
        # 把每一位的最大值当作正确字符，然后比对有多少个字符相等
        actual_indices = actual.reshape(actual.shape[0], DIGITS, len(ALPHA_NUMS)).max(dim=2).indices
        predicted_indices = predicted.reshape(predicted.shape[0], DIGITS, len(ALPHA_NUMS)).max(dim=2).indices
        matched = (actual_indices - predicted_indices).abs().sum(dim=1) == 0
        acc = matched.sum().item() / actual.shape[0]
        return acc
 
    # 划分输入和输出的工具函数
    def split_batch_xy(batch, begin=None, end=None):
        # shape = batch_size, channels, width, height
        batch_x = batch[0][begin:end]
        # shape = batch_size, num_labels
        batch_y = batch[1][begin:end]
        return batch_x, batch_y

    # 开始训练过程
    for epoch in range(1, 10000):
        print(f"epoch: {epoch}")

        # 根据训练集训练并修改参数
        # 切换模型到训练模式，将会启用自动微分，批次正规化 (BatchNorm) 与 Dropout
        model.train()
        training_accuracy_list = []
        for batch_index, batch in enumerate(read_batches("data/training_set")):
            # 切分小批次，有助于泛化模型
            training_batch_accuracy_list = []
            for index in range(0, batch[0].shape[0], 100):
                # 划分输入和输出
                batch_x, batch_y = split_batch_xy(batch, index, index+100)
                # 计算预测值
                predicted = model(batch_x)
                # 计算损失
                loss = loss_function(predicted, batch_y)
                # 从损失自动微分求导函数值
                loss.backward()
                # 使用参数调整器调整参数
                optimizer.step()
                # 清空导函数值
                optimizer.zero_grad()
                # 记录这一个批次的正确率，torch.no_grad 代表临时禁用自动微分功能
                with torch.no_grad():
                    training_batch_accuracy_list.append(calc_accuracy(batch_y, predicted))
            # 输出批次正确率
            training_batch_accuracy = sum(training_batch_accuracy_list) / len(training_batch_accuracy_list)
            training_accuracy_list.append(training_batch_accuracy)
            print(f"epoch: {epoch}, batch: {batch_index}: batch accuracy: {training_batch_accuracy}")
        training_accuracy = sum(training_accuracy_list) / len(training_accuracy_list)
        training_accuracy_history.append(training_accuracy)
        print(f"training accuracy: {training_accuracy}")

        # 检查验证集
        # 切换模型到验证模式，将会禁用自动微分，批次正规化 (BatchNorm) 与 Dropout
        model.eval()
        validating_accuracy_list = []
        for batch in read_batches("data/validating_set"):
            batch_x, batch_y = split_batch_xy(batch)
            predicted = model(batch_x)
            validating_accuracy_list.append(calc_accuracy(batch_y, predicted))
        validating_accuracy = sum(validating_accuracy_list) / len(validating_accuracy_list)
        validating_accuracy_history.append(validating_accuracy)
        print(f"validating accuracy: {validating_accuracy}")

        # 记录最高的验证集正确率与当时的模型状态，判断是否在 20 次训练后仍然没有刷新记录
        if validating_accuracy > validating_accuracy_highest:
            validating_accuracy_highest = validating_accuracy
            validating_accuracy_highest_epoch = epoch
            save_tensor(model.state_dict(), "model.pt")
            print("highest validating accuracy updated")
        elif epoch - validating_accuracy_highest_epoch > 20:
            # 在 20 次训练后仍然没有刷新记录，结束训练
            print("stop training because highest validating accuracy not updated in 20 epoches")
            break

    # 使用达到最高正确率时的模型状态
    print(f"highest validating accuracy: {validating_accuracy_highest}",
        f"from epoch {validating_accuracy_highest_epoch}")
    model.load_state_dict(load_tensor("model.pt"))

    # 检查测试集
    testing_accuracy_list = []
    for batch in read_batches("data/testing_set"):
        batch_x, batch_y = split_batch_xy(batch)
        predicted = model(batch_x)
        testing_accuracy_list.append(calc_accuracy(batch_y, predicted))
    testing_accuracy = sum(testing_accuracy_list) / len(testing_accuracy_list)
    print(f"testing accuracy: {testing_accuracy}")

    # 显示训练集和验证集的正确率变化
    pyplot.plot(training_accuracy_history, label="training")
    pyplot.plot(validating_accuracy_history, label="validing")
    pyplot.ylim(0, 1)
    pyplot.legend()
    pyplot.show()

def eval_model():
    """使用训练好的模型"""
    # 创建模型实例，加载训练好的状态，然后切换到验证模式
    model = MyModel().to(device)
    model.load_state_dict(load_tensor("model.pt"))
    model.eval()

    # 询问图片路径，并显示可能的分类一览
    while True:
        try:
            # 构建输入
            image_path = input("Image path: ")
            if not image_path:
                continue
            with Image.open(image_path) as img:
                tensor_in = image_to_tensor(img).to(device).unsqueeze(0) # 维度 C,W,H => 1,C,W,H
            # 预测输出
            tensor_out = model(tensor_in)
            # 转换到验证码
            code = tensor_to_code(tensor_out[0])
            print(f"code: {code}")
            print()
        except Exception as e:
            print("error:", e)

def main():
    """主函数"""
    if len(sys.argv) < 2:
        print(f"Please run: {sys.argv[0]} prepare|train|eval")
        exit()

    # 给随机数生成器分配一个初始值，使得每次运行都可以生成相同的随机数
    # 这是为了让过程可重现，你也可以选择不这样做
    random.seed(0)
    torch.random.manual_seed(0)

    # 根据命令行参数选择操作
    operation = sys.argv[1]
    if operation == "prepare":
        prepare()
    elif operation == "train":
        train()
    elif operation == "eval":
        eval_model()
    else:
        raise ValueError(f"Unsupported operation: {operation}")

if __name__ == "__main__":
    main()
```

使用 diff 生成相差的部分如下：

``` text
$ diff -U3 example.py.old example.py
@@ -23,6 +23,9 @@
 # 标签数量，字母数字混合*位数
 NUM_LABELS = len(ALPHA_NUMS)*DIGITS
 
+# 用于启用 GPU 支持
+device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
+
 class BasicBlock(nn.Module):
     """ResNet 使用的基础块"""
     expansion = 1 # 定义这个块的实际出通道是 channels_out 的几倍，这里的实现固定是一倍
@@ -203,7 +206,7 @@
 def train():
     """开始训练"""
     # 创建模型实例
-    model = MyModel()
+    model = MyModel().to(device)
 
     # 创建损失计算器
     # 计算多分类输出最好使用 BCELoss
@@ -226,7 +229,7 @@
             path = f"{base_path}.{batch}.pt"
             if not os.path.isfile(path):
                 break
-            yield load_tensor(path)
+            yield [ t.to(device) for t in load_tensor(path) ]
 
     # 计算正确率的工具函数
     def calc_accuracy(actual, predicted):
@@ -327,7 +330,7 @@
 def eval_model():
     """使用训练好的模型"""
     # 创建模型实例，加载训练好的状态，然后切换到验证模式
-    model = MyModel()
+    model = MyModel().to(device)
     model.load_state_dict(load_tensor("model.pt"))
     model.eval()
 
@@ -339,7 +342,7 @@
             if not image_path:
                 continue
             with Image.open(image_path) as img:
-                tensor_in = image_to_tensor(img).unsqueeze(0) # 维度 C,W,H => 1,C,W,H
+                tensor_in = image_to_tensor(img).to(device).unsqueeze(0) # 维度 C,W,H => 1,C,W,H
             # 预测输出
             tensor_out = model(tensor_in)
             # 转换到验证码
```

可以看到只改动了五个部分，在头部添加了 device 的定义，然后在加载模型和 tensor 对象的时候使用 `.to(device)` 即可。

简单吧☺️。

那么训练速度相差如何呢？只训练一个 batch 使用 CPU 和 GPU 消耗的时间分别如下 (单位秒)：

``` text
CPU: 13.60
GPU: 1.90
```

差了整整 7 倍😱，，如果是高端的显卡估计可以看到数十倍的差距。

## 显存占用

如果你想查看训练过程中的显存占用情况，可以使用 `nvidia-smi` 命令，命令会输出以下的信息：

``` text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.57       Driver Version: 450.57       CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  GeForce GTX 1650    Off  | 00000000:06:00.0  On |                  N/A |
| 60%   67C    P3    40W /  90W |   3414MiB /  3902MiB |    100%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A      1237      G   /usr/lib/xorg/Xorg                238MiB |
|    0   N/A  N/A      2545      G   cinnamon                           68MiB |
|    0   N/A  N/A      2797      G   ...AAAAAAAAA= --shared-files      103MiB |
|    0   N/A  N/A     18534      G   ...AAAAAAAAA= --shared-files       82MiB |
|    0   N/A  N/A     20035      C   python3                          2915MiB |
+-----------------------------------------------------------------------------+
```

如果训练过程中出现显存不足，你会看到以下的异常信息：

``` text
RuntimeError: CUDA error: out of memory
```

如果你遇到显存不足的问题，那么可以尝试以下的办法解决，按实用程度排序：

- 出钱买新显卡🤒
- 减少训练批次大小 (例如每个批次 100 条数据，减为每个批次 50 条数据）
- 不使用的对象早回收，例如 `predicted = None`，pytorch 会在对象声明周期结束后自动释放显存
- 计算单值的时候使用 `item()`，例如 `acc_total += acc.item()`，但配合 `backward` 生成运算路径的计算不能用
- 如果你使用桌面 Linux，试试开机的时候添加 `rw init=/bin/bash` 进入命令行界面再训练，这样可以节省个几百 MB 显存

你可能会好奇为什了 pytorch 可以及时释放显存，这是因为 python 的对象使用了引用计数 (Reference Counted)，GC 基本上只负责回收循环引用的对象，对象的引用计数归 0 的时候 python 会自动调用析构函数，不需要等待 GC。而 NET 和 Java 等语言则无法做到及时回收，除非你每个 tensor 对象都及时的去调用 Dispose 方法，或者使用 tensorflow 来编译静态运算路径然后把生命周期管理工作全部交给框架。这也是使用 Python 的一大好处🥳。

## 写在最后

这篇本来应该放在最开始，可惜等到现在才有条件写。下一篇文章预计会介绍对象识别模型，包括 RCNN，FasterRCNN 和 YOLO，看看什么时候能出来吧。
