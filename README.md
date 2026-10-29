# Aliy'aa frontend challenge

Build a chatbot for Aliy'aa: a physician types a clinical question and the answer streams
in on screen.

**API:** https://aliyaa-interview-stream.a7madadlan.workers.dev

No API key needed. It accepts requests from any origin, so a page opened from your disk
works. Use any stack and any AI assistant you like.

## API

### `POST /api/ask`: Aliy'aa answers

Request body: `{"question": "..."}` (3 to 500 characters). The response is
`text/event-stream`. Each event is one frame: a `data: ` line holding JSON, then a blank
line. This is the same contract the Aliy'aa app uses in production:

```ts
type AskEvent =
  | { type: "stage"; stage: Stage }                  // progress through the pipeline
  | { type: "clarify"; title: string; body: string } // the system needs one more detail
  | { type: "assumption"; text: string }             // something the answer assumes
  | { type: "answer"; answer: AnswerPayload }        // the verified answer, once
  | { type: "error"; message: string }
  | { type: "done"; elapsed_ms: number; usage?: object; cost_usd?: number | null };

interface Stage {
  id: string;                     // e.g. "4.1"
  layer: 1 | 2 | 3 | 4 | 5 | 6;
  name: string;                   // "Hybrid retrieval"
  status: "ok" | "warn" | "stop" | "info" | "idle";
  headline: string;               // may contain **bold** and _italic_
  detail: string[];
}

type AnswerPayload =
  | { query: string; abstained: true; reason: string;
      partial_sources: SourcePayload[]; disclaimer: string; suggestion?: string | null }
  | { query: string; abstained: false;
      claims: { text: string; sources: SourcePayload[] }[];
      conflicts: string[]; sources: SourcePayload[]; disclaimer: string };

interface SourcePayload {
  id: string;
  citation: string;
  recommendation_class: string | null;   // the grade as the source wrote it, e.g. "Grade B"
  evidence_level: string | null;
  grading_system: string | null;         // e.g. "USPSTF"
  source_url: string | null;
  // ...plus provenance fields: evidence_tier, design_tier, source_type, doi,
  //    guideline_body, guideline_version, language, published_on
}
```

Errors before the stream starts come back as JSON `{"error": "..."}` with an HTTP status.

### `POST /v1/chat/completions`: a test model

Speaks the OpenAI Chat Completions format, streaming included, so `fetch` or the
official `openai` SDK works against it (`baseURL: "<API>/v1"`, any `apiKey`). Models:
`aliyaa-test` and `aliyaa-test-short`.

```json
{ "model": "aliyaa-test", "stream": true,
  "messages": [{ "role": "user", "content": "your question" }] }
```

Keep the API URL easy to change in your app: during the session we will point it at other
recorded runs.
