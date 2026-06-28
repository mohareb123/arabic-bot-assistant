# خطة المشروع: بوت تيليجرام ذكي + لوحة تحكم

## نظرة عامة
سننقل البوت من Python إلى TypeScript عبر **TanStack Start** (نفس المشروع الحالي) مع لوحة تحكم ويب موحدة. البوت لن يعمل كعملية Python منفصلة بل عبر **Telegram Webhook** يستقبله خادم TanStack.

> ⚠️ **ملاحظة هامة**: تحميل ملفات تيليجرام الكبيرة (>20MB) من القنوات الخاصة يتطلب **MTProto** (مثل Pyrogram/GramJS) وليس Bot API. سنستخدم **GramJS** (مكتبة TypeScript لـ MTProto). الملفات حتى 20MB يمكن تحميلها عبر Bot API العادي.

## البنية التقنية

```text
┌──────────────────────────────────────────────┐
│  Telegram → Webhook → /api/public/telegram   │
│                          ↓                    │
│  ┌──────────────┐   ┌─────────────────┐     │
│  │  AI Agent    │ → │  Tool Calling   │     │
│  │  (Gemini)    │   │  - web_search   │     │
│  └──────────────┘   │  - browse_url   │     │
│         ↓            │  - download_tg  │     │
│  Lovable Cloud DB    └─────────────────┘     │
│         ↑                                     │
│  ┌──────────────────────────────────┐        │
│  │  Web Dashboard (TanStack/React)  │        │
│  │  - محادثات  - تحميلات  - إعدادات │        │
│  └──────────────────────────────────┘        │
└──────────────────────────────────────────────┘
```

## المراحل

### المرحلة 1: البنية التحتية (هذه الدفعة)
1. **تفعيل Lovable Cloud** (قاعدة بيانات + auth)
2. **ربط Telegram connector** (لإرسال الرسائل عبر Bot API)
3. **جداول قاعدة البيانات**:
   - `telegram_users` — المستخدمون المسموح لهم + الأدوار (admin/user)
   - `conversations` — محادثات البوت مع المستخدمين
   - `messages` — رسائل المحادثة (user/assistant/tool)
   - `downloads` — سجل التحميلات (file_id, status, path, size)
   - `bot_settings` — إعدادات (allowed_chats, download_path, etc.)
4. **Webhook endpoint**: `/api/public/telegram/webhook` يستقبل التحديثات

### المرحلة 2: عميل AI ذكي
1. **AI SDK** مع `google/gemini-3-flash-preview` (افتراضي)
2. **أدوات (Tools) للوكيل**:
   - `web_search(query)` → Firecrawl search
   - `browse_url(url)` → Firecrawl scrape (markdown)
   - `download_telegram_media(message_link)` → حفظ في DB
   - `send_message(chat_id, text)` → عبر Telegram connector
3. **ذاكرة المحادثة**: استرجاع آخر 20 رسالة لكل chat من DB
4. **System prompt** بالعربية يوضح القدرات والشخصية

### المرحلة 3: لوحة تحكم ويب
1. **مصادقة**: Lovable Cloud auth (المطور `6570434162` = admin تلقائياً)
2. **صفحات**:
   - `/` لوحة معلومات (إحصائيات: محادثات، تحميلات، استخدام AI)
   - `/conversations` قائمة المحادثات + عرض الرسائل
   - `/downloads` سجل التحميلات مع روابط
   - `/users` إدارة المستخدمين المسموح لهم
   - `/settings` إعدادات البوت (token، system prompt، نموذج AI)
3. **Realtime**: تحديث مباشر عبر Supabase realtime عند رسائل جديدة

### المرحلة 4: تحميل الوسائط
1. ملفات صغيرة (<20MB): Telegram Bot API `getFile` + تنزيل
2. ملفات كبيرة من قنوات خاصة: تتطلب جلسة MTProto منفصلة — سنوفر تعليمات للمستخدم لتشغيل خدمة GramJS مساعدة على VPS، أو نستخدم نفس Python session الحالي كـ microservice

## الأسرار المطلوبة
- `TELEGRAM_BOT_TOKEN` = `8596155951:AAFJsYWwqtTDo5HNMUS_k4pQVKaOR9L7uTA` (موجود)
- `LOVABLE_API_KEY` (تلقائي لـ AI)
- `FIRECRAWL_API_KEY` (عبر connector)
- `DEVELOPER_TELEGRAM_ID` = `6570434162` (في الكود)

## تفاصيل تقنية

**Stack**: TanStack Start + React 19 + Tailwind v4 + shadcn + Supabase (Lovable Cloud) + AI SDK + Telegram/Firecrawl connectors.

**Webhook security**: `X-Telegram-Bot-Api-Secret-Token` مشتق من `TELEGRAM_API_KEY` (SHA256 base64url).

**Allowed users check**: في webhook قبل معالجة أي رسالة، نتحقق من `telegram_users.is_allowed = true` أو `tg_id = DEVELOPER_ID`.

**RLS**: جميع الجداول مفعّلة + سياسات تربط البيانات بـ `auth.uid()` للمستخدمين في لوحة التحكم.

---

## نطاق هذه الدفعة (Phase 1)
سأبدأ بـ:
1. تفعيل Lovable Cloud
2. إنشاء جداول DB + RLS
3. webhook endpoint مع تحقق التوقيع
4. عميل AI أساسي مع رد على الرسائل (بدون أدوات بعد)
5. صفحة dashboard بسيطة تعرض المحادثات

ثم في الدفعات التالية: الأدوات، التحميلات، الواجهة الكاملة.

هل توافق على هذه الخطة؟