---
title: "Building an AI Marketing Platform with MCP: Technical Deep-Dive"
date: 2026-04-13
layout: post
---

# Building an AI Marketing Platform with MCP: Technical Deep-Dive

I spent 6 months building Minnal, an AI-native marketing platform that works inside Claude Desktop via Model Context Protocol (MCP).

Here's everything I learned about building conversational tools that actually work.

## Why MCP Instead of a Traditional Web App?

**The problem with dashboards:** Context switching kills productivity.

Every time you open Buffer, Hootsuite, or any SaaS dashboard, you're:
1. Switching contexts (from building to marketing)
2. Clicking through forms (platform, time, content)
3. Remembering your marketing strategy
4. Interpreting analytics yourself

**The MCP alternative:** Work where you already are (Claude Desktop).

Instead of opening another app, you just ask:
- "Plan this week's marketing for my product"
- "How did last month perform?"
- "Schedule this to Reddit tomorrow at 2pm"

One conversation = complete marketing workflow.

## What is MCP?

**Model Context Protocol (MCP)** is Anthropic's open protocol for connecting Claude to external tools and data sources.

Think of it as REST APIs, but for AI assistants.

**Traditional approach:**
```
User → Web App → Database → User
```

**MCP approach:**
```
User → Claude Desktop → MCP Server → Your Backend → Claude Desktop → User
```

Claude becomes the interface. Your MCP server provides capabilities.

## Architecture Overview

```
┌─────────────────────────────────┐
│      CLAUDE DESKTOP             │
│  Natural language interface     │
└────────────┬────────────────────┘
             │ MCP Protocol (SSE/HTTP)
             ▼
┌─────────────────────────────────┐
│      MCP SERVER (Node.js)       │
│  ┌──────────┐  ┌─────────────┐  │
│  │  Tools   │  │  Scheduler  │  │
│  │          │  │             │  │
│  │ • post   │  │ • Temporal  │  │
│  │ • queue  │  │ • Cron      │  │
│  │ • geo    │  │ • Retry     │  │
│  └──────────┘  └─────────────┘  │
│  ┌───────────────────────────┐  │
│  │   PostgreSQL              │  │
│  │   (credentials, posts)    │  │
│  └───────────────────────────┘  │
└────────────┬────────────────────┘
             │ REST APIs
             ▼
┌─────────────────────────────────┐
│   SOCIAL PLATFORMS              │
│  • Reddit  • LinkedIn           │
│  • Facebook • Instagram         │
└─────────────────────────────────┘
```

## The Tech Stack

**Why I Chose Each Piece:**

### TypeScript + Node.js
**Why:** MCP SDK is TypeScript-first. Fast iteration. Huge ecosystem.

**Alternative considered:** Python (great AI libraries, but worse for real-time systems)

### PostgreSQL
**Why:** Boring technology that won't fail. ACID transactions. Great TypeScript support.

**Schema:**
```sql
CREATE TABLE scheduled_posts (
  id UUID PRIMARY KEY,
  brand_id UUID REFERENCES brands(id),
  platform TEXT NOT NULL,
  content TEXT NOT NULL,
  scheduled_for TIMESTAMPTZ NOT NULL,
  status TEXT DEFAULT 'pending',
  pillar TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_scheduled_for ON scheduled_posts(scheduled_for)
  WHERE status = 'pending';
```

**Alternative considered:** MongoDB (overkill for this use case)

### Temporal.io
**Why:** Handles distributed workflow complexity better than cron.

**Use case:** Scheduling posts with retries, error handling, timezone math.

**Workflow:**
```typescript
export async function publishPostWorkflow(postId: string): Promise<void> {
  // Wait until scheduled time
  await sleep(msUntilScheduledTime);
  
  // Publish with retries
  await retry(
    () => publishToSocialPlatform(postId),
    { maxAttempts: 3, backoff: 'exponential' }
  );
  
  // Update status
  await updatePostStatus(postId, 'published');
}
```

**Alternative considered:** Bull (Redis-based queue) — good, but Temporal handles complexity better

### Railway
**Why:** Deploy from GitHub in 30 seconds. No DevOps overhead.

**Cost:** ~$20/month for hobby projects, scales up easily.

**Alternative considered:** AWS (powerful but requires constant tweaking)

## Building the MCP Server

### Step 1: Define Your Tools

MCP exposes "tools" that Claude can call. Each tool = one capability.

