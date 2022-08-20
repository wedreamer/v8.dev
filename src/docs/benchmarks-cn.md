***

## 标题： “在本地运行基准测试”&#xA;描述：“本文档介绍如何在 d8 中运行经典基准测试套件。

我们有一个简单的工作流程来运行SunSpider，Kraken和Octane的“经典”基准测试。可以使用不同的二进制文件和标志组合运行，并且结果在多次运行中取平均值。

## 中央处理器

构建`d8`shell 按照说明进行操作[使用 GN 进行构建](/docs/build-gn).

在运行基准测试之前，请确保将 CPU 频率缩放调控器设置为性能。

```bash
sudo tools/cpu.sh fast
```

命令`cpu.sh`理解是

*   `fast`、性能（别名`fast`)
*   `slow`、电源保存（别名`slow`)
*   `default`、ondemand（别名`default`)
*   `dualcore`（禁用除两个内核之外的所有内核）、双（别名 为`dualcore`)
*   `allcores`（重新启用所有可用内核），全部（别名`allcores`).

## 套件

`CSuite`是我们简单的基准运行器：

```bash
test/benchmarks/csuite/csuite.py
    (sunspider | kraken | octane)
    (baseline | compare)
    <path to d8 binary>
    [-x "<optional extra d8 command-line flags>"]
```

首次运行`baseline`模式以创建基线，然后在`compare`模式以获取结果。`CSuite`默认为对 Octane 进行 10 次运行，对 SunSpider 执行 100 次运行，对 Kraken 执行 80 次运行，但您可以使用`-r`选择。

`CSuite`在从中运行的目录中创建两个子目录：

1.  `./_benchmark_runner_data`— 这是来自 N 次运行的缓存输出。
2.  `./_results`— 它将结果写入文件主站。您可以保存这些
    具有不同名称的文件，它们将显示在比较模式下。

在比较模式下，您将自然地使用不同的二进制文件或至少不同的标志。

## 用法示例

假设您已经构建了两个版本`d8`，并想看看SunSpider会发生什么。首先，创建基线：

```bash
$ test/benchmarks/csuite/csuite.py sunspider baseline out.gn/master/d8
Wrote ./_results/master.
Run sunspider again with compare mode to see results.
```

按照建议，再次运行，但这次在`compare`具有不同二进制文件的模式：

    $ test/benchmarks/csuite/csuite.py sunspider compare out.gn/x64.release/d8

                                   benchmark:    score |   master |      % |
    ===================================================+==========+========+
                           3d-cube-sunspider:     13.9 S     13.4 S   -3.6 |
                          3d-morph-sunspider:      8.6 S      8.4 S   -2.3 |
                       3d-raytrace-sunspider:     15.1 S     14.9 S   -1.3 |
               access-binary-trees-sunspider:      3.7 S      3.9 S    5.4 |
                   access-fannkuch-sunspider:     11.9 S     11.8 S   -0.8 |
                      access-nbody-sunspider:      4.6 S      4.8 S    4.3 |
                     access-nsieve-sunspider:      8.4 S      8.1 S   -3.6 |
          bitops-3bit-bits-in-byte-sunspider:      2.0 |      2.0 |        |
               bitops-bits-in-byte-sunspider:      3.7 S      3.9 S    5.4 |
                bitops-bitwise-and-sunspider:      2.7 S      2.9 S    7.4 |
                bitops-nsieve-bits-sunspider:      5.3 S      5.6 S    5.7 |
             controlflow-recursive-sunspider:      3.8 S      3.6 S   -5.3 |
                        crypto-aes-sunspider:     10.9 S      9.8 S  -10.1 |
                        crypto-md5-sunspider:      7.0 |      7.4 S    5.7 |
                       crypto-sha1-sunspider:      9.2 S      9.0 S   -2.2 |
                 date-format-tofte-sunspider:      9.8 S      9.9 S    1.0 |
                 date-format-xparb-sunspider:     10.3 S     10.3 S        |
                       math-cordic-sunspider:      6.1 S      6.2 S    1.6 |
                 math-partial-sums-sunspider:     20.2 S     20.1 S   -0.5 |
                math-spectral-norm-sunspider:      3.2 S      3.0 S   -6.2 |
                        regexp-dna-sunspider:      7.6 S      7.8 S    2.6 |
                     string-base64-sunspider:     14.2 S     14.0 |   -1.4 |
                      string-fasta-sunspider:     12.8 S     12.6 S   -1.6 |
                   string-tagcloud-sunspider:     18.2 S     18.2 S        |
                string-unpack-code-sunspider:     20.0 |     20.1 S    0.5 |
             string-validate-input-sunspider:      9.4 S      9.4 S        |
                                   SunSpider:    242.6 S    241.1 S   -0.6 |
    ---------------------------------------------------+----------+--------+

