# NCAA March Madness Platform - Issue Tracker

This document breaks down the MVP implementation into atomic tasks. Work through issues in order within each phase to maintain proper dependencies.

---

## Phase 1: Foundation & Setup

### Issue #1: Initialize TanStack Start Project
**Priority:** Critical
**Dependencies:** None

**Tasks:**
- [ ] Run `npm create @tanstack/start@latest`
- [ ] Configure TypeScript with strict mode
- [ ] Set up project structure (app, lib, components directories)
- [ ] Add `.gitignore` with node_modules, .env, dist
- [ ] Create initial README with setup instructions
- [ ] Verify dev server runs (`npm run dev`)

**Acceptance Criteria:**
- Dev server runs on localhost without errors
- TypeScript compiles with no warnings
- Can navigate to root route and see placeholder page

---

### Issue #2: Configure Tailwind CSS
**Priority:** High
**Dependencies:** #1

**Tasks:**
- [ ] Install Tailwind CSS dependencies
- [ ] Create `tailwind.config.js` with content paths
- [ ] Create `app/styles/globals.css` with Tailwind directives
- [ ] Import globals.css in root layout
- [ ] Test with a colored div to verify Tailwind works

**Acceptance Criteria:**
- Tailwind utility classes work in components
- Dev server rebuilds CSS on changes
- No console warnings about missing CSS

---

### Issue #3: Set Up Drizzle ORM
**Priority:** Critical
**Dependencies:** #1

**Tasks:**
- [ ] Install `drizzle-orm` and `drizzle-kit`
- [ ] Install `postgres` driver
- [ ] Create `drizzle.config.ts`
- [ ] Create `lib/db/index.ts` for database client
- [ ] Create `lib/db/schema.ts` (empty for now)
- [ ] Add `DATABASE_URL` to `.env.example`

**Acceptance Criteria:**
- Drizzle config file is valid
- Can import db client without errors
- `drizzle-kit` commands are available

---

### Issue #4: Set Up Supabase Database
**Priority:** Critical
**Dependencies:** None

**Tasks:**
- [ ] Create Supabase account
- [ ] Create new project (free tier)
- [ ] Copy connection string
- [ ] Add `DATABASE_URL` to `.env`
- [ ] Test connection with simple query

**Acceptance Criteria:**
- Supabase project is created
- Database connection string works
- Can connect from local environment

---

### Issue #5: Configure Authentication (Clerk)
**Priority:** High
**Dependencies:** #1

**Tasks:**
- [ ] Create Clerk account
- [ ] Create new application in Clerk dashboard
- [ ] Install `@clerk/tanstack-start`
- [ ] Add Clerk provider to root layout
- [ ] Add public/private keys to `.env`
- [ ] Create sign-in and sign-up routes
- [ ] Test sign-up flow

**Acceptance Criteria:**
- Users can sign up with email
- Users can sign in
- Protected routes redirect to sign-in
- User info accessible in app

---

### Issue #6: Create Database Schema - Users Table
**Priority:** High
**Dependencies:** #3, #4

**Tasks:**
- [ ] Define `users` table in `schema.ts`
- [ ] Add fields: id, email, name, createdAt
- [ ] Run `drizzle-kit generate` to create migration
- [ ] Run `drizzle-kit push` to apply to database
- [ ] Verify table exists in Supabase dashboard

**Acceptance Criteria:**
- Users table exists in database
- Schema matches specification in SYSTEM_DESIGN.md
- Can insert test user record

---

### Issue #7: Create Database Schema - Teams Table
**Priority:** High
**Dependencies:** #3, #4

**Tasks:**
- [ ] Define `teams` table in `schema.ts`
- [ ] Add fields: id, school, conference, seed, region, wins, losses, ppg, oppPpg, offensiveEfficiency, defensiveEfficiency, tempo, createdAt, updatedAt
- [ ] Generate and push migration
- [ ] Verify table exists in Supabase dashboard

**Acceptance Criteria:**
- Teams table exists with all fields
- Data types match specification
- Can insert test team record

---

### Issue #8: Create Database Schema - Games Table
**Priority:** High
**Dependencies:** #3, #4, #7

**Tasks:**
- [ ] Define `games` table in `schema.ts`
- [ ] Add fields: id, season, round, teamAId, teamBId, teamAScore, teamBScore, winnerId, teamAWinProbability, status, createdAt
- [ ] Add foreign key references to teams table
- [ ] Generate and push migration
- [ ] Verify table and relationships in Supabase

**Acceptance Criteria:**
- Games table exists with all fields
- Foreign keys properly reference teams
- Can insert test game record

