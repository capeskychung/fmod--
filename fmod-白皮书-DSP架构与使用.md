* DSP架构与使用
    * 介绍
    * DSP网络中的一些常用单元
    * 观看DSP网络的构建(带有代码示例)
        * 从一无所有开始，然后播放一些声音
        * 为通道添加DSP效果
        * 向ChannelGroup添加一个效果
        * 创建一个效果，并使所有通道发送给它。
        * 设定DSP单元的输出格式，控制输出信号的pan矩阵
        * 绕过效果/禁用它。
        * 执行顺序和拉/不拉遍历
        * “发送”与“标准”连接类型



#### 介绍
本节将向您介绍FMOD Studio高级DSP系统。使用该系统，您可以实现自定义过滤器或创建复杂的信号链，以创建高质量和动态声音音频。FMOD Studio DSP系统是一个非常灵活的混合引擎，它强调质量、灵活性和效率，当充分利用其潜力时，它是一个非常强大的系统。  
下图显示了一个非常基本的FMOD DSP网络的样子。     
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img001.png)

音频数据从右向左流动，直到最终到达声卡，完全混合和处理。
* 一个蓝色的盒子是一个DSP单元。该单元由FMOD核心头中的DSP类表示。
* 框之间的一条线是DSPConnection。这就是将DSP单元连接成网络的原因。每个DSPConnection都有一个pan矩阵，您可以使用它来配置从输入扬声器/通道到输出扬声器/通道的映射。
* 灰色条内的绿色竖条检测信号电平。您可以看到，可波单元产生一个单声道信号，该单声道信号继续通过通道推杆(未受影响)，然后混合到6个通道(5.1)。因为默认锅mono 5.1输出声音是mono信号衰减了3 db到前面左扬声器,由3 db和信号衰减到前面右扬声器,您可以看到6灰色酒吧只有信号的2演讲者的水平。有关扬声器顺序，请参见FMOD_SPEAKER，由这些条表示。注意:自从FMOD Studio 1.04.08开始，upmix就在主频道组的fader之外的内部发生，因此对于本教程的目的，主频道组的fader已经被强制设置为FMOD_SPEAKERMODE_5POINT1，以便能够可视化。有关通道格式的更多信息，请参阅下面的“设置DSP单元的输出格式，并控制其输出信号的pan矩阵”一节。

上面的图片是使用FMOD Profiler工具拍摄的。只要在初始化核心引擎时指定FMOD_INIT_PROFILE_ENABLE，就可以配置自己的DSP网络。该工具位于SDK的/bin目录中。

#### DSP网络中的一些常用单元
本节将从右到左更详细地描述单元，从数据的原点到声卡。下面的列表描述了您将在图中看到的一些典型DSP单元。
* **可波单元**该单元从声音缓冲区读取原始PCM数据，并将其重定向到与声卡相同的速率。只有当用户调用System::playSound时，才连接可波单元。重新采样后，音频数据将按照声卡的速率处理(或流动)。默认值通常为48khz。在iOS(22千赫)
* **DSPCodec Unit**该单元从FMOD编解码器中读取解码压缩数据，并将其传递给内置的重采样器，然后将解压后的结果传递给输出。
* **通道阅读器**这个单元为通道提供了一个顶层单元，用于为通道插入效果。通道阅读器还控制通道的音量级别，例如，如果用户调用ChannelControl::setVolume。
* **ChannelGroup Fader** 这个单元为一个通道组提供了一个顶层单元，用于保存通道组，并且是一个插入通道组效果的地方。ChannelGroup Fader还控制通道的音量级别，例如，如果用户调用ChannelControl::setVolume。

当FMOD在通道上播放PCM声音(使用System::playSound)时，它创建一个由Fader和可波单元组成的子网络。如果播放流，即使压缩了源数据，也会发生这种情况。

当FMOD在通道上播放压缩的声音(通常是MP3/Vorbis/XMA/ADPCM，加载了FMOD_CREATECOMPRESSEDSAMPLE)时，它会创建一个类似的子网络，由一个Fader和一个DSPCodec单元组成。

