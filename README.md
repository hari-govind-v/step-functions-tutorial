# AWS Step Functions — Tutorial Session

## The Problem

You're building a feature: a user places an order. You need to:

1. Validate the order
2. Charge the customer
3. Reserve inventory
4. Notify the warehouse
5. Send a confirmation email

Each of these calls a different service. What happens when step 3 fails after step 2 already succeeded? How do you retry? How do you track which step ran? How do you handle timeouts?

Without orchestration, you end up writing:
- Lambda functions calling other Lambda functions
- DynamoDB tables tracking workflow state manually
- Custom retry logic scattered across services
- No visibility into what happened when something breaks

This is the coordination problem. AWS Step Functions is built to solve it.

---

## What is AWS Step Functions?

> AWS Step Functions is a **serverless workflow orchestration service** that lets you coordinate multiple AWS services into a sequence of steps — with built-in error handling, retries, state management, and execution history.

- Workflows are called **state machines**
- Each step is a **state**
- Defined in **Amazon States Language (ASL)** — JSON
- Visual drag-and-drop editor: **Workflow Studio**
- Integrates with **220+ AWS services** — Lambda, S3, DynamoDB, SQS, SNS, Glue, SageMaker, Bedrock, ECS, and more
- Full execution history and visual debugging in the AWS console

---

## Where is it Used?

Step Functions shows up wherever a business process has multiple steps, decisions, retries, or hand-offs between services:

| Domain | Example |
|---|---|
| E-commerce | Order processing, refund workflows |
| Media | Video transcoding, image processing pipelines |
| Finance | Loan approval, fraud detection, payment workflows |
| ML/Data | Model training pipelines, ETL, data validation |
| DevOps | CI/CD pipelines, infrastructure automation |
| Healthcare | Patient onboarding, claims processing |
| SaaS | Multi-tenant provisioning, scheduled jobs |
| Security | Incident response automation |

---

## Why Not Just Lambda Chains?

The natural alternative is Lambda calling Lambda — but that approach pushes coordination responsibility into your application code. You end up owning retry logic, progress tracking, timeout handling, and failure visibility yourself. Step Functions makes these declarative:

| Problem | Lambda chains | Step Functions |
|---|---|---|
| Track workflow progress | Custom DynamoDB table | Built-in execution history |
| Retry failed steps | Write retry logic manually | Declarative `Retry` config |
| Branch based on conditions | `if/else` in code + routing | `Choice` state |
| Run steps in parallel | Async invocations + coordination code | `Parallel` state |
| Wait for human approval | Custom polling + tokens in DB | `.waitForTaskToken` |
| Visibility into failures | CloudWatch logs + manual tracing | Visual graph in console |
| Timeout a stuck step | Custom timers | `TimeoutSeconds` on any state |

**Bottom line:** Step Functions handles control flow, retries, state, and visibility so your code only handles business logic.

---

## Real-Life Example: Order Processing

Before writing your own, it helps to read a real one. This is a Step Function that processes an e-commerce order end-to-end.

### What it does

```
[Start]
   │
   ▼
ValidateOrder          ← Lambda: checks stock, validates address
   │
   ▼
ChargeCustomer         ← Lambda: calls payment gateway
   │
   ├── (payment fails) → RefundAndNotify → [End: Fail]
   │
   ▼
ReserveInventory       ← Lambda: locks stock in warehouse system
   │
   ▼
NotifyWarehouse        ← SNS: sends pick-and-pack instruction
   │
   ▼
WaitForWarehouseAck    ← Pauses here, waits for warehouse to callback
   │
   ▼
SendConfirmationEmail  ← SES: sends order confirmation to customer
   │
   ▼
[End: Success]
```

### The ASL definition

