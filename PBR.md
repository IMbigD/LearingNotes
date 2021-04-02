[toc]

# 2 PBR基本认识和应用

### 2.1 Base Introduction

- **Definition**

  利用真实世界原理，通过数学方法推导或简化或模拟出一系列渲染方程，并依赖图形API渲染出真实画面的技术。

- Phone与PBR区别，理想表示漫反射和高光，

  DetailMap，Smoothness，Metallic，Fresnel，Transparent

### 2.2 History

1. **Lambert** 1760

   计算Diffuse，入射光方向**l**，法线方向**n**

   光强
   $$
   Diffuse=LightStrength*dot(nromal,LightDir)
   $$
   ​	

2. **Smith** 1967

3. **Phone** 1973

   相较于Lambert增加了Specular

4. **Cook-Torrance** 1982

   添加了G几何项、Fresnel，Roughness等

5. **Oren Nayarh** 1994

   相较于Lambert的理想环境下的光模拟，Oren Nayarh 考虑了微表面之间的遮挡

6. **Schlick** 1994

   简化了Phone中计算Specular时的Grossness的指数运算即：
   $$
   Specular=pow(dot(normal,halfDir),Grossness)
   $$
   由以下下公式替代
   $$
   F=F_0+(1-F_0)(1-cosθ)^5
   $$

   $$
   F_0=\frac{n_1-n_2}{n_1+n_2}
   $$

7. **GGX** 2007

   将’‘微表面模型’‘推广到’‘粗糙的半透明材质’‘，

   作为一种描述平面**法线分布函数**，也可用于渲染粗糙不透明物体

8. **Disney principled BRDF** 2012

   特点：艺术导向 Art Directable 而非 Physically Correct

9. **Now BxDF**

   代表性技术：

   - PBR Diffuse for GXX + Smith 2017

   - MultiScattering Diffuse 2018
   - Layers Material 分层材质
   - Mixed Material 
   - Mixed BxDF
   - Advanced Rendering

### 2.3 各引擎中的PBR

1. UE4

   参数有BaseColor、Roughness、Metallic、Specular

2. Unity

   参数有Albedo、Smoothness、Metallic、

   Occlusion(材质接收间接光的强度和反射强度)，

   Fresnel(无法自动调节，根据Smoothness调节，Smoothness越低，Fresnel项越低)

---

# 3 PBR基本原理和实现

### 3.1 基础理论推导

满足以下条件的称为PBR光照模型

- 基于微表面模型
- 能量守恒
- 基于物理的BRDF

#### 3.1.1 微表面模型 (Microfacet)

所有表面都是粗糙的，定义一个**Roughness**确定其粗糙度

基于该粗糙度，可以计算出：

​	**某个特定向量的方向与位平面取方向一致**的概率

> 该向量与**光照方向l**、**视角方向v**有关：
>
> *(Phone模型中用于计算Specular)*
>
> 称为**半角向量(Halfway Vector)**

$$
h=\frac{l+v}{||l+v||}
$$

由HalfDir决定的该概率，Roughness决定了Specular聚集程度，类似于Phone中的_Grossness属性



#### 3.1.2 能量守恒 (Energy Conservation)

实际物理情况：光照射到材质表面，分为**反射reflection**与**折射refraction**。反射部分产生**镜面反射Specular**，折射部分进入物体内部，逐渐被物体吸收，同时逐渐进入物体内部发生散射，散射的一部分从物体表面出来作为**漫反射Diffuse**

- PBR简化折射部分，假设材质会完全吸收所有的折射光
- 次表面散射技术（Subsurface Scattering）会计算折射后散开的模拟，（显著提升皮肤、石蜡材质的表现）

**如何做能量守恒？**

简单的做法是

- 先计算出Specular值
- 1-Specular得到折射部分的值



#### 3.1.3 反射方程 (Reflectance Equation)

##### 3.1.3.1 渲染方程

​	渲染方程在理论上给出了一个完美的渲染结果，各种渲染技术只是对该理想结果的近似
$$
L_o = L_e + \int_\Omega f_r\cdot L_i\cdot(\omega_i,n)d\omega_i
$$

- $L_o$ 是p点的出射光亮度。

- $L_e$是p点发出的光亮度。(emission)

- $f_r$是p点入射方向到出射方向光的反射比例，即BxDF，一般为BRDF。

-  $L_i$是p点入射光亮度。

-  $(\omega_i\cdot n)$是入射角带来的入射光衰减

-  $\int_\Omega …d\omega_i$是入射方向半球的积分（可以理解为无穷小的累加和）。

  实时渲染中采用的是渲染方程的一个特例:
  $$
  L_o = \int_\Omega f_r\cdot L_i\cdot(\omega_i,n)d\omega_i
  $$

##### 3.1.3.2 反射方程推导

PBR渲染方程为用于抽象描述PBR光照计算过程特化版本的渲染方程，称为**反射方程**
$$
L_0(p,w_0)=\int_\Omega{f_r(p,w_i,w_o)L_i(p,w_i)n\cdot{w_i}dw_i}
$$
下面对该方程进行解释：

