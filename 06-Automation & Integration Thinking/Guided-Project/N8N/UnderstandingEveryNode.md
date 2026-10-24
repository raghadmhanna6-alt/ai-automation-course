# U6 Project — PlanCraft AI
## Understanding Every Node and Why It Exists

> هذا الملف يشرح **منطق الـ Workflow** في مشروع PlanCraft AI، وليس فقط كيفية بنائه.
>
> الهدف هو أن تفهمي **لماذا استخدمنا كل Node، ما مسؤوليتها، ما البيانات التي تستقبلها، وما البيانات التي تخرجها، وكيف ترتبط بباقي النظام**.

---

# 1. The Big Picture

PlanCraft AI يتكون من ثلاثة أجزاء رئيسية:

```text
Lovable
   │
   │ User Interface
   │
   ▼
Supabase
   │
   │ Data Storage
   │
   ▼
n8n
   │
   │ Workflow + AI Processing
   │
   ▼
Supabase
```

لكن الـ Workflow الفعلي يبدأ عندما يرسل التطبيق طلبًا إلى n8n:

```text
Lovable
   │
   │ POST
   │ { request_id }
   ▼
Webhook
   │
   ▼
Prepare Data
   │
   ▼
Get Request from Supabase
   │
   ▼
Prepare AI Input
   │
   ▼
AI Model
   │
   ▼
Validate / Structure Output
   │
   ▼
Save Plan to Supabase
   │
   ▼
Update Request Status
```

الفكرة الأساسية:

> **كل Node لها مسؤولية محددة.**

لا نضيف Node لأننا نستطيع إضافتها، بل لأن هناك وظيفة واضحة تحتاج إلى تنفيذها.

---

# 2. The Engineering Problem

المستخدم يدخل بيانات مشروع في PlanCraft AI:

- Project Title
- Description
- Main Goal
- Additional Context

ثم يريد الحصول على **Project Plan** مولّد بالـ AI.

لكننا لا نريد أن نجعل Lovable يقوم بكل العمل.

نريد فصل المسؤوليات:

### Lovable

مسؤول عن:

- User Interface
- Form
- User interaction
- Sending the request

### Supabase

مسؤول عن:

- Storing the request
- Storing the generated plan
- Maintaining request state

### n8n

مسؤول عن:

- Receiving the request
- Retrieving the data
- Orchestrating the workflow
- Calling AI
- Validating the result
- Saving the result
- Updating status

وهذا يسمى:

**Separation of Responsibilities**

---

# 3. Why Does Lovable Send `request_id`?

هذه نقطة أساسية في تصميم المشروع.

عندما يرسل Lovable الطلب إلى n8n، لا نرسل كل بيانات المشروع مرة أخرى.

نرسل:

```json
{
  "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

لماذا؟

لأن بيانات المشروع موجودة أصلًا في:

```text
Supabase
    ↓
project_requests
```

فالـ `request_id` يعمل كـ **Reference** إلى السجل.

بدل:

```text
Lovable
   ↓
request_id
title
description
goal
additional_context
   ↓
n8n
```

نستخدم:

```text
Lovable
   ↓
request_id
   ↓
n8n
   ↓
Supabase
   ↓
full request
```

وهذا يمنع تكرار البيانات ويجعل Supabase هو:

> **Source of Truth**

أي المصدر الأساسي للبيانات.

---

# 4. Node 1 — Webhook

## Role

الـ Webhook هو **نقطة دخول البيانات إلى n8n**.

يمكن اعتباره باب النظام:

```text
External Application
        ↓
     Webhook
        ↓
      n8n
```

في مشروعنا، Lovable سيرسل:

```http
POST
```

مع:

```json
{
  "request_id": "..."
}
```

---

## Why Do We Need It?

لأن n8n يحتاج إلى طريقة يستقبل بها الطلب من تطبيق خارجي.

بدون Webhook:

```text
Lovable
     X
n8n
```

مع Webhook:

```text
Lovable
   ↓ HTTP POST
Webhook
   ↓
