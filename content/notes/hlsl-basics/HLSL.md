---
title: "HLSL 与 UE Shader 基础"
date: 2026-10-19
draft: false
description: "HLSL 与 UE Shader 基础"
url: "/notes/hlsl-basics/"
tags: ["HLSL", "Shader", "UE"]
math: true
ShowToc: true
---

# UE核心shader语言----HLSL

## 有关shader语言
### 1.什么是shader(着色器)语言
        . 这是一种用于GPU处理上的特殊语言
### 2.为什么存在shader语言
        . 首先简单了解一下渲染管线的基础流程:
        > CPU处理渲染对象并打包->GPU接收打包数据+数据转换->光栅化->片元处理(上色及抗锯齿)->输出合并
        . 再聊聊CPU和GPU
        > CPU->拥有少量性能极强的核心;适于处理强逻辑性问题.
        > GPU->拥有海量简单核心;适用于处理逻辑性弱但重复性强的问题.
        -----------------------------------------------------------
        那么,针对GPU处理阶段存在的大量顶点数据处理问题,对应的设计了"shader语言"去专门调用,管理
### 3. 常见的几种shader语言
        . HLSL:由微软研发,常用于研发DiectX----UE核心语言
        . GLSL:常用于OpenGL和Vulkan
        . MSL:苹果研发
### 4. shader的主要应用场景
        1. 基础实时渲染:
        > 这是shader的最本职工作,主要做顶点变换(模型3D坐标投影),光照计算,纹理映射
        2. 非写实渲染(例如常见的三渲二):
        > 可实现--光影二分,物体描边,风格化笔触......
        3. 后期效果处理
        4. 环境动态特效:
        > 例如:动态水面(波浪模拟),植被摆动,特效粒子
        5. 帮助部分模拟,计算:
        > 模拟粒子/布料碰撞,在CPU处理之前剔除优化;

---
    
