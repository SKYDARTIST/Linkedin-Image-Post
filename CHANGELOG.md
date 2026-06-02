# Changelog

## v1.0 — Initial working build

- On form submission -> LinkedIn Post agent -> Image prompt agent -> generate image ->
  Gmail, end to end.
- LinkedIn Post agent backed by a Gemini text model with the Tavily Search tool for live
  research.
- Image prompt agent (Gemini text model) derives a single image prompt from the finished
  post.
- Image generated through an HTTP Request node and delivered to Gmail as the email body
  plus an attachment.
- Output delivered to Gmail for human review instead of auto-publishing to LinkedIn.

### Design decisions
- **Text vs image models are separated.** Both agents run on a Gemini text model (free
  tier). Image generation is a separate, paid capability, so it is handled by a free image
  endpoint via HTTP Request to keep the workflow at zero cost.
- **Gmail output instead of LinkedIn.** Delivering to an inbox adds a human review step,
  avoids publishing unfinished drafts during testing, and removes the LinkedIn API setup
  barrier. The final node can be swapped for a publisher once drafts are trusted.

### Fixes made during the build
- Moved the image model out of an agent's chat-model slot; agents run on text models only.
- Replaced the billed Gemini image model (which returned `429 ... limit: 0` on the free
  tier) with a free image endpoint.
- Turned off the leftover JSON body, headers, and authentication after switching the image
  node from `POST` to `GET`, resolving a `500` response.
- Set the image node to `Response Format: File` so the Gmail node can attach the binary
  directly without a base64 conversion step.
- Set the Gmail body to `{{ $('Linkedin Post agent').item.json.output }}` to read the post
  from the writing agent rather than the immediately preceding image node.

### Known limitations
- Output is delivered for review, not auto-published; add a publishing node to go live.
- The free image endpoint can be slow or unavailable; retry or use a billed image model.
- Free-tier Gemini text quotas require a billed key for sustained use.
