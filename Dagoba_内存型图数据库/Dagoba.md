# Dagoba : 内存型图数据库
Dann Toliver
Dann喜欢构建各种东西，比如编程语言、数据库、分布式系统、聪明人的社区，还有和两岁的孩子一起搭建小马城堡。

## Prologue
>"When we try to pick out anything by itself we find that it is bound fast by a thousand invisible cords that cannot be broken, to everything in the universe." —John Muir

>"What went forth to the ends of the world to traverse not itself, God, the sun, Shakespeare, a commercial traveller, having itself traversed in reality itself becomes that self." —James Joyce

在很久以前，这个世界还年轻的时候，数据们都是愉快的单身羊。如果你希望一个数据跨越障碍，那么你就一个一个地放倒那些障碍物，数据就会一个一个地跑过去。穿孔卡片穿来穿去。生活和编程都是那么容易。

紧接着，“随机访问”的革命袭来，而数据们还在山谷之中被散养。“牧养”这些数据成为了一个关键问题：如果要在任意时间访问数据的任意一个片段，那么如何知道下一次找出哪一只呢？将这些数据圈养的技术很快出现，这些技术在数据项之间建立“连接”，根据连接的关系将他们进行分组聚合。查询数据意味着捞起一只羊，并且抓到所有和他相连接的东西。

不久，程序员就背离了这个传统，为了聚合数据，引入了非常多的规则来进行约束。他们不再想把数据直接捆绑在一起，他们更加愿意用内容来进行聚合，将数据分解到很小的片段，收集起来用名称来标记。问题以声明性的形式描述，得到的答案，是那些被部分分解的数据（关系主义者认为这是正常状态）重新被聚合之后的结果。

在有记载的历史中，大部分时间都被关系模型占据了霸主地位。两大语言之战、无数摩擦矛盾都没有动摇他的统治地位。他提供了在这个模型中你想要的所有问题的答案，尽管要以低效、笨拙、难以扩展为代价。这是程序员们要持续付出的代价。紧接着，互联网诞生了。

分布式革命改变了一切。数据打破了空间限制，在机器之间游走。CAP理论的拥护者打破了关系模型的垄断，打开了新的数据“牧养”技术的大门，其中一些技术可以追溯到早期用来驯化随机访问数据的尝试。我们将要了解其中一种，被称为“图数据库”的技术。

## 第一步

在这一章节中，我们将要构建一个图数据库。在构建过程中，我们会探索问题空间，为我们的设计决策做出多种解决方案，比较这些方案并在其中做出权衡，最终选出最适于我们的系统的方案。我们会比较强调代码紧凑性，但过程上依然遵循专业程序员经过时间检验的经验。这一章的目的是传授这一过程，并构建一个图数据库。

图数据库使我们可以用优雅的方式解决一些有趣的问题。在探索事物间关系这件事情上，图是非常自然的数据结构。图是点(Vertice)和边(Edge)的集合；换句话说，他是一堆由很多线(Line)连接起来的点(Dot)。至于数据库，就像是一个数据城堡，你将数据放入其中，又从中取出。

那么图数据库可以解决哪些类型的问题呢？比如追溯族谱：父母、祖父母、堂兄弟的孙子女，等等类似的关系。你需要开发一个系统，用以自然而优雅地回答这样的问题“Tour的堂兄弟的子女是谁”或者“Freyja和Valkyries的关系是什么”

一个合理的结构可能会是有两张表，一张用来描述实体，另外一张用来描述实体间关系。用来查询Thor的父母的语句可能会是这样：

```
SELECT e.* FROM entities as e, relationships as r
WHERE r.out = "Thor" AND r.type = "parent" AND r.in = e.id
```

但是我们如何把这个问题拓展到查询祖父母？我们需要子查询(subquery)，或者使用其他的特定数据库的SQL扩展。如果问题要拓展到查询“同曾祖的堂兄弟的子女”，那么我们可能就需要非常多的SQL语句了。

