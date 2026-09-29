# CLAUDE.md

Working agreement for UpSkillPros repos. Read this at the start of every session.
Same file in every repo. The project sections near the bottom are the only part
that differs by repo; read the one that matches, skim the rest, because lessons
cross over.

---

## 1. Who you are working with

Jon Richards, sole operator of UpSkillPros, a one-person consultancy building
SvelteKit and Supabase apps for nonprofits and small businesses.

Jon is the judgment layer, the human in the loop. He is not a coder and does not
want to be one. He will not claim or defend line-by-line code comprehension, and
you should never write a claim like that on his behalf. His accountability model
is verification: testing shipped work against spec, proving the security model,
and trying to break it.

What this means for you in practice:

- Explain what a change does and why, in plain language, before or alongside the
  diff. "This moves the token spend from GET to POST" is useful. "Refactored the
  confirm handler" is not.
- Never leave him to infer a consequence. If a change has a side effect, an
  ordering requirement, or a moment where the app is broken between two steps,
  say so in the same message.
- He is fluent in VS Code, GitHub Source Control, the Supabase SQL editor, and
  Vercel. Skip hand-holding there. For anything less familiar, name the exact
  control and where it is on screen.
- He runs this alongside other work and will forget details and repeat
  questions. Correct the facts every time, as a gentle reminder rather than a
  verdict.

---

## 2. How to work

**Confirm architectural decisions before writing code.** When there is a real
choice (where state lives, which table owns a fact, whether something is a
trigger or app logic), lay out your recommendation and the tradeoff, then wait.
Do not present a menu of five options; give one confident recommendation with
reasoning.

**One step at a time on anything multi-part.** Finish a step, say what to test,
and stop. Do not bundle three migrations and four files into one turn.

**Commit before you start.** Jon's undo is git. Before any change that touches
more than one file, confirm the working tree is clean or ask him to commit.

**Say where you are in a sequence.** "Step 2 of 3" so he knows what is still
coming and what state the app is in right now.

**Never assume a file's contents.** Read it. Jon edits between sessions and so do
you.

**Tell him what to test, in his words.** After a change, give the specific
clicks and what a pass looks like. Use bracket codes, [PASS] and [FAIL], never
color alone. Distinguish "the VS Code terminal" from "the browser." A 404 is a
pathing problem; a 500 means the code crashed.

---

## 3. Validation gates, non-negotiable

Nothing is delivered until it passes these:

- **Svelte:** compiles clean in runes mode with zero warnings.
- **TypeScript:** checked under strict tsc. `npm run check` is a standing gate,
  and the svelte-check baseline is 0 errors.
- **SQL:** validated against real Postgres before delivery. Migrations are
  idempotent. Risky migrations get validated locally on Postgres 16 first.
- **Conventions:** verify Supabase and SvelteKit conventions against live docs
  before writing anything schema-, RLS-, or framework-adjacent. Never from
  memory. Both move faster than your training data.
- **Introspect before designing.** Check actual column names, sign conventions,
  and existing objects before writing a migration. This catches collisions with
  objects that already exist.

After every migration, remind Jon of the schema snapshot ritual: the single-row
`string_agg` query, then Export and Download CSV, so the snapshot in project
files stays current.

---

## 4. Writing style

Two audiences, two rulebooks.

**Anything a client or end user will read** (UI copy, emails, error messages,
documentation, marketing):

- No em dashes. Use commas, periods, or a new sentence. Em dashes are allowed in
  labels only: kickers, nav and UI labels, meta titles, chart tags, email
  subject lines.
- Use contractions wherever one fits. Don't, can't, what's, you'd. The
  uncontracted form reads stiff.
- Plain language, no jargon. An error message should tell the person what to do
  next, not what the system experienced.
- Minimal semicolons and colons.

**Chat and code comments:** normal rules, em dashes fine.

Code comments carry the reasoning, not the mechanics. Explain why a thing is
built this way, what was rejected, and what breaks if someone changes it. The
existing code in these repos is commented that way on purpose. Match it.

---

## 5. Accessibility

- **Color is never the sole signal**, anywhere. Reinforce with label, shape,
  weight, or position.
- **Internal and admin surfaces:** bracket codes and text labels, [DELETED],
  [PENDING], [PASS].
- **Client-facing product UI:** universal accessibility best practices, designed
  for general audiences.
- Interactive controls work from the keyboard, name themselves to screen
  readers, and have a visible focus state.

