<h1>Open CI Signals Log Specification (Draft)</h1>

**Authors**: [Adam Thomas](adamscybot)

- [1. Disclaimer](#1-disclaimer)
- [2. Introduction](#2-introduction)
- [3. Background](#3-background)
  - [3.1. File based collection](#31-file-based-collection)
  - [3.2. Agent-based observability](#32-agent-based-observability)
  - [3.3. Log-based observability](#33-log-based-observability)
    - [3.3.1. Issues preventing reuse of prior art](#331-issues-preventing-reuse-of-prior-art)
- [4. Specification](#4-specification)
  - [4.1. Signal Log Line](#41-signal-log-line)
    - [4.1.1. Format](#411-format)
      - [4.1.1.1. Syntactical requirements](#4111-syntactical-requirements)
      - [4.1.1.2. Ident wrapper](#4112-ident-wrapper)
      - [4.1.1.3. Signal JSON](#4113-signal-json)
  - [4.2. Message Types](#42-message-types)
    - [4.2.1. Built-in Message Types](#421-built-in-message-types)
  - [4.3. Message](#43-message)
    - [4.3.1. Common message properties](#431-common-message-properties)
      - [4.3.1.1. ID](#4311-id)
      - [4.3.1.2. Execution ID](#4312-execution-id)
      - [4.3.1.3. Type](#4313-type)
      - [4.3.1.4. Timestamp](#4314-timestamp)
      - [4.3.1.5. Links](#4315-links)
        - [4.3.1.5.1. Parent](#43151-parent)
        - [4.3.1.5.2. Resolves](#43152-resolves)
      - [4.3.1.6. Payload](#4316-payload)
- [5. Parser behaviour](#5-parser-behaviour)


## 1. Disclaimer

This document is currently in a DRAFT/UNPUBLISHED state and is subject to change. It has not been finalized and should not be considered authoritative or complete. It is thus currently unversioned.

## 2. Introduction

The Open CI Signals Log Specification aims to provide a standardised logging format for Continuous Integration (CI) systems that enables downstream tools to more easily observe the *granular* state of a CI job, during or after its execution, by analysing these logs. It is inherently decoupled from any one CI software product, or CI automation tooling whilst remaining extensible to allow for custom integrations. It theoretically allows the construction of generic tooling to provide for basic observability of CI jobs without requiring changes to CI agents or platform configuration, or requiring sysadmin level permissions.

The standard is designed to provide for common events which occur *within* processes that are executing inside of the job, to a high level of granularity. For example, formats are defined to represent events relating to test runners, in order for each individual test to be tracked.

The standard also allows for a "meta-programming" approach where log lines can be output which define the definition of a *type* of an event at the start of a CI job. This enables tooling based on this spec to adapt to specialised problem spaces without the user of such tools having to configure anything. It also allows automation frameworks such as test runners to integrate with such tools in a seamless way by registering their own specialised events.

This specification is intended to be implemented by both parsers that are wishing to extract insights from logs using this format, as well as automation tooling which wishes to output in this format.

## 3. Background

It is typically useful to be able to inspect a long running task in CI in a way which allows the user to quickly gain insights on the current progress or results of a CI job. CI systems typically provide a way to inspect the logs of a running or complete job, and those logs serve as a rudimentary way for the user to infer the status of the related job. Typically, with basic tooling, the user would need to spend time understanding the logs before reaching their conclusions.

By finding ways to increase observability of these jobs, the time spent reaching these conclusions can be reduced. For example, tooling could be produced in order for a user to be able to quickly peek into what tests within a job have passed or failed thus far.

This standard largely focuses on some common primitives to describe common events, whilst adopting a simplified log-centric approach to communicate those events, which is designed to lower the bar of entry to basic observability of jobs, and remove the absolute requirement for vendor lock-in and centralisation for the basic level of functionality. 

This approach has some unique advantages but there are trade-offs that should be considered. There are three primary methods of emitting such events to platforms which would consume such events and provide observability and monitoring tooling. 

### 3.1. File based collection

In this method, the user would typically configure, modify or extend their CI automation tooling to output a file in a predefined format. The CI platform itself would then be configured to ingest this file after the job is complete in order to display insights to the user. A common example of this is the [JUnit XML format](https://github.com/testmoapp/junitxml).

This method is highly accessible as the format is well documented. However, it is severely limited in that it can not communicate the status of jobs during execution. In addition, these files are typically privately managed by the CI platform, and may have limited visibility. 

### 3.2. Agent-based observability

These platforms typically use agents installed on the target systems to collect, aggregate, and send metrics to a centralised platform. In order to provide higher granularity, the user would typically hook into or modify their CI automation tooling (e.g. test runners) in order to create and send events direct from that orchestration software to the platform. Different software vendors may provide varying levels of support for common tooling within different language ecosystems.

Typically the format of the messages sent to the remote host is proprietary which promotes vendor lock-in. This visibility of those events to anyone with access to the job is also somewhat restricted in that (unless a sysadmin has intervened), other tooling can not freely access it and inspect it unless proprietary API access has been granted. The number of hops to reach a basic level of single-run observability is therefore increased.

However, these platforms are complete and inherently provide a way to maintain state between runs, allowing for deep analysis of long-term patterns. This specification does not attempt to solve those problems, but may be complementary to them.

### 3.3. Log-based observability

In this method, automation tooling is configured, modified or extended to output events to the logs. Some CI platforms provide a vendor-specific logging format that can be used and understood by that specific platform to provide built-in observability on that platform.

The log-based approach has many key advantages:

* Any automation tooling that can output to stdout is able to be modified to be compatible, even if it doesn't provide inherent pluggable mechanisms.
* Logs are almost always inherently visible in CI platforms to any user who can access the job, usually in a way which enables them to be viewed live either via CLI tooling, API or via the web browser. This makes them an attractive medium to communicate events, since any tooling only needs the same rights as the user who is viewing the logs. This theoretically allows for tooling inside of a web browser to be constructed to inspect events.

There are also key disadvantages and limitations:

* Logs are inherently linear, and so there is a level of complexity in representing concurrent processes that any standard needs to resolve.
* Whilst you can gain a good level of visibility by modifying automation tooling running *inside* the job, some tasks may actually only be possible to report reliably with modifications to the CI build configuration or platform itself.

It is preferable to develop an open format for these messages rather than relying on a vendor-specific format, in order to create an open ecosystem. This specification aims to fill this gap to the end of making possible simple and accessible tools to check the state of an individual job from the users/engineers local machine, which are not necessarily tied to any one overarching CI platform. That user only has to have access to change the automation tooling (e.g. test runner) which is typically just committed configuration or code in version control. However, it could also be used by centralised tools as a way to understand events across jobs, or use in tandem with such tooling.

#### 3.3.1. Issues preventing reuse of prior art

It is technically feasible to reuse a CI logging format in use by other products. However prior art around existing CI logging formats, to varying degrees:

* Tend to contain significant ambiguity in documentation around edge cases, with the canonical truth of how messages are handled often hidden behind closed-source applications.
* Have conceptual flaws such as the message order being conflated with the structure of test runner output which lead to complexity in supporting tooling.
* Have design decisions that lead to a greater risk of introducing hard-to-detect bugs in implementations.
* Use custom non-standard syntax which require parsers that would otherwise be unnecessary.
* Tend to be closely tied to unique functionality within a certain platform, meaning only a subset can be re-used for a standard that strives to be agnostic.
* Tend to lack extensibility.
* Bring about potential legal confusion of adopting such formats in other products.
* Are not properly versioned, and so interoperability is confused.

## 4. Specification



###  4.1. Signal Log Line

A **Signal Log Line** is a serialised representation of some some CI job event. It is the foundational concept in the specification. These log lines will typically be outputted by automation software such as test runners, and analysed by tooling compatible with this specification.

#### 4.1.1. Format

A **Signal Log Line** adopts the format:

```
/* OPEN_CI_SIGNAL <SIGNAL_JSON> */
```

##### 4.1.1.1. Syntactical requirements

To be classed syntactically valid **Signal Log Line** that are considered as such by parsers, they must:

* Always appear on one line.
  * Note: Messages do not have to start at the beginning of a line.
  * Note: Messages can appear on the same line as other messages.
* Never end prematurely.
* Be properly contained by the ident wrapper structure.
* Have valid JSON as the payload.

Text which does not conform to these rules should be considered as miscellaneous log text. Parsers may optionally choose to error on such scenarios where a likely mistake or corruption is detected, as per their own philosophy.

Note, *semantically* invalid signal log lines (i.e. it can be parsed but the containing JSON does not meet the necessary constraints) are dealt with later in this spec.


##### 4.1.1.2. Ident wrapper

`/* OPEN_CI_SIGNAL` serves as the identifier (the *indent*) by parsers to identify that the the next token is a message JSON payload. `*/` serves as the identifier that the message has ended. 

Parsers will treat any text outside of this ident wrapper as non-message text.

##### 4.1.1.3. Signal JSON

`<SIGNAL_JSON>` is a properly serialised JSON string according to the [RFC8259](https://www.rfc-editor.org/rfc/rfc8259) specification. As above, an additional constraint is imposed that this JSON string must not spread over multiple lines.

The structure of `<SIGNAL_JSON>` will either represent a **MessageTypeDefinition** or a [**Message**](#142-message), the overall valid structure of which is defined in the [JSON Schema](./schemas/json-schema.json).

### 4.2. Message Types

 TODO

#### 4.2.1. Built-in Message Types
 TODO

### 4.3. Message

A **Message** is a representation, in JSON format, of a particular event that occurred during a CI jobs execution. The structure of all messages must comply with the definition of the common message properties.

Each **Message** links to a [**Message Type**](#42-message-types), which also may define a stricter definition that demands certain fields from the common message properties are required.

If the type of a message is not one of the built-in types, then it must have been earlier defined by a **MessageTypeDefinition** within the same log.

#### 4.3.1. Common message properties

| Property                       | Type   | Optional* | Default | Summary                                                                                                       |
| ------------------------------ | ------ | --------- | ------- | ------------------------------------------------------------------------------------------------------------- |
| [`id`](#id)                    | string | No        | *N/A*   | Identifier that is unique within this execution, of this event.                                               |
| [`executionId`](#execution-id) | string | Yes       | `null`  | Identifier for execution flow that this event was executed from, to support better concurrency observability. |
| [`type`](#type)                | string | No        | *N/A*   | Type of event defined by `<namespace>:<eventName>`                                                            |
| [`timestamp`](#timestamp)      | string | Yes       | *N/A*   | ISO 8601 date string of when the event occurred.                                                              |
| [`links`](#links)              | object | Yes       | `{}`    | References to related **Message**'s.                                                                          |
| [`payload`](#payload)          | object | No        | *N/A*   | The contents of the event, according to its defined type.                                                     |


*\* may be required by specific event types, either built in or custom defined by **MessageTypeDefinition**. This indicates only if its possible for any event type to exist where this property is not defined*

##### 4.3.1.1. ID

The ID is an ID that uniquely identifies (globally in this log) an event. IDs can be any string. A basic implementation may use UUIDs, but it is also acceptable to use something contextual and stable to the event provided that this is globally unique. Using stable identifiers could ensure easier cross referencing between CI job runs of this event, if using this specification in the context of a system that requires that.

**It is therefore highly preferable for log event producers to consider using a stable ID, which may require users of the tool in question to abide by certain conventions.** E.g. a concatenation of `test suite` and `test name` could be used, but it would require recommendations to the test framework user to not repeat test + suite names. However, this is  not explicitly required by the specification, as such stable IDs can conceivably be inferred from the [`payload`](#4216-payload) contents. But this requires additional effort on the side of the consuming tool.

> :warning: This part of the specification is liable to change.

##### 4.3.1.2. Execution ID

It is expected that CI jobs will often undertake tasks in parallel. For example, a test runner may execute multiple tests at the same time. This means the *order* of the events is explicitly ignored in the Open CI Signals specification, and parsers should not take order into account.

Instead, the Open CI Signals specification utilises globally unique identifiers and hard references between messages to remove ambiguity during concurrent processes, no matter what flow/thread/process they emitted  from.

This can make implementing tools that output into this format easier. However, it can also obscure the execution flow of the events if wanting to debug what specific thread or process an event was emitted from, or group events by their thread or process.

Therefore, an Execution ID can optionally be defined on all messages. If an execution ID has been "seen" by the parser for the first time, it is registered by the consumer, and future uses of that execution ID will consider those messages as part of the same execution flow/thread/process.

Execution IDs can be inferred for a message if the `executionId` is unset and a [`link`](#4313-links) has been defined that once resolved, eventually leads to a related ancestor which *does* have an `executionId` set. See [`parent`](#43131-parent) and [`resolves`](#4313-links).

##### 4.3.1.3. Type

A string that represents a valid message type. Types are of the format:

```
<namespace>:<eventName>
```

The `<namespace>` is used as a way to delimit dynamically registered message types from the built-in types which exist on the `ocis` namespace. 

The entire type string, `<namespace>:<eventName>` is either a custom event registered by a **MessageTypeDefinition**, or one of the predefined built-in events on the `ocis` namespace.

The type infers certain constraints and rules that parsers must consider. For example, a `ocis:testFinished` message must have a [`resolves`](#4213-parent).

##### 4.3.1.4. Timestamp

A [ISO 8601](https://www.iso.org/iso-8601-date-and-time-format.html) date time string which represents when the event occurred, which can be optionally provided.

Note this does not influence the parser's understanding of the event structure, but may be used by tooling for inferences on measuring temporal metrics, and is also useful for auditing purposes. It is is strongly recommended that this is included on all messages.

If used in a CI job which runs across multiple agents, please ensure your agents have reasonably sync'ed time, to prevent erroneous results.

##### 4.3.1.5. Links

Links is an object that contains properties that pertain to how a message interacts with other messages previous to the one in question. Messages have certain constraints on what type of message can be related to this one, and those are defined by the specific event type.

| Property                      | Type   | Optional* | Default | Summary                                                                                                        |
| ----------------------------- | ------ | --------- | ------- | -------------------------------------------------------------------------------------------------------------- |
| [`parent`](#43131-parent)     | string | Yes       | `null`  | If set, defines that this message should be considered to be a child of the message that this ID references.   |
| [`resolves`](#42142-resolves) | string | Yes       | `null`  | If set, defines that this message will close the block that was opened by the message that this ID references. |

*\* may be required by specific event types, either built in or custom defined by **MessageTypeDefinition**. This indicates only if its possible for any event type to exist where this property is not defined*

###### 4.3.1.5.1. Parent

If this is present, the message that is referenced by the ID inside of the value will be used by consuming tooling in order to indicate this message is regarded as a child of that referenced message.

If no [Execution ID](#4212-execution-id) has been explicitly set on this message, and it has been set on the parent message, or its ancestors, the `executionId` of the nearest ancestor will be adopted.

###### 4.3.1.5.2. Resolves

If this is present, the message that is referenced by the ID inside of the value will be used by consuming tooling in order to indicate this message is regarded as a child of that referenced message.

If no [Execution ID](#4212-execution-id) has been explicitly set on this message, and it has been set on the message that is being resolved, or its [`parent`](#42151-parent) ancestors, the [Execution ID](#4212-execution-id) of the nearest ancestor will be adopted.

##### 4.3.1.6. Payload

The payload contains the JSON object that matches the defined type of the message.

## 5. Parser behaviour

TODO



