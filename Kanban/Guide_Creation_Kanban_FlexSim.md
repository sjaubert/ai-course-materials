# 🏭 Guide Pas-à-Pas : Créer un Modèle Kanban dans FlexSim

## Phase 1 : Créer la structure de base (Flux produits)

### Étape 1.1 : Nouveau modèle

1. Ouvrir FlexSim
2. `File → New Model`
3. Sauvegarder : `Kanban_Simple.fsm`

---

### Étape 1.2 : Placer les objets (de gauche à droite)

Glissez depuis la **Library** les objets suivants :

| # | Objet à placer | Nom à donner | Position X |
|---|----------------|--------------|------------|
| 1 | **Source** | `StockMatiere` | 0 |
| 2 | **Queue** | `Buffer0` | 3 |
| 3 | **Processor** | `Poste1` | 6 |
| 4 | **Queue** | `Buffer1` | 9 |
| 5 | **Processor** | `Poste2` | 12 |
| 6 | **Queue** | `Buffer2` | 15 |
| 7 | **Processor** | `Poste3` | 18 |
| 8 | **Queue** | `StockPF` | 21 |
| 9 | **Sink** | `Client` | 24 |

> **Astuce** : Double-clic sur chaque objet → onglet General → changer le nom

---

### Étape 1.3 : Connecter les objets

1. Touche **A** pour activer le mode connexion
2. Cliquez sur `StockMatiere` puis sur `Buffer0`
3. Continuez : `Buffer0 → Poste1 → Buffer1 → Poste2 → Buffer2 → Poste3 → StockPF → Client`

Vous devriez avoir une ligne continue de gauche à droite.

---

### Étape 1.4 : Configurer les objets

**StockMatiere (Source) :**

- Arrival Style : `Arrival Schedule`
- Edit Arrival Schedule :
  - Time : `0`
  - Quantity : `20`
- Repeat Schedule : `Unchecked` (ou décoché)

**Poste1 (Processor) :**

- Process Time : `60` secondes

**Poste2 (Processor) :**

- Process Time : `55` secondes

**Poste3 (Processor) :**

- Process Time : `50` secondes

**Client (Sink) :**

- *(pas de config spéciale)*

---

### Étape 1.5 : Premier test

1. Cliquer sur **Reset** puis **Run**
2. Vérifier que les pièces circulent de gauche à droite
3. **Stop** quand validé

---

## Phase 2 : Ajouter les Plannings Kanban

### Étape 2.1 : Placer les plannings visuels

Ajoutez 3 **Queues** en dessous de la ligne principale :

| Nom | Position X | Position Y |
|-----|------------|------------|
| `PlanningK1` | 6 | -3 |
| `PlanningK2` | 12 | -3 |
| `PlanningK3` | 18 | -3 |

---

### Étape 2.2 : Configurer l'apparence des Plannings

