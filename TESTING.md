# PA3 testing

## Reproducible automated checks

From `server/`:

```powershell
npm test
npm run test:integration
```

From `client/`:

```powershell
npm run lint
npm run build
```

The offline suite uses real HTTP requests with injected database failures. It covers malformed JSON, missing fields, invalid types/IDs, safe 500 responses across the endpoints, duplicate-key races, and direct attempts to delete a lead.

The Atlas suite uses `server/.env` and the existing `pa2` database. It creates three users and three projects with UUID-prefixed test names. Cleanup removes only those exact fixture names/IDs and their memberships; no database/collection is dropped. Existing records are not modified. Do not interrupt the process while cleanup runs.

The Atlas tests cover:

- Signup success, duplicate username, missing fields, correct/incorrect/nonexistent login.
- User responses contain only `_id`, `f_name`, `l_name`, and `username`.
- All three project statuses and automatic lead membership.
- Relationship fields really are BSON ObjectIds in MongoDB.
- Project list includes newly created projects.
- Every required project field, whitespace-only strings, invalid status and IDs.
- Nonexistent lead/project/user and absent membership.
- One user in multiple projects, duplicate addition, lead protection, non-lead removal.
- Two simultaneous adds return one 201 and one 409, leaving exactly one membership.
- A direct duplicate MongoDB insert fails with duplicate-key error 11000.
- Injected failure during the second transaction insert rolls back the first insert in real Atlas.

## Manual browser checks

1. Start the backend and frontend as documented in README.
2. Open Accounts and sign up three users. Repeat one username and check the duplicate message.
3. Test correct and incorrect login credentials.
4. Return to Projects. Newly registered users should appear without reloading the browser.
5. Submit an empty project form; expect a useful validation message.
6. Create Planning and Active projects with different leads. Each new card appears immediately with a readable lead name.
7. Verify each lead appears in the selected project's member list without a Remove button.
8. Add the third user to both projects. Switch between projects and verify independent lists.
9. Add that same user again to one project; expect the duplicate-membership message.
10. Remove a non-lead member; confirm the list changes and the other project's membership stays intact.
11. Check an empty database state if available: clear guidance should appear when no users/projects exist. Do not delete existing data just to test this.
12. Stop the backend and click Refresh data; expect a connection error. Restart it and click Retry.
13. Narrow the browser to a phone-sized viewport; forms should stack without horizontal scrolling.
14. Inspect Atlas: verify project lead and membership references are ObjectIds and the compound membership index is unique.

## Direct API error examples

Run in PowerShell with the server running. For valid requests, replace placeholders with IDs returned by GET `/api/users` and GET `/api/projects`.

```powershell
Invoke-RestMethod http://localhost:9000/api/users
Invoke-RestMethod http://localhost:9000/api/projects

# 400: malformed project identifier
curl.exe -i http://localhost:9000/api/projects/invalid/members

# 400: missing required fields
Invoke-WebRequest http://localhost:9000/api/projects -Method Post -ContentType 'application/json' -Body '{}' -SkipHttpErrorCheck

# For a real project, use its lead's ID here: expect 400.
curl.exe -i -X DELETE http://localhost:9000/api/projects/PROJECT_ID/members/LEAD_ID
```

## Verified on 2026-10-14

- Frontend production build: passed.
- Frontend ESLint: passed.
- Offline API suite: 4/4 passed.
- Live Atlas suite: 6 subtests plus the parent test passed (7/7 reported by Node).
- Actual Atlas transaction rollback and concurrent duplicate protection: passed.
- `server/.env` SHA-256 before/after implementation matched.
- Git ignores `.env`, both `node_modules` directories, and client build output.

A real Chromium browser against the running React/Express/Atlas application passed 10 workflow checks: signup/login, required fields, two projects, automatic leads, many-to-many membership, duplicate feedback, removal/refresh, switching projects, empty/offline/retry states, and mobile layout. Desktop and 390px screenshots were visually inspected. There were no uncaught JavaScript errors. Browser fixtures were removed afterward. Playwright was installed only in the external task scratch directory, not as an application dependency; the manual steps above reproduce the workflow.
