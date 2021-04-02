[toc]



## 前言

概述

- 第1章 渲染流水线
  - 渲染流水线各阶段操作
  - 顶点处理阶段
    - 顶点组织方式、坐标系确认方式
    - 模型空间->世界空间->观察空间(相机空间)->剪裁空间
  - 光栅化阶段
  - 片元处理阶段、输出合并阶段
- 第2章 物理学阐述
  - PBR前置知识
  - 计算机对颜色进行颜色建模
  - 颜色空间基础
  - 伽马校正
- 第3章 外观着色器 

  - 外观着色器与顶点/片元着色器关系
  - 着色器多样体
  - Unity跨平台编译机制
- 第4章 UnityCG.cginc UnityShader Variables.cginc

  - 通用工具函数UnityCG.cginc
  - 着色器变量UnityShaderVariables.cginc
- 第5章 GPU多实例化技术

  - UnityInstancing.cginc
- 第6章 前向渲染和延迟渲染基本原理
- 第7章~第9章 全局光照和阴影计算原理

  - 球谐函数

  - 光照阴影相关宏UnityShadow Library.cginc，AutoLight.cginc
- 第10章 若干简单光照模型
  - Cook-Torrance
- 第11章 Standard.shader
  - UnityStandardInput.cginc, UnityStandard Utils.cginc
- 第12章 编程着色器编程实战案例

## 第1章 实时3D渲染流水线

### 1.1 概述

- 渲染流水线阶段：

  顶点处理->光栅化->片元处理->输出合并

  ![](D:\Studio\Notes\LearningNotes\PictureNotes\RenderingPipeLine.png)

### 1.2 顶点处理阶段

​		计算机图形学中，采用多边形网络是高效组织顶点数据的方式

#### 1.2.1 顶点组织方式

 - 顺序读取方式

   {{0, 1, 1},  {0, 0, 0}, {1, 0, 1}, {1, 0, 0}, {0, 0, 0}}

   表示两个三角形，但是会有重复的顶点，从而出现重复

 - 顶点+顶点索引方式

   现在常用的方式，不赘述

   ```
   struct vertexData{
       Vector3 vertexes[];
       Vector3 triangles[];
   }
   ```

#### 1.2.2 坐标系、顶点法线确定方式

- 笛卡尔坐标系

  Unity、UE4都是左手坐标系

  OpenGL右手坐标系、DirectX左手坐标系

- 三角形的法线确认方式

  假设缓冲区顶点排序为 $<p_1,p_2,p_3>$

  有如下公式
  $$
  n=\frac{v_{12}\times v_{13}}{||v_{12}||v_{13}||}
  $$

  ```
  Vector3 normal[triangles.length];
  for(int i = 0; i < triangles.length; i++)
  {
     	Vector3 n12 = vertexes[triangles[i].x] - 		
     				  vertexes[triangles[i].y];
     	Vector3 n13 = vertexes[triangles[i].x] - 		
     				  vertexes[triangles[i].z];
     	normal[i] = normalize(cross(n12,n13));
  }
  ```

  - 注意：如果仅交换顶点排序方式，或仅交换坐标系左右手方式，都会导致法线结果完全相反。
  
- 顶点法线确认方式

  一般由面法线计算得到顶点法线
  $$
  n=\frac{\sum_{i=1}^n n_i}{||\sum_{i=1}^n n_i||}
  $$

#### 1.2.3 模型空间到世界空间

1.2.2中确认的顶点数据vertexes、triangles、normals都在模型空间中

1. 仿射变换、其次坐标

   缩放Scale：
   $$
   \left[\begin{matrix}
   scale_x&0&0\\
   0&scale_y&0\\
   0&0&scale_z
   \end{matrix}\right]\tag{3}
   $$
   

   旋转Rotation：

   围绕x旋转θ
   $$
   \left[\begin{matrix}
   1&0&0\\
   0&\cos\theta&-\sin\theta\\
   0&\sin\theta&\cos\theta
   \end{matrix}\right]\tag{3}
   $$
   围绕y旋转θ
   $$
   \left[\begin{matrix}
   \cos\theta&0&-\sin\theta\\
   0&1&0\\
   \sin\theta&0&\cos\theta
   \end{matrix}\right]\tag{3}
   $$
   围绕z旋转θ
   $$
   \left[\begin{matrix}
   \cos\theta&-\sin\theta&0\\
   \sin\theta&\cos\theta&0\\
   0&0&1
   \end{matrix}\right]\tag{3}
   $$
   

   平移Position(引入四元数)：
   $$
   \left[\begin{matrix}
   1&0&0&t_x\\
   0&1&0&t_y\\
   0&0&1&t_z\\
   0&0&0&1
   \end{matrix}\right]\tag{4}
   $$

   - 其次向量(x, y, z, w)对应笛卡尔坐标系$(\frac{x}{w},\frac{y}{w},\frac{z}{w})$
   - 如果w=0则(x, y, z, 0)表示方向

2. 模型空间转换到世界空间的矩阵

   综合Scale、Rotation、Position的四元化矩阵

   **先scale、再Rotation、再translate**

   坐标按照列坐标处理，故对坐标左乘矩阵

   $M_{t}M_{r}M_{s}P$

   则综合变换矩阵为

   $M_{model} = M_{t}M_{r}M_{s}$

3. 法线变换

    对法线进行处理时，需要左乘$(M^{-1})^T$

4. UnityShaderLab中调用

   ```Cg
   float4 vInWorld = mul(unity_ObjectToWorld,vInModel);
   ```

   

#### 1.2.4 世界空间到观察空间

