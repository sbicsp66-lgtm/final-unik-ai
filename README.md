# Unick AI — One-Click Samadhan

AI marketing platform for local businesses in India. Phase 1 (foundation) is complete: docs, schema, project setup, auth, app shell and a protected dashboard.

Read first: `docs/01-positioning-and-prd.md`, `docs/02-architecture-and-folder-structure.md`.

## Run it
```bash
npm install
cp .env.example .env.local          # fill NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY, NEXT_PUBLIC_APP_URL
```
1. Create a Supabase project. In the SQL editor run `supabase/migrations/0001_init.sql` (or `supabase db push`).
2. Supabase → Authentication → Providers → enable **Email** and **Google** (add your Google OAuth client ID/secret). Add `http://localhost:3000/auth/callback` and your production URL to Redirect URLs.
3. `npm run dev` → http://localhost:3000
4. To make yourself admin: in the SQL editor run
   `update public.profiles set role = 'admin' where id = (select id from auth.users where email = 'you@example.com');`
5. Optional: `npm run db:types` to generate typed DB helpers.

## Notes
- Google sign-in (Supabase Auth) is separate from connecting a Google Business Profile. The latter uses its own OAuth client with the `business.manage` scope (module 2.3).
- PWA: the service worker registers in production builds only. Test with `npm run build && npm start`.
- shadcn/ui: `components.json` is configured; run `npx shadcn@latest add dialog sheet tabs ...` for more components. Button, Card, Input, Label are already included.

## Phase 2.1: Authentication + Onboarding
1. Run `supabase/migrations/0002_onboarding.sql` after `0001` (adds `locality`, `category_other`, the `replace_business_services` function, and logo limits).
2. Supabase → Authentication → URL Configuration → Redirect URLs: add `<APP_URL>/auth/callback` (needed for Google sign-in, email confirmation and password reset).
3. Flow: sign in → if `profiles.onboarding_completed` is false the app shell sends you to `/onboarding` → 4 steps → `/dashboard`. Progress is saved after steps 1 and 2, so closing the tab resumes where you left off.

## Phase 2.2: AI Growth Dashboard
- Shows **sample data** by default (a "Sample data" tag appears in the header). Set `DASHBOARD_DATA_SOURCE=live` in `.env.local` to read real data from Supabase.
- Preview the empty state: open `/dashboard?demo=empty` (sample mode only).
- Code: data in `src/modules/dashboard/` (types, mock, live queries, one switch in `get-dashboard-data.ts`), UI in `src/components/dashboard/`.
- Channel grouping: Google = `google` + `google_ads`; Meta Ads = `meta_ads` + `facebook` + `instagram`; WhatsApp = `whatsapp`; Other = `website`, `manual`.
- "Today" and "this week" use India time (IST); the week starts on Monday.

## Phase 2.3: Google Business Profile AI Agent (mock)
- Open `/google-business`. First visit shows the connect screen; "Connect" only simulates it (a cookie), no Google call is made.
- Tabs: Overview (health score + SEO description), Reviews, Posts & Photos, Performance. State lives in the browser so actions feel real; it resets on refresh.
- Mock data uses your real onboarding details (name, city, services, description), so the health score and AI text match your business.
- **Going live later:** implement `GbpProvider` and `GbpAi` (see `src/modules/gbp/service.ts`) and return them from `getGbpProvider()` / `getGbpAi()`. UI, actions and health scoring stay as they are. Real Google access needs API approval and a separate OAuth client (`business.manage` scope).
- Health score is calculated in `src/modules/gbp/health.ts` (pure function, 9 weighted checks totalling 100).

## Phase 2.4: AI Creative Studio
- Open `/creative-studio`. Four kinds: Social post, Meta ad, Google ad, Festival / offer (post + Meta ad + WhatsApp message).
- **Real:** the Library. Creatives are saved in `creatives`, publish records in `creative_publications` (needs migration `0001`).
- **Mock:** text (`mock-ai.ts`), images (`mock-image.ts`, drawn from a seed and marked "SAMPLE IMAGE") and publishing (`mock-publisher.ts`). Ads are only ever sent as *drafts*.
- **Going live:** implement `CreativeAi`, `ImageGenerator` and `CreativePublisher` (see `src/modules/creative/service.ts`). For real images return a `url` (upload to the `creatives` storage bucket); the UI shows it automatically.
- **Festival dates** live in `src/modules/creative/festivals.ts`. Fixed-date festivals repeat yearly; lunar-calendar ones carry an explicit date and need a yearly update (current dates checked Oct 2026, some regions differ by a day).
- The AI never invents discounts: with no offer typed, festival copy stays generic.

