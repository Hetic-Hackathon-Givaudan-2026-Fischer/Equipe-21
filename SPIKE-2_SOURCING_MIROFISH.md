# SPIKE-2 · Sourcing des données et inputs MiroFish


## 0. En bref

| Critère d'acceptation | Où c'est traité | Statut |
|---|---|---|
| Sources identifiées, priorisées, justifiées | §1 | ✅ |
| Faisabilité de collecte évaluée par source | §1 | ✅ 9 sources, dont 7 collectées automatiquement **en production** |
| Contraintes légales et éthiques du scraping | §2 | ✅ |
| Formats et volumes MiroFish documentés **et vérifiés** | §3 | ✅ Vérifiés dans le code source officiel |
| Pipeline de raffinage décrit étape par étape | §5 | ✅ En production sur Supabase |
| Échantillon prêt à l'emploi pour les spikes suivants | §4, dossier `data/sample/` | ✅ Régénéré chaque nuit |
| Limites et risques explicités | §7 | ✅ |

---

## 1. Sources de données, par priorité

| Priorité | Source | Signal | Mode de collecte | Faisabilité et fréquence | Qualité | Contraintes légales | Statut réel |
|---|---|---|---|---|---|---|---|
| **P0** | **Google Trends** | Intention de recherche, sur 5 ans | Bibliothèque `pytrends` (non officielle), une fois par semaine ; import CSV manuel en secours ; API officielle en alpha sur candidature | Haute, mais Google peut limiter (erreurs 429) | Élevée (agrégée, 0-100 par requête) | Données agrégées, aucune donnée personnelle. **Accès non officiel : zone grise**, voir §2 | ✅ Automatique |
| **P0** | **Open Food Facts** | Mise en marché réelle (produits, marques, pays) | Fichiers officiels « delta » quotidiens + export complet | Très haute, chaque jour, monde entier | Bonne ; quelques erreurs de catégorie ; `added_at` = date d'ajout dans la base | Licence ODbL (attribution) | ✅ Automatique |
| **P0** | **Wikipédia** (vues) | Curiosité du grand public | API REST Wikimedia | Très haute, chaque jour | Élevée ; **trafic global en baisse (≈ -24 % sur un an)**, donc on compare en relatif | Libre | ✅ Automatique |
| **P1** | **Google News + 11 flux RSS** | Couverture médiatique et industrie | Flux RSS | Haute, toutes les 4 h | Bonne (titres et résumés) | On stocke titre, résumé et lien seulement ; usage de Google News validé | ✅ Automatique |
| **P1** | **YouTube** | Contenus consommateurs (recettes, tests), engagement | API officielle (clé gratuite) | Haute ; quota de 89 recherches par jour, donc chaque arôme est couvert une fois tous les 3 jours | Bonne | Auteurs pseudonymisés | ✅ Automatique |
| **P1** | **Bluesky** | Conversations consommateurs | API officielle (mot de passe d'application) | Haute, chaque jour | Moyenne : bruit sur les mots courants | Auteurs pseudonymisés ; ni lien ni pseudo en couche propre | ✅ Automatique |
| **P2** | **GDELT** | Presse mondiale | API gratuite | ⚠️ Bloque vite les adresses IP ; requêtes groupées par 10 arômes | Bonne | Libre | ⚠️ Bloqué par GDELT |
| **P2** | **Pinterest Trends** | Signal précoce (recettes, desserts) | **Veille manuelle** hebdomadaire (CSV) ; API réservée aux partenaires | Manuelle, 15 min par semaine | Bonne mais partielle | **Pas de scraping** (interdit par les CGU) | 🟡 Prête, veille à démarrer |
| **Enrichissement** | **EFSA / Commission européenne** (arômes autorisés, Novel Food) | Faisabilité réglementaire | Téléchargements ponctuels | Mises à jour rares | Très élevée (officielle) | Libre | ⏳ Prévu (non collecté) |
| **NO-GO** | **Reddit** | Discussions | API soumise à approbation, réservée au non commercial, en fermeture progressive | — | — | Partenariat d'entreprise = usage potentiellement commercial | ❌ |
| **NO-GO** | **TikTok, Instagram, X** | Viralité | API réservées aux chercheurs, ou payantes ; scraping interdit | — | — | CGU, accès frauduleux à un système (STAD), RGPD | ❌ |

**Ce qui est déjà collecté** (Supabase, au 11/11) :
- Wikipédia : 206 955 jours de vues ;
- Open Food Facts : 20 985 produits ;
- Bluesky : 6 349 posts ;
- Google News et RSS : environ 2 000 articles ;
- YouTube : 197 vidéos ;
- Google Trends : environ 30 000 points hebdomadaires sur 5 ans, à compléter cette nuit.

---

## 2. Fiche d'évaluation du scraping

**Règle appliquée :** open data > API officielle > veille manuelle > scraping. Nous n'utilisons **aucun scraping**.

| Plateforme | Scraping ? | Raison | Alternative retenue |
|---|---|---|---|
| TikTok, Instagram | ❌ | CGU, murs de connexion (risque d'accès frauduleux à un STAD), données personnelles | YouTube (API officielle), Bluesky |
| X | ❌ | API payante, CGU | Bluesky |
| Reddit | ❌ | API sur approbation et en fermeture ; scraping contraire aux CGU | Bluesky, YouTube |
| Pinterest | ❌ | CGU ; API réservée aux partenaires | Veille manuelle (CSV) |
| Google Trends | ⚠️ **Zone grise** | `pytrends` utilise l'interface interne de Google, non publique | Volume faible (1 fois par semaine, 42 requêtes espacées de 10 s), arrêt automatique en cas de refus, import manuel en secours, candidature à l'API officielle. **Avant une mise en production commerciale : API officielle ou service payant** |

**Cadre juridique :**
- **RGPD** : un post public reste une donnée personnelle. Nos mesures :
  - les auteurs sont pseudonymisés (HMAC-SHA256 avec un secret) ;
  - les posts de particuliers n'ont ni lien ni pseudo en couche propre ;
  - aucun texte de particulier n'entre dans les fiches MiroFish ;
  - la couche brute est supprimée 3 jours après le nettoyage.
- **Droit des bases de données** (droit *sui generis*) : nous ne copions pas de base commerciale. Open Food Facts est sous ODbL, avec attribution.
- **Fouille de textes** (Art. L.122-5-3 CPI) : uniquement sur des contenus accessibles légalement, en respectant les refus exprimés (CGU, robots.txt).
- **Bonnes pratiques techniques** : User-Agent avec un contact, respect des quotas, reprises espacées, arrêt automatique en cas de refus (GDELT, Google Trends).

---

## 3. Inputs MiroFish : spécification vérifiée

### 3.1 Ce que dit le code officiel (github.com/666ghj/MiroFish)

| Point | Valeur vérifiée | Source |
|---|---|---|
| Formats acceptés | `pdf`, `md`, `txt`, `markdown` | `backend/app/config.py` : `ALLOWED_EXTENSIONS` ; `file_parser.py` |
| Taille maximale de fichier | 50 Mo (`MAX_CONTENT_LENGTH`) | `config.py` (application non vérifiée dans la route d'envoi) |
| Découpage du texte | Morceaux de 500 caractères, chevauchement de 50 | `file_parser.py` |
| LLM | Toute API au format OpenAI : `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL_NAME` | `config.py`, README |
| Mémoire / graphe | **Zep Cloud obligatoire** (`ZEP_API_KEY`) ; un Zep installé en local est refusé | `config.py` : `validate()` |
| Nombre de tours | 10 par défaut (`OASIS_DEFAULT_MAX_ROUNDS`) ; le README conseille moins de 40 tours (coût élevé) | `config.py`, README |

**Ce qui n'est PAS une limite officielle :** les volumes « 1 500 à 4 000 mots optimal, 10 000 mots maximum » de la première version ne figurent nulle part dans MiroFish. Ci-dessous, ce sont **nos recommandations**.

**Conséquences pour le projet :**
- avec la version officielle, **les fiches partent sur Zep Cloud et chez le fournisseur du LLM**, donc **aucune donnée confidentielle de Givaudan** dans les fiches ;
- un LLM local (Ollama) est **possible en théorie** grâce au format OpenAI, mais ce n'est ni documenté ni testé ;
- les petits modèles disponibles sur nos PC (0,8 à 1,5 milliard de paramètres) se sont révélés trop faibles lors de nos tests.

### 3.2 Format de nos fiches (recommandation de l'équipe)

Une fiche Markdown par arôme : `data/sample/mirofish_packets/<arome>.md`, avec 5 sections.

1. **Signaux quantitatifs** : Google Trends (13 semaines, vs l'an dernier, volatilité), Wikipédia relatif, produits Open Food Facts (nombre, marques, pays), mentions dans la presse et les réseaux.
2. **Résultat statistique V1** : statut, probabilité, fiabilité du modèle.
3. **Preuves** : titres réels d'articles et de vidéos, avec la date.
4. **Points de vigilance** : **calculés à partir des données** (volatilité, pic, saisonnalité, peu de produits, part de sponsorisé, peu de mentions). Ils équilibrent la fiche et limitent le biais d'enthousiasme des agents LLM.
5. **Question de simulation** : 4 profils de consommateurs, horizon de 12 mois.

### 3.3 Contrôles qualité avant injection (automatiques : `check_packet()`)

| Contrôle | Règle |
|---|---|
| Taille | 1 500 à 15 000 caractères |
| Structure | Les 5 sections sont présentes |
| RGPD | Aucun « @ », aucun lien vers un compte personnel |
| Équilibre | Au moins un point de vigilance |
| Traçabilité | Zone, période et date des données indiquées |

**Résultat : 102 fiches sur 102 passent les contrôles**, de 2 200 à 3 200 caractères (voir `mirofish_packets/manifest.json`).

### 3.4 Qui fait quoi

| Étape | Responsable |
|---|---|
| Collecte, nettoyage, indicateurs, modèle V1 | **Data** (pipeline automatique) |
| Génération et contrôle des fiches | **Data** (`flavortrends spike2-build`, chaque nuit) |
| Simulation | **Data / IA** avec MiroFish : clé LLM + clé Zep Cloud à obtenir. **Pas encore exécutée** |
| Affichage | **Tech**, via l'API (§8) |

---

## 4. Échantillon pour le POC (`data/sample/`)

| Fichier | Contenu | Utilisation |
|---|---|---|
| `reference_cases.csv` | **82 cas réels** (25 arômes) : un arôme en vraie hausse à une date passée, ses variables mesurées avant cette date, et son étiquette | Modèle V1 ; se rejoue dans **Orange** (*File → Logistic Regression → Test and Score*) |
| `model_metrics.json` | Validation complète du modèle V1 | Rapport, jury |
| `model_v1.json` | Paramètres du modèle (sans pickle, lisibles) | API `/simulate` |
| `trends_scored.json` | État actuel des **102 arômes** : statut, scores, indicateurs, courbe sur 104 semaines, preuves, points de vigilance | Données de l'API |
| `mirofish_packets/*.md` + `manifest.json` | 102 fiches MiroFish et leurs contrôles | Seed des simulations |
| `orange/` (non versionné) | Exports pour Orange, dont des textes de posts | Analyse en équipe |

**Régénération :** `flavortrends spike2-build`. C'est aussi fait automatiquement chaque nuit, après la collecte.

---

## 5. Pipeline de récupération et de raffinage (en production)

```
1. SOURCES        Google Trends · Open Food Facts · Wikipédia · Google News · RSS · YouTube · Bluesky
                  (+ Pinterest manuel, GDELT bloqué)
2. COLLECTE       Python, tâches planifiées (03:00, presse toutes les 4 h) : quotas, reprises, incrémental,
                  verrous avec battement de cœur
3. STOCKAGE BRUT  Supabase, schéma raw : réponse API telle quelle (JSONB), une ligne par version,
                  purge après 3 jours
4. NETTOYAGE      Schéma core : texte, dates UTC, langue, pseudonymisation, sponsorisé,
                  doublons exacts et quasi-doublons (MinHash)
5. INDICATEURS    Croissances (13 sem., vs an dernier), volatilité, pic, Wikipédia relatif,
                  produits par arôme, mentions
6. MODÈLE V1      Régression logistique sur la forme de la courbe, validée « un arôme à la fois »
7. FICHES         Markdown par arôme + contrôles qualité  ──►  MiroFish (V2, à exécuter)
8. API            FastAPI : /api/v1/trends, /trends/{id}, /trends/{id}/packet, /model-metrics, /simulate
9. DASHBOARD      Équipe Tech (supermarché 3D)
```

---

## 6. Modèle V1 : méthode et résultats réels

**Étiquetage, par une règle écrite et sans avis humain :**
- **Un cas** = un arôme en **vraie hausse** à une date passée T0 : au moins +20 % sur 13 semaines, **et** au moins +20 % par rapport à la même période un an avant, ce qui écarte les hausses simplement saisonnières.
- **Pérenne** si, un an plus tard (même saison), l'arôme **conserve au moins la moitié de son gain**.
- **Buzz** sinon.
- Les variables sont mesurées **uniquement avec les données antérieures à T0** (pas de fuite d'information). Une date T0 est testée toutes les 4 semaines sur 5 ans d'historique.

**Variables du modèle** (la forme de la courbe) :
- le pic (pic récent / moyenne) ;
- la volatilité ;
- l'accélération ;
- la croissance sur 13 semaines.

Elles ont été choisies parmi 10 variantes : plus de variables faisaient apprendre du bruit avec seulement 25 arômes.

**Validation :** « leave-one-flavor-out ». On retire un arôme entier, on entraîne sur les autres, on prédit l'arôme retiré.

| | Accuracy | Accuracy équilibrée | Buzz détectés | ROC AUC |
|---|---|---|---|---|
| **Modèle V1** | **82 %** | **77 %** | **68 %** | **0,81** |
| Toujours répondre « pérenne » | 77 % | 50 % | 0 % | — |
| Règle simple « gros pic = buzz » | 73 % | 73 % | 74 % | — |

**Matrice de confusion** (82 cas) :

| | Prédit pérenne | Prédit buzz |
|---|---|---|
| **Vrai pérenne** | 54 | 9 |
| **Vrai buzz** | 6 | 13 |

**Lecture des résultats :**
- **L'objectif métier est de repérer les buzz, pour ne pas lancer de R&D dessus.** Toujours répondre « pérenne » donne une bonne accuracy mais ne détecte **aucun** buzz. Le modèle en détecte 68 %, avec la meilleure accuracy équilibrée.
- Les coefficients sont cohérents : **un pic marqué** (-0,83) et **une montée très brutale** (-0,59) poussent vers « buzz ».

**Limites, à dire au jury :**
- **Petit échantillon** : 82 cas, 25 arômes. Les vraies hausses non saisonnières sont rares sur des mots génériques.
- **Score probablement un peu optimiste** : les variables ont été choisies sur ces mêmes cas. Il faudra le reconfirmer sur les mois à venir.
- **Les seuils** (+20 %, moitié du gain) **sont des choix** : ils sont documentés et modifiables (`spike2.py`).
- **Données encore incomplètes** : la collecte Google Trends sera complétée cette nuit (zone France, arômes de niche). Les chiffres seront recalculés automatiquement.

**Aujourd'hui :** un seul arôme est en vraie hausse (hors saison) dans le monde, le **curry** (+27 % sur 13 semaines, +56 % par rapport à l'an dernier). Sa courbe présente un pic très marqué (3,8 fois la moyenne), et le modèle le classe **« probablement passager »**. Les autres sont stables, en baisse, ou en hausse saisonnière.

---

## 7. Risques : qualité, légalité, pérennité

| Risque | Niveau | Parade |
|---|---|---|
| Google bloque `pytrends` (non officiel) | Moyen | Arrêt automatique, reprise la nuit suivante, import CSV manuel, candidature à l'API officielle |
| GDELT bloque notre adresse IP | Moyen | Requêtes groupées par 10, arrêt automatique ; données redondantes avec Google News et RSS |
| Fermeture ou durcissement d'API (Reddit, TikTok…) | Élevé sur le long terme | Aucune dépendance à une seule source ; triangulation de 7 sources |
| Espace Supabase gratuit (500 Mo, environ 40 % utilisés) | Moyen | Purge et compactage, alertes à 80 % et 95 % ; **offre Pro à décider avant janvier** |
| Biais des sources (Wikipédia en baisse, internautes jeunes et urbains, échantillon Open Food Facts) | Moyen | Indicateurs relatifs, triangulation, indice de confiance affiché |
| Fiches MiroFish envoyées à Zep Cloud et au fournisseur LLM | Moyen | Aucune donnée confidentielle dans les fiches ; version locale à étudier |
| Biais des agents LLM (enthousiasme, effet de groupe) | Élevé | Points de vigilance obligatoires dans chaque fiche ; la simulation (V2) **n'entre jamais seule dans un score** |
| Petit nombre de cas de référence | Moyen | Validation par arôme, comparaison à des références, recalcul chaque nuit |
| RGPD (posts de particuliers) | Faible (maîtrisé) | Pseudonymisation, pas de pseudo ni de lien, purge |

---

## 8. Pour l'équipe Tech (Arnaud)

**Lancer l'API :**
```bash
flavortrends serve          # puis http://localhost:8000/docs (Swagger)
ngrok http 8000             # pour la partager
```

| Route | Rôle |
|---|---|
| `GET /api/v1/trends?category=Boissons&status=hausse` | Liste des 102 arômes (dashboard, supermarché 3D) |
| `GET /api/v1/trends/{trend_id}` | Fiche complète, courbe de 104 semaines comprise (rapport détaillé) |
| `GET /api/v1/trends/{trend_id}/packet` | Fiche Markdown MiroFish |
| `GET /api/v1/model-metrics` | Validation du modèle V1 |
| `POST /api/v1/simulate` | Probabilité V1 pour des variables données |
| `GET /api/v1/health` | Date des données |

**Différences avec le contrat de la première version :**
- `trend_id` est l'identifiant de l'arôme (`matcha`, `yuzu`), et non `TR_003` ;
- `v2_mirofish_perennity_score` vaut **`null`** tant qu'aucune simulation n'a été exécutée ;
- `efsa_regulatory_status` vaut `"non_verifie"` ;
- `v1_statistical_baseline` vaut `null` pour un arôme qui n'est pas en vraie hausse (le modèle ne s'applique qu'aux hausses).

Le front doit afficher ces valeurs `null` comme « non disponible ».

Exemple réel (extrait, matcha, 11/11/2026) :
```json
{
  "trend_id": "matcha", "aroma_name": "Matcha", "category_3d_aisle": "Boissons", "status_label": "Stable",
  "scores": {"v1_statistical_baseline": null, "v2_mirofish_perennity_score": null, "confidence_level": "haute"},
  "metrics": {"search_growth_13w_pct": -11.0, "search_growth_52w_pct": 1.0, "search_volatility_26w": 0.126,
              "openfoodfacts_products_count": 138, "openfoodfacts_brands": 85, "openfoodfacts_countries": 37,
              "press_mentions_30d": 32, "social_mentions_30d": 101, "efsa_regulatory_status": "non_verifie"},
  "evidence": [{"source": "google_news", "date": "2026-11-11",
                "title": "Everyone Is Chasing Matcha. Japan Is Already Moving On to Hojicha."}],
  "mirofish": {"status": "fiche_prete", "packet": "data/sample/mirofish_packets/matcha.md", "simulation": null},
  "data_origin": "reel"
}
```
