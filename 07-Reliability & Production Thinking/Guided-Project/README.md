# 🧠 RAG with n8n
## Build a Knowledge Pipeline & RAG AI Agent

> **From Documents → Embeddings → Vector Database → Retrieval → AI Agent → Grounded Answer**

---

# 1. 🎯 Learning Goal

في هذه التجربة سنعيد بناء **RAG (Retrieval-Augmented Generation)** باستخدام **n8n + Supabase Vector Store**.

تعلمنا RAG سابقًا باستخدام **Dify**.

في Dify كانت الكثير من التفاصيل مخفية خلف واجهة جاهزة.

أما هنا، فسنبني الـ RAG بأنفسنا حتى نفهم:

- كيف تدخل الـ Documents إلى النظام.
- كيف يتم استخراج النص.
- لماذا نحتاج إلى Chunking.
- ما هي Embeddings.
- كيف يتم تخزينها في Vector Database.
- كيف يبحث النظام عن المعلومات المناسبة.
- كيف يستخدم الـ AI Agent الـ Vector Store كـ Tool.
- الفرق بين Knowledge وMemory.
- الفرق بين Retrieval وGeneration.
- كيف نبني RAG system يمكن دمجه لاحقًا داخل حلول AI Automation حقيقية.

---

# 2. 🧠 قبل أن نبدأ: ما هو RAG؟

## RAG = Retrieval-Augmented Generation

الفكرة ببساطة:

```text
Retrieve relevant knowledge
            ↓
Augment the context
            ↓
Generate the answer
```

أي:

> بدل أن نعتمد فقط على المعرفة الموجودة داخل الـ LLM، نجعله يبحث أولًا في Knowledge Base خارجية، ثم يستخدم المعلومات المسترجعة لصياغة الإجابة.

---

# 3. 🏗️ RAG Architecture

سنقسم النظام إلى جزأين:

```text
                    RAG SYSTEM
                        │
          ┌─────────────┴─────────────┐
          │                           │
   1. KNOWLEDGE PIPELINE        2. RAG AI AGENT
      "Prepare Knowledge"          "Use Knowledge"
          │                           │
       PDF File                   User Question
          ↓                           ↓
    Extract Text                 AI Agent
          ↓                           ↓
      Chunking                 Vector Tool
          ↓                           ↓
     Embeddings               Query Embedding
          ↓                           ↓
   Supabase Vector DB ←──── Similarity Search
                                      ↓
                               Relevant Chunks
                                      ↓
                                     LLM
                                      ↓
                                    Answer
```

## مهم

المسار الأول ليس Agent بالمعنى الدقيق.

هو:

> **Knowledge Ingestion Pipeline**

أما المسار الثاني فهو:

> **RAG AI Agent**

---

# 4. 🧩 الصورة الكاملة للـ Data Flow

## أولًا: تجهيز المعرفة

```text
Document
   ↓
Extract Text
   ↓
Document
   ↓
Chunking
   ↓
Embeddings
   ↓
Supabase Vector Store
```

## ثانيًا: استخدام المعرفة

```text
User Question
      ↓
AI Agent
      ↓
Vector Store Tool
      ↓
Question Embedding
      ↓
Similarity Search
      ↓
Relevant Chunks
      ↓
AI Agent / LLM
      ↓
Answer
```

---

# PART 1
# 🟢 Knowledge Ingestion Pipeline

## الهدف

سنقوم ببناء Workflow يأخذ ملف PDF من Google Drive ويحوّله إلى Knowledge قابلة للبحث.

```text
Google Drive
     ↓
Download
     ↓
Extract Text
     ↓
Load Document
     ↓
Split into Chunks
     ↓
Create Embeddings
     ↓
Store in Supabase
```

---

# 5. 🗄️ Step 0 — Prepare Supabase Vector Database

قبل بناء Workflow في n8n، نحتاج إلى قاعدة بيانات لتخزين الـ embeddings.

سنستخدم:

> **Supabase + PostgreSQL + pgvector**

`pgvector` هو امتداد PostgreSQL الذي يسمح بتخزين الـ vectors وتنفيذ عمليات البحث بالتشابه بينها.

---

## 5.1 Create a Supabase Project

افتح:

**Supabase**

ثم:

```text
New Project
```

أنشئ مشروعًا جديدًا، مثل:

```text
AI Automation RAG
```

---

# 5.2 Enable Vector Support

من Supabase يمكن تفعيل:

```text
Database
   ↓
Extensions
   ↓
vector
```

ثم:

```text
Enable
```

