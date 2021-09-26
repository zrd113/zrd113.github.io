---
layout: post
title: redis-整数集合
date: 2021-06-03
Author: zrd
tags: [redis]
toc: true
---

整数集合是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。

## 一、 整数集合的定义

整数集合可以保存类型为int16_t、int32_t或者int64_t的整数值，并且保证集合中不会出现重复元素。
```
//每个intset结构表示一个整数集合
typedef struct intset{
    //编码方式
    uint32_t encoding;
    //集合中包含的元素数量
    uint32_t length;
    //保存元素的数组
    int8_t contents[];
} intset;
```
1. contents数组是整数集合的底层实现：每个元素都是contents数组的一个数组项，各个项在数组中按值的大小从小到大有序地排列，并且数组中不包含任何重复项。
2. length属性记录了整数集合包含的元素数量，也即是contents数组的长度。
3. 虽然contents属性声明为int8_t类型的数组，但实际上contents数组并不保存任何int8_t类型的值，contents数组的真正类型取决于encoding属性的值：
   如果encoding属性的值为INTSET_ENC_INT16，那么contents就是一个int16_t类型的数组，数组里的每个项都是一个int16_t类型的整数值。
   如果encoding属性的值为INTSET_ENC_INT32，那么contents就是一个int32_t类型的数组，数组里的每个项都是一个int32_t类型的整数值。
   如果encoding属性的值为INTSET_ENC_INT64，那么contents就是一个int64_t类型的数组，数组里的每个项都是一个int64_t类型的整数值。

## 二、 整数集合的升级

每当我们要将一个新元素添加到整数集合里面,并且新元素的类型比整数集合现有所有元素的类型都要长时,整数集合需要先进行升级,然后才能将新元素添加到整数集合里面。升级整数集合并添加新元素主要分三步来进行：
    1. 根据新元素的类型,扩展整数集合底层数组的空间大小,并为新元素分配空间。
    2. 将底层数组现有的所有元素都转换成与新元素相同的类型,并将类型转换后的元素放置到正确的位上,而且在放置元素的过程中,需要继续维持底层数组的有序性质不变。
    3. 将新元素添加到底层数组里面。

    注意：整数集合不支持降级。

## 二、 整数集合升级的优点

### 1. 提升灵活性

因为C语言是静态类型语言,为了避免类型错误,我们通常不会将两种不同类型的值放在同一个数据结构里面。但是,因为整数集合可以通过自动升级底层数组来适应新元素,所以我们可以随意地将int16_t、int32_t或者int64_t类型的整数添加到集合中,而不必担心出现类型错误,这种做法非常灵活。

### 2. 节约内存

要让一个数组可以同时保存int16_t、int32_t、int64_t三种类型的值,最简单的做法就是直接使用int64t类型的数组作为整数集合的底层实现。不过这样一来,即使添加到整数集合里面的都是int16_t类型或者int32_t类型的值,数组都需要使用int64_t类型的空间去保存它们,从而出现浪费内存的情况。
而整数集合现在的做法既可以让集合能同时保存三种不同类型的值,又可以确保升级操作只会在有需要的时候进行,这可以尽量节省内存。
