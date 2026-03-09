# NCAA March Madness Analytics Platform - System Design

## Project Overview

**Objective:** Build a web application that helps users create informed NCAA tournament brackets with data-driven team statistics and predictions.

**MVP Core Value:** Users can view team stats, see predictions, fill out brackets, and track scores during the tournament.

**Post-MVP Enhancements:** ML models, live tracking, pools, advanced analytics, Monte Carlo simulations.

---

## MVP Scope

### What We're Building (MVP)
1. **Bracket Builder UI** - Interactive 64-team bracket using library component
2. **Team Stats Pages** - Basic stats (record, PPG, efficiency ratings)
3. **Simple Predictions** - Seed-based win probabilities with stat adjustments
4. **User Accounts** - Save and manage brackets
5. **Bracket Scoring** - Score brackets as tournament progresses
6. **Leaderboard** - See top brackets and your ranking

### What We're NOT Building Yet
- ❌ Complex ML ensemble models
- ❌ Monte Carlo simulations
- ❌ Live score tracking (WebSocket)
- ❌ Pool features (create/join groups)
- ❌ Advanced analytics dashboards
- ❌ Real-time score updates

---

## Architecture (Production-Ready)

```
┌─────────────────────────────────────────────────────────┐
│                  Vercel Edge Network                     │
│  ┌──────────────────────────────────────────────────┐   │
│  │         Rate Limiting (Upstash Redis)            │   │
│  └──────────────────┬───────────────────────────────┘   │
└─────────────────────┼───────────────────────────────────┘
                      │
┌─────────────────────┴───────────────────────────────────┐
│           Frontend (Next.js 15 App Router)               │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Bracket    │  │  Team Stats  │  │  Leaderboard │  │
│  │   Builder    │  │    Pages     │  │              │  │
│  │ (@g-loot lib)│  │              │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │ REST API (ISR + SWR Caching)
┌──────────────────────┴──────────────────────────────────┐
│              Backend (Next.js API Routes)                │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  API Routes with Caching                        │    │
│  │  /api/teams (24h ISR)                           │    │
│  │  /api/games (1h ISR)                            │    │
│  │  /api/leaderboard (5m SWR)                      │    │
│  │  /api/brackets (private, no cache)              │    │
│  └────────────┬────────────────────────────────────┘    │
│               │                                          │
│  ┌────────────┴────────────────────────────────────┐    │
│  │      Prediction Engine (TypeScript)             │    │
│  │   Seed-based formula + stat adjustments         │    │
│  └────────────┬────────────────────────────────────┘    │
│               │                                          │
│  ┌────────────┴────────────────────────────────────┐    │
│  │         Prisma ORM                              │    │
│  └────────────┬────────────────────────────────────┘    │
└───────────────┼─────────────────────────────────────────┘
                │
┌───────────────┴─────────────────────────────────────────┐
│                  Neon PostgreSQL                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Tables: teams, games, users, brackets,         │   │
│  │          bracket_picks (normalized)              │   │
│  │                                                  │   │
│  │  Materialized View: bracket_leaderboard         │   │
│  │  Partitions: brackets_2024, brackets_2025       │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  Features: Serverless, Branching, Connection Pooling    │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│              Supabase Auth (separate service)            │
│  - 50k MAU free tier                                     │
│  - Row-Level Security for bracket data                   │
└──────────────────────────────────────────────────────────┘
```

**Key Production Features:**
- ✅ Rate limiting at edge (handles traffic spikes)
- ✅ Multi-layer caching (reduces DB load by 95%+)
- ✅ Serverless database (auto-scales for March traffic)
- ✅ Optimistic locking (prevents concurrent edit conflicts)
- ✅ Materialized views (fast leaderboards)
- ✅ Connection pooling (prevents exhaustion)

---

## Tech Stack (Production-Ready)

### Full Stack
- **Framework:** Next.js 15 (App Router with Server Components)
- **Language:** TypeScript (strict mode)
- **Styling:** Tailwind CSS
- **UI Components:** shadcn/ui
- **ORM:** Prisma
- **Database:** Neon PostgreSQL (serverless)
- **Auth:** Supabase Auth
- **Bracket UI:** @g-loot/react-tournament-brackets

### Infrastructure
- **Hosting:** Vercel (edge network, zero-config)
- **Rate Limiting:** Upstash Redis (edge-optimized)
- **Monitoring:** Vercel Analytics + Sentry

### Why This Stack

