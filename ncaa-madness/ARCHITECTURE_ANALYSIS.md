# Architecture Analysis & Decision Log

This document details the research, critical analysis, and reasoning behind every architectural decision for the NCAA March Madness platform.

---

## Executive Summary

After deep research into production realities, several critical changes were made to the original MVP design:

### Major Changes
1. **Framework**: TanStack Start → **Next.js 15** (production-ready for traffic spikes)
2. **ORM**: Drizzle → **Prisma** (better team collaboration)
3. **Auth**: Clerk → **Supabase Auth** (better cost at scale)
4. **Database**: Supabase → **Neon** (serverless auto-scaling)
5. **Data Model**: JSONB → **Normalized tables** (analytics performance)
6. **Added**: Bracket UI library (saves 1-2 weeks)
7. **Added**: Rate limiting (essential for March traffic)
8. **Added**: Caching strategy (handles millions of users)

---

## 1. Framework: Next.js 15 vs TanStack Start vs Remix

### Original Decision: TanStack Start
**Reasoning**: Modern, excellent type safety, full-stack TypeScript

### Research Findings

**TanStack Start**:
- ✅ Superior end-to-end type safety
- ✅ No vendor lock-in
- ❌ **Beta status** - not production-ready
- ❌ Small community, limited ecosystem
- ❌ Unproven at scale

**Next.js 15**:
- ✅ **Production-proven** for traffic spikes
- ✅ Largest community (100k+ users)
- ✅ **ISR** (Incremental Static Regeneration) perfect for tournament data
- ✅ App Router with Server Components
- ✅ Vercel deployment optimized
- ⚠️ Recent security CVEs (fixed in latest patches)
- ❌ Vendor lock-in to Vercel

**Remix**:
- ✅ Web-standard APIs, great DX
- ✅ 35% smaller JavaScript than Next.js
- ✅ Runtime-agnostic
- ❌ Smaller ecosystem than Next.js
- ❌ Less optimized for Vercel

### Revised Decision: **Next.js 15 (App Router)**

**Critical Reasoning**:
1. **Traffic Spikes**: March Madness has predictable massive traffic during Selection Sunday and tournament games. Next.js is battle-tested for this (used by TikTok, Hulu, Twitch).

2. **ISR**: Tournament data changes periodically (not real-time):
   - Team stats: Update once daily
   - Game schedules: Static after bracket set
   - Leaderboards: Update after each game
   - ISR revalidates every N seconds, serves stale while fetching new data

3. **Deployment**: Vercel's edge network + automatic HTTPS + zero-config deployment is crucial for getting to market before March.

4. **Risk Mitigation**: TanStack Start's beta status is unacceptable for an app that must handle millions of users during a 3-week window. If it breaks during the tournament, we're dead.

**Trade-offs Accepted**:
- Vercel vendor lock-in (can move to self-hosted if needed later)
- Larger bundle size (mitigated by Server Components)
- Must stay updated on security patches

---

## 2. ORM: Prisma vs Drizzle

### Original Decision: Drizzle ORM
**Reasoning**: Lightweight, SQL-like, fast

### Research Findings

**Drizzle**:
- ✅ Zero runtime overhead (~7.4kb)
- ✅ Better for serverless (fast cold starts)
- ✅ SQL-level control
- ❌ Manual migration management
- ❌ Harder for team collaboration (schema in TypeScript code)
- ❌ No visual database browser
- ❌ Reviewers need SQL knowledge to understand schema changes

**Prisma**:
- ✅ **Automated migrations** (`prisma migrate dev`)
- ✅ **Prisma Studio** - visual database browser
- ✅ **Human-readable schema** - `.prisma` file is single source of truth
- ✅ Better for teams with varying SQL expertise
- ✅ Easier PR reviews (schema changes clear in `.prisma` diffs)
- ❌ Rust engine overhead
- ❌ Slower type generation

### Revised Decision: **Prisma**

**Critical Reasoning**:
1. **Team Collaboration**: During March tournament, team will need to quickly iterate on schema changes (add tiebreaker scores, adjust scoring logic, etc.). Automated migrations reduce errors under pressure.

2. **Debugging**: Prisma Studio invaluable for debugging bracket data issues when users report problems during live tournament.

