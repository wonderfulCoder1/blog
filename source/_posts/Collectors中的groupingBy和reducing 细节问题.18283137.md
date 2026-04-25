---
title: Collectors中的groupingBy和reducing 细节问题
date: 2026-04-25 18:18:42
categories:
  - bug
tags:
  - bug
  - java
---



>stream流中对数据进行先分组在聚合，一般会想到使用groupingBy和reducing，但是reducing中的`identity`是只会初始化一次的，所以我们传参的时候传的是`Object`，不是`XXX::new`,在reducing的合并函数中我们不能返回vo1或者vo2，只能new一个对象
## 正确使用
```java
Map<String, StatisticsVo> collect = statisticsVos.collect(Collectors.groupingBy(e -> e.getMaterialName(), Collectors.reducing(new RawMaterialStatisticsVo(), (vo1, vo2) -> {
            // TODO 这里必须返回一个新对象，而不是修改vo1
            // System.out.println(System.identityHashCode(vo1)); 
            // System.out.println(vo1);
            // System.out.println(vo2);
            // System.out.println("---------");
            return StatisticsVo.add(vo1, vo2);
        })));
```
### 实体中的add方法
```java
public static StatisticsVo add(StatisticsVo vo1,StatisticsVo vo2) {
    StatisticsVo vo = new StatisticsVo();
    if(StrUtil.isEmpty(vo1.getMaterialName())){
        vo.setMaterialName(vo2.getMaterialName());
    }else {
        vo.setMaterialName(vo1.getMaterialName());
    }
    vo.setTotalAmount(NumberUtil.add(vo1.getTotalAmount(),vo2.getTotalAmount()));
    return vo;
}
```
参考： [java-stream-groupby-and-reduce](https://stackoverflow.com/questions/73755679/java-stream-groupby-and-reduce)