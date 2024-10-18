# URP中的Render Graph

## Resource

```mermaid
classDiagram
class IRenderGraphResource{
	virtual void CreatePooledGraphicsResource()
	virtual void CreatePooledGraphicsResource()
	virtual void CreateGraphicsResource()
	virtual void UpdateGraphicsResource()
	virtual void ReleasePooledGraphicsResource(int frameIndex)
}

IRenderGraphResource <|-- RenderGraphResource

class RenderGraphResource{
	DescType desc;
	ResType graphicsResource;
	RenderGraphResourcePool~ResType> m_Pool;
}

RenderGraphResource <|-- BufferResource
RenderGraphResource <|-- TextureResource
RenderGraphResource <|-- RayTracingAccelerationStructureResource

class TextureResource{
	override void CreateGraphicsResource()
}

class BufferResource{
	override void CreateGraphicsResource()
}

class ResourceHandle{
	
}


```

```mermaid
classDiagram
class IRenderGraphResourcePool{
	abstract void PurgeUnusedResources(int currentFrameIndex);
	abstract void Cleanup();
	abstract void CheckFrameAllocation(bool onException, int frameIndex);
}

IRenderGraphResourcePool <|--RenderGraphResourcePool

class RenderGraphResourcePool{
	List~int, Type~ m_FrameAllocatedResources
}

RenderGraphResourcePool<|-- TexturePool
RenderGraphResourcePool<|-- BufferPool

```

```mermaid
classDiagram
class RenderGraphResourceRegistry{
	RenderGraphResourcesData[] m_RenderGraphResources
	DynamicArray~RendererListResource> m_RendererListResources
	DynamicArray~RendererListLegacyResource> m_RendererListLegacyResources
	
	RTHandle GetTexture(int index)
	CoreRendererList GetRendererList(in RendererListHandle handle)
	GraphicsBuffer GetBuffer(in BufferHandle handle)
	RayTracingAccelerationStructure GetRayTracingAccelerationStructure(in RayTracingAccelerationStructureHandle handle)
	void BeginExecute(int currentFrameIndex)
	void EndExecute()
	
	TextureHandle ImportTexture(in RTHandle rt, in ImportResourceParams importParams, bool isBuiltin = false)
	TextureHandle ImportBackbuffer(RenderTargetIdentifier rt, in RenderTargetInfo info, in ImportResourceParams importParams)
	 BufferHandle ImportBuffer(GraphicsBuffer graphicsBuffer, bool forceRelease = false)
	 TextureHandle CreateTexture(in TextureDesc desc, int transientPassIndex = -1)
	 BufferHandle CreateBuffer(in BufferDesc desc, int transientPassIndex = -1)
}

RenderGraphResourceRegistry *-- RenderGraphResourcesData

class RenderGraphResourcesData{
	DynamicArray~IRenderGraphResource> resourceArray
	IRenderGraphResourcePool pool
	
	int AddNewRenderGraphResource~ResType>(out ResType outRes, bool pooledResource = true)
}

RenderGraphResourcesData *-- IRenderGraphResource
class IRenderGraphResource{
	virtual void CreatePooledGraphicsResource()
	virtual void CreatePooledGraphicsResource()
	virtual void CreateGraphicsResource()
	virtual void UpdateGraphicsResource()
	virtual void ReleasePooledGraphicsResource(int frameIndex)
}

IRenderGraphResource <|-- RenderGraphResource

class RenderGraphResource{
	DescType desc;
	ResType graphicsResource;
	RenderGraphResourcePool~ResType> m_Pool;
}

RenderGraphResource <|-- BufferResource
RenderGraphResource <|-- TextureResource
RenderGraphResource <|-- RayTracingAccelerationStructureResource

class TextureResource{
	override void CreateGraphicsResource()
}

class BufferResource{
	override void CreateGraphicsResource()
}

RenderGraph *-- RenderGraphResourceRegistry
   
```

```mermaid
classDiagram
class RenderGraphDefaultResources{
	TextureHandle blackTexture
	TextureHandle whiteTexture
	TextureHandle defaultShadowTexture
	
	void InitializeForRendering(RenderGraph renderGraph)
}

class RenderGraph{
	RenderGraphDefaultResources m_DefaultResources
}
RenderGraph *--RenderGraphDefaultResources
```

```mermaid
classDiagram

class RenderGraph{
	List~RenderGraphPass> m_RenderPasses
	RenderGraphResourceRegistry m_Resources;
	CompiledGraph m_CurrentCompiledGraph
}

RenderGraph *-- CompiledGraph
class CompiledGraph{
	DynamicArray~CompileResourceInfo>[] compiledResourcesInfos
	DynamicArray~CompiledPassInfo> compiledPassInfos
	
}

CompiledGraph *-- CompileResourceInfo
CompiledGraph *-- CompiledPassInfo

class CompileResourceInfo{
	List~int> producers
	List~int> consumers
    int refcount
    bool import
    bool reset
}

class CompiledPassInfo{
	string name
	int index
	List~int>[] resourceCreateList
	List~int>[] resourceReleaseList
	int refCount
	bool culled
	bool culledByRendererList
	bool hasSideEffect
}

```





------

### 编译相关

### Pass相关

