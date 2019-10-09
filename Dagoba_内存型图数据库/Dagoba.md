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

一个精灵(gremlin)是一种在图中游走来完成我们的目标的生物。精灵可能是一个数据库当中最令人惊奇的部分，它们从Tinkerpop里面的Blueprints，以及Gremlin和Pacer查询语句衍生出来。它们记得自己的位置，并且可以回答一些有意思的问题。

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

## 立即(Eager)求值带来的问题

在讲述管道类型之前，让我们先看一下精彩的策略执行部分。这里面有两个非常主要的流派：一个是“值调用(Call By Value)”家族，也被称作“忙碌海狸(eager beavers)”，这一流派要求函数的所有参数都要在被调用之前完成求值；而另一个相反的流派，被称作“按需调用(Call By Needians)”，将所有的事情都推到不得不做的时候才会去做，换个说法，就是惰性的。

JavaScript是一个严格(strict)的语言，会在每个步骤被调用的时候对他们进行处理。要对`g.v('Thor').out().in()`这个表达式进行求值，我们预期会是这样的步骤：首先找到Thor这个节点，然后找出这一节点在出方向上直接相连的节点，然后再找出这些节点在入方向上相连的所有节点

如果使用非严格(non-strict)的语言，我们也会获得同样的结果，策略执行方式并不造成什么影响。但是，如果我们再加一些调用呢？如果Thor是一个社会关系复杂的人，那么`g.v('Thor').out().out().out().in().in().in()`这个查询会产生非常多的结果，因为我们并不要求结果是经过去重(unique)的，结果甚至会比整个图的节点数量都还多。

我们可能只需要得出一部分不重复的结果，所以我们可以把查询改写成这样：`g.v('Thor').out().out().out().in().in().in().unique().take(10)`。现在我们的查询结果最多有10条。如果我们立即计算会发生什么？我们仍然要求出天文数字一样多的所有结果，而最后只返回前10条。

所有的图数据库都为了尽可能的少做一些计算而努力，也因此大多选择了非严格求值模型。因为我们在构建自己的解释器，因此是可以对程序做到惰性求值的。但也因此不得不面对一些后果。

## 这种心智模型(Mental Model)带来的求值策略的结果

目前为止，我们求值的心智模型都是非常简单的：

<li>查询一个节点集合</li>
<li>将求得的集合作为输入传递给一个管道</li>
<li>如果有必要的话，重复这些步骤</li>

给用户呈现这种模型也没有什么问题，因为在这种模型上很容易做一些推断，但是我们上一个小节已经论证过了，实现(implementation)层面不能直接使用这种模型。用户认知的模型和实际实现之间有偏差，这也是痛苦的来源。抽象泄漏(leaky abstraction)还算是个小问题，更严重的，还会导致故障、认知混淆等种种问题。

在这种骗局里面，我们面对的几乎是最佳场景：无论实际的执行模型如何，对于任何查询，答案总是一样的。唯一的区别在于性能。我们可以要求用户在使用系统之前充分了解一个较为复杂的模型，也可以专注于一部分用户，将简单模型迁移到复杂模型来获得更好的查询性能，总之，我们必须在这两者之间做出权衡。

为了做出决策，你需要考虑下面这些因素：

<li>简单模型和复杂模型之间的认知难度差异</li>
<li>先使用简单模型再进阶到复杂模型，还是跳过简单模型直接使用复杂模型。这两者区别带来的认知负担</li>
<li>需要进行过渡的用户群体特征，包括用户群体比例、认知能力和可用时间等等</li>

在我们的场景下，这种权衡是合理的。对于大多数的请求来说，都会很快返回结果，因此用户无须特别的优化查询结构或是去了解更深层次的模型。而有一些用户会在大数据集上面使用一些高级(advanced)查询，这些人同样也是更加适于过渡到新模型的用户。此外，我们希望在学习复杂模型之前使用一个简单模型并不会带来非常大的困难。

我们会很快深入到这个新模型的一些细节当中，在进行下一章的过程中，希望你能记住这些重点：

<li>每个管道一次只返回一条结果，而不是一个结果集合。在执行查询过程中，每个管道都有可能会被激活多次</li>
<li>用一个读写头(read/write head)来控制接下来执行哪个管道。头的初始位置在流水线的最末端，根据当前被激活的管道的输出结果来决定这个头如何移动</li>
 <li>输出结果可能是上面提及的gremlin之一。每个gremlin代表一个可能的查询结果，他们通过管道来携带一些状态信息。gremlin使得读写头右移。</li>
