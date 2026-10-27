# ai-observability

## Patterns


---
  #### **Name**
Langfuse Tracing Setup
  #### **Description**
Open-source LLM tracing with full observability
  #### **When To Use**
Need detailed tracing, open-source, self-host option
  #### **Implementation**
    // lib/langfuse.ts
    import { Langfuse } from "langfuse";
    import OpenAI from "openai";
    
    // Initialize Langfuse client
    export const langfuse = new Langfuse({
      publicKey: process.env.LANGFUSE_PUBLIC_KEY!,
      secretKey: process.env.LANGFUSE_SECRET_KEY!,
      baseUrl: process.env.LANGFUSE_HOST || "https://cloud.langfuse.com",
    });
    
    // Wrap OpenAI with tracing
    const openai = new OpenAI();
    
    interface TracedCompletionOptions {
      model: string;
      messages: OpenAI.ChatCompletionMessageParam[];
      userId?: string;
      sessionId?: string;
      metadata?: Record<string, unknown>;
      tags?: string[];
    }
    
    export async function tracedCompletion({
      model,
      messages,
      userId,
      sessionId,
      metadata,
      tags,
    }: TracedCompletionOptions) {
      // Create trace
      const trace = langfuse.trace({
        name: "llm-completion",
        userId,
        sessionId,
        metadata,
        tags,
      });
    
      // Create generation span
      const generation = trace.generation({
        name: "openai-completion",
        model,
        input: messages,
        metadata: { modelProvider: "openai" },
      });
    
      const startTime = Date.now();
    
      try {
        const response = await openai.chat.completions.create({
          model,
          messages,
        });
    
        const latencyMs = Date.now() - startTime;
        const output = response.choices[0].message;
    
        // End generation with output
        generation.end({
          output,
          usage: {
            promptTokens: response.usage?.prompt_tokens,
            completionTokens: response.usage?.completion_tokens,
            totalTokens: response.usage?.total_tokens,
          },
          metadata: {
            latencyMs,
            finishReason: response.choices[0].finish_reason,
          },
        });
    
        // Calculate cost
        const cost = calculateCost(
          model,
          response.usage?.prompt_tokens || 0,
          response.usage?.completion_tokens || 0
        );
    
        trace.update({
          metadata: { ...metadata, totalCost: cost, latencyMs },
        });
    
        return {
          response,
          traceId: trace.id,
          cost,
          latencyMs,
        };
      } catch (error) {
        generation.end({
          level: "ERROR",
          statusMessage: error instanceof Error ? error.message : "Unknown error",
        });
        throw error;
      } finally {
        // Flush to ensure data is sent
        await langfuse.flushAsync();
      }
    }
    
    // Model pricing (per 1M tokens)
    const MODEL_PRICING: Record<string, { input: number; output: number }> = {
      "gpt-4o": { input: 5, output: 15 },
      "gpt-4o-mini": { input: 0.15, output: 0.6 },
      "gpt-4-turbo": { input: 10, output: 30 },
      "claude-sonnet-4-20250514": { input: 3, output: 15 },
      "claude-3-5-haiku-latest": { input: 0.8, output: 4 },
    };
    
    function calculateCost(
      model: string,
      inputTokens: number,
      outputTokens: number
    ): number {
      const pricing = MODEL_PRICING[model] || MODEL_PRICING["gpt-4o"];
      return (
        (inputTokens / 1_000_000) * pricing.input +
        (outputTokens / 1_000_000) * pricing.output
      );
    }
    

---
  #### **Name**
Helicone Proxy Setup
  #### **Description**
Zero-code LLM observability via proxy
  #### **When To Use**