## HLSL与C++
### 什么是HLSL
* #### HLSL是一种类C语言
        > 语法: HLSL支持C++中的struct,函数,循环与条件判断   
        > 执行: 在执行逻辑上,更像Verilog
    * #### HLSL的特点
        > 数据类型:HLSL对数学运算做了更深度的优化   
        ---向量:三种定义:float2(坐标),float3(颜色/法向),float4(RGBA/齐次坐标).  
        ---矩阵:float4x4->用于MVP变换   
        ---采样器:用于读取贴图数据类型  
        ---
        >语义:HLSL中,我们为了告诉GPU某个变量的用途,因此必须给变量添加标签   
        例子:
        ```HLSL
        float4 Position : SV_POSITION;//告诉GPU这是屏幕空间坐标
        float3 Normal   : NORMAL;     //这是法线
        ```
    ### HLSL函数库
    * #### 核心数学函数(类似C++)
        . abs(x)->绝对值;   
        . pow(x,y)->$x^y$***(注:x最好是非负数)***   
        . sqrt(x)->平方根   
        . exp(x)/log(x)->以e为底的指数/对数运算;    
        . floor(x)/ceil(x)->向下/向上取整;  
        . frac(x)->仅保留浮点数的小数部分;**(可制作动态雨滴以及流动传送门等)**  
        . fwidth(x)->计算x在祥林像素的变化率;**(可用于抗锯齿制作)**
    * #### 向量几何函数(重点)
        . dot(a,b)->点积    
        . cross(a,b)->叉乘  
        . length(v)->计算向量长度   
        . reflect(I,N)->计算反射向量    
        . distance(a,b)->计算两点间距   
        . mul(a,b)->矩阵乘法
    * #### 范围与插值函数(二分实现核心)
        . clamp(x,min,max)->限定x在[min,max]之内    
        . saturate(x)->**即clamp(x,0.0,1.0);**  
        . lerp(a,b,t)->线性插值;**公式为:$a+t \times (b-a)$**   
        . step(edge,x)->若$x>=edge$则返回1,否则返回0;   
        . smoothstep(edge0,edge1,x)->在两个边界之间平滑插值
    * #### 纹理采样函数(UE常用)
        . Texture.Sample(samplar,uv)->根据UV坐标在贴图上"采样"颜色
    ### HLSL基础框架
    * **三部分:**   
    1. 输入输出结构体
        ```cpp
        //从CPU传给顶点着色器的数据
        struct VS_Input {
            float4 Pos    : POSITION;    //顶点位置
            float2 UV     : TEXCOORD0;   //纹理坐标
            float3 Normal : NORMAL;      //法线
        };

        //从顶点着色器转给像素着色器的数据
        struct PS_Input {
            float4 Pos    : SV_POSITION;//系统坐标(必备)
            float2 UV     : TEXCOORD0;  //纹理坐标
            float3 Normal : NORMAL;     //法线
        };
        ```
        * 解析:冒号后的大写字母,就是给GPU的标签,告诉GPU这个变量应该在GPU的哪块硬件槽位进行操作.
    2. 常量缓冲区
        ```cpp
        cbuffer SceneData : register(b0){
            float4x4 WorldViewProj; //MVP矩阵
            float3 LightDir;        //光照方向
        };
        ```
        * 解析:用于存放"全局变量",其中存在的变量对于这一批次处理的所有像素都是一致的;
    3. 核心函数框架(分为**Vertex Shader(VS)**和**Pixel Shader(PS)**)    
        * 顶点着色器(VS):处理空间变换.计算3D模型的顶点应该位于屏幕的哪个位置
        ```cpp
        PS_Input VS_Main(VS_Input input){
            PS_Input output;

            //将顶点坐标与MVP矩阵相乘,实现空间变换
            output.pos=mul(input.pos,WorldViewProj);

            //透传其他数据
            output.UV=input.UV;
            output,Normal=input.Normal;

            return output;
        }
        ```
        ---
        * 像素着色器(PS):计算最终输出颜色.也就是渲染管线流程中常说的"片元处理"
        ```cpp
        float4 PS_Main(PS_Input) : SV_TARGET {
            //基础二次元明暗计算
            float diffuse=saturate(dot(input.Normal,LightDir));

            //使用step函数创建明暗二分
            float cal=step(0.5,diffuse);//避免使用if else判断

            return float4(cel,cel,cel,1.0);//输出RGBA颜色
        }
        ```

    4. 框架之外:    
        * 为什么要尽量避免使用**if else**,转而使用**clamp**或**step**:
            >1. GPU的核心工作模式是:SIMT---单指令多线程 
            >   * GPU会让同一组核心在同一时间里执行相同的指令   
            因此,如果用if else判断,会让一些通过if判断的成员在准备执行else语句时,必须等待其他成员完成if判断,这样就丢失了GPU可以同时处理大量数据的优势,增加耗时
            > 2. 而相对应的**数学函数**不涉及跳转指令,无分支执行,并且,GPU在硬件层面通常对数学函数有着相应的优化     
        ***因此,在HLSL编写过程中,要尽量避免使用if else等分支判断语句***  
    ### HLSL完整框架示例
    ```cpp
    //1.顶点输入
    struct VS_Input{
        float4 Pos : POSITION; //顶点在模型空间的位置
        float2 UV  : TEXCOORD0;//纹理坐标
        float3 Normal : NORMAL;//法线
    };

    struct PS_Input{
        float4 Pos : SV_POSITION; //屏幕空间坐标
        float2 UV  : TEXCOOROD1;  //向像素阶段传递UV
        float3 WorldNormal : TEXCOORD0; //转换到世界空间的法线
    };

    // 固有常量
    cbuffer ConstantData : register(b0) {
        float4x4 WorldViewProj;  //MVP矩阵;
        float3   LightDir     ;  //光照方向(单位向量)
        float3   BaseColor    ;  //物体固有色
        float3   Shadowcolor  ;  //阴影颜色
    };

    //着色器函数

    //顶点着色器
    PS_Input VS_Main(VS_Input input){
        PS_Input output;

        output.Pos=mul(input.Pos,WorldViewProj);

        output.WorldNormal=input.Normal;

        output.UV=input.UV;
        
        return output;
    }

    //像素着色器
    float4 PS_Main(PS_Input input) : SV_TARGET{
        //1.归一化法线
        float3 N=normalize(input.WorldNormal);
        float3 L=normalize(LightDir);

        //2.计算光照强度(法线与光照方向点积)
        //点积结果范围为[-1,1],用saturate限制在[0,1];
        float diff=saturate(dot(N,L));

        //3.用step计算二分光影
        float isLit=step(0.5,diff);

        //4. 使用lerp进行颜色混合
        //当isLit=0时,取ShadowColor,当isLit=1时,取BaseColor;
        float3 finalRGB=lerp(ShadowColor,BaseColor,isLit);

        //输出最终RGBA
        return float4(finalRGB,1.0);
    }
    ```
    ### HLSL基础语义:
    1. 顶点着色器输入语义:
        >* **POSITION:** 顶点的本地坐标,模型的基础形状数据;
        >* **NORMAL:** 顶点的法线方向;
        >* **TANGENT:** 切线,常用于法线贴图;
        >* **COLORn** 顶点颜色;
        ---
    2. 系统数值语义:(通常以 ***SV_*** 开头,它们是由 **GPU渲染管线硬件直接使用或生成的** ,具有强制性的功能意义)
        >* **SV_POSITION:** *1.在VS输出中*:表示顶点在经过投影变换后的 ***屏幕空间剪裁空间坐标*** 
        >2. *在PS输出中* :表示当前像素在屏幕上的具体位置;
        >* **SV_Target:**  用于像素着色器的输出
        >* **SV_VertexID** 当前顶点的索引数字
        >* **SV_InstanceID** 实例化渲染时的ID(比如在渲染草地时,每颗不同ID的草,可以控制它们以不同的频率摆动)
        ---
    3. 像素着色器输入语义:(数据在顶点着色器处理完毕后,通常会经过 ***光栅化插值*** 后,传给像素着色器)
        >* **TEXCOORDn:** 在像素阶段这不一定时纹理坐标,可以是一个通用容器;(比如将世界空间法线从顶点传给像素)
        >* **VFACE(或SV_IsFrontFace):** 告知正面和背面;
        ---

