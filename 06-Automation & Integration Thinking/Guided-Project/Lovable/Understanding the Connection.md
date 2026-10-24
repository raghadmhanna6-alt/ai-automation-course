## PlanCraft AI — Integration Architecture

---

## 1. الهدف من هذا الملف

في هذا الجزء سنتعلم كيف نربط نظامًا مبنيًا باستخدام عدة أدوات بحيث تعمل معًا كـ **System واحد**.

سنستخدم:

- **Lovable** → واجهة التطبيق `Frontend`
- **Supabase** → قاعدة البيانات ومصدر الحقيقة `Source of Truth`
- **n8n** → تنفيذ الـ `Workflow` وعمليات الأتمتة
- **AI Agent** → توليد خطة المشروع
- **Server-side Function / Edge Function** → الجسر الآمن بين التطبيق و n8n

الهدف ليس فقط أن نجعل الأدوات "تتصل".

الهدف أن نفهم:

> **كيف تنتقل البيانات بين الأنظمة، ولماذا تنتقل، ومن المسؤول عن كل خطوة، وكيف نعرف أن العملية نجحت فعلًا؟**

---

# 2. الصورة الكبيرة

العملية الكاملة في PlanCraft AI هي:

```text
User
  │
  │ clicks "Generate My Plan"
  ▼
Lovable Frontend
  │
  │ request_id
  ▼
Server-side Function
  │
  │ POST
  ▼
n8n Production Webhook
  │
  ▼
Get Request from Supabase
  │
  ▼
Prepare AI Input
  │
  ▼
AI Agent
  │
  ▼
Validate / Structure Output
  │
  ▼
Save Plan in Supabase
  │
  ▼
Update Request Status
  │
  ▼
Supabase
  │
  │ status = completed
  ▼
Lovable Frontend
  │
  ▼
Display Real Project Plan
```

هذه هي الـ **Integration Architecture** للمشروع.

---

# 3. أهم فكرة: Lovable لا يشغّل n8n مباشرة

من الأخطاء الشائعة أن نفكر أن العملية هي:

```text
Lovable → n8n
```

بشكل مباشر من الـ browser.

في هذا المشروع، الاتصال الصحيح هو:

```text
Lovable Frontend
       ↓
Server-side Function
       ↓
n8n Webhook
```

لماذا؟

لأن بعض المعلومات يجب ألا تكون موجودة داخل الـ browser، مثل:

- `N8N_WEBHOOK_URL`
- Webhook Secret
- Supabase Service Role Key
- أي credentials أو secrets

لذلك نستخدم طبقة Server-side.

---

# 4. مسؤولية كل نظام

قبل أن نفهم الاتصال، يجب أن نعرف مسؤولية كل جزء.

## Lovable

مسؤوليته:

- عرض الـ UI
- استقبال إدخال المستخدم
- إنشاء `project_request`
- إعطاء المستخدم زر "Generate My Plan"
- بدء عملية التوليد
- متابعة حالة الطلب
- عرض النتيجة الحقيقية

Lovable ليس مسؤولًا عن تنفيذ الـ AI Workflow بالكامل.

---

## Supabase

Supabase هو:

> **Source of Truth**

أي المكان الذي توجد فيه البيانات الأساسية.

مثل:

```text
project_requests
project_plans
```

مثلًا، يوجد طلب:

```json
{
  "id": "8f4c...",
  "title": "Launch an Online Store",
  "description": "I want to build an online store.",
  "goal": "Start selling products online.",
  "additional_context": "Small business"
}
```

هذه البيانات موجودة في Supabase.

لذلك لا نحتاج إلى إرسال كل هذه البيانات من Lovable إلى n8n.

يكفي أن نرسل:

```json
{
  "request_id": "8f4c..."
}
```

---

# 5. لماذا نرسل request_id فقط؟

هذه نقطة مهمة جدًا.

بدل أن نرسل:

```json
{
  "request_id": "8f4c...",
  "title": "...",
  "description": "...",
  "goal": "...",
  "additional_context": "..."
}
```

نرسل فقط:

```json
{
  "request_id": "8f4c..."
}
```

ثم n8n يستخدم هذا الـ ID للوصول إلى Supabase وجلب البيانات الأصلية.

