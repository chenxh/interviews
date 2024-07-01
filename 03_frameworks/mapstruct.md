## spring BeanUtils

BeanUtils.copyProperties ()只能转换类中字段名字一样且类型一样的字段。
BeanUtils.copyProperties ()采用的是反射，实际上当重复调用时效率是比较低的.

## mapstruct
https://blog.csdn.net/qq_40194399/article/details/110162124

## mapstruct 优势

* 反射机制：BeanUtils.copyProperties () 使用反射机制进行属性拷贝，这种方式在处理大量数据时会显著增加性能开销，因为反射操作相对较慢。相比之下，MapStruct最终调用的是setter和getter方法，而非反射，这使得其性能更高。
* 编译时代码生成：MapStruct通过注解处理器（Annotation Processor）在编译阶段自动生成映射代码，这样可以避免运行时的反射调用，从而提高性能。这种方式不仅提高了执行速度，还简化了代码，减少了重复工作。* MapStruct 在编译时生成映射代码，这意味着在运行时没有额外的性能开销。
* 类型安全和无依赖性：MapStruct生成的代码是类型安全的，并且无依赖性，这意味着它不会引入额外的库或框架，从而进一步提升性能。
* 复杂映射场景支持：MapStruct支持复杂的映射场景，如嵌套映射、集合映射等，这些功能在处理复杂对象时表现出色。
* 性能测试结果：多项性能测试表明，随着属性个数的增加，BeanUtils的耗时明显高于MapStruct。这进一步证明了MapStruct在大规模数据处理中的优势。