امتداد `vector` هو امتداد `pgvector` المستخدم لتخزين والبحث في الـ embeddings.

---

# 5.3 Use Supabase Quick Start

من:

```text
SQL Editor
```

ابحث عن:

```text
LangChain
```

ضمن **Quick Start**.

ثم:

```text
Run
```

Supabase توفر Quick Start لإنشاء البنية اللازمة لاستخدام Supabase كـ Vector Store، ويتضمن ذلك جدول `documents` ودالة `match_documents`.

### لماذا نستخدم Quick Start؟

لأن n8n يحتاج إلى بنية قاعدة بيانات يستطيع من خلالها:

1. تخزين الـ chunks.
2. تخزين الـ embeddings.
3. تنفيذ similarity search.
4. استرجاع الـ chunks المناسبة.

---

# 5.4 ماذا أنشأنا؟

في النهاية نحتاج بنية مشابهة لـ:

```text
documents
│
├── id
├── content
├── metadata
└── embedding
```

### `content`

النص الخاص بالـ chunk.

### `metadata`

معلومات إضافية عن الـ chunk.

مثل:

```json
{
  "file_id": "123",
  "file_name": "RAG.pdf"
}
```

### `embedding`

التمثيل الرقمي للنص.

---

# 5.5 لماذا نحتاج `match_documents`؟

لأن الـ Agent سيحتاج لاحقًا أن يسأل قاعدة البيانات:

> "أعطني أكثر الـ chunks تشابهًا مع هذا السؤال."

دالة:

```text
match_documents
```

تقوم بهذا الدور.

n8n يستخدم `match_documents` كـ Query Name افتراضي عند استخدام Supabase Quick Start.

---

# ⚠️ Important: Embedding Dimensions

إذا استخدمنا:

```text
text-embedding-3-small
```

يجب أن يكون تعريف عمود الـ vector متوافقًا مع عدد الأبعاد الناتجة من إعداد الـ embedding المستخدم.

القاعدة:

```text
Embedding Model
       ↓
Vector Dimensions
       ↓
Supabase vector column
```

يجب أن تكون متوافقة.

لا نفترض أن كل نموذج Embeddings له نفس عدد الأبعاد. Supabase توضح أن حجم `vector(...)` يجب أن يتوافق مع أبعاد النموذج المستخدم.

---

# 🔐 Security Note

سنستخدم Credential داخل n8n.

لا نضع:

- API Keys
- Supabase Secret Keys
- Service Role Keys

داخل GitHub أو داخل النصوص التي نشاركها.

GitHub يحتوي على:

> Workflow structure and configuration instructions

وليس:

> Secret credentials.

---

# 6. 🔵 Step 1 — Google Drive Trigger

أضف Node:

```text
Google Drive Trigger
```

### الهدف

تشغيل الـ Workflow عندما يحدث تغيير متعلق بالملفات داخل الـ folder الذي اخترناه.

مثال:

```text
Google Drive
└── AI Automation Knowledge
      ├── RAG.pdf
      ├── AI Agents.pdf
      └── Automation.pdf
```

عند إضافة Document جديد:

```text
New File
   ↓
Trigger
   ↓
Workflow starts
```

---

## لماذا هذه الـ Node مهمة؟

لأننا نتعلم:

# Event-Driven Automation

بدل أن يقوم المستخدم بتشغيل الـ Workflow يدويًا:

```text
Human
 ↓
Run Workflow
```

نصبح:

```text
Event
 ↓
Automation
```

وهذا هو أساس الكثير من أنظمة AI Automation الحقيقية.

---

## إعدادها

اختاري:

```text
Google Drive Trigger
```

ثم الحدث الخاص بالتغييرات في Folder محدد.

اختاري الـ Folder:

```text
AI Automation Knowledge
```

ثم حددي Schedule مناسبًا، مثل:

```text
Every Minute
```

> ملاحظة: خيارات الـ Trigger قد تختلف قليلًا حسب إصدار n8n، لذلك ركزي على المفهوم: **راقب Folder محدد وابدأ Workflow عند التغيير.**

---

# 7. 🟣 Step 2 — Edit Fields: File ID

بعد Trigger نحتاج إلى تحديد:

```text
file_id
```

أضيفي:

```text
Edit Fields
```

أو Node الخاصة بإعادة تسمية/تنظيم البيانات في إصدار n8n لديك.

---

## ماذا نفعل؟

نستخرج:

```text
file_id
```

من بيانات Google Drive.

مثال:

```json
{
  "file_id": "1ABCxyz..."
}
```

