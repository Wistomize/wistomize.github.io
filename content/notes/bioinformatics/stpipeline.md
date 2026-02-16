---
title: "数据预处理和官方Pipeline"
date: 2023-06-01
description: "10x Fresh Frozen空间转录组数据预处理与Space Ranger使用指南"
tags: ["生物信息学", "空间转录组", "Space Ranger", "10x Genomics"]
series: ["生物信息学研究实录"]
---

我分析的是空间转录组数据，所以下面以10x Fresh Frozen空间转录组数据为例。

## 集群搭建与环境配置

实验室暂时没有服务器，租了阿里云服务器。实例规格ecs.r6.4xlarge，16核CPU，128G内存，这个规格后续被证明是刚好够用且好用的。

公司给出的数据就是从阿里云的OSS服务下载，从他们自己的OSS搞到ECS上自然非常方便。具体过程略去。

开搞之前，要先在服务器上安装环境，我只装了一个Ubuntu 22.04 64位系统，别的啥也没有，干脆从头开始。于是需要从网站上开始找需要配置的环境。

官方文档提供了很多信息: [Space Ranger User Guide](https://www.10xgenomics.com/support/software/space-ranger)，可以参考。

使用curl安装Space Ranger：

```bash
curl -o spaceranger-2.1.0.tar.gz "https://cf.10xgenomics.com/releases/spatial-exp/spaceranger-2.1.0.tar.gz?..."
```

同时需要下载参考数据库，也就是基因组注释之类的，我这里下载了老鼠的数据库

```bash
curl -O "https://cf.10xgenomics.com/supp/spatial-exp/refdata-gex-mm10-2020-A.tar.gz"
```

新建一个文件夹来装这些东西，把下载的压缩包挪进去之后解压,并将spaceranger添加到环境变量（路径位置根据自己的来）：

```bash
tar -xzvf spaceranger-2.0.1.tar.gz
tar -xzvf refdata-gex-mm10-2020-A.tar.gz
export PATH=~/ST/spaceranger-2.0.1:$PATH
```

测试运行：

```bash
spaceranger testrun --id=tiny
```

接下来就会出现一大长串测试结果，如果有问题就会报错，根据问题解决就行了。

安装完成，接下来要选择合适的pipeline进行基础分析。关于选择pipeline，官网有一些说明：[Choosing-a-pipeline](https://www.10xgenomics.com/support/software/space-ranger/analysis/running-pipelines/choosing-a-pipeline)。简单地说，就是如果是单独的片子，可以用`spaceranger count`，多个相同处理，可以用`spaceranger aggr`。所以我们这次先用count这个。

## 数据输入格式

spaceranger count的输入主要是FASTQ，如果公司提供的不是fastq而是BCL什么的，那你还需要使用spaceranger mkfastq这种东西再跑一遍。

## 运行spaceranger count

好，重头戏要来了（狂喜）。

运行参数具体介绍可以参考：[Space Ranger Command Line Arguments](https://www.10xgenomics.com/support/software/space-ranger/analysis/running-pipelines/command-line-arguments)。我这里只写一些必要的东西。如以下示例：

```bash
spaceranger count --id=APP-2m \
--fastqs=~/ST/fastqpath \
--transcriptome=~/ST/refdata-gex-mm10-2020-A \
--image=~/ST/APP-2m.tif \
--slide=V19J01-123 \
--area=B1
--create-bam=false
```

接下来分别介绍这些参数的功能：

- `--id`: 指这次运行的代号，你自己指定一个就行，最好不要与过去和未来的重复。
- `--fastqs`: 指输入的FASTQ文件所在的**目录**，注意不是文件名。
- `--transcriptome`: 指参考转录组的位置，就是刚刚下载的那个老鼠的基因组参考数据库，这个是到文件夹的路径，注意最好是绝对路径，之前踩过坑跑不出来。
- `--image`: 这个参数写图片的位置。因为是空转，测序之前肯定会做一个HE染色的图或者免疫染色的图后续映射用，是一张很大的脑片图，一般是一个tiff文件。注意也是到文件本身
- `--slide`: 装片编号。所有可以用于空转测序的载玻片，都必须使用10x官方售卖的耗材，更鸡贼的是他们每个装片都是由单独编号的，这里就写你那个装片耗材上的编号。公司会告诉你编号，如果没告诉你那就肯定是在文件名字上体现的，这个装片编号格式一般为`V**?**-***`，其中?为字母，*为数字。
- `--area`: 装片区域。跟上一条是相对应的，每个官方指定耗材上都有四个区域可以放脑片，所以空转送样一般会让你送四的倍数个。四个区域分别是A1，B1，C1，D1。
- `--create-bam`: 是否创建bam文件，这个参数是必须的，挺奇怪吧。

其它参数的介绍可以参看上面的网址。输入这些参数后，正常就可以开始跑了，跑的其实还挺快的，跑一个数据按照我的配置30-40分钟就能跑完。跑完得到的结果就可以直接下载下来或者留在服务器上继续分析了。
