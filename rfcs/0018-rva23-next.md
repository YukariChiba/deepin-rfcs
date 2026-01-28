# RFC: 建立 deepin RVA23 优化/next 仓库

- 提案发起时间：2026-01-26
- 提案拉取请求：https://github.com/deepin-community/rfcs/pull/0018

## 概要

本提案建议为 deepin 25 建立一个独立叠加仓库。该仓库将集成最新的核心工具链（GCC 15, LLVM 21, Binutils 2.46）与基础库（Glibc 2.42+, OpenSSL 3.5+），旨在为符合 RISC-V RVA23 规范的新一代硬件提供支持与性能优化。同时，该仓库将作为 deepin 25 向下一代版本（next）过渡的技术验证场，降低未来跨大版本迁移的风险。

## 动机

deepin 25 的生命周期将至少持续到 2027 年以后。目前 deepin 25 的基线环境已经冻结，其核心组件版本（GCC 12.3, Glibc 2.38, LLVM 17）虽然稳定，但已无法满足 RISC-V 新硬件的生态需求：

1.  缺乏对 RVA23 Profile 的原生支持：
    现有的 GCC 12 和 Binutils 2.41 不支持通过标准的 -march 参数开启 RVA23 规范支持，且 RVA23 内的指令集拓展支持缺失严重，这使得新一代符合 RVA23 标准的 RISC-V 硬件的诸多指令无法被支持或识别。

2.  缺少关键的性能优化：
    过去几年来，上游社区（Glibc, OpenSSL, LLVM）针对 RISC-V 提交了大量基于 Vector (v)、Bitmanip (b) 和其它 RVA23 内扩展的性能优化补丁。当前 deepin 25 仓库（基于 RVA20 基线构建）完全缺失这些优化，严重制约了硬件潜能。

3.  技术债务与迁移风险：
    若推迟至 deepin next 准备转正时才进行跨越式升级（如从 GCC 12 跳跃至 GCC 15/16），将面临巨大的 ABI 兼容性与构建失败风险。

4.  维护资源不足
    鉴于 RISC-V port 维护人力有限（目前仅 1 人主要维护），我们无法为旧版编译器长期 backport 大量新特性。建立一个基于上游新版本的实验性仓库，是兼顾低维护成本与高性能体验的最佳策略。

## 详细设计/制度正文

### 1. 仓库定位

该仓库（暂定名 `next` 或 `experimental`）将作为 deepin 25 的上层叠加仓库，同时启用 `amd64`,`arm64`,`riscv64`,`loong64` 架构的构建。

用户需手动添加 `sources.list.d/next.list` 并提高优先级以启用此仓库，通过同时使用主线和 `next` 仓库的方式整体保证向后兼容，由于升级了 glibc 等基础库，不保证向前兼容。

对于 RISC-V 设备的使用场景，它不直接替换稳定版用户的核心环境，而是供开发者、高级用户在搭载 RVA23 兼容芯片的 RISC-V 设备使用。

对于其它架构而言，此仓库作为 next 版本的实验性用途，工具链升级对所有架构均有技术验证的价值，且部分 `all` 包需要在 `amd64` 上构建。

该仓库的维护原则：尽量减少下游 Patch 数量，直接使用上游版本解决问题，以适应有限的维护人力。

### 2. 核心组件选型与论证

#### 选型概览表

| 软件包 | 25 当前基线 | 计划目标版本 | 变更性质 | 理由 |
| :--- | :--- | :--- | :--- | :--- |
| GCC | 12.3 | 15.x | 升级 | GCC 15 是首个原生完整支持 RVA23 Profile 的版本，支持通过 `-march` 开启相关优化。GCC 16 发布时间（2026.4）过晚，不符合当前开发节奏。15 可作为 deepin next 的预选编译器版本。 |
| Binutils | 2.41 | 2.46 | 升级 | 需配合 GCC 15，Binutils 2.45 引入了对 RVA23 Profile 的汇编与链接支持。 |
| Glibc | 2.38 | 2.42 + Patches | 升级 + 补丁 | 集成最新的 RISC-V 字符串与内存操作优化等。|
| LLVM | 17 | 21 | 升级 | 对标 Debian Forky 规划。RISC-V 后端在 LLVM 中迭代极快，需尽可能追新以获取改进。 |
| OpenSSL | 3.x | 3.6 + Patches | 升级 + 补丁 | 集成通用加密算法的 RISC-V Vector/Zbb 汇编级优化。 |