---

# لماذا نحتاج File ID؟

لأننا نريد معرفة مصدر الـ knowledge.

عندما نخزن:

```text
Chunk
```

نريد أن نستطيع معرفة:

```text
Which file did this chunk come from?
```

لذلك نستخدم:

```text
Metadata
```

مثال:

```json
{
  "file_id": "1ABCxyz",
  "file_name": "AI Agents.pdf"
}
```

---

# 🧠 Concept

## Content vs Metadata

### Content

المعلومة نفسها:

```text
AI Agents can use tools...
```

### Metadata

معلومات عن هذه المعلومة:

```text
file_id
file_name
source
page
category
```

---

# 8. 🔵 Step 3 — Google Drive: Download File

أضف:

```text
Google Drive
```

واختر عملية:

```text
Download File
```

---

## Input

نستخدم:

```text
file_id
```

القادم من الخطوة السابقة.

```text
Edit Fields
     ↓
file_id
     ↓
Google Drive Download
```

---

## Output

سيتم تحميل الملف كـ:

```text
Binary Data
```

مثلاً:

```text
PDF
 ↓
Binary
```

---

# لماذا نحتاج Download؟

الـ Trigger أعطانا معلومات عن الملف.

لكنه لا يعني بالضرورة أن لدينا محتوى الملف نفسه في الشكل الذي تحتاجه عملية استخراج النص.

لذلك:

```text
File ID
   ↓
Download
   ↓
Actual File
```

---

# 9. 🟡 Step 4 — Extract From File

الآن لدينا:

```text
PDF Binary
```

لكننا نحتاج:

```text
Text
```

أضف:

```text
Extract From File
```

واستخدم العملية المناسبة لاستخراج النص من PDF.

---

## ماذا يحدث؟

```text
PDF
 ↓
Extract
 ↓
Raw Text
```

مثال:

```text
AI Automation is the process of designing
systems that combine AI models with workflows...
```

---

# لماذا هذه الخطوة مهمة؟

لأن الـ Embedding Model يعمل على النص.

لا نريد:

```text
PDF Binary
 ↓
Embedding
```

بل:

```text
PDF
 ↓
Text
 ↓
Chunk
 ↓
Embedding
```

---

# 10. 🧩 Step 5 — Supabase Vector Store

الآن وصلنا إلى أهم Node في Ingestion Pipeline.

أضف:

```text
Supabase Vector Store
```

اختر:

```text
Insert Documents
```

أو اسم العملية المقابل في إصدار n8n لديك لإدخال Documents.

---

# 10.1 Connect Supabase Credential

أنشئ Credential لـ Supabase.

تحتاج المعلومات التي تطلبها Node، والتي تشمل اتصال Supabase والـ service-role credential المناسب لهذه العملية.

n8n يستخدم بيانات Supabase credential لإنشاء الاتصال بالـ vector store.

---

## ⚠️ لا تضع الـ Secret في Workflow Documentation

استخدم:

```text
Credential
```

وليس:

```text
Hard-coded Secret
```

---

# 10.2 Table Name

استخدم:

```text
documents
```

إذا كنتِ تستخدمين Supabase Quick Start الافتراضي.

---

# 10.3 Query Name

استخدم:

```text
match_documents
```

إذا كان هذا هو الاسم الذي أنشأه Quick Start.

n8n يوثق أن `match_documents` هو الـ default Query Name عند اتباع Supabase Quick Start.

---

# 11. 🟠 Supabase Vector Store Sub-Nodes

عندما نضيف Supabase Vector Store ستظهر لنا Connections إضافية.

من أهمها:

```text
Document
Embeddings
```

وسنستخدم:

```text
Default Data Loader
Recursive Character Text Splitter
OpenAI Embeddings
```

الصورة:

```text
                 Supabase Vector Store
                         │
          ┌──────────────┴──────────────┐
          │                             │
      Document                       Embeddings
          │                             │
 Default Data Loader          OpenAI Embeddings
          │                             │
 Recursive Character          text-embedding-3-small
 Text Splitter
```

---

# 12. 📄 Step 6 — Default Data Loader

أضف:

```text
Default Data Loader
```

على مدخل:

```text
Document
```

---

## ما وظيفته؟

يقوم بتحويل البيانات إلى Document objects التي يمكن لبقية الـ LangChain/n8n AI components التعامل معها.

بشكل مبسط:

```text
Extracted Text
      ↓
Document
      ↓
Chunking
      ↓
Embedding
      ↓
Vector Store
```

---

