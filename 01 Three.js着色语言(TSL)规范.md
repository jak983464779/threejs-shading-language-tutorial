# 01 Three.js着色语言(TSL)规范

以下是对 Threejs Shading Language 英文的机翻 + 个人翻译。

想看英文原文的，可以查看：https://github.com/mrdoob/three.js/wiki/Three.js-Shading-Language

特别提醒：

* 本文档是基于 2025.01.21 TSL 规范基础上翻译的。
* 本文将 "Threejs Shading Language" 简称为 TSL 或 Threejs着色语言。
* 本文并不会严格按照英文原文的方式排版
* 对于一些比较重要的英文词语，我会在翻译中文的同时使用 `XX(xx)` 的形式保留英文原词。
* 对于一些拗口、不容易理解的英文原文，我会在下面增加一些 "译者注"。
* 若有翻译、理解不正确的地方，还请批评指正。



<br>

## TSL 规范

* 介绍
  * 为什么选择 TSL ？
  * 简单示例
  * 构建体系
* 常量和显示转换
* 转换
* 统一变量
  * 更新
* 分量重组
* 运算符
* Fn函数
* 条件语句
* 数学
* 方法链
* 纹理
* 属性
* 坐标位置
* 法线
* 切线
* 双切线
* 相机
* 模型
* 屏幕
* 视口
* 混合模式
* 反射
* UV工具
* 插值
* 随机
* 振荡器
* 方向与颜色转换
* 函数
* 节点材质
  * 基础部分
  * 虚线节点材质
  * Phong网格节点材质
  * 标准网格节点材质
  * 物理网格节点材质
  * 精灵节点材质
* 将常见的 GLSL 与 TSL 属性对比表



<br>

## TSL介绍

### 为什么选择 TSL ？

对于大多数开发人员来说创建着色器一直都是一个高级步骤，许多游戏开发人员甚至从来没有从头开始创建过 GLSL 代码。当今业界采用的着色器图形解决方案使开发人员能够更专注于动态，以创建必要的图形效果来满足他们项目的需求。

本项目的目标是创建一个易于使用的着色器创建环境。实现复杂的效果，最初需要在渲染器(`renderer`)上便携，以后就可以在 `TSL` 上编写。

TSL 除了简化着色器创建之外，还带来了其他好处： 

* `渲染器无关性(renderer agnostic)`：代码不依赖于特定的渲染器。
* `摇树优化(tree shaking)`：代码优化的稳定性，材质的所有复杂逻辑都可以导入到不同模块中，并且在这个过程中进行 "摇树优化" 而不会出现问题。



<br>

### 简单示例

细节图(detail map)使游戏中的模型看起来更加真实，因为它为模型表面添加了裂纹和凸起等微小细节。在下面例子中我们将缩放 UV 以改善近距离观察时的细节，并与基础纹理相乘。

#### 以前实现的方式

我们通过 `.onBeforeCompile()` 实现：

```
const material = new THREE.MeshStandardMaterial();
material.map = colorMap;
material.onBeforeCompile = ( shader ) => {

	shader.uniforms.detailMap = { value: detailMap };

	let token = '#define STANDARD';

	let insert = /* glsl */`
		uniform sampler2D detailMap;
	`;

	shader.fragmentShader = shader.fragmentShader.replace( token, token + insert );

	token = '#include <map_fragment>';

	insert = /* glsl */`
		diffuseColor *= texture2D( detailMap, vMapUv * 10.0 );
	`;

	shader.fragmentShader = shader.fragmentShader.replace( token, token + insert );

};
```

在后续中任何简单的效果调整都会使 .onBeforeCompile 中代码变得越来越复杂。像我们今天在 Threejs 社区中看到无数这种类型的参数化材质，它们彼此之间无法沟通、重用。



<br>

#### 新的实现方式

如果采用 TSL 代码如下：

```
import { texture, uv } from 'three/tsl';

const detail = texture( detailMap, uv().mul( 10 ) );

const material = new THREE.MeshStandardNodeMaterial();
material.colorNode = texture( colorMap ).mul( detail );
```

是不是代码简单很多了。

并且 TSL 能够将代码编译成不同的输出，例如 WebGPU 对应的 WGSL 和 WebGL对应的 GLSL。

此外还可以自动优化着色器图形 `节点(Node`) 代码。让开发人员可以专注于生产力，一些图形管理底层代码交给 `节点系统(Node System)` 处理。

图形着色器的另外一个重要特性是：我们不再需要关心组件的创建顺序，因为 `节点系统(Node System)` 只会声明并包含一次。

