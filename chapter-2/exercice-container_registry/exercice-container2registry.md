# Exercice — Containers → Registry

## 🎯 Objectifs pédagogiques (alignés “cloud-native”)

À la fin, l’étudiant sait :
- construire une image Docker à partir d’un Dockerfile
- publier une image dans un registry (Docker Hub)
- comprendre **immutabilité** + **versionnement** (v1 → v2)
- comprendre pourquoi le cloud “sépare” compute et stockage

## Partie 0 — Pré-requis techniques
- Docker Desktop installé et démarré
- Le projet `notes-api` (Dockerfile déjà fourni dans le cours)
- Créer un compte Docker Hub (gratuit) : https://hub.docker.com (il faudra se souvenirs de son **username Docker Hub**)


## Partie 1 — Publier dans Docker Hub

### Étape 1.1 Construire une image locale

À la racine du dossier API (là où se trouve le Dockerfile) :

```bash
docker build -t notes-api .
```

#### Observations
```bash
docker images | grep notes-api
```
> Attendu : une image `notes-api` apparaît (tag `latest` par défaut).

#### Question de réflexion
> Pourquoi construire une image locale ne suffit pas si l'on souhaite exécuter des containeurs dans le cloud ?

### Étape 1.2 — Login
```bash
docker login
```
#### Question de réflexion 
> Pourquoi faire la commande `docker login` ? 

### Étape 1.3 — Tag

**Rapel :**
Format Docker Hub :
```
<username>/<image>:<tag>
```

```bash
docker tag notes-api <username>/notes-api:v1
```

#### Question de réflexion 
> Quelles différences y a-t-il entre `docker tag` et `docker build` ?

### Étape 1.4 — Push
```bash
docker push <username>/notes-api:v1
```

#### Question de réflexion 
> Que ce qui se passe réellement avec un `docker push` ?

#### Observations
- Sur Docker Hub : le repository `<username>/notes-api` existe
- Tag `v1` visible

### Parite 1.5 — Tester la version v1 avec Docker Compose

Dans le `docker-compose.yml`, remplacer l’image locale/API par :
```yaml
image: <username>/notes-api:v1
```

Puis démarrer l’application avec `docker compose up` (ou en arrière-plan :`docker compose up -d`)

Créer une note :

```bash
curl -X POST http://localhost:3000/notes \
  -H "Content-Type: application/json" \
  -d '{"title":"Ma note v1","content":"contenu"}'
```

Essayer ensuite de la modifier :
```bash
curl -X PUT http://localhost:3000/notes/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"Note modifiée","content":"nouveau contenu"}'
```

#### Observations

Le comportement ne correspond pas à celui attendu :
* soit la route `PUT /notes/:id` n’existe pas
* soit elle ne se comporte pas comme attendu

## Partie 2 — Immuabilité (raison de versionner)

### Contexte
La version `v1` est publiée.
Le client demande une évolution :
- ajouter `PUT /notes/:id` (remplacement complet)
- retourner `404` si la note n’existe pas
- retourner `400` si la validation échoue

### Étape 2.1 — Implémenter un changement
- Implémenter les changements demandés par le client.

```js
// =======================
// Helpers
// =======================

function isNonEmptyString(v) {
  return typeof v === "string" && v.trim().length > 0;
}

function isStringOrUndefined(v) {
  return v === undefined || typeof v === "string";
}

function parseId(req, res) {
  const id = Number(req.params.id);
  if (!Number.isInteger(id) || id <= 0) {
    res.status(400).json({ error: "Invalid id. Expected a positive integer." });
    return null;
  }
  return id;
}
...

// =======================
// CRUD NOTES
// =======================

// GET /notes
...

// POST /notes
...

// PUT /notes/:id
app.put("/notes/:id", async (req, res) => {
  const id = parseId(req, res);
  if (id === null) return;

  const { title, content } = req.body;

  if (!isNonEmptyString(title)) {
    return res.status(400).json({
      error: "title is required and must be a non-empty string",
    });
  }

  if (!isStringOrUndefined(content)) {
    return res.status(400).json({
      error: "content must be a string if provided",
    });
  }

  const result = await pool.query(
    `
    UPDATE notes
    SET title = $1,
        content = $2
    WHERE id = $3
    RETURNING *
    `,
    [title.trim(), content ?? "", id]
  );

  if (result.rows.length === 0) {
    return res.status(404).json({ error: "note not found" });
  }

  res.json(result.rows[0]);
});

// GET /notes/:id
...


// DELETE /notes/:id
...
```

#### Questions de réflexion
> Peut-on modifier directement l’image `v1` déjà déployée ?

### Étape 2.2 — Build + Tag + Push
```bash
docker build -t notes-api .
docker tag notes-api <username>/notes-api:v2
docker push <username>/notes-api:v2
```

#### ⚠️ Piège à éviter 
Azure Container Apps (et plus globalement le cloud) attend en pratique une image compatible Linux serveur, très souvent : `linux/amd64`

Pour inspecter le manifest :
```bash
docker buildx imagetools inspect <image_name>:<version>
```

Si `Platform: linux/amd64` ne figure pas, alors il faudra lancer :
```bash
docker buildx build --platform linux/amd64 -t <image_name>:<version> --push .
```

1. builder l’image pour `linux/amd64`
2. la pousser directement sur Docker Hub


#### Observations
* Observer que seules les layers nécéssitant une mise à jour ont été pushed.
* Sur Docker Hub, vous devez voir **v1** et **v2**.

### Étape 2.3 — Tester la version v2 avec Docker Compose

Remplacer :
```yaml
image: <username>/notes-api:v1
```

par :
```yaml
image: <username>/notes-api:v2
```

Puis relancer : `docker compose up -d` (il est possible de devoir forcer la recréation :`docker compose up -d --force-recreate`)

Tester : 

* une modification valide
```bash
curl -X PUT http://localhost:3000/notes/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"Titre mis à jour","content":"contenu mis à jour"}'
```

* une note inexistante 
```bash
curl -X PUT http://localhost:3000/notes/9999 \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","content":"Test"}'
```

* une validation invalide 
```bash
curl -X PUT http://localhost:3000/notes/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"","content":"Test"}'
```

#### Observations attendues

* la route `PUT /notes/:id` existe
* un `404` est retourné si la note n’existe pas
* un `400` est retourné si la validation échoue

### Messages clé

> Changer de version d’image change réellement le comportement applicatif, sans modifier le runtime Docker ni l’environnement.
> Une image est **immuable** : on ne la met pas à jour, on la **remplace** par une nouvelle version.

#### Question de réflexion 
> Pourquoi la solution suivante n’est-elle pas valable dans le Cloud ?