**Example tool definition:**
```typescript
import { MCPServer } from '@anthropic/mcp-sdk';

const server = new MCPServer({
  name: 'minnal-marketing',
  version: '1.0.0',
});

server.registerTool({
  name: 'schedule_post',
  description: 'Schedule a post for future publishing',
  parameters: {
    type: 'object',
    properties: {
      brand_id: { 
        type: 'string',
        description: 'Brand/project ID'
      },
      platform: { 
        type: 'string',
        enum: ['linkedin', 'reddit', 'facebook', 'instagram']
      },
      content: { 
        type: 'string',
        description: 'Post content'
      },
      scheduled_for: { 
        type: 'string',
        format: 'date-time',
        description: 'ISO 8601 datetime'
      },
      subreddit: {
        type: 'string',
        description: 'Required for Reddit'
      }
    },
    required: ['brand_id', 'platform', 'content', 'scheduled_for'],
  },
  handler: async (params, context) => {
    // Validate
    if (params.platform === 'reddit' && !params.subreddit) {
      throw new Error('Subreddit required for Reddit posts');
    }
    
    // Insert into DB
    const post = await db.scheduledPosts.create({
      brandId: params.brand_id,
      platform: params.platform,
      content: params.content,
      scheduledFor: new Date(params.scheduled_for),
      subreddit: params.subreddit,
    });
    
    // Start Temporal workflow
    await temporal.client.workflow.start(publishPostWorkflow, {
      workflowId: `post-${post.id}`,
      args: [post.id],
    });
    
    return {
      success: true,
      post_id: post.id,
      message: `Scheduled for ${params.scheduled_for}`,
    };
  },
});
```

### Step 2: OAuth Token Management

Each social platform has different OAuth flows. Abstraction is key.

**Abstract provider interface:**
```typescript
interface SocialProvider {
  authenticate(code: string): Promise<TokenPair>;
  refreshToken(refreshToken: string): Promise<TokenPair>;
  publishPost(post: Post, token: string): Promise<PublishResult>;
  getAnalytics(postId: string, token: string): Promise<Analytics>;
}
```

**Reddit implementation:**
```typescript
class RedditProvider implements SocialProvider {
  async authenticate(code: string): Promise<TokenPair> {
    const response = await fetch('https://www.reddit.com/api/v1/access_token', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Authorization': `Basic ${base64(clientId:clientSecret)}`,
      },
      body: new URLSearchParams({
        grant_type: 'authorization_code',
        code,
        redirect_uri: REDDIT_REDIRECT_URI,
      }),
    });
    
    const data = await response.json();
    
    return {
      accessToken: data.access_token,
      refreshToken: data.refresh_token,
      expiresAt: Date.now() + (data.expires_in * 1000),
    };
  }
  
  async refreshToken(refreshToken: string): Promise<TokenPair> {
    // Similar to authenticate, but with refresh_token grant
  }
  
  async publishPost(post: Post, token: string): Promise<PublishResult> {
    const response = await fetch('https://oauth.reddit.com/api/submit', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: new URLSearchParams({
        sr: post.subreddit,
        kind: 'self',
        title: extractTitle(post.content),
        text: post.content,
      }),
    });
    
    return await response.json();
  }
}
```

**Token encryption:**
```typescript
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY!, 'hex');

function encrypt(text: string): { encrypted: string; iv: string; tag: string } {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, iv);
  
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  const tag = cipher.getAuthTag();
  
  return {
    encrypted,
    iv: iv.toString('hex'),
    tag: tag.toString('hex'),
  };
}

function decrypt(encrypted: string, iv: string, tag: string): string {
  const decipher = crypto.createDecipheriv(
    ALGORITHM,
    KEY,
    Buffer.from(iv, 'hex')
  );
  
  decipher.setAuthTag(Buffer.from(tag, 'hex'));
  
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  
  return decrypted;
}
```

### Step 3: GEO Analysis Engine

The differentiator: tracking LLM visibility.

