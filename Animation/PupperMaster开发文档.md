## PupperMaster开发文档

### Muscle

#### 组成

ConfigurableJoint、Rigidbody、Collider

#### 功能

利用ConfigurableJoint使Rigidbody跟随动画帧中骨骼的旋转和位置(pinning)，也可以利用ConfigurableJoint和Rigidbody计算出物理上的位置，使骨骼跟随Rigidbody的位置(Mapping)

#### 实现机制：

1. 提供Mucule之间的碰撞忽略
2. 提供对Joint的参数调节
3. 对刚体的管理

#### 参数：

**pinWeight** 作用于刚体的位置和旋转

**MuscleWeight** 只用于计算关节应该施加力和弹簧的扭矩

**mappingWeight** 让骨骼跟随刚体的位置

#### 重要函数：

1. ```C#
   virtual void Initiate(Muscle[] colleagues)
   {
       // 初始化ConnectedBodyTarget joint中ConnectedBody所在的gameObject的Transform组件，一般是父组件
       
       //收集此gameObject上所有不是trigger不是meshCollider的Collider
       // 收集子gameObject中有刚体组件的Collider。
       UpdateColliders();
       
       // 如果没有Collider，则计算刚体的惯性张量
       
       // 设置rebuild相关信息，rebuild相当于设置初始值
       
       // 计算Joint空间
       
       // 计算Target与Muscale相对的位置旋转信息、一般为0
       
       // 保存默认的位置旋转
       
       // 读取动画中骨骼的信息 Read()
       
       // 处理未被转换为Muscles的有动画的子骨骼，保存其默认位置和旋转
   }
   
   void FixTargetTransforms()
   {
       // 恢复骨骼的位置至其默认位置
       // 默认位置指Tpose中的位置
   }
   
   void Read()
   {
       //记录骨骼在当前帧的 
       // 1 质心点
       // 刚体的质心点转换到世界空间空间下
       
       // 2 质心点移动的速度
       // 3 骨骼的位置
       // 4 骨骼的旋转
       // 5 骨骼的局部空间中的旋转
       
       // 这些信息将作为刚体的目标信息
   }
   
   void Update(float pinWeightMaster, float muscleWeightMaster, float muscleSpring, float muscleDamper, float pinPow, float pinDistanceFalloff, bool rotationTargetChanged, bool angularPinning, float deltaTime)
   {
       // 根据rigidBody的速度角速度更新state的速度角速度
       // Clamp props， state
       // pin()
       // MuscleRotation();
   }
   
   // 给刚体施加力使其处于骨骼的位置，施加扭矩处于骨骼的旋转
   // pinWeight
   void Pin(float pinWeightMaster, float pinPow, float pinDistanceFalloff, bool angularPinning, float deltaTime)
   {
       // 更改其速度和角速度
   }
   
   // 给关节一个驱动力，使其能旋转到骨骼的位置
   // muscleWeight
   void MuscleRotation(float muscleWeightMaster, float muscleSpring, float muscleDamper)
   {
       targetParent = connectedBodyTarget;// joint连接刚体所在Muscle的target(骨骼)，记录
       
       targetParentRotation = targetParent.rotation;// 父骨骼的旋转，实时读取
       parentRotation = joint.connectedBody.rotation; //joint连接刚体的旋转，即父刚体的旋转，实时读取
       
       toParentSpace = Quaternion.Inverse(targetParentRotation) * parentRotation; // 父刚体相对于父骨骼的旋转，记录，一般为0
       
       targetLocalRotation = Quaternion.Inverse(targetParentRotation * toParentSpace) * target.rotation;
       // target骨骼相对于父骨骼的旋转，实时读取
       
       localRotation = Quaternion.Inverse(parentRotation) * transform.rotation;// 刚体对与父刚体的旋转，实时读取
       
       localRotationConvert = Quaternion.Inverse(targetLocalRotation) * localRotation;// 需要旋转的差值，记录
       
       targetAnimatedRotation = targetLocalRotation * localRotationConvert;
       
       joint.targetRotation = toJointSpaceInverse * Quaternion.Inverse(targetAnimatedRotation) * toJointSpaceDefault;
       
   }
   
   // 直接更改骨骼的位置至刚体的位置,在LateUpdate中调用
   // mappingWeight
   public void Map(float mappingWeightMaster)
   {
   }
   
   // 将骨骼和关节都恢复到初始状态
   // state 设置为默认
   void Rebuild()
   {
   }
   ```

### PuppetMaster

#### 组成

Muscle、Behaviour、

