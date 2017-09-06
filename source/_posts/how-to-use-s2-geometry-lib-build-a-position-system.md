---
layout: post
title: google-s2简介
date: 2014-11-15 23:38:40
tags: s2
categories: 技术
---

s2-geometry-lib是一个谷歌开发的空间几何坐标库，提供很多空间操作的工具方法，包括Polygon、Loop、Line、Point等。
<!-- more -->
如果要定位一个用户所在的省市区，一般的思路是先构建每一个区域的Ploygon对象，然后根据用户的经纬度坐标，判断该经纬度属于哪一个区域中。这个操作每判断一个点都会进行遍历(最坏情况所有区域)和计算(区域是否contain)，这是非常消耗资源的，对现在互联网的一些服务，也很难满足性能需要。

利用s2库就可能很好的优化上面的问题，s2库提供了一个Cell的概念，每个Cell相当于一个单元格，在一个单元格内的所有Point，都可以算出同一个CellId。

同时可以认为，一个Polygon可以用一个Cell的集合来表示。

那么这时的步骤就是通过区域的坐标，来初始化其内包含的所有的CellId，将这些CellId的id放入redis中做key，对应的value是区域名称(譬如，上海浦东新区)，构建完全国范围的CellId后，查询用户的区域就可以通过用户的坐标获得一个CellId，然后去redis中查询，在使用LocalCache后，基本就没有什么计算成本，完全可以满足性能需求。


下面是判断点的CellId是否能够匹配到一个区域的某个CellId的例子

```
public class App {
    public static void main(String[] args) {
        S2RegionCoverer coverer = new S2RegionCoverer();
        ArrayList<S2CellId> covering = new ArrayList<S2CellId>();
        String city = "30:121,30:122,32:121,32:122";
        coverer.setMaxCells(100000);
        coverer.setMinLevel(11);//设定ZoomLevel
        // 获得能够覆盖city这个区域的所有CellId(Cell的集合表示的范围略微大于city的范围，外接四边形)
        coverer.getCovering(GeometryTestCase.makePolygon(city), covering);
        // 获得能够覆盖city这个区域的所有CellId(Cell的集合表示的范围略微小于city的范围，内接四边形)
        // coverer.getInteriorCovering(GeometryTestCase.makePolygon(city), covering);
        //GeometryTestCase是s2库的一个测试类，makePolygon方法本来是默认的，我改成公共的
        S2CellId s2cellid = S2CellId.fromLatLng(S2LatLng.fromDegrees(30.5, 121.5));
        S2CellId paCellId = s2cellid.parent(11);//获得相同ZoomLevel的CellId
        // 只有相同的ZoomLevel下才能找到相匹配的
        for (int i = 0; i < covering.size(); i++) {
            if (paCellId.id() == covering.get(i).id()) {
                System.out.println(covering.get(i).id());//数据匹配的CellId
            }
        }
    }
}
```
