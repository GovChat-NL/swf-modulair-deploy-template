> # ⚠️ LET OP: tijdelijke naam
> Deze repository heet nu `modulair-deploy-template` omdat dit een pilot is voor een modulaire GovChat,
> gestart door Súdwest-Fryslân met toestemming van GovChat-NL. Werkt de pilot en is iedereen
> het eens, dan kan de naam in de toekomst `GovChat-NL-deploy-template` worden.
> Deze uitrolbasis is tijdelijk en hoort bij `modulair-kern`.

# modulair-deploy-template

De uitrolbasis van GovChat: het startpunt waar een organisatie **één keer** een eigen
repository van maakt. In die eigen repository staan de keuzes van de organisatie: welke
kern, welke apps, in welke versie en met welke instellingen. Er staat geen code in.

Dit is één van vier repositories die bij elkaar horen:

| Repository | Wat |
|---|---|
| [`modulair-kern`](https://github.com/GovChat-NL/modulair-kern) | de kale chat |
| [`modulair-app-time`](https://github.com/GovChat-NL/modulair-app-time) | de proefapp, en het voorbeeld van hoe een app eruitziet |
| [`modulair-manager`](https://github.com/GovChat-NL/modulair-manager) | de app-manager, die de uitrol samenstelt; hier staat ook het appcontract |
| **`modulair-deploy-template`** | de uitrolbasis (deze repository) |

## Wat erin zit

| Waar | Van wie | Wat |
|---|---|---|
| `stacks/<omgeving>.yaml` | de organisatie | **Welke** onderdelen, elk met een adres en een versie. Verder niets |
| `values/<naam>.yaml` | de organisatie | **Hoe** een onderdeel bij deze organisatie is ingesteld |
| `values/<omgeving>/<naam>.yaml` | de organisatie | Wat per omgeving verschilt, zoals het webadres |
| `branding/` en `values/branding.yaml` | de organisatie | Het logo en de pictogrammen |
| `deploy/<omgeving>/` | de app-manager | Wat er wordt uitgerold. Gegenereerd, nooit met de hand |
| `stack.lock` | de app-manager | Per versie de vastgelegde commit |
| `scripts/create-secrets.sh` | | Maakt de Secrets in het cluster. Secrets staan nooit in Git |
| `.github/workflows/samenstellen.yml` | | Controleert bij elk pull request dat `deploy/` past bij `stacks/` en `values/` |

Een organisatie past alleen `stacks/`, `values/` en `branding/` aan.

De omgevingen zijn `acceptatie` (namespace `govchat-acc`), `productie` (namespace `govchat`)
en `lokaal` voor de eigen laptop. De lijsten in `stacks/` zijn tijdens de pilot ook de
standaardlijst: de kern met de proefapp.

## Een eigen repository maken

Geen fork. Maak één keer een kopie; daarna is er geen koppeling meer.

```sh
git clone https://github.com/GovChat-NL/modulair-deploy-template.git govchat-deploy
cd govchat-deploy
rm -rf .git && git init -b main
git remote add origin https://gitea.example.nl/<organisatie>/govchat-deploy.git
```

Daarna:

1. Vul `values/kern.yaml` in: de gegevens van Entra ID en van het model in Azure.
2. Vul `values/acceptatie/kern.yaml` en `values/productie/kern.yaml` in: het webadres en de toegang van buiten.
3. Zet zo nodig een eigen logo in `branding/` en draai `scripts/make-branding.sh`.
4. Stel samen (zie hieronder), commit en push.
5. Maak de Secrets, per omgeving: `AZURE_MODEL_KEY=... OPENID_CLIENT_SECRET=... scripts/create-secrets.sh govchat-acc`
6. Laat Rancher naar de repository kijken, branch `main`, met de paden `deploy/acceptatie` en `deploy/productie`.

In Entra ID heeft de app-registratie per omgeving de redirect-URI
`https://<webadres>/oauth/openid/callback` nodig, en de app-rollen `govchat-user` en
`govchat-admin`. De rollen moeten in het ID-token staan (claim `roles`).

## Samenstellen

Nodig: [Node.js](https://nodejs.org) 20 of nieuwer, `git`, en de app-manager in de versie
die in `stacks/<omgeving>.yaml` staat.

```sh
git clone --branch v0.1.0 https://github.com/GovChat-NL/modulair-manager.git ../modulair-manager
(cd ../modulair-manager && npm ci)

node ../modulair-manager/manager.js samenstellen acceptatie
git diff          # dit is wat er gaat veranderen
```

De app-manager pusht niets. Commit op een branch, open een pull request en laat een tweede
persoon meekijken. **De merge is de uitrol.**

## Een wijziging: eerst acceptatie, dan productie

1. Pas `stacks/acceptatie.yaml` aan (een nieuwe versie, een app erbij of eraf) en stel samen. Pull request, review, merge.
2. Test op acceptatie.
3. Zet precies dezelfde versie in `stacks/productie.yaml` en stel samen. Een eigen pull request. Die merge is het releasemoment.

Terugdraaien is de vorige versie terugzetten, ook met een pull request.

Een app toevoegen is één regel met een adres en een versie:

```yaml
apps:
  - url: https://github.com/GovChat-NL/modulair-app-time
    version: v1.0.0
```

Heeft de app instellingen, dan komen die in `values/<naam>.yaml`. De app-manager meldt
wat verplicht is en nog ontbreekt.

## Op de eigen laptop

Voor wie een app bouwt of een versie wil proberen, op een eigen wegwerpcluster (Rancher
Desktop, k3d of kind). `values/lokaal/kern.yaml` zet een stubmodel en een lokaal account
in de plaats van Azure en Entra.

```sh
node ../modulair-manager/manager.js samenstellen lokaal
scripts/create-secrets.sh govchat
kubectl -n govchat apply -f .manager-cache/github.com_GovChat-NL_modulair-kern/v0.1.0/local/services.yaml
node ../modulair-manager/manager.js lokaal lokaal        # toont de helm-opdrachten
node ../modulair-manager/manager.js lokaal lokaal | sh   # voert ze uit
kubectl -n govchat port-forward service/librechat-librechat 3080:3080
```

Een eigen app in aanbouw wijs je aan met een map in plaats van een adres:

```yaml
# stacks/lokaal.yaml
apps:
  - url: ../modulair-app-mijnapp
```

## Updaten

Alles wat later verandert, zit in de kern, de apps en de app-manager, en die hebben elk een
eigen versie. Updaten is een versie ophogen in `stacks/`, niet mergen met deze repository.

## Herkomst

De instellingen per organisatie, de branding en de scripts komen van de branch
`swf/modulairKern` van `GovChat-NL/GovChat-NL`, commit `7e2474d` (20 november 2026). De
licentie is nog niet gekozen; het voorstel is EUPL-1.2 en het besluit ligt bij de
maintainers van GovChat-NL.
