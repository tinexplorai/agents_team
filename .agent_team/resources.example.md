# Project Resources

> **Concrete values agents look up at build time.** Copy this file to `resources.md` (gitignored) and fill in the values for your project.
>
> **Lookup protocol** *(every agent must follow this — Team Lead enforces):*
> 1. Before asking the user for any concrete identifier (URL, slug, ID, path, project ref), READ `.agent_team/resources.md` first.
> 2. If the value is filled (not `[PLACEHOLDER]` and not `N/A`), use it as-is.
> 3. If the value is `[PLACEHOLDER]` or missing, ask the user in chat.
> 4. After the user answers, **write the value back into `resources.md`** under the matching key, so the next agent doesn't ask again.
> 5. If the value is `N/A` (user explicitly deferred), do NOT ask the user again. Make a reasonable default decision yourself, document it in your deliverable (e.g. `docs/api_contract.md`, `docs/deployment.md`), and flag it via `task_board.md`.
>
> **Secrets** (tokens, API keys, passwords) live in `.env` (gitignored, copy from `.env.example`) — NEVER here. This file holds non-secret identifiers, slugs, paths, and references only.

---

## Database (Supabase)

> Skip this section if the project does not use Supabase.

- **project_ref:** `[xxx-xxx-xxx]` *(found in Supabase dashboard URL: `https://supabase.com/dashboard/project/<REF>`)*
- **project_url:** `https://[REF].supabase.co`

> Keys live in `.env`:
> - `SUPABASE_ACCESS_TOKEN` — MCP management token (build-time, used by Architect/DEV agents).
> - `SUPABASE_URL`, `SUPABASE_ANON_KEY` — runtime, used by the deployed app to connect to the DB.
> - `SUPABASE_SERVICE_ROLE_KEY` *(optional)* — runtime, server-side privileged operations only.

---

## Source control + Web deploy

- **github_repo_url:** `https://github.com/[owner]/[repo]`
- **default_branch:** `main`
- **vercel_team_slug:** `[team-slug]` *(only if web=yes)*
- **vercel_project_slug:** `[project-slug]` *(only if web=yes)*
- **vercel_env_var_names:** `[DATABASE_URL, JWT_SECRET, SUPABASE_URL, SUPABASE_ANON_KEY]` *(names only — values are read from `.env` or set during deploy)*

---

## Mobile distribution

> Skip this section if the project is web-only.

- **bundle_id:** `[com.example.myapp]`
- **apple_team_id:** `[XXXXXXXXXX]` *(only if iOS=yes — found at https://developer.apple.com/account)*
- **ios_distribution_channel:** `[TestFlight / App Store]`
- **android_package:** `[com.example.myapp]` *(only if Android=yes)*
- **android_distribution_channel:** `[Play Console internal track / Firebase App Distribution / Play Store]`
- **github_secrets_required:** *(DevOps Agent lists exact names + how to generate them in `docs/deployment.md` — user adds them in repo Settings → Secrets and variables → Actions)*
  - iOS: signing certificate (.p12 base64), provisioning profile, App Store Connect API key (.json)
  - Android: keystore (base64), keystore password, Play Console service account (.json)

---

## Design

- **design_input_folder:** `docs/design_input/` *(default — drop PDFs / PNG / JPG here)*
- **figma_file_url:** `[https://www.figma.com/file/<KEY>/<NAME>]` *(optional — Designer Agent uses it as fallback if `design_input/` is empty)*
