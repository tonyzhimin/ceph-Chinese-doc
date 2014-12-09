============
 归置组排障
============

归置组总不整洁
==============

When you create a cluster and your cluster remains in ``active``, 
``active+remapped`` or ``active+degraded`` status and never achieve an 
``active+clean`` status, you likely have a problem with your configuration.

You may need to review settings in the `存储池、归置组和 CRUSH 配置参考`_
and make appropriate adjustments.

As a general rule, you should run your cluster with more than one OSD and a
pool size greater than 1 object replica.

One Node Cluster
----------------

Ceph no longer provides documentation for operating on a single node, because
you would never deploy a system designed for distributed computing on a single
node. Additionally, mounting client kernel modules on a single node containing a
Ceph  daemon may cause a deadlock due to issues with the Linux kernel itself
(unless you use VMs for the clients). You can experiment with Ceph in a 1-node
configuration, in spite of the limitations as described herein.

If you are trying to create a cluster on a single node, you must change the
default of the ``osd crush chooseleaf type`` setting from ``1`` (meaning 
``host`` or ``node``) to ``0`` (meaning ``osd``) in your Ceph configuration
file before you create your monitors and OSDs. This tells Ceph that an OSD
can peer with another OSD on the same host. If you are trying to set up a
1-node cluster and ``osd crush chooseleaf type`` is greater than ``0``, 
Ceph will try to peer the PGs of one OSD with the PGs of another OSD on 
another node, chassis, rack, row, or even datacenter depending on the setting.

.. tip:: DO NOT mount kernel clients directly on the same node as your 
   Ceph Storage Cluster, because kernel conflicts can arise. However, you 
   can mount kernel clients within virtual machines (VMs) on a single node.

If you are creating OSDs using a single disk, you must create directories
for the data manually first. For example:: 

	mkdir /var/local/osd0 /var/local/osd1
	ceph-deploy osd prepare {localhost-name}:/var/local/osd0 {localhost-name}:/var/local/osd1
	ceph-deploy osd activate {localhost-name}:/var/local/osd0 {localhost-name}:/var/local/osd1


Fewer OSDs than Replicas
------------------------

If you've brought up two OSDs to an ``up`` and ``in`` state, but you still 
don't see ``active + clean`` placement groups, you may have an 
``osd pool default size`` set to greater than ``2``.

There are a few ways to address this situation. If you want to operate your
cluster in an ``active + degraded`` state with two replicas, you can set the 
``osd pool default min size`` to ``2`` so that you can write objects in 
an ``active + degraded`` state. You may also set the ``osd pool default size``
setting to ``2`` so that you only have two stored replicas (the original and 
one replica), in which case the cluster should achieve an ``active + clean`` 
state.

.. note:: You can make the changes at runtime. If you make the changes in 
   your Ceph configuration file, you may need to restart your cluster.


Pool Size = 1
-------------

If you have the ``osd pool default size`` set to ``1``, you will only have 
one copy of the object. OSDs rely on other OSDs to tell them which objects 
they should have. If a first OSD has a copy of an object and there is no
second copy, then no second OSD can tell the first OSD that it should have
that copy. For each placement group mapped to the first OSD (see 
``ceph pg dump``), you can force the first OSD to notice the placement groups
it needs by running::
   
   	ceph pg force_create_pg <pgid>
   

CRUSH 图错误
------------

Another candidate for placement groups remaining unclean involves errors 
in your CRUSH map.


卡住的归置组
============

有失败时归置组会进入“degraded”（降级）或“peering”（连接建立中）状态，这事时有发\
生，通常这些状态意味着正常的失败恢复正在进行。然而，如果一个归置组长时间处于某个这些\
状态就意味着有更大的问题，因此监视器在归置组卡 （stuck） 在非最优状态时会警告。我们\
具体检查：

* ``inactive`` （不活跃）——归置组长时间无活跃（即它不能提供读写服务了）；
  
* ``unclean`` （不干净）——归置组长时间不干净（例如它未能从前面的失败完全恢复）；

* ``stale`` （不新鲜）——归置组状态没有被 ``ceph-osd`` 更新，表明存储这个归置组的所\
  有节点可能都挂了。

你可以摆出卡住的归置组：
::

	ceph pg dump_stuck stale
	ceph pg dump_stuck inactive
	ceph pg dump_stuck unclean