3. **Risk Mitigation**: Manual migrations with Drizzle risky during tournament season. Prisma's declarative approach prevents migration mistakes.

4. **Scale Reality**: App won't hit scale where Drizzle's performance edge matters. User experience > millisecond optimizations.

**Trade-offs Accepted**:
- Larger bundle size (acceptable for non-edge deployment)
- Generation step (adds ~2 seconds to dev workflow)

---

## 3. Authentication: Supabase Auth vs Clerk vs NextAuth

### Original Decision: Clerk
**Reasoning**: Best developer experience, fastest implementation

### Research Findings

**Clerk**:
- ✅ Fastest implementation (1-3 days)
- ✅ Beautiful pre-built UI
- ✅ Best DX
- ❌ **Expensive at scale**: $287.50/month for 150k users
- ❌ Vendor lock-in

**Supabase Auth**:
- ✅ **Better cost**: $187.50/month for 150k users (35% cheaper)
- ✅ **50k MAU free tier** (Clerk: 10k)
- ✅ Row-Level Security perfect for bracket data
- ✅ Less vendor lock-in (open-source)
- ❌ Simpler feature set than Clerk
- ❌ Less polished UI

**NextAuth.js**:
- ✅ Free (self-hosted)
- ✅ Complete control
- ❌ Most development time
- ❌ Session persistence complexity

### Revised Decision: **Supabase Auth**

**Critical Reasoning**:
1. **Cost at Scale**: If app goes viral during March, 150k users = $187.50 vs $287.50. At 500k users: $1,462 vs $2,850. Significant difference.

2. **Free Tier**: 50k MAU allows healthy growth before costs kick in.

3. **Row-Level Security**: Postgres RLS perfect for ensuring users only access their own brackets:
   ```sql
   CREATE POLICY "Users can only view own brackets"
   ON brackets FOR SELECT
   USING (auth.uid() = user_id);
   ```

4. **Ecosystem Fit**: If using Neon for database, Supabase Auth integrates well (both PostgreSQL-focused).

