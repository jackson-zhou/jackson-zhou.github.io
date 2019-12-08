---
layout: default
categories: [测试]
---
# 入口函数 
`sspShow`  **show/show.go**
[plantuml]
Alice -> Bob: Authentication Request
Bob --> Alice: Authentication Response

Alice -> Bob: Another authentication Request
Alice <-- Bob: another authentication Response<p>
[/plantuml]
```javascript
function add(x, y) {
  return x + y
}
```
```golang 
func main(a int, b int)(ret int) {
    ddd
    return abc
}
```
```Flow
A->B
B-->C
```
- [x] @mentions, #refs, [links](), **formatting**, and <del>tags</del> supported
- [x] list syntax required (any unordered or ordered list supported)
- [x] this is a complete item
- [ ] this is an incomplete item

数学公式
https://katex.org/docs/supported.html
$ s>1 $

$ \sum_{\mathclap{1\le i\le j\le n}}  x_{ij} $

${n \choose k}$



$
\begin{pmatrix}
   a & b \\
   c & d
\end{pmatrix} 
$

https://mermaidjs.github.io/flowchart.html
* TB - top bottom
* BT - bottom top
* RL - right left
* LR - left right
* TD - same as TB
```mermaid
graph TD;
        A-->B;
        A-->C;
        B-->D;
        C-->D;
RL-->D;
D-->B; 
```

```mermaid
sequenceDiagram
    Alice->>John: Hello John, how are you?
    John-->>Alice: Great!

```

```mermaid
gantt
    title A Gantt Diagram
    dateFormat  YYYY-MM-DD
    section Section
    A task           :a1, 2014-01-01, 30d
    Another task     :after a1  , 20d
    section Another
    Task in sec      :2014-01-12  , 12d
    another task      : 24d
```
```mind
-我的
 -时间管理
   -thinkido;
   -Task lists;
 -时间日志
   -日总结
   -每日成长
 -7天时间看一生
   -Task l ists;
   -time manager;
   -兼职市场
```
```mind
	-editor插件开发
		-思维导字
		-构思如何开发 3m
		-查看新插件的提交代码修改量
		-查看修改文件列表
			-比对单个文件主要修改位置
			-插件开发格式要求
		-测试 6m
```

```plantuml
<p>
[plantuml]<p>
Alice -> Bob: Authentication Request
Bob --> Alice: Authentication Response

Alice -> Bob: Another authentication Request
Alice <-- Bob: another authentication Response
<p>
[/plantuml]
```