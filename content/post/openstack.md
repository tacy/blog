---
title: "openstack notes"
date: 2013-08-28
lastmod: 2013-08-28
draft: false
tags: ["tech", "openstack", "cloud", "notes"]
categories: ["tech"]
description: "Openstack的一些基本信息，自己的了解，认为好的一些资源收集，记录下来。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Swift
## Ring builder
  读了一下ringbuilder代码，简单记录一下，免的忘了
### Add device
   添加device的时候可以指定每个device的meta信息和replication ip，Swift会给每
   个device一个id，这个id会一直递增，也就是说新增加的device的id总是比device的id
   大，不重用id
   #+BEGIN_SRC python
   if 'id' not in dev:
       dev['id'] = 0
       if self.devs:
           dev['id'] = max(d['id'] for d in self.devs if d) + 1
   if dev['id'] < len(self.devs) and self.devs[dev['id']] is not None:
       raise exceptions.DuplicateDeviceError(
           'Duplicate device id: %d' % dev['id'])
   # Add holes to self.devs to ensure self.devs[dev['id']] will be the dev
   while dev['id'] >= len(self.devs):
       self.devs.append(None)
   #+END_SRC

   weight和Part之间的关系计算很简单
   #+BEGIN_SRC python
   return self.parts * self.replicas / \
       sum(d['weight'] for d in self._iter_devs())
   #+END_SRC

   在添加device的时候，会重新计算device所期望的Part（原有device需要减去已分配）
   ，这个值很关键，后面part和device的映射会根据这个值确定分配优先级（
   parts_wanted大优先级高）
   #+BEGIN_SRC python
   for dev in self._iter_devs():
       if not dev['weight']:
           # With no weight, that means we wish to "drain" the device. So
           # we set the parts_wanted to a really large negative number to
           # indicate its strong desire to give up everything it has.
           dev['parts_wanted'] = -self.parts * self.replicas
       else:
           dev['parts_wanted'] = \
               int(weight_of_one_part * dev['weight']) - dev['parts']
   #+END_SRC

   添加device并不会导致数据迁移，必须显示调用rebalance才会执行平衡操作。

