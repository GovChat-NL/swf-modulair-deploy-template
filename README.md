> # ⚠️ NOTE: temporary name
> This repository has the name `swf-modulair-deploy-template` because this is a pilot for a
> modular GovChat. Súdwest-Fryslân started the pilot with the approval of GovChat-NL. If the
> pilot is a success and all parties agree, the name can become `GovChat-NL-deploy-template`.

# swf-modulair-deploy-template

This is the start point for the deploy repository of an organisation. An organisation
makes its own repository from this template **one time**. That repository contains the
decisions of the organisation: which kern, which apps, which versions and which settings.
Rancher Fleet reads that repository and deploys it.

This is one of four repositories that belong together:

| Repository | Function |
|---|---|
| [`swf-modulair-kern`](https://github.com/GovChat-NL/swf-modulair-kern) | The basic chat. |
| [`swf-modulair-app-time`](https://github.com/GovChat-NL/swf-modulair-app-time) | The trial app, and the example of an app. |
| [`swf-modulair-manager`](https://github.com/GovChat-NL/swf-modulair-manager) | The `govchat` command, and the rules for an app. |
| **`swf-modulair-deploy-template`** | The start point for a deploy repository (this repository). |

## The rule

Each change is a pull request. A second person approves it. The merge is the deployment.
Nobody changes the cluster manually.

## The files

```
govchat                        starts the govchat command in a fixed version
govchat.yaml                   the environments and the permitted sources
shared/kern.yaml               YOU: settings of the kern for all environments
shared/branding.yaml           generated from the files in branding/
branding/                      YOU: the logo and the icons
acceptatie/
  kern/
    settings.yaml              YOU: the web address and the ingress of this environment
    kern.yaml                  copy from the kern repository. Do not edit.
    kern-values.yaml           copy from the kern repository. Do not edit.
    generated.yaml             GENERATED. Do not edit.
    fleet.yaml                 GENERATED. Do not edit.
  apps/
    time/
      app.yaml                 copy from the app repository. Do not edit.
      settings.yaml            YOU: the settings of the app (only if the app has settings)
      manifests.yaml           GENERATED. Do not edit.
      fleet.yaml               GENERATED. Do not edit.
productie/                     the same structure
lokaal/                        the same structure, for a test cluster on your computer
```

- You write the files with `YOU`.
- The `govchat` command writes all other files.
- A folder in `apps/` means that the app is installed.

The environments are `acceptatie` (namespace `govchat-acc`), `productie` (namespace
`govchat`) and `lokaal`. The template contains the kern and the trial app `time`.

## What you need

- [Node.js](https://nodejs.org) 20 or newer, and `git`.
- For the commands `secrets`, `verify` and `local`: `kubectl`, and `helm` for `local`.
- On Windows: use WSL2.

Run `./govchat --help` to see all commands.

## Make your own repository (one time)

Do not make a fork. Make a copy. After that, there is no connection to this template.

```sh
git clone https://github.com/GovChat-NL/swf-modulair-deploy-template.git govchat-deploy
cd govchat-deploy
rm -rf .git && git init -b main
git remote add origin https://gitea.example.nl/<organisation>/govchat-deploy.git
```

Then:

1. Give the values of your organisation:
   ```sh
   ./govchat init
   ```
   The command asks for the web address of each environment, and for the values of Entra ID and Azure OpenAI.
2. Look at the ingress settings in `acceptatie/kern/settings.yaml` and `productie/kern/settings.yaml`. Change them for your cluster. Then run `./govchat build`.
3. If you have your own logo: replace the files in `branding/` and run `./govchat branding`. Then run `./govchat build`.
4. Commit and push.
5. Protect the branch `main` on your Git server: a pull request is required, with one approval from a second person.
6. Create the Secrets, for each environment:
   ```sh
   AZURE_MODEL_KEY=... OPENID_CLIENT_SECRET=... ./govchat secrets --env acceptatie
   ```
   Secrets are never in Git.
7. In Rancher, make one Git repository (`GitRepo`) for each environment. Use the branch `main`. Use the path `acceptatie` for the first and `productie` for the second.

In Entra ID, the app registration needs for each environment the redirect URI
`https://<web address>/oauth/openid/callback` and the app roles `govchat-user` and
`govchat-admin`. The roles must be in the ID token (claim `roles`).

## Add an app

1. Add the app to `acceptatie`:
   ```sh
   ./govchat app add time --env acceptatie
   ```
   Run `./govchat app list` to see the known apps. For an app that is not in the list, give the full address.
2. If the command shows settings or Secrets, do what the command tells you.
3. Look at the change with `git diff`. Commit on a branch and open a pull request.
4. A second person looks at the pull request and merges it. Fleet deploys the change.
5. Check the environment:
   ```sh
   ./govchat verify --env acceptatie
   ```

## Move a change to production

1. Test on `acceptatie`.
2. Copy the versions to `productie`:
   ```sh
   ./govchat promote
   ```
3. Commit on a branch and open a second pull request. This merge is the release.
4. Run `./govchat verify --env productie`.

To go back to an earlier version, give that version: `./govchat app add time --version v2.0.0 --env productie`.
This is also a pull request.

## Remove an app

```sh
./govchat app remove time --env acceptatie
```

Commit on a branch and open a pull request. After the merge, Fleet deletes all parts of
the app.

## Change the version of the kern

```sh
./govchat kern set v0.2.0 --env acceptatie
```

Use `latest` for the newest version. The pull request shows each change in the settings of
the kern.

## Test on your computer

Use a test cluster on your computer (kind, k3d or Rancher Desktop). The environment
`lokaal` uses a replacement model and a local account. Azure and Entra ID are not necessary.

```sh
./govchat local up
./govchat verify --env lokaal
kubectl -n govchat port-forward service/librechat-librechat 3080:3080
```

Go to http://localhost:3080 and make an account. The replacement model always gives the
same answer. It cannot call tools.

To test an app that you are making, add it from its folder:

```sh
./govchat app add ../my-app --env lokaal
./govchat local up
```

To stop: `./govchat local down`.

## The check in a pull request

The file `.github/workflows/check.yml` runs `./govchat check` for each pull request.
Gitea Actions and GitHub Actions can run this file. The check fails if a person changed an
input file and did not run `./govchat build`.

## Updates

The kern, the apps and the `govchat` command each have a version. To update, change a
version in this repository. Do not merge with the template.

- The kern: `./govchat kern set <version> --env <environment>`.
- An app: `./govchat app add <name> --version <version> --env <environment>`.
- The command: change the version in the file `govchat`.

## Origin

The settings, the branding and the structure come from the branch `swf/modulairKern` of
`GovChat-NL/GovChat-NL`, commit `7e2474d` (20 November 2026). The licence is not chosen
yet. The proposal is EUPL-1.2. The maintainers of GovChat-NL make the decision.
