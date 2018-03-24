![](http://career-pic.oss-cn-beijing.aliyuncs.com/ipfs-smart-object/ipfsgeo/ipfs-geo-dark.jpg)

# ipfs-geo

[TOC]

## 一、概述

### 1.1 项目意义

打造地理位置信息与区块链的关系对象模型，建立一套
**人->位置->真实世界->传递信任->价值转移->位置->人**  的生态模型，实现用区块链来索引真实世界的愿景。

而GeoHash算法可以大幅度提高在庞大位置数据中的检索效率，同时为应用提供便捷的缓存机制。

IPFS&Filecoin技术则可以保证在一个可信的区块链网络中去大规模传递与海量位置信息相关联的海量文件、数据集合，并保证传递过程中数据的产权价值。

### 1.2 名词解释

#### DDApp

DDApp ( Data Decentered Application )：是一个**数据去中心化应用**的概念，介于传统应用和去中心化应用之间，解决了DApp不能依赖中心化的API的问题，又保证部分需要去中心化场景下的数据，在与应用交互之外，还可以独立分布部署、P2P传输。

#### IPFS

IPFS全称InterPlanetary File System，中文名：星际文件系统，是一个旨在创建持久且分布式存储和共享文件的网络传输协议。它是一种内容可寻址的对等超媒体分发协议可以让网络更快、更安全、更开放。它是一个面向全球的、是一个点对点的分布式版本文件系统，试图将所有具有相同文件系统的计算设备连接在一起。


#### GeoHash

Geohash是由Gustavo Niemeyer发明的公共域地理编码系统，它将一个地理位置编码成一串字母和数字。

它是一种层次化的空间数据结构，将空间细分为网格形状的桶，是一种被称为z -阶空间填充曲线的应用，下图中就是GeoHash算法中常用的Peano曲线，一种四叉树线性编码方式。

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/suanfa-geo-hash.png)

GeoHash数据将具有如下3个特点：
 - 1）GeoHash将二维的经纬度转换成字符串，比如下图展示了北京9个区域的GeoHash字符串，分别是WX4ER，WX4G2、WX4G3等等，每一个字符串代表了某一矩形区域。也就是说，这个矩形区域内所有的点（经纬度坐标）都共享相同的GeoHash字符串，这样既可以保护隐私（只表示大概区域位置而不是具体的点），又比较容易做缓存，比如左上角这个区域内的用户不断发送位置信息请求餐馆数据，由于这些用户的GeoHash字符串都是WX4ER，所以可以把WX4ER当作key，把该区域的餐馆信息当作value来进行缓存，而如果不使用GeoHash的话，由于区域内的用户传来的经纬度是各不相同的，很难做缓存。

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/weilan1-geo-hash.png)

 -  2）字符串越长，表示的范围越精确。如图所示，5位的编码能表示10平方千米范围的矩形区域，而6位编码能表示更精细的区域（约0.34平方千米）
 
![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/weilan2-geo-hash.png)

- 3）字符串相似的表示距离相近（特殊情况后文阐述），这样可以利用字符串的前缀匹配来查询附近的POI信息。如下两个图所示，一个在城区，一个在郊区，城区的GeoHash字符串之间比较相似，郊区的字符串之间也比较相似，而城区和郊区的GeoHash字符串相似程度要低些。

`例如，坐标对（116.414597,39.955441），位于北京安定门附近，GeoHash后形成的值为WX4G2`

我们已经知道现有的GeoHash算法使用的是Peano空间填充曲线，这种曲线会产生突变，造成了编码虽然相似但距离可能相差很大的问题，因此在基于个人位置查询附近Poi信息时，首先筛选GeoHash编码相似的POI点，然后进行实际距离计算，来规避算法突变所造成的误差。

当然Geohash只是空间索引的一种方式，特别适合POI点数据，而对线Link、面数据采用R树索引更有优势



## 二、系统设计


### 2.1 架构设计

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/IPFS-Geography.png)


