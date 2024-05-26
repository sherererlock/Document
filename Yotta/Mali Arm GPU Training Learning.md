# Mali Arm GPU Training Learning

## The Rendering Pipeline

VertexShader会被分成两个阶段：

第一阶段是Position Shader，只处理跟顶点位置相关的Shader

接下来进行Primitive Culling阶段，视锥剔除和背面裁剪以及小三角形裁剪等等

光栅化阶段输出fragment时以2x2的像素块为单位，称为quad

Hidden Surface Removal

##  GPU architecture

IMR

TBDR

## Hardware Shader Cores

## Best practice principles



## Mali Offline Compiler

![image-20240525215019031](D:\Games\Document\Yotta\Mali offline Complier.png)

### Work  registers:

目标：减少到32个以下。

原理：Shader使用的物理寄存器个数影响到GPU同时运行的线程组数

方法：降低计算的浮点数精度，从32bit到16bit

### Stack spilling:

别开

### 16-bit arithmetic: 

占比越高越好

高精度计算只用在位置和深度计算上

### Arithmetic Unit

### Load/Store Unit

### Varing Unit

### Texture Unit

### Shader properties

#### Has uniform computation: true

移动到CPU上

#### Has side-effects: false

#### Modifies coverage: true

#### Uses late ZS test: false

#### Uses late ZS update: true

#### Reads color buffer: false