# Build Agents with Fresh Context  
### Elastic Workflows + Tavily Hackathon Resources

Welcome! In this hackathon you’ll build an AI agent that stays up-to-date automatically by combining:

- Tavily Web Search
- Elastic Workflows
- Elastic Agent Builder

Instead of static RAG pipelines that go stale, you’ll create a **live context pipeline** that continuously fetches and indexes fresh information from the web.

By the end of the event, your agent should be able to answer questions using **recent indexed knowledge**

---

# What You’ll Build

Your pipeline will look like this:

Trigger → Tavily search → transform → index into Elasticsearch → retrieve in Agent Builder

Optional advanced version:

Build a custom agent + custom tools in Agent Builder

---

# Prerequisites

Before you start:

- Elastic Cloud Serverless account
- Tavily API key
- Basic familiarity with JSON/YAML helpful but not required

---

# Step 1 — Get Your API Keys/Serverless Account

## Elastic Cloud Serverless

Sign up for a free 14 day trial:

https://cloud.elastic.co/serverless-registration


## Tavily

Create a key here:

https://app.tavily.com

Store it securely.


---

# Step 2 — Create an Elasticsearch Index

Create an index within the dev tools to store fresh web results:

Example mapping:

```json
PUT ai-fresh-context
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "url": { "type": "keyword" },
      "domain": { "type": "keyword" },
      "content": { "type": "semantic_text" },
      "snippet": { "type": "text" },
      "source": { "type": "keyword" },
      "topic": { "type": "keyword" },
      "retrieved_at": { "type": "date" },
      "published_date": { "type": "date" }
    }
  }
}
```

The semantic_text field type enables semantic retrieval inside Agent Builder automatically.

---

# Step 3 — Create an Elastic Workflow

Your workflow should:

1. run on a schedule/trigger
2. call Tavily Search API
3. normalize results
4. index documents into Elasticsearch

Example workflow:

```yaml
name: ai-trends-workflow
enabled: true
description: Fetch fresh AI news for agent retrieval

triggers:
  - type: scheduled
    with:
      every: 12h

steps:
  - name: tavily_search
    type: http
    with:
      url: https://api.tavily.com/search
      method: POST
      headers:
        Authorization: "Bearer TAVILY_API_KEY"
        Content-Type: application/json
      body: |
        {
          "query": "latest AI agent frameworks announcements",
          "search_depth": "advanced",
          "max_results": 5,
          "include_raw_content": true
        }

  - name: index_articles
    type: foreach
    foreach: "{{ steps.tavily_search.output.data.results }}"
    steps:
      - name: upsert_doc
        type: elasticsearch.update
        with:
          index: ai-fresh-context
          id: "{{ foreach.item.url }}"
          doc_as_upsert: true
          doc:
            title: "{{ foreach.item.title }}"
            url: "{{ foreach.item.url }}"
            snippet: "{{ foreach.item.content }}"
            content: "{{ foreach.item.raw_content | default: foreach.item.content }}"
            source: "tavily_search"
            topic: "ai_agents"
            retrieved_at: "{{ 'now' | date: '%Y-%m-%dT%H:%M:%SZ' }}"
```

Run the workflow once manually to confirm documents appear in your index.

---

# Step 4 — Head to the Agents tab to access Agent Builder

Using the built-in chat, ask a query like: What's the latest news in AI?



# Step 5 Bonus — Create an Agent in Agent Builder

Navigate to:

Agents

Create a new agent with instructions like:

You are a research assistant that answers questions using recent indexed web results.

Prefer the newest documents.

If the user asks about trends, summarize themes across sources.

Attach tools:

- Elasticsearch search tool

Set your index:

ai-fresh-context

Now test:

What changed in AI agents this week?

---


# Suggested Project Ideas

## AI Release Tracker

Track announcements from:

- OpenAI
- Anthropic
- Meta
- Elastic
- Hugging Face

---

## Startup Intelligence Agent

Monitor:

latest AI startup funding announcements

---

## Research Paper Watcher

Track:

latest LLM research arxiv

---

## Policy Monitoring Agent

Track:

latest AI regulation announcements US EU

---

## Dev Tool Release Monitor

Track:

latest MCP tools OR agent SDK releases

---


# Judging Criteria

Projects will be evaluated based on:

- use of Tavily Search
- use of Elastic Workflows
- retrieval quality
- freshness strategy
- agent usefulness
- creativity

Bonus points for:

- scheduled refresh pipelines
- metadata enrichment
- filtering or reranking
- workflow-triggered agent actions
- Custom agents and tools within Agent Builder

---

# Architecture Reference

Baseline architecture:

Elastic Workflow (scheduled)
↓
Tavily Search API
↓
Transform results
↓
Elasticsearch index
↓
Agent Builder retrieval


# Helpful Links

Elastic Workflows docs  
https://www.elastic.co/docs/explore-analyze/workflows

Workflows Examples Library:
https://github.com/elastic/workflows/

Agent Builder docs  
https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder

Tavily API docs  
https://docs.tavily.com

---

# Goal of the Hacknight

Build agents that **stay current automatically**, not agents that rely on static knowledge snapshots.

If your agent’s context updates itself without manual re-indexing, you’re doing it right 

