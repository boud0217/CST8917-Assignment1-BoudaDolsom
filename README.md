# Bouda Dolsom
# 041246719
# CST8917
# Serverless Computing - Critical Analysis
# 13/02/2026
# Paper Summary
The paper's central thesis is that first-generation serverless/FaaS platforms (like AWS Lambda and similar) are a genuine step forward in terms of elasticity, autoscaling, and pay-as-you-go simplicity, but at the same time they regress on core capabilities needed for modern, data-centric and distributed systems. Hence the phrase "one step forward, two steps back": we gain operational convenience and automatic scaling, but lose expressiveness, efficiency, and architectural fit for serious cloud innovation, especially in data systems.
The authors argue that current serverless designs are misaligned with the dominant trends of large-scale data processing, distributed computing, and specialized hardware. Instead of unlocking the full potential of the cloud's "exabytes of storage and millions of cores", they constrain developers to a narrow, stateless, short-lived, and I/O-bottlenecked model that makes many important workloads awkward or inefficient.

## Key limitations of first-generation serverless/FaaS
- Functions are limited to short lifetimes. This makes long-running analytics, streaming, or iterative algorithms difficult or impossible without complex workarounds such as manual checkpointing and orchestration. The platform's autoscaling promise clashes with the reality that many real workloads are inherently long-lived or require sustained coordination.
- Functions are not directly addressable. They cannot easily communicate peer-to-peer. All interaction typically goes through external services (queues, storage, APIs), which introduces latency and complexity. Network bandwidth is limited and often asymmetric, turning communication into a bottleneck and making tightly coupled distributed protocols or low-latency coordination impractical on FaaS.
- The paper criticizes the pattern where data is repeatedly pulled from remote storage into short-lived functions. Instead of "moving code to data", serverless often forces "moving data to code", incurring high I/O costs, redundant deserialization, and poor cache locality. This is especially harmful for data-intensive workloads like analytics, machine learning, or OLTP-style operations that benefit from data locality and shared in-memory state.
- First-generation serverless hides the underlying hardware so thoroughly that developers cannot exploit GPUs, FPGAs, or other accelerators, nor can they control memory size, local storage, or network topology. This makes serverless a poor fit for modern ML workloads and other performance-sensitive applications that rely on specialized hardware or careful resource tuning.
- Because functions are stateless and ephemeral, any durable or shared state must live in external services (databases, key-value stores, logs). This separation complicates programming models for distributed algorithms, transactions, and coordination. Building systems like distributed databases, streaming engines, or graph processors on top of FaaS becomes heavy, as developers must manually stitch together state, concurrency control, and fault tolerance outside the function runtime.
Taken together, these constraints make current serverless offerings "a bad fit for cloud innovation and particularly bad for data systems innovation".
## Proposed future directions for cloud programming
The authors sketch what a more powerful "next-generation" cloud programming model should look like, going beyond today's FaaS:
- Richer, data-centric abstractions with locality: Future platforms should support placing computation near data, with explicit control over data locality and caching, so that code-to-data becomes the norm rather than data shipping. This includes tighter integration between compute and storage and abstractions that understand data partitions and placement.
- First-class support for state and distributed coordination: Instead of forcing all state into external services, the platform should offer built-in, fault-tolerant state abstractions (shared logs, stateful operators, or transactional state) that compose naturally with elastic compute. This would make it feasible to implement databases, streaming systems, and other distributed engines directly on the cloud substrate.
- Access to heterogeneous and custom hardware: The cloud programming model should expose GPUs, TPUs, FPGAs, and other accelerators in a way that still preserves elasticity and high-level programmability. Developers should be able to request and program specialized hardware as part of the same unified model, rather than dropping down to separate services or VM-based deployments.
- Better openness and extensibility: The paper also emphasizes alignment with open-source ecosystems and the ability for researchers and practitioners to innovate on the cloud substrate itself, not just at the application layer. That means APIs and runtimes that are transparent enough to be extended, optimized, and experimented with by the broader community.
In short, the authors see today's serverless as an important but incomplete step. To truly "program the cloud", future platforms must marry autoscaling and pay-as-you-go simplicity with data locality, stateful distributed abstractions, and hardware diversity, turning the cloud into a genuine substrate for systems innovation rather than a constrained function host.

