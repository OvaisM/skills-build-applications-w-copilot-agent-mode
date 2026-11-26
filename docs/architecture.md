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

- Presentation tier decision: Use route components in `src/components`, the shared `src/api/client.ts` for URL construction and fetch errors, and Bootstrap markup with accessible loading (`role="status"`), error (`role="alert"`), empty, and success states.
- Logic tier decision: Keep Express routes thin and put validation and business behavior in explicit functions in `src/services/octofitService.ts`. Invalid input and missing records should return deliberate 4xx responses rather than reach Mongoose cast errors or 500 responses.
- Data tier decision: Put required fields and references in Mongoose schemas under `src/models`. Seed stable demo records with upserts, preserve unrelated data, and reconcile demo team memberships so the seed remains safe and idempotent.
- Activity tracking approach: Validate `userId` and `teamId` as ObjectIds and confirm both referenced records exist. Create the activity with the submitted `user`, `team`, `type`, and `minutes`, setting `points` from the elapsed minutes and `occurredAt` to the current date.
- Team management approach: Reject missing names with `400`, preserve the submitted `name` and `mascot`, and convert submitted member IDs to team/user references while defaulting missing members to an empty array.
- Leaderboard approach: Aggregate activity `points` by team, join each total to the team name, sort highest first, and assign sequential 1-based ranks.
- Workout suggestions approach: Decode and match the requested email to a user, inspect that user's recent activities and `fitnessGoal`, select workouts whose focus matches the strongest pattern or goal, and include a reason tied to that pattern. Return `404` for an unknown email.

## Test plan

Run each tier from the repository root in the order below. A failure identifies the lowest tier that still needs implementation.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
  - Verifies Mongoose required fields, references/defaulted members, workout shape, and safe/idempotent seed behavior.
  - Current result: 4 of 5 failures; user, team, activity, and workout validation is incomplete, while seed/idempotency already passes.
- API command: `npm run test:api --prefix octofit-tracker/backend`
  - Verifies the activity, team, leaderboard, and workout `/api/*` contracts, including deliberate 4xx responses.
  - Current result: 4 of 4 failures; activity/team writes still use intentional `501` placeholders, leaderboard is empty, and workout matching lacks a personalized reason.
- Frontend command: `npm test --prefix octofit-tracker/frontend`
  - Verifies routing, API rendering, and loading, empty, success, and accessible error states.
  - Current result: 15 of 15 failures; the starter capability card is replaced instead of rendering fetched capability data.
- Backend build command: `npm run build --prefix octofit-tracker/backend`
  - Verifies backend TypeScript and model/service type correctness.
  - Current result: passes.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`
  - Verifies React/TypeScript correctness and creates the Vite production bundle.
  - Current result: passes.
