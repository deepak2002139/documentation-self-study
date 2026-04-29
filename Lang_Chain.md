# LangChain — A Beginner's Guide 🦜🔗

## What is LangChain?

LangChain is an **open-source Python framework** that makes it easy to build applications powered by Large Language Models (LLMs) like OpenAI's GPT.

Think of it this way: an LLM (like ChatGPT) can understand and generate text, but on its own it can't search the web, read your files, or remember past conversations. **LangChain connects the LLM to the outside world** so it can actually *do* useful things.

---

## Why Use LangChain?

Without LangChain, you'd have to write a lot of glue code yourself — handling prompts, chaining calls together, managing memory, connecting to APIs, etc.

LangChain gives you **ready-made building blocks** so you can:

- Send well-structured prompts to an LLM
- Chain multiple steps together (ask → process → act)
- Give the LLM access to tools (search, calculators, databases)
- Let the LLM make decisions on its own (agents)
- Remember previous parts of a conversation (memory)

---

## Core Concepts (Explained Simply)

### 1. LLMs (Large Language Models)

An LLM is the AI brain — a model trained on massive amounts of text that can understand and generate language. Examples: GPT-4, Claude, Llama.

LangChain lets you swap between different LLMs without rewriting your code.

### 2. Prompts

A prompt is the **instruction or question** you send to the LLM. LangChain provides **prompt templates** so you can reuse prompts with different inputs.

```
"Summarize the following article in 3 bullet points: {article_text}"
```

The `{article_text}` part gets filled in automatically each time.

### 3. Chains

A chain is a **sequence of steps** that run one after another. The output of one step becomes the input of the next.

**Example:** Take user question → format it into a prompt → send to LLM → return the answer.

Chains always follow a **fixed path** — step 1, then step 2, then step 3.

### 4. Agents

An agent is like a chain, but **smarter**. Instead of following fixed steps, the agent **decides on its own** what to do next based on the situation.

