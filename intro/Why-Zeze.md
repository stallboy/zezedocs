---
layout: home
title: intro
nav_order: 10
---

* TOC
{:toc}


# Why Zeze

## 数据修改中途失败怎么处理?

如果你对这个问题已经有经验，快速浏览一下下面的分析即可。
如果一个操作需要修改多项数据，但是修改的中途发生异常，那么已经完成修改应该怎么处
理？此时的情况非常复杂，有的时候是可以接受的，但多数时候数据已经处于不一致状态，
会导致系统出现问题。这种情况的处理方法最合理的方式是放弃所有的修改，恢复到初始状
态。就是一般数据库中事务的要嘛全部成功要嘛全部失败的定义。
但事务一般都是数据库支持的，程序的环境很可能没有事务的概念。在没有事务的情况下。
一般的解决办法是：修改数据前提前检查全部逻辑条件，最后一起修改。即使这样子，还要
祈祷修改过程不要出错。由于一般修改自己的简单变量或者修改已经很可靠的程序语言提供
的容器，通常情况下都不会出错，所以这个办法在简单系统中很有效。对于复杂系统，里面
有很多模块，模块之间需要互相调用。那么提前检查全部条件就会变得困难。当然也有办法：
每个模块提供的每个功能接口分成两部分，一部分是条件检查，一部分是修改。一个业务流
程在使用其他模块时，先集中调用所有依赖的条件检查，最后再集中调用所有修改。这种方
法看起来很麻烦，很显然已经破坏了业务流程的直观性：即因为集中条件检查这个原因，导
致实现代码和实际业务逻辑需求描述不一致。如果被调用的模块又要调用其他模块，会使得
情况变得更复杂，分成两个部分的办法还不能解决代码复用（模块依赖）的问题，还需要对
整体的代码分布进行考量，把代码分成条件检查和修改层，业务逻辑层（业务逻辑层不能相
互依赖）。
有些程序解决中途修改失败的问题的解决办法是先把修改保存在局部变量中最后一起提交，
或者先复制一个拷贝在中途失败时恢复。这种做法实际上很像数据库的事务了，只是都是专
用的，没有通用的容易使用的系统。Zeze就试图提供这样一个通用的支持事务系统：直接
在程序开发时，拥有事务的特性。
为程序开发直接提供事务是Zeze最初的目的。让实现代码和业务描述尽可能近，几乎能对
照起来。业务按什么顺序描述逻辑条件，不会因为“提前检查”这个做法破坏调用顺序。模块
划分的时候也很自然，不用分两个方法，分成两层，模块嵌套调用也很自然。不管中间什么
时候出错，事务保证了所有的修改回滚回初始状态。

## 一个游戏里面得到经验升级的代码例子(c#)。

```c#
	Void TryLevelUp(long NewExperience)
	{
	    Role.Experience += NewExperience; // 累加上新的经验
	    // 根据经验配置表，判断是否达到升级需要的经验。
	    // 增加一次经验，可能升多级。
	    While (Role.Experience >= NextLevelExperienceConfig[Role.Level])
	    {
	        Role.Experience -= NextLevelExperienceConfig[Level];
	        Role.Level += 1;
	        If (Role.Level % 10 == 0)
	        {
	            // 每10级学到一个新技能。
	            Role.Skills.Add(LearnSkillConfig[Level / 10]);
	        }
	    }
	}
```

如果TryLevelUp要求增加一次经验的所有升级是一个事务，而且第13行又可能抛出异常，
那么上面的代码在没有事务时就不好处理，有了事务就很直观。