**High-level approach:**
```typescript
async function checkLLMVisibility(brand: Brand): Promise<GEOReport> {
  const queries = generateQueries(brand.keywords);
  const results = await Promise.all([
    queryLLM('chatgpt', queries),
    queryLLM('claude', queries),
    queryLLM('perplexity', queries),
    queryLLM('gemini', queries),
  ]);
  
  return {
    chatgpt: analyzeVisibility(results[0], brand.name),
    claude: analyzeVisibility(results[1], brand.name),
    perplexity: analyzeVisibility(results[2], brand.name),
    gemini: analyzeVisibility(results[3], brand.name),
  };
}

function generateQueries(keywords: string[]): string[] {
  const templates = [
    "What's a good {keyword}?",
    "Best {keyword} for indie hackers",
    "{keyword} alternatives",
    "How to choose a {keyword}",
  ];
  
  return keywords.flatMap(kw =>
    templates.map(t => t.replace('{keyword}', kw))
  );
}

async function queryLLM(model: string, queries: string[]): Promise<Response[]> {
  // Rate limit: 1 request/second
  const results = [];
  
  for (const query of queries) {
    const response = await fetch(API_ENDPOINTS[model], {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${API_KEYS[model]}` },
      body: JSON.stringify({ prompt: query }),
    });
    
    results.push(await response.json());
    await sleep(1000); // Rate limiting
  }
  
  return results;
}

function analyzeVisibility(responses: Response[], brandName: string): VisibilityScore {
  let mentions = 0;
  let totalQueries = responses.length;
  const positions: number[] = [];
  
  for (const response of responses) {
    const text = response.text.toLowerCase();
    const brandLower = brandName.toLowerCase();
    
    if (text.includes(brandLower)) {
      mentions++;
      
      // Estimate position (1st, 2nd, 3rd recommendation)
      const position = estimatePosition(text, brandLower);
      if (position) positions.push(position);
    }
  }
  
  return {
    score: (mentions / totalQueries) * 100,
    mentions,
    totalQueries,
    averagePosition: positions.length > 0 
      ? positions.reduce((a, b) => a + b) / positions.length 
      : null,
  };
}
```

### Step 4: Content Gap Analysis

Compare your content vs competitors to find opportunities.

**Approach:**
```typescript
async function analyzeContentGaps(brand: Brand): Promise<Gap[]> {
  // 1. Scrape competitor blog titles
  const competitorUrls = brand.competitors;
  const competitorContent = await Promise.all(
    competitorUrls.map(url => scrapeBlogs(url))
  );
  
  // 2. Cluster by topic using embeddings
  const allTitles = competitorContent.flat();
  const embeddings = await getEmbeddings(allTitles);
  const clusters = clusterByTopic(embeddings);
  
  // 3. Compare coverage
  const yourContent = await scrapeBlogs(brand.blogUrl);
  const yourTopics = new Set(
    await classifyIntoTopics(yourContent, clusters)
  );
  
  // 4. Find gaps
  const gaps = [];
  for (const cluster of clusters) {
    if (!yourTopics.has(cluster.id)) {
      gaps.push({
        topic: cluster.name,
        competitorCoverage: cluster.titles.length,
        suggestedPost: generatePostSuggestion(cluster),
        geoTactic: matchGEOTactic(cluster),
      });
    }
  }
  
  return gaps.sort((a, b) => 
    b.competitorCoverage - a.competitorCoverage
  );
}

