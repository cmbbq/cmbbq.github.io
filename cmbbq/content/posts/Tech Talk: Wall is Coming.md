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

<div id="damn">
    <svg width="960" height="500"></svg>
</div>
<style type="text/css">
svg {
    box-shadow: 0 0 10px #999;
    border-radius: 5px;
}
</style>
<script type="module">
import {
  drag,
  color,
  select,
  range,
  randomUniform,
  randomNormal,
  scaleOrdinal,
  selectAll,
  schemePastel1,
} from "https://cdn.skypack.dev/d3@7.8.5";
import {
    gridPlanes3D,
    points3D,
    lineStrips3D,
} from "https://cdn.skypack.dev/d3-3d@1.0.0";
document.addEventListener("DOMContentLoaded", () => {
    const origin = { x: 480, y: 250 };
    const j = 10;
    const scale = 20;
    const key = (d) => d.id;
    const startAngle = Math.PI/2;
    // const startAngle = 0;
    const colorScale = scaleOrdinal(schemePastel1);
    let scatter = [];
    let yLine = [];
    let xLine = [];
    let zLine = [];
    let xGrid = [];
    let beta = 0;
    let alpha = 0;
    let mx, my, mouseX = 0, mouseY = 0;
    const svg = select("svg")
        .call(
          drag()
            .on("drag", dragged)
            .on("start", dragStart)
            .on("end", dragEnd)
        )
        .append("g");
    const grid3d = gridPlanes3D()
        .rows(20)
        .origin(origin)
        .rotateY(startAngle)
        .rotateX(-startAngle)
        .scale(scale);
  const points3d = points3D()
    .origin(origin)
    .rotateY(startAngle)
    .rotateX(-startAngle)
    .scale(scale);
  const yScale3d = lineStrips3D()
      .origin(origin)
      .rotateY(startAngle)
      .rotateX(-startAngle)
      .scale(scale);
  const xScale3d = lineStrips3D()
      .origin(origin)
      .rotateY(startAngle)
      .rotateX(-startAngle)
      .scale(scale);
  const zScale3d = lineStrips3D()
      .origin(origin)
      .rotateY(startAngle)
      .rotateX(-startAngle)
      .scale(scale);
  function processData(data, tt, recolor) {
    /* ----------- GRID ----------- */
    const xGrid = svg.selectAll("path.grid").data(data[0], key);
    xGrid
      .enter()
      .append("path")
      .attr("class", "d3-3d grid")
      .merge(xGrid)
      .attr("stroke", "black")
      .attr("stroke-width", 0.3)
      .attr("fill", (d) => (d.ccw ? "#eee" : "#aaa"))
      .attr("fill-opacity", 0.7)
      .attr("d", grid3d.draw);
    xGrid.exit().remove();
    /* ----------- POINTS ----------- */
    const points = svg.selectAll("circle").data(data[1], key);
    function GetColor(x, y){
      console.log("x: %d, y: %d", x, y);
      // return (x > 0) ? 5 : -5 + (y > 0) ? 3 : -3;
      if (x >= 0 && y >= 0) return schemePastel1[0];
      if (x < 0 && y >= 0) return schemePastel1[1];
      if (x < 0 && y < 0) return schemePastel1[2];
      if (x >= 0 && y < 0) return schemePastel1[3];
    }
    if(recolor){
      points
      .enter()
      .append("circle")
      .attr("class", "d3-3d")
      .attr("opacity", 0)
      .attr("cx", posPointX)
      .attr("cy", posPointY)
      .merge(points)
      .transition()
      .duration(tt)
      .attr("r", 3)
      .attr("stroke", (d) => color(colorScale(d.id)).darker(3))
      .attr("fill", (d) => GetColor(d.projected.x - 480, d.projected.y - 250))
      .attr("opacity", 1)
      .attr("cx", posPointX)
      .attr("cy", posPointY);
    }else{
      points
      .enter()
      .append("circle")
      .attr("class", "d3-3d")
      .attr("opacity", 0)
      .attr("cx", posPointX)
      .attr("cy", posPointY)
      .merge(points)
      .transition()
      .duration(tt)
      .attr("r", 3)
      .attr("stroke", (d) => color(colorScale(d.id)).darker(3))
      .attr("opacity", 1)
      .attr("cx", posPointX)
      .attr("cy", posPointY);
    }
    points.exit().remove();
    /* ----------- x-Scale ----------- */
    const xScale = svg.selectAll("path.xScale").data(data[3]);
    xScale
      .enter()
      .append("path")
      .attr("class", "d3-3d xScale")
      .merge(xScale)
      .attr("stroke", "black")
      .attr("stroke-width", 1.5)
      .attr("d", xScale3d.draw);
    xScale.exit().remove();
    /* ----------- y-Scale ----------- */
    const yScale = svg.selectAll("path.yScale").data(data[2]);
    yScale
      .enter()
      .append("path")
      .attr("class", "d3-3d yScale")
      .merge(yScale)
      .attr("stroke", "black")
      .attr("stroke-width", 1.5)
      .attr("d", yScale3d.draw);
    yScale.exit().remove();
    /* ----------- z-Scale ----------- */
    const zScale = svg.selectAll("path.zScale").data(data[4]);
    zScale
      .enter()
      .append("path")
      .attr("class", "d3-3d zScale")
      .merge(zScale)
      .attr("stroke", "black")
      .attr("stroke-width", 1.5)
      .attr("d", zScale3d.draw);
    zScale.exit().remove();
    /* ----------- y-Scale Text ----------- */
    const yText = svg.selectAll("text.yText").data(data[2][0]);
    function GetSuffix(y){
      if (y==10){
        return "% [Arithmetic Intensity]";
      }else{
        return "%";
      }
    }
    yText
      .enter()
      .append("text")
      .attr("class", "d3-3d yText")
      .attr("font-family", "system-ui, sans-serif")
      .merge(yText)
      .each(function (d) {
        d.centroid = { x: d.rotated.x, y: d.rotated.y, z: d.rotated.z };
      })
      .attr("x", (d) => d.projected.x)
      .attr("y", (d) => d.projected.y)
      .text((d) => (-d.y*10 + 100)/2 + GetSuffix(d.y))
//      .text((d) => (d.y <= 0 ? d.y : ''));
    yText.exit().remove();
    /* ----------- x-Scale Text ----------- */
    const xText = svg.selectAll("text.xText").data(data[3][0]);
    xText
      .enter()
      .append("text")
      .attr("class", "d3-3d xText")
      .attr("font-family", "system-ui, sans-serif")
      .merge(xText)
      .each(function (d) {
        d.centroid = { x: d.rotated.x, y: d.rotated.y, z: d.rotated.z };
      })
      .attr("x", (d) => d.projected.x)
      .attr("y", (d) => d.projected.y)
      .attr("z", (d) => d.projected.z)
      .text((d) =>  d.x == 10 ? "[Hardware Enablement]" : "")
    xText.exit().remove();
    /* ----------- x-Scale Text ----------- */
    const zText = svg.selectAll("text.zText").data(data[4][0]);
    zText
      .enter()
      .append("text")
      .attr("class", "d3-3d zText")
      .attr("font-family", "system-ui, sans-serif")
      .merge(zText)
      .each(function (d) {
        d.centroid = { x: d.rotated.x, y: d.rotated.y, z: d.rotated.z };
      })
      .attr("x", (d) => d.projected.x)
      .attr("y", (d) => d.projected.y)
      .attr("z", (d) => d.projected.z)
      .text((d) =>  d.z == 10 ? "[Work Reduction]" : "")
    zText.exit().remove(); 
    selectAll(".d3-3d").sort(points3d.sort);
  }
  function posPointX(d) {
    return d.projected.x;
  }
  function posPointY(d) {
    return d.projected.y;
  }
  function init() {
    xGrid = [];
    scatter = [];
    yLine = [];
    xLine = [];
    zLine = [];
    let cnt = 0; 
    for (let z = -j; z < j; z++) {
      for (let x = -j; x < j; x++) {
        xGrid.push({ x: x, y: 0, z: z}); // grid position
        scatter.push({
          x: x,
          y: randomNormal(0, 0.8)()*3,
          // y: randomUniform(9, -9)(),
          z: z,
          id: "point-" + cnt++,
        });
      }
    }
    range(-10, 11, 1).forEach((d) => {
      yLine.push({ x: 0, y: -d, z: 0 });
      xLine.push({ x: -d, y: 0, z: 0 });
      zLine.push({ x: 0, y: 0, z: -d });
    });
    const data = [
      grid3d(xGrid),
      points3d(scatter),
      yScale3d([yLine]),
      xScale3d([xLine]),
      zScale3d([zLine]),
    ];
    processData(data, 1000, true);
  }
  function dragStart(event) {
    mx = event.x;
    my = event.y;
  }
  function dragged(event) {
    beta = (event.x - mx + mouseX) * (Math.PI / 230);
    alpha = (event.y - my + mouseY) * (Math.PI / 230) * -1;
    const data = [
      grid3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)(xGrid),
      points3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)(scatter),
      yScale3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)([yLine]),
      xScale3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)([xLine]),
      zScale3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)([zLine]),
    ];
    processData(data, 0, false);
  }
  function dragEnd(event) {
    mouseX = event.x - mx + mouseX;
    mouseY = event.y - my + mouseY;
  }
  selectAll("button").on("click", init);
  init();
});
</script>
