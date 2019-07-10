#### 开始使用
* 介绍
* 初始化
    * FMOD Studio API初始化
* FMOD Core API初始化(不要用如果你正在用FMOD Studio API初始化)
    * 高级初始化设置
* 播放声音(Core API only)
* 使用解压采样 vs 压缩采样 vs 流
    * 解压缩采样
    * 流
    * 压缩采样
* 更新
* 关闭
* 错误检查
* 配置
* 在加载或释放声音时避免停机


#### 介绍
FMOD Studio和Core API被设计为直观和灵活的。在本节中，将介绍如何使用该引擎以及如何有效地使用它所涉及的关键问题。  

FMOD提供了c++ API和C API。它们的功能相同，实际上c++和C函数可以互换使用，c++和C类可以来回转换。下面的示例只显示c++版本。

#### 初始化
#####  FMOD Studio API初始化
使用Studio API时，可以创建一个FMOD Studio系统，然后调用Studio::System::initialize。该函数还将初始化内置的核心FMOD系统。下面是一个简单的例子:
```c++
FMOD_RESULT result;
FMOD::Studio::System* system = NULL;

result = FMOD::Studio::System::create(&system); // Create the Studio System object.
if (result != FMOD_OK)
{
    printf("FMOD error! (%d) %s\n", result, FMOD_ErrorString(result));
    exit(-1);
}

// Initialize FMOD Studio, which will also initialize FMOD Core
result = system->initialize(512, FMOD_STUDIO_INIT_NORMAL, FMOD_INIT_NORMAL, 0);
if (result != FMOD_OK)
{
    printf("FMOD error! (%d) %s\n", result, FMOD_ErrorString(result));
    exit(-1);
}
```

##### FMOD Core API初始化(不要用如果你正在用FMOD Studio API初始化)
可以使用FMOD核心API而不需要使用FMOD Studio API。使用核心API可以访问加载和播放声音、创建DSP效果、设置FMOD通道组以及设置采样精确的淡出点和开始/停止时间等基本功能。然而，仅使用核心API时，将不可能加载Studio bank或加载并播放声音艺术家在Studio工具中设置的Studio事件。直接初始化FMOD核心:


```c++
FMOD_RESULT result;
FMOD::System *system = NULL;

result = FMOD::System_Create(&system);      // Create the main system object.
if (result != FMOD_OK)
{
    printf("FMOD error! (%d) %s\n", result, FMOD_ErrorString(result));
    exit(-1);
}

result = system->init(512, FMOD_INIT_NORMAL, 0);    // Initialize FMOD.
if (result != FMOD_OK)
{
    printf("FMOD error! (%d) %s\n", result, FMOD_ErrorString(result));
    exit(-1);
}
```


##### 高级初始化设置
FMOD可以通过调用System::setAdvancedSettings或Studio::System::setAdvancedSettings来定制高级设置。有关有效虚拟语音的典型设置的描述，请参见虚拟语音系统。

##### 播放声音(Core API only)
最简单的入门方法，也是FMOD核心API的基本功能，是初始化FMOD系统，加载一个声音，然后播放它。所有函数都立即执行，因此开发人员要么在主循环执行期间触发并忘记，要么轮询声音以完成。播放声音不会阻塞应用程序。

执行一个简单的playSound：
* 使用上面描述的系统对象句柄，用System::createSound加载一个声音。这将返回一个声音句柄。这是你对加载的声音的句柄。
* 使用System::playSound播放声音，使用步骤1返回的声音句柄。这将返回一个通道句柄。
* 让它在后台播放，或者使用ChannelControl::isPlaying监控它的状态，使用步骤2返回的通道句柄。当声音结束时，当调用任何相关的基于通道的函数时，通道句柄也会立即失效，所以这是知道声音结束的另一种方法。返回的错误代码将是FMOD_ERR_INVALID_HANDLE。
* 

#### 使用解压缩采样 vs 压缩采样 vs 流

##### 解压缩采样
createSound的默认模式是FMOD_CREATESAMPLE，它将声音解压到内存中。这对于分发压缩后的声音很有用，然后在运行时对它们进行解压，以避免在播放时解压声音的开销。根据格式的不同，这在移动设备上可能很昂贵。解压到PCM在回放期间使用很小的CPU，但在运行时也要使用很多倍的内存。

##### 流
将声音作为流媒体加载，可以获得一个大文件，并一次小块地实时读取/播放它，从而避免了将整个文件加载到内存中的需要。这是典型的保留音乐/声音/对话或长氛围的轨道。用户可以通过向System::createSound函数添加FMOD_CREATESTREAM标志，或者使用System::createStream函数，简单地将声音作为流播放。这两个选项等价于相同的终端行为。

##### 压缩采样
要播放压缩后的声音，只需将FMOD_CREATECOMPRESSEDSAMPLE标志添加到System::createSound函数中  
由于压缩样本比较复杂，它们需要处理更大的上下文(例如vorbis解码信息)，因此对于播放声音，每个声音开销是恒定的(直到一个固定的限制)。  


如果用户调用System::setAdvancedSettings并设置maxCodecs值，或者在第一次加载带有FMOD_CREATECOMPRESSEDSAMPLE标志的声音时，这种分配通常发生在System::init时间。这将不会由用户配置，因此使用默认的32编解码器进行分配。   

例如:vorbis编解码器的每个语音开销为16kb，因此默认的32 vorbis编解码器将消耗512kb的内存。这是可调的，用户可以使用前面提到的System::setAdvancedSettings函数来减少或增加默认值32。用户将为vorbis情况调整FMOD_ADVANCEDSETTINGS maxVorbisCodecs值。其他支持的编解码器也是可调的。



#### 更新
FMOD应该在每次游戏更新时勾选一次。当使用FMOD Studio时，调用Studio::System::update，它内部也将更新核心系统。如果直接使用Core，则调用System::update。

如果FMOD Studio在异步模式下运行(默认情况下，除非FMOD_STUDIO_INIT_SYNCHRONOUS_UPDATE已被指定)，那么Studio::System::update将非常快，因为它只是为异步执行帧的命令交换了一个缓冲区。

#### 关闭
要关闭FMOD Studio，请调用Studio::System::release。如果直接使用内核，则调用System::release。

#### 错误检查
在FMOD示例中，使用一个宏检查错误代码，如果发生意外错误，该宏调用处理函数。这是调用FMOD Studio API函数的推荐方法。当公共FMOD函数出现错误时，还可以接收回调函数。有关更多信息，请参见FMOD_SYSTEM_CALLBACK。

#### 配置
如果您希望不同于默认的行为，可以设置输出硬件、FMOD的资源使用和其他类型的配置选项。这些通常在System::init之前调用。有关这些示例，请参见Studio::System::getCoreSystem, System::setAdvancedSettings, Studio::System::setAdvancedSettings。

#### 在加载或释放声音时避免停机
加载声音是最慢的操作之一。要将声音加载放到后台，使其不影响主应用程序线程中的处理，用户可以在System::createSound或System::createStream中使用fmod_nonblock标志。

立即向用户返回一个声音句柄。然后可以使用sound::getOpenState检查正在加载的声音的状态。如果在仍在加载的声音上调用函数(除了getOpenState)，它通常会返回FMOD_ERR_NOTREADY。等到声音准备好再播放。状态将是FMOD_OPENSTATE_READY。


要避免流媒体声音在尝试释放/释放时出现停顿，请在调用sound::release之前检查状态是否为FMOD_OPENSTATE_READY。