Want minimal setup, automatic cost tracking
  #### **Implementation**
    // lib/helicone-openai.ts
    import OpenAI from "openai";
    
    // Use Helicone proxy for automatic tracing
    const openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY!,
      baseURL: "https://oai.helicone.ai/v1",
      defaultHeaders: {
        "Helicone-Auth": `Bearer ${process.env.HELICONE_API_KEY}`,
      },
    });
    
    // Add custom properties for filtering
    interface HeliconeOptions {
      userId?: string;
      sessionId?: string;
      promptId?: string;
      cache?: boolean;
      retryEnabled?: boolean;
      rateLimitPolicy?: string;
    }
    
    export function createHeliconeClient(options: HeliconeOptions) {
      const headers: Record<string, string> = {
        "Helicone-Auth": `Bearer ${process.env.HELICONE_API_KEY}`,
      };
    
      if (options.userId) {
        headers["Helicone-User-Id"] = options.userId;
      }
      if (options.sessionId) {
        headers["Helicone-Session-Id"] = options.sessionId;
      }
      if (options.promptId) {
        headers["Helicone-Prompt-Id"] = options.promptId;
      }
      if (options.cache) {
        headers["Helicone-Cache-Enabled"] = "true";
      }
      if (options.retryEnabled) {
        headers["Helicone-Retry-Enabled"] = "true";
      }
      if (options.rateLimitPolicy) {
        headers["Helicone-RateLimit-Policy"] = options.rateLimitPolicy;
      }
    
      return new OpenAI({
        apiKey: process.env.OPENAI_API_KEY!,
        baseURL: "https://oai.helicone.ai/v1",
        defaultHeaders: headers,
      });
    }
    
    // For Anthropic
    import Anthropic from "@anthropic-ai/sdk";
    
    export function createHeliconeAnthropic(options: HeliconeOptions) {
      const headers: Record<string, string> = {
        "Helicone-Auth": `Bearer ${process.env.HELICONE_API_KEY}`,
      };
    
      if (options.userId) headers["Helicone-User-Id"] = options.userId;
      if (options.cache) headers["Helicone-Cache-Enabled"] = "true";
    
      return new Anthropic({
        apiKey: process.env.ANTHROPIC_API_KEY!,
        baseURL: "https://anthropic.helicone.ai",
        defaultHeaders: headers,
      });
    }
    

---
  #### **Name**
Self-Hosted Gateway Proxy (Bifrost)
  #### **Description**
Zero-code tracing and cost tracking through an open-source gateway you run yourself
  #### **When To Use**
Want proxy-style observability without routing traffic through a third-party service
  #### **Implementation**
    // lib/bifrost-openai.ts
    // Start the gateway: npx -y @maximhq/bifrost  (or: docker run -p 8080:8080 maximhq/bifrost)
    // Add provider keys in the web UI at http://localhost:8080. Every request is logged there
    // with tokens, cost and latency; Prometheus metrics are at /metrics, and the otel plugin
    // exports traces to any OTLP backend (e.g. Langfuse). https://github.com/maximhq/bifrost
    import OpenAI from "openai";
    
    export const openai = new OpenAI({
      apiKey: process.env.BIFROST_VIRTUAL_KEY!, // sk-bf-... virtual key: per-key budgets and rate limits
      baseURL: "http://localhost:8080/openai",
    });
    
    // Same client, any configured provider: prefix the model with the provider
    await openai.chat.completions.create({
      model: "anthropic/claude-3-5-haiku-latest",
      messages: [{ role: "user", content: "Hello" }],
    });
    
    // For the Anthropic SDK, use baseURL "http://localhost:8080/anthropic"
    

---
  #### **Name**
Cost Tracking Dashboard
  #### **Description**
Track and budget LLM costs per user/feature
  #### **When To Use**
