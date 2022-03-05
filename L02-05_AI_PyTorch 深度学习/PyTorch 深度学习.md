# PyTorch 深度学习

PyTorch 是 Facebook 在 2017 年发布的一个开源库，可以用来编写 Python 深度学习代码。

由于结合了 Torch 的特性和 Chainer 的理念（借鉴了理念，而不是具体代码），这使得 PyTorch 在这几年大为流行。

PyTorch 提供了很多模块，可以帮助管理文本、图像和音频（torchtext、torchvision、torchaudio），还有诸如 ResNet 等流行架构的内置实现（带权重，可以下载权重从而为迁移学习等技术提供帮助）。

## Web Links

https://oreil.ly/pytorch-github



## 为什么 PyTorch 而不是 TensorFlow

传统 TensorFlow 的工作方式与 PyTorch 不同，这对编写代码和调试有一些重要影响。

在 TensorFlow 中，要使用这个库建立神经网络架构的一个图表示，然后在这个图上执行操作，这在 TensorFlow 库中进行。这种声明式编程方法与 Python 的命令式编程模式有些不一致。

另一个问题是，由于使用静态图声明，与 PyTorch 相比，这使得训练和推理时动态修改架构要复杂得多，而且充斥大量要板代码。

不过，TensorFlow 的更新版中发生了一些变化。最近增加了一个名为动态图机制（eager execution）的新特性，允许 TensorFlow 采用与 PyTorch 类似的工作方式。

PyTorch 是实现深度学习和可微分编程的一种更精简、更集中的方法。因为它不用继续支持更老的 API，所以与 TensorFlow 相比，用 PyTorch 更容易学习，也更有效率。



## Keras 和 PyTorch 有什么关系

Keras 是原先支持 Theano 和 TensorFlow 的一个高层深度学习库，现在还支持另外一些框架，如 Apache MXNet。Keras 提供了一些特性，如训练、验证、测试循环等，另外还提供了建立神经网络架构的一些简单方法。

Keras 为 TensorFlow 的广泛应用做出了巨大贡献，现在已经成为 TensorFlow 本身的一部分（即 tf.keras），而且继续作为一个单独的项目在进行。

相比之下，PyTorch 是介于底层 TensorFlow 和 Keras 之间的一个中间层，我们必须写自己的训练和推理例程，但是创建神经网络几乎同样简单。另外需要指出，对于 Python 开发人员来说，PyTorch 建立和重用架构的方法比 Keras 的 "魔法" 更符合逻辑。



