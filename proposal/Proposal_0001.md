# 【ACFUN/BILIBILI】利用磁力链接网络，实现去中心的弹幕视频体验
（以及改装成聊天室、论坛、匿名版等等）？

最近国内著名弹幕网站ACFUN和BILIBILI都因为版权问题受到困扰。  
因为其他网站需要独占，很多视频因此404，弹幕服务也自然unavailable。  
缺少良好的整合的弹幕体验，对于弹幕上瘾的御宅一族来说是个不小的困扰。  

在这个时候我又把箱子底的这个想法翻出来——分布式的弹幕服务。  
把C-S模式的ACFUN与BILIBILI的弹幕中心服务器，打散分布到因特网上。  
一方面弹幕本身是版权中性的，不应，一般也不会受到连坐；  
另一方面一个个节点无从下手，版权方等等也难以破坏其服务。  

以下是服务的协议概要，应该指出，这是一个高度模仿BT/Magnet协议的方案。  
但弹幕内容是随时间延长的，这与一般BT资源长度与哈希不变特点不同。  
因此这个服务是应该是健壮和抗攻击的，但是对完整和及时性较宽容。  
此外这个服务还可以进一步改造成为成为聊天室、论坛、匿名版等等。  
以下正文。  

--------------------------------------------------------------

# 《分布式网络的弹幕通信协议概要》  

这一协议试图实现一个分布式，去中心，持久化的弹幕广播系统。  
以下是该弹幕网络实现的基本思路，未说明部分请参考现有P2P协议:  

## 一、组网  

任意磁链将其sha1经某双射函数f(x)变换，其结果(一条伪磁链)用作弹幕信道标识。  
>>注：这个双射函数，也就是一一映射，是利用了sha1难以近似碰撞的特性。  
>>难以构造sha1相似的资源的另一重意思是，正常情况下两个资源sha1相似更不可能出现。  
>>因此我们可以使用一个恒x!=f(x)的函数，来为这个资源创造唯一的弹幕频道  
>>这样的函数有很多，比如f(x)=x+Const，按位取反，等等  
>>比如我验证过网上的2012年海盗湾存档，160万条里不存在只有最后一位不同的磁链。  
>>因此完全可以对这个40位16进制数的最后一位做凯撒密码变换（比如0-E变成1-F，F变0）  
>>每个资源磁链都可以构造15个弹幕信道，也完全可以根据弹幕信道名还原对应资源的磁链。  

以伪磁链为标识，利用一般BT协议来联结支持弹幕扩展的客户端，组成弹幕通信网络。  
因此，弹幕信道成功组网。扩展协议与BT协议高度相似，具有DHT/PEX网络的一切特性。  
>>注：一般的BT/Magnet网络都能通过btih（种子的哈希）来搜索拥有相同资源的peers。  
>>弹幕信道也是一种资源，其URN自然也可以通过普通的BT/Magnet协议搜索。  
>>这一步不使用独立的协议是因为需要利用现有的巨大BT网络帮忙搜索，不然客户端太少难以组网。  

## 二、发布  

每条弹幕以其哈希值作为弹幕消息标识，弹幕应当封装随机用户UUID与发布时间戳用以除重。  
>>注：也就是说，弹幕由{用户生成的UUID自我标识,发布时用户本地机器的UTC时间，正文}组成。  
>>然后对这一条弹幕做哈希，生成的GUID是这一条弹幕的标识。  

扩展协议以仿BT的HAVE报文宣告它发布了新弹幕，其中弹幕哈希代替了HAVE报文的片段下标。  
>>注：一般BT协议HAVE报文是说，这个种子分成的###片之中，我已有了第$$$片。  
>>但是弹幕池是不断延长的，而且网络分布化之后给每条弹幕排序会频繁出现不一致。  
>>因此我们只用弹幕的GUID做标识，每当产生一条百余/数百字节的弹幕。  
>>产生一条16字节的哈希（当然还可以截断使之更短，以节省网络流量）。  
>>然后用与原始BT/Magnet协议相似的方式宣告自己HAVE这条弹幕。其他基本相同。  

支持弹幕扩展的客户端有义务接收并储存新的弹幕，接收完毕后也应发布新弹幕的HAVE报文。  
客户端有义务接受没有的弹幕，应当定时宣告HAVE所有所存弹幕，应当频繁宣告近期弹幕HAVE。  
>>注：事实上如果有专门的peer做日志服务器，普通客户端的储存和分享义务是低必要性的。  
>>但是为了弹幕网络的健壮性可抗攻击性，这个义务必须至少声明出来。  
>>在具体实现上，可以允许客户端判断，在网络环境良好，用户多且活跃的情况下，  
>>根据时间的久远性，通过协商或者随机的方式，删除自己储存的部分历史弹幕。  
>>这一过程称为风化。尽管有吸血之嫌，但是可以降低普通客户端的存储与带宽压力。  