n8n
```

---

## Input

```json
{
  "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

---

## Output

يصبح الطلب متاحًا داخل الـ Workflow لكي تستخدمه الـ Nodes التالية.

قد يظهر في n8n بالشكل:

```json
{
  "body": {
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

## Important Concept

**Webhook = Entry Point**

أي نقطة دخول النظام.

---

# 5. Node 2 — Edit Fields / Set

## Role

هذه الـ Node تستخدم لـ **Data Preparation**.

أي تجهيز البيانات قبل إرسالها إلى الخطوة التالية.

---

## Why Do We Need It?

البيانات القادمة من Webhook قد تكون داخل:

```text
body.request_id
```

بينما نريد أن تصبح لدينا قيمة واضحة:

```text
request_id
```

يمكن استخدام Edit Fields من أجل:

- اختيار البيانات المطلوبة
- إعادة تسمية الحقول
- إنشاء حقول جديدة
- حذف بيانات غير ضرورية
- تجهيز Payload للخطوة التالية

---

## Important

في تصميم PlanCraft AI:

**لا نحتاج إلى كتابة بيانات المشروع يدويًا هنا.**

أي لا نريد أن نكتب:

```text
request_id
title
description
goal
additional_context
```

ونضع قيمًا ثابتة لها.

لأن بيانات المشروع موجودة في Supabase.

نحتاج في هذه المرحلة إلى تجهيز:

```text
request_id
```

حتى نستخدمه في استرجاع السجل.

---

## Input

```json
{
  "body": {
    "request_id": "123"
  }
}
```

## Output

يمكن أن يصبح:

```json
{
  "request_id": "123"
}
```

---

## Important Concept

**Edit Fields = Data Preparation**

وليست قاعدة بيانات.

وليست مكانًا لتخزين بيانات المشروع.

---

# 6. Node 3 — Supabase: Get Request

## Role

هذه الـ Node تسترجع بيانات المشروع من Supabase باستخدام `request_id`.

التدفق:

```text
request_id
    ↓
Supabase
    ↓
project_requests
    ↓
Project Record
```

---

## Why Do We Need It?

الـ Webhook أعطانا:

```text
request_id
```

لكن الـ AI يحتاج إلى تفاصيل المشروع.

لذلك نستخدم الـ ID للعثور على السجل الحقيقي.

مثلاً:

```json
{
  "id": "123",
  "title": "AI Automation Course",
  "description": "Create a practical AI Automation course.",
  "goal": "Organize the course into eight learning units.",
  "additional_context": "Include practical projects and homework.",
  "status": "pending"
}
```

---

## Why Not Send All Data from Lovable?

يمكن تقنيًا إرسال كل البيانات:

```json
{
  "request_id": "123",
  "title": "...",
  "description": "...",
  "goal": "...",
  "additional_context": "..."
}
```

لكننا اخترنا إرسال `request_id` فقط.

لأن:

```text
Supabase = Source of Truth
```

وبذلك لا نحتفظ بنسختين مختلفتين من نفس البيانات.

---

## Important Concept

**Reference-based Integration**

بدل إرسال البيانات كاملة، نرسل Reference للوصول إليها.

---

# 7. Node 4 — Data Transformation / Preparation

بعد استرجاع المشروع، قد تحتوي البيانات على حقول كثيرة:

```json
{
  "id": "123",
  "title": "...",
  "description": "...",
  "goal": "...",
  "additional_context": "...",
  "status": "pending",
  "created_at": "...",
  "updated_at": "..."
}
```

لكن الـ AI لا يحتاج إلى كل هذه المعلومات.

لذلك نجهز البيانات التي يحتاجها:

```json
{
  "title": "...",
  "description": "...",
  "goal": "...",
  "additional_context": "..."
}
```

---

## Why Is Transformation Important?

لأن الأنظمة المختلفة لا تستخدم دائمًا نفس شكل البيانات.

مثلاً:

```text
Database Format
       ↓
Transformation
       ↓
AI Input Format
```

وهذا أحد أهم مفاهيم Integration.

---

## Important Concept

**Transformation = تحويل البيانات من شكل إلى شكل مناسب للنظام التالي.**

---

# 8. Node 5 — AI Model

## Role

هنا يدخل الذكاء الاصطناعي في الـ Workflow.

لكن من المهم جدًا فهم نقطة:

> **AI ليس هو الـ Workflow.**

AI هو Component داخل Workflow أكبر.

---

## Input

مثلاً:

```json
{
  "title": "AI Automation Course",
  "description": "Create a practical AI Automation course.",
  "goal": "Organize the course into eight learning units.",
  "additional_context": "Include practical projects."
}
```

---

## AI Task

نطلب من النموذج إنشاء خطة مشروع منظمة.

مثلاً:

```text
Analyze the project information and generate
a structured project plan.
```

---

## Output

يجب أن يكون الناتج Structured وليس نصًا عشوائيًا.

مثلاً:

```json
{
  "summary": "...",
  "project_goal": "...",
  "phases": [],
  "recommended_actions": [],
  "risks": [],
  "estimated_duration": "..."
}
```

---

# 9. Why Should AI Return Structured Data?

لو تركنا AI يعيد نصًا حرًا:

```text
Your project should start by...
Then you should...
There are several risks...
```

سيصبح من الصعب على النظام تخزين النتيجة في Database.

لكن عندما نطلب:

```json
{
  "summary": "...",
  "phases": [],
  "recommended_actions": [],
  "risks": [],
  "estimated_duration": "..."
}
```

يمكن للنظام التعامل معها برمجيًا.

وهنا يظهر مفهوم:

**Output Contract**

أي أننا نحدد مسبقًا الشكل الذي يجب أن يعيده AI.

---

# 10. Node 6 — Validate / Structure Output

## Role

نتحقق من أن نتيجة AI صالحة قبل تخزينها.

AI قد:

- ينسى Field
- يعيد نوع بيانات خاطئ
- يعيد JSON غير صالح
- يعيد قيمة فارغة
- ينتج Structure مختلفًا

لذلك لا نريد:

```text
AI
 ↓
Database
```

مباشرة.

نريد:

```text
AI
 ↓
Validate
 ↓
Database
```

---

## Example

نتوقع:

```json
{
  "summary": "...",
  "phases": [],
  "recommended_actions": [],
  "risks": [],
  "estimated_duration": "..."
}
```

فنحتاج إلى التأكد من وجود البيانات المطلوبة وأن شكلها مناسب.

---

## Important Concept

**Validation**

يعني:

> هل البيانات الناتجة صالحة للخطوة التالية؟

---

# 11. Node 7 — Supabase: Save Plan

## Role

بعد أن أصبح لدينا AI Output صالح، نخزنه في:

```text
project_plans
```

---

## Example

```json
{
  "request_id": "123",
  "summary": "A structured project plan...",
  "project_goal": "...",
  "phases": [],
  "recommended_actions": [],
  "risks": [],
  "estimated_duration": "6 weeks"
}
```

---

## Why Separate `project_requests` and `project_plans`?

لأنهما يمثلان شيئين مختلفين.

### `project_requests`

يمثل:

> What did the user ask for?

### `project_plans`

يمثل:

> What did the system generate?

وهذا Separation مهم في تصميم قاعدة البيانات.

---

# 12. Node 8 — Update Request Status

## Role

تحديث حالة الطلب.

مثلاً:

```text
pending
   ↓
processing
   ↓
completed
```

أو عند حدوث مشكلة:

```text
pending
   ↓
processing
   ↓
failed
```

---

## Why Do We Need Status?

لأن إنشاء الخطة عملية قد تستغرق وقتًا.

التطبيق يحتاج إلى معرفة:

> ماذا يحدث لهذا الطلب الآن؟

مثلاً:

```text
pending
```

يعني:

> الطلب موجود وينتظر المعالجة.

```text
processing
```

يعني:

> النظام يعمل على إنشاء الخطة.

```text
completed
```

يعني:

> تم إنشاء الخطة وحفظها.

```text
failed
```

يعني:

> حدث خطأ ولم تكتمل العملية.

---

# 13. The Complete Data Journey

الآن يمكننا رؤية الرحلة كاملة:

```text
USER
 │
 │ fills project form
 ▼
LOVABLE
 │
 │ POST { request_id }
 ▼
WEBHOOK
 │
 │ receives request
 ▼
EDIT FIELDS
 │
 │ prepares request_id
 ▼
SUPABASE
 │
 │ retrieves project_requests record
 ▼
TRANSFORMATION
 │
 │ prepares AI input
 ▼
AI MODEL
 │
 │ generates project plan
 ▼
VALIDATION
 │
 │ checks output
 ▼
SUPABASE
 │
 │ saves project_plans
 ▼
STATUS UPDATE
 │
 │ completed / failed
 ▼
LOVABLE
 │
 │ reads updated state
 ▼
USER
```

---

# 14. Why This Architecture?

نحن لا نبني Workflow عشوائيًا.

كل جزء له مسؤولية:

| Component | Responsibility |
|---|---|
| Lovable | User Interface |
| Webhook | Entry Point |
| Edit Fields | Data Preparation |
| Supabase Get | Retrieve Source Data |
| Transformation | Adapt Data |
| AI | Generate / Reason |
| Validation | Check Output |
| Supabase Save | Persist Result |
| Status Update | Track State |

---

# 15. The Most Important Engineering Principle

لا تسألي فقط:

> "ما الـ Node التي أستخدمها؟"

اسألي:

> **"ما المشكلة التي أحتاج إلى حلها في هذه المرحلة؟"**

مثلاً:

### أحتاج استقبال البيانات

→ Webhook

### أحتاج إعادة تنظيم البيانات

→ Edit Fields / Transformation

### أحتاج بيانات محفوظة

→ Supabase

### أحتاج Generation أو Reasoning

→ AI

### أحتاج التأكد من صحة الناتج

→ Validation

### أحتاج حفظ النتيجة

→ Supabase

### أحتاج معرفة حالة العملية

→ Status

---

# 16. Why Not Put Everything in One Node?

قد يبدو أسهل أن نجعل Node واحدة تقوم بكل شيء.

لكن ذلك يجعل النظام:

- أصعب في الفهم
- أصعب في الاختبار
- أصعب في Debugging
- أصعب في التعديل
- أصعب في اكتشاف مكان الخطأ

بدلًا من ذلك:

```text
Receive
   ↓
Prepare
   ↓
Retrieve
   ↓
Transform
   ↓
Generate
   ↓
Validate
   ↓
Save
   ↓
Update
```

كل مرحلة لها مسؤولية واضحة.

وهذا يسمى:

**Separation of Concerns**

---

# 17. Debugging Through the Nodes

ميزة تقسيم الـ Workflow إلى Nodes واضحة هي أننا نستطيع معرفة مكان الخطأ.

مثلاً:

```text
Webhook        ✓
Edit Fields    ✓
Supabase Get   ✗
```

المشكلة غالبًا في:

- `request_id`
- Supabase credentials
- Query
- Record not found

أما إذا:

```text
Webhook        ✓
Edit Fields    ✓
Supabase Get   ✓
AI             ✗
```

فالمشكلة غالبًا في:

- AI credentials
- Prompt
- Model
- Input
- Output format

وهذا يجعل الـ Workflow **Observable and Debuggable**.

---

# 18. Testing the Webhook Before Lovable

قبل ربط Lovable، نختبر الـ Webhook يدويًا باستخدام Postman.

### Method

```text
POST
```

### Body

```json
{
  "request_id": "REAL_REQUEST_ID_FROM_SUPABASE"
}
```

### Header

```text
Content-Type: application/json
```

الهدف من هذا الاختبار هو محاكاة ما سيقوم به Lovable لاحقًا.

```text
Postman
   ↓
POST
   ↓
Webhook
```

إذا نجح:

```text
Webhook receives request
```

ننتقل إلى اختبار بقية الـ Workflow.

---

# 19. Why Use a Real `request_id`?

إذا كانت الخطوة التالية تبحث في Supabase عن:

```text
request_id
```

فلا يكفي استخدام:

```text
test-request-001
```

إلا إذا كان هذا السجل موجودًا فعلًا.

الأفضل أن نستخدم ID حقيقيًا من:

```text
project_requests
```

مثلاً:

```json
{
  "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

حيث تكون هذه القيمة موجودة فعليًا في قاعدة البيانات.

---

# 20. What We Are Actually Learning in U6

هذا المشروع ليس مجرد مشروع n8n.

هو تطبيق عملي لمفاهيم Integration:

### CONNECT

كيف يتصل Lovable بـ n8n؟

```text
HTTP POST → Webhook
```

### COMMUNICATE

ما البيانات التي تنتقل؟

```json
{
  "request_id": "..."
}
```

### AUTHENTICATE

كيف نحمي الاتصال والـ credentials؟

```text
Secrets / Server-side configuration
```

### TRANSFORM

كيف نغير شكل البيانات بين الأنظمة؟

```text
Webhook Data
    ↓
Database Data
    ↓
AI Input
    ↓
AI Output
    ↓
Database Format
```

### ORCHESTRATE

كيف نرتب العمليات؟

```text
Receive
→ Retrieve
→ Generate
→ Validate
→ Save
→ Update
```

### VERIFY

كيف نعرف أن النظام يعمل؟

```text
Test
→ Inspect
→ Validate
→ Monitor
```

---

# 21. Questions Every Student Should Be Able to Answer

بعد بناء الـ Workflow، يجب أن تكوني قادرة على الإجابة عن:

1. لماذا استخدمنا Webhook؟
2. ما الذي يرسله Lovable إلى n8n؟
3. لماذا نرسل `request_id` بدل كل بيانات المشروع؟
4. أين توجد البيانات الأصلية؟
5. لماذا نستخدم Supabase بعد Webhook؟
6. ما وظيفة Edit Fields؟
7. ما الفرق بين Retrieval وTransformation؟
8. لماذا نحتاج AI داخل الـ Workflow؟
9. لماذا نحتاج Structured Output؟
10. لماذا لا نحفظ AI Output مباشرة دون Validation؟
11. لماذا يوجد جدول `project_requests` وجدول `project_plans`؟
12. لماذا نحتاج `status`؟
13. ماذا يحدث إذا فشل AI؟
14. كيف نعرف أي Node سببت المشكلة؟
15. كيف نختبر Webhook قبل ربط Lovable؟
16. ما الفرق بين Test Webhook وProduction Webhook؟
17. ما البيانات التي يجب أن تبقى في Supabase؟
18. ما البيانات التي يجب أن تنتقل بين الأنظمة؟
19. أين يحدث Transformation؟
20. أين يحدث Verification؟

---

# 22. Final Mental Model

لا تحفظي أسماء الـ Nodes فقط.

احفظي الرحلة:

```text
DATA ENTERS
     ↓
DATA IS PREPARED
     ↓
SOURCE DATA IS RETRIEVED
     ↓
DATA IS TRANSFORMED
     ↓
AI PROCESSES THE DATA
     ↓
OUTPUT IS VALIDATED
     ↓
RESULT IS STORED
     ↓
STATE IS UPDATED
```

أو بشكل أبسط:

> **Receive → Retrieve → Transform → Generate → Validate → Store → Track**

هذه هي الفكرة الهندسية وراء الـ Workflow.

---

# 23. Final Principle

> **A good automation workflow is not a collection of connected nodes.**
>
> **It is a deliberate data flow where every step has a clear responsibility, a defined input, a defined output, and a reason to exist.**

وهذا هو الهدف الحقيقي من بناء PlanCraft AI في U6:

> لا نريد فقط أن تعرفي كيف تربطين Lovable بـ n8n.
>
> نريدك أن تعرفي **كيف تصممين Integration System يمكن فهمه، اختباره، Debugging له، وتطويره بثقة.**
