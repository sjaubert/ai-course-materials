# 🎓 Guide Pédagogique : Reinforcement Learning avec FlexSim

> **Projet HelloWorld** - Apprendre à une IA à trier des boîtes colorées

---

## 📋 Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Architecture du système](#architecture-du-système)
3. [Le modèle FlexSim](#le-modèle-flexsim)
4. [Les programmes Python](#les-programmes-python)
5. [Le flux de communication](#le-flux-de-communication)
6. [Glossaire](#glossaire)

---

## Vue d'ensemble

### 🎯 Objectif du projet

Entraîner une **Intelligence Artificielle** à prendre des décisions optimales dans une simulation FlexSim. L'IA doit apprendre à envoyer chaque boîte colorée vers le bon processeur.

### 🔄 Avant / Après l'entraînement

| Situation | Comportement | Résultat |
|-----------|--------------|----------|
| **Avant** (Random) | Décisions aléatoires | ~50% dans "Correct" |
| **Après** (Server) | Décisions de l'IA | ~100% dans "Correct" |

### 📁 Fichiers du projet

```
HelloWorld/
├── HelloWorld.fsm          # Modèle FlexSim (simulation)
├── paths.py                # Configuration des chemins
├── flexsim_env.py          # Environnement Gym (traducteur)
├── flexsim_training.py     # Script d'entraînement
├── flexsim_inference.py    # Serveur d'inférence
└── HelloWorld.zip          # Agent IA entraîné (généré)
```

---

## Architecture du système

```
┌─────────────────────────────────────────────────────────────────┐
│                        ENTRAÎNEMENT                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐    Socket TCP    ┌──────────────────┐        │
│   │   FlexSim    │◄────────────────►│  flexsim_env.py  │        │
│   │ (Simulation) │   Port 5005      │  (Environnement) │        │
│   └──────────────┘                  └────────┬─────────┘        │
│                                              │                   │
│                                              ▼                   │
│                                     ┌──────────────────┐        │
│                                     │flexsim_training  │        │
│                                     │   (Algorithme    │        │
│                                     │      PPO)        │        │
│                                     └────────┬─────────┘        │
│                                              │                   │
│                                              ▼                   │
│                                     ┌──────────────────┐        │
│                                     │ HelloWorld.zip   │        │
│                                     │  (Agent IA)      │        │
│                                     └──────────────────┘        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         INFÉRENCE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐    HTTP GET      ┌──────────────────┐        │
│   │   FlexSim    │─────────────────►│flexsim_inference │        │
│   │ (Simulation) │   Port 8000      │   (Serveur IA)   │        │
│   │              │◄─────────────────│                  │        │
│   │ ActionMode:  │     Action       │ HelloWorld.zip   │        │
│   │   Server     │                  │   (Agent IA)     │        │
│   └──────────────┘                  └──────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Le modèle FlexSim

### 🏭 Structure du modèle

```
Source1 → Queue1 → [Processor1 / Processor2 / Processor3] → [Correct / Incorrect]
```

| Objet | Rôle |
|-------|------|
| **Source1** | Génère des boîtes de 3 couleurs (🔴 rouge, 🟢 vert, 🔵 bleu) |
| **Queue1** | File d'attente qui décide où envoyer chaque boîte |
| **Processor1** | Traite les boîtes rouges 🔴 |
| **Processor2** | Traite les boîtes vertes 🟢 |
| **Processor3** | Traite les boîtes bleues 🔵 |
| **Correct** | Destination si bonne association couleur-processeur |
| **Incorrect** | Destination si mauvaise association |

### 🧠 L'outil ReinforcementLearning

L'objet **ReinforcementLearning** dans FlexSim gère :

- La communication avec Python (socket ou HTTP)
- L'envoi des **observations** (couleur de la boîte)
- La réception des **actions** (numéro du port)
- Le calcul des **récompenses** (correct = +1, incorrect = 0)

### 📤 Code de Queue1 (Send To Port)

```flexscript
Object item = param(1);           // La boîte qui sort
Object current = ownerobject(c);  // Queue1
return Model.parameters.NextPort; // Le port choisi par l'IA
```

Le paramètre `NextPort` est défini par :

- **Mode Random** : valeur aléatoire (1, 2 ou 3)
- **Mode Server** : valeur retournée par le serveur Python

---

## Les programmes Python

### 📄 `paths.py` - Configuration

```python
flexsim = "C:/Program Files/FlexSim 2025/program/flexsim.exe"
model = "HelloWorld.fsm"
tensorboard = "tensorboard"
agent = "HelloWorld.zip"
```

Centralise tous les chemins pour faciliter la portabilité.

---

### 📄 `flexsim_env.py` - L'Environnement Gym

**Rôle** : Faire le pont entre FlexSim et l'algorithme d'apprentissage.

C'est un **environnement Gymnasium** (anciennement OpenAI Gym) qui standardise la communication.

#### Structure principale

```python
class FlexSimEnv(gymnasium.Env):
    
    def reset(self):
        """Réinitialise la simulation et retourne l'état initial"""
        
    def step(self, action):
        """Exécute une action et retourne (état, récompense, terminé)"""
        
    def _launch_flexsim(self):
        """Lance FlexSim avec les arguments de connexion socket"""
        
    def _take_action(self, action):
        """Envoie l'action à FlexSim via socket"""
        
    def _get_observation(self):
        """Reçoit l'observation de FlexSim"""
```

#### Communication Socket (Port 5005)

```
Python                          FlexSim
   |                               |
   |-------- ActionSpace? -------->|
   |<------- Discrete(3) ---------|  (3 actions possibles)
   |                               |
   |-------- Reset? -------------->|
   |<------- {state, reward} -----|
   |                               |
   |-------- TakeAction:1 -------->|
   |<------- {state, reward} -----|
   |                               |
```

---

### 📄 `flexsim_training.py` - L'Entraînement

**Rôle** : Entraîner l'IA avec l'algorithme PPO (Proximal Policy Optimization).

```python
# Création de l'environnement
env = FlexSimEnv(
    flexsimPath = paths.flexsim,
    modelPath = paths.model,
    visible = False  # Mode invisible pour accélérer
)

# Création du modèle PPO
model = PPO("MlpPolicy", env, verbose=1, tensorboard_log=paths.tensorboard)

# Entraînement (100 000 étapes)
model.learn(total_timesteps=100000)

# Sauvegarde de l'agent
model.save(paths.agent)  # → HelloWorld.zip
```

#### Que fait l'algorithme PPO ?

1. **Exploration** : Essaie des actions aléatoires au début
2. **Observation** : Note les récompenses obtenues
3. **Apprentissage** : Ajuste sa stratégie pour maximiser les récompenses
4. **Convergence** : Trouve la politique optimale (couleur → bon port)

#### Métriques d'entraînement

| Métrique | Signification |
|----------|---------------|
| `ep_rew_mean` | Récompense moyenne par épisode (doit augmenter) |
| `ep_len_mean` | Durée moyenne d'un épisode |
| `total_timesteps` | Nombre d'étapes effectuées |
| `fps` | Vitesse d'entraînement |

---

### 📄 `flexsim_inference.py` - Le Serveur d'Inférence

**Rôle** : Serveur HTTP qui utilise l'agent entraîné pour répondre aux requêtes de FlexSim.

```python
class FlexSimInferenceServer(BaseHTTPRequestHandler):
    
    def _handle_reply(self, params):
        # Récupère l'observation (couleur de la boîte)
        observation = params['observation']
        
        # Demande à l'agent la meilleure action
        action, _ = FlexSimInferenceServer.model.predict(observation)
        
        # Renvoie l'action à FlexSim
        self.wfile.write(json.dumps(action))
```

#### Communication HTTP (Port 8000)

```
FlexSim                                    Serveur Python
   |                                            |
   |-- GET /?observation=2 ------------------->|
   |                                            |
   |   (observation=2 signifie boîte bleue)     |
   |                                            |
   |<-------- Réponse: 3 ----------------------|
   |                                            |
   |   (action=3 signifie Processor3)           |
```

---

## Le flux de communication

### 🔄 Pendant l'entraînement

```
1. Python lance FlexSim avec -training localhost:5005
2. FlexSim se connecte au socket Python
3. Boucle d'entraînement :
   a. FlexSim envoie l'état (couleur boîte)
   b. Python choisit une action (selon PPO)
   c. Python envoie l'action à FlexSim
   d. FlexSim exécute et calcule la récompense
   e. FlexSim envoie (nouvel état, récompense, terminé?)
   f. Python met à jour ses poids
4. Après 100 000 étapes : sauvegarde de l'agent
```

### 🎯 Pendant l'inférence

```
1. Python lance le serveur HTTP sur port 8000
2. L'utilisateur ouvre FlexSim avec ActionMode = Server
3. À chaque boîte dans Queue1 :
   a. FlexSim fait une requête HTTP avec l'observation
   b. Le serveur Python charge l'observation
   c. L'agent prédit la meilleure action
   d. Le serveur renvoie l'action
   e. FlexSim route la boîte vers le bon processeur
```

---

## Glossaire

| Terme | Définition |
|-------|------------|
| **Reinforcement Learning (RL)** | Méthode d'apprentissage où un agent apprend en recevant des récompenses |
| **Agent** | L'IA qui prend des décisions |
| **Environnement** | Le système dans lequel l'agent évolue (ici FlexSim) |
| **État (State)** | Information sur la situation actuelle (couleur de la boîte) |
| **Action** | Décision prise par l'agent (numéro du port) |
| **Récompense (Reward)** | Feedback positif ou négatif après une action |
| **Épisode** | Une simulation complète du début à la fin |
| **PPO** | Proximal Policy Optimization - algorithme d'entraînement moderne |
| **Gymnasium** | Bibliothèque Python standard pour les environnements RL |
| **stable-baselines3** | Bibliothèque Python d'algorithmes RL (dont PPO) |

---

## 🚀 Pour aller plus loin

1. **Modifier les récompenses** : Changer la logique de récompense dans FlexSim
2. **Ajouter des capteurs** : Plus d'observations pour des décisions plus complexes
3. **Changer l'algorithme** : Essayer DQN, A2C au lieu de PPO
4. **Augmenter la complexité** : Plus de couleurs, plus de processeurs

---

*Document généré le 08/02/2026 - Projet HelloWorld FlexSim + Python RL*