- 辐射度量学(Radiometry)

  >定义：用于描述电磁辐射的手段
  >
  >在多种测量某个曲面在某个方向上的电磁辐射强度的方法中
  >
  >选择**辐射率（Radiance）**:
  >
  >​        用于量化表示**单一方向单位时间单位面积上**的光能
  >
  >
  >
  >光通量等于**单位时间**通过面积1的光能
  >$$
  >\phi = \frac{dQ}{dt}
  >$$
  >再加上**单一方向**
  >$$
  >\frac{d\phi}{d\omega} =\frac{d(\frac{dQ}{dt})}{d\omega}
  >$$
  >再加上**单位面积**
  >
  >
  >
  >$$
  >\frac{d(\frac{d\phi}{d\omega})}{dA} =\frac{d(\frac{dQ}{dt})}{d\omega} 
  >$$
  >面积投影到**单一方向**上
  >
  >$$
  >L = \frac{d(\frac{d\phi}{d\omega})}{dAcosθ} =\frac{d(\frac{dQ}{dt})}{d\omega cosθ}
  >$$
  >
  >简化得
  >$$
  >L = \frac{d^2\phi}{dAd\omega\cosθ} =\frac{d(\frac{dQ}{dt})}{d\omega \cosθ}
  >$$
  >
  >
  >
  >![img](https://img2018.cnblogs.com/blog/1617944/201904/1617944-20190425001447089-1378458803.png)
  >
  >指定一个点$p$，指定一个入射方向(光照方向)$w_i$，出射方向(观察方向)$w_o$则：
  >
  >可以定义：
  >
  >p点受$w_i$方向的辐射率 $L_i(p,\omega_i)$
  >
  >p点向$w_o$方向的辐射率 $L_o(p,\omega_o)$
  >
  >
  >
  >**感性分析：**
  >
  >实际上入射$w_i$的光能会辐射到周围整个半球$\Omega^+$(单向)或整个球面$\Omega$(双向)上，置于不同方向上具体的分布情况由物体的属性决定，由此可以定义一个函数表示$f_r(p,\omega_o.\omega_i)$:
  >
  >- 输入：点p，入射方向(光照方向)$w_i$，出射方向(观察方向)$w_o$
  >
  >- 输出：从$w_i$入射到点p的光，向$w_o$方向的辐射率贡献值
  >
  >或者说**从$-w_o$方向观察点p，观察结果中从$w_i$方向来的光辐射率贡献是多少**
  >
  >由输出的第二种说法来看，如果遍历所有$w_i$的所有光辐率贡献并求和，就可以得到
  >
  >**从$-w_o$方向观察点p的光辐射率** 
  >
  >即：
  >$$
  >L_o(p,\omega_o) = \int_\Omega f_r(p,\omega_o.\omega_i)L_i(p,\omega_i)d\omega_i
  >$$
  >这个式子对吗？
  >
  >仔细看可以发现**:积分内部**和**外部**都是**光辐射率**，有些矛盾
  >
  >---
  >
  >**理性修正：**
  >
  >由：
  >$$
  >L(p,\omega) = \frac{d^2\phi(p,\omega)}{dAd\omega\cosθ}=\frac{d(\frac{d\phi(p,\omega)}{dA})}{d\omega\cos\theta}
  >$$
  >
  >令$\frac{d\phi(p,\omega)}{dA} = E(p,\omega)$称为**光照度**(iraadiance)**单位时间、单位面积**能量 
  >
  >上式可化为
  >$$
  >L(p,\omega) = \frac{dE(p,\omega)}{d\omega\cosθ}
  >$$
  >从而有
  >$$
  >\int_\Omega L(p,\omega)\cosθd\omega = \int_\Omega dE(p,\omega)=E(p,\omega)
  >$$
  >
  >
  >
  >实际上上述对于$f_r(p,\omega_o.\omega_i)$的定义过于模糊
  >
  >准确的定义为
  >$$
  >f_r(p,\omega_o.\omega_i) = \frac{dL_o(p,\omega_o)}{dE_i(p,\omega_i)}
  >$$
  >将某处的接收到的所有方向的能量，转化到各个方向上的能量
  >
  >由定义有
  >$$
  >\int_\Omega dL_o(p,\omega_o) =\int_\Omega f_r(p,\omega_o.\omega_i)dE_i(p,\omega_i)
  >$$
  >得到最终正确结果
  >$$
  >L_o(p,\omega_o)=\int_\Omega f_r(p,\omega_o.\omega_i)L_i(p,\omega_i)\cosθd\omega
  >$$
  >
  >
  >
  

#### 3.1.4 双向分布函数(BRDF)

​	基本上所有的实时渲染管线使用的都是Cook-Torrance BRDF
$$
f_r = k_d f_{lambert} + k_s f_{cook-torrance}
$$
$k_d,k_s$分别表示入射光中被折射和反射的比例
$$
f_{lambert} = \frac{c}{\pi}
$$
$c$表示Albedo或者表面颜色​, 除以π是为了格式化漫反射光

>- 除以$\pi$的原因：(将$E$ irradiance转换为$I$ radiance)
>
>  $E$ 为 $I$ 在球面上的积分
>
>- 此处计算漫反射项时，没有使用法线与入射方向点乘，而是移到了BRDF外面

$$
f_{cook-torrance} = \frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}
$$

$$ f_{cook-torrance}$$为BRDF的高光项(Specular)较为复杂，其中

$D$ 表示法线分布函数(Normal Distibution Function)

$F$ 菲涅尔方程

$G$ 几何函数：描述微平面自成阴影的属性，自遮挡Occlustion会减少反射光线

- UE4 的 $D,F,G$分别使用的是Trowbridge-Reitz GGX， Fresnel-Schlick近似法(Approximation)，Smith's Schlick-GGX

##### 3.1.4.1 法线分布函数

近似表示与某些**向量$h$取向一致的微平面的比率**

给定粗糙度Rougness参数和另一个参数Trowbridge-Reitz GGX（GGXTR）： 
$$
NDF_{GGX TR}(n, h, \alpha) = \frac{\alpha^2}{\pi((n \cdot h)^2 (\alpha^2 - 1) + 1)^2}
$$
其中$\alpha$为表面粗糙度Rougness, $n$为表面法线, $h$用于检测微平面取向的半角向量

常用模型：

- Beckmann[1963]
- Blinn-Phong[1977]
- GGX [2007] / Trowbridge-Reitz[1975]
- Generalized-Trowbridge-Reitz(GTR) [2012]
- Anisotropic Beckmann[2012]
- Anisotropic GGX [2015]

##### 3.1.4.2 菲涅尔方程

表示不同观察方向上，表面上$\frac{入射光}{折射光}$的值，实际上只需要计算得到入射光的比率，根据能量守恒则可以得到折射光能量。

故菲涅尔方程就是根据**观察角度**确定**反射系数**

Fresnel-Schlick 近似法： 
$$
F_{Schlick}(h, v, F_0) = F_0 + (1 - F_0) ( 1 - (h \cdot v))^5
$$
$F_0$为基础反射率(即观察角为0时的反照率)，该基础反射率对于不同材质来说不同，为Vector3 表示对RGB的反射率

$F_0$表示光线被反射的比率，$1-F_0$则为光线被折射到表面内的比率

- 导体材质$0.5<F_0<1.0$
- 电介质常取$F_0 = (0.04,0.04,0.04)$

$h$为半角向量$v$为观察向量

常用模型：

- Cook-Torrance [1982]
- Schlick [1994]
- Gotanta [2014]

