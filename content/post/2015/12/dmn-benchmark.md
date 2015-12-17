+++
author = "Philipp Ossler"
categories = ["Execution"]
date = "2015-12-18"
tags = ["DMN"]
title = "Benchmarking the Performance of the camunda DMN Engine"
+++

With camunda 7.4, we released the new [Camunda DMN engine](https://docs.camunda.org/manual/7.4/user-guide/dmn-engine/). Some people asked how fast the DMN engine is. So I created a benchmark measuring the number of decision tables the DMN engine can evaluate per second.

The DMN engine can be used both standalone (as a stateless decision engine) and embedded inside Camunda Process Engine (adding features such as Repository, History and Auditing and integration in BPMN).
In this post I show results for both usage scenarios. This gives insight into the overhead introduced by features such as the repository and the history.

<!--more-->

## Evaluated Decision Tables

I did the benchmarks with different decision tables to monitor the influence of 

* the total number of rules
* the number of matching rules
* the number of inputs 

All decision tables pass the input values from a double variable. The variable is compared with a double value that controls the number of matching rules. The output value is a simple string.

{{< figure class="teaser no-border" src="decision-table.png" alt="One of the evaluated decision tables." caption="Decision Table with five Rules and one Input" >}}

## Benchmark Infrastructure

I did the benchmarks on my local machine:

* Intel Core i7 (4 Cores, 2.69 Ghz)
* 8GB Main Memory
* SSD Hard Disk,
* Windows 10

All Tests use a single thread. For the tests using history and repository, I used a local MySQL database.

Note that we are not interested in absolute numbers anyway since it depends on a huge number of factors (e.g. processing power, main memory, network, database, decision table, expression language etc.). You can easily run the benchmarks on your own infrastructure and with your own decision tables. See the github repositories for details:

* [Standalone DMN Engine Benchmark](https://github.com/camunda/camunda-engine-dmn-benchmark)
* [Process Engine Performance Test Suite](https://github.com/camunda/camunda-bpm-platform/tree/master/qa/performance-tests-engine)

## Standalone DMN Engine

The first benchmark uses the DMN engine standalone (independently from Camunda Process Engine). It measures the number of decision tables that are evaluated per second. To focus on evaluation, the decision tables are parsed before. 

{{< figure class="teaser no-border" src="benchmark-dmn-standalone.png" alt="Benchmark of the Standalone DMN Engine" caption="" >}}

| Rules | 1 input 50% matched rules | 1 input - 100% matched rules | 2 inputs - 50% matched rules | 2 inputs - 100% matched rules |
|---|---|---|---|---|
| 2 | 220.021 | 209.189 | 147.782 | 114.469 |
| 5 | 95.544 | 94.730 | 60.826 | 49.348 |
| 10 | 49.536 | 47.254 | 34.294 | 24.795 |
| 100 | 5.077 | 4.712 | 3.466 | 2.562 |

<br/>

The DMN engine can evaluate up to 220.021 decision tables per second with a single thread. This means that evaluating a single decision table takes 0,0045 Milliseconds. This is a good first result since the engine is not optimized yet.

The results show that the number of the evaluated input entries is the major performance factor. The output entries have less impact which is not surprising since the expression is simpler. The more rules or input entries a decision table has, the more time is needed for evaluation.

The results also show that the performance can be influenced when the decision table has more than one input. During the evaluation of a rule, the input entries are evaluated to check if the rule is matched. If the check of an input entry returns false then the other input entries are not evaluated since the rule can not be matched anymore.

## Integeration in camunda BPM

Let's repeat the measurement with the Camunda Process Engine that includes the DMN engine. The benchmark using the decision service of the process engine to evaluate the decision tables that are deployed before.

{{< figure class="teaser no-border" src="benchmark-dmn-camunda-integration.png" alt="Benchmark of the DMN Engine inside of camunda BPM" caption="" >}}

First, you may recognize that the number of evaluations is much lower than using the DMN engine standalone. The main reason for this is the repository feature: the test starts the "latest" version of a decision table. For each evaluation, the latest version needs to be determined which is effectively a SQL `SELECT` statement. Obviously, this `SELECT` statement has a huge impact on performance.

This `SELECT` statement is a lot more expensive than the other operations necessary to evaluate a decision table. As a result, it dominates the performance of the evaluation. This explains why increasing the number of input entries and rules has propotionally less influence on performance than when the DMN engine is used standalone.
 
## History vs. No History

At History Level None, the process engine does not perform any inserts to the database, it will just perform the single select statement to verify it has the latest version in the cache. Let's now check how switching on history influences the the performance. In case you are not familiar with the history feature in Camunda: it is possinble to record each evaluated decision ("decision instance") including information about the inputs and the matched rules. Read the [Documentation](https://docs.camunda.org/manual/7.4/user-guide/process-engine/decisions/history/) for more information.

{{< figure class="teaser no-border" src="benchmark-dmn-camunda-history-level.png" alt="Benchmark of the History Level" caption="" >}}

As we can see, history comes with an additional performance penalty. With history, the engine evaluates up to 876 decision tables per second. This is five times less than without history.

A look at the SQL Statement Log shows that the engine performes 4 inserts in addition to one select statement. 

{{< figure class="teaser no-border" src="sql-statement-log-full-history.png" alt="SQL Statement Log" caption="" >}}

You can see that the number of inserts increases depending on the number of rules. The detail view shows why. 

```json
{
  "statementType" : "SELECT_MAP",
  "statement" : "selectLatestDecisionDefinitionByKey",
  "durationMs" : 3
}, {
  "statementType" : "INSERT",
  "statement" : "insertHistoricDecisionOutputInstance",
  "durationMs" : 1
}, {
  "statementType" : "INSERT",
  "statement" : "insertHistoricDecisionOutputInstance",
  "durationMs" : 0
}, {
  "statementType" : "INSERT",
  "statement" : "insertHistoricDecisionOutputInstance",
  "durationMs" : 1
}, {
  "statementType" : "INSERT",
  "statement" : "insertHistoricDecisionOutputInstance",
  "durationMs" : 0
}, {
  "statementType" : "INSERT",
  "statement" : "insertHistoricDecisionOutputInstance",
  "durationMs" : 1
}, {
  "statementType" : "INSERT",
  "statement" : "insertHistoricDecisionInstance",
  "durationMs" : 0
}, {
  "statementType" : "INSERT",
  "statement" : "insertHistoricDecisionInputInstance",
  "durationMs" : 1
}
```

The process engine performs one insert for each output value. So the number of matching rules and the number of outputs have an influence of the performance when the history is on.

This concludes my little excursion into the world of DMN benchmarking. Make sure to leave comments below the post in case you have any questions on this.
