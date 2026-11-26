# Démarrage rapide sous Windows (3 fichiers à double-cliquer, dossier `energitn`)
| Fichier | Quand | Ce qu'il fait |
|---|---|---|
| `lancer.bat` | **la première fois** | crée `.venv`, installe les bibliothèques, crée `backend\.env` s'il manque, crée la base PostgreSQL (`python manage.py creer_base`), `bootstrap`, lance les 19 tests, ouvre le site |
| `demarrer.bat` | les fois suivantes | active `.venv` et lance le site sur http://127.0.0.1:8000/ |
| `verifier.bat` | pour contrôler | liste les fichiers présents/manquants, la base utilisée (doit afficher `postgresql`), relance les tests |

Prérequis : Python 3.12+ et PostgreSQL démarré. `backend\.env` contient `DB_USER=postgres` et `DB_PASSWORD=123` : à adapter si votre mot de passe est différent.
Comptes de démo : `foyer@demo.tn`, `analyste@demo.tn`, `admin@demo.tn` — mot de passe `EnergiTN2026!`.

# ÉnergiTN — plateforme de prévision et de maîtrise de la consommation électrique

Projet unique qui relie la **chaîne ML du guide** (`ml/`, `data/`, `models/`) et l'**application Django** (`backend/`) :
espace foyer, espace analyste énergétique, espace administrateur, connexion et création de compte, en français et en arabe.

```
energitn/
  ml/          scripts du guide (fetch_weather, generate_synthetic, prepare, train_forecast, profiles, anomalies) + notebook
  data/        raw/ · synthetic/ · processed/      (produits par les scripts, non versionnés)
  models/      forecast_h1.joblib · forecast_h7.joblib · *_metrics.json
  backend/     projet Django (config/, api/, web/)
  docs/        datasets.md (registre des sources) · CORRECTIONS.md (fautes corrigées)
```

