# AI Engineer — Easy Questions

[← Back to AI_Engineer](../README.md)

Questions marked **(added)** were not in the original list. I added them to fill gaps.

## MCP, Agent Safety & Evaluation

| # | Question | Answer |
|---|----------|--------|
| 1 | What is MCP, and why is it used? | [Answer](ans/01_what_is_mcp.md) |
| 2 | How would you design an MCP server with authentication, authorization, input validation, and audit logging? | [Answer](ans/02_mcp_server_auth_validation_audit.md) |
| 3 | How would you securely connect an MCP server to external services such as Git repositories? | [Answer](ans/03_mcp_secure_external_services_git.md) |
| 4 | How would you ensure that an agent requires approval before executing a sensitive action? | [Answer](ans/04_agent_approval_sensitive_actions.md) |
| 5 | An agent repeatedly calls the same tool. What could cause this infinite loop, and how would you prevent it? | [Answer](ans/05_agent_infinite_tool_loop.md) |
| 6 | Is there a standard design approach for making tools or APIs understandable to an LLM, including during failures? | [Answer](ans/06_designing_llm_friendly_tools.md) |
| 7 | Have you worked with evaluation frameworks such as Ragas or DeepEval? | [Answer](ans/07_ragas_deepeval.md) |
| 8 | What guardrails have you implemented for applications built with Amazon Bedrock? | [Answer](ans/08_bedrock_guardrails.md) |
| 9 | How would you enforce least-privilege access so an AI application cannot perform unauthorized actions? | [Answer](ans/09_least_privilege_ai_apps.md) |
| 20 | **(added)** What are MCP's transports and primitives, and when do you use each? | [Answer](ans/20_mcp_transports_primitives.md) |
| 21 | **(added)** MCP vs native function calling vs A2A (Agent2Agent) protocol: how do they differ? | [Answer](ans/21_mcp_vs_function_calling_vs_a2a.md) |
| 29 | **(added)** How does LLM tool/function calling actually work under the hood? | [Answer](ans/29_tool_calling_how_it_works.md) |

## LangGraph & Multi-Agent Systems

| # | Question | Answer |
|---|----------|--------|
| 10 | How would you implement human approval before executing a tool or node in LangGraph? | [Answer](ans/10_langgraph_human_approval.md) |
| 11 | How does an agent initiate a human-approval request in LangGraph? | [Answer](ans/11_langgraph_agent_initiates_approval.md) |
| 12 | What is checkpointing in LangGraph? | [Answer](ans/12_langgraph_checkpointing.md) |
| 13 | Where is checkpointed state stored, and how is it retrieved? | [Answer](ans/13_checkpoint_storage_retrieval.md) |
| 14 | How would you design a LangGraph state class so new values are appended or merged instead of replacing existing values? | [Answer](ans/14_langgraph_state_reducers.md) |
| 15 | How would you coordinate a multi-agent system containing a triage agent, a root-cause analysis agent, a remediation agent, and a final response agent? | [Answer](ans/15_incident_multi_agent_triage_rca_remediation.md) |
| 16 | How would you design and coordinate multiple agents using LangGraph or another orchestration framework? | [Answer](ans/16_designing_multi_agent_systems.md) |
| 17 | What is the role of a supervisor or master agent in a multi-agent system? | [Answer](ans/17_supervisor_agent_role.md) |
| 18 | How would you maintain shared state and track the work completed by each agent? | [Answer](ans/18_shared_state_tracking_agent_work.md) |
| 19 | What prebuilt agent types are available in LangChain? | [Answer](ans/19_langchain_prebuilt_agents.md) |
| 22 | **(added)** What is the ReAct pattern, and how does it compare to plan-and-execute and reflection? | [Answer](ans/22_react_pattern.md) |
| 23 | **(added)** LangChain vs LangGraph vs LangSmith: what does each do, and when do you choose LangGraph? | [Answer](ans/23_langgraph_vs_langchain.md) |
| 24 | **(added)** Short-term vs long-term memory in LangGraph. | [Answer](ans/24_langgraph_short_vs_long_term_memory.md) |
| 25 | **(added)** How do you run parallel branches / map-reduce in LangGraph with the Send API? | [Answer](ans/25_langgraph_send_api_map_reduce.md) |
| 26 | **(added)** What streaming modes does LangGraph support? | [Answer](ans/26_langgraph_streaming_modes.md) |
| 27 | **(added)** What are subgraphs, and how do state and checkpoints work with them? | [Answer](ans/27_langgraph_subgraphs.md) |
| 28 | **(added)** What is time travel in LangGraph, and how do you fork from a past checkpoint? | [Answer](ans/28_langgraph_time_travel.md) |