卡在 ``stale`` 状态的归置组通过修复 ``ceph-osd`` 进程通常可以修复；卡在 \
``inactive`` 状态的归置组通常是互联问题（参见 :ref:`failures-osd-peering` ）；卡\
在 ``unclean`` 状态的归置组通常是由于某些原因阻止了恢复的完成，像未找到的对象（参\
见 :ref:`failures-osd-unfound` ）。


.. _failures-osd-peering:

归置组挂了——互联失败
========================

在某些情况下， ``ceph-osd`` 连接建立进程会遇到问题，使 PG 不能活跃、可用，例如 \
``ceph health`` 也许显示：
::

	ceph health detail
	HEALTH_ERR 7 pgs degraded; 12 pgs down; 12 pgs peering; 1 pgs recovering; 6 pgs stuck unclean; 114/3300 degraded (3.455%); 1/3 in osds are down
	...
	pg 0.5 is down+peering
	pg 1.4 is down+peering
	...
	osd.1 is down since epoch 69, last address 192.168.106.220:6801/8651

可以查询到 PG 为何被标记为 ``down`` ：
::

	ceph pg 0.5 query

.. code-block:: javascript

 { "state": "down+peering",
   ...
   "recovery_state": [
        { "name": "Started\/Primary\/Peering\/GetInfo",
          "enter_time": "2012-03-06 14:40:16.169679",
          "requested_info_from": []},
        { "name": "Started\/Primary\/Peering",
          "enter_time": "2012-03-06 14:40:16.169659",
          "probing_osds": [
                0,
                1],
          "blocked": "peering is blocked due to down osds",
          "down_osds_we_would_probe": [
                1],
          "peering_blocked_by": [
                { "osd": 1,
                  "current_lost_at": 0,
                  "comment": "starting or marking this osd lost may let us proceed"}]},
        { "name": "Started",
          "enter_time": "2012-03-06 14:40:16.169513"}
    ]
 }

``recovery_state`` 段告诉我们连接建立因 ``ceph-osd`` 进程挂了而被阻塞，本例是 \
``osd.1`` 挂了，启动这个进程应该就可以恢复。

另外，如果 ``osd.1`` 是灾难性的失败（如硬盘损坏），我们可以告诉集群它丢失（ \
``lost`` ）了，让集群尽力完成副本拷贝。

.. important:: 集群不能保证其它数据副本是一致且最新就危险了！

让 Ceph 无论如何都继续：
::

	ceph osd lost 1

恢复将继续。


.. _failures-osd-unfound:

未找到的对象
============

某几种失败相组合可能导致 Ceph 抱怨有找不到（ ``unfound`` ）的对象：
::

	ceph health detail
	HEALTH_WARN 1 pgs degraded; 78/3778 unfound (2.065%)
	pg 2.4 is active+degraded, 78 unfound

这意味着存储集群知道一些对象（或者存在对象的较新副本）存在，却没有找到它们的副本。下\
例展示了这种情况是如何发生的，一个 PG 的数据存储在 ceph-osd 1 和 2 上：

* 1 挂了；
* 2 独自处理一些写动作；
* 1 起来了；
* 1 和 2 重新互联， 1 上面丢失的对象加入队列准备恢复；
* 新对象还未拷贝完， 2 挂了。

这时， 1 知道这些对象存在，但是活着的 ``ceph-osd`` 都没有副本，这种情况下，读写这些\
对象的 IO 就会被阻塞，集群只能指望节点早点恢复。这时我们假设用户希望先得到一个 IO \
错误。

首先，你应该确认哪些对象找不到了：
::

	ceph pg 2.4 list_missing [starting offset, in json]

.. code-block:: javascript

 { "offset": { "oid": "",
      "key": "",
      "snapid": 0,
      "hash": 0,
      "max": 0},
  "num_missing": 0,
  "num_unfound": 0,
  "objects": [
     { "oid": "object 1",
       "key": "",
       "hash": 0,
       "max": 0 },
     ...
  ],
  "more": 0}

如果在一次查询里列出的对象太多， ``more`` 这个域将为 ``true`` ，因此你可以查询更\
多。（命令行工具可能隐藏了，但这里没有）

其次，你可以找出哪些 OSD 上探测到、或可能包含数据：
::

	ceph pg 2.4 query