## Lancer avec PostgreSQL (Windows, PowerShell)
1. Installer PostgreSQL 16 ou 17 (installateur EDB : https://www.postgresql.org/download/windows/), noter le mot de passe de `postgres`.
2. Créer l'utilisateur et la base (SQL Shell « psql », connecté en `postgres`) :

        CREATE USER energitn WITH PASSWORD 'energitn2026' CREATEDB;
        CREATE DATABASE energitn OWNER energitn ENCODING 'UTF8';

   (`CREATEDB` est nécessaire pour `python manage.py test`, qui crée une base de test.)
3. Dans `backend\`, copier `.env.example` en `.env` (mêmes valeurs que ci-dessus, à adapter). Django le lit au démarrage.
4. Puis :

        cd backend
        ..\.venv\Scripts\activate
        pip install -r requirements.txt      # inclut psycopg[binary]
        python manage.py bootstrap           # migrations + import ML + prévisions .joblib + comptes de démo
        python manage.py runserver           # http://127.0.0.1:8000/

Sans fichier `.env` (ou sans `DB_NAME`), Django utilise SQLite (`backend/db.sqlite3`).

Comptes de démonstration (mot de passe **EnergiTN2026!**, boutons directs sur la page de connexion) :
`foyer@demo.tn` · `analyste@demo.tn` · `admin@demo.tn`.

## Comment la partie ML est reliée à Django
| Étape du guide | Côté Django |
|---|---|
| `daily.csv`, `forecast_h*_predictions.csv`, `profiles.csv`, `hourly_profiles.csv`, `anomalies_flagged.csv`, météo, fériés, coupures | `python manage.py import_results` → tables `DailyConsumption`, `Forecast` (backtest), `HouseholdProfile`, `Alert`, `Weather`, `Holiday`, `Outage` |
| `*_metrics.json` | table `ModelRun` (page « Qualité des modèles ») |
| `forecast_h1.joblib`, `forecast_h7.joblib` (modèle **hybride** v2 de l'équipe) | `api/ml.py` reconstruit les mêmes features, ajoute `lin_base` (droite du foyer : a + b·cdd + c·week-end, stockée dans le .joblib) et prévoit les 7 jours suivant le dernier relevé : J+1 avec `forecast_h1`, J+2→J+7 avec `forecast_h7`. Vérifié : même résultat que `ml/inference.py` à 0,0005 kWh près (test automatique). |
| tâche nocturne | `python manage.py run_inference` (cron `0 2 * * *` ou Planificateur de tâches Windows) |
| relancer la chaîne | page **Données et chaîne ML** : exécute les scripts officiels de `ml/` en arrière-plan, journal en direct, puis import + inférence |
| nouveau relevé d'un foyer | détection immédiate (même règle que `anomalies.py` : > 3 sigma robustes et > +30 %) puis recalcul des prévisions |

Météo du jour prévu : table `Weather` (observée, ou prévision CSV importée par l'admin) ; sinon normale de saison de la région (affiché dans l'interface).

## Nouveautés v3 (liaison ML renforcée)
| Fonction | Où | Comment c'est relié au ML |
|---|---|---|
| **Jumeau numérique « Et si une canicule arrivait ? »** | Foyer › Tableau de bord | `GET /api/foyer/scenario/?delta=4` relance les **vrais** modèles J+1/J+7 avec +x °C sur `tmax`, `cdd`, `heatwave` ; affiche kWh, % et facture, et prévient d'un changement de tranche |
| **Prédiction à la demande** (option B du guide) | Admin › Qualité des modèles | `POST /api/ml/predict/` `{"household_id": "H004"}` : Django recharge les `.joblib` et recalcule J+1…J+7 (un foyer ne peut recalculer que le sien) |
| **Santé de la liaison Django ↔ ML** | Admin › Qualité des modèles, `GET /api/ml/health/` | fichiers `.joblib` chargés, même version scikit-learn qu'à l'entraînement, tous les foyers prévus, inférence < 48 h |
| **Surveillance de la dérive** (BF15) | Admin › Qualité des modèles | MAPE J+1 glissante 30 j (jours normaux) comparée à la MAPE de test du `forecast_h1_metrics.json` : Stable / À surveiller / Dérive |
| **Indicateur « Modèles ML connectés »** | en-tête de toutes les pages | vert si `forecast_h1.joblib` et `forecast_h7.joblib` sont présents |

Design : refonte visuelle (barre latérale, fil d'Ariane, cartes, ruban de prévision avec intervalle en moustache, curseurs, connexion), en français, en arabe (RTL) et sur mobile.

## Les trois espaces
- **Foyer** : tableau de bord (ruban de prévision 7 jours coloré par la chaleur, alertes, seuil de tranche), relevés (saisie + import CSV), facture (barème, simulateur d'économies, historique), profil et alertes (comparaison aux foyers du même groupe K-Means), assistant bilingue, rapport mensuel imprimable.
- **Analyste** : demande régionale (jour, heure, scénario de chaleur fondé sur la sensibilité apprise, carte, carte de chaleur heure × région), profils par région, anomalies récurrentes, foyers à surveiller, coupures et canicule, qualité des détecteurs.
- **Administrateur** : utilisateurs et rôles, chaîne ML et registre des fichiers, import de prévision météo, qualité des modèles (baseline contre modèles, erreur par région et par jour, renommage des profils), barème.

## Assistant IA
Il ne reçoit que le **JSON de faits** du foyer (`GET /api/foyer/facts/`) et la base de conseils validée (`api/facts.py`).
- avec `ANTHROPIC_API_KEY` (et optionnellement `LLM_MODEL`) : réponses rédigées par l'IA générative ;
- sans clé : moteur de règles bilingue, chiffré, qui fonctionne hors ligne.
Garde-fous : aide non professionnelle, jamais d'intervention sur l'installation, refus hors périmètre.

## API REST (`/api/`)
`households/`, `forecasts/?kind=future`, `profiles/`, `anomalies/`, `foyer/dashboard/`, `foyer/facts/`, `POST foyer/readings/`,
`operator/overview/`, `operator/profiles/`, `operator/anomalies/`, `models/`, `ml/health/`, `POST ml/predict/`, `foyer/scenario/?delta=4`. Un foyer ne voit que ses données.

## Tests
    python manage.py test api      # 19 tests (SQLite ou PostgreSQL) : import, Django = ml/inference.py, accès par rôle, détection, facture, assistant, admin

## Relancer la chaîne ML à la main (ordre du guide)
    cd ml
    python fetch_weather.py
    python generate_synthetic.py --households 30 --weather ../data/raw/weather.csv --out ../data/synthetic
    python prepare.py
    python train_forecast.py --horizon 1 --weather-error 2     # réglage de l'équipe : météo prévue imparfaite (±2 °C)
    python train_forecast.py --horizon 7 --weather-error 2
    python profiles.py
    python anomalies.py
    cd ../backend && python manage.py import_results && python manage.py run_inference

## Résultats actuels (chaîne ML v2 de l'équipe, test été 2026, jours normaux)
| Modèle | Résultat |
|---|---|
| Prévision J+1 / J+7 (MAPE) | **9,88 % / 9,84 %** contre 27,3 % pour la baseline (gain MAE 63 %) — avec une erreur météo simulée de ±2 °C |
| Couverture de l'intervalle q10–q90 | 78 % (cible 80 %) |
| Anomalies (règles combinées) | F1 0,67 · précision 0,54 · rappel 0,86 |
| Profils K-Means | k = 3 · ARI 0,71 |

## À savoir
- Données **synthétiques**, à présenter comme telles ; capacités régionales en MW fictives (la charge est exprimée en % du record historique).
- Barème par défaut : grille STEG du 1er mai 2022 (presse) — à vérifier auprès de la STEG ; modifiable dans « Barème et référentiel ».
- Production : `DEBUG=0`, `SECRET_KEY`, `ALLOWED_HOSTS`, `python manage.py collectstatic`, HTTPS (activé automatiquement quand `DEBUG=0`).