<li>管道可能会返回一个“拉取(pull)”结果，这代表着它需要输入数据，让读写头进行右移</li>
<li>如果结果是“完成(done)”，这意味着在这之前的管道都不会再次被激活了，读写头会进行左移。

## 管道类型

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

请注意，添加一个新的管道类型会将原来已有的同名类型替换掉，这样就允许在运行过程中修改已经存在的管道类型。这个决策会带来什么？有没有什么替代方案？

```
Dagoba.getPipetype = function(name) {
  var pipetype = Dagoba.Pipetypes[name]                 // a pipetype is a function

  if(!pipetype)
    Dagoba.error('Unrecognized pipetype: ' + name)

  return pipetype || Dagoba.fauxPipetype
}
```

如果我们找不到某个管道类型，会生成一个错误信息，然后返回一个默认的管道类型，这种管道就像一个空管子：如果接收到一个消息，就把它向另一端传递出去。

```
Dagoba.fauxPipetype = function(_, _, maybe_gremlin) {   // pass the result upstream
  return maybe_gremlin || 'pull'                        // or send a pull downstream
}
```

注意到下划线了吗？我们使用这个标记来表示函数当中不会使用这些参数。这三个参数大部分函数都要全部使用，这时候三个参数都会被命名。这让我们可以一眼看出来某个特定的管道类型依赖于哪些参数。

下划线的表示方法非常重要，因为它使得注释行更加容易对齐。哈，并不是，我们严肃一些。“程序一定是为让人阅读而写，只是恰好可以被机器执行”，那么接下来我们最关注的是使我们的代码更漂亮。

### 节点(Vertex)

我们会用到的大多数管道类型都是使用一个gremlin来产生更多的gremlin，但是这个特定的管道类型会根据一个字符串来生成gremlin。给定一个节点编号，它就会返回一个新的gremlin。给定一个查询，它会找到所有匹配的节点，并且每次生成一个gremlin，直到所有匹配的节点都被遍历过。

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

首先我们检查一下是不是所有匹配的节点都已经集齐，如果不是的话我们需要查找一些结果。如果有任何匹配的节点，我们会取出其中一个，然后返回一个新的gremlin，并把这个节点传递给gremlin。每个gremlin都携带了自己的状态，就像一个日志(journal)，记录了它都到过那里，在这个图上的旅途都有什么有意思的事情。如果我们收到了一个gremlin作为一个步骤的输入，我们会复制它所有的日志。

注意我们在这里直接修改了`state`参数，并且没有将它回传。另一个做法可以返回一个对象，而不是返回一个gremlin或者信号，并且通过这个对象来把状态进行回传。这会使得我们的返回值变得复杂，增加一些额外的垃圾(garbage)。如果JS允许多返回值，这个地方还可以写的更为优雅。

我们还是需要解决修改变量(mutation)的问题，因为调用者仍然有可能持有原始变量的引用。有没有什么方法可以让我们确定某个引用是“排他(unique)”的？——也即当前的这个引用是这个对象唯一的引用。

如果我们能确定某个引用是排他的，那么我们就可以利用不变性(immutability)的良好性质，同时避免了写时复制或者复杂的持久化数据结构。仅通过一个引用，我们无法确定某个对象被修改过，还是返回了一个包含了我们请求的修改的新对象——是否保持了“可见的不变性”

有很多常见方法可以用来检测这种排他性：在静态类型系统中，可以使用排他类型(uniqueness type)在编译期保证每个对象只有一个引用。如果我们有一个引用计数器——甚至是仅有两bit的计数器——我们就能在运行时确定这个对象是否只有一个引用，并且利用这些优势。

JavaScript并没有这些方法可用，但是如果我们非常非常的自律，我们也可以做到一样的效果。至少到目前为止是这样。

## 出边和入边(In-N-Out)

遍历一个图就像点菜一样简单。下面的两行代码设定了`in`和`out`两种管道类型

```
Dagoba.addPipetype('out', Dagoba.simpleTraversal('out'))
Dagoba.addPipetype('in',  Dagoba.simpleTraversal('in'))
```