上一次运行的输出缓存在当前目录 （`_benchmark_runner_data`).聚合结果也会缓存在目录中`_results`.运行比较步骤后，可以删除这些目录。

另一种情况是，当您具有相同的二进制文件，但希望看到不同标志的结果时。您感觉相当沉闷，您想看看Octane在没有优化编译器的情况下如何表现。首先是基线：

```bash
$ test/benchmarks/csuite/csuite.py -r 1 octane baseline out.gn/x64.release/d8

Normally, octane requires 10 runs to get stable results.
Wrote /usr/local/google/home/mvstanton/src/v8/_results/master.
Run octane again with compare mode to see results.
```

请注意，一次运行通常不足以确保许多性能优化的警告，但是，我们的“更改”应该仅在一次运行中具有可重现的效果！现在让我们比较一下，通过`--noopt`要关闭的标志[涡轮风扇](/docs/turbofan):

```bash
$ test/benchmarks/csuite/csuite.py -r 1 octane compare out.gn/x64.release/d8 \
  -x "--noopt"

Normally, octane requires 10 runs to get stable results.
                               benchmark:    score |   master |      % |
===================================================+==========+========+
                                Richards:    973.0 |  26770.0 |  -96.4 |
                               DeltaBlue:   1070.0 |  57245.0 |  -98.1 |
                                  Crypto:    923.0 |  32550.0 |  -97.2 |
                                RayTrace:   2896.0 |  75035.0 |  -96.1 |
                             EarleyBoyer:   4363.0 |  42779.0 |  -89.8 |
                                  RegExp:   2881.0 |   6611.0 |  -56.4 |
                                   Splay:   4241.0 |  19489.0 |  -78.2 |
                            SplayLatency:  14094.0 |  57192.0 |  -75.4 |
                            NavierStokes:   1308.0 |  39208.0 |  -96.7 |
                                   PdfJS:   6385.0 |  26645.0 |  -76.0 |
                                Mandreel:    709.0 |  33166.0 |  -97.9 |
                         MandreelLatency:   5407.0 |  97749.0 |  -94.5 |
                                 Gameboy:   5440.0 |  54336.0 |  -90.0 |
                                CodeLoad:  25631.0 |  25282.0 |    1.4 |
                                   Box2D:   3288.0 |  67572.0 |  -95.1 |
                                    zlib:  59154.0 |  58775.0 |    0.6 |
                              Typescript:  12700.0 |  23310.0 |  -45.5 |
                                  Octane:   4070.0 |  37234.0 |  -89.1 |
---------------------------------------------------+----------+--------+
```

整洁看到`CodeLoad`和`zlib`相对没有受到伤害。

## 引擎盖下

`CSuite`基于同一目录中的两个脚本，`benchmark.py`和`compare-baseline.py`.这些脚本中还有更多选项。例如，您可以记录多个基线并执行 3、4 或 5 路比较。`CSuite`针对快速使用进行了优化，并牺牲了一些灵活性。
