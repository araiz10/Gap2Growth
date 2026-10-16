# Gap2Growth — Product Specification

**Status:** Hackathon MVP specification  
**Build dates:** 19–22 October 2026  
**Team size:** 2 members  
**Product:** AI-powered web application for job and internship readiness

## 1. Overview

**Gap2Growth** helps students, fresh graduates, and early-career job seekers understand how their skills compare with a specific job or internship. Users create a profile, paste an opportunity description, and receive an AI-generated report showing relevant strengths, possible skill gaps, priorities, a short learning plan, and a portfolio-project suggestion. Reports are saved privately so users can return to them later.

**One-line pitch:** Gap2Growth identifies your skill gaps for a job or internship and creates a personalized roadmap to help you become a stronger candidate.

### Problem
Applicants often struggle to translate job descriptions into a practical preparation plan. Gap2Growth combines a user's stated skills and projects with the requirements of one opportunity to provide concrete, personalized next steps rather than generic career advice.

## 2. Target users and goals

**Target users:** University students applying for internships, fresh graduates, and early-career applicants targeting entry-level roles.

**MVP goals:**
- Compare a user's profile with a pasted job or internship description.
- Make a live hosted AI call part of the main product workflow.
- Save reports in a real database and retrieve them after a later login.
- Support sign-up, sign-in, and authenticator-app two-factor authentication (2FA).
- Keep each account's profile and reports private from all other accounts.

## 3. Scope

### Must-have MVP
1. Sign-up, sign-in, sign-out, and TOTP authenticator-app MFA.
2. Profile form: skills, experience level, education, and project examples.
3. Opportunity form: `Internship` or `Job`, role title, optional company, and pasted description.
4. Live AI analysis using the opportunity and the user's profile.
5. Structured report: matched requirements, potential gaps, priorities, learning tasks, and a portfolio-project idea.
6. Database-backed report history and report detail page.
7. PostgreSQL row-level security (RLS) and server-side authorization.
8. Responsive interface, validation, loading and error states.
9. Public GitHub repository, required documentation, demo video, and two event-built Devmarketplace code gigs.

### Out of scope for four days
Resume/PDF parsing, OCR, job-board scraping, automatic applications, recruiter accounts, external course integrations, payments, social features, and live labor-market data. Only add optional features after all mandatory requirements work.

## 4. Main user journey

1. User signs up and enrolls in TOTP 2FA by scanning a QR code or entering a setup key, then verifies a code.
2. User fills in their skills, experience, education, and projects.
3. User selects `Internship` or `Job`, enters the role title, and pastes the description.
4. The server validates the request and verifies the authenticated session and required MFA assurance.
5. The server loads the user's profile, calls the hosted AI model, validates its structured response, and saves the input and report to PostgreSQL under the authenticated user's ID.
6. The user views the report and can find it in their saved history.
7. The user signs out, signs back in with the required MFA challenge, and retrieves the same report from the database.
8. A second account must not be able to see, edit, or delete the first user's profile or reports—even if it knows a report ID.

## 5. AI feature

### What the AI does
The model receives the job/internship description, opportunity type, role title, and relevant profile details. It performs four connected tasks:
1. **Extract:** Identify required and preferred skills from the description.
2. **Compare:** Categorize each requirement as matched, a possible gap, or unclear/not evidenced by the profile.
3. **Prioritize:** Recommend which gaps to address first, considering importance to the role.
4. **Plan:** Suggest realistic learning tasks and a small portfolio project related to the opportunity.

The product is not a general chatbot. The user's actual data changes the analysis, and the result is validated, displayed in a structured report, and saved.

### Expected response shape
The server must validate the model's response against a TypeScript/Zod schema (or equivalent). Suggested fields:

```json
{
  "summary": "Short role-specific summary",
  "readiness_level": "starting | developing | strong_foundation",
  "matched_requirements": [
    { "name": "C++", "importance": "required", "evidence": "Evidence from profile" }
  ],
  "skill_gaps": [
    { "name": "SQL joins", "importance": "required", "priority": "high", "reason": "Not evidenced in the profile", "suggested_action": "Practice joins using a small database" }
  ],
  "unclear_requirements": [
    { "name": "Cloud experience", "reason": "Description does not specify platform or level" }
  ],
  "learning_plan": [
    { "focus": "SQL basics", "tasks": ["Practice SELECT", "Practice JOIN"], "deliverable": "A few working queries" }
  ],
  "portfolio_project": {
    "title": "Small role-related project",
    "purpose": "What it demonstrates",
    "features": ["Feature one", "Feature two"],
    "skills_practiced": ["SQL", "Git"]
  },
  "limitations": ["Assessment is based only on the information supplied"]
}
```

This is an illustrative contract, not a hardcoded response. The application must generate the content through a live AI API call.

### AI quality and failure handling
- Compare against the actual profile and description; do not return the same generic report for every user.
- Distinguish “not evidenced in the profile” from “the user definitely does not have this skill.”
- Do not invent qualifications, employer details, courses, URLs, or hiring probabilities.
- Treat pasted descriptions as untrusted data, not instructions that override system rules.
- Present the result as guidance, not a guarantee of employment.
- Keep API keys on the server. Limit input/output size and handle provider timeouts or invalid JSON gracefully.
- If AI fails or output validation fails, show a retryable error and do not save a fabricated successful report.

## 6. Pages and interface

| Page | Purpose |
|---|---|
| Landing page | Explain the product and link to sign-up/sign-in. |
| Sign-up / sign-in | Account access and validation. |
| MFA setup / challenge | Enroll and verify TOTP; complete MFA when required. |
| Profile | Enter and edit skills, education, experience, and projects. |
| Dashboard | Create a new analysis and browse saved reports. |
| New analysis | Select opportunity type and enter role title/company/description. |
| Report detail | Display summary, strengths, skill gaps, learning plan, project idea, and limitations. |

Focus UI polish on the new-analysis form, report readability, and dashboard. Include loading, empty, validation, and failure states. Use accessible labels, keyboard-friendly controls, and do not rely on color alone to communicate status.

## 7. Technical approach

**Recommended stack:** Next.js App Router + TypeScript, Tailwind CSS, Supabase Auth, Supabase PostgreSQL, Supabase TOTP MFA, server-side hosted AI API, Zod validation, and Vercel deployment. Choose one AI provider and one deployment path on Day 1; do not switch stacks late in the event.

### Request flow
1. Browser submits the opportunity form to a server-side route.
2. Server validates input and checks the session and required MFA assurance.
3. Server reads the user's profile using the authenticated identity.
4. Server sends the profile context and opportunity text to the AI provider.
5. Server validates the structured output and writes the input, profile snapshot, result, and metadata to PostgreSQL.
6. UI displays the saved report and history retrieves it from the database on later visits.

Never expose an AI API key or Supabase service-role key to browser code. Derive ownership from the authenticated session; never trust a user ID sent by the client.

### Minimal database schema

**`profiles`**
- `user_id` (UUID, primary key linked to `auth.users.id`)
- `education_level` (text)
- `experience_level` (text)
- `skills` (JSONB)
- `projects` (JSONB)
- `created_at`, `updated_at`

**`analyses`**
- `id` (UUID primary key)
- `user_id` (UUID owner linked to `auth.users.id`)
- `opportunity_type` (`internship` or `job`)
- `role_title`, `company_name` (optional), `description`
- `profile_snapshot` (JSONB, relevant profile fields used for that report)
- `result_json` (JSONB, validated AI output)
- `provider`, `model`, `prompt_version` (where practical)
- `created_at`

Enable RLS on both tables. Policies must restrict select/insert/update/delete to the authenticated owner. Inserts must not allow a user to assign another owner. Check MFA assurance at the server and protect database operations appropriately; the UI alone is not a security boundary. Do not use localStorage as the database.

### Minimal API
- `POST /api/analyses`: verify user/MFA, validate input, call AI, validate output, save and return the report ID/result.
- History/detail operations: read only the signed-in user's records, enforced by RLS.
- Profile operations: create/read/update only the signed-in user's profile.

## 8. Authentication and security