##### 3.1.4.3 几何函数

 模拟微平面互相遮挡导致光线能量减小丢失的现象

显然越**粗糙**，微平面产生**自阴影的概率**越高

GGX和Schlick-Beckmann组合而成的模拟函数Schlick-GGX：

$$
G_{SchlickGGX}(n, v, k) = \frac{n \cdot v} {(n \cdot v)(1 - k) + k }
$$
其中

$$
\begin{eqnarray*} k_{direct} &=& \frac{(\alpha + 1)^2}{8} \ k_{IBL} &=& \frac{\alpha^2}{2} \end{eqnarray*}
$$
$\alpha$的值由粗糙度Rougness转换而成，不同引擎的计算方式不同

合并两式(用**Smith函数**)
$$
G(n, v, l, k) = G_{sub}(n, v, k) G_{sub}(n, l, k)
$$
其中$n$法线方向，$v$视线方向，$l$光线方向，$G_{sub}(n, v, k)$表示视线方向的几何遮挡$G_{sub}(n, l, k)$表示光线方向的几何阴影

常用模型：

- Smith [1967]
- Cook-Torrance [1982]
- Neumann [1999]
- Kelemen [2001]
- Implicit [2013]

##### 3.1.4.4 Cook-Torrance反射方程

根据上面的分析，可得

$$
L_o(p,\omega_o) = \int\limits_{\Omega} (k_d\frac{c}{\pi} + k_s\frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}) L_i(p,\omega_i) n \cdot \omega_i d\omega_i
$$
由于Fresnel项隐含了$k_s$所以可以消去
$$
L_o(p,\omega_o) = \int\limits_{\Omega} (k_d\frac{c}{\pi} + \frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}) L_i(p,\omega_i) n \cdot \omega_i d\omega_i
$$

**这个方程完整定义了一个PBR模型**

#### 3.1.5制作PBR材质

常用纹理

- **反射率**（Albedo）：反射率纹理指定了材质表面每一个像素的颜色，若是材质是金属那纹理包含的就是基础反射率。这个跟咱们以前用过的漫反射纹理很是的相似，可是不包含任何光照信息。漫反射纹理一般会有轻微的阴影和较暗的裂缝，这些在Albedo贴图里面都不该该出现，仅仅只包含材质的颜色（金属材质是基础反射率）。
- **法线**（Normal）：法线纹理跟咱们以前使用的是彻底同样的。法线贴图能够逐像素指定表面法线，让平坦的表面也能渲染出凹凸不平的视觉效果。
- **金属度**（Metallic）：金属度贴图逐像素的指定表面是金属仍是电介质。根据PBR引擎各自的设定，金属程度便可以是[0.0，1.0]区间的浮点值也能够是非0即1的布尔值。
- **粗糙度**（Roughness）：粗糙度贴图逐像素的指定了表面有多粗糙，粗糙度的值影响了材质表面的微平面的平均朝向，粗糙的表面上反射效果更大更模d糊，光滑的表面更亮更清晰。有些PBR引擎用光滑度贴图替代粗糙度贴图，由于他们以为光滑度贴图更直观，将采样出来的光滑度使用（1-光滑度）= 粗糙度 就能转换成粗糙度了。
- **环境光遮挡**（Ambient Occlusion，AO）：AO贴图为材质表面和几何体周边可能的位置，提供了额外的阴影效果。好比有一面砖墙，在两块砖之间的缝隙里Albedo贴图包含的应该是没有阴影的颜色信息，而让AO贴图来指定这一块须要更暗一些，这个地方光线更难照射到。AO贴图在光照计算的最后一步使用能够显著的提升渲染效果，模型或者材质的AO贴图通常是在建模阶段手动生成的。

### 3.2 PBR光照实现

3.1讲述了Cook-Torrance反射方程理论模型，现在讨论如何将理论转换为一个直接光照的渲染器

#### 3.2.1 计算辐照度(iradiance)

已知：光源颜色值lightColor = (r,g,b) 某片元位置fragPos，光源位置lightPos，则有

```glsl
vec3  lightColor  = vec3(r, g, b);
vec3  wi          = normalize(lightPos - fragPos);
float cosTheta    = max(dot(N, Wi), 0.0);
// 计算光源在点fragPos的衰减系数
float attenuation = calculateAttenuation(fragPos, lightPos); 
// 英文原版的radiance类型有误，将它改为了vec3
vec3 radiance  = lightColor * (attenuation * cosTheta);
```

如果是平行光则attenuation项没有，如果是点光源项则有。

由此可见，对单纯的每个fragment的计算来说，并不需要积分，对所有片元遍历即可，只有使用IBL(Image Based Light)时候由于光源方向不确定时才需要使用积分。

#### 3.2.2 PBR表面模型

下面使用GLSL来写一个4个点光源的PBR模型的片元着色器

输入：

```glsl
#version 330 core
out vec4 FragColor;
in vec2 TexCoords;
in vec3 WorldPos;
in vec3 Normal;
  
uniform vec3 camPos;
  
uniform vec3  albedo;
uniform float metallic;
uniform float roughness;
uniform float ao;
```

基本计算：

```glsl
void main()
{
    vec3 N = normalize(Normal); 
    vec3 V = normalize(camPos - WorldPos);
    [...]
}
```

##### 3.2.2.1 直接光照(Direct lighting)

计算光照方向$l$, 半角向量$h$, 同时用$a=\frac{1}{d^2}$来模拟点光源衰减

```glsl
vec3 Lo = vec3(0.0);
for(int i = 0; i < 4; ++i) 
{
    vec3 L = normalize(lightPositions[i] - WorldPos);
    vec3 H = normalize(V + L);
  
    float distance    = length(lightPositions[i] - WorldPos);
    float attenuation = 1.0 / (distance * distance);
    vec3 radiance     = lightColors[i] * attenuation; 
    [...] // 还有逻辑放在后面继续探讨，因此故意在for循环缺了‘}’。
```

菲涅尔项$F$用Schilck模型模拟


```glsl
vec3 fresnelSchlick(float cosTheta, vec3 F0)
{
    return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
} 
```

非金属采用$F0$


```glsl
vec3 F0 = vec3(0.04); 
F0      = mix(F0, albedo, metallic);
vec3 F  = fresnelSchlick(max(dot(H, V), 0.0), F0);
```