`simpleTraversal`函数返回了一个管道类型，接受一个gremlin作为输入，然后每次查询的时候生成一个新的gremlin。一旦这些gremlin都完成了之后，它会返回一个“拉取”请求，来从它的上一级获取新的gremlin。

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

最前面两行代码处理了出方向版本(in version)和入方向版本(out version)的区别。然后准备返回管道类型函数，和我们之前看过的`vertex`管道类型非常像。但有一点令人惊奇的是，这个管道接收了一个gremlin作为参数，而`vertex`管道类型是从无到有创造了一个gremlin出来。

但我们也看到了，出现了一些同样的情形，添加了查询初始化的步骤。如果没有给定gremlin，而且当前没有可用的边，就发起一次拉取。如果我们有一个gremlin但是还没有被设定状态，我们就在指定的方向上找到任意的边，然后把它添加到我们的状态当中。如果gremlin当前的节点没有合适的边可以添加，那么我们就再次发起拉取。最终我们会获取出来一条边，并且在这条边所指向的节点上复制一个新的gremlin以返回。

再看一眼这个代码，`!state.edges.length`在三个代码块里面被分别重复了一次。要不要对这部分进行重构，来降低这些条件的复杂度呢？这看起来很诱人，但是出于以下两点原因，我们不会这么做。

第一个原因比较弱：第三个`!state.edges.length`语句和前两个意义完全不一样，因为在第二个和第三个条件之间`state.edges`已经发生了变化。这实际上鼓励了我们去做重构，因为在一个函数内相同的标签却代表了不同的事物，这通常不是个理想状态。

第二个原因更严重一些。这不是我们需要写的唯一一个管道类型函数，我们会一次又一次的重复这种查询初始化以及状态初始化的工作。在编码过程中，始终要在结构化和非结构化之中做权衡。过分的结构化会让你在复杂的模板和抽象当中花费大量精力。太少的结构化会让你不得不在记忆中维护非常多的细枝末节。

在这个case当中，我们有十几个左右的管道类型需要处理，正确的做法应当是让所有的管道类型函数尽可能保持风格一致，并且在每个要素上添加注释。所以我们需要一直抑制重构的冲动，因为这么做会有损一致性。我们也要抑制那种想要购买一个通用的结构来抽象查询初始化、状态初始化等等步骤的想法。如果有上百种管道函数类型，那么后一种选择可能是合适的，抽象的复杂度带来的消耗是固定不变的，而收益会随着使用这一抽象的单元(units)数量增长而线性增长。当处理许许多多这样的移动部件(moving pieces)时，使用任何强制措施来保证部件之间的规律性都是有益的。

## 属性(Property)

让我们稍停片刻，基于我们已经介绍的三种管道类型，思考下面这个样例查询。我们可以像这样查询Thor的祖父母：

```
g.v('Thor').out('parent').out('parent').run()
```

如果我们还想知道他们的名字呢？我们可以在末尾加一个map调用：

```
g.v('Thor').out('parent').out('parent').run()
 .map(function(vertex) {return vertex.name})
 ```

但是我们可能更愿意采用一些更加普通的方法，比如这样：

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

查询的初始化缓解非常普通：如果没有gremlin，就发起拉取。如果已经有了gremlin，那么我们把属性值赋给它的`result`字段。这个gremlin还可以继续向前移动。如果这个gremlin通过了最后一个管道，那么`result`字段会收集结果并作为查询的结果返回 。不是所有的gremlin都有`result`这个字段，因此并不会返回它最近一次访问的节点信息。

请注意，如果指定的属性不存在，我们选择返回`false`而不是一个gremlin，所以属性管道也起到了按类型过滤的作用。你能想到它的一些用处么？这个设计决策上做了哪些权衡呢？

## 去重

如果我们希望找到Thor的所有祖父母的所有孙子女——他的表亲、他的兄弟姐妹以及他自己，我们会做这样一个查询：`g.v('Thor').in().in().out().out().run()`。然而，这会获得许多重复的结果。事实上，Thor自己就会至少重复出现四次。（你可以想出来什么场景下会重复次数更多么？）

为了解决这个问题，我们引入一个新的管道类型，命名为“去重(unique)”。我们的新查询会得出没有重复的结果：

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

## 过滤

