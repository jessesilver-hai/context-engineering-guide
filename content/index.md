---
title: "Context Engineering: The Complete Map of the Field"
description: "A comprehensive guide covering 50+ areas across 12 categories of context engineering for LLMs — from foundational design to emerging frontiers."
tags:
  - context-engineering
  - index
  - llm
---

# Context Engineering: The Complete Map of the Field

> **Last updated:** 2026-03-08
> **Scope:** Every identifiable area, sub-discipline, and emerging topic within context engineering for LLMs.

---

## What Is Context Engineering?

Context engineering is the discipline of designing, curating, assembling, and managing everything an LLM sees at inference time -- system prompts, retrieved documents, memory, tool definitions, conversation history, state information, examples, and structured data -- to maximize the probability that the model produces the desired output.

The term gained mainstream adoption in mid-2025, popularized by **Tobi Lutke** (Shopify CEO), who [tweeted](https://x.com/tobi/status/1935533422589399127): "I really like the term 'context engineering' over prompt engineering. It describes the core skill better: the art of providing all the context for the task to be plausibly solvable by the LLM." **Andrej Karpathy** immediately endorsed it, [defining it as](https://x.com/karpathy/status/1937902205765607626) "the delicate art and science of filling the context window with just the right information for the next step."

Unlike prompt engineering, which focuses on crafting a single instruction, context engineering is a systems discipline. It encompasses the entire information architecture surrounding the model: what data reaches it, when, in what format, how much, and through what retrieval/memory/tool pipelines. It treats the context window as a precious, finite resource (analogous to RAM) that must be actively managed.

The field was formalized academically in July 2025 with a [comprehensive survey of 1,400+ papers](https://arxiv.org/abs/2507.13334) (Mei et al.), and operationally by Anthropic's influential [engineering blog post](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) in September 2025, which became the de facto practitioner reference.

---

## Deep Dives

This guide contains detailed articles on specific areas of context engineering:

- [[agent-architecture-and-context-isolation|Agent Architecture and Context Isolation]] — How agent decomposition solves context management problems
- [[agent-self-optimization-and-tool-generation|Agent Self-Optimization and Tool Generation]] — Making agents faster by detecting slow operations and auto-generating scripts
- [[context-economics-and-enforcement-mechanisms|Context Economics and Enforcement Mechanisms]] — Every instruction competes for attention budget
- [[context-management-for-monolithic-workflows|Context Management for Monolithic Workflows]] — State files, not context, for sequential pipelines
- [[plugin-and-marketplace-directory-structure|Plugin and Marketplace Directory Structure]] — How to organize a Claude Code plugin marketplace
- [[skill-and-memory-architecture|Skill and Memory Architecture]] — Best practices for structuring skills, memory, and context
- [[skill-authoring-and-evaluation-patterns|Skill Authoring and Evaluation Patterns]] — Writing effective skills and iteratively improving them
- [[skill-naming-conventions|Skill Naming Conventions]] — Comprehensive guide to naming skills
- [[workflow-patterns-from-the-creator|Workflow Patterns from the Creator]] — Actionable patterns from the Head of Claude Code

---

## Part 1: Comprehensive Area Index

### A. Foundational Context Design

#### A1. System Prompt Architecture
Designing the persistent instructions that define an LLM's role, constraints, tone, output format, and behavioral boundaries. Includes structural patterns (XML tags, Markdown headers, delimiters), persona/role assignment, and the "right altitude" principle -- balancing specificity with flexibility.

- **Key authorities:** Anthropic, OpenAI, Google
- **Best single resource:** [Anthropic -- Prompting Best Practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)

#### A2. Prompt Engineering (as a Subset)
The craft of writing individual instructions, questions, and task descriptions to elicit desired model behavior. Now understood as one component within the broader context engineering discipline.

- **Key authorities:** Anthropic, OpenAI, DAIR.AI (Prompt Engineering Guide)
- **Best single resource:** [DAIR.AI Prompt Engineering Guide](https://www.promptingguide.ai/)

#### A3. Few-Shot and In-Context Learning
Providing examples (demonstrations) within the prompt to teach the model a task pattern without fine-tuning. Includes example selection strategies, retrieval-based demonstration selection (RetICL), and the trade-off between example diversity and similarity.

- **Key authorities:** Google Brain (original ICL research), Stanford NLP
- **Best single resource:** [In-context Learning with Retrieved Demonstrations: A Survey](https://arxiv.org/html/2401.11624v1)

#### A4. Chain-of-Thought and Reasoning Context
Techniques that structure the model's reasoning process through intermediate steps. Includes CoT prompting, Tree of Thoughts (ToT), Graph of Thoughts (GoT), ReAct, Self-Consistency, and the broader "Chain-of-X" paradigm family.

- **Key authorities:** Google Brain (Wei et al.), Princeton (Yao et al.), ETH Zurich
- **Best single resource:** [Chain-of-Thought Prompting Elicits Reasoning in LLMs](https://arxiv.org/abs/2201.11903)

#### A5. Structured Output and Constrained Generation
Controlling model output format through JSON schemas, grammar-based decoding, and constrained decoding techniques. Ensures outputs are machine-parseable and integrate directly into software pipelines. Includes XGrammar, Outlines, Guidance, and llguidance.

- **Key authorities:** Microsoft (Guidance), .txt (Outlines), NVIDIA, OpenAI (Structured Outputs API)
- **Best single resource:** [A Guide to Structured Generation Using Constrained Decoding](https://www.aidancooper.co.uk/constrained-decoding/)

#### A6. Document and Information Ordering
Strategic placement of information within the context window to maximize model attention. Addresses the "Lost in the Middle" problem -- the U-shaped attention curve where models attend strongly to the beginning and end of context but poorly to the middle.

- **Key authorities:** Stanford NLP (Liu et al.), Chroma Research
- **Best single resource:** [Found in the Middle: How Language Models Use Long Contexts Better via Plug-and-Play Positional Encoding](https://arxiv.org/html/2403.04797v1)

---

### B. Context Retrieval and Knowledge Integration

#### B1. Retrieval-Augmented Generation (RAG)
The foundational pattern of retrieving external documents at inference time and injecting them into the context window to ground model responses. Includes the full RAG lifecycle: indexing, chunking, embedding, retrieval, re-ranking, and generation.

- **Key authorities:** Meta AI (Lewis et al., original RAG paper), LlamaIndex, LangChain
- **Best single resource:** [RAGFlow: From RAG to Context -- A 2025 Year-End Review](https://ragflow.io/blog/rag-review-2025-from-rag-to-context)

#### B2. Advanced RAG Architectures
Evolution beyond basic RAG: modular RAG, agentic RAG (agents decide when/what to retrieve), multi-hop retrieval (decomposing complex queries), RAG-Fusion (combining multiple reformulated queries), self-RAG (self-reflective retrieval), and hybrid retrieval (combining dense + sparse methods).

- **Key authorities:** LangChain, LlamaIndex, Microsoft Research, Anthropic
- **Best single resource:** [Retrieval-Augmented Generation: A Comprehensive Survey](https://arxiv.org/html/2506.00054v1)

#### B3. Knowledge Graph-Enhanced Retrieval (GraphRAG)
Using knowledge graphs to structure and retrieve context, enabling multi-hop reasoning over entity relationships. Includes GraphRAG, LightRAG, HippoRAG, and RAPTOR. Achieves higher accuracy on complex queries than vector-only retrieval.

- **Key authorities:** Microsoft Research (GraphRAG), Neo4j, Zep (Graphiti)
- **Best single resource:** [IBM -- What is GraphRAG?](https://www.ibm.com/think/topics/graphrag)

#### B4. Embedding Models and Vector Search
The infrastructure layer: converting text to dense vector representations and performing similarity search. Includes embedding model selection, fine-tuning for domain specificity, late interaction models (ColBERT), and hybrid retrieval combining BM25 with dense embeddings.

- **Key authorities:** Hugging Face (MTEB benchmark), Cohere, OpenAI, Jina AI
- **Best single resource:** [Agentset Embedding Model Leaderboard](https://agentset.ai/embeddings)

#### B5. Document Chunking Strategies
How documents are split into retrievable units. Includes fixed-size chunking, recursive character splitting, semantic chunking (embedding-based boundary detection), AST-based code chunking, and the trade-offs between chunk size, overlap, and retrieval quality.

- **Key authorities:** Weaviate, LangChain, LlamaIndex
- **Best single resource:** [Weaviate -- Chunking Strategies to Improve RAG Pipeline Performance](https://weaviate.io/blog/chunking-strategies-for-rag)

#### B6. Contextual Compression and Re-Ranking
Post-retrieval processing to reduce noise: re-ranking retrieved documents by relevance, extracting only the pertinent sentences, and compressing context to fit within token budgets. Includes Cohere Rerank, cross-encoder re-ranking, and LLM-based extraction.

- **Key authorities:** LangChain, Cohere, Anthropic
- **Best single resource:** [LangChain -- Improving Document Retrieval with Contextual Compression](https://blog.langchain.com/improving-document-retrieval-with-contextual-compression/)

#### B7. Progressive Disclosure and Just-in-Time Retrieval
Instead of pre-loading all context, agents incrementally discover information through exploration -- using file search, glob patterns, metadata signals, and targeted queries. Maintains lightweight identifiers and loads full content only when needed.

- **Key authorities:** Anthropic (Claude Code team)
- **Best single resource:** [Anthropic -- Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

---

### C. Context Processing and Optimization

#### C1. Context Compression Techniques
Reducing token count while preserving relevant information. Includes prompt compression (SelectiveContext, LLMLingua, QwenLong-CPRS), token pruning, summarization-based compression, and extraction-based methods. A NAACL 2025 survey formalized the taxonomy.

- **Key authorities:** Microsoft Research, Tsinghua University, NAACL 2025
- **Best single resource:** [Prompt Compression for Large Language Models: A Survey (NAACL 2025)](https://aclanthology.org/2025.naacl-long.368.pdf)

#### C2. Context Summarization and Compaction
Condensing long conversation histories or agent trajectories into compact summaries that preserve decisions, findings, and state. Used when context windows fill up. Includes recursive summarization, hierarchical summarization, and Claude Code's auto-compaction (triggers at 95% context usage).

- **Key authorities:** Anthropic, LangChain, OpenAI
- **Best single resource:** [LangChain -- Context Management for Deep Agents](https://blog.langchain.com/context-management-for-deepagents/)

#### C3. Token Compression and KV Cache Optimization
Hardware-level optimizations: KV cache management, prefix caching, paged attention (vLLM), tiered storage (LMCache), entropy-guided caching, and chunk-level semantic compression (ChunkKV). Reduces inference cost and latency by 75-90%.

- **Key authorities:** UC Berkeley (vLLM/SGLang), NVIDIA, Google Cloud, Anthropic
- **Best single resource:** [BentoML -- KV Cache Offloading (LLM Inference Handbook)](https://bentoml.com/llm/inference-optimization/kv-cache-offloading)

#### C4. Prefix Caching and Prompt Caching
Reusing computed KV cache from shared prompt prefixes across requests. Includes explicit caching (Anthropic, Google) and implicit/automatic caching (Gemini 2.5+). Anthropic offers 90% cost savings; Gemini offers 90% cached read discounts.

- **Key authorities:** Anthropic, Google DeepMind, vLLM project
- **Best single resource:** [Google -- Context Caching (Gemini API)](https://ai.google.dev/gemini-api/docs/caching)

#### C5. Long Context Processing
Architectural innovations enabling models to process very long inputs (100K-10M tokens). Includes FlashAttention, Ring Attention, Infini-attention, StreamingLLM, position interpolation (YaRN, RoPE scaling), and state-space models (Mamba).

- **Key authorities:** Google DeepMind, Meta AI, Stanford, Together AI
- **Best single resource:** [Google -- Long Context (Gemini API)](https://ai.google.dev/gemini-api/docs/long-context)

#### C6. Context Rot and Degradation
The phenomenon where LLM performance degrades as context grows, even on simple tasks. Coined by Chroma Research in 2025. Caused by the "lost in the middle" effect, quadratic attention scaling, and semantic distractor interference. All frontier models are affected.

- **Key authorities:** Chroma Research, Stanford NLP
- **Best single resource:** [Chroma Research -- Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://research.trychroma.com/context-rot)

---

### D. Memory Systems and State Management

#### D1. Short-Term (Working) Memory
The context window itself, functioning as the model's working memory. Includes conversation buffer management, sliding window approaches, and turn-level memory. Managing what stays in and what gets evicted from the active context.

- **Key authorities:** OpenAI (Assistants API), Anthropic, LangChain
- **Best single resource:** [OpenAI -- Assistants API Deep Dive](https://developers.openai.com/api/docs/assistants/deep-dive/)

#### D2. Long-Term Memory Architectures
Persistent storage and retrieval of information across sessions. Three types: **semantic memory** (facts and knowledge), **episodic memory** (specific past interactions and experiences), and **procedural memory** (learned behaviors and instructions). Implementations include vector stores, knowledge graphs, and file-based systems.

- **Key authorities:** LangChain (LangMem), Zep, Letta (MemGPT), Anthropic
- **Best single resource:** [LangChain -- LangMem Conceptual Guide](https://langchain-ai.github.io/langmem/concepts/conceptual_guide/)

#### D3. Virtual Context Management (MemGPT/Letta)
OS-inspired memory management for LLMs: treating the context window as "RAM" and external storage as "disk," with the LLM itself managing paging between tiers. The model decides what to store, summarize, and retrieve through self-directed tool calls.

- **Key authorities:** UC Berkeley (Packer et al.), Letta
- **Best single resource:** [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560)

#### D4. Temporal Knowledge Graphs for Memory
Time-aware memory architectures that track how facts change over time, enabling agents to reason about temporal relationships and invalidate stale information. Zep's Graphiti framework exemplifies this with incremental graph updates and temporal extraction.

- **Key authorities:** Zep (Graphiti), Cognee
- **Best single resource:** [Zep: A Temporal Knowledge Graph Architecture for Agent Memory](https://arxiv.org/abs/2501.13956)

#### D5. Scratchpads and Working Notes
Agents write intermediate findings and plans to external files or state objects during long tasks, preventing information loss during context compaction. Enables multi-hour task continuity across context resets.

- **Key authorities:** Anthropic (Claude Code), LangChain
- **Best single resource:** [LangChain -- Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/)

#### D6. Cross-Session Memory and Personalization
Maintaining user-specific context across separate conversations. Includes user profiles, preference learning, and adaptive behavior. ChatGPT Memories, Claude Memory, Cursor rules files, and CLAUDE.md all implement variants of this pattern.

- **Key authorities:** OpenAI, Anthropic, Cursor
- **Best single resource:** [Anthropic -- Claude Code Skills Documentation](https://code.claude.com/docs/en/skills)

#### D7. Multi-Turn Conversation Management
Maintaining coherence across extended dialogues. Includes dialogue state tracking (DST), context-driven incremental compression (C-DIC), conversation threading, and selective context injection. Research shows a 39% average performance drop in multi-turn vs single-turn settings.

- **Key authorities:** ACM (Multi-turn Dialogue Survey), OpenAI, Anthropic
- **Best single resource:** [ACM -- A Survey on Recent Advances in LLM-Based Multi-turn Dialogue Systems](https://dl.acm.org/doi/full/10.1145/3771090)

---

### E. Tool Use and External Context

#### E1. Tool Definition and Function Calling
Defining tools/functions that the model can invoke, including description writing, parameter schemas, and the tool-calling protocol. Tool definitions are part of the context on every call, so they must be concise yet unambiguous. Models select tools based on description quality.

- **Key authorities:** Anthropic, OpenAI, Google
- **Best single resource:** [Prompt Engineering Guide -- Function Calling in AI Agents](https://www.promptingguide.ai/agents/function-calling)

#### E2. Tool Discovery and Dynamic Tool Selection
When an agent has access to many tools (30+), loading all definitions into context is wasteful. Tool discovery uses semantic search over tool registries to surface only relevant tools for the current task. LangGraph Bigtool achieves 3x improvement in tool selection accuracy.

- **Key authorities:** LangChain (Bigtool), Anthropic (Tool Search), Composio
- **Best single resource:** [Composio -- Tool Calling Explained: The Core of AI Agents](https://composio.dev/blog/ai-agent-tool-calling-guide)

#### E3. Model Context Protocol (MCP)
The open standard (created by Anthropic, Nov 2024; donated to Linux Foundation, Dec 2025) for connecting LLMs to external data sources and tools via a client-server architecture using JSON-RPC 2.0. Now adopted by Anthropic, OpenAI, Google, and Microsoft, with 97M+ monthly SDK downloads and 75+ official connectors.

- **Key authorities:** Anthropic, Linux Foundation (Agentic AI Foundation), OpenAI, Google
- **Best single resource:** [Model Context Protocol -- Specification](https://modelcontextprotocol.io/specification/2025-11-25)

#### E4. Agent-to-Agent Protocol (A2A)
The open standard (created by Google, early 2025) for higher-level coordination between autonomous agents, enabling discovery, delegation, and collaboration across platforms. Complements MCP (tool-level) with agent-level interoperability.

- **Key authorities:** Google, Linux Foundation, Anthropic
- **Best single resource:** [A2A Protocol Specification](https://a2a-protocol.org/latest/)

#### E5. Tool Result Processing and Context Efficiency
Processing and filtering tool outputs before they re-enter the context window. Raw tool results are often bloated; effective context engineering truncates, summarizes, or extracts only relevant fields. Tool result clearing is a "low-hanging optimization" for long-running agents.

- **Key authorities:** Anthropic, LangChain
- **Best single resource:** [Anthropic -- Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

---

### F. Agentic Context Engineering

#### F1. Sub-Agent and Multi-Agent Architectures
Distributing tasks across specialized agents with their own clean context windows. A main orchestrator coordinates sub-agents, each performing focused work and returning condensed summaries. Reduces context pollution but increases token usage (up to 15x).

- **Key authorities:** Anthropic, LangChain, Microsoft (AutoGen), MetaGPT, CrewAI
- **Best single resource:** [LangChain -- Context Engineering for Agents (Isolate Section)](https://blog.langchain.com/context-engineering-for-agents/)

#### F2. Context Isolation Patterns
Techniques for separating context across processing boundaries. Includes multi-agent isolation, environment-based isolation (sandboxed code execution), and state schema isolation (selectively exposing fields from runtime state objects to the LLM).

- **Key authorities:** LangChain, Anthropic, E2B
- **Best single resource:** [LangChain -- Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/)

#### F3. Agentic Context Engineering (ACE)
Self-improving context systems where agents evolve their own contexts through generation, reflection, and curation. Treats contexts as "evolving playbooks" that accumulate strategies over time. Outperforms baselines by 8-11% while reducing adaptation latency by 87%.

- **Key authorities:** UC Berkeley, ACE-Agent project
- **Best single resource:** [ACE: Agentic Context Engineering -- Evolving Contexts for Self-Improving Language Models](https://arxiv.org/abs/2510.04618)

#### F4. Agent Planning and Task Decomposition
How agents break complex goals into sub-tasks, each requiring its own context assembly. Includes plan-then-execute patterns, hierarchical planning, and the interplay between planning context and execution context.

- **Key authorities:** Anthropic, LangChain, Harrison Chase (deep agents)
- **Best single resource:** [Harrison Chase on Deep Agents (Sequoia)](https://sequoiacap.com/podcast/context-engineering-our-way-to-long-horizon-agents-langchains-harrison-chase/)

#### F5. Long-Horizon Agent Context Management
Maintaining coherent context across extended agent runs (hours to days). Combines compaction, structured note-taking, memory tools, and sub-agent delegation. Claude playing Pokemon is a notable example: maintaining objectives, maps, and achievements across thousands of steps.

- **Key authorities:** Anthropic, LangChain
- **Best single resource:** [LangChain -- Context Management for Deep Agents](https://blog.langchain.com/context-management-for-deepagents/)

#### F6. Context Poisoning and Self-Correction
When hallucinations or errors enter the context and compound over subsequent reasoning steps. The model treats its own prior (incorrect) output as ground truth. Mitigation requires self-correction mechanisms, context validation, and structured reflection.

- **Key authorities:** LangChain, Anthropic, OpenAI
- **Best single resource:** [LangChain -- Context Engineering for Agents (Context Poisoning Section)](https://blog.langchain.com/context-engineering-for-agents/)

---

### G. Platform-Specific Context Systems

#### G1. Anthropic Claude -- Context Architecture
Claude's context management: system prompts, 200K-1M token windows, prompt caching (90% cost reduction), tool use, MCP integration, memory features, and the context engineering philosophy outlined in Anthropic's engineering blog.

- **Key authorities:** Anthropic
- **Best single resource:** [Anthropic -- Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

#### G2. Claude Code -- Skills, CLAUDE.md, Hooks, and Subagents
Claude Code's complete context engineering stack: CLAUDE.md project memory (loaded automatically), Skills (modular instruction packs loaded on-demand with a 2% context budget), subagents (fresh context windows for delegated tasks), hooks (deterministic callbacks), and slash commands.

- **Key authorities:** Anthropic
- **Best single resource:** [Claude Code -- Extend Claude with Skills](https://code.claude.com/docs/en/skills)

#### G3. OpenAI -- Assistants API, Threads, and Function Calling
OpenAI's managed context system: persistent Threads with automatic truncation (100K message limit), truncation strategies (auto vs. last_messages), function calling with structured outputs, and the Agents SDK context management layer.

- **Key authorities:** OpenAI
- **Best single resource:** [OpenAI -- Assistants API Deep Dive](https://developers.openai.com/api/docs/assistants/deep-dive/)

#### G4. Google Gemini -- Long Context and Context Caching
Gemini's long-context capabilities (up to 2M tokens, Gemini 3 targeting 10M), explicit and implicit context caching (90% cost reduction on 2.5+ models), and optimization strategies for large-document analysis workflows.

- **Key authorities:** Google DeepMind
- **Best single resource:** [Google -- Context Caching (Gemini API)](https://ai.google.dev/gemini-api/docs/caching)

#### G5. AI Coding Tools -- Context in Cursor, Copilot, Claude Code
How coding assistants manage codebase context: open file prioritization, semantic code search, AST-aware chunking, rules files (.cursorrules, CLAUDE.md, copilot-instructions.md), and project-level context assembly.

- **Key authorities:** Anthropic (Claude Code), Cursor, GitHub (Copilot), Augment Code
- **Best single resource:** [Claude Code -- Best Practices](https://code.claude.com/docs/en/best-practices)

---

### H. Evaluation, Observability, and Quality

#### H1. Context Quality Evaluation
Measuring how well context serves the model. Metrics include context precision, context recall, faithfulness, answer relevance, and groundedness. Frameworks like RAGAS formalize RAG evaluation.

- **Key authorities:** RAGAS project, Langfuse, LangSmith
- **Best single resource:** [Langfuse -- Evaluating LLM Applications: A Comprehensive Roadmap](https://langfuse.com/blog/2025-11-12-evals)

#### H2. LLM Observability and Tracing
Monitoring the complete lifecycle of LLM interactions: prompt traces, tool call sequences, token usage, latency, error rates, and cost tracking. Distributed tracing is the backbone, especially for RAG and multi-agent systems.

- **Key authorities:** LangSmith, Langfuse, Datadog, Opik
- **Best single resource:** [OpenTelemetry -- Introduction to Observability for LLM-based Applications](https://opentelemetry.io/blog/2024/llm-observability/)

#### H3. Benchmarks for Context Engineering
Standardized evaluations: GAIA (general AI assistants), SWE-Bench (code agents), WebArena (web agents), BFCL (function calling), LongMemEval (memory systems), JSONSchemaBench (structured output), and the emerging MCP-RADAR for tool use assessment.

- **Key authorities:** Princeton, Stanford, UC Berkeley, Anthropic
- **Best single resource:** [A Survey of Context Engineering for LLMs -- Section 6](https://arxiv.org/html/2507.13334v1)

---

### I. Security, Safety, and Governance

#### I1. Prompt Injection and Context Security
Ranked first in OWASP's 2025 Top 10 for LLMs. Includes direct injection (overriding system prompts), indirect injection (poisoning retrieved documents), and RAG poisoning (5 crafted documents can manipulate responses 90% of the time). No complete fix exists; defense-in-depth is the only viable strategy.

- **Key authorities:** OWASP, Microsoft Security, Lakera
- **Best single resource:** [OWASP -- LLM01:2025 Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)

#### I2. Grounding and Hallucination Mitigation
Ensuring model outputs are factually grounded in the provided context. Includes retrieval-based grounding, citation generation, factual consistency checks, and small-model guardrails that verify outputs against source documents.

- **Key authorities:** Anthropic, OpenAI, Google, Portkey
- **Best single resource:** [Portkey -- LLM Hallucinations in Production](https://portkey.ai/blog/llm-hallucinations-in-production/)

#### I3. Context Access Control and Data Governance
Managing what information is allowed to reach the model, especially in enterprise settings. Includes PII filtering, role-based context access, document-level permissions in RAG, and compliance with regulations (HIPAA, GDPR, EU AI Act).

- **Key authorities:** OWASP, Microsoft, enterprise RAG vendors
- **Best single resource:** [A Vision for Access Control in LLM-based Agent Systems](https://arxiv.org/html/2510.11108v1)

#### I4. Ethical Context Design -- Bias, Fairness, and Transparency
How context composition affects model bias: curating balanced retrieval corpora, auditing system prompts for implicit bias, ensuring diverse few-shot examples, and maintaining transparency about what context the model received.

- **Key authorities:** NIST, EU AI regulatory bodies, academic AI ethics groups
- **Best single resource:** [Towards Trustworthy AI: A Review of Ethical and Robust LLMs](https://arxiv.org/html/2407.13934v1)

---

### J. Infrastructure and Economics

#### J1. Token Economics and Cost Optimization
Managing the financial cost of context: input vs output token pricing (output costs 3-5x more), prompt caching savings (up to 90%), model cascading (routing to cheaper models for simple tasks), batch processing discounts (50%+), and quantization techniques.

- **Key authorities:** Anthropic, OpenAI, Google, Introl
- **Best single resource:** [Introl -- Cost Per Token Analysis](https://introl.com/blog/cost-per-token-llm-inference-optimization)

#### J2. Tokenization and Token Counting
Understanding how text maps to tokens: BPE, SentencePiece, WordPiece algorithms. Accurate token counting matters for context budgeting. Different models use different tokenizers, so the same text has different token counts across providers.

- **Key authorities:** Hugging Face, Google (SentencePiece), OpenAI (tiktoken)
- **Best single resource:** [Hugging Face -- Tokenization Algorithms](https://huggingface.co/docs/transformers/tokenizer_summary)

#### J3. Prompt Management Systems
Production infrastructure for treating prompts as versioned software artifacts. Includes version control, templating (Jinja2), A/B testing, approval workflows, release labels (dev/staging/prod), and rollback capabilities.

- **Key authorities:** PromptLayer, Langfuse, Agenta, Braintrust
- **Best single resource:** [Agenta -- The Definitive Guide to Prompt Management Systems](https://agenta.ai/blog/the-definitive-guide-to-prompt-management-systems)

#### J4. Context Window Size and Model Selection
Choosing models based on context window requirements. Current landscape: GPT-5 (1M+), Claude Opus 4.6 (1M), Gemini 2.5 Pro (2M), Llama 4 (10M). Larger windows do not inherently raise per-token costs but increase the risk of context rot and runaway costs.

- **Key authorities:** Anthropic, OpenAI, Google, Meta
- **Best single resource:** [AIMultiple -- Best LLMs for Extended Context Windows in 2026](https://aimultiple.com/ai-context-window)

---

### K. Multimodal and Specialized Context

#### K1. Multimodal Context Integration
Managing context that spans text, images, audio, and video. Includes multimodal encoders, cross-attention mechanisms, vision-language token compression, and the challenges of modality bias (models defaulting to text over other modalities).

- **Key authorities:** Google DeepMind (Gemini), OpenAI (GPT-4o), Anthropic (Claude Vision)
- **Best single resource:** [IBM -- What is a Multimodal LLM?](https://www.ibm.com/think/topics/multimodal-llm)

#### K2. Code Context Engineering
Specific patterns for code: AST-aware chunking, semantic code search, repository-level context assembly, test-implementation file linking, import graph traversal, and project rules files (CLAUDE.md, .cursorrules).

- **Key authorities:** Anthropic (Claude Code), GitHub (Copilot), Cursor
- **Best single resource:** [Claude Code -- Best Practices](https://code.claude.com/docs/en/best-practices)

#### K3. Domain-Specific Context (Vertical AI)
Tailoring context engineering for specific industries: healthcare (HIPAA-compliant RAG, medical coding context), legal (contract analysis, regulatory context), finance (compliance monitoring, temporal market data), and manufacturing (technical documentation retrieval).

- **Key authorities:** IBM, enterprise AI vendors, domain-specific startups
- **Best single resource:** [IBM -- What Are Vertical AI Agents?](https://www.ibm.com/think/topics/vertical-ai-agents)

#### K4. Temporal Context and Time-Aware Reasoning
Injecting time awareness into inherently atemporal models. Includes current date/time injection, temporal knowledge graphs (Graphiti), event timestamping, fact invalidation, and reasoning over temporal relationships.

- **Key authorities:** Zep (Graphiti), Cognee
- **Best single resource:** [Cognee -- Temporal Cognification: Time-Aware AI Memory](https://www.cognee.ai/blog/cognee-news/unlock-your-llm-s-time-awareness-introducing-temporal-cognification)

#### K5. Relational and Structured Data Context
Techniques for incorporating structured data (databases, spreadsheets, APIs) into LLM context. Includes text-to-SQL, table serialization, StructGPT-style approaches, and knowledge graph verbalization.

- **Key authorities:** Microsoft Research, academic NLP groups
- **Best single resource:** [Survey of Context Engineering for LLMs -- Section 4.2.4](https://arxiv.org/html/2507.13334v1)

#### K6. User Modeling and Personalization Context
Building and maintaining user profiles within agent context: preference tracking, behavioral modeling, user embeddings (USER-LLM), and adaptive response generation based on individual interaction history.

- **Key authorities:** Google (User-LLM), ChatGPT Memories team, Zep
- **Best single resource:** [User-LLM: Efficient LLM Contextualization with User Embeddings](https://dl.acm.org/doi/abs/10.1145/3701716.3715463)

---

### L. Emerging and Frontier Areas

#### L1. Self-Improving and Evolving Contexts
Agents that autonomously refine their own system prompts, instructions, and memory over time. The ACE framework's Generator-Reflector-Curator pipeline. Represents a shift from static to dynamic context design.

- **Key authorities:** ACE-Agent project, Anthropic
- **Best single resource:** [ACE: Agentic Context Engineering](https://arxiv.org/abs/2510.04618)

#### L1b. Agent Self-Optimization and Automatic Tool Generation
Agents that detect slow operations and auto-generate scripts/tools for future invocations. Covers plan caching (APC), self-programming agents (OpenSage), and practical hook-based patterns for Claude Code. The core insight: agents spend 78-92% of time on LLM inference, not tool execution -- scripting repeated operations yields 25-100x speedups.

- **Key authorities:** NeurIPS 2025 (APC), OpenSage, HAI Operators empirical benchmarks
- **Best single resource:** [[agent-self-optimization-and-tool-generation|Agent Self-Optimization and Tool Generation]]
- **Key papers:** [Agentic Plan Caching (arXiv:2506.14852)](https://arxiv.org/abs/2506.14852), [OpenSage (arXiv:2602.16891)](https://arxiv.org/html/2602.16891v1)

#### L2. Context Engineering for Multi-Agent Coordination
How groups of agents share, negotiate, and synchronize context. Includes communication protocols (KQML, FIPA ACL, MCP, A2A, ACP, ANP), orchestration frameworks (AutoGen, CrewAI, MetaGPT), and the emerging challenge of transactional integrity across agent boundaries.

- **Key authorities:** Google (A2A), Anthropic (MCP), Microsoft (AutoGen), Linux Foundation
- **Best single resource:** [O'Reilly -- Designing Collaborative Multi-Agent Systems with the A2A Protocol](https://www.oreilly.com/radar/designing-collaborative-multi-agent-systems-with-the-a2a-protocol/)

#### L3. Information-Theoretic Foundations
Formalizing context engineering mathematically: context as an optimization problem (minimize tokens while maximizing mutual information with desired output), Bayesian context inference, and attention budget theory.

- **Key authorities:** Academic survey authors (Mei et al., 2025)
- **Best single resource:** [A Survey of Context Engineering for LLMs -- Section 3 (Formulations)](https://arxiv.org/html/2507.13334v1)

#### L4. Context Scaling Dimensions
Beyond token length: scaling context across modalities (text + image + audio + video), temporal dimensions (time-series context), spatial dimensions (geographic/geometric context), participant states (multi-entity tracking), and intentional context (goals and motivations).

- **Key authorities:** Academic survey authors (Mei et al., 2025)
- **Best single resource:** [A Survey of Context Engineering for LLMs](https://arxiv.org/abs/2507.13334)

#### L5. Inference-Time Compute and Context
The relationship between context quality and inference-time reasoning. Extended thinking, test-time compute scaling, and the observation that smarter models need less prescriptive context engineering but benefit from better-curated context.

- **Key authorities:** OpenAI (o1/o3 series), Anthropic (extended thinking), Google (Gemini thinking)
- **Best single resource:** [Sebastian Raschka -- The State of LLMs 2025](https://magazine.sebastianraschka.com/p/state-of-llms-2025)

#### L6. Real-Time and Streaming Context
Managing context in real-time applications: voice assistants, live coding, and streaming scenarios. Includes OpenAI's context summarization for the Realtime API and techniques for maintaining context continuity during live interactions.

- **Key authorities:** OpenAI (Realtime API)
- **Best single resource:** [OpenAI -- Context Summarization with Realtime API](https://developers.openai.com/cookbook/examples/context_summarization_with_realtime_api/)

#### L7. Context Engineering as Software Engineering Practice
Treating context engineering as a first-class software discipline: spec-driven development, test-driven context design, CI/CD for prompts, context regression testing, and the integration of context engineering into development workflows.

- **Key authorities:** Simon Willison, Harrison Chase, Anthropic
- **Best single resource:** [Simon Willison -- Agentic Engineering Patterns](https://simonw.substack.com/p/agentic-engineering-patterns)

---

## Part 2: Leading Authorities

### Individuals

| Person | Affiliation | Contribution |
|--------|------------|--------------|
| **Andrej Karpathy** | Former OpenAI/Tesla | Popularized the term; defined context engineering as "the delicate art and science of filling the context window" |
| **Tobi Lutke** | Shopify CEO | Coined the mainstream usage; catalyzed adoption across the tech industry |
| **Harrison Chase** | LangChain CEO | Built a widely-used framework ecosystem; authored the four-strategy taxonomy (write/select/compress/isolate); advocates "deep agents" and harness engineering |
| **Simon Willison** | Independent (Django co-creator) | Prolific blogger; agentic engineering patterns; early advocate for the term sticking |
| **Chip Huyen** | Author, AI Engineering | Production LLM systems design; "AI Engineering" book covers context as first-class concern |
| **Lilian Weng** | OpenAI | Foundational "LLM-powered Autonomous Agents" survey covering memory, planning, and tool use |
| **Sebastian Raschka** | Lightning AI | State-of-LLMs annual reviews; practical context optimization guidance |
| **Leonie Monigatti** | Weaviate | Clear explainers on memory architectures, MemGPT, and RAG patterns |
| **Preston Rasmussen** | Zep | Temporal knowledge graph architecture for agent memory |

### Organizations

| Organization | Role in Context Engineering |
|-------------|---------------------------|
| **Anthropic** | Published a widely-referenced practitioner guide; created MCP; built Claude Code's context architecture (Skills, CLAUDE.md, subagents) |
| **LangChain** | Built a widely-adopted framework ecosystem (LangChain, LangGraph, LangMem, LangSmith); published the four-strategy taxonomy; maintains Bigtool and context engineering tutorials |
| **OpenAI** | Assistants API/Threads for managed context; function calling standards; Realtime API context management; o-series models pushing inference-time reasoning |
| **Google DeepMind** | Largest context windows (Gemini, 2M+ tokens); context caching pioneer; A2A protocol; long-context research |
| **Chroma** | Context Rot research; foundational work on understanding context degradation |
| **Zep** | Agent memory platform; temporal knowledge graphs (Graphiti); context engineering as core product thesis |
| **Letta (MemGPT)** | OS-inspired virtual context management; pioneered the "LLMs as operating systems" paradigm |
| **Meta AI** | Original RAG paper; Llama models with large context windows; open-source long-context research |
| **Linux Foundation (AAIF)** | Governance of MCP and A2A protocols; standardizing agent interoperability |
| **DAIR.AI** | Prompt Engineering Guide; community-maintained reference covering context engineering fundamentals |

---

## Part 3: Key Resources (Top 15 Definitive References)

### Practitioner Guides

1. **Anthropic -- "Effective Context Engineering for AI Agents"** (Sep 2025)
   A widely-referenced practitioner guide. Covers system prompts, tools, examples, context retrieval, progressive disclosure, compaction, note-taking, and sub-agents.
   https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

2. **LangChain -- "Context Engineering for Agents"** (Jun 2025)
   Establishes the four-strategy framework (write, select, compress, isolate) with implementation examples from production systems.
   https://blog.langchain.com/context-engineering-for-agents/

3. **LangChain -- "Context Management for Deep Agents"** (2025)
   Extends the four strategies to long-horizon agent scenarios with practical LangGraph implementation patterns.
   https://blog.langchain.com/context-management-for-deepagents/

4. **Claude Code Documentation -- Skills and Best Practices**
   Definitive reference for Claude Code's context architecture: CLAUDE.md, Skills, hooks, subagents, and context budget management.
   https://code.claude.com/docs/en/skills

5. **Prompting Guide -- Context Engineering Guide**
   Community-maintained comprehensive overview with cross-provider coverage.
   https://www.promptingguide.ai/guides/context-engineering-guide

### Academic Surveys

6. **Mei et al. -- "A Survey of Context Engineering for Large Language Models"** (Jul 2025, arXiv:2507.13334)
   A broad academic survey. 1,400+ papers analyzed. Establishes formal taxonomy: context retrieval/generation, context processing, context management, and system implementations (RAG, memory, tools, multi-agent).
   https://arxiv.org/abs/2507.13334

7. **"Prompt Compression for Large Language Models: A Survey"** (NAACL 2025, Selected Oral)
   Comprehensive taxonomy of compression techniques: selection-based, extraction-based, and generation-based methods.
   https://aclanthology.org/2025.naacl-long.368.pdf

8. **"Memory in the Age of AI Agents: A Survey"** (Dec 2025, arXiv:2512.13564)
   Systematic survey of agent memory architectures: factual, experiential, and working memory with formation, evolution, and retrieval dynamics.
   https://arxiv.org/abs/2512.13564

### Foundational Research

9. **Chroma Research -- "Context Rot"** (Jul 2025)
   Empirical study of 18 frontier models showing performance degradation with increasing context length, even on simple tasks. Defines the context rot phenomenon.
   https://research.trychroma.com/context-rot

10. **Packer et al. -- "MemGPT: Towards LLMs as Operating Systems"** (Oct 2023)
    Pioneered virtual context management with OS-inspired memory tiering. Foundation for the Letta framework.
    https://arxiv.org/abs/2310.08560

11. **"Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models"** (Oct 2025, arXiv:2510.04618)
    Introduces ACE -- agents that evolve their own contexts through generation, reflection, and curation. Achieves 87% reduction in adaptation latency.
    https://arxiv.org/abs/2510.04618

### Protocol Specifications

12. **Model Context Protocol (MCP) -- Specification** (Nov 2025)
    The universal standard for LLM-to-tool communication. JSON-RPC 2.0, client-server architecture, now governed by the Linux Foundation.
    https://modelcontextprotocol.io/specification/2025-11-25

13. **Agent-to-Agent (A2A) Protocol** (2025)
    The open standard for inter-agent communication, enabling discovery, delegation, and collaboration across platforms.
    https://a2a-protocol.org/latest/

### Curated Collections

14. **Awesome Context Engineering** (GitHub)
    Community-curated collection of papers, tools, frameworks, and best practices. Continuously updated.
    https://github.com/Meirtz/Awesome-Context-Engineering

15. **Zep -- "What is Context Engineering, Anyway?"**
    Clear, opinionated definition piece that explains why context engineering matters from an agent memory platform perspective.
    https://blog.getzep.com/what-is-context-engineering/

---

## Part 4: Quick-Reference Taxonomy

```
Context Engineering
|
+-- A. Foundational Context Design
|   +-- System Prompt Architecture
|   +-- Prompt Engineering (subset)
|   +-- Few-Shot / In-Context Learning
|   +-- Chain-of-Thought / Reasoning Context
|   +-- Structured Output / Constrained Generation
|   +-- Document Ordering (Lost in the Middle)
|
+-- B. Context Retrieval & Knowledge Integration
|   +-- RAG (Retrieval-Augmented Generation)
|   +-- Advanced RAG (Modular, Agentic, Multi-hop)
|   +-- GraphRAG / Knowledge Graphs
|   +-- Embedding Models / Vector Search
|   +-- Chunking Strategies
|   +-- Contextual Compression / Re-Ranking
|   +-- Progressive Disclosure / Just-in-Time Retrieval
|
+-- C. Context Processing & Optimization
|   +-- Context Compression Techniques
|   +-- Summarization / Compaction
|   +-- KV Cache / Token Compression
|   +-- Prefix Caching / Prompt Caching
|   +-- Long Context Processing (Architectures)
|   +-- Context Rot / Degradation
|
+-- D. Memory Systems & State
|   +-- Short-Term (Working) Memory
|   +-- Long-Term Memory (Semantic/Episodic/Procedural)
|   +-- Virtual Context Management (MemGPT)
|   +-- Temporal Knowledge Graphs
|   +-- Scratchpads / Working Notes
|   +-- Cross-Session Memory / Personalization
|   +-- Multi-Turn Conversation Management
|
+-- E. Tool Use & External Context
|   +-- Tool Definition / Function Calling
|   +-- Tool Discovery / Dynamic Selection
|   +-- Model Context Protocol (MCP)
|   +-- Agent-to-Agent Protocol (A2A)
|   +-- Tool Result Processing
|
+-- F. Agentic Context Engineering
|   +-- Sub-Agent / Multi-Agent Architectures
|   +-- Context Isolation Patterns
|   +-- Agentic Context Engineering (ACE)
|   +-- Agent Planning / Task Decomposition
|   +-- Long-Horizon Context Management
|   +-- Context Poisoning / Self-Correction
|
+-- G. Platform-Specific Systems
|   +-- Anthropic Claude Context Architecture
|   +-- Claude Code (Skills, CLAUDE.md, Hooks)
|   +-- OpenAI Assistants API / Threads
|   +-- Google Gemini Long Context / Caching
|   +-- AI Coding Tools (Cursor, Copilot)
|
+-- H. Evaluation & Observability
|   +-- Context Quality Evaluation
|   +-- LLM Observability / Tracing
|   +-- Benchmarks (GAIA, SWE-Bench, BFCL, etc.)
|
+-- I. Security, Safety, & Governance
|   +-- Prompt Injection / Context Security
|   +-- Grounding / Hallucination Mitigation
|   +-- Access Control / Data Governance
|   +-- Ethical Context Design (Bias, Fairness)
|
+-- J. Infrastructure & Economics
|   +-- Token Economics / Cost Optimization
|   +-- Tokenization / Token Counting
|   +-- Prompt Management Systems
|   +-- Context Window Size / Model Selection
|
+-- K. Multimodal & Specialized
|   +-- Multimodal Context (Image/Audio/Video)
|   +-- Code Context Engineering
|   +-- Domain-Specific (Healthcare/Legal/Finance)
|   +-- Temporal Context / Time-Aware Reasoning
|   +-- Structured Data Context
|   +-- User Modeling / Personalization
|
+-- L. Emerging & Frontier
    +-- Self-Improving / Evolving Contexts
    +-- Multi-Agent Coordination Protocols
    +-- Information-Theoretic Foundations
    +-- Context Scaling Dimensions
    +-- Inference-Time Compute & Context
    +-- Real-Time / Streaming Context
    +-- Context Engineering as SW Engineering
```

---

*This index covers 50+ distinct areas across 12 categories. The field is evolving rapidly -- Gartner predicts 40% of enterprise apps will feature AI agents by late 2026, all requiring robust context engineering. The discipline sits at the intersection of information retrieval, systems engineering, cognitive science, and software architecture.*