> Think of a chain as a recipe (follow steps in order).
> Think of an agent as a chef (decides what to do based on what's available).

### 5. Tools

Tools are **abilities** you give to an agent. Examples:

- A **search tool** to look things up on the web
- A **calculator** to do math
- A **database tool** to query data

The agent decides *which* tool to use and *when*.

### 6. Memory

By default, each call to an LLM is independent — it doesn't remember what you said before. **Memory** lets LangChain store conversation history so the LLM can reference earlier messages.

---

## How LangChain Helps Build AI Agents

Building an AI agent from scratch is hard. You'd need to:

1. Send a prompt to the LLM
2. Parse the LLM's response to figure out what it wants to do
3. Run the right tool based on that response
4. Feed the tool's result back to the LLM
5. Repeat until the LLM has a final answer

LangChain handles **all of this for you**. You just define the tools and the LLM, and LangChain manages the reasoning loop.

```
User asks a question
        ↓
Agent thinks: "I need to search the web for this"
        ↓
Agent uses the Search tool
        ↓
Agent reads the result
        ↓
Agent thinks: "Now I have enough info to answer"
        ↓
Agent gives the final answer
```

---

## Chain vs Agent — What's the Difference?

| Feature        | Chain                        | Agent                              |
| -------------- | ---------------------------- | ---------------------------------- |
| Steps          | Fixed, predefined            | Dynamic, decided at runtime        |
| Decision-making| None — follows a set path    | Yes — the LLM chooses what to do   |
| Tools          | Not typically used           | Can pick and use tools             |
| Best for       | Simple, predictable tasks    | Complex tasks needing flexibility  |

---

## Real-World Use Cases

- **Customer support chatbot** — answers questions using your company's docs
- **Research assistant** — searches the web and summarises findings
- **Data analyst** — queries a database and explains results in plain English
- **Code helper** — reads code files and suggests fixes
- **Personal assistant** — books meetings, sends emails, looks up information

---

## Installation

Make sure you have Python 3.9 or higher installed.

```bash
# Install LangChain
pip install langchain

# Install the OpenAI integration (most common LLM provider)
pip install langchain-openai

# Install community tools (search, Wikipedia, etc.)
pip install langchain-community
```

You'll also need an **OpenAI API key**. Get one at [platform.openai.com](https://platform.openai.com/).

Set it as an environment variable:

```bash
# Linux / macOS
export OPENAI_API_KEY="your-api-key-here"

# Windows (Command Prompt)
set OPENAI_API_KEY=your-api-key-here
```

---

## Python Examples

### Example 1 — Basic LLM Call

The simplest thing you can do: send a question to the LLM and get an answer.

```python
from langchain_openai import ChatOpenAI

# 1. Create an LLM instance (uses your OPENAI_API_KEY automatically)
llm = ChatOpenAI(model="gpt-4o-mini")

# 2. Send a message and get a response
response = llm.invoke("What is Python in one sentence?")

# 3. Print the response
print(response.content)
```

---

### Example 2 — Using a Prompt Template + Chain

Here we create a reusable prompt and chain it with the LLM.

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# 1. Create the LLM
llm = ChatOpenAI(model="gpt-4o-mini")

# 2. Create a reusable prompt template
#    {topic} is a placeholder that we fill in later
prompt = ChatPromptTemplate.from_template(
    "Explain {topic} in 3 simple bullet points for a beginner."
)

# 3. Create a chain: prompt → LLM
#    The "|" operator connects them (like a pipe)
chain = prompt | llm

# 4. Run the chain with a specific topic
response = chain.invoke({"topic": "machine learning"})

print(response.content)
```

**What happens under the hood:**
1. `{topic}` is replaced with `"machine learning"`
2. The full prompt is sent to the LLM
3. The LLM's answer is returned

---

### Example 3 — Simple Agent with Tools

This is where it gets interesting. We give the LLM a tool (a math calculator) and let it **decide on its own** whether to use it.

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

# 1. Create the LLM
llm = ChatOpenAI(model="gpt-4o-mini")

# 2. Define a custom tool using the @tool decorator
#    The docstring tells the agent WHEN to use this tool
@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers together."""
    return a * b

@tool
def add(a: int, b: int) -> int:
    """Add two numbers together."""
    return a + b

# 3. Create an agent and give it the tools
tools = [multiply, add]
agent = create_react_agent(llm, tools)

# 4. Ask the agent a question
#    The agent will DECIDE which tool to use (or none at all)
response = agent.invoke(
    {"messages": [{"role": "user", "content": "What is 6 multiplied by 7?"}]}
)

# 5. Print the final answer
for message in response["messages"]:
    print(f"{message.type}: {message.content}")
```

**What the agent does internally:**
1. Reads the question: *"What is 6 multiplied by 7?"*
2. Thinks: *"I have a `multiply` tool — I should use it"*
3. Calls `multiply(6, 7)` → gets `42`
4. Responds: *"6 multiplied by 7 is 42"*

> **Note:** Install `langgraph` for the agent example:
> ```bash
> pip install langgraph
> ```

---

### Example 4 — Agent with Memory (Conversation)

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import MemorySaver

# 1. Create the LLM
llm = ChatOpenAI(model="gpt-4o-mini")

# 2. Create a memory saver so the agent remembers past messages
memory = MemorySaver()

# 3. Create the agent with memory enabled
agent = create_react_agent(llm, tools=[], checkpointer=memory)

# 4. We use a "thread_id" to group messages into one conversation
config = {"configurable": {"thread_id": "my-conversation"}}

# 5. First message
response = agent.invoke(
    {"messages": [{"role": "user", "content": "My name is Deepak."}]},
    config=config,
)
print(response["messages"][-1].content)

# 6. Second message — the agent remembers the first one!
response = agent.invoke(
    {"messages": [{"role": "user", "content": "What is my name?"}]},
    config=config,
)
print(response["messages"][-1].content)
# Output: "Your name is Deepak."
```

---

## Summary

| Concept  | One-Line Explanation                                      |
| -------- | --------------------------------------------------------- |
| LLM      | The AI brain that understands and generates text           |
| Prompt   | The instruction you give to the LLM                        |
| Chain    | A fixed sequence of steps (prompt → LLM → output)         |
| Agent    | A smart chain that decides its own steps at runtime        |
| Tool     | An ability you give to an agent (search, math, etc.)       |
| Memory   | Lets the LLM remember earlier parts of a conversation      |

**LangChain = the framework that connects all of these together.**

---

## Next Steps

1. 📖 **Official Docs** — [python.langchain.com/docs](https://python.langchain.com/docs/introduction/)
2. 🧪 **LangSmith** — Debug and trace your chains/agents: [smith.langchain.com](https://smith.langchain.com/)
3. 🕸️ **LangGraph** — Build more advanced, stateful agents: [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/)
4. 💡 **Try building**:
   - A chatbot that answers questions from a PDF
   - A research agent that searches the web
   - A SQL agent that queries a database using plain English

---

*Built with ❤️ for beginners. Happy building!*
