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

Record your plan here before editing code.

- Presentation tier decision: Keep route-specific rendering in React components and use the shared API client for all requests. Components own loading, empty, error, and success states; business rules stay in the API.
- Logic tier decision: Keep Express routes thin and delegate capability behavior to the service layer. Validate requests and map invalid input or missing records to deliberate 4xx responses rather than database or cast errors.
- Data tier decision: Put required fields, defaults, and references in Mongoose schemas. Keep seed data idempotent and non-destructive so it preserves unrelated records.
- Activity tracking approach: Validate the activity type, positive minutes, and well-formed IDs for existing users and teams before writing. Use minutes as the initial points value and assign `occurredAt` on the server.
- Team management approach: Require a non-empty name, preserve the submitted name and mascot, and store valid member IDs in the `members` array, defaulting it to empty when omitted.
- Leaderboard approach: Sum activity points by team, include teams with no activity at zero points, sort highest first with a stable tie-break, and assign ranks starting at 1.
- Workout suggestions approach: Find the user by email, prefer workouts matching their fitness goal and use recent activity as context, and include a concise reason. Return 4xx when no user matches.

## Test plan

Run the narrow contract first after changing its tier, then run both builds. A contract failure points to behavior in that tier; a build failure points to a TypeScript or bundling issue. The root exercise validator checks repository metadata and workflow safeguards, not OctoFit runtime behavior.

| Check | Command | A failure means | Baseline |
| --- | --- | --- | --- |
| Data contracts | `npm run test:data --prefix octofit-tracker/backend` | Model validation, defaults, references, or seed behavior violates the data contract. | 4 failed, 1 passed; the four model-validation tests fail, while seed idempotence passes. |
| API contracts | `npm run test:api --prefix octofit-tracker/backend` | A route/service does not meet its response, validation, or capability behavior contract. | 4 failed; activity/team creation, leaderboard, and workout suggestions fail. |
| Presentation contracts | `npm test --prefix octofit-tracker/frontend` | A route does not render API data or the required loading, empty, error, and accessible states. | 15 failed; routes still show the implementation-pending placeholder. |
| Backend build | `npm run build --prefix octofit-tracker/backend` | Backend TypeScript has a compile/type error. | Passed. |
| Frontend build | `npm run build --prefix octofit-tracker/frontend` | Frontend TypeScript or Vite production bundling fails. | Passed. |
| Exercise validation | `node --test tests/exercise-validation.test.mjs` | Repository exercise metadata, grading contracts, or workflow safeguards fail validation. | 36 passed, 4 failed; failures concern baseline metadata, toolkit checkout pinning, and block-scalar parsing. |