---

## 6. Stack

SvelteKit 2, Svelte 5 in runes mode, TypeScript, Supabase (Postgres 16, Auth
SSR, RLS), Vercel. CSS-variable theming, no Tailwind.

Repos, all under SGRJon:

| Repo | What it is |
|---|---|
| `mbca-command-center` | Multi-tenant nonprofit finance and operations platform. Live. |
| `recognition-platform` | Multi-tenant peer-to-peer employee recognition. In build. |
| `story-engine` | Reusable tap-through story engine. FindMe pitch is story one. |
| `upskillpros-site` | upskillpros.com. Plain static HTML, no framework. |

Vercel auto-deploys on push to main.

---

## 7. Hard-won lessons

These were each paid for once. Do not rediscover them.

### Postgres and Supabase

- Table-level GRANT and RLS policy are separate layers, and the grant is checked
  first. An RLS-enabled table with a correct policy still 500s if the role lacks
  the DML grant.
- `FORCE ROW LEVEL SECURITY` silently returns zero rows to SECURITY DEFINER
  functions. Deliberately avoided in these projects.
- Any read that must see every row has to paginate. The default cap is 1000.
- `ON CONFLICT (col)` requires a formal unique constraint, not just a unique
  index.
- `USING (true)` policies are a future leak even when they do not leak today.
- CHECK constraints cannot contain subqueries. Use an `IMMUTABLE` helper
  function that the CHECK calls.
- `security_invoker = true` views enforce RLS at the caller's privilege level.
- `pg_advisory_xact_lock` for check number assignment, with the number derived
  from max rather than a stored counter. Proven safe under concurrent load.
- Changing a function's `RETURNS TABLE` signature requires DROP then CREATE, and
  DROP takes its grants with it. Re-grant in the same migration.

### SvelteKit and Svelte 5

- Server logs success but the page hangs: that is a client render crash. Look in
  the browser DevTools console, not the terminal.
- Never key an `{#each}` on data that can legitimately collide. Append the index.
- `use:enhance` with `update({ reset: false })` is needed on forms that display
  persisted values. In Svelte 5, `form.reset()` restores `defaultValue`, an
  empty string, not the DOM property value.
- `+page.server.ts` permits only `load`, `actions`, config options, or
  underscore-prefixed named exports. A plain named export 500s the route.
- `$effect` that writes state it reads causes an infinite loop. Use `untrack`
  for initialization.
- Use `onMount`, not `$effect`, for localStorage initialization.
- Dirty-tracking baselines must be re-seeded after save. Use a
  `savedAt: Date.now()` token from the server so consecutive saves are
  distinguishable.
- `margin: 0 auto` centering can silently fail in a production build. Flex-parent
  centering is robust to dev-versus-prod build order.
- Dark mode: double the root selector, `:root:root`. Vite injects app.css after
  the SSR'd head block in dev, so an equal-specificity `:root` rule loses on
  source order.
- `display: contents` on a panel wrapper fights media queries. Use `matchMedia`
  in JS for mobile and desktop presentation splits.

### Architecture principles

- **Confirm-to-commit for anything irreversible.** GET renders only; POST is the
  sole commit point. This is why emailed auth links land on a page with a button
  instead of spending their token on load: corporate and university mail
  scanners open every link on arrival, and a GET that commits gets consumed by a
  robot.
- **A confidently wrong value is worse than a blank one.** Blank signals "needs
  attention." Wrong looks finished and quietly corrupts a report.
- **A control that sets a value the database ignores is a lie.** Do not ship the
  UI ahead of the enforcement.
- **Resolve names server-side.** A page showing record ids is a page nobody
  reads. If a log or list stores ids, look up the labels in the loader.
- **Git discard recovers an overwritten file.** Check before reconstructing
  anything by hand.

---

## 8. Project notes

### mbca-command-center

Multi-tenant. Tenants today: MBCA (Missouri Basketball Coaches Association), 22
Fund, and a demo org. The client contact is Denny Hunt, the admin who uses this
daily. Tenancy is enforced in the database, not the app. Every table carries
`org_id`; RLS policies use `has_org_role` and `has_org_capability`.

- **Roles:** admin, editor, viewer, event_staff. Capabilities are additive grants
  on top of a role, in `membership_capabilities`. `has_org_capability`
  short-circuits to true for admin and editor, so grants on those roles would be
  stored and never consulted.
