# 使用手册

> 本文档记录使用操作指南，文档中的template是指目录docs/template

### 1.template文件目录结构

```shell
--------------------------------  
  |-----template基本使用（文件夹）
       |-----001.文档一.md
       |-----002.文档二.md
  |-----template源码学习（文件夹）
       |-----001.文档一.md
       |-----002.文档二.md
  |-----_sidebar.md
  |-----README.md
```

- _sidebar.md 是进入到template目录下的侧边栏
- README.md 是进入到这个目录的首页显示页面

### 2.复制template文件夹

- 复制`template`，将其修改成将要修改的名称如:`Flume`
- 将子文件中的 `template基本使用` 重命名为 `Flume-ng基本使用`
- 将子文件中的 `template源码学习` 重命名为 `Flume-ng源码学习`

### 3.修改侧边栏

- 修改_sidebar.md文件，将编写好的目录指定相关的md文件，如：

  ```markdown
  * **Flume-ng**
    * **Flume-ng基本使用**
      * [（一）安装部署](docs/Flume/Flume-ng基本使用/001.安装部署.md)
      * [（二）自定义Sink](docs/Flume/Flume-ng基本使用/004.自定义Sink.md)
  ```

  **大标题** ： Flume-ng

  **小标题**  ： Flume-ng基本使用 

  **指定目录与md文档**  ：* [（一）安装部署](docs/Flume/Flume-ng基本使用/001.安装部署.md)

### 4.修改README.md

​      进入到目录要显示的内容

### 5.指定导航栏路径

​     修改根目录下的`_navbar.md`文件，指定路径

```markdown
* 大数据
  * [Flume](/docs/Flume/README.md)
  * [Kafka](/docs/Kafka/README.md)
```

将`Flume`指定到`/docs/Flume/README.md`路径

### 6.部署到githup

将本docsify代码提交到githup，打开githup page功能，如用域名访问必须提交域名文件。