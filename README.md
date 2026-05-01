# ⬆️ publish-to-o2switch 🐯

All-in-one action to publish content to o2switch 🐯, managing IP whitelisting, archiving, releases, exclusions...

Also tested and validated on [Gitea](https://docs.gitea.com/usage/actions/ "Gitea") 🍵.

## Quickstart example

```yaml
name: publish-to-o2switch quickstart

on:
  push:
    branches: [ main ]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      # Custom steps...
      - name: Build
        run: npm ci && npm run build

      # Publishing with the action
      - name: Publish to o2switch
        uses: dix/publish-to-o2switch@v1
        with:
          o2switch_host: ${{ secrets.O2SWITCH_HOST }}
          o2switch_username: ${{ secrets.O2SWITCH_USER }}
          o2switch_api_token: ${{ secrets.O2SWITCH_API_TOKEN }}
          ssh_private_key: ${{ secrets.O2SWITCH_SSH_PRIVATE_KEY }}
          remote_path: ~/publish-to-o2switch
          local_path: ./
          exclude: /node_modules/,/dist/
```

## Documentation

### Setup

We need to create the required secrets in the GitHub repository.

- Go to https://github.com/USER/PROJECT/settings/secrets/actions or `Settings -> Secrets and variables -> Actions`
- On cPanel, retrieve O2Switch host related to subscription by going to Technical Space and extract the DNS from the URL
  that should looks like `https://xyz.o2switch.net:2083/`; in this case host is `xyz.o2switch.net`. Then create a
  Repository Secret called `O2SWITCH_HOST` with this value
- On cPanel, retrieve 02Switch username at the top right under `General Information -> Current User` and create a
  Repository Secret called `O2SWITCH_USER` with this value
- On cPanel, get an API Token under `Manage API Tokens` and create a Repository Secret called `O2SWITCH_API_TOKEN` with
  this value
- Go through the `SSH key setup` part below
- For the other inputs, you can hard-code them in the workflow YAML or use variables or secrets

#### SSH key setup

First, we recommend creating a dedicated SSH key for the publishing process, to avoid mixing things with actual human
access from your computer.

```bash
ssh-keygen -t ed25519 -C "publish-to-o2switch" -f "/REPLACE/LOCAL/PATH/publish-to-o2switch" -N ""
```

Create a Repository Secret called `O2SWITCH_SSH_PRIVATE_KEY` with the content of the newly created `publish-to-o2switch`
file, don't forget to put a newline at the end:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
xyz
-----END OPENSSH PRIVATE KEY-----

```

To allow this key, first copy the content of `publish-to-o2switch.pub`, then two options:

- Connect in SSH to the server, then add the public key to the existing ones with
  `echo "ssh-ed25519 XYZ... publish-to-o2switch" >> ~/.ssh/authorized_keys`
- On cPanel
    - Go to `SSH Access -> Manage SSH Keys -> Import Key`
    - Use `publish-to-o2switch` for the `name` field
    - Paste the content of `publish-to-o2switch.pub` in the `public key` field
    - Leave `private key` and `passphrase`
    - Click `Import`
    - Next click `Back to Manage Keys`
    - Under `Public Keys` and next to the new `publish-to-o2switch` key, click `Manage`
    - Finally, click on `Authorize`

Note: it's not clear if this step is doable through the API at the moment, thus the manual actions...

### Inputs

| name                 | details                                          | description                                                                 |
|----------------------|--------------------------------------------------|-----------------------------------------------------------------------------|
| `o2switch_host`      | **required**                                     | O2Switch host, e.g. `tigers.o2switch.net`                                   |
| `o2switch_username`  | **required**                                     | cPanel username                                                             |
| `o2switch_api_token` | **required**                                     | cPanel API token                                                            |
| `ssh_user`           | optional (defaults to `o2switch_username`)       | SSH username                                                                |
| `ssh_private_key`    | **required**                                     | SSH private key                                                             |
| `remote_path`        | **required**                                     | Remote path to deploy to (can be /home/USERNAME/xxx or ./xxx or ~/xxx       |
| `local_path`         | optional (default `./`)                          | Local path to deploy from                                                   |
| `ssh_port`           | optional (default `22`)                          | SSH port                                                                    |
| `rsync_args`         | optional (default `-rlgoDzvc -i --delete-after`) | rsync flags                                                                 |
| `exclude`            | optional                                         | Comma-separated list of publishing exclusions, e.g. `/dist/,/node_modules/` |
| `use_archive`        | optional (default `false`)                       | Use a tar.gz archive instead of rsync                                       |
| `use_releases`       | optional (default `false`)                       | Deploy to release directories and update a symlink                          |
| `releases_dir`       | optional (required if `use_releases=true`)       | Remote releases directory (only for `use_releases`)                         |
| `current_link`       | optional (required if `use_releases=true`)       | Remote symlink path for the current release (only for `use_releases`)       |
| `keep_releases`      | optional (default `3`)                           | Number of releases to keep (only for `use_releases`)                        |
| `skip_cleanup_whitelist` | optional (default `false`)                   | Skip the cleanup whitelist step                                              |

### Modes

The action support two modes: `default` and `releases`.

Managed with input `use_releases`.

#### default

Simply replace the existing content on the remote server by the new one.

#### releases

Creates a new subdirectory to publish content, updates symlink to point to new release and remove old releases.

The goal is to have multiple releases available on the remote server to easily switch/rollback between them.

Example:

1. Create `/home/USERNAME/tigers` directory on the remote server.
2. On cPanel, declare subdomain `tigers.mi5.com` to have its root at `tigers/current`.
3. Use the action with options `use_releases: true`, `releases_dir: "~/tigers/releases"` and
   `current_link: "~/tigers/current"`
4. The action will create a new release subdirectory in `/home/USERNAME/tigers/releases/xyz` and once published, will
   create/update a symlink on `/home/USERNAME/tigers/current` pointing to `/home/USERNAME/tigers/releases/xyz`
5. Subdomain `tigers.mi5.com` will then serve content of the new release
6. Rolling-back to an older release can be achieved simply by updating the symlink to make it point to an older release
   subdirectory

### Archiving

Depending on the number of files to publish, creating a local archive of the files, uploading it to the server and
extracting it there can provide much better performances than dealing with each file one-by-one.

The input `use_archive` is here to activate this option.

Usage greatly recommended in `releases` mode, since all files are uploaded in the new release directory.
In `default` mode, it's debatable. The use of `rsync` limits uploads to only the modified files, but a check is required
for each file, which can take quite some time if the number of files is in the hundreds or more.

### Whitelisting

By default, SSH access is blocked by o2switch, so we first need to allow the GitHub runner's random public IP through cPanel
API.

Since the number of authorized IPs is limited, we always delete the IP in the end, even if the deployment failed,
avoiding leaving a mess in your account.

For self-hosted runners, particularly for Gitea, use `skip_cleanup_whitelist: true` to skip this step and keep the IP whitelisted.

### Exclusions

The `exclude` input provides with the option to exclude files and directories, through a comma separated list, from the
deployment.
Supported for rsync/archiving and all modes.

## Notes

- Greatly inspired by the work of webaxones and friends
  in [this Gist](https://gist.github.com/webaxones/54a9aee13bd9152e900ef30a0fcef3ed "GitHub").
- This project is not affiliated with or endorsed by [O2Switch](https://www.o2switch.fr/ "o2switch") 🐯.