当FMOD在通道上播放DSP(使用System::playDSP)时，它创建一个由一个Fader和一个独立的重采样单元组成的子网络。由重采样器执行的用户指定的DSP作为重采样器的子网络，在分析器上不可见。

#### 观看DSP网络的构建(带有代码示例)
##### 从一无所有开始，然后播放一些声音
在本节中，我们将研究一些基本的技术，可以用来操作DSP网络。我们将从最基本的信号链开始(如下图所示)，并使用提供的代码识别DSP网络发生的变化。      
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img002.png)

注意，网络只存在一个单元。主信道组的DSP Fader单元(FMOD_DSP_TYPE_FADER)。如果需要，这个单元可以用来控制整个混合的输出。

现在我们将使用System::playSound播放PCM声音。   
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img003.png)

注意，主信道组的DSP Fader单元上附加了DSP Fader单元的子网络(FMOD_DSP_TYPE_FADER)和系统级DSP可波单元。

让我们再次播放声音，结果是2个频道是活跃的。    
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img004.png)  
注意，现在新的通道目标是相同的主通道组DSP Fader单元，当2条线合并成一个单元时，就会发生“混合”。这只是两个信号的和。

#####  为通道添加DSP效果
在本例中，我们将通过将DSP效果单元连接到通道来为声音添加效果。下面的代码首先播放一个声音，然后创建一个带有System::createDSPByType的DSP单元，并使用ChannelControl::addDSP将其添加到DSP网络中。

```
FMOD::Channel *channel;
FMOD::DSP *dsp_echo;
result = system->playSound(sound, 0, false, &channel);
result = system->createDSPByType(FMOD_DSP_TYPE_ECHO, &dsp_echo);
result = channel->addDSP(0, dsp_echo);
```
下图显示了插入在“信道头”或位置0处的FMOD回波效果，该效果由ChannelControl::addDSP命令指定(位置= 0)。曾经是头单位的通道推杆，现在被移到1号位置。  

![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img005.png)

如果我们调用ChannelControl::setDSPIndex

```
result = channel->setDSPIndex(dsp_echo, 1);
```
我们可以看到下面，echo现在向下移动了1,Channel Fader回到了位置0。   
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img006.png)


创建一个新的ChannelGroup并将我们的Channel添加到其中
在这个例子中，我们将引入信道组，它们被有效地用作子混合总线。我们可以添加一个效果到一个信道组，如果信道被分配到那个信道组，所有的信道都会受到插入到一个信道组中的DSP的影响。  
然后可以嵌套和操纵这些通道组来创建分层混合。  

```
result = system->createChannelGroup("my channelgroup", &channelgroup);
result = channel->setChannelGroup(channelgroup);
```
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img007.png)

现在，我们可以看到新创建的通道组作为一个独立的DSP通道组Fader，位于右边的通道和左边的主通道组Fader之间。

#####  向ChannelGroup添加一个效果
向通道组添加效果与向通道添加效果相同。使用ChannelControl:: addDSP。  

```
FMOD::DSP *dsp_lowpass;
result = system->createDSPByType(FMOD_DSP_TYPE_LOWPASS, &dsp_lowpass);
result = channelgroup->addDSP(1, dsp_lowpass);
```
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img008.png)
我们现在可以像以前一样看到，一个附加在通道组Fader上的效果，在位置1中，通道组的整体由围绕2个单元的框表示。  


##### 创建一个效果，并使所有通道发送给它。
这个例子演示了一个更复杂、更典型的场景，在这个场景中，我们创建了一个新效果，每当一个声音在通道上播放时，我们就将新通道连接到效果上。

**重要提示!**请不要用这个例子作为设置混响的标准方法。只需调用System::setReverbProperties，就可以自动处理所有连接逻辑。注意，下面的逻辑不处理语音变为虚拟并从图中删除后返回的情况。通常情况下，只有当你想单独控制“湿”混合水平时，你才会使用这种逻辑。否则，一个简单de  ChannelControl::addDSP就足够了。

第一步是向主通道组添加一个效果。为此，我们再次调用System::createDSPByType，然后使用DSP API手动添加连接。