假设您将 `positionWorld` 导入到代码中，即使多个组件都试用它，为获得 `positionWorld` 而执行的计算也只会执行一次，其他的(如 normalWorld、modelPosition)也一样。



<br>

### 构建体系

所有 TSL 组件都从 `节点(Node)` 扩展而来。节点允许与任何其他组件进行通信，值转换可以是自动的或手动的。节点可以接受父级期望的输出值并修改自己的输出片段。

在着色器构建过程中可以使用 `摇树优化(tree shaking)` 来调节它们，节点将拥有几何体、材质、渲染器等其他重要信息，这些信息将会影响输出的类型和值。

负责构建代码的类是 `节点构建器(NodeBuilder)`，它可以扩展到任何输出编程语言。

目前 `节点构造器(NodeBuilder)` 有 2 个扩展类：

* 针对 WebGPU 的 WGSLNodeBuilder
* 针对 WebGL 2 的 GLSLNodeBuilder

构建过程由 3 个部分组成：设置(setup)、解析(anslyze)、生成(generate)

* `设置(setup)`：使用 TSL 为节点输出创建完整的自定义代码。节点本身可以使用其他节点，可以没有数据输入但一定有数据输出。
* `解析(anslyze)`：此过程将检查创建的节点，为生成代码片段创建有用的信息，例如是否需要创建缓存、变量来优化节点。
* `生成(generate)`：每一个节点将生成一段 代码片段(string)。任何节点也可以在 着色器流程 中创建代码，支持多行。

节点还有一个由 `update()` 函数调用的本地更新过程，这些事件由 `frame`、`render` 或 `对象绘制(object draw)` 调用。

节点还支持使用 `serialize()` 和 `deserialize()` 来序列化或反序列化。



<br>

## 常量和显式转换

输入函数可用于创建常量并进行显式转换。

> 如果输出和输入的类型不同，也会自动执行转换。

| 函数                                                   | 返回常量或类型转换 |
| ------------------------------------------------------ | ------------------ |
| float( node\|number )                                  | float              |
| int( node\|number )                                    | int                |
| uint( node\|number )                                   | uint               |
| bool( node\|value )                                    | boolean            |
| color( node\|hex\|r,g,b )                              | color              |
| vec2( node\|Vector2\|x,y )                             | vec2               |
| vec3( node\|Vector3\|x,y,z )                           | vec3               |
| vec4( node\|Vector4\|x,y,z,w )                         | vec4               |
| mat2( node\|Matrix2\|a,b,c,d )                         | mat2               |
| mat3( node\|Matrix3\|a,b,c,d,e,f,g,h,i )               | mat3               |
| mat4( node\|Matrix4\|a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p ) | mat4               |
| ivec2( node\|x,y )                                     | ivec2              |
| ivec3( node\|x,y,z )                                   | ivec3              |
| ivec4( node\|x,y,z,w )                                 | ivec4              |
| uvec2( node\|x,y )                                     | uvec2              |
| uvec3( node\|x,y,z )                                   | uvec3              |
| uvec4( node\|x,y,z,w )                                 | uvec4              |
| bvec2( node\|x,y )                                     | bvec2              |
| bvec3( node\|x,y,z )                                   | bvec3              |
| bvec4( node\|x,y,z,w )                                 | bvec4              |



<br>

示例：

```
import { color, vec2, positionWorld } from 'three/tsl';

// 常量赋值
material.colorNode = color( 0x0066ff );

// 显示转换赋值
material.colorNode = vec2( positionWorld ); // 结果 positionWorld.xy
```



<br>

## 转换

也可以使用下面函数执行转换。

| 函数       | 返回常量或类型转换 |
| ---------- | ------------------ |
| .toFloat() | float              |
| .toInt()   | int                |
| .toUint()  | uint               |
| .toBool()  | boolean            |
| .toColor() | color              |
| .toVec2()  | vec2               |
| .toVec3()  | vec3               |
| .toVec4()  | vec4               |
| .toMat2()  | mat2               |
| .toMat3()  | mat3               |
| .toMat4()  | mat4               |
| .toIVec2() | ivec2              |
| .toIVec3() | ivec3              |
| .toIVec4() | ivec4              |
| .toUVec2() | uvec2              |
| .toUVec3() | uvec3              |
| .toUVec4() | uvec4              |
| .toBVec2() | bvec2              |
| .toBVec3() | bvec3              |
| .toBVec4() | bvec4              |



<br>

示例：

```
import { positionWorld } from 'three/tsl';

// 转换
material.colorNode = positionWorld.toVec2(); // 结果 positionWorld.xy
```