#### 重要概念

**UpdateMode** 对应于Animator的UpdateMode（允许选择 Animator何时更新以及应使用哪个时间标度）

- Normal 对应于Animator的normal或者Unscaled Time
- AnimatePhysics 针对Animation component（Animate Physics）
- FixedUpdate 对应于Animator的Physics模式，此模式下PuppetMaster会控制Animator在FixUpdate中更新

**targetRoot**

动画更新的gameObject;

**rebuildFlag**

骨架可以被外力冲散，需要复原需要将此flag置为true

**disconnectMuscleFlags**

每个骨骼都可以与父骨骼断开，这个flag记录了每根骨骼Muscle的是否断开

**MuscleDisconnectMode**

骨骼的状态

- Sever 表示分离
- Explode 表示被炸开

**MuscleRemoveMode**

- Sever
- Explode
- Numb

#### 重要函数

```c#
void Initiate()
{
    // 设置TargetRoot，与PuppetMaster在同一层级下的具有Animator的组件
    // 设置targetAnimator
    // 应用配置文件
    
    // 寻找behaviours组件
    
    // 初始化管理的所有Musucle对象
    // 给Muscle对应的gameObject添加MuscleCollisionBroadcaster组件和JointBreakBroadcaster组件
    
    // 更新层级关系
    
    // 初始化附加物Musucle对象
    
    // 层级处理
    
    // 初始化behaviour
    
   	SwitchStates();
    
    SwitchModes();
    
    // Musucle Read;
    
    // 记录下骨骼在当前的位置和旋转targetMappedPosition、targetMappedRotation、targetSampledPosition、targetSampledRotation
    
    //  激活behaviour， BehaviourPuppet优先级最高
    
    // Clone MuSucle到defaultMuscles
    // rebuild时使用
}

void FixedUpdate()
{
    // 调用behaviour的FixedUpdateB
    
    // 这一帧是否rebuild整个Puppet
    if (rebuildFlag) OnRebuild();
    
    // 更新附加物的Muscle
    
    // 每根骨骼都可以被断开
    // 断开处理
    // 重新连接处理
    
    // Musucle的rigidbody是否有插值
    
    // 如果动画在物理更新时更新的话
    {
        //复原骨骼位置
        //调用Animator的Upate函数更新动画
        
        // IK的处理
        
        Read();
    }
    
    // 未冻结的情况下
    // 设置每个Muscle的solverIterationCount
    // 调用每个Muscle的Update函数
    // 1 pin 2 update muscle rotation
    // 给刚体addForce、addTorque
    // 给joint驱动力
    // 为进行物理引擎模拟做准备
}

// 使用场景
// 当人物骨骼被物理作用断开之后，如果需要恢复，则调用此函数
// defaultMuscles用来记录完成的骨骼
void OnRebuild()
{
    // Rebuild defaultMuscles中的每个Muscle
    // 补齐muscles数组
    // 调用behaviour的OnReactivate
}

void FixTargetTransforms()
{
    behaviour.OnFixTransforms();
    // 骨骼位置复原到初始位置
}

void Read()
{
    // 传送
    // 将Musucle一并传送到目标位置
    
    // Editor模式画出动画姿势
    VisualizeTargetPose();
    
    Muscle.Read();
    // 计算connectedAnchor
    muscles[i].UpdateAnchor(supportTranslationAnimation);
}

void Update()
{
    // 复原骨骼位置
}

void LateUpdate()
{
     behaviours.LateUpdateB(Time.deltaTime);
    
}

void OnLateUpdate()
{
    // Muscle读取动画当前骨骼位置
    
    SwitchStates();

    // Switching modes
    SwitchModes();
    
    // 没有冻结的情况下
    {
        //如果mapweight 大于0
        // 将骨骼的位置更新到rigidbody的位置
        Map();
        
        // 如果处于Kinematic模式
        // 移动刚体到骨骼的位置
        
        // 记录骨骼当前采样位置
        
        // CalculateMappedVelocity
        // 在Puppet被杀死使用
    }
    
    // 处理断开的骨骼
    
    // 处理冻结的情况
}
```

------

### BehaviourPuppet

#### 重要概念

State

- Puppet 		处于正常状态，并且固定在动画位置上
- Unpinned     处于失去平衡，并且用物理进行位置的计算
- GetUp          从unpinned状态到Puppet的过渡状态

NormalMode
在没有跟物体接触时，行为如何?

- Active  			保持PuppetMaster激活状态，并且一直Map
- Unmapped      完全利用动画进行蒙皮
- Kinematic        使PuppetMaster处于Kinematic模式

