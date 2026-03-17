# BrandSocial CMO Agent Architecture Spec

## Context

BrandSocial is a full-featured social media management platform (Next.js 16 + Supabase + Inngest) with 6 platform integrations (Instagram, Facebook, LinkedIn, TikTok, Threads, Google Business Profile), AI-powered content generation, design editing (CESDK), analytics, and scheduling. AI capabilities are currently spread across disconnected endpoints on a Python backend (`brand-social-graph` agent, `/get-copy-v2`, `/v2/get-hashtags`, `/searchForBestMatchAsset`, etc.) and Inngest workflows. The goal is to consolidate into a **Central CMO Agent** that orchestrates specialized sub-agents, enabling a unified conversational AI experience for social media managers.

### Related Repos
- **App**: `/Users/saad/brandsocial-app` — Next.js frontend + API routes + Inngest functions
- **BrandSocial AI**: Python AI backend — LLM module (OpenAI, Anthropic, Bedrock, Gemini via LiteLLM), vector indexing, agent graph endpoints

---

## 1. Central CMO Agent (Orchestrator)

The single entry point for all AI interactions. Receives user intent (chat, UI action, or background trigger), assembles brand context, and delegates to the right sub-agent(s).

### Intent Routing

| User Intent | Primary Agent | Secondary Agent(s) |
|---|---|---|
| "Create a content calendar for April" | Content Planner | Brand Intelligence (voice/guidelines) |
| "Write a caption for this image" | Content Creator | Brand Intelligence (tone) |
| "How did my posts perform last week?" | Analytics | — |
| "Adapt this post for LinkedIn and TikTok" | Publishing & Distribution | Content Creator (caption adaptation) |
| "Generate 10 posts from my calendar" | Content Creator (PowerMode) | Brand Intelligence, Publishing |
| "Analyze my competitor @brandX" | Analytics | Content Planner (strategic recs) |
| UI: Schedule post | Publishing & Distribution | — |
| UI: Train AI on reviews | Brand Intelligence | — |

### Multi-Agent Orchestration Patterns
- **Sequential**: Planner generates calendar → Creator generates posts from it (current `flatplan` → `convert-flatplan-ideas-to-posts`)
- **Parallel Fan-Out**: PowerMode generates multiple posts simultaneously (current `callPowerModeAI` with `additional_run_ids`)
- **Feedback Loop**: Analytics feeds performance data back into Planner for next month's strategy

### Brand Context Bundle
Every agent invocation gets a pre-assembled context:
- Brand attributes (color palette, typography, personality, tone, values, mission, target audience) — from `brand_attribute`
- Brand input data (audit data, business goals, guidelines, social media content prefs) — from `brand_input_attribute`
- Connected channels with handles/bios — from `brand_channel`
- Recent performance summary (top posts, engagement rates, best times) — from Tinybird
- AI training results (brand voice profile) — from `brand.ai_training_data`
- Relevant brand knowledge (retrieved docs/guidelines) — from vector store (RAG)
- Conversation history — from `agent_message`

### Research Items
- **R1.1**: Build orchestrator in Python backend (extending `brand-social-graph`) vs. TypeScript agent layer (Vercel AI SDK)?
- **R1.2**: **Phase out LangGraph/LangChain → Agno** — evaluate Agno as the agent framework for orchestration. Agno is lighter-weight, model-agnostic, and avoids LangChain's abstraction overhead. Research: Agno's multi-agent coordination primitives, tool definition patterns, and state management compared to current LangGraph usage.
- **R1.3**: Chat UI design — persistent side panel (Copilot-style) vs. full-page experience?
- **R1.4**: Model routing strategy for cost optimization across sub-agents (different agents may need different model tiers — e.g. Haiku for intent classification, Sonnet/GPT-4o for content creation)
- **R1.5**: Evaluate LiteLLM for multi-provider model routing to avoid vendor lock-in
- **R1.6**: **Langfuse for agent tracing** — integrate Langfuse as the observability layer for all agent interactions. Research: trace propagation across CMO orchestrator → sub-agents → tool calls, cost tracking per agent/brand, latency monitoring, prompt versioning, and evaluation/scoring pipelines within Langfuse.

---

## 2. Content Planner Agent

### Existing (REUSE)
| Capability | Location |
|---|---|
| Flatplan generation | `inngest/functions/flatplan.ts`, `generate-specific-flatplan.ts` → Python `/getFlatPlan` |
| Website audit/brand research | `inngest/functions/fetch-audit-data.ts` → Python `/extractAuditData` |
| Content pillars | Categories: inspirational, educational, testimonial, informational |
| Idea regeneration | `/api/multiselect-idea-generate` → Python backend |
| Holiday idea tweaking | `/api/tweak-holiday-ideas` |
| Brand summary | Python `/getBrandSummary` |
| Document summarization | `inngest/functions/document-summary.ts` → Python `/summaryDocument` |

### New Capabilities
- **Competitor analysis tool** — accept competitor social handles, analyze their public content (mix, frequency, engagement)
- **Real-time trend monitoring** — continuously surface trending topics, viral formats, and content opportunities from X/Twitter, TikTok, Google Trends, and industry news
- **Content gap analysis** — compare posted content vs. what performs well in niche (requires Analytics Agent data)
- **Conversational calendar editing** — "Move educational posts to Wednesdays", "Add more video content", "Replace product posts with UGC ideas"
- **Brand data upload processing** — parse uploaded PDFs/brand books/product catalogs, summarize, index into vector store
- **Opportunity alerts** — proactive notifications when a trending topic aligns with the brand's niche ("X topic is trending in your industry — here's a post idea")

