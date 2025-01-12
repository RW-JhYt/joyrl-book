# 基本概念

本部分主要讲述`JoyRL`的基本原理与框架说明

## 概念回顾

在强化学习中，智能体（agent）在环境（environment）中与环境进行交互产生样本，并且不断更新自身的策略（policy），以获得最大化的奖励（reward），如下图：

<div align=center>
<img width="400" src="../figs/joyrl_docs/interaction_mdp.png"/>
<div align=center>图 1 智能体与环境交互的过程</div>
</div>

以`DQN`算法为例，如图2所示，智能体在与环境交互的过程中，会不断地产生样本，即`transition`，主要包括`state`、`action`、`reward`、`next_state`等等，然后将这些样本存入经验池（experience replay buffer）中，再从经验池中随机采样出一批样本，进行算法更新，更新完之后再产生一批样本，如此循环往复，直到达到终止条件。几乎所有强化学习算法都遵循这个思路，这就是强化学习的基本训练逻辑。

<div align=center>
<img width="500" src="../figs/joyrl_docs/DQN_pseu.png"/>
<div align=center>图 2 DQN算法训练伪代码</div>
</div>

目前主流的深度强化学习算法中，更新策略的过程主要是更新神经网络的参数，即模型，从这方面来看，其与深度学习是很相似的，因此很多深度学习中的问题在强化学习中也有可能遇到，例如过拟合、梯度爆炸与消失、陷入局部最优解等等。但是与深度学习不同的是，强化学习用来更新模型的样本是需要不断地通过与环境交互产生的，而不像深度学习那样是一次性准备好的。尽管有些离线强化学习正在研究如何尽量避免与环境交互，但是目前大部分强化学习算法仍然是在线的，这也是强化学习与深度学习的一个重要区别。

从实际训练过程来看，强化学习的训练过程往往比较复杂，需要考虑很多因素，比如如何与环境交互、如何存储样本、如何更新策略、如何设置终止条件等等。这些因素都会影响到训练的效果，因此需要对这些因素进行统一的管理，这也是`JoyRL`设计的初衷之一。与此同时，强化学习中的样本数据流向也比较复杂，仅仅是`DQN`算法先将样本存到经验回放中，然后再从经验回放中随机采样出一批样本进行更新，这样的数据流向就已经比较复杂了，而对于一些复杂的算法，比如`HER`、`PER`、`PPO`等等，数据流向则会更加复杂。为了让用户能够自定义一些样本的处理方式，`JoyRL`也提供了相应的接口，用户可以根据自己的需求自定义一些样本的处理方式，具体会在后面的部分中详细介绍。

## 交互器与学习器

前面提到，强化学习的训练过程主要包括交互采样和策略更新两个过程，因此`JoyRL`中这两个过程分别抽象成两个模块，即交互器（interactor）和学习器（learner）。如图3所示，交互器每次从学习器中获取更新的策略，然后与环境交互产生样本(experiences)。学习器则不断从交互器中获取样本，然后进行算法更新。其中，交互器在有些资料中也被称为`worker`，这里为了与`JoyRL`中的命名保持一致，统一称为交互器和学习器。

<div align=center>
<img width="600" src="../figs/joyrl_docs/interactor_learner.png"/>
<div align=center>图 3 交互器与学习器</div>
</div>

对比图1和图3，可以看到为了方便实际训练，`JoyRL`中将图1中的智能体拆分成了图3中的交互器和学习器。这是因为实际训练中，交互采样和策略更新是可以分开并行的，这样可以提高训练效率，拆分成两个模块之后，同时也方便用户自定义一些样本的处理方式。

## 收集器与模型管理器

