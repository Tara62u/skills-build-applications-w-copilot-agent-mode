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

- **Data tier first**: Define required fields and types for users (`name`, `email`, `fitnessGoal`), teams (`name`, `mascot`), activities (user/team references, `type`, `minutes`, `points`, `occurredAt`), and workouts (`focus`, `minutes`, `difficulty`). Default team members to an empty array. Seed stable demo records with upserts, preserve unrelated records, and remove a seeded user's membership from the previous team when the fixture is moved.
- **Logic tier second**: Keep Express routes thin and delegate validation and behavior to explicit service functions. Return the documented response shapes and deliberate 4xx errors for invalid input or missing records; validate IDs before database lookups so malformed references do not become cast-error 500s.
- **Activity tracking**: Validate the request fields and that both IDs are well-formed and refer to existing records. Store references, type, minutes, a server-generated occurrence date, and points computed by one deterministic service rule (initially one point per minute). Invalid requests must not create activities.
- **Team management**: Require a nonblank name, preserve the submitted name and mascot, and default members to an empty array. Resolve member references through the data layer and return created/listed teams in the API contract shape.
- **Leaderboard**: Aggregate activity points by team, sort totals descending, and assign sequential 1-based ranks. Keep aggregation in the logic tier rather than calculating it in the UI.
- **Workout suggestions**: Find the user by email and return a 4xx if there is no match. Prefer workouts matching recent activity focus, otherwise the user's fitness goal, and include a reason explaining the recommendation.
- **Presentation tier last**: Keep page components responsible for rendering and user interaction; use `frontend/src/api/client.ts` for API requests. Implement each capability using the shared client and expose accessible loading, success, empty, and error states. Keep API and persistence logic out of components, and do not add authentication.

Implement and verify one failing contract behavior at a time. Preserve the existing API response shapes and port/proxy configuration; do not broaden the work into authentication or unrelated features.

## Test plan

Run the focused contracts in dependency order, stopping to fix a tier before moving to its dependents. The data contract is the prerequisite for API behavior, and the API contract is the prerequisite for the presentation tier. A contract passes only when all of its assertions pass; after all contracts pass, both builds must succeed.

| Order | Tier/check | Command | What a failure indicates | Baseline before implementation |
| --- | --- | --- | --- |
| 1 | Data contract | `npm run test:data --prefix octofit-tracker/backend` | Schema constraints/defaults or seed behavior fail, including repeatability, preservation of unrelated records, and reconciliation of Mona's old team membership. | 4 failed, 1 passed: required User, Team, Activity, and Workout validation is missing; seed contract passed. |
| 2 | API contract | `npm run test:api --prefix octofit-tracker/backend` | Request validation, response shapes, activity creation/listing, team creation/listing, leaderboard totals/ranks, or user-specific suggestions violate the API contract. | 4 failed: invalid activity/team requests returned 501 instead of 400, leaderboard was empty, and Mona's first workout suggestion did not match her activity/goal. |
| 3 | Presentation contract | `npm test --prefix octofit-tracker/frontend` | A capability page does not render API data or an accessible loading, empty, or error state, or workout suggestions do not request the seeded user. | 15 failed: expected content and UI states were not found. |
| 4 | Backend build | `npm run build --prefix octofit-tracker/backend` | Backend TypeScript does not type-check. Run after the backend contracts pass. | Not run yet. |
| 5 | Frontend build | `npm run build --prefix octofit-tracker/frontend` | Frontend TypeScript or Vite production build fails. Run after the presentation contract passes. | Not run yet. |

For a failing test, make the smallest change in the tier that owns the behavior, rerun that tier's focused command, and then rerun downstream contracts as appropriate. The baseline column records the previously observed state; replace it with fresh results when the suites are run.