async function getEmbeddings(texts: string[]): Promise<number[][]> {
  const response = await fetch('https://api.openai.com/v1/embeddings', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${OPENAI_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'text-embedding-3-small',
      input: texts,
    }),
  });
  
  const data = await response.json();
  return data.data.map(d => d.embedding);
}
```

## Hardest Technical Challenges

### 1. OAuth Token Refresh Timing

**Problem:** Each platform expires tokens differently. Some at fixed intervals, some at usage-based intervals.

**Solution:** Proactive refresh before expiration + fallback refresh on 401 errors.

```typescript
async function ensureFreshToken(credential: Credential): Promise<string> {
  const expiresIn = credential.expiresAt - Date.now();
  const REFRESH_BUFFER = 5 * 60 * 1000; // 5 minutes
  
  if (expiresIn < REFRESH_BUFFER) {
    const provider = getProvider(credential.platform);
    const newTokens = await provider.refreshToken(credential.refreshToken);
    
    await db.credentials.update(credential.id, {
      accessToken: encrypt(newTokens.accessToken),
      expiresAt: newTokens.expiresAt,
    });
    
    return newTokens.accessToken;
  }
  
  return decrypt(credential.accessToken);
}
```

### 2. GEO Analysis Accuracy

**Problem:** LLMs mention products in various forms (full name, abbreviation, description).

**Solution:** Fuzzy matching + context analysis.

```typescript
function detectMention(text: string, brandName: string): boolean {
  const variations = [
    brandName.toLowerCase(),
    brandName.toLowerCase().replace(/\s/g, ''),
    // Common abbreviations
    brandName.split(' ').map(w => w[0]).join('').toLowerCase(),
  ];
  
  const textLower = text.toLowerCase();
  
  // Exact match
  for (const variant of variations) {
    if (textLower.includes(variant)) {
      return true;
    }
  }
  
  // Fuzzy match with Levenshtein distance
  const words = textLower.split(/\s+/);
  for (const word of words) {
    if (levenshteinDistance(word, brandName.toLowerCase()) <= 2) {
      return true;
    }
  }
  
  return false;
}
```

### 3. MCP Error Handling UX

**Problem:** Claude Desktop expects graceful failures, not stack traces.

**Solution:** Structured error responses with user-friendly messages.

```typescript
server.registerTool({
  name: 'schedule_post',
  handler: async (params, context) => {
    try {
      // Validation
      if (params.platform === 'reddit' && !params.subreddit) {
        return {
          success: false,
          error: {
            code: 'MISSING_SUBREDDIT',
            message: 'Reddit posts require a subreddit. Try: schedule_post(..., subreddit: "SideProject")',
            hint: 'Popular subreddits: SideProject, IndieHackers, SaaS',
          },
        };
      }
      
      // Execute
      const result = await schedulePost(params);
      
      return {
        success: true,
        data: result,
      };
      
    } catch (error) {
      // Never expose stack traces to Claude
      return {
        success: false,
        error: {
          code: 'INTERNAL_ERROR',
          message: 'Failed to schedule post. Please try again or contact support if this persists.',
          debug: process.env.NODE_ENV === 'development' ? error.message : undefined,
        },
      };
    }
  },
});
```

## Performance Optimizations

### 1. Database Connection Pooling

**Problem:** MCP servers get hit with bursts of requests.

**Solution:** Connection pooling with limits.

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: DATABASE_URL,
  max: 20, // Maximum connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

### 2. Caching GEO Results

**Problem:** Querying multiple LLMs is slow (30-60 seconds).

**Solution:** Cache for 7 days, background refresh.

```typescript
async function getGEOContext(brandId: string): Promise<GEOReport> {
  const cached = await redis.get(`geo:${brandId}`);
  
  if (cached) {
    const data = JSON.parse(cached);
    
    // Refresh in background if older than 7 days
    if (Date.now() - data.timestamp > 7 * 24 * 60 * 60 * 1000) {
      refreshGEOInBackground(brandId);
    }
    
    return data;
  }
  
  // No cache, fetch now
  const report = await checkLLMVisibility(brandId);
  await redis.set(`geo:${brandId}`, JSON.stringify({
    ...report,
    timestamp: Date.now(),
  }));
  
  return report;
}
```

## Deployment

**Railway config:**
```toml
[build]
builder = "nixpacks"
buildCommand = "npm run build"

[deploy]
startCommand = "npm start"
restartPolicyType = "on-failure"
restartPolicyMaxRetries = 3

[[services]]
name = "mcp-server"
port = 3000

[[services]]
name = "temporal-worker"
command = "npm run worker"
```

**Environment variables:**
```bash
DATABASE_URL=postgresql://...
ENCRYPTION_KEY=...
REDDIT_CLIENT_ID=...
REDDIT_CLIENT_SECRET=...
LINKEDIN_CLIENT_ID=...
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
```

## Lessons Learned

### 1. MCP is amazing for conversational tools

Natural language > forms for complex workflows. Users love it.

### 2. OAuth is harder than you think

Every platform has quirks. Abstract early, test extensively.

### 3. Error messages matter 10x more with AI

Claude shows errors to users. Make them helpful, not technical.

### 4. Boring technology wins

PostgreSQL, Node.js, TypeScript — nothing fancy. It just works.

### 5. Temporal > Cron for anything complex

Retries, error handling, timezone math — Temporal handles it all.

## Open Source Plans

**Planning to open source:**
- Core MCP server template
- OAuth abstraction layer
- GEO analysis engine

**Why:** Lower barrier for others building MCP tools.

**When:** Q3 2026

## Try It

Minnal is live and built on this stack.

**Check it out:** [minnal.io](https://minnal.io)

**Connect via MCP:** Add to Claude Desktop and start planning your marketing in natural language.

---

*Questions? Hit me up on Twitter @moongastudio or comment on r/IndieHackers*