### 2.2 对象模型设计

**Geo Object Model**
|属性|类型|备注|
|-|-|-|
|geo_id|INT|唯一标识|
|geo_address|STRING|地址名|
|geo_lng|FLOAT|位置经度|
|geo_lat|FLOAT|位置纬度|
|geo_hash|STRING|位置生成的GeoHash值|
|ipfs_hash|STRING|所存数据的IpfsHash值|
|addGeoInfoByParam()|FUNCTION|添加位置信息方法|
|getGeoInfoByParam()|FUNCTION|获取位置信息方法|
|mixGeoHashByParam()|FUNCTION|GeoHash生成算法|
|addIpfsDataByParam()|FUNCTION|添加Ipfs数据方法|
|mixIpfsHashByParam()|FUNCTION|关联Ipfs数据方法|


### 2.3 数据库对象映射


#### 2.3.1 数据库选型


>这是网友以 100万 poi 数据查询范围 3km 内的点（最多取100条）的性能测试统计

**以下是各数据库的对比情况：**

|数据库|耗时|区域查询|多条件支持|
|-|-|-|-|
|redis(3.2.8)|1-10ms|支持|不支持|
|mongo(3.4.4)|10-50ms|支持|支持|
|postgreSQL(9.6.2)|3-8ms|支持|支持|
|mysql（5.7.18）|8-15ms|支持|支持|

**综合比较后，个人选择了MySql 来进行后文Demo的支撑数据库**
 - MySql在5.7.4以前版本的童鞋可以通过myISAM引擎提供的Geom内置函数来实现
 - MySql在5.7.4以后版本的童鞋可以舒服的继续使用InnoDB引擎，官方对其添加了对空间索引的支持，感兴趣的朋友也可以对比下性能。

**PS：** 
- 1 . 数据库没有哪个一定好，只要适合场景即可。
- 2 . 在研究IPFS存储性能的过程中，由于测试网络节点问题，有很严重的数据传输瓶颈，且不稳定，短期内，很难将**需要频繁更新**以及**百万级别数据**的检索逻辑事务放在IPFS这一层中来做。
- 3 . 在IPFS节点网络性能目前并不乐观的情况下，尝试去寻找能实现具有商业级别能力IPFS应用的**过渡方案**



#### 2.3.2 对象模型映射成表结构


```sql


-- 表的结构 `geo_object`
--

CREATE TABLE `geo_object` (
  `geo_id` bigint(20) NOT NULL  AUTO_INCREMENT,
  `geo_loc` point NOT NULL,
  `geo_address` varchar(255) NOT NULL,
  `ipfs_hash` varchar(255) NOT NULL
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COMMENT='geo对象模型';

-- Indexes for table `geo_object`
ALTER TABLE `geo_object`
  ADD PRIMARY KEY (`geo_id`),
  ADD SPATIAL KEY `geo_loc` (`geo_loc`);

```




## 三、Demo

有了以上的概念和设计模型，接下来，给大家看一个简单的Demo实现：

### 3.1 通过IPFS上传位置数据