```json
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {

    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ValidateOrder",
      "Next": "ChargeCustomer"
    },

    "ChargeCustomer": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ChargeCustomer",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["PaymentDeclinedError"],
          "Next": "RefundAndNotify"
        }
      ],
      "Next": "ReserveInventory"
    },

    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ReserveInventory",
      "Next": "NotifyWarehouse"
    },

    "NotifyWarehouse": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish.waitForTaskToken",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:WarehouseTopic",
        "Message": {
          "orderId.$": "$.orderId",
          "taskToken.$": "$$.Task.Token"
        }
      },
      "Next": "SendConfirmationEmail"
    },

    "SendConfirmationEmail": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:SendEmail",
      "End": true
    },

    "RefundAndNotify": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:RefundAndNotify",
      "End": true
    }

  }
}
```

### What to notice

- **Every state** has either `"Next"` or `"End": true`
- **ChargeCustomer** has both `Retry` (transient errors) and `Catch` (business logic errors)
- **NotifyWarehouse** uses `.waitForTaskToken` — execution pauses until the warehouse sends the token back
- Data flows as JSON — `$.orderId` reads from the current state's input
- `$$.Task.Token` is injected by Step Functions — it's the callback token for this specific execution

---

## Tutorial — Build Your Own

Now that you've seen what a state machine looks like end-to-end, let's cover how to build one and everything the language supports.

---

### Workflow Type: Standard vs Express

The first decision when creating a state machine is the workflow type. **You can't change it after creation.**

| | Standard | Express |
|---|---|---|
| Max duration | **1 year** | **5 minutes** |
| Execution guarantee | **Exactly-once** | **At-least-once** |
| Max execution rate | 2,000 / sec | 100,000 / sec |
| Execution history | Full history in console | CloudWatch Logs only |
| Integration patterns | Request Response, `.sync`, `.waitForTaskToken` | Request Response only |