#### 重要函数

```c#
// 在PuppetMaster的Initiate方法中调用
void OnInitiate() 
{
    // 检查是否有无效层碰撞乘子的设置
    
    // 检查Ragdoll的碰撞体是否与CharacterController冲突
    
    // 计算向前向上的spine方向
    
    // 将状态设置为Unpinned
}

// 在PuppetMaster的Initiate方法中调用
void OnActivate()
{
    // 设置初始状态
    // 规则
    // 如果pinWight大于等于1，则再查询每个Muscle的pinWeightMlp，如果都大于0.5，则将状态设置为Puppet正常状态，否则设置为Unpinned状态
}

// AliveToDead中调用
void KillStart()
{
    // 肌肉处于断开连接状态时，设置其pinWeightMlp为0，让其完全由物理控制
    // 设置其碰撞体的物理材质
}

// AliveToDead中调用
void KillEnd()
{
    // 设置状态到Unpinned
}

// ToAlive时调用
void Resurrect()
{
    getupDisabled = false;
}

// 处理状态转换
void SetState(State newState)
{
    // 转换为Puppet状态时
    // 设置Muscle State的各种参数
    // 设置Collider的物理材质
    // 触发onRegainBalance

    // 转换为Unpinned时
    // 设置刚体的速度
    // 设置Collider的物理材质
    // 处理附加物
    // 设置每个Muscle的MuscleWeightMlp
    // 触发LoseBalance
    // 将每个Muscle的PinWeightMlp设置为0
    
   	// 转换为GetUp状态时
    // 触发起身函数
    // 设置每个Muscle的Collider的物理材质
   	// 计算并设置骨骼的位置旋转
}

// PuppetMaster FixedUpdate中调用
void OnFixedUpdate()
{
    // 首先处理附加物体，如🗡，枪等等
    // 掉落处理
    
    // 如果puppetMaster不在Active模式或者不在转换模式
    // 将state设置为Puppet后直接返回
    
    
    // 如果PuppetMaster处于Dead或者Frozen状态
    // 将每个连接状态的Muscle的pinWeightMlp设置为0, Mappingweight设置为1
    
    // Boosting 处理
    // 设置每个Muscle的immunity和impulseMlp
  
    // 处理Unpinned状态
    // 计时增加
    // 如果超过了GetUp延时且达到GetUp需要的条件，设置状态为GetUp，等下次循环
    // 否则设置每个骨骼的pinWeightMlp和mappingWeightMlp
    
    // 如果处于Puppet模式或者GetUp模式 
    // 对每个Muscle
    // 如果达到了Unpinned的条件，则设置状态为Unpinned，返回
    // 设置Muscle的muscleWeightMlp和pinWeightMlp乘子
    
   // 更新每个Muscle State的PinWeithMlp
    
    // 从GetUp恢复到正常模式
}

// PuppetMaster的LateUpdate中调用
void OnLateUpdate(float deltaTime)
{
    // Normal Mode转换
    // 如果上一个状态是Unmapped状态，修改每个Muscle的mappingWeightMlp至1
    // 如果上一个状态时Kinematci状态，设置为Active状态
    
   	// 当前状态转换
 	// 如果转换为Unmapped，计算mappingWiehtMlp
    // 如果转换为Kinematci，则直接将PuppetMaster的状态设置为Kinematci
    
}

// 在Read动画骨骼的信息时调用
void OnReadBehaviour(float deltaTime)
{
    // 未被冻结
    // 状态是Unpinned，将root移动到根骨骼刚体的位置
    // 找到与地面的交点，将roor移动到交点的位置
    // getUpPosition设置为root的位置
    
    // 状态时GetUp时，并且刚进入此状态
    //在root的up方向上投影，然后设置GetUpPosition，将root移动到此位置
    
    //getupAnimationBlendWeight>0时
    // GetUp状态时
    // 在root的up方向上投影，然后设置GetUpPosition，将root移动到此位置
    // 将各个骨骼的位置旋转固定到动画的位置和旋转上
}
```

------

#### 基本流程总结

