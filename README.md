# SGFuzz: Stateful Greybox Fuzzer
SGFuzz（Stateful Greybox Fuzzer）是一个用于在 [LibFuzzer](https://llvm.org/docs/LibFuzzer.html) 之上构建的有状态软件系统的灰盒模糊器，它涉及额外的反馈，以便于探索有状态软件的状态空间，从而暴露有状态错误。

# Publication

请查看我们在 Usenix Security 2022 上发表的论文的更多技术细节： [https://www.usenix.org/conference/usenixsecurity22/presentation/ba](https://www.usenix.org/conference/usenixsecurity22/presentation/ba).

# How to run?
1. 我们提供了一个 docker 文件来使用 OpenSSL 执行 SGFuzz。请参阅文档 [example/openssl/Readme.md](https://github.com/bajinsheng/SGFuzz/tree/master/example/openssl)。

2. 您可以在 [FuzzBench](https://github.com/bajinsheng/SGFuzz_Fuzzbench) 中直接运行 SGFuzz。

# 详细用法
## 运行环境
Linux (Tested in Ubuntu 20.04LTS)

LLVM >= 6.0 (tested in LLVM 10)

Clang >= 6.0 (tested in Clang 10)

Python3.8

### 1. 构建 SGFuzz 驱动
输入存储库的根目录，然后运行以下命令来编译 SGFuzz 驱动程序。
```
./build.sh
```


### 2. 状态插桩
SGFuzz 需要用于状态反馈的静态插桩。它是独立于任何编译器的源代码级指令插入。

1) 状态插桩:
```
python3 sanitizer/State_machine_instrument.py target_folder [-b blocked_variable_file]
```
* 可选步骤:
我们还在 python 脚本中提供了一个调试选项，用于输出所有插入指令的变量名称和位置。
最好检查所有插入指令的变量，并通过 [-b blocked_variable_file] 筛选出一些不合适的变量，例如不是枚举变量或对其值过于敏感的变量
这不是必要的，但可以提供更好的结果。
原因是我们的方法使用正则表达式匹配来提取用于状态反馈的所有枚举变量。然而，有时，一些非枚举变量会被提取为枚举变量，因为它们与某些枚举变量具有完全相同的变量名。它可以通过一些编译工具来解决，如 clangAST，以准确识别枚举变量。工程方面的工作留待将来做。


### 3. 编译
在这里，我们按照 LibFuzzer 的做法，按照正常步骤编译目标程序。有关更多信息，请参阅 [LibFuzzer](https://llvm.org/docs/LibFuzzer.html) 官方文档。然后，在链接阶段，我们将 SGFuzz 库链接到目标程序"```libsfuzzer.a```"。
```
clang -o a.o a.c -fsanitize=fuzzer-no-link
clang++ -o program a.o b.o c.o ... libsfuzzer.a -ldl -lpthread -fsanitize=fuzzer-no-link
```

### 4. 运行
```
./program
```

# 许可证
此项目是根据 Apache License 2.0 获得许可的。有关详细信息，请参阅 [LICENSE](./LICENSE) 文件。
