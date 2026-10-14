# Building a Private Local AI Code-Fixing Agent
Reducing Claude/API Costs with Ollama, Local Coding LLMs, RAG and Agentic AI
# 1. Introduction

AI coding assistants such as Claude are increasingly effective at understanding large codebases, identifying root causes, tracing dependencies, suggesting fixes, generating tests and helping developers troubleshoot complex defects.

For legacy applications, particularly large C#, VB.NET, SQL Server and desktop/server applications, this can significantly reduce the effort required to investigate defects.

However, extensive code analysis can consume a large number of tokens. When AI agents repeatedly analyse repositories, search dependencies and investigate tickets, API or subscription costs can become an important consideration.

An alternative is to introduce a local AI coding layer.

The idea is not necessarily to replace Claude completely. Instead, we can build a hybrid architecture where:

A local LLM handles repository understanding, code search, routine analysis and simpler fixes, while a powerful cloud model is used selectively for complex problems.

This article describes a reference architecture for such a solution.

# 2. The Objective

The goal is to build a private AI coding agent capable of performing tasks such as:

Understand an existing repository
Search relevant code automatically
Trace dependencies and call flows
Analyse a bug or ticket
Identify potentially impacted files
Suggest a root cause
Generate code changes
Generate unit tests
Run builds and selected tests
Inspect compiler/test failures
Refine a proposed fix
Produce a developer-friendly explanation
Prepare a pull/merge request description

The key design principle is:

Plain Text
Don't send the entire repository to the LLM.
 
Find the right code first,
then give the LLM the right context.

This improves efficiency regardless of whether the final reasoning model is local or cloud-based.

# 3. Do We Need to Train Our Own LLM?

Usually, no.
Training a software-engineering LLM from scratch would require significant amounts of training data, GPU infrastructure, ML expertise and model evaluation.
For most enterprises, a better approach is:

Existing coding LLM
+
Repository Intelligence
+
RAG
+
Agentic Tools
+
Optional Fine-Tuning

In other words:

Build the intelligence around the model before attempting to build the model itself.

Start with an open-weight coding model and run it locally.

Ollama can provide the local model-serving layer.

Possible model families should be evaluated based on requirements such as:

Coding capability
Context window
Hardware requirements
Licence
Language support
Latency
Quantisation support
Tool-use capability

The correct model should be determined through benchmarking against your application's real defects rather than simply choosing the largest model.

5. Core Components
5.1 Source Code Repository

The source repository remains the system of record.

For a legacy enterprise application it may contain:

Plain Text
/src
/VB.NET
/CSharp
/Services
/Desktop
/Libraries
 
/database
/StoredProcedures
/Functions
/Scripts
 
/tests
 
/docs
 
/config

The AI system should never assume that all required information exists in one file.

A defect may involve:

Plain Text
UI
↓
Business Layer
↓
Service
↓
Data Access
↓
Stored Procedure
↓
Database

Therefore, repository relationships are critical.

# 6. Repository Intelligence

A good coding agent needs more than embeddings.

Before asking the LLM to fix a problem, build a lightweight representation of the repository.

For example:

Plain Text
Solution
│
├── Projects
│
├── Files
│
├── Classes
│
├── Interfaces
│
├── Methods
│
├── Dependencies
│
├── SQL
│
└── Configuration

For .NET applications this can include extraction of:

Namespace
Class
Interface
Method
Constructor
Property
Method calls
Interface implementations
Project references
Configuration references
SQL calls
Stored procedure names

For modern C# projects, Roslyn-based analysis is worth considering.

Legacy VB.NET repositories may require a combination of compiler APIs, syntax analysis and fallback parsing depending on the project structure.

# 7. Why AST-Aware Chunking Matters

A common RAG implementation divides documents every N characters or tokens.

That works reasonably well for natural language.

It is less effective for source code.

Consider:

Class
│
├── Method A
│
├── Method B
│
└── Method C

A fixed-size chunker could split Method B halfway through.

The resulting embedding loses useful semantic context.

A better strategy is:

Namespace
↓
Class
↓
Method
↓
Code Chunk

Each indexed entity should retain metadata such as:

JSON
{
"repository": "application",
"project": "BusinessLayer",
"file": "CustomerService.cs",
"namespace": "Application.Services",
"class": "CustomerService",
"method": "UpdateCustomer",
"language": "CSharp"
}

This makes retrieval much more useful.

An internal architecture example, AI-AugmentedSDLC, similarly describes AST/regex parsing, embeddings, hybrid retrieval and provenance-aware code chunks.

# 8. Build the Code Knowledge Index

After extracting meaningful chunks, calculate embeddings.

A simplified pipeline becomes:

Source Repository
│
▼
File Discovery
│
▼
Language Detection
│
▼
AST / Parser
│
▼
Semantic Code Chunks
│
├──────────► Metadata
│
▼
Embedding Model
│
▼
Vector Database

Possible storage options include:

PostgreSQL + pgvector
Vector database
Local embedded vector store

For enterprise applications already using PostgreSQL infrastructure, pgvector can be convenient.

# 9. Vector Search Alone Is Not Enough

Code contains very specific identifiers.

For example:

CustomerId
SiteId
FrmPrintOrEmail
UpdateTenant
GetPayment
sp_GetCustomer

Vector search is useful for conceptual questions such as:

"Where is payment validation performed?"

But exact search works better when a ticket contains:

UpdateTenant()

Therefore use hybrid search:

User Question
│
├─────────────┐
│ │
▼ ▼
Semantic Search Keyword Search
│ │
Vector Lexical
Similarity Search
│ │
└──────┬──────┘
│
▼
Ranking
│
▼
Best Code
Chunks

The internal AI-AugmentedSDLC also describes a hybrid vector and full-text search architecture.

# 10. The Local LLM Layer

The selected coding LLM is served locally.

Conceptually:


Developer
↓
AI Agent
↓
Ollama
↓
Coding LLM

One internal example specifically uses qwen2.5-coder:7b with Ollama as the local generative model in an AI-augmented SDLC pipeline.

Another internal discussion describes Microsoft Phi-4 being used locally through Ollama for code review and identifying security issues and fixes.

These should be treated as candidate approaches, not automatically as the best model for every repository.

# 11. The Agent Layer

The model alone should not control the entire workflow.

Introduce an agent/orchestration layer.

The agent is responsible for deciding what information the model needs.

For example:

Bug
│
▼
Planner
│
├── Search repository
│
├── Find relevant class
│
├── Find callers
│
├── Find interfaces
│
├── Find SQL
│
├── Read related tests
│
└── Construct context
│
▼
LLM

This fundamentally changes the interaction from:

LLM, read my repository and find the problem.

to:

Plain 
Agent:
I found the 8 most relevant pieces
of code and their relationships.
 
LLM:
Analyse this evidence and determine
the likely cause.

That distinction is critical for cost and context management.

# 12. Example Defect Investigation

Imagine a ticket:

Customer's Site ID is not updated correctly after saving.

The agent receives the ticket.

Step 1: Extract key concepts
Site ID
customer
save
update
Step 2: Search repository

Hybrid retrieval finds candidate methods such as:
SaveCustomer()
UpdateSite()
CustomerRepository.Save()
sp_UpdateCustomer
Step 3: Dependency expansion

Suppose:

Plain Text
Save Button
↓
SaveCustomer()
↓
CustomerService.Update()
↓
CustomerRepository.Save()
↓
sp_UpdateCustomer
 

The agent can add these relationships to the context.

Step 4: Ask the LLM

Instead of sending thousands of files, send:

Plain Text
Bug description
 
+
 
Relevant code
 
+
 
Call graph
 
+
 
Database interaction
 
+
 
Current tests

The LLM can then reason over a much smaller and more relevant context.

13. Root Cause Analysis Output

The first AI output should ideally be analysis rather than an immediate code mutation.

For example:

Plain Text
Issue
-----
Site ID is not updated after customer save.
 
Potential Root Cause
--------------------
A value passed from the business layer differs from
the value expected by the persistence layer.
 
Impact
------
Customer update workflow.
 
Potentially Impacted Components
-------------------------------
Customer form
Customer service
Repository
Stored procedure
 
Suggested Validation
--------------------
Verify parameter mapping and reproduce the behaviour
with a controlled test.

This output should be treated as a hypothesis until validated.