# لماذا لا نرسل النص مباشرة؟

لأن الـ Vector Store يعمل مع:

```text
Documents
```

والـ Document يحتوي على:

```text
pageContent
+
metadata
```

وهذا يسمح لنا بحفظ:

```text
Content
+
Source Information
```

معًا.

---

# 13. ✂️ Step 7 — Recursive Character Text Splitter

الآن داخل Data Loader نضيف:

```text
Recursive Character Text Splitter
```

---

# لماذا نحتاج Chunking؟

تخيل أن لدينا:

```text
50-page PDF
```

لا نريد تحويله إلى:

```text
One huge vector
```

بل:

```text
PDF
│
├── Chunk 1
├── Chunk 2
├── Chunk 3
├── Chunk 4
└── ...
```

---

# لماذا؟

لأن المستخدم قد يسأل:

> What is an AI Agent?

ولا نريد إرجاع الـ PDF كاملًا.

نريد:

```text
The relevant chunk
```

فقط.

---

# 13.1 Chunk Size

ابدئي بقيمة عملية صغيرة ومناسبة للتجربة، ثم اختبري النتائج.

مثلاً:

```text
Chunk Size:
500
```

يمكن لاحقًا تجربة:

```text
500
800
1000
```

بحسب طبيعة المحتوى.

---

# 13.2 Chunk Overlap

مثلاً:

```text
Chunk Overlap:
50
```

أي أن جزءًا من نهاية chunk سيظهر أيضًا في بداية chunk التالي.

مثال:

```text
Chunk 1:
A B C D E F

Chunk 2:
        E F G H I J
```

---

# لماذا Overlap؟

لأننا لا نريد أن ينقطع المعنى عند الحدود بين الـ chunks.

---

# 🧠 Important

لا يوجد:

> Perfect Chunk Size

لكل الحالات.

اختيار Chunk Size يعتمد على:

- طبيعة المحتوى.
- طول الفقرات.
- نوع الأسئلة.
- حجم السياق.
- جودة Retrieval.

لذلك:

> **Chunking is an engineering decision, not a magic number.**

---

# 14. 🔢 Step 8 — OpenAI Embeddings

الآن نصل إلى:

```text
Embeddings
```

أضف:

```text
OpenAI Embeddings
```

واختر:

```text
text-embedding-3-small
```

---

# ما هي Embedding؟

هي طريقة لتمثيل النص كـ Vector رقمي.

مثلاً:

```text
"AI Agent uses tools"
```

يصبح بشكل مبسط:

```text
[0.12, -0.44, 0.81, ...]
```

---

# لماذا نفعل ذلك؟

لأننا نريد أن نبحث عن:

> **Semantic Similarity**

وليس فقط تطابق الكلمات.

Supabase تستخدم pgvector لتخزين الـ embeddings وتنفيذ عمليات vector similarity search.

---

# 15. 🔗 Step 9 — Connect Embeddings to Supabase

الآن:

```text
OpenAI Embeddings
        ↓
Supabase Vector Store
```

وعملية الإدخال تصبح:

```text
Text
 ↓
Chunks
 ↓
Embedding
 ↓
Supabase
```

كل chunk سيصبح له:

```text
Content
+
Embedding
+
Metadata
```

---

# 16. 🧪 Step 10 — Run the Ingestion Workflow

الآن:

1. احفظ Workflow.
2. شغله في Test/Manual mode حسب إعداد الـ Trigger.
3. ارفع PDF تجريبي إلى Google Drive Folder.
4. انتظر تشغيل الـ Trigger.
5. راقب كل Node.

يجب أن نرى:

```text
Google Drive Trigger
        ✓
Edit Fields
        ✓
Download
        ✓
Extract
        ✓
Data Loader
        ✓
Text Splitter
        ✓
Embeddings
        ✓
Supabase
        ✓
```

---

# 17. 🔍 Verify Supabase

افتح:

```text
Supabase
   ↓
Table Editor
   ↓
documents
```

يجب أن تبدأ برؤية records.

مثلاً:

```text
id
content
metadata
embedding
```

إذا وجدنا records، فهذا يعني أن:

> **Knowledge Ingestion Pipeline works.**

---

# 🧠 توقف هنا

قبل بناء Agent الثاني، يجب أن تفهم المتدربة:

```text
We are NOT answering questions yet.

We are preparing knowledge
for future retrieval.
```

---

# PART 2
# 🤖 RAG AI Agent

الآن سنبني الجزء الذي يتفاعل مع المستخدم.

---

# 18. 🎯 الهدف

