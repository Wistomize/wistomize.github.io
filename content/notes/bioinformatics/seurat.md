---
title: "Seurat处理"
date: 2023-07-01
description: "使用Seurat包处理空间转录组数据"
tags: ["生物信息学", "Seurat", "R语言", "空间转录组"]
series: ["生物信息学研究实录"]
---

## 环境配置

新版的Seurat有处理空间转录组数据的功能，所以就顺手用一下子。

IDE就用Rstudio，相关包安装就用Rstudio自带的就行了。主要是seurat本体 + ggplot + patchwork这几个，另外还需要dplyr和stringr。

```r
library(Seurat)
library(ggplot2)
library(gridExtra)
library(patchwork)
library(dplyr)
library(stringr)
```

空间转录组分析