This is particularly important for production issues.

In an earlier internal discussion about Claude Code effectiveness, the limitation that the model cannot see runtime/production data was explicitly identified, with conclusions requiring verification against operational evidence such as logs, Splunk and database data.

14. Code Fix Generation

Once the probable cause is identified, the agent can request a minimal patch.

A useful principle is:

Generate the smallest safe change rather than refactoring unrelated code.

The agent should provide:

Plain Text
Original code
+
Root cause
+
Related callers
+
Known behavioural constraints
+
Relevant tests

Then request:

Plain Text
Minimal fix
+
Explanation
+
Potential side effects
+
Tests
15. Automated Test Generation

Every proposed fix should ideally include validation.

Plain Text
AI Fix
│
├── Existing Tests
│
└── Generated Tests
│
▼
Test Runner

Useful scenarios include:

Plain Text
Expected path
Boundary case
Null/empty input
Regression scenario
Original defect reproduction

The tests themselves should still undergo developer review.

16. Build and Test Feedback Loop

This is where the solution starts moving from a chatbot towards an engineering agent.

Plain Text
Generate Fix
│
▼
Apply Patch
│
▼
Compile
│
┌───┴────┐
│ │
Pass Fail
│ │
│ ▼
│ Compiler Error
│ │
│ ▼
│ Agent
│ │
│ Revised Patch
│ │
└────────┘
│
▼
Run Tests

Compiler and test output becomes additional context for subsequent reasoning.

This is far stronger than repeatedly asking:

"Can you fix this code?"

The model has objective feedback from the engineering toolchain.

17. Local vs Cloud Model Routing

Not every defect should automatically reach an expensive frontier model.

Introduce a routing mechanism.

Ticket
│
▼
Local Agent
│
Initial Analysis
│
┌───────┴───────┐
│ │
Routine Problem Complex Problem
│ │
▼ ▼
Local LLM Cloud LLM
│ Claude
│ │
└───────┬───────┘
│
▼
Validation

Potential escalation signals could include:

Local model cannot produce a valid build
Tests repeatedly fail
Retrieval confidence is insufficient
Multiple subsystems are involved
Architectural reasoning is required
Security-sensitive code is affected
Developer explicitly requests escalation

This is a suggested design approach. The appropriate routing policy should be determined experimentally for the target repository.

18. Cost Optimisation Strategy

Without local retrieval:

Large Repository
↓
Large Prompt
↓
Cloud LLM
↓
Large Token Usage

With the proposed architecture:

Large Repository
↓
Local Index
↓
Hybrid Search
↓
Relevant Code Only
↓
Local LLM
│
│ if necessary
▼
Cloud LLM

The strategy therefore attacks cost in two ways:

1. Reduce unnecessary context

Only relevant code is sent for reasoning.

2. Reduce cloud calls

Routine operations can be routed through locally hosted models.

An internal discussion has similarly raised token cost and vendor-dependency concerns while describing local specialised models and fine-tuning as possible approaches.

19. Do Not Fine-Tune Initially

A frequent mistake is:

Plain Text
"We have company code.
Let's train a model."

Instead:

Plain Text
Step 1
Base coding model
 
Step 2
Repository RAG
 
Step 3
Agent tools
 
Step 4
Benchmark
 
Step 5
Collect successful examples
 
Step 6
Consider fine-tuning

Fine-tuning becomes interesting once you have a clean dataset containing examples such as:

Plain Text
Ticket
Issue
Relevant code
Root cause
Approved fix
Tests
Review comments
Final outcome

Even then, repository knowledge that changes frequently is better supplied dynamically rather than permanently encoded into model weights.

20. Future Fine-Tuning Architecture

If benchmarking demonstrates a need for specialisation:

Plain Text
Historical Issues
│
▼
Remove Sensitive Data
│
▼
Quality Filtering
│
▼
Training Examples
│
▼
LoRA / QLoRA
│
▼
Specialised Coding Model
│
▼
Evaluation
│
▼
Ollama / Local Serving

The purpose would not be to make the model memorise the repository.

Instead, fine-tuning could target behaviours such as:

Organisation-specific coding patterns
Defect-analysis style
Structured root-cause output
Test-generation conventions
Common legacy coding patterns

