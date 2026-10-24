# PlanCraft AI — Build the n8n Automation

---

# 1. Project Objective

في هذا الجزء، سنبني سير عمل آليًا باستخدام n8n Cloud، ثم نربطه بتطبيق PlanCraft AI الذي بنيناه باستخدام Lovable وSupabase.

يسمح التطبيق للمستخدم بإدخال فكرة مشروع، ثم طلب توليد خطة منظمة باستخدام الذكاء الاصطناعي.

دور n8n هو تنفيذ عملية التوليد وربط مكوناتها معًا.

سيقوم الـ Workflow بالمهام التالية:

1. استقبال `request_id` من التطبيق.
2. استرجاع بيانات الطلب من Supabase.
3. تمرير بيانات المشروع إلى AI Agent.
4. توليد خطة منظمة وفقًا لبنية محددة.
5. حفظ الخطة في Supabase.
6. تحديث حالة الطلب إلى `completed`.

بعد ذلك، يكتشف التطبيق أن الطلب اكتمل، ويسترجع الخطة الحقيقية لعرضها للمستخدم.

## The Engineering Goal

لسنا بصدد بناء Chatbot يجيب عن سؤال فقط.

نحن نبني **Automation System** يستقبل بيانات حقيقية، وينفذ عدة عمليات مترابطة، ويتعامل مع حالات المعالجة، ويحفظ النتائج، ويتيح لتطبيق آخر استخدامها.

---

# 2. System Architecture

```text
USER
  |
  v
LOVABLE APPLICATION
  |
  | Click "Generate My Plan"
  v
SERVER-SIDE GENERATION FUNCTION
  |
  | Set request status to processing
  | POST { request_id }
  v
n8n CLOUD WEBHOOK
  |
  +--------------------------+
  |                          |
  v                          v
RESPOND TO WEBHOOK      GET PROJECT REQUEST
Return request accepted       |
                              v
                           AI AGENT
                              |
                              v
                     STRUCTURED OUTPUT
                              |
                              v
                      SAVE PROJECT PLAN
                              |
                              v
                    UPDATE REQUEST STATUS
                      status = completed
                              |
                              v
                       SUPABASE DATABASE
                              |
                              v
                     LOVABLE APPLICATION
                              |
                              v
                      DISPLAY REAL PLAN
```

لاحظي أن هناك مسارين بعد استقبال الطلب:

- **مسار الاستجابة:** يعيد استجابة أولية للتطبيق بأن الطلب استُقبل.
- **مسار المعالجة:** يستمر في جلب بيانات المشروع وتوليد الخطة وحفظها.

هذا تصميم غير متزامن (Asynchronous Processing).

استجابة Webhook الأولية لا تعني أن الخطة جاهزة.

---

# 3. Before You Start

تأكدي من توفر العناصر التالية:

- تطبيق PlanCraft AI مبني باستخدام Lovable.
- قاعدة بيانات Supabase موجودة.
- جدول `project_requests` موجود.
- جدول `project_plans` موجود.
- مشروع n8n Cloud جاهز.
- إمكانية الوصول إلى إعدادات Supabase.
- مزود نموذج لغوي يمكن استخدامه مع AI Agent في n8n.
- Server-side generation function في التطبيق، أو إمكانية تجهيزها ضمن مشروع Lovable.

## Important

هذا الدليل يشرح إعداد الـ Workflow نفسه وربطه بالتطبيق.

إذا كان لديك Workflow موجود بالفعل، راجعيه أولًا، وأكملي الإعدادات الناقصة بدلًا من إنشاء Workflow مكرر.

لا تنشئي قاعدة بيانات جديدة، ولا تعيدي بناء التطبيق.

---

# 4. Understand the Data Contract

قبل إضافة Nodes، يجب أن نفهم البيانات التي تدخل النظام والبيانات التي ستخرج منه.

## 4.1 Incoming Data

سيستقبل Webhook طلبًا يحتوي على `request_id` فقط.

مثال توضيحي:

```json
{
  "request_id": "EXISTING_PROJECT_REQUEST_UUID"
}
```

استبدلي القيمة التوضيحية بمعرّف طلب موجود فعلًا في جدول `project_requests`.

لا ترسلي عنوان المشروع ووصفه والهدف منه مرة أخرى من الواجهة إذا كان من الممكن استرجاعها من قاعدة البيانات باستخدام المعرّف.

### Why Send Only request_id?

لأن Supabase هي مصدر البيانات المعتمد.

عندما يستقبل n8n المعرّف، يستطيع استرجاع تفاصيل الطلب الأصلي.

هذا يقلل تكرار البيانات ويجعل النظام أوضح وأسهل في الصيانة.

## 4.2 Existing Supabase Tables

### `project_requests`

| Field | Purpose |
|---|---|
| `id` | Unique request ID |
| `title` | Project title |
| `description` | Project description |
| `goal` | Project goal |
| `additional_context` | Optional additional information |
| `status` | Request processing status |
| `created_at` | Request creation time |
| `updated_at` | Last update time |

### `project_plans`

