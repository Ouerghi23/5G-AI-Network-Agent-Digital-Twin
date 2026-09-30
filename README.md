# 5G AI Network Agent + Digital Twin

Workflow **n8n** qui surveille un réseau 5G RAN, détecte les anomalies, simule des scénarios *what-if* sur un **Digital Twin**, puis demande à un **agent IA** (Gemini) un diagnostic et une recommandation. Un ingénieur valide ou rejette cette recommandation depuis **Telegram**.

> **Le code calcule, l'IA raisonne.** Tous les chiffres (KPI, anomalies, prédictions) sont calculés par du code déterministe. Le LLM ne produit aucune valeur numérique.
## 🔄 Workflow

The complete automation workflow is built in **n8n**:

<p align="center">
  <img src="5G IA Network.png" alt="HireSense AI n8n Workflow" width="100%">
</p>

## Fonctionnement

1. **Acquisition** : toutes les 15 min, le dataset (`data.xlsx`) est téléchargé depuis Google Drive puis converti en JSON.
2. **KPI Snapshot** : le dataset est rejoué dans le temps, par fenêtres de 60 mesures. Chaque mesure est comparée à l'historique : un z-score ≥ 3 signale une anomalie, un écart de plus de 2σ déclenche une alerte. Le réseau est classé `NORMAL`, `WARNING` ou `ANOMALY`.
3. **AI Network Agent** (seulement en `WARNING` ou `ANOMALY`) : l'agent appelle le Digital Twin sur 4 scénarios (trafic +10 %, +20 %, +50 %, capacité PRB −15 %). Il renvoie ensuite une sévérité, un diagnostic et une recommandation au format JSON.
4. **Journal** : chaque décision est ajoutée à Google Sheets.
5. **Validation humaine** : si la sévérité est `HIGH` ou `CRITICAL`, un message Telegram avec les boutons ✅ / ❌ met le workflow en pause jusqu'à la réponse de l'ingénieur.

### Digital Twin (modèle M/M/1)

| Grandeur | Formule |
|---|---|
| Charge | ρ′ = ρ × trafic / capacité |
| Délai | D′ = D × (1 − ρ) / (1 − ρ′) |
| Débit | T′ = T × capacité × (1 − ρ′) / (1 − ρ) |

Risque : `CRITICAL` si ρ′ ≥ 1 · `HIGH` si ρ′ ≥ 0,9 · `MEDIUM` si ρ′ ≥ 0,75 · `LOW` sinon.

## Dataset

Série temporelle O-RAN d'un UE (1 138 mesures) : `Latency` (horodatage en µs), `RRU.PrbTotDl/Ul`, `DRB.UEThpDl/Ul`, `DRB.RlcSduDelayDl`, `DRB.PdcpSduVolumeDL/UL`.

## Installation

1. Dans n8n, faire **Import from File**, puis choisir `workflow.json`.
2. Mettre `data.xlsx` sur Google Drive et créer un Google Sheet vide. Les sélectionner dans les nœuds *Download 5G Dataset* et *Save AI Decision*.
3. Créer les credentials :
   - **Google Drive / Sheets** : *Sign in with Google* ;
   - **Gemini** : une clé gratuite sur [Google AI Studio](https://aistudio.google.com) ;
   - **Telegram** : le token d'un bot créé avec [@BotFather](https://t.me/BotFather).
4. Remplacer `YOUR_TELEGRAM_CHAT_ID` par ton chat ID (obtenu avec [@userinfobot](https://t.me/userinfobot)), puis envoyer `/start` au bot.
5. Tester avec **Execute workflow**, puis activer le workflow.

## Limites

- **Un seul UE :** la détection d'anomalies est temporelle, pas comparative entre UE.
- **PRB :** l'utilisation est normalisée par le maximum observé, faute de connaître la capacité réelle de la cellule.
- **Modèle M/M/1 :** c'est une approximation de premier ordre, qui sert de baseline.
- **Action :** *Execute Action* est simulée ; aucune configuration réseau n'est modifiée.

## Évolutions

- [ ] Dashboard Grafana (PostgreSQL)
- [ ] Modèle ML comparé à la baseline M/M/1
- [ ] Données temps réel (OpenAirInterface + FlexRIC) et actions via xApp O-RAN

## Structure

```
.
├── README.md
├── workflow.json      # workflow n8n (sans credentials)
└── docs/
    └── workflow.png   # capture du workflow
```
