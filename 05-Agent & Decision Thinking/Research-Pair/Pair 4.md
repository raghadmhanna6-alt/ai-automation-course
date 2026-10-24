# 🔍 AI Agent Observability & Tracing with LangSmith
> **Research Unit 5: Control Its Agency**  
> *Discover Tools. Explore Concepts. Build Better AI Agents.*

---

## 👥 Prepared By
- **Haneen Awad** - حنين عوض
- **Aseel Abu Ghosein** - أسيل أبو غصين

---

## 📌 Executive Summary
When building autonomous AI agents, receiving an unexpected response or experiencing a silent task failure (e.g., missing records in a database) presents a major challenge. Knowing *that* an error occurred is insufficient; developers must understand *why* and *where* it occurred. 

This research investigates the concepts of **Observability** and **Tracing** using **LangSmith**, demonstrating how tracing logs enable complete visibility into the AI Agent's decision-making process, prompt parameters, and tool execution cycles.

---

## 🎯 Research Questions & Core Concepts

### 1. What is Observability & Tracing?
* **Observability:** The capability to continuously monitor, analyze, and inspect the internal execution state and behavior of an AI Agent ("peering into the black box").
* **Tracing:** The step-by-step chronological record (Trace) capturing every event within an agent's run—including user inputs, LLM thought chains, token usage, latency, and individual tool calls.
* **The Key Distinction:** Knowing a result is wrong is simply observing the final output failure. Tracing allows us to pinpoint the root cause (e.g., malformed prompt, API timeout, or incorrect tool parameter mapping).

---

### 2. When Would You Use It?
In production-ready AI automation projects:
1. **Debugging Failed Workflows:** Identifying why an agent hallucinated, selected the wrong tool, or formatted output incorrectly.
2. **Performance & Cost Optimization:** Tracking token consumption, cost per call, and execution latency across different model providers.
3. **Quality Assurance & Evaluation:** Testing agents against benchmark datasets to evaluate response accuracy and reliability before deployment.

---

### 3. Case Study: Real Estate Lead Qualification Agent

#### 🏢 Problem Scenario
Our real estate qualification agent receives customer details via Telegram and processes them, but the lead fails to appear in the **`XYZ Real Estate Leads`** Google Sheet.

#### 🛠️ How Tracing Resolves the Issue
By opening the execution Trace in **LangSmith**, we can review:
1. **Extraction Step:** Did the LLM successfully extract all 9 lead attributes from the raw Telegram message?
2. **Tool Selection:** Did the agent decide to call the `Google Sheets Tool`?
3. **Payload Inspection:** Were the extracted attributes mapped correctly to the tool arguments?
4. **Execution Response:** Did Google Sheets return a `200 OK` or an authorization/schema error?

---

## 🛠️ Tool Spotlight: LangSmith Interface

LangSmith provides a unified dashboard to monitor agents:
* **Traces View:** Detailed breakdown of input/output chains for every request.
* **LLM Calls:** Exact system prompts, user messages, and completion text.
* **Tool Logs:** Arguments passed into external tools (e.g., Google Sheets, Web Search) and their returned payloads.

---

## 💡 Conclusion
Observability transforms AI Agent development from guesswork into an engineering discipline. Tools like **LangSmith** ensure our automation pipelines remain robust, transparent, and easy to maintain as they scale.