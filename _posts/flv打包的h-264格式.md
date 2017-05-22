---
title: flv打包的h.264格式
date: 2017-05-14 21:19:50
tags: [音视频]
---
FLV格式非常简单，头信息数据量很少，适合网络传输，因此被广泛的应用。 

### H264 NALU结构 

```
    h264 NALU:  0x00 00 00 01 | nalu_type(1字节)| nalu_data (N 字节) | 0x00 00 00 01 | ... 
                      起始码(4字节)          类型                            数据               下一个NALU起始码 
             H264 NALU固定以 0x00 00 00 01为起始，NALU_data部分不会出现这个起始码； 
             在找到下一个起始码之前，当前NALU数据长度不知； 
             NALU_type 1字节，定义为：1比特禁止位 | 2比特 重要性指示位  | 5比特 类型 
                                                             固定为0           11重要 不能少          1-12 由h264使用 
                                                                                    00不重要 可以丢弃      
             几个常用Nalu_type: 
                               0x67 (0 11 00111) SPS    非常重要       type = 7 
                               0x68 (0 11 01000) PPS     非常重要       type = 8 
                               0x65 (0 11 00101) IDR帧  关键帧  非常重要 type = 5 
                               0x41 (0 10 00001) P帧      重要         type = 1      
                               0x01 (0 00 00001) B帧     不重要        type = 1 
                               0x06 (0 00 00110) SEI     不重要        type = 6 
                               
```                               
   
                               
## FLV tag 

    前面讲过FLV文件就是由无数个Tag组成的，Tag有Video Tag, Audio Tag和Script Tag. 
    A/V Tag里面存储的就是音视频编码数据，Script Tag里面是一些码流描述信息。 
    理论上来说，不解析Script tag也可以对A/V Tag完整解码。tag的固定格式是： 
     Tag Type(1字节) | DataSize(3字节) | TimeStamp(3字节) | TimeStampExtended (1字节)| StreamID (3) | ... 
     下面将分别介绍各种NALU封到tag里面的结构。 

### 一般Video tag 

```
                                         字节位置    意义 
0x09,                              // 0,        TagType 
0xzz, 0xzz, 0xzz,              // 1-3,     DataSize,     
0xzz, 0xzz, 0xzz, 0xzz,    // 4-6, 7 TimeStamp | TimeStampExtend     
0x00, 0x00, 0x00,            // 8-10,   StreamID 
  
0xz7,                                  // 11,       FrameType | CodecID 
0x01,                                  // 12,       AVCPacketType        
0x00, 0x00, 0x00,                // 13-15, CompositionTime 
  
0xzz, 0xzz, 0xzz, 0xzz,        // 16-19,   NaluLength   NBytes 
0xzz, ...., ...., 0xzz,             // NBytes,  NaluData   
  
0xzz, 0xzz, 0xzz, 0xzz.        // N+1-N+3, PreviousTagSize 

    其中 0xzz的意思是该字节根据实际情况付不同的值 
    2.1 DataSize[0,1,2] = NaluLength + 5 + 4; 
           5 是 AVCPacket头5比特，（FrameType+CodecID | AVCPacketType | CompositionTime） 
           4 是写入NaluLength        

    2.2 对于一个裸h264流，没有时间戳的概念，可以默认以25fps，即40ms一帧数据。 
        int cts = 0; 
        TimeStamp[0,1,2]   = cts[0,1,2]; 
        TimeStampExtend[0] = cts[3];    
        cts += 40; 
     
    2.3  if(nalu_type == IDR)  FrameType | CodecID = 0x17; 
            else                          FrameType | CodecID = 0x27; 

    2.4 NaluLength就是nalu长度，然后紧跟N字节的Nalu数据。 
    2.5 PreviousTagSize在这里计算最为方便，PreviousTagSize = 11 + 5 + 4 + NaluLength 
                                                            11 是video tag头数据 （TagType到StreamID） 
     IDR，I，P，B帧的NALU都是这个结构 

```

### SPS/PPS NALU 

