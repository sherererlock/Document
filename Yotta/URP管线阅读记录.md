# URP管线阅读记录

## 类图

```mermaid
classDiagram
class UniversalRenderPipelineAsset{
	ScriptableRenderer[] m_Renderers
	ScriptableRendererData m_RendererData
	ScriptableRendererData[] m_RendererDataList
	bool m_RequireDepthTexture
	bool m_RequireOpaqueTexture
	bool m_SupportsHDR 
	
	float m_ShadowDistance
	
	 bool m_UseSRPBatcher
	 
	 RenderPipeline CreatePipeline()
	 void CreateRenderers()
}

UniversalRenderPipelineAsset *-- ScriptableRendererData

class ScriptableRendererData{
	abstract ScriptableRenderer Create()
	List~ScriptableRendererFeatur~ m_RendererFeatures
	bool m_UseNativeRenderPass
}

ScriptableRendererData *-- ScriptableRendererFeature
class ScriptableRendererFeature{
	void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
	void SetupRenderPasses(ScriptableRenderer renderer, in RenderingData renderingData)
}

ScriptableRendererData <|--UniversalRendererData
class UniversalRendererData{
	PostProcessData postProcessData
	LayerMask m_OpaqueLayerMask
	LayerMask m_TransparentLayerMask
	bool m_ShadowTransparentReceive
	RenderingMode m_RenderingMode
	DepthPrimingMode m_DepthPrimingMode
	CopyDepthMode m_CopyDepthMode
	override UniversalRenderer Create()
}

UniversalRenderPipelineAsset *-- ScritableRenderer
class ScritableRenderer{
	List~ScriptableRenderPass~ m_ActiveRenderPassQueue
	List~ScriptableRendererFeature~ m_RendererFeatures
	RTHandleRenderTargetIdentifierCompat m_CameraColorTarget
	RTHandleRenderTargetIdentifierCompat m_CameraDepthTarget
	RTHandleRenderTargetIdentifierCompat m_CameraResolveTarget
	RenderBufferStoreAction m_ActiveDepthStoreAction
	abstract void Setup()
	virtual void SetupLights()
	virtual void SetupCullingParameters()
	void SetPerCameraProperties()
	void Execute()
}

ScritableRenderer <|-- UniversalRenderer
class UniversalRenderer{
	DepthOnlyPass m_DepthPrepass;
    CopyDepthPass m_PrimedDepthCopyPass;
    DrawObjectsPass m_RenderOpaqueForwardOnlyPass;
    DrawObjectsPass m_RenderOpaqueForwardPass;
    DrawSkyboxPass m_DrawSkyboxPass;
    DrawObjectsPass m_RenderTransparentForwardPass;
    DrawScreenSpaceUIPass m_DrawOverlayUIPass;
    
    RTHandle m_ActiveCameraColorAttachment;
    RTHandle m_ActiveCameraDepthAttachment;
    
    ForwardLights m_ForwardLights;
    
    PostProcessPasses m_PostProcessPasses;
    
    override void Setup()
    override void SetupLights()
    override void SetupCullingParameters()
    RenderPassInputSummary GetRenderPassInputs(ref RenderingData renderingData)
	void CreateCameraRenderTarget()
}
```

```mermaid
classDiagram
class UniversalRenderPipeline{
	RenderGraph s_RenderGraph;
	UniversalRenderPipelineAsset pipelineAsset
	
	void Render(ScriptableRenderContext renderContext, List<Camera> cameras)
	void InitializeCameraData()
	void InitializeStackedCameraData()
	void InitializeAdditionalCameraData()
	void InitializeRenderingData()
	void InitializeShadowData()
	void InitializePostProcessingData()
	void InitializeLightData()
}

```

```mermaid
classDiagram
class UniversalAdditionalCameraData{
	CameraRenderType m_CameraType
	List<Camera> m_Cameras
}

class CameraData{
    Matrix4x4 m_ViewMatrix;
    Matrix4x4 m_ProjectionMatrix;
    Matrix4x4 m_JitterMatrix;
    RenderTextureDescriptor cameraTargetDescriptor;
    int pixelWidth;
    int pixelHeight;
    float renderScale;
    bool requiresDepthTexture;
    float maxShadowDistance;
    LayerMask volumeLayerMask;
    ScriptableRenderer renderer;
    bool postProcessEnabled;
}

class LightData{
	int mainLightIndex;
	int additionalLightsCount;
	NativeArray<VisibleLight> visibleLights;
	bool reflectionProbeBoxProjection;
	bool supportsAdditionalLights;
}

class RenderingData{
	CommandBuffer commandBuffer;
	CullingResults cullResults;
	CameraData cameraData;
	LightData lightData;
	ShadowData shadowData;
	PostProcessingData postProcessingData;
	PerObjectData perObjectData;
	bool postProcessingEnabled;
}

RenderingData *-- CameraData
```

## 重要函数

```c#
UniversalRenderer.Setup(ScriptableRenderContext context, ref RenderingData renderingData)
```

1. 当camera的targetTexture格式是Depth时，渲染一张离线的深度图（不透明+半透明Pass）

2. 遍历所有的RenderFeature, 获取渲染过程需要用到的RT（Depth, Normal, Color, Motion, DepthPrePass）

3. 遍历所有的RenderFeature处理RenderLayer

4. 对depthTexture的处理

   需要DepthTexture的三种情况

   - camera或者管线设置了DepthTexture
   - Render Feature中需要用到DepthTexture
   - 强制开启了DepthPrimingMode（PreZ）
   - 后处理中需要用到DepthTexture

   需要PreZ Pass的几种情况

   - 后续渲染步骤中要用到DepthTexture，但GPU不支持拷贝pass
   - 强制开启pre z pass
   - Render Feature中需要用到PreZPass或者需要用的NormalTexture
   - 开启了DepthPriming

   设置CopyDepth Pass的执行时机

   - RenderData中的设置
   - 如果Render Feature中设置了更小的值，则使用Render Feature中的
   - 如果只是后处理需要这个值，那么直接在AfterRenderingTransparents之后进行Copy Depth Pass

   创建DepthTexture的情况

   - 需要DepthTexture但不需要PrePass（PrePass会自己创建DepthTexture）
   - 使用DepthPriming时

   创建DepthTexture

   - m_CameraDepthAttachment
   - 名字是_CameraDepthAttachment

5. 对ColorTexture的处理

   