计算法线分布$N$项(NDF)和几何自遮挡项 $G$

分别使用：

Trowbridge-Reitz GGX

Schlick-GGX


```glsl
float DistributionGGX(vec3 N, vec3 H, float roughness)
{
    float a      = roughness*roughness;
    float a2     = a*a;
    float NdotH  = max(dot(N, H), 0.0);
    float NdotH2 = NdotH*NdotH;
	
    float num   = a2;
    float denom = (NdotH2 * (a2 - 1.0) + 1.0);
    denom = PI * denom * denom;
	
    return num / denom;
}


float GeometrySchlickGGX(float NdotV, float roughness)
{
    float r = (roughness + 1.0);
    float k = (r*r) / 8.0;

    float num   = NdotV;
    float denom = NdotV * (1.0 - k) + k;
	
    return num / denom;
}

float GeometrySmith(vec3 N, vec3 V, vec3 L, float roughness)
{
    float NdotV = max(dot(N, V), 0.0);
    float NdotL = max(dot(N, L), 0.0);
    float ggx2  = GeometrySchlickGGX(NdotV, roughness);
    float ggx1  = GeometrySchlickGGX(NdotL, roughness);
	
    return ggx1 * ggx2;
}
```

```glsl
float NDF = DistributionGGX(N, H, roughness);       
float G   = GeometrySmith(N, V, L, roughness);  
```

而后就能够计算Cook-Torrance BRDF了：

```glsl
vec3 numerator    = NDF * G * F;
float denominator = 4.0 * max(dot(N, V), 0.0) * max(dot(N, L), 0.0);
vec3 specular     = numerator / max(denominator, 0.001);  
```

计算$k_d,k_f$

```glsl
vec3 kS = F;
vec3 kD = vec3(1.0) - kS;
  
kD *= 1.0 - metallic; // 因为金属表面不折射光，没有漫反射颜色，经过归零kD来实现这个规则
```

最终得到$L_o$

```glsl
    const float PI = 3.14159265359;
  
    float NdotL = max(dot(N, L), 0.0);        
    Lo += (kD * albedo / PI + specular) * radiance * NdotL;
}
```

```glsl
vec3 ambient = vec3(0.03) * albedo * ao;
vec3 color   = ambient + Lo;  
```

##### 3.2.2.2 线性颜色空间和HDR渲染

所有的计算都需要在线性空间进行，最终输出的Color值仍然在线性空间中。

为了贴合实际，需要将值域映射到Gamma空间，为了颜色范围更加宽，从而能够表现更多的颜色(通俗来说即：深浅可以有更多种)，需要进行**色调映射**和**伽马校正**

- **色调映射(Tone Mapping)**

  即最亮的物体亮度，和最暗的物体亮度之比为$10^8$， 而人类的眼睛所能看到的范围是$10^5$左右，但是一般的显示器，照相机能表示的只有256种不同的亮度。

  实际上，对于每个人来说，能够感光的范围是不同的，可能在光亮条件下，人的瞳孔较小，可以感受某个标准下RGB值在(0.15, 0.12, 0.19)到(0.95,0.98,0.97)更够感亮更多，然而在暗的条件下，人的瞳孔会变小，可以感受的范围变为(0.02,0.01,0.04)到(0.75,0.70,0.69)，高于最高边界值的都认为为白色，低于最低边界值的都认为为黑色，这就是感光范围存在差异的问题。

  实际上**色调映射**来源于相片打印，照相机打印时的颜色范围有限，小于真实世界人类感受的颜色范围，假设照相机的打印结果范围在(0.10,0.08,0.09)~(0.95,0.98,0.90)，而人能感知(0,0,0)~(1,1,1)的范围，则拍照时在亮处和暗处的细节就会丢失，故需要做一个缩放变换，将0.10~0.95的所有值映射到0~1的空间中，这个缩放变换就叫色调映射。

  可以粗俗的理解为：其中较小的亮度域叫做**LDR(Low Dynamic Range)**较大的亮度域叫**HDR(Heigh Dynamic Range)** 。

  实际上HDR指的是一种图像，是由不同曝光时间的LDR根据最佳细节合成的，由于单次拍照时的亮度色域有限，不断曝光，较暗的地方的细节就会不断累积，从而暗处的细节能够更好的显现出来。HDR则需要用来用256个数字来模拟$2^{16}$所能表示的信息。

    整个Tone Mapping的过程就是首先要根据当前的场景推算出**场景的平均亮度**，再根据这个平均亮度**选取一个合适的亮度域**，再将整个场景映射到这个亮度域得到正确的结果。

- **伽马校正(Gamma correction)**

  RGB值与功率并非简单的线性关系，而是幂函数关系，这个函数的指数称为Gamma值，一般为2.2，而这个换算过程，称为Gamma校正。

  **为什么显示器要Gamma校正呢？**因为人眼对亮度的感知和物理功率不成正比，而是幂函数的关系，这个函数的指数通常为2.2，称为Gamma值。

```glsl
color = color / (color + vec3(1.0)); // 色调映射
color = pow(color, vec3(1.0/2.2)); 	 // 伽马校订
```

此处采用莱因哈特算法（Reinhard operator）对HDR进行Tone Map

#### 3.2.3 使用纹理的PBR

美术制作色纹理贴图(albedo ,AO)通常是sRGB空间的，需要转换到线性空间进行计算

### 3.3 基于图像的光照 (IBL image Based Lighting)

将周围环境作为一个大光源结合CubeMap环境贴图技术

从而可以模拟获得更加真实的环境光照ambient

之前介绍的是光照信息统一的**点光源、平行光**，对于每个fragment来说，只需要循环光源个数次即可得到结果，而如果再用IBL技术，则<u>半球面上每个点的采样结果都不同</u>，则需要**积分计算**，如何实现？

---

- 在渲染时通过采样CubeMap贴图，确认任意方向的场景辐射

```glsl
vec3 radiance = texture(_cubemapEnvironment, w_i).rgb; 
```

