# LLM-as-Judge Evaluation Pipeline: 3 Agents, 3 Models (n8n)

An n8n workflow that evaluates an LLM with other LLMs. A candidate model answers a test set, two independent judges from different providers score each answer against a reference, and the pipeline reports judge pass rate, human pass rate, judge-human agreement, mean absolute error and judge disagreement.

▶ **Demo video:** https://drive.google.com/file/d/1vDzIwpyDzpFNKIA7SerZ5ieIrlEFRG1s/view

## How it works

1. A test set of 5 questions arrives (facts, math, tech, reasoning), each with a reference answer and a human score
2. The Candidate agent (the model under test) answers each question briefly
3. Judge A and Judge B, two different providers, each score the answer from 1 to 5 against the question and the reference. Neither judge is the candidate model, so there is no self-preference
4. A Code node calculates per item: average judge score, absolute error against the human score, `passed` (average of 4 or more), `agrees` (within 1 point of the human) and `judge_disagree` (judges differ by 2 or more)
5. Aggregate Run computes the rates for the whole run, Slack and email send the summary, items and the run are saved to Postgres

```
Manual Trigger -> Test Set -> Candidate Agent -> Parse Answer
-> Judge A Agent -> Parse Judge A -> Judge B Agent -> Parse Judge B
-> Save Item Results (Postgres) -> Aggregate Run
-> Slack Run Summary + Send Run Report (email) + Save Run (Postgres)
```

## Agents and models

Each agent uses its own model, from three providers (all have free tiers):

| Agent | Provider | Model |
|---|---|---|
| Candidate Agent | Groq | `llama-3.1-8b-instant` |
| Judge A Agent | Google Gemini | `models/gemini-2.5-flash` |
| Judge B Agent | OpenRouter | `meta-llama/llama-3.3-70b-instruct:free` |

## Rules in code

- Judge scores are clamped to whole numbers from 1 to 5, and an answer that cannot be parsed counts as 1
- `passed` means the average of both judges is 4 or more
- `agrees` means the judge average is within 1 point of the human score
- `judge_disagree` means the two judges differ by 2 or more points

## Tech stack

n8n, AI Agents, Groq, OpenRouter, Google Gemini, Slack, Postgres (Supabase), SMTP

## Setup

1. Copy the JSON from `workflow.md` into a file named `llm-judge-pipeline-cloud.json` and import it into n8n
2. Create credentials: Groq (API key), OpenRouter (API key), Google Gemini (API key), Slack (Bot token and signing secret), Postgres (Supabase Session pooler, SSL: Require), SMTP (for the email node, for example Gmail with an app password)
3. Open each model node and pick your own credential for its provider
4. In the Slack node set channel `eval-reports`, then run `/invite @your-bot` in that channel
5. In the Send Email node replace `YOUR_EMAIL@gmail.com` with your address
6. Create the database tables (below)
7. In the Postgres save nodes remove the `id` and `created_at` fields, the database fills them
8. Click **Execute workflow**: the mock inputs run through all branches

Free-tier models have rate limits. If you see "Too Many Requests", wait a few minutes and run again.

## Database

```sql
create table if not exists public.eval_items (
  id bigint generated always as identity primary key,
  run_id text,
  question_id text,
  category text,
  candidate_answer text,
  judge_a numeric,
  judge_b numeric,
  judge_avg numeric,
  human_score numeric,
  passed text,
  agrees text,
  judge_disagree text,
  created_at timestamptz default now()
);

create table if not exists public.eval_runs (
  id bigint generated always as identity primary key,
  run_id text,
  items numeric,
  pass_rate numeric,
  human_pass_rate numeric,
  agreement_rate numeric,
  mean_abs_error numeric,
  judge_disagreement_rate numeric,
  created_at timestamptz default now()
);
```

## Demo data

| Mock input | Expected result |
|---|---|
| Q1 capital of Australia (facts) | Short factual answer, judged against "Canberra." |
| Q2 days from 1 January to 1 March (math) | Judged against "59 days." |
| Q3 HTTP status for Too Many Requests (tech) | Judged against "429." |
| Q4 what the n8n IF node does (tech) | Judged against the reference sentence |
| Q5 7 pens at 4 dollars with 10 percent discount (reasoning) | Judged against "25.20 dollars" |

AI answers can vary between runs. The rules in code do not.

## Notes

- IMPORTANT: the `human_score` values in the Test Set node are starter labels. Read the candidate answers once and replace them with your own scores, otherwise "agreement" means nothing
- Five questions are too few for reliable statistics. Use a bigger labeled test set for real evaluations
- The run id is built from the date and time, so each execution is a separate run in `eval_runs`
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