<br>

## 统一变量

统一变量(Uniform)可用于更新颜色、光照、变换等变量的值，而无需重新创建着色器程序。从 GPU 的角度来看他们是真正的变量。

* uniform( boolean | number | Color | Vector2 | Vector3 | Vector4 | Matrix3 | Matrix4, type = null )：动态值



<br>

示例：

```
const posY = uniform( mesh.position.y );

// 可以使用 posY.value 手动更新该值
posY.value = mesh.position.y;

material.colorNode = posY;
```



<br>

### `uniform.on*Update()`

还可以创建更新事件 `uniforms`，用户可以定义：

* .onObjectUpdate(function)：每次在 材质(material) 中使用此节点渲染网格等对象时，它都会更新。
* .onRenderUpdate(function)：每次渲染、共享材质、雾、色调映射时都会更新一次。
* .onFrameUpdate(function)：每帧会更新一次，建议用于每帧只更新一次的值，无论帧的渲染过程(render pass)何时进行，例如计时器(timer)



<br>

示例：

```
const posY = uniform( 0 );

// 或使用事件自动完成
// { object } 是当前渲染对象
posY.onObjectUpdate( ( { object } ) => object.position.y );

material.colorNode = posY;
```



<br>

## 分量重组

分量重组(Swizzling) 是一种允许你使用 TSL 中的特定符号、重新排序或复制向量组件的技术。

> 译者注："swizzling" 单词本意为 "调酒"，这里将其翻译为 "分量重组"。

这是通过组合标识符来实现的：

```
const original = vec3( 1.0, 2.0, 3.0 ); // (x, y, z)
const swizzled = original.zyx; // 分量重组之后 = (3.0, 2.0, 1.0)
```

类似的还有 `xyzw`、`rgba` 或 `stpq` 。



<br>

## 运算符

| 名称                               | 描述               |
| ---------------------------------- | ------------------ |
| .add( node \| value, ... )         | 两个或多个值的加法 |
| .sub( node \| value )              | 两个或多个值的减法 |
| .mul( node \| value )              | 两个或多个值的乘积 |
| .div( node \| value )              | 两个或多个值的除法 |
| .assign( node \| value )           | 合并属性           |
| .mod( node \| value )              | 取余               |
| .modInt( node \| value )           | 取余，余数为整数   |
| .equal( node \| value )            | 是否相等           |
| .notEqual( node \| value )         | 是否不相等         |
| .lessThan( node \| value )         | 是否小于           |
| .greaterThan( node \| value )      | 是否大于           |
| .lessThanEqual( node \| value )    | 是否小于等于       |
| .greaterThanEqual( node \| value ) | 是否大于等于       |
| .and( node \| value )              | 执行逻辑与         |
| .or( node \| value )               | 执行逻辑或         |
| .not( node \| value )              | 执行逻辑非         |
| .xor( node \| value )              | 执行逻辑异或       |
| .bitAnd( node \| value )           | 按位与运算         |
| .bitNot( node \| value )           | 按位非运算         |
| .bitOr( node \| value )            | 按位或运算         |
| .bitXor( node \| value )           | 按位异或运算       |
| .shiftLeft( node \| value )        | 将节点向左移动     |
| .shiftRight( node \| value )       | 将节点向右移动     |



<br>

示例：

```
const a = float( 1 );
const b = float( 2 );

const result = a.add( b ); // 输出: 3
```



<br>

## Fn函数

### `Fn(function)`

可以使用经典 JS 函数或 `Fn()` 接口。主要区别在于 `Fn()` 创建了一个可控环境，允许使用堆栈，在堆栈中可以使用赋值和条件，而经典函数只允许内联方法。

示例：

```
// TSL 函数
const oscSine = Fn( ( [ t = timer ] ) => {

	return t.add( 0.75 ).mul( Math.PI * 2 ).sin().mul( 0.5 ).add( 0.5 );

} );

// 经典 JS 内联函数
export const oscSine = ( t = timer ) => t.add( 0.75 ).mul( Math.PI * 2 ).sin().mul( 0.5 ).add( 0.5 );
```

> 以上两者都可以用来调用 `oscSin(value)` 。



<br>

TSL允许将参数作为对象输入，这对于具有许多可选参数的函数很有用。

示例：

```
const oscSine = Fn( ( { timer = timerGlobal } ) => {
	return timer.add( 0.75 ).mul( Math.PI * 2 ).sin().mul( 0.5 ).add( 0.5   );
} );

const value = oscSine( { timer: value } );
```

