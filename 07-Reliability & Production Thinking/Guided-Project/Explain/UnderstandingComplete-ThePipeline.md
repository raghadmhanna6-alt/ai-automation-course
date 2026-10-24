# 🧠 RAG with n8n & Supabase
## Understand the Pipeline: From Documents to Vector Search

## 1. 🎯 Learning Objectives

في هذا الدليل، سنتعلم كيف تعمل مكونات RAG معًا، وكيف نربطها باستخدام **n8n وSupabase**.

بنهاية هذا الدليل، يجب أن نستطيع تفسير:

- كيف نحول الملفات إلى بيانات قابلة للمعالجة.
- ما الفرق بين `Document` و`Chunk` و`Embedding` و`Vector`.
- ما وظيفة `Default Data Loader` و`Recursive Character Text Splitter` و`Embedding Model`.
- كيف نجهز Supabase لتخزين الـ Embeddings والبحث فيها.
- لماذا ننسخ SQL من Supabase LangChain Quickstart ونشغّله.
- ما وظيفة LangChain، وهل نحتاجه حتى تعمل Supabase Vector؟
- كيف نسترجع المعلومات المناسبة ونرسلها إلى LLM لتوليد الإجابة.

---

# 2. 🗺️ The Big Picture — RAG Architecture

قبل أن نفهم كل مكوّن، علينا رؤية العملية كاملة.

يتكون RAG من مرحلتين أساسيتين:

1. **Knowledge Ingestion:** تجهيز المعرفة وتخزينها.
2. **Query & Retrieval:** البحث عن المعرفة المناسبة عند وصول سؤال.

ثم تأتي مرحلة توليد الإجابة باستخدام الـ LLM.

### Phase A — Knowledge Ingestion

```text
Documents
    ↓
Load / Extract Content
    ↓
Documents / Text
    ↓
Chunking
    ↓
Chunks
    ↓
Embedding Model
    ↓
Embeddings / Vectors
    ↓
Supabase + pgvector
    ↓
Store Chunks + Vectors + Metadata
```

### Phase B — Query & Retrieval

```text
User Question
    ↓
Embedding Model
    ↓
Query Vector
    ↓
Supabase + pgvector
    ↓
Similarity Search
    ↓
Relevant Chunks
```

### Phase C — Generation

```text
User Question
       +
Retrieved Context
       ↓
      LLM
       ↓
   Final Answer
```

**الفكرة الأساسية:** نجهز المعرفة مرة، ثم نسترجع الأجزاء المناسبة منها عندما تصل الأسئلة. وإذا تغيرت المستندات، فقد نحتاج إلى تحديث المعرفة المخزنة.

---

# 3. 📄 Step One — Load the Knowledge

## ما الذي نبدأ به؟

لنفترض أن لدينا ملفًا اسمه:

`HR_Policy.pdf`

يحتوي على معلومات عن الإجازات والتأمين وسياسة العمل.

لكن وجود الملف وحده لا يعني أن نظام RAG أصبح قادرًا على البحث في محتواه.

نحتاج إلى تحويل محتواه إلى بيانات يمكن للمكونات التالية معالجتها.

هنا تظهر مكونات مثل:

- `Extract From File`
- `Default Data Loader`

### A. Extract From File

تُستخدم هذه العقدة لاستخراج المحتوى من ملف مدعوم، مثل PDF، عندما يصل إليها الملف بصيغة مناسبة.

```text
PDF File
    ↓
Extract From File
    ↓
Extracted Text
```

الناتج هو محتوى مستخرج يمكن استخدامه في الخطوات التالية.

### B. Default Data Loader

تُستخدم هذه العقدة في n8n لتجهيز البيانات على هيئة `Document` يمكن لمكونات LangChain التعامل معها.

عادةً يحتوي الـ Document على:

- `pageContent`: محتوى النص.
- `metadata`: معلومات إضافية مرتبطة بالمحتوى، بحسب المصدر والإعدادات.

مثال توضيحي:

