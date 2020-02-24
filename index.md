# Aggregate Functions and Joins

## Aggregating data

When aggregating data in SQL there are is one large caveat to keep in mind: all selected data must either be used to group data or aggregatable (otherwise the returned results will be non-deterministic). Due to this caveat, you must also consider that showing aggregated and non-aggregated data will require you to use subqueries. Let’s look at an example to make this a bit more concrete:

```sql
select user_id, count(*)
from user_courses
where passed_at is not null
group by user_id
```

This query is very simple but let’s go over what it’s doing:

1. It groups the results in the table by the `user_id` column
2. It filters the results based on whether the `passed_at` column is **not** `NULL`
3. It selects out the `user_id` column and a count of the returned records for that `user_id`

Now let’s consider the following updated query:

```sql
select user_id, count(*), uuid
from user_courses
where passed_at is not null
group by user_id
```

This new query has a significant issue: what should the value of `uuid` be for an aggregated row? If a user has 5 passed courses (and each user course has a unique `uuid`) then how can the query know which one to use here? The answer is that it can’t. Thankfully for us Postgres does not allow queries like this! Instead you will see something like:

```
ERROR:  column "user_courses.uuid" must appear in the GROUP BY clause or be used in an aggregate function
LINE: 4 select user_id, count(*), uuid
                                  ^. (Line 4)
```

**NB:** It should be noted that older versions of MySQL (5.6 and lower) and current versions of Sqlite allow queries like this to run — meaning that the results are non-deterministic. See https://www.db-fiddle.com/f/jaTriDCETfZgHgKdZMs8dJ/1 for an example.

## Joins

Before we can do more interesting aggregation we will need to discuss joins. Joins are a way to query “relations” between different tables. For example consider our `users` and `user_courses` tables from REX. The `user_courses` table has a `user_id` column which is a reference to the `users` table. By using these references and the SQL join operators, we can combine the data of these tables. Let’s take a look at a very simplified data model and some example queries:

### Example

Users table:

| id  | org_id | name   |
| --- | ------ | ------ |
| 1   | 1      | Jacob  |
| 2   | 1      | Kazuho |
| 3   | 2      | Benson |

---

User courses table:

| id  | passed_at                | user_id | name                     |
| --- | ------------------------ | ------- | ------------------------ |
| 1   | 2020-02-24T05:21:06.351Z | 1       | Basic SQL                |
| 2   | 2019-02-24T05:21:06.351Z | 1       | SQL for dummies          |
| 3   | 2020-02-23T05:21:06.351Z | 2       | Advanced Web Development |

---

Now lets look at how this looks when joined:

```
select *
from users
join user_courses on users.id = user_courses.user_id;
```

| id  | org_id | name   | id  | passed_at                | user_id | name                     |
| --- | ------ | ------ | --- | ------------------------ | ------- | ------------------------ |
| 1   | 1      | Jacob  | 1   |                          | 1       | Basic SQL                |
| 1   | 1      | Jacob  | 2   | 2019-02-24T05:30:21.264Z | 1       | SQL for dummies          |
| 2   | 1      | Kazuho | 3   | 2020-02-23T05:30:21.264Z | 2       | Advanced Web Development |

There are a few things we should note about the above query and result:

1. Joins require you to specify the column(s) that relate the tables being joined. In our case `users.id` and `user_courses.user_id`.
2. “Jacob” has two rows in the result table, this is because they took two courses. When results are joined, **all** relevant rows are combined so if Jacob had taken 100 courses, they would have 100 entries in the resulting table.
3. Note that the user Benson does not appear in the joined result. This is because they did not take any courses

### Other types of joins

After looking at the previous example, you might be wondering “can I include users that have not taken any courses in the result?”. The answer is “YES!”. This is now the perfect time to discuss different types of joins! By default, the `join` keyword results in what is called an `INNER JOIN` (you can also write your query with `inner join` instead of only `join`). An `INNER JOIN` only returns rows which have a corresponding row in the tables being joined (this is why Benson was left out of the result above).

Let’s now discuss `LEFT JOIN`s. `LEFT JOIN`s basically say: “Give me all matching rows from the table on the left, even if there is no match in the table I am joining on”. If you look at our previous example again:

```
select *
from users
join user_courses on users.id = user_courses.user_id;
```

The “left” table would be the `users` table (meaning that order matters when joining tables). Let’s now see what we get when we change the above to use a `LEFT JOIN`!

```
select *
from users
left join user_courses on users.id = user_courses.user_id;
```

