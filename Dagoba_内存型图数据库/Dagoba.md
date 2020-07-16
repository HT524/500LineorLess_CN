# Dagoba : 内存型图数据库
Dann Toliver
Dann喜欢建造各种东西，比如编程语言、数据库、分布式系统、聪明人聚集的社区，此外，还跟他两岁的孩子一起搭建小马城堡。

## 序言
>"When we try to pick out anything by itself we find that it is bound fast by a thousand invisible cords that cannot be broken, to everything in the universe." —John Muir

>"What went forth to the ends of the world to traverse not itself, God, the sun, Shakespeare, a commercial traveller, having itself traversed in reality itself becomes that self." —James Joyce

很久以前，世界还很年轻，所有数据都快乐的生活在同一个文件中。如果你希望数据去跨栏，就沿路放置围栏，这样，数据就依次跳过它们。在打孔卡上面穿来穿去。生活和编程都是那么容易。

紧接着，“随机访问”革命袭来，而此时，数据们还散养在山谷之中。“牧养”这些数据成为关键问题：如果要在任意时间访问数据的任意片段，如何才能知道下次找出哪只呢？“圈养数据”的技术很快出现，这些技术在数据项之间建立“连接”<sup>[1](#ref1)</sup>，根据连接的关系进行分组、整理。查询数据意味着抓起一只羊，并且扯出所有和它相连的东西。

不久，程序员就背离了这一传统，引入了一套规则用来聚合数据<sup>[2](#ref2)</sup>。不再把不同的数据直接捆绑在一起，而是按照内容聚集，把数据分解成很小的片段，聚合起来，标记上名称。用声明性的形式描述问题，而答案，是那些被部分分解的数据（关系主义者认为这是正常状态）重新被聚合之后的结果。

关系模型在历史上占据了相当久的霸主地位。两次语言大战、无数摩擦冲突都没有动摇它的统治地位。它提供了在这个模型中你想要的所有问题的答案，尽管要以低效、笨拙、难以扩展为代价。这是程序员们要持续付出的代价。紧接着，互联网诞生了。

分布式革命改变了一切。数据被打破了空间限制，在机器之间游走。CAP理论的拥护者打破了关系模型的垄断，打开了新的数据“牧养”技术的大门，其中某些技术可以追溯到早期随机访问数据的尝试。我们将要了解其中一种，被称为“图数据库”的技术。

## 第一步

这一章，我们会构建一个图数据库<sup>[3](#ref3)</sup>。在构建过程中，我们会探索问题空间，对设计决策给出多种解决方案，比较这些方案并了解其中的折衷，最终选择出适合当前系统的方案。我会优先强调代码的紧凑性，这一过程反映了专业人员历经时间考验的经验。这一章的目的是传授这个过程，并构建图数据库<sup>[4](#ref4)</sup>。

图数据库让我们可以优雅的解决一些问题。在探索事物之间联系时，图是一种非常自然的结构。图是点(Vertice)和边(Edge)的集合，换句话说，它是一堆由很多线(Line)连起来的点(Dot)。而“数据库”呢，就像是一个堡垒，将数据放进去，又取出来。

图数据库可以解决哪类问题呢？比如追溯族谱：父母、祖父母、隔两辈的表亲，等等类似的关系。你要开发一个系统，这个系统要用优雅的方式来回答这样的问题：“Tour隔一辈的两代表亲都有谁”或是“Freyja和Valkyries的关系是什么”

一个合理的结构可能有两张表，一张描述实体，另一张描述实体间关系。比如查询Thor的父母的语句可能是这样：

```
SELECT e.* FROM entities as e, relationships as r
WHERE r.out = "Thor" AND r.type = "parent" AND r.in = e.id
```

如果我们要把问题扩展到查询祖父母呢？这就需要子查询(subquery)，或者使用其他特定数据库的SQL扩展。如果问题要延伸到“同一个曾祖的堂兄弟的子女”，那就需要非常多的SQL语句了。

我们希望写什么样的查询？应当是简洁而灵活的；让我们的问题，可以用一种自然的方式建模，并且容易扩展到类似问题。`second_cousins('Thor')`这种表达足够简洁，但完全没有灵活性。上面的SQL相当灵活，但缺乏简洁。

像`Thor.parents.parents.parents.children.children.children`这样的语句，可以在简洁和灵活之间取得平衡。这一原语(primitive)给我们来查询许多类似问题的灵活性，同时他也简洁又自然。这一语句会返回非常多的结果，其中包含了表亲和直系的平辈，不过我们将从这里开始。

如果要提供这种接口，最简单的办法是什么呢？可以仿照关系模型，构建边和点的列表，在此之上构建一些辅助方法，比如这样：

```
V = [ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15 ]
E = [ [1,2], [1,3],  [2,4],  [2,5],  [3,6],  [3,7],  [4,8]
    , [4,9], [5,10], [5,11], [6,12], [6,13], [7,14], [7,15] ]

parents = function(vertices) {
  var accumulator = []
  for(var i=0; i < E.length; i++) {
    var edge = E[i]
    if(vertices.indexOf(edge[1]) !== -1)
      accumulator.push(edge[0])
  }
  return accumulator
}
```

上述方法遍历了一个列表，对每一项执行了特定计算，最终汇聚出一个结果。现在这个版本还不够简洁，循环结构带来了不必要的复杂度。

如果有专门为此设计的循环结构就更好了。这就是`reduce`函数的功能：对给定的列表和函数，它会将列表中的每一项对函数进行求值，并将每一个结果依次汇聚到累加器里面去。

用函数式的风格来写这段代码会更加简短清晰：

```
parents  = (vertices) => E.reduce( (acc, [parent, child])
         => vertices.includes(child)  ? acc.concat(parent) : acc , [] )
children = (vertices) => E.reduce( (acc, [parent, child])
         => vertices.includes(parent) ? acc.concat(child)  : acc , [] )
```

给定一个点的列表，然后我们在边的集合上做reduce，如果某条边的子节点在输入列表当中，那就把它加入到累加器中。children函数也是一样的，只不过是检验边的父节点来决定是否把边的子节点加入到累加器中。

这些函数都是合法的JavaScript代码，但是用了一些目前尚未被浏览器支持的特性。下面这个版本是可以直接运行的：

```
parents  = function(x) { return E.reduce(
  function(acc, e) { return ~x.indexOf(e[1]) ? acc.concat(e[0]) : acc }, [] )}
children = function(x) { return E.reduce(
  function(acc, e) { return ~x.indexOf(e[0]) ? acc.concat(e[1]) : acc }, [] )}
```

现在我们可以使用这样的查询：

```
    children(children(children(parents(parents(parents([8]))))))
```

阅读这段代码的时候，需要不断回溯，而且容易搞错成对的括号。不过，已经离我们想要的东西更进一步了。再看看这段代码，还能做哪些优化呢？

边被当成了一个全局变量(global variable)，这意味着，使用这些函数的时候，不能同时使用多个数据库。这实在太过局限。

我们完全没有用到节点的信息。这说明什么？这意味着我们仅需要边的列表，前提是：节点是一组标量，可以独立于边而存在。如果要回答像是“Freyja和Valkyries是什么”这类问题，就不得不向节点添加更多信息。这意味着节点会变成复合(compound)数据，而边应当引用这些节点，而不是直接复制这些节点的值。

边也是同样的情况，他们各有一个“入”点和一个“出”点<sup>[5](#ref5)</sup>，但是没有加优雅的方式来附带一些额外的信息。如果要回答像是“Loki有多少个继父母”或者是“Thor出生前Odin有多少个孩子”这类问题，我们就需要附加信息。

显而易见的，这两个筛选函数的代码非常相似，这种相似表明，其中蕴含着更深层次的抽象。

你还看出其他问题了么？

## 构建更好的图

我们来解决一些已经发现的问题。由于边和节点被当成全局结构，因此每次只能使用一张图，但是我们希望一次使用更多的图。我们加一些结构来解决它。首先，可以从命名空间(namespace)开始

```
Dagoba = {}                                     // the namespace
```

我们使用一个对象(object)来作为命名空间。Javascript当中的对象可以看作是一个键值对的无序集合(unordered set of key/value pairs)。JavaScript只有四种基础数据结构供我们选择，因此我们会大量使用对象类型。（你可以在聚会上提这个有趣的问题“JavaScript的四种基本数据结构是什么？”）

现在我们需要一些图，可以用面向对象(OOP)的方法来构建图，但是JavaScript只提供了原型继承(prototypal inheritance)模型，我们可以构造一个原型对象，称之为`Dagoba.G`，然后通过一些工厂函数来进行实例化。采用这种方法的好处在于，可以通过工厂返回各种不同类型的对象。这样更加灵活。如果把构造逻辑都放到某一个类的构造函数里面就会丧失这种灵活性。

```
Dagoba.G = {}                                   // the prototype

Dagoba.graph = function(V, E) {                 // the factory
  var graph = Object.create( Dagoba.G )

  graph.edges       = []                        // fresh copies so they're not shared
  graph.vertices    = []
  graph.vertexIndex = {}                        // a lookup optimization

  graph.autoid = 1                              // an auto-incrementing ID counter

  if(Array.isArray(V)) graph.addVertices(V)     // arrays only, because you wouldn't
  if(Array.isArray(E)) graph.addEdges(E)        //   call this with singular V and E

  return graph
}
```

我们引入两个可选参数：一个节点列表和一个边列表。JavaScript对于参数没有严格限制，所以所有命名的参数都是可选的，如果不指定，这些参数会默认为“未定义”(undefined)<sup>[6](#ref6)</sup>。我们通常在构建图之前就已经拥有了所有的节点和边的信息，并且使用这些信息来构建图。但是在创建时没有这些信息，需要在程序当中逐步构建图的场景也非常常见<sup>[7](#ref7)</sup>。

我们创建了一个新的对象，集成了原型模型的优点，同时避免了原型的缺陷。我们构建了全新的数组（另外一个JS的基本数据结构）来描述边还有节点。此外还有一个新的对象vertexIndex，以及一个自增的ID计数器，我们稍后会再介绍这两个对象（想一想：我们为什么不把这些直接放到原型里面？）

接下来我们在工厂函数里面调用addVertices和addEdges方法，这两个方法的定义如下：

```
Dagoba.G.addVertices = function(vs) { vs.forEach(this.addVertex.bind(this)) }
Dagoba.G.addEdges    = function(es) { es.forEach(this.addEdge  .bind(this)) }
```

好了，剩下的事情就简单了 —— 把工作交给addVertex和addEdge就可以了。这两个函数的定义如下：

```
Dagoba.G.addVertex = function(vertex) {         // accepts a vertex-like object
  if(!vertex._id)
    vertex._id = this.autoid++
  else if(this.findVertexById(vertex._id))
    return Dagoba.error('A vertex with that ID already exists')

  this.vertices.push(vertex)
  this.vertexIndex[vertex._id] = vertex         // a fancy index thing
  vertex._out = []; vertex._in = []             // placeholders for edge pointers
  return vertex._id
}
```

如果被添加的节点缺少_id属性，那就使用autoid来进行赋值<sup>[8](#ref8)</sup>。如果图中已经有_id这个节点，就拒绝添加进来。稍等？这样会导致什么？一个节点的本质又究竟是什么？

在传统的面向对象系统中，我们需要一个节点类(vertex class)，所有节点都是这个类的实例。但我们采用了不同的方法，把节点看作是任何拥有_id、_in和_out这三个属性的对象。为什么是这样？归结起来，是为了让Dagoba有能力操作和宿主程序共享的数据。

如果我们在`addVertex`函数里面创建了`Dagoba.Vertex`的实例，那我们的内部数据就永远无法和宿主程序共享。而如果我们将一个`Dagoba.Vertex`实例作为参数传递给addVertex函数，宿主程序就可以通过引用来操作这个节点，从而破坏我们的不变式(invariant)[译注1](#yz1)

在创建一个节点的实例时，我们不得不做抉择：究竟是将提供给我们的数据复制到一个新对象中——这可能会使内存使用量翻倍；还是允许宿主程序不加约束地对数据库中的对象进行修改。性能和安全性上存在这样的矛盾，如何权衡则取决于你的使用场景。

我们将节点的属性设计为鸭子类型(duck typing)，这样可以在运行时决定，要对输入数据进行深拷贝(deep copying)<sup>[9](#ref9)</sup>还是直接使用<sup>[10](#ref10)</sup>。我们不会总把这种权衡的责任推给用户，但是这两种场景区别实在太大，以至于这种额外的灵活性非常重要。

现在我们已经把新节点加入到了图的节点列表当中，并加入到了`vertexIndex`中，以便使用`_id`来高效的查询这些节点，这些节点上还有两个附加属性`_out`和`_in`，他们都是边的列表<sup>[11](#ref11)</sup>。

```
Dagoba.G.addEdge = function(edge) {             // accepts an edge-like object
  edge._in  = this.findVertexById(edge._in)
  edge._out = this.findVertexById(edge._out)

  if(!(edge._in && edge._out))
    return Dagoba.error("That edge's " + (edge._in ? 'out' : 'in')
                                       + " vertex wasn't found")

  edge._out._out.push(edge)                     // edge's out vertex's out edges
  edge._in._in.push(edge)                       // vice versa

  this.edges.push(edge)
}
```

添加边的时候，我们首先查找这条边连接的两个点，如果有任何一个节点找不到，都要拒绝添加这条边。我们会用一个辅助函数来将这些拒绝的错误记录到日志中。所有错误都传递到这个辅助函数当中，这样，我们就可以为每个应用定制行为。稍后我们会对它扩展，让它成为允许外部注册(register)的onError回调函数(handler)，应用可以使用它自己的回调(callbacks)，而无需覆盖原来的辅助函数。我们可能会把这个回调注册到图级别、应用级别，或者两者都有，这个取决于我们需要的灵活程度。

```
Dagoba.error = function(msg) {
  console.log(msg)
  return false
}
```

接下来我们会把新的边添加到以下列表当中：这条边的出点的出边列表，这条边的入点的入边列表。

以上，这就是我们的图需要的所有结构！

## 输入查询

这个系统实际上只有两部分：一个部分负责维护这个图的结构，另外一个负责响应对于这个图的查询。负责维护图的结构的部分比较简单，我们已经介绍过了。而负责查询的部分就要复杂一些。

我们和之前一样，从一个原型和查询工厂开始。

```
Dagoba.Q = {}

Dagoba.query = function(graph) {                // factory
  var query = Object.create( Dagoba.Q )

  query.   graph = graph                        // the graph itself
  query.   state = []                           // state for each step
  query. program = []                           // list of steps to take
  query.gremlins = []                           // gremlins for each step

  return query
}
```

是时候来介绍一些朋友了。

一个程序(program)是由一系列步骤(step)组成的。每一个步骤都像是流水线(pipeline)里面的一段小管道(pipe) —— 数据从其中一端进来，用某种方式进行一些变换，然后从另一端出去。我们的流水线其实并不完全是这样的，不过已经很近似了。

程序里面的每个步骤都可以有状态，`query.state`是一个由每个步骤的状态组成的列表，这个列表和描述步骤的列表`query.program`相对应。

gremlin是一种在图中游走，来为我们完成目标的生物。gremlin可能是这个数据库当中最令人惊奇的部分，其历史可以追溯到：Tinkerpop的Blueprint，还有Gremlin以及Pacer查询语言。它们记得自己的轨迹，并且可以找到一些有趣的问题的答案。

还记得那个问题么——“Thor隔一辈的两代表亲都有谁？”。我们认为`Thor.parents.parents.parents.children.children.children`这种表达方式已经足够优雅。每个父/子的实例都是我们程序中的一个步骤，每一个步骤都有它自己的管道类型(pipetype)，也就是进行这个步骤操作的函数。

这个查询在实际的系统中可能是这样的：

```
    g.v('Thor').out().out().out().in().in().in()
```

每步都是一个函数调用，可以传递参数。解释器(interpreter)将每步的参数传递给这一步的管道。所以在`g.v('Thor').out(2,3)`当中，`out`这个管道会将`[2,3]`作为它的第一个参数。

我们需要可以在查询当中添加步骤的方法，可以定义这样一个函数：

```
Dagoba.Q.add = function(pipetype, args) { // add a new step to the query
  var step = [pipetype, args]
  this.program.push(step)                 // step is a pair of pipetype and its args
  return this
}
```

每个步骤都是一个复合体(composite entity)，这个复合体组合了管道和调用时使用的参数。在这一阶段，我们会把这两部分组成一个偏应用函数(partially applied function)，而不是用元组(tuple)简单组合<sup>[12](#ref12)</sup>。这样做会损失一些自省(introspective)能力，尽管后续这些能力可能会很有帮助。

我们会用一系列查询构造器(query initializer)来从图中构建新查询。从最简单的例子开始：v函数。它先是构建了一个查询，然后用辅助函数`add`来填充初始的查询程序。这里用了`vertex`管道，我们很快会介绍它。

```
Dagoba.G.v = function() {                       // query initializer: g.v() -> query
  var query = Dagoba.query(this)
  query.add('vertex', [].slice.call(arguments)) // add a step to our program
  return query
}
```

请注意`[].slice.call(arguments)`是JS的语法糖，这表示“把这个函数的参数们作为一个数组传递给我”。把参数列表表示成数组，这应该可以理解，因为很多情况下确实就像这样，但是参数列表的特性并不完全，缺少那些我们在现代的JavaScript当中所使用的数组的特性。

## 立即(Eager)求值带来的问题

在讲述管道类型之前，让我们先看一下精彩的策略执行部分。这分为两个主要流派：一个是“值调用(Call By Value)”流派，也被称作“忙碌海狸(eager beavers)”，这一流派要求在函数调用之前完成所有参数的求值；而另一个流派截然相反，被称作“按需调用(Call By Needians)”，将所有的事情都推到不得不做的时候才去做，换个说法，就是惰性的。

JavaScript是一个严格(strict)的语言，会在每个步骤被调用的时候对他们进行处理。对`g.v('Thor').out().in()`这个表达式进行求值，预期会是这样的步骤：首先找到Thor这个节点，然后找出这一节点在出方向上直接相连的节点，然后再找出这些节点在入方向上相连的所有节点。

如果使用非严格(non-strict)的语言，结果也是一样的，策略执行方式并不造成什么影响。但是，如果我们再加一些调用呢？如果Thor是一个社会关系复杂的人，那么`g.v('Thor').out().out().out().in().in().in()`这个查询会产生非常多的结果，因为我们并不要求结果是经过去重(unique)的，结果甚至会比整个图的节点数量还多。

我们可能只需要得出一部分不重复的结果，所以我们可以把查询改写成这样：`g.v('Thor').out().out().out().in().in().in().unique().take(10)`。现在我们的查询结果最多有10条。如果我们立即计算会发生什么？我们仍然要求出天文数字一样多的所有结果，而最后只返回前10条。

所有的图数据库都不得不尽力少做一些计算，也因此大多选择了非严格求值模型。因为我们在构建自己的解释器，因此是可以对程序做到惰性求值的。但也因此不得不面对一些后果。

## 心智模型(Mental Model)决定求值策略带来的结果

目前为止，我们求值的心智模型都是非常简单的：

<li>查询一个节点集合</li>
<li>将求得的集合作为输入传递给一个管道</li>
<li>如果有必要的话，重复这些步骤</li>

我们可以给用户直接呈现这种模型，因为这种模型易于做一些推断，但是上一节我们已经论证过，实现层面无法直接使用这种模型。如果用户的认知模型和实际实现之间存在偏差，也会带来问题。抽象泄露(leaky abstraction)还是个小问题，更严重的，会导致故障、认知混淆等各种问题。

如果要做这种障眼法，我们面对的几乎是最佳场景：无论实际执行的模型是什么样的，对于任何查询，答案总没有区别。唯一的区别在于性能。我们可以要求用户在使用系统之前充分了解一个较为复杂的模型，也可以专注于一部分用户，将简单模型迁移到复杂模型来获得更好的查询性能，总之，我们必须在这两者之间做出权衡。

为了做出决策，你需要考虑下面这些因素：

<li>简单模型和复杂模型之间的认知难度差异</li>
<li>先使用简单模型再进阶到复杂模型，还是跳过简单模型直接使用复杂模型。这两者区别带来的认知负担</li>
<li>需要在认知模型之间进行过渡的用户群体特征，包括用户群体比例、认知能力和可用时间等等</li>

在我们的场景下，这种权衡是合理的。对于大多数的请求来说，都会很快返回结果，因此用户无须特别的优化查询结构或是去了解更深层次的模型。而有一些用户会在大数据集上面使用一些高级(advanced)查询，这些人同样也是更加适于过渡到新模型的用户。此外，我们希望在学习复杂模型之前使用一个简单模型并不会带来非常大的困难。

我们会很快深入到这个新模型的一些细节当中，在进行下一章的过程中，希望你能记住这些重点：

<li>每个管道一次只返回一条结果，而不是一个结果集合。在执行查询过程中，每个管道都有可能会被激活多次</li>
<li>用一个读写头(read/write head)来控制接下来执行哪个管道。头的初始位置在流水线的最末端，根据当前被激活的管道的输出结果来决定这个头如何移动</li>
<li>输出结果可能是上面提及的gremlin之一。每个gremlin代表一个可能的查询结果，他们通过管道来携带一些状态信息。gremlin使得读写头右移。</li>
<li>管道可能会返回一个“拉取(pull)”结果，这代表着它需要输入数据，让读写头进行右移</li>
<li>如果结果是“完成(done)”，这意味着在这之前的管道都不会再次被激活了，读写头会进行左移。</li>

## 管道类型(Pipetypes)

管道类型构成了我们系统的核心。通过理解了每个管道类型是如何工作的，就能更好的了解他们在解释器当中如何被调用和组合到一起。

首先我们找一个地方来存放我们的管道类型，并且提供一个方法来添加新管道类型。

```
Dagoba.Pipetypes = {}

Dagoba.addPipetype = function(name, fun) {              // adds a chainable method
  Dagoba.Pipetypes[name] = fun
  Dagoba.Q[name] = function() {
    return this.add(name, [].slice.apply(arguments)) }  // capture pipetype and args
}
```

管道类型的函数被添加到队列里面，然后一个新的函数被添加到查询对象里面。每个管道类型必须对应一个查询函数。这个函数给查询程序添加了一个新的步骤，以及对应的参数。

当我们对`g.v('Thor').out('parent').in('parent')`这个表达式求值，`v`这个调用会返回一个查询对象，`out`调用会添加一个新步骤然后返回一个查询对象，`in`调用也是一样。这样就组成了我们的调用链接口(method-chaining API)。

请注意，添加一个新的管道类型会将原来已有的同名类型替换掉，这样运行过程中就可以修改已经存在的管道类型。这个决策会带来什么？有没有什么替代方案？

```
Dagoba.getPipetype = function(name) {
  var pipetype = Dagoba.Pipetypes[name]                 // a pipetype is a function

  if(!pipetype)
    Dagoba.error('Unrecognized pipetype: ' + name)

  return pipetype || Dagoba.fauxPipetype
}
```

如果我们找不到某个管道类型，会生成一个错误信息，然后返回一个默认的管道类型，这种管道就像一个空管子：如果接收到一个消息，就直接传递到另外一端。

```
Dagoba.fauxPipetype = function(_, _, maybe_gremlin) {   // pass the result upstream
  return maybe_gremlin || 'pull'                        // or send a pull downstream
}
```

注意到下划线了吗？我们使用这个标记来表示函数当中不会使用这些参数。这三个参数大部分函数都要全部使用，这时候三个参数都会被命名。这让我们可以一眼看出来某个特定的管道类型依赖于哪些参数。

下划线的表示方法非常重要，因为它使得注释行更加容易对齐。哈，并不是，我们严肃一些。“程序一定是为让人阅读而写，只是恰好可以被机器执行”，那么接下来我们最关注的是使我们的代码更漂亮。

### 节点(Vertex)

我们会用到的大多数管道类型都是使用一个gremlin，然后生成更多的gremlin，但是下面这个特定的管道类型会根据字符串来生成一个gremlin。给定一个节点编号，它就会返回一个新的gremlin。给定一个查询，它会找到所有匹配的节点，并且每次生成一个gremlin，直到所有匹配的节点都被遍历过。

```
Dagoba.addPipetype('vertex', function(graph, args, gremlin, state) {
  if(!state.vertices)
    state.vertices = graph.findVertices(args)       // state initialization

  if(!state.vertices.length)                        // all done
    return 'done'

  var vertex = state.vertices.pop()                 // OPT: requires vertex cloning
  return Dagoba.makeGremlin(vertex, gremlin.state)  // gremlins from as/back queries
})
```

首先检查一下是不是已经收集到了匹配的节点，如果不是的话我们需要查找一些结果。如果有任何匹配节点，我们会取出其中一个，然后生成一个位置在这个节点上的gremlin，并将这个gremlin作为返回值。每个gremlin都携带了自己的状态，就像一个日志(journal)，记录了它都路过哪里，在这个图上的旅途都有什么有意思的事情。如果某个步骤的输入是gremlin，并且这个gremlin即将退出，我们会复制它的所有日志。

注意我们直接修改了`state`参数，并且没有将它回传。另一个做法是，我们可以返回一个对象，而不是返回一个gremlin或者信号，并且通过这个对象来把状态进行回传。这会使得我们的返回值变得复杂，增加一些额外的垃圾(garbage) <sup>[13](#ref13)</sup>。如果JS允许多返回值，这个地方还可以写的更为优雅。

我们还是需要解决修改变量(mutation)的问题，因为调用者仍然有可能持有原始变量的引用。有没有什么方法可以让我们确定某个引用是“排他(unique)”的？——也就是说当前的这个引用是这个对象唯一的引用。

如果我们能确定某个引用是排他的，那么我们就可以利用不变性(immutability)的良好性质，同时避免了写时复制或者复杂的持久化数据结构。仅通过一个引用，我们无法确定某个对象被修改过，还是返回了一个包含了我们请求的修改的新对象——是否保持了“可见的不变性”<sup>[14](#ref14)</sup>。

有很多常见方法可以检测这种排他性：在静态类型系统中，可以使用排他类型(uniqueness type)<sup>[15](#ref15)</sup>在编译期保证每个对象只有一个引用。如果我们有引用计数器——甚至是仅有两bit的计数器<sup>[16](#ref16)</sup>——我们就能在运行时确定这个对象是否只有一个引用，并且利用这些优势。

JavaScript并没有这些方法可用，但是如果我们非常非常的自律，也可以做到一样的效果。至少到目前为止是这样。

### 出边和入边(In-N-Out)

遍历一个图就像点菜一样简单。下面的两行代码设定了`in`和`out`两种管道类型

```
Dagoba.addPipetype('out', Dagoba.simpleTraversal('out'))
Dagoba.addPipetype('in',  Dagoba.simpleTraversal('in'))
```

`simpleTraversal`函数返回了一个管道类型，这个管道接受一个gremlin作为输入，然后每次查询的时候生成一个新的gremlin。一旦这些gremlin都完成了之后，它会返回一个“拉取”请求，来从它的上一级获取新的gremlin。

```
Dagoba.simpleTraversal = function(dir) {
  var find_method = dir == 'out' ? 'findOutEdges' : 'findInEdges'
  var edge_list   = dir == 'out' ? '_in' : '_out'

  return function(graph, args, gremlin, state) {
    if(!gremlin && (!state.edges || !state.edges.length))          // query initialization
      return 'pull'

    if(!state.edges || !state.edges.length) {                      // state initialization
      state.gremlin = gremlin
      state.edges = graph[find_method](gremlin.vertex)             // get matching edges
        .filter(Dagoba.filterEdges(args[0]))
    }

    if(!state.edges.length)                                        // nothing more to do
      return 'pull'

    var vertex = state.edges.pop()[edge_list]                      // use up an edge
    return Dagoba.gotoVertex(state.gremlin, vertex)
  }
}
```

最前面两行代码处理了出方向版本(in version)和入方向版本(out version)的区别。然后准备返回管道，这和我们之前看过的`vertex`管道类型非常像。但有一点令人惊奇的是，这个管道接收了一个gremlin作为参数，而`vertex`管道类型是从无到有创造了一个gremlin出来。

但我们在添加查询初始化步骤上面，看到了一些同样的情况。如果没有给定gremlin，而且当前没有可用的边，就发起一次拉取；如果我们有一个gremlin但是还没有被设定状态，我们就在指定的方向上找到任意的边，然后把它添加到我们的状态当中。如果gremlin当前的节点没有合适的边可以添加，那么我们就再次发起拉取。最终我们会获取出来一条边，并且在这条边所指向的节点上复制一个新的gremlin以返回。

再看一眼这个代码，`!state.edges.length`在三个代码块里面被分别重复了一次。要不要对这部分进行重构，来降低这些条件的复杂度呢？这看起来很诱人，但是出于以下两点原因，我们不会这么做。

第一个原因比较弱：第三个`!state.edges.length`语句和前两个意义完全不一样，因为在第二个和第三个条件之间`state.edges`已经发生了变化。这实际上鼓励了我们去做重构，因为在一个函数内相同的标签却代表了不同的事物，这通常不是个理想状态。

第二个原因更严重一些。这不是我们需要写的唯一一个管道类型函数，我们会一次又一次的重复这种查询初始化以及状态初始化的工作。在编码过程中，始终要在结构化和非结构化之中做权衡。过分的结构化会让你在复杂的模板和抽象当中花费大量精力。太少的结构化会让你不得不在记忆中维护非常多的细枝末节。

在我们这篇文章中，大概有十几个左右的管道类型需要处理，正确的做法应当是让所有的管道类型函数尽可能保持风格一致，并且在每个要素上添加注释。所以我们需要一直抑制重构的冲动，因为这么做会有损一致性。我们也要抑制那种想要购买一个通用的结构来抽象查询初始化、状态初始化等等步骤的想法。如果有上百种管道函数类型，那么后一种选择可能是合适的。抽象的复杂度带来的消耗是固定不变的，而收益会随着使用这一抽象的单元(units)数量增长而线性增长。当处理许许多多这样的移动部件(moving pieces)时，使用任何强制措施来保证部件之间的规律性都是有益的。

### 属性(Property)

让我们稍停片刻，基于已经介绍过的三种管道类型，考虑下面这个样例查询。我们可以像这样查询Thor的祖父母<sup>[17](#ref17)</sup>：

```
g.v('Thor').out('parent').out('parent').run()
```

如果我们还想知道他们的名字呢？我们可以在末尾加一个map调用：

```
g.v('Thor').out('parent').out('parent').run()
 .map(function(vertex) {return vertex.name})
 ```

但是我们更倾向于采用一些普通的方法，比如这样：

```
g.v('Thor').out('parent').out('parent').property('name').run()
```

这种方式下，`property`这个管道是属于查询这个整体的，并不是缀在后面的一部分。这也有一些有趣的好处，我们会很快加以说明

```
Dagoba.addPipetype('property', function(graph, args, gremlin, state) {
  if(!gremlin) return 'pull'                                  // query initialization
  gremlin.result = gremlin.vertex[args[0]]
  return gremlin.result == null ? false : gremlin             // false for bad props
})
```

查询的初始化环节非常普通：如果没有gremlin，就发起拉取。如果已经有了gremlin，那么我们把属性值赋给它的`result`字段。这个gremlin还可以继续向前移动。如果这个gremlin通过了最后一个管道，那么`result`字段会收集结果并作为查询的结果返回 。不是所有的gremlin都有`result`这个字段，他们并不返回它最近一次访问的节点信息。

请注意，如果指定的属性不存在，我们选择返回`false`而不是一个gremlin，所以属性管道也起到了按类型过滤的作用。你能想到它的一些用处么？这个设计决策上做了哪些权衡呢？

### 去重(Unique)

如果我们希望找到Thor的所有祖父母的所有孙子女——他的表亲、他的兄弟姐妹以及他自己，我们会做这样一个查询：`g.v('Thor').in().in().out().out().run()`。然而，这会获得许多重复的结果。事实上，Thor自己就会至少重复出现四次。（你可以想出来什么场景下会重复次数更多么？）

为了解决这个问题，我们引入一个新的管道类型，命名为“去重(unique)”。新查询会得出没有重复的结果：

```
g.v('Thor').in().in().out().out().unique().run()
```

这个管道类型的实现：

```
Dagoba.addPipetype('unique', function(graph, args, gremlin, state) {
  if(!gremlin) return 'pull'                                  // query initialization
  if(state[gremlin.vertex._id]) return 'pull'                 // reject repeats
  state[gremlin.vertex._id] = true
  return gremlin
})
```

去重管道是一个纯粹的过滤器：它要么直接传递没有被修改过的gremlin，要么从上一个管道拉取新的gremlin。

我们从收集新的gremlin开始。如果这个gremlin当前的节点在我们的缓存当中，这意味着在此之前我们已经遍历过它了。否则，我们将这个gremlin的当前节点加入到缓存当中，并且将它原样传递。简单至极。

### 过滤(Filter)

我们已经讲过两种简单的过滤了，但是有些时候，我们会需要一些更加复杂的限制条件。如果想要找出Thor的兄弟们当中那些体重大于身高的人<sup>[18](#ref18)</sup>，我们可以这样查询:

```
g.v('Thor').out().in().unique()
 .filter(function(asgardian) { return asgardian.weight > asgardian.height })
 .run()
```

如果我们想要知道Thor的兄弟当中哪些人还活着，我们可以插入这样一个过滤器：

```
g.v('Thor').out().in().unique().filter({survives: true}).run()
```

它是这样工作的：

```
Dagoba.addPipetype('filter', function(graph, args, gremlin, state) {
  if(!gremlin) return 'pull'                            // query initialization

  if(typeof args[0] == 'object')                        // filter by object
    return Dagoba.objectFilter(gremlin.vertex, args[0])
      ? gremlin : 'pull'

  if(typeof args[0] != 'function') {
    Dagoba.error('Filter is not a function: ' + args[0])
    return gremlin                                      // keep things moving
  }

  if(!args[0](gremlin.vertex, gremlin)) return 'pull'   // gremlin fails filter
  return gremlin
})
```

如果过滤器的第一个参数不是一个对象或者函数，那我们就触发一个错误，然后将gremlin继续传递。稍等一下，考虑一下这个地方。错误已经发生了，我们为什么还决定继续执行这个查询呢？

这个地方发生错误有两种原因。第一种原因，程序员在查询当中输入了一个函数，无论是在REPL当中还是直接写在了代码里。一种情况，运行这个查询会返回结果，与此同时产生一个程序员可以观测到的错误。程序员会纠正这个错误，直到计算出正确的结果集合。或者，系统只显示出错误，而不产出任何结果，直到修复所有的错误才显示出结果。

第二种可能的原因是过滤器是在运行过程中被动态修改的。这是一种更加重要的场景，因为调用查询的人不一定是这个查询代码的作者。在Web领域，我们默认的守则是，永远显示结果，不打破任何东西。通常来说，遇到麻烦时硬着头皮撑场面要好于把严重的错误信息展示在用户面前。

对于那些显示太少结果好于显示过多结果的场景而言，可以重写`Dagoba.error`来抛出错误，来避开原生的控制流程。

### 截取(Take)

我们不总是想一次取回所有结果。有时只想取回相当少量的结果；比如说我们想找到十几个Thor的同龄人，我们又回到这个初始的问题：

```
g.v('Thor').out().out().out().out().in().in().in().in().unique().take(12).run()
```

如果没有这个截取管道，这个查询会运行非常长时间，但是由于我们的惰性求值策略，这个查询实际上非常高效。

有时我们只需要每次取一条结果：我们取回结果，对它做一些处理，然后返回来再取下一条。这种管道类型就允许我们这样做。

```
q = g.v('Auðumbla').in().in().in().property('name').take(1)

q.run() // ['Odin']
q.run() // ['Vili']
q.run() // ['Vé']
q.run() // []
```

查询可以运行在一个异步化的环境里面，让我们可以再需要获取更多的结果时进行收集。当所有结果都取出之后，返回一个空数组。

```
Dagoba.addPipetype('take', function(graph, args, gremlin, state) {
  state.taken = state.taken || 0                              // state initialization

  if(state.taken == args[0]) {
    state.taken = 0
    return 'done'                                             // all done
  }

  if(!gremlin) return 'pull'                                  // query initialization
  state.taken++
  return gremlin
})
```

如果`state.taken`还没有被初始化过，那就设它为0。JavaScript有隐式转换，但是默认会将未定义变量初始化为NaN，所以我们在这里进行显示的操作<sup>[19](#ref19)</sup>。

当`state.taken`到达了`args[0]`的限制，我们会返回'done'，停掉前面的管道。同时清掉`state.taken`计数器，允许我们稍后重复执行这个请求。

我们处理了这两个步骤之后才做查询的初始化操作，这是为了解决`take(0)`和`take()`的问题<sup>[20](#ref20)</sup>。然后我们增加计数器，然后返回gremlin。

### 别名(As)

接下来要介绍的四个管道类型会组合在一起工作，让我们可以使用更多的进阶(advanced)查询。这个管道类型允许我们给当前的节点进行标记。而下面的两个管道类型会使用这些标记。

```
Dagoba.addPipetype('as', function(graph, args, gremlin, state) {
  if(!gremlin) return 'pull'                                  // query initialization
  gremlin.state.as = gremlin.state.as || {}                   // init the 'as' state
  gremlin.state.as[args[0]] = gremlin.vertex                  // set label to vertex
  return gremlin
})
```

在查询完成初始化之后，我们要确保gremlin的本地状态当中一定有一个`as`参数。我们将这个gremlin当前的节点复制给这个`as`参数。

### 融合(Merge)

我们标记好了节点之后就可以使用融合操作来提取出来将它们提取出来。如果我们想查找Thor的父母、祖父母、曾祖父母，我们可以这样做：

```
g.v('Thor').out().as('parent').out().as('grandparent').out().as('great-grandparent')
           .merge('parent', 'grandparent', 'great-grandparent').run()
```

下面是融合管道类型的实现：

```
Dagoba.addPipetype('merge', function(graph, args, gremlin, state) {
  if(!state.vertices && !gremlin) return 'pull'               // query initialization

  if(!state.vertices || !state.vertices.length) {             // state initialization
    var obj = (gremlin.state||{}).as || {}
    state.vertices = args.map(function(id) {return obj[id]}).filter(Boolean)
  }

  if(!state.vertices.length) return 'pull'                    // done with this batch

  var vertex = state.vertices.pop()
  return Dagoba.makeGremlin(vertex, gremlin.state)
})
```

我们遍历每一个参数，来检查它是不是在gremlin记录的被标记的节点中。如果我们能找到，那就把这个节点复制到gremlin里面。请注意，只有进入了这个管道的gremlin才会出现在这个merge操作当中——如果Thor的母亲的父母不在这个图中，她也不会在结果集里面。

### 去除(Except)

我们已经介绍过了“给出Thor的所有不是Thor本人的兄弟姐妹”这种查询。我们可以通过`filter`来完成：

```
g.v('Thor').out().in().unique()
           .filter(function(asgardian) {return asgardian._id != 'Thor'}).run()
```

用`as`和`except`这两种管道会更直观一些：

```
g.v('Thor').as('me').out().in().except('me').unique().run()
```

也有一些查询很难进行直接过滤。如果我们想要查找Thor的叔叔和阿姨呢？我们怎样过滤掉他的父母呢？使用`as`和`except`会比较容易<sup>[21](#ref21)</sup>：

```
g.v('Thor').out().as('parent').out().in().except('parent').unique().run()

Dagoba.addPipetype('except', function(graph, args, gremlin, state) {
  if(!gremlin) return 'pull'                                  // query initialization
  if(gremlin.vertex == gremlin.state.as[args[0]]) return 'pull'
  return gremlin
})
```

这里我们检查当前的节点是不是和我们之前存储的一样。如果是这样就跳过它。

### 回溯(Back)

某些问题需要我们做进一步的验证，如果验证的结果是肯定的，就回到原来的地方。比如我们想知道Fjörgynn的哪些女儿育有Bestla的一个儿子？

```
g.v('Fjörgynn').in().as('me')       // first gremlin's state.as is Frigg
 .in()                              // first gremlin's vertex is now Baldr
 .out().out()                       // clone that gremlin for each grandparent
 .filter({_id: 'Bestla'})           // keep only the gremlin on grandparent Bestla
 .back('me').unique().run()         // jump gremlin's vertex back to Frigg and exit
```

这是回溯的定义：

```
Dagoba.addPipetype('back', function(graph, args, gremlin, state) {
  if(!gremlin) return 'pull'                                  // query initialization
  return Dagoba.gotoVertex(gremlin, gremlin.state.as[args[0]])
})
```

我们在这里使用了`Dagoba.gotoVertex`这个辅助函数来做实际的计算。我们稍后会介绍这个函数以及其他的一些辅助函数

## 辅助函数(Helpers)

以上介绍的管道类型依赖于一些辅助函数来完成计算。让我们在深入解释器之前快速浏览一下这些函数。

### Gremlin

Gremlin是一种简单的生物：包含一个当前节点，和一些局部状态。只要构造出来一个包含这两项的对象就完成了新建Gremlin的任务。

```
Dagoba.makeGremlin = function(vertex, state) {
  return {vertex: vertex, state: state || {} }
}
```

在这种定义下，拥有一个节点和一个状态属性的对象就可以是一个gremlin，所以构造器也可以嵌到代码里面(inline)。但是把它包装成一个函数的好处在于，在这个函数里面我们可以同时向所有的gremlin添加新属性。

我们也可以将一个现有的gremlin传递到一个新的节点上面，就像我们在回溯(back)类型和`simpleTraversal`函数中看到的那样。

```
Dagoba.gotoVertex = function(gremlin, vertex) {               // clone the gremlin
  return Dagoba.makeGremlin(vertex, gremlin.state)
}
```

请注意这个函数返回了一个新的gremlin：将原有的gremlin复制了一份，并且传递到我们想要的节点上。这意味着一个gremlin可以固定在一个节点上，而同时他的副本已经遍历了其他节点。这正是`simpleTraversal`函数中的行为。

举一个潜在优化的例子，我们可以加一些状态信息来记录这个gremlin所访问过的节点，并且设计一个新的管道类型来利用这些路径信息。

### 查找(Finding)

节点管道使用了`findVertices`函数来收集执行我们的查询时所需要的初始节点。

```
Dagoba.G.findVertices = function(args) {                      // vertex finder helper
  if(typeof args[0] == 'object')
    return this.searchVertices(args[0])
  else if(args.length == 0)
    return this.vertices.slice()                              // OPT: slice is costly
  else
    return this.findVerticesByIds(args)
}
```

这个函数的参数是一个列表。如果第一个参数是一个对象，那就把它传递给`searchVertices`函数，我们的查询可能是这样的：

```
  g.v({_id:'Thor'}).run()
  g.v({species: 'Aesir'}).run()
```

如果第一个参数不是对象，就把参数传递给`findVerticesByIds`函数，可以处理像是`g.v('Thor', 'Odin').run()`这样的查询。

如果完全没有任何参数，比如`g.v().run()`这样的查询。在一个非常大的图上面，你不会经常做这种操作，特别是在返回结果之前我们对节点列表进行了`slice`操作。因为有些调用点(call site)可能会直接从结果列表里面取出各个项来进行操作，所以才进行了slice操作。我们有两种优化办法，一种是在调用点进行数据复制，另外一种是避免这种操作。(我们在state当中维护一个计数器，而不是进行取出(popping))。

```
Dagoba.G.findVerticesByIds = function(ids) {
  if(ids.length == 1) {
    var maybe_vertex = this.findVertexById(ids[0])            // maybe it's a vertex
    return maybe_vertex ? [maybe_vertex] : []                 // or maybe it isn't
  }

  return ids.map( this.findVertexById.bind(this) ).filter(Boolean)
}

Dagoba.G.findVertexById = function(vertex_id) {
  return this.vertexIndex[vertex_id]
}
```

注意我们对`vertexIndex`的使用，如果没有这个索引的话，我们将不得不对节点列表进行遍历，对每个节点分别判断ID是否匹配——让一个常数时间的操作变成了一个线性时间的操作。任何`O(n)`复杂度的操作只要依赖这个查询就会变成O(n<sup>2</sup>)的复杂度。

```
Dagoba.G.searchVertices = function(filter) {        // match on filter's properties
  return this.vertices.filter(function(vertex) {
    return Dagoba.objectFilter(vertex, filter)
  })
}
```

`searchVertices`使用了`objectFilter`这个辅助函数对图中的每个节点进行了运算。我们会在下个章节里面介绍`objectFilter`这个函数，不过，你有什么办法惰性地查找节点么？

### 过滤(Filtering)

前文所述的`simpleTraversal`使用了一个过滤函数，这样对它获取的每一条边进行过滤。这是一个简单的函数，但是已经足够实现我们的目标。

```
Dagoba.filterEdges = function(filter) {
  return function(edge) {
    if(!filter)                                 // no filter: everything is valid
      return true

    if(typeof filter == 'string')               // string filter: label must match
      return edge._label == filter

    if(Array.isArray(filter))                   // array filter: must contain label
      return !!~filter.indexOf(edge._label)

    return Dagoba.objectFilter(edge, filter)    // object filter: check edge keys
  }
}
```

第一种场景是完全不进行任何过滤：`g.v('Odin').in().run()`，这会遍历Odin这个节点的所有入方向的边。

第二种场景根据边的标签来进行过滤：`g.v('Odin').in('parent').run()`，这个查询只遍历了标记为'父母(parent)'的边。

第三种场景使用了一组标签：`g.v('Odin').in(['parent', 'spouse']).run()`，这个查询同时过滤了代表父母(parent)和配偶(spouse)的边。

第四种场景的使用方式我们之前曾见过：

```
Dagoba.objectFilter = function(thing, filter) {
  for(var key in filter)
    if(thing[key] !== filter[key])
      return false

  return true
}
```

这样我们就可以使用一个过滤对象(filter object)来进行过滤：

```
`g.v('Odin').in({_label: 'spouse', order: 2}).run()`    // finds Odin's second wife
```

## 解释器的本质(The Interpreter's Nature)

我们在论述之山上即将登顶，等待我们的是最终的奖励：解释器。我们的代码已经相当紧凑，但是模型上还有一些微妙之处。

和开始的流水线相比较，现在我们程序的心智模型已经很适于编写查询。但是，我们还需要一个不太一样的模型来做具体的实现。这个模型其实不像是一个流水线而更像是一个图灵机(Turing machine)：用一个读写头来标识当前指定的步骤，读取当前这个步骤，改变它的状态，然后向右或者向左移动。

读取一个步骤，意味着对这个管道类型函数求值。如前所述，这些函数的输入参数是：整个图；它自己的特定参数(可能是一个gremlin)；它的本地状态(local state)。它的输出有几种：一个gremlin；否定状态(false)；一个'pull'或是'done'信号。这些输出会被我们的准图灵机读取，并且用来改变机器状态。

这个状态仅由两个变量组成：一个变量用来记录哪些步骤处于“已完成”状态，另外一个用来记录查询结果。这些状态更新完成后，如果查询完成就会结束查询并返回结果，如果未完成就对读写头进行移动。

现在我们会开始描述状态机里面的所有状态信息。首先我们有个结果列表，这个列表初始的时候是空的。

```
  var results = []
```

还有一个下标，用来标识在第一步开始之后最后一个“完成”的步骤：

```
  var done = -1
```

我们还需要存储最近一个步骤的输出，这个输出有可能是一个gremlin，也有可能什么都没有，我们称之为`maybe_gremlin`

```
  var maybe_gremlin = false
```

最终，我们需要一个计数器来标识当前读写头的位置。

```
  var pc = this.program.length - 1
```

最后……稍等一下。我们怎样做到惰性求值呢<sup>[22](#ref22)</sup>？要在一个立即求值的系统上构建一个惰性求值系统，传统做法是把参数和函数调用打包成一个`thunk`，而不是对它们进行求值。你可以将`thunk`看成是一个未求值的表达式。在JS里面，函数和闭包是第一类的(first-class)，我们可以把函数和它的参数打包进一个新的函数里面，这个函数是匿名而且不带参数的，`thunk`就可以通过这种方式进行实现。

```
function sum() {
  return [].slice.call(arguments).reduce(function(acc, n) { return acc + (n|0) }, 0)
}

function thunk_of_sum_1_2_3() { return sum(1, 2, 3) }

function thunker(fun, args) {
  return function() {return fun.apply(fun, args)}
}

function thunk_wrapper(fun) {
  return function() {
    return thunker.apply(null, [fun].concat([[].slice.call(arguments)]))
  }
}

sum(1, 2, 3)              // -> 6
thunk_of_sum_1_2_3()      // -> 6
thunker(sum, [1, 2, 3])() // -> 6

var sum2 = thunk_wrapper(sum)
var thunk = sum2(1, 2, 3)
thunk()                   // -> 6
```

除非真的需要进行求值，否则thunk不会被实际执行，thunk一旦被执行了就表示其他调用者现在需要它的输出了：在我们的场景里面，是需要查询的结果。每当解释器添加了新的函数调用，我们都会把它包装成一个thunk。回忆一下我们最开始的查询形式：`children(children(children(parents(parents(parents([8]))))))`，这个查询的每一层都是一个thunk，一层层包裹起来就像一个洋葱。

这个方法有两个折衷：首先是空间开销难以预估，因为在某些图中，可能会创建非常大量的thunk。另外一点是，我们的程序现在被表达为一个单一的thunk，此时做不了更多的事情。

上述的第二点通常不是个问题，因为编译器进行优化和我们在运行时创建thunk是两个分离的阶段。但在我们的场景里面，不存在这个优势：因为我们使用链式调用来实现了一个连贯接口(fluent interface)<sup>[23](#ref23)</sup>，如果我们使用thunk来达成惰性求值，那么每当一个新方法被调用的时候，我们都要生成一个新的thunk。这意味着，在我们运行`run()`的时候，我们只有一个thunk作为输入，完全没办法优化我们的查询。

但有意思的是，我们的查询语言所实现的连贯接口和常规编程语言的有所区别。如果我们不使用调用链的话，我们也可以把查询`g.v('Thor').in().out().run()`改写成`run(out(in(v(g, 'Thor'))))`。在JS这个语言里，我们会先处理`g`和`Thor`，然后是`v`，然后依次是`in`、`out`还有`run`，从里向外执行。在一个非严格语法的语言里，我们应该从外向内地处理，仅当需要的时候才把嵌套的下一层参数进行求值。

所以如果我们在语句的结尾处开始执行查询，然后再回到`v('Thor')`，仅在需要的时候才计算结果，那么我们已经有效地实现了非严格语法。这个秘密在于我们的查询是线性的。分支极大的使图处理变得复杂起来，并且引入了重复调用的可能性，这种问题可能需要记忆化方法(memoization)来避免浪费性的工作。我们的查询语言非常简单，这意味着我们也可以在线性的读写头模型上实现一个简单的解释器。

除运行时优化之外，这个形式对于方便的检测也很有好处：历史记录、可回溯性、单步调试、查询统计。这些功能都非常容易动态添加，因为我们控制了解释器，并且把它做成了虚拟机求值器的形式，而不是把程序当成单独个体来处理。

## 解释器，揭开面纱(Interpreter, Unveiled)

```
Dagoba.Q.run = function() {                 // a machine for query processing
  
  var max = this.program.length - 1         // index of the last step in the program
  var maybe_gremlin = false                 // a gremlin, a signal string, or false
  var results = []                          // results for this particular run
  var done = -1                             // behindwhich things have finished
  var pc = max                              // our program counter
  
  var step, state, pipetype
  
  while(done < max) {
    var ts = this.state
    step = this.program[pc]                 // step is a pair of pipetype and args
    state = (ts[pc] = ts[pc] || {})         // this step's state must be an object
    pipetype = Dagoba.getPipetype(step[0])  // a pipetype is just a function
```

这里的`max`是一个常量，`step`、`step`以及`pipetype`都是和当前这个步骤相关的缓存信息。我们进入这个循环之后就不会再退出，直到最后一个步骤完成。

```
    maybe_gremlin = pipetype(this.graph, step[1], maybe_gremlin, state)
```

调用当前这个步骤的管道类型函数，并提供对应的参数。

```
    if(maybe_gremlin == 'pull') {           // 'pull' means the pipe wants more input
      maybe_gremlin = false
      if(pc-1 > done) {
        pc--                                // try the previous pipe
        continue
      } else {
        done = pc                           // previous pipe is done, so we are too
      }
    }
```

处理结果为`pull`的情况时，我们首先把`maybe_gremlin`<sup>[24](#ref24)</sup>设置成`false`。这里我们赋予了'maybe'多种用途，把它当作一个传递`pull`和`done`信号的管道，但是一旦这些信号被使用完毕，我们就要重新把它看作是原本的'maybe'。

如果前面的一个步骤不是`done`状态<sup>[25](#ref25)</sup>，我们会回退读写头，并且再尝试一次。否则就把自己也标记成`done`状态，并且自然的把读写头向前移动。

```
    if(maybe_gremlin == 'done') {           // 'done' tells us the pipe is finished
      maybe_gremlin = false
      done = pc
    }
```

处理结果为`done`的情况就更加简单了，把`maybe_gremlin`设置成`false`，然后把这个步骤标记成`done`状态。

```
    pc++                                    // move on to the next pipe
  
    if(pc > max) {
      if(maybe_gremlin)
        results.push(maybe_gremlin)         // a gremlin popped out of the pipeline
      maybe_gremlin = false
      pc--                                  // take a step back
    }
  }
```

完成当前这个步骤之后，就可以把读写头移到下一个步骤。如果我们到达了程序的结尾，并且`maybe_gremlin`包含一个`gremlin`结果，那么我们就把它添加到结果集里面，然后把`maybe_gremlin`设置成`false`，再把读写头回退到程序的最后一个步骤。

这同时也是初始状态，此时pc的值是它的最大值。所以我们从这里开始处理，并且不断回溯。最终至少会再一次回到这个结尾的位置，此时我们收集完了这个查询的所有返回结果。

```
  results = results.map(function(gremlin) { // return projected results, or vertices
    return gremlin.result != null
           ? gremlin.result : gremlin.vertex } )

  return results
}
```

现在我们已经退出了循环，这个查询结束了，所有的结果都已经生成，现在我们需要处理和返回这些结果。如果有任何gremlin有自己的结果集，我们会将它返回，否则我们会返回这个gremlin标识的最后一个节点。我们还需要返回其他东西么？这里有什么权衡呢？

## 查询变换器(Query Transformers)

现在我们已经有了一个又好又结构紧凑的解释器来执行我们的程序，但是还差一些东西。所有现代的数据库管理系统(DBMS)都有一个查询优化器，这是一个重要的系统部件。和关系型数据库相比，非关系型数据库优化查询计划通常不会收获同样多的速度提升<sup>[26](#ref26)</sup>。但仍然是数据库设计的一个重要方面。

如果要做一些可以称之为查询优化的东西，那哪些是最简单的做法呢？我们可以写一些简单的函数，让它们在查询程序运行之前做一些优化。这个函数的输入是查询程序，输出是一个和原始程序不同的程序

```
Dagoba.T = []                               // transformers (more than meets the eye)

Dagoba.addTransformer = function(fun, priority) {
  if(typeof fun != 'function')
    return Dagoba.error('Invalid transformer function')

  for(var i = 0; i < Dagoba.T.length; i++)  // OPT: binary search
    if(priority > Dagoba.T[i].priority) break

  Dagoba.T.splice(i, 0, {priority: priority, fun: fun})
}
```

现在我们就可以向系统中添加查询变换器了，查询变换器是一个以一个程序为输入并且输出一个程序的函数，并且附有一个优先级别。更高级别的变换器会更靠近列表的头部。这里要确认`fun`是一个函数，因为稍后我们要执行它<sup>[27](#ref27)</sup>。

我们假设了不会添加非常多的变换器，因此添加新变换器的时候，在列表中线性遍历然后插入。我们在这个地方做了注释，以防这个假设被打破——如果列表非常长，二分查找对性能更有益，但是增加了一些复杂度，并且对于短列表也无法起到优化作用。

要运行这些变换器，我们可以在解释器的开头里面嵌入这样一行代码：

```
Dagoba.Q.run = function() {                     // our virtual machine for querying
  this.program = Dagoba.transform(this.program) // activate the transformers
```

这个变换实际上是这样操作的，把我们的程序依次交给每个变换器进行处理：

```
Dagoba.transform = function(program) {
  return Dagoba.T.reduce(function(acc, transformer) {
    return transformer.fun(acc)
  }, program)
}
```

直到这里，我们的引擎牺牲了一点简洁性，换取了一些性能提升。但是这种策略还有一点好处，就是为全局优化留有余地，如果我们的系统一开始就选择了局部优化的策略，那么很可能就无法去做全局优化了。

优化一个程序往往会增加复杂度，并且有损架构的优雅，使得系统难以理解和维护。最令人震惊的优化做法，是为了提升性能，而打破抽象的边界。即使是一些看起来完全无害的东西，比如把一些面向性能的代码嵌入到业务逻辑里面，都会使得维护工作变得大大困难。

鉴于这些原因，这种“正交优化”尤为诱人。我们可以在模块里面甚至是用户代码里面添加优化器，而不是把它们紧紧的耦合到引擎内部。这些代码可以被单独或是成组的测试，通过添加生成测试，我们甚至可以把这个过程自动化起来，保证所有优化器可以很好的协同工作。

我们也会通过这个转换器的系统来添加一些新功能，这些功能和性能优化完全无关。我们来看一些例子。


### 别名(Aliases)

像`g.v('Thor').out().in()`这样的查询是非常紧凑的，但这个查询是指Thor的兄弟姐妹还是他的伴侣呢？哪种解释看起来都是可以的。或许这样说会更清晰：是`g.v('Thor').parents().children()`这样，还是`g.v('Thor').children().parents()`这样呢？

我们可以用查询变换器来添加别名，只需要两个辅助函数：

```
Dagoba.addAlias = function(newname, oldname, defaults) {
  defaults = defaults || []                     // default arguments for the alias
  Dagoba.addTransformer(function(program) {
    return program.map(function(step) {
      if(step[0] != newname) return step
      return [oldname, Dagoba.extend(step[1], defaults)]
    })
  }, 100)                                     // 100 because aliases run early

  Dagoba.addPipetype(newname, function() {})
}
```

我们给已经存在的步骤增加了新名字，所以我们需要创建一个查询变换器，在遇到新名字的时候把它转换为原来的名字。我们还需要能够把有了新名称的函数添加到查询对象里面，这样才能在查询程序里面添加这个函数。

如果我们在调用到未定义的函数时会进行捕捉，并且把它路由到一个处理函数，那么这个变换器的优先级就可以比较低。但是现在还做不到这种机制。所以我们用高优先级来运行这个变换器，这样所有别名方法都会在被调用之前添加进来。

我们调用了另外一个辅助函数来把输出步骤的参数和别名步骤的默认参数合并在一起。如果输入的步骤缺少参数，就使用别名方法的参数进行替代。

```
Dagoba.extend = function(list, defaults) {
  return Object.keys(defaults).reduce(function(acc, key) {
    if(typeof list[key] != 'undefined') return acc
    acc[key] = defaults[key]
    return acc
  }, list)
}
```

现在就可以添加我们想要的别名了：

```
Dagoba.addAlias('parents', 'out')
Dagoba.addAlias('children', 'in')
```

我们所使用的数据模型还可以再特化一些，对每个父母和孩子中间的边加上`parent`标签。我们的别名会变成这个样子：

```
Dagoba.addAlias('parents', 'out', ['parent'])
Dagoba.addAlias('children', 'in', ['parent'])
```

我们还可以添加很多种关系的边，比如配偶、继父母，甚至是被抛弃的前任。如果进一步加强增强`addAlias`函数，我们还能进一步给祖父母、兄弟姐妹甚至是表兄弟姐妹增加别名。

```
Dagoba.addAlias('grandparents', [ ['out', 'parent'], ['out', 'parent']])
Dagoba.addAlias('siblings',     [ ['as', 'me'], ['out', 'parent']
                                , ['in', 'parent'], ['except', 'me']])
Dagoba.addAlias('cousins',      [ ['out', 'parent'], ['as', 'folks']
                                , ['out', 'parent'], ['in', 'parent']
                                , ['except', 'folks'], ['in', 'parent']
                                , ['unique']])
```

`cousins`这个别名看起来冗杂了一些。也许，还可以进一步增强`addAlias`函数，让我们可以在别名里面再去使用其他的别名，`cousins`这个调用就会变成这样：

```
Dagoba.addAlias('cousins',      [ 'parents', ['as', 'folks']
                                , 'parents', 'children'
                                , ['except', 'folks'], 'children', 'unique'])
```

我们原先写过的下面这个查询：

```
Dagoba.addAlias('cousins',      [ 'parents', ['as', 'folks']
                                , 'parents', 'children'
                                , ['except', 'folks'], 'children', 'unique'])
```
现在可以被简化成`g.v('Forseti').cousins()`。

我们同时也引入了一些麻烦，如果`addAlias`函数要解析一个别名，那么它也必须解析和它相关的其他别名。如果`parents`这个别名还调用了其他别名，那么在解析`cousins`这个别名的时候，就不得不停下来去解析`parents`别名，同时再去解析`parents`所使用的其他别名，以此类推。如果我们使用的`parents`别名最终调用了`cousins`别名呢？

这把我们拉进了依赖解析的问题里面<sup>[28](#ref28)</sup>，这是现代的包管理器(package manager)的核心组件。在选择理想版本、冗余消除(tree shaking)以及通用优化这些方面，有很多花哨的技巧，但是基本的思想是非常简单的，我们对所有的依赖和它们之间的关系构建一个图，试图找到一个把节点连接起来的方法，这种连接将使所有的箭头都是从左向右的。如果可以做到这件事情，这种特定的对节点排序的方法就称之为“拓扑排序(topological ordering)”，我们保证了描述依赖关系的图里面没有任何环：它是一个有向无环图(DAG)。如果无法完成这样的排序的话，那么我们的图至少存在一个环。

另一方面，我们预期了我们的查询通常很短（100个步骤就已经很多了），并且变换器数量会相当少。我们可以不使用DAG和依赖管理，而是修改变换器，如果变换器产生了任何改动就返回`true`，然后不断运行它，直到不再产生任何变换为止。这个方法要求所有的变换器都具有幂等性(idempotent)，这个性质对变换器来说是个必要性质。这两种方法的利弊各是什么？

### 性能

所有图数据库产品都有一个特定的性能特性：图的遍历查询耗时相对于图的大小来说是恒定的。假如不使用图数据库，如果要列出某个人的所有朋友，所需时间会和总的条目数成正比，因为在最差情况下将不得不遍历所有条目。这意味着如果一个遍历10个条目的查询需要1毫秒时间，那么一个需要遍历10,000,000个条目的查询需要将近两个星期的时间。如果要用Pony Express，说不定你的好友列表还能早一点送达<sup>[30](#ref30)</sup>。

为了降低这种令人沮丧的性能开销，大部分数据库都会在经常查询的字段上建立索引，将检索的复杂度从O(n)降低到O(logn)。但这是以牺牲一部分写入性能以及大量的存储空间为代价。在时间和空间开销之间做精细的权衡是大多数数据库调整优化的重要工作。

图数据库为了避免此类问题，直接在节点和边之间建立了直接的联系。所以图遍历仅仅是一个指针跳转过程，既不需要全局扫描，也不需要索引，没有多余的工作需要完成。现在，查找你的朋友所需要的时间就是恒定的了，和图里面的总人数完全无关。既没有额外的空间开销，也没有写入性能的损失。这种方法的一个缺陷在于，当整张图都在同一台机器的内存里面时，这种指针的工作效率才是最佳的。在多机环境有效的分布一个图数据库依然是一个活跃的课题<sup>[31](#ref31)</sup>。

如果我们把查找边的函数替换掉，就能在Dagoba的缩影当中看到这个问题。这里是一个非常原始的版本，以线性时间查找所有的在所有的边里面搜索。这和我们一开始的实现非常相像，不过使用了我们目前为止建造的所有结构。

```
Dagoba.G.findInEdges  = function(vertex) {
  return this.edges.filter(function(edge) {return edge._in._id  == vertex._id} )
}
Dagoba.G.findOutEdges = function(vertex) {
  return this.edges.filter(function(edge) {return edge._out._id == vertex._id} )
}
```

我们可以给边加一个索引，对于小规模的图而言，这个方法解决大部分问题了，不过在大规模的图上面，这种方法还是会遇到传统索引的问题

```
Dagoba.G.findInEdges  = function(vertex) { return this.inEdgeIndex [vertex._id] }
Dagoba.G.findOutEdges = function(vertex) { return this.outEdgeIndex[vertex._id] }
```

然后我们回到老朋友这里：纯净、甜美的，没有索引的邻接结构。

```
Dagoba.G.findInEdges  = function(vertex) { return vertex._in  }
Dagoba.G.findOutEdges = function(vertex) { return vertex._out }
```

你可以自行运行这些代码，来体验图数据库的不同<sup>[32](#ref32)</sup>。

### 序列化

拥有一个内存中的图数据库是很棒的，但我们一开始把它放在哪里呢？图的构造函数获取了节点和边的列表，以此来创建一个图，但是一旦图构建完成，我们怎么才能把节点和边取回。

我们自然会倾向于做类似于`JSON.stringify(graph)`这样的事情，但是这样会得到一个错误信息`TypeError: Converting circular structure to JSON`。在构建图的时候，我们把节点和边连接起来，同时边也连接到它们各自的节点上，所以所有东西都是互相连接的。那么我们怎么才能把漂亮整洁的列表重新提取出来呢？JSON替换函数可以帮助我们。

`JSON.stringify`函数把一个值转换成字符串，但是它还有两个额外的参数，一个替换函数和一个空白数字<sup>[33](#ref33)</sup>。替换函数可以让我们定制这个字符串化的过程。

我们需要稍微区别对待节点和边，所以我们将手动的把两者合成一个单一字符串。

```
Dagoba.jsonify = function(graph) {
  return '{"V":' + JSON.stringify(graph.vertices, Dagoba.cleanVertex)
       + ',"E":' + JSON.stringify(graph.edges,    Dagoba.cleanEdge)
       + '}'
}
```

然后定义节点和边分别使用的替换函数。

```
Dagoba.cleanVertex = function(key, value) {
  return (key == '_in' || key == '_out') ? undefined : value
}

Dagoba.cleanEdge = function(key, value) {
  return (key == '_in' || key == '_out') ? value._id : value
}
```

它们之间仅有的区别在于如何去表示循环：对于节点，我们直接忽略掉所有的边列表。对于边，就直接把它连接的节点替换成对应的ID。这样就避开了我们在图中所构建的环。

我们在`Dagoba.jsonify`当中手动处理了JSON，但是JSON的格式通常是非常固定的，所以通常不建议这么做。即便是这么一小段代码，也很容易遗漏一些东西，很难一眼判断出它的正确性。

我们可以把两个替换函数合并成一个替换函数，然后用新的替换函数来处理我们的整个图，就像是`JSON.stringify(graph, my_cool_replacer)`这样。这样我们就不必手动处理JSON的输出了。但如果要这样处理，那代码就会冗杂一些。你可以自行尝试，看看能不能精心设计一个方案，来避免手动编码JSON（合适会有得分哦）。

### 持久化

持久化通常是一个数据库最为棘手的地方：磁盘相对安全，却运行缓慢。批量处理写入，保持原子性，日志记录请求——这些事情很难做到快速而正确。

还好，我们构建的是一个内存数据库，所以完全不必担心这些！但有时我们可能会希望在本地保存一个数据库的副本，以便页面加载的时候进行快速重启。我们可以用刚才做的序列化器来做到这件事。首先把它包装成一个辅助函数：

```
Dagoba.G.toString = function() { return Dagoba.jsonify(this) }
```

在JavaScript中，只要把一个对象强制转换成字符串，就会调用这个对象的`toString`函数。所以如果g是一个图，那么`g+''`就会得到这个图的序列化JSON字符串。

`fromString`函数并不是语言规范的一部分，但是很好实现。

```
Dagoba.fromString = function(str) {             // another graph constructor
  var obj = JSON.parse(str)                     // this can throw
  return Dagoba.graph(obj.V, obj.E)
}
```

接下来我们就会在持久化函数当中使用这些代码了。`toString`函数就隐藏在其中——你可以找到它吗？

```
Dagoba.persist = function(graph, name) {
  name = name || 'graph'
  localStorage.setItem('DAGOBA::'+name, graph)
}

Dagoba.depersist = function (name) {
  name = 'DAGOBA::' + (name || 'graph')
  var flatgraph = localStorage.getItem(name)
  return Dagoba.fromString(flatgraph)
}
```

我们在名字前面加了一个伪命名空间，避免污染这个域在本地存储(localstorage)中的属性，因为本地存储可能确实很拥挤。此外本地存储通常也有比较小的容量限制，所以对于大一点的图，我们可能会考虑使用某种Blob对象来存储。

另外一个关键问题是，在多个窗口下，可能会对同一个域的数据同时进行序列化和反序列化。本地存储空间是在这些窗口间共享的，这些窗口可能在不同的事件循环当中，所以很有可能其中一个窗口会把另外一个窗口的数据覆盖掉。依标准而言，对本地存储的读写访问应当是有互斥量(mutex)进行保护的，但是不同浏览器的实现并不一致，即使我们这样的简单实现也可能会出问题。

如果我们希望持久化方法可以应对多窗口并发，那么我们可以利用本地存储变更时触发的事件，在这时去更新我们的本地图数据。

### 更新

现在`out`这个管道会复制一个节点的所有出方向(out-going)的边，然后每次按需取出其中一个。构建这个新的数据结构既消耗时间也消耗空间。或许我们可以换种方法，直接使用节点的出向边列表，并且使用一个计数变量来追踪我们的位置。你能想到这种做法存在什么问题么？

如果我们正在执行某个查询的时候，某个人删除了我们正在访问的边，这就会改变边列表的大小，并且此时对应的计数器处于关闭状态，我们因此跳过一条边。为了解决这个问题，我们可以对查询当中使用的边加锁保护，但是这样一来，我们就无法常规更新图，也无法在执行一个生命周期较长的查询时去响应其他的请求。即便在一个单线程的事件循环当中，查询也可能被多次异步重入，这意味着并发问题是现实存在的。

因此，复制边列表，并为此付出成本，在所难免。然而，这依然存在问题，对于一个长生命周期(long-lived)的查询，可能无法看到一个一致的时间顺序。在访问一个节点时，我们会遍历此时和它相连的所有边，但是在这个查询中，我们在不同时间访问了这些节点。假如有这样一个查询`g.v('Odin').children().children().take(2)`，并且调用`q.run()`来计算出Odin的两个孙子。过了一会，我们又想查询Odin的另外两个孙子，所以我们再次调用`q.run()`。如果在这两个事件发生的时间中间，Odin有了一个新的孙子，我们有可能无法查询到他，这取决于第一次查询的时候我们是否遍历到了这个新生儿的父母。

要解决这种非确定性问题，一种做法是修改`update`函数，让它给数据增加版本信息。同时还要修改事件循环来把图的当前版本信息传递给查询任务，这样我们的查询任务就能看到一个一致的视图，这时对它来说，世界就是它刚刚初始化时候的那样子。给数据库增加版本信息给我们打开了一扇大门，门后便是事务，还有STM风格的自动回滚/重试。

### 未来发展

我们已经见过了一种查询祖辈的方式：

```
g.v('Thor').out().as('parent')
           .out().as('grandparent')
           .out().as('great-grandparent')
           .merge(['parent', 'grandparent', 'great-grandparent'])
           .run()
```

这种方法笨拙而且难以扩展，如果需要查询六辈以内的长辈该怎么办呢？或者要一直向上查询，直到查到我们想要的结果呢？

如果我们换成这样的方式就好很多：

```
g.v('Thor').out().all().times(3).run()
```

我们想要摆脱的是上面那样的查询——在所有的查询变换器都运行完成之后，可能是这样子：

```
g.v('Thor').out().as('a')
           .out().as('b')
           .out().as('c')
           .merge(['a', 'b', 'c'])
           .run()
```

我们可以首先运行`times`这个转换器，生成这样的查询：

```
    g.v('Thor').out().all().out().all().out().all().run()
```

然后运行`all`这个变换器，把所有的all运算转换成一个不重复的`as`标签，然后在最后一个`as`后面加一个`merge`管道。

然而，还有一些问题。至少，这种`as/merge`方法的有效性是有前提的，只有每个路径都出现在图中才可以：假如图中缺少Thor的曾祖父母之一的记录，我们就会跳过一些有效的数据。再如，如果我们仅仅想对一部分查询而不是全部查询做`all`的变换呢？如果存在多个`all`管道呢？

要解决第一种问题，我们需要特殊处理`all`管道，而不仅仅是`as/merge`。我们需要每个父节点gremlin实际的跳过中间步骤。可以把这种操作看成是隐形传送（从一个管道直接跳转到另外一个管道当中），也可将其视为一种分支管道(branching pipeline)，无论是哪种方法都会使我们的模型变得复杂。另外一种方法，可以把正在穿过中间管道的gremlin看成处于假死状态，直到被某个特殊的管道唤醒。无论如何，限定假死和苏醒状态的管道都会非常棘手。

另外两个问题简单一些。想要仅对查询的一部分做修改，我们可以用特殊的start/end步骤把这个范围包装起来，像是`g.v('Thor').out().start().in().out().end().times(4).run()`。实际上，如果解释器对这些特殊管道类型有感知的话，我们就不需要`end`这个步骤，因为一个序列的`end`总是一个特殊的管道类型。我们可以称这些特殊的管道类型为“副词”(adverbs)，因为它们会修饰常规的管道类型，就像副词修饰动词那样。

要解决多个`all`管道调用的问题，我们需要运行两次所有的`all`管道：第一次在`times`管道之前，给每个`all`管道一个唯一的标记；在`times`管道之后还要运行一次，再次给所有的`all`管道打上唯一标记。

要没有限制地搜索祖辈仍然是个问题，比如，我们如何找到Ymir的哪些后代在Ragnarök幸存？我们可以给出独立的查询，比如`g.v('Ymir').in().filter({survives: true})`或是`g.v('Ymir').in().in().in().in().filter({survives: true})`，然后手动收集结果，但是这实在糟糕。

我们倾向于使用这样的副词：

```
g.v('Ymir').in().filter({survives: true}).every()
```

这个查询会像`all`+`times`管道的组合那样，但是并没有限制。我们可能想给这种遍历指定一种特定策略，比如固定的`BFS`或是`YOLO DFS`，因此`g.v('Ymir').in().filter({survives: true}).bfs()`更加灵活一些。以这种方式组织查询，我们就能方便描述这种复杂查询——“查询所有Ragnarök幸存者，但是跳过所有其他世代”，查询可以是这样的直接形式：`g.v('Ymir').in().filter({survives: true}).in().bfs()`。

## 总结

我们到目前为止学会了什么呢？图数据库非常适合存储那些互相有联系的数据<sup>[34](#ref34)</sup>，尤其是你希望通过在图中遍历来查询这些数据的场景。添加非严格(non-strict)语义允许我们流畅的查询数据，而这些查询在严格(eager)系统中则因为性能原因难以表达，此外还可以跨越异步边界。时间因素使得事情复杂化，而从多种角度来看(比如并发)，时间使得事情变得异常复杂，所以只要我们避免引入现时依赖(temporal dependency)(比如：状态，可观察的影响，等等)，就可以合理的使我们的系统更为简化。以简单、解耦以及痛苦的未优化方式实现系统，这种方式给后续做全局优化敞开了大门，使用驱动循环则使得我们可以做正交优化——每个优化都不会引入脆弱性和复杂性，而这是大多数优化技术的特点。

最后一点无论怎么强调都不算高估：保持简单。为了简单可以避免一些优化。找到正确的模型来努力达成简单性。探索各种可能性。本书的各章都用充分的证据表明了一个观点：那些极其不平凡的程序都有一个极小的紧凑的内核。一旦你找到了构建这个程序的内核，就要努力避免它被复杂性所污染。构建一些钩子(hooks)来附加那些额外的功能，尽最大努力来维护你的抽象屏障。用好这些技术并不容易，但是它们可以帮你解决其他棘手的问题。

## 致谢

非常感谢Amy Brown，Michael DiBernardo，Colin Lupton，Scott Rostrup，Michael Russo，Erin Toliver和Leo Zovic对本章节做出的宝贵贡献。



<a name="yz1"> 译注1 </a> 不变式一词参考了《算法导论》一书中的译法

<a name="ref1">1. </a> 层次模型(hierarchical model)是一种非常早期的数据库设计，把数据分组整理到树型的层次结构上，至今IBM公司的IMS产品还是以此为基础的，IMS是一个告诉的事务处理系统。在XML、文件系统以及地理信息存储当中依然能看到这种模型的影子。网络模型(network model)由Charles Bachmann发明，CODASYL为之标准化，网络模型在层次模型的基础上泛化，允许有多重父母关系，从树型结构变成了有向无环图(DAG)。这些导航数据模型(navigational database models)在1960年代流行起来，直到关系数据库随着性能提升在1980年代可用并取代了其统治地位。

<a name="ref2">2. </a> Edgar F. Codd在IBM工作期间发展了关系型数据库理论，但是深蓝(Big Blue)担心关系型数据库会侵蚀IMS的销量。IBM最终构建了一个研究的原型——System R，它基于一种新的非关系型语言——·SEQUEL，而不是Codd一开始设计的Alpha语言。SEQUEL是Larry Ellison根据发布前的会议论文在他的Oracle数据库当中复制的，为了避免商标纠纷，更名为SQL。

<a name="ref3">3. </a> 这个数据库起初是一个管理有向无环图(DAG)的库。"Dagoba"这个名字原本在末尾有一个不发音的'h'，以向虚拟的星球致敬(译者注：出自《星球大战》)。但某天偶然在巧克力棒的背面发现，没有'h'的版本指代了一个地方，这个地方用来默默地考虑事物之间的联系，这样似乎更加贴切了。

<a name="ref4">4. </a> 这篇文章的两个主要目的，一是教授这个过程，构建一个图数据库，另外一个就是"玩得开心"。
The two purposes of this chapter are to teach this process, to build a graph database, and to have fun.↩

<a name="ref5">5. </a> 请注意我们把边看成了一个节点对(a pair of vertices)。同时要注意这些对都是有序的，因为我们使用了数组。这意味着我们构建的是一个有向图，每条边都有一个起始节点和一个终止节点。我们的“点和线”可视化模型实际上是一个“点和箭头”模型。这使我们的模型复杂化了，因为我们不得不追踪边的方向，但是这同时也使得我们可以回答一些更加有意思的问题，比如“哪些节点指向了3号节点？”，再或者是“哪个节点拥有最多的出向边？”。如果我们需要给一个无向图建模，可以仅仅给每一条已知的边都增加一条反向的边。但是反过来就很难解决了：要用无向图模拟有向图。你能想到办法么？

<a name="ref6">6. </a> 另一个方向来说也是不严格的：所有的函数都是可变(variadic)的，所有的参数都是通过在参数列表中的位置确定的，很像是个数组，但又不完全是(“可变”是一种有趣的说法，用来形容一个函数有不确定的参数。“一个函数有不确定的参数”也是一种有趣的说法，用来形容它可以接受可变数量的变量)。

<a name="ref7">7. </a> 这里的`Array.isArray`是用来区分两种不同的用途，但通常来说，我们不会像生产环境的代码那样做这么多验证，我们会更加关注软件架构而不是垃圾箱。

<a name="ref8">8. </a> 为什么不直接使用`this.vertices.length`呢？

<a name="ref9">9. </a> 如果要解决深拷贝导致的空间泄露问题，通常会用一种复制路径的持久化数据结构(path-copying persistent data structure)，这种数据结构可以使用logN规模的额外空间完成无冲突的修改。但是问题依然存在：如果宿主应用程序持有这个节点数据的指针，无论我们的数据库当中有什么样的约束，它都可以在任意时间修改这个数据。Dagoba的使用场景中，节点数据对宿主程序来说是不可变的，可以避免这个问题，但是这要求用户侧相当的自律。

<a name="ref10">10. </a> 我们可以根据一个Dagoba级别的参数来决定，也可以根据一个图级别的配置来决定，或者是采用某种启发式的方法。

<a name="ref11">11. </a> 我们用列表(list)这个词用来指代一类抽象的数据结构，这种结构需要压入(push)和迭代(iterate)操作。我们使用了JavaScript的数组(array)数据类型来作为这个API的参数类型，这一结构可以满足列表的抽象。“边的列表(list of edges)”和“边的数组(array of edges)”这两种术语都是正确的，所以什么时候使用哪个术语取决于上下文：如果我们依赖于JavaScript的数组的实现细节，比如`.length`这个属性，那么我们就会使用“边的数组”。其他情况我们就会说“边的列表”，表明任何满足列表抽象的实现都是可以的。

<a name="ref12">12. </a> 元组(tuple)是另外一种抽象数据结构——比列表有更多的约束。通常元组都是固定大小，在这个例子当中，我们使用了一个二元组（在数据结构的研究者当中也会使用术语称之为“对”(pair)）。对于将来的开发人员来说，“所需限制最严格的抽象数据结构”这个术语更为合适。

<a name="ref13">13. </a> 尽管垃圾的生命周期非常短，这也是一种次优的做法。

<a name="ref14">14. </a> 对一个可变数据结构的两个引用，就像是一对对讲机一样，持有对讲机的人可以通过它直接通信。对讲机可以在函数之间传递，并且可以复制出来一大堆对讲机。这完全颠覆了你的代码经常使用的那种通信方式。在没有并发性的系统中，你可以随时摆脱它，但是一旦引入多线程和异步行为，这些对讲机之间叽叽喳喳的通信就变成了真正的障碍

<a name="ref15">15. </a> Clean语言将这种排他类型抹除了，并且和线性类型(linear type)有非线性关系，它们本身是子结构类型的子类型。

<a name="ref16">16. </a> 现代的JS运行时都有分代垃圾回收器(generational garbage collectors)，并且语言本身有意和引擎的内存管理保持距离，以此避免一些程序性的不确定性。

<a name="ref17">17. </a> 查询结尾的`run()`调用解释器并得到结果。

<a name="ref18">18. </a> 很自然的，体重的单位是船磅，而身高的单位是英寻。根据Asgardian的肉密度，这可能会得到很多结果，也可能根本没有(或者如果我们允许Jack Kirby通过万神殿允许莎士比亚进入，那就只有Volstagg)。(译者注：原作者这里在玩神话梗，实在不了解)

<a name="ref19">19. </a> 有的人会认为最好在所有情况下都显式地做这些操作。另外一些人就认为，一个优秀的隐式系统可以使得代码更加简洁，可读性更强，同时模板式的代码和bug都会变少。但是我们可以达成一个共识：有效利用JavaScript的隐式转换需要记住许多反直觉的特殊情况，这对初学者来说就是一个又一个陷阱。

<a name="ref20">20. </a> 你预期中这两个表达式分别会返回什么？它们实际上又返回了什么呢？

<a name="ref21">21. </a> 有一些特定条件可以让这部分查询产生非预期的结果。你能想到是哪种么？是否可以稍加修改让它可以处理这种问题？(译者注：如果祖父母辈产生了重组家庭，让父母辈拥有同一个祖父母就会产生问题)

<a name="ref22">22. </a> 技术上来说，我们需要实现解释器来实现非严格求值语义，这意味着只会在必须求值的时候才发起计算。惰性求值是一种用来实现非严格求值的技术。我们将两者混为一谈其实是偷懒的，但是我们会在必要的时候区分歧义。

<a name="ref23">23. </a> 链式调用让我们可以写`g.v('Thor').in().out().run()`这样的代码，而不是六行不连贯的JS代码。

<a name="ref24">24. </a> 我们用`maybe_gremlin`这个名称来提醒自己，它有可能是个gremlin，但也有可能是别的什么。设计之初，它可能是一个gremlin或者是空的(Nothing)。

<a name="ref25">25. </a> 回忆一下`done`这个变量一开始的值是-1，所以第一个步骤的前一个步骤总是done状态。

<a name="ref26">26. </a> 或者，说的更明确一些，一个没有被精细调整过的查询通常也不会引起指数级的性能退化。作为关系型数据库的终端用户，对于查询语句质量的审美通常是模糊的。

<a name="ref27">27. </a> 请注意我们没有限定优先级参数的域(domain)，它可以是整数、有理数、负数甚至是`Infinity`或者`NaN`这样的东西。

<a name="ref28">28. </a> 你可以在本书中的Contingent章节深入了解依赖分析。

<a name="ref29">29. </a> 换个更合适的说法，“无索引邻接(index-free adjacency)”。

<a name="ref30">30. </a> 尽管由于跨州电报和美国内战爆发，Pony Express只经营了18个月，但是它仅在10天之内就可以让邮件穿越海峡，这至今依然为人铭记。

<a name="ref31">31. </a> 对图数据库分片需要对图进行切割。最优图划分是一个NP难的问题，即便是处理树或者网格这样简单的图也是如此，一些比较好的近似算法也有指数渐进的复杂度。

<a name="ref32">32. </a> 在现代JavaScript引擎中，过滤一个列表是非常快的——对于比较小的图，我们的原始版本实际上要比无索引版本快很多，这有赖于底层的数据结构以及JIT编译的代码。尝试把它们应用在不同规模的图上，看看这两种方法在不同规模下的效果。

<a name="ref33">33. </a> 专业建议：给定一个足够深的树`deep_tree`，在控制台运行代码`JSON.stringify(deep_tree, 0, 2)`是一种可以快速使得这个树具有可读性的方法。

<a name="ref34">34. </a> 但联系不是那么紧密，也就是说——你希望边的数量和节点的数量增长是近似线性的。换而言之，平均每个节点连接的边数量不应该随着图的大小变化而剧烈变化。大多数我们会考虑放入图数据库的系统都有这种特性：如果Loki再增加100,000个孙子，Thor节点的度数并不会随之增加。