```text
Document
├── pageContent
│   └── Employees receive 20 days of annual leave.
│
└── metadata
    ├── source: HR_Policy.pdf
    └── page: 4
```

### هل نحتاج العقدتين دائمًا؟

لا.

يعتمد ذلك على مصدر البيانات وطريقة وصول المحتوى إلى الـ workflow.

إذا كانت لديك عقدة استخراج سابقة، فقد تستخدمين ناتجها في تجهيز الـ Documents. وفي مسارات أخرى، قد يستطيع الـ Data Loader قراءة المصدر المدعوم مباشرةً.

**لا نضيف عقدة لمجرد وجودها في tutorial؛ نحدد أولًا ما البيانات الداخلة إليها وما الناتج الذي نحتاجه منها.**

---

# 4. ✂️ Step Two — Chunking

بعد تجهيز المحتوى، قد يكون لدينا مستند طويل جدًا.

مثلًا، يحتوي ملف سياسة الموارد البشرية على 50 صفحة.

لا نريد بالضرورة التعامل مع الملف كاملًا كوحدة واحدة عند البحث عن إجابة محددة.

لذلك نقسم المحتوى إلى أجزاء أصغر تسمى:

**Chunks**

## Recursive Character Text Splitter

هذا المكوّن يقسم المستند إلى أجزاء وفق إعدادات محددة.

يحاول التقسيم باستخدام فواصل مناسبة، مثل نهايات الفقرات والجمل، ثم ينتقل إلى فواصل أصغر عند الحاجة.

مثال:

```text
Original Document
       │
       ▼
Recursive Character Text Splitter
       │
       ├── Chunk 1: Introduction
       │
       ├── Chunk 2: Annual Leave
       │
       ├── Chunk 3: Sick Leave
       │
       └── Chunk 4: Remote Work
```

هذه أمثلة توضيحية؛ التقسيم الفعلي يعتمد على بنية النص وإعدادات الـ Splitter.

### لماذا نحتاج Chunking؟

لأننا نريد استرجاع الجزء الذي يجيب عن السؤال، بدل استرجاع مستند طويل لا نحتاج إلى معظم محتواه.

لكن حجم الـ Chunk يؤثر في جودة الاسترجاع:

- إذا كان كبيرًا جدًا، فقد يحتوي على موضوعات متعددة ويصبح أقل تركيزًا.
- إذا كان صغيرًا جدًا، فقد يفقد معلومات ضرورية لفهم السياق.

لذلك نضبط إعدادات مثل:

- `chunk_size`: الحجم المستهدف للـ Chunk.
- `chunk_overlap`: مقدار التداخل بين الأجزاء المتجاورة.

التداخل قد يساعد في الحفاظ على السياق الذي يقع عند حدود الأجزاء، لكنه ليس بديلًا عن اختيار حجم مناسب.

**الناتج من هذه الخطوة هو Chunks، وليس Embeddings بعد.**

---

# 5. 🔢 Step Three — Embeddings

بعد تقسيم المستند، أصبح لدينا مجموعة من النصوص.

مثال:

```text
Chunk 1:
Employees receive 20 days of annual leave.

Chunk 2:
Sick leave requires medical documentation.

Chunk 3:
Remote work requires manager approval.
```

نريد تخزين هذه النصوص بطريقة تسمح لنا بالبحث عن المعلومات ذات المعنى المشابه للسؤال، حتى عندما تختلف الكلمات المستخدمة.

هنا نستخدم:

**Embedding Model**

## ما وظيفة Embedding Model؟

يحوّل النص إلى تمثيل رقمي يسمى:

**Embedding / Vector**

مثال توضيحي:

```text
Text:
Employees receive 20 days of annual leave.
               ↓
        Embedding Model
               ↓
Vector:
[0.12, -0.43, 0.81, 0.22, ...]
```

الأرقام هنا افتراضية للتوضيح.

النموذج هو الذي ينتج الـ Embedding، وليس Supabase أو SQL.

### لماذا نحتاج Embeddings؟