## Phase 2.5: WhatsApp AI System
1. Run `supabase/migrations/0003_whatsapp.sql` (adds `leads.birthday`, `whatsapp_campaigns.variables`, and file limits for the `business-assets` bucket).
2. Open `/whatsapp`. First visit shows a connect screen (simulated). Then four tabs: Chats, AI Agent, Campaigns, Audience.
- **Real (saved in Supabase):** agent settings and FAQs (`ai_agent_settings`), files the AI can send (`business_assets` + private storage bucket), audience groups (`audience_segments`), templates and campaigns (`whatsapp_templates`, `whatsapp_campaigns`).
- **Mock:** the inbox and the AI's replies (`mock-agent.ts`, rule-based, no LLM), campaign sending, and audience sizes (stable made-up numbers).
- **Try it:** Chats → "Try your AI agent". Ask for prices, a price list, the address, or "book tomorrow 5 pm" (also in Hindi/Hinglish). Use "Customer goes quiet" to see a follow-up.
- **Going live:** implement `WhatsAppAgent`, `CampaignSender`, `AudienceEstimator` (`src/modules/whatsapp/service.ts`): LLM + WhatsApp Cloud API webhook, real sending with rate limits, and real counts from `leads`.
- **WhatsApp rules already built in:** 24-hour reply window shown per chat, business-started messages only from approved templates, marketing only to opted-in customers with a STOP line, AI says it is an AI, and hand-over to a person.
- The AI never invents prices or opening hours. If a price is not set, it says the team will confirm.

## Phase 2.6: CRM
1. Run `supabase/migrations/0004_crm.sql` (atomic helpers for tags and follow-up steps, and two indexes).
2. Open `/crm`. Everything here is **real** and saved in Supabase: leads, timeline, tags, appointments, follow-up sequences.
   - Leads tab: list or board (New → Contacted → Qualified → Booked → Closed / Lost), search, stage/source/tag/"follow-up due" filters, "Coming up this week" appointments, and a detail drawer (stage, deal value or lost reason, notes, calls, tags, appointments, next follow-up date, WhatsApp consent).
   - Follow-ups tab: sequences start by themselves when a lead reaches their stage, and stop when the lead moves on. Messages go only to customers who agreed to WhatsApp messages. Add the ready-made ones or build your own.
   - Tags tab: create, colour, delete.
3. "Load sample leads" adds 12 clearly marked sample leads (removable in one click). With `DASHBOARD_DATA_SOURCE=live` the dashboard then shows real numbers.
4. The WhatsApp test chat now creates a marked sample lead and appointment in the CRM when the AI captures a lead or books a visit.
5. **Follow-up sending:** `/api/cron/followups` (secured with `CRON_SECRET`) is scheduled every 15 minutes in `vercel.json` (Vercel Hobby only allows daily cron jobs; change the schedule or use Pro). For now sending is simulated: each follow-up is recorded in the lead's timeline. Plug the real WhatsApp sender into `FollowupSender` in `src/modules/crm/sequences.ts`.
6. Lead sources (Google reviews/calls, WhatsApp, ads, website forms) will write into the same `leads` table when those integrations go live; the CRM needs no change.

## Phase 2.7: Paid Ads Manager
1. Run `supabase/migrations/0005_ads.sql` (adds `ad_campaigns.paused_on`).
2. Open `/ads`. Connect Meta and/or Google (simulated), then use **Dashboard** and **Campaigns**.
- **Real (Supabase `ad_campaigns`):** campaigns, targeting, budgets, status, and the creative linked from Creative Studio.
- **Mock:** the platform calls (launch, pause, resume, budget) and the numbers. Sample numbers are stable per campaign. With `ADS_DATA_SOURCE=live` the dashboard reads `ad_metrics_daily` instead (a server sync job must fill it; owners can only read that table). Platform calls are not built yet and are refused in `live` mode.
- **Spending safety:** AI only creates *drafts*. Launching needs the owner to tick a confirmation showing the daily budget, days and the most that can be spent. Changing a budget needs a confirmation too, and a rise of more than 50% needs a second one. Budget limits: ₹100 to ₹50,000 a day, up to 90 days, adults (18+) only (see `src/modules/ads/constants.ts`). The server re-checks every amount, so nothing can change between "confirm" and "launch".
- **AI suggestions** are plain rules in `src/modules/ads/optimize.ts` (no leads after ₹800, cost per lead above 1.5× your typical, low CTR, falling CTR, strong ROAS). They wait until a campaign has run 3 days and spent about ₹500. The planner's budget and result ranges (`planner.ts`) are placeholders to calibrate with real data.
- **Going live:** implement `AdsPlatform`, `AdsPlanner` and a sync job for `ad_metrics_daily` (`src/modules/ads/service.ts`). Needs a Google Ads developer token and Meta app review. Leads from ads should land in `leads` with source `meta_ads` / `google_ads`.
- "Load sample campaigns" adds three marked samples (one doing well, one poorly, one paused) so the dashboard and suggestions can be seen.

