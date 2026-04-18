# o2switch Publish Action

Whitelist the GitHub runner IP on O2Switch, deploy with rsync, then remove the IP.

## Inputs

- `o2switch_host` (required): O2Switch host, e.g. `srv123.o2switch.net`
- `o2switch_username` (required): cPanel username
- `o2switch_api_token` (required): cPanel API token
- `ssh_user` (required): SSH username
- `ssh_private_key` (required): SSH private key
- `remote_path` (required): Remote path to deploy to
- `local_path` (optional, default `./`): Local path to deploy from
- `ssh_port` (optional, default `22`): SSH port
- `rsync_args` (optional, default `-rlgoDzvc -i --delete-after`): rsync flags
- `exclude` (optional): comma-separated list of rsync excludes, e.g. `/dist/,/node_modules/`

## Outputs

- `public_ip`: public IP of the GitHub runner

## Example workflow

```yaml
name: Build and Deploy

on:
  push:
    branches: [ main ]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: npm ci && npm run build

      - name: Publish to O2Switch
        uses: your-org/o2switch-publish@v1
        with:
          o2switch_host: ${{ secrets.O2SWITCH_HOST }}
          o2switch_username: ${{ secrets.O2SWITCH_USERNAME }}
          o2switch_api_token: ${{ secrets.O2SWITCH_API_TOKEN }}
          ssh_user: ${{ secrets.O2SWITCH_SSH_USER }}
          ssh_private_key: ${{ secrets.O2SWITCH_SSH_PRIVATE_KEY }}
          remote_path: ${{ secrets.O2SWITCH_REMOTE_PATH }}
          local_path: ./
          exclude: /node_modules/,/dist/
```

## Notes

- This action uses the cPanel UAPI `SshWhitelist` module via:
  `https://HOST:2083/execute/SshWhitelist/...`
- The runner IP is added before deploy and removed on exit (even if rsync fails).