Use managed authentication rather than implementing passwords from scratch. Implement actual TOTP enrollment, code verification, and MFA challenge—not just a setup screen. Protected actions must check the authenticated session and required MFA assurance (for Supabase, normally `aal2`). Test RLS for every exposed table and operation. Never store passwords or TOTP secrets in application tables, commit secrets, or log one-time codes/tokens. Explain in a short privacy notice that profile and opportunity text are sent to the configured AI provider to produce the report.

## 9. Acceptance tests (mandatory)

- [ ] User can sign up, enroll and verify TOTP, sign out, and sign back in with MFA.
- [ ] User can save a profile and submit a job/internship description.
- [ ] The deployed app makes a real hosted AI call using that user's profile and submitted description.
- [ ] The AI result is schema-validated and saved in PostgreSQL.
- [ ] After logout and login, the user retrieves the same report from the database.
- [ ] Two test accounts cannot read or modify each other's profiles or reports, including by guessing/changing a report ID.
- [ ] AI/API failures show an understandable error and do not produce fake successful reports.
- [ ] Public live URL works in a private browser window and has no placeholder or broken core screens.
- [ ] Public repository includes required docs, demo video, and two published code gigs.

## 10. Four-day plan and two-person responsibilities

### Member A — Frontend, product flow, integration
Owns the landing page, profile/opportunity forms, dashboard/report UI, responsive styling, and end-to-end integration. Helps validate the deployed user journey.

### Member B — Backend, AI, data, security
Owns Supabase schema/RLS, authentication and MFA integration, AI API route, output validation, persistence, and account-isolation tests.

**Both members** review code, test the full flow, write documentation, record the demo, and contribute to the code gigs. Coordinate early on route names, schema, and response shape to avoid integration problems.

| Date | Focus | Exit condition |
|---|---|---|
| **19 Oct** | Create public repo/deploy skeleton; configure Supabase, schema/RLS, sign-up/sign-in and TOTP. | Two accounts can authenticate; protected data is isolated. |
| **20 Oct** | Build profile and opportunity forms; connect live AI call, schema validation, and save flow. | One user receives and saves a real AI report. Publish code gig 1. |
| **21 Oct** | Build dashboard/history/detail, handle errors, test MFA and cross-account access. Publish code gig 2. | Full journey works after logout/login; no cross-account access. |
| **22 Oct** | Fix bugs, test private-browser deployment, finish docs and demo video. No new features unless all gates pass. | Submission checklist completed. |

## 11. Repository and event deliverables

The public repository must include:
- `README.md`: purpose, stack, live URL, setup steps, environment variables, and run instructions.
- `SPEC.md`: this specification, updated to reflect the final build.
- `AI_DISCLOSURE.md`: AI models/tools used and all reused/open-source components with links and licences.
- `PRIOR_WORK.md` only if required by the event's reuse rules.
- Commits distributed across **19–22 October**, not one final bulk commit.

Publish at least two code gigs made from code written during the event, with clear titles, descriptions, setup instructions, and teammate contributor credits. Suggested gigs:
1. **Supabase TOTP MFA flow for Next.js** — reusable enrollment/challenge/verification implementation.
2. **Structured AI Skill-Gap Analyzer** — server-side AI wrapper with schema validation and example use.

Demo video should show sign-up and 2FA setup, a real AI analysis, saved report retrieval after signing back in, and two accounts with separate data.

## 12. Definition of done

Gap2Growth is ready for submission only when the core AI journey works reliably on the deployed app, the result persists across sessions, TOTP MFA is actually enforced for protected access, account isolation passes tests, the public repository has all required documentation, the demo video is available, and both code gigs are published. Optional features must not delay these requirements.

## Technical references
- Supabase TOTP MFA: https://supabase.com/docs/guides/auth/auth-mfa/totp
- Supabase MFA overview: https://supabase.com/docs/guides/auth/auth-mfa
- PostgreSQL RLS in Supabase: https://supabase.com/docs/guides/database/postgres/row-level-security
- OpenAI Structured Outputs (if OpenAI is selected): https://developers.openai.com/api/docs/guides/structured-outputs