| Field | Purpose |
|---|---|
| `id` | Unique plan ID |
| `request_id` | Associated request ID |
| `summary` | Project plan summary |
| `project_goal` | Goal addressed by the plan |
| `phases` | Structured project phases |
| `recommended_actions` | Recommended next actions |
| `risks` | Risks and mitigations |
| `estimated_duration` | Estimated project duration |
| `created_at` | Plan creation time |
| `updated_at` | Last update time |

لا تعيدي إنشاء هذه الجداول إذا كانت موجودة بالفعل.

---

# 5. Step 1 — Create the n8n Workflow

افتحي n8n Cloud.

1. أنشئي Workflow جديدًا، أو افتحي الـ Workflow الموجود.
2. أعطيه اسمًا واضحًا:

   `PlanCraft AI — Generate Project Plan`

3. أضيفي Nodes بالترتيب الموضح في هذا الدليل.
4. احفظي الـ Workflow باستمرار.

سنستخدم هذه المكونات:

| Node | Responsibility |
|---|---|
| Webhook | استقبال طلب التوليد |
| Edit Fields | استخراج `request_id` وتنظيم البيانات |
| Respond to Webhook | إرسال استجابة أولية |
| Supabase — Get Request | استرجاع بيانات المشروع |
| AI Agent | توليد خطة المشروع |
| Structured Output Parser | فرض بنية محددة للمخرجات |
| Supabase — Save Project Plan | حفظ الخطة |
| Supabase — Update Request Status | تحديث حالة الطلب |

قد تختلف أسماء بعض خيارات Nodes قليلًا بحسب إصدار n8n، لكن يجب أن تبقى مسؤولية كل خطوة واضحة.

---

# 6. Step 2 — Configure the Webhook

## Purpose

يستقبل Webhook طلبًا من التطبيق ويبدأ تنفيذ الـ Workflow.

في هذه المرحلة، لن يرسل التطبيق بيانات المشروع كاملة؛ سيرسل معرّف الطلب فقط.

## Configuration

أضيفي Node من نوع:

`Webhook`

اضبطي الإعدادات التالية:

| Setting | Value |
|---|---|
| HTTP Method | `POST` |
| Path | `plancraft-generate` |
| Respond | `Using Respond to Webhook Node` |

إذا كان الـ Path مستخدمًا في Workflow آخر داخل المشروع نفسه، اختاري مسارًا فريدًا.

### Production URL vs Test URL

يقدم n8n عادةً رابطين للـ Webhook:

- Test URL
- Production URL

أثناء تجربة الـ Workflow داخل المحرر، استخدمي Test URL بالطريقة التي يتطلبها n8n.

أما التطبيق الفعلي، فيجب أن يستخدم **Production URL** بعد تفعيل الـ Workflow والتحقق من عمله.

لا تضعي الرابط في كود الواجهة.

سنضعه في إعداد خادمي باسم:

`N8N_WEBHOOK_URL`

### Authentication

للتجربة الأولية، يمكن استخدام إعداد مؤقت دون مصادقة إذا كانت بيئة التدريب تسمح بذلك.

لكن لا تتركي Webhook مكشوفًا في نشر فعلي.

استخدمي آلية مصادقة مناسبة، مثل Header Authentication أو سر مشترك، مع حفظ السر في إعدادات الخادم وإرساله بالطريقة التي يتوقعها n8n.

إذا كان التطبيق يستخدم `N8N_WEBHOOK_SECRET` بالفعل، فحافظي على الاسم وطريقة الإرسال المتفق عليهما في التكامل.

لا تخترعي سرًا جديدًا داخل كود الواجهة.

---

# 7. Step 3 — Configure Edit Fields

## Purpose

توحيد البيانات الواردة من Webhook حتى تصبح سهلة الاستخدام في بقية الـ Workflow.

أضيفي Node:

`Edit Fields (Set)`

وصليها بمخرج Webhook.

## Configuration

أنشئي الحقل التالي:

| Field | Value |
|---|---|
| `request_id` | `{{ $json.body.request_id }}` |

اختاري وضع Expression للحقل.

### Why Do We Need This Node?

قد يحتوي خرج Webhook على بيانات الطلب داخل `body`، إلى جانب معلومات أخرى.

بدلًا من استخدام المسار الكامل في كل Node، نستخرج المعرّف ونضعه في حقل واضح.

بعد هذه الخطوة، تستطيع بقية الـ Workflow الوصول إلى المعرّف باستخدام:

```javascript
{{ $json.request_id }}
```

لكن إذا كانت Node لاحقة تستقبل خرج Node مختلفة، فقد تحتاج إلى الإشارة إلى اسم Node المصدر صراحةً.

مثلًا:

```javascript
{{ $('Edit Fields').item.json.request_id }}
```

استخدمي المرجع الذي يتوافق مع مسار البيانات الفعلي في الـ Workflow.

---

# 8. Step 4 — Add Respond to Webhook

## Purpose

إرسال استجابة أولية إلى التطبيق دون انتظار اكتمال توليد الخطة.

هذه الخطوة مهمة لأن AI Agent قد يستغرق وقتًا في تحليل الطلب وإنتاج الخطة.

