## HikariCP性能y优化

* HikariCP通过优化(concurrentBag，fastStatementList )集合来提高并发的读写效率。
* HikariCP使用threadlocal缓存连接及大量使用CAS的机制，最大限度的避免lock。单可能带来cpu使用率的上升。
* 从字节码的维度优化代码。 (default inline threshold for a JVM running the server Hotspot compiler is 35 bytecodes ）让方法尽量在35个字节码一下，来提升jvm的处理效率

