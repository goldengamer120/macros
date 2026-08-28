# Suivi Macros

Suivi nutritionnel dans **un seul fichier HTML**. Pas de build, pas de npm, pas de serveur : tu copies `index.html` sur GitHub Pages et c'est en ligne.

- Journal jour par jour, recherche tolérante aux fautes de frappe
- Base de 1000 aliments (26 catégories) + tes propres aliments et recettes
- 5 modes d'ajout : recherche, photo, **scanner de code-barres**, voix, manuel
- Statistiques semaine / mois / année, objectifs historisés par date
- Mode sombre, 5 couleurs d'accent
- Sync multi-appareils via Firebase (optionnelle)

Pensé pour être ajouté à l'écran d'accueil d'un iPhone.

---

## 1. Mettre en ligne (5 minutes)

1. Crée un dépôt GitHub **public** (GitHub Pages est gratuit sur les dépôts publics).
2. Copie `index.html` dedans, commit, push.
3. *Settings → Pages → Source: Deploy from a branch → `main` / `root`*.
4. Ton app est sur `https://TON-PSEUDO.github.io/NOM-DU-DEPOT` d'ici ~2 min.

À ce stade **tout fonctionne déjà** sauf la sync et l'IA : les données sont stockées en local dans le navigateur (`localStorage`). Si ça te suffit, tu peux t'arrêter ici.

---

## 2. ⚠️ Remplace mes clés par les tiennes

Le fichier contient **mes** identifiants. Ils ne marcheront pas chez toi, et les laisser est une mauvaise idée pour nous deux. Cherche ces 4 endroits dans `index.html` :

| Cherche | Remplace par |
|---|---|
| `const firebaseConfig = {` | ta config Firebase (étape 3) |
| `window._GEMINI_ALLOWED_EMAILS = [` | ton adresse Google |
| `const GEMINI_KEY=` | ta clé Gemini (étape 4) |
| `<meta name="theme-color"` | facultatif, la couleur de la barre du navigateur |

> **Pourquoi c'est important.** Une clé API dans un fichier HTML statique est **visible par tout le monde** — c'est inévitable sans serveur. Le garde-fou en place est `_GEMINI_ALLOWED_EMAILS` : seules les adresses listées peuvent déclencher les fonctions Photo/Voix.
>
> Ce n'est pas théorique : sur une version précédente, un ami a utilisé le lien avec la clé active et **Google a suspendu tout le projet Cloud** (Firebase et Gemini compris). D'où la liste blanche. Ne partage jamais ton lien avec la clé active sans ajouter l'adresse de la personne — et préviens-la que le quota gratuit (~20 requêtes/jour) est partagé.

---

## 3. Firebase — sync entre appareils (optionnel mais recommandé)

Sans ça, chaque appareil a ses propres données, sans partage.