لأنها تتيح إجراء بحث دلالي `Semantic Search`، بحيث يمكن للنظام العثور على نص مرتبط بمعنى السؤال، حتى لو لم تتطابق الكلمات حرفيًا.

مثلًا:

```text
Stored Text:
Employees receive 20 days of annual leave.

User Question:
How many vacation days can I take?
```

الكلمات مختلفة، لكن المعنى متقارب.

يمكن تحويل النص المخزن والسؤال إلى Embeddings باستخدام النموذج نفسه، ثم البحث عن المتجهات الأقرب وفق مقياس التشابه المستخدم.

**مهم:** الـ Embedding ليس الإجابة، ولا يضمن وحده أن النص المسترجع صحيح أو مناسب. إنه تمثيل رقمي يُستخدم للمقارنة والاسترجاع.

---

# 6. 🗄️ Step Four — What Is Supabase Vector?

هنا نصل إلى الجزء الأساسي.

عندما نقول `Supabase Vector` في هذا المشروع، نقصد استخدام قاعدة بيانات Supabase لتخزين الـ Embeddings وإجراء البحث المتجهي، بالاعتماد على PostgreSQL وامتداد `pgvector`.

تخيلي البنية التالية:

```text
Supabase
   │
   └── PostgreSQL
          │
          └── pgvector
                 │
                 ├── Vector Data Type
                 ├── Vector Operations
                 └── Similarity Search
```

## ما وظيفة pgvector؟

`pgvector` هو امتداد لـ PostgreSQL يضيف القدرة على تخزين بيانات من النوع `vector` وإجراء عمليات عليها، بما فيها البحث عن المتجهات المتشابهة.

إذن، لدينا ثلاثة مفاهيم منفصلة:

| المكوّن | وظيفته |
|---|---|
| Supabase | توفر قاعدة البيانات والخدمات المحيطة بها |
| PostgreSQL | نظام إدارة قاعدة البيانات |
| pgvector | يضيف دعم تخزين الـ Vectors وعمليات البحث المتجهي |

Supabase ليست هي التي تنشئ الـ Embeddings في هذا المسار؛ نموذج الـ Embeddings هو الذي ينشئها.

---

# 7. 📦 ماذا نخزن داخل Supabase؟

نحتاج إلى الاحتفاظ بالنص الأصلي، والـ Vector المرتبط به، والمعلومات الإضافية التي نحتاجها.

مثلًا، يمكن أن يحتوي جدول `documents` على:

| Column | Purpose |
|---|---|
| `id` | معرّف السجل |
| `content` | النص أو الـ Chunk |
| `metadata` | معلومات مثل اسم الملف أو الصفحة |
| `embedding` | الـ Vector المرتبط بالنص |

مثال توضيحي:

```text
documents
──────────────────────────────────────

id: 1

content:
"Employees receive 20 days of annual leave."

metadata:
{
  "source": "HR_Policy.pdf",
  "page": 4
}

embedding:
[0.12, -0.43, 0.81, 0.22, ...]
```

لماذا نخزن النص والـ Vector معًا؟

لأننا نبحث باستخدام الـ Vector، لكننا نحتاج إلى استرجاع النص الأصلي وإرساله إلى الـ LLM.

الـ Vector يساعدنا على العثور على المحتوى المناسب؛ أما النص فهو الذي يوفّر المعلومات التي سيستخدمها النموذج في الإجابة.

---

# 8. 🧩 Why Do We Run the SQL from Supabase's LangChain Quickstart?

عندما نفتح SQL Editor في Supabase وننسخ كودًا من LangChain Quickstart، قد يبدو لنا أننا نستخدم LangChain لبناء الـ Vector Database.

لكن ما نشغّله في SQL Editor هو **SQL يُنفَّذ داخل قاعدة البيانات**، وليس كود LangChain البرمجي نفسه.

في المثال الشائع الخاص بتكامل LangChain مع Supabase، يجهّز SQL ثلاثة أشياء رئيسية.

## A. Enable pgvector

مثال:

```sql
create extension if not exists vector
with schema extensions;
```

