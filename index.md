# Real-time Realistic Rendering and Lighting of Forests

森林光线要点

* 基于视角的反射行为
* 基于地表斜面的反射行为
* 相对效应 高光点出现于太阳正对面
* 银边效果
* 天光

## Eric Bruneton模型

### 森林

由densitymap生成树的位置集合。seed表示一颗树的三维坐标、旋转、缩放、颜色、特征等。提及coveragemap有一个密度channel和一个表达地面被树木覆盖百分比的channel。近处的树根据seed数据表达，远处的则由coveragemap表达成地面的某种贴花。这两种数据都跟着terrain quadtree走的，提到说seed数据是一个大的vertexbufferobject，coveragemap是一个texturearray,每层对应一个terrain tile。提及LOD，tree的大小超过3个像素则0-L级别使用coveragemap，在L级别则用seed,L是quadtree的最小细分级别。



### 树

提出一种全新的表达方式 "z-field"。它基于预计算的depthmap表达，加上渲染和光照模型的一些提升。每个树的模型都会预烘焙一组低分辨率的view，每个view的纹素包含4个值：树的最小和最大深度、环境遮罩、透明度。深度计算是设置一个垂直于视线方向且穿过树中心的plane，计算就是像素点到这个平面的距离。环境遮罩是把单独一棵树放在平地上进行预处理的。总共用了181个视线角度，均分到树的半球空间，每个view用256x256 rgba8的纹理。45Mb每个模型。

实时重建三维树的形状，需取181个角度中的3个。具体步骤如下：

1. 找最接近相机视角$\omega_0$的三个预计算角度及其权重。
2. 找view Ray和plane相交的点p



3. p点正交投影到前面找的三个最近的view。能采样到三个纹素。通过公式插值。深度加权求和再除以透明度加权求和，这样p往v的方向偏移这段距离就得到点q

$$
q = p + \left(\sum w_i z_i\right) / \left(\sum w_i \alpha_i\right) v
$$

4. 点q也正交投影到那三个view，再采样三个纹素。由公式插值。$p_v$用于片元depth计算。

$$
p_v = p + \left(\sum w_i z'_i / \sum w_i \alpha'_i\right) v
$$



5. 重复前3步，不过视线方向改成了光源方向，找到$p_l$,这里光线经过前面的$p_v$



$p_v$的透明度$\alpha_v = \sum w_i \alpha'_i$及AO$\sum w_i \delta'_i / \alpha_v$用于光照计算。

### 太阳光

模型中森林受太阳光由以下三个系数决定：

消光系数 extinction coefficient $\tau$

相位函数 phase function $p$

反照率 albedo $\rho$

假定了光的散射主要发生在树的表面，所以只考虑$p_v$到$p_l$的光路。太阳光$I_t$散射到$p_v$就可以如此表达：

$$
\rho p(v,l) \exp(-\tau \|p_l - p_v\|) L_{sun}
$$

也就是太阳光经过$p_l$到$p_v$间的衰减(extinction)，再从光线方向到视线方向的折射(phase)后，再经过反照率拿到最终光强。这里phase模型中是各向同性。这样能得到叶片组大小的银边效果和自阴影。为了得到叶片大小的效果，加了经验热点公式.$a_1$振幅和$a_2$宽度可调节。

$$
I_t = \rho p(v,l) \exp(-\tau \|p_l - p_v\|) F(v,l) V(p_l) L_{sun}
$$

$$
F(v,l) \stackrel{def}{=} 1 - a_1[1 - \exp(-a_2 \cos^{-1}(v \cdot l))]
$$

其中$V(p_l)$是太阳的可见性，由CSM计算（感觉就是阴影）

### 天光

在点$p_v$反射的天光$J_t$用平均天空辐照度乘上AO来近似

$$
L_{sky} \delta(p_v)
$$

假定树的内外的遮罩是不相关的，则有$\delta = \delta_v \delta_e$其中$\delta_e$是由其他树产生的遮罩。这个值用树周围圆筒型的洞$\delta_h$来近似

半径$2 / \sqrt{\Lambda_h(x)}$所谓到最近树的平均距离。这个洞的深度$d$,平地半径$r$,AO计算为

$$
1 / (1 + d^2 r^{-2})
$$

再乘上在斜坡上的那个洞。$(1 + n \cdot u_z) / 2$,比较糙但高效

$$
J_t(p_v) = \frac{p}{2} \left(\int_{\Omega^2} p(v,w)  dw\right) \frac{\delta_v(p_v)(1 + n \cdot u_z)}{1 + d^2(p_v) \Lambda_h(x)/4} L_{sky}
$$

### 地面光

$\Gamma(x)$ coverage of ground by trees in top view

$A$ average horizontal tree area $E[\pi R^2]$

$\Lambda$ tree density on ground, in transformed space

树下的地面更黑，所以调整有树的地面的AO。地面AO $\delta_g$由经验公式$\Delta(x)$随到树木的距离的减少而增加。$\Delta(x)$在树下必须小于1，而平均值必须是1。因此，我们通过coveragemap计算它

$$
\Delta(x) \stackrel{def}{=} 1 - a_3[\Gamma(x) - \Lambda_h(x) A]
$$

$a_3$是自定义对比因子。如果r是地面BRDF的话，太阳和天空的地面反射计算如下：

$$
I_g(x) = r(v,l,n) \max(n \cdot l, 0) V(x) L_{sun}
$$

$$
J_g(x) = \delta_g \Delta(x) \int_{\Omega^+} r(v,w,u_z) u_z \cdot w  dw L_{sky}
$$

### 远处树的表达和渲染

用shader-map表达，terrain map存了俯视角中树和地面的比例。树的像素的重建用4个view-light相关的系数和4个辐照度相关的系数。嗯，不感兴趣。

### 无缝过渡

将怎么把coveragemap的树无缝过渡到近景树。



### 限制

45Mb一个模型数据量大限制同屏树种类

树不能和风交互
