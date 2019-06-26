<style>
img {
	max-width:400px;
}
</style>

# iOS音频之Audio Unit & AudioEngine


### 概要
*AudioUnit是iOS最底层音频框架，它为操作音频提供了最灵活的控制度，最好的性能.有两种API提供了操作AudioUnit的功能，AudioUnit相关API和AUGraph相关API*
 
*两者相比，AUGraph为audio unit管理添加了线程安全功能，可以让你随时重新配置处理链，所以除非必须直接使用AudioUnit API否则就去使用AUGraph提供的API*
 
*由于AUGraph已经被废弃,替代而来的AVAudioEngine就值得我们去学习，*
 
*AudioEngine提供了与AUGraph不一样，AVAudioEngine提供了Objc/swift的API，它更像是AudioQueueService和Audio Processing Service的混合*



### Audio Unit概述
![Audio frameworks in iOS](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/AudioUnitHostingGuide_iOS/Art/Audio_frameworks_2x.png)

Audio units框架为操作音频提供了最灵活的控制度，最好的性能。
当你需要如下功能时：

* 同时低延迟的处理音频I/O,例如一个VoIP应用
* 综合的响应回放音乐，例如音乐游戏/综合乐器
* 用到回音消除、混音、均衡等特定特性的audio unit
* 一个能够让你组装处理模块到一个灵活网络的链式处理架构。（只有iOS提供了这个能力）

才应该去使用audio unit框架。
 
#### iOS中Audio unit的类型

| 用途 | Audio units | 说明 |
---------|----------------|---------------|
| Effect(效果) | iPod Equalizer(均衡) |提供了bass Booster、Pop、Spken Word等一系列均衡曲线
| Mixing(混响) |3D Mixer<br />Multichannel Mixer|如果只是想用3D Mixer最好用OpenAL
| I/O    | Remote I/O<br/>Voice-Processing I/O<br/>Generic Output| Remote I/O audio unit了连接了音频输入、输出硬件
|Format conversion(格式转换)|Format Converter|提供格式转换

### audio unit的结构
![Audio unit scope and elements](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/AudioUnitHostingGuide_iOS/Art/audioUnitScopes_2x.png)


* **scope**:audio unit的编程上下文(context)
* **element**:一个嵌套在scope的编程上下文。当一个element是输入/输出scope的一部分，它就和物理音频设备的数据总线类似，因此有时候也把element称为总线（bus）。

* **gloal scope**：只有element 0是global的。

上图描述了一个通用的audio unit架构，它的输入输出element数量相同。然而一些复杂的audio unit使用更加复杂的架构。比如，mix unit有多个输入element但只有一个输出element。

##### I/O unit的必要特征
一个I/O unit确定有两个element（只有I/O确定是两个）

![I/O unit的架构](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/AudioUnitHostingGuide_iOS/Art/IO_unit_2x.png)

