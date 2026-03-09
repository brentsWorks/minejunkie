# NCAA March Madness Analytics Platform - System Design

## Project Overview

**Objective:** Build a web application that helps users create informed NCAA tournament brackets with data-driven team statistics and simple predictions.

**MVP Core Value:** Users can view team stats, see basic matchup predictions, and fill out a tournament bracket.

**Post-MVP Enhancements:** ML models, live tracking, pools, advanced analytics, simulations.

---

## MVP Scope

### What We're Building (MVP)
1. **Bracket Builder UI** - Interactive 64-team bracket editor
2. **Team Stats Pages** - Basic stats (record, PPG, efficiency ratings)
3. **Simple Predictions** - Seed-based win probabilities with basic adjustments
4. **User Accounts** - Save and manage brackets
5. **Bracket Scoring** - Manual score updates as tournament progresses

### What We're NOT Building Yet
- ❌ Complex ML ensemble models
- ❌ Monte Carlo simulations
- ❌ Live score tracking (WebSocket)
- ❌ Pool features and leaderboards
- ❌ Advanced analytics dashboards
- ❌ Real-time updates

---

## Simplified Architecture

```
┌────────────────────────────────────────────┐
│      Frontend (TanStack Start)             │
│                                            │
│  ┌──────────────┐  ┌──────────────┐       │
│  │   Bracket    │  │  Team Stats  │       │
│  │   Builder    │  │    Pages     │       │
│  └──────────────┘  └──────────────┘       │
└──────────────┬─────────────────────────────┘
               │ REST API
┌──────────────┴─────────────────────────────┐
│     Backend (TanStack Start SSR)           │
│                                            │
│  ┌─────────────────────────────────────┐   │
│  │        API Routes                   │   │
│  │  /api/teams, /api/brackets, etc.   │   │
│  └────────────┬────────────────────────┘   │
│               │                             │
│  ┌────────────┴────────────────────────┐   │
│  │      Simple Prediction Engine       │   │
│  │   (Seed-based + basic stat adj)     │   │
│  └────────────┬────────────────────────┘   │
└───────────────┼─────────────────────────────┘
                │
       ┌────────┴────────┐
       │   PostgreSQL    │
       │  (via Drizzle)  │
       └─────────────────┘
```

**Key Simplifications:**
- No separate Python ML service initially
- No Redis cache layer
- No WebSocket server
- Single TanStack Start app handles everything
- Predictions calculated in TypeScript (simple formulas)

---

## Tech Stack (MVP)

### Full Stack
- **Framework:** TanStack Start (React + SSR)
- **Language:** TypeScript
- **Styling:** Tailwind CSS
- **UI Components:** shadcn/ui (or headless UI)
- **ORM:** Drizzle ORM
- **Database:** PostgreSQL (Supabase free tier)
- **Auth:** Clerk or NextAuth

### Deployment
- **Hosting:** Vercel (frontend + API routes)
- **Database:** Supabase (free tier)
- **Monitoring:** Vercel Analytics + Sentry (optional)

**Why This Stack:**
- All TypeScript (no Python initially)
- Free tier hosting (Vercel + Supabase)
- Fast iteration, minimal setup
- Easy to add complexity later

---

## Database Schema (MVP)

### Core Tables

