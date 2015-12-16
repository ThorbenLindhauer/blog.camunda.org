+++
author = "Philipp Ossler"
categories = ["Execution"]
date = "2015-12-18"
tags = ["DMN"]
title = "Benchmarking the Performance of the camunda DMN Engine"
+++

With camunda 7.4, we released the new camunda DMN engine. Some people asked how fast the DMN engine is. So I created a benchmark that measure the number of decision tables the DMN engine can evaluate per second. Since the DMN engine may usually used inside of camunda BPM (e.g. as Business Rule Task), I also did the measurement in combination with camunda BPM and determine the impact of writing the history.

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

Using a local MySQL database and only one thread.

Note that we are not interested in absolute numbers anyway since it depends on a huge number of factors (e.g. processing power, main memory, network, database, decision table, expression language etc.). You can easily run the benchmarks on your own infrastructure and with your own decision tables. See the github repositories for details:

* [Standalone DMN Engine Benchmark](https://github.com/camunda/camunda-engine-dmn-benchmark)
* [Process Engine Performance Test Suite](https://github.com/camunda/camunda-bpm-platform/tree/master/qa/performance-tests-engine)

## Standalone DMN Engine

The first benchmark uses the DMN engine independent from camunda BPM as standalone DMN engine. It embeds the engine as library and measure the number of decision tables that are evaluated per second. To focus on evaluation, the decision tables are parsed before. 

{{< figure class="teaser no-border" src="benchmark-dmn-standalone.png" alt="Benchmark of the Standalone DMN Engine" caption="" >}}

| Rules | 1 input 50% matched rules | 1 input - 100% matched rules | 2 inputs - 50% matched rules | 2 inputs - 100% matched rules |
|---|---|---|---|---|
| 2 | 220.021 | 209.189 | 147.782 | 114.469 |
| 5 | 95.544 | 94.730 | 60.826 | 49.348 |
| 10 | 49.536 | 47.254 | 34.294 | 24.795 |
| 100 | 5.077 | 4.712 | 3.466 | 2.562 |

<br/>

The DMN engine can evaluate up to 220.021 decision tables per seconds. This means that evaluating a single decision table of this takes 0,0045 Milliseconds. That is a good first result since the engine is not optimized yet.

The results show that the number of the evaluated input entries is the major performance factor. The output entries has less impact which is not surprising since the expression is simpler. So the more rules or input entries a decision table have, the more time is needed for evaluation.

Note that the performance can be influenced when the decision table has more than one input. During the evaluation of a rule, the input entries are evaluated to check if the rule is matched. If the check of an input entry return false then the other input entries are not evaluated since the rule can not be matched anymore.

## Integeration in camunda BPM

Let's repeat the measurement with the camunda BPM engine that includes the DMN engine. The benchmark using the decision service of the process engine to evaluate the decision tables that are deployed before.

{{< figure class="teaser no-border" src="benchmark-dmn-camunda-integration.png" alt="Benchmark of the DMN Engine inside of camunda BPM" caption="" >}}

First, you may recognised that the number of evaluations is much lower than using the DMN engine without camunda BPM. The main reason is the database where the decision table is deployed. Every time a decision table is evaluated, it has to be fetched from the database first. This select statement has a huge impact against the evaluation in the DMN engine. 

Since the sql statement is only executed once, the influence is less when more input entries are evaluated. For example, the decision table with 100 rules can be evaluated only 2,5 times less than using the standalone DMN engine.
 
## History vs. No History

At History Level None, the process engine does not perform any inserts to the database, it will just perform the single select statement. Turning on history the process engine insert history entries after each evaluation of a decision table. This has obviously a big performance impact.

{{< figure class="teaser no-border" src="benchmark-dmn-camunda-history-level.png" alt="Benchmark of the History Level" caption="" >}}

With history, the engine can still evaluate up to 876 decision tables per second. This is five times less than without history. 

A look at the SQL Statement Log shows that the engine performed four inserts, additionally to the one select statement. 

{{< figure class="teaser no-border" src="sql-statement-log-full-history.png" alt="SQL Statement Log" caption="" >}}

You can see that the number of inserts increased depending on the number of rules. The detail view shows why. 

```
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
