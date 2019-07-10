* 3D 回响
    * 介绍
    * 3D回响
    * 创建D回响
    * 设置3D属性
    * 完成


#### 介绍
环境在不同的位置表现出不同的混响特性是很常见的。理想情况下，当听众在虚拟环境中移动时，混响的声音应该相应地改变。混响属性的这种变化可以在FMOD Studio中使用内置的Reverb3D API建模。

#### 3D回响
3D混响系统允许您在3D世界中放置多个虚拟混响。每个混响定义:
* 在3D世界中的位置
* 受混响影响的面积或范围(以最小和最大距离)
* 区域的混响特性

在运行时，FMOD Studio会根据听众的邻近程度和混响的位置和重叠程度，在3D混响的特征之间插入(或变形)。这种方法允许FMOD Studio使用一个混响DSP单元在3D世界中提供动态混响。这个过程如下图所示。
![image](https://www.fmod.com/docs/2.0/api/images/3d-reverb.png)


当监听器处于一个或多个3D混响的效果范围内时，监听器将听到影响混响的加权组合。当监听器位于所有3D混响的覆盖范围之外时，不应用混响。重要的是要注意,默认情况下,2d听起来共享相同的物理混响实例,以避免2d声音混响,使用ChannelControl:: setReverbProperties并设置 wet= 0,或2d的声音转向一个不同的混响实例,使用相同的功能(添加第二混响会招致一个命中CPU和内存)。


三维混响的插值仅仅是对环境中多重混响如何发声的一种估计。在某些情况下，需要更现实一些。在这些情况下，我们建议使用多个物理混响，如教程“使用多个混响”中所述。


#### 创建3D回响
现在，我们将使用调用系统::createReverb3D创建一个虚拟混响，然后使用Reverb3D::setProperties设置混响的特性。

```
result = system->createReverb3D(&reverb);
FMOD_REVERB_PROPERTIES prop2 = FMOD_PRESET_CONCERTHALL;
reverb->setProperties(&prop2);
```

#### 设置3D属性

现在必须设置混声的3D属性。方法Reverb3D::set3DAttributes允许我们使用最小距离和最大距离设置原点位置以及覆盖面积。

```
FMOD_VECTOR pos = { -10.0f, 0.0f, 0.0f };
float mindist = 10.0f; 
float maxdist = 20.0f;
reverb->set3DAttributes(&pos, mindist, maxdist);
```

由于3D混响在权重计算中使用了监听器的位置，我们还需要确保使用System:: set3dlisteneratrbutes设置监听器的位置。

```
FMOD_VECTOR  listenerpos  = { 0.0f, 0.0f, -1.0f };
system->set3DListenerAttributes(0, &listenerpos, 0, 0, 0);
```
#### 完成
这是所有需要得到虚拟3d混响区工作。从这一点开始，根据听众的位置，混响预置应该互相变形，如果他们重叠，并衰减基于听众的距离三维混响球的中心。