- 如何求积分呢？

  解决积分能只考虑一个方向的辐射，要考虑环境贴图的半球$\Omega$的全部可能的方向$\omega_i$，但常规积分方法在片元着色器中开销很是大。为了有效解决积分问题，可采用预计算或预处理的方法。所以，须要深究一下反射方程： $$ L_o(p,\omega_o) = \int\limits_{\Omega} (k_d\frac{c}{\pi} + k_s\frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}) L_i(p,\omega_i) n \cdot \omega_i d\omega_i $$

  可将上述的$k_d$和$k_s$项拆分： $$ L_o(p,\omega_o) = \int\limits_{\Omega} (k_d\frac{c}{\pi}) L_i(p,\omega_i) n \cdot \omega_i d\omega_i+ \int\limits_{\Omega} (k_s\frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}) L_i(p,\omega_i) n \cdot \omega_i d\omega_i $$

  对于Diffuse 和 Specular分别计算irradiance

  

##### 3.3.1 漫反射辐照度 (Diffuse irradiance)

$$
\int\limits_{\Omega} (k_d\frac{c}{\pi}) L_i(p,\omega_i) n \cdot \omega_i d\omega_i
$$

常量移出
$$
k_d\frac{c}{\pi} \int\limits_{\Omega} L_i(p,\omega_i) n \cdot \omega_i d\omega_i
$$

- 假设**p点永远在半球中心**，则所有物体都假设为一个**无限小球体**，p点**法线方向**取决于**观察方向**

  其中$L_i(p,\omega_i)$为原CubeMap从p向$\omega_i$方向采样结果

  积分内的仅仅与$n$有关，而$n$仅仅与$\omega_o$有关，即：

  **积分结果是$\omega_o$的函数**，于是可以提前生成一张新CubeMap，计算了每个$\omega_o$的积分结果，从而无需每次重新计算。(如何计算？像素卷积)

  生成的新cubemap叫做**辐照度图(Irradiance map)**

  下面是cubemap环境图(下图左）和对应的辐照度图（下图右）： ![img](https://oscimg.oschina.net/oscnet/e34524848b7a00c8f490cdc3e2787b6721b.png)

有时也使用**球体图(Equirectangular map)** 则只需要使用一张，可以由6张CubeMap经过算法转换

###### 3.3.1.1 非直接光辐射度光照 indirect irradiance lighting

即如何实现对辐照度图的采样，应用到物体上？

##### 

```glsl
uniform samplerCube irradianceMap;
```

经过表面的法线，得到环境光能够简化成下面的代码：

```glsl
// vec3 ambient = vec3(0.03);
vec3 ambient = texture(irradianceMap, N).rgb;
```

尽管如此，在以前所述的反射方程中，非直接光依旧包含了漫反射和镜面反射两个部分，因此咱们须要加个权重给漫反射。下面采用了菲涅尔方程来计算漫反射因子：

```glsl
vec3 kS = fresnelSchlick(max(dot(N, V), 0.0), F0);
vec3 kD = 1.0 - kS;
vec3 irradiance = texture(irradianceMap, N).rgb;
vec3 diffuse    = irradiance * albedo;
vec3 ambient    = (kD * diffuse) * ao; 
```

```glsl
vec3 fresnelSchlickRoughness(float cosTheta, vec3 F0, float roughness)
{
    return F0 + (max(vec3(1.0 - roughness), F0) - F0) * pow(1.0 - cosTheta, 5.0);
}   
```

考虑了表面粗糙度后，菲涅尔相关计算最终以下：

```glsl
vec3 kS = fresnelSchlickRoughness(max(dot(N, V), 0.0), F0, roughness); 
vec3 kD = 1.0 - kS;
vec3 irradiance = texture(irradianceMap, N).rgb;
vec3 diffuse    = irradiance * albedo;
vec3 ambient    = (kD * diffuse) * ao; 
```

这里就会出现一个问题了：

**不是计算漫反射项吗？怎么还有菲涅尔效应？**

实际上，计算流程是计算indirectLight和directLight，两者都含有Diffuse、Specular项

|                     | Diffuse | Specualr |
| ------------------- | ------- | -------- |
| indirectLight(IBL)  |         |          |
| directLight(场景光) |         |          |



##### 3.3.2 镜面反射辐照度(Specular irradiance)

$$
\int\limits_{\Omega} (k_s\frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}) L_i(p,\omega_i) n \cdot \omega_i d\omega_i
$$

由于同时依赖$\omega_o,\omega_i$显然不可能计算全部入射光线和出射光线，一种解决方法是**分裂近似法(split sum approximation)**:
$$
\begin{eqnarray*} L_o(p,\omega_o) & = & \int\limits_{\Omega} (k_s\frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)} L_i(p,\omega_i) n \cdot \omega_i d\omega_i \ & = & \int\limits_{\Omega} f_r(p, \omega_i, \omega_o) L_i(p,\omega_i) n \cdot \omega_i d\omega_i \end{eqnarray*}
$$

- 第一部分$\int\limits_{\Omega} L_i(p,\omega_i) d\omega_i$ 

  由这部分生成的cubemap称为**预过滤环境图（pre-filtered environment map）**，相似于辐射度图的预计算环境卷积图，但会加入粗糙度。随着粗糙度等级的增长，环境图使用更多的散射采样向量来卷积，建立出更模糊的反射。使用mipmap存储不同模糊等级的结果

- 第二部分$\int\limits_{\Omega} f_r(p, \omega_i, \omega_o) n \cdot \omega_i d\omega_i$

  这部分生成**BRDF积分图（BRDF integration map）**假设全部方向的入射辐射率是全白的（那样$L(p, x) = 1.0$），那就能够用**给定**的粗糙度和一个法线$n$和光源方向$\omega_i$之间的角度或$n \cdot \omega_i$来预计算BRDF的值。
  
  >LUT(look-up Texture)查找图
  >
  >通常为一张2D Texture 用于双输入参数对应结果的直接查找，这里的p,x正好符合，可以使用

```glsl
float lod             = getMipLevelFromRoughness(roughness);

// 采样预过滤环境图
vec3 prefilteredColor = textureCubeLod(PrefilteredEnvMap, refVec, lod);

// 采样BRDF积分图
// x 表示
vec2 envBRDF          = texture2D(BRDFIntegrationMap, vec2(NdotV, roughness)).xy;

// 结合两个结果，F为菲涅尔项
vec3 indirectSpecular = prefilteredColor * (F * envBRDF.x + envBRDF.y) 
```

### 3.3.3 完整的IBL

实际就是计算ambient

首先是声明IBL镜面部分的两个纹理采样器：

```glsl
uniform samplerCube prefilterMap;
uniform sampler2D   brdfLUT; 
```