**Pick Standard when:** the workflow can run longer than 5 minutes, you need exactly-once semantics (payments, inventory, anything that can't run twice), you need an audit trail, or you use `.sync` / `.waitForTaskToken`. This also makes Standard the right choice when integrating with long-running AWS services like Glue jobs or ECS tasks — `.sync` polls until the job finishes, which Express can't do (Express has no `.sync` support and must complete within 5 minutes anyway).

**Pick Express when:** you're handling high-volume, short-lived events (IoT, streaming, API fan-outs) where throughput and cost per execution matter more than an audit trail.

---

**Real-world example — Standard:** An e-commerce order fulfillment workflow. When a customer places an order, the workflow runs: charge card → reserve inventory → notify warehouse → send confirmation email. This can take hours if the warehouse integration is slow, and a duplicate charge or double-shipment is catastrophic. Standard is the only choice here — exactly-once guarantee, full audit trail for finance, and `.waitForTaskToken` to pause while the warehouse system confirms the pick.

```
  [Order Placed]
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  Charge Card          (Lambda .waitForTaskToken)                │
│  • sends taskToken to payment gateway                           │
│  • workflow PAUSES until gateway calls SendTaskSuccess          │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  Reserve Inventory    (DynamoDB .sync)                          │
│  • decrements stock atomically                                  │
│  • workflow waits for DynamoDB write confirmation               │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  Notify Warehouse     (SQS .waitForTaskToken)                   │
│  • drops message + taskToken into warehouse queue               │
│  • workflow PAUSES — could be hours — until worker calls back   │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  Send Confirmation    (SNS publish)                             │
│  • fires email/SMS to customer                                  │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
  [Order Complete]
```

Standard was chosen because: (1) `.waitForTaskToken` is required for the warehouse pause — Express doesn't support it; (2) charging a card must be exactly-once — Express's at-least-once guarantee could cause a double charge on retry; (3) the audit trail is needed for dispute resolution.

---

**Real-world example — Express:** A media processing pipeline triggered by S3 uploads. Every time a video is uploaded, the workflow runs: extract metadata → generate thumbnails → update search index. Thousands of uploads per minute, each step completes in seconds, and if a thumbnail job runs twice it just overwrites itself — harmless. Express is the right fit here.

```
  [S3 Upload Event]
        │
        ▼
┌───────────────────────────────────────────────────────┐
│  Extract Metadata     (Lambda)                        │
│  • reads video file, pulls duration + resolution      │
│  • completes in ~1s                                   │
└───────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────┐
│  Generate Thumbnails  (Lambda)                        │
│  • creates thumbnail images at multiple sizes         │
│  • idempotent — safe to run twice                     │
│  • completes in ~3s                                   │
└───────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────┐
│  Update Search Index  (Lambda)                        │
│  • upserts video metadata into search index           │
│  • idempotent — safe to run twice                     │
│  • completes in ~1s                                   │
└───────────────────────────────────────────────────────┘
        │
        ▼
  [Done — ~5s total]
```

Express was chosen because: (1) the whole pipeline finishes in under 30 seconds — well within the 5-minute cap; (2) at high upload volume (thousands/min), Standard's 2,000 executions/sec limit would become a bottleneck; (3) all steps are idempotent — running twice produces the same result, so at-least-once delivery is safe; (4) there's no need for `.sync` or `.waitForTaskToken`; (5) cost per execution is ~10× lower than Standard at this scale.

---

#### Redrive

If a Standard execution fails, you can resume it from the failed state — without re-running the states that already completed:

```bash
aws stepfunctions redrive-execution \
  --execution-arn "arn:aws:states:...:execution:OrderWorkflow:exec-123"
```

Or in the console: open the failed execution → click **Redrive**.

Redrive uses the current state machine definition and jumps directly to the failed state by name:

| Scenario | Behavior |
|---|---|
| Fix Lambda behind the failed state | Resumes correctly — intended use case |
| Add state after the failed state | New state runs normally |
| Add state before the failed state | New state is **skipped** — jumps straight to failed state |
| Rename or delete the failed state | Redrive errors — can't find state by name |

To re-run from the beginning, you must start a new execution. There's no way to restart an existing one from `StartAt`.

---

### Creating a State Machine

**In the console:**
1. Go to **AWS Step Functions** → **State machines** → **Create state machine**
2. Choose **Workflow Studio** (visual drag-and-drop) or write ASL directly
3. Select workflow type and set an IAM role (Step Functions needs permission to call Lambda, SNS, etc.)
4. Create → run an execution with a JSON input

---

### State Machines

A state machine is the top-level workflow definition:

| Field | Description |
|---|---|
| `Comment` | Human-readable description |
| `StartAt` | Name of the first state to run |
| `States` | Map of all state definitions |
| `QueryLanguage` | `JSONPath` (default) or `JSONata` for expressions |

```json
{
  "Comment": "My workflow",
  "StartAt": "FirstStep",
  "States": {
    "FirstStep": { ... },
    "SecondStep": { ... }
  }
}
```

One run of a state machine is an **execution**. Each execution has a unique ID, carries its own JSON data independently of all other executions, and has its own history. You can run thousands of executions of the same state machine simultaneously.

---

### States

States are the individual steps in your workflow. Each state has a `Type` and either a `Next` pointing to the next state, or `End: true` to terminate. There are eight types.

---

#### Task
Calls an AWS service. The primary "do work" state.

```json
"ProcessPayment": {
  "Type": "Task",
  "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ProcessPayment",
  "TimeoutSeconds": 30,
  "Next": "SendReceipt"
}
```

Can call Lambda, Glue, SageMaker, DynamoDB, SQS, SNS, Bedrock, and 200+ more services.

---

#### Choice
Branches based on conditions — like an `if/else`. Choice states have no top-level `Next` — routing is defined inside `Choices` and `Default`.

```json
"CheckOrderValue": {
  "Type": "Choice",
  "Choices": [
    {
      "Condition": "{% $orderValue > 1000 %}",
      "Next": "RequireApproval"
    },
    {
      "Condition": "{% $orderValue <= 1000 %}",
      "Next": "AutoApprove"
    }
  ],
  "Default": "RejectOrder"
}
```

---

#### Parallel
Runs multiple branches **simultaneously** and waits for all of them to finish before moving on.

```json
"ProcessMediaFile": {
  "Type": "Parallel",
  "Branches": [
    {
      "StartAt": "GenerateThumbnail",
      "States": { "GenerateThumbnail": { "Type": "Task", "Resource": "...", "End": true } }
    },
    {
      "StartAt": "ExtractMetadata",
      "States": { "ExtractMetadata": { "Type": "Task", "Resource": "...", "End": true } }
    },
    {
      "StartAt": "RunModerationCheck",
      "States": { "RunModerationCheck": { "Type": "Task", "Resource": "...", "End": true } }
    }
  ],
  "Next": "StoreAllResults"
}
```

Output is an **array** where each element is the output of one branch.

Max **40 branches** per `Parallel` state. If you need more than 10–15 parallel workloads, prefer `Map` — it fans out dynamically over an array with no hardcoded branch limit.

---

#### Map
Runs the same steps for **each item in an array** — iterations run in parallel.

```json
"ProcessLineItems": {
  "Type": "Map",
  "ItemsPath": "$.lineItems",
  "MaxConcurrency": 10,
  "Iterator": {
    "StartAt": "ProcessOneItem",
    "States": {
      "ProcessOneItem": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:...",
        "End": true
      }
    }
  },
  "Next": "AggregateResults"
}
```

`MaxConcurrency` controls how many items run at once. `0` = unlimited.

**Distributed Map** is a variant that can process **millions of items** from S3 or SQS at scale.

---

#### Wait
Pauses execution for a fixed duration or until a specific timestamp.

```json
"WaitOneDay": {
  "Type": "Wait",
  "Seconds": 86400,
  "Next": "CheckDeliveryStatus"
}
```

Or until a timestamp from the input:

```json
"WaitUntilScheduled": {
  "Type": "Wait",
  "TimestampPath": "$.scheduledAt",
  "Next": "RunJob"
}
```

---

#### Pass
Passes input to output unchanged. Can inject or transform data without calling any service — useful for testing or setting defaults.

```json
"InjectDefaults": {
  "Type": "Pass",
  "Result": { "currency": "USD", "region": "us-east-1" },
  "ResultPath": "$.defaults",
  "Next": "ProcessOrder"
}
```

---

#### Succeed
Terminates the execution successfully.

```json
"Done": {
  "Type": "Succeed"
}
```

---

#### Fail
Terminates the execution with an error.

```json
"WorkflowFailed": {
  "Type": "Fail",
  "Error": "ValidationError",
  "Cause": "Order validation failed — missing required fields"
}
```

---

### State Transitions

Every non-terminal state must declare where it goes next. There are four transition forms.

**Linear** — move to a specific state when done:
```json
"StateA": {
  "Type": "Task",
  "Resource": "...",
  "Next": "StateB"
}
```

**Terminal** — end the execution:
```json
"FinalState": {
  "Type": "Task",
  "Resource": "...",
  "End": true
}
```

**Conditional** — branch based on data:
```json
"RouteRequest": {
  "Type": "Choice",
  "Choices": [
    { "Condition": "{% $type = 'premium' %}", "Next": "PremiumFlow" },
    { "Condition": "{% $type = 'standard' %}", "Next": "StandardFlow" }
  ],
  "Default": "RejectRequest"
}
```

**Error** — route to a fallback state on failure:
```json
"RiskyTask": {
  "Type": "Task",
  "Resource": "...",
  "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "ErrorHandler" }],
  "Next": "HappyPath"
}
```

Rules:
- State names must be **unique** within the state machine
- `Choice` states use `Choices[].Next` and `Default` — no top-level `Next`
- Circular transitions (loops) are allowed — use `Choice` or `Wait` to break out

---

### Passing Data Between States

Step Functions passes a **single JSON document** through the workflow. Each state receives it as input and produces output. You control exactly what each state sees and what it contributes.

```
Execution Input → State 1 → State 1 Output → State 2 → State 2 Output → ...
```

#### Input and Output Control

**`InputPath`** — filter what the state receives:
```json
"InputPath": "$.order"   ← state only sees the order sub-object
```

**`ResultPath`** — where to store the state's output. Three behaviors:

```
No ResultPath        → output replaces the entire input (default)
"ResultPath": "$.x"  → output merged at $.x, original input preserved
"ResultPath": null   → output discarded, original input passed through unchanged
```

Example:
```
Input:  { "orderId": "123", "amount": 50 }

No ResultPath   → { "chargeId": "ch_abc" }                          ← orderId/amount gone
"$.chargeResult"→ { "orderId": "123", "amount": 50, "chargeResult": { "chargeId": "ch_abc" } }
null            → { "orderId": "123", "amount": 50 }                 ← state ran, output thrown away
```

**`OutputPath`** — filter what gets passed to the next state:
```json
"OutputPath": "$.paymentResult.transactionId"   ← only this goes forward
```

**`Parameters`** — transform input before sending it to the resource. Keys ending in `.$` are dynamic (read from input); keys without `.$` are static:
```json
"Parameters": {
  "TopicArn": "arn:aws:sns:...",
  "Message.$": "$.confirmationMessage",   ← dynamic
  "Subject": "Order Confirmed"            ← static
}
```

#### The Accumulation Pattern

The standard pattern for multi-step workflows: use `ResultPath` on every state to **accumulate** data rather than replace it. By the end of the workflow, every state's output is available in the document:

```
Input:  { "orderId": "ORD-123", "customer": { "email": "a@b.com" }, "amount": 500 }

ValidateOrder → ResultPath: "$.validation"
→ { "orderId": "ORD-123", ..., "validation": { "valid": true } }

ChargeCustomer → ResultPath: "$.payment"
→ { ..., "validation": {...}, "payment": { "transactionId": "TXN-456", "status": "success" } }

SendEmail sees the full enriched object — all prior step outputs available
```

#### Execution Context Variables

Beyond the input (`$`), Step Functions injects a context object (`$$`) with metadata about the running execution:

```json
"$$.Execution.Id"        ← full execution ARN
"$$.Execution.Name"      ← execution name
"$$.Execution.StartTime" ← ISO timestamp when execution started
"$$.State.Name"          ← current state name
"$$.Task.Token"          ← task token (for waitForTaskToken)
```

Use `$$` anywhere you need execution-level metadata — for example, passing the execution ID as a correlation ID to a downstream service.

#### Variables (JSONata mode)

JSONPath reads from the current input only. If you need a value from early in the workflow to be available many steps later, you'd have to carry it through every `ResultPath` in between.

`Assign` solves this by letting you set named variables that persist across all states for the lifetime of the execution:

```json
"ExtractOrderId": {
  "Type": "Pass",
  "Assign": {
    "orderId": "{% $states.input.order.id %}"
  },
  "Next": "ProcessOrder"
}
```

Later states reference `$orderId` directly — no need to thread it through intermediate states.

To use JSONata (and `Assign`), set `"QueryLanguage": "JSONata"` at the state machine level, or per-state to mix modes:

```json
{
  "QueryLanguage": "JSONata",
  "StartAt": "FirstStep",
  "States": { ... }
}
```

---

### Error Handling

Step Functions has three error handling mechanisms: **Retry**, **Catch**, and **Timeouts**.

#### Retry

Automatically retries a failed state with configurable backoff. Multiple retry rules can be stacked — they're evaluated top to bottom and the first match wins.

```json
"CallExternalAPI": {
  "Type": "Task",
  "Resource": "arn:aws:lambda:...",
  "Retry": [
    {
      "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException"],
      "IntervalSeconds": 1,
      "MaxAttempts": 3,
      "BackoffRate": 2.0,
      "MaxDelaySeconds": 30
    },
    {
      "ErrorEquals": ["States.TaskFailed"],
      "IntervalSeconds": 5,
      "MaxAttempts": 2,
      "BackoffRate": 1.5
    }
  ],
  "Next": "ProcessResult"
}
```

| Field | Description |
|---|---|
| `ErrorEquals` | Error names to match |
| `IntervalSeconds` | Initial wait before first retry |
| `MaxAttempts` | Max retries (`0` = no retry) |
| `BackoffRate` | Multiplier applied to wait time each retry |
| `MaxDelaySeconds` | Cap on wait between retries |

#### Catch

When retries are exhausted — or no retry is defined — route to a fallback state. `ResultPath` merges the error details into the document so the fallback state can inspect what went wrong:

```json
"Catch": [
  {
    "ErrorEquals": ["PaymentDeclinedError"],
    "ResultPath": "$.errorInfo",
    "Next": "HandlePaymentFailure"
  },
  {
    "ErrorEquals": ["States.ALL"],
    "ResultPath": "$.errorInfo",
    "Next": "GenericErrorHandler"
  }
]
```

Catch rules are evaluated top to bottom — put specific errors before `States.ALL`.

#### Timeouts

Prevent stuck tasks from running forever. If no heartbeat arrives within `HeartbeatSeconds`, Step Functions raises `States.HeartbeatTimeout`.

```json
"LongRunningTask": {
  "Type": "Task",
  "Resource": "arn:aws:lambda:...",
  "TimeoutSeconds": 300,
  "HeartbeatSeconds": 60,
  "Next": "Done"
}
```

#### Built-in Error Names

| Error | When it occurs |
|---|---|
| `States.ALL` | Matches any error |
| `States.TaskFailed` | Task threw an exception |
| `States.Timeout` | Task exceeded `TimeoutSeconds` |
| `States.HeartbeatTimeout` | Task missed a heartbeat |
| `States.NoChoiceMatched` | No Choice rule matched and no `Default` |
| `States.ItemReaderFailed` | Map state failed reading items |
| `Lambda.ServiceException` | Lambda service error |

---

### Integration Patterns

Task states can call services in three ways, depending on whether you need to wait for the result.

#### Request Response (default)
Fire and move on. Step Functions calls the service and continues as soon as it gets HTTP 200 — it does not wait for the job to finish.

```json
"TriggerGlueJob": {
  "Type": "Task",
  "Resource": "arn:aws:states:::glue:startJobRun",
  "Parameters": { "JobName": "my-etl-job" },
  "Next": "NextStep"
}
```

#### Run a Job (.sync)
Call a service and wait for the job to **actually complete** before moving on.

```json
"RunGlueJob": {
  "Type": "Task",
  "Resource": "arn:aws:states:::glue:startJobRun.sync",
  "Parameters": { "JobName": "my-etl-job" },
  "Next": "NextStep"
}
```

Supported by: Glue, Batch, EMR, SageMaker, ECS/Fargate, CodeBuild, EKS, and nested Step Functions.

#### Wait for Callback (.waitForTaskToken)
Pause the execution until an external system sends back a task token. The execution is suspended — not polling — so you're not charged while waiting.

```json
"WaitForHumanApproval": {
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
  "Parameters": {
    "QueueUrl": "https://sqs.../approval-queue",
    "MessageBody": {
      "taskToken.$": "$$.Task.Token",
      "orderId.$": "$.orderId"
    }
  },
  "Next": "ProcessApproval"
}
```

The external system resumes the execution by calling:

```bash
aws stepfunctions send-task-success \
  --task-token "TOKEN_FROM_MESSAGE" \
  --task-output '{"approved": true}'
```

---

## Quick Reference

```
State Types:
  Task       → call a service / Lambda
  Choice     → conditional branching (if/else)
  Parallel   → run branches simultaneously, wait for all
  Map        → iterate over array items in parallel
  Wait       → pause for time / until timestamp
  Pass       → pass-through, inject or transform data
  Succeed    → end successfully
  Fail       → end with error

Data control:
  InputPath     → filter state's input
  ResultPath    → where to store output (merge vs replace)
  OutputPath    → filter what goes to next state
  Parameters    → transform before calling resource
  Assign        → set named variables (JSONata mode)
  $$            → execution context (ID, name, task token)

Error handling:
  Retry             → auto-retry with exponential backoff
  Catch             → fallback state on exhausted retries
  TimeoutSeconds    → kill stuck tasks
  HeartbeatSeconds  → kill tasks that stop sending heartbeats

Workflow types:
  Standard   → up to 1 year, exactly-once, full audit trail
  Express    → up to 5 min, at-least-once, high throughput

Integration patterns:
  (default)           → fire and move on
  .sync               → wait for job to complete
  .waitForTaskToken   → suspend until external callback
```

---

*Next: Demo — build a Step Function from scratch in the console.*
