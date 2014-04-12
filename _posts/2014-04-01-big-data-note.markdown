---
layout: post
title:  "初识大数据"
date:   2014-04-01 14:06:25
categories: hadoop ruby bigdata
---

## 大数据定义

- 数据量：TB以上
- 数据格式：结构数据、非结构数据

## 大数据处理

- 数据存储：保存所有的数据
- 数据处理：以廉价的方式处理所有的数据，不论数据有多大，并能进行廉价的扩展

## hadoop的核心

- mapreduce：提供基于map-reduce思想的api，能有效的分解大型的计算任务
- hdfs：分布式文件存储系统，提供对*largefile*的分布式存储，对外提供统一的访问接口

## hadoop生态系统

- hbase：基于hadf的数据库系统
- hive、pig：数据查询组件，提供类似sql和脚本的查询语法，将查询工作分解为mapreduce任务
- sqoop：提供对传统关系型数据库和hbase间的映射，做到异构数据库间数据的导入导出

## 效能

- 处理能力随节点的增加线性增长
- 处理效率随节点的增加有轻微损失，当节点达到10个以上时，基本稳定在80%，即10个节点为1个节点的8倍

## mapreduce的基本原理

这里推荐大家查看这里的[视频](http://pragprog.com/screencasts/v-jamapr/processing-big-data-with-mapreduce), 里面通过扑克牌的方式演绎了mapreduce的实现原理  
非常生动，非常清楚
  
## ruby与mapreduce

- 只能使用streaming API
- 会有10-30%的性能损失
- 只适合基于文本的工作流

*例子*

```ruby

# mapper
ARGF.each do |line|
  line = line.chomp

  # 数据清洗，排除掉非数字的牌
  if match = line.match(/(.*) (\d+)/)
    key, value = match.captures
    puts key.downcase + "\t" + value
  end
end

# 这里的关键就是hadoop平台能够在每一个节点上  
# 对mapper输出的键值对进行排序和聚类, 并根据数据大小进行合并  
# 最终将数据按照key的顺序输出到reducer

# reducer
cardsum = 0
cardcolor = nil

ARGF.each do |line|
  line = line.chomp

  parts = line.split("\t")

  if parts.length != 2
    next
  end

  newcardcolor = parts[0]
  cardnum = Integer(parts[1])

  if cardcolor.nil?
    cardcolor = newcardcolor
  end

  # 由于stream api的限制，必须自己对key进行判断
  # 并将reducer的数据进行输出
  if cardcolor != newcardcolor 
    puts cardcolor + "\t" + cardsum.to_s()
    cardcolor = newcardcolor
    cardsum = 0
  end

  cardsum = cardsum + cardnum
end

unless cardcolor.nil?
  puts cardcolor + "\t" + cardsum.to_s()
end 
```
