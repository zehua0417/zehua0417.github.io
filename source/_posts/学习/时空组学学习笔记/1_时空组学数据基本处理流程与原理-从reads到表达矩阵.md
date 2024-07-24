---
create_time: 2024-07-23 12:48
modification_date: 2024-07-23T12:48:08
subject: bio
direction: setereo-seq
tags:
  - Course_note
  - BGI
  - bio
  - bioinformatics
  - stereo
  - single-cell
title: 时空专题1_时空组学数据基本处理流程与原理_从reads到表达矩阵
---
# ❗**时空组学数据基本处理流程与原理-从reads到表达矩阵**

## Stereo-seq技术基础原理
### 文件
- **mask文件**: 记录CID序列与空间位置的文件
- **image文件**: 图像质控结果文件
- **Fastq文件**: 转录粗测序数据文件
### 芯片
- 芯片大小: 1cm\*1cm
- 纳米球尺寸: 直径220nm
- 纳米球间距: 500nm
- spot数量: $4\times10^8$个
- barcode长度: 25bp, $4^{25}种$
- UMI长度: 10bp
- CID: 25bp
## 数据分析SAW(STOmics analysis workflow)整体流程介绍
[SAW github link](https://github.com/BGIResearch/SAW)
### Stereo-seq
SAW 处理 Stereo-seq 的测序数据以生成空间基因表达矩阵，用户可以将这些文件作为起点进行下游分析。SAW 包含十三个基本和建议的流程和辅助工具，以支持其他便捷功能。
#### 输入文件
##### reference
- **输入文件**
	- 基因组序列文件: genome.fa
	- 基因注释文件: genes.gtf$/$gff
```txt
.
└── specieName
    ├── STAR_SJ100
    │   ├── Genome
    │   ├── SA
    │   ├── SAindex
    │   ├── chrLength.txt
    │   ├── chrName.txt
    │   ├── chrNameLength.txt
    │   ├── chrStart.txt
    │   ├── exonGeTrInfo.tab
    │   ├── exonInfo.tab
    │   ├── geneInfo.tab
    │   ├── genomeParameters.txt
    │   ├── sjdbInfo.txt
    │   ├── sjdbList.fromGTF.out.tab
    │   ├── sjdbList.out.tab
    │   └── transcriptInfo.tab
    ├── genes
    │   └── genes.gtf
    └── genome
        └── genome.fa

4 directories, 17 files
```
- **输出文件**
	- 参考基因组索引目录: /path/to/genomeDir
- **检查索引文件的小工具**
	- SAW: checkGTF
- 参考脚本: [example script](https://github.com/STOmics/SAW/tree/c6a058239d944a427278ee262008d1828a96b13f/Scripts/pre_buildIndexedRef)
##### mask
- **格式**: *.h5
##### image
- **格式**: 
  - SN*.ipr
  - SN*.tar.gz
##### fastq
**PE format**
- read1 = CID + MID
- read2 = mRNA
- 不需要splitMask
**SE format**
- read_name = CID + MID
- read = mRNA
- 需要splitMask
#### 流程
##### mapping 比对
- **Step1 BarcodeMapping**: 原始测序读取（存储在FASTQ文件中）的CID与Stereo-seq Chip T MASK文件中保存的CID-坐标键值对记录进行匹配（允许1个错配）。根据MASK文件中的记录，为可以配对的读取添加坐标信息。获得坐标注释的读取被称为有效CID mRNA读取（有效CID读取）。
- **Step2 Filtering**: 
	- 丢弃不合格的MID读取（不满足进一步分析要求）
	- 过滤含有接头adapter的读取
	- 过滤长度(去除poly-A后)小于30的短读取
- **Step3 GenomeMapping**: mRNA比对到参考物种基因组
**输出文件**:
- **BAM格式比对结果文件**: 
	- *.Aligned.sortedByCoord.out.bam
	- tag列包含每条reads的空间位置信息
- 比对结果数据统计文件
	- *_barcodeMap.stat: BarcodeMapping信息
	- *.Log.final.out: GenomeMapping信息
	- *.barcodeReadsCount.txt: 每个barcode的reads数
##### merge 合并
##### count 注释

| 注释类型          | 注释逻辑                        |
| ----------------- | ------------------------------- |
| Intron类型        | reads与intron有超过50%的overlap |
| antisense统计     | 比对上任意反链的gene            |
| transcriptome统计 | 注释上exon或intron且同链        |

**输出文件**
- **BAM格式比对结果文件与GEF格式表达矩阵**
	- *.Aligned.sortedByCoord.out.merge.q10.deduo.target.bam$, Tag列新增了注释成功的基因信息
	- $.raw.gef, 基因表达矩阵
- **日志文件**
	- logs/Bam2Gem_data_time.log, 记录了被跳过的基因
##### register/rapidRegister (图像)配准
基于空间位置信息, 将影像图最优映射到基因表达矩阵上
配准逻辑:
- 基于TrackLine配准
	- 选模板(scale & rotation)
	- 根据模板推全局
- 算法原理
	- 重心法
- 修正/微调
	- 根据TrackLine进行微调
##### tissueCut 组织分隔
在影像图上获取组织边界
- 有ssDNA图像: 基于影像图分隔
- 无ssDNA图像: 基于表达矩阵分隔
基于组织分隔掩模图, 结合矩阵进行组织区域表达矩阵提取, 输出SN.tissue.gef
##### spatialCluster 空间聚类
筛选高变菊英后进行PCA和UMAP降维, 再使用Leiden等算法进行聚类分析
##### saturation 饱和度计算
根据饱和度曲线判断数据是否需要加测
$$
sequence~saturation=1-\frac{Unique~Reads}{Total Reads}
$$
##### report 报告输出

{% gallery %}
![img](/img/obsimg/SAW_v7.1_workflow.png)
{% endgallery %}
### 蛋白质组学和转录组学
SAW 蛋白质组学和转录组学工作流程处理来自单芯片的测序数据以生成空间基因和蛋白质表达矩阵。您可以使用这两个组学信息开始更高维度的研究。此工作流程包括 23 个基本和建议流程。
{% gallery %}
![img](/img/obsimg/SAW_pt_v7.1_workflow.png)
{% endgallery %}
