Spring Session
---

Spring Session を使うと、
Session が Spring Session 提供のものに置き換えられる。
そして、HttpSessionListener が効かなくなる。


標準のセッション設定
```
server.servlet.session.cookie.comment
server.servlet.session.cookie.domain
server.servlet.session.cookie.http-only
server.servlet.session.cookie.max-age
server.servlet.session.cookie.name
server.servlet.session.cookie.path
server.servlet.session.cookie.same-site
server.servlet.session.cookie.secure
server.servlet.session.persistent
server.servlet.session.store-dir
server.servlet.session.timeout
server.servlet.session.tracking-modes
```

Spring Session設定
```
spring.session.hazelcast.**
spring.session.jdbc.**
spring.session.mongodb.**
spring.session.redis.**
spring.session.servlet.filter-dispatcher-types
spring.session.servlet.filter-order
spring.session.store-type
spring.session.timeout
```

|設定項目|プロパティ1|プロパティ2|
|-|-|-|
|セッションタイムアウト|spring.session.timeout|server.servlet.session.timeout|
|セッションストアタイプ|spring.session.store-type|
|セッション保存領域|spring.session.jdbc.table-name, spring.session.redis.namespace|



