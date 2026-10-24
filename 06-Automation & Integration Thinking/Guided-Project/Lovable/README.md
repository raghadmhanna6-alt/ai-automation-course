# PlanCraft AI — Build the Application

---

## 1. Project Overview

في هذا المشروع، ستبنين تطبيقًا يساعد المستخدم على تحويل فكرة مشروع أو وصف أولي إلى خطة عمل منظمة.

تسمح الواجهة للمستخدم بإدخال معلومات المشروع، وحفظ الطلب في قاعدة البيانات، ثم عرض خطة المشروع عند توفرها.

سنبدأ ببناء التطبيق وتجهيز قاعدة البيانات باستخدام:

- **Lovable:** لبناء واجهة التطبيق ومنطقه الأساسي.
- **Supabase:** لتخزين طلبات المشاريع والخطط الناتجة عنها.
- **n8n Cloud:** سنربطه لاحقًا لتنفيذ عملية توليد الخطط آليًا.

في هذه المرحلة، نركز على بناء تطبيق منظم وقابل للتكامل، وليس على تنفيذ الأتمتة كاملة.

### كيف سيعمل التطبيق؟

1. يفتح المستخدم التطبيق.
2. يدخل معلومات مشروعه.
3. يُحفظ طلب المشروع في Supabase.
4. يصبح الطلب جاهزًا للمعالجة الآلية.
5. بعد توليد الخطة وحفظها، يستطيع المستخدم عرضها داخل التطبيق.

> **مبدأ هندسي مهم:** نبني التطبيق وقاعدة البيانات أولًا، ثم نربطهما بنظام الأتمتة. فصل المسؤوليات يجعل النظام أسهل في التطوير والصيانة والتوسع.

---

## 2. System Architecture

```text
                  USER
                    |
                    v
             LOVABLE APPLICATION
                    |
          Submit Project Request
                    |
                    v
                SUPABASE
                    |
           Store Project Request
                    |
                    v
             n8n CLOUD (NEXT)
                    |
              Generate Plan
                    |
                    v
                SUPABASE
                    |
             Store Project Plan
                    |
                    v
             LOVABLE APPLICATION
                    |
                    v
             DISPLAY THE PLAN
```

في هذه المرحلة، سنبني الجزء الخاص بـ Lovable وSupabase، مع تجهيز التطبيق لاستقبال نتيجة الأتمتة لاحقًا.

---

## 3. Step 1 — Prepare Your Project

قبل البدء:

1. سجّلي الدخول إلى [Lovable](https://lovable.dev/).
2. أنشئي مشروعًا جديدًا، أو افتحي مشروعًا مخصصًا لهذا التطبيق.
3. إذا كان لديك مشروع سابق، تأكدي أنك تعملين في المشروع الصحيح.
4. تأكدي من إمكانية ربط المشروع بـ Supabase.

### Important

إذا كنتِ تستخدمين مشروعًا موجودًا بالفعل، فلا تنشئي مشروعًا جديدًا دون حاجة.

الـ Prompt التالي مصمم لبناء التطبيق وتجهيز تكامله مع Supabase، مع توجيه Lovable إلى مراجعة المشروع الحالي قبل إجراء تغييرات غير ضرورية.

---

## 4. Step 2 — Build the Application with Lovable

### Your Mission

استخدمي الـ Prompt التالي لبناء تطبيق PlanCraft AI.

**طريقة الاستخدام:**

1. انسخي الـ Prompt كاملًا.
2. افتحي محادثة المشروع داخل Lovable.
3. الصقي الـ Prompt.
4. اتركي Lovable ينفذ المطلوب.
5. راجعي النتيجة قبل الانتقال إلى الخطوة التالية.

### Copy This Prompt

```text
# Build PlanCraft AI — Complete Working Application

Build a complete, lightweight web application called PlanCraft AI using the EXISTING Supabase connection already configured in this Lovable project.

IMPORTANT: I have connected Supabase manually, but I have NOT created the database tables yet.

You must build the application, create the required database tables, configure their relationships and appropriate security policies, and connect the UI to the real database.

Do not just design the frontend or give me SQL instructions without implementing what you can through the available Supabase integration.

Do not create a new Supabase project or replace the existing connection.

==================================================
1. PRODUCT CONCEPT
==================================================

PlanCraft AI transforms a user's idea into a structured, actionable plan using AI automation.

Example:

Project title:
Launch a Handmade Accessories Store

Description:
I want to launch an online store selling handmade accessories. Help me organize the necessary steps, identify important requirements, and understand potential risks.

The intended workflow:

1. The user submits a project request.
2. The application saves it in Supabase.
3. The user clicks "Generate My Plan".
4. n8n Cloud receives the request through a Webhook.
5. An AI model analyzes the request and generates a structured plan.
6. The plan is saved in Supabase.
7. The application displays the generated plan.

For this implementation, build the complete frontend and database functionality and prepare a clean integration point for n8n.

The actual n8n workflow will be configured separately.

Never fabricate AI results or claim that AI generation works before the real integration is connected and tested.

==================================================
2. INSPECT THE EXISTING SUPABASE CONNECTION
==================================================

Before making changes:

- Verify that the existing Supabase integration is available.
- Inspect the current database schema.
- Check whether the required tables already exist.
- Reuse compatible tables if they exist.
- Do not create duplicate tables.
- Preserve the existing connection and configuration.
- Do not replace or recreate the Supabase project.

I have not intentionally created the required application tables yet, so create them if they do not already exist.

Use the available Supabase integration and supported database migration workflow to apply the required schema changes.

If manual approval or SQL execution is required, provide the exact migration and clearly identify the action I must approve or execute.

Do not report successful database creation unless the changes were actually applied and verified.

==================================================
3. CREATE THE DATABASE TABLES
==================================================

Create the following two tables in the connected Supabase project.

TABLE 1: project_requests

Purpose:
Store the project ideas submitted by users.

Fields:

- id: UUID, primary key, generated automatically.
- title: TEXT, required.
- description: TEXT, required.
- goal: TEXT, nullable.
- additional_context: TEXT, nullable.
- status: TEXT, NOT NULL, default 'pending'.
- created_at: TIMESTAMPTZ, default now().
- updated_at: TIMESTAMPTZ, default now().

Allowed status values:

- pending
- processing
- completed
- failed

Add appropriate constraints and indexes.

TABLE 2: project_plans

Purpose:
Store the structured plans generated by the future n8n AI workflow.

Fields:

- id: UUID, primary key, generated automatically.
- request_id: UUID, NOT NULL, referencing project_requests.id.
- summary: TEXT, NOT NULL.
- project_goal: TEXT, nullable.
- phases: JSONB, NOT NULL, default an empty JSON array.
- recommended_actions: JSONB, NOT NULL, default an empty JSON array.
- risks: JSONB, NOT NULL, default an empty JSON array.
- estimated_duration: TEXT, nullable.
- created_at: TIMESTAMPTZ, default now().
- updated_at: TIMESTAMPTZ, default now().

Add a UNIQUE constraint on project_plans.request_id so each request has at most one current plan.

Configure the foreign key so that deleting a request also deletes its associated plan.

Support updating an existing plan when a request is analyzed again.

Create an appropriate mechanism to update updated_at when a record changes.

Use appropriate database constraints and indexes.

After applying the schema, verify that both tables exist and their columns match the intended structure.

==================================================
4. SECURITY AND ACCESS
==================================================

This is an educational prototype without user authentication.

Do not create a fake authentication system.

Do not expose Supabase service-role keys or any privileged credentials in frontend code.

Do not disable Row Level Security merely to make the application work.

Do not create unrestricted public database policies that allow arbitrary users to read, modify, or delete all records.

Keep privileged operations server-side.

If anonymous access is needed for a public demonstration, propose and implement an appropriately restricted approach rather than silently opening the entire database.

If a secure policy or backend configuration cannot be completed automatically, explain exactly what remains to be configured.

Do not claim that records are private or user-owned because authentication is not implemented in this version.

Prepare the architecture so authentication and per-user ownership can be added later.

==================================================
5. APPLICATION INTERFACE
==================================================

Build a polished, responsive application with two main routes:

- / — Main workspace.
- /plans/:id — Project request and plan details.

Keep the application lightweight and easy to understand.

Use a modern, professional visual style suitable for an AI productivity application.

Application name:
PlanCraft AI

Tagline:
Turn your ideas into actionable plans.

The main workflow should be immediately clear:

Describe your idea → Generate a plan → Review the result.

==================================================
6. MAIN PAGE
==================================================

The main page must contain three sections.

A. HEADER

Display:

PlanCraft AI

Subtitle:
"Transform your ideas into structured plans and practical next steps."

Keep the header simple and responsive.

B. CREATE A PLAN FORM

Fields:

1. Project Title
   - Required text input.

2. Project Description
   - Required textarea.
   - Ask the user to explain the idea and what they want to achieve.

3. Main Goal
   - Optional text input.

4. Additional Context
   - Optional textarea.
   - The user can describe constraints, budget, audience, or other useful details.

Primary button:
"Generate My Plan"

The button must not pretend to generate an AI plan until n8n is connected.

For this initial implementation, save the request first and clearly explain that AI generation will become available after the automation integration is configured.

Validate required fields.

Show loading, success, and error states.

Prevent duplicate form submissions.

Save the request to the real project_requests table in Supabase.

After saving, display a confirmation and make the saved request available in the My Plans section.

C. MY PLANS

Fetch actual records from Supabase.

Display:

- Project title.
- Short description.
- Current status.
- Creation date.
- Link to open the request.

Include useful empty, loading, and error states.

Sort the newest requests first.

Do not insert fake data, sample records, or hardcoded statistics.

==================================================
7. PLAN DETAILS PAGE
==================================================

At /plans/:id, fetch the selected request from Supabase.

Display:

- Project title.
- Project description.
- Main goal.
- Additional context.
- Current status.
- Creation date.

If a real plan exists in project_plans, display its contents.

Otherwise, show an empty state explaining:

"This request has been saved. AI plan generation will be available once the n8n workflow is connected."

Do not invent or display a sample plan.

Include a "Back to My Plans" action.

Include a "Generate My Plan" button, but until the real backend integration is configured, clicking it must display an honest message that plan generation is not yet connected.

Do not send requests to an invented webhook URL.

==================================================
8. DISPLAY THE GENERATED PLAN
==================================================

Design the plan interface to display the following fields from the actual project_plans database record.

A. SUMMARY

Display the overall plan summary.

B. PROJECT GOAL

Explain the intended outcome.

C. EXECUTION PHASES

Render the phases array as an ordered list of cards.

Each phase may contain:

- title
- objective
- description
- actions
- priority

D. RECOMMENDED ACTIONS

Render recommended_actions as a clear list of practical next steps.

E. RISKS AND CONSIDERATIONS

Render risks as a readable list.

F. ESTIMATED DURATION

Display estimated_duration when available.

Handle missing optional values gracefully.

Never display empty sections as if they contain results.

Do not assume the AI response structure is valid without checking it.

==================================================
9. PREPARE THE N8N INTEGRATION
==================================================

The intended architecture is:

Lovable frontend
    ↓
Secure backend endpoint
    ↓
n8n Cloud Webhook
    ↓
AI model
    ↓
Structured plan
    ↓
Supabase
    ↓
Lovable displays the plan

Prepare a dedicated service or integration module for triggering plan generation.

Keep the integration logic separate from the UI.

The future endpoint should:

1. Receive the project request ID.
2. Validate the incoming data.
3. Verify that the project request exists and is eligible for analysis.
4. Send the necessary project data to n8n using POST.
5. Keep private credentials server-side.
6. Handle failures and timeouts.
7. Return a clear response to the frontend.

Use a Supabase Edge Function or another appropriate secure server-side endpoint for the actual n8n call.

Do not put a private n8n webhook URL, webhook secret, or Supabase service-role key in frontend code.

Do not invent endpoint URLs, secret values, or environment variable values.

Do not implement a fake AI response.

Do not claim the integration is operational until it has been configured and tested.

Because AI processing may take time, keep the design compatible with asynchronous processing and status polling if necessary.

Do not require the frontend to wait indefinitely for a long-running workflow.

==================================================
10. EXPECTED N8N WORKFLOW
==================================================

The workflow will be built separately in n8n Cloud.

It will follow this sequence:

1. Webhook — receive the project request.
2. Validate Input — check required fields.
3. Prepare Context — organize the submitted information.
4. AI Analysis — generate a structured project plan.
5. Validate Output — verify the generated structure.
6. Save Plan — insert or update project_plans in Supabase.
7. Update Request Status — set the request to completed or failed.
8. Respond — return an appropriate response when applicable.

Use POST for the real analysis request.

The generated plan must match the database structure.

The frontend must display the plan retrieved from Supabase, not a hardcoded result returned by a demonstration function.

Do not build the n8n workflow inside Lovable.

==================================================
11. IMPLEMENTATION QUALITY
==================================================

Use the existing project structure and dependencies where practical.

Requirements:

- Reusable components where appropriate.
- Clean and maintainable code.
- Responsive desktop and mobile layouts.
- Accessible form labels and controls.
- Clear status indicators.
- Proper error handling.
- No unnecessary dependencies.
- No unrelated pages or features.
- No fake charts or statistics.
- No unnecessary animations.
- No complex dashboard features.

Focus on a working educational integration project, not a large SaaS product.

==================================================
12. FINAL VERIFICATION
==================================================

After implementation, verify as much as possible.

Check:

- The application loads correctly.
- The form validates required fields.
- Requests can be inserted into Supabase.
- Saved requests can be retrieved from Supabase.
- The My Plans list displays real database records.
- The details page loads the correct request.
- A real saved plan can be displayed when one exists.
- Empty and error states work.
- Database constraints and relationships exist.
- Security policies are configured appropriately.
- No private credentials are exposed.
- No fake AI output is displayed.

Do not claim that an operation works unless it has actually been tested.

==================================================
13. FINAL REPORT
==================================================

When finished, provide a concise report containing:

1. Pages and components created.
2. Whether project_requests was created and verified.
3. Whether project_plans was created and verified.
4. The columns, constraints, and relationships configured.
5. The security policies configured.
6. Whether saving and retrieving requests were tested successfully.
7. What remains to connect n8n Cloud.
8. Any manual setup or unresolved errors.

Build the application now.

Do not respond with only a plan or tutorial.
Implement the application and database changes using the existing Supabase connection.
```

---

## 5. Step 3 — Understand the Database

بعد تنفيذ الـ Prompt، راجعي بنية قاعدة البيانات التي أنشأها Lovable أو أعاد استخدامها.

المطلوب وجود جدولين أساسيين.

### A. `project_requests`

هذا الجدول يمثل **الطلب الذي يقدمه المستخدم**.

| Field | Purpose |
|---|---|
| `id` | المعرّف الفريد للطلب |
| `title` | عنوان المشروع |
| `description` | وصف المشروع |
| `goal` | الهدف من المشروع |
| `additional_context` | معلومات إضافية اختيارية |
| `status` | حالة معالجة الطلب |
| `created_at` | تاريخ إنشاء الطلب |
| `updated_at` | تاريخ آخر تحديث |

### B. `project_plans`

هذا الجدول يمثل **الخطة التي ينتجها نظام الأتمتة**.

| Field | Purpose |
|---|---|
| `id` | المعرّف الفريد للخطة |
| `request_id` | الطلب المرتبط بالخطة |
| `summary` | ملخص الخطة |
| `project_goal` | الهدف الذي تعالجه الخطة |
| `phases` | مراحل المشروع بصيغة JSONB |
| `recommended_actions` | الإجراءات المقترحة |
| `risks` | المخاطر وطرق التعامل معها |
| `estimated_duration` | المدة الزمنية التقديرية |
| `created_at` | تاريخ إنشاء الخطة |
| `updated_at` | تاريخ آخر تحديث |

### Why Two Tables?

لأن الطلب والخطة يمثلان مرحلتين مختلفتين من دورة حياة البيانات.

قد يكون الطلب موجودًا دون أن تكون الخطة جاهزة.

فصل الجدولين يسمح للنظام بحفظ الطلب أولًا، ثم إضافة الخطة لاحقًا دون الحاجة إلى إعادة بناء الطلب أو تخزين بيانات الخطة في حقول غير مناسبة.

### Relationship

```text
project_requests
       |
       | id
       |
       v
project_plans
       |
       | request_id
       |
       v
The generated plan for that request
```

العلاقة المطلوبة: **كل طلب يمكن أن يمتلك خطة واحدة على الأكثر**.

ويُفرض ذلك باستخدام `UNIQUE` على `project_plans.request_id`.

---

## 6. Step 4 — Review the Application Structure

بعد انتهاء Lovable من التنفيذ، راجعي التطبيق من خلال هذه الخطوات الأساسية.

### 1. Submit a Project Request

افتحي صفحة إنشاء المشروع وأدخلي معلومات مشروع تجريبي من اختيارك.

تأكدي من أن النموذج يحتوي على:

- Project Title
- Project Description
- Project Goal
- Additional Context

أرسلي الطلب، ثم تأكدي من ظهور رسالة مناسبة بعد نجاح الحفظ.

### 2. Review My Projects

افتحي صفحة `My Projects`.

يجب أن يظهر الطلب الذي أدخلتِه، مع عنوانه وحالته وتاريخ إنشائه.

### 3. Open Project Details

افتحي تفاصيل الطلب.

يجب أن تظهر المعلومات التي أدخلتِها، وأن تعكس الصفحة حالة الطلب الحقيقية.

إذا لم تكن هناك خطة محفوظة بعد، فيجب أن توضح الواجهة ذلك بدلًا من عرض خطة وهمية.

**هذا يكفي للمراجعة الأساسية في هذه المرحلة.** لا حاجة إلى تكرار الاختبارات نفسها لكل صفحة أو تنفيذ اختبارات موسعة قبل الانتقال إلى مرحلة التكامل.

---

## 7. Step 5 — Review Supabase Security

قبل الانتقال إلى n8n، راجعي إعدادات الأمان في Supabase.

تحققي من:

- تفعيل `Row Level Security (RLS)` حيث يلزم.
- عدم وجود سياسات تسمح لأي شخص بقراءة البيانات أو تعديلها دون مبرر.
- عدم وضع `service_role key` في كود الواجهة.
- عدم وضع أسرار الـ Webhook داخل كود المتصفح.

### Important Security Note

هذا المشروع لا يُعد جاهزًا للنشر العام متعدد المستخدمين لمجرد أن التطبيق يعمل.

إذا لم يتم تطبيق المصادقة وربط كل سجل بصاحبه، فيجب التعامل مع ذلك باعتباره قيدًا أمنيًا قائمًا.

كذلك، يجب أن تتم عمليات الكتابة الحساسة أو الوصول الخاص عبر مسار خادمي آمن، وفقًا لتصميم الصلاحيات المستخدم.

لا تعالجي مشكلات الصلاحيات بإضافة سياسات عامة تسمح بالوصول غير المقيد إلى البيانات.

---

## 8. Expected Result

بعد إتمام هذه المرحلة، يجب أن يكون لديك تطبيق يحتوي على:

- واجهة لإنشاء طلب مشروع.
- نموذج مرتبط فعليًا بـ Supabase.
- صفحة تعرض طلبات المشاريع المحفوظة.
- صفحة تعرض تفاصيل كل طلب.
- بنية قاعدة بيانات منفصلة للطلبات والخطط.
- حالات واضحة للطلبات التي لم تُعالج بعد.
- بنية قابلة للربط مع n8n Cloud.
- فصل واضح بين واجهة المستخدم، وتخزين البيانات، وعملية توليد الخطط.

في هذه المرحلة، **لا يُشترط أن يكون توليد الخطط الآلي قد أصبح يعمل بعد**؛ فهذا هو الجزء الذي سننفذه في مرحلة التكامل.

---

## 9. Engineering Principles

### 1. Separate Responsibilities

لا ينبغي أن تتولى واجهة المستخدم جميع المسؤوليات.

- Lovable مسؤول عن الواجهة وتجربة الاستخدام.
- Supabase مسؤول عن تخزين البيانات واسترجاعها وفق الصلاحيات.
- n8n مسؤول عن تنفيذ سير العمل الآلي عند ربطه.

### 2. Design for Asynchronous Processing

توليد الخطة قد يستغرق وقتًا.

لذلك، يجب أن يستطيع التطبيق التعامل مع طلب محفوظ لم تكتمل معالجته بعد، بدلًا من افتراض أن كل عملية تنتهي فور إرسال النموذج.

### 3. Use Real Data

يجب أن تعرض الواجهة البيانات المحفوظة فعلًا.

لا تستبدلي النتائج الحقيقية ببيانات ثابتة لمجرد جعل التطبيق يبدو مكتملًا.

### 4. Treat Status as System State

الحالة ليست مجرد نص يظهر للمستخدم.

إنها تمثل مرحلة الطلب داخل النظام، ويجب أن تتوافق مع ما حدث فعلًا في قاعدة البيانات وسير العمل.

### 5. Prepare for Integration

التطبيق الجيد لا يحتاج إلى إعادة بنائه من الصفر عند إضافة أداة أتمتة جديدة.

عندما نفصل المسؤوليات ونحدد شكل البيانات مسبقًا، يصبح ربط n8n أكثر وضوحًا وأقل عرضة للأخطاء.

---

## 10. Before Moving to the Next Stage

راجعي النقاط التالية:

- [ ] تم بناء التطبيق باستخدام Lovable.
- [ ] تم ربط التطبيق بـ Supabase.
- [ ] يمكن حفظ طلب مشروع واسترجاعه.
- [ ] يمكن فتح تفاصيل الطلب.
- [ ] تتعامل الواجهة مع غياب الخطة بصورة صحيحة.
- [ ] تمت مراجعة الجداول والعلاقة بينها.
- [ ] تمت مراجعة إعدادات الأمان.
- [ ] لم تتم إضافة أسرار أو مفاتيح خاصة إلى كود الواجهة.
- [ ] أصبح التطبيق جاهزًا للانتقال إلى مرحلة التكامل.

---

## 11. Next Step — Connect the Application to n8n Cloud

في المرحلة التالية، سنربط التطبيق بسير عمل في n8n Cloud.

سيكون الهدف هو تحويل الطلب المحفوظ إلى خطة مشروع فعلية، ثم تخزين الخطة في Supabase وإظهارها داخل التطبيق.

سيصبح التدفق:

```text
Lovable
   |
   v
Supabase: Save Request
   |
   v
Protected Integration
   |
   v
n8n Cloud Workflow
   |
   v
Generate Structured Plan
   |
   v
Supabase: Save Plan
   |
   v
Update Request Status
   |
   v
Lovable: Display Plan
```

يجب تنفيذ التكامل من خلال مسار آمن، وعدم وضع بيانات اعتماد خاصة أو أسرار Webhook في كود المتصفح.

**الهدف النهائي:** بناء تطبيق يعمل مع نظام أتمتة حقيقي، وليس مجرد واجهة تعرض خطة ثابتة.

---

---

## 12. Step 6 — Complete the Real n8n Integration

هذه الخطوة تأتي **بعد** بناء التطبيق وربطه بـ Supabase، وبعد تجهيز الـ Workflow في n8n Cloud. لا تستخدمي هذا البرومبت بدلًا من برومبت بناء التطبيق؛ فهو مخصص لإكمال الاتصال داخل المشروع الموجود.

### متى تستخدمين هذا البرومبت؟

استخدميه عندما تكون الأمور التالية موجودة بالفعل:

- تطبيق PlanCraft AI وصفحاته الأساسية في Lovable.
- جداول Supabase الحالية، ومنها `project_requests` و`project_plans` أو ما يعادلها في المخطط الموجود.
- سير العمل المطلوب في n8n Cloud، مع إعداد الـ Production Webhook.

إذا لم يكن الـ Workflow جاهزًا بعد، جهّزيه أولًا. لا تفترضي أن لصق البرومبت وحده ينشئ أو يضبط إعدادات n8n الخارجية.

### طريقة الاستخدام

1. افتحي **مشروع PlanCraft AI نفسه** في Lovable، وليس مشروعًا جديدًا.
2. افتحي محادثة المشروع.
3. انسخي برومبت التكامل كاملًا من القسم التالي والصقيه في Lovable.
4. راجعي التغييرات المقترحة، خصوصًا وظائف الخادم وصلاحيات Supabase.
5. أضيفي `N8N_WEBHOOK_URL` إلى إعدادات الأسرار أو متغيرات البيئة **الخادمية** المناسبة. استخدمي Production Webhook URL الحقيقي من n8n؛ لا تضعيه في كود الواجهة الأمامية.
6. إذا كان التكامل يتطلب `N8N_WEBHOOK_SECRET`، أضيفيه فقط إذا كان هذا السر مدعومًا ومضبوطًا فعليًا في الطرفين.
7. اختبري المسار الكامل باستخدام طلب حقيقي، ولا تعتمدي على تقرير Lovable وحده لإثبات نجاح الاتصال.

### Copy This Prompt

انسخي البرومبت التالي كاملًا إلى محادثة مشروع PlanCraft AI في Lovable:

```text
# PlanCraft AI — Complete the Existing n8n Integration

Continue working on the EXISTING PlanCraft AI application.

The Supabase database, frontend pages, request flow, and n8n workflow already exist. Do NOT rebuild the application, database, tables, or UI.

## Goal

Complete the real connection between the existing PlanCraft AI application and the existing n8n Cloud workflow.

The desired flow is:

User clicks "Generate My Plan"
→ server-side generation action / Edge Function
→ n8n Production Webhook
→ n8n AI Agent generates the plan
→ n8n saves the plan into `project_plans`
→ n8n updates `project_requests.status`
→ frontend detects the completed request
→ frontend displays the real plan.

## n8n Configuration

Use the n8n Production Webhook URL as the server-side environment variable:

`N8N_WEBHOOK_URL`

Do NOT hardcode the webhook URL into frontend code.

Do NOT expose any webhook secret or Supabase service-role key to the browser.

If the existing backend supports `N8N_WEBHOOK_SECRET`, keep it server-side and send it to n8n using the existing expected header.

Do not invent a webhook URL or secret.

## Important

Inspect the current implementation first.

The application already has the generation flow prepared. Identify the exact existing server-side function/action responsible for starting generation and connect it to `N8N_WEBHOOK_URL`.

Do not create duplicate tables, duplicate functions, duplicate routes, or a second integration.

Keep the current database schema.

Use the existing `request_id` when calling n8n.

Send:

{
  "request_id": "<existing request id>"
}

to the n8n Production Webhook using POST.

## Frontend behavior

When the user clicks "Generate My Plan":

1. Use the existing request ID.
2. Call the existing server-side generation function.
3. Mark the request as `processing`.
4. Do not claim that a plan was generated merely because n8n accepted the request.
5. Continue checking the existing Supabase request status.
6. When status becomes `completed`, fetch the real `project_plans` row.
7. Display the actual saved plan.
8. If status becomes `failed`, show a clear failure message.
9. Prevent duplicate generation clicks while processing.

## Do NOT

- Do not put `N8N_WEBHOOK_URL` in frontend source code.
- Do not put the Supabase service-role key in frontend code.
- Do not create fake AI results.
- Do not simulate successful generation.
- Do not create unrestricted public database policies.
- Do not rebuild the existing application.
- Do not create a new Supabase project.
- Do not modify the existing n8n workflow unless required for the connection.

## Verification

After implementing the connection, report:

1. Exact file/function responsible for calling n8n.
2. Whether `N8N_WEBHOOK_URL` is configured server-side.
3. Exact request body sent to n8n.
4. Whether the frontend waits for the real database status.
5. Whether the real `project_plans` record is displayed.
6. Any remaining configuration I must provide manually.

Do not say the integration is complete unless it has actually been verified.
```

### ما الذي يجب أن يحدث بعد التكامل؟

```text
User clicks "Generate My Plan"
        |
        v
Existing server-side generation action
        |
        v
n8n Production Webhook
        |
        v
n8n generates the plan
        |
        v
Supabase: save the real plan in project_plans
        |
        v
Supabase: update project_requests.status
        |
        v
Frontend checks the real request status
        |
        v
Frontend fetches and displays the saved plan
```

### Verification Checklist

لا تعتبري التكامل مكتملًا إلا بعد التحقق من النقاط التالية:

- [ ] استُخدمت وظيفة الخادم الموجودة أو عُدّلت بدل إنشاء مسار مكرر.
- [ ] تم ضبط `N8N_WEBHOOK_URL` في بيئة خادمية آمنة.
- [ ] يرسل التطبيق `request_id` الموجود فعلًا إلى Production Webhook عبر `POST`.
- [ ] لا يظهر عنوان الـ Webhook أو سرّه أو مفتاح Supabase `service_role` في كود المتصفح.
- [ ] لا تتحول الحالة إلى `completed` لمجرد أن الـ Webhook قبل الطلب.
- [ ] يتابع التطبيق حالة الطلب الفعلية في Supabase.
- [ ] عند اكتمال المعالجة، يجلب التطبيق سجل الخطة الحقيقي من `project_plans` ويعرضه.
- [ ] عند الفشل، تظهر رسالة واضحة، ويُمنع تكرار الضغط على زر التوليد أثناء المعالجة.
- [ ] لم تُضف سياسات وصول عامة غير مقيّدة.
- [ ] يذكر تقرير التنفيذ الملفات والدوال التي عُدّلت، وما تم اختباره فعلًا، وأي إعدادات ما زالت مطلوبة.

> **ملاحظة هندسية:** إذا كان هناك نقص في إعدادات n8n أو الأسرار أو الصلاحيات، فقد ينفّذ Lovable تغييرات في الكود لكنه لا يستطيع إثبات نجاح التكامل الكامل من دون إعدادات صحيحة واختبار فعلي. وثّقي ما تم التحقق منه وما بقي، ولا تعلني نجاح التكامل قبل ظهور خطة حقيقية محفوظة في قاعدة البيانات.