因此，以上流程保证，经过充足的时间，每一条弹幕都能知会到每一个在线时间足够长的客户端。  
>>注：也就是说，有的弹幕只有部分人收到，并且这些人关机之后再也没有人收到。  
>>因此这一条弹幕最终没有保存在弹幕网络之中，这一现象称做湮灭。  
>>还有的弹幕很晚才被每个人收到。不过因为对于完整和及时性较宽容，这两点其实无所谓。  

## 三、传送

拥有弹幕的客户端接到请求有义务传送匹配的弹幕，弹幕可以用任何协议加密(鼓励)或不加密。  
>>注：除非弹幕有触发网关的关键词或者个人信息，否则加密意义不大，毕竟是公开发布的。  
>>但是这里仍然鼓励使用普通、多样、安全的协议传输，目的是增强P2P健壮性。  

为了防止长度扩展攻击，弹幕在封装的正文之后，应当有统一、明确、长度恰当的结尾标识符。  
为了混淆，必要时发送方可以在结尾标识符后面添加随机字符，但没有明确必要时不应这么做。  
为了防止一般碰撞攻击，接收方应当检查接收消息的哈希是否匹配，并从结尾标识符处截断。  
>>注：据说SHA1是扩展攻击不安全的，也就是在正文之后可以添加某些特殊字符串而不影响SHA1值。  
>>另一方面应对网关过滤哈希，那么定义一个EOF也不算全无意义的。  

因此，以上流程保证，弹幕能够安全、准确、可靠地传达到请求者的手中。  
>>注：这是说，只有客户端知悉这一弹幕的存在并请求，才能确实获得无错的正文。  

## 四、弹幕

一般情况下，弹幕正文应当用json封装，包括播放时间、显示位置、特效类型、显示文本等四要素。  
>>注：也就是类ACFUN弹幕，不过uid和timestamp已经提到正文之外（与正文地位并列）了。  
>>这一点不需要重复造轮子，只要是ACFUN/CommentCoreLibrary兼容格式就好。  

但是弹幕也可以封装其他信息，标准应当容许二次扩展，比如适用于漫画的页数、坐标、字号要素。  
>>注：例如有妖气、微漫画的按页漫画吐槽，轻小说文库按行轻小说吐槽，还有纯音乐的弹幕等等。  

当多个磁链对应不同压制和字幕的同一视频，可封装对其他弹幕池的引用，各分镜片段时间偏移量。  
>>注：这是用来对付同一视频，但来自不同录制源，不同字幕组，压制不同，存储在合集中等情况。  
>>因为以上原因，同一视频不但哈希不同，还可能出现起始点差数秒，中间插入广告等时间轴问题。  
>>解决方案是，对不同视频首先进行文件地址引用，即<btih1[:filename1]>:<btih2[:filename2]>  
>>接下来指出共同的片段：<video1_time01,video2_time01,duration01>,...,  
>>这些片段中的弹幕由两视频共享，要注意尽管片段结束了，共通弹幕可能还在书写中，可予宽限。  

>>注2：关于共同的片段，当然可以人工设定，但也可以由软件识别产生。  
>>事实上可以通过对视频进行分镜切割，并记录每个首帧哈希（比如SURF，SIFT专利没过期）  
>>将这个序列作为视频内容标识码，作为匹配同源视频的模糊搜索凭藉。  
>>不过如何实现和怎样提高识别率比较复杂，大概需要单独开一贴讨论，这里就不谈了。  

弹幕可以用作对其他弹幕表示支持或反对，这时应当封装对方弹幕哈希、对应用户的UUID，及态度。  
>>注：即云屏蔽。和弹幕池引用一样都是不可见弹幕。用来给予弹幕和弹幕池引用做好评、差评。  
>>弹幕正文中，特效类型可以给不可见弹幕保留一个标志，然后在显示文本里记录这些信息。  

普通弹幕被支持可放大字号改变颜色，被反对则可缩小及屏蔽，弹幕池引用如广受支持可自动开启。  
>>注：受到反对的弹幕会缩小字号、甚至屏蔽。差评越多越小。客户端可以连坐此作者的其他弹幕。  
>>也可以对支持的弹幕给予好评，不过放大字号可能影响该弹幕视觉效果，由客户端自己决定方式。  
>>对于恰当的弹幕池引用应当给予好评，这样其他人能够默认开启这个弹幕池引用，看到更多弹幕。  
>>在观看时，弹幕池引用被全程使用的，客户端可以自动生成好评,因为用户去评价可能不够积极。  
>>对于捣乱的弹幕池引用，因为差评多于好评，当然关闭。客户端可以设定好评不够多的也关闭。  
Proposal_0001 by Schezuk on 2015.02.11