## 示例展示
### 1.做一个简单的动态网格圆

![](<image-2.png>)
### 图中涉及代码
#### 1. 如何画一个中心坐标为pos,大小可控的圆
```glsl
float d=length(uv-pos);
return step(d,r);
```
输入变量:**uv,pos,r**;

#### 2. 进阶,如何让圆进行动态缩放
* 了解节点**TimeSine**;
    > Time:读取系统时间,随时间变化递增赋值  
    > Sine:三角函数sin();   
* 实现HLSL代码
```glsl
float d=length(uv-pos);
float r=abs(sin(t));
return step(d,r);
```

### 3. 加入网格化操作
* 关键函数:**frac**->读取浮点数的小数部分
* 代码实现
```glsl
//网格化uv
uv=frac(uv*n);
//定圆
float d=length(uv-pos);
float r=abs(sin(t));
float mask=step(d,r);
float3 color=float3(0.5,0.7,1.0);
//输出
return float4(color*mask,1.0);
```

### 2.制作一面砖墙

![](<image-4.png>)

### shader节点代码设计
```glsl

//1.制作一个长宽可控的长方形网格
float2 tmp=frac(uv*grid);
float2 tex=frac(tmp+offtset);

//2.将网格内的uv坐标值限定在[0,0.5]之内
float2 centerTex=abs(tex,0.5);

//3. 划出内嵌长方形范围
float2 mk2D=step(centerTex,dim*0.5);

//4. 真值判断(整合x和y方向读取的边框条,只有x,y都为1才显示1)
float mask=mk2D.x*mk2D.y;

//赋值颜色
float3 color=lerp(colorBehind,colorFront,mask);

//输出
return(color);

```

#### 代码详解
1. 可控网格输出:分别操作frac内uv的x,y赋值即可;
2. **限定坐标在[0,0.5]**:实际是控制在[-0.5,0.5],为了设计一个坐标原点在中心,方便后续计算(类推可设计各种形状);
3. 划出内嵌长方形范围:输入一个dim向量用来规定边长距离中心点多少范围的地方赋值1输出;
4. **真值判断**:上述操作后,x,y方向上都会判定一次,哪些区域在限定范围内,找出在两个方向上都满足要求的地方(均为1),就是最终输出的地方;
5. **赋值颜色**:活用 ***lerp(A,B,alpha)*** 线性插值函数($A \times (1-alpha)+B \times alpha$),当mask为1时,输出colorFront,当mask为0时,输出colorBehind;

#### 实操途中问题
1. **有关相关节点输出数据类型**:在UE中,白色连接点对应 **vector数据** ,红色接点对应 **x坐标(R)** ,绿色为 **y坐标(G)** ,蓝色为 **z坐标(B)** 
2. ***注意自定义节点的输出类型***
3. step(a,b)的判定条件为 **a<=b** 时输出1;
4. 出错时对照一下变量名字有没有打对;


