# Temporal Data Roadmap<br/>

Paul A. Jungwirth

PGConf.dev 2026

TODO

Notes:

- Hi, I'm Paul Jungwirth, a freelance programmer from Portland, Oregon.
- I've been working on adding support for SQL:2011 temporal features to Postgres, in particular application time.
- I want to talk about what we have, what's coming in v19, and most of all what else we could do.



# SQL:2011

[x] `PRIMARY KEY` + `UNIQUE` `WITHOUT OVERLAPS`
[x] `FOREIGN KEY` `PERIOD`
[ ] `UPDATE/DELETE FOR PORTION OF`
[ ] `CASCADE/SET NULL/SET DEFAULT`
[ ] `PERIOD FOR valid_at (valid_from, valid_til)`
[ ] `SYSTEM TIME`


Notes:

- Here are things laid out by the SQL standard.
    - Temporal primary and foreign keys got into v18 (so far!),
        - We let you define these constraints on any rangetype column. Or it could be a multirange.
    - I've been working on temporal `UPDATE` and `DELETE` for v19, but it looks like they'll need to wait 'til next time.
    - I did get something else committed, which enables some cool temporal things, but it's just for temporal.
    - I also have patches out there for letting temporal foreign keys cascade, and also for SQL:2011 PERIODs.
    - The `PERIOD`s patch doesn't add anything that ranges can't do, but it you write more standards-compatible SQL.
    - Under the hood it's really rangetypes.
- If all that gets in, Postgres will be the first RDBMS to support all of SQL:2011 application time.
- And then there is system time! Corey Huinker has been working on that.


# Arbitrary Types
## PKs and UQs

getcomparetype....

Notes:

- Postgres has a long history of extensibility. I think this is was folks meant when they called it an "object-relational" database: you can define new types by implementing various interfaces to do what you want.
- So I think it would be great if we could define more than just ranges and multiranges.
- This might be useful for geometric types like Box or the types from PostGIS.
    - This feature can be useful for more than just time.
    - Or as we'll see later, I have my own use-case for multiple temporal dimensions.
- For a primary key or unique constraint, we are almost there. It needs an overlaps operator.
    - At first there was a problem: we identify operators by strategy number, but only btree enforces well-known strategy numbers.
    - For gist, each opclass can use whatever strategy numbers it wants.
    - Within core, we have some constants that start with `RT`, and our opclasses use those.
    - But within the `btree_gist` extension, we use the btree constants instead.
    - So even in our own repo, there is no guaranteed strategy number.
    - So how do we ask for "the overlaps operator"?
    - In v18 we added a new gist support function for that.
    - You call .... and it translates the compare type to a strategy number.
    - In the case of gist, it delegates to our support function.
    - Other IAMs could have support functions too.