# Azure Durable Functions Deep Dive
So how does Azure Durable Functions stack up against these criticisms? It's an extension of Azure Functions that lets developers write stateful functions in a serverless environment, essentially adding workflow orchestration and state management on top of the basic FaaS model.
## Orchestration model
Durable Functions extends Azure Functions with an orchestration model built around client, orchestrator, and activity functions. Client functions start orchestrations (for example via HTTP). Orchestrator functions define the workflow in code, calling activity functions, waiting for results, handling retries, and composing sequential or parallel steps. Activity functions do the actual work and are invoked by the orchestrator.
Compared to basic FaaS, where each function is an isolated, stateless, short-lived handler, Durable Functions gives you a built-in workflow engine with reliable execution and coordination. This directly responds to Hellerstein et al.'s criticism that first-generation serverless makes it hard to express multi-step, distributed computations and forces developers to hand-roll orchestration logic around simple functions.
## State management
Durable Functions manages state using an event sourcing model. Every significant event in an orchestration is appended to a history log in durable storage. On each "await" or yield, the framework checkpoints progress. If the orchestrator is unloaded or the host restarts, it replays the event history to reconstruct local variables and control flow deterministically.
This means orchestrator functions are logically stateful and reliable, even though they run on a stateless compute substrate. That directly addresses the paper's complaint that FaaS functions are inherently stateless and push all durable state into external services, complicating distributed algorithms and long-running workflows. Durable Functions still stores state externally, but it turns that pattern into a first-class programming model instead of ad-hoc glue code.
## Execution timeouts
Regular Azure Functions are subject to execution time limits that depend on the hosting plan. Activity functions in Durable Functions still obey these underlying limits, because they are just normal functions doing work.
However, Orchestrator functions are designed to be long-running. Their progress is checkpointed whenever they await an activity or a durable timer. When an orchestrator "sleeps" on a timer, the runtime unloads it and later reloads and replays it, effectively bypassing traditional timeout constraints for the overall workflow. This directly tackles the paper's criticism that first-generation serverless platforms only support short-lived functions, making long-running analytics or business processes awkward or impossible without complex workarounds.
## Communication between functions
In Durable Functions, orchestrators and activities communicate indirectly via the Durable Task framework and a storage provider. With the default Azure Storage provider, orchestration state and history are stored in tables, and work is driven by storage queues. Blobs and leases handle distribution across workers. However, the developer sees a simple `CallActivityAsync` API and awaits results as if calling local functions.
This partially addresses the paper's criticism that serverless functions must communicate via "slow storage intermediaries". Under the hood, Durable Functions still uses storage and queues, but it hides the complexity and provides ordered, reliable messaging and correlation. It improves the programming model for coordination, though it does not fundamentally remove the storage-mediated communication path or its latency/bandwidth constraints.
## Parallel execution (fan-out/fan-in)
Durable Functions has a built-in fan-out/fan-in pattern: an orchestrator can schedule many activity functions in parallel (fan-out), collect their task handles, and then await `Task.WhenAll` to aggregate results (fan-in). This enables large-scale parallel processing scenarios such as backing up many sites, processing large files in pieces, or running many independent computations concurrently.
This directly responds to Hellerstein et al.'s concern that first-generation serverless makes distributed computing and parallelism clumsy, because each function is an isolated, short-lived unit with no native support for coordination patterns. Durable Functions turns fan-out/fan-in into a first-class orchestration primitive, making it much easier to express and manage parallel workflows, though underlying network and storage limits still bound performance.
# Critical Evaluation
## Limitations that remain unresolved
- One of the strongest criticisms in "Serverless Computing: One Step Forward, Two Steps Back" is that first-generation serverless platforms force all communication through slow, remote storage systems, preventing efficient distributed computing and eliminating data locality. Azure Durable Functions still relies on storage-backed coordination. The default backend uses Azure Storage tables, queues, and blobs to persist orchestration history and schedule work, as documented in Microsoft's official overview . Even advanced backends like Netherite (developed by Microsoft Research) acknowledge that reliable workflow execution in an elastic environment "poses significant performance challenges" because it still depends on durable logs and mediated by storage messaging.
This means Durable Functions improves the programming model but not the underlying architecture. The orchestrator cannot directly address or communicate with activity workers. All interactions are serialized through storage. As a result, the system still suffers from the I/O bottlenecks and latency issues the paper identifies.

