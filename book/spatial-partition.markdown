^title 空间分区
^section Optimization Patterns

## 意图

*将对象根据它们的位置存储在数据结构中，来有效的定位对象。*

## 动机

游戏让我们的拜访其他世界，但这些世界通常和我们的没有什么不同。
它们通常有和我们宇宙同样的基础物理和感知系统。
这就是我们为什么会认为这些由比特和像素构建的东西是真实的。

我们这里注意的假事实是*位置*。游戏世界有*空间*的感觉，对象都在空间的某处。
它用很多种方式证明这点。最明显的是物理——对象移动，碰撞，交互——但是还有其他方式。
音频引擎也许会根据声源和玩家的距离来确定，越远的声音越小。
在线交流也许局限在较近的玩家之间。

这意味着游戏引擎通常需要回答这个问题，“哪些对象在这个位置周围？”
如果每帧都需要回答这个问题，这就会变成性能瓶颈。

### 在战场上的单位

假设我们在做实时战略游戏。双方成百上千的单位在战场上撞在一起。
战士需要挥舞刀锋向最近的那个敌人砍去。
最简单的处理方法是检查每对单位，然后看看它们互相之间有多么近：

^code pairwise

现在有了双重内嵌循环，每个循环都会遍历战场上的<span name="all">所有</span>单位。
这就是意味着每帧进行的测试对数会随着单位数量的*平方*增长。
每个附加单位都需要和*所有*之前的相比较。
如果有大量单位，这就完全失控了。

<aside name="all">

内层循环实际没有遍历所有的单位。
它只遍历那些外部循环没有拜访的对象。
这避免了比较一对单位*两次*，在每个顺序中一次。
如果我们已经处理了A和B之间的碰撞，我们不必为B和A再做一次。

用大O术语，这还是*O(n&sup2;)*的。

</aside>

### 描绘战线

我们这里碰到的问题是没有指明数组中潜藏的对象顺序。
为了在某个位置附近找到单位，我们需要遍历整个数组。
现在，我们简化一下游戏。
不使用2D的战*场*，想象这是一个1D的战*线*。

<img src="images/spatial-partition-battle-line.png" alt="A number line with Units positioned at different coordinates on it." />

