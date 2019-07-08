---
title: springboot use mongo-cursor
date: 2019-07-08 21:11:17
tags: mongo Springboot
---

本文主要介绍Mongodb中，如何使用游标。
有时候我们想着对满足条件的全部数据进行操作，如果量很大，那么肯定不能直接全部查出来，这时候使用游标进行扫描就好了。
<!-- more -->

## 实例

假定我们有如下的一个entity，用来存储某些用户访问的最后日期，他只有用户、年、月、日四项。
```java
@Data
@Document(collection = COL_NAME)
@CompoundIndex(name = "idx_month_day", def = "{'month':1, 'day':1}", background = true)
public class AccessDay {
    public static final String COL_NAME = "access_day";
    public static final String FIELD_USER_ID = "_id";
    public static final String FIELD_YEAR = "year";
    public static final String FIELD_MONTH = "month";
    public static final String FIELD_DAY = "day";
    @Id
    private Long userId;
    @Field(FIELD_YEAR)
    private Integer year;
    @Field(FIELD_MONTH)
    private Integer month;
    @Field(FIELD_DAY)
    private Integer day;
}
```

如果我想对这个月所有的用户进行处理，那么显然，不能直接查出来，这样的话量太大了，因此可以通过游标进行处理，具体逻辑处理就是通过Consumer就可以了。
每200个记录，访问一次db。

```java
    public void scanUsers(Integer month, Integer day, Consumer < Long > consumer, long maxExecTimestamp) {
        Criteria criteria = Criteria.where(FIELD_MONTH).is(month).and(FIELD_DAY).is(day);
        Query query = Query.query(criteria);
        query.fields().include(FIELD_UID);

        mongoTemplate.execute(AccessDay.class, collection - > {
            FindIterable < Document > cursorIterator = collection.find(query.getQueryObject()).projection(query.getFieldsObject()).batchSize(200);
            try (MongoCursor < Document > cursor = cursorIterator.iterator()) {
                while (cursor.hasNext()) {
                    Document document = cursor.next();
                    try {
                        consumer.accept((Long) document.get(FIELD_USER_ID));
                    } catch (Exception e) {
                        log.error("scanUsers exception. month = {}", month, e);
                    }
                }

                if (System.currentTimeMillis() > maxExecTimestamp) {
                    log.info("scanUsers, stop cause overflow maxExecTimestamp. month = {}", month, maxExecTimestamp);
                    return null;
                }
            }
            return null;
        });
    }
```