- IPFS单节点的部署就不详细介绍了，这边可以参考文章[【应用】利用IPFS构建自己的去中心化分布式Wiki系统](https://www.daijiale.cn/%E5%8C%BA%E5%9D%97%E9%93%BE/%E3%80%90%E5%8C%BA%E5%9D%97%E9%93%BE%E3%80%91%E5%88%A9%E7%94%A8ipfs%E6%9E%84%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84%E5%8E%BB%E4%B8%AD%E5%BF%83%E5%8C%96%E5%88%86%E5%B8%83%E5%BC%8Fwiki%E7%B3%BB%E7%BB%9F.html) 的实现过程

- 官方提供了Curl的API方式，我们可以通过`addIpfsDataByParam()`方法实现RPC调用。
```
curl -F file=@myGeoFile "http://localhost:5001/api/v0/add?recursive=false&quiet=false&hash=sha2-256"
```


- **PS：**这边Demo采用的是本地单节点的数据上传，为了保障服务的稳定性，建议使用**ipfs-cluster的节点集群解决方案**，具体方案可以参考**IPFS资深大牛（飞向未来）**的文章[《IPFS家族二》](http://ipfser.org/2018/03/02/r26/)


### 3.2 获取IPFS网络返回值，并关联数据

响应体为 ‘multipart/form-data’格式，成功，会返回如下Body数据：
```json
{
    "Name":"myGeoFile"
    "Hash":"QmYftndCvcEiuSZRX7njywX2AGSeHY21Sa7VryCq1mK1Ew"
    "Bytes":"2428803"
    "Size": ""
}
```

拿到Hash值后，再通过`mixIpfsDataByParam()`方法关联到我们的Geo位置数据上


### 3.3 Geo模型预演

#### 选取第一个基准位置点（模拟用户所在定位）
![Alt text](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-poi-3w.png)

 - 维度lat：39.989049
 - 经度lng：116.313658


```sql
INSERT INTO `geo_object`(`geo_loc`, `geo_address`, `ipfs_hash`) VALUES (GeomFromText('POINT(39.989049 116.313658)'),'3W咖啡馆','QmYftndCvcEiuSZRX7njywX2A21Sa7VryCq1mK1Ew21')
```


#### 选取第二个Geo位置点（模拟近点）

![Alt text](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-poi-zhongguancun.png)

 - 维度lat：39.988878
 - 经度lng：116.313352
```sql
INSERT INTO `geo_object`(`geo_loc`, `geo_address`, `ipfs_hash`) VALUES (GeomFromText('POINT(39.988878 116.313352)'),'中关村创业大街南广场','WCJIEFSCvcE231233HY21Sa7Vr1Cq1mK1Ew')
```


#### 选取第三个Geo位置点（模拟远点）

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-poi-ymy.png)

 - 维度lat：40.005466
 - 经度lng：116.315938
 
```sql
INSERT INTO `geo_object`(`geo_loc`, `geo_address`, `ipfs_hash`) VALUES (GeomFromText('POINT(40.005466 116.315938)'),'圆明园','KBYftndCvcEiuSZRX7njyw1332Y21Sa723mKASDED')
```

#### 球面距离围栏算法

假设球面围栏对角点坐标A1（x1，y1），B1（x2，y2）：

```
x1 = lat + distance / ( 111.1 / COS(RADIANS(lng))),  
y1 = lng + distance / 111.1  

x1 = lat - distance / ( 111.1 / COS(RADIANS(lng))),  
y1 = lng - distance / 111.1  

//构建一阶空间填充曲线
LineString(A1,B1)  
                           
```

**PS：**
- 1.赤道上经度的每个度大约相当于111.1km，经度的每个度的距离从0km到111.1km
- 2.RADIANS()为弧度计算内置函数
- 3.LineString() 为构建一阶空间填充曲线内置函数ANA

### 3.4 获取地理区域内的IPFS数据服务


#### 获取1km以内的IPFS数据

```sql
SELECT  *  
    FROM    geo_object  
    WHERE   MBRContains  
                    (  
                    LineString  
                            (  
                            Point  
                                    (  
                                    39.989049 + 1 / ( 111.1 / COS(RADIANS(116.313658))),  
                                    116.313658 + 1 / 111.1  
                                    ),  
                            Point  
                                    (  
                                    39.989049 - 1 / ( 111.1 / COS(RADIANS(116.313658))),  
                                    116.313658 - 1 / 111.1  
                                    )   
                            ),  
                    geo_loc  
                    )  

```

如下图所示，我们拿到了距离`3W咖啡馆1Km以内中关村大街南广场附近`相关联的IPFS数据
![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-search-1km.png)


#### 获取10km以内的IPFS数据

```sql
SELECT  *  
    FROM    geo_object  
    WHERE   MBRContains  
                    (  
                    LineString  
                            (  
                            Point  
                                    (  
                                    39.989049 + 10 / ( 111.1 / COS(RADIANS(116.313658))),  
                                    116.313658 + 10 / 111.1  
                                    ),  
                            Point  
                                    (  
                                    39.989049 - 10 / ( 111.1 / COS(RADIANS(116.313658))),  
                                    116.313658 - 10 / 111.1  
                                    )   
                            ),  
                    geo_loc  
                    )  

```

如下图所示，我们拿到了距离`3W咖啡馆10Km以内圆明园附近`相关联的IPFS数据

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-search-10km.png)


