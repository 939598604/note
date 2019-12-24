![](https://gitee.com/chenjinhua_939598604/resources/raw/master/static/20190904180559.png)

- 选择自定义安装 
- 填写安装路径
- 将python安装路径加入到path环境变量下

C:\Python37;

C:\Python37\Scripts;

#二.anaconda的安装

![](https://gitee.com/chenjinhua_939598604/resources/raw/master/static/20190904180645.png)

- 将Anaconda3安装路径加入到path环境变量下

   C:\Anaconda3\Scripts;

   C:\Anaconda3\condabin;



##1.组件简要介绍

> **Anaconda Navigator**：一个桌面图形用户界面，它允许您启动应用程序并轻松管理conda工具包和环境，可以用界面管理代替命令行命令。

> **Anaconda Prompt**：顾名思义，就是一个Anaconda终端，比如输入命令“python”可以进入python交互界面，输入“conda list”可以查看已安装的conda包和版本。

> **Jupyter Notebook**：是一个交互式笔记本，本质是一个 Web 应用程序，便于创建和共享文档，支持实时代码，数学方程，可视化和叙事文本。要想实现“左手程序员右手作家”，必须要借用这个文学式编程工具。

> **Spyder**：是一个使用Python编程语言进行科学计算的集成开发环境。



 ## 2.使用Jupyter Notebook完成“Hello Python”的打印

在以后的数据分析或者数据科学实践，使用最多的一个工具应该就是Jupyter Notebook，我们简单介绍一下怎么使用这个工具。

直接打开Jupyter Notebook，或在Anaconda Prompt中输入命令“jupyter notebook”，就会启动jupyter notebook的web服务。

 这时默认的浏览器会被调用打开并进入jupyter notebook的页面（http://localhost:8888/tree） 

![](https://gitee.com/chenjinhua_939598604/resources/raw/master/static/20190904180726.png)



## 3.修改Anaconda中的Jupyter Notebook默认工作路径

直接打开Jupyter Notebook，或在Anaconda Prompt中输入命令“jupyter notebook --generate-config ”

可以看到路径为D:\Users……找到此路径修改jupyter_notebook_config.py文件   

![](https://gitee.com/chenjinhua_939598604/resources/raw/master/static/20190904180802.png)

- 打开此文件找到

```shell
## The directory to use for notebooks and kernels. #c.NotebookApp.notebook_dir = ''
```

将其改为 

````shell
## The directory to use for notebooks and kernels. c.NotebookApp.notebook_dir = 'E:\Jupyter'
````

其中E:\Jupyter为我的工作空间，你可以改成你自己的， 
**注意:**

> 1.`#c.NotebookApp.notebook_dir = ''`中的#必须删除，且前面不能留空格。 
> 2.E:\Jupyter,Jupyter文件夹必须提前新建，如果没有新建，Jupyter Notebook会找不到这个文件，会产生闪退现象。
>
> 3.如果是windows的话，要填写 c.NotebookApp.notebook_dir = 'M:/workspace/pythonProject/notebook'
>
> 反斜杠

#  三.Python数据可视化numpy

##  1.arange

```python
t1 = np.arange(0, 5, 0.1)
print(t1)
```

arange是生成0-5的步长为0.1的矩阵

[ 0.   0.1  0.2  0.3  0.4  0.5  0.6  0.7  0.8  0.9  1.   1.1  1.2  1.3  1.4
  1.5  1.6  1.7  1.8  1.9  2.   2.1  2.2  2.3  2.4  2.5  2.6  2.7  2.8  2.9

```
3.   3.1  3.2  3.3  3.4  3.5  3.6  3.7  3.8  3.9  4.   4.1  4.2  4.3  4.4
  4.5  4.6  4.7  4.8  4.9]
```

##  2.random.rand

```
x=np.random.rand(5)
print(x)
```

生成0到1之间的五个随机数的矩阵

[ 0.64428406  0.5349657   0.11827496  0.7172195   0.37633937]

##  3.random.randint

x=np.random.randint(1,50,47)

随机生成1到50的矩阵，其中个数是47个

[39 27 44 23 23 27 49 46 43 16 23 35 18 34 13 42  5 22 13 46 16 38 18  6 15
 48 13  8 38  8 39 34 10  1 13 37 41 25 23 16 30 20 26  2 29 26 31]













#  四.Python数据可视化matplotlib

##  一.折线图

1.准备csv数据，命名为1.csv，格式如下

```csv
date,count
1923/3/23,34
1923/3/24,35
1923/3/25,36
1923/3/26,37
1923/3/27,38
1923/3/28,39
1923/3/29,40
1923/3/30,41
1923/3/31,42
1923/4/1,43
1923/4/2,44
1923/4/3,45
```

2.代码

```python
import matplotlib.pyplot as plt
import pandas as pd

data=pd.read_csv(filepath_or_buffer="C:/Users/Administrator/Desktop/1.csv",delimiter=",")
date=pd.to_datetime(data["date"])
plt.plot(date,data["count"])
plt.xticks(rotation=45 )
plt.show()
```

- pd.read_csv读取csv文件，其中filepath_or_buffer指定文件路径，delimiter是数据分隔符
- pd.to_datetime是将时间转换成时间格式
- plt.plot画折线图
- plt.xticks是设置x轴坐标下面的旋转角度rotation=45

3.图形展示效果

![](https://gitee.com/chenjinhua_939598604/resources/raw/master/static/20190904180837.png)

##  二.条形图





## 三.子图操作

### 1.子图排列

![](https://gitee.com/chenjinhua_939598604/resources/raw/master/static/20190904180856.png)

### 2.使用面向对象的方式

```python
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np

import numpy as np
import matplotlib.pyplot as plt

x = np.arange(0, 100)

fig=plt.figure(figsize=(6,8))
ax1 = fig.add_subplot(221)
ax1.plot(x, x)

ax2 = fig.add_subplot(222)
ax2.plot(x, -x)

ax3 = fig.add_subplot(223)
ax3.plot(x, x ** 2)

ax4 = fig.add_subplot(224)
ax4.plot(x, np.log(x))

plt.show()
```

- plt.figure(figsize=(6,8)) 新建一个画布 长为6 宽为8
- ax1 = fig.add_subplot(221)  在2行2列的第一个为准添加第一个画布

![](https://gitee.com/chenjinhua_939598604/resources/raw/master/static/20190904180912.png)