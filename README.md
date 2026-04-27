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

[https://cloud.elastic.co/serverless-registration](https://cloud.elastic.co/serverless-registration?utm_campaign=hack-night-signups&utm_source=onsite&utm_medium=sf)


## Tavily

Create a key here:

https://app.tavily.com

Store it securely.


---

# Step 2 — Create an Elastic Workflow

Your workflow should:

1. run on a schedule/trigger
2. check if your index exists/create one if not
3. call Tavily Search API
4. normalize results
5. index documents into Elasticsearch

Example workflow:

```yaml
name: ai-trends-workflow
enabled: true
description: Fetch fresh AI news for agent retrieval

inputs:
  - name: index_name
    type: string
    description: "Target Elasticsearch index name"
    default: "ai-fresh-context"

triggers:
  - type: scheduled
    with:
      every: 12h

steps:
  # Step 1: Check if index exists
  - name: get_index
    type: elasticsearch.indices.exists
    with:
      index: "{{ inputs.index_name }}"

  # Step 2: Create index only if missing
  - name: check_if_index_missing
    type: if
    condition: 'steps.get_index.output : false'
    steps:
      - name: create_index
        type: elasticsearch.indices.create
        with:
          index: "{{ inputs.index_name }}"
          mappings:
            properties:
              title:
                type: text
              url:
                type: keyword
              domain:
                type: keyword
              content:
                type: semantic_text
              snippet:
                type: text
              source:
                type: keyword
              topic:
                type: keyword
              retrieved_at:
                type: date

  # Step 3: Call Tavily Search API
  - name: tavily_search
    type: http
    with:
      url: https://api.tavily.com/search
      method: POST
      headers:
        Authorization: "Bearer tavily-api-key"
        Content-Type: application/json
      body: |
        {
          "query": "latest AI agent frameworks announcements",
          "search_depth": "advanced",
          "max_results": 5,
          "include_raw_content": true
        }

  # Step 4: Index results
  - name: index_articles
    type: foreach
    foreach: "{{ steps.tavily_search.output.data.results }}"
    steps:
      - name: upsert_doc
        type: elasticsearch.update
        with:
          index: "{{ inputs.index_name }}"
          id: "{{ foreach.item.url }}"
          doc_as_upsert: true
          doc:
            title: "{{ foreach.item.title }}"
            url: "{{ foreach.item.url }}"
            domain: "{{ foreach.item.url | replace: 'https://', '' | replace: 'http://', '' | split: '/' | first }}"
            snippet: "{{ foreach.item.content }}"
            content: "{{ foreach.item.raw_content | default: foreach.item.content | truncate: 12000 }}"
            source: "tavily_search"
            topic: "ai_agents"
            retrieved_at: "{{ 'now' | date: '%Y-%m-%dT%H:%M:%SZ' }}"
```

Run the workflow once manually to confirm documents appear in your index.

---

# Step 3 — Head to the Agents tab to access Agent Builder

Using the built-in chat, ask a query like: What's the latest news in AI?



# Bonus — Create an Agent in Agent Builder

Navigate to:

Agents

Create a new agent with instructions like:

You are a research assistant that answers questions using recent indexed web results.

Prefer the newest documents.

If the user asks about trends, summarize themes across sources.

Attach tools:

- platform.core.search
- any custom tools

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