**PS：**
关于Demo这块，后续会另外新开一篇实战文章[【应用】基于IPFS和GeoHash构建具有地理位置价值服务的DDApp（实战篇）]()来做专门介绍，让大家也能自己动手编写一个功能相对完善（可视化界面）DDApp 。




## 四、应用场景

- **Vevue：**一个在选定的区域内街拍可获得代币奖励的DApp，鼓励用户分享原创内容，激励场景化广告。
- **地理位置签到：**只有到达指定位置坐标点，才可取得可信签到密码凭证，进行核对，确认地理位置信任问题。(滴滴加班公司打车报销场景)
- **到店红包（糖果）：**，吸引用户到达指定实体店铺位置，通过位置可信核对，分发代币糖果凭证/快照，激励用户到店消费，体验现场活动。
- **车主停车位产权保护：**经过购买的专用停车位实为车主用户的资产，应受到产权保护，且车位的转移、交易需要在一套依赖地理位置的信任体系中进行。
- **景区、名胜古迹、历史遗迹信息保护：**沧海桑田，地转星移，也许有一天名胜古迹不复存在，但它们的电子信息（地理位置、图像、所属国家、历史文化、视频、VR全景等信息）将永远被保存在区块链上，真实且不被篡改，源远流长。
- **与位置AR游戏结合：**之前很火的Pokémon Go如果再加上Filecoin的奖励机制会是一种什么样的场面？也可以参考MANA区块链项目的价值。
- **物联网结合：** 充电桩，ETC这些具有支付属性、位置属性的智能设备创新等等。


## 五、开源计划

**初衷：**期望能让大家看到区块链的实际应用场景，为区块链和传统技术的结合做更多预演、布道、分享，不去听币圈熙熙攘攘的声音，用技术创造真实的价值，也期待更多和我一样想法的朋友加入，带一些正能量给这个圈子。

### IPFS-Geo

**意义：**是一个具有地理位置特征的IPFS智能对象，其元数据具备Geo相关特性，支持**千万级别空间数据**的快速索引，对象内还提供LBS相关功能的接口服务。

![](http://career-pic.oss-cn-beijing.aliyuncs.com/ipfs-smart-object/ipfsgeo/ipfs-geo-dark.jpg)

 

## 参考文献

- [Wikipedia GeoHash](https://en.wikipedia.org/wiki/Geohash)
- [黎跃春区块链博客](http://liyuechun.org/2017/11/22/ipfs-api/#35-%E5%AE%89%E8%A3%85`ipfs-api`)
- [GeoHash核心原理解析](https://www.cnblogs.com/LBSer/p/3310455.html)
- [【应用】利用IPFS构建自己的去中心化分布式Wiki系统](https://www.daijiale.cn/%E5%8C%BA%E5%9D%97%E9%93%BE/%E3%80%90%E5%8C%BA%E5%9D%97%E9%93%BE%E3%80%91%E5%88%A9%E7%94%A8ipfs%E6%9E%84%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84%E5%8E%BB%E4%B8%AD%E5%BF%83%E5%8C%96%E5%88%86%E5%B8%83%E5%BC%8Fwiki%E7%B3%BB%E7%BB%9F.html)
- [Vevue：一个在选定的区域内街拍可获得代币奖励的DApp](http://cn.vevue.com/)





