# react-native-supabase-twilio-sms
Sending SMS in React Native using Supabase Edge Functions &amp; Twilio
# 📩 react-native-supabase-twilio-sms

A secure SMS implementation in React Native using **Supabase Edge Functions** and **Twilio**.

---

## 🚀 Overview

This project demonstrates how to send SMS **securely** from a React Native app without exposing sensitive credentials.

### Architecture

```
React Native App
        ↓
Supabase Edge Function
        ↓
Twilio API
        ↓
SMS Delivered ✅
```

---

## 🔐 Why This Approach?

Twilio credentials must **never** be exposed in the frontend.

❌ Wrong:

* Storing Twilio keys in `.env` (EXPO_PUBLIC)

✅ Correct:

* Store secrets in Supabase
* Call via Edge Function

---

## ⚙️ Setup

### 1. Create Edge Function

```bash
supabase functions new send-sms
```

---

### 2. Add Twilio Secrets

```bash
supabase secrets set TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxx
supabase secrets set TWILIO_AUTH_TOKEN=xxxxxxxxxxxxxxxx
supabase secrets set TWILIO_FROM_NUMBER=+1xxxxxxxxxx
```

---

### 3. Deploy Function

```bash
supabase functions deploy send-sms
```

---

## 📡 Supabase Edge Function

📁 `supabase/functions/send-sms/index.ts`

```ts
// Supabase Edge Function: send-sms
// Deploy from your Supabase project (not from this mobile repo) with:
//   supabase functions deploy send-sms
//
// Required secrets (set once per project):
//   supabase secrets set TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxx
//   supabase secrets set TWILIO_AUTH_TOKEN=xxxxxxxxxxxxxxxx
//   supabase secrets set TWILIO_FROM_NUMBER=+1xxxxxxxxxx
//
// Request body:  { "to": "+1XXXXXXXXXX", "message": "Hello" }
// Response:      { "success": true, "sid": "SMxxxx" }  or  { error: "..." }

import { serve } from "https://deno.land/std@0.224.0/http/server.ts";

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers":
    "authorization, x-client-info, apikey, content-type",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
};

interface SendSmsPayload {
  to: string;
  message: string;
}

serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  try {
    const { to, message } = (await req.json()) as SendSmsPayload;

    if (!to || !message) {
      return new Response(
        JSON.stringify({ error: "Missing 'to' or 'message' in body" }),
        {
          status: 400,
          headers: { ...corsHeaders, "Content-Type": "application/json" },
        },
      );
    }

    const accountSid = Deno.env.get("TWILIO_ACCOUNT_SID");
    const authToken = Deno.env.get("TWILIO_AUTH_TOKEN");
    const fromNumber = Deno.env.get("TWILIO_FROM_NUMBER");

    if (!accountSid || !authToken || !fromNumber) {
      return new Response(
        JSON.stringify({ error: "Twilio credentials not configured" }),
        {
          status: 500,
          headers: { ...corsHeaders, "Content-Type": "application/json" },
        },
      );
    }

    const twilioUrl = `https://api.twilio.com/2010-04-01/Accounts/${accountSid}/Messages.json`;
    const basicAuth = btoa(`${accountSid}:${authToken}`);

    const form = new URLSearchParams();
    form.append("To", to);
    form.append("From", fromNumber);
    form.append("Body", message);

    const twilioRes = await fetch(twilioUrl, {
      method: "POST",
      headers: {
        Authorization: `Basic ${basicAuth}`,
        "Content-Type": "application/x-www-form-urlencoded",
      },
      body: form.toString(),
    });

    const twilioData = await twilioRes.json();

    if (!twilioRes.ok) {
      return new Response(
        JSON.stringify({
          error: twilioData.message ?? "Twilio request failed",
          code: twilioData.code,
        }),
        {
          status: twilioRes.status,
          headers: { ...corsHeaders, "Content-Type": "application/json" },
        },
      );
    }

    return new Response(
      JSON.stringify({ success: true, sid: twilioData.sid }),
      {
        status: 200,
        headers: { ...corsHeaders, "Content-Type": "application/json" },
      },
    );
  } catch (e) {
    return new Response(
      JSON.stringify({ error: (e as Error).message ?? "Unknown error" }),
      {
        status: 500,
        headers: { ...corsHeaders, "Content-Type": "application/json" },
      },
    );
  }
});
```

---

## 📱 React Native Integration

### SMS Service

📁 `src/services/sms/index.ts`

```ts
import { supabase } from "../../lib/supabase";

export const sendSms = async ({
  to,
  message,
}: {
  to: string;
  message: string;
}) => {
  const {
    data: { session },
  } = await supabase.auth.getSession();

  if (!session?.access_token) {
    throw new Error("User not authenticated");
  }

  console.log("Sending SMS to:", to, "with message:", message);
  console.log("annon key", process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY);
  const { data, error } = await supabase.functions.invoke("send-sms", {
    headers: {
      Authorization: `Bearer ${process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY}`,
    },
    body: { to, message },
  });

  if (error) {
    throw error;
  }
  return data;
};
```

---

### Usage Example

```ts
/* ================= SMS CUSTOMER ================= */
  const handleSmsCustomer = useCallback(async (phoneNumber: string) => {
    try {
      const res = await sendSms({
        to: phoneNumber,
        message: "Hi i am priyank, why you are not with me eat idlis? 😄",
      });
      console.log("SMS sent:", res);
    } catch (e) {
      console.log("Failed to send SMS:", e);
    }
  }, []);
```

---

## ⚠️ Common Issues

### ❌ Invalid JWT

Cause:
Manually passing Authorization header

Fix:

```ts
// REMOVE THIS
headers: {
  Authorization: `Bearer ${process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY}`,
}
```

Supabase automatically handles auth.

---

### ❌ Twilio Error 21211 (Invalid Number)

Cause:
Phone number not in E.164 format

✅ Correct:

```
+919725727670
```

---

### ❌ Trial Account Restriction (21608)

* Can only send SMS to verified numbers

👉 Verify in Twilio Console

---

## 🧠 Key Learnings

* Never expose secrets in frontend
* Always use Edge Functions for sensitive APIs
* Normalize phone numbers
* Let Supabase handle authentication

---

## 📦 Tech Stack

* React Native (Expo)
* Supabase Edge Functions
* Twilio API
* TypeScript

---

## 🚀 Future Improvements

* OTP authentication
* SMS templates
* Bulk messaging
* Delivery tracking

---

## 👨‍💻 Author

Built with ❤️ using React Native + Supabase
