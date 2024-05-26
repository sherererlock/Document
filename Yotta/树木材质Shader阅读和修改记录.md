# 树木材质Shader阅读和修改记录

## 修改

1. Attributes加上DynamicLightMapUV
2. viewWs的处理
3. 对tangentTo World的处理
4. diffuse贴图采样了两次
5. normalizeUV算了两边

## 阅读记录

### Base风的处理

#### 第一部分：叶子的晃动offset

输入：时间，风的burst速度和power，风的power，顶点的世界坐标xz分量

函数：柏林噪声函数（更符合自然的噪声函数）

输出：一个偏移值

#### 第二部分：叶子离树干的offset

输入：顶点的rgba其中一个分量，_WindTrunkPosition，_WindTrunkContrast

函数：对比度调整函数

输出：离树干的偏移值

#### 第三部分：计算xz的偏移值

输入：叶子的偏移值，叶子离树干的偏移值

函数：sin、cos

输出：叶子的世界位置偏移

### 微风的处理

输入：微风速度，频率，时间，顶点颜色rgba分量，顶点的法线

函数：柏林噪声

输入：偏移值

更倾向向叶子顶点的法线方向移动

## 裁剪的处理

抖动值：根据当前Fragment在屏幕上的坐标算出一个抖动（拜尔抖动函数），8x8的像素块内有不同的抖动

视线与法线的夹角的余弦值：越大越不容易被裁剪

hiderPower， 越大越容易被裁剪

diffuse贴图的Alpha值，越大越容易被裁剪

## 颜色的处理

#### 渐变色和主要颜色的混合

根据法线y值，GradientPosition，GradientFalloff三者计算一个插值参数，在color与gradientColor之间插值。

Y值越大，越靠近Gradient Color

gradientPosition越大，越靠近Gradient Color

GradientFalloff，越靠近MainColor

#### 上述算法得到颜色与变化颜色的混合，使得树叶更加多变

根据世界位置的xz坐标采样noise贴图，得到噪声值，

使得不同位置的叶子呈现不同的颜色

#### 上述颜色与diffuse贴图的颜色相乘得到最终颜色

#### 叶子半透的模拟

直接光+环境光





