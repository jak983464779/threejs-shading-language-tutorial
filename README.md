# threejs-shading-language-tutorial

从今天 2025年01月21日 开始学习和探索 Three.js 着色器语言 TSL。

<img src="https://raw.githubusercontent.com/puxiao/notes/master/imgs/wechat.jpg" style="zoom: 67%;" />

<br>

### TSL简介

* **TSL 抹平了  GLSL 和 WGSL 着色器语言的差异**

  你使用 TSL 编写一套着色器代码可以被分别编译成：

  * 通过 GLSLNodeBuilder 编译成适用于 WebGL 2 的 GLSL
  * 通过 WGSLNodeBuilder 编译成适用于 WebGPU 的 WGSL
  * GLSLNodeBuilder 和 WGSLNodeBuilder 都继承于 NodeBuilder，你甚至可以基于 NodeBuilder 扩展到任何着色器编程语言。

  

* **TSL 由一个个 Node 节点构成，不同 Node 节点扮演了不同的功能角色**

  这些 Node 节点可以复用、自由组合，最终可以实现一个个复杂的着色器应用场景。

  你也可以理解成 **"把原本大段大段的 顶点、片段阶段着色器代码 拆分成不同的 Node 节点"**。

  

* **绝大多数 Node 节点实际上就是将原本材质中某个属性值改为一个个可编程的函数**

  例如：

  * 和材质颜色有关的是 colorNode
  * 和位置有关的 positionNode
  * 和MeshPhongMaterial材质 shininess 属性有关的 metalnessNode
  * ...

* **最后，TSL 还会自动帮我们优化最终输出的着色器代码**



<br>

### TSL文档

目前 TSL 文档还处于内测中，可以通过下面方式本地构建查看：

```
git clone --depth=1 https://github.com/mrdoob/three.js.git

pnpm i
pnpm run build-docs
pnpm run dev
```

接下来就可以访问：http://localhost:8080/docs_new/three/0.172.0



<br>

### 本教程写作计划

* [x] Three.js着色语言(TSL)规范 中文版翻译
* [ ] TSL 一些简单示例
* [ ] TSL 复杂场景示例



<br>

我也是边学习边写教程的，若发现有讲解错误的地方，欢迎指正。

**大家都是 TSL 小白新手，一起加油！**

