## Animancer

对比Mecanim优势：

- 简单 相比之下播放动画更简单
- 透明 源码开放，容易查bug
- 安全 直接引用动画资源，而不是用State的字符串
- 清晰 脚本可以定义自己的逻辑和需求，不像Mecanim中Animator Controller和Script的关系
- 可靠 代码中怎么写，程序就怎么跑。unity原生系统中为了维护自己的代码行为，有时候会延迟命令，甚至忽略。
- 可适配性 

------

#### Animancer类图

```mermaid
classDiagram
class PlayableHandle{
	-IntPtr m_Handle
	-bool Play()
	-bool SetInputWeight(int inputIndex, float weight)
	- ...
}
class IPlayable{
	+ PlayableHandle GetHandle()
}

class Playable{
	-PlayableHandle m_Handle
	+static Playable Create(PlayableGraph graph, int inputCount)
}

class AnimationClipPlayable{
	-PlayableHandle m_Handle
	+static AnimationClipPlayable Create(PlayableGraph graph, AnimationClip clip)
	-PlayableHandle CreateHandle(PlayableGraph graph, AnimationClip clip)
}

IPlayable <|--Playable
IPlayable<|--AnimationClipPlayable
Playable o-- PlayableHandle
AnimationClipPlayable o-- PlayableHandle
```

```mermaid
classDiagram
class IPlayableBehaviour{
	+ void OnGraphStart(Playable playable)
	+ void OnPlayableCreate(Playable playable)
	+ void PrepareFrame(Playable playable, FrameData info)
	+ void ProcessFrame(Playable playable, FrameData info, object playerData)
	+ ...()
}

class PlayableBehaviour
IPlayableBehaviour<|--PlayableBehaviour

```

```mermaid
classDiagram
class PlayableGraph{
	+IntPtr m_Handle
	+static PlayableGraph Create()
	+static PlayableGraph Create(string name)
	+bool IsPlaying()
	+void Play()
	+void Evaluate(float deltaTime)
	+void SetTimeUpdateMode(DirectorUpdateMode value)
	+PlayableOutput GetOutput(int index)
}
```

```mermaid
classDiagram
class IPlayableOutput{
	+PlayableOutputHandle GetHandle()
}
class PlayableOutputHandle{
	+ IntPtr m_Handle
	+ void SetWeight(float weight)
}
class PlayableOutput{
	+ PlayableOutputHandle m_Handle
}

class AnimationPlayableOutput{
	PlayableOutputHandle m_Handle
	+ Animator GetTarget()
	+static AnimationPlayableOutput Create(PlayableGraph graph, string name, Animator target)
}

IPlayableOutput<|--PlayableOutput
IPlayableOutput<|--AnimationPlayableOutput
AnimationPlayableOutput o-- PlayableOutputHandle
PlayableOutput o-- PlayableOutputHandle
```

------

```mermaid
classDiagram
class IPlayableWrapper{
	+ float Weight 
    +  float Speed
	+ Playable Playable
	+ int ChildCount
	
	AnimancerNode GetChild(int index);
}

class AnimancerPlayable{
	+ PlayableGraph _Graph
	+ Playable _RootPlayable
	+ Playable _LayerMixer
}
IPlayableWrapper<|--AnimancerPlayable
PlayableBehaviour<|--AnimancerPlayable

class AnimancerNode{
	Playable _Playable
	+ AnimancerPlayable Root
	+ AnimancerLayer Layer
	+virtual void CreatePlayable()
	+abstract void CreatePlayable(out Playable playable)
	
	void ConnectToGraph()
}
IPlayableWrapper<|--AnimancerNode

class AnimancerLayer{
	AnimationMixerPlayable _Playable
}
AnimancerNode<|--AnimancerLayer

class AnimancerState{
	+ AnimationClip Clip
	+ bool _IsPlaying
	+ float _Time;
	+ float RawTime
	+ void Play()
	+ void Stop()
	+ void MoveTime(float time, bool normalized)
}
AnimancerNode<|--AnimancerState

class LayerList{
	+AnimancerLayer[] _Layers
	+AnimationLayerMixerPlayable LayerMixer 
}

LayerList o-- AnimancerLayer
AnimancerPlayable o-- LayerList

class StateDictionary{
	-Dictionary~object, AnimancerState~ States
}

StateDictionary o--AnimancerState
AnimancerPlayable o-- StateDictionary

class ClipState{
	+ AnimationClip _Clip
	+ AnimationClipPlayable _Playable
	+float Length => _Clip.length
	+override void CreatePlayable(out Playable playable)
}

AnimancerState<|--ClipState

class AnimancerComponent{
	+Animator _Animator
	+AnimancerPlayable _Playable
	
	+AnimancerState Play(AnimationClip clip)
	+AnimancerState Play(AnimancerState state)
	+AnimancerState Play(AnimationClip clip, float fadeDuration, FadeMode mode)
	
	AnimancerState TryPlay(object key)
	+void Stop()
	+void Evaluate(float deltaTime)
}
AnimancerComponent o-- AnimancerPlayable
```

## 流程

使用PlayableAPI播放动画的一般流程是：

1. 创建一个PlayableGraph
2. 创建AnimationPlayableOutput
3. 创建Playable
4. 连接Playable与AnimationPlayableOutput
5. 调用playableGraph.Play()

#### 初始化

```c#
void AnimancerComponent.InitializePlayable()
{
    // Animator
    
    //创建Grpha和AnimancerPlayable
    static AnimancerPlayable Create();
    {
        var graph = PlayableGraph.Create();
        return ScriptPlayable<AnimancerPlayable>.Create(graph, Template, 2).GetBehaviour();
    }
    
    //创建Layers和States
    void AnimancerPlayable.OnPlayableCreate(Playable playable)
    {
        LayerList(AnimancerPlayable root, out Playable layerMixer)
        {
            //创建LayerMixer，连接到RootPlayable
            AnimationLayerMixerPlayable.Create(root._Graph, 1)
           	Root._Graph.Connect(layerMixer, 0, Root._RootPlayable, 0); 
        }
    }
    
    // 创建AnimationPlayableOutput
    _Playable.CreateOutput(_Animator, this);
    {
     	// 给Output连接playable
        output.SetSourcePlayable(playable, 0);
    }
}

```

#### 播放动画

```C#
AnimancerState Play(AnimationClip clip);
{
    AnimancerState State = States.GetOrCreate(clip);
    {
        // 创建ClipState
        return Root.Layers[0].CreateState(key, clip);
    }
}
```

