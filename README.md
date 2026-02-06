# kestra-orchestrator-sandbox

Ce projet est un environnement de test pour l'orchestrateur [Kestra](https://kestra.io/) utilisant Docker Compose.

## Prérequis

- Installer [Taskfile](https://taskfile.dev/docs/installation) pour le lancement des tâches automatisées
- Installer [Docker](https://docs.docker.com/get-docker/) et [Docker Compose](https://docs.docker.com/compose/install/)


## Commandes utiles (Taskfile)

Toutes les commandes ci-dessous utilisent [Taskfile](https://taskfile.dev/). Remplacez `task <commande>` par la commande souhaitée :

| Commande                | Description                                 |
|-------------------------|---------------------------------------------|
| task docker-compose     | Démarrer Kestra en arrière-plan             |
| task down               | Arrêter les services Kestra                 |
| task down-prune         | Arrêter et supprimer les volumes Kestra     |
| task restart            | Redémarrer les services Kestra              |
| task logs               | Afficher les logs de Kestra                 |
| task ps                 | Lister les conteneurs Kestra                |
| task prune-volumes      | Supprimer tous les volumes non utilisés     |

Exemples :

```sh
# Démarrer Kestra
task docker-compose

# Arrêter Kestra
task down

# Afficher les logs
task logs
```

L'interface Kestra sera accessible sur [http://localhost:8080](http://localhost:8080)

## Volumes et persistance

- Les données Kestra sont stockées dans le volume Docker `kestra_data` (voir docker-compose.yaml)
- Le socket Docker est monté pour permettre l'orchestration de conteneurs
- Le dossier `/tmp` est partagé avec le conteneur

## Arrêt et suppression

Pour arrêter Kestra :

```sh
docker-compose down
```

Pour supprimer les volumes (données persistantes) :

```sh
docker-compose down -v
```

## Liens utiles

- Documentation Kestra : https://kestra.io/docs/
- Issues & support : https://github.com/kestra-io/kestra/discussions