1. [console.firebase.google.com](https://console.firebase.google.com) → **Ajouter un projet** (le plan gratuit Spark suffit largement).
2. **Authentication → Sign-in method → Google → Activer.**
3. **Firestore Database → Créer une base** (mode production).
4. ⚙️ **Paramètres du projet → Tes applications → Web (`</>`)** → copie l'objet `firebaseConfig` et colle-le dans `index.html`.
5. **Authentication → Settings → Authorized domains** → ajoute `TON-PSEUDO.github.io`, sinon la connexion Google est refusée.

**Règles de sécurité Firestore** — onglet *Rules*, remplace tout par ceci :

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

Sans cette règle, n'importe qui peut lire et écrire dans ta base. Chacun ne voit que ses propres données.

Les données sont écrites dans `users/{uid}/data/state`, sous forme d'une chaîne JSON dans le champ `payload`.

---

## 4. Gemini — analyse photo et vocale (optionnel)

Permet de photographier une assiette ou de dicter un repas, et d'en extraire les macros.

1. [aistudio.google.com/apikey](https://aistudio.google.com/apikey) → **Create API key**.
2. Colle-la dans `const GEMINI_KEY=`.
3. Mets ton adresse Google dans `_GEMINI_ALLOWED_EMAILS`.

Le modèle est appelé via l'alias `gemini-flash-latest`. **Ne le remplace pas par un nom figé** (`gemini-2.5-flash` etc.) : Google déprécie ses versions régulièrement et l'app casserait sans prévenir.

Le scanner de code-barres, lui, utilise [Open Food Facts](https://world.openfoodfacts.org) — gratuit, sans clé, rien à configurer.

---

## 5. Modifier le code

Tout est dans `index.html` : le CSS dans `<style>`, le JS dans le dernier `<script>`, la base d'aliments dans `const BASE_DB=[...]`.

**Valider avant de commit.** Il n'y a aucun test, et une erreur de syntaxe casse toute l'app *silencieusement* : la page s'affiche, mais plus aucun bouton ne répond.

La seule vérification fiable est de **compiler chaque bloc `<script>` séparément**. Ouvre la page dans un navigateur, puis colle ceci dans la console (F12) :

```js
fetch(location.pathname+'?v='+Date.now()).then(r=>r.text()).then(h=>{
  const b=[...h.matchAll(/<script(?![^>]*src)(?![^>]*type="module")[^>]*>([\s\S]*?)<\/script>/g)];
  console.table(b.map((m,i)=>{try{new Function(m[1]);return{bloc:i,ok:'OK'}}
                              catch(e){return{bloc:i,ok:'ERREUR',detail:e.message}}}));
});
```

`new Function()` compile sans exécuter : tu vois l'erreur exacte, sans effet de bord.

> **N'utilise pas un simple comptage d'accolades.** Ça paraît suffisant, ça ne l'est pas : le compte porte sur la concaténation de tous les blocs `<script>` et ignore les chaînes de caractères. Une apostrophe non échappée dans `'d'historique'` ferme la chaîne, casse tout le script — et le compteur reste au vert. C'est arrivé.

Recharge ensuite la page avec un paramètre anti-cache (`?v=2`) et vérifie la console : zéro erreur. Le cache du navigateur sert très volontiers l'ancienne version et fait croire qu'un correctif n'a rien changé.

### Pièges déjà rencontrés

- **N'injecte jamais un nom d'aliment ou de catégorie dans un `onclick` en ligne.** Une apostrophe casse le handler sans lever d'erreur — le filtre « McDonald's » n'a rien fait pendant des semaines à cause de ça. Utilise `data-*` + `addEventListener`.
- **La couleur d'accent est une variable CSS**, pas un hex : `var(--accent)`, `--accent-text-on`, `--accent-hover`, `--accent-soft-bg`, `--accent-soft-border`. Un remplacement de hex à l'aveugle a déjà créé une référence circulaire dans `:root`.
- **Le fichier est en CRLF.** Un script d'édition en masse doit lire/écrire avec `newline=''` et re-normaliser, sinon git voit tout le fichier comme modifié.
- **Vérifie le contraste en mode clair *et* sombre.** Assombrir un gris pour le mode clair l'a déjà rendu illisible en mode sombre.
- Les valeurs nutritionnelles de `BASE_DB` sont **pour 100 g**, sauf si `dw` (poids par défaut) et `unit` sont présents. Confondre les deux fausse les calculs d'un facteur 5 ou 10 — c'est arrivé sur les doses de sirop.

---

## 6. Brancher les calories sur autre chose (optionnel)

Firestore rend les données lisibles depuis n'importe quel langage. Exemple en Python avec `firebase-admin` et une clé de service :

```python
snap = db.collection("users").document(uid).collection("data").document("state").get()
data = json.loads(snap.to_dict()["payload"])["data"]
# { "2026-08-05": [ {"nm":"Flocons d'avoine","w":80,"cal":311,"p":11,"l":6,"g":47}, ... ] }
kcal = sum(f["cal"] for f in data["2026-08-05"])
```

`nm` nom · `w` grammes · `cal` kcal · `p`/`l`/`g` protéines/lipides/glucides en g.

Reste **en lecture seule** depuis l'extérieur : l'app fusionne son état local avec Firestore, une écriture concurrente peut corrompre le journal.

---

## Licence

Fais-en ce que tu veux. Aucune garantie — vérifie tes propres valeurs nutritionnelles avant de t'appuyer dessus.