<br>

如果想使用(例如 webpack)兼容的导出功能 `tree shaking`，请记得使用 `/*@__PURE__*/`

```
export const oscSawtooth = /*@__PURE__*/ Fn( ( [ timer = timerGlobal ] ) => timer.fract() );
```



<br>

## 条件语句

### If-else

`If-else` 条件语句可以在 `tsnFun()` 中使用。条件语句在 TSL 中使用 `If` 构建：

```
If( conditional, function )
.ElseIf( conditional, function )
.Else( function )
```

> 注意这里的 "If" 开头的 "i" 是大写的



<br>

示例：

在下面的例子中，我们将 y 位置限制为 10。

```
const limitPosition = Fn( ( { position } ) => {

	const limit = 10;

	// 使用 `.toVar()` 转换为变量，以便能够使用赋值
	const result = position.toVec3().toVar();

	If( result.y.greaterThan( limit ), () => {

		result.y = limit;

	} );

	return result;

} );

material.positionNode = limitPosition( { position: positionLocal } );
```



<br>

使用示例 `elseif`：

```
const limitPosition = Fn( ( { position } ) => {

	const limit = 10;

	// 使用 `.toVar()` 转换为变量，以便能够使用赋值
	const result = position.toVec3().toVar();

	If( result.y.greaterThan( limit ), () => {

		result.y = limit;

	} ).ElseIf( result.y.lessThan( limit ), () => {

		result.y = limit;

	} );

	return result;

} );

material.positionNode = limitPosition( { position: positionLocal } );
```



<br>

### 三元运算

与 `If-else` 不同，三元条件将返回一个值，并且可以在 `Fn()` 之外使用。

使用 `select()`：

```
const result = select( value.greaterThan( 1 ), 1.0, value );
```

> 在 JS 中等价的应该是：value > 1 ? 1.0 : value



<br>

## 数学

| 名称                              | 描述                                       |
| --------------------------------- | ------------------------------------------ |
| EPSION                            | 用于处理浮点精度错误的小值。               |
| INFINITY                          | 代表无穷大                                 |
| abs( x )                          | 绝对值                                     |
| acos( x )                         | 反余弦                                     |
| all( x )                          | 如果 x 的所有组成部分都为真，则返回 true   |
| any( x )                          | 如果 x 的任意一组成部分为真，则返回 true   |
| asin( x )                         | 反正弦                                     |
| atan( y, x )                      | 反正切                                     |
| atan2( y, x )                     | 反正切                                     |
| bitcast( x, y )                   | 将值的各个位重新解释为不同的类型。         |
| cbrt( x )                         | 立方根                                     |
| ceil( x )                         | 向上取整                                   |
| clamp( x, min, max )              | 将 x 限定在 min 和 max 范围之间            |
| cos( x )                          | 余弦                                       |
| cross( x, y )                     | 计算两个向量的叉积                         |
| dFdx( p )                         | x 的偏导数                                 |
| dFdy( p )                         | y 的偏导数                                 |
| degrees( radians )                | 将弧度转化为度数                           |
| difference( x, y )                | 计算两个值的绝对差                         |
| distance( x, y )                  | 计算两点之间的距离                         |
| dot( x, y )                       | 点积                                       |
| equals( x, y )                    | 检查是否相等                               |
| exp( x )                          | 返回自然指数                               |
| exp2( x )                         | 返回 2 的参数幂                            |
| faceforward( N, I, Nref )         | 返回指向另一个向量相同方向的向量。         |
| floor( x )                        | 向下取整                                   |
| fract( x )                        | 小数部分                                   |
| fwidth( x )                       | 返回x和y的绝对导数之和。                   |
| inverseSqrt( x )                  | 返回参数平方根的倒数。                     |
| invert( x )                       | 反转 alpha 参数（1. - x）。                |
| length( x )                       | 计算向量的长度                             |
| lengthSq( x )                     | 计算向量的平方长度。                       |
| log( x )                          | 返回参数的自然对数。                       |
| log2( x )                         | 返回参数以 2 为底的对数。                  |
| max( x, y )                       | 取最大                                     |
| min( x, y )                       | 取最小                                     |
| mix( x, y, a )                    | 在两个值之间进行线性插值                   |
| negate( x )                       | 取反                                       |
| normalize( x )                    | 向量归一化                                 |
| oneMinus( x )                     | 返回 1 减去参数                            |
| pow( x, y )                       | 幂                                         |
| pow2( x )                         | 平方                                       |
| pow3( x )                         | 立方                                       |
| pow4( x )                         | 四次方                                     |
| radians( degrees )                | 将度数转换为弧度                           |
| reciprocal( x )                   | 返回参数的倒数 (1/x)                       |
| reflect( I, N )                   | 计算入射矢量的反射方向                     |
| refract( I, N, eta )              | 计算入射矢量的折射方向                     |
| round( x )                        | 四舍五入取整                               |
| saturate( x )                     | 将值限制在 0 和 1 之间                     |
| sign( x )                         | 提取参数的符号                             |
| sin( x )                          | 正弦                                       |
| smoothstep( e0, e1, x )           | 在两个值之间执行 Hermite 插值              |
| sqrt( x )                         | 平方根                                     |
| step( edge, x )                   | 通过比较两个值来生成阶跃函数               |
| tan( x )                          | 正切                                       |
| transformDirection( dir, matrix ) | 用矩阵变换向量的方向，然后对结果进行归一化 |
| trunc( x )                        | 截断参数，删除小数部分                     |