أي:

```text
request_id
    ↓
Supabase
    ↓
Full Project Request
```

### لماذا هذا التصميم أفضل؟

لأنه يجعل Supabase هو مصدر الحقيقة.

إذا كانت البيانات موجودة أصلًا في قاعدة البيانات، فلا يوجد داعٍ لنسخها وإرسالها مرة أخرى بين الأنظمة.

كما أن `request_id` يعمل كـ **Correlation ID** يربط كل أجزاء العملية ببعضها.

---

# 6. ما هو Correlation ID؟

`request_id` هو المفتاح الذي يسمح لنا بمعرفة:

> هذا الـ Workflow يعمل على أي Request؟

مثلًا:

```text
request_id = 8f4c...
```

يبدأ الطلب في:

```text
project_requests
```

ثم n8n يستخدم نفس الـ ID.

وفي النهاية يتم إنشاء:

```text
project_plans
```

مع:

```text
request_id = 8f4c...
```

وبالتالي نستطيع معرفة:

```text
Request
   ↓
request_id
   ↓
AI Workflow
   ↓
Generated Plan
```

هذا مهم جدًا عندما نتعامل مع أكثر من مستخدم وأكثر من Request في نفس الوقت.

---

# 7. الخطوة الأولى — المستخدم يبدأ العملية

المستخدم يضغط:

```text
Generate My Plan
```

الـ Frontend يعرف الـ Request الحالي.

مثلًا:

```text
request_id = 8f4c...
```

ثم يستدعي الـ Server-side Function.

---

# 8. لماذا نستخدم Server-side Function؟

هذه الطبقة تعمل كـ **Secure Bridge**.

بدل:

```text
Browser → n8n
```

نستخدم:

```text
Browser
   ↓
Server-side Function
   ↓
n8n
```

الـ Function تعرف:

```text
N8N_WEBHOOK_URL
```

لكن الـ Browser لا يعرفه.

---

# 9. Environment Variables

رابط n8n يجب أن يكون محفوظًا كـ Environment Variable:

```text
N8N_WEBHOOK_URL
```

وليس داخل كود الـ Frontend.

مثال مفاهيمي:

```text
N8N_WEBHOOK_URL
        │
        ▼
Server-side Function
        │
        ▼
n8n Production Webhook
```

### لا تفعل:

```javascript
const webhookUrl = "https://...";
```

داخل Frontend code.

لأن كود الـ Frontend يصل إلى Browser المستخدم.

---

# 10. ماذا يرسل Server-side Function إلى n8n؟

يرسل HTTP `POST`.

الـ Body:

```json
{
  "request_id": "8f4c..."
}
```

إذن الاتصال بين النظامين يعتمد على:

```text
HTTP
+
POST
+
JSON
```

---

# 11. n8n Webhook

أول Node في Workflow هو:

```text
Webhook
```

وظيفته:

> استقبال الطلب القادم من نظام خارجي وبدء الـ Workflow.

يمكن التفكير فيه كـ:

```text
Door into the Workflow
```

أي أن Webhook هو نقطة الدخول إلى n8n.

---

# 12. ماذا يدخل إلى Webhook؟

في مشروعنا:

```json
{
  "request_id": "8f4c..."
}
```

وهذا هو الـ Input الأساسي للـ Workflow.

لا يحتاج Webhook إلى معرفة كل تفاصيل المشروع.

هو يحتاج فقط إلى المرجع:

```text
request_id
```

---

# 13. Webhook لا يعني أن العملية انتهت

هذه نقطة مهمة جدًا.

إذا أرسل Lovable الطلب إلى n8n وحصل على:

```text
200 OK
```

هذا لا يعني:

> AI انتهى من إنشاء الخطة.

بل يعني غالبًا:

> تم قبول الطلب / التعامل معه بنجاح على مستوى الاتصال.

قد يكون الـ Workflow ما زال يعمل.

لذلك لا يجوز أن نقول:

```text
200 OK = Plan Generated
```

الصحيح:

```text
200 OK
      ↓
Request Accepted
      ↓
Workflow Processing
      ↓
AI Generation
      ↓
Database Save
      ↓
Status = completed
```

