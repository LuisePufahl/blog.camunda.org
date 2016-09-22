+++
author = "Johannes Heinemann, Christopher Zell, Roman Smirnov, Askar Akhmerov and Daniel Meyer"
categories = ["Execution"]
date = "2016-09-22T12:00:00+01:00"
tags = ["Release Note"]
title = "Camunda BPM 7.6.0-alpha4 Released"

+++

Camunda 7.6.0-alpha4 is here and it is packed with new features. The highlights are:

* Reporting for Tasks
* Support for Decisions with Literal Expressions
* CMMN Engine Improvements
* Rolling Upgrades
* [23 Bug Fixes](https://app.camunda.com/jira/issues/?jql=issuetype%20%3D%20%22Bug%20Report%22%20AND%20fixVersion%20%3D%207.6.0-alpha3)

The [complete release notes](https://app.camunda.com/jira/secure/ReleaseNote.jspa?projectId=10230&version=14609) are available in Jira.

You can [Download Camunda For Free](https://camunda.org/download/)
or [Run it with Docker](https://hub.docker.com/r/camunda/camunda-bpm-platform/).

<!--more-->
# Batch cancellation of process instances

The new alpha version comes along with a new batch operation.
It is now possible to cancel process instances asynchronously based on search criteria and\or list of process instance Id's.

{{< figure class="teaser no-border" src="batch_cancelation.png" alt="Batch Process Instances Cancellation" caption="" >}}

this page is accessible from process instances search on dashboard in cockpit. [Info](https://docs.camunda.org/manual/latest/webapps/cockpit/bpmn/dashboard/#search)

In addition please check out following resources to learn about fresh API adjustments:

 * it is possible to cancel process instances asynchronously using [REST API](https://docs.camunda.org/manual/7.5/reference/rest/process-instance/post-delete/)

# Reporting for Tasks

{{< figure class="teaser no-border" src="batch_cancelation.png" alt="Batch Process Instances Cancellation" caption="" >}}

The new alpha version comes along with a new reporting feature for Tasks.
It is now possible to query reports of completed tasks, which are completed before or after a given date.

{{< figure class="teaser no-border" src="historic-task-instance-report.png" alt="Historic Task Report" caption="" >}}

Two different task reports are now available:

 * duration report contains the average, minimum and maximum duration of all completed tasks for a given timeframe.
   (Monthly and quarterly aggregation of the duration times are supported.)
 * completed task report indicates how many tasks are completed in a time span

## Reporting API for Tasks
The following example shows how to query a duration report for the completed tasks.
```java
   List<DurationReportResult> taskReportResults = historyService
      .createHistoricTaskInstanceReport()
      .completedBefore(calendar.getTime())
      .duration(PeriodUnit.MONTH);
```        
The query returns a list of duration report objects, which contains for each month the average, minimum and maximum duration of all completed tasks.

In the next example you see the query to get the report for the completed tasks grouped by the process definition key.
Each report object contains a number of tasks which are completed in the time span and the task or process definition key on which the group by was done.
```java
   List<HistoricTaskInstanceReportResult> historicTaskInstanceReportResults = historyService
      .createHistoricTaskInstanceReport()
      .countByProcessDefinitionKey();
```

For more information about the task reports and the REST API see the [reference guide](https://docs.camunda.org/manual/latest/reference/rest/history/task/get-task-report/).

# Improvement of Metric API

In the new alpha release the Metric API is improved and a new feature comes along, the metric interval query.

Currently there are lot of metrics, which are written at execution time (if enabled), like `activity-instance-start`, `job-acquisition-attempt`, `job-acquired-success`, `job-acquired-failure`, `job-execution-rejected`, `job-successful`, `job-failed`, `job-locked-exclusive` and `executed-decision-elements`. 
A new metric is introduced the `activity-instance-end` metric, which is incremented every time an activity ends. With the help of the metric interval query it is now possible to retrieve a list of metrics, on which the metric values are aggregated in defined intervals.
This feature enables new possibilities to display execution data, see for example the pictures below.

See the following example for the usage of the new metric interval query:
```java
    List<MetricIntervalValue> interval =  managementService.createMetricsQuery()
                                                           .startDate(startTime)
                                                           .endDate(endTime)
                                                           .name(Metrics.ACTIVTY_INSTANCE_START)
                                                           .interval();
```
As you can see it is possible to specify start and end time and also the name of the metric. The start and end time represents the borders of the metric interval query. The given `name` corresponds to a specific metric, only the metrics with the given name are aggregated,
if the name is not set the returned list contains entries for each interval and metric. The `MetricsQuery#interval` method returns a list of intervals with aggregated metric values. In that case the default interval length is used, which is 15 minutes (900 seconds).

Say the `startTime` is `Thu Sep 22 13:30:00 CEST 2016` and the `endTime` is `Thu Sep 22 14:00:00 CEST 2016` then the returned list contains the intervals 
`Thu Sep 22 13:30:00 CEST 2016` and`Thu Sep 22 13:45:00 CEST 2016`, since the `endTime` is exclusive.

To define a custom interval you can use the `MetricsQuery#interval(long)` method. The parameter represents the interval length in seconds, which means a call of `MetricsQuery#interval(1800)`
will aggregate the metrics in 30 minutes intervals. So the returned interval will be in that case `Thu Sep 22 13:30:00 CEST 2016`.
The metric interval query return a list of intervals with a maximum of 200 entries, they are ordered descending by the interval timestamp.

The new metric interval query is also available as REST resource.

See the following pictures for an example of the usage of metric interval queries:

{{< figure class="teaser no-border" src="metricActivityStart.png" alt="Activity Start Interval Metric" caption="" >}}

The picture above shows the interval query for the `activity-instance-start` metric, this can be used to see how often activities are started in
an interval. In that case the interval is set to 30 minutes.

{{< figure class="teaser no-border" src="metricIntervalExample.png" alt="Metric Interval Example" caption="" >}}

The picture above shows the result of the default metric interval query, which returns a list with 15 minutes intervals and aggregated values for
that interval and each metric.

# Support for Decisions with Literal Expressions

In addition to decision tables, the DMN engine now supports literal expressions as decision implementations.
This kind of decision allows to specify the decision logic as an expression. The following snippet shows an example decision:

```xml
<definitions xmlns="http://www.omg.org/spec/DMN/20151101/dmn11.xsd" id="dish" name="Desired Dish" namespace="party">

  <decision id="season" name="Season">
    <variable name="season" typeRef="string" />
    <literalExpression expressionLanguage="groovy">
      <text>calendar.getSeason(date)</text>
    </literalExpression>
  </decision>

</definitions>
```

The `literalExpression` element contains the expression and allows to set the expression language. The name of the result variable and their type is specified by the `variable` element.
You can use the expression to aggregate the result of required decisions, or to invoke a bean which provides the decision logic.

To evaluate decisions with literal expressions, the DMN engine provides new methods that work with any kind of decision logic:

```java
DmnDecisionResult result = dmnEngine.evaluateDecision(decision, variables);

DmnDecisionResult result = dmnEngine.evaluateDecision("key", inputStream, variables);
```

You can also evaluate decisions with literal expressions using the Decision Service, a Business Rule Task or a Decision Task. See the [user guide](https://docs.camunda.org/manual/latest/user-guide/process-engine/decisions/decision-service/) and the [reference guide](https://docs.camunda.org/manual/latest/reference/dmn11/decision-literal-expression/) for details.

Expect Camunda Modeler support for literal expressions in the future.

# CMMN Engine Improvements

Based on user feedback, a number of improvements have been made to the CMMN engine.

## Variable On-Parts
Probably the most interesting new feature is [Variable On-Parts](http://docs.camunda.org/manual/latest/reference/cmmn11/sentry/#variableonpart) for sentries. Variable On-Parts allow a sentry to react to `Create`, `Update` and `Delete` events for Variables. Or in other words, it is now possible to control tasks and other plan items in a case based on data. The following is an example of how to define a Sentry with a Variable On-Part:

```xml
<sentry id="Sentry_1">
  <extensionElements>
    <camunda:variableOnPart variableName="interestRate">
      <camunda:variableEvent>update</camunda:variableEvent>
    </camunda:variableOnPart>
  </extensionElements>
</sentry>
```

The above sentry has an on part which is satisfied as the variable `interestRate` gets updated.

## Case Workers improvement
Case Workers can now also terminate a Case:

```java
caseService.terminateCaseExecution(...);
```

## Manual Activation Rule
Also, please note the following bugfix concerning the interpretation of the manual activation rule attribute of CMMN. This [bugfix](https://app.camunda.com/jira/browse/CAM-6362) is included in this release and originates from an official [bugfix](https://app.camunda.com/jira/browse/OMG-12) in the OMG CMMN standard.

# Rolling Upgrades

In the past some of our Customers wanted to upgrade their Camunda cluster with minimized downtime. One solution to do this are rolling upgrades.

A rolling upgrade is a process on which the nodes are updated one by one or in groups. During the upgrade process, it is ensured that at least one node is available to handle incoming requests, guaranteeing availability and minimizing downtime.

{{< figure class="teaser no-border" src="architecture.png" alt="Architecture" caption="" >}}

Starting from 7.6.0 Camunda ensures backwards compatibility of the database schema. Backwards compatibility makes it possible to operate an older version of the process engine on a newer version of the database schema. This guarantee enables the possibility to execute rolling upgrades.

For more information about rolling upgrades see the [rolling upgrade](https://docs.camunda.org/manual/latest/update/rolling-upgrade/) documentation.

# Perspective

We have also started work providing monitoring and operation features for CMMN inside Camunda Cockpit. The next alpha release will allow users to preview these features.

{{< figure class="teaser no-border" src="cockpit-case-definition.png" alt="CMMN Cockpit" caption="" >}}

# Feedback Welcome

Please try out the awesome new features of this release and provide feedback by commenting on this post or reaching out to us in the [forum](https://forum.camunda.org/).
