# SSH Action

A GitHub Action that writes a private key under `~/.ssh/`, appends host keys to `~/.ssh/known_hosts`, and adds a `Host` entry to `~/.ssh/config` so you can run `ssh` against your server from the workflow.

## Inputs

| Input | Required | Default | Description |
|--------|----------|---------|-------------|
| `NAME` | No | `server` | Unique label for this server. Slugified (lowercase, safe characters) and used as the SSH config `Host` alias. |
| `SSH_USER` | Yes | — | SSH username for the **target** host. |
| `SSH_HOST` | Yes | — | Hostname or IP of the **target** host. |
| `SSH_PORT` | No | `22` | SSH port on the target. |
| `SSH_KEY` | Yes | — | Full PEM/OpenSSH private key string for the **target** host. |
| `SSH_BASTION_HOST` | No | — | If set, connect via a bastion/jump host using `ProxyJump`. |
| `SSH_BASTION_USER` | No | same as `SSH_USER` | SSH username on the bastion (only used when `SSH_BASTION_HOST` is set). |
| `SSH_BASTION_PORT` | No | `22` | SSH port on the bastion. |
| `SSH_BASTION_KEY` | No | — | Optional separate private key for the bastion when it differs from `SSH_KEY`. If omitted, the bastion entry has no `IdentityFile` line; SSH may use the agent or default keys. |

## Outputs

| Output | Description |
|--------|-------------|
| `SERVER` | The `Host` alias written to `~/.ssh/config` (derived from `NAME`). Use as `ssh ${{ steps.<id>.outputs.SERVER }} …`. |

## Usage

### Direct connection

```yaml
name: ssh command
on: push
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - id: ssh
        uses: ./ # or your-org/ssh-action@v1
        with:
          SSH_HOST: ${{ secrets.SSH_HOST }}
          SSH_PORT: "22"
          SSH_USER: ${{ secrets.SSH_USER }}
          SSH_KEY: ${{ secrets.SSH_KEY }}
      - run: ssh ${{ steps.ssh.outputs.SERVER }} pwd
```

### Bastion / jump host

When `SSH_BASTION_HOST` is set, the action:

1. Runs `ssh-keyscan` for both the bastion and the target host.
2. Adds a `Host <name>-bastion` entry for the jump host.
3. Adds `ProxyJump <name>-bastion` on the target `Host <name>` entry.

If the bastion uses a **different** key than the final host, set `SSH_BASTION_KEY`. The bastion entry then uses `IdentityFile` for that key; the target still uses `SSH_KEY`.

```yaml
      - id: ssh
        uses: ./
        with:
          NAME: prod
          SSH_HOST: ${{ secrets.SSH_HOST }}
          SSH_USER: ${{ secrets.SSH_USER }}
          SSH_KEY: ${{ secrets.SSH_KEY }}
          SSH_BASTION_HOST: ${{ secrets.BASTION_HOST }}
          SSH_BASTION_USER: ${{ secrets.BASTION_USER }}
          SSH_BASTION_PORT: "22"
          SSH_BASTION_KEY: ${{ secrets.BASTION_SSH_KEY }}
      - run: ssh ${{ steps.ssh.outputs.SERVER }} 'hostname'
```

### Multiple servers

Run the action once per server with a **unique** `NAME` each time.

```yaml
      - id: ssh-foo
        uses: ./
        with:
          NAME: foo
          SSH_HOST: ${{ secrets.FOO_HOST }}
          SSH_PORT: "22"
          SSH_USER: ${{ secrets.FOO_USER }}
          SSH_KEY: ${{ secrets.FOO_KEY }}
      - id: ssh-bar
        uses: ./
        with:
          NAME: bar
          SSH_HOST: ${{ secrets.BAR_HOST }}
          SSH_USER: ${{ secrets.BAR_USER }}
          SSH_KEY: ${{ secrets.BAR_KEY }}
      - run: ssh ${{ steps.ssh-foo.outputs.SERVER }} pwd
      - run: ssh ${{ steps.ssh-bar.outputs.SERVER }} pwd
```
