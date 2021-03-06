# 1 R Bioconductor初步探索

[Bioconductor](http://www.bioconductor.org/)提供了用于分析和理解高通量基因组数据的工具。 Bioconductor使用R统计编程语言，并且是开源和开放开发的。每年有两个版本，以及一个活跃的用户社区。

# 2 RNA-Seq后续应该怎么分析？

## 2.1 GO富集分析（GO enrichment analysis）

### 2.1.1 基因本体论（Gene Ontology，GO）[^1]

  可以依据以下三个层面对基因进行注释分类。

  1. 细胞组分（Cellular Component，CC）
     
     细胞的每个部分和细胞外环境。
  
  2. 生物过程（Biological Process，BP）
  
      生物学过程系指由一个或多个分子功能有序组合而产生的系列事件。其定义有广义和狭义之分，在词义上可以区分为泛指和特指。一般规律是，一个过程是由多个不同的步骤组成。需要指出的是，生物学过程与途径或通路（pathway）不是同一回事。
  
  3. 分子功能（Molecular Function，MF）
  
      可以描述为分子水平的活性（activity），如催化（catalytic）或结合（binding）活性。
      
  为了实现基因的GO富集（GO enrichment），我们采用`GO富集分析`:
  
### 2.1.2 GO富集分析流程

1. 加载`cuffdiff`结果
        
    ```r
    cuffdiff_result = read.table("~/files/data/rna-seq/cuffdiff_test_data_gene_exp.diff", header = T, sep="\t")
    cuffdiff_result$sample_1 <- "treat"
    cuffdiff_result$sample_2 <- "ctrl"
    dim(cuffdiff_result)
    ```
    
2. 选择`DEG`
  
    满足以下三个条件的基因即可视为差异基因：
      
    - \\(FPKM1>1\\)并且\\(FPKM2>1\\)
      
    - \\(\left|\log_{2}{FC}\right|\geq 1\\)
      
    - \\(p < 0.05\\)

    ```r
    select_vector <-
      (cuffdiff_result$value_1>1|cuffdiff_result$value_2>1)&
      (abs(cuffdiff_result$log2.fold_change.)>=1) &
      (cuffdiff_result$p_value<0.05)
    cuffdiff_result.sign <- cuffdiff_result[select_vector,]
    dim(cuffdiff_result.sign)
    ```
    
    通过数据筛选后，基因数已经从原来的22857个缩减到84个，这84个就是我们期望的`DEG`，但是84个基因做GO富集分析是做不出什么好的结果的（**为什么呢？**），因此在这里将条件放宽一下，\\( \left|\log_{2}{FC}\right|\geq 1 \\)变为\\( \left|\log_{2}{FC}\right|\geq 0.5 \\)，得到差不多1000个左右的基因即可。
    
    ```r
    select_vector <-
      (cuffdiff_result$value_1>1|cuffdiff_result$value_2>1)&
      (abs(cuffdiff_result$log2.fold_change.)>=0.5) &
      (cuffdiff_result$p_value<0.05)
    cuffdiff_result.sign <- cuffdiff_result[select_vector,]
    dim(cuffdiff_result.sign)
    ```

3. 富集分析（网站法）

    首先将`gene_id`提取出来。

    ```r
    output.gene_id <- data.frame(gene_id=cuffdiff_result.sign$gene_id)
    write.table(output.gene_id, 
                file="~/files/data/rna-seq/sign_gene_id.txt", 
                col.names = F, row.names = F,
                sep="\t", quote = F)
    ```
    
    将提出出来的基因粘贴到[Enrichr](http://amp.pharm.mssm.edu/Enrichr/)的输入框里面，然后点击`Submit`。
    
    **注意**此网站适用于模式动物。如果要做植物的可以使用[DAVID](https://david.ncifcrf.gov/home.jsp)，这个网站的优点是动物植物都可以做，但这个网站在国内加载奇慢。

    {{% figure class="center" src="https://files.zzmath.top/enrichr.png" title="图1. Enrichr官网主页面." alt="enrichr" width="100%"%}}
    
    稍等片刻后即可得到GO，KEGG，DO等分析结果。
    
    {{% figure class="center" src="https://files.zzmath.top/enrichr_result.png" title="图2. Enrichr运行完成后输出页面." alt="enrichr result" width="100%"%}}
    
4. 富集分析（R包大法）
    
    安装以下的R包[^2]：
    
    ```r
    BiocManager::install("clusterProfiler")
    BiocManager::install("topGO")
    BiocManager::install("Rgraphviz")
    BiocManager::install("pathview")
    BiocManager::install("org.Hs.eg.db")
    ```
    
    安装完成后加载包：
    
    ```r
    library("clusterProfiler")
    library("topGO")
    library("Rgraphviz")
    library("pathview")
    library("org.Hs.eg.db")
    ```
    
    在进行GO富集分析之前，对`output.gene_id`进行字符串转化，转化完后进行GO富集分析，在这里值得注意的是还进行了一个`ID转换`：
    
    ```r
    DEG.gene_symbol <- as.character(output.gene_id$gene_id)
    DEG.entrez_id <- mapIds(x = org.Hs.eg.db, 
                            keys = DEG.gene_symbol,
                            keytype = "SYMBOL",
                            column = "ENTREZID")
    DEG.entrez_id <- na.omit(DEG.entrez_id)
    ```
    
    ```r
    erich.go.CC <- enrichGO(
      gene = DEG.entrez_id, OrgDb = org.Hs.eg.db, 
      keyType = "ENTREZID", ont = "CC", 
      pvalueCutoff = 0.5, qvalueCutoff = 0.5,
      readable = T
    )
    barplot(erich.go.CC)
    dotplot(erich.go.CC)
    ```
    
    {{% figure class="center" src="https://files.zzmath.top/go_barplot.png" title="图3. Barplot of Cellular Components." alt="GO barplot" width="100%"%}}
    
    {{% figure class="center" src="https://files.zzmath.top/go_dotplot.png" title="图4. Dotplot of Cellular Components." alt="GO dotplot" width="100%"%}}
    
    除此之外还可以使用`plotGOgraph`绘制树形图
 ：
 
    ```r
    plotGOgraph(erich.go.CC)
    ```
    <center style="font-size:16px;color:#C0C0C0;margin-block-start: 1em;margin-block-end: 1em;">图5. PlotGOgraph of Cellular Components.</center>
    
    如果图片看不清楚，可以将结果保存为`pdf`文件
    
    ```r
    pdf("~/files/data/rna-seq/enrich.go.cc.tree.pdf")
    plotGOgraph(erich.go.CC)
    dev.off()
    ```

## 2.2 KEGG富集分析（KEGG enrichment analysis）

将生物体的`pathway`进行富集分析。

```r
erich.KEGG <- eichKEGG(gene = DEG.entrez_id, organism = "hsa", 
                 keyType = "kegg", alueCutoff = 0.05, 
                 qvalueCutoff = 0.2)
```

```{r, echo=FALSE,out.width=c('50%', '50%'), fig.show='hold', fig.align="center", fig.cap="Barplot (left) and dotplot (right) of KEGG."}
load("~/files/data/rna-seq/go_kegg_do.Rdata")
barplot(erich.KEGG)
dotplot(erich.KEGG)
```

{{% figure class="center" src="https://files.zzmath.top/kegg_barplot.png" title="图6. Barplot of KEGG." alt="KEGG barplot" width="100%"%}}

{{% figure class="center" src="https://files.zzmath.top/kegg_dotplot.png" title="图7. Dotplot of KEGG." alt="KEGG dotplot" width="100%"%}}


## 2.3 DO富集分析（DO enrichment analysis）

看基因是否在某个疾病上富集，临床上用得比较多。

```r
library("DOSE")
erich.DO <- enrichDO(gene = DEG.entrez_id, ont = "DO",
                     pvalueCutoff = 0.5, qvalueCutoff = 0.5)
barplot(erich.DO)
dotplot(erich.DO)
```

{{% figure class="center" src="https://files.zzmath.top/do_barplot.png" title="图8. Barplot of DO." alt="DO barplot" width="100%"%}}

{{% figure class="center" src="https://files.zzmath.top/do_dotplot.png" title="图9. Dotplot of DO." alt="DO dotplot" width="100%"%}}

以上所有内容学习自[2018-01-07 生物信息学教程-GO, KEGG, DO富集分析
](https://www.bilibili.com/video/BV14W411q7gi?t=2387)

[^1]: [基因本体](https://baike.baidu.com/item/%E5%9F%BA%E5%9B%A0%E6%9C%AC%E4%BD%93)
[^2]: [org.Hs.eg.db](https://bioconductor.org/packages/release/data/annotation/html/org.Hs.eg.db.html)