## 渲染时常用的模型

### 兰伯特光照模型(漫反射)
1. **基础思想** :将光线方向与物体法线方向的点积作为光照强度的表现值
2. **方程** :
    $$I_{diffuse}=I_{light} \times k_{d} \times (N \cdot L) $$
3. **优缺点** :
    * 优点:效率快,简单易懂
    * 缺点:暗部全黑,没考虑二次反射之类的情况;
4. 进阶: **半兰伯特模型**
    $$ I_{diffuse}=((N \cdot L) \cdot 0.5+0.5) $$
    将强度值映射在了[0,1]之内,使暗部全黑的空间会有一些微弱的光照

### Kajiya-Kay(KK)模型(各向异性高光)(写实化)
1. **基础思想** :对于头发等由无数个细长圆柱体构成的事物,我们可以通过 **光线和物体的切线** 来确定此处的光照强度;
2. **方程** : $$ I_{specular} = k_s \cdot (sin(T,H))^n =k_s \cdot (1-(T \cdot H)^2)^{n/2}  $$
**T** :头发的切线方向;   
**H** :半角方向(也就是光线方向和视线方向的中间向量)->$normalize(L+V)$;
3. **优缺点**:
    * 优点:更精确的物理效果;可实现高光随光源与视线改变而动态变化
    * 缺点:开销更大;对物体切线方向数据的准确性需求高

### MatCap光照模型(各向异性高光)(风格化)
1. **基础思想** :将复杂的光照效果预烘焙进一个球体贴图中,然后根据物体法线信息,投射进纹理球中直接读取颜色信息
2. **计算** :

```glsl
vec3 normalVS=TransformWorldToViewNormal(input.normalWS,true);//将模型的法线信息转换到视图空间

vec2 matcupUV=(normalVS.xy*0.5+0.5);//将法线坐标映射到uv坐标

vec3 matcup_color=texture2D(MatCup_Texture,matcupUV);//读取颜色信息

```

3. **优缺点** :
    * 优点:开销低;风格统一;简单
    * 缺点:缺乏物理准确性;反射效果与环境变化无关

### 轮廓线模型(描边)
#### 一.背面挤出法:
1. **基础思想** :
    * ***翻转法线*** 
    * ***挤出偏移*** :将物体的顶点沿着法线方向移动;
    * ***着色*** : 将偏移出的"外壳"部分上色,实现描边效果
2. **方程** :$$ V_{new}=V_{pos}+N \cdot Thickness$$
**Thickness**通常由顶点颜色或贴图来定义;
3. **优缺点** :
    * 优点:
        * 线条粗细受透视影响
        * 开销固定
        * 可通过 **Vertex Color**具体控制每根线的粗细
    * 缺点:
        * 模型面数翻倍
        * 在法线断开处容易线条断裂

#### 二.后处理法
1. **基础思想** :
    * 在屏幕空间进行全屏扫描,检测三个部分 **深度**, **法线** , **颜色**,在三者变化剧烈,夹角过大的地方画线;
2. **实现**:在UE的 **Post Material** 中使用 **SceneDepth** 和 **WorldNormal** 节点,配合卷积核进行采样;
3. **优缺点** :
    * 优点:
        * 线条宽度在屏幕上是恒定的;
        * 无需修改模型,适合复杂的场景;
    * 缺点:
        * 模型内部线条难以区分;
        * 远处的物体线条易挤在一起,增加抗锯齿开销;

#### 三.进阶改进方案:平滑法线
1. **问题分析**:对于各种模型,若直接用建模时的法线,在硬表面转角处都会存在 **线条断裂** 现象
2. **解决方案** :
    * **法线平移**:在建模软件中,将轮廓线专用的法线进行平滑处理,或直接存入 **顶点色** ;
    * 在渲染时,不参考几何法线,而是参考这组平滑后的法线.
    * **结果** :即便是直角,也能渲染出连续均匀的轮廓线;

## 认识常用重要节点

### 一. Texture Sample(特效贴图)


![](<image-6.png>)


 * **各接点含义** :
    * **UVs**:规定贴图附着在模型的哪个地方(默认输入TexCoord[0]:默认范围值为[0,1]渐变的UV坐标);
    * **Apply View MipBias** :控制Mipmap(纹理分级细化)的偏移(用的少)