Pour que les Queues (files d'attente) ressemblent à des tableaux Kanban :

1. Cliquez sur chaque PlanningK.
2. Dans **Quick Properties** (à droite), modifiez la taille (Size) pour former un "tableau" :
   - `X=2.0`, `Y=0.5`, `Z=0.1`
3. Dans la section **Queue** -> **Item Placement** :
   - Choisissez `Stack Horizontally` ou `Stack Into Line` (pour aligner les cartes côte à côte).

**Couleurs de fond (optionnel) :**

- PlanningK1 : Bleu clair
- PlanningK2 : Jaune clair
- PlanningK3 : Rouge clair

*(Note : La forme et la couleur des cartes elles-mêmes seront définies par le script à l'étape 3.2)*

---

## Phase 3 : Logique Kanban (Scripts)

### Étape 3.1 : Créer un Global Table pour les cartes

1. Menu `Tools → Global Tables`
2. Créer : `CarteKanban`
3. Colonnes : `ID, Poste, EnCirculation`

*(Optionnel - on peut aussi gérer via labels)*

---

### Étape 3.2 : Script OnReset (Initialiser les cartes via le Modèle)

Ce script doit s'exécuter globalement quand on appuie sur le bouton **Reset** de la simulation. Il ne se configure pas sur un objet 3D spécifique (comme une machine), mais directement sur le modèle.

1. Allez dans l'onglet **Toolbox** (à gauche, à côté de Library).
2. Cliquez sur le **+** (vert) > **Modeling Logic** > **Model Trigger**.
3. Dans la fenêtre qui s'ouvre, cliquez sur le **+** (vert) à côté de *Triggers*.
4. Choisissez **On Reset**.
5. Cliquez sur l'icône de code (parchemin) à droite de la ligne *On Reset* pour ouvrir l'éditeur de code.
6. Copiez-collez le script suivant :

```flexscript
// Créer 5 cartes Kanban par poste au démarrage
treenode boxClass = Model.find("Tools/FlowItemBin/Box"); // Récupérer la référence à l'objet Box du modèle

if (!boxClass) {
    // Si pas trouvé, essayer la librairie standard (Plan B)
    boxClass = library().find("?Box");
}

if (!boxClass) {
    msg("Erreur", "Impossible de trouver l'objet 'Box' dans le modèle (FlowItemBin) ou la Library.");
    return 0;
}

for (int poste = 1; poste <= 3; poste++) {
    Object planning;
    if (poste == 1) planning = Model.find("PlanningK1");
    if (poste == 2) planning = Model.find("PlanningK2");
    if (poste == 3) planning = Model.find("PlanningK3");
    
    if (!planning) {
        msg("Erreur", "PlanningK" + string.fromNum(poste) + " non trouvé.");
        continue;
    }
    
    for (int i = 1; i <= 5; i++) {
        // createinstance(Class, Container)
        Object carte = createinstance(boxClass, planning);
        
        if (carte) {
            // Configuration des labels (via commande setlabel, plus robuste)
            setlabel(carte, "Poste", poste);
            setlabel(carte, "NumCarte", i);
            
            // Taille et Couleur
            carte.size = [0.3, 0.3, 0.05];
            
            if (poste == 1) carte.color = Color.blue;
            if (poste == 2) carte.color = Color.yellow;
            if (poste == 3) carte.color = Color.red;
        }
    }
}
```

---

### Étape 3.3 : Logique de production (Flux tiré)

**Poste3 - Trigger "OnEntry" :**

```flexscript
Object item = param(1);
Object current = ownerobject(c);
Object planning = Model.find("PlanningK3");

// Vérifier si une carte Kanban est disponible
// (Syntaxe 2024 : accès aux subnodes)
if (planning.subnodes.length == 0) {
    // Pas de carte = pas de production, renvoyer la pièce au buffer
    return 0;  // Bloquer l'entrée
}

// Prendre une carte du planning (la dernière entrée)
Object carte = planning.subnodes[planning.subnodes.length];
moveobject(carte, current);  // Attacher la carte à la pièce
```

*(Note : Au moment de la sortie (OnExit), la carte reste simplement attachée à la pièce et part avec elle vers le StockPF.)*

---

### Étape 3.4 : Libération des cartes (Client)

**Client (Sink) - Trigger "OnEntry" :**

```flexscript
Object item = param(1);
treenode boxClass = Model.find("Tools/FlowItemBin/Box");

if (!boxClass) boxClass = library().find("?Box"); // Plan B
if (!boxClass) return 0; // Sécurité

// Récupérer la carte attachée et la renvoyer au planning
// Ici on simplifie en recréant une carte virtuelle pour l'exercice

Object planning3 = Model.find("PlanningK3");
if (planning3) {
    Object carte = createinstance(boxClass, planning3);
    if (carte) {
        setlabel(carte, "Poste", 3);
        carte.size = [0.3, 0.3, 0.05];
        carte.color = Color.red;
    }
}

// Signaler le besoin en amont (cascade Kanban)
// Envoyer une carte au PlanningK2
Object planning2 = Model.find("PlanningK2");
if (planning2) {
    Object carte2 = createinstance(boxClass, planning2);
    if (carte2) {
        carte2.color = Color.yellow;
        carte2.size = [0.3, 0.3, 0.05];
    }
}
```

---

## Phase 4 : Test et Validation

### Checklist de test

- [ ] Les cartes apparaissent sur les plannings au Reset
- [ ] La production ne démarre que s'il y a des cartes
- [ ] Les cartes circulent avec les produits
- [ ] Les cartes reviennent au planning après consommation
- [ ] Le système s'auto-régule (pas de surproduction)

---

## Prochaines étapes

Une fois ce modèle de base fonctionnel, on pourra ajouter :

- [ ] Visualisation Dashboard avec graphiques
- [ ] Calcul du nombre optimal de cartes
- [ ] Simulation de pannes et variabilité
- [ ] Comparaison avec flux poussé

---

**Prêt à commencer ? Ouvrez FlexSim et suivez la Phase 1 !**
