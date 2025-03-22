```
layout: post
title: Java排行榜
excerpt: 不使用其他中间件实现的线程安全排行榜
```

最近有一个竞技场的排行榜需求, 由于历史原因, 无法使用Redis等中间件实现. 

其需求为:

1. 可以通过玩家对象获取玩家当前排名

2. 可以获取玩家前后指定数量的玩家

3. 线程安全

4. 用户规模大概在几万人

根据以上需求, 适合的方案有:

1. 使用`TreeSet`, 其内部使用红黑树实现, 可以自动排序, 且可以获取玩家前后集合. 线程不安全, 可以通过加锁或者使用单线程服务来维护

2. 使用`ConcurrentSkipListSet`, 类似于zset, 其内部使用跳表实现, 可以自动排序, 且可以获取玩家前后集合. 线程安全.

两个方案均满足需求2, 3, 4, 但对于需求1, 两个方案均需要通过遍历的方式来查找玩家所在位置.

此时, 我们可以通过添加一个`Map<Long, Player>`来加速索引, 通过玩家ID就可以直接索引到排行榜中的玩家对象.

但是这样又带来了新的问题, 两个集合的维护并不是原子操作, 方案2需要引入同步机制来确保两个集合修改的一致性, 既然加锁, 那还是回到了方案1.

最终酌定采取`TreeSet` + `HashMap` + `ReadWritetLock` 的方式来实现.

以下是代码实现: 

```java
class Leaderboard {
    // Player对象不可变
    private TreeSet<Player> rankings;
    private HashMap<String, Player> playerMap;
    private ReadWriteLock lock;

    public Leaderboard() {
        rankings = new TreeSet<>();
        playerMap = new HashMap<>();
        lock = new ReentrantReadWriteLock();
    }

    // 更新积分 使用写锁
    public void updateScore(String playerId, int score) {
        lock.writeLock().lock();
        try {
            Player newPlayer = new Player(playerId, score);
            Player oldPlayer = playerMap.put(playerId, newPlayer);
            if (oldPlayer != null) {
                rankings.remove(oldPlayer);
            }
            rankings.add(newPlayer);
        } finally {
            lock.writeLock().unlock();
        }
    }

    // 获取topN 使用读锁
    public List<Player> getTopN(int n) {
        lock.readLock().lock();
        try {
            List<Player> topN = new ArrayList<>();
            Iterator<Player> iterator = rankings.iterator();
            int count = 0;
            while (iterator.hasNext() && count < n) {
                topN.add(iterator.next());
                count++;
            }
            return topN;
        } finally {
            lock.readLock().unlock();
        }
    }

    // 读操作：使用读锁
    public int getRank(String playerId) {
        lock.readLock().lock();
        try {
            Player player = playerMap.get(playerId);
            if (player == null) return -1;
            int rank = rankings.headSet(player, true).size();
            return rank;
        } finally {
            lock.readLock().unlock();
        }
    }

    // 获取前N名
    public List<Player> getPreviousN(String playerId, int n) {
        lock.readLock().lock();
        try {
            Player target = playerMap.get(playerId);
            if (target == null || n <= 0) return new ArrayList<>();
            SortedSet<Player> headSet = rankings.headSet(target, false);
            List<Player> previousN = new ArrayList<>(Math.min(n, headSet.size()));
            Iterator<Player> iterator = headSet.descendingIterator();
            int count = 0;
            while (iterator.hasNext() && count < n) {
                previousN.add(0, iterator.next());
                count++;
            }
            return previousN;
        } finally {
            lock.readLock().unlock();
        }
    }

    // 获取后N名
    public List<Player> getNextN(String playerId, int n) {
        lock.readLock().lock();
        try {
            Player target = playerMap.get(playerId);
            if (target == null || n <= 0) return new ArrayList<>();
            SortedSet<Player> tailSet = rankings.tailSet(target, false);
            List<Player> nextN = new ArrayList<>(Math.min(n, tailSet.size()));
            Iterator<Player> iterator = tailSet.iterator();
            int count = 0;
            while (iterator.hasNext() && count < n) {
                nextN.add(iterator.next());
                count++;
            }
            return nextN;
        } finally {
            lock.readLock().unlock();
        }
    }
}

```