## Connection

صلي `Respond to Webhook` بمخرج `Edit Fields` في مسار مستقل.

يصبح الشكل:

```text
Webhook
   |
   v
Edit Fields
   |
   +----------------------+
   |                      |
   v                      v
Respond to Webhook    Supabase Get Request
                           |
                           v
                        AI Agent
```

هذا التصميم يستخدم استجابة مبكرة، بينما يستمر مسار المعالجة في العمل.

**مهم:** تحققي عمليًا من أن إصدار n8n وإعدادات الـ Workflow يسمحان بإرسال الاستجابة المبكرة واستمرار تنفيذ المسار الآخر كما هو متوقع.

## Configuration

في Node `Respond to Webhook`، اختاري استجابة JSON.

مثال توضيحي:

```json
{
  "success": true,
  "message": "Project plan generation has started.",
  "request_id": "DYNAMIC_REQUEST_ID",
  "status": "processing"
}
```

يجب أن تكون قيمة `request_id` مرتبطة بالطلب الفعلي، وليست نصًا ثابتًا.

إذا كان المحرر يدعم Expression داخل جسم الاستجابة، استخدمي القيمة الديناميكية من `Edit Fields`، مثل:

```javascript
{{ $('Edit Fields').item.json.request_id }}
```

أو أنشئي جسم الاستجابة باستخدام Expression كامل يعيد JSON صحيحًا، وفقًا لوضع الإدخال في إصدار n8n لديك.

### What Does This Response Mean?

تعني الاستجابة:

- تم استقبال الطلب.
- بدأ مسار المعالجة أو أصبح جاهزًا للمتابعة.

ولا تعني:

- أن AI Agent أنهى العمل.
- أن الخطة حُفظت في Supabase.
- أن حالة الطلب أصبحت `completed`.

لهذا السبب، سيعتمد التطبيق على حالة الطلب الحقيقية في قاعدة البيانات.

---

# 9. Step 5 — Configure Supabase Credentials

قبل إضافة عمليات القراءة والكتابة، يجب أن يستطيع n8n الوصول إلى Supabase.

في n8n، افتحي قسم Credentials وأنشئي بيانات اعتماد مناسبة لعقدة Supabase.

اختاري نوع المصادقة الذي يدعمه إصدار n8n لديك وطريقة الاتصال المتاحة.

عادةً ستحتاجين إلى بيانات مثل:

- Supabase Project URL.
- API key أو بيانات اعتماد مدعومة من Node.

إذا احتجتِ إلى مفتاح ذي صلاحيات مرتفعة لتنفيذ عمليات الخادم، فيجب حفظه داخل Credentials أو إعدادات الخادم فقط.

لا تضعيه في:

- كود React.
- ملفات الواجهة العامة.
- GitHub repository.
- رسائل الاستجابة المرسلة إلى المتصفح.

ولا تفعّلي سياسات عامة غير مقيدة في Supabase لمجرد تجاوز مشكلة في الاتصال.

### Credential Verification

تأكدي من أن بيانات الاعتماد تستطيع تنفيذ العمليات المطلوبة:

- قراءة طلب من `project_requests`.
- إدراج خطة في `project_plans`.
- تحديث حالة الطلب.

إذا فشلت العملية، تحققي من بيانات الاعتماد والصلاحيات قبل الانتقال إلى بقية الخطوات.

---

# 10. Step 6 — Retrieve the Project Request

## Purpose

استرجاع البيانات الأصلية التي أدخلها المستخدم في التطبيق.

أضيفي Node من نوع:

`Supabase`

وسمّيها:

`Get Project Request`

صليها بمسار المعالجة الخارج من `Edit Fields`، وليس بمسار الاستجابة المبكرة.

## Configuration

اختاري عملية قراءة سجل أو سجلات وفقًا لواجهة Node المتاحة.

اضبطي الجدول:

`project_requests`

واستخدمي الفلتر التالي:

| Setting | Value |
|---|---|
| Column | `id` |
| Condition | `Equals` |
| Value | `{{ $('Edit Fields').item.json.request_id }}` |

قد تختلف أسماء خيارات الفلترة قليلًا بحسب إصدار Node.

الهدف هو استرجاع الطلب الذي يتطابق `id` الخاص به مع `request_id` الوارد من Webhook.

## Expected Data

يجب أن تتوفر في خرج هذه الخطوة بيانات الطلب الفعلي، مثل:

```json
{
  "id": "EXISTING_PROJECT_REQUEST_UUID",
  "title": "Community Learning Platform",
  "description": "A platform for organizing learning resources.",
  "goal": "Help learners organize and track their learning.",
  "additional_context": "Focus on a simple first version.",
  "status": "processing"
}
```

هذا مثال توضيحي لشكل البيانات، وليس سجلًا يجب إدخاله يدويًا.

### Important

لا تضعي بيانات المشروع ثابتة داخل AI Agent.

يجب أن يستخدم AI Agent بيانات الطلب التي جرى استرجاعها من Supabase.

إذا لم يوجد الطلب، فلا ينبغي أن يستمر النظام وكأنه يملك بيانات صحيحة.