- So that's solved, and I thought we would achieve arbitrary types, but in v17 a problem came up: empty ranges!
    - We had to add some custom code to reject empty ranges, which have no temporal meaning and allow duplicates to slip in to a supposedly-unique index.
    - Consider that a temporal unique constraint is equivalent to an exclusion constraint with equals on the first column and overlaps on the second, like this: `(id WITH =, valid_at WITH &&)`.
      If all those operators return true, then it's a "duplicate" and should be rejected.
      But with ranges, empty never overlaps anything, even another empty.
      So that operator will always return false.
      That means you can have `(5, 'empty')` and `(5, 'empty')`, and it's allowed.
      But we can't allow that. A temporal unique constraint should be at least as restrictive as a regular one.
      If we did allow it, we would get duplicate rows in GROUP BY output, and probably other errors from the planner over-optimizing.
      So today we hardcode ranges and multiranges to forbid empties.
    - How would we support other types here?
      They could come with a function to reject possible values.
      Maybe that's another support function.
      An IAM that wanted to support WITHOUT OVERLAPS would have a callback, and (in GiST's case) it would call the support function from the opclass to validate the value before using it.
      Alternately we could put the onus on the user to forbid these records manually, e.g. with a domain or check constraint. But that seems like a big footgun.



# Arbitrary Types
## Foreign Keys

```
typedef struct RI_ConstraintInfo
{
  ....

  /* anyrange <@ anyrange */
  Oid   period_contained_by_oper;

  /* fkattr <@ range_agg(pkattr) */
  Oid   agged_period_contained_by_oper;

   /* anyrange * anyrange */
  Oid   period_intersect_oper;
}
```

Notes:

- Foreign keys also don't *quite* support arbitrary types, although I wish they did.
- They are so close. Already almost everything we need in core is provided by operators or support functions.
- The missing piece is an intersect operator.
- Unlike the others, we can't look this up by stratnum in `pg_amop`.
  - That table is for opclasses, and opclasses pertain to *indexes*.
  - This really has nothing to do with indexes.
  - Although I recently came across some place in the docs admitting that opclasses were already used for more than just indexes.
  - But everything in `pg_amop` is either a "search" operator or an "ordering" operator. We call this the "purpose".
  - Intersects isn't either of those.
- It seems like we need a "type support function".
  Apparently this same idea was raised last year for ... TODO ....
  It's amazing how types have all kinds of callbacks, like in and out and send and recv, and many other things, but we don't have a formalizing concept covering them all, like we do with opclass support functions.
- Maybe opclass *is* the right place to put this though, so that users can override a type's normal behavior. And since an FK always references a unique index, we could get the opclass from there.
  We would just need to add a third "purpose", or maybe "other".



# Aribtrary Types
## FOR PORTION OF

TODO picture

Notes:

- Similarly we can almost support temporal update/delete on arbitrary types, but a couple things are missing.
- One is the intersect operator, so if we solve that for foreign keys, we can use it here too. But note that we may not have any indexes at all, so that's a reason not to attach it to opclasses.
- The second things we need is a way to get leftovers. When a temporal update or delete happens, it might touch only part of the history represented by the row being changed. In that case we apply the update or delete, but we trim the start & end time to match only the updated/deleted portion, and whatever history was untouched needs to be inserted as extra rows, to preserve that history. I call these new records "leftovers". We have a helper function that gets the row's application time and the history targeted by `FOR PORTION OF`, and it returns one or more leftovers. It's a set-returning function, so it can return as many leftovers as it needs. We have this function for ranges and multiranges. We need a way for other types to offer it. Here there may be no index involved, so we truly can't rely on opclasses.



# Joins

```
SELECT  *,
        -- intersection:
        a.valid_at * b.valid_at AS valid_at
FROM    a
JOIN    b
ON      a.id = b.id
        -- overlaps:
AND     a.valid_at && b.valid_at;
```

Notes:

- SQL:2011 says nothing about how to do a temporal join, although there is research going back to at least the early 90s, maybe the 80s.
- The issue is that when you join two rows, you want to know the new valid time. It's the intersection of the left row and the right row. Okay, that's easy to do. A join is just this. We don't need no freakin' standards!
- Well, inner join.
- But for the rest, we could really use some help.



# Semijoin

```
SELECT	*, a.valid_at * b.valid_at
FROM	a
WHERE	EXISTS (
          SELECT  1
          FROM    b
          WHERE   a.id = b.id
          AND     a.valid_at && b.valid_at
        );
```

Notes:

- Here is a semijoin.
  But this doesn't work!
  As before, we want the intersection. But b isn't in scope out there.



# Semijoin

```
SELECT  a.id,
        UNNEST(multirange(a.valid_at) * j.valid_at) AS valid_at
FROM    a
JOIN (
  SELECT  b.id, range_agg(b.valid_at) AS valid_at
  FROM    b
  GROUP BY b.id
) AS j
ON a.id = j.id AND a.valid_at && j.valid_at;
```

Notes:

- Here is what actually works.
  We do a regular join, but we have to take care with one-to-many relationships,
  because even if `a` matches two or three `b`s, we still want it only *once* in the output.
  That's what the grouping is for.
  We combine all the matchin `b`s into one multirange with `range_agg`. This prevents duplicating `a`.
- Then once all the matching bs are in one row, we intersect with `a`.
- Then to get back to rangetypes we `UNNEST` the multirange.
- You can imagine this would be a lot harder without multiranges.



# Antijoin

```
TODO (bold or fade in the outer join)
```

Notes:

- Antijoin is very similar.
- It has the same scoping problem, so that a correlated subquery doesn't work.
- The key difference is we use an outer join instead.
- Intuitively that should make sense: an antijoin needs to find what *doesn't match*, and that's what an outer join gives us.



# Outer Join

```
```

Notes:

- The hardest of all is an outer join.
- Boris Novikov and I have traded a bunch of implementations for this.
- The most theoretically straightforward is to take an inner join, then append an antijoin: everything that matches plus everything that doesn't match.
- But if I just write the SQL for that, say with UNION ALL, I wind up scanning both tables twice. That shouldn't be necessary.
- Boris gave me several queries that avoid double-scanning. This is my favorite.
- You can find all these queries in my github repo called "temporal ops".
  It's in the references at the end.



# Encapsulating

```
SELECT	a, b
FROM	temporal_semijoin(
          'a', 'id', 'valid_at',
          'b', 'a_id', 'valid_at'
        ) AS j(a a, b b)
WHERE	a.id = 5;
```

Notes:

- But memorizing these SQL patterns is asking too much.
- We want to encapsulate them somehow. Like this.
- This function builds a SQL string using the passed-in table and column names,
  executes it, and returns the result.
- Here's the problem: this can't be a SQL function; it has to be PL/pgSQL or something similar.
  And those functions are opaque to the planner.
  Since the planner knows SQL, it can inline a SQL function and then do normal optimizations.
  Most important, it can push a qual down into the subquery, like this line here.
  With an opaque function definition, it can't do that.
  So this function will join every row in both tables, and then throw most of it away.
- That's no good either!



# `SupportRequestInlineInFrom`

Notes:

- But in v19 we are getting a new support request, `SupportRequestInlineInFrom`.
- When we define a function, you can attach a "support function",
  which is a helper function that can answer questions for the planner,
  like "How many rows do you expect to return?" or "What is your selectivity?"
- I added a new support request so that a set-returning function can replace itself with an equivalent subquery. Then the planner can inline it just as it inlines SQL functions.



# `temporal_semijoin_sql`

```
appendStringInfo(&q,
  "SELECT %2$s, UNNEST(multirange(%2$s.%3$s) * %4$s.%5$s) AS %6$s\n"
  "FROM %1$s\n"
  "JOIN (\n"
  "  SELECT ",
  left_nsp_rel_q, left_rel_q, left_valid_col_q,
  subquery_alias, right_valid_col_q, result_valid_col_q);
```

Notes:

- Here is a snippet from that support function.
  - I picked the nastiest part I could.
- Actually the support function and the main function share the same SQL-generating code,
  so we can be pretty sure they are equivalent.
- Note you don't have to build SQL strings; you could construct a node tree directly.
  But doing it this ways makes things easy, and it lets you share code between main function and support function.



# Custom Scan

TODO: docs for CustomScan or maybe .h snippet?

- SQL implementations are nice because they let people do these things now,
  but it would be nicer to have an executor node that does it.
  Then we could really think about an optimal algorithm for getting what we want.
- One way for an extension to achieve that might be a CustomScan.
- What I haven't yet figured out is how do I inject a CustomScan into the node tree?
  - It's an exec node, not a plan node, so my support request can't return one (I think).
  - I think other extensions have used planner hooks to add them. This is something I want to explore soon.
  - Dignoes paper on indexes?....



# In Core?

TODO: picture of dignoes post from mailing list archives

Notes:

- Even better than a CustomScan would be to have this stuff in core!
- Of course there is no standard syntax for it.
- Some folks actually submitted a patch to do this, back in .... TODO.
- They responded to feedback for a year or two, but eventually they stopped posting updates.
- It might be worth resurrecting their patch, or maybe doing something else.
- I think we probably need something else.
- Their approach was very elegant, because they only need two new primitive node types,
  and they showed that you could implement a temporal analogue to every relational operator by combining those two primitives with existing non-temporal operators.
  - But my guess is that the way they are transforming a temporal op into a whole tree of exec nodes is not optimal, a bit like doing it in SQL.



# Syntax: `valid_at`?

```
.... TODO .... imaginary syntax with valid_at
```

Notes:

- One question is how do you name the result's application time?
  This is a syntax question.
- It's important to retain the input valid-times as well.
  Those are useful for scaling aggregate function inputs, and maybe for other things as well.
  Who knows what the user will do with them.
  So we need a way to name the result valid-time so it doesn't conflict with those other columns, if the user needs them.
- With my PL/pgSQL functions, it wasn't an issue, because (1) I was returning separate rowtype values for the left & right inputs (2) the user has to give a column definition list as part of the FROM alias, so they can name the result valid-time whatever they want.



# Relational Algebra

Notes:

- If we implemented these joins in core, we would need to consider the algebraic identities we use to transform plan trees. I've found one paper from the 90s that addresses this, and I've had some helpful correspondence with ...TODO... and Boris Novikov. This is my best understanding so far. The main issue is that the result has new valid-times, and some customary transformations don't preserve them correctly.



# Setops



# Aggregates



# MERGE



# Optimizations

- Use btree
  - Skip scan?
- Improve gist
  - Binary search
- Recognize snapshot queries
  - `n_distinct`
  - `GROUP BY`

Notes:

- What are we giving up by using GiST indexes instead of B-Tree? Need to benchmark, but I suspect it's a lot.
  - Can we support temporal constraints backed by a B-Tree index?
  - I feel like there is some affinity with skip scans here, but so far it's just a vague feeling.
- I'm sure there is a lot of optimization potential left in GiST indexes as well.
  - For instance today we scan *all tuples on the page* for a match.
  - Btree is able to use binary search to avoid that.
  - One of the original GiST papers suggests that for datatypes with a ordering function, we could arrange their tuples so that binary search is possible.
- A `WITHOUT OVERLAPS` index is *both* temporally unique *and* traditionally unique, but also if your query is selecting one point in time, we can guarantee that the remaining keys are alone sufficient for uniqueness. (TODO: Image here of a timeline with different ids and their versions, and a dashed vertical line cutting through.)
  - These are often called "snapshot queries". Snodgrass in the 90s called them a "time-slice queries".
    - It got a special name because it has special properties.
    - In such a query, a temporal table goes in and a non-temporal table comes out.
  - I'm sure there are missed optimization opportunities here. 
    - Since it's a new feature, there will be a lot of low-hanging fruit.
    - For instance index statistics track `n_distinct`.
      - Temporal indexes could track `n_scalar_distinct`, which is the same thing but ignoring the valid-time column.
      - Then if we have a snapshot query we could use that.
  - And the improvements are not limited just to optimizations:
    - For instance when you `GROUP BY`, you're allowed to select non-grouping columns if we can prove that they are "functionally dependent" on the grouping columns. For instance if I group by product id (say because I'm joining to all its sales), I can still select the product name.
    - In a snapshot query, the temporal primary key "decays" to a primary key with just its scalar columns, so grouping by those is sufficient to prove functional dependency.


# mdrange



# DDL Changes

TODO: picture of schema evolution paper

show the syntax

Notes:

- TODO: explain how it works, introduce the example data
- Both valid-time (i.e. application time) and transaction time:
    - At (transaction) time x, we thought the schema at (valid) time y was S.
- Supports adding and dropping columns.
- Doesn't support changing the type of a column. Or dropping then adding with a different type.
- What does the table's file layout look like?



# Thanks

TODO

Notes:

- Thanks for listening!
- I feel like there is enough here to keep me busy for years.
- If you are excited about any of this, let's collaborate!
  - Of course even reviews are very welcome.
- Or if you want to hire me to work on it, that would be exciting too.



# References
<!-- .slide: class="references" -->

- https://github.com/pjungwir/temporal-roadmap
- TODO: sql:2011 papers, e.g. kulkarni
- TODO: dignoes paper
- TODO: dignoes index paper
- https://github.com/pjungwir/temporal_ops
- TODO: schema evolution paper, 1992



# Thank you

https://github.com/pjungwir/temporal-roadmap
