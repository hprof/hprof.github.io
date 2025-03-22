```
layout: post
title: 行为树目标查询器
excerpt: 通过仿造ElasticSearch DSL来实现一套用于为行为树节点筛选目标的查询器
```

### 查询过程

目标查询步骤分为四大步:

将目标放到筛选池(目标来源) -> 按照过滤条件过滤筛选池里的目标(目标过滤) -> 按一定规则排序(目标排序) -> 按照数量截断(目标数量限制)

**目标来源**

指从哪些途径将目标添加到池子, 目前有范围, 仇恨系统等等途径

**目标过滤**

过滤器分为两大类, 叶过滤器和组合过滤器, 其中:

叶过滤器主要负责直接对实体的筛选, 例如检查实体具体是否符合距离过滤器的要求, 具体查阅子页

组合过滤器主要定义叶过滤器之间的关系, 有以下三种:

1. must: 在must内的叶过滤器必须通过

2. mustNot: 在mustNot内的叶过滤器必须不通过

3. should: 在should内的叶过滤器至少一项通过

**目标排序**

过滤的目标按照一定规则排序, 例如离自己的距离

**目标数量限制**

限制筛选出的目标数量



其参数结构示例如下:

```json
{
    // 目标来源
    "source":{
        "scope":{},
        "hatred":{},
        ...
    },
    // 目标必须通过以下所有筛选
    "must":{
        "scope":{},
        "type":{},
        ...
    },
    // 目标必须不通过以下所有筛选
    "mustNot":{
        "group":{},
        ...
    },
    // 目标至少通过以下一个筛选
    "should":{
        "buff":{},
        ...
    },
    // 按照距离排序
    "sort":["distance"],
    // 数量限制
    "limit":10
}
```

### 应用实例

举例: 现有一目标筛选需求, 需要筛选对象距离在1000个单位内, 类型不可以是触发器, 给定一些类型的sn, 要求至少满足一项, 其参数为:

```json
{
    // 待筛选对象来源于周围1000个单位和仇恨系统的首个仇恨对象
    "source":{
        "scope":{"center":"self", "radius":1000},
        "hatred":"first"
    },
    // 必须要满足的需求 以下所有筛选通过 则加入
    "must":{
        // 按照范围筛选
        "scope":{
            "center":"self",
            "radius":[0, 1000]
        }
    },
    // 必须不满足的需求 以下所有筛选通过 则踢出
    "mustNot":{
        // 按照类型筛选
        "type":["trigger"]
    }
    // 至少满足一项的需求 待筛选对象只要满足以下任意要求即可通过 都不满足则踢出
    "should":{
        // 如果是怪物 允许的sn列表
        "monster":["monster#1", "monster#2"],
        // 如果是子物体 允许的sn列表
        "subObj":["subObj#1", "subObj#2"]
    }
}
```

现有:

怪物A在1000单位内, 其sn为monster#1, 满足以上所有条件, 则加入

怪物B在1000单位内, 其sn为monster#3, 不满足should中的所有条件, 则踢出

触发器A在1000单位内, 满足mustNot中的条件, 则踢出

实体生成器A在1000单位内, 不满足should中的所有条件, 则踢出

子物体A在1000单位外, 不满足must中的所有条件, 则踢出

### 实现思路

查询思路参考 MySQL + ElasticSearch DSL

实现思路参考 MyBatis Plus

类似SQL解析

1. 解析FROM子句, 确定查询表, 本查询先解析查询源source确定查询范围

2. 解析WHERE子句, WHERE查询有字段匹配和逻辑连接, 本查询参考DSL查询参数来丰富查询语句, 包括: 
   组合查询:must, mustNot, should, 分别对应WHERE子句中的AND, NOT, OR 
   叶查询:scope, type, group等等, 对应WHERE子句里的字段匹配, 比如 sex="m"

3. 解析ORDER BY子句, 对应本查询的sort

4. 解析LIMIT子句, 对应本查询limit

### 源码示例