Need granular cost visibility and budgets
  #### **Implementation**
    // lib/cost-tracking.ts
    import { db } from "./db";
    
    interface TokenUsage {
      userId: string;
      model: string;
      inputTokens: number;
      outputTokens: number;
      cost: number;
      feature?: string;
      traceId?: string;
    }
    
    // Record usage
    export async function recordUsage(usage: TokenUsage) {
      await db.tokenUsage.create({
        data: {
          ...usage,
          timestamp: new Date(),
        },
      });
    }
    
    // Get user's usage for period
    export async function getUserUsage(
      userId: string,
      period: "day" | "week" | "month" = "month"
    ) {
      const startDate = new Date();
      if (period === "day") startDate.setDate(startDate.getDate() - 1);
      else if (period === "week") startDate.setDate(startDate.getDate() - 7);
      else startDate.setMonth(startDate.getMonth() - 1);
    
      const usage = await db.tokenUsage.aggregate({
        where: {
          userId,
          timestamp: { gte: startDate },
        },
        _sum: {
          inputTokens: true,
          outputTokens: true,
          cost: true,
        },
        _count: true,
      });
    
      return {
        totalInputTokens: usage._sum.inputTokens || 0,
        totalOutputTokens: usage._sum.outputTokens || 0,
        totalCost: usage._sum.cost || 0,
        requestCount: usage._count,
      };
    }
    
    // Check budget before request
    export async function checkBudget(
      userId: string,
      estimatedCost: number
    ): Promise<{ allowed: boolean; remaining: number; reason?: string }> {
      const user = await db.user.findUnique({
        where: { id: userId },
        select: { monthlyBudget: true, tier: true },
      });
    
      if (!user) {
        return { allowed: false, remaining: 0, reason: "User not found" };
      }
    
      const currentUsage = await getUserUsage(userId, "month");
      const remaining = user.monthlyBudget - currentUsage.totalCost;
    
      if (remaining < estimatedCost) {
        return {
          allowed: false,
          remaining,
          reason: `Budget exceeded. Used $${currentUsage.totalCost.toFixed(2)} of $${user.monthlyBudget} limit.`,
        };
      }
    
      return { allowed: true, remaining };
    }
    
    // Cost breakdown by feature
    export async function getCostByFeature(period: "day" | "week" | "month") {
      const startDate = new Date();
      if (period === "day") startDate.setDate(startDate.getDate() - 1);
      else if (period === "week") startDate.setDate(startDate.getDate() - 7);
      else startDate.setMonth(startDate.getMonth() - 1);
    
      return db.tokenUsage.groupBy({
        by: ["feature"],
        where: { timestamp: { gte: startDate } },
        _sum: { cost: true, inputTokens: true, outputTokens: true },
        _count: true,
      });
    }
    

---
  #### **Name**
RAG Evaluation with RAGAS
  #### **Description**
Automated RAG quality evaluation
  #### **When To Use**
