# 冲顶大会有前端什么事吗？




最近的冲顶大会只是一个知识问答的模式，不过结合直播起来可能就带来不一样的效果了。总的来看主要火起来的有几个基本点：

 - 低门槛：人人都可以答题
 - 奖金高：差不多都是百万以上，但是不排除机器人和你分钱 :)
 - 延时低：同时出题，同时答题。这点对于延时直播来说，压力非常大，差不多延时必须控制到 2s 之内。所以，HLS 基本上就凉了。
 - KOL 带节奏，比如王思聪 撒币
 - 主播美眉
 - 。。。

作为一个前端开发来说，我觉得直播的世界离我们太遥远，其中有两座大山，一个是 X5 ，一个是 苹果。因为，X5 和苹果这逼不提供 MSE 给前端用，导致了一个结果，要么用 HLS 直播，要么你就不播。现在，鄙人正在腾讯打工，最近和 X5 杠上了，问他们为什么不支持 MSE？他们给的答复是：

![image.png-47.2kB][1]

可以说，在明年，MSE 技术应该会让前端能力得到极大的扩展。但是，MSE 只是作为一个播放端的技术，流从哪里来？H5 直播是怎么搞的？

对于一些基础知识，推荐大家直接去这篇文章里面查看：

[ https://www.villianhr.com/2017/03/31/全面进阶 H5 直播][2]

现阶段，比较流行的协议主要有：HLS，RTMP，HTTPFLV。延时性就属 HLS 最菜，这是苹果自研直播流播放器，差不多播放延时有 10s+ 左右。而 RTMP 和 HTTPFLV 都和 Adobe 公司有点关系，这有种让人离不开 Flash 的感觉，他们俩的延时性都出奇的好，差不多只有 3s 左右，但是里面流确实 FLV 格式的。

但是，W3C 不同意，老子绝壁不想和 Flash 再有半点关系，我现在有 WebM，FMP4，TS 凭啥再去迁就你的 FLV？

为了从源头解决这个问题，MPEG 推出了 MPEG-DASH 直播标准来统一各种比较尴尬的流描述文件。它主要是基于 mpd 文件来做的切片和文件的 download。先来看一份简单的 MPEG-DASH 文件：

```
<?xml version="1.0" encoding="utf-8"?>
<MPD xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="urn:mpeg:dash:schema:mpd:2011" xsi:schemaLocation="urn:mpeg:dash:schema:mpd:2011 DASH-MPD.xsd" profiles="urn:mpeg:dash:profile:isoff-live:2011,http://dashif.org/guidelines/dash-if-simple" maxSegmentDuration="PT2S" minBufferTime="PT2S" type="static" mediaPresentationDuration="PT10S">
   <ProgramInformation>
      <Title>Media Presentation Description from DASHI-IF live simulator</Title>
   </ProgramInformation>
   <BaseURL>http://vm2.dashif.org/dash/vod/testpic_2s/</BaseURL>
   <Period id="precambrian" start="PT0S">
      <AdaptationSet contentType="audio" mimeType="audio/mp4" lang="eng" segmentAlignment="true" startWithSAP="1">
         <Role schemeIdUri="urn:mpeg:dash:role:2011" value="main"/>
         <SegmentTemplate startNumber="1" initialization="$RepresentationID$/init.mp4" duration="2" media="$RepresentationID$/$Number$.m4s"/>
         <Representation id="A48" codecs="mp4a.40.2" bandwidth="48000" audioSamplingRate="48000">
            <AudioChannelConfiguration schemeIdUri="urn:mpeg:dash:23003:3:audio_channel_configuration:2011" value="2"/>
         </Representation>
      </AdaptationSet>
      <AdaptationSet contentType="video" mimeType="video/mp4" segmentAlignment="true" startWithSAP="1" par="16:9" minWidth="640" maxWidth="640" minHeight="360" maxHeight="360" maxFrameRate="60/2">
         <Role schemeIdUri="urn:mpeg:dash:role:2011" value="main"/>
         <SegmentTemplate startNumber="1" initialization="$RepresentationID$/init.mp4" duration="2" media="$RepresentationID$/$Number$.m4s"/>
         <Representation id="V300" codecs="avc1.64001e" bandwidth="300000" width="640" height="360" frameRate="60/2" sar="1:1"/>
      </AdaptationSet>
   </Period>
</MPD>

```

基本原理是，后台将一份完整的流文件切片，生成 Initial Segment 和 Media Segments，这些就是流的片段文件，比如 `.mp4`，`.ts`等。更详细的内容，下面会进行详述。

那该协议的延时性怎么样呢？

这个主要取决于你的协议，最极致的时延是比 HTTP-FLV 还要快，差不多 <= 2~3s。其中，最主要的是它定义的模板解析机制，直接根据时间戳来协定。
 

## MPD 基本简介

DASH 其实只是一系列概念，原意是 `Dynamic Adaptive Streaming over HTTP`。它主要是基于 mpd 文件来做的切片和文件的 download。整个模式有点类似于 HLS，但是其可以应用于所有的 视频格式，比如 mp4，webm。它将整个的视频文件，切片分为一个个具体的 HTTPFLV segments。不过，其更优于 HLS，这个已经在上面阐述清楚了。

MPD 的播放模式，其实就是根据 XML 的内容，协商出来播放的切片 URL 地址。一个简易的 MPD 文件为：

```
<?xml version="1.0" encoding="utf-8"?>
<MPD availabilityStartTime="1970-01-01T00:00:00Z" id="Config part of url maybe?" maxSegmentDuration="PT2S" minBufferTime="PT2S" minimumUpdatePeriod="P100Y" profiles="urn:mpeg:dash:profile:isoff-live:2011,http://dashif.org/guidelines/dash-if-simple" publishTime="2018-01-06T06:57:00Z" timeShiftBufferDepth="PT5M" type="dynamic" ns1:schemaLocation="urn:mpeg:dash:schema:mpd:2011 DASH-MPD.xsd" 
      xmlns="urn:mpeg:dash:schema:mpd:2011" 
      xmlns:ns1="http://www.w3.org/2001/XMLSchema-instance">
      <ProgramInformation>
            <Title>Media Presentation Description from DASHI-IF live simulator</Title>
      </ProgramInformation>
      <BaseURL availabilityTimeOffset="10.000000">https://vm2.dashif.org/livesim-dev/ato_10/testpic_2s/</BaseURL>
      <Period id="p0" start="PT0S">
            <AdaptationSet contentType="audio" lang="eng" mimeType="audio/mp4" segmentAlignment="true" startWithSAP="1">
                  <Role schemeIdUri="urn:mpeg:dash:role:2011" value="main" />
                  <SegmentTemplate duration="2" initialization="$RepresentationID$/init.mp4" media="$RepresentationID$/$Number$.m4s" startNumber="0" />
                  <Representation audioSamplingRate="48000" bandwidth="48000" codecs="mp4a.40.2" id="A48">
                        <AudioChannelConfiguration schemeIdUri="urn:mpeg:dash:23003:3:audio_channel_configuration:2011" value="2" />
                  </Representation>
            </AdaptationSet>
            <AdaptationSet contentType="video" maxFrameRate="60/2" maxHeight="360" maxWidth="640" mimeType="video/mp4" minHeight="360" minWidth="640" par="16:9" segmentAlignment="true" startWithSAP="1">
                  <Role schemeIdUri="urn:mpeg:dash:role:2011" value="main" />
                  <SegmentTemplate duration="2" initialization="$RepresentationID$/init.mp4" media="$RepresentationID$/$Number$.m4s" startNumber="0" />
                  <Representation bandwidth="300000" codecs="avc1.64001e" frameRate="60/2" height="360" id="V300" sar="1:1" width="640" />
            </AdaptationSet>
      </Period>
</MPD>
```

XML 文件里面的每一个标签内容，都是遵循 MPEG-DASH LA 的标准。其内部包含 N+1 个 Periods Tag。每一个 Period 包含相关的 media stream，比如 video：不同的编码，视角，带宽等、 audio：不同语言，类型，带宽等。一些共同的参数，比如 codec，frame rate, audio-channel 在同一个 Period 是不能改变的。所以，尽可能在一个 Period 提供更丰富的描述信息来说是非常重要的。虽然，Period 里面的信息不能动态改变，但是，用户可以自己手动选择，自己想要的分辨率，带宽大小等等。

Period 可以包含多个流，所以，还可以提供插入广告，或者视角流的切分等功能。这个也是 MPEG-DASH 设计的初衷，通过一份文件协商出多个媒体流的内容。其基础架构内容为：

![此处输入图片的描述][3]

这里，我们先简单介绍一下，几个重点 TAG 具体的内容。

### MPD

该是 MPD 里面文件的最外层的  Tag，有相关的属性来对其进行描述。该标签里面的属性极为重要，它决定了该 MPD 描述的文件属性和 媒体流 的播放顺序和内容。

> 再说三遍：
> MPD 文件非常重要!
> MPD 文件非常重要!
> MPD 文件非常重要!

其基本属性为：

![image.png-127.6kB][4]

接下来，我们一个一个简单介绍一下：

 - id: 设置 MPD 的 identifier，一般不需要。
 - profiles: 设置 MPD 的基本标准，具体内容，参考下表 profles。

![此处输入图片的描述][5]

 - type: 用来设置 MPD 文件的基本属性。取值有两个
    - static：segment 的时长需要在 @availabilityStartTime 和 @availabilityEndTime 之间。
    - dynamic：主要根据 availabilityStartTime 时间决定。
 - availabilityStartTime: 设置所有 segment 的参考时间，如果 type 为 dynamic，则该属性是必须的。
 - availabilityEndTime: 对于 static 文件来说的静态文件范围。
 - pulishTime: 设置 MPD 文件生成的绝对时间（wall-clock）

 - mediaPresentationDuration: 设置当前 mediaSegments 的总时间长。例如 `PT20M`，表示时长为 20min。当 MPD.minimumUpdatePeriod 和 Period.duration 没有设置时，该属性可以作为替代。

 - minimumUpdatePeriod: 设置当前 MPD 文件的更新时间。当 type = static 时，该属性不应该出现。

 - minBufferTime：用来设置最小 segment 时长，例如：`PT2S`。该通常和 `Representation.bandwidth(bps)` 一起使用，来计算最小 segment 大小值。
 - timeShiftBufferDepth: 用来设置 MPD 中 segment 的有效区间。或者可以理解为 过期时间。这里，后面我们会详述一下。
 
 - maxSegmentDuration: 设置最大 segment 时长。

上面我们已经了解 MPD 标签里面的基本属性，这些属性在整个 MPEG-DASH 里面非常重要，后面，我们将简单讲解一下关于 MPD 更新和文件过期的点。

### MPD 文件更新机制

MPD 文件的更新主要取决于 @minimumUpdatePeriod 属性设置，例如：`minimumUpdatePeriod="PT10S"`。用来标识，MPD 文件从上一次 MPD 获取时间开始，10s 后检查一次更新。如果没有该属性，或者为 0，则代表该 MPD 文件不会过期。整个更新逻辑为：

![image.png-97.2kB][6]

那当 MPD 文件发生更新时，有一些内容需要注意：

 - MPD.id 属性值必须和以前的 MPD 一致
 - Period.id 属性必须和以前的 Period 一致
 - MPD.publishTime 需要和更新时间一致

更详细的内容，可以参考 ISO/ITU 官方标准 `5.4 Media Presentation Description updates`。


### Segment 文件过期

特别针对于直播切片文件来说，由于服务器容量有限，所以，并不会一直被保存在服务器当中。最好的办法就是协定对应 Segment 的过期时间。那这个过期时间是怎么确定的呢？

主要根据以下公式：

```
[max(AST, now-timeShiftBufferDepth), now]
```

 - AST：是 MPD.availabilityStartTime 属性.
 - timeShiftBufferDepth: 是 MPD.timeShiftBufferDepth 属性，用来协定过期时间。


因为通常情况，`max(AST, now-timeShiftBufferDepth) = now-timeShiftBufferDepth`。所以，上面的范围可以精简为：

```
now-timeShiftBufferDepth <= time <= now
```
整个时长区间是由 `MPD.timeShiftBufferDepth` 属性值来控制的。例如，MPD 标签的设置值为：

```
timeShiftBufferDepth="PT5M"
```

那么，文件有效期范围为，具体最新 MPD 文件 publishTime 前 5min 之内的链接是有效的。



### Period 

Period 主要是用来包含具体 流媒体 的数据，其本身只是起到设置 `startTime` 和区分多个 `Period@id` 的作用。它的基本属性没有多少：


![image.png-43.6kB][7]

这里，我直接用文本的形式表达一下：

```
 
<xs:sequence>
  <xs:element name="BaseURL" type="BaseURLType" minOccurs="0" maxOccurs="unbounded"/> 
  <xs:element name="SegmentBase" type="SegmentBaseType" minOccurs="0"/>

  <xs:element name="SegmentList" type="SegmentListType" minOccurs="0"/> 
  <xs:element name="SegmentTemplate" type="SegmentTemplateType" minOccurs="0"/> 
  <xs:element name="AssetIdentifier" type="DescriptorType" minOccurs="0"/>

  <xs:element name="EventStream" type="EventStreamType" minOccurs="0" maxOccurs="unbounded"/> 
  <xs:element name="AdaptationSet" type="AdaptationSetType" minOccurs="0" maxOccurs="unbounded"/>
  <xs:any namespace="##other" processContents="lax" minOccurs="0" maxOccurs="unbounded"/>
</xs:sequence>
<xs:attribute ref="xlink:href"/>
<xs:attribute ref="xlink:actuate" default="onRequest"/>
<xs:attribute name="id" type="xs:string" />
<xs:attribute name="start" type="xs:duration"/>
<xs:attribute name="duration" type="xs:duration"/>
<xs:attribute name="bitstreamSwitching" type="xs:boolean" default="false"/> 
<xs:anyAttribute namespace="##other" processContents="lax"/>
```

对于 Period 我们主要关注一下 attribute 属性内容。

 - id: 用来标识 Period 的唯一性。如果 MPD.type=`dynamic`，则该 id 不会在 MPD 更新时改变。
 - start: 用来设置 Period 下，Segment 参考的开始时间。不过，该 start 的确定并不是根据本身的 Period 来决定，还需要根据前一个 Period 来确定：
    - 如果当前的 Period 没有 start 属性，但是前一个 Period 是正常的，有 start 和 duration 属性。那么当前的 `Period.start = prevPeriod.start + prevPeriod.duration`
    - 如果当前的 Period 没有 start 属性，`MPD.type = static`，该 Period 是该 MPD 第一个出现的 Period 标签，那么 start 值默认会被设置为 0

 - duration: 用来决定当前 Period 的时长
 - bitstreamSwitching: 是否允许改变切流，默认为 false。

例如：

```
  <Period id="p0" start="PT0S">
    // ...
  </Period>
```

### AdapationSet

AdaptionSet 主要是用来对当前包含的流进行基本信息的表述，相当于 MP4 中的 ftype+moof box 信息。

主要属性如下，

```
<xs:sequence>

  <xs:element name="Accessibility" type="DescriptorType" minOccurs="0" maxOccurs="unbounded"/>

  <xs:element name="Role" type="DescriptorType" minOccurs="0" maxOccurs="unbounded"/> 
  <xs:element name="Rating" type="DescriptorType" minOccurs="0" maxOccurs="unbounded"/> 
  <xs:element name="Viewpoint" type="DescriptorType" minOccurs="0" maxOccurs="unbounded"/> 
  <xs:element name="ContentComponent" type="ContentComponentType" minOccurs="0" maxOccurs="unbounded"/>

  <xs:element name="BaseURL" type="BaseURLType" minOccurs="0" maxOccurs="unbounded"/> 
  <xs:element name="SegmentBase" type="SegmentBaseType" minOccurs="0"/>

  <xs:element name="SegmentList" type="SegmentListType" minOccurs="0"/>

  <xs:element name="SegmentTemplate" type="SegmentTemplateType" minOccurs="0"/> 
  <xs:element name="Representation" type="RepresentationType" minOccurs="0" maxOccurs="unbounded"/>
</xs:sequence>
  <xs:attribute ref="xlink:href"/>
  <xs:attribute ref="xlink:actuate" default="onRequest"/> <xs:attribute name="id" type="xs:unsignedInt"/> 
  <xs:attribute name="group" type="xs:unsignedInt"/> <xs:attribute name="lang" type="xs:language"/> 
  <xs:attribute name="contentType" type="xs:string"/> <xs:attribute name="par" type="RatioType"/>
  <xs:attribute name="minBandwidth" type="xs:unsignedInt"/> <xs:attribute name="maxBandwidth" type="xs:unsignedInt"/> 
  <xs:attribute name="minWidth" type="xs:unsignedInt"/> <xs:attribute name="maxWidth" type="xs:unsignedInt"/> 
  <xs:attribute name="minHeight" type="xs:unsignedInt"/> <xs:attribute name="maxHeight" type="xs:unsignedInt"/> 
  <xs:attribute name="minFrameRate" type="FrameRateType"/> <xs:attribute name="maxFrameRate" type="FrameRateType"/>
  <xs:attribute name="segmentAlignment" type="ConditionalUintType" default="false"/> 
  <xs:attribute name="subsegmentAlignment" type="ConditionalUintType" default="false"/> 
  <xs:attribute name="subsegmentStartsWithSAP" type="SAPType" default="0"/> 
  <xs:attribute name="bitstreamSwitching" type="xs:boolean"/>
```

看图会不会更清晰：

![image.png-295.5kB][8]

这些属性，我就不一一多说了，大家看一个 Demo 理解一下：

```
<AdaptationSet contentType="video" maxFrameRate="60/2" maxHeight="360" maxWidth="640" mimeType="video/mp4" minHeight="360" minWidth="640" par="16:9" segmentAlignment="true" startWithSAP="1">
    <Role schemeIdUri="urn:mpeg:dash:role:2011" value="main" />
    <SegmentTemplate initialization="$RepresentationID$/init.mp4" media="$RepresentationID$/t$Time$.m4s" timescale="90000">
        <SegmentTimeline>
            <S d="180000" r="150" t="136377021600000" />
        </SegmentTimeline>
    </SegmentTemplate>
    <Representation bandwidth="300000" codecs="avc1.64001e" frameRate="60/2" height="360" id="V300" sar="1:1" width="640" />
</AdaptationSet>
```

这里的描述信息，并不只局限于 AS(AdaptionSet) 一个属性，还可以用在 Representation 上。



### Representation

Representation 是嵌套在 AS 里面，它也是流的相关描述信息，其实也就相当于是一个 IS（Initial Segment）。

主要信息有：

![image.png-273.2kB][9]


主要用法就是给当前的流提供基本描述信息：

```
<Representation bandwidth="300000" codecs="avc1.64001e" frameRate="60/2" height="360" id="V300" sar="1:1" width="640" />
```

指定 Rep（Representation）的生效范围是在当前的 Period 的范围之内，也就是从：`Period.start - Period.endTime`。Rep 的描述信息为了使用率最大，被 Rep 包含的 Stream 都可以用该内容作为描述信息。

另外，我们还可以结合 `dependencyId` 和 `id` 来重复利用 Rep 的内容。通过 `dependencyId` 指定模板 Rep@id，实现信息的直接复用。

```
<Representation bandwidth="300000" codecs="avc1.64001e" frameRate="60/2" height="360" id="1" sar="1:1" width="640" />

<Representation dependencyId="1" id="2" mimeType="video/mp4" />
```


## MPD 如何表达 Segments

在 MPD 中，描述 Segments 主要由 `SegmentBase, SegmentTemplate and SegmentList`。一个完整的 Rep 需要：

 - N+1 SegmentList 标签
 - 1 SegmentTemplate
 - N+1 BaseURL 标签，[0,1] SegmentBase，没有 SegmentTemplate 和 SegmentList 标签。

这里，我们简单描述一下对应 Segment tag 的内容。

### SegmentBase

SegmentBase 只在 static MPD 文件中使用，其基本内容其实是几个 Segments tag 共享内容，基本属性为：

```
<xs:sequence>
<xs:element name="Initialization" type="URLType" minOccurs="0"/>
<xs:element name="RepresentationIndex" type="URLType" minOccurs="0"/>
<xs:any namespace="##other" processContents="lax" minOccurs="0" maxOccurs="unbounded"/>
</xs:sequence>
<xs:attribute name="timescale" type="xs:unsignedInt"/>
<xs:attribute name="presentationTimeOffset" type="xs:unsignedLong"/>
 <xs:attribute name="timeShiftBufferDepth" type="xs:duration"/>
 <xs:attribute name="indexRange" type="xs:string"/>
<xs:attribute name="indexRangeExact" type="xs:boolean" default="false"/>
 <xs:attribute name="availabilityTimeOffset" type="xs:double"/>
 <xs:attribute name="availabilityTimeComplete" type="xs:boolean"/>
 
 
 // extension prop
 <xs:extension base="SegmentBaseType">
   <xs:sequence>
  <xs:element name="SegmentTimeline" type="SegmentTimelineType" minOccurs="0"/>
  <xs:element name="BitstreamSwitching" type="URLType" minOccurs="0"/> 
   </xs:sequence>
  <xs:attribute name="duration" type="xs:unsignedInt"/>
  <xs:attribute name="startNumber" type="xs:unsignedInt"/>
</xs:extension>
```

如果觉得不清楚，可以看这张图：

![image.png-194.7kB][108]


`Initialization` Tag 可以作为元素，也可以直接作为 Segments Tag 的属性，来设置 Initial Segment 的 URL 链接。基本设置为：

```
 <BaseURL>v-0144p-0100k-libx264.mp4</BaseURL>
 <SegmentBase indexRange="678-1597" timescale="12288">
    <Initialization range="0-677"/>
</SegmentBase>
```
indexRange 设置 media Segment 的字节范围，range 则设置 Initial Segment 的字节范围。

另外，还有几个属性比较重要：

 - presentationTimeOffset：设置当前 Segment 相对于 Period 开始的偏移时间。
 - timeShiftBufferDepth：默认覆盖 MPD 的属性，并且该值不能比 MPD 的小。
 - availabilityTimeOffset：设置当前 Segment 可以获得的时间偏移量。

不过，在实际使用时，使用 SegmentBase 的情况很少，大家对其有印象即可。

### SegmentList

该标签是用来标识一整套 Segment 的 URL list 内容。具体内容信息可以参考：

```
<xs:element name="SegmentURL" type="SegmentURLType" minOccurs="0" maxOccurs="unbounded"/>

<xs:attribute name="media" type="xs:anyURI"/>
<xs:attribute name="mediaRange" type="xs:string"/>
<xs:attribute name="index" type="xs:anyURI"/>
<xs:attribute name="indexRange" type="xs:string"/>
```

主要内容为：

![image.png-125.4kB][119]

该 Tag 常常用来包含一个完整的视频 List 内容。

```
 <SegmentList timescale="90000" duration="5400000">
       <RepresentationIndex sourceURL="representation-index.sidx"/>
       <SegmentURL media="segment-1.ts"/>
       <SegmentURL media="segment-2.ts"/>
       <SegmentURL media="segment-3.ts"/>
       <SegmentURL media="segment-4.ts"/>
       <SegmentURL media="segment-5.ts"/>
       <SegmentURL media="segment-6.ts"/>
       <SegmentURL media="segment-7.ts"/>
       <SegmentURL media="segment-8.ts"/>
       <SegmentURL media="segment-9.ts"/>
       <SegmentURL media="segment-10.ts"/>
</SegmentList>
```

### SegmentTemplate

SegmentTemplate 主要应用在文件切片数量过于庞大，如果使用 `SegmentList` 标签，则有点得不偿失。对于一些 live scenarios，比如直播，连麦等等。使用 `SegmentTemplate` 应该是非常有用的。它可以通过 startNumber，以及对应的 模板URL，来提供给客户端进行相关链接解析。例如：

```
<Representation mimeType="video/mp4"
                   frameRate="24"
                   bandwidth="1558322"
                   codecs="avc1.4d401f" width="1277" height="544">
  <SegmentTemplate media="http://cdn.bitmovin.net/bbb/video-1500/segment-$Number$.m4s" 
  initialization="http://cdn.bitmovin.net/bbb/video-1500/init.mp4" 
  startNumber="0" 
  timescale="24" 
  duration="48"/>
</Representation>
```

具体属性内容为：

![image.png-35.5kB][120]



 - index: 对于 Template 字段，对 \$Number\$ 和 \$Time\$ 标识符进行替换。
 - initialization: 用来标识 Initialization Segment 的具体地址，在该属性里面只能使用标识符\$RepresentationID\$。


使用 template 的方式，能够很大的减小 MPD 文件大小，不过会额外增加以下客户端解析 MPD 的时间。除了上面使用 `$Number$` 形式进行解码外，还可以使用 `$RepresentationID$` 的标识符来进行字符替换：

```
<SegmentTemplate
    duration="2"     
    initialization="$RepresentationID$/init.mp4"
    media="$RepresentationID$/$Number$.m4s" 
    startNumber="0" />
<Representation 
    bandwidth="300000" 
    codecs="avc1.64001e" 
    frameRate="60/2" 
    height="360" 
    id="V300" 
    sar="1:1" 
    width="640" />
```

那一共有哪些标识符可以使用呢？在哪些标签里面才能使用这些标识符呢？我们简单描述一下：

### Identifier 用途

Identifier 常常用在 SegmentTemplate 标签中，只有以下属性可以使用标识符：`media`, `index`, `initialization` 和 `bitstreamSwitching` 属性。
 
 - \$RepresentationID\$: 对应于 `Representation` 标签中的 id 属性，比如上面就是 `V300`。
 - \$Number\$: 对应于 Representation.startNumber 的数值，或者是 `SegmentTimeLine.S@r` 的顺序值。
 - \$Bandwidth\$: 对应于 `Representation.bandwidth` 的属性值。
 - \$Time\$: 对应于 `SegmentTimeline.t` 属性值，并且 Number 和 Time 标识符不能同时使用。 


另外，除了 SegmentTemplate 能够可以来作为 Template 模板之外，还有 SegmentTimeline 也可以完成该模板效果，只是一个是基于 Number，一个是基于 Time。

### SegmentTimeline

SegmentTimeline 里面会通过多个 `S` 标签，来标识在同一个 `MPD duration` 内的 Segment 内容。也就是说，SegmentTimeline 只是提供了一个模板容器，具体的流的标识还是由 `S` 标签内容来决定。直接看一个最简单的 ST(SegmentTimeline)：

```
<SegmentTimeline timescale="1000">
    <S t="0" r="10" d="24000"/>
</SegmentTimeline>
```

这里，我们先来看一下 `S` 标签下的基本属性值：

 - t：标识当前 Segment 在 Period 流中开始时间。一般来说，不用设置即可。如果设置了，则其值为 `previous S@t + @d * (@r + 1)`。就是之前一个 `S` 标签的结束时间戳。
 - d：标识当前 Segment 的 duration
 - r：设置当前的 Segment 出现的个数。

我们来看一个 DEMO：

```
    <SegmentTemplate media="segment-$Time$.ts" timescale="90000">
        <RepresentationIndex sourceURL="representation-index.sidx"/>
        <SegmentTimeline>
            ## 直接替换 $Time$ 的取值范围为 0-10.
            ## 并且设置对应的 duration
            <S t="0" r="10" d="54"/>
         </SegmentTimeline>
    </SegmentTemplate>
```
基本含义为，S 标签定义的 Segment 会出现 11 次，并且每个 Segment 的 duration 时长为 54/90000s。起始时间相对于 Period 的偏移量为 0。并且，根据 t 来替换 `$Time$` 标识符，比如，获取的连接为：

```
segment-0.ts
segment-54.ts
segment-108.ts
segment-162.ts
...
```


  [1]: http://static.zybuluo.com/jimmythr/s94zomjlemnofqpxaun6enat/image.png
  [2]: https://www.villainhr.com/page/2017/03/31/%E5%85%A8%E9%9D%A2%E8%BF%9B%E9%98%B6%20H5%20%E7%9B%B4%E6%92%AD
  [3]: http://blogpictures-1252111119.file.myqcloud.com/Screen%20Shot%202018-01-07%20at%208.16.04%20PM.png
  [4]: http://static.zybuluo.com/jimmythr/uarakewiwy9g9ln98xe841cp/image.png
  [5]: http://blogpictures-1252111119.file.myqcloud.com/Screen%20Shot%202018-01-07%20at%208.23.21%20PM.png
  [6]: http://static.zybuluo.com/jimmythr/mukmkw3k70cegjvcgod607ir/image.png
  [7]: http://static.zybuluo.com/jimmythr/i4t8ksex82jq0zlj720fe5vk/image.png
  [8]: http://static.zybuluo.com/jimmythr/yj990tu4b7bwov14kt1mvpiw/image.png
  [9]: http://static.zybuluo.com/jimmythr/pfgb4tdfq84bvh0wqbxkzpj8/image.png
  [10]: http://static.zybuluo.com/jimmythr/41jjc5k9qluo9kyph250f5aw/image.png
  [119]: http://static.zybuluo.com/jimmythr/f4gtj0rmtu20unvja9fm4f92/image.png
  [120]: http://static.zybuluo.com/jimmythr/vttqkkwalwdzibqcv9psuoxu/image.png