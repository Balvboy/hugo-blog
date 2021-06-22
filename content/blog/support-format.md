---
layout:     post
title:      "格式测试 "
description: "格式测试"
header_image: /img/zhouyang.png
excerpt: "格式测试"
author:     "zhouyang"
date:     2021-01-28
tags:
    - format
categories: [ format ]  
draft: true  
mermaid: true

---

# Markdown原生

## 代码块

```java
public static void main(String args[]){
    System.out.println("hello world");
}


```

## 图片


![](/img/zhouyang.png)

## 待办事项

- [ ] 支持以 PDF 格式导出文稿
- [ ] 改进 Cmd 渲染算法，使用局部渲染技术提高渲染效率
- [x] 新增 Todo 列表功能
- [x] 修复 LaTex 公式渲染问题
- [x] 新增 LaTex 公式编号功能

## 表格(自定义css实现)

| 项目        | 价格   |  数量  |
| --------   | -----:  | :----:  |
| 计算机     | \$1600 |   5     |
| 手机        |   \$12   |   12   |
| 管线        |    \$1    |  234  |







# 引入JS实现

## 数学表达式

$$ E=mc^2 $$

$$
f(x) = \sin(x)
$$

# shortcodes实现

## youtube

{{< youtube kXeBsuO3D10 >}}


## bilibili

{{< bilibili 2271112 >}}

## 4. 绘图 [流程图](https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown#7-流程图)

{{< mermaid >}}
sequenceDiagram
    participant Alice
    participant Bob
    Alice->John: Hello John, how are you?
    loop Healthcheck
        John->John: Fight against hypochondria
    end
    Note right of John: Rational thoughts <br/>prevail...
    John-->Alice: Great!
    John->Bob: How about you?
    Bob-->John: Jolly good!
{{< /mermaid >}}


{{< mermaid >}}
    graph TD
  A[Christmas] -->|Get money| B(Go shopping)
  B --> C{Let me think}
  C -->|One| D[Laptop]
  C -->|Two| E[iPhone]
  C -->|Three| F[fa:fa-car Car]
{{< /mermaid >}}


{{< mermaid >}}
gantt
dateFormat  YYYY-MM-DD
title Adding GANTT diagram to mermaid
excludes weekdays 2014-01-10

section A section
Completed task            :done,    des1, 2014-01-06,2014-01-08
Active task               :active,  des2, 2014-01-09, 3d
Future task               :         des3, after des2, 5d
Future task2               :         des4, after des3, 5d

{{< /mermaid >}}


{{< mermaid >}}
classDiagram
Class01 <|-- AveryLongClass : Cool
Class03 *-- Class04
Class05 o-- Class06
Class07 .. Class08
Class09 --> C2 : Where am i?
Class09 --* C3
Class09 --|> Class07
Class07 : equals()
Class07 : Object[] elementData
Class01 : size()
Class01 : int chimp
Class01 : int gorilla
Class08 <--> C2: Cool label

{{< /mermaid >}}