```
   SPS和PPS在FLV里面称为序列头信息sequence header，它的AVCPacketType为0x00 
                                      字节位置   意义 
0x09,                               // 0,       TagType 
0xzz, 0xzz, 0xzz,               // 1-3,     DataSize,     
0x00, 0x00, 0x00, 0x00,      // 4-6, 7;  TimeStamp | TimeStampExtend     
0x00, 0x00, 0x00,             // 8-10,    StreamID 
  
                                            // AVC video tage header 5Bytes   
0x17,                                  // 11,      FrameType | CodecID 
0x00,                                  // 12,      AVCPacketType        
0x00, 0x00, 0x00,                // 13-15,   CompositionTime 
        
                                            // AVCDecoderConfigurationRecord 6 Bytes 
0x01,                                  // 16, ConfigurationVersion 
0xzz,                                   // 17,  AVC Profile 
0x00,                                  // 18,  profile_compatibility 
0xzz,                                  // 19,  AVC Level 
0xFF,                                  // 20,  lengthSizeMinusOne, 
                                           //         reserved 6bits | NAL unit length-1, commonly be 3 
0xzz,                                  // 21,  numOfSequenceParameterSets, 
                                           //         reserved 3bits | SPS count, commonly be 1 

0xzz, 0xzz,                       // 22-23,   SPS0 Length N0 Byte 
0xzz, ...., 0xzz,                 // N0 Byte  SPS0 Data 
0xzz, 0xzz,                       // SPSm Length Nm Byte (如果存在)  循环存放最多31个SPS        
0xzz, ...., 0xzz,                 // Nm Byte  SPSm Data 
         
0xzz,                             //          PPS count 
0xzz, 0xzz,                      //          PPS0 Length 
0xzz, ...., 0xzz,                // N0 Byte  PPS0 Data 
0xzz, 0xzz,                      //          PPSm Length Nm Byte (如果存在)  循环存放最多255个PPS        
0xzz, ...., 0xzz,                // Nm Byte  PPSm Data 
  
0xzz, 0xzz, 0xzz, 0xzz. // N+1-N+3, PreviousTagSize 

  3.1 在H.264码流里面reserved bit一般为0; 而在FLV码流里面reserved bit定义为1 
  3.2 在H.264里面 SPS和PPS是对立的NALU，但是在FLV里面会把他们统一写在一个Video Tag里面。 
         而且这个tag必须是FLV里面第一个Video Tag,否则接收到其他video tag也没法解码. 
         为了防止SPS,PPS数据丢失，有些编码器会在每个IDR帧之前重复发SPS,PPS。这些SPS其实是一样的。 
         但也不排除有些变态的编码器前后的SPS会不同，比较标准容许这样做。 
         这样就需要首先遍历一边h264码流，将其中不同的SPS,PPS提起出来，先记录下来，然后再统一写到FLV。 
         也可以大胆一点接收到第一个SPS和第一个PPS后就结束这个遍历，就当作码流里面只有一个SPS和一个PPS。 
  
  3.3 DataSize=5 +                  // AVC video tag header (FrameType + CodecID | .. CompositionTime) 
                       6 +                   // AVCDecoderConfigurationRecord 
                      SPSCount*2 +               // 每个SPS长度2字节 
                      各个 SPSDataLength + // 所有SPS数据长度和 
                      1 +                                     // PPS个数 
                      PPSCount*2 +                // 每个PPS长度2字节 
                      各个 PPSDataLength;   // 所有PPS数据长度和

  3.4 AVC Profile和 AVC Level就等于SPS NALU里面第1字节和第3字节 （第0字节为NaluType）  
  3.5 lengthSizeMinusOne，这个定义没有理解，不知道低2比特是什么含义，看到很多文档里面就直接设为0b11, 所有这个字节为 0xFF

  3.6 numOfSequenceParameterSets, 低5比特是SPS个数，H.264标准里面定义最多SPS个数为255,这里只有31。 
        不知道会不会存在问题，当然一般情况下就一个SPS,该值为 0xE1 (0b111 00001) 
  3.7 每个SPS，PPS数据长度都用两个字节来表述，

  3.8 这个tag的 PreviousTagSize = 11 + DataSize。 11 是Video tag （TagType到StreamID）    
```


4. FLV头 

```
       'F', 'L', 'V',                              // 0-2 FLV file Signature, also can be 'f''l''v' 
       0x01,                                     // FLV version, 
       0x0z,                                     // AV tag Enable.  0x05 AV both, 0x03 audio only, 0x01 video only 
       0x00, 0x00, 0x00, 0x09,        // Length of this header. 
       0x00, 0x00, 0x00, 0x00.        // PreviousTagLength. 
```

### SEI NALU 

```
   SEI是H.264里面的附加增强信息NALU，他对解析解码没有帮助，但提供一些编码器控制参数等信息。 
   FLV没有一个Tag单独包含SEI数据，它把SEI数据和紧随其后那个视频NALU数据打在同一个Video Tag里面。 
   包含SEI数据的VideoTag结构如下 
                                      字节位置   意义 
0x09,                               // 0,       TagType 
0xzz, 0xzz, 0xzz,               // 1-3,     DataSize,     
0xzz, 0xzz, 0xzz, 0xzz,     // 4-6, 7;  TimeStamp | TimeStampExtend     
0x00, 0x00, 0x00,             // 8-10,    StreamID 
  
0x27,                                   // 11,      FrameType | CodecID 
0x01,                                   // 12,      AVCPacketType        
0x00, 0x00, 0x00,                 // 13-15,   CompositionTime 

0xzz, 0xzz, 0xzz, 0xzz,         // 16-19,   SEILength   NBytes 
0xzz, ...., ...., 0xzz,              // NBytes,  SEIData   
  
0xzz, 0xzz, 0xzz, 0xzz,         //          NaluLength   NBytes 
0xzz, ...., ...., 0xzz,              // NBytes,  NaluData   
  
0xzz, 0xzz, 0xzz, 0xzz.     // PreviousTagSize 

   5.1 DataSize[0,1,2] = （NaluLength + 5 + 4） + （SEILength + 4）; 
```

### 得到NALU代码 

```
// 输入:  H264_fp 264文件指针 
// 输出： 找到的Nalu长度， 
          *nalu_type返回找到的NALU类型 
int h264_get_nalu(FILE *h264_fp, uint8_t *nalu_type) { 
    int start_pos = -1; 
    int nalu_size = 0; 
    int zero_num = 0; 
    uint8_t tmp; 
    while(!feof(h264_fp)){ 
        fread(&tmp, 1, 1, h264_fp); 
        if(tmp == 0) zero_num++; 
        else if(tmp == 1) { 
            if(zero_num >= 3) { 
                if(start_pos == -1) { 
                    start_pos = ftell(h264_fp); 
                    fread(nalu_type, 1, 1, h264_fp); 
                } else { 
                    nalu_size = ftell(h264_fp) - start_pos - 4; 
                    fseek(h264_fp, start_pos, 0); 
                    break; 
                } 
            } 
        } else 
            zero_num = 0; 
    } 
    return nalu_size; 
}
```