هذا يفعّل امتداد `pgvector` إذا لم يكن مفعّلًا بالفعل.

الهدف: توفير نوع بيانات `vector` وعمليات البحث المتجهي التي يدعمها الامتداد.

**هذا هو الجزء الذي يجهّز PostgreSQL لدعم الـ Vectors.**

## B. Create the Documents Table

مثال مبسط:

```sql
create table documents (
    id bigserial primary key,
    content text,
    metadata jsonb,
    embedding extensions.vector(1536)
);
```

هذا المثال ينشئ جدولًا يحتوي على النص والـ Metadata والـ Embedding.

القيمة `1536` مثال على عدد أبعاد الـ Vector. يجب أن يتطابق العدد الفعلي مع أبعاد الـ Embeddings التي ينتجها النموذج المستخدم.

قد تختلف صيغة الجدول أو مكان الامتداد بحسب إعدادات قاعدة البيانات والقالب المستخدم.

## C. Create the Similarity Search Function

قد ينشئ القالب دالة باسم:

`match_documents`

تستقبل عادةً Query Vector وعدد النتائج المطلوبة، وقد تقبل أيضًا شروط فلترة.

وظيفتها إجراء البحث عن الـ Vectors الأقرب إلى Query Vector وإرجاع السجلات المرتبطة بها.

مفهوميًا:

```text
Query Vector
      ↓
match_documents()
      ↓
Similarity Search
      ↓
Relevant Records
```

الدالة لا تولّد الـ Embeddings بنفسها. إنها تستخدم الـ Vector الذي يقدمه التطبيق لإجراء البحث في البيانات المخزنة.

### إذن ماذا يحقق تشغيل SQL؟

```text
Run SQL
   │
   ├── Enable pgvector
   │
   ├── Create documents table
   │
   └── Create search function
```

بعد تنفيذ هذه الخطوات، تصبح البنية المطلوبة متاحة للتخزين والبحث وفقًا لما أنشأه القالب.

لكن **قاعدة البيانات لن تحتوي تلقائيًا على معرفة ملفاتنا**. ما زلنا بحاجة إلى إرسال الـ Chunks والـ Embeddings إليها.

إذا كانت هذه العناصر موجودة بالفعل، يجب التحقق منها قبل إعادة تشغيل أو تعديل القالب.

---

# 9. 🤝 What Is LangChain's Role?

LangChain هو Framework يوفر مكونات وتكاملات تساعد المطور على بناء تطبيقات تعتمد على نماذج اللغة والبحث والاسترجاع.

في هذا المشروع، قد نستخدم تكاملًا باسم:

`SupabaseVectorStore`

هذا التكامل يساعد التطبيق على التعامل مع Supabase بوصفها Vector Store.

مفهوم الـ Vector Store هنا هو طبقة تتعامل مع تخزين الـ Embeddings واسترجاعها.

يمكن أن ينسق التكامل بين عمليات مثل:

- أخذ Documents.
- استدعاء Embedding Model لإنشاء Embeddings.
- إرسال المحتوى والـ Embeddings إلى Supabase.
- استرجاع المحتوى باستخدام البحث المتجهي.

تعتمد التفاصيل الدقيقة على التكامل وإعداداته وطريقة استخدامه.

### كيف تبدو العلاقة؟

```text
Application using LangChain
             │
             ▼
     SupabaseVectorStore
             │
             ▼
          Supabase
             │
             ▼
     PostgreSQL + pgvector
```

لاحظي أن `SupabaseVectorStore` هو تكامل برمجي، وليس بديلًا عن قاعدة البيانات أو امتداد `pgvector`.

### هل نحتاج LangChain حتى تعمل Supabase Vector؟

لا.

يمكن استخدام Supabase و`pgvector` مباشرةً من خلال SQL أو API أو أدوات أخرى تدعم الاتصال بهما.

يمكن لـ n8n مثلًا تنسيق عمليات تجهيز المعرفة والبحث فيها من دون الحاجة إلى إدخال LangChain كطبقة مستقلة في كل تصميم.