```typescript
// schema.ts (Drizzle)

export const teams = pgTable('teams', {
  id: text('id').primaryKey(),              // "duke-2024"
  school: text('school').notNull(),         // "Duke"
  conference: text('conference').notNull(), // "ACC"
  seed: integer('seed'),                    // 1-16
  region: text('region'),                   // "South"

  // Basic stats
  wins: integer('wins').notNull(),
  losses: integer('losses').notNull(),
  ppg: real('ppg'),                         // Points per game
  oppPpg: real('opp_ppg'),                  // Opponent PPG

  // Efficiency (from external source like KenPom)
  offensiveEfficiency: real('offensive_efficiency'),
  defensiveEfficiency: real('defensive_efficiency'),
  tempo: real('tempo'),

  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});

export const games = pgTable('games', {
  id: text('id').primaryKey(),
  season: integer('season').notNull(),
  round: text('round').notNull(),           // "Round of 64", "Round of 32", etc.

  teamAId: text('team_a_id').references(() => teams.id),
  teamBId: text('team_b_id').references(() => teams.id),

  // Scores (null if not played yet)
  teamAScore: integer('team_a_score'),
  teamBScore: integer('team_b_score'),
  winnerId: text('winner_id'),

  // Simple prediction
  teamAWinProbability: real('team_a_win_prob'),

  status: text('status').notNull(),         // "scheduled" | "final"
  createdAt: timestamp('created_at').defaultNow(),
});

export const users = pgTable('users', {
  id: text('id').primaryKey(),
  email: text('email').notNull().unique(),
  name: text('name'),
  createdAt: timestamp('created_at').defaultNow(),
});

export const brackets = pgTable('brackets', {
  id: text('id').primaryKey(),
  userId: text('user_id').references(() => users.id).notNull(),
  season: integer('season').notNull(),
  name: text('name').notNull(),

  // Store picks as JSONB
  picks: jsonb('picks').notNull(),          // { gameId: teamId }

  pointsEarned: integer('points_earned').default(0),
  locked: boolean('locked').default(false),
  lockedAt: timestamp('locked_at'),

  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});
```

**JSONB Structure for `picks`:**
```typescript
type BracketPicks = {
  [gameId: string]: {
    pickedTeamId: string;
    round: string;
  };
};
```

---

## Simple Prediction Algorithm (MVP)

Instead of complex ML, use a formula-based approach:

```typescript
// lib/predictions.ts

interface Team {
  seed: number;
  offensiveEfficiency: number;
  defensiveEfficiency: number;
  wins: number;
  losses: number;
}

export function calculateWinProbability(teamA: Team, teamB: Team): number {
  // Seed advantage (higher seed = lower number = better)
  const seedDiff = teamB.seed - teamA.seed;
  const seedFactor = seedDiff * 0.08; // 8% per seed difference

  // Efficiency differential
  const teamANetEff = teamA.offensiveEfficiency - teamA.defensiveEfficiency;
  const teamBNetEff = teamB.offensiveEfficiency - teamB.defensiveEfficiency;
  const effDiff = (teamANetEff - teamBNetEff) / 20; // Normalize

  // Win percentage factor
  const teamAWinPct = teamA.wins / (teamA.wins + teamA.losses);
  const teamBWinPct = teamB.wins / (teamB.wins + teamB.losses);
  const winPctDiff = (teamAWinPct - teamBWinPct) * 0.3;

  // Combined probability (logistic function)
  const advantage = seedFactor + effDiff + winPctDiff;
  const probability = 1 / (1 + Math.exp(-advantage * 3));

  // Clamp between 15% and 85%
  return Math.max(0.15, Math.min(0.85, probability));
}
```

**Why This Works for MVP:**
- No training data needed
- Instant calculation (no API calls)
- Uses proven predictors (seed, efficiency, record)
- Good enough for user guidance
- Can be replaced with ML later

---

## API Routes

```typescript
// API structure

// Teams
GET    /api/teams                 // List all tournament teams
GET    /api/teams/:id             // Single team details

// Games
GET    /api/games                 // All tournament games
GET    /api/games/:id             // Single game details
GET    /api/games/:id/prediction  // Win probability

// Brackets
GET    /api/brackets              // User's brackets
POST   /api/brackets              // Create bracket
GET    /api/brackets/:id          // Get bracket
PUT    /api/brackets/:id          // Update picks
POST   /api/brackets/:id/lock     // Lock bracket (no more edits)
DELETE /api/brackets/:id          // Delete bracket

// Scoring (admin only, MVP manual)
POST   /api/admin/games/:id/result  // Update game result
POST   /api/admin/score-brackets    // Recalculate all bracket scores
```

---

## Key Pages

### 1. Home Page (`/`)
- Tournament overview
- "Create Bracket" CTA
- User's saved brackets list (if logged in)

### 2. Bracket Builder (`/bracket/new` or `/bracket/:id`)
- Interactive bracket visualization
- Click to pick winners
- Show win probabilities on hover
- Auto-save draft (if logged in)
- Lock button (irreversible)