### Brand Research Capabilities

The Content Planner needs deep research capabilities to build informed content strategies. This is a multi-source intelligence system:

#### Existing Social Content Analysis
- Pull all published content from connected channels (Instagram, Facebook, LinkedIn, TikTok, Threads, GBP) via existing platform APIs
- Analyze content patterns: posting frequency, content types, engagement by category, caption length patterns, hashtag effectiveness, best media types
- Identify the brand's top 10% performing content and extract what makes it work (format, topic, tone, time posted, visual style)
- Detect content fatigue — topics/formats with declining engagement over time
- Map content to funnel stages (awareness → consideration → conversion) and identify imbalances

#### Web Research & Industry Intelligence
- Crawl the brand's website to understand products, services, messaging, tone, and value propositions (extends existing `/extractAuditData`)
- Monitor industry blogs, news sites, and publications for trending topics and narratives in the brand's vertical
- Track relevant subreddits, forums, and community discussions for audience pain points and interests
- Analyze industry reports and whitepapers for content themes (agent summarizes and extracts key talking points)

#### Competitor Research
- Accept competitor social handles across all platforms — pull their public content and engagement metrics
- Analyze competitor content strategy: posting frequency, content mix, engagement rates, top performing posts, hashtag strategy, visual style
- Identify competitor content gaps (topics they're not covering that the brand could own)
- Track competitor campaign launches and seasonal content patterns
- Benchmark: side-by-side comparison of brand vs. competitors on key metrics

#### Trend & Opportunity Detection
- **X/Twitter via Grok API** — real-time trending topics, viral conversations, breaking news relevant to the brand's industry. Use Grok's real-time knowledge to identify emerging narratives before they peak.
- **TikTok trending** — trending sounds, effects, formats, and hashtags via TikTok's discover/trending endpoints. Surface viral content formats the brand could adapt.
- **Google Trends** — search interest trends by topic, geography, and time. Identify rising queries related to the brand's products/services.
- **Instagram Explore / Reels trends** — trending audio, formats, and content styles on Instagram
- **LinkedIn trending articles** — professional/B2B trending topics for LinkedIn-focused brands
- **Reddit/community signals** — emerging discussions and sentiment shifts in relevant communities
- **News monitoring** — RSS feeds, Google News API, or news aggregation for industry-specific events, product launches, regulatory changes that create content opportunities
- **Seasonal & cultural calendar** — holidays, awareness months, cultural events, industry conferences (extends existing `holidays` table)

### Tools
| Tool | Status | Details |
|---|---|---|
| `generate_flatplan` | REUSE | Calls `/getFlatPlan` via Inngest |
| `fetch_audit_data` | REUSE | Calls `/extractAuditData` |
| `get_brand_summary` | REUSE | Calls `/getBrandSummary` |
| `tweak_holiday_ideas` | REUSE | Existing endpoint |
| `multiselect_idea_generate` | REUSE | Calls `/api/multiselect-idea-generate` |
| `analyze_own_content` | NEW | Deep analysis of brand's published content across all channels — patterns, top performers, fatigue detection |
| `analyze_competitors` | NEW | Pull competitor public content, benchmark engagement, identify their strategy and gaps |
| `search_web` | NEW | General web research — crawl URLs, search industry topics, summarize articles |
| `get_trending_x` | NEW | Grok/X API — real-time trending topics, viral conversations, breaking news by industry |
| `get_trending_tiktok` | NEW | TikTok trending sounds, formats, hashtags |
| `get_trending_google` | NEW | Google Trends data by topic, geography, time period |
| `get_trending_topics` | NEW | Aggregated trending across platforms, filtered by brand's industry/vertical |
| `monitor_industry_news` | NEW | RSS/news API monitoring for industry events, launches, regulatory changes |
| `analyze_content_gaps` | NEW | Cross-reference brand content with competitor content and industry benchmarks |
| `detect_opportunities` | NEW | Match trending topics against brand's niche, products, and content pillars — return actionable content ideas |
| `process_brand_document` | EXTEND | Extend `document/summary` to also extract guidelines + index into vector store |
| `modify_calendar` | NEW | Conversational calendar editing |

### Research Items
- **R2.1**: Keep flatplan generation as Inngest background job vs. streaming agent output (SSE/WebSocket) for real-time conversational building?
- **R2.2**: Competitor analysis data sources — platform APIs (limited for public data), web scraping (Apify, Browserbase), third-party APIs (Socialinsider, Sprout Social, Phyllo)
- **R2.3**: **Grok API for X/Twitter trend intelligence** — evaluate xAI's Grok API for real-time trending topic detection. Key questions: rate limits, cost, latency, quality of industry-filtered results vs. raw trending. Can Grok summarize "what's trending in [industry] right now" with context?
- **R2.4**: **TikTok trend detection** — evaluate TikTok's Research API and Creative Center API for trending sounds, hashtags, and content formats. Alternative: scrape TikTok's Discover page via headless browser.
- **R2.5**: **Google Trends API** — evaluate official API (limited) vs. third-party wrappers (SerpAPI Google Trends, Pytrends) for programmatic trend data by vertical.
- **R2.6**: Expand content pillars beyond current 4 — add: promotional, user-generated, behind-the-scenes, entertainment, thought-leadership
- **R2.7**: Evaluate knowledge graph approaches for modeling relationships between brand attributes, pillars, and audience segments
- **R2.8**: **Existing content deep analysis** — research how to efficiently pull and analyze all historical published content from connected channels. Consider: rate limits on platform APIs for historical data, storing content embeddings for semantic analysis, clustering content by topic/format.
- **R2.9**: **Web research tooling** — evaluate tools for agent-driven web research: Tavily (search API built for LLMs), Perplexity API (research-grade search), Firecrawl (web scraping for LLMs), Jina Reader (URL → clean text). Which combination gives best results for brand/industry research?
- **R2.10**: **Opportunity detection pipeline** — design the system that continuously monitors trends and matches against brand context to generate proactive content ideas. Should this run on a cron (daily/hourly Inngest job) or on-demand when the user opens the planner?
- **R2.11**: **Reddit/community monitoring** — evaluate Reddit API, Snoowrap, or third-party social listening tools (Brandwatch, Mention) for tracking community discussions relevant to the brand's niche.
- **R2.12**: **News monitoring** — evaluate news APIs (NewsAPI, Google News RSS, Bing News API) for industry-specific news tracking. How to filter signal from noise for content opportunities?
- **R2.13**: **Content funnel mapping** — research frameworks for automatically categorizing content by funnel stage (TOFU/MOFU/BOFU) and ensuring calendar balance across the funnel.

---

## 3. Content Creator Agent

### Existing (REUSE)
| Capability | Location |
|---|---|
| Caption generation | Python `/get-copy-v2` with reasoning levels (low/medium/high) |
| Hashtag generation | `src/lib/ai-api/hashtags.ts` → Python `/v2/get-hashtags` |
| Magic template search | `src/lib/magic-templates/actions.ts` → Python `/searchForBestMatchAsset` |
| PowerMode batch creation | `src/utils/powermode-graph-agent-call.ts` → Python `/api/brand-social-graph/call_agent` |
| Post duplication/adaptation | `src/utils/duplicate-post-graph-agent.ts` → Python `/api/brand-social-graph/call_agent_post_duplication` |
| Image generation | FAL.ai (RecraftV3) via `/api/fal/`, Kling AI (video) via `/api/kling/` |
| Stock images | Pexels, Pixabay, Freepik, Yay via `/api/stock-images/` |
| Template text/image fills | `/api/magic-templates/text-fill/`, `ai-fills.ts` |
| Asset indexing | `src/lib/ai-api/indexAIAsset.ts` → Python `/ingestAssetInIndex` |

### New Capabilities
- **Unified content creation** — single brief ("Create an educational Instagram carousel about our product launch") orchestrates caption + hashtags + image + template selection in one flow
- **Format-aware generation** — different outputs for post vs. story vs. reel vs. carousel (slide count, aspect ratios, etc.)
- **Iterative refinement** — "Make it more casual", "Use a different image style", "Shorter caption" via conversational editing
- **Video script generation** — scripts for Reels/TikTok with hook, body, CTA structure
- **A/B variant generation** — generate 2-3 variants for user selection or automated testing

### Tools
| Tool | Status | Details |
|---|---|---|
| `generate_caption` | REUSE | Calls `/get-copy-v2` |
| `generate_hashtags` | REUSE | Calls `/v2/get-hashtags` |
| `generate_magic_template` | REUSE | Calls `/searchForBestMatchAsset` |
| `generate_image` | REUSE | FAL.ai RecraftV3 |
| `generate_video` | REUSE | Kling AI |
| `powermode_batch_create` | REUSE | Calls `/api/brand-social-graph/call_agent` |
| `duplicate_post` | REUSE | Calls `/api/brand-social-graph/call_agent_post_duplication` |
| `text_fill_template` | REUSE | `/api/magic-templates/text-fill/` |
| `search_stock_images` | REUSE | Pexels, Pixabay, Freepik, Yay |
| `generate_video_script` | NEW | LLM tool for Reels/TikTok scripts |
| `create_carousel_slides` | NEW | Orchestrates multiple template generations |
| `refine_content` | NEW | Takes content + feedback, produces refined version |

### Research Items
- **R3.1**: Consolidate all creation into single agent call vs. keep as individual tools the agent orchestrates?
- **R3.2**: CESDK programmatic API for agent-driven template manipulation without user opening editor
- **R3.3**: Multi-modal models (GPT-4o, Claude vision) for agent to "see" generated templates and self-correct
- **R3.4**: A/B variant generation patterns and automated testing framework
- **R3.5**: Leverage `proxy-provider` endpoint as unified image generation router (RecraftV3, DALL-E, Flux, etc.)

---

## 4. Analytics Agent

### Existing (REUSE)
| Capability | Location |
|---|---|
| Platform analytics fetching | `inngest/functions/instagram-post-analytics.ts`, `facebook-post-analytics.ts`, `linkedin-post-analytics.ts`, `tiktok-analytics-dispatcher.ts` |
| Tinybird time-series | `src/lib/analytics/tinybirdClient.ts` — pipes for content growth per platform |
| Brand performance AI | `src/lib/brand-performance/actions.ts` → Python `/getBrandsPerformanceData` |
| Goal tracking | `inngest/functions/goal-tracking.ts` |
| GBP ROI | `src/lib/analytics/actions/gbp-roi-actions.ts` |
| Bulk analytics | `inngest/functions/bulk-analytics-fetcher.ts` |

### New Capabilities
- **Natural language analytics queries** — "Which posts got the most engagement last month?", "Compare Instagram vs LinkedIn performance"
- **Proactive insights** — periodic background analysis surfacing findings ("Educational content on LinkedIn gets 3x more engagement than promotional")
- **Audience segment analysis** — demographics, activity patterns, content preferences
- **Competitor benchmarking** — compare brand metrics against industry averages or specific competitors
- **Content attribution** — track which flatplan ideas → posts → performance, closing the planning-to-performance loop
- **ROI calculation** — map social metrics to business outcomes via UTM/tracking data

### Tools
| Tool | Status | Details |
|---|---|---|
| `fetch_channel_analytics` | REUSE | Tinybird pipes per platform |
| `fetch_brand_performance` | REUSE | `/getBrandsPerformanceData` |
| `fetch_post_metrics` | REUSE | `getPostMetrics` from tinybirdClient |
| `fetch_content_growth` | REUSE | `getPostGrowth` from tinybirdClient |
| `track_goal_progress` | REUSE | Goal tracking Inngest function |
| `analyze_performance_trends` | NEW | LLM analysis of time-series data, pattern detection |
| `compare_channels` | NEW | Cross-channel comparison with recommendations |
| `identify_top_content` | NEW | Rank content by engagement metrics |
| `generate_insights_report` | NEW | Structured insights with actionable recommendations |
| `competitor_benchmark` | NEW | Compare metrics against benchmarks |

### Research Items
- **R4.1**: NL → Tinybird query (LLM-generated analytics queries) vs. pre-computed summaries? Risk: slow/incorrect LLM SQL
- **R4.2**: Google Analytics / UTM tracking integration for attribution beyond social metrics
- **R4.3**: Cross-platform "performance score" that normalizes Instagram likes vs. LinkedIn impressions vs. TikTok views
- **R4.4**: Automated anomaly detection on analytics data (sudden engagement drops, viral posts, unusual growth)
- **R4.5**: Tinybird capacity for additional agent queries vs. secondary analytics store (ClickHouse, BigQuery)

---

## 5. Brand Intelligence Agent

### Existing (REUSE)
| Capability | Location |
|---|---|
| Brand attributes | `brand_attribute` table — color palette, typography, personality, tone, values, mission, target audience |
| Brand input attributes | `brand_input_attribute` table — audit data, goals, guidelines, social content prefs |
| AI training on reviews | `inngest/functions/ai-training.ts` → Python `/analyze_user_reviews` |
| Website audit | `inngest/functions/fetch-audit-data.ts` → Python `/extractAuditData` |
| Document summarization | `inngest/functions/document-summary.ts` → Python `/summaryDocument` |
| Asset indexing | `src/lib/ai-api/indexAIAsset.ts` → Python `/ingestAssetInIndex` |

### New Capabilities
- **Brand voice model** — persistent representation of how the brand speaks (formality, vocabulary, sentence patterns, emoji usage, CTA style). Goes beyond current `tone`/`personality` enum values
- **Brand guideline enforcement** — validate all content against brand guidelines before publishing (logo usage, color compliance, banned words, required disclaimers)
- **Visual identity model** — photography style, illustration style, layout preferences, white space usage beyond color/typography
- **Product/service knowledge base** — RAG-indexed collection of product info, pricing, features, FAQs that Content Creator queries
- **Brand consistency scoring** — score generated content on brand voice/visual adherence
- **Multi-brand isolation** — strict separation of brand knowledge between brands in multi-brand accounts
- **Brand data ingestion pipeline** — users upload guidelines PDFs, brand books, product shots, product catalogs → parsed, summarized, indexed

### Tools
| Tool | Status | Details |
|---|---|---|
| `get_brand_attributes` | REUSE | Query `brand_attribute` table |
| `get_brand_input_attributes` | REUSE | Query `brand_input_attribute` table |
| `train_on_reviews` | REUSE | `/analyze_user_reviews` via Inngest |
| `audit_website` | REUSE | `/extractAuditData` via Inngest |
| `summarize_document` | REUSE | `/summaryDocument` via Inngest |
| `index_asset` | REUSE | `/ingestAssetInIndex` |
| `validate_brand_compliance` | NEW | Check content against brand guidelines |
| `score_brand_consistency` | NEW | Rate content on voice/visual adherence |
| `query_product_knowledge` | NEW | RAG query against product knowledge base |
| `update_brand_voice_model` | NEW | Learn from approved content to refine voice model |
| `extract_brand_guidelines` | NEW | Parse uploaded brand books/PDFs into structured rules |

### Research Items
- **R5.1**: Embedding models for brand voice — fine-tune on approved content to create "brand voice vector" measuring similarity?
- **R5.2**: Evaluate vector indexing options for brand knowledge RAG store (pgvector, Vespa, Pinecone, Qdrant)
- **R5.3**: PDF/document parsing for brand guidelines — Unstructured.io, LlamaParse, or vision models for image-heavy brand books
- **R5.4**: Channel-specific voice modulation (brand says "casual" but LinkedIn should be "professional")
- **R5.5**: Multi-modal brand consistency checking — vision model verifying generated images match brand visual style

---

## 6. Publishing & Distribution Agent

### Existing (REUSE)
| Capability | Location |
|---|---|
| Post scheduling/publishing | `inngest/functions/scheduler.ts` — all 6 platforms (800+ lines) |
| Platform API clients | `src/lib/external-apis/` — facebook.ts, instagram.ts, linkedin.ts, tiktok.ts, threads/, google-business.ts |
| Post duplication/adaptation | `duplicate-post-graph-agent.ts` — adapts across brands/channels |
| Approval workflows | `src/lib/approval-flow/actions.ts`, `src/lib/post-approval/` |
| Scheduled post processing | `inngest/functions/process-scheduled-posts.ts` |
| Token management | `inngest/functions/access-token-validity.ts`, `validate-access-token-for-brands.ts` |
| Post validation | `/api/composer/validate/route.ts` |
| Dynamic fields | `src/lib/dynamic-fields/utils.ts` |

### New Capabilities
- **Optimal posting time recommendations** — use analytics data to recommend best times per channel per audience
- **Cross-platform content adaptation** — transform format (LinkedIn article → Instagram carousel → TikTok script)
- **Publishing queue management** — unified view of scheduled content with AI-suggested reordering for content mix balance
- **Automated approval routing** — route to right approval team based on content type/sensitivity
- **Post-publish monitoring** — monitor initial engagement, alert on underperformance or viral posts

### Tools
| Tool | Status | Details |
|---|---|---|
| `schedule_post` | REUSE | Inngest `scheduler.schedule` event |
| `publish_to_platform` | REUSE | Platform-specific logic in scheduler.ts |
| `validate_post` | REUSE | `/api/composer/validate/` |
| `duplicate_to_channels` | REUSE | `call_agent_post_duplication` |
| `check_token_validity` | REUSE | Token validation functions |
| `process_dynamic_fields` | REUSE | Dynamic variable substitution |
| `recommend_posting_time` | NEW | Analyze historical performance for optimal times |
| `adapt_content_for_platform` | NEW | Transform content format between platforms |
| `manage_publishing_queue` | NEW | Reorder/balance scheduled content |
| `route_for_approval` | NEW | Auto-assign approval teams based on rules |
| `monitor_post_performance` | NEW | Post-publish performance tracking |

### Research Items
- **R6.1**: Best-time-to-post algorithms — simple historical analysis, ML models, or third-party APIs?
- **R6.2**: Cross-platform adaptation in Python backend vs. TypeScript agent layer?
- **R6.3**: Webhook-based post-publish monitoring (Instagram/LinkedIn webhooks) for alerts
- **R6.4**: Content queue balancing algorithms for content pillar/format/channel mix
- **R6.5**: Inngest cron + event scheduling for "smart scheduling" that dynamically adjusts based on real-time data

---

## 7. New Tools Summary (Priority)

| Tool | Agent | Priority | Complexity |
|---|---|---|---|
| `modify_calendar` | Content Planner | P1 | Medium |
| `analyze_competitors` | Content Planner | P1 | High |
| `create_carousel_slides` | Content Creator | P1 | Medium |
| `refine_content` | Content Creator | P1 | Medium |
| `analyze_performance_trends` | Analytics | P1 | Medium |
| `generate_insights_report` | Analytics | P1 | Medium |
| `validate_brand_compliance` | Brand Intelligence | P1 | Medium |
| `query_product_knowledge` | Brand Intelligence | P1 | Medium |
| `recommend_posting_time` | Publishing | P1 | Medium |
| `get_trending_topics` | Content Planner | P2 | Medium |
| `analyze_content_gaps` | Content Planner | P2 | Medium |
| `generate_video_script` | Content Creator | P2 | Low |
| `compare_channels` | Analytics | P2 | Low |
| `score_brand_consistency` | Brand Intelligence | P2 | High |
| `extract_brand_guidelines` | Brand Intelligence | P2 | High |
| `adapt_content_for_platform` | Publishing | P2 | Medium |
| `competitor_benchmark` | Analytics | P3 | High |
| `manage_publishing_queue` | Publishing | P3 | Medium |

---

## 8. Phased Migration Plan

### Phase 1: Foundation (Weeks 1-4) — "Wire the Existing"

**Goal**: Create agent infrastructure without changing existing functionality.

1. **Define agent protocol** — TypeScript interfaces for AgentRequest, AgentResponse, ToolCall, ToolResult. Compatible with existing thread_id/run_id pattern from Python backend.
2. **Create database tables** — `agent_session`, `agent_message`, `agent_tool_invocation` in Supabase.
3. **Build Brand Context Bundle assembler** — single function pulling from `brand_attribute`, `brand_input_attribute`, `brand_channel` + caching. Replaces scattered query-param construction.
4. **Wrap existing AI API calls as tools** — `tools/` directory with thin wrappers around every existing AI call (`generateFlatplan`, `generateCaption`, `generateHashtags`, `searchMagicTemplate`, `callPowerModeAI`, `callDuplicatePostAgent`, etc.). Standard interface: `{ name, description, parameters (Zod), execute(params, context) }`.
5. **Map existing features** — document which UI actions map to which tools.

**Reuses**: Everything. This phase adds a layer; nothing existing changes.

### Phase 2: First Agents (Weeks 5-10) — "Content Creator + Brand Intelligence"

**Goal**: Build the two agents that directly improve the core workflow: creating posts.

1. **Content Creator Agent** — composes existing tools (caption + hashtag + template + image) into single orchestrated flow. Input: brief + brand context. Output: complete post draft.
2. **Brand Intelligence Agent** — wraps existing brand data + AI training + doc processing. Adds brand compliance validation.
3. **Agent chat interface** — conversational UI panel. "Create a post about our summer sale" → agent generates caption, selects template, generates image → user says "make it more playful" → agent refines.
4. **Connect to existing Inngest flows** — agent triggers PowerMode via existing Inngest pipeline.
5. **Vector store for brand knowledge** — pgvector in Supabase for brand guidelines and product knowledge RAG.

**Reuses**: `callPowerModeAI`, `/get-copy-v2`, `/v2/get-hashtags`, `/searchForBestMatchAsset`, FAL.ai proxy, AI training, document summary.
**New**: Agent orchestration, chat UI, brand compliance tool, vector store.

### Phase 3: Intelligence Layer (Weeks 11-16) — "Analytics + Content Planner"

**Goal**: Add strategic intelligence and planning capabilities.

1. **Analytics Agent** — wraps Tinybird queries + brand performance API. NL query interface. Trend analysis + insights generation.
2. **Content Planner Agent** — wraps flatplan generation. Conversational calendar editing. Integrates with Analytics for data-driven planning.
3. **Planner → Creator pipeline** — select calendar ideas → trigger Content Creator interactively (replacing fire-and-forget `convert-flatplan-ideas-to-posts`).
4. **Proactive insights** — background Inngest jobs running Analytics Agent periodically, storing insights in Supabase.

**Reuses**: Flatplan pipeline, Tinybird analytics, brand performance API, multiselect idea generation, goal tracking.
**New**: NL analytics, conversational calendar editing, trend analysis, proactive insights.

### Phase 4: Full Orchestration (Weeks 17-24) — "The CMO"

**Goal**: Central CMO Agent + advanced capabilities.

1. **CMO Agent orchestrator** — intent classification, sub-agent routing, multi-agent coordination.
2. **Publishing & Distribution Agent** — wraps scheduling with smart timing and cross-platform adaptation.
3. **Feedback loops** — Analytics → Planner (next cycle strategy). Brand Intelligence ← approved content (learning). Publishing adjusts timing from results.
4. **Competitor analysis tools** — external data sources.
5. **Agent memory system** — long-term memory across sessions (past strategies, preferences, what worked).
6. **Migrate existing UI flows** — gradually replace direct API calls with agent-mediated flows.

---

## 9. Data & Infrastructure

### Vector Store (Brand Knowledge RAG)

**Recommended**: Start with pgvector in Supabase (same DB, low ops overhead). Migrate to dedicated store if needed.

```sql
brand_embeddings:
  id          uuid PK
  brand_id    string FK
  content     text           -- original chunk
  embedding   vector(1536)
  metadata    jsonb          -- { source_type: guideline|product|approved_content|review, source_id }
  chunk_index integer
```

**Alternatives**: Vespa, Pinecone, or Qdrant if pgvector performance becomes a bottleneck at scale.

### New Database Tables

```sql
agent_session:
  id                uuid PK
  brand_id          string FK
  user_id           string FK
  session_type      text       -- chat | background | proactive
  status            text       -- active | completed | archived
  context_snapshot  jsonb      -- rolling summary of conversation
  created_at        timestamptz
  last_active_at    timestamptz

agent_message:
  id          uuid PK
  session_id  uuid FK
  role        text       -- user | assistant | system | tool_call | tool_result
  content     text
  metadata    jsonb      -- { sub_agent, model_used, tokens, latency_ms, tool_name }
  created_at  timestamptz

agent_tool_invocation:
  id            uuid PK
  message_id    uuid FK
  session_id    uuid FK
  tool_name     text
  input_params  jsonb
  output_result jsonb
  status        text       -- pending | running | completed | failed
  started_at    timestamptz
  completed_at  timestamptz
  duration_ms   integer

agent_task:  -- replaces powermode_status_history + posts_duplication_status
  id               uuid PK
  session_id       uuid FK (nullable)
  brand_id         string FK
  task_type        text       -- flatplan_generation | powermode | post_duplication | analytics_report
  status           text       -- queued | in_progress | completed | failed | cancelled
  input_params     jsonb
  result           jsonb
  progress         integer    -- 0-100
  inngest_event_id text
  thread_id        text
  run_id           text
  created_at       timestamptz
  updated_at       timestamptz
  completed_at     timestamptz
```

### Async Job Handling
Continue using **Inngest** for durable async execution. Key patterns to preserve and generalize:
- Event-driven dispatch (`inngest.send()`)
- Polling for AI backend results (generalize `process-powermode-history` into reusable `poll_agent_result`)
- Progressive updates (standardize `generation_progress` in `agent_task`)
- Cancellation (generalize `cancel-process-powermode-history` pattern)

### Real-time Updates
Use **Supabase Realtime** (already available) — subscribe to `agent_task` and `agent_message` changes for live UI updates.

---

## 10. Critical Files for Implementation

| File | Purpose |
|---|---|
| `src/inngest/types.ts` | Event type definitions; new agent events register here |
| `src/utils/powermode-graph-agent-call.ts` | Pattern for all AI backend agent calls (thread_id/run_id) |
| `src/inngest/functions/generate-specific-flatplan.ts` | Template for complex multi-step agent tasks |
| `src/inngest/functions/scheduler.ts` | Publishing logic for all 6 platforms (wrap as tool, don't rewrite) |
| `src/database.types.ts` | Schema reference; new tables added via Supabase migrations |
| `src/lib/magic-templates/actions.ts` | Template AI fill patterns to reuse |
| `src/lib/ai-api/hashtags.ts` | Existing tool wrapper pattern |
| `src/lib/composer/hooks/useCopyGenV2.tsx` | Caption generation hook to wrap |
| `src/lib/brand-performance/actions.ts` | Brand performance AI pattern |
| `src/lib/analytics/tinybirdClient.ts` | Analytics query patterns |
| `src/lib/flat-plan/actions.ts` | Flatplan CRUD operations |
| `src/inngest/functions/ai-training.ts` | AI training workflow pattern |
| `src/inngest/functions/document-summary.ts` | Document processing pattern |

---

## 11. Generative UI — Interactive Chat Experience

The agent chat is not just text-in/text-out. The CMO Agent streams **rich, interactive UI components** directly into the conversation. Users can preview, edit, approve, and schedule content without leaving the chat.

### Core Concept
When the agent performs an action (generates a post, builds a calendar, fetches analytics), it returns structured data that the frontend renders as interactive components inline in the chat. The user can interact with these components (edit caption, swap image, approve post, drag calendar items) and the agent sees those interactions as follow-up context.

### Generative UI Components

#### Post Preview Card
- **Triggered by**: Content Creator Agent generates a post
- **Renders**: Platform-specific post preview (Instagram feed mock, LinkedIn post mock, TikTok video mock, etc.)
- **Shows**: Caption with hashtags, media (image/video/carousel), template design preview, platform badge
- **Interactive actions**:
  - Edit caption inline (rich text editor)
  - Swap/regenerate image ("Try another image" button → agent generates alternative)
  - Swap template ("Try another template" → agent picks new magic template)
  - Edit in full composer (opens CESDK editor with pre-loaded content)
  - Approve & schedule (date/time picker → sends to Publishing Agent)
  - Approve & publish now
  - Request refinement via text ("Make the caption shorter")
- **Multi-variant view**: When A/B variants are generated, show side-by-side comparison cards

#### Content Calendar View
- **Triggered by**: Content Planner Agent generates a flatplan
- **Renders**: Mini calendar grid (week/month view) with post idea cards on each date
- **Shows**: Post ideas with category color coding, channel icons, content type badges (post/reel/story/carousel)
- **Interactive actions**:
  - Drag-and-drop to rearrange posts between dates
  - Click idea to expand details (full idea brief, suggested caption direction, visual style)
  - Delete/regenerate individual ideas ("Give me a different idea for March 20")
  - Select multiple ideas → "Generate these posts" (triggers batch Content Creator via PowerMode)
  - "Add a post on [date]" — agent fills the gap
  - Approve entire calendar → converts to draft posts in the system
- **Filters**: Toggle by channel, content pillar, content type

#### Analytics Dashboard Cards
- **Triggered by**: Analytics Agent responds to performance queries
- **Renders**: Inline charts and metric cards
- **Component types**:
  - **Metric summary cards**: Engagement rate, reach, followers gained — with trend arrows and period comparison
  - **Bar/line charts**: Performance over time, channel comparison, content type breakdown
  - **Top posts grid**: Thumbnails of best performing posts with key metrics, clickable to view full post
  - **Insights callout**: Highlighted finding with recommendation ("Your Reels get 4x more reach than static posts — consider increasing Reel frequency")
- **Interactive actions**:
  - Change date range (dropdown: last 7 days, 30 days, 90 days)
  - Drill down into specific channel
  - "Act on this insight" → routes to Content Planner to adjust strategy
  - Export as report (PDF/email)

#### Brand Asset Viewer
- **Triggered by**: Brand Intelligence Agent references brand guidelines or assets
- **Renders**: Brand identity card showing logo, color swatches, typography samples, tone descriptors
- **Interactive actions**:
  - Upload new brand assets (drag-and-drop zone)
  - Upload brand guidelines PDF → agent processes and indexes
  - Edit brand voice settings
  - View brand compliance score for recent content

#### Approval Flow Card
- **Triggered by**: Post ready for review or approval status update
- **Renders**: Post preview with approval status badge, reviewer info, feedback thread
- **Interactive actions**:
  - Approve / request changes / reject
  - Add feedback comment
  - View version history (before/after edits)
  - Reassign to different reviewer

#### Publishing Queue View
- **Triggered by**: User asks "What's scheduled?" or Publishing Agent shows queue
- **Renders**: Timeline/list view of upcoming scheduled posts across all channels
- **Shows**: Post thumbnails, scheduled time, channel, status (draft/approved/scheduled)
- **Interactive actions**:
  - Reschedule (drag to new time or date picker)
  - Reorder queue
  - Cancel/unschedule
  - Quick edit caption
  - "Fill gaps" — agent identifies empty slots and suggests content

#### Carousel/Multi-Slide Editor
- **Triggered by**: Content Creator generates carousel content
- **Renders**: Horizontal slide strip with individual slide previews
- **Interactive actions**:
  - Reorder slides via drag-and-drop
  - Add/remove slides
  - Edit individual slide text/image
  - Apply consistent template across all slides
  - Preview as platform-native carousel swipe

### Data Flow: Agent → Generative UI → Agent

```
1. User: "Create an Instagram carousel about our summer collection"
2. Agent returns structured response:
   {
     type: "post_preview",
     data: { caption, hashtags, slides: [...], template_id, channel: "instagram" },
     actions: ["edit_caption", "swap_image", "regenerate", "schedule", "open_editor"]
   }
3. Frontend renders PostPreviewCard with interactive controls
4. User clicks "Make the caption more playful" or edits inline
5. Frontend sends user action back as a message:
   { type: "user_action", action: "edit_caption", data: { new_caption: "..." } }
   OR
   { type: "user_action", action: "refine", feedback: "Make it more playful" }
6. Agent processes and returns updated PostPreviewCard
7. User clicks "Schedule for Tuesday 10am"
8. Agent calls Publishing Agent → confirms with ScheduleConfirmationCard
```

### Streaming & Progressive Rendering
- Agent streams partial results as they become available:
  - Caption appears first (text streams in)
  - Then image generation starts (loading skeleton → image loads)
  - Then template application (skeleton → rendered template)
- Each component has loading/skeleton states for progressive disclosure
- Long-running operations (PowerMode batch, calendar generation) show progress bar components with real-time updates via Supabase Realtime

### Component Library Architecture
- Build as a shared component library under `src/components/agent-ui/`
- Each component is a self-contained React component that accepts structured agent data
- Components emit user actions as typed events back to the agent conversation
- Reuse existing app components where possible:
  - Post preview → extend existing `PostInfoDialog` / `EventDetailsView`
  - Calendar → extend existing `ContentCalendar` component
  - Analytics charts → extend existing analytics dashboard components
  - Design editor → embed existing CESDK editor
  - Upload → reuse existing Uppy `UppyDashboardModal`

### Research Items
- **R11G.1**: Evaluate Vercel AI SDK's `createStreamableUI` / generative UI primitives vs. custom structured response rendering. Trade-off: SDK convenience vs. flexibility for complex interactive components.
- **R11G.2**: Research streaming structured data (not just text) from Python AI backend to Next.js frontend. Options: SSE with JSON chunks, WebSocket, or Vercel AI SDK's tool-based UI streaming.
- **R11G.3**: Design the agent response schema — standardize the JSON structure for all generative UI component types so the frontend renderer can switch on `type` and render the right component.
- **R11G.4**: Evaluate whether interactive actions (edit, approve, schedule) should go back through the agent (agent-mediated) or bypass the agent and call APIs directly (faster but loses conversational context). Recommendation: hybrid — simple actions (schedule, approve) go direct; refinement actions ("make it shorter") go through agent.
- **R11G.5**: Research optimistic UI updates for agent actions — when user clicks "Schedule", show confirmation immediately while agent confirms in background.
- **R11G.6**: Evaluate mobile responsiveness of generative UI components — carousel previews, calendar grids, and analytics charts need to work in compact chat views.
- **R11G.7**: Research accessibility (a11y) patterns for interactive components within a chat stream — keyboard navigation, screen reader support for inline editors and drag-and-drop.

---

## 12. Framework Migration: LangChain/LangGraph → Agno + Langfuse

### Why Move Off LangChain/LangGraph
- Heavy abstraction overhead — LangChain's deep class hierarchies add complexity without proportional value for our use case
- Vendor coupling — LangChain's opinionated patterns make it harder to swap models or customize agent behavior
- Debugging difficulty — deeply nested chains obscure what's actually happening in agent execution
- Dependency bloat — LangChain pulls in a large dependency tree that increases attack surface and bundle size

### Why Agno
- Lightweight, model-agnostic agent framework with minimal abstraction
- First-class multi-agent support with agent teams and coordination
- Clean tool definition interface that maps well to our existing tool wrappers
- Built-in support for structured outputs, memory, and knowledge bases
- Native async support for parallel tool execution
- Simpler debugging — agent logic is transparent, not hidden behind chain abstractions

### Why Langfuse for Tracing
- Open-source observability platform purpose-built for LLM applications
- End-to-end trace visualization: CMO orchestrator → sub-agent → tool call → LLM call
- Cost tracking per agent, per brand, per session — critical for multi-tenant SaaS
- Latency monitoring and bottleneck identification across agent pipelines
- Prompt management and versioning — track which prompts produce best results
- Evaluation and scoring — rate agent outputs, build evaluation datasets
- Self-hostable for data sovereignty, or use cloud offering
- Native Python SDK with decorator-based tracing (minimal code changes)

### Migration Research Items

- **R11.1**: Audit all current LangChain/LangGraph usage in the Python AI backend — identify every chain, graph, agent, and tool definition that needs migration
- **R11.2**: Map LangChain abstractions to Agno equivalents:
  - LangGraph `StateGraph` → Agno agent teams / workflow agents
  - LangChain `Tool` → Agno `tool` decorator / `Toolkit`
  - LangChain `ChatPromptTemplate` → Agno agent `instructions` + `description`
  - LangChain `RunnableSequence` / chains → Agno agent `run()` with tool composition
  - LangGraph checkpointing → Agno memory + session storage
- **R11.3**: Evaluate Agno's `Agent` teams mode for CMO orchestrator → sub-agent delegation (does it support our sequential, parallel fan-out, and feedback loop patterns?)
- **R11.4**: Test Agno's knowledge base integration (PDFs, vector stores) as replacement for LangChain's document loaders + retrievers for brand knowledge RAG
- **R11.5**: Prototype Langfuse integration with Agno — verify trace propagation works across multi-agent calls, tool invocations, and streaming responses
- **R11.6**: Evaluate Langfuse's cost tracking accuracy across multiple LLM providers (OpenAI, Anthropic, Gemini) via Agno's model-agnostic interface
- **R11.7**: Design Langfuse trace schema for BrandSocial — define trace naming conventions, metadata tags (brand_id, agent_type, task_type), and custom scores (brand_consistency, content_quality)
- **R11.8**: Plan Langfuse deployment — self-hosted (Docker/k8s) vs. cloud, data retention policies, access controls for the team

### Phased Migration Strategy

**Phase 1 (during Foundation, Weeks 1-4)**:
- Set up Langfuse instance and integrate tracing into existing Python AI backend alongside current LangChain code (Langfuse has a LangChain callback handler for drop-in tracing)
- Begin collecting baseline traces, latency, and cost data from current agent flows

**Phase 2 (during First Agents, Weeks 5-10)**:
- Build new Content Creator and Brand Intelligence agents directly in Agno (greenfield, no migration needed)
- Instrument all new Agno agents with Langfuse tracing from day one
- Existing LangChain-based endpoints (flatplan, PowerMode, post duplication) continue running unchanged

**Phase 3 (during Intelligence Layer, Weeks 11-16)**:
- Migrate flatplan generation agent from LangGraph → Agno
- Migrate PowerMode batch agent from LangGraph → Agno
- Migrate post duplication agent from LangGraph → Agno
- Compare Langfuse traces before/after migration to validate parity

**Phase 4 (during Full Orchestration, Weeks 17-24)**:
- Remove all LangChain/LangGraph dependencies from the codebase
- Full Agno-based agent stack with comprehensive Langfuse observability
- Build Langfuse evaluation pipelines for automated agent quality scoring
- Set up Langfuse dashboards for per-brand cost tracking and agent performance monitoring