图3中的数据流向基本都是单向的，即更新的策略只从学习器流向交互器，而交互样本只从交互器流向学习器。但是在实际训练过程中，有些数据是需要双向流动的，比如在`PER`算法中，交互样本中的`reward`、`done`等信息需要传回学习器，用来更新经验回放中的`priority`，这样才能保证`PER`算法的正常运行。另外`PPO`算法，在PPO算法中，策略更新的过程中，需要将`log_prob`传回学习器，用来计算`ratio`。但是交互器和学习器都分别有自己的主要任务，只在这两个模块中同时进行数据的收集和处理也会降低训练效率，因此我们定义了一个收集器（collector）模块，用来管理交互样本和策略样本的收集和处理，起到一个数据中间站的作用。此外，由于更新模型即神经网络反向传播时往往会占用较多的时间，因此我们定义了一个模型管理器（model manager）模块，用来管理模型的保存和加载，这样可以避免在训练过程中频繁地保存和加载模型，从而进一步提高训练效率。

如图4所示，模型管理在不断给交互器提供更新的策略的同时，也会不断地从学习器中获取更新的模型，然后保存到本地。收集器则不断地从交互器中获取交互样本和策略样本，然后将交互样本存入经验回放中，将策略样本传回学习器，用来更新经验回放中的`priority`。这样，交互器和学习器就可以专注于自己的主要任务，而免于频繁地进行数据的收集和处理，从而提高训练效率。

<div align=center>
<img width="600" src="../figs/joyrl_docs/collector.png"/>
<div align=center>图 4 收集器与模型管理器</div>
</div>

## 数据追踪器与记录器

在进行复杂的训练时，为了提高效率，必然要提高交互器和学习器的数量，有些信息例如当前回合数和更新的步数需要全局共享，因此我们定义了一个数据追踪器（tracker）模块，用来追踪这些信息。此外，为了方便用户对训练过程进行监控，我们定义了一个记录器（recorder）模块，用来记录训练过程中产生的奖励曲线、损失曲线等等，并将这些信息写到`tensorboard`等文件中，方便用户进行监控。

如图5所示，其中追踪器是所有模块共享的，因此没有在图中画出。记录器则主要从交互器和学习器中获取奖励和损失等信息，然后记录到本地文件中。

<div align=center>
<img width="600" src="../figs/joyrl_docs/recorder.png"/>
<div align=center>图 5 数据追踪器与记录器</div>
</div>

## 在线测试器

由于强化学习的训练过程往往比较复杂且不稳定，因此在训练过程中，我们往往需要对策略进行定期的测试，以便于及时发现问题。因此，我们定义了一个在线测试器（online tester）模块，用来在线测试策略的性能。在线测试器会定期从模型管理器中获取更新的模型，然后在测试环境中测试策略的性能，测试完之后会将测试结果记录到记录器中。

如图 6 所示，在线测试器更像是一个独立的模块，它并不影响训练过程，因此在图中特别将其分离出来。

<div align=center>
<img width="600" src="../figs/joyrl_docs/recorder.png"/>
<div align=center>图 6 整体框架</div>
</div>

到这里，`JoyRL`的整体框架就介绍完了，感兴趣的读者可以参考`JoyRL`的源码。

## 目录树

`JoyRL`的目录树如下所示：

```python
|-- joyrl
    |-- algos # 算法文件夹
        |-- [Algorithm name] # 指代算法名称比如DQN等
            |-- config.py # 存放每个算法的默认参数设置
                |-- class AlgoConfig # 算法参数设置的类
            |-- policy.py # 存放策略
            |-- data_handler.py # 存放数据处理器
    |-- framework # 框架文件夹，存放一些模块等
        |-- envs # 存放内置的环境
            |-- gym # 存放gym环境
                |-- config.py # 存放gym环境的默认参数设置
        |-- config.py # 存放通用参数设置
|-- presets # 预设的参数，对应的结果存放在benchmarks下面
|-- benchmarks # 存放训练好的结果
|-- docs # 说明文档目录
|-- tasks # 训练的时候会自动生成
|-- README.md # 项目README
|-- requirements.txt # Pyhton依赖列表
```
