+++
title = "Tech Talk: Wall is Coming"
date = "2024-02-22"
tags = ["sys", "talk"]
description = "Tech Talk文稿，梳理内存墙问题的历史渊源，尝试给出对优化空间的理解，推导出相应的启发性策略，并列举一些访存优化技术。"
showFullContent = false
+++

## 内存墙和登纳德定律失效
“内存墙（the Memory Wall）”和“登纳德定律失效（the break down of Dennard Scaling）”是计算生态演化中的两个核心矛盾。
- 为了对抗Dennard Scaling的失效，计算硬件的架构从单核向多核、众核突围。
- 为了掩盖内存墙问题，内存层级（memory hierarchy）变得越来越深，off-chip互连带宽不得不迅速提高。

起初单核到多核是巨变，逼迫软件架构进行痛苦重构，并发编程问题木秀于林，因此被近20年的工业界实践+学术界研究集火秒了——如今我们有多线程编程范式、异步回调范式、goroutine式的有栈协程、C++/Rust的async/await无栈协程、lockfree/waitfree数据结构等诸多工具。
相较而言，这几十年来内存墙问题由于其隐蔽性、不紧急性、棘手性，不仅没得到妥善解决，反而根深蒂固，愈发遮掩不住，暴露在软件工程师面前，因此这次分享的重点是内存墙。

不过在进入主题之前，还需先介绍一下数据中心硬件和微处理器架构演化的些许背景。

<div id="div">
</div>
<style type="text/css">
	#div {
		text-align: center
	}
</style>
<script>
   const svg = d3.select("#div")
                 .append("svg")
                 .attr("width", "550")
                 .attr("height", "100")
                 .style("background-color", "lightblue")
                 .attr("id", "demo1")

   let rect = d3.select("#demo1")
	            .append("rect")
	            .attr("x", "200")
	            .attr("y", "20")
	            .attr("width", "100")
	            .attr("height", "70")
	            .attr("fill", "orange")
                .attr("stroke", "blue")
                .attr("stroke-width", "3px")
</script>