**Next.js 15 (over TanStack Start)**:
- ✅ Production-proven for traffic spikes (TikTok, Hulu, Twitch use it)
- ✅ ISR perfect for tournament data that updates periodically
- ✅ Largest ecosystem and community
- ✅ Vercel deployment optimized
- ⚠️ Trade-off: Vercel vendor lock-in (acceptable)

**Prisma (over Drizzle)**:
- ✅ Automated migrations (critical during tournament season)
- ✅ Prisma Studio for debugging bracket data
- ✅ Better team collaboration (reviewable schema files)
- ⚠️ Trade-off: Larger bundle (acceptable for non-edge)

**Neon (over Supabase database)**:
- ✅ Serverless: auto-scales for traffic spikes
- ✅ Branching: test scoring logic on production data copy
- ✅ Connection pooling: prevents exhaustion
- ✅ Cost: separates compute/storage, near-zero off-season
- ⚠️ Trade-off: Database-only (using Supabase Auth separately)

**Supabase Auth (over Clerk)**:
- ✅ Better cost at scale: $187 vs $287 for 150k users
- ✅ 50k MAU free tier (Clerk: 10k)
- ✅ Row-Level Security perfect for bracket data
- ⚠️ Trade-off: Less polished than Clerk (but sufficient)

**Bracket Library (over custom build)**:
- ✅ Saves 1-2 weeks development time
- ✅ Pan/zoom built-in (essential for mobile)
- ✅ Battle-tested, actively maintained
- ⚠️ Trade-off: May need customization for advanced features

---

## Database Schema (Production-Ready)

### Core Tables

```typescript
// schema.prisma

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  name          String?
  createdAt     DateTime  @default(now())
  brackets      Bracket[]
}

model Team {
  id         String   @id  // "duke-2024"
  school     String        // "Duke"
  conference String        // "ACC"
  seed       Int?          // 1-16
  region     String?       // "South"
  season     Int           // 2024

  // Basic stats
  wins               Int
  losses             Int
  ppg                Float?
  oppPpg             Float?  // Opponent PPG

  // Advanced stats
  offensiveEfficiency Float?
  defensiveEfficiency Float?
  tempo              Float?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  // Relations
  homeGames  Game[] @relation("HomeTeam")
  awayGames  Game[] @relation("AwayTeam")
  picks      BracketPick[]

  @@index([season, region])
  @@index([seed])
}

model Game {
  id       String @id @default(uuid())
  season   Int
  round    String  // "Round of 64", "Round of 32", etc.

  teamAId  String
  teamBId  String
  teamA    Team   @relation("HomeTeam", fields: [teamAId], references: [id])
  teamB    Team   @relation("AwayTeam", fields: [teamBId], references: [id])

  // Scores (null if not played yet)
  teamAScore Int?
  teamBScore Int?
  winnerId   String?

  // Prediction
  teamAWinProbability Float?

  status    String   // "scheduled" | "in_progress" | "final"
  createdAt DateTime @default(now())

  @@index([season, round])
  @@index([status])
}

model Bracket {
  id       String  @id @default(uuid())
  userId   String
  user     User    @relation(fields: [userId], references: [id])
  season   Int
  name     String

  pointsEarned Int      @default(0)
  locked       Boolean  @default(false)
  lockedAt     DateTime?

  // Optimistic locking
  version      Int      @default(1)

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  picks BracketPick[]

  @@index([userId, season])
  @@index([locked, season])
  @@index([pointsEarned])
}

// NORMALIZED picks table (not JSONB)
// Better for analytics: "most picked teams", "percentage picking upsets"
model BracketPick {
  id         String @id @default(uuid())
  bracketId  String
  bracket    Bracket @relation(fields: [bracketId], references: [id], onDelete: Cascade)

  gameId     String
  round      String  // "Round of 64", "Round of 32", etc.

  pickedTeamId String
  pickedTeam   Team   @relation(fields: [pickedTeamId], references: [id])

  // For scoring
  points     Int     @default(0)
  isCorrect  Boolean?  // null until game completes

  createdAt  DateTime @default(now())

  @@unique([bracketId, gameId])  // One pick per game per bracket
  @@index([pickedTeamId])        // Fast "most picked teams" queries
  @@index([bracketId])
}
```

### Materialized View for Leaderboard

