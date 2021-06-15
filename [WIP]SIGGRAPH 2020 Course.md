# SIGGRAPH 2020 Course: PBR


## Samurai Shading in Ghost of Tsushima

## SGGX
SGGX是一个渲染volume的算法，比如毛发、纺织物等。对于这类物体的渲染（特别是各向异性的毛发），通常是建模为microflake，光线在volume中传播时会和microflake相互作用，发生散射、吸收、发射等现象。在SGGX之前，相关的渲染算法有两个问题

1. 模型复杂，渲染开销大。
2. 不易downsample，进行低分辨率渲染时性价比低。

因此，为了应对现有模型的问题，SGGX是一种便于LoD，也更加简单的模型。

SGGX模型的关键点有两个

1. 

