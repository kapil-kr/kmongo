# Extensions Overview

This overview presents the extensions for the synchronous driver. 

Equivalent methods are available for async drivers.

## Insert, Update & Delete

### Insert
Note that by default, KMongo serializes null values during insertion of instances with the Jackson mapping engine. This is not the case with the "native" POJO engine.

```kotlin
class TFighter(val version: String, val pilot: Pilot?)

database.getCollection<TFighter>().insertOne(TFighter("v1", null))
val col = database.getCollection("tfighter")
//collection of org.bson.Document is returned when no generic used

println(col.findOne()) //print: Document{{_id=(...), version=v1, pilot=null}}
```

The query shell format is supported:

```kotlin
database.getCollection<TFighter>().insertOne("{'version':'v1'}")

```

### Save

KMongo provides a convenient [```MongoCollection<T>.save(target: T)```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/save.html) method. If the instance passed as argument has no id, or a null id, it is inserted (and the id generated on server side or client side, respectively). Otherwise, a ```replaceOneById``` with ```upsert=true``` is performed.

### Update

KMongo provides various [```MongoCollection<T>.updateOne```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/update-one.html) methods.

- you can perform an update using shell query format:

```kotlin
col.updateOne("{name:'Paul'}", "{$set:{name:'John'}}")
```

- or with typed queries:

```kotlin
col.updateOne(Friend::name eq "Paul", set(Friend::name, "John"))
//or with annotation processor ->
col.updateOne(Name eq "Paul", set(Name, "John"))

```

- You can also update an instance by its _id (all properties of the instance are updated):

```kotlin
//explicitly
col.updateOneById(friend._id, newFriend)
//or implicitly
//note that if newFriend does not contains an _id value, the operation fails
col.updateOne(newFriend)

```

[```updateMany```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/update-many.html)
is also supported.

### Replace

The [```replaceOne```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/replace-one.html)
and [```replaceOneById```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/replace-one-by-id.html) 
methods remove automatically the _id of the replacement when using shell query format, and looks like update methods:

```kotlin
//ObjectId of friend replacement does not matter 
col.replaceOne("{name:'Peter'}}", Friend(ObjectId(), "John"))
//explicit id filter ->
col.replaceOneById(friend._id, Friend(ObjectId(), "John"))
//implicit id filter ->
col.replaceOne(Friend(friend._id, "John"))

``` 
But for typed queries you need to use [```replaceOneWithFilter```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/replace-one-with-filter.html)
as the default java driver method does not remove the _id:

```kotlin
//ObjectId of friend replacement is removed from the replace query 
//before it is sent to mongo 
col.replaceOneWithFilter(Name eq "Peter", Friend(ObjectId(), "John"))
// if the mongo java driver method is used, it will fail at runtime
// as the _id is updated ->
col.replaceOne(Name eq "Peter", Friend(ObjectId(), "John"))
```

### Delete

[```deleteOne```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/delete-one.html)
and [```deleteOneById```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/delete-one-by-id.html)
work as expected:

```kotlin
col.deleteOne("{name:'John'}")
col.deleteOne(Friend::name eq "John")
```

As does [```deleteMany```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/delete-many.html).

### FindOneAnd...

The java driver variants are also supported:

- [```findOneAndUpdate```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/find-one-and-update.html)
- [```findOneAndReplace```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/find-one-and-replace.html)
- [```findOneAndDelete```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/find-one-and-delete.html) 

### Bulk Write

If you want to do many operations with only one query (for performance reasons),
[```bulkWrite```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/bulk-write.html)
is supported.

With query shell format : 

```kotlin
val friend = Friend("John", "22 Wall Street Avenue")
val result = col.bulkWrite(
   """ [
      { insertOne : { "document" : ${friend.json} } },
      { updateOne : {
                      "filter" : {name:"Fred"},
                      "update" : {$set:{address:"221B Baker Street"}},
                      "upsert" : true
                    }
      },
      { updateMany : {
                          "filter" : {},
                          "update" : {$set:{address:"nowhere"}}
                      }
      },
      { replaceOne : {
                          "filter" : {name:"Max"},
                          "replacement" : {name:"Joe"},
                          "upsert" : true
                      }
      },
      { deleteOne :  { "filter" : {name:"Joe"} }},
      { deleteMany :  { "filter" : {} } }
      ] """
 )
 
 assertEquals(1, result.insertedCount)
 assertEquals(2, result.matchedCount)
 assertEquals(3, result.deletedCount)
 assertEquals(2, result.modifiedCount)
 assertEquals(2, result.upserts.size)
 assertEquals(0, col.count())
```