في هذه الحالة، يجب التعامل مع الخطأ وإيقاف مسار التوليد.

---

# 11. Step 7 — Configure the AI Model

قبل استخدام AI Agent، اختاري مزود نموذج لغوي متاحًا لديك.

أضيفي إعدادات Credentials الخاصة بمزود النموذج في n8n، ثم استخدمي النموذج المتوافق مع احتياجات المشروع والميزانية المتاحة.

لا يتطلب هذا المشروع تدريب نموذج خاص.

المطلوب هو استخدام نموذج قادر على فهم وصف المشروع وتوليد خطة منظمة.

### Model Responsibilities

يجب أن يستطيع النموذج:

1. فهم وصف المشروع.
2. تحديد الهدف.
3. تقسيم العمل إلى مراحل منطقية.
4. اقتراح إجراءات عملية.
5. تحديد المخاطر وطرق تخفيفها.
6. تقديم تقدير زمني تقريبي.

لا تفترضي أن كل نموذج سينتج JSON صحيحًا بصورة ثابتة؛ سنستخدم `Structured Output Parser` لتحديد البنية المطلوبة والتحقق من شكل المخرجات.

---

# 12. Step 8 — Configure the AI Agent

أضيفي Node:

`AI Agent`

وسمّيها:

`Generate Project Plan`

صليها بعد `Get Project Request`.

أضيفي Chat Model إلى مدخل النموذج في AI Agent، وفقًا للواجهة المتاحة في إصدار n8n.

## 12.1 Agent Input

يجب أن يستخدم الـ Agent الحقول الفعلية من سجل Supabase:

- `title`
- `description`
- `goal`
- `additional_context`

مثال على إعداد User Message باستخدام Expression:

```text
Project Title:
{{ $('Get Project Request').item.json.title }}

Project Description:
{{ $('Get Project Request').item.json.description }}

Project Goal:
{{ $('Get Project Request').item.json.goal || 'Not specified' }}

Additional Context:
{{ $('Get Project Request').item.json.additional_context || 'Not specified' }}
```

إذا غيّرتِ اسم Node، عدّلي المرجع في الـ Expression ليتوافق مع الاسم الفعلي.

## 12.2 System Message

استخدمي الـ System Message التالي:

```text
You are PlanCraft AI, an expert project planning assistant.

Your task is to transform a user's project request into a realistic, structured, actionable project plan.

Analyze the project information carefully before generating the plan.

RULES:

1. Base the plan on the actual project information provided.
2. Do not invent facts about the user's organization, budget, team, or resources.
3. If important information is missing, make reasonable and clearly limited assumptions.
4. Keep the plan practical and relevant to the stated project goal.
5. Divide the work into 3 to 6 logical phases.
6. Give every phase a clear title, objective, description, actionable steps, and priority.
7. Use only High, Medium, or Low for phase priority.
8. Recommend concrete next actions.
9. Identify meaningful project risks and practical mitigation strategies.
10. Provide a reasonable estimated duration. State assumptions when the available information is insufficient for a precise estimate.
11. Avoid generic advice that does not help move the project forward.
12. Do not claim that external actions have been completed.
13. Do not claim that the plan has been saved to the database.
14. Produce output that follows the required structured output schema exactly.

The plan should be useful to someone who needs to understand what to do, in what order, and why.

Write the plan in clear, professional English.

Return the structured plan only through the required output format.
```

### Why Use a System Message?

لأنه يحدد الدور العام للـ Agent، ومعايير جودة الخطة، والقيود التي يجب أن يلتزم بها عند معالجة الطلبات المختلفة.

أما بيانات المشروع الفعلية، فتمر في رسالة المستخدم الديناميكية.

بهذا الشكل، نستطيع استخدام الـ Agent نفسه مع طلبات متعددة دون إعادة كتابة التعليمات العامة في كل مرة.

---

# 13. Step 9 — Add Structured Output Parser

## Purpose

نريد مخرجات يمكن للنظام التعامل معها بصورة متوقعة.

إذا أعاد النموذج نصًا طويلًا بتنسيق مختلف في كل مرة، ستصبح عملية حفظ النتائج في Supabase أكثر صعوبة.

لذلك، سنحدد شكل الخطة المطلوب.

أضيفي Node:

`Structured Output Parser`

وصليها بمدخل الـ Output Parser في AI Agent.

إذا كانت واجهة n8n لديك تتطلب Schema أو JSON Example، استخدمي البنية التالية كمرجع.

## Output Schema

```json
{
  "summary": "A concise project summary.",
  "project_goal": "The main goal of the project.",
  "phases": [
    {
      "title": "Phase title",
      "objective": "The objective of this phase.",
      "description": "A clear explanation of the work.",
      "actions": [
        "First actionable step",
        "Second actionable step"
      ],
      "priority": "High"
    }
  ],
  "recommended_actions": [
    "First recommended action",
    "Second recommended action"
  ],
  "risks": [
    {
      "risk": "A potential project risk.",
      "mitigation": "A practical mitigation strategy."
    }
  ],
  "estimated_duration": "An estimated duration with assumptions when needed."
}
```

