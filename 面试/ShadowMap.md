# ShadowMap

#### 像素取余

### Moving the Light in Texel-Sized Increments

阴影地图中常见的一种现象是闪烁边缘效应。随着摄像机的移动，沿着阴影边缘的像素会变亮和变暗。这在静止图像中是看不到的，但在实时中非常明显和分散注意力。图16突出了这个问题，图17展示了阴影边缘应该是什么样子的。

闪烁边缘错误是因为每次摄像机移动时都会重新计算光投影矩阵造成的。这会导致生成的阴影地图中出现细微差异。下列因素都会影响用于限定场景的矩阵的创建

- Size of the view frustum
- Orientation of the view frustum
- Location of the light
- Location of the camera

每次这个矩阵变化时，阴影边缘都可能会改变。

随着摄像机从左向右移动，阴影边界上的像素会进入和退出阴影。

随着摄像机从左向右移动，阴影边缘保持不变。

对于定向光源，解决这个问题的方法是将X和Y的最小/最大值（构成正交投影边界）四舍五入到像素大小的增量。这可以通过除法运算、向下取整操作和乘法来实现。

```
vLightCameraOrthographicMin /= vWorldUnitsPerTexel;
vLightCameraOrthographicMin = XMVectorFloor( vLightCameraOrthographicMin );
vLightCameraOrthographicMin *= vWorldUnitsPerTexel;
vLightCameraOrthographicMax /= vWorldUnitsPerTexel;
vLightCameraOrthographicMax = XMVectorFloor( vLightCameraOrthographicMax );
vLightCameraOrthographicMax *= vWorldUnitsPerTexel;
```

vWorldUnitsPerTexel值是通过将视锥体的边界取值，然后除以缓冲区大小来计算的。

```
        FLOAT fWorldUnitsPerTexel = fCascadeBound /
        (float)m_CopyOfCascadeConfig.m_iBufferSize;
        vWorldUnitsPerTexel = XMVectorSet( fWorldUnitsPe
```

将视锥体的最大尺寸限定在一个较大的范围内会导致正交投影的适配程度变得更宽松。

需要注意的是，在使用这种技术时，纹理的宽度和高度会比实际需要的大一个像素。这样做是为了防止阴影坐标超出阴影地图的索引范围。

## CascadedShadowMap

why?

基础的shadowMap由于是正交投影的关系，在shadowmap中的物体不管离相机远近，都占同样的精度，而相机由于是透视投影，物体有近大远小的效果，这样就近处的物体阴影精度不足，导致了阴影锯齿等等问题

what？

CSM会根据物体离摄像机的距离提供不同分辨率的shadowmap，将相机的视锥体分成若干个部分，每个部分提供独立的shadowmap，解决上述问题

how？