Public guidance on local LLM fine-tuning describes QLoRA as one approach for adapting an existing model, with the resulting model subsequently deployable locally.

21. Security Architecture

Running models locally has significant architectural implications, but local does not automatically mean secure.

The system still requires security controls.

Plain Text
Developer
│
Authentication
│
▼
AI Agent Layer
│
┌──────────┴──────────┐
│ │
Repository Local LLM
Access Control Environment
│ │
└──────────┬──────────┘
│
Audit
│
▼
Security Logs

Important areas include:

Repository permissions
Secrets management
Prompt/context logging policy
Model access control
Dependency security
Data retention
Audit trails
Sensitive code classification
Agent tool permissions
22. Never Index Secrets

Before repository ingestion, exclude or carefully handle sensitive material such as:

Plain Text
.env
connection strings
private keys
certificates
API keys
passwords
tokens
production configuration
credential files

A repository index should not become a second uncontrolled secrets database.

Existing secret-scanning controls should run before ingestion where appropriate.

23. Agent Security

An AI coding agent becomes more dangerous as more capabilities are granted.

Consider the difference:

Plain Text
Level 1
Read repository
 
Level 2
Generate recommendation
 
Level 3
Modify working tree
 
Level 4
Run commands
 
Level 5
Create commits
 
Level 6
Create pull requests
 
Level 7
Merge code

A production implementation should follow least privilege.

A safer initial design is:

Plain Text
AI proposes
Developer approves
Automation validates
Developer merges

rather than:

Plain Text
AI decides
AI modifies
AI merges
AI deploys
24. Observability

Measure the system like an engineering platform.

Example metrics:

Plain Text
Tickets analysed
 
Local-model requests
 
Cloud-model requests
 
Escalation rate
 
Prompt/context size
 
Build success rate
 
Test success rate
 
Fix acceptance rate
 
Fix rejection rate
 
Average investigation effort
 
Cost per resolved issue

Without measurement, it is difficult to prove whether the local architecture actually saves money or improves developer productivity.

25. Evaluation Framework

Create a benchmark from previously solved defects.

For example:

Plain Text
20-50 representative defects

Include a mixture of:

Plain Text
Simple UI defect
Business logic defect
SQL problem
Data mapping problem
Configuration issue
Cross-layer defect
Regression
Performance issue
Complex architectural problem

For every issue compare:

Plain Text
Local LLM
vs
Cloud LLM
vs
Hybrid architecture

Evaluate:

Plain Text
Root cause accuracy
Relevant file identification
Fix correctness
Test correctness
Build success
Developer acceptance
Investigation effort
Infrastructure/API cost

This provides evidence for deciding where local models are sufficient.

26. Recommended POC Architecture

For a legacy .NET environment, a practical experimental stack could look like:

Plain Text
┌─────────────────────────────────────────┐
│ VS Code │
└───────────────────┬─────────────────────┘
│
▼
┌─────────────────────────────────────────┐
│ AI Agent │
│ │
│ Planner │
│ Repository Search │
│ Context Builder │
│ Tool Executor │
│ Model Router │
└───────────────┬─────────────────────────┘
│
┌─────────┴─────────┐
│ │
▼ ▼
┌──────────────┐ ┌───────────────────┐
│ Repository │ │ Local LLM │
│ Intelligence │ │ │
│ │ │ Ollama │
│ AST │ │ Coding Model │
│ Embeddings │ └─────────┬─────────┘
│ pgvector │ │
│ Text Search │ │ escalation
└──────────────┘ ▼
┌──────────────┐
│ Cloud LLM │
│ Claude │
└──────────────┘

An internal solution in AI-AugmentedSDLC provides a useful precedent for this direction, describing Qwen2.5-Coder 7B, pgvector, hybrid retrieval, local fix generation, tests and a cloud-model path.

27. Suggested POC Phases
Phase 1: Repository Understanding

Build:

Plain Text
Repository scanner
↓
AST-aware chunking
↓
Embeddings
↓
Vector + keyword index
↓
Repository search

Goal:

Ask architectural/code questions and retrieve the correct implementation areas.

Phase 2: AI Code Analyst

Add a local coding LLM.

Capabilities:

Plain Text
Explain code
Find related code
Trace flow
Identify possible root cause
Identify impacted components

Do not modify source automatically yet.

Phase 3: Fix Generator

Add:

Plain Text
Patch generation
Unit-test generation
Diff presentation
Developer approval
``
Phase 4: Engineering Agent

Add tool execution:

Plain Text
Apply patch
↓
Build
↓
Test
↓
Analyse failures
↓
Refine
Phase 5: Hybrid Routing

Introduce cloud escalation.

Plain Text
Local first
│
├── solved → validation
│
└── unresolved
↓
Cloud LLM
Phase 6: Organisation-Specific Optimisation

After sufficient usage data:

Plain Text
Evaluate failures
↓
Build training dataset
↓
Fine-tune if justified
↓
Re-evaluate
28. Git Repository Structure

A future POC repository could be organised as:

Plain Text
local-ai-code-agent/
 
├── README.md
├── docs/
│ ├── architecture.md
│ ├── security.md
│ ├── rag-design.md
│ └── evaluation.md
│
├── agent/
│ ├── planner/
│ ├── router/
│ ├── tools/
│ └── prompts/
│
├── indexing/
│ ├── scanners/
│ ├── parsers/
│ ├── chunking/
│ └── embeddings/
│
├── retrieval/
│ ├── vector/
│ ├── lexical/
│ └── ranking/
│
├── llm/
│ ├── local/
│ └── cloud/
│
├── tools/
│ ├── build/
│ ├── tests/
│ ├── git/
│ └── static-analysis/
│
├── evaluation/
│ ├── datasets/
│ ├── benchmarks/
│ └── reports/
│
└── tests/

This separates repository intelligence, retrieval, model inference and agent execution, allowing each component to evolve independently.

29. Key Architectural Principle

The biggest lesson is:

The LLM is only one component of the solution.

A capable code-fixing platform is closer to:

Plain Text
AI Coding Agent
=
LLM
+
Repository Intelligence
+
RAG
+
Dependency Analysis
+
Engineering Tools
+
Build/Test Feedback
+
Security Controls
+
Human Review

This is why a relatively small local coding model can potentially perform useful work when surrounded by strong repository intelligence.

30. Target End State

The long-term architecture becomes:

Plain Text
BUG / JIRA
│
▼
AI ENGINEERING AGENT
│
┌────────────┼─────────────┐
│ │ │
▼ ▼ ▼
RAG Code Graph Tools
│ │ │
└────────────┼─────────────┘
│
▼
LOCAL LLM
│
Can solve safely?
/ \
Yes No
│ │
│ ▼
│ CLAUDE
│ │
└────┬────┘
│
▼
Proposed Fix
│
Build + Tests
│
Security Scan
│
Developer Review
│
▼
MR

Cloud AI therefore becomes an escalation capability, rather than being responsible for every token of repository exploration.

Conclusion

Running an open-weight coding model locally is a realistic way to investigate reducing reliance on continuously metered cloud LLM usage.

However, simply installing a model in Ollama will not create a Claude Code equivalent.

The real engineering value comes from building the ecosystem around the model:

Repository indexing + AST understanding + hybrid RAG + dependency analysis + agent orchestration + build/test feedback + human approval.

The recommended strategy is therefore:

Plain Text
1. Do not train an LLM from scratch.
 
2. Start with an existing coding model.
 
3. Run it locally through Ollama.
 
4. Build AST-aware repository indexing.
 
5. Implement hybrid semantic + lexical retrieval.
 
6. Add an agent capable of navigating repository relationships.
 
7. Integrate build and tests as objective feedback.
 
8. Keep developers in the approval loop.
 
9. Escalate difficult problems to Claude.
 
10. Benchmark real historical issues.
 
11. Collect high-quality accepted solutions.
 
12. Fine-tune only when benchmarking proves there is
additional value.

The target is not necessarily:

"Replace Claude."

A more practical objective is:

"Build a private, cost-efficient AI software-engineering platform where local models perform most repository intelligence and routine engineering tasks, while frontier models are reserved for the problems where their additional reasoning capability provides real value."

That approach provides a much stronger foundation for an enterprise Agentic AI Software Engineering platform than simply switching from one LLM provider to another.

