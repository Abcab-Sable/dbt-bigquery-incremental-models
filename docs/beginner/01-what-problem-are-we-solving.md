# 1. What problem are we even solving?

Before any of the dbt vocabulary, here's the actual situation.

## You have a table, and it keeps growing

Imagine your company records every click on its website. Every click becomes a
row in a table called `raw_clicks`:

| click_id | user_id | clicked_at          | page      |
| -------- | ------- | ------------------- | --------- |
| 1        | 42      | 2026-08-30 09:14:00 | /home     |
| 2        | 42      | 2026-08-30 09:14:30 | /pricing  |
| 3        | 77      | 2026-08-30 11:02:00 | /home     |

A busy site produces millions of these a day. After two years you have billions
of rows.

Now your analyst wants a cleaned-up version: bad rows removed, timestamps
converted to your local timezone, bot traffic filtered out. Call it
`clean_clicks`.

## The obvious approach, and why it stops working

The obvious way to build `clean_clicks` is to write a query that reads all of
`raw_clicks`, cleans it, and saves the result:

```sql
select
    click_id,
    user_id,
    clicked_at,
    page
from raw_clicks
where user_id is not null
  and page not like '/bot%'
```

You run it every morning. It throws away yesterday's `clean_clicks` entirely and
rebuilds it from scratch.

This works fine for a while. Then it doesn't, for two reasons.

**It gets slow.** Reading two years of clicks takes longer every single day, even
though the amount of *new* data each day is constant.

**It gets expensive.** This is the part that bites hardest on BigQuery. BigQuery
charges you based on **how much data your query reads**. Rebuilding from scratch
means reading all two years, every morning. You pay for two years of data to
process one day of new clicks. Tomorrow you'll pay for two years and one day.

Here's the shape of the waste:

```
Day 1:    read 1 day    -> build 1 day of output
Day 2:    read 2 days   -> build 2 days of output   (1 day was already correct)
Day 3:    read 3 days   -> build 3 days of output   (2 days were already correct)
...
Day 730:  read 730 days -> build 730 days of output (729 days were already correct)
```

By day 730 you are redoing 729 days of work you already did correctly, to add one
day of new information.

## The fix, in one sentence

**Only process the new data, and add it to what's already there.**

That's it. That's what "incremental" means. Instead of rebuilding the whole
table, you work out what's new, process only that, and append or update it into
the existing table.

```
Day 730 (incremental):  read 1 day -> add 1 day to the existing table
```

You've gone from reading 730 days to reading 1. On BigQuery's pricing model,
that's roughly a 730× cost reduction for that step.

## Why this is harder than it sounds

If it were as simple as "append the new rows", nobody would need documentation
for it. Three things make it genuinely tricky.

### 1. What counts as "new"?

You need a rule. Usually it's a timestamp: "anything with `clicked_at` later than
the newest row I already have." But:

- What if data arrives **late**? A click from Tuesday shows up on Thursday. If
  your rule is "later than the newest row I have", and you already have Wednesday
  rows, Tuesday's late click is silently skipped forever.
- What if data gets **corrected**? Someone fixes a mistake in Monday's data. Your
  table still has the old wrong version. "Only new rows" won't catch it.

### 2. What if a row you already have has changed?

Say you're tracking orders, not clicks. An order's status changes from `pending`
to `shipped` to `delivered`. The row already exists in your table — you don't
want a second copy, you want to **update** the one that's there.

Now you need a way to say "this incoming row is the same order as that existing
row". That's what a **unique key** is for, and we'll come back to it a lot.

### 3. What if a row you already have should be deleted?

Someone cancels an order. It disappears from the source. Your table still has it,
because "process the new data" never told you to remove anything.

This one is the source of the nastiest surprise in the whole system, and it gets
[its own section later](06-when-things-go-wrong.md#the-partition-that-would-not-empty).

## What dbt does about it

**dbt** is a tool that manages these SQL transformations for you. You write the
`select` statement describing what you want; dbt handles building the table,
working out the order to build things in, and — the part we care about — the
machinery of adding new data to an existing table.

dbt gives you a setting called `materialized='incremental'`. Turn it on, and dbt
stops rebuilding your table from scratch and starts doing the add-only-the-new-
data dance instead.

The rest of this track is about what that dance actually looks like, because dbt
offers you several different dances and choosing the wrong one produces wrong
data rather than an error message.

## The thing to hold onto

Incremental models trade **simplicity for cost**. A full rebuild is always
correct — it recomputes everything from source, so it cannot drift. An
incremental model is cheaper but can be *subtly wrong* in ways that don't
announce themselves.

Almost every problem in this documentation comes from that trade. Keep it in
mind and the rest will make sense.

---

Next: [2. The words people will use at you](02-vocabulary.md)
