---
title: Android 项目中使用 MVVM 模式的一些思考
date: 2020-04-12 22:51:45
tags:
    - Android
    - MVVM
---

目前在维护的项目中，有很多页面的代码比较久远，将网络请求、数据处理、页面布局等代码都写在一个 Fragment 中。在项目前期需求比较简单的时候，这样确实能快速开发页面完成需求，但在后面维护的过程中则带来了不便。后来有重构这些页面的机会，在此记录一下实际遇到的问题，以及自己的思考过程。

在我的项目中，页面通常是一个数据列表，通过网络加载数据，然后在页面上显示多个种类的卡片；页面的业务可能需要根据需要对返回数据进行预处理，或者为不同块的数据卡片加上头、尾；同时需要响应用户交互，实时更新数据并且将更新后的数据同步给服务器。

## 代码写在一个页面里的弊端

网上也有不少文章分析过逻辑代码全写在一个页面类中的弊端，这里记录一下自己在项目中遇到的痛点和解决问题的思考过程。

### 逻辑混杂，难以梳理

在项目的常见页面中，一般会有 数据加载、数据处理、页面整体布局、Recycler + Adapter 等 4 部分代码逻辑存在。如果这些逻辑都写在一个代码文件中，就容易出现本该属于不同部分的变量、方法混杂在一起交替出现的情况，给阅读、维护带来了不便。

### 数据与页面强绑定

当数据加载、数据处理逻辑写在页面内中，成为这个页面的内部代码后，这些逻辑将成为这个页面的内部逻辑，其他模块无法，也不应该访问到这些逻辑。这容易导致代码多处复制的问题。

### 页面之间强依赖

数据保存在页面内部的话，如果页面之间（如 Fragment 与宿主 Activity 之间，或者同级的 Fragment 之间）需要传递数据，会很容易出现页面级别的强依赖。例如 Fragment 需要获取宿主 Activity 进行类型判断，然后强转类型之后进行交互，页面结构将失去灵活性。

### 不便于单元测试

因为业务逻辑代码都封在页面文件中，外界无法访问，需要单元测试的话就要对整个页面进行操作，而这在实际操作上会有很多问题：

1. 需要 mock 整个页面的生命周期；
2. 需要为这个页面准备好依赖页面；
3. 需要为数据加载 mock 网络请求或者数据库；
4. 输入输出过多，前置条件复杂，流程不可控，边界情况难以触发；
5. 输出不明确，页面级别的对外输出是一整个 view ，难以对其进行断言；
6. 业务逻辑代码对页面外是黑箱状态，不能对关心的核心代码块进行测试；
7. ...

种种限制几乎断绝了对其进行单元测试的可能性

## 代码结构的优化

上述问题可以总结为一个概念，就是耦合过深，代码结构不合理。在重构的过程中，我的思考重点便放在了代码结构与解耦上。

### 想要达到的目标

针对上面碰到的问题，我希望新的页面代码结构能满足这样的条件：

1. 数据加载、数据处理与页面代码分离，各自成为独立模块；
2. 各模块之间交互简单，输入输出接口尽可能少；
3. 数据存放和通信与页面脱钩，需要通信的页面不再强依赖其他页面，只根据数据进行显示。

经过一番查找资料后，我认为 MVVM 模式能满足我对页面的要求，于是尝试将 MVVM 模式应用到项目的页面中。

## MVVM 在项目中的应用

关于“ MVVM 是什么”之类的介绍我就不再赘述了，我想分享的是 MVVM 模式在我的项目中应用的经历、带来的好处和仍需解决的问题。

### 模块的相互独立

MVVM 模式将一个业务页面分成 `View` `ViewModel` `Model` 三个部分，分别承载着独立、内敛的业务逻辑：

`Model` 层负责管理数据的存取。在我的项目中，因数据来源基本都是网络加载，我会在 `Model` 层管理网络访问，维护是否有下一页、记录 offset 或者页数之类的状态。同理此处也可以负责读取数据库或者文件。

`ViewModel` 层则是整个模式的核心，负责从 `Model` 层获取原始数据后进行业务加工，以得到方便页面展现的业务数据；同时承载相应的业务逻辑，如页面响应点击事件后数据修改更新、调用 `Model` 层传递给服务端等，相当于把业务的核心逻辑都放到了 `ViewModel` 层。

`View` 层是业务页面，如用户可以直接可见、交互的 Activity 、 Fragment，其作用是从 `ViewModel` 中获取数据展现到页面上，并且响应用户的交互行为，调用 `ViewModel` 的对应业务处理入口。

把一个页面拆分为上述三层之后，形成了一个 `View` -> `ViewModel` -> `Model` 的依赖链条，前者从后者获取所需的数据，向后者反馈此时所需要的操作，而又不直接依赖后者，在实践中可以做到按需可配置和可替换，

