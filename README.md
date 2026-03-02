# Kibana + Filebeat for GeoNature Docker

## 1. But

Ce dossier fournit une stack locale prête à l'emploi pour:
- collecter les logs Docker de GeoNature avec Filebeat,
- envoyer ces logs dans Elasticsearch,
- les consulter dans Kibana.

Le filtrage est déjà configuré pour le besoin client:
- uniquement messages warning/error (et variantes).

Note:
- le filtrage strict "GeoNature only" par label est documenté plus bas en option,
  car le nom exact du champ label peut varier selon environnement Filebeat.
- le filtrage warning/error est appliqué au niveau input (`include_lines`) pour
  éviter les différences de mapping de champs (`message`, `log.original`, etc).

## 2. Architecture

```text
GeoNature containers (stdout/stderr)
        ->
Docker json logs on host (/var/lib/docker/containers/*/*-json.log)
        ->
Filebeat
        ->
Elasticsearch
        ->
Kibana
```

## 3. Prérequis

- Docker + Docker Compose.
- GeoNature lancé sur la même machine Docker.
- Les services GeoNature doivent avoir les labels:
  - `io.geonature.observability.logs=true`
  - `io.geonature.service=<service>`
- Driver de logs Docker en `json-file` (c'est déjà le cas dans `GeoNature-Docker-services/essential.yml`).
- Version stack Elastic conseillée: `>= 8.15.2` (sinon `add_docker_metadata`
  peut ne pas enrichir les labels avec des daemons Docker récents).

## 4. Démarrage

```bash
cd <path-to-filebeat>/kibana_filebeat
cp -n .env.sample .env
docker compose up -d
```

Vérifier que tout est `healthy`:

```bash
docker compose ps
curl -fsS http://localhost:9200 >/dev/null && echo "elasticsearch ok"
curl -fsS http://localhost:5601/api/status >/dev/null && echo "kibana ok"
docker compose logs --tail 200 filebeat
```

Si Elasticsearch apparaît en `red` sur une machine très chargée disque:

```bash
curl -s 'http://localhost:9200/_cluster/health?pretty'
```

La stack est déjà configurée en dev pour désactiver le `disk threshold` afin
d'éviter ce blocage sur environnement local.

## 5. Vérification de collecte

### 5.1 Test pipeline (rapide)

Génère des logs "warning/error" dans un conteneur Docker avec label GeoNature:

```bash
docker run --rm \
  --label io.geonature.observability.logs=true \
  --label io.geonature.service=onf-log-sim \
  alpine:3.20 sh -c \
  'i=1; while [ "$i" -le 20 ]; do echo "WARNING ONF_SIM idx=$i"; echo "ERROR ONF_SIM idx=$i" >&2; i=$((i+1)); sleep 1; done'
```

Attendu:
- Filebeat ne doit pas afficher d'erreur d'output.
- Des documents arrivent dans Elasticsearch.

Vérification Elasticsearch:

```bash
curl -s 'http://localhost:9200/_cat/indices?v' | rg filebeat
```

### 5.2 Test avec les logs GeoNature

Fais une requête invalide pour générer du log côté app (exemple 404):

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/__probe_geonature_404__
```

Puis vérifie les logs applicatifs:

```bash
cd <path-to-filebeat>/GeoNature-Docker-services
docker compose logs --tail 100 geonature-backend geonature-worker usershub geonature-frontend
```

## 6. Vérification dans Kibana

1. Ouvre `http://localhost:5601`.
2. Va dans `Stack Management -> Data Views`.
3. Crée un Data View: `filebeat-*`.
4. Va dans `Discover`.
5. Filtre par:
   - `docker.container.labels.io_geonature_observability_logs : "true"`  
     ou `docker.container.labels.io.geonature.observability.logs : "true"`
   - `message : *ERROR* OR message : *WARNING*`

Pour segmenter par service:
- `docker.container.labels.io_geonature_service : "geonature-backend"`  
  ou `docker.container.labels.io.geonature.service : "geonature-backend"`

Si les champs labels Docker ne remontent pas dans ton environnement:
1. récupère les IDs des conteneurs GeoNature:
```bash
cd <path-to-filebeat>/GeoNature-Docker-services
docker compose ps -q geonature-backend geonature-worker usershub geonature-frontend
```
2. filtre dans Discover avec `log.file.path`:
```text
log.file.path : *<container_id_1>* OR log.file.path : *<container_id_2>*
```

## 7. Pourquoi les labels peuvent être absents (cause + résolution)

Symptôme:
- `container.labels.*` / `docker.container.labels.*` n'apparaissent pas.
- Les logs arrivent bien dans Elasticsearch, mais sans metadata Docker.

Cause rencontrée ici:
- Filebeat `8.13.4` utilise un client Docker API trop ancien pour ce daemon.
- Message diagnostic (mode debug):
  - `add_docker_metadata: docker environment not detected: Error response from daemon: client version 1.43 is too old. Minimum supported API version is 1.44`

Vérification de la cause:

```bash
cd <path-to-filebeat>/kibana_filebeat
docker compose run --rm filebeat \
  filebeat -e --strict.perms=false \
  -c /usr/share/filebeat/filebeat.yml \
  -d "add_docker_metadata,docker"
```

Résolution:
- Mettre `ELASTIC_STACK_VERSION` à `8.15.2` ou plus récent.
- Redémarrer la stack:

```bash
cd <path-to-filebeat>/kibana_filebeat
docker compose down
docker compose pull
docker compose up -d
```

Vérification après correction:

```bash
curl -s 'http://localhost:9200/filebeat-*/_field_caps?fields=container.labels.*,docker.container.labels.*' | jq
```

## 8. Fichiers fournis

- `docker-compose.yml`: Elasticsearch + Kibana + Filebeat.
- `.env.sample`: versions/ports/mémoire.
- `filebeat/filebeat.yml`: collecte Docker logs + enrichissement metadata + filtres GeoNature.

## 9. Points importants

- Filebeat lit les logs Docker de l'hôte: `/var/lib/docker/containers/...`.
- `docker.sock` est monté en lecture seule pour enrichir les metadata container/labels.
- Le service Filebeat tourne en root dans son conteneur pour lire ces chemins Docker.
- Cette stack est faite pour dev/intégration locale. En prod, réutiliser la même logique Filebeat mais avec votre Elasticsearch/Kibana existants.

## 10. Option: activer le filtre GeoNature strict au niveau ingestion

Par défaut, la config garde warning/error sur tous les conteneurs.
Si tu veux "GeoNature only" directement dans Filebeat, ajoute ce bloc processor dans `filebeat/filebeat.yml`:

```yaml
- drop_event:
    when:
      not:
        or:
          - equals:
              docker.container.labels.io_geonature_observability_logs: "true"
          - equals:
              "docker.container.labels.io.geonature.observability.logs": "true"
          - equals:
              container.labels.io_geonature_observability_logs: "true"
          - equals:
              "container.labels.io.geonature.observability.logs": "true"
```

Puis:

```bash
docker compose restart filebeat
```

Important:
- selon version/config Beats, le champ de label peut être `docker...` ou `container...`.
- vérifier dans Kibana Discover les champs réels avant de durcir le filtre.
- si les labels ne sont pas enrichis, utiliser le filtre `log.file.path` avec IDs
  des conteneurs GeoNature.

## 11. Arrêt

```bash
cd <path-to-filebeat>/kibana_filebeat
docker compose down
```