```sql
-- Run via Prisma migration (raw SQL)
CREATE MATERIALIZED VIEW bracket_leaderboard AS
SELECT
  b.id as bracket_id,
  b.user_id,
  b.name as bracket_name,
  u.name as user_name,
  u.email,
  b.points_earned,
  b.locked_at,
  DENSE_RANK() OVER (ORDER BY b.points_earned DESC, b.locked_at ASC) as rank,
  b.season
FROM brackets b
JOIN users u ON u.id = b.user_id
WHERE b.locked = true
ORDER BY rank ASC;

-- Create indexes
CREATE INDEX idx_leaderboard_rank ON bracket_leaderboard(rank);
CREATE INDEX idx_leaderboard_user ON bracket_leaderboard(user_id);
CREATE INDEX idx_leaderboard_season ON bracket_leaderboard(season);

-- Refresh function (call after scoring)
CREATE OR REPLACE FUNCTION refresh_leaderboard()
RETURNS void AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY bracket_leaderboard;
END;
$$ LANGUAGE plpgsql;
```

**Why normalized picks table over JSONB?**
- ✅ 10-100x faster for analytics queries ("most picked teams")
- ✅ Data integrity via foreign keys (prevents invalid picks)
- ✅ Easy to add features (confidence scores, etc.)
- ✅ Predictable performance at scale
- ❌ More tables (but clearer schema)

**Why materialized view for leaderboard?**
- ✅ <50ms query for 100k brackets (vs 500-2000ms real-time)
- ✅ Refreshed after scoring (a few times per day during tournament)
- ✅ No expensive JOINs on every leaderboard view

---

## Rate Limiting Strategy

### Implementation

```typescript
// middleware.ts (Vercel Edge)
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, '1 m'), // 100 requests per minute
  analytics: true,
});

export async function middleware(request: NextRequest) {
  const ip = request.ip ?? '127.0.0.1';

  // Only rate limit API routes
  if (request.nextUrl.pathname.startsWith('/api/')) {
    const { success, limit, reset, remaining } = await ratelimit.limit(ip);

    if (!success) {
      return new NextResponse('Too Many Requests', {
        status: 429,
        headers: {
          'X-RateLimit-Limit': limit.toString(),
          'X-RateLimit-Remaining': remaining.toString(),
          'X-RateLimit-Reset': reset.toString(),
        },
      });
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: '/api/:path*',
};
```

### Rate Limits by Endpoint

| Endpoint | Limit | Window | Reasoning |
|----------|-------|--------|-----------|
| `/api/teams` | 1000/min | Per IP | Read-heavy, generous |
| `/api/brackets` (POST/PUT) | 5/min | Per user | Prevent spam edits |
| `/api/brackets/:id/lock` | 1/hour | Per user | Irreversible action |
| `/api/leaderboard` | 10/min | Per IP | Expensive query |
| `/api/admin/*` | 100/hour | Per admin | Sensitive operations |

**Why Upstash Redis?**
- ✅ Edge-optimized (multi-region, low latency)
- ✅ Serverless-friendly (no connection management)
- ✅ Free tier: 10k requests/day
- ✅ Caches rate limit data while function is "hot"

---

## Caching Strategy

### Multi-Layer Caching

**Layer 1: ISR (Incremental Static Regeneration)**

```typescript
// app/api/teams/route.ts
export const revalidate = 86400; // 24 hours

export async function GET() {
  const teams = await prisma.team.findMany({
    where: { season: 2024 },
    orderBy: [{ region: 'asc' }, { seed: 'asc' }],
  });

  return Response.json(teams);
  // Cached for 24h, revalidated in background
}
```

**Layer 2: Stale-While-Revalidate Headers**

```typescript
// app/api/leaderboard/route.ts
export async function GET() {
  const leaderboard = await prisma.$queryRaw`
    SELECT * FROM bracket_leaderboard
    WHERE season = 2024
    ORDER BY rank ASC
    LIMIT 100
  `;

  return new Response(JSON.stringify(leaderboard), {
    headers: {
      'Content-Type': 'application/json',
      'Cache-Control': 'public, s-maxage=300, stale-while-revalidate=600',
      // Cache for 5min, serve stale for up to 10min while revalidating
    },
  });
}
```

**Layer 3: Edge Caching (CDN)**

| Endpoint | Cache-Control | TTL | Reasoning |
|----------|--------------|-----|-----------|
| `/api/teams` | `s-maxage=86400` | 24h | Teams don't change during tournament |
| `/api/games` | `s-maxage=3600` | 1h | Schedule static, scores update periodically |
| `/api/leaderboard` | `s-maxage=300, stale-while-revalidate=600` | 5m | Acceptable staleness |
| `/api/brackets/:id` | `private` | 0 | User data, never cache |
| `/api/predictions` | `s-maxage=3600` | 1h | Predictions recalculated periodically |

