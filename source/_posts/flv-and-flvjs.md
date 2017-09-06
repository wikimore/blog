---
title: flvjs不能播放纯音频问题解决
date: 2017-04-29 13:52:49
tags: [flv,flvjs]
---
最近有个项目要用flvjs播放httpflv格式的纯音频流，在实践的过程中发现数据是在不停的加载，可是没有声音。
<!-- more -->
遇到这样的问题，第一个想法就是协议是不是哪里不匹配，先了解flv协议是最好的选择。

### flv协议简单说明(audio部分)
FLV协议是adobe开发设计的，现在已经公开，在数据格式上类似SWF格式，主要分为`Header`和`Body`两个部分。

#### Header

负责记录协议类型、版本、流信息、长度等信息。

前3个字节固定是`FLV`3个字母的ASCII码，表明是FLV格式

第4个字节表示FLV协议的版本，目前是第一个版本

第5个字节表示FLV的流信息，暂时只有第6bit和第8bit有意义，其他位都应该是0，第6bit标识是否存在Audio，第8bit标识是否存在Video。

第6-9个字节表示整个`Header`的长度，是一个UI32类型的数字，值一般是9，官方SPEC说在未来的版本可能会有更大的`Header`。

#### Body

FLV协议的`Body`主要由一个UI32的指针和`TAG`交替循环组成。

指针标识前一个`TAG`的长度，为什么不是后一个`TAG`的长度呢？可能和数据是流有关，有时候没法知道后面一个`TAG`的大小。

`TAG`包含各种数据类型，先说说`TAG`的格式定义吧。

第一个字节的前2个bit为预留，第3个bit标识包是否需要过滤或预处理，后5个bit标识类型，目前支持的类型有

- 8 Audio
- 9 Video
- 18 Script Data

之后3个字节表示整个`TAG`的数据长度，从StreamID后开始计算的数据的长度。

之后4个字节表示相对第一个`TAG`被接受的毫秒时间戳，第一个`TAG`始终是0，需要注意的是，前3字节是低位，第4字节表示高位。

第9-11字节表示StreamID，SPEC要求始终为0。

再后面分别根据TAG类型的不同标识不同类型Header，这个需要具体类型具体解析，已知的包括：

- AudioTagHeader 如果tagtype==8
- VideoTagHeader 如果tagtype==9
- EncryptionHeader 如果是否过滤 == 1
- FilterParams 如果是否过滤 == 1

最后是真正的数据，如果是AudioTag那么就是音频数据，如果是VideoTag就是视频数据，如果是ScriptTag就是Script数据。

> 因为本例只讨论纯音频流，所以我也没去了解VideoTag，下面介绍下AudioTag

##### Audio Tag

AudioTag会定义声音的格式、采样率、采样大小、类型等等，最后是真正的音频数据。

前4bit定义声音的格式：包括PCM、MP3、Nellymoser、AAC、Speex等等，5-6bit定义声音的采样率：从5.5 kHz~44kHz总共4种，第7bit标识声音采样大小，仅针对非压缩格式，第8bit标识声音类型，mono或者stereo。

如果音频格式是AAC的话，还有一个字节表示AACPacketType，0标识AAC sequence header，1表示AAC raw data

>具体可以参考[flv-spec](https://www.adobe.com/content/dam/Adobe/en/devnet/flv/pdfs/video_file_format_spec_v10.pdf)

### flvjs
简单了解了FLV协议，再回到flvjs不能播放的问题上，先了解下flvjs的原理，看了一下[设计图](https://github.com/Bilibili/flv.js/blob/master/docs/design.md)，感觉是FLV自己下载数据，然后按照flv的格式解析出真正的音频或者视频数据，再交由原生的播放器来播放。

参照之前说的，数据下载是正常的，那么只有可能是编解码的问题，debug看看到底是哪里有问题，但是debug之后我发现所有的编解码都是正常的，只是有一个AMF解析错误，主要原因是onMetaData的数据是空的，而flvjs没有处理这个情况，造成数组越界，具体代码如下：

```
try {
    var name = AMF.parseValue(arrayBuffer, dataOffset, dataSize); // dataOffset=24  dataSize=13
    // name = "onMetaData" 然后下面的dataSize-name.size=0
    var value = AMF.parseValue(arrayBuffer, dataOffset + name.size, dataSize - name.size);
    data[name.data] = value.data;
} catch (e) {
    _logger2.default.e('AMF', e.toString());
}
```

但是这个错误并不会影响后面数据解析的流程，到底是哪里的问题呢？之后我发现在解析完音频数据后，并没有将数据交给播放器播放。

```
if (this._isInitialMetadataDispatched()) {
    // Non-initial metadata, force dispatch (or flush) parsed frames to remuxer
    if (this._dispatch && (this._audioTrack.length || this._videoTrack.length)) {
        this._onDataAvailable(this._audioTrack, this._videoTrack);
    }
} else {
    this._audioInitialMetadataDispatched = true;
}
```

_isInitialMetadataDispatched这个方法永远返回false。

```
function _isInitialMetadataDispatched() {
    if (this._hasAudio && this._hasVideo) {
        // both audio & video
        return this._audioInitialMetadataDispatched && this._videoInitialMetadataDispatched;
    }
    if (this._hasAudio && !this._hasVideo) {
        // audio only
        return this._audioInitialMetadataDispatched;
    }
    if (!this._hasAudio && this._hasVideo) {
        // video only
        return this._videoInitialMetadataDispatched;
    }
    return false;
}
```

为什么这个方法一直返回false呢，原来flvjs会探测协议，发现既有音频数据也有视频数据(这就是flv协议Header中流信息定义的)，当flvjs解析完音频后，期望解析视频后一起交由播放器来播放，但是因为实际是没有视频数据，所以音频数据也不会被播放。

目前该问题已经通过强制设置_hasVideo=false解决，并且也反馈给flvjs的作者，新版flvjs也提供了配置来解决纯音频不能播放的问题。

### 总结
不停的解决问题其实就是不停的学习的过程，这是程序员必备的素质之一，对未知领域需要敬畏，但不能因为不了解而放弃。只要找到问题的原因，即使最后不能解决，也一样会有收获。



