---
title: "ggplot2 函数式批量作图"
date: 2023-08-01
description: "使用ggplot2对Seurat对象进行函数式批量作图的技巧"
tags: ["生物信息学", "ggplot2", "R语言", "数据可视化"]
series: ["生物信息学研究实录"]
---

## 以处理Seurat对象为例

Seurat内置的函数作图出的图对象都是ggplot类型的对象，所以全都可以用ggplot2的方法进行图的格式修改。

对于一个常见的seurat作图函数，一般函数内部会有一些参数可供修改：

```r
objheatmap <- DoHeatmap(obj,
	features = rownames(objmarker),
	group.by = 'orig.ident',
	size = 4,
	angle = 0,
	hjust = 0.5,
	vjust = 0.1,
	lines.width = 5)
# 这些参数都是seurat的DoHeatmap函数里自带的
# 但是自带函数里所带参数比较少，有一些细节需要用ggplot2的参数进行修改
```

然而，有些细节参数是改不了的，所以这个时候要借用ggplot2中的参数theme来调整：

```r
dp <- dp + labs(title = paste(Namelist[num], 'markers'))
# 注释1
# 注释2
dp <- dp + theme( # theme是最为重磅的参数，可以调节很多东西
	plot.title = element_text(size = 12, hjust = 0.5),
	axis.title.y = element_blank(),
	axis.title = element_blank(),
	axis.text.x = element_text(angle = 45, vjust = 1.1, hjust = 1.1, size = 10)
)
# 注释3
```

注释1：很神奇的是，ggplot中修改某个绘图参数并不是改变对象中的某个内容（其实也可以，但一般不这么做）或者改变某个函数的参数，而是通过这种奇怪的加法

注释2：labs() 是修改各种标签的函数（即labels的缩写），除了标题，还可以修改坐标轴的scale等等

- [labs() 函数官方文档](https://ggplot2.tidyverse.org/reference/labs.html)
- [ggplot2 book相关页面](https://ggplot2-book.org/annotations#sec-titles)

注释3：theme中提供了很多可以修改的参数，可以直接看[theme() 官方文档](https://ggplot2.tidyverse.org/reference/theme.html)

## 多图批量处理

如果要做多个图，一个一个作图很麻烦不说，如果要做一些调整，每一个图的参数都要重新在代码里修改，这样就会十分复杂，就像这样：

```r
plotWTFS <- FeatureScatter(WTdata, feature1 = "nCount_Spatial", feature2 = "nFeature_Spatial", cols = DEEPGRAY) + theme(axis.text.x = element_text(size = rel(0.7)), plot.title = element_blank()) + NoLegend()
plotA2FS <- FeatureScatter(A2data, feature1 = "nCount_Spatial", feature2 = "nFeature_Spatial", cols = LIGHTRED) + theme(axis.text.x = element_text(size = rel(0.7)), plot.title = element_blank()) + NoLegend()
plotA6FS <- FeatureScatter(A6data, feature1 = "nCount_Spatial", feature2 = "nFeature_Spatial", cols = DARKCYAN) + theme(axis.text.x = element_text(size = rel(0.7)), plot.title = element_blank()) + NoLegend()
```

写三遍不说，每次需要修改某个参数的时候，都要改三遍，虽然说可以先改一个，改好了再应用到那三个上面，但架不住有时候会忘记改，最后出来乱七八糟什么都有。更架不住有人让你天天改……

所以，为了应对这种情况，就要用一些批量处理的办法，毕竟写程序的最终目的就是要省去重复繁杂的工作。

开始介绍之前必须吐槽一下：R语言的函数系统并不非常令人舒适，R这个语言出来的时候本来是以所谓的"善于处理向量"著称。在代码层面上做向量和矩阵运算也确实很方便，但是到了应用层级上，在我们这些赖掉包侠手上，这个东西就显得非常的鸡肋、反直觉。

我个人最常用的批量处理方法就是**自定义函数**和**lapply函数**一起使用。假设我们需要绘制点状图（DotPlot），并且有5个处理方法完全相同的对象要做，那么我们就可以这样定义一个函数：

```r
dotplotProcess <- function(obj){
	dp <- dp + labs(title = paste(Namelist[num], 'markers'))
	dp <- dp + theme(
		plot.title=element_text(size = 12, hjust = 0.5),
		axis.title.y = element_blank(),
		axis.title = element_blank(),
		axis.text.x = element_text(angle = 45, vjust = 1.1, hjust = 1.1, size = 10)
	)
	return(dp)
}
```
