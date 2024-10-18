# 项目Pass列表

| Pass                         | Load Color | Load DepthStencil | Store Color | Store Depthstencil | FullScreen | 画质     |
| ---------------------------- | ---------- | ----------------- | ----------- | ------------------ | ---------- | -------- |
| **Hair ShadowDepth**         | DontCare   | Load              | DontCare    | Store              | ✘          | 全       |
| **MixedCharacterShadow**     | DontCare   | Load              | DontCare    | Store              | ✘          | 全       |
| **HairShadow**               | DontCare   | DontCare          | Store       | **DontCare**       | ✓          | 高、极高 |
| **HairShadowBlit**           | DontCare   | DontCare          | Store       | **DontCare**       | ✓          | 高、极高 |
| **WriteClothStencilAndPreZ** | DontCare   | Load              | DontCare    | Store              | ✓          | 全       |
| **DrawOpaqueObjects**        | Load       | Load              | Store       | Store              | ✓          | 全       |
| **WriteStockStencil**        | DontCare   | Load              | DontCare    | Store              | ✓          | 全       |
| **RenderSkybox**             | Load       | Load              | Store       | Store              | ✓          | 全       |
| **HairPreDepth**             | DontCare   | Load              | DontCare    | Store              | ✓          | 全       |
| **SceneFurShellPass**        | Load       | Load              | Store       | Store              | ✓          | 全       |
| **CopyDepth**                | DontCare   | Load              | DontCare    | Store              | ✓          | 高、极高 |
| **SpecialTransparentPreZ**   | DontCare   | Load              | DontCare    | Store              | ✓          | 全       |
| **DrawTransparentObjects**   | Load       | Load              | Store       | **DontCare**       | ✓          | 全       |
| **CustomURPDOF**             | Load       | DontCare          | Store       | DontCare           | ✓          | 高、极高 |
| **CopyTransparentColor**     | Load       | DontCare          | Store       | DontCare           | ✓          | 高、极高 |
| **GlassRealTime**            | Load       | Load              | Store       | DontCare           | ✓          | 全       |
| **Bloom**                    | Load       | DontCare          | Store       | DontCare           | ✓          | 全       |

注：

1. Load表示需要把System Mem中的内容加载到GPU On-Chip Buffer上
2. Store表示需要把GPU On-Chip Buffer中的内容存储到System Mem
3. Dont Care则表示没有带宽的消耗，表明这个Pass不需要读取和写入相应的Buffer
4. 由于项目中的DepthStencilt贴图的格式是Depth32Stencil8，所以1个像素占用64Bit，其中24Bit是为了内存对齐，DepthStencil的读取和写入更耗带宽