### 通过数据驱动的代码逻辑

将业务逻辑分为多层之后，各层之间如何进行交互和信息传递将成为连接各组件的重点。谷歌为我们提供的方案是 `LiveData` ，一个自带状态管理的观察者模式，能将发生改变的数据通知给存活着的观察者。

基于这个现成的工具，我最后选择了数据驱动的项目结构，即 `ViewModel` 持有数据对应的 `LiveData`，`View` 向其注册数据的监听，在响应了用户交互后通过调用 `ViewModel` 的公开方法反馈事件，`ViewModel` 根据具体业务处理数据，然后更新到 `LiveData` 中；`View` 在监听回调中根据数据的变化来更新页面的展现。

在这种模式下，数据成为了把 `View` `ViewModel` `Model` 三部分串联起来的链条，只要收到数据更新的通知，当前模块就需要为其响应进行处理；当前模块调用依赖模块的业务入口后，只需要等待数据更新的通知回调即可。

### 使用接口的思想维护不同模块之间的交互

虽然不同模块之间都属于一个业务页面，但在开始编码之前便按照接口的思想，为各个模块之间如何进行交互确定了方案。此后为不同模块进行编码的过程中，就不再需要关心其他模块是如何实现的，只需要当前模块能确实按照方案响应交互、返回数据即可。

### 每份通知的数据完备且内敛

对于页面来说，为了使其中的代码简洁，不包含不必要的逻辑处理代码，需要 `ViewModel` 层每次发送的数据都包含整个页面的完整数据（完备性），页面如需获知当前的业务逻辑状态也可以从数据中得知，而不需要通过其他途径获取或者维护状态值（内敛性）。

### 不保存不属于自己逻辑的状态

在项目中，经常有的一个逻辑是 “如果当前正在加载请求，就不发起下一个请求”，页面就需要维护一个 “当前是否正在加载” 的状态。

业务逻辑分层之后，加载逻辑被分离到了 `Model` 层，上面所说的 “是否正在加载” 就不属于页面 `View` 的维护范畴了。因此在我的实际使用中，页面每次可能需要需要加载数据的时机都调用了对应的加载入口，传递到负责加载的 `Model` 模块，由其自己判断是否发起对应的请求。而因为这些事件的返回是通过数据更新完成的，只要页面没有接收到新的数据更新通知，那么这一次调用的发起就相当于无效，不会对页面产生影响。

## 新模式下的优缺点

基于 MVVM 模式改造后的页面，在实际使用的感受中，上述原则框架内实现代码的优缺点还是比较明显的：

### 优点

#### 数据更新入口统一

首先是将回调响应的入口收拢到了一处，所有的 UI 变化都可以对应到一种数据的改变，在开发过程中可以清楚地看到某一时刻的数据状态，与其对应的 UI 展现应该是怎样的；并且因为每次收到更新通知的数据都是全量、完备的，可以方便地使用 `DiffUtil` 来进行数据变动的检查，体现在 UI 界面上。

#### 输入、输出收敛可控

逻辑分层之后，对于每个模块的输入输出点变得收敛且可控，方便进行单元测试；

以业务逻辑最复杂的 `VideModel` 为例，需要进行单元测试时，只需要为其mock一个 `Model` 模块，不需要实际进行或者mock网络请求，而是按需从 mock 的 `Model` 模块中直接向 `ViewModel` 发送测试数据，从  `ViewModel` 的唯一出口 `LiveData` 中注册更新通知，获取处理过后的数据进行断言测试即可。

### 缺点

#### 数据包装过度

为了保证通知数据的完备性，在实际的业务数据之外，我们需要为其包装好当前的状态信息，例如当前的加载是否成功、加载失败的错误信息、不属于列表数据的其他装饰性数据等，在发送、使用时均显得繁琐，不便于代码优化。

#### 业务逻辑的发起和接受脱离

当数据更新成为业务处理的唯一返回回调时，每个业务处理的发起和返回将不再是一个方法的调用与返回的关系，而是需要在回调的通知中进行，对于开发者来说会有反直觉的感觉，也不便于代码的调试。

#### 庞大的数据回调处理

因为把数据响应都收拢到里一个回调方法体里，其中的逻辑将会很庞大，并且将会有各种状态的判断来分别处理不同状态、不同业务所对应的 UI 显示，对于代码维护是一个难点。

## 总结

把我的项目的部分页面使用 MVVM 模式重构以后，结构确实得到了优化，我能在不同的模块之中专心处理不同层级的代码逻辑，为他们单独编写详尽的单元测试，代码质量得到了提升。但是为了保持前面提到的原则，需要额外耗费精力维护代码，则确实是属于有待提升的点。