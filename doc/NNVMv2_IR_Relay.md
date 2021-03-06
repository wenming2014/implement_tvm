# Relay a new high level IR for TVM

[链接](https://github.com/dmlc/tvm/issues/1673)

[TOC]

**Relay是一种新的高级中间表示（IR），被看作NNVM的v2.0。**

## Motivation

计算图是第一代DL框架所展示的强大的程序表示。 最流行的框架均使用计算图作为其输入，中间表示和数据计算。

但是，随着工作负载(workload)的不断发展，高级IR的设计需要不断发展，以更好地支持开发人员和用户的需求

控制流和子图等图层级挑战已成为原生支持和优化的必要功能。

运行时表示和编译时表示之间的紧密耦合具有有限的灵活性让开发人员苦恼; Reley将解耦两个表示。

最后，我们认为高层图必须与底层IR一起设计，允许两个层在编译期间进行通信以实现最佳性能。

## Design

NNVM的第一个版本旨在解决其中的一些挑战，我们将Relay视为第二代IR专门设计用于集成到TVM堆栈中作为输入层。 我们的目标是将TVM作为我们的主要后端(relay作为TVM的前端之一)，简化TVM开发人员和当前NNVM用户的开发和维护，以及启用新功能。

为了解决上面提出的挑战，我们设计了Relay来构建计算图表擅长的东西（干净，数据流，组件），并改进他们所挣扎的东西（控制流，子图，区分运行时/编译时）。

## Core IR

Relay是一种类型化的纯函数IR，具有一些基本功能，如函数，if-then-else控制流，递归，运算符和函数调用以及变量绑定。

在过去的8个月里，我们重复了Relay的设计。 这个版本代表了我们实验的顶峰。 此PR不包含先前版本的所有部分，而是专注于引入核心IR，其相关数据构建和一些完整的pass。

核心IR仅在几个文件中定义：

- include / tvm / relay / base.h（基类和公共数据）
- include / tvm / relay / type.h（类型系统和所有相关节点）
- include / tvm / relay / expr.h（表达式语言）

### Typing

所有Relay程序都是键入的，类似于更传统的语言，如C++。
类型系统允许我们静态地（即在编译时）区分不同种类的值。 这意味着我们知道一个表达式是否会计算一个张量，一个函数（即（float32，float32） - > float32）或一个元组（float32，int32）。 此外，我们的类型系统具有形状通用(shape generic)的能力（即多态，模板）。

在传统的计算图形样式IR中，类型推断和检查取代了形状(shape)推理。

这个PR实现了类型推断和检查Relay，代码可以在src / tvm / relay / pass / type_infer.cc中找到，相关的帮助工具在src / tvm / relay / pass中找到。

### Control Flow

Relay以简单的if（cond）{true_branch} else {false_branch}的形式向IR添加控制流的概念。 Relay要求条件变量计算是控制采用哪个分支的单个布尔值。 if是Relay中的表达式，意味着整个表达式的结果是分支的结果。

我们引入这个来添加一种区分数据流和控制流的正式方法，而不必将两者混合在表示中。 因为我们将控制信号分开，所以我们可以轻松地批处理程序而不会影响控制流程。

控制流的定义可以在include / tvm / relay / expr.h中找到。

### Abstraction

Relay支持函数的定义，可用于表示“子图”（即可重用计算的块）。

Relay函数类似于传统函数：它们代表一组参数（即占位符）和一个函数体，它是涉及参数（即子图）的计算块。 我们可以通过组合函数来构建完整的网络/模型。

### Compilation(编译)

Relay IR设计为模型的编译时表示。 新特征仅在Relay的抽象语法树中公开，并用于编译时程序操作。 我们不打算使用Relay的IR作为正式的解释或执行的数据结构。

### Runtime(运行时)

这些新特征增加了当前计算模型的表现力，人们可能会问如何使用现有运行时使用这些功能来执行程序。 我们的目标是在此PR中引入Relay作为编译器表示，并在前端和后端重用现有的运行时维护兼容性。 我们预计新版本的运行时将在未来为Relay的新版本提供原生支持。

## TVM Co-design

我们努力在TVM之后对Relay的实现进行建模，并重用大部分现有基础设施，以便在TOPI 操作库和Relay计划之间提供更好的兼容性。 一个重大的设计决策是重用TVM节点系统以TVM的方式将Relay语言暴露给Python。 熟悉TVM表达式语言的用户应该会习惯使用C ++和Python中的Relay AST定义。 我们还共享许多数据结构的表示。 例如，张量容器（即tvm :: runtime :: NDArray）,像共享结构这样的通用属性可以在Relay和TVM之间共享.

## Transitioning from NNVM

我们计划添加一个指南，用于将程序从NNVM转换到Relay。 这是发布Relay Alpha之前的剩余工作项之一。 目标是用户可以使用Relay运算符和构建器API来构建Relay程序，我们将跟进兼容层，以便顺利地从NNVM过渡。

有关实现，请参阅[＃1672](https://github.com/dmlc/tvm/pull/1672)实现这一点。

## Specific Technical Points(关键技术点)

- 与TVM节点系统紧密集成。 NNVM是在tvm之前设计的，所以我们没有考虑到tvm运行时。这使得注册python回调，遍历IR和当前nnvm的交互很难。Relay重构直接带来这种紧密集成，现在每个IR都可以访问，检查，我们可以轻松地在python和c++中进行原型设计。这是一个与IR规范无关的功能，但对开发人员来说从未如此重要。
- 将Shape / dtype集成为TensorType，重用符号整数（tvm :: Expr）。 TOPI描述的一个优点是在许多情况下可以支持符号整数，因此代码在一个维度（例如批处理）上可以是通用的，TensorType也可以重用它。这使得与TOPI保持一致，并允许声明特定程序，如输入大小（n，128,128）
- 控制流，如果，（通过尾递归）和函数递归。
- 编译器时和运行时的IR分离。这又有两个原因：
  - 我们不必过多考虑编译时到运行时并保持运行时最小化
  - 我们可以保持当前的图运行时。

## Some Possible Point to Discuss

这些是从我脑海中弹出的东西，随意添加更多。

- 需要澄清的是，请说明，因为我们需要让它合宜
- 如何支持特定pass，以及当前提案的容易程度
- 具体的用例场景（例如变压器）以及我们需要的东西
- 什么有助于构成推理的最小运行时间
- 我们需要为构建JIT运行时进行train所需的任何注意事项。

