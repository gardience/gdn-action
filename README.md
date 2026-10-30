# Gardience gdn action

Run [`gdn`](https://gardience.com/docs/administration/gdn-cli.html) against your own Gardience server from a GitHub
Actions workflow: check the fleet before a deploy, run a command on a node group, or run an Ansible playbook or a
runbook. Every change still goes through Gardience's permission checks, approvals, change windows, and audit log.

`gdn` is downloaded from your own server on each run, so it always matches your Gardience version.

## Before you start

- **The runner must reach your console.** Gardience runs inside your network, so most teams use a
  [self-hosted runner](https://docs.github.com/actions/hosting-your-own-runners) on that network.
- **An API key** from **Settings > API keys** in the console, saved as a repository or environment secret. A read key
  is enough for checks. Running anything needs an operate key. If the key requires approval, the job waits while a
  second person approves the run in the console.
- **Python 3** on the runner. `gdn` uses only the standard library.

## Inputs

| Input | Required | Meaning |
|---|---|---|
| `server` | yes | Console address, for example `https://gardience.example.com`. |
| `api-key` | yes | The API key. Pass it from a secret. |
| `args` | yes | The `gdn` command and its arguments, as you would type them. |
| `ca-cert` | no | The console's certificate as PEM text, when it is self-signed (`config/tls/gardience.crt` on the server). |
| `insecure` | no | `true` turns certificate checks off. For a quick test only. |

`args` is split the way a shell would split it, but it is never run through a shell.

The step fails when `gdn` exits with a non-zero code: `1` when the request could not be made, `2` when a run finished
without succeeding.

## Examples

Check that a command works on the web group before deploying:

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, linux]
    steps:
      - uses: gardience/gdn-action@v1
        with:
          server: ${{ vars.GARDIENCE_URL }}
          api-key: ${{ secrets.GARDIENCE_KEY }}
          ca-cert: ${{ vars.GARDIENCE_CA }}
          args: run web 'systemctl is-active nginx' --reason "pre-deploy check, ${{ github.sha }}"
```

Run a playbook from the repository in check mode on every pull request, and for real on merge:

```yaml
      - uses: actions/checkout@v4
      - uses: gardience/gdn-action@v1
        with:
          server: ${{ vars.GARDIENCE_URL }}
          api-key: ${{ secrets.GARDIENCE_KEY }}
          args: playbook site.yml -l web ${{ github.event_name == 'pull_request' && '--check' || '' }} --reason "${{ github.repository }}@${{ github.sha }}"
```

Run a runbook:

```yaml
      - uses: gardience/gdn-action@v1
        with:
          server: ${{ vars.GARDIENCE_URL }}
          api-key: ${{ secrets.GARDIENCE_KEY }}
          args: runbook restart-service -e nodegroup=web -e service=nginx --reason "release ${{ github.ref_name }}"
```

Use `gdn runbooks` to see the runbook IDs and inputs on your server.

## A fleet gate without gdn

To stop a pipeline when machines are out of compliance, read the REST API directly:

```yaml
      - name: Stop if any web node is not compliant
        env:
          GARDIENCE_URL: ${{ vars.GARDIENCE_URL }}
          GARDIENCE_KEY: ${{ secrets.GARDIENCE_KEY }}
        run: |
          bad=$(curl -fsS -H "Authorization: Bearer $GARDIENCE_KEY" "$GARDIENCE_URL/gardience/api/v1/fleet" \
            | jq -r '.nodes[] | select(.node | startswith("web")) | select(.compliant == false or .stale) | .node')
          if [ -n "$bad" ]; then echo "Not compliant: $bad"; exit 1; fi
```

## Other CI systems

The same commands work in Jenkins, GitLab CI, and anything else that can run a shell. See
[Gardience in your pipeline](https://gardience.com/blog/gardience-ci-pipeline-jenkins-github-gitlab/).

## License

MIT. See [LICENSE](LICENSE).
