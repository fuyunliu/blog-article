---
title: Python 通过 Thrift 操作 Hbase
date: 2018-11-14 17:04:20
categories:
- 笔记
tags:
- Python3
---

<!-- toc -->

记录 `Python` 通过 `Thrift` 操作 `Hbase` 的通用操作方法。

<!-- more -->

```python
# -*- coding: utf-8 -*-

from thrift.transport import TSocket
from thrift.protocol import TBinaryProtocol
from thrift.transport import TTransport
from elasticsearch import Elasticsearch
from hbase import Hbase

# Connect to HBase Thrift server
transport = TTransport.TBufferedTransport(TSocket.TSocket('localhost', 9090))
protocol = TBinaryProtocol.TBinaryProtocolAccelerated(transport)

# Create and open the client connection
client = Hbase.Client(protocol)
transport.open()

# Connect to Elasticsearch server
es = Elasticsearch('localhost',
                   http_auth=('username', 'password'), port='9200',
                   timeout=30, max_retries=10, retry_on_timeout=True
                   )


def fetch_one(index, doc_type, body, size=1):
    """查询es获取第一条匹配的数据

    Arguments:
        index {str} -- 索引
        doc_type {str} -- 类型
        body {dict} -- 查询语句

    Keyword Arguments:
        size {int} -- 返回数量 (default: {1})

    Returns:
        dict -- 一条数据，没有结果返回 None
    """

    res = es.search(index=index, doc_type=doc_type,
                    scroll='2m', body=body,
                    size=size)
    hits = res['hits']['hits']
    return hits[0] if hits else None


def fetch_all(index, doc_type, body, size=100):
    """查询es获取所有匹配的结果

    Arguments:
        index {str} -- 索引
        doc_type {str} -- 类型
        body {dict} -- 查询语句

    Keyword Arguments:
        size {int} -- 返回数量 (default: {100})

    Returns:
        list -- 结果集
    """

    res = es.search(index=index, doc_type=doc_type,
                    scroll='2m', body=body,
                    size=size)
    return res['hits']['hits']


def build_term(field, value):
    """term

    Arguments:
        field {str} -- 字段
        value {str} -- 值

    Returns:
        dict -- 查询语句
    """

    body = {
        "query": {
            "term": {
                field: {
                    "value": value
                }
            }
        }
    }
    return body


def build_terms(field, values):
    """terms

    Arguments:
        field {str} -- 字段
        values {list} -- 列表

    Returns:
        dict -- 查询语句
    """

    body = {
        "query": {
            "terms": {
                field: values
            }
        }
    }
    return body


def get_row_with_columns(table_name, rowkey, columns):
    """根据 rowkey 从 hbase 获取一条数据

    Arguments:
        table_name {str} -- 表名
        rowkey {str} -- rowkey
        attributes {list} -- 属性列表

    Returns:
        dict -- 一条数据，没有则返回None
    """
    table_name = table_name.encode()
    rowkey = rowkey.encode()
    columns = [('0:' + c).encode() for c in columns]
    res = client.getRowWithColumns(table_name, rowkey, columns, None)

    if not res:
        return None
    d = {
        k.decode().split(':')[1]: v.value.decode()
        for k, v in res[0].columns.items()
    }
    d['rowkey'] = res[0].row.decode()
    return d


def get_rows_with_columns(table_name, rowkeys, columns):
    """根据 rowkeys 从 hbase 获取所有匹配的数据

    Arguments:
        table_name {str} -- 表名
        rowkeys {list} -- rowkey 列表
        columns {list} -- 指定返回字段

    Returns:
        list -- 数据结果集
    """
    data = []
    table_name = table_name.encode()
    rowkeys = [k.encode() for k in rowkeys]
    columns = [('0:' + c).encode() for c in columns]
    res = client.getRowsWithColumns(table_name, rowkeys, columns, None)

    for r in res:
        d = {
            k.decode().split(':')[1]: v.value.decode()
            for k, v in r.columns.items()
        }
        d['rowkey'] = r.row.decode()
        data.append(d)
    return data

```
