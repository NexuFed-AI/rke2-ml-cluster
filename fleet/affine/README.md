# Further instructions

## [Add "unofficial" AI to Affie](https://docs.affine.pro/self-host-affine/administer/ai)
1. Create a new config file `/root/.affine/config/config.json` with the following content:
```json
{
  "$schema": "https://github.com/toeverything/affine/releases/latest/download/config.schema.json",
  "copilot": {
    "enabled": true,
    "providers.openai": {
      "apiKey": "ollama",
      "baseUrl": "http://ollama.ollama.svc.cluster.local:11434/v1"
    }
  }
}
```
    - You can do so by attaching to the running container using VS Code.
2. Import the config file into Affine or restart the container in Rancher (e.g. delete `affine-9f66d596f-m78r8`):
```bash
node --import ./scripts/register.js ./dist/data/index.js import-config /root/.affine/config/config.json
```

## Create Backup
```bash
kubectl create job --from=cronjob/pgvector16-backup pgvector16-backup -n affine
```

## Update Affine
Make changes to the `deployment.yaml` and other files, if necessary, then apply the changes.
use the [`docker-compose.yml`](https://docs.affine.pro/self-host-affine/references/docker-compose-yml) as a reference for the image version and other configurations.
```bash
kubectl apply -f fleet/affine/base/migration-job.yaml -n affine
kubectl rollout restart deployment/affine -n affine
```