- **event_staff** changes what a person can SEE, not just what they can change:
  their world is their grant list in `membership_events`.
- **Reading auth.users requires a SECURITY DEFINER function.** `org_members()`
  is that function, and it re-checks admin in the same transaction as the join.
  Do not reach for `supabaseAdmin.auth.admin.listUsers()`; it returns every auth
  user in the project, across tenants.
- **`audit_log` (Sep 2026)** records updates and deletes on transactions,
  checks, invoices, accounts, memberships, contacts, events, sponsors, and
  board, written only by the generic `audit_row()` trigger. Updates store only
  changed fields; deletes store the whole row; no-op saves write nothing. It has
  a SELECT policy for admins and deliberately no write policy, so the app cannot
  alter history. Adding another table is one CREATE TRIGGER line.
- **The transactions category trap:** the `active_transactions` view and the
  app's read path use the legacy TEXT `transactions.category` column, not
  `category_id`. Any write or seed path must populate BOTH until the 990-prep
  rebuild retires one.
- **Check printing:** TROY 200 Mobile MICR printer over USB, no vendor software.
  Amount is optional while a check is queued and required at print.

### recognition-platform

Multi-tenant peer-to-peer recognition, resale-ready, aimed at organizations up to
roughly 500 people.

- **`Recognition_Platform_DESIGN-LANGUAGE.md` is authoritative** for every visual
  decision. When a screen and that document disagree, the document wins until it
  is deliberately changed first. Its token source of truth is the `:root` block
  in `src/routes/app.css`; when the doc and the file disagree, the file wins and
  the doc gets corrected.
- **Two rulebooks on purpose.** The product surfaces employees touch are
  mobile-first, designed at 380px and thumb-reachable, then expanded for
  desktop. Admin and reporting are desktop-first, dense and capable, with mobile
  as a degraded quick-edit mode. Keep them from contaminating each other.
- The recognition entry experience is the screen the product lives or dies on.
- Executives are a real user class with their own capabilities, such as
  high-value awards. Driving executive participation up is a product goal.
- **GIFs come from KLIPY**, free with attribution ("Search KLIPY" placeholder).
  The provider sits behind a thin app-layer adapter and the database stores only
  the final `media_url`, so the provider stays swappable.
- **Rewards go through Tremendous**, adapter stubbed.

### story-engine and FindMe

A reusable tap-through story engine; the FindMe pitch is its first story.
FindMe is a prospective client product: a child-safety wearable with a QR code
and NFC tag that a stranger can scan.

- **The tag holds a pointer and nothing else.** A unique id in a URL. Never
  encode contact data, vCards, or SMS payloads on the physical tag. NFC uses an
  NDEF URI record with the same URL as the etched QR.
- **Scanning requires no keys.** A scan is a URL opening in a browser. The public
  scan page is server-rendered and featherweight, with no client framework boot
  in the critical path.
- **Privacy is enforced, not displayed.** Hidden fields are filtered server-side
  and never sent to the browser.
- **First claim wins permanently.** Never silently reassign a wearable. The owner
  view is gated on ownership, not on mere authentication.
- **Data minimization is the legal posture.** The parent is the user; the child
  never touches the system. Parent contact plus a parent-chosen label only. No
  ages, DOBs, photos, or schools. Attorney review before any sale is a hard gate,
  not a recommendation.
- **QR codes always encode the stable owned-domain URL**, never a raw
  vercel.app deployment URL, and are generated only after the final URL exists.
- Story content is file-based in v1. No admin dashboard until two or three real
  stories have taught the authoring pain.

### upskillpros-site

Plain static HTML, no framework, auto-deployed by Vercel.

- **GoDaddy holds only the domain, DNS, and the Microsoft 365 mailbox. It has
  never served the site.** Jon has misremembered this more than once; correct it
  rather than planning around it.
- **Sitemap rule:** whenever a page is added or meaningfully changed, update
  `sitemap.xml`. Add the new `<url>` entry and update `lastmod` on anything that
  changed. A wrong date is worse than none.
- Lead capture lives on `consult.html` (Web3Forms with a mailto fallback,
  honeypot and a 4 second time gate). `index.html` slide 7 is a button to
  `consult.html#book` and has no form of its own.
- `proof.html` is the portfolio page, linked in site nav as "THE WORK."

---

## 9. When you are unsure

Ask. A question costs a minute. A wrong assumption about tenancy, money, or
access control costs a client's trust.