هذه البنية توضيحية لتحديد شكل البيانات.

يجب أن ينتج الـ Agent ما يلي:

- `summary`: نص.
- `project_goal`: نص.
- `phases`: مصفوفة من 3 إلى 6 مراحل.
- `recommended_actions`: مصفوفة نصوص.
- `risks`: مصفوفة من المخاطر وطرق تخفيفها.
- `estimated_duration`: نص.

وكل مرحلة يجب أن تحتوي على:

- `title`
- `objective`
- `description`
- `actions`
- `priority`

يجب أن تكون `priority` إحدى القيم التالية:

- `High`
- `Medium`
- `Low`

### Important

الـ Structured Output Parser يساعد على ضبط بنية المخرجات، لكنه لا يضمن أن الخطة منطقية أو صحيحة من الناحية الواقعية.

ما زلنا بحاجة إلى تقييم جودة المحتوى، والتعامل مع حالات فشل النموذج، والتحقق من البيانات قبل حفظها.

---

# 14. Step 10 — Save the Project Plan in Supabase

## Purpose

بعد توليد الخطة، يجب حفظها في قاعدة البيانات حتى يستطيع التطبيق استرجاعها لاحقًا.

أضيفي Node:

`Supabase`

وسمّيها:

`Save Project Plan`

صليها بعد `AI Agent`.

استخدمي عملية إدراج سجل في جدول:

`project_plans`

## Field Mapping

| Supabase Field | Value |
|---|---|
| `request_id` | `{{ $('Edit Fields').item.json.request_id }}` |
| `summary` | قيمة `summary` من خرج AI Agent |
| `project_goal` | قيمة `project_goal` من خرج AI Agent |
| `phases` | مصفوفة `phases` من خرج AI Agent |
| `recommended_actions` | مصفوفة `recommended_actions` من خرج AI Agent |
| `risks` | مصفوفة `risks` من خرج AI Agent |
| `estimated_duration` | قيمة `estimated_duration` من خرج AI Agent |

### JSONB Fields

الحقول التالية من نوع JSONB:

- `phases`
- `recommended_actions`
- `risks`

يجب حفظها بصفتها بيانات JSON فعلية، وليس تحويلها إلى نص غير مناسب لبنية قاعدة البيانات.

تأكدي من أن Expressions المستخدمة تعيد القيم الصحيحة من خرج الـ Agent، وأن Node Supabase تقبل أنواع البيانات الناتجة.

قد تختلف مسارات الوصول إلى الحقول باختلاف شكل خرج AI Agent وإعدادات Parser.

إذا كان خرج الـ Agent يحتوي على كائن الخطة داخل حقل فرعي، استخدمي المسار الفعلي الذي يظهر في بيانات التنفيذ.

## Important: Duplicate Plans

قد يحتوي الجدول على قيد فريد يمنع وجود أكثر من خطة واحدة لكل `request_id`.

هذا جيد للحفاظ على العلاقة بين الطلب والخطة، لكنه يعني أن إعادة تنفيذ الطلب نفسه قد تؤدي إلى خطأ تكرار.

لذلك:

- استخدمي طلبًا جديدًا عند إجراء تجربة مستقلة.
- لا تعيدي تشغيل التوليد عشوائيًا لطلب لديه خطة محفوظة.
- في النظام المتقدم، تحققي من وجود خطة سابقة قبل الإدراج، أو صممي آلية آمنة لإعادة المحاولة وتحديث السجل الموجود.

لا تزيلي القيد الفريد لمجرد التخلص من خطأ التكرار.

---

# 15. Step 11 — Update the Request Status

## Purpose

بعد نجاح حفظ الخطة، نحدّث حالة الطلب.

أضيفي Node:

`Supabase`

وسمّيها:

`Update Request Status`

صليها بعد `Save Project Plan`.

اختاري عملية تحديث سجل في:

`project_requests`

## Configuration

استخدمي الفلتر:

| Setting | Value |
|---|---|
| Column | `id` |
| Condition | `Equals` |
| Value | `{{ $('Edit Fields').item.json.request_id }}` |

ثم حدّثي الحقل التالي:

| Field | Value |
|---|---|
| `status` | `completed` |

إذا كان جدولك يحتوي على `updated_at` ويعتمد على تحديثه يدويًا، حدّثيه أيضًا. أما إذا كان هناك Trigger مسؤول عن ذلك، فاتركيه يعمل وفق تصميم قاعدة البيانات.

### Critical Rule

لا تحدّثي الحالة إلى `completed` قبل نجاح حفظ الخطة.

الترتيب الصحيح:

```text
AI Agent
   |
   v
Save Project Plan
   |
   v
Update Request Status
   |
   v
completed
```

إذا فشل حفظ الخطة، فلا ينبغي أن تظهر حالة الطلب على أنها مكتملة.

بهذا تصبح `completed` إشارة موثوقة يمكن للتطبيق الاعتماد عليها.

---

# 16. Step 12 — Handle Processing Failures

الـ Workflow الحقيقي يجب أن يتعامل مع الفشل، وليس النجاح فقط.