#### 详细技术论证

##### GCC 15 & Binutils 2.46：

根据上游的提交记录，开发者 `pz9115` 在 GCC/Binutils 中引入了 `RISC-V Profiles 23` 支持。

> GCC: RISC-V: Support RISC-V Profiles 23. (pz9115, 2025-05-10)
> GCC: RISC-V: Update Profiles string in RV23. (pz9115, 2025-06-16)
> Binutils: RISC-V: Add support for RISC-V Profiles 23. (pz9115, 2025-05-11)

没有这两个版本，我们将被迫使用非标准的 flag 组合来开启指令集(参考 ubuntu)，不仅难以维护，且难以确保生成的二进制文件一定符合 RVA23 规范。

GCC 16 预计于 2026 年 4 月发布，等待该版本将导致我们错过当前的硬件适配窗口，且后续也有观望和调整基线的空间，因此先确定使用 GCC 15 是最佳选择。

##### Glibc 2.42 + Patches：

Glibc 是系统性能的瓶颈所在。计划中的 2.42 版本将通过补丁向后移植 RVA23 相关的优化：
> riscv: Add RVV memset for both multiarch and non-multiarch builds (peter-bergner, 2025-12-20)
> riscv: Add Zbkb optimized repeat_bytes helper (peter-bergner, 2025-11-01)

##### OpenSSL 3.5 + Patches：

安全通信性能直接依赖于 OpenSSL。观察到 `crypto/perlasm/riscv.pm` 存在一些变更：

> SM4-CBC performance improvement on RISC-V (zl523856, 2025-12-31)
> SM3: Performance optimized with RISC-V Vector Crypto (cxx194832, 2025-12-31)
> SHA512 performance optimized by RISCV RVV (cxx194832, 2025-12-23)
> RISC-V: Add Zbb orn and its pseudo instruction opcode to rv64gc in riscv.pm (HeliC829, 2025-08-13)
> RISC-V: Add Zbb rori opcode in riscv.pm (HeliC829, 2025-03-28)
> riscv: Provide a vector only implementation of Chacha20 cipher (cyyself, 2024-05-08)

这些补丁均在 2025 年下半年至年底提交，现有 deepin 仓库完全缺失这些优化。

##### LLVM 21：

LLVM 在 RISC-V 的自动向量化方面进展神速，有大量提交合入。此时选择最新的稳定版本 LLVM 21，可以确保 Rust、Clang 构建的软件包能够利用上最新的优化。而 LLVM 22 还刚发布 RC2，尚未稳定（按照经验，发布之后依然需要一段时间供 rust 等下游软件修复 bug）

##### rust 1.91:

rust 方面，rva23 作为 `riscv64a23-unknown-linux-gnu` 新的 target 在 1.91 被引入，2025 年转为 tier-2 without host tools 支持等级，是否引入 rust 的大版本升级有待考证。

## 潜在风险

### 与自研应用兼容性问题

由于默认行为的改变，部分 deepin/dde 组件可能在最新的 GCC 下无法完成编译，但考虑到 RVA23 优化通常不涉及重构整个 dde 软件包组，可以在打包时处理。

### 降级风险

已经确认 deepin RVA23 分支不需要遵循 deepin 25 的版本锁定规则，但不排除未来存在不可抗力因素企图强行降级版本，此事需要被确认。

## 参考文档

- OpenSSL AES 性能测试（基于 b 拓展）: https://gist.github.com/ZenithalHourlyRate/7b5175734f87acb73d0bbc53391d7140
- LLVM 性能测试（基于 RVA22+V）：https://blogs.igalia.com/compilers/2025/05/05/boosting-risc-v-application-performance-an-8-month-llvm-journey/
- LLVM/GCC 关于 V 拓展的性能测试：https://llvm.org/devmtg/2025-03/slides/riscv_on_spec_cpu.pdf