### 3. Team Page (`/teams/:id`)
```typescript
interface TeamPageData {
  team: Team;
  stats: {
    record: string;          // "28-6"
    ppg: number;             // 78.4
    oppPpg: number;          // 68.2
    efficiency: {
      offensive: number;
      defensive: number;
      net: number;
    };
  };
  upcomingGame?: {
    opponent: Team;
    round: string;
    winProbability: number;
  };
  schedule: Game[];          // Past games (if available)
}
```

Display:
- Team info card (seed, conference, record)
- Key stats with simple visualizations
- Next matchup with prediction
- Tournament path visualization

### 4. My Brackets (`/my-brackets`)
- List of user's brackets
- Points earned (if tournament started)
- Edit/delete draft brackets

---

## Bracket Builder UI

### Component Structure
```typescript
<BracketBuilder>
  <Region name="South">
    <Round name="Round of 64">
      <Matchup game={game1}>
        <Team seed={1} name="Houston" />
        <Team seed={16} name="Norfolk State" />
      </Matchup>
      {/* ... more matchups */}
    </Round>
    <Round name="Round of 32">
      {/* ... */}
    </Round>
  </Region>
  {/* East, West, Midwest regions */}
  <FinalFour />
  <Championship />
</BracketBuilder>
```

### Interaction
1. User clicks on a team → that team advances
2. Show win probability next to each team
3. Gray out eliminated teams
4. Auto-propagate winner to next round
5. Show "Pick" badges for user selections vs "Predicted" badges for suggested picks

### Visual States
- **Empty:** No picks made yet
- **Draft:** Partially filled, auto-saving
- **Complete:** All 63 picks made
- **Locked:** Can't edit, tournament started
- **Scored:** Show correct/incorrect picks with colors (green/red)

---

## Scoring Logic

### Standard Scoring (MVP)
```typescript
const ROUND_POINTS = {
  "Round of 64": 1,
  "Round of 32": 2,
  "Sweet 16": 4,
  "Elite 8": 8,
  "Final Four": 16,
  "Championship": 32,
};

function calculateBracketScore(
  picks: BracketPicks,
  results: GameResult[]
): number {
  let score = 0;

  for (const result of results) {
    const pick = picks[result.gameId];
    if (pick?.pickedTeamId === result.winnerId) {
      score += ROUND_POINTS[result.round];
    }
  }

  return score;
}
```

**Future Enhancements:**
- Upset bonus (extra points for picking lower seed wins)
- Seed differential scoring
- Custom scoring systems

---

## Data Ingestion Strategy (MVP)

### Initial Data Load
1. **Manual CSV import** of tournament teams
   - Team name, seed, region, conference
   - Basic stats (W-L, PPG, efficiency if available)

2. **One-time setup script:**
```typescript
// scripts/seed-tournament.ts

const teams = [
  { id: "houston-2024", school: "Houston", seed: 1, region: "South", ... },
  { id: "duke-2024", school: "Duke", seed: 2, region: "East", ... },
  // ... all 64 teams
];

async function seedDatabase() {
  for (const team of teams) {
    await db.insert(teams).values(team);
  }

  // Generate all 63 tournament games
  const games = generateTournamentBracket(teams);
  for (const game of games) {
    await db.insert(games).values(game);
  }
}
```

3. **Manual game result updates** (admin UI)
   - After each game, update `winnerId` and scores
   - Trigger bracket rescoring

### Where to Get Data
- **Team seeds/bracket:** NCAA official site (manual copy)
- **Basic stats:** Sports Reference, ESPN (manual entry or scrape)
- **Efficiency ratings:** KenPom (subscription, manual entry for 64 teams)

**Post-MVP:** Automate with APIs, scheduled jobs, etc.

---

## Development Phases (Simplified)

### Phase 1: Foundation
- [ ] TanStack Start project setup
- [ ] Database schema with Drizzle
- [ ] User auth (Clerk)
- [ ] Basic routing and layout

**Deliverable:** Users can sign up and navigate the app.

### Phase 2: Team Data
- [ ] Seed database with 64 tournament teams
- [ ] Team list page
- [ ] Individual team page with stats
- [ ] Simple prediction function

**Deliverable:** Users can browse teams and see win probabilities.

### Phase 3: Bracket Builder
- [ ] Bracket UI component (all 4 regions + Final Four)
- [ ] Click to pick interaction
- [ ] Save bracket to database
- [ ] Display user's brackets

