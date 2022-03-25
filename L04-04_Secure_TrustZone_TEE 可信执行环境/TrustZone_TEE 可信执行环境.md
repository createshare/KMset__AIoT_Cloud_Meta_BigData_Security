# TrustZone_TEE 可信执行环境



# ARM TrustZone

ARM TrustZone是ARM公司推出的SoC及CPU系统范围的安全解决方案，目前已在一些采用ARM指令集的应用处理器上广泛使用。

ARM TrustZone是基于硬件的安全功能，它通过对原有硬件架构进行修改，在处理器层次引入了两个不同权限的保护域——安全世界和普通世界，任何时刻处理器仅在其中的一个环境内运行。

同时这两个世界完全是硬件隔离的，并具有不同的权限，正常世界中运行的应用程序或操作系统访问安全世界的资源受到严格的限制，反过来安全世界中运行的程序可以正常访问正常世界中的资源。

这种两个世界之间的硬件隔离和不同权限等属性为保护应用程序的代码和数据提供了有效的机制：

- 通常正常世界用于运行商品操作系统（例如Android、iOS等），该操作系统提供了正常执行环境（Rich Execution Environment，REE）；
- 安全世界则始终使用安全的小内核（TEE-kernel）提供可信执行环境（Trusted Execution Environment，TEE），机密数据可以在TEE中被存储和访问。
- 这样一来即使正常世界中的操作系统被破坏或入侵（例如iOS已被越狱或Android已被ROOT），黑客依旧无法获取存储在TEE中的机密数据。

## Cortex-A 上采用的 TrustZone 架构

图中描述了Cortex-A上采用的TrustZone架构，该架构中还引入了一种称为监视模式的处理器模式，该模式负责在世界过渡时保留处理器状态，两个世界可以通过称为安全监视器调用（SMC）的特权指令进入监视模式并实现彼此切换。

![TrustZone 架构](figures/TrustZone 架构.jpg)

## Cortex-M 上采用的 TrustZone 架构

除了Cortex-A微架构外，ARM发布的新一代Cortex-M微架构同样为TrustZone提供了硬件支持。

- 与Cortex-A相同的是，Cortex-M依旧将处理器运行状态划分为安全世界和正常世界，并阻止运行于正常世界的软件直接访问安全资源。
- 不同的是，Cortex-M已针对更快的上下文切换和低功耗应用进行了优化。
- 具体来说，Cortex-M中世界之间的划分是基于内存映射的，并且转换是在异常处理代码中自动发生的（如图所示）。
- 这意味着，当从安全内存运行代码时，处理器状态为安全，而当从非安全内存运行代码时，处理器状态为非安全。
- Cortex-M中的TrustZone技术排除了监视模式，也不需要任何安全的监视软件，这大大减少了世界切换延迟，使得世界之间的转换为更高效。
- 为了在两个世界之间架起桥梁，Cortex-M引入了三个新指令：
  - secure gateway（SG）：SG指令用于在安全入口点的第一条指令中从非安全状态切换到安全状态
  - branch with exchange to non-secure state（BXNS）：安全软件使用BXNS指令来返回到非安全程序
  - branch with link and exchange to non-secure state（BLXNS）：安全软件使用BLXNS指令来调用非安全功能。
  - 最后，此外，Cortex-M 中的状态转换也可以由异常和中断触发。

















