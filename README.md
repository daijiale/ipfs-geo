![](http://career-pic.oss-cn-beijing.aliyuncs.com/ipfs-smart-object/ipfsgeo/ipfs-geo-dark.jpg)

# [ipfs-geo]

[![jaywcjlove/sb](https://jaywcjlove.github.io/sb/lang/chinese.svg)](README-zh.md)

 This repo is a research of **IPFS-GEO** (a geography relation object model demo with ipfs). Feel free to **star** and **fork**.

Any comments, suggestions? [Let us know!](https://github.com/daijiale/ipfs-geo). We love PRs :) Please take a look at the Contributing guidelines before opening one.

## Explanation

[English](README.md) | [中文](README-zh.md)

## 0\. Summarize

### 0.0 Project Goal

This project goal is to build a relation pattern with position information and block chain. Implement a ecological model for **people->position->real world->transfer trust->transfer worth->position->people** and use block chain to index real world.

### 0.1 Noun Explaination

- DDApp(Data Decentered Application): For resolve DApp can't rely on the center API, DDApp is a new concept that only the data is decentration. This concept not only can communicate with each other, but also can deploy independent and support p2p transfer.

- [IPFS(InterPlanetary File System)](https://ipfs.io/): An distributed file system.

- [GeoHash](https://en.wikipedia.org/wiki/Geohash): The algorithm can greater increase the searching efficiency in large position data, and support convenient cache for app.

## 1\. System Design

### 1.0 Architecture Design

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/IPFS-Geography.png)

### 1.1 Object Model Disign

property             | type     | comment
-------------------- | -------- | ---------------------------------
geo_id               | INT      | uniqueness identification
geo_address          | STRING   | address
geo_lng              | FLOAT    | longitude
geo_lat              | FLOAT    | latitude
geo_hash             | STRING   | geo hash, created by position
ipfs_hash            | STRING   | ipfs hash, created by saving data
addGeoInfoByParam()  | FUNCTION | add position message
getGeoInfoByParam()  | FUNCTION | get position message
mixGeoHashByParam()  | FUNCTION | get GeoHash
addIpfsDataByParam() | FUNCTION | add ipfs data
mixIpfsHashByParam() | FUNCTION | associate ipfs data

### 1.2 Database Object Map

#### 1.2.0 Chose Database

> There is a test report of search 1 million poi data by some friends.

Database Name     | Proc Time | Support Regional Search |     | Support Multi Condition
----------------- | --------- | ----------------------- | --- | -----------------------
redis(3.2.8)      | 1-10ms    | yes                     | no
mongo(3.4.4)      | 10-50ms   | yes                     | yes
postgreSQL(9.6.2) | 3-8ms     | yes                     | yes
mysql（5.7.18）     | 8-15ms    | yes                     | yes

**compare these statistics data, I chose MySql to support the geo system demo.**

- Older than MySql 5.7.4, the `myISAM` engine has `Geom` to implement this function.
- Newer than MySql 5.7.4, `InnoDB` engine also support for space index.

**PS:**

- 1. Not the best database, only have the suit database.

- 1. Researching the storage effiency, since the network node is test now and so instability that there have serious slow problem of transmission data, we can't reach `high frequency to update` and `millions of data to index` on IPFS.

- 1. Maybe we should find a efficiency plan to use IPFS under the current situation now.

#### 1.2.1 Object Model Map To Table Struct

```sql


-- table struct `geo_object`
--

CREATE TABLE `geo_object` (
  `geo_id` bigint(20) NOT NULL  AUTO_INCREMENT,
  `geo_loc` point NOT NULL,
  `geo_address` varchar(255) NOT NULL,
  `ipfs_hash` varchar(255) NOT NULL
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COMMENT='geo object model';

-- Indexes for table `geo_object`
ALTER TABLE `geo_object`
  ADD PRIMARY KEY (`geo_id`),
  ADD SPATIAL KEY `geo_loc` (`geo_loc`);
```

## 2\. Demo

Base on these concept and design model above, now we can implement a simple demo:

### 2.0 Upload Position Data By IPFS

- The detail of deploy a single node of ipfs can refer to [use ipfs to build a decentration wiki system](https://www.daijiale.cn/%E5%8C%BA%E5%9D%97%E9%93%BE/%E3%80%90%E5%8C%BA%E5%9D%97%E9%93%BE%E3%80%91%E5%88%A9%E7%94%A8ipfs%E6%9E%84%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84%E5%8E%BB%E4%B8%AD%E5%BF%83%E5%8C%96%E5%88%86%E5%B8%83%E5%BC%8Fwiki%E7%B3%BB%E7%BB%9F.html)

- Now the official support `curl` method, we can use `addIpfsDataByParam()` to implement RPC.

  ```
  curl -F file=@myGeoFile "http://localhost:5001/api/v0/add?recursive=false&quiet=false&hash=sha2-256"
  ```

- **PS:** This demo use location node to upload data, for protect the service available, we suggest use **ipfs-cluster** and the detail can refer to the article [ipfs family II](http://ipfser.org/2018/03/02/r26/).

### 2.1 Get Data From IPFS and Relate it.

The response data is 'multipart/form-data' type. If success, these data would be recived:

```json
{
    "Name":"myGeoFile"
    "Hash":"QmYftndCvcEiuSZRX7njywX2AGSeHY21Sa7VryCq1mK1Ew"
    "Bytes":"2428803"
    "Size": ""
}
```

After get the hash, we can use `mixIpfsDataByParam()` function to relate our Geo position data.

### 2.2 Run The Geo demo

#### Choice The First Base Geo Position (Simulate the position of user)

![Alt text](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-poi-3w.png)

- latitude: 39.989049
- longitude: 116.313658

```sql
INSERT INTO `geo_object`(`geo_loc`, `geo_address`, `ipfs_hash`) VALUES (GeomFromText('POINT(39.989049 116.313658)'),'3W Coffee','QmYftndCvcEiuSZRX7njywX2A21Sa7VryCq1mK1Ew21')
```

#### Choice The Second Geo Position (Simulate the close position)

![Alt text](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-poi-zhongguancun.png)

- latitude：39.988878
- longitude：116.313352

  ```sql
  INSERT INTO `geo_object`(`geo_loc`, `geo_address`, `ipfs_hash`) VALUES (GeomFromText('POINT(39.988878 116.313352)'),'Sourth Street of Zhong Guan Village','WCJIEFSCvcE231233HY21Sa7Vr1Cq1mK1Ew')
  ```

#### Choice The Third Geo Position(Simulate the far position)

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-poi-ymy.png)

- latitude：40.005466
- longitude：116.315938

```sql
INSERT INTO `geo_object`(`geo_loc`, `geo_address`, `ipfs_hash`) VALUES (GeomFromText('POINT(40.005466 116.315938)'),'Old Summer Palace','KBYftndCvcEiuSZRX7njyw1332Y21Sa723mKASDED')
```

#### Sphere Distance Algorithm

Assume the diagonal points of sphere fence is A1(x1, y1), B1(x2, y2):

```
x1 = lat + distance / ( 111.1 / COS(RADIANS(lng))),  
y1 = lng + distance / 111.1  

x1 = lat - distance / ( 111.1 / COS(RADIANS(lng))),  
y1 = lng - distance / 111.1  

// build the first order space filling curve
LineString(A1,B1)
```

**PS:**

- 1. One longitude of the equator is about equal 111.1km.

- 1. RADIANS() is a function to compute radians.

- 1. LineString() is a function to build the first order space filling curve.

### 2.3 Get The IPFS Data In A Region

#### Get The IPFS Data Within 1 KM Distance

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

As shown in the following figure, we get the ipfs data within 1 km distance of `3W Coffee` ![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-search-1km.png)

#### Get The IPFS Data Within 10 KM Distance

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

As shown in the following figure, we get the ipfs data within 10 km distance of `3W Coffee` ![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/geo-search/geo-search-10km.png)

**PS:** About this demo detail, we will write a new article to describe these.

## 3 Application Scene

- **Vevue:** A dapp to encourage user to take a picture on special location and reward some token.
- **sign in by position:** only arrive special location can reward some things or get some proof.
- **money in store:** attract people to go to some store and reward some money.
- **protect parking space:** protect some parking space that someone has bought.
- **unite with AR game:** AR game unite with position and reware some token.
- **IOT:** some IOT have position property, these may unite with this system.

## 4 Open Source Plan

**goal:** Expect everyone can realize some application scene by ipfs even block chain. Use technology to crate real value. Welcome everyone to join.

### IPFS-Geo

**meaning:** This is a intelligent IPFS object and it's meta data has Geo property and support fast index `10 millions space data`. It also provides LBS function api.

![](http://career-pic.oss-cn-beijing.aliyuncs.com/ipfs-smart-object/ipfsgeo/ipfs-geo-dark.jpg)

## reference

- [Wikipedia GeoHash](https://en.wikipedia.org/wiki/Geohash)
- [block chain blog by YueChunLi](http://liyuechun.org/2017/11/22/ipfs-api/#35-%E5%AE%89%E8%A3%85`ipfs-api`)
- [GeoHash core theory](https://www.cnblogs.com/LBSer/p/3310455.html)
- [【application】use ipfs to build a decentration wiki system](https://www.daijiale.cn/%E5%8C%BA%E5%9D%97%E9%93%BE/%E3%80%90%E5%8C%BA%E5%9D%97%E9%93%BE%E3%80%91%E5%88%A9%E7%94%A8ipfs%E6%9E%84%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84%E5%8E%BB%E4%B8%AD%E5%BF%83%E5%8C%96%E5%88%86%E5%B8%83%E5%BC%8Fwiki%E7%B3%BB%E7%BB%9F.html)
- [Vevue：a DApp to take photos in special position and reward tokens](http://cn.vevue.com/)