**Deliverable:** Users can fill out and save brackets.

### Phase 4: Scoring & Polish
- [ ] Lock bracket functionality
- [ ] Admin UI to update game results
- [ ] Bracket scoring calculation
- [ ] Show correct/incorrect picks
- [ ] Basic styling and responsive design

**Deliverable:** Functional MVP ready for users.

---

## Post-MVP Roadmap

### Enhancement 1: Better Predictions
- Train simple ML model (logistic regression or XGBoost)
- Use historical tournament data (2015-2023)
- Add feature: "Show ML Picks" vs "Show Seed Picks"

### Enhancement 2: Live Tracking
- Integrate live score API (ESPN, TheOddsAPI)
- WebSocket for real-time updates
- Auto-update bracket scores as games complete

### Enhancement 3: Pool Features
- Create/join pools
- Pool leaderboards
- Invite codes
- Different scoring systems

### Enhancement 4: Advanced Analytics
- Monte Carlo simulations
- Team comparison tool
- Upset probability indicators
- Historical bracket analysis

### Enhancement 5: Social Features
- Share brackets (public URLs)
- Compare with friends
- Expert picks aggregation
- Bracket export (PDF, image)

---

## Testing Strategy (MVP)

### Unit Tests
```typescript
// Prediction logic
describe('calculateWinProbability', () => {
  it('favors higher seed', () => {
    const teamA = { seed: 1, offEff: 120, defEff: 90, wins: 30, losses: 4 };
    const teamB = { seed: 16, offEff: 100, defEff: 105, wins: 20, losses: 12 };

    const prob = calculateWinProbability(teamA, teamB);
    expect(prob).toBeGreaterThan(0.85);
  });
});

// Scoring logic
describe('calculateBracketScore', () => {
  it('awards correct points per round', () => {
    const picks = { "game1": { pickedTeamId: "duke", round: "Round of 32" } };
    const results = [{ gameId: "game1", winnerId: "duke", round: "Round of 32" }];

    expect(calculateBracketScore(picks, results)).toBe(2);
  });
});
```

### Manual Testing
- [ ] Create bracket end-to-end
- [ ] Lock bracket, verify can't edit
- [ ] Update game results, verify scores update
- [ ] Test on mobile (responsive design)

---

## Deployment Checklist

### Prerequisites
- [ ] Supabase project created (PostgreSQL database)
- [ ] Clerk account (auth)
- [ ] Vercel account (hosting)

### Environment Variables
```bash
# .env
DATABASE_URL=postgresql://...
CLERK_SECRET_KEY=sk_...
CLERK_PUBLISHABLE_KEY=pk_...
```

### Deployment Steps
1. Push to GitHub
2. Connect Vercel to repo
3. Set environment variables in Vercel
4. Run database migrations: `npm run db:push`
5. Seed tournament data: `npm run seed`
6. Deploy: `vercel --prod`

### Post-Deploy
- [ ] Test sign up / sign in
- [ ] Create test bracket
- [ ] Verify database persistence

---

## Success Metrics (MVP)

### Technical
- **Page Load:** <2s for bracket builder
- **Prediction Calc:** <100ms per game
- **Uptime:** 99% during tournament

### Product
- **Bracket Completion Rate:** >60% of started brackets get locked
- **Return Rate:** >40% of users create multiple brackets
- **Error Rate:** <5% of bracket saves fail

---

## Open Questions

1. **Data Source:** Can we scrape KenPom legally? Or manually enter 64 teams?
2. **Scoring Updates:** Manual admin UI or automated (requires live score API)?
3. **Free vs Paid:** Completely free MVP, or gate features (e.g., multiple brackets)?
4. **Mobile:** Bracket UI on mobile - simplify layout or keep full bracket?

---

## Next Steps

1. **Initialize TanStack Start project**
2. **Set up Supabase database**
3. **Create database schema with Drizzle**
4. **Add Clerk authentication**
5. **Build team list and detail pages**
6. **Implement bracket builder UI**
7. **Add bracket save/load functionality**
8. **Deploy to Vercel**

---

**Document Version:** 2.0 (MVP Simplified)
**Last Updated:** 2026-03-08
**Status:** Ready for Implementation