قد يحدث الفشل بسبب:

- مشكلة في الاتصال بمزود النموذج.
- فشل AI Agent في إنتاج المخرجات المطلوبة.
- خطأ في Structured Output.
- فشل إدراج الخطة في Supabase.
- خطأ في بيانات الاعتماد.
- عدم وجود الطلب المطلوب.

## Expected Status Behavior

| Situation | Expected Status |
|---|---|
| Request saved, not started | `pending` |
| Generation started | `processing` |
| Plan saved successfully | `completed` |
| Generation failed | `failed` |

يجب أن تكون حالة `processing` قد ضُبطت قبل بدء المعالجة، إما من خلال Server-side generation function في التطبيق، أو عبر خطوة موثوقة في بداية الـ Workflow قبل بدء العمل الطويل.

لا تعتمدي على أن حالة الطلب ستتغير تلقائيًا لمجرد استدعاء Webhook.

## Failure Handling Strategy

في نسخة التدريب، افهمي أولًا كيف ينتقل الخطأ من Node إلى Node، ثم أضيفي معالجة مناسبة للأخطاء.

يمكن استخدام إعدادات `On Error` المتاحة في Nodes المعنية، أو مسار منفصل لمعالجة الفشل، بحسب تصميم الـ Workflow.

عند فشل التوليد:

1. احتفظي بمعرّف الطلب الأصلي.
2. حدّثي الطلب إلى `failed`.
3. اعرضي رسالة خطأ مناسبة في التطبيق.
4. لا تحفظي خطة ناقصة وكأنها خطة مكتملة.
5. لا تحدّثي الحالة إلى `completed`.

يمكن الاحتفاظ بتفاصيل الخطأ في سجل تنفيذ n8n. وإذا أردتِ تخزينها في قاعدة البيانات، فتحققي أولًا من وجود حقل مناسب في المخطط الحالي؛ لا تفترضي وجود `error_message` دون مراجعة الجدول.

### Production Consideration

إذا كان تحديث الحالة إلى `failed` يتم داخل مسار معالجة الأخطاء، تأكدي من أنه يستطيع الوصول إلى `request_id` حتى لو فشلت Node لاحقة.

كما يجب التمييز بين فشل إرسال الطلب إلى n8n وفشل توليد الخطة بعد قبوله؛ فهما حالتان مختلفتان في دورة حياة النظام.

---

# 17. Step 13 — Understand the Complete Workflow

يجب أن يكون ترتيب التنفيذ المنطقي كما يلي:

```text
Webhook
   |
   v
Edit Fields
   |
   +--------------------------------+
   |                                |
   v                                v
Respond to Webhook          Get Project Request
                                    |
                                    v
                                 AI Agent
                                    |
                                    v
                          Structured Output Parser
                                    |
                                    v
                           Save Project Plan
                                    |
                                    v
                         Update Request Status
                                    |
                                    v
                                completed
```

قد يظهر `Structured Output Parser` كمدخل متصل بـ AI Agent بدلًا من Node عادية بعده؛ اتبعي طريقة التوصيل التي يدعمها إصدار n8n لديك.

لا تعني الرسمة أن Parser يجب أن يُنفذ دائمًا كخطوة مستقلة بعد خرج Agent؛ بل توضح دوره في إنتاج البنية المطلوبة.

## Why This Architecture?

لأن كل جزء له مسؤولية محددة:

| Component | Responsibility |
|---|---|
| Webhook | استقبال الطلب |
| Edit Fields | تنظيم البيانات |
| Respond to Webhook | إرسال الاستجابة الأولية |
| Get Project Request | استرجاع البيانات الأصلية |
| AI Agent | تحليل المشروع وتوليد الخطة |
| Structured Output Parser | ضبط بنية المخرجات |
| Save Project Plan | حفظ الخطة |
| Update Request Status | تسجيل نجاح المعالجة |

هذا الفصل يجعل النظام أسهل في الفهم والتشخيص والتطوير.

---

# 18. Step 14 — Connect Lovable to n8n Cloud

بناء الـ Workflow وحده لا يكفي؛ يجب أن يعرف التطبيق كيف يبدأ تشغيله وكيف يتابع النتيجة.

يجب أن يتم استدعاء n8n من خلال Server-side Function أو مسار خادمي آمن، وليس من كود المتصفح مباشرة.

## 18.1 Configure the Server-side Environment

يحتاج المسار الخادمي إلى متغير البيئة:

`N8N_WEBHOOK_URL`

ويجب أن يحتوي على Production URL الخاص بالـ Webhook.

لا تضعي قيمة وهمية، ولا تضعي رابط Webhook في ملفات الواجهة.

إذا كان المشروع يستخدم `N8N_WEBHOOK_SECRET`، فاحتفظي به خادميًا وأرسليه في الـ Header الذي يتوقعه إعداد n8n الحالي.

لا تنشئي متغيرًا أو آلية مصادقة مختلفة دون حاجة.

## 18.2 Call the Existing Generation Function

عندما يضغط المستخدم:

`Generate My Plan`

ينبغي أن يستدعي التطبيق دالة التوليد الموجودة أصلًا.

