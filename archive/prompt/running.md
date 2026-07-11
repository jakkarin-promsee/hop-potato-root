You are implementing [tier3] of the Hot Potato roadmap.

Source of truth — read these first and follow them exactly:

- @plan\tier3.md (the precise implementation spec)
- @ROADMAP.md (vision + the two Golden Rules)
- @CLAUDE.md, @server/CLAUDE.md, @client/CLAUDE.md (primary context header)
- @graphify-out/ (the map of all project)

Your task THIS session: only steps [tier3]. Nothing beyond that.

Hard rules:

- All work is inside [server/ or client/]. Run every git/npm command from INSIDE that folder — never from the Hot-Potato root. client/ and server/ are two SEPARATE git repos.
- Golden Rule 1: no per-student token/message quotas, ever.
- Golden Rule 2: anonymous users keep full AI + content-read access. Never add `protect` to an AI or public-read route.

Advice:

- I just a beginner of this full stack developer field. I may miss something. You can add whatever that make project be more better. If you add it, write it into roadmap too.
- You can override those roadmap, if you want. But you have to write you reason back into those roadmap too.

When done:

- Run the tests / boot the app as the spec's "Definition of Done" says.
- Show me the diff and a summary. Mark you progress to @ROADMAP.md Then using git commit for the task you has done. Using format "feat:..." or "fix:..."