### Rebalance
   relalance入口
   #+BEGIN_SRC python
   def rebalance(self, seed=None):
       """
       Rebalance the ring.

       This is the main work function of the builder, as it will assign and
       reassign partitions to devices in the ring based on weights, distinct
       zones, recent reassignments, etc.

       The process doesn't always perfectly assign partitions (that'd take a
       lot more analysis and therefore a lot more time -- I had code that did
       that before). Because of this, it keeps rebalancing until the device
       skew (number of partitions a device wants compared to what it has) gets
       below 1% or doesn't change by more than 1% (only happens with ring that
       can't be balanced no matter what -- like with 3 zones of differing
       weights with replicas set to 3).

       :returns: (number_of_partitions_altered, resulting_balance)
       """

       if seed:
           random.seed(seed)
       #如果为空，表示第一次做balance,这里的操作和重做balance差不多
       self._ring = None
       if self._last_part_moves_epoch is None:
           self._initial_balance()
           self.devs_changed = False
           return self.parts, self.get_balance()
       retval = 0
       self._update_last_part_moves()
       last_balance = 0
       #收集哪些part需要分配dev
       new_parts, removed_part_count = self._adjust_replica2part2dev_size()
       retval += removed_part_count
       #执行分配操作
       self._reassign_parts(new_parts)
       retval += len(new_parts)
       while True:
           #统计移除的device持有的part,和超载的device，执行分配操作,重复执行
           #balance，直到平衡
           reassign_parts = self._gather_reassign_parts()
           self._reassign_parts(reassign_parts)
           retval += len(reassign_parts)
           while self._remove_devs:
               self.devs[self._remove_devs.pop()['id']] = None
           balance = self.get_balance()
           if balance < 1 or abs(last_balance - balance) < 1 or \
                   retval == self.parts:
               break
           last_balance = balance
       self.devs_changed = False
       self.version += 1
       return retval, balance
   #+END_SRC

   _reassign_parts()完成核心的part到dev的映射
   #+BEGIN_SRC python
   def _reassign_parts(self, reassign_parts):
       """
       For an existing ring data set, partitions are reassigned similarly to
       the initial assignment. The devices are ordered by how many partitions
       they still want and kept in that order throughout the process. The
       gathered partitions are iterated through, assigning them to devices
       according to the "most wanted" while keeping the replicas as "far
       apart" as possible. Two different regions are considered the
       farthest-apart things, followed by zones, then different ip/port pairs
       within a zone; the least-far-apart things are different devices with
       the same ip/port pair in the same zone.

       If you want more replicas than devices, you won't get all your
       replicas.

       :param reassign_parts: An iterable of (part, replicas_to_replace)
                              pairs. replicas_to_replace is an iterable of the
                              replica (an int) to replace for that partition.
                              replicas_to_replace may be shared for multiple
                              partitions, so be sure you do not modify it.
       """
       for dev in self._iter_devs():
           #生成按照parts_wanted排序的key
           dev['sort_key'] = self._sort_key_for(dev)

       available_devs = \
           sorted((d for d in self._iter_devs() if d['weight']),
                  key=lambda x: x['sort_key'])

       tier2devs = defaultdict(list)
       tier2sort_key = defaultdict(list)
       max_tier_depth = 0
       for dev in available_devs:
           for tier in tiers_for_dev(dev):
               #生成tier和dev的关系结构，包括
               #{(region)[dev]}，{(region,zone):[dev]}
               #{(region,zone,machine):[dev]}
               #{(region,zone,machine,id):[dev]}
               tier2devs[tier].append(dev)  # <-- starts out sorted!
               tier2sort_key[tier].append(dev['sort_key'])
               if len(tier) > max_tier_depth:
                   max_tier_depth = len(tier)

       #生成tiger和children关系，{():[region]}
       #{(region):[(region,zone)]}
       #{(region,zone):[(region,zone,machine)]
       #{(region,zone,machine):[region,zone,machine,devid]}
       tier2children_sets = build_tier_tree(available_devs)
       tier2children = defaultdict(list)
       tier2children_sort_key = {}
       tiers_list = [()]
       depth = 1
       while depth <= max_tier_depth:
           new_tiers_list = []
           for tier in tiers_list:
               child_tiers = list(tier2children_sets[tier])
               child_tiers.sort(key=lambda t: tier2sort_key[t][-1])
               tier2children[tier] = child_tiers
               tier2children_sort_key[tier] = map(
                   lambda t: tier2sort_key[t][-1], child_tiers)
               new_tiers_list.extend(child_tiers)
           tiers_list = new_tiers_list
           depth += 1

       #遍历tier2children关系，找最需要分配的dev
       for part, replace_replicas in reassign_parts:
           # Gather up what other tiers (regions, zones, ip/ports, and
           # devices) the replicas not-to-be-moved are in for this part.
           other_replicas = defaultdict(int)
           unique_tiers_by_tier_len = defaultdict(set)
           for replica in self._replicas_for_part(part):
               if replica not in replace_replicas:
                   dev = self.devs[self._replica2part2dev[replica][part]]
                   for tier in tiers_for_dev(dev):
                       other_replicas[tier] += 1
                       unique_tiers_by_tier_len[len(tier)].add(tier)

           for replica in replace_replicas:
               tier = ()
               depth = 1
               while depth <= max_tier_depth:
                   # Order the tiers by how many replicas of this
                   # partition they already have. Then, of the ones
                   # with the smallest number of replicas, pick the
                   # tier with the hungriest drive and then continue
                   # searching in that subtree.
                   #
                   # There are other strategies we could use here,
                   # such as hungriest-tier (i.e. biggest
                   # sum-of-parts-wanted) or picking one at random.
                   # However, hungriest-drive is what was used here
                   # before, and it worked pretty well in practice.
                   #
                   # Note that this allocator will balance things as
                   # evenly as possible at each level of the device
                   # layout. If your layout is extremely unbalanced,
                   # this may produce poor results.
                   #
                   # This used to be a cute, recursive function, but it's been
                   # unrolled for performance.
                   candidate_tiers = tier2children[tier]
                   candidates_with_replicas = \
                       unique_tiers_by_tier_len[len(tier) + 1]
                   if len(candidate_tiers) > len(candidates_with_replicas):
                       # There exists at least one tier with 0 other replicas,
                       # so work backward among the candidates, accepting the
                       # first which isn't in other_replicas.
                       #
                       # This optimization is to avoid calling the min()
                       # below, which is expensive if you've got thousands of
                       # drives.
                       for t in reversed(candidate_tiers):
                           if other_replicas[t] == 0:
                               tier = t
                               break
                   else:
                       min_count = min(other_replicas[t]
                                       for t in candidate_tiers)
                       tier = (t for t in reversed(candidate_tiers)
                               if other_replicas[t] == min_count).next()
                   depth += 1
               dev = tier2devs[tier][-1]
               dev['parts_wanted'] -= 1
               dev['parts'] += 1
               old_sort_key = dev['sort_key']
               #重新根据新的parts_wanted计算sort key
               new_sort_key = dev['sort_key'] = self._sort_key_for(dev)
               for tier in tiers_for_dev(dev):
                   other_replicas[tier] += 1
                   unique_tiers_by_tier_len[len(tier)].add(tier)

                   index = bisect.bisect_left(tier2sort_key[tier],
                                              old_sort_key)
                   #弹出该dev
                   tier2devs[tier].pop(index)
                   tier2sort_key[tier].pop(index)

                   new_index = bisect.bisect_left(tier2sort_key[tier],
                                                  new_sort_key)
                   #用新的sort key插入到tier2devs中
                   tier2devs[tier].insert(new_index, dev)
                   tier2sort_key[tier].insert(new_index, new_sort_key)

                   #同样对tier2childre进行操作
                   # Now jiggle tier2children values to keep them sorted
                   new_last_sort_key = tier2sort_key[tier][-1]
                   parent_tier = tier[0:-1]
                   index = bisect.bisect_left(
                       tier2children_sort_key[parent_tier],
                       old_sort_key)
                   popped = tier2children[parent_tier].pop(index)
                   tier2children_sort_key[parent_tier].pop(index)

                   new_index = bisect.bisect_left(
                       tier2children_sort_key[parent_tier],
                       new_last_sort_key)
                   tier2children[parent_tier].insert(new_index, popped)
                   tier2children_sort_key[parent_tier].insert(
                       new_index, new_last_sort_key)

               #建立part和dev映射
               self._replica2part2dev[replica][part] = dev['id']

       # Just to save memory and keep from accidental reuse.
       for dev in self._iter_devs():
           del dev['sort_key']
   #+END_SRC


# 经典文章
  - [[http://greg.brim.net/page/building_a_consistent_hashing_ring.html][Building a Consistent Hashing Ring]]


# Nova
## Develop environment
1. install package
   sudo apt-get install python-dev libssl-dev python-pip git-core libxml2-dev
   libxslt-dev libmysqld-dev
2. get source code
   git clone https://github.com/openstack/nova.git nova-dev
3. configure virtualenv[fn:1]
   easy_install Virtualenv
   cd nova-dev
   python tools/install_env.py
   source .venv/bin/activate
4. run unit test
   export PYTHONPATH=$PYTHONPATH:/home/tacy/workspace/nova-dev
   ./run_tests.sh


# Footnotes
[fn:1] [[http://pypi.python.org/pypi/virtualenv][using virtualenv to create isolated python environments]]