---

# 14. بعد Webhook — نحتاج بيانات المشروع

الـ Webhook أعطانا:

```text
request_id
```

لكن AI يحتاج معلومات المشروع.

لذلك نذهب إلى Supabase.

---

# 15. Get Request from Supabase

n8n يستخدم:

```text
request_id
```

لجلب الـ Request من:

```text
project_requests
```

مثلًا:

```json
{
  "id": "8f4c...",
  "title": "Launch an Online Store",
  "description": "I want to build an online store.",
  "goal": "Start selling products online.",
  "additional_context": "Small business"
}
```

الآن أصبح لدى Workflow كل المعلومات المطلوبة.

---

# 16. لماذا لا نرسل البيانات مباشرة من Webhook؟

يمكن تقنيًا أن نصمم النظام بحيث يرسل:

```json
{
  "request_id": "...",
  "title": "...",
  "description": "...",
  "goal": "..."
}
```

لكن هذا ليس التصميم الذي نستخدمه هنا.

نحن نريد:

```text
Supabase = Source of Truth
```

لذلك:

```text
Lovable
   ↓
request_id
   ↓
n8n
   ↓
Supabase
   ↓
Full Request
```

هذا يقلل تكرار البيانات ويحافظ على مصدر مركزي للحقيقة.

---

# 17. Data Transformation

بعد الحصول على البيانات، قد نحتاج إلى تجهيزها قبل إرسالها إلى AI.

وهنا تظهر فكرة:

> **Transformation**

أي تحويل البيانات من شكل إلى شكل مناسب للخطوة التالية.

مثلًا:

```text
Database Record
        ↓
AI Input
```

يمكن أن يصبح:

```text
Project Title:
Launch an Online Store

Description:
I want to build an online store.

Goal:
Start selling products online.

Additional Context:
Small business
```

المهم أن نفهم:

> Transformation لا يعني بالضرورة تعديل قاعدة البيانات.

هو فقط تجهيز البيانات للمرحلة التالية.

---

# 18. AI Agent

الآن وصلت البيانات إلى:

```text
AI Agent
```

وظيفة الـ AI هنا هي:

> توليد Project Plan بناءً على بيانات المشروع.

AI ليس مسؤولًا عن:

- استقبال HTTP
- حفظ البيانات
- معرفة أين تخزن النتائج
- إدارة حالة الطلب
- ربط الأنظمة

هذه مسؤولية الـ Workflow.

AI مسؤول عن الجزء الذي يحتاج:

```text
Understanding
Reasoning
Generation
```

---

# 19. AI ليس هو الـ Workflow

من الأخطاء الشائعة أن نفكر:

```text
AI = Automation
```

لكن الصحيح:

```text
Workflow
├── Receive Data
├── Retrieve Data
├── Prepare Data
├── AI Reasoning
├── Validate Output
├── Save Result
└── Update Status
```

الـ AI هو **Component داخل النظام**.

أما n8n فهو الذي يقوم بـ **Orchestration**.

---

# 20. Structured Output

نحن لا نريد من AI أن يعيد نصًا عشوائيًا فقط.

نريد نتيجة يمكن للنظام التعامل معها.

مثلًا:

```json
{
  "summary": "A concise project overview.",
  "project_goal": "Launch an online store.",
  "phases": [],
  "recommended_actions": [],
  "risks": [],
  "estimated_duration": "6 weeks"
}
```

هذا يسمى:

> **Structured Output**

أي أن نتيجة الـ AI لها شكل متوقع.

---

# 21. لماذا Structured Output مهم؟

لأن النتيجة ستذهب بعد ذلك إلى Database.

قاعدة البيانات تحتاج بيانات منظمة.

بدل:

```text
Here is your plan...
blah blah...
```

نحتاج:

```text
summary
project_goal
phases
recommended_actions
risks
estimated_duration
```

وهذا يجعل النظام:

- أسهل في التخزين
- أسهل في التحقق
- أسهل في العرض
- أسهل في المعالجة لاحقًا

---

# 22. Validation

بعد AI تأتي خطوة مهمة:

```text
Validate / Structure Output
```

نسأل:

- هل النتيجة موجودة؟
- هل `summary` موجود؟
- هل `phases` بالشكل المتوقع؟
- هل `risks` مصفوفة؟
- هل البيانات صالحة للحفظ؟

الفكرة:

> لا نثق بأن مخرجات AI صحيحة لمجرد أن AI أعاد شيئًا.

AI احتمالي.

أما النظام فيحتاج بيانات يمكن الاعتماد عليها.

---

# 23. Save Plan to Supabase

إذا كانت النتيجة صالحة، يقوم n8n بحفظها في:

```text
project_plans
```

ويرتبط الـ Plan بالـ Request عن طريق:

```text
request_id
```

أي:

```text
project_requests
       │
       │ request_id
       ▼
project_plans
```

وهكذا نعرف لأي Request تنتمي الخطة.

---

# 24. Update Request Status

لدينا في:

```text
project_requests
```

حقل:

```text
status
```

يمكن أن يمثل حالة العملية.

مثلًا:

```text
pending
processing
completed
failed
```

التدفق الطبيعي:

```text
pending
   ↓
processing
   ↓
completed
```

وفي حالة وجود مشكلة:

```text
processing
   ↓
failed
```

---

# 25. لماذا نحتاج Status؟

لأن عملية AI قد لا تكون فورية.

قد يكون هناك وقت بين:

```text
Generate My Plan
```

و:

```text
Plan Ready
```

لذلك يحتاج Frontend إلى معرفة:

> ماذا يحدث الآن؟

مثلًا:

```text
pending
```

يعني أن العملية لم تبدأ بعد.

```text
processing
```

يعني أن الـ Workflow يعمل.

```text
completed
```

يعني أن الخطة أصبحت محفوظة.

```text
failed
```

يعني أن العملية فشلت.

---

# 26. كيف يعرف Lovable أن الخطة أصبحت جاهزة؟

Lovable لا يعتمد فقط على رد Webhook.

بدل ذلك، يتابع حالة الـ Request في Supabase.

مثلًا:

```text
Supabase

status = processing
```

ثم بعد انتهاء n8n:

```text
status = completed
```

عندها يعرف Frontend أن عليه جلب:

```text
project_plans
```

وعرض الخطة الحقيقية.

---

# 27. الصورة الكاملة للبيانات

دعونا نتتبع البيانات نفسها.

### البداية

المستخدم ينشئ Request:

```text
Project Request
```

ويتم حفظه في:

```text
project_requests
```

مع:

```text
request_id
```

---

### بدء التوليد

Lovable يرسل:

```json
{
  "request_id": "8f4c..."
}
```

---

### n8n

Webhook يستقبل:

```text
request_id
```

ثم يستخدمه للحصول على:

```text
Project Request
```

من Supabase.

---

### AI

يستلم:

```text
Project Information
```

ويولد:

```text
Project Plan
```

---

### Database

n8n يحفظ:

```text
project_plans
```

ويربطها بـ:

```text
request_id
```

---

### Status

n8n يحدث:

```text
project_requests.status
```

إلى:

```text
completed
```

---

### Frontend

Lovable يلاحظ:

```text
status = completed
```

ثم يجلب:

```text
project_plans
```

ويعرض النتيجة.

---

# 28. Architecture النهائي

```text
                 ┌─────────────────────┐
                 │       User          │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │   Lovable Frontend  │
                 └──────────┬──────────┘
                            │
                      request_id
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Server-side Function│
                 │   Secure Bridge     │
                 └──────────┬──────────┘
                            │
                         POST
                            │
                            ▼
                 ┌─────────────────────┐
                 │    n8n Webhook      │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Get Request         │
                 │ from Supabase       │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Data Transformation │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │      AI Agent       │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Validate Output     │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Save Project Plan   │
                 │   to Supabase       │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Update Status       │
                 │ pending → completed │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │   Supabase          │
                 └──────────┬──────────┘
                            │
                       status check
                            │
                            ▼
                 ┌─────────────────────┐
                 │   Lovable Frontend  │
                 │ Display Real Plan   │
                 └─────────────────────┘
```

---

# 29. كيف نربط هذا بمفاهيم Unit 6؟

هذا المشروع يطبق Layers الوحدة السادسة:

## 1. CONNECT

ربط الأنظمة:

```text
Lovable
   ↓
Server-side Function
   ↓
n8n
```

والدخول إلى n8n يتم عن طريق:

```text
Webhook
```

---

## 2. COMMUNICATE

تحديد البيانات التي تنتقل بين الأنظمة.

في مشروعنا:

```json
{
  "request_id": "..."
}
```

إذن نحن لا نقول فقط:

> "الأدوات متصلة."

بل نحدد:

> ما الذي يرسله نظام إلى نظام آخر؟ وبأي شكل؟

---

## 3. AUTHENTICATE

حماية الاتصال.

مثل:

```text
N8N_WEBHOOK_URL
N8N_WEBHOOK_SECRET
Supabase credentials
```

المبدأ الأساسي:

> Secrets belong on the server, not in the browser.

---

## 4. TRANSFORM

تحويل البيانات بين أشكال مختلفة:

```text
Webhook Payload
      ↓
Database Record
      ↓
AI Input
      ↓
Structured AI Output
      ↓
Database Record
```

---

## 5. ORCHESTRATE

ترتيب العمليات:

```text
Receive
  ↓
Retrieve
  ↓
Prepare
  ↓
Generate
  ↓
Validate
  ↓
Save
  ↓
Update Status
```

وهنا يظهر دور n8n الحقيقي:

> **Workflow Orchestration**

---

## 6. VERIFY

لا نفترض أن النظام يعمل.

نختبره.

نراقب:

- Webhook input
- Supabase query
- AI output
- Validation
- Database insert
- Status update
- Frontend result

الهدف:

> إثبات أن البيانات وصلت، عولجت، حُفظت، وعادت للواجهة بالشكل الصحيح.

---

# 30. اختبار Webhook قبل ربط Lovable

قبل ربط Lovable، نستطيع محاكاة الطلب باستخدام أداة مثل Postman.

نستخدم:

```text
POST
```

إلى:

```text
n8n Test Webhook URL
```

والـ Body:

```json
{
  "request_id": "YOUR_REAL_SUPABASE_REQUEST_ID"
}
```

مع:

```text
Content-Type: application/json
```

### لماذا نفعل ذلك؟

لأننا نريد اختبار n8n بشكل مستقل.

بدل أن نواجه مشكلتين في نفس الوقت:

```text
Lovable Problem
+
n8n Problem
```

نختبر أولًا:

```text
n8n
```

ثم نربطه بـ:

```text
Lovable
```

هذه فكرة مهمة في الـ Debugging:

> **Isolate the components.**

---

# 31. Test URL vs Production URL

أثناء بناء الـ Workflow يوجد فرق بين:

```text
Test Webhook URL
```

و:

```text
Production Webhook URL
```

### Test Webhook

نستخدمه أثناء التطوير والاختبار.

### Production Webhook

هو الرابط الذي سيستخدمه النظام الحقيقي.

في PlanCraft:

```text
Lovable
   ↓
N8N_WEBHOOK_URL
   ↓
Production Webhook
```

ولا يجب وضع Test URL في التطبيق النهائي.

---

# 32. ما الذي لا يجب فعله؟

## لا تضع Webhook URL داخل Frontend

❌

```text
Frontend → hardcoded n8n URL
```

---

## لا تضع Service Role Key داخل Frontend

❌

أي secret يجب أن يبقى Server-side.

---

## لا تنشئ Fake AI Result

❌

لا يجوز أن يعرض التطبيق خطة وهمية فقط لكي يبدو أن النظام يعمل.

---

## لا تعتبر 200 OK نجاحًا كاملًا

❌

```text
200 OK
```

لا يعني بالضرورة:

```text
AI Plan Completed
```

---

## لا ترسل البيانات نفسها بلا حاجة

❌

إذا كان Supabase يحتوي البيانات، لا نحتاج إلى نسخها في كل اتصال.

---

## لا تنشئ Workflow جديدًا دون حاجة

❌

إذا كان Workflow موجودًا، نكمل الـ Integration بدل إنشاء نسخة ثانية.

---

# 33. كيف نفكر كـ AI Automation Engineer؟

عند ربط نظامين، لا نسأل فقط:

> كيف أجعلهم يتصلوا؟

نسأل:

### 1. What connects?

ما الأنظمة التي تتصل؟

```text
Lovable
n8n
Supabase
```

### 2. Why do they connect?

ما سبب الاتصال؟

```text
Start AI generation
```

### 3. What data moves?

ما البيانات التي تنتقل؟

```text
request_id
```

### 4. In what format?

بأي شكل؟

```json
{
  "request_id": "..."
}
```

### 5. Who owns the data?

من هو مصدر الحقيقة؟

```text
Supabase
```

### 6. Who performs the processing?

من ينفذ الـ Workflow؟

```text
n8n
```

### 7. Who performs reasoning?

من يقوم بالـ AI reasoning؟

```text
AI Agent
```

### 8. How do we know it worked?

كيف نتحقق؟

```text
Database status
+
Saved plan
+
Frontend display
```

هذه هي طريقة التفكير التي نريد بناءها في هذه الوحدة.

---

# 34. مسؤولية كل Component في جملة واحدة

| Component | Responsibility |
|---|---|
| Lovable | User Interface + start/check the process |
| Server-side Function | Secure bridge to n8n |
| Webhook | Entry point into n8n |
| Supabase | Source of Truth + persistence |
| n8n | Workflow orchestration |
| AI Agent | Reasoning + generation |
| Validation | Check AI output |
| Status | Represent process state |
| project_plans | Store generated result |

---

# 35. Mental Model

احفظي هذه الصورة:

```text
Lovable
= Interface

Supabase
= Source of Truth

Server-side Function
= Secure Bridge

n8n
= Orchestrator

AI
= Reasoning Component

Webhook
= Entry Point

request_id
= Correlation ID

Status
= Process State

Validation
= Quality Gate
```

---

# 36. الخلاصة

الاتصال بين Lovable وn8n ليس مجرد:

```text
Webhook URL
```

بل هو تصميم لنظام متكامل.

في PlanCraft AI:

```text
User
 ↓
Lovable
 ↓
Server-side Function
 ↓
n8n Webhook
 ↓
request_id
 ↓
Supabase
 ↓
Project Data
 ↓
AI Agent
 ↓
Structured Output
 ↓
Validation
 ↓
project_plans
 ↓
Status = completed
 ↓
Lovable
 ↓
Real Project Plan
```

والفكرة الأهم:

> **Lovable لا "يشغّل AI".**
>
> Lovable يبدأ العملية.
>
> **n8n لا "يخزن الحقيقة".**
>
> Supabase هو مصدر الحقيقة.
>
> **AI لا يدير النظام بالكامل.**
>
> AI هو جزء من الـ Workflow مسؤول عن reasoning and generation.
>
> **Webhook لا يعني أن العملية انتهت.**
>
> هو نقطة دخول للـ Workflow.
>
> **request_id ليس مجرد قيمة.**
>
> هو الرابط الذي يحافظ على هوية العملية عبر الأنظمة المختلفة.

---

# 37. السؤال النهائي الذي يجب أن تستطيع الطالبة الإجابة عنه

إذا سألك أحد:

> "كيف يتصل Lovable مع n8n في مشروعك؟"

يجب ألا تكون الإجابة:

> "عن طريق Webhook."

الإجابة الأقوى هي:

> "عندما يبدأ المستخدم عملية توليد الخطة من Lovable، يتم إرسال `request_id` من خلال Server-side Function إلى n8n Production Webhook باستخدام HTTP POST. يستخدم n8n هذا الـ ID للوصول إلى بيانات الطلب في Supabase، ثم يجهز البيانات ويرسلها إلى AI Agent لتوليد خطة منظمة. بعد التحقق من النتيجة، يحفظ n8n الخطة في `project_plans` ويحدث حالة الطلب إلى `completed`. بعدها يقرأ Lovable الحالة من Supabase ويعرض الخطة الحقيقية المحفوظة."

إذا استطعتِ شرح هذا التدفق وفهم **لماذا كل طبقة موجودة**، فأنتِ لا تتعلمين فقط كيف تربطين أدوات.

أنتِ تتعلمين كيف **تصممين Integration داخل نظام AI Automation حقيقي**.