尽管这两个element属于一个audio unit，但是你的app把它们视为不同的实体。例如，你可以通过（(kAudioOutputUnitProperty_EnableIO）属性独立的启用或禁用这两者之一。

element 1直接连接着设备的音频输入硬件，在上图中是麦克风。element 1的输入scope对你是不透明的，但你可以从element 1的输出域（scope）访问到进入硬件的音频数据。
与此类似，element 0直接连接输出硬件，上图中是话筒。你可以输入音频数据到element 0的输入域，但是它的输出域是不透明的

##### 使用Audio Processing Graph管理Audio Units
AUGraph：是你创建和管理audio unit处理链的对象，一个音频处理图可以利用多个audio unit和多个渲染回调函数的能力实现几乎任何你所想到的功能。

AUGraph为audio unit管理添加了线程安全功能，可以让你随时重新配置处理链。例如：你可以在播放音频的过程中安全的添加一个均衡器，甚至是为一个混响输入换一个完全不同的回调函数。只有iOS的API提供了可以动态配置的能力。

AUNode：代表一个单独的audio unit在图中的上下文。当使用图的时候，你经常要直接和node交互并把他们作为audio unit的代理，而非直接与audio unit交互。

在连接Node时，你必须使用audio unit API去配置每个audio unit。

你可以使用AUNode 实例作为复杂图中的一个元素，通过定义这个节点代表一个完整的对象处理子图（audio processing subgraph）。在这种情况下，在在图最末尾的I/O unit 必须是通用输出unit（Generic Output）一个不连接任何硬件的unit。

广泛来说，建立一个音频处理图需要三个任务。

1. 添加node到图
2. 直接配置node的audio unit
3. 连接这些节点


### Audio Unit API
两种API：Audio Unit、AUGraph
这两种API都提供了以下功能

* 引用audio units定义的动态连结库（只有系统提供的audio unit才可以）
* 实例化audio units
* audio unit的相互连接 & 添加render callback函数
* 启动、停止音频图（audio flow）

#### 使用标识符指定和获取audio unit
使用AudioComponentDescription的type、subtype、manufacturer查找相应的audio unit

	AudioComponentDescription ioUnitDescription;
 	
 	ioUnitDescription.componentType          = kAudioUnitType_Output;
	ioUnitDescription.componentSubType       = kAudioUnitSubType_RemoteIO;
	ioUnitDescription.componentManufacturer  = kAudioUnitManufacturer_Apple;
	ioUnitDescription.componentFlags         = 0;
	ioUnitDescription.componentFlagsMask     = 0;

所有的iOS audio unit使用 kAudioUnitManufacturer_Apple作为componentManufacturer的值

把type或subtype中一者或一起设置为0，将创建一个统配的description

获取一个audio unit实例

	AudioComponent foundIoUnitReference = AudioComponentFindNext (
                                          NULL,
                                          &ioUnitDescription
                                      );
	AudioUnit ioUnitInstance;
	AudioComponentInstanceNew(
   	   foundIoUnitReference,
       &ioUnitInstance
	);

使用AUGraph API获取audio unit 实例

	// Declare and instantiate an audio processing graph
	AUGraph processingGraph;
	NewAUGraph (&processingGraph);
 
	// Add an audio unit node to the graph, then instantiate the audio unit
	AUNode ioNode;
	AUGraphAddNode (
    	processingGraph,
    	&ioUnitDescription,
    	&ioNode
	);
	AUGraphOpen (processingGraph); // indirectly performs audio unit instantiation
 
	// Obtain a reference to the newly-instantiated I/O unit
	AudioUnit ioUnit;
	AUGraphNodeInfo (
   		processingGraph,
    	ioNode,
    	NULL,
    	&ioUnit
	);

##### 使用属性配置audio unit

	UInt32 busCount = 2;
 
	OSStatus result = AudioUnitSetProperty (
    	mixerUnit,
    	kAudioUnitProperty_ElementCount,   // the property key
    	kAudioUnitScope_Input,             // the scope to set the property on
    	0,                                 // the element to set the property on
    	&busCount,                         // the property value
    	sizeof (busCount)
	);


### 构建Audio Unit App
 
#### 选择设计模式
Audio Unit中有6种基本的设计模式，它们的共同点为：

* 只有一个I/O unit
* 始终在Audio progcessing graph中使用一种音频流格式，结合他们的格式可能变化，就像单声道和立体声流feed一个混音unit
* 需要在特定位置设置一部分或所有的流格式

大部分情况下使用AGGraph能够精简代码、并支持动态重配置

##### I/O Pass Through:直接将输入音频发送到输出硬件上，不做任何处理

![同时I/O通过](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/AudioUnitHostingGuide_iOS/Art/IOPassThrough_2x.png)

如图，两边外侧的Remote I/O硬件强制设置了的流格式，你可以设置App的流格式，如果格式不同的话audio unit会进行转换，为了避免转换带来的损耗，请使用硬件的采样率。

##### I/O Without a Render Callback Function：

![没有渲染回调函数情况下同时处理I/O](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/AudioUnitHostingGuide_iOS/Art/IOWithoutRenderCallback_2x.png)

在这种模式，remote I/O unit的每个element被设置的和pass though模式一模一样。为了设置 Multichannel Mixer unit,你必须设置混音unit的输出采样率。

混音器的输入流格式通过高Remote I/O unit的输入element的输出域的传播被自动设置，同样Remote I/O unit的输出element的输入域的流格式也被mixer的输出域自动设置

##### I/O with a Render Callback Function

![有渲染回调函数情况下同时处理I/O](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/AudioUnitHostingGuide_iOS/Art/IOWithRenderCallback_2x.png)

##### 其他的Audio unit设计模式
input-only app with a render callback function.
 offline audio processing

#### 构建audio unit app的步骤

##### 配置audio session

	NSError *audioSessionError = nil;
	AVAudioSession *mySession = [AVAudioSession sharedInstance];     
	[mySession setPreferredHardwareSampleRate: graphSampleRate       
	                                    error: &audioSessionError];
	[mySession setCategory: AVAudioSessionCategoryPlayAndRecord      
	                                    error: &audioSessionError];
	[mySession setActive: YES                                       
	               error: &audioSessionError];
	self.graphSampleRate = [mySession currentHardwareSampleRate];
	
另一个硬件你可能希望配置的特性是： audio hardware I/O buffer duration。44.1kHz采样率下的默认值是23ms，相当于一片1024个采样。如果你的app要求更低的延迟，你可以设置最小为0.005ms（相当于256个样）

	self.ioBufferDuration = 0.005;
	[mySession setPreferredIOBufferDuration: ioBufferDuration
                                  error: &audioSessionError];

##### 指定需要的audio units
在运行时，在你配置audio session配置的代码运行之后，你的app还没有获得audio units。你需要为每一个指定一个AudioComponentDescription结构体。

##### 创建音频处理图，获取其中的audio unit

在这一步，你创建一个本章节中描述的设计模式：

  1. 实例化一个AUGraph的不透明类型，这个实例相当于一个音频处理图
  2. 实例一个或多个AUNode不透明类型，其中的每一个相当于图中一个audio unit
  3. 添加node到图中
  4. 开启图，并实例化audio units
  5. 获取audio unit的引用

下面的清单展示了一个如何为一个包含一个Remote I/O unit和一个Multichannel unit的图执行上述步骤。它假定你已经为每一个audio units定义AudioComponentDescription结构体
 
 
	AUGraph processingGraph;
	NewAUGraph (&processingGraph);
 
	AUNode ioNode;
	AUNode mixerNode;
 
	AUGraphAddNode (processingGraph, &ioUnitDesc, &ioNode);
	AUGraphAddNode (processingGraph, &mixerDesc, &mixerNode);

*AUGraphAddNode* 函数使用了 *ioUnitDesc* 、*mixerDesc*的*AudioComponentDescription*。
在这个时候图实例化并拥有了你将要在app中用到的节点。要开启图并实例化audio units，请调用 *AUGraphOpen*

	AUGraphOpen (processingGraph);

然后，使用*AUGraphNodeInfo*获取到被引用的audio unit 实例，如下所示：

	AudioUnit ioUnit;
	AudioUnit mixerUnit;
 
	AUGraphNodeInfo (processingGraph, ioNode, NULL, &ioUnit);
	AUGraphNodeInfo (processingGraph, mixerNode, NULL, &mixerUnit);

*ioUnit*和*mixerUnit*变量持有了audio unit实例的引用，并允许你配置、连接这些audio unit

##### 配置audio unit

每一个iOS的audio unit需要他自己的配置，在使用特定的audio unit中有描述。然而，有一些通用的配置是每一个iOS音频开发者应当熟悉的。

Remote I/O unit，默认它的输出开启但是输入被禁用。如果你的app需要同时进行I/O，或者只是用输入，你就必须重新配置I/O unit。更详细的请看，[Audio Unit Properties Reference](https://developer.apple.com/documentation/audiounit/audio_unit_properties)的*kAudioOutputUnitProperty_EnableIO*属性.

所有的除了remote I/O 和 Voice-Processing I/O units 之外的iOS audio unit，都需要配置*kAudioUnitProperty_MaximumFramesPerSlice*属性。这个属性确保在响应渲染调用时，能够产生足够数量的音频数据帧。更详细的请看，[Audio Unit Properties Reference](https://developer.apple.com/documentation/audiounit/audio_unit_properties)的kAudioUnitProperty_MaximumFramesPerSlice。

所有的audio unit需要定义它们的输入、输出音频流格式。

##### 写入并添加一个渲染callback函数

对于那些使用可渲染回调函数的设计模式，你必须写写着函数，并把它们固定到正确点上。

立即设置渲染回调函数

	AURenderCallbackStruct callbackStruct;
	callbackStruct.inputProc        = &renderCallback;
	callbackStruct.inputProcRefCon  = soundStructArray;
 
	AudioUnitSetProperty (
    	myIOUnit,
    	kAudioUnitProperty_SetRenderCallback,
    	kAudioUnitScope_Input,
    	0,                 // output element
    	&callbackStruct,
    	sizeof (callbackStruct)
	);

你可以通过使用音频处理图API，以线程安全的方式添加一个渲染回调，即使audio正在流动。
	AURenderCallbackStruct callbackStruct;
	callbackStruct.inputProc        = &renderCallback;
	callbackStruct.inputProcRefCon  = soundStructArray;
 
	AUGraphSetNodeInputCallback (
    	processingGraph,
    	myIONode,
    	0,                 // output element
    	&callbackStruct
	);
	// ... some time later
	Boolean graphUpdated;
	AUGraphUpdate (processingGraph, &graphUpdated);


##### 连接audio unit 节点
可以使用AUGraph的AUGraphConnectNodeInput & AUGraphDisconnectNodeInput 线程安全的建立/移除连接<br />
也可以用 AudioUnitSetProperty的
 kAudioUnitProperty_MakeConnection属性，这种方式需要为每个连接定一个AudioUnitConnection结构体
 
 使用图建立连接
 
	AudioUnitElement mixerUnitOutputBus  = 0;
	AudioUnitElement ioUnitOutputElement = 0;
 
	AUGraphConnectNodeInput (
	    processingGraph,
	    mixerNode,           // source node
	    mixerUnitOutputBus,  // source node bus
	    iONode,              // destination node
	    ioUnitOutputElement  // desinatation node element
	);
	
##### 初始化并启动音频处理图
在启动音频流之前，必须要先调用AUGraphInitialize函数。关键点包括：
* 调用AUGraphInitialize函数自动初始化它所持有的audio unit。（如果没有使用AUGraph创建处理链，就需要自己一一将audio unit初始化）
* 校验图的连接和音频流的格式
* 在audio unit中传播音频流格式

# AVAudioEngine

### 组件

**Engine (AVAudioEngine)** <br/>
**Node (AVAudioNode)**

* Output node (AVAudioOutputNode)
* Mixer node (AVAudioMixerNode)
* Player node (AVAudioPlayerNode)

引擎管理节点图，使用引擎设置节点之间的连接，允许动态节点配置

### 工作流

1. 创建引擎
2. 创建节点
3. 将节点添加到引擎
4. 连接这些节点
5. 开启引擎

### AVAudioNode
* Source—Player, microphone（麦克风）
* Process—Mixer, effect（音效）
* Destination—Speaker

N 输入/M 输出, 被称为总线（buses）
每一个总线有其音频格式

### 连接节点
![](https://ws3.sinaimg.cn/large/005BYqpggy1g32jq8sbfij31n60e4q4j.jpg)

### AVAudioOutputNode

Engine有一个名为output node的隐式目标节点

输出节点将音频数据提供给输出硬件

无法创建独立实例

### AVAudioInputNode class
Engine有一个隐式输入节点

输入节点从输入硬件接收音频数据

数据在活动连接链中拉出

无法创建独立实例

### AVAudioMixerNode
Engine具有隐式混音器节点

可以创建附加的混音节点并将其连接到引擎

混音器输入可以具有不同的音频格式

![](https://ws3.sinaimg.cn/large/005BYqpggy1g32jyfc8v7j31ru0u0gpb.jpg
)

### AVAudioPlayerNode

playerNode从文件和缓冲区播放数据

调度事件 - 在指定时间播放数据

调度缓冲区

 - 具有单独回调的多个缓冲区
 - 循环的单个缓冲区
 
调度文件和文件段
### Node tap-从Node中获取输出的音频buffer
```
[mixer installTapOnBus:0 bufferSize:4096 format:[mixer outputFormatForBus:0]
		block:^(AVAudioPCMBuffer *buffer, AVAudioTime *when) {
 // 存储或其他操作
	}];
```

![](https://ws3.sinaimg.cn/large/005BYqpggy1g32jyfc8v7j31ru0u0gpb.jpg
)

### Effect Nodes

| AVAudioUnitEffect | AVAudioUnitTimeEffect |
---------|----------------|---------------|
N frames in, N frames out| X frames in, Y frames out
不能被连接到input Node| 不能被连接到input Node

| AVAudioUnitEffect | AVAudioUnitTimeEffect |
-----|---
AVAudioUnitDelay | AVAudioUnitVarispeed
AVAudioUnitDistortion | AVAudioUnitTimePitch
AVAudioUnitEQ | 
AVAudioUnitReverb |

# 创建音频流播放器
 
 因为AVAudioEngine像Audio Queue Services 和Audio Unit Processing Graph Services 的混合，所以我们可以结合我们之前学到的创建一个像队列一样调度音频播放器，但是也支持像audio graph的实时效果。
 

 
 ![播放器整体的架构图](https://ws3.sinaimg.cn/large/005BYqpggy1g325oaq229j314v0u0mz9.jpg)
 
 这里是播放器的组件拆解：
 
 1. 从互联网下载音频数据。我们知道我们需要从某个地方提取原始音频数据。只要我们以二进制格式接收音频数据（即Swift 4中的数据），我们如何实现下载器并不重要。
 
 2. 将二进制音频数据解析为音频数据包。为此，我们将使用经常令人困惑但非常棒的音频文件流服务API。
 
 3. 将解析后的音频数据包读入LPCM音频数据包。要处理所需的任何格式转换（特别是压缩到未压缩），我们将使用音频转换器服务API。
 
 4. 使用AVAudioEngine通过将它们安排到引擎头部的AVAudioPlayerNode上来流式传输（即回放）LPCM音频数据包。
