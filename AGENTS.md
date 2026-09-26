# Instructions for agents

## Upstream is read-only

- `upstream` points to `MBrekhof/xafskills` and is for fetch and read operations only.
- Never push commits or submit pull requests to `MBrekhof/xafskills`.
- Publish changes only to the `kashiash/xafskills` fork when the user requests publication.
- Do not change the upstream push URL or bypass the local `pre-push` hook.
- Enable the repository hook after checkout with `git config core.hooksPath .githooks`.
