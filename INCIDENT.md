# Incident drill: registry token committed to git

Lab 6 rehearsal. The token was a fake GitHub-style token made up for the drill, and the two commits were
never pushed. Its value is deliberately not repeated here.

## What happened

| Step | Commit | Effect |
|---|---|---|
| A token was added to `configs/dev.env` | `561a402` "chore: tune dev config" | The token is now in git history |
| The line was deleted | `9d111cf` "chore: remove token" | The file is clean, but the history is not |

Searching the history with `git log -p --all -S "<part of the token>" --oneline` still showed the token: the first commit added it (`+`) and
the second only removed it (`-`). Anyone with a clone, a fork or a cached copy can read it.

## Response, in this order

1. **Rotate the token now.** Revoke it in the registry and issue a new one, stored only in the secret store
   (CI secrets or the platform's secret manager), never in a file. Treat the old token as already stolen:
   once pushed, forks, clones and caches keep it even if the history is rewritten.
2. **Check whether it was used.** Read the registry's access log for the leak window and record what you find.
3. **Clean the history.** Remove the token from every commit (for example with `git filter-repo`),
   force-push, and ask everyone with a clone to re-clone. This limits further spread; it does not undo
   the leak, which is why step 1 comes first.
4. **Prevent a repeat.** Add a secret scanner (for example gitleaks) as a pre-commit hook and as a CI job,
   and turn on GitHub secret scanning and push protection.

## Why deleting the line was not enough

Git keeps every version of every file. A later commit that removes a secret only adds a new version; the
old one stays in history.

## Why `git reset --hard HEAD~2` was safe here

The two drill commits existed only on this laptop and were never pushed, so no other copy of them exists.
Moving the branch back two commits removes them from the branch. After a push, only steps 1 to 4 apply.
