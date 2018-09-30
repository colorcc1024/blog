原文出处：<https://code.fb.com/android/improving-facebook-s-performance-on-android-with-flatbuffers/>

在Facebook上，用户可以通过阅读状态更新或者浏览照片的方式来保持跟家人，朋友的联系。在服务后端，我们存储了所有的数据来生成这些联系的社交图谱。但在移动端，我们无法下载全部的图谱，所以我们下载一个节点和它所对应的联系信息在本地以树形结构表示。

下图说明了一个带有图片附件的故事是如何工作的。在这个例子中，John创建了故事，然后他的朋友们赞了这个故事并且评论了这个故事。左手边的社交图谱描述了Facebook的后端是如何表示这些关系的。当Android应用请求这个故事，会返回一个包含了角色、反馈和附件等信息的树形结构。

![Alt Image Text](https://github.com/songzeyang99/blog/blob/master/pic/fb1 "Optional Title") 

我们处理的一个关键问题是在应用中如何展示和存储这些数据。如果将这些数据规范化的存储在SQLite数据库中的各个表中这是不现实的，因为我们从后端有太多不同的方式查询节点和关联的属性结构。所以最终我们的想法就是直接将整个属性结构存储下来。其中一种方案是用JSON来存储树形结构，这就要求我们在数据填充UI页面之前将JSON解析、变换成Java对象，另外，解析JSON也要消耗一定时间。我们使用过Jackson JSON解析器来解析JSON但是发现了几个问题：

* 解析速度  解析一个20KB大小的JSON数据流消耗了35ms，这超过了UI 16.6ms的刷新频率。使用这种方式，我们就不能从磁盘缓存中按需加载故事，这会导致用户在滑动屏幕是丢帧（视觉上断断续续）。
* 解析序列化  JSON解析器开始解析之前需要构建字段映射，这会花费100ms到200ms，大大的增加了应用的启动时间。
* 垃圾搜集  在JSON解析期间会产生大量的小对象，在我们的研究中，解析20KB的JSON数据会产生100KB的瞬时内存，这会给java垃圾收集器带来巨大的压力

我们想要寻找一种更高效的存储格式来提升Android App的体验。

### FlatBuffers

在我们寻找更好的存储格式时，我们发现了FlatBuffers，它来自Google的一个开源工程。FlatBuffers是protocol buffers的一个演进格式，它包含了数据的原信息，可以直接访问数据的子对象而不用预先反序列化整个对象。

想象一下我们有一个person类对象，它有4个属性：名字，友谊状态，配偶，朋友列表。其中配偶和朋友字段也包含person对象，同样是属性结构。
下面简单的插图说明了person对象（John和他的妻子Mary）在FlatBuffer中是怎样被展示的。

```
class Person {
    String name;
    int friendshipStatus;
    Person spouse;
    List<Person>friends;
}
```


![Alt Image Text](https://github.com/songzeyang99/blog/blob/master/pic/fb2 "Optional Title") 



根据上面的结构，你会意识到：

* 每一个对象被分割成了2部分：以中轴点为基准（4）左边是元数据，右边是真实的数据。
* 在元数据部分每一个属性对应一个卡槽，它的值代表这个字段的真实数据相对中轴点的偏移量。举个例子，左边元数据的第一个卡槽是1，它表示John的名字存储位置在距离中轴点1个字节的距离。
* 对于其他字段，元数据中的值（偏移量）指向子对象的中轴点。举个栗子，在John的元数据中第3个卡槽的值指向的是他老婆Mary的中轴点。
* 至于那么null值，我们可以在卡槽中用0表示。

下面的代码片段展示了如何在上面的结构中查找John的老婆（Mary）的名字

```
// Root object position is normally stored at beginning of flatbuffer.
int johnPosition = FlatBufferHelper.getRootObjectPosition(flatBuffer);
int maryPosition = FlatBufferHelper.getChildObjectPosition(flatBuffer,
   johnPosition, // parent object position
   2 /* field number for spouse field */);
   
String maryName = FlatBufferHelper.getString(flatBuffer,
   johnPosition, // parent object position
   2 /* field number for name field */);
```

上面的过程并没有涉及到中间对象的创建，减少了瞬间内存的分配。我们可以通过将FlatBuffer的数据直接存储到文件中并将它映射到内存的方式来进一步优化。这意味着我们只需要按需加载加载文件中数据，这大大的减少了整体内存的占用。而且在读取某个字段前也不用反序列化对象，这缩短了存储层到UI层的延迟，提高了应用的整体体验。


### FlatBuffers变种

有时我们需要修改FlatBuffers里面的值，但FlatBuffers设计之初就被设定成不可更改。为此我们想到了一个解决办法，紧密的监测原始FlatBuffers的变化。

* John的友谊状态被标记在FlatBuffer的元数据中索引为2的位置。为了更改这个值，我们只需要记录索引为2的值是1
* Mary的名字（“Mary”）被标记在元数据中索引为13的位置。同样的，为了修改Mary的名字我们只需要将新的名字对应到索引为13位置就好了。

最后，我们将所有的变种打包到变种数据中去。变种数据由2部分构成：变种索引和变种数据。变种索引记录了一种映射关系，键是原FlatBuffers中索引，值是修改数据内容的索引。变种数据存储着和FlatBuffer一样格式的修改数据的内容。

![Alt Image Text](https://github.com/songzeyang99/blog/blob/master/pic/fb3 "Optional Title")


当查询FlatBuffers中的一段数据时，我们可以找出数据的绝对位置，然后在变种数据中查看这段数据是否有修改，如果有修改则将修改的数据返回，否则的话就返回原始数据。


### Flat模型

FlatBuffers不仅仅可以用于存储模型还可以应用在网络、内存模型，它消除了服务器响应到UI显示之间的数据转换。正因为如此一个以Flat模型为导向的应用架构诞生了，它可以消除UI层和数据存储层之间的复杂性。

之前我们用JSON作为存储数据的结构时，我们需要添加内存缓存来解决反序列化所带来的体验问题，并且最终我们还要添加应用逻辑、网络逻辑在UI层和存储层之间。类似这样

![Alt Image Text](https://github.com/songzeyang99/blog/blob/master/pic/fb4 "Optional Title")



虽然这种3层架构在iOS和桌面平台很流行，但是在Android平台却有些问题需要解决：

* 添加内存缓存意味着我们要消耗更多的内存（相比UI层展示数据所需要的内存）。市面上大多数的设备依然还是为每个app分配48M或者更少的内存空间。当然内存缓存中的内存占用量超过了Java垃圾回收器的限制，就会对应用体验造成影响。
* 应用逻辑里需要处理内存缓存、UI和存储等相关事务，开发者通常会将UI和存储放在不用的线程进行处理。所以在大型App中保持简单的线程模型也是很困难的。
* UI层一般会接受不同来源的数据，比如缓存数据，网络数据，本地数据的变种等。这就需要UI层需要处理不同种类的数据变化从而会导致UI的过度绘制。

Flat模型出现后，UI层和存储层就可以很轻松的交互了，如下结构所示。


![Alt Image Text](https://github.com/songzeyang99/blog/blob/master/pic/fb5 "Optional Title")


* UI构建在存储层的上层，采用Android标准cursor进行交互。这种方式在当下众多app中是很流行的，提升UI响应。
* 应用逻辑和网络组件被搬到了存储层的下面，所用的逻辑都可以通过后台线程处理并保证将结果第一时间通知给存储层。存储层通知UI层进行重新绘制。
* 这种架构将UI层和应用逻辑成清晰的分隔开来，并且我们还可以对每一个逻辑进行简化。UI层只需要影响存储层的数据变化，而应用逻辑层只需要将正确的数据写入到存储层中。UI层和应用逻辑层在不同的线程上工作，彼此之间也不会直接通信。

### 结论

FlatBuffers是一种数据格式，作用是消除存储层和UI层之间的数据转换。使用期间，我们还推动像Flat Models这样的架构来提升我们的应用体验。基于FlatBuffers扩展我们还可以在一个数据结构中监测服务数据、变种数据和本地状态，简化了我们的数据模型，并且可以只提供一套API给UI组件。

在过去的六个月中，我们推动Android平台的Facebook产品将FlatBuffers作为存储格式。下面是一些提升的数据结果：
* 从磁盘缓存加载故事消耗的时间从35ms缩短至4ms
* 瞬时内存的分配降低了75%
* 应用启动速度提升了10-15%
* 整体的存储空间减少了15%

数据格式的变化让人们可以花更多的时间来阅读朋友的状态，浏览家人的照片，这是一件令人兴奋的事。谢谢你，FlatBuffers！