# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- **Presentation tier**: Keep page components responsible for rendering and user interaction; use `frontend/src/api/client.ts` for API requests. Show accessible loading, success, empty, and error states. Keep API and persistence logic out of components, and do not add authentication.
- **Logic tier**: Keep Express routes thin and delegate validation and behavior to explicit service functions. Return the documented response shapes and deliberate 4xx errors for invalid input or missing records; do not expose database cast errors as 500s.
- **Data tier**: Enforce required fields, types, and defaults in Mongoose schemas. Keep demo seed data idempotent with upserts; preserve unrelated records and reconcile a seeded user's old team membership when changing teams.
- **Activity tracking**: Validate the request fields and that both IDs are well-formed and refer to existing records. Store references, type, minutes, a server-generated occurrence date, and points computed by a single deterministic service rule (initially one point per minute).
- **Team management**: Require a nonblank name, preserve the submitted name and mascot, and default members to an empty array. Resolve member references through the data layer and return the created/listed teams in the API contract shape.
- **Leaderboard**: Aggregate activity points by team, sort totals descending, and assign sequential 1-based ranks. Keep aggregation in the logic tier rather than calculating it in the UI.
- **Workout suggestions**: Find the user by email; return a 4xx if there is no match. Prefer workouts matching the user's recent activity focus, otherwise their fitness goal, and include a reason explaining the recommendation.

## Test plan

Run the focused contracts in dependency order, then run both builds. A failure in a tier's contract points to a violation in that tier or its documented boundary; a build failure indicates a TypeScript or bundling error.

| Tier/check | Command | What a failure indicates | Last observed result |
| --- | --- | --- | --- |
| Data | `npm run test:data --prefix octofit-tracker/backend` | Schema constraints/defaults or idempotent seed behavior do not meet the data contract. | 4 failed, 1 passed: required User, Team, Activity, and Workout validation is missing; seed contract passed. |
| API | `npm run test:api --prefix octofit-tracker/backend` | Route/service validation or business behavior does not meet the API contract. | 4 failed: invalid activity/team requests returned 501 instead of 400, leaderboard was empty, and Mona's first workout suggestion did not match her activity/goal. |
| Presentation | `npm test --prefix octofit-tracker/frontend` | A page does not render API data or its accessible loading, empty, or error states as required. | 15 failed: expected content and UI states were not found. |
| Backend build | `npm run build --prefix octofit-tracker/backend` | Backend TypeScript does not type-check. | Not run yet. |
| Frontend build | `npm run build --prefix octofit-tracker/frontend` | Frontend TypeScript or Vite production build fails. | Not run yet. |