```java
public class TargetQuery {

    public static class TargetQueryBuilder {
        /**
         * 实体来源规则
         */
        private final Multimap<EntitySource, Value> entitySources;
        /**
         * 必须满足规则
         */
        private final Multimap<EntityFilter, Value> mustFilters;
        /**
         * 必须不满足规则
         */
        private final Multimap<EntityFilter, Value> mustNotFilters;
        /**
         * 至少满足一项规则
         */
        private final Multimap<EntityFilter, Value> shouldFilters;
        /**
         * 排序规则
         */
        private Pair<EntitySorter, Boolean> sorter;
        /**
         * 限制数量
         */
        private int limit = Integer.MAX_VALUE;

        private TargetQueryBuilder() {
            this.entitySources = ArrayListMultimap.create();
            this.mustFilters = ArrayListMultimap.create();
            this.mustNotFilters = ArrayListMultimap.create();
            this.shouldFilters = ArrayListMultimap.create();
        }

        /**
         * 添加实体来源
         *
         * @param source 实体源
         * @return EntityQueryBuilder
         */
        public TargetQueryBuilder addSource(EntitySource source, Value param) {
            this.entitySources.put(source, param);
            return this;
        }

        /**
         * 添加必须满足规则
         *
         * @param filter 规则
         * @return EntityQueryBuilder
         */
        public TargetQueryBuilder addMustFilter(EntityFilter filter, Value param) {
            this.mustFilters.put(filter, param);
            return this;
        }

        /**
         * 添加必须不满足规则
         *
         * @param filter 规则
         * @return EntityQueryBuilder
         */
        public TargetQueryBuilder addMustNotFilter(EntityFilter filter, Value param) {
            this.mustNotFilters.put(filter, param);
            return this;
        }

        /**
         * 添加至少满足一项规则
         *
         * @param filter 规则
         * @return EntityQueryBuilder
         */
        public TargetQueryBuilder addShouldFilter(EntityFilter filter, Value param) {
            this.shouldFilters.put(filter, param);
            return this;
        }

        /**
         * 设置排序规则 重复设置会覆盖
         *
         * @param sorter 排序器
         * @param desc   是否逆序
         * @return EntityQueryBuilder
         */
        public TargetQueryBuilder setSorter(EntitySorter sorter, boolean desc) {
            this.sorter = Pair.of(sorter, desc);
            return this;
        }

        /**
         * 限制查询数量 重复设置会覆盖
         *
         * @param limit 数量限制
         * @return EntityQueryBuilder
         */
        public TargetQueryBuilder limit(int limit) {
            this.limit = limit;
            return this;
        }

        /**
         * 构建查询
         *
         * @return 查询器
         */
        public Query build() {
            return new Query(this);
        }
    }

    /**
     * 创建查询构造器
     *
     * @return EntityQueryBuilder
     */
    public static TargetQueryBuilder builder() {
        return new TargetQueryBuilder();
    }

    // 查询缓存
    private static final WeakHashMap<Value, Query> queryCache = new WeakHashMap<>(30);

    /**
     * 参数解析器
     *
     * @param param 查询参数
     * @return 实体查询
     */
    public static Query paramParser(Value param) {
        // 缓存命中
        if (queryCache.containsKey(param)) {
            return queryCache.get(param);
        }
        TargetQueryBuilder targetQueryBuilder = TargetQuery.builder();
        try {
            param.path("source").optIfPresent(sources ->
                    ((ObjectValue) sources).forEach(s -> targetQueryBuilder.addSource(EntitySource.valueOf(s.getKey()), s.getValue())));
            param.path("must").optIfPresent(filters ->
                    ((ObjectValue) filters).forEach(f -> targetQueryBuilder.addMustFilter(EntityFilter.valueOf(f.getKey()), f.getValue())));
            param.path("mustNot").optIfPresent(filters ->
                    ((ObjectValue) filters).forEach(f -> targetQueryBuilder.addMustNotFilter(EntityFilter.valueOf(f.getKey()), f.getValue())));
            param.path("should").optIfPresent(filters ->
                    ((ObjectValue) filters).forEach(f -> targetQueryBuilder.addShouldFilter(EntityFilter.valueOf(f.getKey()), f.getValue())));
            param.path("sort").optIfPresent(sort ->
                    targetQueryBuilder.setSorter(EntitySorter.valueOf(sort.path("key").asText()), sort.path("reverse").asBoolean()));
            param.path("limit").optIfPresent(limit -> targetQueryBuilder.limit(limit.asInt()));
        } catch (Exception e) {
            Log.ai.error("目标筛选参数错误 param={}", param);
        }
        return targetQueryBuilder.build();
    }

    /**
     * 查询器
     */
    public static final class Query {
        // 过滤器字典顺序与比较器 小表驱动大表 优先按照区分度高的过滤器来筛选 优化过滤性能
        // 这里省略了具体的过滤项
        private static final List<String> filterOrder = List.of("type"...);
        private static final Comparator<String> filterComparator = Comparator.comparingInt(filterOrder::indexOf);

        private final List<Function<ECS.Entity, Stream<ECS.Entity>>> entitySources = new ArrayList<>();
        private final Map<String, BiPredicate<ECS.Entity, ECS.Entity>> allMatches = new TreeMap<>(filterComparator);
        private final Map<String, BiPredicate<ECS.Entity, ECS.Entity>> anyMatches = new TreeMap<>(filterComparator);
        private final Function<ECS.Entity, Comparator<ECS.Entity>> sorter;
        private final int limit;

        private Query(TargetQueryBuilder builder) {
            builder.entitySources.forEach((s, p) -> this.entitySources.add(self -> s.source.apply(self, p)));
            builder.mustFilters.forEach((f, p) -> this.allMatches.put(f.name(), (self, target) -> f.filter.apply(self, p).test(target)));
            builder.mustNotFilters.forEach((f, p) -> this.allMatches.put(f.name(), (self, target) -> !f.filter.apply(self, p).test(target)));
            builder.shouldFilters.forEach((f, p) -> this.anyMatches.put(f.name(), (self, target) -> f.filter.apply(self, p).test(target)));
            this.sorter = builder.sorter != null
                    ? self -> builder.sorter.getLeft().comparator.apply(self, builder.sorter.getRight())
                    : self -> EntitySorter.distance.comparator.apply(self, false);
            this.limit = builder.limit;
        }
        // 执行查询
        public Stream<ECS.Entity> execute(ECS.Entity self) {
            return entitySources.stream()
                    .flatMap(e -> e.apply(self))
                    .distinct()
                    .filter(t -> t != null
                            && allMatches.values().stream().allMatch(f -> f.test(self, t))
                            && (anyMatches.isEmpty() || anyMatches.values().stream().anyMatch(f -> f.test(self, t))))
                    .sorted(sorter.apply(self))
                    .limit(limit);
        }
    }

    /**
     * 实体来源 这里省略了具体的配置
     */
    public enum EntitySource {
        /**
         * 按范围添加
         * <pre>
         *     参数示例:
         *     "scope":{
         *          "center":"blankArea#01",
         *          "radius":1000
         *      }
         * </pre>
         */
        scope((self, param) -> {
            // 查找指定距离怪物逻辑 在此省略
        }),
        ;
        public final BiFunction<ECS.Entity, Value, Stream<ECS.Entity>> source;

        EntitySource(BiFunction<ECS.Entity, Value, Stream<ECS.Entity>> source) {
            this.source = source;
        }
    }

    /**
     * 实体过滤 这里省略了具体的配置
     * 注意: 添加新的过滤器时 一定要在 TargetQuery.TargetQueryBuilder#filterOrder 中按照区分度添加索引!!!
     */
    public enum EntityFilter {
        /**
         * 按范围筛选
         * <pre>
         *     参数示例:
         *      "scope":{
         *          "center":"self",
         *      }
         * </pre>
         */
        scope((self, param) -> {
                // 距离判定逻辑 返回是否保留 在此省略
            };
        }),
        
        ;
        public final BiFunction<ECS.Entity, Value, Predicate<ECS.Entity>> filter;

        EntityFilter(BiFunction<ECS.Entity, Value, Predicate<ECS.Entity>> filter) {
            this.filter = filter;
        }
    }

    /**
     * 实体排序 这里省略了具体的配置
     */
    public enum EntitySorter {
        /**
         * 按照距离排序
         */
        distance((self, reverse) -> {
            // 距离判定逻辑, 返回一个Comparator 在此省略
        }),
        ;
        public final BiFunction<ECS.Entity, Boolean, Comparator<ECS.Entity>> comparator;

        EntitySorter(BiFunction<ECS.Entity, Boolean, Comparator<ECS.Entity>> comparator) {this.comparator = comparator;}
    }
}
```


