---
layout: post
title: "Another key value store - 1"
description: "A simple key value store written in Haskell"
category:
tags: [database, haskell]
---
{% include JB/setup %}

This is the first part in the series of posts describing a simple key value store written in haskell.
The code is available on [github](https://github.com/igniting/keyvalue).

###Why another keyvalue store?
The aim was not to implement another key value store, but to implement a keyvalue store in haskell. Infact, this keyvalue database is inspired from [Bitcask](https://github.com/basho/bitcask). Bitcask is based on the idea of Log-Structured Hash Table.

[Haskell](https://www.haskell.org) is purely functional, statically typed and lazy language. Writing a program in Haskell requires thinking functionally and is quite different from programming in imperative languages like C++ and Java.

###Basic idea behind Bitcask
Bitcask stores all the keys in a hash table in memory. This is also one of the limitations of Bitcask that all of it's keys should fit in memory. The hash table contains the location where exactly is the key stored. So when we want to read a key's value, only one disk seek is required to read the value from the disk.

Writes are append only. There is only one active file at any moment, and the key and value is appended to this file. To delete a key, the key is inserted with a special "tombstone" value.

This simple model may use a lot of space over time, hence there is a merge procedure, which merges all the log files into one, deleting all duplicate entries.

Each log file also has a corresponding hint file which is used when the database is initialized. This hint file contains the key and it's location in the log file. While initializing the database, only the hint files are read.
