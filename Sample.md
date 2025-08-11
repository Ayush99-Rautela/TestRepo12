## **Updated Architecture – Google-style Context + Embeddings**

### **1. Three-Tier Memory System**

Your bot will have **short-term**, **session**, and **long-term** memory.

| Tier                 | Storage                          | Purpose                                                                    | Lifetime                |
| -------------------- | -------------------------------- | -------------------------------------------------------------------------- | ----------------------- |
| **Turn Memory**      | In-memory (Java object)          | Holds only the current request/response data                               | One request             |
| **Session Memory**   | Redis or fast in-memory DB table | Tracks ongoing conversation context for follow-ups                         | Idle timeout (5–10 min) |
| **Long-Term Memory** | PostgreSQL + pgvector            | Embeddings for all entities (tasks, collections, etc.) for semantic recall | Persistent              |

---

### **2. Flow for Handling a Prompt**

#### **Step 1 – Intent Classification**

* Use `openAIClient.classifyIntent()` → Decide: **task**, **collection**, **chat/help**
* If task/collection:

  * Call the respective microservice to get **entity data**
  * Extract key attributes (`title`, `dueDate`, `status`, etc.)
  * Generate an **embedding** for the entity summary
  * Store it in **Session Memory** and optionally in **Long-Term Memory**

#### **Step 2 – Follow-up Handling**

When a prompt arrives:

1. Generate **prompt embedding**
2. Check **Session Memory** for highest similarity to active entity embeddings

   * If **similarity ≥ threshold** (e.g., 0.85) → treat as follow-up
3. If no match:

   * Query **Long-Term Memory** for closest entity embedding
   * If match found → load entity into Session Memory
4. If no match in either:

   * Treat as new request (Step 1 again)

---

### **3. Answering the Prompt**

* **If attribute question** (due date, title, status, etc.) → Answer **directly from memory** (no LLM)
* **If reasoning question** (summarize, compare, filter) → Pass **only relevant context snippet** to LLM:

```plaintext
You are answering about the task:
Title: Buy groceries
Due Date: 2025-08-12
Status: Pending
Description: Buy vegetables and milk
```

* **If ambiguous match** (multiple entities with high similarity) → Ask disambiguation:

```
Did you mean:
1. Buy groceries (due today)
2. Finish report (due tomorrow)
```

---

### **4. Context Management Rules**

* **Auto-expire** session memory after inactivity (5–10 min)
* **Clear** session when:

  * User explicitly says "done", "exit", "stop"
  * A new unrelated topic is detected
* **Replace** session memory when a new entity is fetched
* **Confirm** with user if resuming old context from long-term memory:

```
I think you might be talking about "Buy groceries". Should I continue?
```

---

### **5. Embedding Index Structure**

PostgreSQL table for long-term memory:

```sql
CREATE TABLE entity_embeddings (
    id UUID PRIMARY KEY,
    entity_type TEXT,
    entity_id TEXT,
    summary TEXT,
    embedding vector(1536)
);
```

* **summary** → human-readable key info for LLM prompts
* **embedding** → used for cosine similarity searches

---

### **6. Updated Service Responsibilities**

* **HelperBotService** → Orchestrator:

  * Detects intent
  * Checks memory
  * Routes to Task/Collection handlers
  * Decides if LLM is needed
* **MemoryService** → Handles Redis & DB context storage/retrieval
* **EmbeddingService** → Handles embedding creation & vector DB queries
* **DisambiguationService** → Prompts user to pick from multiple possible matches

---

### **7. Example Conversation Flow**

**User:** “Show me today’s tasks”
✅ Fetch tasks → store in session memory with embeddings

**User:** “What’s the due date of the groceries one?”
✅ Prompt embedding matches “Buy groceries” in session memory → answer directly

**User:** “Mark it as complete”
✅ Same match → send API call to Task Service → clear session (since task state updated and flow complete)

**User (next day):** “When is that shopping task due?”
✅ No session match → Long-term memory embedding match → confirm:

```
I think you mean 'Buy groceries' (due Aug 12). Continue?
```

---

### **8. Advantages of This Design**

✅ **Token-efficient** — only send relevant snippets to LLM
✅ **Context-aware** — remembers across turns and even sessions
✅ **Smart disambiguation** — avoids wrong assumptions
✅ **Google-like** — exactly how real assistants handle memory & follow-ups
✅ **Scalable** — works for multiple microservices (tasks, collections, future entities)

---