```
FMOD::DSP *dsp_reverb;
FMOD::DSP *dsp_tail;
FMOD::ChannelGroup *channelgroup_master;
result = system->createDSPByType(FMOD_DSP_TYPE_SFXREVERB, &dsp_reverb);             /* Create the reverb DSP */
result = system->getMasterChannelGroup(&channelgroup_master);                       /* Grab the master ChannelGroup / master bus */
result = channelgroup_master->getDSP(FMOD_CHANNELCONTROL_DSP_TAIL, &dsp_tail);      /* Grab the 'tail' unit for the master ChannelGroup.  This is the last DSP unit for the ChannelGroup, in case it has other effects already in it. */
result = dsp_tail->addInput(dsp_reverb);
```

这将导致    
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img009.png)

这就是频道将要播放的。在本例中，我们之所以在这里使用ChannelGroup，是为了让通道首先在图中执行，然后是混响。这就引出了一个名为“执行顺序”的话题，你可以在下面找到更多的信息，以及它对你是否重要。

意味着它是不活动/禁用的。所有的单位在默认情况下都是不活动的，所以我们必须激活它们。你可以做到这一点与DSP::setActive。


```
result = dsp_reverb->setActive(true);
```

![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img010.png)

现在你可以看到混响已经从黑色/非活动变为活动。代码播放暂停后的声音，得到其通道DSP头单元，将通道DSP头单元添加到混响中，然后停止声音。


```
FMOD::DSP *channel_dsp_head;
result = system->playSound(sound, channelgroup, true, &gChannel[0]);                /* Play the sound.  Play it paused so we dont hear the sound play before it is connected to the reverb. */
result = channel->getDSP(FMOD_CHANNELCONTROL_DSP_HEAD, &channel_dsp_head);          /* Grab the 'head' unit for the Channel */
result = dsp_reverb->addInput(channel_dsp_head);                                    /* Manually add a connection from the Channel DSP head to the reverb. */
result = channel->setPaused(false);                                                 /* Unpause the channel and let it be audible. */
```

注意，调用ChannelControl:: setpause在内部只调用DSP::setActive在通道的头部DSP单元上。
结果：

![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img011.png)
这里有趣的部分是，通道DSP头单元现在每个通道有2个输出，每组输出混合到用户创建的通道组，然后作为“dry”信号传递到输出。第二组输出可以被认为是“wet”路径，类似地混合到混响单元，然后由混响处理器处理。
 
控制DSPConnections的混合电平和pan矩阵
DSP单元之间的每个连接都由一个DSPConnection对象表示。这是两个盒子之间的线。

这个对象类型的主要目的是允许用户控制体积/混合2处理单元之间的水平,并控制扬声器/通道2单位之间的映射,可以批评,这样一个信号,映射到任何输出信号和输入信号,以任何方式,是必要的。

让我们回到上面的例子，但是使用一个声音，将它的混合声音从通道改为混响，从1.0 (0db)改为0.0 (-80db) 

围绕playsound的代码有一个不同之处，那就是addInput也将接受指向结果DSPConnection对象的指针。

```
FMOD::DSP *channel_dsp_head;
FMOD::DSPConnection *dsp_connection;
result = system->playSound(sound, channelgroup, true, &gChannel[0]);                /* Play the sound.  Play it paused so we dont hear the sound play before it is connected to the reverb. */
result = channel->getDSP(FMOD_CHANNELCONTROL_DSP_HEAD, &channel_dsp_head);          /* Grab the 'head' unit for the Channel */
result = dsp_reverb->addInput(channel_dsp_head, &dsp_connection);                   /* Manually add a connection from the Channel DSP head to the reverb. */
result = channel->setPaused(false);                                                 /* Unpause the channel and let it be audible. */
```

然后，我们可以使用DSPConnection::setMix简单地更新卷。

```
result = dsp_connection->setMix(0.0f);
```
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img012.png)

您可以看到，在混响的仪表中没有signal level，因为它的唯一输入是无声的。