接着用法线`N`和视线`-V`算出反射向量`R`，再结合`MAX_REFLECTION_LOD`和粗糙度等参数采样预过滤环境图：

```glsl
void main()
{
    [...]
    vec3 R = reflect(-V, N);   

    const float MAX_REFLECTION_LOD = 4.0;
    vec3 prefilteredColor = textureLod(prefilterMap, R,  roughness * MAX_REFLECTION_LOD).rgb;    
    [...]
}
```

而后用视线、法线的夹角及粗糙度采样BRDF查找纹理，结合预过滤环境图的颜色算出IBL的镜面部分：

```glsl
vec3 F        = FresnelSchlickRoughness(max(dot(N, V), 0.0), F0, roughness);
vec2 envBRDF  = texture(brdfLUT, vec2(max(dot(N, V), 0.0), roughness)).rg;
vec3 specular = prefilteredColor * (F * envBRDF.x + envBRDF.y);
```

自此，反射方程的非直接的镜面部分已经算出来了。能够将它和上一小节的IBL的漫反射部分结合起来：

```glsl
vec3 F = FresnelSchlickRoughness(max(dot(N, V), 0.0), F0, roughness);

vec3 kS = F;
vec3 kD = 1.0 - kS;
kD *= 1.0 - metallic;	  
  
vec3 irradiance = texture(irradianceMap, N).rgb;
vec3 diffuse    = irradiance * albedo;
  
const float MAX_REFLECTION_LOD = 4.0;
vec3 prefilteredColor = textureLod(prefilterMap, R,  roughness * MAX_REFLECTION_LOD).rgb;   
vec2 envBRDF  = texture(brdfLUT, vec2(max(dot(N, V), 0.0), roughness)).rg;
vec3 specular = prefilteredColor * (F * envBRDF.x + envBRDF.y);
  
vec3 ambient = (kD * diffuse + specular) * ao; 
```

# 4 PBR与渲染光学原理

### 4.1 光与物质相互作用模式

光与物质的交互模式本质上只有两种**散射(scattering)**和**吸收(absorption)**

- **散射（scattering）决定介质的浑浊程度。** 大多数情况下，固体和液体介质中的颗粒都比光的波长更大，并且倾向于均匀地散射所有可见波长的光。高散射会产生不透明的外观。
- **吸收（absorption）决定材质的外观颜色。** 几乎任何材质的外观颜色通常都是由其吸收的波长相关性引起的。

- **散射（Scattering）和吸收（absorption）都与观察尺度有关**。

  例如观察**水杯中的水(小尺度)**更多的看到的是其**低散射特性(透明)**

  而观察**海洋上的水(大尺度)**更多的看到的是其**高散射特性(高光明显)**

人眼所看到即为**物质散射出的光**，这里的散射指的是广义的散射，即光通过介质后未被吸收的部分(该部分可能RGB值与路径被改变)

##### 4.1.1 光与介质边界交互类型

根据4.1所述，根据光通过介质发生的不同变化，可以将光的交互过程分解为以下类型

