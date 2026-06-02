# Workflow build notes

## Build phases

1. **Trigger.** Add the Form Trigger with a single required `Topic` text field and fetch a
   test submission so downstream nodes have real data to reference.
2. **Post agent.** Add the AI Agent (Tools Agent). Set the prompt source to "Define below"
   and the user message to research and write about `{{ $json.Topic }}`. Write a system
   message that defines the LinkedIn voice (hook, short lines, hashtags, post-only output).
   Attach a Gemini text chat model.
3. **Research tool.** Attach the Tavily Search tool to the post agent. Set the query to
   "From AI" so the agent writes its own search query.
4. **Image prompt agent.** Add a second AI Agent on a Gemini text chat model. Feed it
   `{{ $json.output }}` and instruct it to return a single image-generation prompt only.
5. **Image generation.** Add an HTTP Request node. Use `GET`, no body, no auth, and set
   `Response Format: File` so the returned image is stored as binary in `data`.
6. **Delivery.** Add the Gmail node in `Send` mode. Set `To`, set the body to
   `{{ $('Linkedin Post agent').item.json.output }}`, and attach the `data` binary field.

## Setup checklist

- [ ] Google Gemini API credential created and attached to both agent chat models.
- [ ] Tavily API credential created and attached to the search tool.
- [ ] Gmail OAuth2 credential created and attached to the Send node.
- [ ] Both agents use a Gemini text model (for example `gemini-2.5-flash-lite`).
- [ ] Image node returns a file in the `data` field.
- [ ] Gmail `To` field changed from `review@example.com` to the review inbox.
- [ ] Test topic submitted and a single run verified end to end.
- [ ] Email received with the post body and the attached image.

## Debugging tips

- Verify each node in isolation with "Execute step" before running the whole workflow.
- A `429 ... limit: 0` from a Gemini image model is a billing block, not a temporary rate
  limit. Use a billed key or generate the image through a free image endpoint.
- After changing an HTTP node from `POST` to `GET`, manually turn off Send Body, Send
  Headers, and Authentication. n8n does not clear them automatically, and a leftover body
  or auth header can cause a `500` from the image endpoint.
- If the email body is empty, check the node-name reference and its capitalization in
  `{{ $('Linkedin Post agent').item.json.output }}`.
- A free image endpoint can occasionally return `500` under load; wait a few seconds and
  retry.

## Cost and quota notes

- Text generation (both agents) runs on the Gemini free tier.
- Image generation is the only paid capability in the chain; it is handled by a free image
  endpoint to keep the workflow at zero cost.
- Pin the post agent during development so iterating on the image and Gmail steps does not
  re-call the model or Tavily on every test.
- Free Gemini text quotas are per-minute and per-day, and are per-model; a billed key is
  required for sustained use.
