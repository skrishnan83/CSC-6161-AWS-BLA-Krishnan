Agents and Performance on AWS: Troubleshooting Guide

LinkedIn Activity:  https://lnkd.in/p/gg336Kva , 
                    https://lnkd.in/p/gEfy3V8v , 
                    https://lnkd.in/p/giMRMuvT

View Tutorial on YouTube: https://youtu.be/SDZXeqYZLs0?si=2f73JaqeYLY2zsRe , 
                          https://youtu.be/ZtWkJ31yCSQ?si=sfvopwTULw_eCN53 , 
                          https://youtu.be/3EPwcnZ73ok?si=7gQOPxciSt1aiP1I

Three troubleshooting walkthroughs based on the labs and lessons of the AWS Certified Generative AI Developer – Professional course (Agentic AI and Operational Efficiency and Optimization).

Scenario-based walkthroughs written from the course labs and AWS documentation. 

1: Agent Squad Sends the Question to the Wrong Agent

  Lab: Strands Agents, Amazon Bedrock AgentCore, Agent Squad 
  Concepts: multi-agent workflows, Agent Squad, intent classification, agent descriptions

  Scenario

  Three agents are registered with Agent Squad: Billing, Orders and Tech Support. A customer asks "I was charged twice for order A100 — can I get one payment back?"

  Symptoms

  The question goes to Tech Support, which replies with troubleshooting steps for the website. Other billing questions are routed correctly, so the problem looks random.

  Diagnosis

  Agent Squad's classifier picks an agent by comparing the question with each agent's description. Print which agent was chosen:

        python
        response = await orchestrator.route_request(user_input, user_id, session_id)
        print(response.metadata.agent_name)

  Then look at the descriptions that were registered:

        python
        BedrockLLMAgentOptions(name="Billing Agent", description="Helps customers with their account.")
        BedrockLLMAgentOptions(name="Tech Support Agent", description="Helps customers with problems.")

  "Charged twice" sounds like a problem, and both descriptions are vague — so the classifier has nothing to separate them on.

  Fix

  Make each description say what the agent handles and when to use it, with clear boundaries:

python
from agent_squad.orchestrator import AgentSquad
from agent_squad.agents import BedrockLLMAgent, BedrockLLMAgentOptions

orchestrator = AgentSquad()

orchestrator.add_agent(BedrockLLMAgent(BedrockLLMAgentOptions(
    name="Billing Agent",
    description="Handles payments, charges, double charges, invoices and refunds. "
                "Use for any question about money the customer paid or is owed.",
)))

orchestrator.add_agent(BedrockLLMAgent(BedrockLLMAgentOptions(
    name="Tech Support Agent",
    description="Handles website and app errors, login problems and password resets. "
                "Do NOT use for payments or refunds.",
)))

  Then keep a short list of test questions with the agent each should reach, and re-run it after any change.

  Verification

  "Charged twice" now routes to Billing Agent. All test questions reach the expected agent.



2: Strands Agent Can't Use Tools from an MCP Server

Lab: Strands Agents, Amazon Bedrock AgentCore, Agent Squad 
· Model Context Protocol (MCP) 
Concepts: Strands Agents, MCP clients and servers, tool discovery

Scenario

  A Strands agent should answer AWS questions using tools from the AWS Documentation MCP server.

python
from mcp import stdio_client, StdioServerParameters
from strands import Agent
from strands.tools.mcp import MCPClient

mcp_client = MCPClient(lambda: stdio_client(StdioServerParameters(
    command="uvx", args=["awslabs.aws-documentation-mcp-server@latest"])))

tools = mcp_client.list_tools_sync()      # called outside the client session
agent = Agent(tools=tools)
agent("What is the maximum Lambda timeout?")
Symptoms

  One of these:

  An error saying the MCP client session is not running
  FileNotFoundError for uvx
  The agent answers from general knowledge and never calls a documentation tool
  Diagnosis
  Symptom	Cause
  Session not running	Tools were listed and used outside the with mcp_client: block — the connection to the server only exists inside it
  FileNotFoundError: uvx	uv isn't installed, so the server can't start
  No tool calls	tools was empty or not passed to the agent

  Print what the agent actually received:

python
with mcp_client:
    tools = mcp_client.list_tools_sync()
    print([t.tool_name for t in tools])

Fix
Install uv (which provides uvx):
bash
   pip install uv
Keep everything that uses the tools inside the session:
python
   with mcp_client:
       tools = mcp_client.list_tools_sync()
       agent = Agent(tools=tools)
       agent("What is the maximum Lambda timeout?")

  Verification

The tool list prints several documentation tools. The agent's answer is based on a documentation search, and the log shows a tool call before the final answer.

3: ThrottlingException When Traffic Picks Up

Related lessons: Exponential Backoff and Connection Pooling · Amazon Bedrock Cross-Region Inference · Token Efficiency
Concepts: throttling, retries with jitter, connection pooling, cross-Region inference, token cost

Scenario

An assistant works well in testing. At a busy time, a few hundred users arrive at once, and requests start failing. The bill for the month is also higher than expected.

Symptoms
ThrottlingException: Too many requests, please wait before trying again.
Diagnosis

Check the Bedrock metrics in CloudWatch (AWS/Bedrock):

bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/Bedrock --metric-name InvocationThrottles \
  --dimensions Name=ModelId,Value=MODEL_ID \
  --start-time 2026-09-20T00:00:00Z --end-time 2026-09-21T00:00:00Z \
  --period 3600 --statistics Sum

Repeat for InputTokenCount. Two problems appear:

The code retries immediately. Every client retries at the same moment, so throttling gets worse.
Input tokens per request are high. Each call sends 15 retrieved documents and the whole chat history.
Fix
Retry with backoff, pool connections, and use a cross-Region inference profile:
python
   import boto3
   from botocore.config import Config

   client = boto3.client("bedrock-runtime", config=Config(
       retries={"max_attempts": 5, "mode": "adaptive"},   # backoff between retries
       max_pool_connections=20,                            # reuse connections
   ))
   MODEL_ID = "us.amazon.nova-lite-v1:0"                   # can route across US Regions
Send fewer tokens — retrieve 3 documents instead of 15, and summarise old chat turns.
Cache a long, repeated system prompt on models that support prompt caching:
python
   system=[{"text": LONG_SYSTEM_PROMPT}, {"cachePoint": {"type": "default"}}]
Verification

InvocationThrottles drops close to zero and average InputTokenCount falls. On supported models, repeat calls show cacheReadInputTokens in the response's usage.

Cleanup (avoid surprise charges)
AgentCore runtimes and their ECR images
OpenSearch Serverless collections created by the Bedrock Agents lab (lecture 101)
Amazon Q Business applications (lecture 122) and any Amazon Quick resources
Common debugging checklist for agents and performance
See what was chosen — which agent or tool, with what input.
Read the descriptions — agents and tools are picked by their descriptions.
Check the tool list — an agent can't call a tool it never received.
Keep MCP work inside the session.
Watch the metrics — InvocationThrottles, InputTokenCount, OutputTokenCount, InvocationLatency.
Back off, don't hammer — retries need a growing wait.