#####  设定DSP单元的输出格式，控制输出信号的pan矩阵
在本节中，我们将从channel_dsp_head获取第一个输出，并对其应用pan矩阵，以允许将输入信号映射到混合音箱中的任何输出扬声器。
首先要注意的是，通道Fader向通道组Fader输出mono。这意味着这里没有太多可以映射的东西。任何表示这个信号的矩阵都是1进1出。

为了让它更有趣，我们可以用DSP::setChannelFormat来改变DSP单元的输出格式。


```
result = channel_dsp_head->setChannelFormat(0, 0, FMOD_SPEAKER_QUAD);
```
结果：    
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img013.png)  
您将注意到ChannelFader现在输出4个通道，并通过网络传播。Quad to 5.1 pan的默认upmix与mono to 5.1的默认upmix不同，因此您将看到，在最终的ChannelGroup Fader单元上，前端现在稍微低一些，环绕左右扬声器中现在引入了一些信号。现在，我们将使用一些代码来做一些有趣的事情，我们将把新的四频道阅读器信号的前2个频道放到四频道输出的后2个扬声器中。


```
FMOD::DSPConnection *channel_dsp_head_output_connection;
float matrix[4][4] =
{   /*                                    FL FR SL SR <- Input signal (columns) */
    /* row 0 = front left  out    <- */ { 0, 0, 0, 0 },     
    /* row 1 = front right out    <- */ { 0, 0, 0, 0 },     
    /* row 2 = surround left out  <- */ { 1, 0, 0, 0 },     
    /* row 3 = surround right out <- */ { 0, 1, 0, 0 }      
};
result = channel_dsp_head->getOutput(0, 0, &channel_dsp_head_output_connection);
result = channel_dsp_head_output_connection->setMixMatrix(&matrix[0][0], 4, 4);
```
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img014.png)   

我们现在可以看到前两个通道现在对输出保持静默，因为它们在矩阵中有0，其中前两个输入列映射到前两个输出列。
相反，前2个输入列有1，其中行映射到环绕左侧和环绕右侧的输出扬声器。

#####  绕过效果/禁用它。
要禁用效果，只需使用setBypass方法。下面的代码播放一个声音，添加一个效果，然后绕过它。

```
result = dsp_reverb->setBypass(true);
```
这样做的好处是不会像参数一样禁用所有输入单元，比如DSP::setActive，并允许信号不受影响地通过混响单元(不调用混响处理函数，从而节省CPU)。    
![image](https://www.fmod.com/docs/2.0/api/images/dspnet-img015.png)  

旁路混响用灰色表示。  
请注意，许多FMOD效果会自动绕过它们自己，在没有信号或检测到静默之后，保存CPU，并且效果的有效“尾巴”已经发挥出来。

#####  执行顺序和拉/不拉遍历

DSP图形的执行顺序从右到左，也从上到下。顶部的单位将先于底部的单位执行。   
有时不希望用户创建的效果执行通道的DSP单元，而不是它所属的通道组。这通常无关紧要，但有一种情况很重要，即如果用户在通道上调用ChannelControl::setDelay，或者在父通道组上调用ChannelControl::setDelay，以便在启动之前发出声音延迟。   
混响单元没有延迟的概念，因为它所延迟的时钟存储在它所属的信道组中。  
结果是，混响将拉信号和声音通过混响处理器，干路径仍然是无声的，因为它是在一个延迟状态。   
在上面的混响示例中，解决方法是在创建通道将播放的通道组之后将混响附加到主通道组，以便首先执行通道组，然后执行混响。  

#####  “发送”与“标准”连接类型

第二种方法是停止混响从输入中提取数据。这可以通过使用DSP::addInput的FMOD_DSPCONNECTION_TYPE 'type'参数来实现。如果使用FMOD_DSPCONNECTION_TYPE_SEND而不是FMOD_DSPCONNECTION_TYPE_STANDARD，则不会执行输入，而所有的混和将处理前面遍历到输入的混合内容。   
延迟将工作，但这种方法的缺点是，如果混响是第一个，从通道的信号将在混响处理后发送。这意味着它必须等到下一次混响才能处理该数据，因此混响引入了一个延迟块。  