```mermaid
classDiagram
class RenderGraphPass{
	string name
	int index
	bool allowPassCulling
	bool allowGlobalState
	TextureAccess depthAccess
	TextureAccess[] colorBufferAccess
	TextureAccess[] fragmentInputAccess
	RandomWriteResourceInfo[] randomAccessResource
	List<ResourceHandle>[] resourceReadLists
	List<ResourceHandle>[] resourceWriteLists
	List<ResourceHandle>[] transientResourceList
	List<RendererListHandle> usedRendererListList 
	
	void AddResourceWrite(in ResourceHandle res)
	void AddResourceRead(in ResourceHandle res)
	void AddTransientResource(in ResourceHandle res)
	void AllowPassCulling(bool value)
	void SetFragmentInputRaw(in TextureHandle resource, int index, AccessFlags accessFlags, int mipLevel, int depthSlice)
	
	void Execute()
	
}

RenderGraphPass <|-- BaseRenderGraphPass

class BaseRenderGraphPass{
	PassData data;
	void Initialize(int passIndex, PassData passData, string passName, RenderGraphPassType passType, ProfilingSampler sampler)
}

BaseRenderGraphPass <|-- RenderGraphPassData

class RenderGraphPassData{
	BaseRenderFunc~PassData, RenderGraphContext> renderFunc;
	RenderGraphContext c
}

BaseRenderGraphPass <|-- ComputeRenderGraphPass
BaseRenderGraphPass <|-- RasterRenderGraphPass
BaseRenderGraphPass <|-- UnsafeRenderGraphPass

class ComputeRenderGraphPass{
	BaseRenderFunc~PassData, ComputeGraphContext> renderFunc;
	ComputeGraphContext c 
}

class RasterRenderGraphPass{
	BaseRenderFunc~PassData, RasterGraphContext> renderFunc;
	RasterGraphContext c	
}

class UnsafeRenderGraphPass{
	BaseRenderFunc~PassData, UnsafeGraphContext> renderFunc;
	UnsafeGraphContext c	
}
```



# 流程

```c#
static void RenderSingleCamera(ScriptableRenderContext context, UniversalCameraData cameraData)
```

1. Renderer.Clear()

   对Attachment和Target做清理工作

2. RTHandles.SetReferenceSize

3. Cull

   - 回调PreCull函数
   - Cull

4. 创建渲染相关的的数据

   - UniversalLightData
   - UniversalShadowData
   - UniversalPostProcessingData
   - UniversalCameraData
   - UniversalRenderingData

5. CreateShadowAtlasAndCullShadowCasters

6. 添加RenderFeature中的RenderPasses

7. RecordAndExecuteRenderGraph

## RecordAndExecuteRenderGraph

1. BeginRecording

2. RecordRenderGraph

   

------

# URP流程

## 第一阶段 准备渲染数据

```mermaid
classDiagram
class UniversalResourceData{
	TextureHandle allRenderTexture
	void Reset()
}
```



## 第二阶段 记录RenderGraph

```c#
UniversalRenderPipeline.RecordAndExecuteRenderGraph(RenderGraph renderGraph, ScriptableRenderContext context, ScriptableRenderer renderer, CommandBuffer cmd, Camera camera, string cameraName)
{
    renderGraph.BeginRecording(RenderGraphParameters)
    {
        // renderGraph记录各种渲染过程中用的变量
        // cmd、scriptableRenderContext、
        // 初始化DefaultResources
        // 分配或者释放Shared Resources
    }
    
    ScriptableRenderer.RecordRenderGraph(RenderGraph renderGraph, ScriptableRenderContext context)
    {
        OnBeginRenderGraphFrame(); // 初始化UniversalResourceData，使其资源变得可读
        InitRenderGraphFrame();
        {
            // 设置全局关键字(Shadow)
            // 设置shader中的时间变量
        }
        
        UniversalRenderer.OnRecordRenderGraph(renderGraph, context);
        {
            UniversalRenderer.SetupRenderGraphLights(renderGraph, renderingData, cameraData, lightData);
            {
              // 设置Light，初始化，设置shader变量,keyword   
            }
            
           UniversalRenderer.CreateRenderGraphCameraRenderTargets(renderGraph, isCameraTargetOffscreenDepth);
            {
                // 只分配了cameraColor和cameraDepth
                // 其他只创建了Handle
            }
            
            // 记录BeforeRendering的RenderFeature
            RecordCustomRenderGraphPasses(renderGraph, RenderPassEvent.BeforeRendering);
            
            SetupRenderGraphCameraProperties(); // 设置Camera相关的shader变量，比如matrix
            
            OnBeforeRendering(renderGraph);
            {
                m_ForwardLights.PreSetup(renderingData, cameraData, lightData); // forward + 设置
                
                // 记录在阴影渲染前的pass
                RecordCustomRenderGraphPasses(renderGraph, RenderPassEvent.BeforeRenderingShadows);
                
                // 分配阴影贴图
                m_MainLightShadowCasterPass.Setup(renderingData, cameraData, lightData, shadowData);
                // 记录主光源的阴影
                m_MainLightShadowCasterPass.Render(renderGraph, frameData); 
                
                // 记录在阴影渲染后的pass
                RecordCustomRenderGraphPasses(renderGraph, RenderPassEvent.AfterRenderingShadows);
                
                OnMainRendering(renderGraph, context);
                {
                    // 记录ClearTargetsPass
                    
                    //
                }
            }
        }
    }
}
```



## 第三阶段 编译RenderGraph

## 第四阶段 执行RenderGraph
