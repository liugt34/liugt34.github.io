---
title: Postgresql Import Export Shapefile
date: 2023-02-15 11:33:00 +0800
categories: [Postgresql, PostGIS]
tags: [shapefile]
mermaid: true
---

some times we want to import a shapefile to the postgresql or export shapefile from postgresql, so we can do it with the bash command 


# import 

**template**

``` bash
shp2pgsql -s [<srid>] -a -W GBK [<file_name>] | psql -h [<server_host>] -U [<user_name>] -d [<db_name>] -p 5432
```

**demo**

``` bash
shp2pgsql -s 3857 -d -W GBK nyc_homicides.shp  | psql -h 192.168.1.1 -U postgres -d mydb -p 5433
```

**参数**

| 参数 | 含义 |
| ---- | ---- |
| -s|空间参考标识符（[SRID](https://epsg.io/)）|
|-d|重新建立表，并插入数据|
|-a|在同一个表中增加数据|
|-c|建立新表，并插入数据|
|-p|只创建表|
|-g|指定要创建的表的空间字段名称(在追加数据时有用)|
|-D|使用dump方式，比缺省生成sql的速度快|
|-G|使用类型geography|
|-k|保持标识符（列名，模式，属性）大小写。|
|-i|将所有整型都转为标准的32-bit整数|
|-I|在几何列上建立GIST索引|
|-S|生成简单几何，而非MULTI几何|
|-t|指定几何的维度|
|-w|指定输出格式为WKT|
|-W|输入的dbf文件编码方式|
|-N|指定几何为空时的操作|
|-n|只导入dbf文件|
|-T|指定表的表空间|
|-X|指定索引的表空间|
|-?|帮助|



# export

**template export all data**

```bash
pgsql2shp [<options>] <database> [<schema>.]<table>
```

**demo**

```bash
pgsql2shp -f E:/shp/nyc_streets.shp -h localhost -u postgres -P 123456 -p 5433 Testpg  public.nyc_streets;
```


**template export data use query**

```
pgsql2shp [<options>] <database> <query>
```

 **demo**

```bash
pgsql2shp -f E:/shp/nyc_streets.shp -h localhost -u postgres -P 123456 -p 5433 Testpg  "SELECT * from  nyc_streets";
```


**parameters**

| 参数 | 含义 |
| ---- | ---- |
|-f <filename>|	导出的shp文件名称|
|-h <host>|	主机地址|
|-p <port>|	端口号|
|-u <user>|	用户名|
|-P <password>|	密码|
|-g <geometry column>|	In the case of tables with multiple geometry columns, the geometry column to use when writing the shape file.|
|-b|	Use a binary cursor. This will make the operation faster, but will not work if any NON-geometry attribute in the table lacks a cast to text.|
|-r	|Raw mode. Do not drop the gid field, or escape column names.|
|-m <filename>|	Remap identifiers to ten character names. The content of the file is lines of two symbols separated by a single white space and no trailing or leading space.|
