![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/wbhiRD1.webp)

```
"registry-mirrors": [ "https://dockerproxy.com" ]
```

Phát triển và gỡ lỗi cục bộ

```
PG_HOST=pg

REDIS_HOST=redis
```

đổi thành

```
PG_HOST=127.0.0.1

REDIS_HOST=127.0.0.1

```

Và nhận xét `NODE_ENV=production`