**إذن، لماذا نستخدم قالب LangChain؟**

لأنه يجهّز قاعدة البيانات بالشكل الذي يتوقعه المثال الخاص بتكامل LangChain، بما في ذلك جدول المستندات ودالة البحث.

نحن ننسخ SQL لتجهيز قاعدة البيانات، لا لأن LangChain هو الذي ينشئ الـ Vectors أو لأن Supabase لا تعمل من دونه.

---

# 10. 🔍 Step Five — Retrieval: Finding the Relevant Chunks

أصبحت قاعدة المعرفة جاهزة، والمستندات والـ Embeddings مخزنة في Supabase.

الآن يسأل المستخدم:

> How many annual leave days can I take?

لا نبحث بالضرورة عن الكلمات نفسها فقط.

بدلًا من ذلك، نحوّل السؤال إلى Query Embedding.

```text
User Question
      ↓
Embedding Model
      ↓
Query Vector
```

بعد ذلك نرسل الـ Query Vector إلى عملية البحث في Supabase.

```text
Query Vector
      ↓
Supabase / pgvector
      ↓
Similarity Search
      ↓
Ranked Results
      ↓
Top-K Chunks
```

## ما معنى Top-K؟

إذا حددنا `K = 3`، فنحن نطلب أفضل ثلاث نتائج وفق طريقة البحث وإعداداته.

مثال توضيحي:

```text
1. Annual leave policy
2. Vacation approval policy
3. Employee benefits
```

لكن أعلى نتيجة تشابهًا ليست بالضرورة الإجابة الصحيحة. لذلك نحتاج إلى اختبار جودة الاسترجاع والتأكد من أن النتائج تتضمن المعلومات المطلوبة.

### ماذا تعيد Supabase؟

تستطيع عملية البحث إعادة النصوص المرتبطة بالـ Vectors المطابقة، إلى جانب البيانات الأخرى التي تحددها الدالة.

مثلًا:

```json
[
  {
    "content": "Employees receive 20 days of annual leave.",
    "source": "HR_Policy.pdf"
  }
]
```

هنا استرجعنا الـ Chunk الذي نحتاجه.

---

# 11. 💬 Step Six — Generation: Let the LLM Answer

بعد الاسترجاع، أصبح لدينا:

1. سؤال المستخدم.
2. النصوص المسترجعة من قاعدة المعرفة.

نرسل الاثنين إلى الـ LLM ضمن Prompt مناسب.

مثال:

```text
SYSTEM INSTRUCTION:

Answer the question using the provided context.
If the context does not contain the answer,
say that you do not have enough information.

CONTEXT:

Employees receive 20 days of annual leave.

QUESTION:

How many annual leave days can I take?
```

يمكن أن ينتج النموذج:

> Employees receive 20 days of annual leave.

هنا حدثت مرحلة `Generation`.

لاحظي الفرق:

- Supabase استرجعت النصوص المناسبة.
- الـ LLM استخدم النصوص والسؤال لصياغة الإجابة.

وجود RAG لا يضمن صحة كل إجابة؛ يجب اختبار الاسترجاع، ودقة الإجابة، والتعامل مع الأسئلة التي لا تغطيها المعرفة.

---

# 12. 🔄 The Complete RAG Loop in n8n

الآن نجمع المكونات في صورة هندسية واحدة.

## A. Ingestion Workflow

هذا المسار مسؤول عن تجهيز المعرفة وإضافتها إلى قاعدة البيانات.

```text
Document Source
      ↓
Extract / Load Content
      ↓
Default Data Loader (when needed)
      ↓
Recursive Character Text Splitter
      ↓
Chunks
      ↓
Embedding Model
      ↓
Supabase Vector Store
      ↓
Supabase Database
```

في n8n قد تظهر بعض هذه العمليات داخل عقدة AI واحدة أو من خلال عقد مترابطة، بحسب نوع العقدة والإعدادات.