### Performance Impact

**Without caching:**
- 1M leaderboard views = 1M database queries
- Database overwhelmed during tournament

**With caching:**
- 1M leaderboard views = ~200 database queries (5min TTL)
- 95%+ reduction in database load
- Users never see loading spinners (stale-while-revalidate)

---

## Optimistic Locking (Concurrent Edits)

### Problem
User opens bracket on phone and laptop, edits both. Which version wins?

### Solution: Version Column

```typescript
// app/api/brackets/[id]/route.ts
export async function PUT(
  request: Request,
  { params }: { params: { id: string } }
) {
  const body = await request.json();
  const { picks, version: clientVersion } = body;

  try {
    // Attempt optimistic update
    const updated = await prisma.bracket.updateMany({
      where: {
        id: params.id,
        version: clientVersion, // Only update if version matches
      },
      data: {
        version: { increment: 1 },
        updatedAt: new Date(),
      },
    });

    if (updated.count === 0) {
      // Version conflict - bracket was modified elsewhere
      const current = await prisma.bracket.findUnique({
        where: { id: params.id },
      });

      return new Response(
        JSON.stringify({
          error: 'CONFLICT',
          message: 'Bracket was modified on another device',
          currentVersion: current?.version,
        }),
        { status: 409 }
      );
    }

    // Update picks
    await prisma.bracketPick.deleteMany({
      where: { bracketId: params.id },
    });

    await prisma.bracketPick.createMany({
      data: picks.map((pick: any) => ({
        bracketId: params.id,
        gameId: pick.gameId,
        round: pick.round,
        pickedTeamId: pick.pickedTeamId,
      })),
    });

    return new Response(JSON.stringify({ success: true }), { status: 200 });
  } catch (error) {
    return new Response('Internal error', { status: 500 });
  }
}
```

**Frontend Conflict Resolution:**

```typescript
// components/ConflictModal.tsx
export function ConflictModal({ conflict, onResolve }: ConflictModalProps) {
  return (
    <Dialog open={!!conflict}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Bracket Conflict Detected</DialogTitle>
          <DialogDescription>
            This bracket was modified on another device. Choose how to resolve:
          </DialogDescription>
        </DialogHeader>

        <div className="space-y-3">
          <Button onClick={() => onResolve('use-server')}>
            Use Server Version (Most Recent Save)
          </Button>
          <Button onClick={() => onResolve('use-local')}>
            Use This Device's Version (Your Unsaved Changes)
          </Button>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

---

## Prediction Algorithm

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

**Why formula-based (not ML)?**
- ✅ No training data needed
- ✅ Instant calculation (<1ms)
- ✅ Uses proven predictors
- ✅ Good enough for MVP
- ✅ Can be replaced with ML later

---

## API Routes

### Teams
```
GET    /api/teams                 # All tournament teams (ISR: 24h)
GET    /api/teams/:id             # Single team (ISR: 24h)
GET    /api/teams/search?q=duke   # Search teams (simple LIKE query)
```

### Games
```
GET    /api/games                 # All tournament games (ISR: 1h)
GET    /api/games/:id             # Single game (ISR: 1h)
GET    /api/games/:id/prediction  # Win probability
```

### Brackets
```
GET    /api/brackets              # User's brackets (private)
POST   /api/brackets              # Create bracket
GET    /api/brackets/:id          # Get bracket (private)
PUT    /api/brackets/:id          # Update picks (optimistic locking)
POST   /api/brackets/:id/lock     # Lock bracket (rate limited: 1/hour)
DELETE /api/brackets/:id          # Delete bracket
```

### Leaderboard
```
GET    /api/leaderboard           # Top 100 + user's rank (SWR: 5m)
```

### Admin
```
POST   /api/admin/games/:id/result    # Update game result
POST   /api/admin/score-brackets      # Recalculate all scores + refresh materialized view
```

---

## Key Pages

### 1. Home (`/`)
- Tournament overview
- "Create Bracket" CTA
- User's brackets list (if logged in)

### 2. Bracket Builder (`/bracket/:id`)
```typescript
import { Bracket } from '@g-loot/react-tournament-brackets';