Have RAG pipeline, need to measure quality
  #### **Implementation**
    // lib/rag-evaluation.ts
    import Anthropic from "@anthropic-ai/sdk";
    
    const anthropic = new Anthropic();
    
    interface RAGSample {
      question: string;
      contexts: string[];
      answer: string;
      groundTruth?: string; // Optional for some metrics
    }
    
    interface RAGASScores {
      faithfulness: number;
      answerRelevancy: number;
      contextPrecision: number;
      contextRecall?: number; // Requires ground truth
    }
    
    // Evaluate faithfulness: Is answer supported by context?
    async function evaluateFaithfulness(sample: RAGSample): Promise<number> {
        const prompt = `You are evaluating whether an answer is faithful to the provided context.
    
        Context:
        ${sample.contexts.join("\n\n")}
    
        Question: ${sample.question}
        Answer: ${sample.answer}
    
        Task:
        1. List each claim made in the answer.
        2. For each claim, determine if it can be inferred from the context.
        3. Calculate the ratio of supported claims to total claims.
    
        Return JSON:
        {
          "claims": [{"claim": "...", "supported": true/false, "evidence": "..."}],
          "faithfulness_score": 0.0-1.0
        }`;
    
      const response = await anthropic.messages.create({
        model: "claude-sonnet-4-20250514",
        max_tokens: 1024,
        messages: [{ role: "user", content: prompt }],
      });
    
      const text = response.content[0].type === "text" ? response.content[0].text : "";
      const json = JSON.parse(text.match(/\{[\s\S]*\}/)?.[0] || "{}");
    
      return json.faithfulness_score || 0;
    }
    
    // Evaluate answer relevancy: Is answer relevant to question?
    async function evaluateAnswerRelevancy(sample: RAGSample): Promise<number> {
      const prompt = `Evaluate how relevant the answer is to the question.
    
      Question: ${sample.question}
      Answer: ${sample.answer}
    
      Score from 0-1:
        - 1.0: Directly and completely answers the question
        - 0.7-0.9: Answers the question with minor irrelevant info
        - 0.4-0.6: Partially answers, missing key aspects
        - 0.1-0.3: Mostly irrelevant with some tangential relation
        - 0.0: Completely irrelevant
    
        Return JSON: {"score": 0.0-1.0, "reasoning": "..."}`;
    
      const response = await anthropic.messages.create({
        model: "claude-sonnet-4-20250514",
        max_tokens: 256,
        messages: [{ role: "user", content: prompt }],
      });
    
      const text = response.content[0].type === "text" ? response.content[0].text : "";
      const json = JSON.parse(text.match(/\{[\s\S]*\}/)?.[0] || "{}");
    
      return json.score || 0;
    }
    
    // Evaluate context precision: Are retrieved contexts relevant?
    async function evaluateContextPrecision(sample: RAGSample): Promise<number> {
      const prompt = `Evaluate the precision of retrieved contexts for answering the question.
    
      Question: ${sample.question}
    
      Contexts:
      ${sample.contexts.map((c, i) => `[${i + 1}] ${c}`).join("\n\n")}
    
      For each context, determine if it's relevant to answering the question.
      Return JSON:
      {
        "context_relevance": [{"index": 1, "relevant": true/false, "reason": "..."}],
        "precision_score": 0.0-1.0
      }`;
    
      const response = await anthropic.messages.create({
        model: "claude-sonnet-4-20250514",
        max_tokens: 512,
        messages: [{ role: "user", content: prompt }],
      });
    
      const text = response.content[0].type === "text" ? response.content[0].text : "";
      const json = JSON.parse(text.match(/\{[\s\S]*\}/)?.[0] || "{}");
    
      return json.precision_score || 0;
    }
    
    // Full RAGAS evaluation
    export async function evaluateRAGAS(sample: RAGSample): Promise<RAGASScores> {
      const [faithfulness, answerRelevancy, contextPrecision] = await Promise.all([
        evaluateFaithfulness(sample),
        evaluateAnswerRelevancy(sample),
        evaluateContextPrecision(sample),
      ]);
    
      return {
        faithfulness,
        answerRelevancy,
        contextPrecision,
      };
    }
    
    // Batch evaluation with aggregation
    export async function evaluateBatch(
      samples: RAGSample[]
    ): Promise<{
      individual: Array<RAGSample & { scores: RAGASScores }>;
      aggregate: RAGASScores;
    }> {
      const results = await Promise.all(
        samples.map(async (sample) => ({
          ...sample,
          scores: await evaluateRAGAS(sample),
        }))
      );
    
      // Calculate averages
      const aggregate: RAGASScores = {
        faithfulness:
          results.reduce((sum, r) => sum + r.scores.faithfulness, 0) / results.length,
        answerRelevancy:
          results.reduce((sum, r) => sum + r.scores.answerRelevancy, 0) / results.length,
        contextPrecision:
          results.reduce((sum, r) => sum + r.scores.contextPrecision, 0) / results.length,
      };
    
      return { individual: results, aggregate };
    }
    

---
  #### **Name**
Prompt Caching Optimization
  #### **Description**
Reduce costs 50-90% with prompt caching
  #### **When To Use**