or typed queries :

```kotlin
with(friend) {
        col.bulkWrite(
            insertOne(friend),
            updateOne(
                ::name eq "Fred",
                set(::address, "221B Baker Street"),
                upsert()
            ),
            updateMany(
                EMPTY_BSON,
                set(::address, "nowhere")
            ),
            replaceOne(
                ::name eq "Max",
                Friend("Joe"),
                upsert()
            ),
            deleteOne(::name eq "Max"),
            deleteMany(EMPTY_BSON)
        )
```

## Retrieve Data

### find

[```findOne```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/find-one.html),
[```findOneById```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/find-one-by-id.html)
and [```find```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/find.html)
works as expected.

With the shell query format:

```kotlin
col.findOne("{name:'John'}")
```

or the type-safe query format:

```kotlin
col.findOne(Friend::name eq "John")
//implicit $and is used if more than one criterion is set ->
col.findOne(Name eq "John", Address eq "22 Wall Street Avenue")
```

There are also extensions for [```FindIterable```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-find-iterable/index.html).
The most used is usually the [```toList```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-iterable/to-list.html)
extension on ```MongoIterable```:

```kotlin
val list : List<Friend> = col.find(Friend::name eq "John").toList()
```

### count

[```count```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/count.html)
is supported:

```kotlin
col.count("{name:{$exists:true}}")
```

### distinct

As is [```distinct```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/distinct.html):

With shell query format:

```kotlin
col.distinct<String>("address")
```

Or type-safe queries:

```kotlin
col.distinct(Friend::address)
```

## Map-Reduce

[```mapReduce```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/map-reduce.html)
and equivalent method (to avoid name conflict with java driver) [```mapReduceWith```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/map-reduce-with.html)
are available:

```kotlin
data class KeyValue(val _id:String, val value:Long)

col.mapReduce<KeyValue>(
            """
                function() {
                       emit("name", this.name.length);
                   };
            """,
            """
                 function(name, l) {
                          return Array.sum(l);
                      };
            """
        ).first()

``` 

## Aggregate

The [aggregation framework](https://docs.mongodb.com/manual/aggregation/) is supported.

It works with shell query format, with [several limitations](mongo-shell-support/index.html):

```kotlin
col.aggregate<Article>("[{$match:{tags:'virus'}},{$limit:1}]").toList()
```

But this is an area or type-safe style shines:

```kotlin
val r = col.aggregate<Result>(
            match(
                Article::tags contains "virus"
            ),
            project(
                Article::title from Article::title,
                Article::ok from cond(Article::ok, 1, 0),
                Result::averageYear from year(Article::date)
            ),
            group(
                Article::title,
                Result::count sum Article::ok,
                Result::averageYear avg Result::averageYear
            ),
            sort(
                ascending(
                    Result::title
                )
            )
        )
        .toList()             
```

## Indexes

KMongo provides several methods to manage indexes.

### ensureIndex

[```ensureIndex```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/ensure-index.html)
and [```ensureUniqueIndex```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/ensure-unique-index.html) 
ensure that index is created even if it's structure has changed.

You can create an index with shell query format:

```kotlin 
col.ensureUniqueIndex("{'id':1,'type':1}")
```

or with type-safe style:

```kotlin
col.ensureUniqueIndex(Id, Type)
```

### dropIndex

With [```dropIndexOfKeys```](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/drop-index-of-keys.html),
 you can drop index, based on their keys:
 
```kotlin 
col.dropIndexOfKeys("{'id':1,'type':1}")
```  


## KDoc

Please consult [KDoc](https://litote.org/kmongo/dokka/kmongo/org.litote.kmongo/com.mongodb.client.-mongo-collection/index.html)
for an exhaustive list of the KMongo extensions.