export default function BracketBuilderPage() {
  const { data: bracket } = useBracket(params.id);
  const { data: games } = useGames();

  const rounds = transformToLibraryFormat(games, bracket.picks);

  return (
    <div>
      <h1>{bracket.name}</h1>
      <div className="progress">
        {bracket.picks.length} / 63 picks made
      </div>

      <Bracket
        rounds={rounds}
        renderSeedComponent={CustomMatchup}
        onMatchClick={handlePick}
        theme={marchMadnessTheme}
      />

      <div className="actions">
        <Button onClick={saveDraft}>Save Draft</Button>
        <Button onClick={lockBracket} disabled={bracket.picks.length < 63}>
          Lock Bracket
        </Button>
      </div>
    </div>
  );
}
```

### 3. Team Page (`/teams/:id`)
- Team info card
- Stats (efficiency, tempo, record)
- Next matchup with prediction
- Simple visualizations

### 4. Leaderboard (`/leaderboard`)
```typescript
export default function LeaderboardPage() {
  const { data } = useLeaderboard();

  return (
    <div>
      <h1>Leaderboard</h1>
      <p>Last updated: {data.lastUpdated}</p>

      <table>
        <thead>
          <tr>
            <th>Rank</th>
            <th>User</th>
            <th>Bracket</th>
            <th>Points</th>
          </tr>
        </thead>
        <tbody>
          {data.topN.map(entry => (
            <tr key={entry.bracketId} className={entry.isCurrentUser ? 'highlight' : ''}>
              <td>{entry.rank}</td>
              <td>{entry.userName}</td>
              <td>{entry.bracketName}</td>
              <td>{entry.pointsEarned}</td>
            </tr>
          ))}
        </tbody>
      </table>

      {data.currentUser && data.currentUser.rank > 100 && (
        <div className="your-rank">
          <p>Your Rank: #{data.currentUser.rank}</p>
          <p>Points: {data.currentUser.pointsEarned}</p>
        </div>
      )}
    </div>
  );
}
```

### 5. My Brackets (`/my-brackets`)
- List of user's brackets
- Points, locked status, created date
- Edit/delete buttons

---

## Scoring Logic

```typescript
// lib/scoring.ts

const ROUND_POINTS = {
  'Round of 64': 1,
  'Round of 32': 2,
  'Sweet 16': 4,
  'Elite 8': 8,
  'Final Four': 16,
  'Championship': 32,
};

export async function scoreBracket(bracketId: string) {
  const picks = await prisma.bracketPick.findMany({
    where: { bracketId },
  });

  const games = await prisma.game.findMany({
    where: {
      id: { in: picks.map(p => p.gameId) },
      status: 'final',
    },
  });

  let totalPoints = 0;

  for (const pick of picks) {
    const game = games.find(g => g.id === pick.gameId);
    if (!game) continue; // Game not complete yet

    const isCorrect = game.winnerId === pick.pickedTeamId;
    const points = isCorrect ? ROUND_POINTS[pick.round as keyof typeof ROUND_POINTS] : 0;

    await prisma.bracketPick.update({
      where: { id: pick.id },
      data: { isCorrect, points },
    });

    totalPoints += points;
  }

  await prisma.bracket.update({
    where: { id: bracketId },
    data: { pointsEarned: totalPoints },
  });

  return totalPoints;
}

