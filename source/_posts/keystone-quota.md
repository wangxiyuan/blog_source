---
title: CERN的多层级quota模型(译)
date: 2017-10-24 11:13:47
tags:
  - OpenStack
  - Keystone
categories: 翻译
---
> 原文链接：http://openstack-in-production.blogspot.my/2017/07/nested-quota-models.html, 译文删掉了原文中一些没有内容的话。注：CERN（欧洲粒子物理研究所）

波士顿峰期间，有许多关于多层级quota模型的讨论(https://etherpad.openstack.org/p/BOS-forum-quotas )。在之前的blog中(http://openstack-in-production.blogspot.fr/2016/04/resource-management-at-cern.html )，已经讨论了一些使用背景，但是我们还有更多的工作要做。
<!-- more -->
在社区同意把quota limit放到Keystone之后（http://specs.openstack.org/openstack/keystone-specs/specs/keystone/ongoing/unified-limits.html ）,现在的问题在于多层级quota模型的设计上。这是个很复杂的问题。针对不同的场景，可能需要不同的模型。，现有的关于quota模型的各种设计，我们很难说哪个是对的，哪个是错的。因此我们需要把quota模型设计成插件式，使之可替换，以应对各种场景和需求。

现在的问题是，默认的quota模型（参考实现）应该是什么样的。以下是CERN中考虑的一些场景和结论。

- 在CERN中，一个子租户的初始limit的值是0. 如果不是0，意味着子租户的quota可能会超过父租户，因此不被允许。
- 可以在租户拓扑中任意一个租户内创建资源。如果用户想把一个租户拆分成两个子租户，那么他需要在这个租户之下创建两个新的子租户，然后迁移或重建VM。
- 在参考实现中，所有子租户的limit之和不能超过他们的父租户。（如果允许的话，维护逻辑将会变得相当复杂）
- 父租户的管理员可以修改子租户的limit。当然可以通过policy.json来控制这种权限。
- 允许管理员把limit的值设置的比usage小。这样就避免了Keystone还要去调用其他服务的问题（计算资源的usage）。
- 每个租户最多只能有一个父租户。
- 不考虑用户级的quota和limit
- 不论怎么设计API和quota模型，我们都不建议Keystone去调用其他的服务，也不建议服务之间有过多的交互。

CERN的quota模型与spec（https://review.openstack.org/#/c/441203 ）中的Strict Hierarchy Default Closed模型类似。

在进步讲述之前，我先做一下定义：
- Limit是指一个租户中最大的可创的资源数。作为租户的属性存储在Keystone中。
- Used是指租户实际已使用的资源数。
- Hierarchical Usage是指该租户以及该租户的所有子租户已使用资源数的和。

下面举例说明，假设ATLAS租户的limit（L）是100，但其中没有used(U)的资源，并且他有一些子租户：Physics和Operations。他们的used数被算在ATLAS的Hierarchical Usage(HU)，并且他们各自也有两个子租户。

![Quota Example](/images/keystone-quota.png)

该图遵循如下规则：
- quota管理员不能增大子租户的limit以至所有子租户limit之和大于父租户。
	- ATLAS管理员不能增大他的子租户的limit。
	- Physics可以增加10个Higgs或Simulation的limit
	-  Operations管理员不能增加Workflow或Web的limit
- quota管理员不能把父租户的limit设置成小于子租户limit之和的值。
	- ATLAS管理员不能把Operations的limit设置成50，因为这样的话Limit(Workflow) + Limit(Web) > Limit(Operations)
- quota管理员可以把limit设置的比usage小。
	- Physics管理员可以把Simulation的limit改成5，即使会小于used的10。
- 创建一个新资源后，要求该租户树中的所有租户的的used小于等于limit。
	- Simulation中无法再创建资源，因为Used(Simulation) >= Limit(Simulation)。
	- 允许Web中再创建25个新资源，因为Used(Web)+25 <= Limit(Web)、HierarchicalUsage(Operations) + 25 <= Limit(Operations)并且HierarchicalUsage(ATLAS) + 25 <= Limit(ATLAS)。
		- Used(Web) = 30
		- HierarchicalUsage(Operations) = 60
		- HierarchicalUsage(ATLAS) = 90

根据以往Nova中quota的经验，我们的目标是要计算出所有的使用量(包括在资源创建时Used和HierarchicalUsage的动态量)。计算HierarchicalUsage的工作可能会很复杂，因为它贯穿了整个租户拓扑。各个服务还要加上调用Keystone的开销，因此性能可能会下降。