المستخدم يسأل:

```text
What is RAG?
```

والـ Agent يجب أن:

```text
Understand Question
       ↓
Use Knowledge Tool
       ↓
Search Supabase
       ↓
Retrieve Relevant Chunks
       ↓
Use LLM
       ↓
Generate Answer
```

---

# 19. 🟢 Step 1 — When Chat Message Received

أضف:

```text
When Chat Message Received
```

أو:

```text
Chat Trigger
```

بحسب تسمية إصدار n8n لديك.

---

# وظيفته

يستقبل:

```text
User Message
```

مثلاً:

```text
What is RAG?
```

ثم يمرر السؤال إلى:

```text
AI Agent
```

---

# لماذا هذه Node؟

لأننا نحتاج:

> Entry Point

للمستخدم.

بدل:

```text
PDF → Workflow
```

أصبح لدينا:

```text
User → Workflow
```

---

# 20. 🧠 Step 2 — AI Agent

أضف:

```text
AI Agent
```

هذه هي الـ Core Node في هذا الجزء.

---

# ماذا يفعل AI Agent؟

الـ Agent لديه:

```text
Model
Memory
Tools
Instructions
```

وفي مشروعنا:

```text
AI Agent
│
├── Gemini
├── Simple Memory
└── Supabase Vector Store
```

---

# 21. 🟣 Step 3 — Connect Gemini Model

أضف:

```text
Google Gemini Chat Model
```

واربطه مع:

```text
AI Agent
```

---

# ما وظيفة الـ Model؟

الـ LLM مسؤول عن:

- فهم سؤال المستخدم.
- تحديد ما إذا كان يحتاج Tool.
- قراءة المعلومات المسترجعة.
- صياغة الإجابة.

---

# مهم جدًا

Gemini ليس هو Knowledge Base.

أي:

```text
Gemini
≠
Your Documents
```

الـ Knowledge موجودة في:

```text
Supabase Vector Store
```

---

# 22. 🧠 Step 4 — Simple Memory

أضف:

```text
Simple Memory
```

واربطها بـ:

```text
AI Agent
```

---

# لماذا؟

حتى يستطيع الـ Agent الحفاظ على سياق المحادثة.

مثلاً:

```text
User:
What is RAG?

Agent:
RAG is...

User:
Why is it useful?
```

الـ Agent يحتاج معرفة أن:

```text
"It"
```

يشير إلى:

```text
RAG
```

---

# ⚠️ Memory ≠ Knowledge Base

هذه من أهم الأفكار في الدرس.

## Memory

تتذكر:

> ماذا حدث في المحادثة؟

## Vector Store

يحتوي:

> ماذا نعرف عن المجال؟

مثال:

```text
Memory
↓
Conversation Context
```

بينما:

```text
Vector Store
↓
External Knowledge
```

---

# 23. 🔵 Step 5 — Give the Agent a Vector Store Tool

الآن نضيف:

```text
Supabase Vector Store
```

لكن هذه المرة لا نستخدمه لإدخال Documents.

نستخدمه كـ:

```text
Tool
```

n8n يدعم توصيل Supabase Vector Store مباشرة بالـ AI Agent ليستخدمه كـ Tool.

---

# 24. 🔧 Configure Vector Store as Tool

اختاري العملية:

```text
Retrieve Documents (As Tool for AI Agent)
```

أو الاسم المقابل في إصدار n8n لديك.

---

# Tool Name

استخدمي اسمًا واضحًا:

```text
search_course_knowledge_base
```

---

# Tool Description

اكتبي:

```text
Search the course knowledge base to retrieve relevant information for answering questions about the course.
```

---

# لماذا Name وDescription مهمان؟

لأن الـ Agent يحتاج أن يعرف:

```text
What can this tool do?
```

وليس فقط:

```text
What is this node?
```

الـ Description يساعد الـ LLM على اختيار الأداة المناسبة. n8n توصي بوصف محدد للـ vector-store tool حتى يتمكن الـ LLM من استخدامه بالشكل المتوقع.

---

# 25. 🔢 Limit / Top K

اضبطي عدد النتائج، مثلاً:

```text
Limit = 4
```

أي:

> حاول إرجاع أفضل 4 chunks ذات صلة.

مثال:

```text
Question
   ↓
Vector Search
   ↓
Chunk 7
Chunk 19
Chunk 3
Chunk 21
```

---

# هل 4 رقم سحري؟

لا.

هو:

> **Starting point**

يمكن لاحقًا اختبار:

```text
Top K = 3
Top K = 4
Top K = 5
```

