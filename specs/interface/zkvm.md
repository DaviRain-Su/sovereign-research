# zkVM

主权 SDK 旨在支持任何能够运行 Rust 代码的 zkVM。但是，VM 必须能够支持一组标准的 API。

## 关于性能的说明

该规范没有定义任何与性能相关的标准：证明大小、证明者工作、验证时间、延迟。这一省略不应被理解为暗示 SDK 对于所有选择的证明系统都同样有效。但是，当在任何隔音系统中定义时，SDK 都会正确运行，我们没有定义任何特定要求。我们强烈建议用户考虑使用 Risc0 等高性能虚拟机。


## 方法

- log
    - usage: log 方法将一个项目添加到证明的公共输出中。这些输出被提交到证明中，因此对输出的任何篡改都会导致证明验证失败。
    - input: 要附加到输出的项目。可以是任何支持序列化的结构