```c#
void OnEable(){}

void Awake()
{
    PuppetMaster.Awake();
    {
        // targetRoot和targetAnimator设置
        // 读取并应用配置文件
        // 寻找Behaviours
        // 对每个Muscle进行初始化
        {
            Muscle.Initiate();
            {
                // 初始pinWeight、mapWeight等相关变量
                // 读取Muscle上的Collider
                // 计算刚体的惯性张量
                // 保存位置旋转等初始值，一边Rebuild时使用
                // 保存target与Muscle的相对位置旋转
                // 初始化Joint的相关信息
                // 读取当前动画资源中target的位置
            }
            
            // 注册Muscle的碰撞事件和Break事件
        }
        
        // 层级更新：flat or Tree
        
        // Behaviour处理
        {
            BehaviourPuppet.OnInitiate()
            {
                // Ragdoll和CharacterController的碰撞冲突处理
                // hipsForward、hipsUp的设置
            }
        }
        
        Muscle.Read();
        {
            // 读取或计算重心、动画中骨骼的位置旋转等信息
        }
        
        StoreTargetMappedState();
        {
            // 读取target骨骼的位置旋转并记录在targetMap中
        }
        
        // 激活behaviour
        {
            // 初始化behaviour的状态（unpined或者正常）
        }
        
        // 浅拷贝Muscles到defaultMuscles中
    }
}

void FixedUpdate()
{
    // 调用每个Behaviour的OnFixedUpdate函数
    // 根据模式和状态设置每个Muscle的state的各种乘子以决定蒙皮骨骼的位置在Ragdoll和动画骨骼之间怎么选
    BehaviourPupper.OnFixedUpdate();
    {
        // 对附加物的处理
        // puppetMaster处于未激活状态时，设置为正常模式
        
        // 正在转换状态时
        // 设置每个Muscle的state的pinweightMlp为0，mapWeight从0到1
        
        // 设置Muscle的state的immunity
    }
    
    // 如果需要rebuild， 则rebuild骨骼
    
    // 附加物骨骼的OnUpdate函数
    
    // 处理骨骼的连接状态
    
    // 确保各种参数的有效性
    
    // 调用每个Muscle的Update函数，这个函数里面根据参数计算了蒙皮骨骼的位置，使之前设置的参数起效
    //
}

// Internal PhysX Update

void OnCollisionEnter()
//void OnCollisionStay() 
{
    Behaviour.OnMuscleCollision(new MuscleCollision(muscleIndex, collision));
    {
        OnMuscleCollisionBehaviour()
        {
            // 达到以下条件不做处理
            // 1 Unpinned state
            // 2 collision layer 检测不通过
            // 3 GetUp状态下于groundlayer的碰撞
            // 4 Foot Group的ground layer的碰撞
            // 5 模式是Kinematic且PuppetMaster不处于活跃状态且物体是静态的且静态碰撞未开启
            // 6 这一帧处理的碰撞次数达上限
            
            float impulse = GetImpulse();
            {
                // 获取unity Collision的impulse
                // i = impulse/rigidbody.mass * 0.3f
                
                // 对特殊层的处理
                // i /= multiplier
                
                return i;
            }
            
            // 如果impulse小于阈值，不处理这次碰撞
            // 碰撞次数+1
            
            // 处理与其他Muscle的碰撞
            // impulse *= impulseMlp;
            
            // 激活PuppetMaster
            
            Unpin(impulse);
            {
                // 对每个Muscle进行Unpin处理
                UnPinMuscle()
                {
                    // immunity大于1不处理
                    // damage计算
                    // 与impulse成正比
                    // 与collisionResistance成反比
                    
                    // damage *= 1f - immunity;
                    
                    // pinWeightMlp = pinWeightMlp - damage
                }
            }
        }
    }
}

void OnCollisionExit()
{
    
}
    
void Update()
{
    // 调用Behaviour的OnUpdate函数
    // BehaviourPuppet没有重写OnUpdate函数
    
    // 确保当target骨骼没被动画帧控制时表现无抖动
    void FixTargetTransforms();
    {
        // 调用每个Behaviour的FixTargetTransforms
        // BehaviourPuppet没有重写FixTargetTransforms函数
        
        // 调用每个Muscle的FixTargetTransforms
        // 直接将骨骼位移旋转设置为默认值
    }
}

void LateUpdate()
{
    // 调用每个Behaviour的LateUpdate函数
    BehaviourPupper.OnLateUpdate();
    {
        // 根据要转换的状态设置每个Muscle的state的mappingWeightMlp(blending处理)
        // 
    }
    
	void OnLateUpdate();
    {
        // IK的处理
        
        // 读取动画中骨骼的数据并计算
        
        // 模式转换处理
        // 状态转换处理
        
        // Muscle.Map()
        // 根据参数和Radoll骨骼的位置修正蒙皮骨骼的位置
        
        // 保存targetMappedPostion
    }
}

```