export async function scoreAllBrackets() {
  const brackets = await prisma.bracket.findMany({
    where: { locked: true },
  });

  for (const bracket of brackets) {
    await scoreBracket(bracket.id);
  }

  // Refresh materialized view after scoring
  await prisma.$executeRaw`SELECT refresh_leaderboard()`;
}
```

---

## Development Phases

### Phase 1: Foundation
- [ ] Next.js 15 project setup
- [ ] Neon database + Prisma schema
- [ ] Supabase Auth integration
- [ ] Basic routing and layout
- [ ] Upstash Redis rate limiting

**Deliverable:** Users can sign up, auth works, rate limiting active.

### Phase 2: Team Data
- [ ] Seed 64 tournament teams
- [ ] Team list and detail pages
- [ ] Prediction function implementation
- [ ] Compute predictions for all games

**Deliverable:** Users can browse teams and see predictions.

### Phase 3: Bracket Builder
- [ ] Integrate @g-loot/react-tournament-brackets
- [ ] Pick state management with optimistic locking
- [ ] Save/load bracket functionality
- [ ] My Brackets page

**Deliverable:** Users can fill out and save brackets.

### Phase 4: Scoring & Leaderboard
- [ ] Lock bracket functionality
- [ ] Admin UI for game results
- [ ] Scoring calculation
- [ ] Materialized view for leaderboard
- [ ] Leaderboard page

**Deliverable:** Functional MVP with scoring and rankings.

---

## Post-MVP Enhancements

### Enhancement 1: Machine Learning
- Train XGBoost model on historical data
- Replace formula with ML predictions
- Add "ML Confidence" indicator

### Enhancement 2: Live Tracking
- Integrate live score API (ESPN, TheOddsAPI)
- WebSocket for real-time updates
- Auto-score brackets as games complete

### Enhancement 3: Pool Features
- Create/join pools with invite codes
- Pool-specific leaderboards
- Different scoring systems (upset bonus, etc.)

### Enhancement 4: Monte Carlo Simulations
- Run 10,000 tournament simulations
- Show team's probability to reach each round
- Bracket optimization strategies

### Enhancement 5: Advanced Analytics
- Team comparison tool
- Historical bracket analysis
- Consensus picks heatmap
- Bracket export (PDF, image)

---

## Testing Strategy

### Unit Tests
```typescript
// lib/predictions.test.ts
describe('calculateWinProbability', () => {
  it('favors higher seed significantly', () => {
    const team1Seed = { seed: 1, offEff: 120, defEff: 90, wins: 30, losses: 4 };
    const team16Seed = { seed: 16, offEff: 100, defEff: 105, wins: 20, losses: 12 };

    const prob = calculateWinProbability(team1Seed, team16Seed);
    expect(prob).toBeGreaterThan(0.85);
  });
});

// lib/scoring.test.ts
describe('calculateBracketScore', () => {
  it('awards correct points per round', () => {
    // Test implementation
  });
});
```

### Integration Tests
- End-to-end bracket creation flow
- Optimistic locking conflict resolution
- Leaderboard materialized view refresh
- Rate limiting enforcement

### Load Tests
- Simulate Selection Sunday traffic (1M concurrent users)
- Verify caching reduces DB queries by 95%+
- Verify connection pooling prevents exhaustion

---

## Deployment

### Prerequisites
- [ ] Neon project created
- [ ] Supabase Auth configured
- [ ] Upstash Redis account
- [ ] Vercel account

### Environment Variables
```bash
# .env
DATABASE_URL=postgresql://...
DIRECT_URL=postgresql://...  # For migrations

NEXT_PUBLIC_SUPABASE_URL=https://...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_KEY=...

UPSTASH_REDIS_REST_URL=...
UPSTASH_REDIS_REST_TOKEN=...
```

### Deploy
```bash
# Push schema to Neon
npx prisma db push

# Generate Prisma Client
npx prisma generate

# Seed tournament data
npm run seed

# Deploy to Vercel
vercel --prod
```

---

## Success Metrics

### Technical
- **Page Load:** <2s for bracket builder
- **API Response:** <200ms p95
- **Uptime:** 99.9% during tournament
- **Cache Hit Rate:** >95%

### Product
- **Bracket Completion:** >60% of started brackets get locked
- **Return Rate:** >40% create multiple brackets
- **Leaderboard Engagement:** >50% check rankings

---

## Cost Estimates

### MVP (Free Tier)
- Vercel: $0
- Neon: $0 (100 compute-hours)
- Supabase Auth: $0 (50k MAU)
- Upstash: $0 (10k requests/day)
- **Total: $0/month**

### Moderate Success (50k users)
- Vercel: $20 (pro plan)
- Neon: $20 (compute overage)
- Supabase Auth: $0
- Upstash: $10
- **Total: $50/month**

### Viral (500k users)
- Vercel: $20
- Neon: $150
- Supabase Auth: $1,462
- Upstash: $50
- **Total: $1,682/month**

---

## Open Questions

1. **Data Source:** Manually enter 64 teams or scrape from NCAA site?
2. **Scoring Automation:** Manual admin UI or integrate live score API?
3. **Monetization:** Free forever, or premium features (pools, advanced analytics)?

---

## Next Steps

1. Review architecture analysis document
2. Begin Phase 1 implementation
3. Set up CI/CD pipeline
4. Create monitoring dashboards

---

**Document Version:** 3.0 (Production-Ready Architecture)
**Last Updated:** 2026-03-08
**Status:** Ready for Implementation
**See Also:** ARCHITECTURE_ANALYSIS.md for detailed research and decision rationale