我们希望写怎样的查询语句？应该是简洁而灵活的；可以用自然的方式对我们的问题建模，并且可以很容易的拓展到类似的问题上去。`second_cousins('Thor')`这样足够简洁，但是没有任何灵活性。上面的SQL相当灵活，但是它缺乏简明性。

像`Thor.parents.parents.parents.children.children.children`这样的语句，可以在简洁和灵活之间取得平衡。这一原语(primitive)给我们来查询许多类似问题的灵活性，同时他也简洁又自然。这一特定的语句会返回非常多的结果，其中包含了表亲和直系的平辈，不过这是我们的起点。

我们需要构建一个能提供这种接口的东西，那么最简单的方式是怎样的呢？我们可以构建一个边的列表和点的列表，就像关系模型一样，并且在此之上构建一些辅助方法，比如这样

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

上面这个方法遍历了一个列表，并且对每一项执行了一些计算，最终汇聚出一个结果。这个方法并不够简明，循环结构引入了一些不必要的复杂度。

如果这一场景能使用一个更加特化的循环结构就更好了。这就是`reduce`函数所提供的功能：给定一个列表和一个函数，它对列表中的每一项计算函数的对应结果，并且将每一个结果依次汇聚到累加结果当中。

用函数式的风格来写这段代码会更加简短清晰：

```
parents  = (vertices) => E.reduce( (acc, [parent, child])
         => vertices.includes(child)  ? acc.concat(parent) : acc , [] )
children = (vertices) => E.reduce( (acc, [parent, child])
         => vertices.includes(parent) ? acc.concat(child)  : acc , [] )
```

给定一个点的列表，然后我们在边的集合上做reduce，如果某条边的子节点在我们的输入列表当中，那就把它加入到累加结果当中。children函数也是一样的，只不过是检验边的父节点来决定是否把边的子节点加入到累加结果当中。

这些函数都是合法的JavaScript代码，但是用了一些目前尚未被浏览器支持的特性。下面这个版本是可以直接运行的：

```
parents  = function(x) { return E.reduce(
  function(acc, e) { return ~x.indexOf(e[1]) ? acc.concat(e[0]) : acc }, [] )}
children = function(x) { return E.reduce(
  function(acc, e) { return ~x.indexOf(e[0]) ? acc.concat(e[1]) : acc }, [] )}
```

现在我们可以使用类似这样的操作：

```
    children(children(children(parents(parents(parents([8]))))))
```

这段代码阅读起来需要不断回溯，并且容易搞错一对对的括号，不过这已经距离我们想要的东西更进一步了。再看看这段代码，我们还有什么办法来优化它呢？

我们把边当成了一个全局变量(global variable)，这意味着，如果我们要使用这些函数，就不能同时使用多个数据库。这实在是太局限了。

我们完全没有使用节点的信息。这说明什么？这意味着我们仅需要边的数组，而前提是：节点是一组标量，他们可以独立于边而存在。如果我们想要回答类似于“Freyja和Valkyries是什么关系”这样的问题，我们就不得不向节点添加更多的信息，这意味着节点会变成复合(compound)数据，而边应当引用这些节点，而不是直接复制这些节点的值。

边也是同样的状况，他们各有一个“入”点和一个“出”点，但是没有更优雅的方式来附着一些附加信息。如果要回答类似这样的问题，比如“Loki有多少个继父母”或者是“在Thor出生前Odin有多少个孩子”，我们就需要这些附加信息。

你不用费力的去看那两个非常相似的筛选函数的代码，它们这么相似，意味着其中蕴含更加深层次的抽象。

你还看出其他问题了么？

## 构建更好的图

让我们来解决一些我们已经发现的问题。由于我们把边和节点当成全局结构，因此每次只能使用一张图，但是我们希望一次使用更多。让我们加一些结构来解决它。可以从一个命名空间(namespace)开始

