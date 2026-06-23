# 8917-Assignment 1

**Student Name**: Faiza Boudehane
**Student ID**: 041273470
**Course**: CST8917-Serverless
**Semester**: Spring 2026

---
## PART 1 :

1.	What is the central thesis of the paper? What do the authors mean by "one step forward, two steps back"?
•	One step forward is the autoscaling provided by FaaS in cloud programming. The two steps backward come from ignoring the importance of efficient data processing and from hindering the development of distributed systems.

2.	Key Limitations Identified: Summarize the specific limitations the authors identify with first-generation serverless/FaaS platforms. Your summary should cover:
o	Execution time constraint : given that a function has a fixed execution time—15 minutes for AWS—its execution stops once that limit is reached. Lambda can keep the function’s state in cache on the VM, but nothing guarantees that future invocations of the same function will run on that same VM. Therefore, the developer must take this into account when designing the function.
o	Communication and network limitations (I/O bottleneck, lack of direct addressability): because data must be transferred between storage and the Lambda function, and because bandwidth is limited—and functions from the same user share that same bandwidth—the proportion of bandwidth available to each function decreases significantly as compute power increases. 
o	The "data shipping" anti-pattern (moving data to code vs. code to data): as already seen with the addressability issue, data must be transferred to the code during the execution of a function. This goes against normal practice and results in added latency, additional cost, and bandwidth-related problems.
o	Limited hardware access (no GPUs or specialized hardware): in this current generation of functions, it is only possible to reserve CPU and RAM, but there is no way to access specialized hardware, even though such hardware is expected to accelerate significantly in the coming years. This limitation also hinders the Big Data ecosystem, which becomes constrained by the RAM limits imposed per Lambda instance.
o	Challenges for distributed computing and stateful workloads: the major issue for distributed systems is that they rely on communication protocols between agents, whereas function instances can communicate only through storage. Adding to this, at the end of each function cycle, and in order to ensure communication between functions, they must repeatedly perform leader election. This makes the connection slow and costly as well.
3.	Proposed Future Directions: What characteristics do the authors propose for future cloud programming? Mention at least three of their proposals :
•	Smooth placement of code and data: this highlights the importance of sending the code to the data, rather than the opposite, as is currently the case in FaaS.
•	Support for heterogeneous hardware: this highlights the importance of allowing the platform to make dynamic physical decisions regarding the allocation of code to different resources, based on programmer requirements or information extracted from the code.    
•	Addressable long lived virtual agents: the platform should amortize the affinity costs between code and data across multiple requests. This would encourage developers to create software agents that are not ephemeral.
## PART 2 :
• In Azure Durable Functions, client, orchestrator, and activity functions work together to create stateful workflows on top of a serverless platform. A client function starts and manages workflow instances. The orchestrator function defines the workflow in code by scheduling tasks, waiting for results, and controlling execution flow. Activity functions perform the actual work, such as calling APIs or writing to databases. The Durable runtime coordinates these components through event sourcing and checkpoints, allowing long running, reliable workflows.

This differs from basic FaaS, where functions are stateless, short lived, and communicate only through external storage like queues or databases. Durable Functions introduces a built in workflow engine, enabling stateful, deterministic, multi step processes without manually wiring services together.

•	Azure Durable Functions manages state using a combination of event sourcing, checkpointing, and deterministic replay. Every significant action in an orchestration—such as scheduling an activity, receiving a result, or waiting on a timer—is recorded as an event in durable storage. When the orchestrator reaches an await, the runtime creates a checkpoint, persisting progress so the workflow can safely unload. If the function app restarts or scales, the orchestrator is reconstructed through replay, where the runtime re-executes the orchestrator code using the stored event history to restore all variables and control flow.
This model directly addresses the paper’s criticism that serverless functions are inherently stateless. Durable Functions provides the illusion of a long running, stateful workflow while the runtime handles persistence, recovery, and reliability—something basic FaaS platforms do not natively support.

•	Orchestrator functions in Azure Durable Functions avoid normal Azure Function timeout limits because they never run continuously. Instead, an orchestrator executes only in short bursts: when it schedules an activity, waits for a timer, or pauses for an external event, the Durable runtime persists a checkpoint and unloads the orchestrator. Since it is not actively running, it does not accumulate execution time and can therefore support workflows that last hours, days, or even months. When progress resumes, the orchestrator is replayed from history, restoring its state without violating timeout rules.
Activity functions, however, are still regular Azure Functions. They remain subject to the platform’s execution limits—such as the Consumption plan’s typical 5–10 minute timeout, memory constraints, and cold start behavior. Long-running work must therefore be broken into smaller activities or moved to more suitable compute services.

