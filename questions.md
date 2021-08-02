问题记录

- [`FileSystem.rename(Path src, Path dest)`](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/filesystem/filesystem.html#boolean_rename.28Path_src.2C_Path_d.29) ：重命名，在不同 `src`、`dest` 情况下会有不同的语义：

  先决条件（preconditions）：`src` 必须存在，`dest` 不能是 `src` 的子项目（子目录、子文件），`dest` 必须是根目录，或者 `dest` 的父级目录必须存在；`dest` 的父级目录不能是文件；`dest` 路径下不能有文件存在。

  后置条件（postconditions）：

  1. 重命名目录为它自己是无效操作（no-op）；返回值不定（POSIX 中结果是 `False`，HDFS 中结果是 `True`）。

  2. 重命名文件为它自己是无效操作（no-op）；返回值是 `True`。

  3. 重命名文件为一个不存在的路径，结果是将文件移动到目的路径下，会保留源文件的文件名。

  4. 重命名一个目录为另一个目录，`src` 目录下的所有内容都会原样的移动到 `dest` 目录下（`src` 目录和它的子项目将不复存在）。

     **奇异的是**：当 `dest` 目录不存在时（当然 `dest` 的父目录是必须存在的），将 `src` 的子项目移动到 `dest` 路径（会同时创建 `dest` 目录）；而当 `dest` 目录存在时，结果是将 `src` 目录及其子项目移动到 `dest` 目录下。

  5. 重命名到一个父路径不存在的路径，在 HDFS 中不会产生任何影响，返回 `False`；本地文件系统中，是一个常规的重命名，也会创建目的路径的父目录。