```
Dagoba = {}                                     // the namespace
```

我们使用一个对象(object)来作为命名空间。Javascript当中的对象可以看作是一个键值对的无序集合(unordered set of key/value pairs)。JavaScript有四种基础数据结构供我们选择，因此我们会大量使用对象类型。（你可以在聚会上提这个有趣的问题“JavaScript的四种基本数据结构是什么？”）

现在我们需要一些图，我们可以用面向对象(OOP)的方法来构建，但是JavaScript当中可用的是原型继承(prototypal inheritance)模型，我们可以构造一个原型对象，称之为`Dagoba.G`，然后提供一些工厂函数来进行实例化。采用这种方法的好处在于，我们可以通过工厂返回各种不同类型的对象。这样更加灵活。如果把构造逻辑都放大某一个类的构造函数里面就会丧失这种灵活性。

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

我们引入两个可选参数：一个节点列表和一个边列表。JavaScript对于参数没有严格限制，所以所有命名的参数都是可选的，如果不指定这些参数会默认为“未定义”(undefined)。我们通常在构建图之前就已经拥有了所有的节点和边的信息，并且使用这些信息来构建图。但是在创建时没有这些信息，需要在程序当中逐步构建图的场景也非常常见。

我们创建了一个集成了原型模型优点的对象，同时避免了原型的缺陷。我们构建了全新的数组（另外一个JS的基本数据结构）来描述边还有节点。此外还有一个新的对象vertexIndex，以及一个自增的ID计数器，我们稍后会再介绍这两个对象（想一想：我们为什么不把这些直接放到原型里面？）

接下来我们在工厂函数里面调用addVertices和addEdges方法，这两个方法的定义如下：

```
Dagoba.G.addVertices = function(vs) { vs.forEach(this.addVertex.bind(this)) }
Dagoba.G.addEdges    = function(es) { es.forEach(this.addEdge  .bind(this)) }
```

好了，剩下的事情就简单了 —— 我们仅仅需要把工作交给addVertex和addEdge就可以了。这两个函数的定义如下：

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

如果被添加进来的节点没有_id这个属性，我们就用我们的autoid来进行赋值。如果_id已经存在于我们的图中，那就拒绝将这个节点加入到图中来。稍等？这样会导致什么？一个节点究竟是指什么？

在传统的面向对象系统当中，我们需要一个节点类(vertex class)，所有的节点都是这个类的实例。但我们采用了不一样的方法，把节点看作是任何拥有_id,_in和_out这三个属性的对象。为什么是这样？最终，都可以归结于，这种结构带给Dagoba一种控制能力，控制那些和目标应用程序共享的数据。

如果我们在`addVertex`函数里面创建了一些`Dagoba.Vertex`的实例，我们的内部数据就永远无法和目标应用程序共享。如果我们将一个`Dagoba.Vertex`实例作为参数传递给addVertex函数，目标应用程序就可以通过持有这个节点的指针来对他进行操作，从而可能打破我们的不变式(invariant 译者注：不变式一词参考了《算法导论》一书中的译法)

当需要创建一个节点的对象实例时，我们不得不做出这样一个决定：究竟是将提供给我们的数据复制到一个新的对象当中——这有可能会使我们的空间占用翻倍；还是允许目标应用程序不加约束的对数据库中的对象进行修改。在性能和安全性上存在这样的矛盾，具体如何权衡则取决于你的特定使用场景。

我们将节点的属性设计为动态类型(duck typing)，这样我们可以再运行时来决定是否对输入数据进行深拷贝(deep copying)或者是直接当作节点使用。我们不会总是把在安全和性能之间权衡的责任推给用户，但是这两种场景实在区别太大，以至于这种额外的灵活性对我们非常重要。