•	In Durable Functions, orchestrator and activity functions communicate through the Durable Task Framework, which uses event sourcing to record scheduled tasks and durable queues to deliver work items to activity workers. When an orchestrator calls an activity, it does not invoke it directly; instead, the runtime writes a task message to durable storage. An activity function picks up this message, executes, and writes back a completion event. During replay, the orchestrator reads this event from history and resumes with the activity’s result.
Although this mechanism still relies on durable storage, it abstracts away the manual wiring of queues, blobs, or databases that basic FaaS requires. This directly addresses the paper’s criticism: developers no longer manage slow intermediaries themselves. The runtime handles reliability, correlation, and ordering, giving the developer a clean, code first workflow model while still ensuring durable, fault tolerant communication.

•	In Durable Functions, the fan out/fan in pattern allows an orchestrator to run many activity functions in parallel and then wait for all of them to complete before continuing. The orchestrator “fans out” by scheduling multiple activities at once, each becoming an independent work item processed by the Durable Task Framework. While these activities run concurrently, the orchestrator is unloaded. When results arrive, the orchestrator is replayed and performs the “fan in” step using constructs like Task. When All to aggregate outputs. This pattern directly addresses the paper’s concern about distributed computing in FaaS, where developers must manually coordinate parallel tasks using queues or storage. Durable Functions abstracts this complexity: the runtime handles correlation, ordering, retries, and aggregation. Developers write simple, sequential code while the platform manages the distributed execution behind the scenes, providing a reliable and scalable parallel workflow model.
## PART 3:
### 1.	Limitations that remain unresolved:
•	Data Locality — Durable Functions Still Moves Data to the Code
The first limitation that persists in the Function as a Service (FaaS) model is that it requires data to be transferred from storage to the function. It also relies on limited bandwidth shared across functions, which can become a significant issue during periods of high function invocation. This results in increased latency and costs, as well as poor efficiency for Big Data workloads.
This is known as the "send data to code" problem.
Does Durable Functions Solve This Problem?
No. Durable Functions does not change the underlying data movement model.
Durable Functions adds an orchestration layer, but Activity Functions still execute within the same FaaS environment as traditional Azure Functions. As a result, the execution model remains unchanged:
•	Functions still retrieve data from external storage services (Cosmos DB, Blob Storage, SQL databases, etc.). 
•	Functions still have no control over the physical placement of compute resources relative to the data. 
•	Functions still experience bandwidth limitations and incur the costs associated with accessing external data sources. 
Durable Functions is built on top of Azure Functions and therefore inherits the same execution model, which consists of:
•	Ephemeral compute 
•	Stateless Activity Functions 
•	External storage as the sole communication mechanism 
While Durable Functions improves workflow orchestration, it does not alter the fundamental physical architecture of the FaaS model.

•	Lack of Access to Heterogeneous Hardware — Durable Functions Still Cannot Use GPUs, FPGAs, or Specialized Accelerators
  Current FaaS platforms:
	Only allow CPU and memory (RAM) resource allocation. 
	Do not provide access to specialized hardware. 
	Limit the execution of Big Data and Machine Learning workloads. 
	Risk becoming less relevant as hardware accelerators become increasingly essential. 
	Durable Functions does not provide access to GPUs or other specialized hardware. It only orchestrates Activity Functions and does not change the underlying compute capabilities of Azure Functions:
	No GPU support. 
	No FPGA support. 
	No high-memory instances. 
	No hardware affinity. 
	This hardware limitation persists because Azure Functions was designed for:
	Short-lived workloads. 
	Stateless functions. 
	General-purpose processing. 
	Cost-efficient and elastic compute. 
	Sandboxed environments. 
	Adding heterogeneous hardware such as GPUs or FPGAs would conflict with the economic and operational model of serverless computing.
	Durable Functions cannot address this limitation because it is not a new compute model. It is simply a workflow orchestration layer built on top of the existing Azure Functions execution model.
### 2.	Verdict: 
Azure Durable Functions is an important technical improvement, but it does not represent the architectural revolution called for by the authors. It addresses certain symptoms, such as execution time limitations, state management, and process coordination. However, it unfortunately does not tackle the fundamental causes, such as data locality, access to specialized hardware, and the inherent constraints of distributed systems.


## References

Microsoft. (n.d.). Durable Functions orchestrations. Microsoft Learn. https://learn.microsoft.com/fr-ca/azure/durable-task/common/durable-task-orchestrations?tabs=csharp&pivots=durable-functions

Microsoft. (n.d.). Durable Functions bindings. Microsoft Learn. https://learn.microsoft.com/fr-ca/azure/durable-task/durable-functions/durable-functions-bindings?tabs=in-process&pivots=programming-language-python

Microsoft. (n.d.). Durable Functions overview. Microsoft Learn. https://learn.microsoft.com/fr-ca/azure/durable-task/durable-functions/durable-functions-overview

Microsoft. (n.d.). Durable Functions code constraints. Microsoft Learn. https://learn.microsoft.com/fr-ca/azure/durable-task/common/durable-task-code-constraints?tabs=csharp&pivots=durable-functions

Microsoft. (n.d.). Durable Functions overview: Fan-out/fan-in. Microsoft Learn. https://learn.microsoft.com/fr-ca/azure/durable-task/durable-functions/durable-functions-overview#fan-outfan-in