<br>

示例：

```
const value = float( -1 );

// 也可以使用 `value.abs()`
const positiveValue = abs( value ); // output: 1
```



<br>

## 方法链

`方法链(Method chaining)` 仅包含运算符、转换器、数学和一些核心函数。但是这些函数可用于任何 节点。

示例：

`oneMinus()` 是一个数学函数，例如 `abs()`、`sin()`。此示例使用 `.oneMinus()` 类中的内置函数返回一个新节点，而不是像经典 C 函数 `oneMinus( texture(map).rgb )`。

```
// 它将反转纹理颜色
material.colorNode = texture( map ).rgb.oneMinus();
```



<br>

可以在任意节点上使用数学运算符，例如：

```
const contrast = .5;
const brightness = .5;

material.colorNode = texture( map ).mul( contrast ).add( brightness );
```



<br>

## 纹理

`texture( texture, uv = uv(), level = null )`：

类型 vec4，从纹理中检索纹素。



<br>

`cubeTexture( texture, uvw = reflectVector, level = null )`：

类型 vec4，从立方体纹理中检索纹素。



<br>

`triplanarTexture( textureX, textureY = null, textureZ = null, scale = float( 1 ), position = positionLocal, normal = normalLocal )`：

类型 vec4，根据提供的参数使用三平面贴图映射计算纹理。



<br>

## 属性

`attribute( name, type = null, default = null )`：

类型为 any，使用名称和类型获取几何属性。



<br>

`uv( index = 0 )`：

类型为 vec2，UV 属性名为 `uv + index` 。



<br>

`vertexColor( index = 0 )`：

类型为 color，设置节点指定索引的顶点颜色。



<br>

## 坐标位置

* positionGeometry：几何体的位置坐标

* positionLocal：局部相对坐标

  > positionLocal 表示通过蒙皮(skinning)、变形器(morpher)等进行修改后的位置。

* positionWorld：世界坐标

* positionWorldDirection：归一化的世界方向

* positionView：视线位置

* positionViewDirection：归一化的视线方向

> 以上类型均为 vec3



<br>

## 法线

* normalGeometry：几何体法线

* normalLocal：局部法线

* normalView：视图法线

* normalWorld：世界法线

* transformedNormalView：视图空间中变换后的法线

* transformedNormalWorld：世界空间中变换后的法线

* transformedClearcoatNormalView：在视图空间中变换后的透明涂层法线

  > `transformedXxx` 这种表示经过蒙皮(skinning)、变形器(morpher)等修改后的法线

> 以上类型均为 vec3



<br>

## 切线

* tangentGeometry：几何体切线
* tangentLocal：局部切线
* tangentView：视图切线
* tangentWorld：世界切线
* transformedTangentView：视图空间中变换后的切线
* transformedTangentWorld：世界空间中变换后的切线

> 以上类型均为 vec3



<br>

## 双切线

* bitangentGeometry：几何体中归一化的双切线
* bitangentLocal：局部空间中归一化的双切线
* bitangentView：视图空间中归一化的双切线
* bitangentWorld：世界空间中归一化的双切线
* transformedBitangentView：视图空间变换后归一化的双切线
* transformedBitangentWorld：世界空间变换后归一化的双切线

> 以上类型均为 vec3



<br>

## 相机

