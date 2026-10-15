# ICSI418Y PA3 — Projects and Project Memberships

PA3 extends the existing PA2 signup/login application with project creation, project listing, and many-to-many project membership management. It reuses the existing MongoDB Atlas connection, database `pa2`, and lowercase `users` collection.

## Features

- Existing Signup and Login remain available under **Accounts**.
- Create projects with a name, description, status, and registered Project Lead.
- Status is exactly `Planning`, `Active`, or `Completed`.
- Display project leads and members by name and username.
- Automatically add the lead as a member in the same database transaction as project creation.
- Add a user to several projects; reject duplicate membership within one project.
- Remove non-lead members; protect the lead in both the UI and API.
- Update lists after changes; show loading, empty, success, and failure states with retry controls.
- Refresh registered users after successful signup.

## Technologies and structure

React, JavaScript, Vite, CSS; Node.js, Express 5, CommonJS, and the native MongoDB driver. No Mongoose, JWT, or React Router was added.

```text
client/
  src/
    App.jsx                Shared data, navigation, project selection
    Signup.jsx, Login.jsx  Existing PA2 account forms
    CreateProject.jsx      Project form
    ProjectList.jsx        Project cards and readable lead names
    ProjectMembers.jsx     Member retrieval, addition, removal
    api.js                 JSON requests and user display labels
    App.css, index.css     Responsive styling
server/
  server.js                Existing .env configuration, database connection, startup
  app.js                   Existing auth routes, Express setup, safe error responses
  projects.js              PA3 routes, validation, transaction, unique index
  test/
    app.test.js            Offline HTTP validation and failure tests
    integration.test.js    Real Atlas integration tests with fixture cleanup
    helpers.js             Temporary HTTP test server
README.md
TESTING.md
WALKTHROUGH_ZH.md
```

## Install and run

Use Node.js 24 LTS (tested with 24.15.0). From the repository directory, install dependencies if necessary:

```powershell
cd server
npm ci
cd ../client
npm ci
```

Keep the existing `server/.env`. For a fresh clone, create that file with your own Atlas URI:

```dotenv
MONGO_URI=your_mongodb_connection_string
```

Never commit the actual URI. `.env`, `node_modules`, and the client build directory are ignored by Git. The implementation does not change the existing connection string or DNS settings.

Start the backend in terminal 1:

```powershell
cd 'D:\大学\UAlbany\Courses\ICSI418Y Software Engineering\Assignment\3\ICSI418Y_PA3\server'
npm start
```

Start the frontend in terminal 2:

```powershell
cd 'D:\大学\UAlbany\Courses\ICSI418Y Software Engineering\Assignment\3\ICSI418Y_PA3\client'
npm run dev
```

Open the URL printed by Vite, normally http://localhost:5173. The API listens on http://localhost:9000. The browser uses this local API address, so both processes must be running on your computer.

The backend waits for Atlas and the membership uniqueness index before accepting HTTP requests. If startup fails, verify Atlas network access, credentials, and database permissions. It prints an error name/code without printing the URI. Existing duplicate membership pairs must be resolved before the unique index can be created; startup does not silently delete records.

## MongoDB relationships

Collection names are case-sensitive. `users` is the original PA2 collection; do not create a second `Users` collection.

```javascript
// Projects
{
  _id: ObjectId("..."),
  name: "Campus Event Planner",
  description: "Organize campus events.",
  status: "Planning",
  project_lead_id: ObjectId("...") // references users._id
}

// ProjectMemberships
{
  _id: ObjectId("..."),
  project_id: ObjectId("..."), // references Projects._id
  user_id: ObjectId("...")     // references users._id
}
```

Each membership is one user/project pair. A user in two projects has two membership documents. A project with three members has three membership documents. Names are display values, never relationship keys.

A compound unique index on `{ project_id: 1, user_id: 1 }` prevents duplicate pairs, including concurrent requests. Project creation uses a MongoDB transaction to insert the project and its lead membership together; a failed membership insert rolls back the project. The configured Atlas deployment supports this. A standalone local MongoDB server without transaction support is not a supported substitute.

JSON transports ObjectIds as strings. The API validates 24-character hexadecimal IDs and converts them to `ObjectId` before database operations. React matches returned ID strings to the public user list to show names.

## API

All responses are JSON. Errors contain `{ "message": "..." }`.

| Method | Path | Request body / result |
| --- | --- | --- |
| POST | `/signup` | `{ f_name, l_name, username, password }`; existing signup |
| POST | `/login` | `{ username, password }`; existing credential check |
| GET | `/api/users` | Array containing only `_id`, `f_name`, `l_name`, `username` |
| POST | `/api/projects` | `{ name, description, status, project_lead_id }`; returns `{ message, project }` |
| GET | `/api/projects` | Array of project documents |
| GET | `/api/projects/:projectId/members` | Array of membership documents |
| POST | `/api/projects/:projectId/members` | `{ user_id }`; returns `{ message, membership }` |
| DELETE | `/api/projects/:projectId/members/:userId` | Returns `{ message }` |

Expected status codes: `200` for reads/login/removal, `201` for creation, `400` for invalid input or lead removal, `401` for invalid login, `404` for missing records, `409` for duplicate membership/username, and `500` for database failures. Malformed JSON returns `400`; oversized JSON returns `413`.

The backend verifies required strings, exact status values, IDs, and referenced users/projects. The UI also checks required project fields, but server validation is authoritative. Public user responses whitelist safe fields and never return passwords.

## Verification

```powershell
# From server/
npm test
npm run test:integration

# From client/
npm run lint
npm run build
```

`npm test` runs without Atlas. `npm run test:integration` deliberately connects to the configured Atlas `pa2` database, creates uniquely named temporary fixtures, and removes only those fixtures afterward. It does not delete existing users/projects or drop collections. The membership index remains part of the application. See `TESTING.md` for coverage and manual UI checks.

## Video and study guide

See `WALKTHROUGH_ZH.md` for a beginner-friendly code walkthrough and a 3–5 minute recording sequence. Before recording, use Signup to prepare at least three users and create at least two projects. Automated tests clean their fixtures and do not leave demonstration accounts behind.

## Scope

As in PA2, login only checks credentials and displays feedback. The inherited password storage is unchanged, and this coursework application has no session-based access control. The project-management pages are accessible without a login session. Authentication hardening, project editing, project deletion, and lead reassignment are outside this assignment.

Review the changes before committing and pushing. No automatic GitHub push is part of this implementation.
