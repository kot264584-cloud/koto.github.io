---
title: "光照模型与球谐函数"
date: 2026-10-17
draft: false
description: "光照模型与球谐函数"
tags: ["光照", "Shader", "UE"]
math: true
ShowToc: true
---

### 1.核心公式
$$ Diffuse=saturate((N \cdot L+Wrap))/(1+wrap)$$

### 2.定义
* **目的:** 避免LamBert模型导致的暗面"死黑",同时令明暗交接处转折更通透;
* **基础思想:** 模拟光线透过明暗交界处,向暗面延伸的效果;

### 3.使用
* **核心公式代码**:
```cpp
//基础向量归一化
float3 N=normalize(Normal);
flaot3 L=normalize(LightDir);

//计算基础光照
float NdotL=dot(N,L);

//1.Wrap Lighting
float wrapNdotL = saturate((NdotL+Wrap))/(1+Wrap);
```
* **实现次表面效果**:
```cpp
//2.SSS衰减控制
float scatterFalloff = pow(wrapNdotL,ScatterPower);

//3.提取明暗交界线处的SSS遮罩
float baseLight = saturate(NdotL);
float scatterMask = saturate(scatterFalloff-baseLight);

//4.最终光照合成
float3 diffuseTerm = baseLight * LightColor;
float3 sssTerm = scatterMask*ScatterColor*LightColor;

return diffuseTerm+sssTerm;
```
* **加入环境光**
    * **球谐函数**
    1. **是什么**:定义球面坐标系的一组特殊函数,球面上的傅里叶级数;
    2. **用途**:将球面上任何连续分布的数据(如,球表面温度或光照强度)分解成一组基函数
    3. **用于渲染** :以较低的代价获取静态光环境下的大气光照;

```cpp
//球谐函数

float4 NormalVector = float4 (Normal,1);

float3 Intermediate0,Intermediate1,Intermediate2;
Intermediate0.x=dot(View.SkyIrradianceEnvironmentMap[0],NormalVector);
Intermediate0.y=dot(View.SkyIrradianceEnvironmentMap[1],NormalVector);
Intermediate0.z=dot(View.SkyIrradianceEnvironmentMap[2],NormalVector);
return Intermediate0;
```
* **获取平行光**
```cpp
return ResolvedView.DirectionalLightColor;
```
* **修改Wrap Lighting** 
```cpp
float3 N=normalize(Normal);
float3 L=normalize(LightDir);

float NdotL=dot(N,L);

float wrapNdotL = saturate((NdotL+Wrap))/(1+Wrap);

float scatterFalloff = pow(wrapNdotL,ScatterPower);

float baseLight = saturate(NdotL);
float scatterMask = saturate(scatterFalloff-baseLight);

float3 diffuseTerm = baseLight * LightColor;
float3 sssTerm = scatterMask*ScatterColor*LightColor;
float3 envColor=EnvironmentLight;

return diffuseTerm+sssTerm+envColor;
```
* ***输入:***

    * **Normal**:皮肤法线(世界空间);    
    * **LightDir**:平行光(世界空间);    
    * **Wrap**:常量;
    * **LightColor**:获取平行光步骤输出;
    * **ScatterColor**:次表面反射颜色'
    * **EnvironmentLight**:通过球谐函数得出的物体光照信息;

### 4.优缺点分析
* **优点**:
    * 纯指令计算,无额外贴图开销
    * 方便动态修改效果
* **缺点**:
    * 色彩单调;

# 二.LamBert+Ramp

### 1.核心公式:
$$ uv=(N \cdot L)*0.5+0.5$$
$$ Diffuse=tex2D(RampMap,float2(uv,0.5)).rgb$$
### 2.基础思想
使用预设好的不同光照强度下的颜色纹理图,替换在对应光照下的颜色;

### 3.使用
```cpp
//1. 计算N*L并映射到[0,1];
float uv=dot(N,L)*0.5+0.5;

//2.用光照强度结果读取Ramp图;
float3 RampColor=RampTex.SampleLevel(RampSampler,float2(uv,0.5),0).rgb;

return RampColor;
```
### 4.优缺点
* **优点:**
    * 色彩丰富,风格可控,风格拓展空间大
* **缺点:**
    * 额外的Ramp图资产开销

# 三.球谐函数

### 傅里叶级数:
* 是什么:**一个复杂波形可以拆解为任意正弦波和余弦波的叠加**;
* **思想:**
    * 例如一个棱角分明的*方波*,跟圆润的*正弦波*之前有很大的差异;
    * 但,傅里叶告诉我们,只要能找对一系列***频率不同***,***振幅不同***的正弦波,就能叠加出一个*方波*;
    * 叠加的正弦波越多,结果越接近目标值;

## 而球谐函数就是球面上的*傅里叶级数*

###  球谐函数大致可分为三阶

* **第一阶**:仅有一个基函数,代表一个球面的***平均亮度***,无论从哪种方向看都一样.
* **第二阶**:含三个基函数,沿***x,y,z***方向分布,捕捉了光照的*方向性*,告诉你哪边暗,哪边亮;
* **第三阶**:含更多基函数,它包含了更多的***高频光照细节***(例如高光,更深层的阴影等等);

### UE中怎么使用球谐函数

* **存放函数**:$$View.SkyIrradianceEnvironmentMap$$
    * 这是一个**float4**类型的数组(通常长度为9);
    * **SkyIrradianceEnvironmentMap[0]:** 平均亮度;
* **使用三阶球谐函数获取高频光照细节**:
1. 
```cpp
// 假设 Normal 是世界空间法线 (float3)
float4 SH[7];
for (int i = 0; i < 7; ++i) {
    SH[i] = View.SkyIrradianceEnvironmentMap[i];
}

float3 N = Normal;

// 1. 计算 L0 和 L1 (前 4 个系数)
// 这部分和你之前的逻辑类似，但这里我们拆解开看
float3 Irradiance = SH[0].xyz; // L0 (常数项)
Irradiance += SH[1].xyz * N.y; // L1_y
Irradiance += SH[2].xyz * N.z; // L1_z
Irradiance += SH[3].xyz * N.x; // L1_x

// 2. 计算 L2 (后 3 个系数，包含 5 个二次项分量)
// UE 使用了特殊的打包算法将 5 个 L2 分量塞进 3 个 float4 的 xyz 中
float4 N2 = float4(N.x * N.y, N.y * N.z, N.z * N.z, N.z * N.x);
float N3 = N.x * N.x - N.y * N.y;

Irradiance += SH[4].xyz * N2.x; // L2_xy
Irradiance += SH[5].xyz * N2.y; // L2_yz
Irradiance += SH[6].xyz * N2.z; // L2_z2
Irradiance += SH[4].xyz * N2.w; // 这里的打包逻辑可能随版本微调，通常建议使用内置宏
Irradiance += SH[5].xyz * N3;   // L2_x2-y2

return max(float3(0,0,0), Irradiance);
```
2. (更推荐,更方便的做法)
```cpp
// 1. 获取当前像素的 SH 向量（基于法线）
FThreeBandSHVector RGB_SH = GetSkySHDiffuse(Normal);
//Normal输入模型的世界空间法线信息


// 2. 将该向量与 View 里的 7 个系数进行卷积计算
return ComputeSHDiffuseLighting(RGB_SH, View.SkyIrradianceEnvironmentMap);
```