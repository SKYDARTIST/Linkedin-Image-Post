# Architecture

## System flow

1. **On form submission** (Form Trigger) emits one item per submission. The submitted
   topic is available at `$json.Topic`.
2. **LinkedIn Post agent** (Tools Agent) receives the topic. Its user message asks it to
   research and write a post about `{{ $json.Topic }}`; its system message defines the
   LinkedIn voice and output rules. It runs on a Gemini text model and has the Tavily
   Search tool attached on its `ai_tool` input.
3. **Image prompt agent** (Tools Agent) receives the finished post on `{{ $json.output }}`
   and returns a single image-generation prompt. It runs on a Gemini text model.
4. **generate image** (HTTP Request) takes the image prompt and returns the picture as a
   binary file in the `data` field.
5. **Send a message** (Gmail) sends the post as the email body and attaches the generated
   image.

## Data dependencies

| From | To | Channel | Field used |
| --- | --- | --- | --- |
| On form submission | LinkedIn Post agent | main | `Topic` |
| LinkedIn Post agent | Image prompt agent | main | `output` |
| Image prompt agent | generate image | main | `output` |
| generate image | Send a message | main | binary `data` |
| Gemini Chat Model | LinkedIn Post agent | ai_languageModel | - |
| Tavily Search | LinkedIn Post agent | ai_tool | search query |
| Gemini Chat Model | Image prompt agent | ai_languageModel | - |

## Model selection contract

- **Text agents** (post + image prompt) use a Gemini text model, for example
  `gemini-2.5-flash-lite`. Text generation is available on the Gemini free tier.
- **Image generation** is a separate, paid capability. Gemini image models such as
  `gemini-2.5-flash-image-preview` return `429 ... limit: 0` on the free tier — a billing
  block, not a temporary rate limit. To keep the workflow free, image generation runs
  through an HTTP Request node against a free image endpoint. A billed Gemini image model
  can be substituted when higher quality is required.

## Image node contract

- Method: `GET`, no body, no authentication, no extra headers.
- The endpoint returns raw image bytes, so the node uses `Response Format: File` and writes
  the result to the `data` binary field.
- If a billed Gemini image model is used instead, the call becomes a `POST` with
  `generationConfig.responseModalities: ["TEXT","IMAGE"]`, and the returned base64 lives at
  `candidates[0].content.parts.filter(p => p.inlineData)[0].inlineData.data` and must be
  converted to a file before the Gmail node.

## Expression conventions

- Inside agent nodes, prefer `{{ $json.<field> }}` (the direct input item) over
  `{{ $('Node').item.json.<field> }}`. The `.item` accessor depends on paired-item
  tracking, which is unavailable inside the agent's batch executor.
- The Gmail node reads the post text from the writing agent by name —
  `{{ $('Linkedin Post agent').item.json.output }}` — because the node directly before it
  (generate image) holds the binary image, not the post text. Gmail is a standard node, so
  the `.item` accessor is safe here.
- The Tavily tool query is set to "From AI" so the agent writes its own search query rather
  than using a hardcoded string.

## Security posture

- The committed workflow JSON contains no credentials, webhook IDs, instance ID, or
  internal record IDs. Those are stripped before publishing.
- Secrets live only in the n8n credential store and are referenced by node configuration at
  run time, never serialized into the exported file in plaintext.
- `.gitignore` blocks `.env`, `node_modules`, the `.n8n/` data directory, and any
  key/secret/token files from ever being committed.
