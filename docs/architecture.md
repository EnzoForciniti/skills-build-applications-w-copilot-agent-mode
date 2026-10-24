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

- Presentation tier decision: Build the UI with React 19, Vite, React Router, and Bootstrap, and keep all API calls in a shared client so routes remain simple and the browser only sees relative `/api` requests with clear loading and error states.
- Logic tier decision: Keep the Express API thin but deterministic: route handlers validate request shape and IDs, delegate domain behavior to service functions, and return deliberate 4xx errors instead of leaking cast exceptions or database crashes to clients.
- Data tier decision: Use Mongoose schemas to enforce the domain model and defaults, and use idempotent upserts in the seed script so demo users, teams, activities, and workouts can be re-run without wiping app data.
- Activity tracking approach: Store each activity with a real `user`, `team`, `type`, and `minutes`, calculate `points` from the minutes/type pattern, and record an ISO `occurredAt` timestamp for leaderboard and recommendation logic.
- Team management approach: Require a team `name` and `mascot`, default `members` to an empty array, and update member membership with set-based inserts so seeded users and created teams stay consistent.
- Leaderboard approach: Aggregate team totals by summing activity points, sort descending, and assign a 1-based rank for deterministic ordering in the `GET /api/leaderboard` response.
- Workout suggestions approach: Match the requested user to their goal or most recent activity pattern, prioritize workouts in the same focus area, and include a human-readable `reason` explaining why the suggestion matches.

## Test plan

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` — validates schema requirements and seed idempotency before the HTTP layer is exercised.
- API command: `npm run test:api --prefix octofit-tracker/backend` — validates the `/api` contracts, including 4xx handling, team/member behavior, leaderboard ordering, and workout recommendation logic.
- Frontend command: `npm test --prefix octofit-tracker/frontend` — validates the React UI behavior once the API contracts are stable, especially loading, empty, success, and error states.
- Backend build command: `npm run build --prefix octofit-tracker/backend` — checks that the TypeScript backend and route/service contracts compile cleanly.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` — confirms the React app bundles successfully with the expected API client configuration.