لا يعني ذلك أن كل عقدة تؤدي جميع الوظائف بنفسها؛ يجب فهم مسؤولية كل مكوّن وطريقة الاتصال بينه وبين بقية المكونات.

## B. Query Workflow

هذا المسار يعمل عند وصول سؤال.

```text
User Question
      ↓
Embedding Model
      ↓
Query Vector
      ↓
Supabase Vector Search
      ↓
Relevant Chunks
      ↓
Build Context
      ↓
LLM
      ↓
Answer
```

وقد تكون عملية البحث متاحة عبر Supabase Vector Store integration أو دالة SQL مثل `match_documents`، وفق طريقة البناء.

## C. الفرق بين المسارين

| Ingestion | Query & Retrieval |
|---|---|
| يبدأ من المستندات | يبدأ من سؤال المستخدم |
| يقسم المحتوى إلى Chunks | يحول السؤال إلى Query Embedding |
| ينشئ Embeddings للمحتوى | يستخدم الـ Query Vector للبحث |
| يخزن المحتوى والـ Embeddings | يسترجع المحتوى الأقرب |
| يجهز المعرفة للاستعمال لاحقًا | يجهز Context للإجابة الحالية |

**مهم:** نستخدم نموذج Embeddings متوافقًا مع النموذج الذي أنشأنا به الـ Vectors المخزنة. تغيير نموذج الـ Embeddings أو أبعادها يتطلب التعامل مع توافق البيانات وإعادة فهرستها عند الحاجة.

---

# 13. 🧠 Final Mental Model

احفظي وظيفة كل مكوّن:

| Component | Responsibility |
|---|---|
| Extract From File | استخراج المحتوى من ملف عندما يكون ذلك مطلوبًا |
| Default Data Loader | تجهيز المحتوى على هيئة Documents |
| Recursive Character Text Splitter | تقسيم المحتوى إلى Chunks |
| Embedding Model | تحويل النصوص إلى Embeddings / Vectors |
| pgvector | توفير نوع بيانات Vector وعمليات البحث المتجهي |
| Supabase | استضافة قاعدة البيانات وتخزين السجلات وإتاحتها للتطبيق |
| SQL Search Function | تنفيذ البحث عن المتجهات وفق منطق الدالة |
| LangChain | توفير تكاملات ومكونات برمجية لتنسيق هذه العمليات |
| Retriever | استرجاع المحتوى المناسب من مصدر المعرفة |
| LLM | توليد الإجابة باستخدام السؤال والـ Context |

## القاعدة الذهبية

**Chunking ≠ Embedding ≠ Storage ≠ Retrieval ≠ Generation**

كل واحدة من هذه مسؤولية مختلفة.

ويمكن أن تجمع أداة واحدة عدة مسؤوليات في واجهة جاهزة، لكن هذا لا يجعلها الوظيفة نفسها.

---

# 14. ✅ What We Should Understand Before Building RAG

قبل أن نبني RAG، يجب أن نستطيع الإجابة عن هذه الأسئلة:

1. لماذا نقسم المستندات إلى Chunks؟
2. من الذي ينشئ الـ Embeddings؟
3. لماذا نخزن النص الأصلي بجانب الـ Vector؟
4. ما الذي يضيفه `pgvector` إلى PostgreSQL؟
5. ما الذي يجهزه SQL الذي نشغّله في Supabase؟
6. لماذا قد نستخدم `match_documents`؟
7. ما الفرق بين LangChain وSupabase وpgvector؟
8. كيف يتحول سؤال المستخدم إلى Query Vector؟
9. كيف ننتقل من Query Vector إلى Retrieved Chunks؟
10. لماذا يحتاج الـ LLM إلى الـ Context المسترجع؟
11. كيف نختبر أن الـ Chunks المسترجعة مناسبة للسؤال؟
12. كيف نعرف أن الإجابة مدعومة بالمعلومات المسترجعة؟

إذا استطعنا شرح هذه النقاط، نكون قد فهمنا الأساس الهندسي للـ RAG بدل الاكتفاء بتوصيل العقد وتشغيلها.
