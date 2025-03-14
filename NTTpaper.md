# Efficient number theoretic transform implementation on GPU for homomorphic encryption

## 摘要

基于格的密码学是同态加密的数学基础，它允许直接在加密数据上执行计算。同态加密使得隐私保护应用，如安全云计算成为可能；然而，同态运算的高计算复杂度限制了其实际应用。同态加密方案的快速实现高度依赖于高效的多项式算术，特别是在多项式环上非常大型多项式的乘法。数论变换（NTT）显著加速了多项式乘法，因此它是大多数同态加密方案实现中的核心算术运算。因此，实际的同态应用需要在不同计算平台上高效且快速地实现 NTT。在本工作中，我们提出了适用于 GPU 平台的 NTT、逆 NTT（INTT）以及基于 NTT 的多项式乘法运算的高效快速实现。 为了证明我们的 GPU 实现可以作为实际的加速器，我们在 GPU 上对 Brakerski/Fan-Vercauteren（BFV）同态加密方案的关键生成、加密和解密操作进行了实验，该方案在 Microsoft 的 SEAL 同态加密库中实现，这些操作高度依赖于基于 NTT 的多项式乘法。我们的 GPU 实现将这三个 BFV 操作的性能分别提高了 141.95 倍、105.17 倍和 90.13 倍，与在 Intel i9-7900X CPU 上运行的优化过的 SEAL 库相比，在 Tesla v100 GPU 上。

## 背景

基于格的密码学被推测在面对量子计算机的攻击时具有安全性，因此支持后量子密码学（PQC），它为完全同态加密（FHE）方案提供了数学基础。完全同态加密(FHE)允许在加密数据上进行计算，无需解密或密钥，因此能够安全处理敏感数据。尽管FHE的潜在应用具有划时代的性质，但其高算法复杂度是高效和实用实现的一个障碍。在各种FHE方案的不同核心算术运算中，多项式环上的乘法可能是最耗时的。

## 贡献

1. 提出了**高性能和高效的GPU实现**，用于NTT、INTT和基于NTT的多项式乘法操作。
2. 为了在GPU上并行化NTT和INTT操作，对算法结构进行了重大修改，并尽可能多地消除了它们之间的依赖。提出了一种**混合方法**，即当NTT块较大时，为每个NTT迭代进行单独的内核调用；但是，一旦NTT块足够小，就切换到不同的工作模式，在该模式下，剩余的NTT迭代在一个内核调用中完成。
3. BFV(Brakerski/Fan-Vercauteren)全同态加密方案的密钥生成、加密和解密操作在GPU上完全实现，BFV方案中的密钥生成和加密操作需要来自均匀、三值和离散高斯分布的随机多项式，还引入了这些分布的随机多项式采样器在GPU上的实现。p.s. BFV方案受微软的SEAL同态加密库支持

## 具体方案

1. 化简取模运算：巴雷特缩减算法将缩减操作中的除法替换为乘法、移位和减法操作，这些操作都比除法操作快得多。

![image-20250216145605005](NTTpaper.assets/image-20250216145605005.png)

2. 设计了一个128位无符号整数库，重载了所有算术、逻辑和二进制运算符，重点关注速度，受CUDA和C/C++支持。
3. NTT 操作通过 3 层嵌套循环执行。如果展开循环，可以看出n点NTT操作实际上由log(n)次迭代组成。GPU 中实现 NTT/INTT 算法的挑战在于**高效分配线程**以实现高利用率。为了获得最佳性能，所有线程都应该处于忙碌状态，并且**每个线程的工作量应该相等**。

![image-20250216145630751](NTTpaper.assets/image-20250216145630751.png)

![image-20250216145643260](NTTpaper.assets/image-20250216145643260.png)

4. i）在单内核方法中，为整个数组的NTT/INTT计算调用内核一次（导致单次内核调用中总共有 log(n)次迭代）。
   ii）在多内核方法中，为NTT/INTT操作的每次迭代调用一个内核（导致总共log(n)次内核调用）。
   对于较小的数组大小，单内核方法提供更好的结果。对于较大的数组大小（从n=2^14^到n=2^16^)，多内核方法是有利的。

   最左侧的列中，进行内核调用，其中16个块并行地对数组a的单独2048个元素进行操作。对于NTT计算的第二次和第三次迭代，以相同的方式进行两次更多的内核调用。到目前为止，采用多内核方法。
   然而，当步骤组中的元素数量为2^12^时，切换到单内核方法。在这里，总共八个GPU块在一个内核调用中处理数组其余部分的迭代。请注意，当步骤组大小为2^12^时，数组的上下 2048 个元素是顺序处理的。通过实验发现，当步骤组大小为2^12^时，内核调用开销在执行时间方面更高。换句话说，2^12^的步组大小是从多核方法切换到单核方法的最佳点。

![image-20250216145734437](NTTpaper.assets/image-20250216145734437.png)