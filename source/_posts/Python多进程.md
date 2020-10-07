---
title: Python多进程
date: 2020-09-30 15:19:40
categories:
- Python
tags:
- 多进程
- multiprocessing
---

## 前言
本文记录一下python多进程包`multiprocessing`的使用、遇到的问题和注意事项。

## 使用方法
`multiprocessing`提供了`Pool`、`Queue`和`Manager`等多种模块使用多进程，这里只记录4个使用`Pool`执行下面`func()`的方法。
```py
import multiprocessing
NUM_CPU = 2
inputs = [1, 2, 3, 4]
```
### 最简单的方式
```py
def func(input):
    result = run(input)
    return result
    
with multiprocessing.Pool(NUM_CPU) as p:
    results = p.map(func, inputs)
```

### 保留顺序的方式
重新构造`inputs`，为其每一项增加一个`idx`用于得到结果后排序。同时，`func()`也需重新构造，输入需要**解包**，返回值需要**增加idx**。
```py
def func(input):
    input, idx = input
    result = run()
    return result, idx

inputs_with_idx = list(zip(inputs, range(len(inputs))))
with multiprocessing.Pool(NUM_CPU) as p:
    results = p.map(func, inputs_with_idx)
results = list(sorted(results, key=lambda x: x[1]))
results = [r[0] for r in results]
```

### 传入共享数据
有些场景下，每个进程中执行的`func()`均需要访问同一个数据，例如，`func()`中需要访问一个字典`vocab`进行查找。这可以通过类似于*2. 保留顺序的方式*中那样，将`vocab`进行复制，然后作为一个参数传入`func()`中来实现，但这种方式无疑会导致主进程的内存开销剧增。更好的方式是利用`Pool()`本身提供的参数，如下所示：
```py
def init_func(vocab_1, vocab_2):
    global g_vocab_1
    global g_vocab_2
    g_vocab_1 = vocab_1
    g_vocab_2 = vocab_2

def func(input):
    a = g_vocab_1[input]
    b = g_vocab_2[input]
    result = run(a, b)
    return result

with multiprocessing.Pool(NUM_CPU, initializer=init_func, initargs=(vocab_1, vocab_2)) as p:
    results = p.map(func, inputs)
```

### 显示进度
结合`imap()`与`tqdm`来显示多进程处理的进度。
```py
from tqdm import tqdm
with multiprocessing.Pool(NUM_CPU) as p:
    results = list(tqdm(
        p.imap(func, inputs), total=len(inputs)
    ))
```


## 遇到过的问题
### 时间开销异常
使用多进程时，遇到最多的问题，就是时间开销异常，包括多进程时间开销**未达到预期的那么低**、**和单进程开销一样**以及**比单进程开销高**。一开始发现时间开销异常时，简单地以为是进程间的通信导致的，直到后来才发现原因。

- 开销未达到预期那么低
    通过查阅一些资料，发现是代码写法存在问题。当时的代码中，需要用多进程执行的是**类成员函数**，而类中的**类成员变量**`self.xxx`存储了大量的数据。每次创建一个新的进程时，所有成员变量均需要**序列化**和**反序列化**（多进程自身的机制），这便导致了大量的时间开销。
    **解决方法：**
    (1) 避免类成员变量存储大量的数据，特别是超过4G的数据时，pickle的序列化将会出错。
    (2) 为类成员函数加上`@staticmethod`修饰器，将其转换为静态函数，从而避免传入参数`self`。必要时，可以通过`initializer`来传入一些必须的数据。

- 开销与单进程一样
    通过检查CPU利用率，发现实际上并没有使用多进程，实际可使用的CPU个数为1，然而代码中通过`multiprocessing.cpu_count()`得到的CPU个数却不止一个。原因是在向集群申请资源时，没有指定申请的CPU个数。未指定的情况下，即使集群的节点上存在多个CPU，但用户实际能只用的CPU只有一个。
    **解决方法：**
    申请资源时，应该通过`-n`参数指定所需CPU的个数。
    ```py
    ps -u  # 查看进程信息
    ps -o pid,psr,comm -p <pid>  # 查看进程<pid>在哪块CPU上运行。其中，psr为CPU编号。

    srun -p cpu -n2 --pty bash  # 通过slurm，向集群申请2块CPU，并进入bash环境。
    ```

- 开销比单进程要高
    `multiprocessing`似乎会与`nltk`产生冲突。当时多进程执行的函数中，调用了`nltk`的一些函数。
    **解决方法：**
    未想到很好的方法，只能替换`nltk`。

### pickle无法序列化
- 存在数据类型无法序列化
    `multiprocessing.Pool`自身机制便是要使用`pickle`将参数等数据序列化，然后在子进程中反序列化。可能存在的问题就是，有些数据类型是无法使用`pickle`序列化的，例如，曾经遇到过将`pandas`的`Series`作为参数传递时报错。
    **解决方法：**
    将不可序列化的数据转换为可序列化的，例如，将`Series`转换为`list`。

- 数据量太大，导致超过pickle能处理的上限
    这个问题实际上并没有解决，因为当时并没有找到问题究竟出现在哪里。出现问题的原因可能是当时类成员变量存储了大量的数据，导致多进程执行类成员函数时，对超过4G的数据进行了序列化。


## 注意事项
1. 因为多进程的程序在调试时更加的麻烦，因此在写多进程的程序时，要先按照单进程的程序来写，保证单进程不出错后，再将程序转换为多进程程序。
2. 在使用多进程执行**类成员函数时**，要注意**类成员变量是否存入大量数据**，同时，最好将需要多进程执行的函数转为**静态方法**。


## 总结
由于初衷仅是记录自己实验过程中对多进程的简单使用，方便以后再次使用时进行查阅，因此本文仅涉及4个简单的使用方法和2个相关问题，并没有深入分析原理。同时，由于是在事后通过回忆写下，没有真正复现错误，因此文中并没有给出具体的报错信息。将来若遇见同样的或新的问题，再进行更新和补充。

## 参考文章
[https://thelaziestprogrammer.com/python/multiprocessing-pool-expect-initret-proposal](https://thelaziestprogrammer.com/python/multiprocessing-pool-expect-initret-proposal)
[https://thelaziestprogrammer.com/python/multiprocessing-pool-a-global-solution](https://thelaziestprogrammer.com/python/multiprocessing-pool-a-global-solution)