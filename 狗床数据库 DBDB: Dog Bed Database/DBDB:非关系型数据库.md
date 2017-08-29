# DBDB: Dog Bed Database

As the newest bass (and sometimes tenor) in [Countermeasure](https://www.countermeasuremusic.com/), Taavi strives to break the mould... sometimes just by ignoring its existence. This is certainly true through the diversity of workplaces in his career: IBM (doing C and Perl), FreshBooks (all the things), Points.com (doing Python), and now at PagerDuty (doing Scala). Aside from that—when not gliding along on his Brompton folding bike—you might find him playing Minecraft with his son or engaging in parkour (or rock climbing, or other adventures) with his wife. He knits continental.

## 简介

DBDB (狗床数据库) 是一个python实现的简单的键值对存储数据库(key/value database)。它把键与值相关联，并将该关联存储在磁盘上。

DBDB 的特点是在电脑崩溃或程序出错的时候也能保证数据的安全。它也避免了把所有数据同时保存在内存中，所以你可以储存比内存容量更多的数据。

## 往事

记得有一次我在找一个BASIC程序里的bug的时候，电脑屏幕上突然出现了几个闪烁的光点，然后程序就提前终止了。当我再次检查代码的时候，代码的最后几行消失了。

我问了我麻麻的一个懂编程的朋友，然后我意识到程序崩溃的原因是程序太大了，占用到了电脑的显存。屏幕上闪烁的光点是苹果BASIC电脑的一个特性，这表示内存不够用了。

从那一刻开始，我变得非常注意内存的使用。我学习了指针和malloc，我学习了数据是怎么存放进内存的。我会非常非常小心的使用内存

几年后，在学习一个面想过程的语言 Erlang 的时候，我懂得了进程间的通信不需要把数据再拷贝一份以供另一个进程使用，因为所有数据都是不可变的（immutable）。然后我就喜欢上了 Clojure 的那些不可变的数据结构。

当我在2013年看到CouchDB的时候，我认识到了管理复杂数据的结构和机制。

我认识到了可以设计一个以不可变的(immutable)数据为基础的系统。

我写了一了一些文章，

因为我觉着描述 CouchDB (根据我的理解) 的核心数据储存原理会很有有意思。

当我写到一个二叉树自平衡的算法的时候，我才发现这个算法描述起来好复杂啊，边缘情况(edge case) 太多了，比如说解释二叉树的一处改变的时候，树的其他部分为什么也会改变。然后我就不知道怎么写了。

吸取了那次的经验教训，我仔细观察了递归算法来更新不可变二叉树，感觉这个算法更简单明了，和蔼可亲。

所以我又双叒叕一次感受到了不可变的结构的好处。

然后，就有了DBDB

## 为啥这个项目很有趣？

很多项目都会使用到数据库。但是你没有必要去自己写一个，就算只是存储json文件在磁盘上，也会有各种各样的边缘情况(edge case)需要去考虑，比如说：


- 存储空间不足时会发生什么？
- 电脑没电了，数据怎么保存？
- 如果数据大小超过可用内存怎么办？ （台式机上的大多数应用程序基本不会出现这种情况，但移动设备或服务器端Web应用程序可能会发生）

如果自己写一个数据库，你就知道数据库是怎么处理这些情况的了。

我们在这里讨论的技术和概念应适用于应对各种情况（包括发生故障时）。

关于不足。。。

## DBDB 的不足之处

数据库为保证事务（transaction）是正确可靠的，所必须具备的四个特性(ACID属性)：原子性(atomicity)，一致性(consistency)，独立性(isolation)和持久性(durability)。

DBDB 的数据更新(Updates)方法具有原子性和持久性。这两个属性在后面的章节会做详细介绍。因为我们没有对储存的数据做限制，所以DBDB 不能保持数据的一致性。独立性在DBDB里也没有实现。

应用程序的代码当然可以保证自己的一致性，但是实现独立性需要一个事务管理器(transaction manager)。 我们不会在这里试图增加事务管理器的部分，但是你可以通过 [CircleDB chapter](http://aosabook.org/en/500L/an-archaeology-inspired-database.html) 来进一步了解事务管理器。

还有一些维护系统的问题需要考虑。 在DBDB中，陈旧数据(Stale data)不会被收回处理，因此更新(Updates)（甚至是使用相同的键值）将会导致终消耗掉所有的磁盘空间。 （你会很快发现为什么会这样）。[PostgreSQL]（https://www.postgresql.org/）把处理陈旧数据的过程称之为“吸尘器(vacuuming)”，这使得旧的行空间可以重新使用，[CouchDB] （http://couchdb.apache.org/）把处理陈旧数据的过程称之为“压缩”，通过把正常数据(live data)的部分重写入新文件，并替代旧的文件。

DBDB 可以添加“压缩旧数据”的功能，这给就留给读者们作为小练习了【尾注1】。

## The Architecture of DBDB

DBDB separates the concerns of "put this on disk somewhere" (how data are laid out in a file; the physical layer) from the logical structure of the data (a binary tree in this example; the logical layer) from the contents of the key/value store (the association of key `a` to value `foo`; the public API).

Many databases separate the logical and physical aspects as it is is often useful to provide alternative implementations of each to get different performance characteristics, e.g. DB2's SMS (files in a filesystem) versus DMS (raw block device) tablespaces, or MySQL's [alternative engine implementations](https://dev.mysql.com/doc/refman/5.7/en/storage-engines.html).

## Discovering the Design

Most of the chapters in this book describe how a program was built from inception to completion. However, that is not how most of us interact with the code we're working on. We most often discover code that was written by others, and figure out how to modify or extend it to do something different.

In this chapter, we'll assume that DBDB is a completed project, and walk through it to learn how it works. Let's explore the structure of the entire project first.

## Organisational Units

Units are ordered here by distance from the end user; that is, the first module is the one that a user of this program would likely need to know the most about, while the last is something they should have very little interaction with.

- `tool.py` defines a command-line tool for exploring a database from a terminal window.

- `interface.py` defines a class (`DBDB`) which implements the Python dictionary API using the concrete `BinaryTree` implementation. This is how you'd use DBDB inside a Python program.

- `logical.py` defines the logical layer. It's an abstract interface to a key/value store.

    - `LogicalBase` provides the API for logical updates (like get, set, and commit) and defers to a concrete subclass to implement the updates themselves. It also manages storage locking and dereferencing internal nodes.

    - `ValueRef` is a Python object that refers to a binary blob stored in the database. The indirection lets us avoid loading the entire data store into memory all at once.

- `binary_tree.py` defines a concrete binary tree algorithm underneath the logical interface.

  - `BinaryTree` provides a concrete implementation of a binary tree, with methods for getting, inserting, and deleting key/value pairs. BinaryTree represents an immutable tree; updates are performed by returning a new tree which shares common structure with the old one.

  - `BinaryNode` implements a node in the binary tree.

  - `BinaryNodeRef` is a specialised `ValueRef` which knows how to serialise and deserialise a `BinaryNode`.

- `physical.py` defines the physical layer. The `Storage` class provides persistent, (mostly) append-only record storage.

These modules grew from attempting to give each class a single responsibility. In other words, each class should have only one reason to change.

## Reading a Value

We'll start with the simplest case: reading a value from the database. Let's see what happens when we try to get the value associated with key `foo` in `example.db`:

```
$ python -m dbdb.tool example.db get foo
```

This runs the `main()` function from module `dbdb.tool`:

```python
# dbdb/tool.py
def main(argv):
    if not (4 <= len(argv) <= 5):
        usage()
        return BAD_ARGS
    dbname, verb, key, value = (argv[1:] + [None])[:4]
    if verb not in {'get', 'set', 'delete'}:
        usage()
        return BAD_VERB
    db = dbdb.connect(dbname)          # CONNECT
    try:
        if verb == 'get':
            sys.stdout.write(db[key])  # GET VALUE
        elif verb == 'set':
            db[key] = value
            db.commit()
        else:
            del db[key]
            db.commit()
    except KeyError:
        print("Key not found", file=sys.stderr)
        return BAD_KEY
    return OK
```

The `connect()` function opens the database file (possibly creating it, but never overwriting it) and returns an instance of `DBDB`:
```python
# dbdb/__init__.py
def connect(dbname):
    try:
        f = open(dbname, 'r+b')
    except IOError:
        fd = os.open(dbname, os.O_RDWR | os.O_CREAT)
        f = os.fdopen(fd, 'r+b')
    return DBDB(f)
```

```python
# dbdb/interface.py
class DBDB(object):

    def __init__(self, f):
        self._storage = Storage(f)
        self._tree = BinaryTree(self._storage)
```
We see right away that `DBDB` has a reference to an instance of `Storage`, but it also shares that reference with `self._tree`. Why? Can't `self._tree` manage access to the storage by itself?

The question of which objects "own" a resource is often an important one in a design, because it gives us hints about what changes might be unsafe. Let's keep that question in mind as we move on.

Once we have a DBDB instance, getting the value at `key` is done via a dictionary lookup (`db[key]`), which causes the Python interpreter to call `DBDB.__getitem__()`.

```python
# dbdb/interface.py
class DBDB(object):
# ...
    def __getitem__(self, key):
        self._assert_not_closed()
        return self._tree.get(key)

    def _assert_not_closed(self):
        if self._storage.closed:
            raise ValueError('Database closed.')
```

`__getitem__()` ensures that the database is still open by calling `_assert_not_closed`. Aha! Here we see at least one reason why `DBDB` needs direct access to our `Storage` instance: so it can enforce preconditions. (Do you agree with this design? Can you think of a different way that we could do this?)

DBDB then retrieves the value associated with `key` on the internal `_tree` by calling `_tree.get()`, which is provided by `LogicalBase`:

```python
# dbdb/logical.py
class LogicalBase(object):
# ...
    def get(self, key):
        if not self._storage.locked:
            self._refresh_tree_ref()
        return self._get(self._follow(self._tree_ref), key)
```

`get()` checks if we have the storage locked. We're not 100% sure why there might be a lock here, but we can guess that it probably exists to allow writers to serialize access to the data. What happens if the storage isn't locked?

```python
# dbdb/logical.py
class LogicalBase(object):
# ...
def _refresh_tree_ref(self):
        self._tree_ref = self.node_ref_class(
            address=self._storage.get_root_address())
```

`_refresh_tree_ref` resets the tree's "view" of the data with what is currently on disk, allowing us to perform a completely up-to-date read.

What if storage is locked when we attempt a read? This means that some other process is probably changing the data we want to read right now; our read is not likely to be up-to-date with the current state of the data. This is generally known as a "dirty read". This pattern allows many readers to access data without ever worrying about blocking, at the expense of being slightly out-of-date.

For now, let's take a look at how we actually retrieve the data:

```python
# dbdb/binary_tree.py
class BinaryTree(LogicalBase):
# ...
    def _get(self, node, key):
        while node is not None:
            if key < node.key:
                node = self._follow(node.left_ref)
            elif node.key < key:
                node = self._follow(node.right_ref)
            else:
                return self._follow(node.value_ref)
        raise KeyError
```

This is a standard binary tree search, following refs to their nodes. We know from reading the `BinaryTree` documentation that `Node`s and `NodeRef`s are value objects: they are immutable and their contents never change. `Node`s are created with an associated key and value, and left and right children. Those associations also never change. The content of the whole `BinaryTree` only visibly changes when the root node is replaced. This means that we don’t need to worry about the contents of our tree being changed while we are performing the search.

Once the associated value is found, it is written to `stdout` by `main()` without adding any extra newlines, to preserve the user’s data exactly.

## Inserting and Updating

Now we’ll set key `foo` to value `bar` in `example.db`:

```
$ python -m dbdb.tool example.db set foo bar
```

Again, this runs the `main()` function from module `dbdb.tool`. Since we’ve seen this code before, we’ll just highlight the important parts:

``` python
# dbdb/tool.py
def main(argv):
    ...
    db = dbdb.connect(dbname)          # CONNECT
    try:
        ...
        elif verb == 'set':
            db[key] = value            # SET VALUE
            db.commit()                # COMMIT
        ...
    except KeyError:
        ...
```

This time we set the value with `db[key] = value` which calls `DBDB.__setitem__()`.

``` python
# dbdb/interface.py
class DBDB(object):
# ...
    def __setitem__(self, key, value):
        self._assert_not_closed()
        return self._tree.set(key, value)
```

`__setitem__` ensures that the database is still open and then stores the association from `key` to `value` on the internal `_tree` by calling `_tree.set()`.

`_tree.set()` is provided by `LogicalBase`:

``` python
# dbdb/logical.py
class LogicalBase(object):
# ...
    def set(self, key, value):
        if self._storage.lock():
            self._refresh_tree_ref()
        self._tree_ref = self._insert(
            self._follow(self._tree_ref), key, self.value_ref_class(value))
```

`set()` first checks the storage lock:

``` python
# dbdb/storage.py
class Storage(object):
    ...
    def lock(self):
        if not self.locked:
            portalocker.lock(self._f, portalocker.LOCK_EX)
            self.locked = True
            return True
        else:
            return False
```

There are two important things to note here:

-   Our lock is provided by a 3rd-party file-locking library called [portalocker](https://pypi.python.org/pypi/portalocker).
-   `lock()` returns `False` if the database was already locked, and `True` otherwise.

Returning to `_tree.set()`, we can now understand why it checked the return value of `lock()` in the first place: it lets us call `_refresh_tree_ref` for the most recent root node reference so we don’t lose updates that another process may have made since we last refreshed the tree from disk. Then it replaces the root tree node with a new tree containing the inserted (or updated) key/value.

Inserting or updating the tree doesn’t mutate any nodes, because `_insert()` returns a new tree. The new tree shares unchanged parts with the previous tree to save on memory and execution time. It’s natural to implement this recursively:

``` python
# dbdb/binary_tree.py
class BinaryTree(LogicalBase):
# ...
    def _insert(self, node, key, value_ref):
        if node is None:
            new_node = BinaryNode(
                self.node_ref_class(), key, value_ref, self.node_ref_class(), 1)
        elif key < node.key:
            new_node = BinaryNode.from_node(
                node,
                left_ref=self._insert(
                    self._follow(node.left_ref), key, value_ref))
        elif node.key < key:
            new_node = BinaryNode.from_node(
                node,
                right_ref=self._insert(
                    self._follow(node.right_ref), key, value_ref))
        else:
            new_node = BinaryNode.from_node(node, value_ref=value_ref)
        return self.node_ref_class(referent=new_node)
```

Notice how we always return a new node (wrapped in a `NodeRef`). Instead of updating a node to point to a new subtree, we make a new node which shares the unchanged subtree. This is what makes this binary tree an immutable data structure.

You may have noticed something strange here: we haven’t made any changes to anything on disk yet. All we’ve done is manipulate our view of the on-disk data by moving tree nodes around.

In order to actually write these changes to disk, we need an explicit call to `commit()`, which we saw as the second part of our `set` operation in `tool.py` at the beginning of this section.

Committing involves writing out all of the dirty state in memory, and then saving the disk address of the tree’s new root node.

Starting from the API:

``` python
# dbdb/interface.py
class DBDB(object):
# ...
    def commit(self):
        self._assert_not_closed()
        self._tree.commit()
```

The implementation of `_tree.commit()` comes from `LogicalBase`:

``` python
# dbdb/logical.py
class LogicalBase(object)
# ...
    def commit(self):
        self._tree_ref.store(self._storage)
        self._storage.commit_root_address(self._tree_ref.address)
```

All `NodeRef`s know how to serialise themselves to disk by first asking their children to serialise via `prepare_to_store()`:

``` python
# dbdb/logical.py
class ValueRef(object):
# ...
    def store(self, storage):
        if self._referent is not None and not self._address:
            self.prepare_to_store(storage)
            self._address = storage.write(self.referent_to_string(self._referent))
```

`self._tree_ref` in `LogicalBase` is actually a `BinaryNodeRef` (a subclass of `ValueRef`) in this case, so the concrete implementation of `prepare_to_store()` is:

``` python
# dbdb/binary_tree.py
class BinaryNodeRef(ValueRef):
    def prepare_to_store(self, storage):
        if self._referent:
            self._referent.store_refs(storage)
```

The `BinaryNode` in question, `_referent`, asks its refs to store themselves:

``` python
# dbdb/binary_tree.py
class BinaryNode(object):
# ...
    def store_refs(self, storage):
        self.value_ref.store(storage)
        self.left_ref.store(storage)
        self.right_ref.store(storage)
```

This recurses all the way down for any `NodeRef` which has unwritten changes (i.e., no `_address`).

Now we’re back up the stack in `ValueRef`’s `store` method again. The last step of `store()` is to serialise this node and save its storage address:

``` python
# dbdb/logical.py
class ValueRef(object):
# ...
    def store(self, storage):
        if self._referent is not None and not self._address:
            self.prepare_to_store(storage)
            self._address = storage.write(self.referent_to_string(self._referent))
```

At this point the `NodeRef`’s `_referent` is guaranteed to have addresses available for all of its own refs, so we serialise it by creating a bytestring representing this node:

``` python
# dbdb/binary_tree.py
class BinaryNodeRef(ValueRef):
# ...
    @staticmethod
    def referent_to_string(referent):
        return pickle.dumps({
            'left': referent.left_ref.address,
            'key': referent.key,
            'value': referent.value_ref.address,
            'right': referent.right_ref.address,
            'length': referent.length,
        })
```

Updating the address in the `store()` method is technically a mutation of the `ValueRef`. Because it has no effect on the user-visible value, we can consider it to be immutable.

Once `store()` on the root `_tree_ref` is complete (in `LogicalBase.commit()`), we know that all of the data are written to disk. We can now commit the root address by calling:

``` python
# dbdb/physical.py
class Storage(object):
# ...
    def commit_root_address(self, root_address):
        self.lock()
        self._f.flush()
        self._seek_superblock()
        self._write_integer(root_address)
        self._f.flush()
        self.unlock()
```

We ensure that the file handle is flushed (so that the OS knows we want all the data saved to stable storage like an SSD) and write out the address of the root node. We know this last write is atomic because we store the disk address on a sector boundary. It’s the very first thing in the file, so this is true regardless of sector size, and single-sector disk writes are guaranteed to be atomic by the disk hardware.

Because the root node address has either the old or new value (never a bit of old and a bit of new), other processes can read from the database without getting a lock. An external process might see the old or the new tree, but never a mix of the two. In this way, commits are atomic.

Because we write the new data to disk and call the `fsync` syscall[2] before we write the root node address, uncommitted data are unreachable. Conversely, once the root node address has been updated, we know that all the data it references are also on disk. In this way, commits are also durable.

We’re done!

## How NodeRefs Save Memory

To avoid keeping the entire tree structure in memory at the same time, when a logical node is read in from disk the disk address of its left and right children (as well as its value) are loaded into memory. Accessing children and their values requires one extra function call to `NodeRef.get()` to dereference (“really get”) the data.

All we need to construct a `NodeRef` is an address:

    +---------+
    | NodeRef |
    | ------- |
    | addr=3  |
    | get()   |
    +---------+

Calling `get()` on it will return the concrete node, along with that node’s references as `NodeRef`s:

    +---------+     +---------+     +---------+
    | NodeRef |     | Node    |     | NodeRef |
    | ------- |     | ------- | +-> | ------- |
    | addr=3  |     | key=A   | |   | addr=1  |
    | get() ------> | value=B | |   +---------+
    +---------+     | left  ----+
                    | right ----+   +---------+
                    +---------+ |   | NodeRef |
                                +-> | ------- |
                                    | addr=2  |
                                    +---------+

When changes to the tree are not committed, they exist in memory with references from the root down to the changed leaves. The changes aren’t saved to disk yet, so the changed nodes contain concrete keys and values and no disk addresses. The process doing the writing can see uncommitted changes and can make more changes before issuing a commit, because `NodeRef.get()` will return the uncommitted value if it has one; there is no difference between committed and uncommitted data when accessed through the API. All the updates will appear atomically to other readers because changes aren’t visible until the new root node address is written to disk. Concurrent updates are blocked by a lockfile on disk. The lock is acquired on first update, and released after commit.

## Exercises for the Reader

DBDB allows many processes to read the same database at once without blocking; the tradeoff is that readers can sometimes retrieve stale data. What if we needed to be able to read some data consistently? A common use case is reading a value and then updating it based on that value. How would you write a method on `DBDB` to do this? What tradeoffs would you have to incur to provide this functionality?

The algorithm used to update the data store can be completely changed out by replacing the string `BinaryTree` in `interface.py`. Data stores tend to use more complex types of search trees such as B-trees, B+ trees, and others to improve the performance. While a balanced binary tree (and this one isn’t) needs to do O(log2(n)) random node reads to find a value, a B+ tree needs many fewer, for example O(log{32}(n)) because each node splits 32 ways instead of just 2. This makes a huge different in practice, since looking through 4 billion entries would go from log2(2^{32}) = 32 to log{32}(2^{32}) approx 6.4 lookups. Each lookup is a random access, which is incredibly expensive for hard disks with spinning platters. SSDs help with the latency, but the savings in I/O still stand.

By default, values are stored by `ValueRef` which expects bytes as values (to be passed directly to `Storage`). The binary tree nodes themselves are just a sublcass of `ValueRef`. Storing richer data via [json](http://json.org) or [msgpack](http://msgpack.org) is a matter of writing your own and setting it as the `value_ref_class`. `BinaryNodeRef` is an example of using [pickle](https://docs.python.org/3.4/library/pickle.html) to serialise data.

Database compaction is another interesting exercise. Compacting can be done via an infix-of-median traversal of the tree writing things out as you go. It’s probably best if the tree nodes all go together, since they’re what’s traversed to find any piece of data. Packing as many intermediate nodes as possible into a disk sector should improve read performance, at least right after compaction. There are some subtleties here (for example, memory usage) if you try to implement this yourself. And remember: always benchmark performance enhancements before and after! You’ll often be surprised by the results.

## Patterns and Principles

Test interfaces, not implementation. As part of developing DBDB, I wrote a number of tests that described how I wanted to be able to use it. The first tests ran against an in-memory version of the database, then I extended DBDB to persist to disk, and even later added the concept of NodeRefs. Most of the tests didn’t have to change, which gave me confidence that things were still working.

Respect the Single Responsibility Principle. Classes should have at most one reason to change. That’s not strictly the case with DBDB, but there are multiple avenues of extension with only localised changes required. Refactoring as I added features was a pleasure!

## Summary

DBDB is a simple database that makes simple guarantees, and yet things still became complicated in a hurry. The most important thing I did to manage this complexity was to implement an ostensibly mutable object with an immutable data structure. I encourage you to consider this technique the next time you find yourself in the middle of a tricky problem that seems to have more edge cases than you can keep track of.

------------------------------------------------------------------------

1. Bonus feature: Can you guarantee that the compacted tree structure is balanced? This helps maintain performance over time.

2. Calling `fsync` on a file descriptor asks the operating system and hard drive (or SSD) to write all buffered data immediately. Operating systems and drives don’t usually write everything immediately in order to improve performance.
