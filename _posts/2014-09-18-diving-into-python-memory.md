---
layout: post
category: Dev
title: Diving into Python Memory to track leak and ballooning
---


Memory leak in python can’t really happens due to the work done by the [garbage collector](http://www.digi.com/wiki/developer/index.php/Python_Garbage_Collection).

However, ballooning is still possible, and cyclic references sometimes are hard to track.

Fortunately, we can get help from several tools :

* [objgraph:](http://mg.pov.lt/objgraph/) A super usefull module to visually explore Python object graphs
* [guppy:](http://guppy-pe.sourceforge.net) A Heap Analysis Toolset
* [ipdb/ipython:](http://ipython.org) A Powerful interactive python shell

Let’s see with a real example how these tools can be useful to track origin of memory issues.

As a pet project I’m contributing to [Archipel](http://www.archipelproject.org) a virtual machine orchestrator based on libvirt and using XMPP as an underlying communication system (which is really cool !).

This software is based on [xmpppy](http://xmpppy.sourceforge.net) a XMPP python library no longer maintained nor developed but pretty easy to use and flexible enough to make everything we wanted to.

I notice that the main daemons (archipel-agent and archipel-central-agent ) were really eating more and more memory accros the time and sometime took all the available memory (32Gb for a python daemon is pretty *huge* regarding what was really done).

So I needed some tools to see what was really stored in the memory and why I grown endlessly.

## Bring it on !

This issue is quite hard to reproduce as we have to wait enough time to see what is eating the memory.

So first thing was to add an ipdb trace handler (so I can fire up ipdb console whenever I wanted to) by adding the following code to the main program loop:

```python
    sig.signal(sig.SIGTERM, exit_handler)
    import signal,resource
    try:
        import ipdb as pdb
    except:
        import pdb

    def handle_signal(signal_number, frame_stack):
        import objgraph, resource
        print 'Memory : %s (kb)' % (resource.getrusage(resource.RUSAGE_SELF).ru_maxrss))
        pdb.set_trace()

    signal.signal(signal.SIGINT, handle_signal)
```

This way, each time I hit **ctrl+c**, it raises an ipdb console (and also showing the actual state of memory).

From this prompt, I can use whatever python thing to track and understand where the issue hide.

**TIPS:*** to be able to write multilines function you can run inside your ipdb prompt type the following:

``` python
!import code; code.interact(local=vars())
```

From there I can start to inspect the memory using [guppy](http://guppy-pe.sourceforge.net) and create some useful graph using [objgraph](http://mg.pov.lt/objgraph/).

## Clean the dust

First of all, you should clean everything cleanable using the garbage collector and only after that you can have some informations about the memory taken by your process.

```python
>>> import gc
>>> gc.collect()
6551
>>> resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
64316
```
Now let’s digg on memory.

## These are not the droids you are looking for

With [guppy](http://guppy-pe.sourceforge.net) you can print the heap:

```python
In : import guppy
In : hp=guppy.hpy()
In : h= hp.heap()
In : h
Partition of a set of 343006 objects. Total size = 59627592 bytes.
 Index  Count   %     Size   % Cumulative  % Kind (class / dict of class)
     0 115457  34 12238120  21  12238120  21 str
     1  36156  11 11455008  19  23693128  40 dict (no owner)
     2   7591   2  7955368  13  31648496  53 dict of xmpp.simplexml.Node
     3  55817  16  4811344   8  36459840  61 tuple
     4  23899   7  3405328   6  39865168  67 unicode
     5   2439   1  2556072   4  42421240  71 dict of xmpp.protocol.Iq
     6  23194   7  2505488   4  44926728  75 list
     7    588   0  1931808   3  46858536  79 dict of module
     8  14401   4  1728120   3  48586656  81 types.CodeType
     9  14104   4  1692480   3  50279136  84 function
<830 more rows. Type e.g. '_.more' to view.>
```

As you can see, there is a lot of strings/dicts... but hold on, what are we really looking for ?

Our daemons are quite simple manipulate XML objects, receive and send XMPP stanza (which are also XML objects) and that's all, so we will digg on these items.

A Stanza can be a *Message* or an *IQ* and composed from *Nodes*, Nodes contains *dicts* and dicts contains *strings*.

As we can see we have Node and IQ taking a big part of memory. Let’s dig on it

```
In : h[5].domisize
25025512
```
It seems we have 25MB of IQ stored in memory, this is huge ! (We do not keep an sort of history)

```
In : h[5].byid
Set of 2439 <dict of xmpp.protocol.Iq> objects. Total size = 2556072 bytes.
 Index     Size   %   Cumulative  %   Owner Address
     0     1048   0.0      1048   0.0 0x19446d0
     1     1048   0.0      2096   0.1 0x1f9f790
     2     1048   0.0      3144   0.1 0x23e5750
     3     1048   0.0      4192   0.2 0x25517d0
     4     1048   0.0      5240   0.2 0x2551ad0
     5     1048   0.0      6288   0.2 0x2551b90
     6     1048   0.0      7336   0.3 0x2a75dd0
     7     1048   0.0      8384   0.3 0x2a75f10
     8     1048   0.0      9432   0.4 0x2a961d0
     9     1048   0.0     10480   0.4 0x2a96790
<2429 more rows. Type e.g. '_.more' to view.>
```

2500 object of size 1048, interresting… Let’s use [objgraph](http://mg.pov.lt/objgraph/) now to get more information about relationship

With [objgraph](http://mg.pov.lt/objgraph/) you can walk through object types like this:

```python
import objgraph
In : objgraph.show_most_common_types()
dict                       59731
tuple                      56281
list                       24216
function                   14161
Node                       7857
instance                   5824
instancemethod             3791
weakref                    2704
Iq                         2512
builtin_function_or_method 1759
```

So here we can see that we have a lot of Iq objects as we saw previously. Let’s see what are they:

```python
In: for iq in objgraph.by_type('Iq'): print iq
...
```

---snip---

```xml
<iq to="admin@central-server.archipel.priv/ArchipelController" from="agent-1.archipel.priv@central-server.archipel.priv/agent-1.archipel.priv" id="9394860" type="result"><query xmlns="archipel:xmppserver"><users xmpp="True" xmlrpc="False" /><groups xmpp="False" xmlrpc="True" /></query></iq>
<iq to="admin@central-server.archipel.priv/ArchipelController" from="agent-1.archipel.priv@central-server.archipel.priv/agent-1.archipel.priv" id="9396011" type="result"><query xmlns="archipel:xmppserver"><users xmpp="True" xmlrpc="False" /><groups xmpp="False" xmlrpc="True" /></query></iq>
```

---snip---

It's Iq responses… there is a lot of them almost identical (except the id) they *should* be cleaned by the GC, but it's not the case, they must have some pending references, let's see whitch are the hodlers.

Objgraph can generate pretty usefull graph regarding references:

```python
In : objgraph.show_backrefs(objgraph.by_type('Iq')[0], filename="/vagrant/g.png",refcounts=True, max_depth=5)
Graph written to /tmp/objgraph-a08L96.dot (10 nodes)
Image generated as /vagrant/g.png
```

Here is the result:

<p align="center">
  <img src="/data/xmpp-cyclicref.png" title="Back Refences on IQ object" />
</p>


Oh a Cyclic reference (in red) ! Normally the garbage collector should take care of that, but it is strongly recommended to deal with them at the source. There is plenty of way to fix this, I choose to use the [Weakref](https://docs.python.org/2/library/weakref.html).

Parent object are now weakref.proxy and the garbage collector can clean everything easily.



## Straight to the root

I thought fixing the cyclic reference issue would fix all the memory eating but not. Something is still eating my memory ! And using the same tools as before I just figure out that I still have a tons of IQ hanging in memory. Using objgraph again helps a lot to find the real source:


```python
In : objgraph.show_backrefs(objgraph.by_type('Iq')[:3], filename="/vagrant/g.png",refcounts=True, max_depth=5)
Graph written to /tmp/objgraph-NZsvZL.dot (24 nodes)
Image generated as /vagrant/g.png
```

<p align="center">
  <img src="/data/xmpp-iqref_expected.png" title="Back Refences on Several IQ object" />
</p>


This is interesting… but wait we also have a bunch of Message hanging (Message is a type of IQ). Let’s see what’s behind and if it’s related to the previous one:

```python
In : objgraph.show_backrefs(objgraph.by_type('Message')[:5],max_depth=8, filename="/vagrant/g.png")
Graph written to /tmp/objgraph-Dhqhdu.dot (58 nodes)
Image generated as /vagrant/g.png
```

<p align="center">
  <img src="/data/xmpp-messagesClosure.png" title="Back Refences on Several Messages object" />
</p>

What a mess !

As we can see, the *Dispatcher* instance keep a list of every stanza dispatched (and actually most of them are related to callbacks).

For the record, a *closure* contains a *cell* which contains details of the *variables defined in the enclosing scope*.

So as we can read in this diagram, **xmpp.dispatcher.Dispatcher** instance, has an dict attribute named *_expected* and this attribute contain **every** IQ/Messages (identify with ID as key) sent and pending for a response.

I took a look to the xmpppy library and I found that in specific case of the *Dispatcher.Dispatch*, the `session._expected[ID]` was never cleaned. I smartly add a `del session._expected[ID]` just after all have been processed and tada, no more memory leak !

## Are we done ?

After stressing a bit the new code, I notice that some objects are still in memory:

```python
In : print 'Memory : %s (kb) with %s Node %s iq %s message %s dict' % (resource.getrusage(resource.RUSAGE_SELF).ru_maxrss,objgraph.count('Node'),objgraph.count('Iq'),objgraph.count('Message'),objgraph.count('dict') )
Memory : 115188 (kb) with 377 Node 0 iq 0 message 11235 dict
```

We can see that our previous fix works as we don't have any IQ/Message hanging.

But I still have Nodes, I know that we are using the simplexml object to deal with XML file (libvirt domain definition for example) so let’s see what these nodes are.

First I try to filter the parent nodes (as nodes can have leaf nodes and parent node with:

```python
In : topNodes=[]
In : for node in objgraph.by_type('Node'):
...:     if not node.parent:
...:         topNodes.append(node)
In : len(topNodes)
70
```

So we have 70 Nodes lefts (on 377) without parents, let’s sort them and count them by .name attribute

```python
In:  topNodesByName=sorted(topNodes, key=lambda node: node.name)
In : for item in topNodesByName:
...:     if str(item.name) not in NodeCount.keys():
...:             NodeCount[str(item.name)] = 1
...:     NodeCount[str(item.name)] += 1
In : NodeCount
{'domain': 5, 'features': 6, 'success': 6, 'iq': 11, 'capabilities': 2, 'item': 34, 'groups': 2, 'stream:stream': 11, 'users': 2}
```

We see that Node with *item* as **name** is almost 50% of the Nodes, let’s see how they are referenced:

Gather the item togethers

```python
In : items=[]
In : for item in topNodesByName:
...:    if item.name == 'item':
...:            items.append(item)
```

Let’s pick one and find it’s index:


```python
In: objgraph.by_type('Node').index(items[0])
93
In: del topNodes
In: del topNodeByName
In: del items
```

**Note:** before generating graph you must del every intermediate values (topNodes….) as they could appear in the graph also...

Now generate the chain of back references:

```python
In : objgraph.show_chain(
...:     objgraph.find_backref_chain(
...:         objgraph.by_type('Node')[93],
...:         objgraph.is_proper_module),
...:     filename='/vagrant/chain.png')

Graph written to /tmp/objgraph-cujzXy.dot (13 nodes)
Image generated as /vagrant/chain.png
```

<p align="center">
  <img src="/data/xmpp-chainpubsub.png" title="Chain of back Refences on an item" />
</p>


We can see that the *TNPubSubNode* instance is holding these Nodes. By having at the code we figure out that theses Nodes are the **PubSub Events** retrieved. Theses are stored to avoid fetch everything each time (and by the way the limit set is 1000 so it will never grow endlessly).

To be sure that we tracked all wanted too, I check with guppy to see what remains in memory:

```python
In : import guppy
In : hp = guppy.hpy()
In : h = hp.heap()
In : h
Partition of a set of 241435 objects. Total size = 34193088 bytes.
 Index  Count   %     Size   % Cumulative  % Kind (class / dict of class)
     0 119302  49 12690504  37  12690504  37 str
     1  53350  22  4633584  14  17324088  51 tuple
     2   4954   2  2573296   8  19897384  58 dict (no owner)
     3    588   0  1931808   6  21829192  64 dict of module
     4  14400   6  1728000   5  23557192  69 types.CodeType
     5  14109   6  1693080   5  25250272  74 function
     6   1318   1  1187584   3  26437856  77 type
     7   1318   1  1138192   3  27576048  81 dict of type
     8   2079   1   944200   3  28520248  83 unicode
     9   3902   2   717680   2  29237928  86 list
<832 more rows. Type e.g. '_.more' to view.>
```

All seems fine, we have a lot of str and dict but nothing relevant.

### Final word

It could sound easy to fix like this but theses issues gave me some hard time...