在这种情况下，我们可以通过根据他们在战线上的位置*排序*数组元素来简化问题。
一旦我们那样做，我们可以使用<span name="array">像</span>[二分查找](http://en.wikipedia.org/wiki/Binary_search)之类的东西找到最近的对象而不必扫描整个数组。

<aside name="array">

二分查找有*O(log n)*的复杂度，意味着找所有战斗单位的复杂度从*O(n&sup2;)*降到*O(n log n)*。像[pigeonhole sort](http://en.wikipedia.org/wiki/Pigeonhole_sort)可将其降至*O(n)*。

</aside>

这里的经验很明显：如果我们根据位置存储组织数据结构中的对象，就可以更快的找到它们。
这个模式是关于将这个思路应用到超过一个维度上。

## 模式

对于一系列**对象**，每个都有**空间上的位置**。
将它们存储在根据位置组织对象的**空间数据结构**中，让你**有效查询在某处或者附近的对象**。
当对象的位置改变时，**更新空间数据结构**，这样它可以继续找到对象。

## 何时使用

这是存储活跃的，移动的游戏对象的常用模式，也可用于静态美术和世界地理。
复杂的游戏中，不同的内容有不同的空间划分。

这个模式的基本需求是你有一系列有位置的对象，你做了太多的通过位置寻找对象的查询，导致性能下降。

## 记住

空间划分的存在是为了将*O(n)*或者*O(n&sup2;)*的操作降到更加可控的数量级。
你拥有的对象*越多*，这就越重要。相反的，如果*n*足够小，也许不需要担心这个。

由于这个模式包含了通过位置组织对象，可以*改变*位置的对象更难处理。
你需要<span name="hash-change">重新组织</span>数据结构来追踪在新位置的对象，这添加了更多的复杂性*并*消耗CPU循环。
保证交换是值得的。

<aside name="hash-change">

想象一章哈希表，其中对象的键可以自动改变，那你就知道为什么这需要技巧。

</aside>

空间划分也会因为记录划分的数据结构而使用附加的内存。
就像很多优化一样，它用内存换速度。如果比起时钟周期，内存更加短缺，这就是个失败的提议。

## 示例代码

模式自然*变化*——每种实现都略有不同，空间划分也不例外。
不像其他的模式，它的<span name="variations">变化</span>都很好的被记录下来。
学术界发表文章证明其性能优势。
由于我只关注模式背后的观念，我会给你展示最简单的划分：*固定网格*。

<aside name="variations">

看看本章的最后一节，那里有游戏中常用的空间划分方法列表。

</aside>

### 一张网格纸

想象整个战场。现在，叠加一张固定大小的网格方块在上面，就好像一张网格纸。
不是在单独的数组中存储我们的对象，我们将它们存到网格的格子中。
每个格子存储一列单位，它们的位置在格子的边界内部。

<img src="images/spatial-partition-grid.png" alt="A grid with Units occupying different cells. Some cells have multiple Units." />

当我们处理战斗时，我们只需考虑在同一格子中的单位。
不是将每个游戏中的单位与其他所有单位比较，我们将战场*划分*为多个小战场，每个都有更少的单位。

### 一网格链接单位

好了，让我们编码把。首先，一些准备工作。这是我们的基础`Unit`类。

^code unit-simple

每个单位都有位置（2D表示），以及一个指针指向它存在的`Grid`。
我们让`Grid`成为一个`friend`类，因为，就像将要看到的，当单位的位置改变时，它需要和网格做复杂的交互，以确保任何事情都正确的更新了。

这里是网格的表示：

^code grid-simple


注意每个格子是一个指向单位的<span name="stl">指针</span>。
下面我们扩展`Unit`，增加`next`和`prev`指针：

^code unit-linked

这让我们将对象组织为[双向链表](http://en.wikipedia.org/wiki/Doubly_linked_list)，而不是数组。

<img src="images/spatial-partition-linked-list.png" alt="A Cell pointing to a a doubly linked list of Units." />

每个网格中的指针都指向网格中的元素列表的第一个，每个对象都有个指针指向它前面的对象，以及另一个指针指向它后面的对象。我们等会会知道为什么。

<aside name="stl">

在这本书中，我避免使用任何C++标准库内建的集合类型。
我想让理解例子的所需知识越小越好，然后，就像魔术师的“我的袖子里什么也没有”，
我想明晰代码中*确实*在发生什么。
细节很重要，特别是性能相关的模式。

但这是我*解释*模式的方式。
如果你在真实代码中*使用*它们，使用内建在几乎每种程序语言集合避免头疼。
人生苦短，不要浪费在编写链表上。

</aside>

### 进入战场

我们需要做的第一件事是保证新单位创建时被放置到了网格中。
我们让`Unit`在它的构建器中处理这个：

^code unit-ctor

`add()`方法像这样定义：

<span name="floor"></span>

^code add

<aside name="floor">

除以网格大小将世界坐标转换到网格空间。
然后，缩短为`int`消去了分数部分，这样可以获得网格索引。

</aside>

除了链表通常有的繁琐，基本思路是非常简单的。
我们找到单位所在的网格，然后将它添加到列表前部。
如果那已经有一个列表单位了，我们把它链接到新的后面。

### 刀剑碰撞

一旦所有的单位都安定在他们的网格中，我们可以让它们开始互相交互。
使用这个新网格，处理战斗的主要方法看上去是这样的：

^code grid-melee

它遍历每个网格，然后在它上面调用`handleCell()`。
就像你看到的那样，我们真的需要将战场分割为分离的小冲突。
每个网格之后像这样处理它的战斗：

^code handle-cell

除了遍历链表的指针把戏，注意它和我们原先处理战斗的原始方法<span name="nested">完全一样</span>。
它对比每对单位，看看它们是否在同一位置。

不同之处是，我们不必再互相比较战场上*所有的*单位——只与那些近在一个格子中的相比较。
这就是优化的核心。

<aside name="nested">

简单分析一下，似乎我们让性能*更糟*了。
我们从对单位的双重循环变成了对格子然后单位的*三重*循环。
这里的技巧是两层内部循环现在只在很少的单位上运行，这足够抵消在格子上的外部循环的代价。

但是，这一依赖于我们格子的粒度。如果它们太小，外部循环就会造成影响。

</aside>

### 加油向前

我们解决了我们的性能问题，但是我们创建了新问题。
单位现在陷在它的格子中。
如果将单位移出了包含它的格子，格子中的单位就再也看不到它了，但其他单位也看不到它。
我们的战场有点*过度*划分了。

为了解决那个，我们需要每次单位移动时都做些工作。
如果它跨越了格子的边界，我们需要将它从原来的格子删除，添加到新的格子中。
首先，我们给`Unit`一个方法来改变它的位置：

^code unit-move

可推测的是，它会被AI代码调用控制电脑的单位，也会被玩家输入代码调用来控制玩家的单位。
它做的只是交换格子的控制权，之后：

^code grid-move

这是一大块代码，但它很直观。
第一位检查我们是否穿越了格子的边界。
如果没有，需要做的所有事情就是更新单位的位置，搞定。

如果单位*已经*离开了现在的格子，我们从格子的链表中移除它，然后再添加到网格中。
就像添加一个新单位，它会插入新格子的链表。

这就是为什么我们使用双向链表——我们可以通过设置一些指针飞快的添加和删除单位。
每帧都有很多单位移动时，这就很重要了。

### 一臂之长

这看起来很简单，但我们某种程度上作弊了。
在我展示的例子中，单位在它们有*完全相同的*位置时才交互。
这对于西洋棋和国际象棋是真的，但是对于更加实际的游戏就不那么准确了。
这通常需要将攻击*距离*引入考虑。

这个模式仍然可以好好工作，与检查同一位置匹配相反，我们做的事情更接近于：

^code handle-distance

当范围被牵扯进来，需要考虑一个边界情况：
在不同网格的单位也许仍然足够接近，可以相互交互。

<img src="images/spatial-partition-adjacent.png" alt="Two Units in adjacent Cells are close enough to interact." />

这里，B在A的攻击半径内，哪怕它们的中心点在不同的网格。
为了处理这个，我们不仅需要比较同一网格的单位，同时需要比较邻近网格。
为了达到这点，首先我们让内层循环摆脱`handleCell()`：

^code handle-unit

现在有函数接受一个单位和一列表的其他单位看看有没有碰撞。
然后我们让`handleCell()`使用它：

^code handle-cell-unit

注意我们同样传入了网格的坐标，而不仅仅是对象列表。
现在，这也许和前面的例子没有什么区别，但是我们会稍微扩展它：

^code handle-neighbor

这些新加的`handleCell()`调用在现在的单位和周围八个邻近格子中的<span name="neighbor">四个</span>寻找是否有任何碰撞。
如果任何邻近格子的单位离边缘近到单位的攻击半径内，就找到碰撞了。

<aside name="neighbor">

有单位的格子是`U`，它查找的邻近格子是`X`。

<img src="images/spatial-partition-neighbors.png" width="240" alt="The set of neighbors for a Cell with the four being considered highlighted." />

</aside>

我们只查询*一半*的近邻，基于内层循环从当前单位*之后*的单位开始同样原因——避免将每对单位比较两次。考虑如果我们检查全部八个近邻格子会发生什么。

假设我们有两个在邻近格子的单位近到可以互相攻击，就像前一个例子。
这是我们检查全部8个格子会发生的事情：

 1. 当找谁打了A时，我们检查它的右边找到了B。所以注册一次A和B之间的攻击。

 2. 然后，当找谁打了B时，我们检查它的*左边*找到了A。所以注册*第二次*A和B之间的攻击。

只检查一半的近邻格子修复了这点。检查*哪一*半无关紧要。

我们还需要考虑另外的边界情况。
这里，我们假设最大攻击距离小于一个格子。
如果我们有小格子和长攻击距离，我们也许需要扫描几行外的近邻格子。

## 设计决策

好的位置划分数据结构相对较少，说明方法之一是一一列举。
但是，我试图从它们的本质特性来组织。
我期望一旦你学会了四叉树和二分空间查找（BSPs）和其他类似的，就可以帮助你理解它们是*如何*工作，*为什么*工作，以帮你选择。

### 划分是层次的还是平面的？

我们的网格例子将空间划分成平面格子的集合。
相反，层次空间划分将空间分成<span name="couple">几个</span>区域。
然后，如果其中一个区域还包含多个对象，再划分它。
这个过程递归进行，直到每个区域都有少于最大数量的对象在其中。

<aside name="couple">

它们通常分为2,4,8块——对程序员很熟悉的数值。

</aside>

 *  **如果是平面划分：**

     *  *<span name="simpler">更简单。</span>*平面数据结构更容易想到也更简单实现。

        <aside name="simpler">

        我在每章中都提到了这个设计点，这是有理由的。只要你能，用简单的选项。大多数软件工程都是与复杂度做斗争。

        </aside>

     *  *内存使用是常量。*由于添加新对象不需要添加新划分，空间划分的内存使用通常在之前就可以确定。

     *  *在对象改变位置时更新的更快。*当对象移动，数据结构需要更新来找到它在哪个位置。通过层次性空间划分，这可能导致在调整多层层次结构。

 *  **如果是层次性的：**

     *  *它更有效率的处理空的区域。*考虑我们之前的例子，如果战场的一遍是空的。我们需要分配一堆空的格子，要在它们上面浪费内存，每帧还要遍历它们。

        由于层次空间划分不再分割空区域，更大的空区域保持在一个划分上。不需要遍历很多小空间，那里只有一个大的。

     *  *它处理密集空间更有效率。*这就是硬币的另一面了：如果你有一堆对象堆在一起，无层次的划分没有效率。你最终将所有对象都划分到了一起，就跟没有划分一样。一个层次划分会适应地划成小块，让你同时只需考虑一点对象。

### 划分依赖于对象集合吗？

在示例代码中，网格空间事先被固定了，我们在格子里追踪单位。另外的划分策略是适应性的——它们根据现有的对象集合在世界中的位置划分边界。

目标是*平衡的*划分每个区域为相同的单位数量以获得最好性能。
考虑网格的例子，如果所有的单位都挤在战场的角落里。
它们都会在同一格子中，找寻攻击的代码退化为原来的*O(n&sup2;)*问题。

 *  **如果划分与对象无关：**

     *  *对象可以增量添加。*添加对象意味着找到正确的划分然后放入其中，这点可以一次完成，没有任何性能问题。

     *  * 对象移动的更快。*通过固定的划分，移动单位意味着从格子移除然后添加到另一个。如果划分它们的边界跟着集合而改变，那么移动对象会引起<span name="sort">边界</span>移动，导致很多其他对象也要移到其他划分。

        <aside name="sort">

        这可与排序二叉搜索树比如红黑树或AVL树相类比：当你添加事物时，你也许最终需要重排树，并重排一堆节点。

        </aside>

     *  *划分也许不平衡。*当然，固定的缺点就是你对划分缺少控制。如果对象挤在一起，你浪费了内存在空区域上，这会造成更糟的性能。

 *  **如果划分适应对象集合：**

    空间划分像BSPs和k-d树这样切分世界，让每部分都包含相同数目的对象。为了做到这一点，选择边界时，你需要计算每边有多少对象。层次包围盒是另外一种为特定集合对象优化的空间划分。

     *  *你可以保证划分是平衡的。*这不仅提供了好的性能，还提供了*稳定*的性能：如果每个区域的数量保持一致，你可以保证游戏世界中的所有查询都会消耗同样的时间。一旦你需要固定帧率，这种一致性也许比性能本身更重要。

     *  *依赖一组数据划分更加有效率。*当对象*组*影响了边界在哪里，最好让所有的对象在划分前都出现。这就是为什么美术和地理更多的使用这种划分。

 *  **如果划分与对象无关，但*层次*与对象相关：**

    有一种数据划分需要特殊注意，因为它拥有固定划分和适应划分两者的优点：<span name="quad">四叉树</span>。

    <aside name="quad">

    四叉树划分2D空间。它的3D实现是*八叉树*，获取一个“空间”，分割为8个*正方体*。除了有额外的维度，它和平面划分一样工作。

    </aside>

    一个四叉树开始时将整个空间视为单一的划分。如果空间中对象的数目超过了临界值，它将其切为四小块。这些块的*边界*是确定的：它们总是将空间一切为二。

    然后，对于四个区域中的每一个，我们递归地做相同的事情，直到每个区域都有较少数目的对象在其中。由于我们递归地分割有较多对象的区域，这种划分适应了对象集合，但是划分本身没有*移动*。

    你可以从这里从左向右看到划分的过程：

    <img src="images/spatial-partition-quadtree.png" alt="A quadtree." />

     *  *对象可以增量增加。*添加一个新对象意味着找到并添加到正确的区域。如果区域中的数目超过了最大限度，它就被划分。在区域中的其他对象被添加到新的小区域中。这需要一点小小的工作，但是工作总量是*固定的*：你需要移动的对象数目总是少于最大的数目临界值。添加对象从来不会引发超过一次划分。

        删除对象也同样简单。你从它的格子中移除对象，如果它的父格子中的计数少于临界值，你可以合并这些子划分。

     *  *移动对象很快。*这，当然，如上所述，“移动”对象只是添加和移除，两者在四叉树中都很快。

     *  *划分是平衡的。*由于任何给定的区域都有少于最大对象数量的对象，哪怕对象都堆在一起，你也不会有划分有很多对象的区域。

### 对象只存储在划分之中吗？

你可将空间划分作为在游戏中对象存储的*唯一*地方，或者将其作为更快查找的二级缓存，使用另一个集合直接包含对象列表。

 *  **如果它是对象唯一存储的地方：**

     *  *这避免了内存天花板和两个集合带来的复杂度。*当然，存储对象一遍总比存两遍来的轻松。同样，如果你有两个集合，你需要保证它们同步。每次对象创建或删除，它都得从两者中添加或删除。

 *  **如果有对象的其他集合：**

     *  *遍历所有的对象更快。*如果所有对象都是“活的”而且它们需要做些处理，你也许会发现你需要频繁拜访每个对象而不在乎它的位置。想想看，早先的例子中，大多数格子都是空的。访问那些空的格子是对时间的浪费。
     
        存储对象的第二集合给了你直接遍历它们的方法。你有两个数据结构，每种为各种的用况优化。

## 参见

 *  我试图在这里不讨论特定的空间划分结构细节来保证这章处于高层（而且不过长！），但你的下一步应该是学习一下常见的结构。不管它们的恐怖的名字，它们都令人惊讶的直观。常见的有：

    * [Grid](http://en.wikipedia.org/wiki/Grid_(spatial_index))
    * [Quadtree](http://en.wikipedia.org/wiki/Quad_tree)
    * [BSP](http://en.wikipedia.org/wiki/Binary_space_partitioning)
    * [k-d tree](http://en.wikipedia.org/wiki/Kd-tree)
    * [Bounding volume hierarchy](http://en.wikipedia.org/wiki/Bounding_volume_hierarchy)

 *  每种空间划分数据结构基本上都是将一维数据结构扩展成更高维度的数据结构。知道它的直系子孙有助于分辨它对问题是不是好解答：

    * 网格是持久的[桶排序](http://en.wikipedia.org/wiki/Bucket_sort)。
    * BSPs，k-d trees和包围盒是[线性搜索树](http://en.wikipedia.org/wiki/Binary_search_tree)。
    * 四叉树和八叉树是[多叉树](http://en.wikipedia.org/wiki/Trie)。