![img](https://pic3.zhimg.com/80/v2-b855ab10a0b290f023898332e0360dce_720w.jpg)

- **反射（Reflection）。**光线在两种介质交界处的直接反射即**镜面反射（Specular）**。金属的镜面反射颜色为三通道的彩色，而非金属的镜面反射颜色为单通道的单色。

- **折射（Refraction）。**从表面折射入介质的光，会发生吸收（absorption）和散射（scattering），而介质的整体外观由其散射和吸收特性的组合决定，其中：

- **散射（Scattering）。**折射率的快速变化引起散射，光的方向会改变（分裂成多个方向），但是光的总量或光谱分布不会改变。散射最终被视作的类型与观察尺度有关：

- **次表面散射（Subsurface Scattering）。**观察像素小于散射距离，散射被视作次表面散射**。**

- **漫反射（Diffuse）。**观察像素大于散射距离，散射被视作漫反射**。**

- **透射（Transmission）**。入射光经过折射穿过物体后的出射现象。透射为次表面散射的特例。

- **吸收（Absorption）。**具有复折射率的物质区域会引起吸收，具体原理是光波频率与该材质原子中的电子振动的频率相匹配。复折射率（complex number）的虚部（imaginary part）确定了光在传播时是否被吸收（转换成其他形式的能量）。发生吸收的介质的光量会随传播的距离而减小（如果吸收优先发生于某些波长，则可能也会改变光的颜色），而光的方向不会因为吸收而改变。任何颜色色调通常都是由吸收的波长相关性引起的。

  >######  4.1.1.1 复折射率
  >
  >除了代表光的相速度的实部**n**之外，还用希腊字母**κ**（kappa）表示介质将**光能转为为其他形式能量的吸收性**。**n**和**κ**通常都随波长而变化，两者组合成复数**n +iκ**，称为**复折射率（complex index of refraction）**。
  >
  >##### 4.1.1.2 漫反射与次表面反射一致性
  >
  >![img](https://pic4.zhimg.com/80/v2-954db956c9d289a838994e2ade0fec83_720w.jpg)
  >
  >漫反射即为如图像素点内的光线结果平均值
  >
  >- 像素半径＞>散射最大距离 $\to$ 漫反射
  >- 像素半径 <= 散射最大距离$\to$ 漫反射+次表面反射

### 4.2 不同物质与光交互

不同物质与光线的不同交互类型呈现不同的特点，总结如下：

- **金属（Metal）**。金属的外观主要取决于光线在两种介质的交界面上的直接反射（即镜面反射）。金属的镜面反射颜色为三通道的彩色，R、G、B各不相同。而折射入金属内部的光线几乎立即全部被自由电子吸收，且折射入金属的光不存在散射。
- **非金属（No-Metal）**。非金属即电介质，其的整体外观主要由其吸收和散射的特性组合决定。同样，非金属与光的交互分为反射和折射两部分。而折射按介质类型的散射和吸收特性，分为多类：
- **反射（Reflection）**。非金属的镜面反射颜色为单通道单色，即R=G=B。
- **折射（Refraction）**。光从表面折射入非金属介质，则介质的整体外观由其散射和吸收的特性组合决定。不同的介质类型的散射和吸收特性不一：
- **均匀介质（Homogeneous Media）**。主要为透明介质，无折射率变化。不存在散射，光总以直线传播并且不会改变方向。存在吸收，光的强度会通过吸收减少，传播距离越远，吸收量越高。
- **非均匀介质（Nonhomogeneous Media）**。通常可以建模为具有嵌入散射粒子的均匀介质。具有折射率变化，分为几类。
- **混浊介质（Cloudy Media）**。混浊介质具有弱散射，散射方向略微随机化。根据组成的不同，具有复数折射率的物质区域引起吸收。
- **半透明介质（Translucent Media）**。半透明介质具有强散射，散射方向完全随机化。根据组成的不同，具有复数折射率的物质区域引起吸收。
- **不透明介质（Opaque Media）**。不透明介质和半透明介质一致。具有强散射，散射方向完全随机化。根据组成的不同，具有复数折射率的物质区域引起吸收。

### 4.3 菲涅尔效应

简单来说即为：

​		反射与散射的比率与观测视角有关，且观察入射角越接近于0反射比率越高

需要注意：**我们在宏观层面看到的菲涅尔效应实际上是微观层面微平面菲涅尔效应的平均值。**

- 绝对光滑的材质，在观察入射角为90°时，反射率为100%，观察入射角为0°时，反射率为0%

- 粗糙表面，观察入射角为90时，反射率小于100%，观察入射角为0°时，反射率大于100%

  

4.3.1 $F_0$

$F_0$指的是入射角为0时反射率，一般物体都是粗糙表面，所以$F_0 >0$

![img](https://pic1.zhimg.com/80/v2-86bafcaee3c8d4d8248476ad40077214_720w.jpg)

物理上，$F_0$是可以计算的，如下：
$$
F_0 = (\frac{n_1-n_2}{n_1+n_2})^2
$$
$n_1,n_2$分别是两种介质的折射率，如果假设其中一个为空气，则有
$$
F_0 = (\frac{n-1}{n+1})^2
$$

# 5 迪士尼的BxDF

### 5.1 迪士尼的BRDF

迪士尼先经过对各种材质的实际物理结果进行观测，然后再通过观测结果，指定一套标准的工作流，从而降低PBR的美术流程繁杂度

##### 5.1.1 迪士尼BRDF可视化方案

- MERL 100 BRDF 材质库
- BRDF Explorer
- BRDF Image Slice切片

大体流程就是对**MERL材质库**进行**观察**，通过**BRDF Explorer** 和**BRDF Image Slice** 切片，对材质库进行分析，从而得到各种观察结论，从而研究出各种物体的**渲染模型**和**参数特性**

###### 5.1.1.1 Diffuse项观察结论

基本定义：

- 漫反射（Diffuse）表示折射（refracted）到表面，经过散射（scattered）和部分吸收（partially absorbed），最终重新出表面出射的光。
- 被着色的非金属材质的任意出射部分都可以视为漫反射。

观察结果：

- 通过观察得出，很少有材质的漫反射表现和Lambert反射模型相吻合。即需要更准确的漫反射模型。
- 通过观察得出掠射逆反射（grazing retroreflection）有明显的着色现象，即可以将掠射逆反射（grazing retroreflection）也看做一种漫反射现象。
- 粗糙度会对菲涅尔折射造成影响，而一般的漫反射模型如Lambert忽略了这种影响。

Lambert 的改进模型：

![img](https://pic4.zhimg.com/80/v2-eed03ec11181b94e396edc136b159ad3_720w.jpg)

- **Oren-Nayar模型（1995）**预测粗糙漫反射表面逆向反射的增加会使漫反射形状变平。然而，其逆向反射波峰不像测量数据那样强，并且粗糙测量的材质通常不显示漫反射的平坦化。
- **Hanrahan-Krueger模型（1993）**，源自次表面散射理论，也预测了漫反射形状的平坦化，但在边缘处没有足够强的峰值。与Oren-Nayar相比，该模型呈现出完美光滑的表面。上图中比较了Oren-Nayar、Hanrahan-Krueger和Lambert模型。



###### 5.1.1.2 Specular D项观察结论

- 微观分布函数D（θh）可以从测量材质的**逆反射（retroreflective）响应**观察得到。
- 绝大多数MERL材质都有镜面波瓣（specular lobes），且尾部比传统的镜面模型长得多。 即反射分布项需要更宽的尾部。
- GGX比其他分布具有更长的尾部，但仍然无法捕捉到铬金属（chrome）样本的闪亮亮点。

###### 5.1.1.3 Specular F项的观察结论

- 菲涅尔反射系数F(θd)表示了当**光和视图矢量分开时镜面反射的增加**。
- 光滑表面在切线入射时有将接近100％的镜面反射。
- 对于粗糙表面，无法实现100％的镜面反射，但反射率仍会将变得越来越高。
- 每种材质在掠射角附近都显示出一些反射率的增加。
- 掠射角入射附近的许多曲线的陡度已经大于菲涅尔效应的预测值。

###### 5.1.1.3 Specular G项的观察结论

- 几何项的影响可以间接地看作**其对方向反射率（directional albedo）的影响**
- 大多数材质的方向反射率（directional albedo）对于前70度是相对平坦的，并且切线入射处的反射率与表面粗糙度密切相关。
- 几何项的选择会对反射率产生影响，反过来又会对表面外观产生影响。
- 完全省略G项和1/cosθl cosθv项的模型，被称为“No G”模型，会导致在**掠射角处过暗**的响应。

###### 5.1.1.4 **布料(Fabric)** 材质观察结论

- 许多布料样本在掠射角处呈现出镜面反射的色调，并且还具有比具有十分粗糙的材质更强的菲涅尔波峰。
- 布料具有有色的掠射反射，可以理解为是其轮廓附近获取到材质颜色的透射纤维（transmissive fibers）造成的。
- 布料在掠射角处的额外光泽增加，超出了普通微平面模型的预测范围。

##### 5.1.2 迪士尼BRDF结论

迪士尼的BRDF提出之前PBR需要大量复杂的参数，虽然效果拔群，但美术流程非常繁杂，工作效率低下，从而和其他渲染方式相比没有明显优势。

迪士尼的理念是开发一种“原则性”的易用模型，而不是严格的物理模型。正因为这种艺术导向的易用性，能让美术同学用非常直观的少量参数，以及非常标准化的工作流，就能快速实现涉及大量不同材质的真实感的渲染工作。而这对于传统的着色模型来说，是不可能完成的任务。

迪士尼根据5.1.1的观察结论，推导总结出PBR渲染方式

---

迪士尼原则的BRDF**核心理念**如下：

1. 使用直观参数，而非晦涩物理参数
2. 尽可能少的参数
3. 参数范围0~1
4. 允许参数有意义时超过合理范围
5. 所有参数的组合尽可能合理

按照以上原则，得到如下参数

- **baseColor（基础色）**：表面颜色，通常由纹理贴图提供。
- **subsurface（次表面）**：使用次表面近似控制漫反射形状。
- **metallic（金属度）**：金属（0 =电介质，1=金属）。这是两种不同模型之间的线性混合。金属模型没有漫反射成分，并且还具有等于基础色的着色入射镜面反射。
- **specular（镜面反射强度）**：入射镜面反射量。用于取代折射率。
- **specularTint（镜面反射颜色）**：对美术控制的让步，用于对基础色（base color）的入射镜面反射进行颜色控制。掠射镜面反射仍然是非彩色的。
- **roughness（粗糙度）**：表面粗糙度，控制漫反射和镜面反射。
- **anisotropic（各向异性强度）**：各向异性程度。用于控制镜面反射高光的纵横比。（0 =各向同性，1 =最大各向异性）
- **sheen（光泽度）**：一种额外的掠射分量（grazing component），主要用于布料。
- **sheenTint（光泽颜色）**：对sheen（光泽度）的颜色控制。
- **clearcoat（清漆强度）**：有特殊用途的第二个镜面波瓣（specular lobe）。
- **clearcoatGloss（清漆光泽度）**：控制透明涂层光泽度，0 =“缎面（satin）”外观，1 =“光泽（gloss）”外观。

而从本质上而言，Disney Principled BRDF模型是金属和非金属的混合型模型，最终结果是基于金属度（metallice）在金属BRDF和非金属BRDF之间进行线性插值。

![img](https://pic3.zhimg.com/80/v2-879917eeb9c582dcd293b78cf39b8966_720w.jpg)

###### 5.1.2.1 漫反射项(Diffuse)

迪士尼在漫反射Diffuse上由于观测到边缘太暗，于是尝试添加菲涅尔因子。使得边缘变亮但整体变暗。
$$
f_d=\frac{baseColor}{\pi}(1+(F_{D90}-1)(1-\cos\theta_l)^5)(1+(F_{D90})(1-\cos\theta_v)^5)\\
F_{D90}=0.5+2roughness\cos^2\theta_d\\
\cos\theta_l=light\cdot normal\\
\cos\theta_v=view\cdot normal\\
\cos\theta_d=view\cdot half
$$
这里使用Schlick Fresnel近似，并修改**掠射逆反射**（grazing retroreflection response）以达到其特定值**由粗糙度值确定**，而不是简单为0。

###### 5.1.2.2 法线分布项(Soecular D)

有多种公式

Trowbridge-Reitz(TR)\GGX
$$
D_{TR}=c/(\alpha^2\cos^2\theta_h+sin^2\theta_h)^2
$$
Berry分布
$$
D_{Berry}=c/(\alpha^2\cos^2\theta_h+sin^2\theta_h)
$$
上述两种综合推广
$$
D_{GTR}=c/(\alpha^2\cos^2\theta_h+sin^2\theta_h)^\gamma
$$
![img](https://pic1.zhimg.com/80/v2-97153e46022402eae28c35cff2115560_720w.jpg)

迪士尼主波瓣用**GGX**次级波瓣用**Berry**（具体主波瓣次级波瓣啥意思不太懂）

###### 5.1.2.4 菲涅尔项 (Specular F)

迪士尼观察得到用近似的**Schlick Fresnel**已经ok
$$
F_{Schlick}=F_0+(1-F_0)(1-cos\theta_d)^5
$$

###### 5.1.2.5 几何项(Specualr G)

迪士尼主镜面波瓣用**Smith GGX 的G项**，并对粗糙度进行映射
$$
G_{GGX}(v)=\frac{2(n\cdot v)}{(n\cdot v)\sqrt{\alpha^2+(1-\alpha^2)(n\cdot v)}}\\
\alpha=(0.5+roughness/2)^2
$$
次级波瓣直接用GGX G项 固定粗糙度为0.25

##### 5.1.3 迪士尼分层材质(Layer Material)

分层材质即在两种或多种材质之间进行插值得到中间材质

### 5.2 迪士尼BSDF

BRDF是用metallic混合金属非金属的理念，

BSDF是用specTrans(镜面反射透明度)混合透明物体和非透明物体。

- 对于普通表面，Disney BSDF在Disney BRDF的基础上新增specTrans（镜面反射透明度）和scatterDistance（散射距离）两个参数，共12个。
- 对于薄表面（Thin-surface），Disney BSDF在Disney BRDF的基础上新增specTrans（镜面反射透明度）、scatterDistance（散射距离）和flatness（平坦度）三个参数，共13个。

### 5.3 迪士尼次表面散射

- **在Disney BRDF中加入次表面散射模型**。具体思路是首先将漫射波瓣重构为两部分：方向性的微表面效应（microsurface effect），主要为逆反射（retroreflection）；非方向性的次表面效应（subsurface effect），即Lambertian。然后，用散射模型（diffusion model）或体积散射模型（volumetric scattering model）替换漫反射波瓣中的Lambert部分。这样，便能保留微表面效应（microsurface effect），让散射模型在散射距离较小时收敛到与漫反射BRDF相同的结果。
- **提出基于两个指数项总和的次表面漫射（Subsurface diffusion）模拟模型。** 次表⾯漫射（Subsurface diffusion）。Disney通过蒙特卡洛模拟（Monte Carlo simulation），观察到对于典型的散射参数，包括单次散射的扩散剖面（diffusion profile），使用两个指数项的总和（a sum of two exponentials）便可以很好地进行模拟，且得到了比偶极子剖面（dipole diffusion）更好的渲染结果。如下图所示。
- **薄表面BSDF（Thin-surface BSDF）**。对于薄的半透明表⾯，Disney选择在单个着色点处模拟入射和出射散射事件，作为镜面反射和漫反射传输的组合，由specTrans和diffTrans参数控制，并用各向同性的波瓣近似薄表面漫反射传输。如下图所示。