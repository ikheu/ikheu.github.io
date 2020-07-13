---
layout: post
title:  Redis 中的对象系统
categories: 数据库
tags: Redis
---
* content
{:toc}

前文介绍了 Redis中的 基本数据结构，然而与用户直接打交道的并不是这些数据结构，而是 Redis 中构建的一套数据对象。数据对象使用了这些数据结构作为编码。

Redis 中的数据对象包括：
- 字符串 string
- 列表 list
- 哈希 hash
- 集合 set
- 有序集合 zset

数据编码包括：
- 整数 int
-  embstr
-  raw
- 哈希表 hashtab
- 链表 linkedlist
- 压缩列表 ziplist
- 整数集合 intset
- 跳跃表 skiplist

分层设计，抽象隔离，关注下一级提供的 api 或基本语言功能。

string list hash set zset

int embstr raw hashtab linkedlist ziplist intset

struct malloc pointer typedef

关注为什么采用这种编码，何时基于何种理由进行编码转化。

## 字符串对象

## 列表对象

## 哈希对象

## 集合对象

## 有序集合对象