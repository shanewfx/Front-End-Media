## MP4 (MPEG-4 Part 14)

## 前言
MP4 是 MPEG-4 标准中第14部分，即 [MPEG-4 Part 14](http://www.telemidia.puc-rio.br/~rafaeldiniz/public_files/normas/ISO-14496/ISO_IEC_14496-14_2003_PDF_version_(en).pdf)，是一种常见的音视频容器格式，由国际标准化组织（ISO）和国际电工委员会（IEC）下属的“动态图像专家组”（Moving Picture Experts Group，即MPEG）制定，扩展名常为 `.mp4`。MPEG-4 Part 14 是更通用的 [ISO / IEC 14496-12:2004(Part 14: MP4 file format)](http://www.telemidia.puc-rio.br/~rafaeldiniz/public_files/normas/ISO-14496/ISO_IEC_14496-14_2003_PDF_version_(en).pdf) 的一个实例，他们的关系如下图：

![关系](./docs/images/relations.png)

## MP4 文件结构
MP4文件由很多个 `box` 组成，所有的视频信息、视频数据都在这些 `box` 中。`box` 内部分为两部分：`header` 和 `data`，`header` 描述了当前 `box` 的大小（size）和类型（type），而 `data` 里面存放了具体的数据或子 `box` （包含子box的盒子称为容器盒子 `container box`）。
- `box header` 中的 **size** 表示整个盒子的长度（包含header自身、子box长度等），占32位。特殊情况：当size为0时，表示是最后一个 `box` ；当size为1时，表示 `box` 的长度需要更多的bit来描述（64位的largesize表示）。
- `box header` 中的 **type** 表示盒子的类型，占32位，内容即为盒子英文类型的 ASCII 码。特殊情况：当type为uuid时，后续会有个用户自定义盒子类型。

`box` 结构的伪代码：
```c++
aligned(8) class Box (unsigned int(32) boxtype, optional unsigned int(8)[16] extended_type) {
  unsigned int(32) size;
  unsigned int(32) type = boxtype;
  if (size==1) {
    unsigned int(64) largesize;
  } else if (size==0) {
    // box extends to end of file
  }
  if (boxtype==‘uuid’) {
    unsigned int(8)[16] usertype = extended_type;
  }
} 
```

除了上述的 `box` ，还有一种基于它的扩展类 `full box` ，就是在其基础上增加了 8bit version 和 24bit flags，伪代码如下：
```c++
aligned(8) class FullBox(unsigned int(32) boxtype, unsigned int(8) v, bit(24) f)
 extends Box(boxtype) {
 unsigned int(8) version = v;
 bit(24) flags = f;
} 
```

综上，mp4的文件格式如下图所示：

![mp4文件格式](./docs/images/mp4-format.png)

## MP4 盒子类型
|类型10|  |  |  |  |  |重要性|章节|描述|
|--|--|--|--|--|--|--|--|--|
|ftyp||||||*|4.3| file type and compatibility|
|pdin|||||||8.43|progressive download information|
|moov||||||*|8.1| container for all the metadata|
||mvhd|||||*|8.3|movie header, overall declarations|
||trak|||||*|8.4|container for an individual track or stream|
|||tkhd||||*|8.5|track header, overall information about the track|
|||tref|||||8.6|track reference container|
|||edts|||||8.25|edit list container|
||||elst||||8.26|an edit list|
|||mdia||||*|8.7|container for the media information in a track|
||||mdhd|||*|8.8|media header, overall information about the media|
||||hdlr|||*|8.9|handler, declares the media (handler) type|
||||minf|||*|8.10|media information container|
|||||vmhd|||8.11.2|video media header, overall information (video track only)|
|||||smhd|||8.11.3|sound media header, overall information (sound track only)|
|||||hmhd|||8.11.4|hint media header, overall information (hint track only)|
|||||nmhd|||8.11.5|Null media header, overall information (some tracks only)|
|||||dinf||*|8.12|data information box, container|
||||||dref|*|8.13|data reference box, declares source(s) of media data in track|
|||||stbl||*|8.14|sample table box, container for the time/space map|
||||||stsd|*|8.16|sample descriptions (codec types, initialization etc.)|
||||||stts|*|8.15.2|(decoding) time-to-sample|
||||||ctts||8.15.3|(composition) time to sample|
||||||stsc|*|8.18|sample-to-chunk, partial data-offset information|
||||||stsz||8.17.2|sample sizes (framing)|
||||||stz2||8.17.3|compact sample sizes (framing)|
||||||stco|*|8.19|chunk offset, partial data-offset information|
||||||co64||8.19|64-bit chunk offset|
||||||stss||8.20|sync sample table (random access points)|
||||||stsh||8.21|shadow sync sample table|
||||||padb||8.23|sample padding bits|
||||||stdp||8.22|sample degradation priority|
||||||sdtp||8.40.2|independent and disposable samples|
||||||sbgp||8.40.3.2|sample-to-group|
||||||sgpd||8.40.3.3|sample group description|
||||||subs||8.42|sub-sample information|
||mvex||||||8.29|movie extends box|
|||mehd|||||8.30|movie extends header box|
|||trex||||*|8.31|track extends defaults|
||ipmc||||||8.45.4|IPMP Control Box|
|moof|||||||8.32|movie fragment|
||mfhd|||||*|8.33|movie fragment header|
||traf||||||8.34|track fragment|
|||tfhd||||*|8.35|track fragment header|
|||trun|||||8.36|track fragment run|
|||sdtp|||||8.40.2|independent and disposable samples|
|||sbgp|||||8.40.3.2|sample-to-group|
|||subs|||||8.42|sub-sample information|
|mfra|||||||8.37|movie fragment random access|
||tfra||||||8.38|track fragment random access|
||mfro|||||*|8.39|movie fragment random access offset|
|mdat|||||||8.2|media data container|
|free|||||||8.24|free space|
|skip|||||||8.24|free space|

## JS 封装 mp4 box
 `box` 的封装，只需要指定 `box type` 和数据内容 `data` （忽略 `largesize` 和 `uuid` 这两种特殊情况）。示例代码如下：
```typescript
function Box (type: string, ...payloads: Uint8Array[]) {
  // 盒子长度为 头部8字节 + 载荷数据总长度
  const size = 8 + payloads.reduce((sum, cur) => (sum += cur.byteLength), 0);
  const box = new Uint8Array(size);

  // size
  box[0] = (size >> 24) & 0xff;
  box[1] = (size >> 16) & 0xff;
  box[2] = (size >> 8) & 0xff;
  box[3] = size & 0xff;

  // type
  box[4] = type.charCodeAt(0);
  box[5] = type.charCodeAt(1);
  box[6] = type.charCodeAt(2);
  box[7] = type.charCodeAt(3);

  // data
  let offset = 8;
  payloads.forEach(playload => {
    box.set(playload, offset);
    offset += playload.byteLength;
  });

  return box;
}
```

 `full box` 是 box 的扩展，box 结构的基础上在 header 中增加 8bits version 和 24bits flags。示例代码如下：
```typescript
function FullBox (type: string, version: number, flags: number, ...payloads: Uint8Array[]) {
  return Box(
    type,
    new Uint8Array([
      version & 0xff,
      (flags >> 16) & 0xff,
      (flags >> 8) & 0xff,
      flags & 0xff
    ]),
    ...payloads
  );
}
```

接下来用 JS 去实现不同 mp4 box。

### ftyp
ftyp 是 `File Type Box` 的简写，它描述了该 mp4 的解码标准，如解码版本、兼容格式等。通常 ftyp 都是放在 mp4 等开头。格式如下：
```cpp
aligned(8) class FileTypeBox
  extends Box(‘ftyp’) {
  unsigned int(32) major_brand;
  unsigned int(32) minor_version;
  unsigned int(32) compatible_brands[]; // to end of the box
} 
```
上述字段的含义如下：
- major_brand: 比如常见的 isom、mp41、mp42、avc1、qt等。它表示“最好”基于哪种格式来解析当前的文件。一般而言都是使用 isom 这个万金油即可。如果是需要特定的格式，可以自行定义。
- minor_version: 指major_brand的最低兼容版本。
- compatible_brands: 和 major_brand 类似，通常是针对 MP4 中包含的额外格式，比如，AVC，AAC 等相当于的音视频解码格式。

js 实现方式：
```typescript
function ftyp (
  major_brand = 'isom',
  minor_version = 1,
  compatible_brands = ['avc1']
) {
  return Box(
    'ftyp',
    new Uint8Array([
      major_brand.charCodeAt(0),
      major_brand.charCodeAt(1),
      major_brand.charCodeAt(2),
      major_brand.charCodeAt(3),  // major_brand
      (minor_version >> 24),
      (minor_version >> 16) & 0xFF,
      (minor_version >>  8) & 0xFF,
      minor_version & 0xFF,       // minor_version
    ]),
    ...compatible_brands.map(brand => new Uint8Array([
      brand.charCodeAt(0),
      brand.charCodeAt(1),
      brand.charCodeAt(2),
      brand.charCodeAt(3)
    ]))                           // compatible_brands
  );
}

// eg: Uint8Array(20) [0 0 0 20 102 116 121 112 105 115 111 109 0 0 0 1 97 118 99 49]
```

### moov
moov 是 `Movie Box` 的缩写，它本身并不具备有用信息，但子盒子中包含了媒体播放所需的元数据，如视频的编码等级、分辨率，音频的声道、采样率等。格式如下：
```cpp
aligned(8) class MovieBox extends Box(‘moov’){ }
```
次级盒子主要有：**mvhd**、**trak**、**mvex**

### moov::mvhd
mvhd 是 `MovieHeaderBox` 的缩写，描述了媒体文件的整体信息，如创建时间、文件时长等。格式如下：
```cpp
aligned(8) class MovieHeaderBox extends FullBox(‘mvhd’, version, 0) {
  if (version==1) {
    unsigned int(64) creation_time;
    unsigned int(64) modification_time;
    unsigned int(32) timescale;
    unsigned int(64) duration;
  } else { // version==0
    unsigned int(32) creation_time;
    unsigned int(32) modification_time;
    unsigned int(32) timescale;
    unsigned int(32) duration;
  }
  template int(32) rate = 0x00010000; // typically 1.0
  template int(16) volume = 0x0100; // typically, full volume˝
  const bit(16) reserved = 0;
  const unsigned int(32)[2] reserved = 0;
  template int(32)[9] matrix =
  { 0x00010000,0,0,0,0x00010000,0,0,0,0x40000000 };
  // Unity matrix
  bit(32)[6] pre_defined = 0;
  unsigned int(32) next_track_ID;
}
```
上述字段的含义如下：
- creation_time 表示文件的创建时间（从UTC时间的1904年1月1日0点至今的秒数）；
- modification_time 表示文件的修改时间；
- timescale 表示一秒包含的时间单位（文件内所有时间的计算都要以它为参照）；
- duration 表示媒体的可播放时长，**duration / timescale = 可播放时长（S）**；
- rate 表示媒体的原始倍速，高16位代表整数部分、低16位代表小数部分；
- volume 表示媒体的音量，高8位代表整数部分、低8位代表小数部分；
- matrix 表示媒体的转换矩阵，忽略；
- next_track_ID 表示下个轨道的ID，非零整型，忽略。

JS 实现方式：
```typescript
function mvhd (timescale: number, duration: number) {
  return FullBox(
    'mvhd', 0, 0,
    new Uint8Array([
      0x00, 0x00, 0x00, 0x00,       // creation_time
      0x00, 0x00, 0x00, 0x00,       // modification_time
      (timescale >> 24) & 0xFF,
      (timescale >> 16) & 0xFF,
      (timescale >>  8) & 0xFF,
      timescale & 0xFF,             // timescale
      (duration >> 24) & 0xFF,
      (duration >> 16) & 0xFF,
      (duration >>  8) & 0xFF,
      duration & 0xFF,              // duration
      0x00, 0x01, 0x00, 0x00,       // 1.0 rate
      0x01, 0x00,                   // 1.0 volume
      0x00, 0x00,                   // reserved
      0x00, 0x00, 0x00, 0x00,       // reserved
      0x00, 0x00, 0x00, 0x00,       // reserved
      0x00, 0x01, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x01, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x40, 0x00, 0x00, 0x00,       // matrix
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,       // pre_defined
      0xff, 0xff, 0xff, 0xff        // next_track_ID
    ])
  );
}
```

### moov::trak
trak 是 `Track Box` 的缩写，它是容器盒子，其次级盒子中包含了单个媒体轨道（track）的描述信息。格式如下：
```cpp
aligned(8) class TrackBox extends Box(‘trak’) {}
```
次级盒子主要有：**tkhd**、**mdia** 等。

### moov::trak::tkhd
tkhd 是 `Track Header Box` 的缩写，它包含了单个媒体轨道的描述信息。格式如下：
```cpp
aligned(8) class TrackHeaderBox
  extends FullBox(‘tkhd’, version, flags){
  if (version==1) {
    unsigned int(64) creation_time;
    unsigned int(64) modification_time;
    unsigned int(32) track_ID;
    const unsigned int(32) reserved = 0;
    unsigned int(64) duration;
  } else { // version==0
    unsigned int(32) creation_time;
    unsigned int(32) modification_time;
    unsigned int(32) track_ID;
    const unsigned int(32) reserved = 0;
    unsigned int(32) duration;
  }
  const unsigned int(32)[2] reserved = 0;
  template int(16) layer = 0;
  template int(16) alternate_group = 0;
  template int(16) volume = {if track_is_audio 0x0100 else 0};
  const unsigned int(16) reserved = 0;
  template int(32)[9] matrix=
  { 0x00010000,0,0,0,0x00010000,0,0,0,0x40000000 };
  // unity matrix
  unsigned int(32) width;
  unsigned int(32) height;
}
```
上述字段的含义如下：
- flags 通过按位或计算得到，默认值为7（即0b0111），表示这个track是启用的、用于播放的且用于预览的。
  - Track_enabled：值为0x000001，表示这个track是启用的，当值为0x000000，表示这个track没有启用；
  - Track_in_movie：值为0x000002，表示当前track在播放时会用到；
  - Track_in_preview：值为0x000004，表示当前track用于预览模式；
- creation_time 文件创建时间；
- modification_time 文件修改时间；
- track_ID 轨道ID，不能为0；
- duration 当前轨道的时长（需要与timescale计算）；
- layer 视频轨道的叠加层级，越小越处于顶级（靠近观看者）；
- alternate_group 当前track的分组ID。同一分组的track，在同一时间只能有一个track处于播放状态；当分组ID为0时，表示没有其他track跟当前track处于同一分组；一个分组内，也可以只有一个track；
- volume 音频轨道的音量；
- matrix 视频的变换矩阵；
- width 视频宽，高16位代表整数部分、低16位代表小数部分；
- height 视频高，高16位代表整数部分、低16位代表小数部分；

JS 实现方式：
```typescript
function tkhd (track: IBoxTrack) {
  const { id, duration, isAudio, width, height } = track;
  return FullBox(
    'tkhd', 0, 7,
    new Uint8Array([
      0x00, 0x00, 0x00, 0x00, // creation_time
      0x00, 0x00, 0x00, 0x00, // modification_time
      (id >> 24) & 0xFF,
      (id >> 16) & 0xFF,
      (id >> 8) & 0xFF,
      id & 0xFF, // track_ID
      0x00, 0x00, 0x00, 0x00, // reserved
      (duration >> 24),
      (duration >> 16) & 0xFF,
      (duration >>  8) & 0xFF,
      duration & 0xFF, // duration
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00, // reserved
      0x00, 0x00, // layer
      0x00, 0x00, // alternate_group
      (isAudio ? 0x01 : 0x00), 0x00, // track volume
      0x00, 0x00, // reserved
      0x00, 0x01, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x01, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00,
      0x40, 0x00, 0x00, 0x00, // transformation: unity matrix
      (width >> 8) & 0xFF,
      width & 0xFF,
      0x00, 0x00, // width
      (height >> 8) & 0xFF,
      height & 0xFF,
      0x00, 0x00 // height
    ])
  );
}
```

# 参考
- [Part 12: ISO base media file format](https://sce.umkc.edu/faculty-sites/lizhu/teaching/2018.fall.video-com/ref/mp4.pdf)
- [Part 14: MP4 file format](http://www.telemidia.puc-rio.br/~rafaeldiniz/public_files/normas/ISO-14496/ISO_IEC_14496-14_2003_PDF_version_(en).pdf)
- [MP4文件格式入门【blog】](https://www.imooc.com/article/313004)
- [mp4文件格式解析【blog】](https://www.jianshu.com/p/529c3729f357)