1. 定义观察空间

   摄像机由Eye、LootAt、Up定义

   - Eye:世界坐标系下摄像机位置
   - LookAt:世界坐标系下观察方向向量
   - Up:摄像机朝上的方向向量

   原点Eye位置，坐标轴{**u,v,n**}

   **n** = normalize(LookAt)

   **u** = normalize(Up)

2. 观察矩阵推导

   先计算将摄像机

   - Eye移动到世界坐标(0,0,0)
   - 坐标轴和原世界坐标系坐标轴重合

   对应的转换矩阵

   然后求该矩阵逆矩阵即可
   $$
   M_t =\left[\begin{matrix}
   1&0&0&-Eye_x\\
   0&1&0&-Eye_y\\
   0&0&1&-Eye_z\\
   0&0&0&1
   \end{matrix}\right]\tag{1}
   $$

   $$
   M_r =\left[\begin{matrix}
   u_x&u_y&u_z&0\\
   v_x&v_y&vz&0\\
   n_x&n_y&n_z&0\\
   0&0&0&1
   \end{matrix}\right]\tag{2}
   $$

   结合两个矩阵:
   $$
   M_rM_t =M_r =
   \left[\begin{matrix}
   u_x&u_y&u_z&-Eye_x\cdot u\\
   v_x&v_y&vz&-Eye_y\cdot v\\
   n_x&n_y&n_z&-Eye_z\cdot n\\
   0&0&0&1
   \end{matrix}\right]\tag{4}
   $$
   Unity3D中**世界坐标系**为**左手坐标系**，**观察坐标系**是**右手坐标系**

   故z轴需要取负，最终矩阵为
   $$
   M_{view} =
   \left[\begin{matrix}
   u_x&u_y&u_z&-Eye_x\cdot u\\
   v_x&v_y&vz&-Eye_y\cdot v\\
   -n_x&-n_y&-n_z&-Eye_z\cdot n\\
   0&0&0&1
   \end{matrix}\right]\tag{4}
   $$
   
3. UnityShaderLab中调用

   ```
   float4 vInViewSpace = mul(unity_MatrixV,vInWorldSpace);
   ```

   

#### 1.2.5 观察空间到裁剪空间

1. 视截体

   即一个正棱台(regular prismoid),由以下4个参数定义：

   - fovY (field of view)定义垂直方向视野区域

   - aspect 底面宽度和高度

   - n (near) 近截面

   - f (far)远截面

     ![无标题](D:\Studio\Notes\LearningNotes\PictureNotes\无标题.png)

     定义视角截体之后，则可以进行剔除工作，只留下在视截体内部的数据

2. 投影矩阵推导

   将视截体变为正方形，同时在其内部的物体随之发生变化，该变换对应的矩阵称为投影矩阵

   ![无标题66](D:\Studio\Notes\LearningNotes\PictureNotes\无标题66.png)
   $$
   M_{projection}  = 
   \left[\begin{matrix}
   \frac{ctan(\frac{fovY}{2})}{aspect}&0&0&0\\
   0&ctan(\frac{fovY}{2})&0&0\\
   0&0&-\frac{f+n}{f-n}&-\frac{2nf}{f-n}\\
   0&0&-1&0\end{matrix}\right]\tag{4}
   $$
   z’取值范围

   右手坐标系：Direct3D [-1,0] ；OpenGL[1,-1]

   左手坐标系：Direct3D [0,1] ；OpenGL[-1,1]

   四维其次坐标降维称为三维笛卡尔坐标，称为透视除法，在光栅化阶段进行

3. UnityShaderLab中调用 

   ```
   float4 vInClipSpace = UnityViewToClipPos(
   					  float3 vInViewPos.x,vInViewPos.y,vInViewPos.z);
   ```

   

### 1.3 光栅化阶段

顶点处理完之后的顶点，硬件先将顶点信息作为顶点输入流(vertices input stream)，把顶点组装为**图元(primitive)**:图元包括线段和三角形。图元进行进一步处理确定其在二维屏幕上的绘制方式，光栅化为一些列**片元fragment**集合。将顶点数据赋值给光栅化的片元称为**图元组装**和**光栅化**。

光栅化包括以下子过程：

- 裁剪操作
- 透视除法
- 背面剔除
- 视口变换
- 扫描转换

主流流水线中这些过程都是不可编程的，在硬件电路中实现

#### 1.3.1裁剪操作



#### 1.3.2 透视除法



#### 1.3.3 背面剔除



#### 1.3.4 视口变换



#### 1.3.5 扫描变换

(<u>顶点插值为片元</u>)

定义**图元**覆盖屏幕空间像素位置，对顶点属性插值计算从而定义**片元属性**

### 1.4 片元处理 输出合并阶段

#### 1.4.1纹理操作

将图像渲染在待渲染物体表面

1.纹理映射坐标

- 纹理中最小的单元称为**纹素(Texel)**，区分于位于有效颜色缓冲区中的像素**图素(image)**,
- 每个**纹素**包含一个纹素阵列的**横纵索引**(u, v)

2.纹理阵列索引

#### 1.4.2 输出合并中的深度值操作

#### 1.4.3 输出合并中的Alpha值操作

#### 1.4.4 UnityShaderLab Alpha混合指令、深度测试指令

## 第2章 辐射度、光度和色度学

### 2.1 辐射度基本理论

#### 2.1.1 立体角

#### 2.1.2 点光源、辐射强度、辐射亮度

#### 2.1.3 辐射出射度和辐射入射度



### 2.2 光度学基本理论



### 2.3 色度学基本理论

#### 2.3.1 颜色定义

#### 2.3.2 颜色数字化,CIE1931-RGB颜色模型

#### 2.3.3 CIE1931-XYZ颜色模型

#### 2.3.4 CIE1931-Yxy颜色模型

2.4 伽马校正和sRGB空间