ثم نرى أي إعداد يعطي أفضل Retrieval.

---

# 26. 🔢 Step 6 — Connect OpenAI Embeddings to the Vector Tool

الآن نحتاج:

```text
OpenAI Embeddings
```

بنفس النموذج الذي استخدمناه أثناء Ingestion:

```text
text-embedding-3-small
```

---

# لماذا نستخدم Embeddings مرة ثانية؟

هذه المرة لا نعمل Embedding للـ Documents.

نحن نعمل Embedding للسؤال.

مثلاً:

```text
User Question:

What is RAG?
```

يصبح:

```text
Question
   ↓
Embedding
   ↓
Query Vector
```

ثم:

```text
Query Vector
      ↓
Supabase
      ↓
Similarity Search
```

---

# 🧠 قاعدة مهمة

لدينا Embeddings في مكانين:

## أثناء Ingestion

```text
Document Chunk
     ↓
Embedding
     ↓
Supabase
```

## أثناء Retrieval

```text
User Question
     ↓
Embedding
     ↓
Supabase Search
```

ثم تتم مقارنة:

```text
Question Vector
        ↕
Document Vectors
```

---

# 27. 🔎 ماذا يحدث عند السؤال؟

الآن أصبح لدينا النظام كاملًا.

المستخدم يسأل:

```text
What is RAG?
```

---

## Step 1

Chat Trigger يستقبل السؤال.

```text
User
 ↓
Chat Trigger
```

---

## Step 2

السؤال يدخل إلى AI Agent.

```text
Chat Trigger
 ↓
AI Agent
```

---

## Step 3

الـ Agent يقرر أن السؤال يحتاج إلى Knowledge Base.

```text
AI Agent
 ↓
search_course_knowledge_base
```

---

## Step 4

السؤال يتحول إلى Embedding.

```text
Question
 ↓
OpenAI Embeddings
 ↓
Query Vector
```

---

## Step 5

Supabase يبحث عن أقرب vectors.

```text
Query Vector
      ↓
Supabase Vector Store
      ↓
Similarity Search
      ↓
Top 4 Chunks
```

---

## Step 6

الـ chunks ترجع إلى الـ Agent.

```text
Relevant Chunks
       ↓
AI Agent
```

---

## Step 7

Gemini يستخدم المعلومات المسترجعة لصياغة الإجابة.

```text
Retrieved Context
       +
User Question
       ↓
Gemini
       ↓
Answer
```

---

# 28. 🧩 الصورة الكاملة للإيجنت الثاني

```text
              USER
                │
                ↓
        Chat Message Trigger
                │
                ↓
            AI AGENT
          /     |      \
         /      |       \
     Gemini   Memory    Tool
                         │
                         ↓
                Supabase Vector Store
                         │
                         ↓
                 OpenAI Embeddings
                         │
                         ↓
                  Query Vector
                         │
                         ↓
                 Similarity Search
                         │
                         ↓
                  Relevant Chunks
                         │
                         ↓
                     AI Agent
                         │
                         ↓
                      Gemini
                         │
                         ↓
                      Answer
```

---

# 29. 🧠 Step 7 — System Instructions

داخل AI Agent نضع Instructions واضحة.

استخدم:

```text
You are a helpful course assistant.

Use the search_course_knowledge_base tool whenever the user's question requires information from the course knowledge base.

Answer using the retrieved knowledge.

Do not invent information that is not supported by the retrieved content.

If the required information cannot be found in the knowledge base, clearly say that you do not have enough information to answer.
```

---

# لماذا نحتاج System Instructions؟

لأن الـ Agent يحتاج إلى معرفة:

```text
Role
+
Behavior
+
Knowledge Policy
+
Tool Usage
```

---

# ⚠️ لماذا لا نقول دائمًا "Always call the tool"?

ليس كل رسالة تحتاج Knowledge Retrieval.

مثلاً:

```text
Hello
```

لا تحتاج بالضرورة إلى البحث في Vector Database.

الأفضل:

> Use the knowledge-base tool whenever the request requires information from the knowledge base.

وهذا يعلمنا مفهوم:

# Tool Selection

وليس:

# Blind Tool Calling

---

# 30. 🧪 Step 8 — Test the RAG Agent

لا نقول:

> "اشتغل."

نختبره بطريقة منهجية.

---

# Test 1 — Known Information

اسأل:

```text
What is RAG?
```

المعلومة موجودة في الـ PDF.

يجب أن يحدث:

```text
Question
 ↓
Tool
 ↓
Relevant Chunks
 ↓
Answer
```