قبل تنفيذ أي تغييرات، راجعي المشروع وحددي:

- الملف الذي يستدعي التوليد.
- الدالة المسؤولة عن الاتصال بـ n8n.
- مكان قراءة `N8N_WEBHOOK_URL`.
- آلية تحديث حالة الطلب.
- آلية متابعة حالة الطلب واسترجاع الخطة.

لا تنشئي دالة ثانية تؤدي الغرض نفسه إذا كانت هناك دالة قائمة يمكن إصلاحها.

## 18.3 Request Body

يجب إرسال:

```json
{
  "request_id": "EXISTING_PROJECT_REQUEST_UUID"
}
```

باستخدام HTTP `POST`.

استبدلي المعرّف التوضيحي بمعرّف الطلب الحقيقي الموجود في قاعدة البيانات.

## 18.4 Frontend Status Handling

بعد بدء التوليد:

1. عطّلي زر التوليد لمنع النقرات المتكررة أثناء المعالجة.
2. اعرضي حالة `processing`.
3. تابعي حالة الطلب في Supabase بالطريقة التي يعتمدها التطبيق.
4. عندما تصبح الحالة `completed`، استرجعي الخطة الحقيقية من `project_plans`.
5. اعرضي الخطة للمستخدم.
6. إذا أصبحت الحالة `failed`، اعرضي رسالة مناسبة.
7. أعيدي تمكين الإجراءات المناسبة بعد انتهاء المعالجة.

يمكن استخدام Polling أو آلية تحديث مناسبة أخرى.

إذا استخدمتِ Polling، فضعي فترة انتظار وحدًا مناسبًا للمدة، وتعاملِي مع انتهاء الانتظار دون إظهار نجاح غير حقيقي.

لا يكفي أن يستقبل التطبيق استجابة `success: true` من Webhook؛ يجب أن يعتمد عرض الخطة على البيانات المحفوظة فعلًا.

---

# 19. Step 15 — Test the Complete Integration

بعد الانتهاء من الإعداد، نفذي تجربة متكاملة واحدة بطلب جديد.

لا حاجة إلى تكرار عشرات الاختبارات المتشابهة؛ المهم أن تتحققي من دورة حياة طلب حقيقية من البداية إلى النهاية.

## Test Scenario

### 1. Create a New Request

افتحي PlanCraft AI وأدخلي مشروعًا تجريبيًا جديدًا.

### 2. Start Generation

اضغطي:

`Generate My Plan`

### 3. Observe the Workflow

افتحي Executions في n8n.

تحققي من أن التنفيذ استقبل `request_id` الصحيح، وأنه استطاع استرجاع بيانات المشروع.

### 4. Confirm Database Results

بعد اكتمال التنفيذ، راجعي Supabase.

المطلوب:

- وجود خطة في `project_plans`.
- ارتباط الخطة بالطلب الصحيح من خلال `request_id`.
- تحوّل حالة الطلب إلى `completed`.

### 5. Confirm the Application Result

ارجعي إلى التطبيق.

يجب أن تظهر الخطة المحفوظة فعلًا في صفحة تفاصيل المشروع.

إذا فشلت أي خطوة، استخدمي سجل التنفيذ لتحديد الـ Node التي توقفت أو أعادت خطأ.

لا تعيدي تشغيل الطلب نفسه دون مراجعة احتمال وجود خطة محفوظة بالفعل.

---

# 20. Troubleshooting

## Problem 1 — The Webhook Is Not Triggered

راجعي:

- هل يستخدم التطبيق Production URL الصحيح؟
- هل الـ Workflow مفعّل؟
- هل عنوان Webhook مضبوط؟
- هل الطلب يستخدم `POST`؟
- هل إعداد المصادقة متوافق بين التطبيق وn8n؟

## Problem 2 — The Request Cannot Be Found

راجعي:

- هل أرسل التطبيق `request_id` صحيحًا؟
- هل وصل المعرّف إلى `Edit Fields`؟
- هل فلتر Supabase يستخدم عمود `id`؟
- هل بيانات اعتماد Supabase تشير إلى المشروع الصحيح؟

## Problem 3 — The AI Agent Does Not Return the Required Structure

راجعي:

- هل Chat Model متصل بالـ AI Agent؟
- هل تعليمات System Message واضحة؟
- هل Structured Output Parser مضبوط؟
- هل خرج النموذج يطابق أنواع البيانات المطلوبة؟

لا تعالجي المشكلة بتحويل كل المخرجات إلى نص ثم حفظها في حقول JSONB دون فهم البنية.

## Problem 4 — Saving the Plan Fails

راجعي:

- هل أسماء الحقول مطابقة لجدول `project_plans`؟
- هل أنواع البيانات صحيحة؟
- هل حقول JSONB تصل في شكل مناسب؟
- هل `request_id` يشير إلى طلب موجود؟
- هل توجد خطة سابقة لهذا الطلب؟
- هل تسمح بيانات الاعتماد بعملية الإدراج؟

## Problem 5 — The Plan Is Saved but the Status Is Not Completed

