title: Upsert
date: 2015-01-26 21:40:03
categories: Database
tags:
---
**Upsert**是个合成词，指的是：在数据库表中首先Update一条可能记录，如果目标记录并不存在，则Insert一条新的记录，也就是所谓的merge操作。  
听起来这是一个数据库操作中很常见的场景，写起来很简单，不是吗？最直接的做法：我先搜一下记录存在与否，如果存在，就Update；否则就Insert，不是很直观吗？是的，就是这么简单－－如果您能保证您的Upsert操作对象永远不会出现**并发**更新的情况的话。而如果您使用的数据库本身就支持merge操作的话，这个问题甚至可以更简单：您只需一条SQL就能实现上面三条SQL的功能。  
比如MySQL的内建**INSERT...ON DUPLICATE KEY UPDATE**语法，就提供了这样的便利——如果insert的数据会引起唯一索引（包括主键索引）的冲突，即这个唯一值重复了，则不会执行insert操作，而执行后面的update操作。  
```
INSERT INTO test (a,b) values (1,2) ON DUPLICATE KEY UPDATE b = b + 1;
```  
在这个例子里，假设表中只有a=1,b=1这一条记录，当试图插入a=1,b=2时，因为a=1记录已经存在，则执行update后面的b＝b+1,记录被更新为a=1,b=2(这个2是被b=b+1更新的，而不是插入的)。  
```
INSERT INTO test (a,b) values (2,2) ON DUPLICATE KEY UPDATE b = b + 1;
```
此条语句因为a=2的记录不存在，则执行Insert的操作。  
但是，不是所有的数据库都是MySQL这样。另一个应用广泛的数据库PostgreSQL，本身就不提供**ON DUPLICATE KEY UPDATE**的语法支持。有意思的是，在PostgreSQL的官方文档中提供的[例子](http://www.postgresql.org/docs/current/static/plpgsql-control-structures.html#PLPGSQL-UPSERT-EXAMPLE)中，这个实现是依赖**错误处理**来实现的。代码如下：  
```  
CREATE TABLE db (a INT PRIMARY KEY, b TEXT);

CREATE FUNCTION merge_db(key INT, data TEXT) RETURNS VOID AS
$$
BEGIN
    LOOP
        -- first try to update the key
        UPDATE db SET b = data WHERE a = key;
        IF found THEN
            RETURN;
        END IF;
        -- not there, so try to insert the key
        -- if someone else inserts the same key concurrently,
        -- we could get a unique-key failure
        BEGIN
            INSERT INTO db(a,b) VALUES (key, data);
            RETURN;
        EXCEPTION WHEN unique_violation THEN
            -- Do nothing, and loop to try the UPDATE again.
        END;
    END LOOP;
END;
$$
LANGUAGE plpgsql;

SELECT merge_db(1, 'david');
SELECT merge_db(1, 'dennis');  
```  
这段代码在一个LOOP中先做update，如果成功就返回；否则继续尝试Insert，在这个操作时要捕获唯一性冲突的异常，如果出现此异常，说明在update时拥有该键值的记录不存在，但是尝试Insert操作的时候却存在了，所以会有冲突的错误。就如注释指出的，这种情况是**并发**时常见的问题，所以要用LOOP保证此时还有机会返回重新尝试Update。  
软件设计中如何实现一个功能，是脱离不开功能具体的应用场景和上下文的。之所以在PostgreSQL中用这样一个有趣的方式来实现Upsert，是和数据库应用的常用场景分不开的。而在数据库应用中，**并发**和**效率**可以说是功能性和非功能性需求最具代表性的两个方面。  
[这篇文章](http://www.depesz.com/2012/06/10/why-is-upsert-so-complicated/)从思考的循序渐进的方式结合实验很好地展示了为什么要有这样的设计，简单地串起来，从直观但是问题很大的方案，一步步优化，到最终的最优方案，是经历了这样的一个筛选和比较的过程：  
1. 简单到两条SQL顺序调用，先update，无效后执行Insert。如前所述，并发时在Insert时只要两个调用不是同时进行，后来到必然失败，这种对竞态条件不做任何处理对方式，对并发的支持可以说是0；  
2. 进一步，初学者会想到那就把两个操作捆绑起来，放在一个事务里执行不就好了？但是等等，仔细想一下，事务保证的只是事务内所有的操作作为一个整体表现为最终的结果，都成功或者都rollback回去；事务本身并不保证事务之间的操作是资源独占的，所以竞态条件依然存在。  
3. 默认的事务是**Default Isolation Level**,可是事务不是还有一个**Serializable Level**么？从这个level本身的描述看，似乎可以让serializable transaction按照一定顺序来执行。但是根据引文的作者的实际实验，这条路行不通。表现为在并发时，虽然不会出现唯一性破坏的错误，但是在事务Commit的时候，大量的读写依赖关系会导致数据库的access无法真正穿行化（这是直接来自于报错信息，深层的原因有待深入研究）。  
4. 那好吧，要保证资源的独占访问，我给数据库表加锁总可以了吧（因为要update的记录可能不存在，所以不能直接给记录加锁）。当然可以！功能没有任何问题，在持有操作的表的锁的情况下，上面的两部分操作不会引起任何冲突。但是，效率啊，可以想像地惨不忍睹。  
5. 放弃表加锁，使用**advisory locks**呢？即所谓的劝告性锁，不在任何真实的实体上加锁，而是使用一个生成的数字，这个数字和操作的对象没有一毛钱的关系，操作完成之后加于其上的锁释放。这样实现效率可以提高很多。问题在于：**如果别人不通过我们的程序操作相同的资源，怎么让他们也保证使用相同的advisory locks？**没人能保证，我们总不能让别人都不能从console简单地执行一条Insert或者Update的SQL语句吧？  
6. 好吧，看来官方文档里的例子是最佳方案了。可是等等，为什么在PL/pgSQL一定要使用LOOP？我在Insert失败抛出错误之后在异常处理里直接再做一次Update不就好了么？有道理，但是不全面，仍然忽略了一种可能性，尽管这种可能性出现的概率不高——在我们Insert之前Insert的数据，导致了我们Insert失败，但是当我们尝试在异常处理中Update之前，这条记录被Delete了...那么我们再次Update必然就又失败了，如果此处没有LOOP，显然我们就失去了再次尝试并可能成功的机会。  
所以，我们看到的官方文档中的例子，是目前考虑了所有情况，兼顾并发和效率的最优的通用解决方案。  
这是一个很好的例子，不仅仅是最终的方案，重要的是整个思维过程，关于问题是怎么得到最优解的，很有借鉴的意义。