---

# Test 2 — Semantic Search

إذا كانت الوثيقة تقول:

```text
AI Agents can use external tools.
```

اسأل:

```text
Can an AI Agent interact with external systems?
```

إذا وجد الـ Agent المعلومة رغم اختلاف الكلمات:

> Semantic Retrieval يعمل.

---

# Test 3 — Unknown Information

اسأل عن معلومة غير موجودة في الـ Knowledge Base.

مثلاً:

```text
What is the admission policy of Harvard University?
```

إذا لم تكن موجودة في الوثائق، يجب ألا يخترع الـ Agent إجابة.

يجب أن يقول بوضوح:

```text
I don't have enough information in the knowledge base to answer this.
```

---

# Test 4 — Conversation Memory

ابدأ:

```text
User:
What is RAG?
```

ثم:

```text
User:
Why is it useful?
```

نختبر هنا:

```text
Memory
+
RAG
```

---

# 31. 🔍 Debugging — كيف نعرف أين المشكلة؟

إذا لم يعمل النظام، لا نقول:

> "n8n لا يعمل."

نحدد Layer المشكلة.

---

## Problem 1 — لا يوجد Record في Supabase

راجع:

```text
Google Drive Trigger
↓
Download
↓
Extract
↓
Data Loader
↓
Splitter
↓
Embeddings
↓
Supabase
```

---

## Problem 2 — Supabase يعطي Error في Search

راجع:

```text
Table Name
```

يجب أن يتطابق.

مثلاً:

```text
documents
```

ثم:

```text
Query Name
```

يجب أن يتطابق مع:

```text
match_documents
```

n8n يستخدم هذا الاسم افتراضيًا عند اتباع Supabase Quick Start.

---

## Problem 3 — Dimension Error

راجع:

```text
Embedding Model
        ↓
Vector Dimension
```

يجب أن يكونا متوافقين.

---

## Problem 4 — Agent يجيب معلومات غير موجودة

راجع:

```text
System Instructions
+
Tool Description
+
Retrieved Chunks
```

---

## Problem 5 — Retrieval سيئ

جرّب:

```text
Chunk Size
Chunk Overlap
Top K
```

ولا تغيّر كل شيء في الوقت نفسه.

اختبر متغيرًا واحدًا في كل مرة.

---

# 32. 🧠 RAG Mental Model

في نهاية التطبيق يجب أن تستطيع المتدربة رسم:

```text
              KNOWLEDGE
                  │
              Document
                  ↓
             Extract Text
                  ↓
               Chunk
                  ↓
              Embedding
                  ↓
           Vector Database
                  │
                  │
                  │
USER ───────→ Question
                  ↓
              Embedding
                  ↓
          Similarity Search
                  ↓
          Relevant Chunks
                  ↓
                 LLM
                  ↓
               Answer
```

---

# 33. 🔄 RAG vs Memory

| Memory | RAG |
|---|---|
| Conversation context | External knowledge |
| Remembers previous messages | Retrieves documents |
| Short-term conversational context | Knowledge retrieval |
| Helps understand follow-up questions | Helps answer domain questions |

### مثال

```text
Memory:
"What were we discussing?"

RAG:
"What does our knowledge base say?"
```

---

# 34. 🔄 Retrieval vs Generation

## Retrieval

وظيفته:

> Find relevant information.

```text
Question
 ↓
Embedding
 ↓
Vector Search
 ↓
Relevant Chunks
```

## Generation

وظيفته:

> Create the answer.

```text
Question
+
Retrieved Context
 ↓
LLM
 ↓
Answer
```

---

# 35. 🔄 Dify RAG vs n8n RAG

لقد تعلمنا RAG سابقًا في Dify.

الآن نفكك نفس الفكرة في n8n.

| Concept | Dify | n8n |
|---|---|---|
| Knowledge Base | Dify Knowledge | Supabase Vector Store |
| Document Processing | Built-in | Workflow Nodes |
| Chunking | Built-in | Recursive Character Text Splitter |
| Embeddings | Configured in Dify | OpenAI Embeddings |
| Vector Storage | Managed by Dify | Supabase |
| Retrieval | Dify Retrieval | Vector Store |
| LLM | Model Provider | Gemini |
| Memory | Conversation Memory | Simple Memory |
| Agent | Dify App/Agent | AI Agent |
| Tool | Built-in capability | Vector Store Tool |

---

# 36. 🧠 لماذا نعيد بناء RAG على n8n؟

ليس لأن n8n أفضل من Dify في كل شيء.