**Trade-offs Accepted**:
- Less polished than Clerk (but sufficient for MVP)
- Fewer advanced features (but don't need them)

---

## 4. Database: Neon vs Supabase vs Railway

### Original Decision: Supabase
**Reasoning**: All-in-one BaaS (database + auth + storage)

### Research Findings

**Neon**:
- ✅ **Serverless**: Compute auto-scales for traffic spikes
- ✅ **Branching**: Copy-on-write branches for testing
- ✅ **Never-expiring free tier**: 100 compute-hours/month
- ✅ **Connection pooling**: Vercel Fluid Compute prevents exhaustion
- ✅ **Separate compute/storage**: Costs drop to near-zero off-season
- ✅ Sub-10ms latency
- ❌ Database-only (no storage/auth)

**Supabase**:
- ✅ Full BaaS platform
- ✅ 500MB free tier
- ❌ **Auto-suspend after 7 days inactivity** (annoying for off-season)
- ❌ 500MB may be tight for multi-year historical data
- ❌ Dedicated compute (always running)

**Railway**:
- ✅ Simple deployment
- ❌ $5/month credit (limited free tier)
- ❌ Traditional PostgreSQL (not serverless)

### Revised Decision: **Neon (Vercel Postgres)**

**Critical Reasoning**:
1. **Traffic Spikes**: Selection Sunday will bring millions of concurrent users. Neon's serverless compute auto-scales instantly. Traditional databases require manual scaling.

2. **Connection Pooling**: Vercel Fluid Compute + Neon's `attachDatabasePool` allows function invocations to share connection pools. Prevents connection exhaustion during traffic spikes.

3. **Branching**: Before tournament starts, can test bracket scoring logic on branch with production data. Critical for avoiding bugs during live tournament.

4. **Cost**: Compute shuts down when idle. In July-February (8 months), database costs near-zero. Supabase charges for dedicated compute year-round.

5. **Free Tier**: Never expires, 100 compute-hours sufficient for MVP scale.

**Trade-offs Accepted**:
- Database-only (but using Supabase Auth separately is fine)
- 7-day inactivity suspension on free tier (but compute restarts instantly)

**Architecture**:
- **Database**: Neon
- **Auth**: Supabase Auth
- **Storage** (if needed): Cloudflare R2 or Vercel Blob

---

## 5. Data Model: JSONB vs Normalized Tables

### Original Decision: JSONB column for picks
**Reasoning**: Simple, all picks in one field

### Research Findings

**JSONB Approach**:
```sql
CREATE TABLE brackets (
  id uuid,
  picks jsonb  -- {"game1": {"team": "duke", "round": "Round of 64"}, ...}
);
```
- ✅ Simple schema
- ✅ Fast full-bracket retrieval (1 row)
- ❌ **Analytics queries slow**: "Most picked teams" requires JSONB path extraction
- ❌ No data integrity (can't foreign key into JSONB)
- ❌ Non-linear performance degradation at scale

**Normalized Approach**:
```sql
CREATE TABLE brackets (id uuid, user_id uuid, ...);
CREATE TABLE bracket_picks (
  bracket_id uuid REFERENCES brackets(id),
  game_id text,
  team_id text REFERENCES teams(id),
  round text,
  points int
);
CREATE INDEX ON bracket_picks(team_id);
```
- ✅ **Analytics queries 10-100x faster**: Simple aggregations
- ✅ **Data integrity**: Foreign keys prevent invalid team picks
- ✅ **Predictable performance** at scale
- ✅ Easy to add features (confidence scores, etc.)
- ❌ Requires joins for full bracket

### Revised Decision: **Normalized Tables**

**Critical Reasoning**:
1. **Analytics Requirements**: Core features need aggregations:
   - "Most picked teams" for consensus view
   - "Percentage picking upsets" for insights
   - "Chalk vs contrarian brackets" for strategy

   These queries are 10-100x faster on normalized tables.

2. **Data Integrity**: Foreign keys prevent users from picking invalid teams (e.g., picking a team that was eliminated earlier).

3. **Future Features**: Easy to add:
   - Confidence scores per pick
   - Tiebreaker total points
   - Historical bracket comparisons

   All require normalized structure.

4. **Performance**: For 63 picks per bracket, joins are negligible (<5ms). Premature optimization to use JSONB.

**Hybrid Option** (if needed later):
- Normalized `bracket_picks` table (source of truth)
- Denormalized JSONB column (for fast display)
- Trigger to sync JSONB on insert/update

**Trade-offs Accepted**:
- More tables (but clearer schema)
- Joins required (but fast with proper indexes)

---

## 6. Bracket UI: Build vs Library

### Original Decision: Build from scratch
**Reasoning**: Full control, custom design

### Research Findings

**Building from Scratch**:
- ✅ Complete control
- ✅ Perfect fit for design
- ❌ **1-2 weeks development time**
- ❌ Complex logic for pan/zoom, connections, responsive
- ❌ Risk of bugs during tournament

**@g-loot/react-tournament-brackets**:
- ✅ **Saves 1-2 weeks** (critical for March deadline)
- ✅ Pan/zoom built-in (essential for mobile)
- ✅ Customizable theming
- ✅ Actively maintained
- ❌ May need customization for advanced features

**Other libraries evaluated**:
- react-tournament-bracket (moodysalem): Different data structure
- @oliverlooney/react-brackets: Less maintained

### Revised Decision: **Use @g-loot/react-tournament-brackets**

**Critical Reasoning**:
1. **Time to Market**: Must launch before Selection Sunday (mid-March). Saving 1-2 weeks on bracket rendering allows focus on predictions, analytics, and polish.

2. **Mobile UX**: Pan/zoom is critical for 63-game bracket on mobile. Building this from scratch is complex.

3. **Risk Mitigation**: Library is battle-tested. Custom implementation risks bugs during live tournament.

4. **Customization**: Library is theme-able. Can brand for March Madness while leveraging core rendering logic.

**Implementation**:
```typescript
import { Bracket } from '@g-loot/react-tournament-brackets';

<Bracket
  rounds={bracketData}
  renderSeedComponent={CustomMatchup}
  theme={marchMadnessTheme}
/>
```

**Trade-offs Accepted**:
- Dependency on external library (but maintained)
- May hit edge cases requiring workarounds

---

## 7. Rate Limiting (NEW ADDITION)

### Original Decision: Not specified

### Research Findings

**Problem**: March Madness traffic is extreme:
- **Selection Sunday**: Millions visit simultaneously when bracket released
- **First weekend**: Constant traffic as games complete
- **Without rate limiting**: Database overwhelmed, app crashes

**Solutions Evaluated**:

**Upstash Redis + Vercel Edge Middleware**:
- ✅ Edge-optimized (multi-region)
- ✅ Works with serverless
- ✅ Free tier: 10k requests/day
- ✅ Sliding window algorithm

**Alternative: Vercel Edge Config**:
- ✅ No external dependency
- ❌ Less flexible than Redis

### Decision: **Upstash Redis + Vercel Edge Middleware**

**Implementation**:
```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'),
});

export async function middleware(request: Request) {
  const { success } = await ratelimit.limit(request.ip);
  if (!success) return new Response('Too Many Requests', { status: 429 });
}
```

**Rate Limits**:
- Bracket submission: 5 edits per minute
- API reads: 100 requests per minute per IP
- Leaderboard: 10 requests per minute

**Critical Reasoning**:
1. **Protects database** during traffic spikes
2. **Prevents abuse** (spam bracket submissions)
3. **Edge deployment** ensures low latency globally
4. **Cost**: Free tier sufficient for MVP, scales with usage

---

## 8. Caching Strategy (NEW ADDITION)

### Original Decision: Not specified

### Research Findings

**Problem**: Database can't handle millions of concurrent reads during tournament.

**Solution: Multi-Layer Caching**

**Layer 1: ISR (Incremental Static Regeneration)**
```typescript
export const revalidate = 300; // 5 minutes

export async function getLeaderboard() {
  // Cached for 5 minutes, revalidated in background
  return db.query('SELECT * FROM leaderboard');
}
```

**Layer 2: Stale-While-Revalidate**
```typescript
return Response.json(data, {
  headers: {
    'Cache-Control': 'public, s-maxage=300, stale-while-revalidate=600'
  }
});
```

**Layer 3: Edge Caching (CDN)**
- Static assets: Cache for 1 year
- Team data: Cache for 24 hours
- Leaderboards: Cache for 5 minutes
- User brackets: Never cache (private data)

### Decision: **ISR + SWR Headers**

**Cache Strategy by Endpoint**:
| Endpoint | Strategy | TTL | Reasoning |
|----------|----------|-----|-----------|
| `/api/teams` | ISR | 24h | Teams don't change during tournament |
| `/api/games` | ISR | 1h | Schedule static, scores update periodically |
| `/api/leaderboard` | SWR | 5m | Acceptable staleness, revalidates in background |
| `/api/brackets/:id` | Private | 0 | User data, never cache |
| `/api/predictions` | ISR | 1h | Predictions recalculated periodically |

**Critical Reasoning**:
1. **Reduces DB load by 95%+**: Without caching, every leaderboard view hits database. With caching, DB hit every 5 minutes.

2. **Stale-While-Revalidate**: Users never see loading spinners. Serves stale data while fetching fresh data in background.

3. **Traffic Spikes**: During championship game, millions of concurrent users can view leaderboard without crushing database.

**Performance**:
- Without caching: 1M requests = 1M DB queries
- With caching: 1M requests = ~200 DB queries (5min TTL)

---

## 9. Database Best Practices (NEW ADDITIONS)

### 9.1 Optimistic Locking

**Problem**: User opens bracket on phone and laptop, edits both. Which version wins?

**Solution: Version Column**
```sql
ALTER TABLE brackets ADD COLUMN version INTEGER DEFAULT 1;

-- Update only if version matches (optimistic lock)
UPDATE brackets
SET picks = $1, version = version + 1
WHERE id = $2 AND version = $3;
```

**UI**: Show conflict resolution modal if update fails.

---

### 9.2 Leaderboard Performance

**Problem**: 100k locked brackets, need top 100 + user's rank. Real-time query takes 500-2000ms.

**Solution: Materialized View**
```sql
CREATE MATERIALIZED VIEW bracket_leaderboard AS
SELECT
  bracket_id,
  user_id,
  points_earned,
  DENSE_RANK() OVER (ORDER BY points_earned DESC) as rank
FROM brackets
WHERE locked = true AND season = 2024;

-- Refresh after scoring (a few times per day)
REFRESH MATERIALIZED VIEW CONCURRENTLY bracket_leaderboard;
```

**Performance**: <50ms for 100k brackets (vs 500-2000ms real-time).

---

### 9.3 Historical Data

**Problem**: Keep brackets from previous years? Impact on query performance?

**Solution: Partitioning by Season**
```sql
CREATE TABLE brackets (...) PARTITION BY RANGE (season);
CREATE TABLE brackets_2024 PARTITION OF brackets FOR VALUES FROM (2024) TO (2025);
CREATE TABLE brackets_2025 PARTITION OF brackets FOR VALUES FROM (2025) TO (2026);
```

**Benefits**:
- Current season queries only scan one partition (10-100x faster)
- Easy archival (detach old partitions)
- Smaller indexes per partition

---

### 9.4 Search

**Problem**: Users search for teams "duke blue devils"

**Solution: Simple LIKE Query**
```sql
SELECT * FROM teams
WHERE LOWER(school) LIKE '%duke%'
LIMIT 10;
```

**Why not pg_trgm or Algolia?**
- Only 64 teams (overkill for full-text search)
- Simple LIKE executes in <1ms
- Don't overcomplicate

**If needed later**: Add pg_trgm for typo tolerance.

---

## Summary of Revisions

| Component | Original | Revised | Impact |
|-----------|----------|---------|--------|
| **Framework** | TanStack Start | Next.js 15 | Production-ready, handles traffic spikes |
| **ORM** | Drizzle | Prisma | Better team DX, automated migrations |
| **Auth** | Clerk | Supabase Auth | 35% cheaper at scale, less vendor lock-in |
| **Database** | Supabase | Neon | Serverless auto-scaling, branching |
| **Data Model** | JSONB | Normalized tables | 10-100x faster analytics queries |
| **Bracket UI** | Build from scratch | @g-loot library | Saves 1-2 weeks development time |
| **Rate Limiting** | None | Upstash Redis | Essential for traffic spikes |
| **Caching** | None | ISR + SWR | Reduces DB load by 95%+ |
| **Locking** | None | Version column | Prevents concurrent edit conflicts |
| **Leaderboard** | Real-time query | Materialized view | 10-50x faster |
| **Historical Data** | Not specified | Partitioning | 10-100x faster current season queries |

---

## Risk Assessment

### High-Confidence Decisions
- ✅ Next.js 15 (battle-tested for traffic spikes)
- ✅ Prisma (team productivity critical)
- ✅ Normalized data model (analytics requirements clear)
- ✅ Rate limiting (essential for traffic spikes)
- ✅ Caching (reduces DB load dramatically)

### Medium-Confidence Decisions
- ⚠️ Neon vs Supabase (both excellent, Neon slight edge for serverless)
- ⚠️ Supabase Auth vs Clerk (cost vs DX trade-off)
- ⚠️ Bracket library (may need customization)

### Monitoring Required
- 📊 Next.js security patches (stay updated)
- 📊 Upstash rate limit free tier (may need paid plan at scale)
- 📊 Neon compute-hours (may exceed free tier during March)

---

## Cost Estimates

### MVP (Free Tier)
- Vercel: $0 (hobby)
- Neon: $0 (100 compute-hours/month)
- Supabase Auth: $0 (50k MAU)
- Upstash Redis: $0 (10k requests/day)
- **Total: $0/month**

### Moderate Success (50k users, 200k brackets)
- Vercel: $20/month (pro plan)
- Neon: $10-30/month (compute overage)
- Supabase Auth: $0 (under 50k MAU)
- Upstash Redis: $10/month
- **Total: $40-60/month**

### Viral Success (500k users, 2M brackets)
- Vercel: $20/month
- Neon: $100-200/month (compute + storage)
- Supabase Auth: $1,462/month (450k MAU @ $0.00325 each)
- Upstash Redis: $50/month
- **Total: $1,632-1,732/month**

**Cost Drivers**: Auth is biggest cost at scale. Consider NextAuth.js if costs exceed budget.

---

## Next Steps

1. ✅ Update SYSTEM_DESIGN.md with all revisions
2. ✅ Update ISSUES.md with new tech stack
3. ⏭️ Begin implementation with Issue #1 (Next.js setup)

**Last Updated**: 2026-03-08
**Review Status**: Comprehensive research completed
