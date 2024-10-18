# H项目Lua类

```mermaid
classDiagram
class BaseCharRenderer{
	AttachCore attachCore
	CharAnimatorHelper animatorHelper
	HairRenderer hairrenderer
}

BaseCharRenderer<|-- MainRenderer

class MainRenderer{
	DressCore dressCore
	List~SkinRenderer~ skinRenderers
	List~ClothRenderer~ clothRenderers
	List~MakeUpNodes~ makeUpNodes
}
```

