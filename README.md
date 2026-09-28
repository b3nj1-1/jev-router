# jev-router
This is a router to reduce the cost of token for your suscription of opencode


## JSON
```json
{
  "state": "<user_prompt>",
  "questions": {
    "model_tier": {
      "type": "choice",
      "instructions": "Which model tier is required to handle this request?",
      "criteria": {
        "fast": "Simple syntax lookups, quick scripts, typos, git commands, formatting, basic explanations",
        "balanced": "Standard multi-file refactoring, writing unit tests, everyday feature implementation",
        "deep": "Complex architectural design, subtle race conditions, performance optimization, difficult algorithms or math"
      }
    },
    "effort_level": {
      "type": "choice",
      "instructions": "What reasoning effort is required to solve this problem correctly?",
      "criteria": {
        "low": "Straightforward implementation with minimal ambiguity",
        "medium": "Multi-step logic requiring careful step planning",
        "high": "Novel algorithmic design, deep edge-case analysis, or debugging complex state interactions"
      }
    }
  }
}
```

## Opencode plugin 

OpenCode provides the chat.params lifecycle hook. It fires immediately before a prompt is sent to the LLM, giving you access to the user message and letting you mutate execution options.   

Create the plugin file in either:

* Global scope: ~/.config/opencode/plugin/jev-router/index.ts
* Project scope: .opencode/plugin/jev-router/index.ts

The idea will be global scope since we want to save token and usage.


```
import type { Plugin } from "@opencode-ai/plugin";

// Map Jev tiers to your target models
const MODEL_MAP = {
  fast: "deepseek/deepseek-v4.1-flash",
  balanced: "deepseek/deepseek-v4-pro",
  deep: "xai/grok-4.7",
};

// Map Jev effort levels to provider-specific reasoning controls
const EFFORT_CONFIG = {
  low: {
    reasoning_effort: "low",
    thinking: { type: "enabled", budget_tokens: 1024 },
  },
  medium: {
    reasoning_effort: "medium",
    thinking: { type: "enabled", budget_tokens: 8192 },
  },
  high: {
    reasoning_effort: "high",
    thinking: { type: "enabled", budget_tokens: 24576 },
  },
};

async function evaluateWithJev(promptText: string) {
  const apiKey = process.env.TYPESAFE_API_KEY;
  if (!apiKey) return null;

  try {
    const res = await fetch("https://api.typesafe.ai/v1/evaluate", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        state: promptText,
        questions: {
          model_tier: {
            type: "choice",
            instructions: "Classify which model capability is required for this developer request.",
            criteria: {
              fast: "Simple one-line edits, shell commands, syntax questions, typos, git commands, trivial scripts",
              balanced: "Standard full-file implementations, feature development, unit testing, boilerplate, typical refactors",
              deep: "System architecture, high-concurrency race conditions, complex multi-file debugging, mathematical algorithms, security auditing",
            },
          },
          effort_level: {
            type: "choice",
            instructions: "Determine the reasoning depth needed to resolve this correctly.",
            criteria: {
              low: "Direct answers without multi-step deliberation",
              medium: "Standard coding tasks requiring step-by-step logic",
              high: "Complex root-cause analysis, subtle edge-case prevention, or novel logic design",
            },
          },
        },
      }),
    });

    if (!res.ok) return null;
    return await res.json();
  } catch (error) {
    console.error("[jev-router] Evaluation failed:", error);
    return null;
  }
}

export const JevRouterPlugin: Plugin = async ({ client }) => {
  return {
    "chat.params": async (input, output) => {
      const promptText = input.message?.text || "";
      if (!promptText.trim()) return;

      const evalResult = await evaluateWithJev(promptText);
      if (!evalResult?.answers) return;

      const modelTier = evalResult.answers.model_tier?.choice as keyof typeof MODEL_MAP;
      const effortLevel = evalResult.answers.effort_level?.choice as keyof typeof EFFORT_CONFIG;

      // 1. Assign selected model
      const targetModel = MODEL_MAP[modelTier] || MODEL_MAP.balanced;
      await client.session.update({
        sessionID: input.sessionID,
        model: targetModel,
      });

      // 2. Set reasoning parameters
      const effort = EFFORT_CONFIG[effortLevel] || EFFORT_CONFIG.medium;
      output.options = {
        ...output.options,
        reasoning_effort: effort.reasoning_effort,
        thinking: effort.thinking,
      };

      // 3. UI Status Log
      client.terminal.log(
        `\x1b[36m[Jev]\x1b[0m Routing to \x1b[32m${targetModel}\x1b[0m (\x1b[33m${effortLevel} effort\x1b[0m)`
      );
    },
  };
};

export default JevRouterPlugin;
```

## Remember to do this step
export TYPESAFE_API_KEY="ts_your_typesafe_key_here"
