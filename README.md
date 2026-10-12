# task-master

A personal to-do list that works from the Claude app on PC and mobile. I add and prioritize tasks and hand any of them to a Claude agent, which works in its own session and asks me only critical questions.

- **App:** https://claude.ai/artifact/9XBCgHprdE8jEVJRrBLaS3. Open it in the Claude app on PC or phone. It's private to the owner. Source: [app/index.html](app/index.html)
- **Requirements and status:** [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md)
- **Agent rules:** [CLAUDE.md](CLAUDE.md)

## Updating the app

Edit `app/index.html`, then ask Claude to republish it to the same artifact URL. Keep the `ARTIFACT_URL` constant in the file pointing at that URL, because delegated agents use it to find the task store.