بل لأن الهدف التعليمي مختلف.

في Dify:

> نستخدم RAG.

في n8n:

> **نفهم كيف يعمل RAG.**

وهذا يجعلنا قادرين لاحقًا على اختيار الحل المناسب:

```text
Dify
n8n
Supabase
Other Vector DB
Other Embedding Model
Other LLM
```

---

# 37. 🏗️ Final Architecture

## Knowledge Pipeline

```text
Google Drive Trigger
        ↓
Edit Fields
        ↓
Google Drive Download
        ↓
Extract From File
        ↓
Default Data Loader
        ↓
Recursive Character Text Splitter
        ↓
OpenAI Embeddings
        ↓
Supabase Vector Store
```

---

## RAG Agent

```text
Chat Trigger
      ↓
   AI Agent
   /   |   \
Gemini Memory Vector Store Tool
              ↓
       OpenAI Embeddings
              ↓
        Supabase Vector Store
              ↓
       Relevant Documents
              ↓
           AI Agent
              ↓
            Gemini
              ↓
            Answer
```

---

# 38. 🎯 What You Should Understand After This Lab

بعد الانتهاء من التطبيق، يجب أن تكون قادرًا على شرح:

### 1. لماذا نحتاج Vector Database؟

لتخزين الـ embeddings وإجراء semantic similarity search.

### 2. لماذا نحتاج Chunking؟

حتى نبحث عن أجزاء صغيرة ومرتبطة من الوثيقة بدل استرجاع الوثيقة كاملة.

### 3. لماذا نحتاج Embeddings؟

لتحويل النص إلى representation رقمي يمكن استخدامه للمقارنة الدلالية.

### 4. لماذا نستخدم Embeddings مرتين؟

مرة للـ Documents أثناء ingestion، ومرة للسؤال أثناء retrieval.

### 5. ما دور Supabase؟

تخزين والبحث في الـ vectors والـ documents.

### 6. ما دور AI Agent؟

فهم السؤال واختيار الـ Tool واستخدام النتائج لصياغة الإجابة.

### 7. ما دور Memory؟

الحفاظ على سياق المحادثة.

### 8. ما دور Vector Store Tool؟

إعطاء الـ Agent قدرة على البحث في Knowledge Base.

### 9. ما الفرق بين Retrieval وGeneration؟

Retrieval يجد المعلومات.

Generation يصيغ الإجابة.

### 10. ما هو RAG؟

```text
Retrieve
   +
Augment
   +
Generate
```

---

# 39. 🚀 The Big Picture

ما بنيناه اليوم يمكن تلخيصه في:

```text
              RAG
               │
       ┌───────┴───────┐
       │               │
   INGESTION       RETRIEVAL
       │               │
   Documents       Questions
       ↓               ↓
   Extraction       Embedding
       ↓               ↓
   Chunking       Vector Search
       ↓               ↓
   Embedding      Relevant Chunks
       ↓               ↓
   Vector DB ───────→ Context
                         ↓
                        LLM
                         ↓
                      Answer
```

---

# 💡 Final Engineering Principle

> **RAG is not a node.**
>
> **RAG is a system architecture.**

و:

> **Vector Database does not generate answers.**
>
> **The LLM does not automatically know your private documents.**
>
> **The Agent does not replace the Knowledge Base.**
>
> **Memory does not replace RAG.**

كل Layer لها وظيفة محددة:

```text
Trigger
→ Starts the process

Loader
→ Loads the content

Splitter
→ Breaks content into useful chunks

Embeddings
→ Represents content numerically

Vector Store
→ Stores and retrieves knowledge

Memory
→ Maintains conversation context

AI Agent
→ Decides what to do

LLM
→ Understands and generates the final response
```

---

# 🧠 Final Question

قبل أن ننتقل إلى بناء حلول أكبر، اسأل نفسك:

> **If I need to build an AI solution for a real business, do I actually need RAG?**

ليس كل AI solution يحتاج RAG.

نستخدم RAG عندما نحتاج أن يجعل النظام قادرًا على:

```text
Answer from
private
domain-specific
changing
or external knowledge.
```

وهنا تبدأ المرحلة الأهم في **AI Automation Solution Design**:

```text
AI-Powered Workflow
        ↓
AI Assistant
        ↓
AI Agent
        ↓
RAG
        ↓
RAG + Agent
        ↓
Complete AI Automation Solution
```

> **The goal is not to use RAG because we learned RAG.**
>
> **The goal is to know when RAG is the right architecture for the problem.**

