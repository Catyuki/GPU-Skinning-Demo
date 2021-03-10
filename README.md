# GPU-Skinning-Demo  
  
一种利用GPU来实现角色动画效果，减少CPU蒙皮开销的技术实现
- - -

  当场景中有很多人物动画模型的时候，性能会产生大量开销，其中很大一部分来自于骨骼动画。GPU Skinning是将CPU中的蒙皮工作转移到GPU中进行，从而能够较大地提升多角色场景的运行效率，在大规模群体动画模拟如MMO、RTS等游戏中有较大的应用性。
可以发现如果要渲染的动画角色数量很大时主要会有以下两个巨大的开销：

- CPU在处理动画时的开销。
- 每个角色一个Draw Call造成的开销。

CPU的这两大开销限制了我们使用传统方式渲染大规模角色的可能性，主要瓶颈是角色动画的处理都集中在CPU端。瓶颈之二是CPU和GPU之间的Draw Call问题，如果利用批处理（包括Static Batching和Dynamic Batching）或是从Unity5.4之后引入的GPU Instancing就可以解决这个问题。但是，不幸的是这两种技术都不支持动画角色的SkinnedMeshRender。

那么解决方案就呼之欲出了，那就是将动画相关的内容从CPU转移到GPU，同时由于CPU不需要再处理动画的逻辑了，因此CPU不仅省去了这部分的开销而且SkinnedMeshRender也可以替换成一般的Mesh Render，我们就可以很开心的使用GPU Instancing来减少Draw Call了。
- - - 
  
由于目标是去掉SkinnedMeshRender，它的作用是蒙皮； 蒙皮需要的关键信息:
- **每个顶点的关联骨骼信息(Bone Index与Bone Weight)**
- **每帧骨骼的变换信息（每帧每个骨骼的变换）**
  
因此就要想方法将这两种信息传入GPU，通过GPU来计算骨骼并进行蒙皮。以下是简单的步骤说明；

## 1.渲染骨骼矩阵到材质 : [CreateBoneTex](https://github.com/Minghou-Lei/GPU-Skinning-Demo/blob/99febe38218011850e97795687cc2c8864aad8d7/Assets/Scripts/AnimationBoneBaker.cs#L111)
可以通过将每一帧每一根骨骼的变换转为Matrix4X4矩阵，将变换信息转为RGBA值，作为材质（Texture2D）交给Shader进行蒙皮； 其中骨骼变换矩阵的获取方法：

```c#
//采样当前帧
clip.SampleAnimation(gameObject, i / clip.frameRate);

//模型空间-骨骼空间-世界空间-模型空间
Matrix4x4 matrix = skinnedMeshRenderer.transform.worldToLocalMatrix * bones[j].localToWorldMatrix *
bindPoses[j];
```
骨骼变换的原理如下图：
  
![spaceTransform](https://github.com/Minghou-Lei/GPU-Skinning-Demo/blob/main/Assets/imgs/space.jpg)

获得矩阵信息后，将其渲染成Texture2D。为了保持精度，可以通过EncodeFloatRGBA函数将每一个float值转为RGBA空间上的一个点，然后将其逐一渲染到Texture2D上面来,渲染出的图片：
  
![BoneMatrix2Texture2D](https://github.com/Minghou-Lei/GPU-Skinning-Demo/blob/99febe38218011850e97795687cc2c8864aad8d7/Assets/Ch36_nonPBR%40Dancing%20Running%20Man.Dancing%20Running%20Man.BoneMatrix.jpg "BoneMatrix2Texture2D")
  
每一帧每一块骨骼将会被取样12次，作为12个Pixel储存在材质中。

## 2.添加骨骼索引信息与权重信息到Mesh的UV通道 : [MappingBoneIndexAndWeightToMeshUV](https://github.com/Minghou-Lei/GPU-Skinning-Demo/blob/99febe38218011850e97795687cc2c8864aad8d7/Assets/Scripts/AnimationBoneBaker.cs#L181)

虽然我们有了每帧每骨骼的变换信息，但是还有一点，就是一个顶点受哪些骨骼的影响及其程度如何，还没有实现。但是仔细一想就明白，这个索引跟权重是一个Mesh静态的数据。

既然是Mesh静态的，那就直接写到Mesh里，最常见的当然就是UV通道了。UV是一个Vector2的向量，因此一次只能存一对权重索引数据。如果你对精度提出了更高的要求，那可以用2个UV通道或者4个UV通道。
  
```c#
//深拷贝一个原来的Mesh
var bakedMesh = new Mesh();
bakedMesh = Instantiate(mesh);

//为新的Mesh的UV2、UV3通道添加骨骼信息和权重信息
MappingBoneIndexAndWeightToMeshUV(bakedMesh, UVChannel.UV2, UVChannel.UV3);
```

## 3.将蒙皮所需信息在Shader中合并（代替原来的CPU蒙皮） : [BoneAnimationShader](https://github.com/Minghou-Lei/GPU-Skinning-Demo/blob/99febe38218011850e97795687cc2c8864aad8d7/Assets/Shaders/BoneAnimationShader.shader)