.. code-block:: javascript

   "recovery_state": [
        { "name": "Started\/Primary\/Active",
          "enter_time": "2012-03-06 15:15:46.713212",
          "might_have_unfound": [
                { "osd": 1,
                  "status": "osd is down"}]},

本例中，集群知道 ``osd.1`` 可能有数据，但它挂了（ ``down`` ）。所有可能的状态有：

 * 已经探测到了
 * 在查询
 * OSD 挂了
 * 尚未查询

有时候集群要花一些时间来查询可能的位置。

还有一种可能性，对象存在于其它位置却未被列出，例如，集群里的一个 ``ceph-osd`` 停止\
且被剔出，然后完全恢复了；后来的失败、恢复后仍有未找到的对象，它也不会觉得早已死亡\
的 ``ceph-osd`` 上仍可能包含这些对象。（这种情况几乎不太可能发生）。

如果所有位置都查询过了仍有对象丢失，那就得放弃丢失的对象了。这仍可能是罕见的失败组合\
导致的，集群在写入完成前，未能得知写入是否已执行。以下命令把未找到的（ unfound ）对\
象标记为丢失（ lost ）。
::

	ceph pg 2.5 mark_unfound_lost revert|delete

上述最后一个参数告诉集群应如何处理丢失的对象。

delete 选项将导致完全删除它们。

revert 选项（纠删码存储池不可用）会回滚到前一个版本或者（如果它是新对象的话）删除\
它。要慎用，它可能迷惑那些期望对象存在的应用程序。


无根归置组
==========

拥有归置组拷贝的 OSD 都可以失败，在这种情况下，那一部分的对象存储不可用，监视器就不\
会收到那些归置组的状态更新了。为检测这种情况，监视器把任何主 OSD 失败的归置组标记\
为 ``stale`` （不新鲜），例如：
::

	ceph health
	HEALTH_WARN 24 pgs stale; 3/300 in osds are down

你能找出哪些归置组 ``stale`` 、和存储这些归置组的最新 OSD ，命令如下：
::

	ceph health detail
	HEALTH_WARN 24 pgs stale; 3/300 in osds are down
	...
	pg 2.5 is stuck stale+active+remapped, last acting [2,0]
	...
	osd.10 is down since epoch 23, last address 192.168.106.220:6800/11080
	osd.11 is down since epoch 13, last address 192.168.106.220:6803/11539
	osd.12 is down since epoch 24, last address 192.168.106.220:6806/11861

如果想使归置组 2.5 重新在线，例如，上面的输出告诉我们它最后由 ``osd.0`` 和 \
``osd.2`` 处理，重启这些 ``ceph-osd`` 将恢复之（还有其它的很多 PG ）。


只有几个 OSD 接收数据
=====================

如果你的集群有很多节点，但只有其中几个接收数据，\ `检查`_\ 下存储池里的归置组数量。\
因为归置组是映射到多个 OSD 的，这样少量的归置组将不能分布于整个集群。试着创建个新存\
储池，其归置组数量是 OSD 数量的若干倍。详情见\ `归置组`_\ ，存储池的默认归置组数量\
没多大用，你可以参考\ `这里`_\ 更改它。


不能写入数据
============

如果你的集群已启动，但一些 OSD 没起来，导致不能写入数据，确认下运行的 OSD 数量满足\
归置组要求的最低 OSD 数。如果不能满足， Ceph 就不会允许你写入数据，因为 Ceph 不能保\
证复制能如愿进行。详情参见\ `存储池、归置组和 CRUSH 配置参考`_\ 里的 \
``osd pool default min size`` 。


归置组不一致
============

If you receive an ``active + clean + inconsistent`` state, this may happen
due to an error during scrubbing. If the inconsistency is due to disk errors,
check your disks.

You can repair the inconsistent placement group by executing:: 

	ceph pg repair {placement-group-ID}

If you receive ``active + clean + inconsistent`` states periodically due to 
clock skew, you may consider configuring your `NTP`_ daemons on your 
monitor hosts to act as peers. See `网络时间协议`_ and Ceph 
`时钟选项`_ for additional details.



.. _检查: ../../operations/placement-groups#get-the-number-of-placement-groups
.. _这里: ../../configuration/pool-pg-config-ref
.. _归置组: ../../operations/placement-groups
.. _存储池、归置组和 CRUSH 配置参考: ../../configuration/pool-pg-config-ref
.. _NTP: http://en.wikipedia.org/wiki/Network_Time_Protocol
.. _网络时间协议: http://www.ntp.org/
.. _时钟选项: ../../configuration/mon-config-ref/#clock