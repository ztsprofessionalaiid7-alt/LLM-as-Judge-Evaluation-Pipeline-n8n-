# LLM-as-Judge Evaluation Pipeline: 3 Agents, 3 Models (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `llm-judge-pipeline-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "LLM-as-Judge Evaluation Pipeline (3 agents, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## LLM-as-Judge Evaluation Pipeline (n8n Cloud) - 3 agents, 3 different models\nCandidate (model under test): Groq llama-3.1-8b\nJudge A: Gemini | Judge B: OpenRouter free model (independent judges, neither is the candidate model, so no self-preference)\nMetrics: judge pass rate, human pass rate, judge-human agreement, mean absolute error, judge disagreement.\nIMPORTANT: human_score values in Test Set are starter labels. Read the candidate answers once, then replace them with YOUR own scores, otherwise agreement means nothing.\nSetup: credentials Groq, OpenRouter, Gemini, Slack, Postgres (Supabase), SMTP. Tables eval_items and eval_runs. Slack channel eval-reports.",
        "height": 360,
        "width": 640
      },
      "id": "4cc4f9e0-d68b-47ac-b8d9-276df4f78978",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -300
      ]
    },
    {
      "parameters": {},
      "id": "fcd3b3c3-6b4e-4098-88f5-83bf3cd86ac3",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "const run_id = 'RUN-' + new Date().toISOString().slice(0, 16).replace(/[-:T]/g, '');\nreturn [\n  { json: { run_id, question_id: 'Q1', category: 'facts', question: 'What is the capital city of Australia?', reference: 'Canberra.', human_score: 5 } },\n  { json: { run_id, question_id: 'Q2', category: 'math', question: 'How many days are there from 1 January to 1 March in a non-leap year?', reference: '59 days.', human_score: 4 } },\n  { json: { run_id, question_id: 'Q3', category: 'tech', question: 'Which HTTP status code means Too Many Requests?', reference: '429.', human_score: 5 } },\n  { json: { run_id, question_id: 'Q4', category: 'tech', question: 'In one sentence, what does the IF node do in n8n?', reference: 'It routes items to a true or false branch depending on a condition.', human_score: 5 } },\n  { json: { run_id, question_id: 'Q5', category: 'reasoning', question: 'A pen costs 4 dollars. How much do 7 pens cost with a 10 percent discount on the total?', reference: '25.20 dollars (7 x 4 = 28, minus 10 percent = 25.20).', human_score: 4 } }\n];"
      },
      "id": "49519970-7693-45b7-a7b3-f2236a2a1a84",
      "name": "Test Set",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Answer briefly (1-3 sentences).\nQuestion: {{ $json.question }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a helpful assistant. Answer concisely and accurately."
        }
      },
      "id": "92d2961d-6bd0-41d2-889d-650367f0ba23",
      "name": "Candidate Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        480,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "model": "llama-3.1-8b-instant",
        "options": {}
      },
      "id": "3e783b7b-bbe7-41c8-b741-481868b8c317",
      "name": "Candidate Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGroq",
      "typeVersion": 1,
      "position": [
        460,
        440
      ],
      "credentials": {
        "groqApi": {
          "id": "REPLACE_GROQ",
          "name": "Groq account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Test Set').item.json;\nreturn { json: Object.assign({}, ctx, { answer: String($json.output || '').slice(0, 600) }) };"
      },
      "id": "1a2857dd-d17d-494c-8b42-46261cbb0808",
      "name": "Parse Answer",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        730,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Evaluate the candidate answer.\nQuestion: {{ $json.question }}\nReference answer: {{ $json.reference }}\nCandidate answer: {{ $json.answer }}\n\nScore from 1 to 5: 5 = fully correct and complete, 4 = correct with minor omissions, 3 = partly correct, 2 = mostly wrong, 1 = wrong or irrelevant.",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a strict, fair evaluator. Judge only against the reference answer and the question. Respond with ONLY a JSON object, no markdown, no extra text: {\"score\":1,\"reason\":\"one sentence\"}. score is a whole number from 1 to 5."
        }
      },
      "id": "4076e2e0-cf6a-4310-9b25-e3012bc1337f",
      "name": "Judge A Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        1000,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "701bd215-802d-4dc2-8179-036743611b63",
      "name": "Judge A Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        980,
        440
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Parse Answer').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst s = Math.max(1, Math.min(5, Math.round(Number(r.score) || 1)));\nreturn { json: Object.assign({}, ctx, { judge_a: s, judge_a_reason: String(r.reason || '').slice(0, 250) }) };"
      },
      "id": "6f286b37-463c-45d2-8bfe-71f6da4b8957",
      "name": "Parse Judge A",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1250,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Evaluate the candidate answer.\nQuestion: {{ $json.question }}\nReference answer: {{ $json.reference }}\nCandidate answer: {{ $json.answer }}\n\nScore from 1 to 5: 5 = fully correct and complete, 4 = correct with minor omissions, 3 = partly correct, 2 = mostly wrong, 1 = wrong or irrelevant.",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a strict, fair evaluator. Judge only against the reference answer and the question. Respond with ONLY a JSON object, no markdown, no extra text: {\"score\":1,\"reason\":\"one sentence\"}. score is a whole number from 1 to 5."
        }
      },
      "id": "496153a5-6230-4d96-bf2d-7296bd19a3dd",
      "name": "Judge B Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        1520,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "model": "meta-llama/llama-3.3-70b-instruct:free",
        "options": {}
      },
      "id": "e397cdc2-9bdb-4da7-824e-959b5edbb785",
      "name": "Judge B Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenRouter",
      "typeVersion": 1,
      "position": [
        1500,
        440
      ],
      "credentials": {
        "openRouterApi": {
          "id": "REPLACE_OPENROUTER",
          "name": "OpenRouter account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Parse Judge A').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst b = Math.max(1, Math.min(5, Math.round(Number(r.score) || 1)));\nconst a = ctx.judge_a;\nconst avg = Math.round((a + b) / 2 * 10) / 10;\nconst err = Math.abs(avg - ctx.human_score);\nreturn { json: {\n  run_id: ctx.run_id,\n  question_id: ctx.question_id,\n  category: ctx.category,\n  candidate_answer: ctx.answer,\n  judge_a: a,\n  judge_b: b,\n  judge_avg: avg,\n  human_score: ctx.human_score,\n  abs_error: err,\n  passed: avg >= 4 ? 'yes' : 'no',\n  agrees: err <= 1 ? 'yes' : 'no',\n  judge_disagree: Math.abs(a - b) >= 2 ? 'yes' : 'no'\n}};"
      },
      "id": "47a1a00b-d9c1-4eed-be66-f65337c4745b",
      "name": "Parse Judge B",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1770,
        200
      ]
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "eval_items",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "run_id": "={{ $json.run_id }}",
            "question_id": "={{ $json.question_id }}",
            "category": "={{ $json.category }}",
            "candidate_answer": "={{ $json.candidate_answer }}",
            "judge_a": "={{ $json.judge_a }}",
            "judge_b": "={{ $json.judge_b }}",
            "judge_avg": "={{ $json.judge_avg }}",
            "human_score": "={{ $json.human_score }}",
            "passed": "={{ $json.passed }}",
            "agrees": "={{ $json.agrees }}",
            "judge_disagree": "={{ $json.judge_disagree }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "run_id",
              "displayName": "run_id",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "question_id",
              "displayName": "question_id",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "category",
              "displayName": "category",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "candidate_answer",
              "displayName": "candidate_answer",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "judge_a",
              "displayName": "judge_a",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "judge_b",
              "displayName": "judge_b",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "judge_avg",
              "displayName": "judge_avg",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "human_score",
              "displayName": "human_score",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "passed",
              "displayName": "passed",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "agrees",
              "displayName": "agrees",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "judge_disagree",
              "displayName": "judge_disagree",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "aa821590-1885-48c3-849b-78523d0f17b5",
      "name": "Save Item Results",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        2050,
        440
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "const items = $input.all().map((i) => i.json);\nconst n = items.length || 1;\nconst pct = (c) => Math.round(c / n * 1000) / 10;\nconst passed = items.filter((i) => i.passed === 'yes').length;\nconst humanPassed = items.filter((i) => i.human_score >= 4).length;\nconst agrees = items.filter((i) => i.agrees === 'yes').length;\nconst disagree = items.filter((i) => i.judge_disagree === 'yes').length;\nconst mae = Math.round(items.reduce((s, i) => s + i.abs_error, 0) / n * 100) / 100;\nconst summary = 'Run ' + items[0].run_id + ': ' + items.length + ' items. Judge pass rate ' + pct(passed) + '%, human pass rate ' + pct(humanPassed) + '%, judge-human agreement ' + pct(agrees) + '%, mean absolute error ' + mae + ', judge disagreement ' + pct(disagree) + '%.';\nreturn [{ json: { run_id: items[0].run_id, items: items.length, pass_rate: pct(passed), human_pass_rate: pct(humanPassed), agreement_rate: pct(agrees), mean_abs_error: mae, judge_disagreement_rate: pct(disagree), summary } }];"
      },
      "id": "74d74d9d-8eeb-46ad-acf0-c9f670307a53",
      "name": "Aggregate Run",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2050,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "eval-reports",
          "mode": "name"
        },
        "text": "=:bar_chart: *LLM eval run {{ $json.run_id }}*\nItems: {{ $json.items }} | Judge pass rate: {{ $json.pass_rate }}% | Human pass rate: {{ $json.human_pass_rate }}%\nJudge-human agreement: {{ $json.agreement_rate }}% | Mean abs error: {{ $json.mean_abs_error }}\nJudge disagreement: {{ $json.judge_disagreement_rate }}%",
        "otherOptions": {}
      },
      "id": "701a6a2d-5ca9-46ae-9393-6d0ad041c5ce",
      "name": "Slack Run Summary",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        2330,
        60
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "fromEmail": "YOUR_EMAIL@gmail.com",
        "toEmail": "YOUR_EMAIL@gmail.com",
        "subject": "=Eval run {{ $json.run_id }}",
        "emailFormat": "text",
        "text": "={{ $json.summary }}",
        "options": {}
      },
      "id": "b5afb9a5-2079-421f-9709-2b9eaa9b11be",
      "name": "Send Run Report",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [
        2330,
        240
      ],
      "credentials": {
        "smtp": {
          "id": "REPLACE_SMTP",
          "name": "SMTP account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "eval_runs",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "run_id": "={{ $json.run_id }}",
            "items": "={{ $json.items }}",
            "pass_rate": "={{ $json.pass_rate }}",
            "human_pass_rate": "={{ $json.human_pass_rate }}",
            "agreement_rate": "={{ $json.agreement_rate }}",
            "mean_abs_error": "={{ $json.mean_abs_error }}",
            "judge_disagreement_rate": "={{ $json.judge_disagreement_rate }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "run_id",
              "displayName": "run_id",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "items",
              "displayName": "items",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "pass_rate",
              "displayName": "pass_rate",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "human_pass_rate",
              "displayName": "human_pass_rate",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "agreement_rate",
              "displayName": "agreement_rate",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "mean_abs_error",
              "displayName": "mean_abs_error",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "judge_disagreement_rate",
              "displayName": "judge_disagreement_rate",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "b6f3f3e7-c566-41a9-8c2b-61941ada6d55",
      "name": "Save Run",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        2330,
        440
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "Candidate Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Candidate Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Candidate Agent": {
      "main": [
        [
          {
            "node": "Parse Answer",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Judge A Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Judge A Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Judge A Agent": {
      "main": [
        [
          {
            "node": "Parse Judge A",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Judge B Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Judge B Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Judge B Agent": {
      "main": [
        [
          {
            "node": "Parse Judge B",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Test Set",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Test Set": {
      "main": [
        [
          {
            "node": "Candidate Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Answer": {
      "main": [
        [
          {
            "node": "Judge A Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Judge A": {
      "main": [
        [
          {
            "node": "Judge B Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Judge B": {
      "main": [
        [
          {
            "node": "Save Item Results",
            "type": "main",
            "index": 0
          },
          {
            "node": "Aggregate Run",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Aggregate Run": {
      "main": [
        [
          {
            "node": "Slack Run Summary",
            "type": "main",
            "index": 0
          },
          {
            "node": "Send Run Report",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Run",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
