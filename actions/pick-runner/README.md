# pick-runner

Route a job to an org **self-hosted** runner when one is online, otherwise to a
**GitHub-hosted** runner — instead of queueing forever (GitHub `runs-on` labels are
an AND, there is no native "A else B").

```yaml
jobs:
  pick:
    runs-on: ubuntu-latest
    outputs:
      runs-on: ${{ steps.pick.outputs.runs-on }}
    steps:
      - id: pick
        uses: eurocontrol-asu/.github/actions/pick-runner@<commit-sha>
        with:
          label: ubbla
          client-id: ${{ vars.RUNNERS_APP_CLIENT_ID }}
          private-key: ${{ secrets.RUNNERS_APP_PRIVATE_KEY }}

  test:
    needs: pick
    runs-on: ${{ fromJSON(needs.pick.outputs.runs-on) }}
    steps: [...]
```

| Input | Default | |
|---|---|---|
| `label` | — | self-hosted label to look for |
| `fallback` | `ubuntu-latest` | GitHub-hosted label used otherwise |
| `wait-minutes` | `5` | wait for a matching runner to come online (e.g. after a reboot) |
| `require-idle` | `false` | `false`: online is enough, the job queues behind running jobs |
| `client-id`, `private-key` | empty | org GitHub App `asu-runner-picker` (Self-hosted runners: Read-only) |

Outputs: `runs-on` (JSON for `fromJSON`), `selected` (`self-hosted` / `fallback`).

**Fallback is loud**: a `::warning::` and a step-summary line. Tests that need the
self-hosted hardware (e.g. `skipif(not torch.cuda.is_available())`) are *skipped* on
the fallback — a green fallback run does **not** validate the GPU path.

**Security**: no credentials (pull requests from forks get no secrets) => immediate
fallback, never routed to self-hosted. The App can only *read* runner status. Which
repos may use the runners at all is enforced by the org runner group, not by this action.
