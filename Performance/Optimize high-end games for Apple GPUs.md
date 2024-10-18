# Optimize high-end games for Apple GPUs

## Optimization Process

![image-20240529125513429](D:\games\Document\Performance\image-20240529125513429.png)

1. Mearsure
2. Analyze
3. Improve
4. Verify

## CheckList

![image-20240529125710237](D:\games\Document\Performance\image-20240529125710237.png)

![image-20240529125803424](C:\Users\ruizu\AppData\Roaming\Typora\typora-user-images\image-20240529125803424.png)

## Metal Debugger

### Summary页面

#### Show Dependencies

#### BandWidth

Loseless Compression Warnnings for textures and Reasons

#### API

Redundant Bindings

### Memory页面

1. Filter by RenderTargets
2. Texture---Usage
3.  

### Group by API Call 

###  Group By Pipeline State

#### Function Statistics

观察指令数

#### Pipeline Stage

观察计算过程中绑定的资源

#### Compile Options

Fast math flag

![image-20240529133122002](D:\games\Document\Performance\image-20240529133122002.png)

## Instruments Tools

#### Game Performance

#### Metal System Trace(GPU)

## 优化

### Shader

![image-20240529130803679](D:\games\Document\Performance\image-20240529130803679.png)

High ALU + Low shader Occupancy 指示寄存器压力



## BandWidth

![image-20240529131711729](D:\games\Document\Performance\image-20240529131711729.png)

- Color Attachment不需要ShaderWrite标志

## Memory 

## Redunt Bindings

## GPU Timeline

### GPU

#### Fragment

所有pass的时间分布

shader时间

load/store

#### Encode

一次pass的所有信息

固定ALU和Occupancy两条Timeline

High ALU + Low shader Occupancy 指示寄存器压力

Floating Point Utilization

 Shader right click openg shader

Shader Profiler

### Counters



