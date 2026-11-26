# ÉnergiTN

**Prévision et maîtrise de la consommation électrique des foyers tunisiens**

## À propos
ÉnergiTN prévoit la consommation de chaque foyer sur **7 jours**, détecte les **journées anormales** et estime la **facture STEG**. L'interface est disponible en français et en arabe.
L'application Django est reliée à une chaîne de Machine Learning : prévision hybride, profils K-Means et détection d'anomalies.

| Espace | Fonctions principales |
|---|---|
| **Foyer** | prévision 7 jours, alertes, facture, simulateur de canicule, assistant |
| **Analyste** | demande régionale, profils de consommation, anomalies |
| **Admin** | utilisateurs, chaîne ML, qualité des modèles, barème |

## Installation (Windows)
**Prérequis :** [Python 3.12+](https://www.python.org/downloads/) · [PostgreSQL 17](https://www.postgresql.org/download/windows/) · [Git](https://git-scm.com/download/win)

```bat
git clone https://github.com/<pseudo>/energitn.git
cd energitn
lancer.bat
```
`lancer.bat` installe tout, demande votre mot de passe PostgreSQL, crée la base, entraîne les modèles, lance les tests et ouvre http://127.0.0.1:8000/.
Ensuite : `demarrer.bat` pour lancer le site, `verifier.bat` pour contrôler l'installation.

Guide détaillé et résolution des problèmes : [docs/INSTALLATION.md](docs/INSTALLATION.md)

## Connexion
| Rôle | E-mail | Mot de passe |
|---|---|---|
| Foyer | `foyer@demo.tn` | `EnergiTN2026!` |
| Analyste | `analyste@demo.tn` | `EnergiTN2026!` |
| Admin | `admin@demo.tn` | `EnergiTN2026!` |

## Résultats
| Modèle | Résultat |
|---|---|
| Prévision J+1 / J+7 | MAPE **9,88 % / 9,84 %** (baseline 27,3 %) |
| Détection d'anomalies | F1 **0,67** |
| Profils K-Means | k = 3 · ARI **0,71** |

*Les consommations sont synthétiques. La météo est réelle (Open-Meteo).*

## Structure
```
ml/        chaîne Machine Learning
backend/   application Django (api, web, config)
docs/      guide d'installation, captures, fichiers de test
```

## Tests
```bat
cd backend
python manage.py test api
```

## Équipe
*Nom Prénom* — Django · *Nom Prénom* — Machine Learning