### 二.自定义节点
#### 1. M_Mask_Adjust(调节Mask的Smooth和Offset)

![](<image-3.png>)

* Smooth(边缘锐利度):数值越大越锐,过渡越快;
* Offset(偏移):遮罩偏移;
#### 2. M_Mask(混合颜色)

![](<image-5.png>)

* BaseColor:混合颜色1;
* Color:混合颜色2;
* Mode:混合模式,0为用Color替换BaseColor;1为正片叠底;
* Mask:遮罩;
### 三.向量计算节点
#### 1. TransformVector(切向空间/世界空间转换);
#### 2. FlattenNormal(向量强度计算(赋值越接近1越小));
#### 3. BlendAngelCorrectsNormals(向量叠加);

### 四.常用计算节点
1. Multiply:乘法;
2. dot:点积;
3. Divide:除法;
4. step(x,y):y大于x时输出1,否则输出0;
5. normalize:求单位向量;
6. lerp(A,B,Alpha):线性插值(A为Alpha=0时,B为Alpha=1时);
7. 1-x;
8. Mask(R G):遮罩通道,仅输出R,G(可更改通道)值;
---
# 实操运用:

## 一.关于角色面部SDF;

### 什么是SDF
    * 对于三渲二,我们一般不通过实时光影计算来赋予面部具体的阴影形状,而是通过 ***SDF贴图*** 来赋予具体的阴影形状细节;
    * 因此,所谓 **SDF阴影贴图** 就是存放了面部在何种光照下会产生阴影的判定数值

### 如何绘制一张SDF贴图
    * 通常是通过 **3D软件** 烘培制作的;

        1. **光照扫描** : 让灯光在水平面上从0°旋转到180°;
        2. **黑白记录** : 每转1°,就记录一张纯黑白的阴影图;
        3. **叠合** :将这几张图累加起来,求取平均值.
        4. **结果** : 输出的贴图中,白色区域代表接受光照时间最长的地方;

### 如何使用SDF贴图
    * 核心: $$ ShadowMask=step(LightAngle,SDFValue) $$
        * **LightAngle** : 灯光相对于脸部的夹角;
        * **SDFValue** : SDF贴图中的灰度数值;

### UE渲染具体实现步骤

#### 整体节点连接

![](<image-8.png>)

#### 1.目标分析:
* SDF贴图是记录了脸部各处出现阴影的光影阈值的贴图
* 我们需要计算在特定光照下,脸部各处的阴影强度值,划分出小于SDF和大于SDF的两块区域
* 因为SDF贴图一般只记录单侧光照的情况,所以我们还需要增加一个判断光照左右方向的模块,若是左侧光则对贴图进行U坐标镜像翻转;

#### 2.计算板块拆分:
* **计算LdotR:**$$LdotR=dot(L,R)$$
* **计算LdotF:**$$LdotF=dot(L,F)$$
* **判断左右:** LdotR为正在右,为负在左
* **计算光线夹角:** $$arctan2(LdotR,LdotF)$$
* **将角度映射到[0,1]:** 取绝对值;
* **判断阴影区** SmoothStep/Step(前者能修改明暗交界线的平滑度);

---
#### 效果展示
![](<image-9.png>)
---

## 二.有关实现刘海阴影
### 核心思想:
    * 基础模型对阴影部分做了额外的建模面
    * 可以尝试通过深度判断,判断该模型底层是否为皮肤,从而控制其不透明度;
### 材质节点连接:

![](<Pasted image 20261019120631.png>)

---
### HLSL代码实现:
```cpp
// UV:ScreenPOsition

//1.获取底层颜色并转HSV
float3 sceneRGB = SceneTextureLookup(UV,14,false).rgb;
float3 sceneHSV = RGBToHSV(sceneRGB);

//2.HSV色调偏移
float3 adjustedHSV=sceneHSV*BackHSV;

//3. 转RGB并调色
float3 adjustedRGB=HSVToRGB(adjustedHSV);
float3 finalCOlor =adjustedRGB*Color;

//4. 获取深度并计算遮罩
float sceneDepth=CalcSceneDepth(UV);
float depthMask=1.0-saturate(sceneDepth/Depth);

//输出
return float4(finalColor,depthMask);
```
### 实现效果

![](<Pasted image 20261019120703.png>)






















