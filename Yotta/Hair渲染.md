# Hair渲染

## Pass

### HairShadingMarschnerPreDepth

#### State

```
Cull Off
ZTest LEqual
ZWrite On
ColorMask 0
```

#### Vertex

输出：ClipSpace的位置，UV，溶解效果需要worldPos

#### Fragment

采样MainMap决定AlphaTest通过与否

溶解

step函数，世界位置y分量+溶解宽度与溶解方向比大小，大的被溶解

### HairShadingMarschnerTransparent

#### State

```
Cull Off
ZTest LEqual
ZWrite Off
Blend SrcAlpha OneMinusSrcAlpha
```

#### Vertex

输出：TBN矩阵，

#### Fragment



### ShadowCaster

#### State

```
Cull Off
ZTest LEqual
ZWrite On
```

#### Vertex

计算：position和normal的世界位置转换，ShadowBias的计算

输出：世界坐标（溶解效果用）

#### Fragment

采样：mainMap的alpha值

AlphaTest：顶点颜色-0.5， Alpha-AlphaCutoff, 溶解

### CharacterShadowCaster

#### State

```
Cull Off
ZTest LEqual
ZWrite On
```

#### Vertex

计算：position和normal的世界位置转换，ShadowBias的计算

输出：世界坐标（溶解效果用），grapPos（屏幕位置）

#### Fragment

采样：mainMap的alpha值

AlphaTest：顶点颜色-0.5， Alpha-AlphaCutoff, 溶解，以及很多

### DeepOpacityPass

#### State

```
Cull Off
ZTest Off
ZWrite Off
ColorMask RGBA
Blend One One
```

#### Vertex



#### Fragment



### DepthOnly

#### State



#### Vertex



#### Fragment



### ShadowPassOpaque

#### State

```
Cull Off
ZTest LEqual
ZWrite On
```

#### Vertex

输出：TBN

#### Fragment

采样：MainMap, ShadowMap

Clip:

输出：R shadow, A alpha, GB view下的Flow方向 

### ShadowPassTransparent

#### State

```
Cull Off
ZTest LEqual
ZWrite Off
Blend SrcAlpha OneMinusSrcAlpha
```

#### Vertex

输出：TBN

#### Fragment

采样：MainMap, ShadowMap

Clip:

输出：R shadow, A alpha, GB view下的Flow方向 

### Mask

#### State



#### Vertex



#### Fragment

#### Blit For Flow

### 流程

1. shadowCasterPass渲染shadow
2. HairShadowPass 发丝走向的flow，采样shadow
3. BlurShadowPass 根据走向模糊shadow
4. HairPreDepth 
5. HairShadingMarschnerTransparent 颜色