Have repeated system prompts or context
  #### **Implementation**
    // lib/prompt-caching.ts
    import Anthropic from "@anthropic-ai/sdk";
    
    const anthropic = new Anthropic();
    
    interface CachedMessage {
      role: "user" | "assistant";
      content: string | Anthropic.ContentBlock[];
    }
    
    // Anthropic prompt caching with cache_control
    export async function cachedAnthropicCall(options: {
      systemPrompt: string;
      staticContext?: string;
      userMessage: string;
      model?: string;
    }) {
      const { systemPrompt, staticContext, userMessage, model = "claude-sonnet-4-20250514" } = options;
    
      // Build content blocks with cache markers
      const systemContent: Anthropic.ContentBlockParam[] = [
        {
          type: "text",
          text: systemPrompt,
          cache_control: { type: "ephemeral" }, // Cache this
        },
      ];
    
      if (staticContext) {
        systemContent.push({
          type: "text",
          text: staticContext,
          cache_control: { type: "ephemeral" }, // Cache this too
        });
      }
    
      const response = await anthropic.messages.create({
        model,
        max_tokens: 1024,
        system: systemContent,
        messages: [{ role: "user", content: userMessage }],
      });
    
      // Check cache performance
      const usage = response.usage;
      const cacheHit = (usage as any).cache_read_input_tokens > 0;
      const cachedTokens = (usage as any).cache_read_input_tokens || 0;
      const uncachedTokens = usage.input_tokens - cachedTokens;
    
      return {
        response,
        cacheStats: {
          hit: cacheHit,
          cachedTokens,
          uncachedTokens,
          savingsPercent: cacheHit
            ? Math.round((cachedTokens / usage.input_tokens) * 100)
            : 0,
        },
      };
    }
    
    // OpenAI automatic caching (just structure prompts correctly)
    import OpenAI from "openai";
    
    const openai = new OpenAI();
    
    export async function optimizedOpenAICall(options: {
      staticSystemPrompt: string;
      staticContext?: string;
      dynamicUserMessage: string;
      model?: string;
    }) {
      const { staticSystemPrompt, staticContext, dynamicUserMessage, model = "gpt-4o" } = options;
    
      // OpenAI caches automatically for prompts > 1024 tokens
      // Structure: static content first, dynamic content last
      const messages: OpenAI.ChatCompletionMessageParam[] = [
        {
          role: "system",
          content: staticContext
            ? `${staticSystemPrompt}\n\nContext:\n${staticContext}`
            : staticSystemPrompt,
        },
        {
          role: "user",
          content: dynamicUserMessage,
        },
      ];
    
      const response = await openai.chat.completions.create({
        model,
        messages,
      });
    
      // OpenAI returns cached_tokens in usage
      const cached = (response.usage as any)?.cached_tokens || 0;
    
      return {
        response,
        cacheStats: {
          cachedTokens: cached,
          uncachedTokens: (response.usage?.prompt_tokens || 0) - cached,
        },
      };
    }
    

---
  #### **Name**
Hallucination Detection
  #### **Description**
Detect and prevent LLM hallucinations
  #### **When To Use**