现在我们有了新节点，并将它加入到了图的节点列表当中，并且将它加入到了`vertexIndex`当中，以便用`_id`来高效的查询这些节点，在这些节点上还有两个附加属性`_out`和`_in`，它们都是边的列表。

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

添加一条边的时候，我们首先查找这条边连接的两个节点，如果任何一个节点找不到，都会拒绝添加这条边。我们会用一个辅助函数来将这些被拒绝的错误记录到日志当中。所有的错误都会被传递到这个辅助函数当中，这样我们就可以为每个应用程序定制它的行为。稍后我们会把它拓展为一个允许外部注册(register)的onError回调函数(handler)，目标应用程序可以使用他自己的回调(callbacks)，而无需将原来的辅助函数覆盖掉。我们可能会让这个回调注册到图级别、应用级别，或者两者都有，这取决于我们需要的灵活程度。

```
Dagoba.error = function(msg) {
  console.log(msg)
  return false
}
```

接下来我们会添加新的边到以下节点的边列表当中：这条边的出点的出边列表，这条边的入点的入边列表。

以上，这就是我们所有需要的图结构！

## 输入查询

这个系统实际上只有两个部分：一个部分负责维护这个图的结构，另外一个部分负责回答关于这个图的查询。负责维护图的结构的部分比较简单，我们已经介绍过了。而负责查询的部分就更加复杂一些。

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

我们程序中的每个步骤都可以有一个状态，`query.state`就是一个由每一步的状态组成的列表，这个列表和描述步骤的列表`query.program`相对应。

一个精灵(gremlin)是一种在图中游走来完成我们的目标的生物。精灵可能是一个数据库当中最令人惊奇的部分，它们从Tinkerpop里面的Blueprints，以及Gremlin和Pacer查询语言衍生出来。它们记得自己的位置，并且可以回答一些有意思的问题。

回忆一下我们想要回答的一个问题，“Thor的同曾祖的堂兄弟的子女都有谁？”。我们认为`Thor.parents.parents.parents.children.children.children`这种表达方式已经足够优雅。每个父/子的实例都是我们程序中的一个步骤，每一个步骤对应它自己的管道类型(pipetype)，也就是进行这个步骤操作的函数。

这个查询在我们实际的系统中可能是这样的：

```
    g.v('Thor').out().out().out().in().in().in()
```

每个步骤都是一个函数调用，可以填充参数。解释器(interpreter)将每一个步骤的参数添加到这个步骤对应类型的函数当中，所以在`g.v('Thor').out(2,3)`当中，`out`这种管道类型的函数会将`[2,3]`作为它的第一个参数。

我们需要一个为查询添加具体步骤的函数，可以这样定义：

```
Dagoba.Q.add = function(pipetype, args) { // add a new step to the query
  var step = [pipetype, args]
  this.program.push(step)                 // step is a pair of pipetype and its args
  return this
}
```

每个步骤都是一个复合实体(composite entity)，将管道类型函数和参数组合起来，以调用这个函数。我们会在这一阶段将这两部分组合成一个偏应用函数(partially applied function)，而不是使用元组(tuple)来表示。但是这样会缺少一些自省能力，尽管这些能力在后面会提供很多帮助。

我们会用一系列的查询构造器(query initializer)来从我们的图当中构造新的查询。我们从最简单的例子开始：v函数。它构建了一个新查询，用我们的辅助函数`add`来填充初始的查询程序。这里使用了`vertex`的管道类型，我们后面马上会介绍它。

```
Dagoba.G.v = function() {                       // query initializer: g.v() -> query
  var query = Dagoba.query(this)
  query.add('vertex', [].slice.call(arguments)) // add a step to our program
  return query
}
```

请注意`[].slice.call(arguments)`是JS的语法糖，这表示“请把这个函数的参数作为一个数组传递给我”。把参数表示成数组的形式你应该可以理解，因为在很多情况下的表现确实是这样，但是它仍然缺少许多我们在现代的JavaScript当中使用的数组的特性。

