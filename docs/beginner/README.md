# The beginner track

This track assumes you know **nothing**. Not dbt, not BigQuery, not what a
partition is. If you've been dropped onto a data team and told to "fix the
incremental model", start here and read straight through.

It is deliberately long. Every term is defined before it's used, and every claim
is worked through with an example you can follow. Nothing is left as "you'll pick
it up".

## Read in order

1. [What problem are we even solving?](01-what-problem-are-we-solving.md)
2. [The words people will use at you](02-vocabulary.md)
3. [Your first incremental model](03-your-first-incremental-model.md)
4. [Partitions, explained properly](04-partitioning-explained.md)
5. [The three strategies](05-the-three-strategies.md)
6. [When things go wrong](06-when-things-go-wrong.md)
7. [Where to go next](07-where-to-go-next.md)

## How to use this alongside the others

There are three tracks covering the same material at different depths. When you
stop needing the explanations here, move to the
[balanced track](../balanced/01-how-the-materialization-runs.md) — same
behaviours, less hand-holding, more precision about the source code.

The [expert track](../expert/README.md) is a condensed reference. It will not
make sense yet, and that's fine. It's where you'll end up in six months.

## Two promises about accuracy

**Simplified in presentation, never in claims.** Where something is genuinely
more complicated than the explanation lets on, there's a note saying so and
pointing at the fuller version. You will not have to unlearn anything you read
here.

**Two kinds of fact, and they're sourced differently.** Everything this track
says about *dbt's behaviour* was verified by reading the adapter source at the
commit pinned in [the repository README](../../README.md). Everything it says
about *BigQuery itself* — the pricing model, the 4,000-partition limit, how
pruning works — is standard platform behaviour, not read from dbt's code. Both
are accurate; only the first is version-pinned. Check the BigQuery documentation
if a platform limit matters to a decision you're making.

---

Back to [the repository README](../../README.md)