Need to verify factual accuracy, especially in RAG
  #### **Implementation**
    // lib/hallucination-detection.ts
    import Anthropic from "@anthropic-ai/sdk";
    
    const anthropic = new Anthropic();
    
    interface FactCheckResult {
      isHallucination: boolean;
      confidence: number;
      unsupportedClaims: string[];
      suggestedCorrections?: string[];
    }
    
    // Self-consistency hallucination check
    // Generate multiple responses and check for consistency
    export async function selfConsistencyCheck(
      prompt: string,
      originalResponse: string,
      numSamples: number = 3
    ): Promise<{
      consistent: boolean;
      consistencyScore: number;
      samples: string[];
    }> {
      // Generate additional samples
      const samples: string[] = [];
    
      for (let i = 0; i < numSamples; i++) {
        const response = await anthropic.messages.create({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1024,
          messages: [{ role: "user", content: prompt }],
          temperature: 0.7, // Add variation
        });
    
        const text = response.content[0].type === "text" ? response.content[0].text : "";
        samples.push(text);
      }
    
      // Check consistency across samples
      const consistencyPrompt = `Compare these responses for factual consistency:
    
      Original response: ${originalResponse}
    
      Additional samples:
      ${samples.map((s, i) => `Sample ${i + 1}: ${s}`).join("\n\n")}
    
      Identify any factual claims that differ significantly between responses.
      Return JSON:
      {
        "consistent": true/false,
        "consistency_score": 0.0-1.0,
        "inconsistent_claims": ["claim1", "claim2"]
      }`;
    
      const checkResponse = await anthropic.messages.create({
        model: "claude-sonnet-4-20250514",
        max_tokens: 512,
        messages: [{ role: "user", content: consistencyPrompt }],
      });
    
      const text = checkResponse.content[0].type === "text" ? checkResponse.content[0].text : "";
      const result = JSON.parse(text.match(/\{[\s\S]*\}/)?.[0] || "{}");
    
      return {
        consistent: result.consistent ?? true,
        consistencyScore: result.consistency_score ?? 1.0,
        samples,
      };
    }
    
    // Ground truth fact checking against context
    export async function factCheckAgainstContext(
      response: string,
      context: string
    ): Promise<FactCheckResult> {
      const prompt = `You are a fact-checker. Verify if the response is supported by the context.
    
      Context:
      ${context}
    
      Response to verify:
      ${response}
    
      Instructions:
      1. Extract each factual claim from the response.
      2. Check if each claim is supported by the context.
      3. A claim is a hallucination if it cannot be inferred from the context.
    
      Return JSON:
      {
        "claims": [
          {"claim": "...", "supported": true/false, "evidence": "quote from context or 'not found'"}
        ],
        "is_hallucination": true/false,
        "confidence": 0.0-1.0,
        "unsupported_claims": ["claim1", "claim2"],
        "suggested_corrections": ["correction1"]
      }`;
    
      const checkResponse = await anthropic.messages.create({
        model: "claude-sonnet-4-20250514",
        max_tokens: 1024,
        messages: [{ role: "user", content: prompt }],
      });
    
      const text = checkResponse.content[0].type === "text" ? checkResponse.content[0].text : "";
      const result = JSON.parse(text.match(/\{[\s\S]*\}/)?.[0] || "{}");
    
      return {
        isHallucination: result.is_hallucination ?? false,
        confidence: result.confidence ?? 1.0,
        unsupportedClaims: result.unsupported_claims || [],
        suggestedCorrections: result.suggested_corrections,
      };
    }
    

## Anti-Patterns


---
  #### **Name**
No Tracing in Production
  #### **Description**
Running LLM apps without observability
  #### **Why Bad**
    Debugging production issues is impossible. Can't identify slow calls,
    high-cost requests, or quality issues. Flying completely blind.
    
  #### **Instead**
Add Langfuse/Helicone from day one. Setup takes <10 minutes.

---
  #### **Name**
Ignoring Token Costs
  #### **Description**
Not tracking costs per user/feature
  #### **Why Bad**
    A single agent loop can spend $50+ in minutes. One power user can
    consume entire monthly budget. Costs spiral without visibility.
    
  #### **Instead**
Track costs per request, set per-user budgets, alert on anomalies.

---
  #### **Name**
No RAG Quality Metrics
  #### **Description**
Deploying RAG without evaluation
  #### **Why Bad**
    Retrieval quality degrades silently. Users get irrelevant results.
    Can't measure impact of changes to chunking, embeddings, or prompts.
    
  #### **Instead**
Run RAGAS evals on sample of production queries weekly.

---
  #### **Name**
Missing Prompt Caching
  #### **Description**
Repeated system prompts without caching
  #### **Why Bad**
    Same 2000-token system prompt sent with every request. At 1000 requests/day,
    that's 2M tokens wasted = $10/day on just repeated context.
    
  #### **Instead**
Use Anthropic cache_control or structure prompts for OpenAI auto-cache.

---
  #### **Name**
Batch Size One
  #### **Description**
Processing items one at a time
  #### **Why Bad**
    API overhead per request. Rate limits hit faster. Costs higher.
    Processing 100 items takes 100x the time and overhead.
    
  #### **Instead**
Batch API calls where possible. Use async patterns for parallelism.