---
layout: post
title:  "Test mermaid"
date:   2025-07-22 23:29:59 +0200
categories: Guides
tags: [test]
---



```mermaid!
architecture-beta
    group api(cloud)[API]

    service db(database)[Database] in api
    service disk1(disk)[Storage] in api
    service disk2(disk)[Storage] in api
    service server(server)[Server] in api

    db:L -- R:server
    disk1:T -- B:server
    disk2:T -- B:db

```

