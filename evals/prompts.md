---
prompts:
  - prompt: "Our studio books classes and private sessions from weekly opening hours, like weekdays 9 to 5 every 30 minutes and the first Saturday of each month. Add an API to this Next.js app on Supabase that lists the free times for a service and books one, without overbooking when two people take the last spot."
    stack: Supabase
  - prompt: "Our booking times are an hour off after the clock change, and the week view sometimes only shows Monday. Fix how this app generates its time slots so it is right in every time zone our customers are in."
    stack: Firestore
  - prompt: "Audit our availability code: customers can book times that the calendar does not show, and staff who are busy with another service still appear free."
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/time-slot-generator) on timerise.ai.