* cameraNear：相机的近平面距离，类型为 float
* cameraFar：相机的远平面距离，类型为 float
* cameraLogDepth：相机的对数深度值，类型为 float
* cameraProjectionMatrix：相机的投影矩阵，类型为 mat4
* cameraProjectionMatrixInverse：相机的逆投影矩阵，类型为 mat4
* cameraViewMatrix：相机的视图矩阵，类型为 mat4
* cameraWorldMatrix：相机的世界矩阵，类型为 mat4
* cameraNormalMatrix：相机的法线矩阵，类型为 mat4
* cameraPosition：相机的世界位置，类型为 vec3



<br>

## 模型

* modelDirection：模型的方向，类型为 vec3
* modelViewMatrix：模型的视图空间矩阵，类型为 mat4
* modelNormalMatrix：模型的法线矩阵，类型为 mat3
* modelWorldMatrix：模型的世界矩阵，类型为 mat4
* modelPosition：模型的位置，类型为 vec3
* modelScale：模型的缩放，类型为 vec3
* modelViewPosition：模型的视图空间位置，类型为 vec3
* modelWorldMatrixInverse：模型的逆世界矩阵，类型为 mat4
* highpModelViewMatrix：使用 64 位 CPU 计算的模型视图空间矩阵，类型为 mat4
* highpModelNormalViewMatrix：使用 64 位 CPU 计算的模型视图空间法线矩阵，类型为 mat3



<br>

## 屏幕

屏幕节点将返回与当前 `帧缓冲区(frame buffer)` 相关的值，可以是归一化的，也可以是考虑当前 `像素比(Pixel Ratio)` 的物理像素单位。

* screenUV：返回归一化的帧缓冲区坐标，类型为 vec2
* screenCoordinate：以物理像素单位返回帧缓冲区坐标，类型为 vec2
* screentSize：以物理像素单位返回帧缓冲区大小，类型为 vec2



<br>

## 视口

`视口(viewport)` 受到 `renderer.setViewport()` 中定义的区域影响，与渲染器中定义的 `逻辑像素单位(logical pixel units)` 不同，它使用 `物理像素单位(physical pixel units)` 来考虑当前的像素比。

* viewportUV：归一化的视口坐标，类型为 vec2
* viewport：以物理像素单位返回视口尺寸，类型为 vec4
* viewportCoordinate：以物理像素单位返回视口坐标，类型为 vec2
* viewportSize：以物理像素单位返回视口大小，类型为 vec2



<br>

## 混合模式

* blendBurn( a, b )：返回刻录混合模式结果
* blendDodge( a, b )：返回减淡混合模式结果
* blendOverlay( a, b )：返回覆盖混合模式结果
* blendScreen( a, b )：返回屏幕混合模式结果
* blendColor( a, b )：返回颜色混合模式结果

> 以上返回类型均为 color



<br>

## 反射

* reflectView：计算视图空间中的反射方向
* reflectVector：将反射方向变换至世界空间

> 以上类型均为 vec3



<br>

## UV工具

* matcapUV：`材质捕获(matcap)纹理` 的 UV 坐标

  > 译者注：matcap 是单词 "material capture(捕获)" 的简写

* rotateUV( uv, rotation, centerNode = vec2( 0.5 ) )：围绕中心点旋转 UV 坐标

* spherizeUV( uv, strength, centerNode = vec2( 0.5 ) )：利用围绕中心点的球面效果扭曲 UV 坐标

* spritesheetUV( count, uv = uv(), frame = float( 0 ) )：根据帧数、UV 坐标 和帧索引计算精灵图的 UV 坐标

* equirectUV( direction = positionWorldDirection )：根据方向向量计算等矩形映射的 UV 坐标

> 以上返回类型均为 vec2



<br>

示例：

```
import { texture, matcapUV } from 'three/tsl';

const matcap = texture( matcapMap, matcapUV );
```



<br>

## 插值

* remap( node, inLow, inHigh, outLow = float( 0 ), outHigh = float( 1 ) )：将一个值从一个范围重新映射到另外一个范围
* remapClamp( node, inLow, inHigh, outLow = float( 0 ), outHigh = float( 1 ) )：将一个值从一个范围重新映射到另一个范围，并进行限制

> 以上返回类型均为 any



<br>

## 随机

* hash( seed )：从给定的 "种子(seed)" 生成 [0,1] 范围内的哈希值，返回类型为 float
* range( min, max )：生成介于最小值和最大值之间的一系列属性值。当您想在实例之间而不是像素之间随机化值时，属性随机化非常有用。返回类型为 any



<br>

## 振荡器

* oscSine( timer = timerGlobal )：根据计时器产生正弦波振荡
* oscSquare( timer = timerGlobal )：根据计时器产生方波振荡
* oscTriangle( timer = timerGlobal )：根据计时器产生三角波振荡
* oscSawtooth( timer = timerGlobal )：根据计时器产生锯齿波振荡