---

### Issue #9: Create Database Schema - Brackets Table
**Priority:** High
**Dependencies:** #3, #4, #6

**Tasks:**
- [ ] Define `brackets` table in `schema.ts`
- [ ] Add fields: id, userId, season, name, picks (JSONB), pointsEarned, locked, lockedAt, createdAt, updatedAt
- [ ] Add foreign key reference to users table
- [ ] Generate and push migration
- [ ] Verify JSONB column works with test data

**Acceptance Criteria:**
- Brackets table exists with JSONB support
- Foreign key to users works
- Can insert bracket with JSON picks

---

### Issue #10: Create Basic Layout Component
**Priority:** Medium
**Dependencies:** #2, #5

**Tasks:**
- [ ] Create `components/Layout.tsx`
- [ ] Add header with app name and user menu
- [ ] Add navigation (Home, Teams, My Brackets)
- [ ] Add footer
- [ ] Style with Tailwind CSS
- [ ] Make responsive for mobile

**Acceptance Criteria:**
- Layout renders on all pages
- Navigation links work
- User menu shows sign-out button when logged in
- Mobile responsive (hamburger menu optional for MVP)

---

### Issue #11: Create Home Page
**Priority:** Medium
**Dependencies:** #1, #10

**Tasks:**
- [ ] Create `app/routes/index.tsx`
- [ ] Add hero section with app description
- [ ] Add "Create Bracket" CTA button
- [ ] Add "View Teams" secondary button
- [ ] Add basic styling