我们已经讲过两种简单的过滤了，但是有些时候，我们会需要一些更加复杂的限制条件。如果想要找出Thor的兄弟们当中那些体重大于身高的人，我们可以这样查询:

```
g.v('Thor').out().in().unique()
 .filter(function(asgardian) { return asgardian.weight > asgardian.height })
 .run()
```

如果我们想要知道Thor的兄弟当中哪些人幸存下来，我们可以插入这样一个过滤器：

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
There are two reasons this error might arise. The first involves a programmer typing in a query, either in a REPL or directly in code. When run, that query will produce results, and also generate a programmer-observable error. The programmer then corrects the error to further filter the set of results produced. Alternatively, the system could display only the error and produce no results, and fixing all errors would allow results to be displayed.

第二种可能的原因是过滤器是在运行过程中被动态修改的。这是一种更加重要的场景，因为调用查询的人不一定是这个查询代码的作者。在Web领域，我们默认的守则是，永远显示结果，不打破任何东西。通常来说，遇到麻烦时硬着头皮撑场面要好于把严重的错误信息展示在用户面前。

对于那些显示太少结果好于显示过多结果的场景而言，可以重写`Dagoba.error`来抛出错误，来避开原生的控制流程。

## 截取(Take)

我们不总是想一次取回所有结果。有时我们只想取回相当少量的结果；比如说我们想找到十几个Thor的同龄人，我们又回到这个初始的问题：

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

我们的查询可以运行在一个异步化的环境里面，让我们可以再需要获取更多的结果时进行收集。当所有结果都取出之后，返回一个空数组。

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

如果`state.taken`还没有被初始化过，那就设它为0。JavaScript有隐式转换，但是默认会将未定义变量初始化为NaN，所以我们在这里进行显示的操作。

当`state.taken`到达了`args[0]`的限制，我们会返回'done'，停掉前面的管道。同时清掉`state.taken`计数器，允许我们稍后重复执行这个请求。

我们处理了这两个步骤之后才做查询的初始化操作，这是为了解决`take(0)`和`take()`的问题。然后我们增加计数器，然后返回gremlin。

## 别名(As)

接下来要介绍的四个管道类型组合在一起工作，让我们可以使用更多的进阶(advanced)查询。这一个管道类型允许我们给当前的节点进行标记。而下面的两个管道类型会使用这些标记。

```
Dagoba.addPipetype('as', function(graph, args, gremlin, state) {
  if(!gremlin) return 'pull'                                  // query initialization
  gremlin.state.as = gremlin.state.as || {}                   // init the 'as' state
  gremlin.state.as[args[0]] = gremlin.vertex                  // set label to vertex
  return gremlin
})
```

在查询完成初始化之后，我们要确保gremlin的本地状态当中一定有一个`as`参数。我们将这个gremlin当前的节点复制给这个`as`参数。

## 融合(Merge)

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

## 异常(Except)

我们已经介绍过了“给出Thor的所有不是Thor本人的兄弟姐妹”这种查询。我们可以通过`filter`来完成：

```
g.v('Thor').out().in().unique()
           .filter(function(asgardian) {return asgardian._id != 'Thor'}).run()
```

用`as`和`except`这两种管道会更直观一些：

```
g.v('Thor').as('me').out().in().except('me').unique().run()
```

也有一些查询很难进行直接过滤。如果我们想要查找Thor的叔叔和阿姨呢？我们怎样过滤掉他的父母呢？使用`as`和`except`会比较容易：

```
g.v('Thor').out().as('parent').out().in().except('parent').unique().run()

Dagoba.addPipetype('except', function(graph, args, gremlin, state) {
  if(!gremlin) return 'pull'                                  // query initialization
  if(gremlin.vertex == gremlin.state.as[args[0]]) return 'pull'
  return gremlin
})
```

这里我们检查当前的节点是不是和我们之前存储的一样。如果是这样就跳过它。

## 回溯(Back)

面对某一些问题时，我们可能需要在某一步去做更进一步的验证，如果验证得到的结果是肯定的，再回到原来的那个地方。设想我们想知道Fjörgynn的哪些女儿育有Bestla的一个儿子？

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

# 辅助函数

以上介绍的管道类型依赖于一些辅助函数来完成计算。让我们在深入解释器之前快速浏览一下这些函数。

