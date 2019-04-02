---
layout: post
title: 如何修改jar包中文件
categories: jar
tags: [jar]
date: 2019-03-01 21:00:00
description: 使用jar 指令修改jar 包中的文件
---

## 修改jar 包中的文件

```bash
#1.列出jar包中的文件清单
jar tf genesys_data_etl-0.0.1-SNAPSHOT.jar

#2.提取出内部jar包的指定文件
jar xf genesys_data_etl-0.0.1-SNAPSHOT.jar BOOT-INF/classes/realtime/t_ivr_data_bj.json

#3.然后可以修改文件
vim BOOT-INF/classes/realtime/t_ivr_data_bj.json

#4.更新配置文件到内部jar包.(存在覆盖，不存在就新增)
jar uf genesys_data_etl-0.0.1-SNAPSHOT.jar BOOT-INF/classes/realtime/t_ivr_data_bj.json      

#4.1更新内部jar包到jar文件
jar uf genesys_data_etl-0.0.1-SNAPSHOT.jar 内部jar包.jar     

#5.可以查看验证是否已经更改
vim genesys_data_etl-0.0.1-SNAPSHOT.jar
```