| id  | org_id | name   | id  | passed_at                | user_id | name                     |
| --- | ------ | ------ | --- | ------------------------ | ------- | ------------------------ |
| 1   | 1      | Jacob  | 1   |                          | 1       | Basic SQL                |
| 1   | 1      | Jacob  | 2   | 2019-02-24T05:39:26.233Z | 1       | SQL for dummies          |
| 2   | 1      | Kazuho | 3   | 2020-02-23T05:39:26.233Z | 2       | Advanced Web Development |
| 3   | 2      | Benson |     |                          |         |                          |

You can see that Benson is now in the result set and that all columns from the `user_courses` table are `null` for their row.

Another great use of `LEFT JOIN`s is to find rows that do not match in the join table. For example, in the above, Benson has not taken any courses, let’s see how we can get all users that have never taken a course:

```
select *
from users
left join user_courses on users.id = user_courses.user_id
where user_courses.id is null
```

| id  | org_id | name   | id  | passed_at | user_id | name |
| --- | ------ | ------ | --- | --------- | ------- | ---- |
| 3   | 2      | Benson |     |           |         |      |

Since we are selecting all columns from both the `users` and `user_courses` tables, the best way to find users that have no matching row is to find users whose corresponding `user_courses.id` value is `null`, neat!

**NB:** There are other types of joins but we will ignore them for now since `INNER` and `LEFT` joins are the most commonly used by far.

To play around this this dataset yourself go to: https://www.db-fiddle.com/f/veaETuNTKrR1QKrcbBXRUe/0

## Aggregating data across joins

Now that we have gone over the basics of joins, let’s get back to data aggregation. Now that we have the knowledge of joins, let’s do a slightly more interesting query; let’s think about how we can find out how many courses each user has passed (including users that have not passed any courses)!

```
select users.name, COUNT(user_courses.id)
from users
left join user_courses on users.id = user_courses.user_id
group by users.id
```

| name   | count |
| ------ | ----- |
| Kazuho | 1     |
| Benson | 0     |
| Jacob  | 2     |


As you can see this combines what we have already learned! We do the join, define the how we want to aggregate with the `group by` and then we count the number of user courses per user! An interesting note here: we previously noted that non-aggregated or grouped columns do not work in aggregated queries; however, if Postgres can determine that the column only has a single matching row, it will allow the query as above.

## Filtering by aggregated results

Now what if we were asked to only return users who had passed more than 1 course? Your first thought might be something like:

```
select users.name, COUNT(user_courses.id) as count
from users
left join user_courses
  on users.id = user_courses.user_id
where count > 1
group by users.id
```

This does not work, we cannot refer to values calculated in the `select` clause as filters; however, there are two ways that we can do this!

### The `HAVING` clause

SQL has a nifty keyword called `HAVING` that can only be used in queries with a `GROUP BY` clause. Let’s see how this works:

```
select users.name, COUNT(user_courses.id) as count
from users
left join user_courses on users.id = user_courses.user_id
group by users.id
having COUNT(user_courses.id) > 1;
```

| name  | count |
| ----- | ----- |
| Jacob | 2     |

As you can see all we have to do is write the `HAVING` clause afer the `GROUP BY` and then specify the comparison we want, in this case it’s the same as in the `SELECT` clause.

### Subqueries

Subqueries are ways to combine the results of multiple queries. In more complicated aggregations, you will find yourself unable to use a `HAVING` clause and need to reach for subqueries (in fact these are useful for many use cases!).

Let’s see how we could re-write the above to use a subquery:

```
select users.name, user_courses_with_count.count
from (
  select user_courses.user_id, count(user_courses.id) as count
  from user_courses
  group by user_courses.user_id
) user_courses_with_count
left join users on users.id = user_courses_with_count.user_id
where user_courses_with_count.count > 1;
```

| name  | count |
| ----- | ----- |
| Jacob | 2     |

Let’s break this query down a bit:

The first thing you might notice is that the `FROM` clause is not selecting from a table but from another query, that’s the subquery! You can see we only select the `user_id` (to be used in a join later) and a count of the number of courses per user. The next thing to note is the name `user_courses_with_count` which is used to **name** the subquery (SQL requires us to give this a name so that it knows how to refer to it), in addition, you can optionally write this like `as user_courses_with_count` if that makes it more explicit for you.

Next we left join onto the `users` table as before and then use a `WHERE` to filter out users that have less than or equal to one course.

This example probably looks far more complicated than the first one. In this instance, a subquery really isn’t need since we have `HAVING` but there will be times where you will find yourself needing subqueries and they can be very helpful in the right circumstances.

## Challenge

Now that we have gone over the basics of joins and aggregation I have a challenege for you using the REX database:

Find the names of all programs that have more than 2 courses.