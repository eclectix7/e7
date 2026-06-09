Here's what I've found after researching the landscape. I have three architecture paths and some questions before I write the plan.



Henderson Church Guest Welcome System -- Research Summary

Free Tool Options by Component

Component: Visitor Form
Option A (Easiest): Tally.so (free, unlimited forms & submissions)
Option B (Balanced): Formspree + custom HTML (free tier, 50 submissions/mo)
Option C (Most Flexible): Custom Next.js form on Vercel (free hosting)
────────────────────────────────────────
Component: Database
Option A (Easiest): Airtable Free (1,000 records, Interface Designer included)
Option B (Balanced): Supabase Free (50K MAU, 500MB DB, real-time API)
Option C (Most Flexible): Supabase Free
────────────────────────────────────────
Component: SMS
Option A (Easiest): Twilio Free Trial ($15 credit, ~75 SMS to US numbers)
Option B (Balanced): Twilio Free Trial or Vonage Free Trial
Option C (Most Flexible): Twilio (pay-as-you-go after trial) or self-hosted via SMPP gateway
────────────────────────────────────────
Component: Email
Option A (Easiest): Resend Free (3,000 emails/mo, 100/day)
Option B (Balanced): SendGrid Free (100 emails/day)
Option C (Most Flexible): Resend or SendGrid on paid plan at scale
────────────────────────────────────────
Component: Automation
Option A (Easiest): Make.com Free (1,000 operations/mo)
Option B (Balanced): Make.com Free or n8n self-hosted
Option C (Most Flexible): Custom serverless functions on Vercel/Cloudflare Workers
────────────────────────────────────────
Component: Admin Panel
Option A (Easiest): Airtable Interface Designer (free) or Glide
Option B (Balanced): Retool Free (10 apps) or custom dashboard
Option C (Most Flexible): Full custom admin UI in Next.js

Three Architecture Paths

Path 1 -- All No-Code (Recommended to start)
- Form: Tally.so embeds on your church website
- Data flows automatically via Make.com into Airtable
- Airtable Interface Designer becomes the admin panel
- Make.com triggers Twilio (SMS) and Resend (email) when a visitor opts in
- Cost: $0 (all free tiers). Risk: Airtable caps at 1,000 records on free.

Path 2 -- Hybrid (Recommended for growth)
- Custom HTML/React form hosted on Vercel (free) or your church website
- Formspree or direct Supabase API handles submissions
- Supabase stores all visitor data with full API access
- Supabase Edge Functions or Make.com trigger SMS/email
- Retool (free tier) or custom Next.js page as admin dashboard
- Cost: $0 to start, scales gracefully

Path 3 -- Full Custom (Most powerful)
- Next.js app on Vercel with custom welcome page + form
- Supabase for auth + database + real-time admin updates
- Direct Twilio and Resend API calls from serverless functions
- Custom admin panel with CRUD, template editor, search/filter
- Cost: Free hosting, pay-per-use for SMS/email after free credits



Questions I Need Before Writing the Plan

1. Technical skill level -- Does your church have anyone comfortable with code, or should this be entirely no-code/managed?

2. Expected volume -- How many visitors per week/month? (This determines free tier feasibility)

3. Existing website -- Do you already have a church website/domain where the form would be embedded?

4. Admin count -- How many people need admin access to manage visitor data?

5. Follow-up content -- Will the SMS/email be a simple generic message ("Thanks for visiting!") or personalized with the visitor's name and a link to resources?

6. SMS compliance -- US-based phone numbers only? Do visitors text in a keyword (e.g., "WELCOME") or does the church initiate the SMS after they submit the form?

7. Data sensitivity -- Any privacy concerns (e.g., children's data, GDPR for EU visitors)? How long should visitor data be retained?

8. Timeline -- When do you need this live?

9. Budget ceiling -- You mentioned free is preferred -- is there any wiggle room if the free tiers prove limiting?

10. Follow-up flow -- One-time welcome message, or a series (e.g., Day 1 welcome, Day 3 follow-up, Day 7 small group invite)?



Once you answer these, I'll write the detailed plan and save it to .hermes/plans/. Which path feels right to you?

1. I am a senior developer that will install and set up the application
2. Less than 100/week
3. We have a website that we want to mimic. We may embed the vercel form/page in the site later, but it will be a stand alone page initially
4. 2-10 people will need to be able to access the information
5. The messages will be personalized messages to the visitors
6. US numbers only. Initiate after form submit if the add their number & agree to SMS
7. Visitors may be from any country, but we will only support US numbers. I'm not sure about the GDPR requirements for a local church.
8. Within a week
9. There would be some margin when if there's an unusual number of visitors
10. There should be the immediate welcome with 'opt out' message. admin should be able to add, edit / update the delays for additional messages as well as view records of outgoing messages