> 以上返回类型均为 float



<br>

## 方向与颜色转换

* directionToColor( value )：将方向向量转换为颜色，返回类型为 color
* colorToDirection( value )：将颜色转换为方向向量，返回类型为 vec3



<br>

## 函数

### `.toVar(name=null)`

要从节点创建变量，请使用 `.toVar()`。

参数 name 用于为其添加名称，否则节点系统将自动命名，它在调试或使用 `wgslFn` 访问时很有用。

```
const uvScaled = uv().mul( 10 ).toVar();

material.colorNode = texture( map, uvScaled );
```



<br>

### `varying(node,name=null)`

假设你想在 `顶点阶段(vertex stage)` 优化其中某些计算，但又在 `片段阶段(frame stage)` 类似于 `material.colorNode` 的插槽中使用它。

> 译者注：直白一点就是说在 `顶点阶段` 创建一个变量 传递到 `片段阶段`。

例如：

```
// 乘法(multiplication)将在顶点阶段执行
const normalView = varying( modelNormalMatrix.mul( normalLocal ) );

// 归一化(normalize)将在片段阶段执行
// 因为默认情况下 .colorNode 是片段阶段的插槽
material.colorNode = normalView.normalize();
```

`varying` 的第一个参数 `modelNormalMatrix.mul( normalLocal )` 将在顶点阶段执行，并且 `varying()` 返回的值将是可变的，因为我们在 WGSL/GLSL 中使用它，这可以优化片段阶段的额外计算。

第二个参数允许你为变量添加自定义名称。



<br>

如果 `varying()` 仅添加到 `.positionNode`，它将仅返回一个简单变量，而不会创建真正的变量。

> 译者注：用代码来解释上面这句话
>
> ```
> // 这样使用只会返回一个简单变量，不会创建 varying
> const position = varying(positionNode.positionNode);
> 
> // 正确的使用方式是直接对节点使用
> const position = varying(positionNode);
> ```
>
> 这是因为：
>
> 1. `.positionNode` 已经是一个处理过的节点属性
> 2. 这种情况下 `varying()` 无法正确识别需要在着色器之间传递的数据
> 3. 系统会退化为返回一个普通变量而不是创建真正的 varying 变量
>
> 总之，在使用 varying() 时要直接对原始节点使用，而不是对节点的属性使用。



<br>

## 节点材质

请参阅下文了解 `节点材质(NodeMaterial)` 的更多输入信息。

<br>

#### 核心

* fragmentNode：替换片段阶段使用的内置材质逻辑，类型为 vec4。
* vertexNode：替换顶点阶段使用的内置材质逻辑，类型为 vec4。
* geometryNode：允许执行 TSL 函数来处理几何体，类型为 Fn()



<br>

#### 基本的

* .colorNode：替换 `material.color * material.map` 逻辑，对应 `materialColor`，类型为 vec4
* .depthNode：自定义 `depth` 输出，对应 `depth`，类型为 float
* .opacityNode：替换 `material.opacity * material.alphaMap` 逻辑，对应 `materialOpacity`，类型为 float
* .alphaTestNode：设置阈值以丢弃低于不透明的像素，对应 materialAlphaTest，类型为 float



<br>

#### 灯光

* .emissiveNode：替换 `material.emissive * material.emissiveIntensity * material.emissiveMap` 逻辑，对应 materialEmissive，类型为 color
* .normalNode：表示视图空间中的法线方向，替换 `material.normalMap * material.normalScale` 和 `material.bumpMap * material.bumpScale` 逻辑，对应 materialNormal，类型为 vec3
* .lightsNode：定义材质将使用的灯光和照明模型。类型为 `lights()`
* .envNode：替换 `material.envMap * material.envMapRotation * material.envMapIntensity`，类型为 color



<br>

#### 背景

* .backdropNode：在应用 `镜面反射(specular)` 之前渲染节点输入，这对透视(transmission)和折射(refraction)材质很有用。类型为 vec3

* .backdropAlphaNode：定义 `backdropNode` 的透明通道，类型为 float。

  > 译者注：alpha 指  rgba 中的 a 的分量值



<br>

#### 位置

* .positionNode：表示局部空间中的顶点位置，替换 `material.displacementMap * material.displacementScale + material.displacementBias` 逻辑，对应 positionLocal，类型为 vec3



<br>

#### 阴影