**Acceptance Criteria:**
- Home page renders at `/`
- CTA buttons navigate to correct routes
- Looks presentable (doesn't need to be fancy)

---

## Phase 2: Team Data

### Issue #12: Create Team Data Seed Script Structure
**Priority:** High
**Dependencies:** #7

**Tasks:**
- [ ] Create `scripts/seed-teams.ts`
- [ ] Set up script to read from CSV or JSON file
- [ ] Add database connection in script
- [ ] Add error handling and logging
- [ ] Add npm script: `"seed": "tsx scripts/seed-teams.ts"`

**Acceptance Criteria:**
- Script runs without errors
- Can parse team data from file
- Has rollback on error

---

### Issue #13: Collect 2024 Tournament Team Data
**Priority:** High
**Dependencies:** None

**Tasks:**
- [ ] Find NCAA 2024 tournament bracket
- [ ] Create `data/teams-2024.json` file
- [ ] Add all 64 teams with: school name, seed, region, conference
- [ ] Research and add basic stats: wins, losses, ppg, oppPpg
- [ ] Add efficiency ratings if available (KenPom or manual estimates)

**Acceptance Criteria:**
- All 64 teams have complete data
- JSON file is valid and parseable
- Stats are accurate (verified against source)

---

### Issue #14: Seed Teams into Database
**Priority:** High
**Dependencies:** #12, #13

**Tasks:**
- [ ] Update seed script to read from `teams-2024.json`
- [ ] Insert all teams into database
- [ ] Run seed script successfully
- [ ] Verify all 64 teams in Supabase dashboard
- [ ] Test querying teams from database

**Acceptance Criteria:**
- All 64 teams inserted successfully
- No duplicate IDs
- All required fields populated
- Can query teams by region, seed, etc.

---

### Issue #15: Create Tournament Bracket Structure Generator
**Priority:** High
**Dependencies:** #8, #14

**Tasks:**
- [ ] Create `scripts/generate-bracket.ts`
- [ ] Write function to generate Round of 64 matchups (1 vs 16, 2 vs 15, etc.)
- [ ] Generate Round of 32, Sweet 16, Elite 8, Final Four, Championship
- [ ] Assign unique game IDs
- [ ] Set proper round labels
- [ ] Set status to "scheduled"

**Acceptance Criteria:**
- Generates exactly 63 games
- Matchups follow correct seeding structure
- Each region has correct bracket structure
- Game IDs are unique and logical

---

### Issue #16: Seed Games into Database
**Priority:** High
**Dependencies:** #15

**Tasks:**
- [ ] Run bracket generation script
- [ ] Insert all 63 games into database
- [ ] Verify game structure in Supabase
- [ ] Test querying games by round
- [ ] Verify foreign key relationships work

**Acceptance Criteria:**
- All 63 tournament games exist
- Team references are correct
- Can query games by round, region
- Bracket structure is valid

---

### Issue #17: Implement Win Probability Calculation Function
**Priority:** High
**Dependencies:** #7

**Tasks:**
- [ ] Create `lib/predictions.ts`
- [ ] Implement `calculateWinProbability(teamA, teamB)` function
- [ ] Use seed differential factor (8% per seed)
- [ ] Use efficiency differential factor
- [ ] Use win percentage differential factor
- [ ] Apply logistic function and clamp to 15%-85%

**Acceptance Criteria:**
- Function returns number between 0.15 and 0.85
- Higher seeds have higher probabilities
- Better efficiency ratings increase probability
- Matches formula in SYSTEM_DESIGN.md

---

### Issue #18: Write Unit Tests for Prediction Function
**Priority:** Medium
**Dependencies:** #17

**Tasks:**
- [ ] Set up testing framework (Vitest)
- [ ] Create `lib/predictions.test.ts`
- [ ] Test: 1-seed vs 16-seed returns >0.85
- [ ] Test: Equal teams return ~0.50
- [ ] Test: Efficiency advantage increases probability
- [ ] Test: Output always between 0.15 and 0.85

**Acceptance Criteria:**
- All tests pass
- Edge cases covered
- Tests run with `npm test`

---

### Issue #19: Create API Route - Get All Teams
**Priority:** High
**Dependencies:** #14

**Tasks:**
- [ ] Create `app/routes/api/teams/index.ts`
- [ ] Implement GET handler
- [ ] Query all teams from database
- [ ] Sort by region and seed
- [ ] Return JSON response
- [ ] Add error handling

**Acceptance Criteria:**
- GET `/api/teams` returns all 64 teams
- Response is valid JSON
- Teams sorted logically
- Returns 200 status code

---

### Issue #20: Create API Route - Get Single Team
**Priority:** High
**Dependencies:** #14

**Tasks:**
- [ ] Create `app/routes/api/teams/$id.ts`
- [ ] Implement GET handler with ID parameter
- [ ] Query single team from database
- [ ] Return 404 if not found
- [ ] Return team JSON on success

**Acceptance Criteria:**
- GET `/api/teams/duke-2024` returns team data
- Returns 404 for invalid ID
- Response includes all team fields

---

### Issue #21: Create Team List Page
**Priority:** High
**Dependencies:** #19

**Tasks:**
- [ ] Create `app/routes/teams/index.tsx`
- [ ] Fetch teams from API using TanStack Query
- [ ] Display teams grouped by region
- [ ] Show seed, school name, record
- [ ] Make team names clickable links
- [ ] Style with Tailwind CSS

**Acceptance Criteria:**
- Page shows all 64 teams
- Grouped by 4 regions
- Teams link to detail pages
- Loading state shows while fetching
- Responsive on mobile

---

### Issue #22: Create Team Detail Page
**Priority:** High
**Dependencies:** #20

**Tasks:**
- [ ] Create `app/routes/teams/$id.tsx`
- [ ] Fetch single team from API
- [ ] Display team info card (name, seed, region, conference)
- [ ] Display stats (record, PPG, opp PPG, efficiency)
- [ ] Add simple stat visualizations (progress bars or numbers)
- [ ] Style with Tailwind CSS

**Acceptance Criteria:**
- Shows all team details
- Stats are clearly formatted
- Page is responsive
- Shows 404 for invalid team ID
- Links back to team list

---

### Issue #23: Create API Route - Get Game Prediction
**Priority:** High
**Dependencies:** #17, #16

**Tasks:**
- [ ] Create `app/routes/api/games/$id/prediction.ts`
- [ ] Get game by ID from database
- [ ] Get both teams' data
- [ ] Calculate win probability using prediction function
- [ ] Return JSON with teamAWinProbability and teamBWinProbability

**Acceptance Criteria:**
- Returns prediction for valid game ID
- Probabilities sum to 1.0
- Uses calculation function correctly
- Returns 404 for invalid game

---

### Issue #24: Compute and Store All Game Predictions
**Priority:** Medium
**Dependencies:** #23

**Tasks:**
- [ ] Create script `scripts/compute-predictions.ts`
- [ ] Loop through all 63 games
- [ ] Calculate win probability for each
- [ ] Update game record with teamAWinProbability
- [ ] Run script and verify in database

**Acceptance Criteria:**
- All games have win probabilities
- Probabilities are reasonable
- Can re-run script to update predictions
- Logs progress and completion

---

## Phase 3: Bracket Builder

### Issue #25: Create Bracket Data Types
**Priority:** High
**Dependencies:** None

**Tasks:**
- [ ] Create `lib/types.ts`
- [ ] Define `Team` interface
- [ ] Define `Game` interface
- [ ] Define `BracketPicks` type
- [ ] Define `Bracket` interface
- [ ] Export all types

**Acceptance Criteria:**
- Types match database schema
- Can import types throughout app
- TypeScript compilation succeeds

---

### Issue #26: Create API Route - Get All Games
**Priority:** High
**Dependencies:** #16

**Tasks:**
- [ ] Create `app/routes/api/games/index.ts`
- [ ] Implement GET handler
- [ ] Query all tournament games
- [ ] Include team data (join or separate queries)
- [ ] Return games sorted by round
- [ ] Add error handling

**Acceptance Criteria:**
- Returns all 63 games
- Each game includes team info
- Sorted by round order
- Response is valid JSON

---

### Issue #27: Create Bracket Builder UI - Basic Structure
**Priority:** High
**Dependencies:** #25, #26

**Tasks:**
- [ ] Create `components/BracketBuilder.tsx`
- [ ] Fetch all games from API
- [ ] Create basic grid layout for 4 regions
- [ ] Render placeholder for each region
- [ ] Add Final Four section
- [ ] Add Championship section

**Acceptance Criteria:**
- Component renders without errors
- Shows 4 region placeholders
- Shows Final Four placeholder
- Shows Championship placeholder
- Layout is responsive (can be basic for MVP)

---

### Issue #28: Create Matchup Component
**Priority:** High
**Dependencies:** #25

**Tasks:**
- [ ] Create `components/Matchup.tsx`
- [ ] Accept game prop with two teams
- [ ] Display both teams with seeds
- [ ] Make teams clickable
- [ ] Show selected state visually
- [ ] Add hover states

**Acceptance Criteria:**
- Displays team names and seeds
- Click handler works
- Visual feedback on hover and selection
- Styled with Tailwind CSS

---

### Issue #29: Implement Bracket Pick State Management
**Priority:** High
**Dependencies:** #25

**Tasks:**
- [ ] Create `lib/bracket-state.ts`
- [ ] Set up Zustand store for bracket picks
- [ ] Create actions: pickWinner, resetBracket, loadBracket
- [ ] Store picks as { [gameId]: teamId }
- [ ] Create selector to get pick for game

**Acceptance Criteria:**
- Can pick winners for games
- State persists across re-renders
- Can reset bracket
- Can load saved bracket

---

### Issue #30: Implement Pick Propagation Logic
**Priority:** High
**Dependencies:** #29

**Tasks:**
- [ ] Create function to find next game for winner
- [ ] When team picked, auto-populate next round
- [ ] Clear downstream picks if earlier pick changes
- [ ] Update bracket state accordingly

**Acceptance Criteria:**
- Picking Round of 64 winner populates Round of 32
- Changing pick clears future rounds
- Logic handles all rounds correctly
- Final Four and Championship work

---

### Issue #31: Create Region Component
**Priority:** High
**Dependencies:** #28, #29

**Tasks:**
- [ ] Create `components/Region.tsx`
- [ ] Accept region name and games for that region
- [ ] Render rounds vertically (Round of 64, 32, Sweet 16, Elite 8)
- [ ] Use Matchup component for each game
- [ ] Connect to bracket state
- [ ] Style bracket lines/connections (optional, can be simple)

**Acceptance Criteria:**
- Shows all games in region
- Organized by round
- Picks work and propagate
- Visually clear structure

---

### Issue #32: Integrate Regions into Bracket Builder
**Priority:** High
**Dependencies:** #31

**Tasks:**
- [ ] Import Region component in BracketBuilder
- [ ] Render 4 Region components (South, East, West, Midwest)
- [ ] Filter games by region for each
- [ ] Pass games to respective Region
- [ ] Arrange in 2x2 grid or horizontal layout

**Acceptance Criteria:**
- All 4 regions render
- Each shows correct games
- Layout is organized
- All 64 teams visible

---

### Issue #33: Create Final Four Component
**Priority:** High
**Dependencies:** #28, #29

**Tasks:**
- [ ] Create `components/FinalFour.tsx`
- [ ] Show 2 matchups (region champions)
- [ ] Pull winners from Elite 8 games
- [ ] Use Matchup component
- [ ] Connect to bracket state

**Acceptance Criteria:**
- Shows Final Four matchups
- Teams auto-populate from Elite 8 picks
- Can pick Final Four winners
- Winners advance to Championship

---

### Issue #34: Create Championship Component
**Priority:** High
**Dependencies:** #28, #29

**Tasks:**
- [ ] Create `components/Championship.tsx`
- [ ] Show final matchup
- [ ] Pull winners from Final Four
- [ ] Use Matchup component
- [ ] Allow picking champion

**Acceptance Criteria:**
- Shows championship game
- Teams populate from Final Four
- Can pick champion
- Visual indication of final pick

---

### Issue #35: Create Bracket Builder Page
**Priority:** High
**Dependencies:** #32, #33, #34

**Tasks:**
- [ ] Create `app/routes/bracket/new.tsx`
- [ ] Import BracketBuilder component
- [ ] Add page title and instructions
- [ ] Add "Save Draft" button
- [ ] Add "Lock Bracket" button
- [ ] Add progress indicator (X/63 picks made)

**Acceptance Criteria:**
- Page renders full bracket
- Can fill out entire bracket
- Progress shows correct count
- Buttons are styled and positioned

---

### Issue #36: Create API Route - Create Bracket
**Priority:** High
**Dependencies:** #9

**Tasks:**
- [ ] Create `app/routes/api/brackets/index.ts`
- [ ] Implement POST handler
- [ ] Validate user is authenticated
- [ ] Accept bracket name and picks
- [ ] Insert bracket into database
- [ ] Return created bracket with ID

**Acceptance Criteria:**
- Authenticated users can create bracket
- Unauthenticated users get 401
- Bracket saved to database
- Returns bracket ID
- Validates required fields

---

### Issue #37: Create API Route - Update Bracket
**Priority:** High
**Dependencies:** #9

**Tasks:**
- [ ] Create `app/routes/api/brackets/$id.ts`
- [ ] Implement PUT handler
- [ ] Verify user owns bracket
- [ ] Check bracket is not locked
- [ ] Update picks in database
- [ ] Return updated bracket

**Acceptance Criteria:**
- Can update own bracket
- Cannot update others' brackets
- Cannot update locked brackets
- Returns 403 for permission errors
- Returns updated data

---

### Issue #38: Implement Save Draft Functionality
**Priority:** High
**Dependencies:** #36, #37

**Tasks:**
- [ ] Add save handler to Bracket Builder page
- [ ] Show name input modal/dialog
- [ ] Call create or update API based on state
- [ ] Show success/error messages
- [ ] Update URL with bracket ID after save

**Acceptance Criteria:**
- Can save new bracket
- Can update existing bracket
- Success message appears
- Errors handled gracefully
- Bracket ID in URL after save

---

### Issue #39: Create API Route - Get User's Brackets
**Priority:** High
**Dependencies:** #9

**Tasks:**
- [ ] Implement GET `/api/brackets`
- [ ] Filter by authenticated user ID
- [ ] Return array of user's brackets
- [ ] Sort by most recent first
- [ ] Include pick count in response

**Acceptance Criteria:**
- Returns only user's brackets
- Requires authentication
- Sorted by date
- Includes metadata (name, locked, points)

---

### Issue #40: Create My Brackets Page
**Priority:** High
**Dependencies:** #39

**Tasks:**
- [ ] Create `app/routes/my-brackets.tsx`
- [ ] Fetch user's brackets from API
- [ ] Display list of brackets
- [ ] Show name, created date, points, locked status
- [ ] Add "Edit" and "Delete" buttons
- [ ] Add "Create New" button

**Acceptance Criteria:**
- Shows all user's brackets
- Can navigate to edit bracket
- Delete button works (with confirmation)
- "Create New" navigates to builder
- Shows empty state if no brackets

---

### Issue #41: Implement Load Bracket Functionality
**Priority:** High
**Dependencies:** #29, #37

**Tasks:**
- [ ] Create `app/routes/bracket/$id.tsx`
- [ ] Fetch bracket by ID from API
- [ ] Load picks into bracket state
- [ ] Render BracketBuilder with loaded state
- [ ] Disable editing if locked
- [ ] Show "Locked" badge if applicable

**Acceptance Criteria:**
- Bracket loads with saved picks
- Locked brackets are read-only
- Save updates existing bracket
- Shows loading state while fetching

---

## Phase 4: Scoring & Polish

### Issue #42: Create Scoring Utility Function
**Priority:** High
**Dependencies:** #25

**Tasks:**
- [ ] Create `lib/scoring.ts`
- [ ] Implement `calculateBracketScore(picks, results)` function
- [ ] Define ROUND_POINTS constants (1, 2, 4, 8, 16, 32)
- [ ] Loop through results and add points for correct picks
- [ ] Return total score

**Acceptance Criteria:**
- Function returns correct score
- Handles missing picks gracefully
- Follows standard scoring system
- Well-tested with unit tests

---

### Issue #43: Write Unit Tests for Scoring Function
**Priority:** Medium
**Dependencies:** #42

**Tasks:**
- [ ] Create `lib/scoring.test.ts`
- [ ] Test: Correct Round of 64 pick = 1 point
- [ ] Test: Correct Championship pick = 32 points
- [ ] Test: Wrong picks = 0 points
- [ ] Test: Partial bracket scoring
- [ ] Test: Edge cases (empty picks, etc.)

**Acceptance Criteria:**
- All tests pass
- Coverage of scoring logic
- Tests run in CI

---

### Issue #44: Create API Route - Lock Bracket
**Priority:** High
**Dependencies:** #9

**Tasks:**
- [ ] Create `app/routes/api/brackets/$id/lock.ts`
- [ ] Implement POST handler
- [ ] Verify user owns bracket
- [ ] Check bracket is complete (63 picks)
- [ ] Set locked = true, lockedAt = now
- [ ] Return updated bracket

**Acceptance Criteria:**
- Can lock own bracket
- Cannot lock incomplete bracket
- Cannot unlock once locked
- Returns error if already locked
- Updates database correctly

---

### Issue #45: Implement Lock Bracket UI
**Priority:** High
**Dependencies:** #44

**Tasks:**
- [ ] Add lock button to Bracket Builder
- [ ] Show confirmation modal with warning
- [ ] Disable if picks incomplete
- [ ] Call lock API endpoint
- [ ] Update UI to read-only after lock
- [ ] Show success message

**Acceptance Criteria:**
- Lock button visible on unlocked brackets
- Confirmation modal appears
- Warning explains action is irreversible
- Button disabled if incomplete
- UI updates after lock

---

### Issue #46: Create Admin Authentication
**Priority:** Medium
**Dependencies:** #5

**Tasks:**
- [ ] Add admin role to user metadata in Clerk
- [ ] Create `lib/auth.ts` with `isAdmin()` function
- [ ] Create admin middleware for API routes
- [ ] Test with admin and non-admin users

**Acceptance Criteria:**
- Can identify admin users
- Admin middleware blocks non-admins
- Returns 403 for unauthorized access
- Works across all admin routes

---

### Issue #47: Create API Route - Update Game Result
**Priority:** High
**Dependencies:** #8, #46

**Tasks:**
- [ ] Create `app/routes/api/admin/games/$id/result.ts`
- [ ] Implement POST handler (admin only)
- [ ] Accept winnerId and scores
- [ ] Update game in database
- [ ] Set status to "final"
- [ ] Return updated game

**Acceptance Criteria:**
- Admin can update game results
- Non-admins get 403
- Validates winnerId is one of the teams
- Updates database correctly
- Returns updated data

---

### Issue #48: Create API Route - Score All Brackets
**Priority:** High
**Dependencies:** #42, #46

**Tasks:**
- [ ] Create `app/routes/api/admin/score-brackets.ts`
- [ ] Implement POST handler (admin only)
- [ ] Fetch all locked brackets
- [ ] Fetch all completed games
- [ ] Calculate score for each bracket
- [ ] Update pointsEarned in database
- [ ] Return summary (X brackets scored)

**Acceptance Criteria:**
- Admin can trigger scoring
- All locked brackets get scored
- Scores are accurate
- Database updates successfully
- Returns count of scored brackets

---

### Issue #49: Create Admin Dashboard Page
**Priority:** Medium
**Dependencies:** #46, #47, #48

**Tasks:**
- [ ] Create `app/routes/admin/index.tsx`
- [ ] Require admin authentication
- [ ] Show list of games grouped by round
- [ ] Add form to update each game result
- [ ] Add "Score All Brackets" button
- [ ] Show success/error messages

**Acceptance Criteria:**
- Only admins can access
- Can update any game result
- Can trigger bracket scoring
- Shows feedback on actions
- Simple, functional UI (doesn't need polish)

---

### Issue #50: Display Correct/Incorrect Picks on Bracket
**Priority:** Medium
**Dependencies:** #42

**Tasks:**
- [ ] Update Matchup component
- [ ] Check if game is complete
- [ ] If complete, show if user's pick was correct
- [ ] Add green checkmark for correct
- [ ] Add red X for incorrect
- [ ] Gray out losing teams

**Acceptance Criteria:**
- Correct picks show green indicator
- Incorrect picks show red indicator
- Only shows on completed games
- Visual feedback is clear
- Works across all rounds

---

### Issue #51: Add Progress Indicator to Bracket Builder
**Priority:** Low
**Dependencies:** #35

**Tasks:**
- [ ] Calculate picks made / 63 total
- [ ] Display progress bar or counter
- [ ] Update in real-time as picks made
- [ ] Show different color when complete
- [ ] Add to top of Bracket Builder

**Acceptance Criteria:**
- Shows "X/63 picks made"
- Updates when picks change
- Visual indicator for completion
- Positioned prominently

---

### Issue #52: Add Win Probability Display to Matchups
**Priority:** Low
**Dependencies:** #24, #28

**Tasks:**
- [ ] Fetch game predictions
- [ ] Display win probability next to each team
- [ ] Show as percentage (e.g., "72%")
- [ ] Style smaller/subtle
- [ ] Show on hover or always visible

**Acceptance Criteria:**
- Probabilities display correctly
- Percentages sum to 100%
- Styled to not clutter UI
- Works across all matchups

---

### Issue #53: Improve Bracket Builder Mobile Responsiveness
**Priority:** Medium
**Dependencies:** #32

**Tasks:**
- [ ] Test bracket on mobile viewport
- [ ] Adjust layout for small screens
- [ ] Make regions stack vertically on mobile
- [ ] Ensure tap targets are large enough
- [ ] Test on real mobile device

**Acceptance Criteria:**
- Bracket usable on mobile (320px width)
- No horizontal scroll
- All games accessible
- Tap targets meet accessibility standards
- Tested on iOS and Android

---

### Issue #54: Add Loading States Throughout App
**Priority:** Low
**Dependencies:** All API routes

**Tasks:**
- [ ] Add loading spinners to all data fetches
- [ ] Use Suspense where applicable
- [ ] Create reusable Loading component
- [ ] Show skeleton loaders for lists
- [ ] Test with throttled network

**Acceptance Criteria:**
- No blank screens while loading
- Loading indicators on all async operations
- Smooth user experience
- Works with slow connections

---

### Issue #55: Add Error Handling Throughout App
**Priority:** Medium
**Dependencies:** All API routes

**Tasks:**
- [ ] Add error boundaries
- [ ] Show error messages for failed API calls
- [ ] Create reusable Error component
- [ ] Add retry buttons where appropriate
- [ ] Log errors to console (or Sentry)

**Acceptance Criteria:**
- App doesn't crash on errors
- User sees helpful error messages
- Can recover from errors
- Errors are logged for debugging

---

### Issue #56: Create API Route - Delete Bracket
**Priority:** Low
**Dependencies:** #9

**Tasks:**
- [ ] Implement DELETE `/api/brackets/:id`
- [ ] Verify user owns bracket
- [ ] Prevent deleting locked brackets
- [ ] Delete from database
- [ ] Return success response

**Acceptance Criteria:**
- Can delete own unlocked brackets
- Cannot delete others' brackets
- Cannot delete locked brackets
- Returns appropriate status codes

---

### Issue #57: Implement Delete Bracket UI
**Priority:** Low
**Dependencies:** #56

**Tasks:**
- [ ] Add delete button to My Brackets page
- [ ] Show confirmation modal
- [ ] Disable for locked brackets
- [ ] Call delete API
- [ ] Remove from UI on success

**Acceptance Criteria:**
- Delete button appears on brackets
- Confirmation prevents accidents
- Locked brackets can't be deleted
- UI updates after deletion

---

### Issue #58: Style Home Page
**Priority:** Low
**Dependencies:** #11

**Tasks:**
- [ ] Improve hero section design
- [ ] Add tournament imagery or graphics
- [ ] Style CTA buttons prominently
- [ ] Add "How It Works" section
- [ ] Make fully responsive

**Acceptance Criteria:**
- Home page looks polished
- CTAs are prominent
- Responsive on all devices
- Follows design system

---

### Issue #59: Add Navigation Active States
**Priority:** Low
**Dependencies:** #10

**Tasks:**
- [ ] Detect current route
- [ ] Highlight active nav item
- [ ] Add underline or background color
- [ ] Update on route change

**Acceptance Criteria:**
- Current page highlighted in nav
- Works on all routes
- Visual feedback is clear

---

### Issue #60: Create 404 Not Found Page
**Priority:** Low
**Dependencies:** #1

**Tasks:**
- [ ] Create custom 404 component
- [ ] Add helpful message
- [ ] Add link back to home
- [ ] Style with Tailwind

**Acceptance Criteria:**
- Shows for invalid routes
- User can navigate back
- Styled consistently

---

### Issue #61: Add Meta Tags for SEO
**Priority:** Low
**Dependencies:** #1

**Tasks:**
- [ ] Add title tags to all pages
- [ ] Add meta descriptions
- [ ] Add Open Graph tags
- [ ] Add favicon
- [ ] Test with social media preview tools

**Acceptance Criteria:**
- All pages have unique titles
- Meta descriptions present
- Previews look good when shared
- Favicon displays

---

### Issue #62: Write End-to-End Tests
**Priority:** Low
**Dependencies:** #41

**Tasks:**
- [ ] Set up Playwright or Cypress
- [ ] Test: Sign up → Create bracket → Save → Load
- [ ] Test: Fill complete bracket → Lock
- [ ] Test: View teams → View team detail
- [ ] Run tests in CI

**Acceptance Criteria:**
- Critical user flows covered
- Tests pass consistently
- Run in CI pipeline
- Documentation for running tests

---

### Issue #63: Create Deployment Documentation
**Priority:** High
**Dependencies:** All implementation issues

**Tasks:**
- [ ] Document environment variables needed
- [ ] Create deployment checklist
- [ ] Document database migration process
- [ ] Document seeding process
- [ ] Add troubleshooting section

**Acceptance Criteria:**
- Team can deploy from docs alone
- All env vars documented
- Migration steps clear
- Common issues covered

---

### Issue #64: Deploy to Production
**Priority:** Critical
**Dependencies:** All implementation issues

**Tasks:**
- [ ] Create Vercel project
- [ ] Connect GitHub repository
- [ ] Set environment variables in Vercel
- [ ] Run database migrations on production DB
- [ ] Run seed scripts
- [ ] Deploy and test
- [ ] Verify all features work in production

**Acceptance Criteria:**
- App is live at production URL
- All features work
- Database is seeded
- Authentication works
- No console errors

---

## Post-MVP Issues (Future)

### Enhancement Track 1: Machine Learning
- [ ] Issue #65: Collect historical tournament data (2015-2023)
- [ ] Issue #66: Set up Python ML environment
- [ ] Issue #67: Train logistic regression model
- [ ] Issue #68: Train XGBoost model
- [ ] Issue #69: Create ML prediction API endpoint
- [ ] Issue #70: Add "ML Picks" toggle to bracket builder

### Enhancement Track 2: Live Tracking
- [ ] Issue #71: Research live score APIs
- [ ] Issue #72: Set up WebSocket server
- [ ] Issue #73: Implement score polling service
- [ ] Issue #74: Create live score display component
- [ ] Issue #75: Auto-update brackets when games complete

### Enhancement Track 3: Pool Features
- [ ] Issue #76: Design pool data model
- [ ] Issue #77: Create pool CRUD APIs
- [ ] Issue #78: Build pool creation UI
- [ ] Issue #79: Implement invite system
- [ ] Issue #80: Create pool leaderboard
- [ ] Issue #81: Add different scoring systems

### Enhancement Track 4: Advanced Analytics
- [ ] Issue #82: Implement Monte Carlo simulation
- [ ] Issue #83: Create team comparison tool
- [ ] Issue #84: Add advanced stat visualizations
- [ ] Issue #85: Build historical analysis dashboard

### Enhancement Track 5: Social Features
- [ ] Issue #86: Add public bracket sharing
- [ ] Issue #87: Implement bracket comparison
- [ ] Issue #88: Integrate expert picks APIs
- [ ] Issue #89: Add bracket export (PDF/image)
- [ ] Issue #90: Create social sharing features

---

## Issue Workflow

### Labels
- `priority: critical` - Blocking, must be done
- `priority: high` - Important for MVP
- `priority: medium` - Nice to have for MVP
- `priority: low` - Polish, can defer
- `phase: 1` - Foundation
- `phase: 2` - Team Data
- `phase: 3` - Bracket Builder
- `phase: 4` - Scoring & Polish
- `type: feature` - New functionality
- `type: bug` - Fix broken functionality
- `type: docs` - Documentation
- `type: test` - Testing

### Issue States
- `todo` - Not started
- `in-progress` - Being worked on
- `review` - Ready for code review
- `done` - Completed and merged

### Work Tips
1. **Start with dependencies** - Check "Dependencies" field before starting
2. **Keep PRs small** - One issue per PR ideally
3. **Test thoroughly** - Check acceptance criteria before marking done
4. **Update this doc** - Check off tasks as completed
5. **Ask questions** - If acceptance criteria unclear, discuss with team

---

**Last Updated:** 2026-03-08
**Total MVP Issues:** 64
**Estimated MVP Completion:** All Phase 1-4 issues completed
