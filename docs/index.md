# 一号标题
## 二号标题
### 三号标题

链接：[mkdocs.org](http://mkdocs.org)

自动链接：<http://www.baidu.com>

高亮： `hello` world

加粗强调：**hello**

斜体：*hello*  

分割线: 你可以在一行中用三个或以上的星号*、减号-、底线_来建立一个分隔线，行内不能有其他东西。你也可以在星号或是减号中间插入空格。下面每种写法都可以建立一条分隔线
* * *
***
*****
- - -
---
____

加粗又斜体：***hello***

无序列表：
* 1
* 2
* 3
有序列表:
1. 1
2. 2
3. 3

图片：
![链接名](链接URL)

引用：
> cite

段落:每行打一个TAB

    这是一个段落

文本中可直接用html标签，但是要前后加上空行:
<img width=300 src='https://www.baidu.com/img/baidu_resultlogo@2.png'>

代码：
``` java
public void main();
````

表格：
First Header  | Second Header
------------- | -------------
Content Cell  | Content Cell
Content Cell  | Content Cell

Markdown如何添加多个空行？答：加\<br/>
<br/><br/><br/><br/><br/>


mkdocs发布：

    mkdocs gh-deploy