## Phase 2.8: Billing, Settings and Admin
1. Run `supabase/migrations/0006_billing.sql` (atomic AI-credit metering). `SUPABASE_SERVICE_ROLE_KEY` must be set: plans, payments and credits are written only by the server.
2. **Settings** (`/settings`): Business (profile, services, logo), Connections (every connected account in one place, with disconnect), Notifications (saved; sending them is not built yet), Account (name, language, delete my account). Hours of operation are not editable yet.
3. **Billing** (`/settings/billing`): current plan, monthly use, plan cards (monthly/yearly), payment history, cancel (the plan runs to the end of the paid period).
4. **AI credits are now enforced** on: Google description, review reply and post; Creative Studio text and images; ad planning (costs in `src/modules/billing/constants.ts`, placeholders to tune). Spending is one atomic database call (`consume_usage`), so parallel requests cannot go past the limit. If the credit check itself fails, the AI call does not run. **WhatsApp campaign messages** are limited by plan too (Free has none). Contacts, locations, ad accounts and seats are shown but not enforced yet.
5. **Razorpay**
   - Without keys: *demo checkout*. Choosing a plan switches it instantly, nothing is charged, and the payment is marked "demo".
   - With keys (`NEXT_PUBLIC_RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`): create the four plans in the Razorpay dashboard (Subscriptions → Plans), paste their ids in **Admin → Plans**, and add a webhook to `<APP_URL>/api/webhooks/razorpay` with `RAZORPAY_WEBHOOK_SECRET` and the events `subscription.activated`, `charged`, `pending`, `halted`, `cancelled`, `completed`. Checkout creates a subscription, the browser returns a signature that the server verifies, and the verified webhook is the final source of truth. Test in Razorpay test mode first.
   - Switching between paid plans starts the new subscription and cancels the old one immediately (no part-refund). `TOTAL_COUNT` in `constants.ts` is the number of billing cycles; check Razorpay's allowed maximum for your account.
   - Prices come from the `plans` table. Decide whether they include GST; checkout adds GST where it applies.
6. **Admin** (`/admin`, role `admin` only; others get a 404): overview (users, paying customers, estimated MRR, money collected in 30 days), users with plan and status, a support tool to set a plan (does not touch Razorpay), and Razorpay plan ids. Make yourself admin with: `update public.profiles set role = 'admin' where id = (select id from auth.users where email = 'you@example.com');`
7. **Delete my account** removes the account, business, customers, messages and files (needed for the DPDP Act). With live Razorpay it asks you to cancel a paid plan first.

## Home redesign (after Phase 2.8)
1. Run `supabase/migrations/0007_home.sql` (daily Marketing Score history). The Home page still opens without it; the "Growth" chart then shows sample history.
2. New look: navy header, sidebar and bottom bar, brighter teal and saffron (tokens in `src/app/globals.css`).
3. Header on every page: business name and phone, **Help** (`/help`), **bell** (in-app notifications, saved in `notifications`), profile menu. Set `NEXT_PUBLIC_SUPPORT_EMAIL` / `NEXT_PUBLIC_SUPPORT_WHATSAPP` to show contact buttons on the Help page.
4. **Home** (`/dashboard`):
   - *Unlock your growth* banner (Free plan only; price read from the plans table).
   - Carousel: **Marketing Score** (10 checks, 100 points, `src/modules/home/score.ts`), **Growth in Marketing Score** (one row a day in `marketing_scores`), **Marketing Tracker** (this week / last week, Monday to Sunday, India time; targets in `WEEKLY_TARGETS`).
   - Summary cards, Google performance (sample numbers until Google is live), next scheduled post (Edit / Cancel schedule), social calendar, channels, recent leads, social connections (shown as "Coming soon"), quick actions.
   - Floating **Growth Action** button with the 3 best next steps (`src/modules/home/growth-cards.ts`).
5. A scheduled post whose time has passed counts as *posted* until a real publishing job exists. In-app notifications are created when the WhatsApp test chat captures a lead and when a campaign is sent.
6. Not built yet: AI-written weekly posts with Accept/Reject, the voice (mic) button, and a website builder.