- The CIDR paper argues that serverless platforms regress by hiding hardware capabilities such as GPUs, fast networks, and large-memory nodes, essential capabilities for modern data systems and machine learning. Durable Functions does not change this. It runs on the same Azure Functions infrastructure, which abstracts away hardware and provides no direct access to GPUs or specialized accelerators. Microsoft's documentation emphasizes stateful workflows and orchestration patterns, not hardware-level control or data-local execution.
This limitation persists because Durable Functions is fundamentally a workflow engine layered on top of the existing FaaS substrate. It cannot alter the underlying compute model, which remains optimized for elasticity and isolation rather than high-performance, data-intensive workloads. The platform still cannot support the "code-to-data" paradigm the paper advocates.

## Verdict
Azure Durable Functions represents a significant improvement in developer experience, but it does not constitute the kind of architectural progress the CIDR authors envisioned. It primarily offers a workaround, a sophisticated orchestration layer that compensates for the weaknesses of the underlying FaaS model without fundamentally solving them.

Durable Functions directly addresses several pain points the paper highlights:
- It introduces reliable, long-running workflows through event sourcing and replay.
- It provides built-in patterns for fan-out/fan-in parallelism.
- It offers a deterministic, stateful programming model atop stateless compute.

These are meaningful advances. But they are advances in abstraction, not in cloud substrate capabilities. 
The authors of the CIDR paper call for a future cloud model that supports:
- Data locality and code-to-data execution
- First-class distributed state
- Direct communication between compute nodes
- Access to heterogeneous hardware
Durable Functions does not deliver these. It still relies on remote storage for coordination, still lacks direct function-to-function networking, and still runs on opaque, hardware-abstracted compute nodes. Even Microsoft Research acknowledges that workflow execution in serverless environments continues to face "significant performance challenges" due to these architectural constraints.

Durable Functions is an elegant and powerful layer built on top of a limited foundation. It mitigates, but does not eliminate, the shortcomings identified in the paper. It represents progress in usability, reliability, and expressiveness, but not the deeper systems-level innovation the authors argue is necessary for the next generation of cloud computing.

In short, Durable Functions is a clever workaround, not the architectural leap the paper calls for.

# References
## Primary Paper
Hellerstein, J. M., et al. Serverless Computing: One Step Forward, Two Steps Back. CIDR 2019.[Link](https://www.cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf)

## Microsoft Documentation
- Microsoft Docs - Durable Functions Overview  [Link](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-overview)

- Microsoft Docs - Durable Functions Storage Providers  [link](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-storage-providers)

- Microsoft Docs - Orchestrator Function Code Constraints  [link](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-orchestrations)

- Microsoft Docs - Durable Functions Patterns (Fan-out/Fan-in)  [link](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-overview#fan-outfan-in)

- Microsoft Research
Netherite: High-Performance Backend for Durable Functions [link](https://www.microsoft.com/en-us/research/project/netherite/)

# AI Disclosure Statement
Used ChatGPT to get the links for this research and to summarize the PDF. Used Amazon Q to reformulate and correct the sentences.