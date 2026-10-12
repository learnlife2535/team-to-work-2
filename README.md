# TEAM OS V4.5

Real-estate team OS (Auth + multi-tenant organizations + jobs/drafts) on Supabase + Netlify.

## Setup

1. Open the deployed site
2. Paste **Supabase Project URL** and **anon key** (Dashboard → Project Settings → API)
3. Sign up / log in
4. Create your organization (you become Owner)
5. Add jobs and manage drafts

## Netlify site

- Project: `team-os-v45-hzqk`
- URL: https://team-os-v45-hzqk.netlify.app

## Supabase

Tables: organizations, profiles, organization_members, properties, content_drafts, jobs, audit_logs (+ related)
RLS enabled. Profile auto-created via trigger on `auth.users`.