راجعي Node `Update Request Status`.

تأكدي من:

- استخدام `project_requests`.
- تحديث السجل الذي يطابق `request_id`.
- ضبط `status` على `completed`.
- وجود الصلاحيات اللازمة للتحديث.
- عدم توقف التنفيذ بسبب خطأ في Node السابقة.

## Problem 6 — n8n Returns Success but the App Shows No Plan

راجعي:

- هل حفظت الخطة في Supabase فعلًا؟
- هل حالة الطلب أصبحت `completed`؟
- هل التطبيق يتابع السجل الصحيح؟
- هل استعلام الخطة يستخدم `request_id` الصحيح؟
- هل تعرض الواجهة بيانات قاعدة البيانات بدلًا من الاعتماد على استجابة Webhook؟

تذكري أن الاستجابة الأولية من Webhook لا تعني أن الخطة اكتملت.

---

# 21. Security and Reliability

قبل اعتبار المشروع جاهزًا للاستخدام الفعلي، راجعي النقاط التالية.

## Secrets

- احتفظي بـ `N8N_WEBHOOK_URL` في الإعدادات الخادمية.
- احتفظي بأسرار Webhook في الخادم.
- احتفظي ببيانات اعتماد Supabase في Credentials المناسبة.
- لا تضعي `service_role key` في الواجهة.
- لا ترفعي الأسرار إلى GitHub.

## Database Access

- حافظي على Row Level Security حيث يلزم.
- لا تستخدمي سياسات عامة غير مقيدة لتجاوز مشكلات الصلاحيات.
- تأكدي من أن المستخدمين لا يستطيعون الوصول إلى بيانات غير مصرح لهم بها.

## Duplicate Execution

- امنعي النقر المتكرر أثناء المعالجة.
- تحققي من وجود خطة سابقة قبل إعادة توليد الطلب.
- لا تزيلي القيد الفريد على `project_plans.request_id` لمجرد تجاوز أخطاء التكرار.

## Status Integrity

- `processing` تعني أن المعالجة بدأت.
- `completed` تعني أن الخطة حُفظت بنجاح.
- `failed` تعني أن المعالجة فشلت.
- لا تعرضي نتيجة وهمية عند الفشل.
- لا تعتبري قبول Webhook دليلًا على اكتمال الخطة.

## Production Readiness

نجاح تجربة واحدة يثبت أن المسار الأساسي يعمل في تلك التجربة، لكنه لا يثبت أن النظام جاهز لجميع ظروف الاستخدام.

قبل النشر العام، يجب أيضًا مراعاة الصلاحيات، وإعادة المحاولة، وتكرار الطلبات، وحدود الانتظار، وتسجيل الأخطاء، وسلوك النظام عند فشل كل خطوة.

---

# 22. Final Deliverable

عند الانتهاء، يجب أن يكون لديك:

- [ ] Webhook يستقبل `request_id`.
- [ ] Edit Fields يستخرج المعرّف الصحيح.
- [ ] Respond to Webhook يعيد استجابة أولية مناسبة.
- [ ] Supabase يسترجع طلب المشروع الفعلي.
- [ ] AI Agent يستخدم بيانات المشروع الحقيقية.
- [ ] Structured Output Parser يحدد بنية الخطة.
- [ ] Supabase يحفظ الخطة في `project_plans`.
- [ ] Supabase يحدّث حالة الطلب إلى `completed` بعد نجاح الحفظ.
- [ ] التطبيق يستخدم Production Webhook من إعداد خادمي.
- [ ] التطبيق يتابع حالة الطلب الفعلية.
- [ ] التطبيق يعرض الخطة المحفوظة في قاعدة البيانات.
- [ ] تم التحقق من دورة حياة طلب حقيقي.
- [ ] تمت مراجعة الأسرار والصلاحيات ومعالجة الأخطاء.

---

# 23. Final Engineering Lesson

بناء الأتمتة ليس مجرد توصيل Nodes ببعضها.

المهندس يحتاج إلى فهم:

1. **Input:** ما البيانات التي تدخل النظام؟
2. **Data Retrieval:** من أين نحصل على البيانات المعتمدة؟
3. **Processing:** كيف تُحلل البيانات وتُعالج؟
4. **Structured Output:** كيف نضمن أن النتيجة قابلة للاستخدام؟
5. **Persistence:** أين تُحفظ النتيجة؟
6. **State Management:** كيف نعرف أن العملية بدأت أو اكتملت أو فشلت؟
7. **Integration:** كيف يتواصل التطبيق مع الـ Workflow؟
8. **Reliability:** ماذا يحدث عندما تفشل إحدى الخطوات؟
9. **Security:** كيف نحمي البيانات والأسرار؟
10. **Verification:** كيف نثبت أن النظام يعمل فعلًا؟

عندما تفهمين هذه المسؤوليات، يصبح n8n أكثر من مجرد أداة لتوصيل الخدمات؛ يصبح وسيلة لبناء أنظمة مترابطة يمكن اختبارها وصيانتها وتطويرها.

**The Goal: Build an automation system that works with real data, produces structured results, and integrates reliably with a real application.**