* .castShadowNode：控制材质投射阴影的颜色和透明度，类型为 vec4
* .receivedShadowNode：处理投射在材质上的阴影，类型为 `Fn()`
* .shadowPositionNode：定义世界空间中阴影投影位置，对应 shadowPositionWorld，类型为 vec3
* aoNode：替换 `material.aoMap * aoMapIntensity` 逻辑，对应 materialAO，类型为 float



<br>

#### 输出

* .mrtNode：定义一个与 `pass()` 中定义的 MRT 不同的 MRT。类型为 `mrt()`
* .outputNode：定义材质的最终输出，对应 output，类型为 vec4



<br>

## 虚线节点材质

* .dashScaleNode：替换 `material.scale` 逻辑，对应 materialLineScale
* .dashSizeNode：替换 `material.dashSize` 逻辑，对应 materialLineDashSize
* .gapSizeNode：替换 `material.gapSize`，对应 `materialLineGapSize`
* .offsetNode：替换 `material.dashOffset`，对应 materialLineDashOffset

> 以上类型均为 float



<br>

## Phong网格节点材质

* .shininessNode：替换 `material.shininess`，对应 materialShininess，类型为 float
* .specularNode：替换 `material.specular`，对应 materialSpecular，类型为 color



<br>

## 标准网格节点材质

* .metalnessNode：替换 `material.metalness * material.metalnessMap`，对应 metalnessNode
* .roughnessNode：替换 `material.roughness * material.roughnessMap`，对应 roughnessNode

> 以上类型均为 float



<br>

## 物理网格节点材质

* .clearcoatNode： 替换 `material.clearcoat * material.clearcoatMap` 逻辑，对应 materialClearcoat，类型为 float
* .clearcoatRoughnessNode：替换 `material.clearcoatRoughness * material.clearcoatRoughnessMap`，对应 materialClearcoatRoughness，类型为 float
* .clearcoatNormalNode：替换 `material.clearcoatNormalMap * material.clearcoatNormalMapScale`，对应 materialClearcoatNormal，类型为 vec3
* .sheenNode：替换 `material.sheenColor * material.sheenColorMap`，对应 materialSheen ，类型为 color
* .iridescenceNode：替换 `material.iridescence`，对应 materialIridescence，类型为 float
* .iridescenceIORNode：替换 `material.iridescenceIOR`，对应 materialIridescenceIOR，类型为 float
* .iridescenceThicknessNode：替换 `material.iridescenceThicknessRange * material.iridescenceThicknessMap`，对应 materialIridescenceThickness，类型为 float
* .specularIntensityNode：替换 `material.specularIntensity * material.specularIntensityMap`，对应 materialSpecularIntensity，类型为 float
* .specularColorNode：替换 `material.specularColor * material.specularColorMap`，对应 materialSpecularColor，类型为 color
* .iorNode：替换 `material.ior`，对应 materialIOR，类型为 float
* .transmissionNode：替换 `material.transmission * material.transmissionMap`，对应 materialTransmission，类型为 color
* .thicknessNode：替换 `material.thickness * material.thicknessMap`，对应 materialTransmission，类型为 float
* .attenuationDistanceNode：替换 `material.attenuationDistance`，对应 materialAttenuationDistance，类型为 float
* .attenuationColorNode：替换 `material.attenuationColor`，对应 materialAttenuationColor，类型为 color
* .dispersionNode：替换 material.dispersion，对应 materialDispersion，类型为 float
* .anisotropyNode：替换 `material.anisotropy * material.anisotropyMap`，对应 materialAnisotropy，类型为 vec2



<br>

## 精灵节点材质

* .positionNode：定义位置
* .rotationNode：定义旋转
* .scaleNode：定义缩放

> 以上类型均为 vec3



<br>

## 将常见的 GLSL 与 TSL 属性对比表

| GLSL              | TSL                    | 类型 |
| ----------------- | ---------------------- | ---- |
| position          | positionGeometry       | vec3 |
| transformed       | positionLocal          | vec3 |
| transformedNormal | normalLocal            | vec3 |
| vWorldPosition    | positionWorld          | vec3 |
| vColor            | vertexColor()          | vec3 |
| vUv`|`uv          | uv()                   | vec2 |
| vNormal           | normalView             | vec3 |
| viewMatrix        | cameraViewMatrix       | mat4 |
| modelMatrix       | modelWorldMatrix       | mat4 |
| modelViewMatrix   | modelViewMatrix        | mat4 |
| projectionMatrix  | cameraProjectionMatrix | mat4 |
| diffuseColor      | material.colorNode     | vec4 |
| gl_FragColor      | material.fragmentNode  | vec4 |
