### 6、删除数据（Deleting Data）

使用FileSystem的delete\(\)方法，可以永久的删除文件或目录：

```java
public boolean delete(Path f, boolean recursive) throws IOException;
```

如果参数_**f**_是一个文件或空目录，参数_**recursive**_会被忽略。如果_**recursive**_是**true**，当删除目录时，它的内容也会被一起删除；而如果不是空目录并且_**recursive**_为**false**，则抛出IOException异常。

