# Brief Tactique du Moteur de Match — Des Livres au Code

> **Document de référence tactique pour les développeurs et agents de la ruche.**
> Fonde les grandeurs, les seuils et la dynamique du moteur de match de `FootballEcosystemLifeSim` sur les principes tactiques éprouvés de la littérature de référence (*Comment regarder un match de foot*, *Comment gagner un match de foot*, doctrine *Beau Jeu* et canon *Football Manager 2024*).
> **Règle méthodologique absolue** : aucune affirmation abstraite. Chaque concept tactique est traduit en une **grandeur physique mesurable** (mètres, secondes, ratios, fréquences), confronté au code existant, et assorti de sa **procédure de réfutation par la mesure**.

---

## 1. Ce qu'il faut faire d'abord — Les cinq chantiers prioritaires

Classés par **rapport entre effet sur le réalisme perçu et coût d'implémentation** (du plus rentable au plus lourd). Ce sont les cinq leviers qui transformeront immédiatement la simulation sans réécrire le moteur.

```
+---------------------------------------------------------------------------------------------------+
| 1. OFFRE DE PASSE PROACTIVE (Levier A Beau Jeu)  --> Réveille l'offre courte & cibles dynamiques|
| 2. DÉVERROUILLAGE DES COMBINAISONS (Levier B)     --> 1-2, troisième homme, fin du jeu stérile     |
| 3. RÈGLE « BALLON COUVERT / DÉCOUVERT » (Bloc)   --> Recul-frein vs step-up (Sacchi / Gourcuff)   |
| 4. RACCORDEMENT UNIFIÉ MENTALITÉ & DOCTRINE TIR   --> Du pont vers le moteur local (canonique)     |
| 5. ERREUR DE PASSE MULTI-ATTRIBUTS (Levier C)     --> Réduction du badpass (77% des pertes)        |
+---------------------------------------------------------------------------------------------------+
```

### Chantier 1 — Offre de passe proactive (Levier A du Cahier du Beau Jeu) & Réveil de `offTheBall`
- **Le problème dans le code** : Dans `src/modules/matchEngine/decision/offBall/base.ts:40`, la position de base d'un coéquipier est une simple interpolation passive vers le ballon : `anchor + alpha * (ball - anchor)`. Bien que l'attribut `offTheBall` soit exploité ponctuellement dans les courses de rupture ou l'encombrement de surface (`positionalStructure.ts`, `depthHeir.ts`, `boxSupport.ts`), il est **strictement absent de l'offre de passe en soutien direct du porteur** dans `base.ts:40`. Aucun coéquipier ne se déplace pour ouvrir un angle de passe dans l'intervalle entre deux défenseurs face au porteur en possession médiane. Résultat : le porteur score ses options (`passOptions.ts`) sur des piquets immobiles, produisant une circulation lente et individuelle.
- **L'action** : Insérer une couche `applyPassOffer` dans le pipeline off-ball (`src/modules/matchEngine/decision/offBall/`), bornée par `BJ_MAX_OFFERS = 2`, pilotée par `offTheBall` et `anticipation`. Le coéquipier évalue le cône d'ombre du défenseur le plus proche et applique un delta de 1,5 à 3,5 m vers la ligne de passe la plus nette, sous contrainte d'alignement `applyOnsideClamp`.
- **Rapport Effet / Coût** : **10 / 10**. Coût modéré (une couche off-ball pure, aucun changement dans le moteur physique), effet maximal : il crée les récepteurs dynamiques sans lesquels aucune combinaison n'est possible.
- **Mesure de validation** : `% assisted` des tirs passant de 44-60 % à ~75 % (standard IRL/Opta) ; taux de passes progressives maintenu sans gonfler les tirs.

### Chantier 2 — Déverrouillage et enchaînement des combinaisons (Levier B du Beau Jeu) : une-deux et troisième homme
- **Le problème dans le code** : L'audit du *Cahier du Beau Jeu* a mesuré **0 combinaison collective réelle sur 8 matchs** ! Le module `src/modules/matchEngine/tactics/patterns/thirdManRun.ts` existait mais était asphyxié par deux verrous : le seuil `MIN_P_SUCCESS_AB` (qui comparait un produit de contrôles à une probabilité de complétion) et un verrouillage rigide à 1 seul bid actif par équipe (`THIRD_MAN_TTL_TICKS = 35`), avec un décalage causal de 1 tick entre l'intention du porteur et la course du receveur.
- **L'action** : Lisser la rampe de décision dans `decideCarrier.ts:proposePasses`, exposer la cible top-k du porteur dans le state du tick courant pour éliminer le délai de réaction de 100 ms du coureur, et permettre l'enchaînement remise en 1 touche (`actionCooldown <= 0.2s` pour le pivot B) vers le coureur lancé C.
- **Rapport Effet / Coût** : **9.5 / 10**. Le code est déjà à 80 % écrit dans `thirdManRun.ts` et `centralPenetration.ts`. Il s'agit de débrider un flux étranglé.
- **Mesure de validation** : Nombre d'occurrences réelles de combinaisons à trois ou une-deux réussies (`scheduledRun.kind === 'third_man'` arrivant à terme) passant de 0 à **8–15 par match**.

### Chantier 3 — La règle cardinale du bloc défensif : « Ballon couvert / Ballon découvert »
- **Le problème dans le code** : Dans `src/modules/matchEngine/decision/offBall/defensiveLine.ts:70-84`, la hauteur de la ligne défensive recule uniquement en fonction de l'abscisse brute du ballon (`ballNorm < DEF_LINE_TAPER_KNEE_NORM`). Le fichier orphelin `src/modules/matchEngine/tactics/contextualPressing.ts:dynamicLineHeight` calculait même un offset sur l'axe Y au lieu de l'axe X ! La défense ne sait pas si le porteur adverse est sous pression ou libre de tout marquage.
- **L'action** : Conditionner le comportement de la ligne défensive à la pression effective sur le porteur adverse (`pressureOn(state, oppCarrier)`) :
  - **Ballon couvert** (porteur adverse cadré à `< 2m`, dos au but ou enfermé) : la ligne remonte d'un pas (`step-out` de +2 à +4 m) pour compacter l'interligne avec les milieux et mettre les attaquants adverses en position de hors-jeu.
  - **Ballon découvert** (porteur adverse libre à `> 3m`, face au jeu, tête levée) : la ligne enclenche immédiatement un recul-frein préventif (`drop` de -3 à -6 m) pour couvrir la profondeur.
- **Rapport Effet / Coût** : **9 / 10**. Faible coût de calcul, élimine les buts absurdes sur balles par-dessus concédés pendant que les défenseurs montent passivement.
- **Mesure de validation** : Nombre de ballons en profondeur concédés dans le dos de la défense sans contestation chutant de > 60 % ; 2 à 4 hors-jeux adverses provoqués par match en défense médiane/haute.

### Chantier 4 — Raccordement unifié Mentalité & Doctrine de tir (FM24) : Du Pont vers le Moteur Local
- **Le problème dans le code** :
  Le commentaire de `src/modules/matchEngine/coachOrders.ts:51` déclarait : « ÉCARTÉS (non patchables) : mentality + shotDoctrine (déclarés, ZÉRO lecteur moteur) ». Ce commentaire est un témoignage trompeur et incomplet. En réalité :
  - `mentality` **EST bien lue et consommée** par le pont vers le moteur de référence dans `src/modules/matchEngine/traducteurTactique.ts:308` (`MENTALITE_VERS_LEUR`), puis transmise à leur simulation vivante via `src/modules/matchEngine/injecterDansLeurMatch.ts`. Elle fonctionne donc chez EUX, mais elle est **totalement ignorée par notre propre boucle locale de décision** (`decideCarrier.ts`, `lambda.ts`, `defensiveLine.ts`). Le champ `tactics/types.ts:162` définit d'ailleurs déjà `mentalityScalar: number`, mais il est actuellement orphelin et figé à `0.5` dans `gameStateContext.ts:256` sans jamais dériver de `tactic.mentality`.
  - `shotDoctrine` est documentée dans `src/modules/matchEngine/cognition/intent/technique/shotStyle.ts:13` et spécifiée dans `archive/cahiers-realises/Cahier_ShotStyle_PassStyle.md:270` comme une « gate de fréquence dans `shotDecision.ts`, PAS le geste », mais son branchement effectif dans `shotDecision.ts` n'a pas été réalisé.
  - **Le piège architectural à éviter absolument** : Si un agent implémenteur prend au pied de la lettre le commentaire « zéro lecteur » de `coachOrders.ts:51`, il va créer un second lecteur indépendant dans notre moteur local, avec son propre mapping ad-hoc divergent de `traducteurTactique.ts`. On fabriquerait alors **deux vérités concurrentes** sur la même mentalité d'équipe (le défaut récurrent de split-brain déjà combattu sur la masse salariale et les blessures).
- **L'action et où doit vivre le lecteur canonique** :
  1. **Lecteur canonique de la Mentalité** : Il doit vivre dans le résolveur de contexte unifié `src/modules/matchEngine/tactics/gameStateContext.ts`. Il dérive `mentalityScalar` ($0..1$, centré sur 0.5) en appliquant à `tactic.mentality` la table de conversion canonique déjà posée dans `traducteurTactique.ts:MENTALITE_VERS_LEUR` (`very_defensive: 0.1, defensive: 0.3, balanced: 0.5, attacking: 0.7, very_attacking: 0.9`). Les consommateurs de notre moteur local (`lambda.ts:riskExponent`, `defensiveLine.ts:computeDefensiveLineOffsetM`, `decideCarrier.ts`) lisent UNIQUEMENT ce `mentalityScalar` canonique.
  2. **Lecteur canonique de la Doctrine de tir** : Il doit vivre dans `src/modules/matchEngine/decision/onBall/shotDecision.ts` (en tant que gate de fréquence sur le seuil `minShootXG` et modulation du tirage d'EV), conformément aux spécifications du cahier de réalisation.
- **Rapport Effet / Coût** : **9 / 10**. Coût très faible, unifie les deux moteurs (pont et local) sur une seule et même source de vérité et honore les consignes maîtresses de FM24.
- **Mesure de validation** : Test d'isolation A/B : en « Très défensif » (`mentalityScalar = 0.1`) + « Travailler le ballon dans la surface », l'équipe produit 6–9 tirs/match avec un xG médian de 0.13 ; en « Très offensif » (`mentalityScalar = 0.9`) + « Tirer à vue », elle produit 16–22 tirs/match avec un xG médian de 0.06.

### Chantier 5 — Erreur de passe multi-attributs (Levier C du Beau Jeu)
- **Le problème dans le code** : 76,8 % des ballons perdus (`loose ball`) sont causés par des mauvaises passes (`badpass`), contre seulement 12 % par des duels ou interceptions. La fonction `passErr` dans `src/modules/matchEngine/actions/` génère un déchet uniforme sans distinction qualitative suffisante entre un passeur d'élite sous pression et un joueur limité.
- **L'action** : Moduler l'amplitude de déviation et la dispersion de la trajectoire de passe en introduisant les attributs réels : `composure` (atténue la dégradation sous pression de 0 à 40 %), `technique` (réduit l'angle d'erreur balistique de 0,5° à 4°), `vision` (détection des lignes) et `decisions`.
- **Rapport Effet / Coût** : **8 / 10**. Assainit la distribution des pertes de balle sans toucher à la cascade de choix du porteur.
- **Mesure de validation** : Décrue de la part de `badpass` dans les ballons perdus de 77 % vers 50–55 % ; hausse proportionnelle des pertes sur duels et interceptions actives (12 % -> 30-35 %) ; différenciation nette du taux de passe réussie par division (Ligue 1 élite : 83–86 % vs National : 72–76 %).

---

## 2. Le Modèle : Dimensions tactiques, Grandeurs et Mesures réfutables

Chaque dimension tactique est définie par sa **grandeur physique**, son **ordre de grandeur dans le football réel** et sa **métrique de validation dans le moteur**.

```
+---------------------------------------------------------------------------------------------------------------------------------+
| Dimension                  | Grandeur & Unité           | Valeur Réelle / Cible FM24       | Mesure de Contrôle Moteur          |
+----------------------------+----------------------------+----------------------------------+------------------------------------+
| 1. Compacité Verticale     | Distance en mètres (m)     | 25 m à 32 m (médian), 15-20m bas | defensiveBlockShape.compactness    |
| 2. Hauteur de Ligne Déf.   | Abscisse X (m sur 105m)    | Bas: 20-28m / Méd: 35-45m / H: 50| median(players.CB.x)               |
| 3. Déclenchement Pressing  | Rayon (m) & Délai (ms)     | Rayon: 1.5 - 3.0 m, Réact: <250ms| pressureOn(carrier) à réception   |
| 4. Contre-Pressing (Gegen) | Temps post-perte (s)       | Fenêtre: 3.0 s à 5.0 s           | % regain dans les 5s post-perte    |
| 5. Sortie de Balle (6m)    | Écartement (m)             | Centraux: 38-46m, Latéraux: 58-64m| distance(CB1.y, CB2.y) sur 6m      |
| 6. Combinaison Troisième H.| Durée de séquence (s)      | Cycle complet: 1.5 s à 2.5 s     | scheduledRun.completionRate        |
| 7. Rest Defense (Sécurité) | Effectif & Écartement (m)  | 4 à 5 joueurs, 18-25m en retrait | count(opp_half_defenders)          |
| 8. Découpage Largeur (5 C.)| Bandes Y (m sur 68m)       | 5 couloirs de 13.6 m chacun      | grid20Zone channel occupancy       |
| 9. Rythme & Tempo          | Durée par possession (s)   | Court: 0.9-1.4s, Direct: 1.8-2.6s| median(timeCarrierHoldingBall)     |
| 10. Sélection du Tir       | Seuil d'Attente xG         | 0.05 à 0.09 hors box, 0.12+ box  | shotsPerMatch & median(shot.xG)    |
+---------------------------------------------------------------------------------------------------------------------------------+
```

### 2.1 Compacité d'équipe & Écartement interligne
- **Grandeur physique** : Distance longitudinale entre le joueur le plus reculé du champ (hors gardien) et le joueur le plus avancé : $D_{\text{comp}} = x_{\text{max}} - x_{\text{min}}$ (en mètres). Distance interligne défensive $\Delta x_{\text{inter}} = \bar{x}_{\text{mid}} - \bar{x}_{\text{def}}$.
- **Ordre de grandeur réel** :
  - Bloc médian : **25 à 32 m** de profondeur totale (Arrigo Sacchi impose 25 m maximum ; Marcelo Bielsa : « pas plus de 25 mètres de l'attaque à la défense »).
  - Bloc bas : **15 à 22 m** de profondeur (Christian Gourcuff : interligne de 6 à 8 m).
  - Largeur du bloc défensif : **36 à 44 m** sur les 68 m de la largeur du terrain (Christian Gourcuff : le bloc est étroit, concédant volontairement les couloirs extérieurs).
- **Mesure de contrôle dans le moteur** :
  - Fonction `computeDefensiveBlockShape(team)` dans `src/modules/matchEngine/team/defensiveBlockShape.ts`.
  - Condition de réfutation : $D_{\text{comp}} > 38\,\text{m}$ pendant plus de 3 secondes consécutives en phase de bloc placé (`RECOVERY` ou `TRANS_OFF_DEF`), ou $\sigma_x(\text{défenseurs}) > 3.0\,\text{m}$ (rupture d'alignement de la ligne).

### 2.2 Hauteur du bloc et Règle « Ballon couvert / Ballon découvert »
- **Grandeur physique** : Position médiane $X_{\text{backline}}$ des défenseurs centraux (repère $x \in [0, 105]$) et vecteur vitesse de recul $\vec{v}_{\text{drop}}$ en fonction de l'état binaire du porteur adverse :
  $$\text{État} = \begin{cases} \text{Couvert} & \text{si } d(\text{porteur}, \text{presseur}) \le 2.0\,\text{m} \text{ ou porteur dos au jeu} \\ \text{Découvert} & \text{si } d(\text{porteur}, \text{presseur}) > 3.5\,\text{m} \text{ et porteur face au jeu} \end{cases}$$
- **Ordre de grandeur réel** :
  - Bloc bas : $X_{\text{backline}} \in [18, 26]\,\text{m}$.
  - Bloc médian : $X_{\text{backline}} \in [34, 44]\,\text{m}$.
  - Ligne haute : $X_{\text{backline}} \in [48, 56]\,\text{m}$.
  - Vitesse de recul-frein sur ballon découvert : $-3.5$ à $-5.0\,\text{m/s}$ sur une distance de 4 à 8 mètres.
- **Mesure de contrôle dans le moteur** :
  - Sonde sur le délai d'amorce du recul : $\Delta t_{\text{recul}} \le 200\,\text{ms}$ (2 ticks moteur à $\Delta t = 0.1\,\text{s}$) dès l'instant où un porteur adverse entre dans le tiers médian sans opposant dans son cône de vision.
  - Condition de réfutation : la ligne avance ou reste immobile ($v_x \ge 0$) alors qu'une passe aérienne en profondeur est déclenchée depuis un ballon découvert.

### 2.3 Intensité et Déclencheurs de Pressing
- **Grandeur physique** : Rayon de harcèlement actif $R_{\text{press}}$ (m), vitesse d'engagement $v_{\text{press}}$ (m/s) et délai d'activation $T_{\text{trigger}}$ (ms).
- **Ordre de grandeur réel** :
  - Distance d'intervention au contact : **1,2 à 1,8 m** sans faire faute (Peter Zeidler : « rentrer dans le joueur sans faire faute »).
  - Déclencheur sur passe latérale ou en retrait : engagement initié dès que le ballon quitte le pied du passeur (anticipation sur temps de vol).
  - Durée de harcèlement individuel maximal : 4 à 6 secondes avant relais par un coéquipier.
- **Mesure de contrôle dans le moteur** :
  - Métrique de pression subie `pressureOn(state, carrier)` mesurée à l'instant du premier contrôle : doit être $\ge 1.5$ sur les passes ciblées par `pressTriggers` (`back_pass`, `slow_first_touch`).

### 2.4 Contre-Pressing (Gegenpressing)
- **Grandeur physique** : Fenêtre temporelle de contre-charge $T_{\text{gegen}}$ (s), nombre de joueurs participants $N_{\text{gegen}}$ et distance de sprint initiale $D_{\text{sprint}}$ (m).
- **Ordre de grandeur réel** :
  - Fenêtre critique : **3 à 5 secondes** (la règle des 5 secondes de Pep Guardiola ; Roger Schmidt : « gagner le ballon le plus vite possible dans les secondes qui suivent la perte »).
  - Nombre de joueurs mobilisés : **3 à 5 joueurs** encerclant la zone du ballon (Peter Zeidler : « 6 ou 7 joueurs actifs sur la récupération haute »).
  - Vitesse du premier sprint : $> 5.5\,\text{m/s}$ (sprint maximal).
- **Mesure de contrôle dans le moteur** :
  - Taux de récupération de possession dans les 5 secondes suivant une perte dans le camp adverse sous tactique `gegenpress` : cible **18 % à 26 %** des pertes hautes.
  - Condition de réfutation : moins de 2 joueurs réduisent leur distance au porteur dans les 1.5 seconde post-perte.

### 2.5 Sortie de Balle Basse & Relance (Salida La Volpe)
- **Grandeur physique** : Écartement latéral des défenseurs centraux $W_{\text{CB}} = |y_{\text{CB1}} - y_{\text{CB2}}|$ (m), décrochage du milieu sentinelle $X_{\text{DM}} - X_{\text{CB}}$ (m), hauteur des latéraux $X_{\text{FB}}$ (m).
- **Ordre de grandeur réel** :
  - Écartement des centraux sur 6 m : **38 à 46 m** (occupant la largeur des 16,5 m jusqu'aux lignes de touche de la surface).
  - Descente du milieu défensif (Busquets / sentinelle) : se place entre les deux centraux ou 2 à 4 m devant le gardien.
  - Hauteur des latéraux : positionnés à hauteur de la ligne médiane ($x \approx 48-55\,\text{m}$) et collés aux lignes de touche ($y \in [1, 6]\,\text{m}$ ou $[62, 67]\,\text{m}$).
- **Mesure de contrôle dans le moteur** :
  - Télémétrie sur 6 mètres joués court (`goalKickStyle === 'short'`) : $W_{\text{CB}} \ge 35\,\text{m}$ ; taux de passes courtes réussies du gardien $\ge 88\,\%$ ; zéro perte de balle plein axe à moins de 20 m de son but.

### 2.6 Combinaisons Collectives (Une-deux & Troisième Homme)
- **Grandeur physique** : Temps de cycle total de la séquence $\Delta T_{\text{combo}}$ (s), temps de maintien du ballon par le pivot $T_{\text{pivot}}$ (s) et avance temporelle du coureur $\Delta T_{\text{anticipation}}$ (ms).
- **Ordre de grandeur réel** :
  - Séquence $A \to B \to C$ : durée totale **1.4 à 2.2 secondes**.
  - Le pivot $B$ joue en une touche : temps de contrôle $T_{\text{pivot}} \le 0.3\,\text{s}$ (remise ou déviation).
  - Le troisième homme $C$ enclenche sa course d'appel **200 à 400 ms AVANT** que le pivot $B$ ne reçoive la passe (Xavi : « Piqué joue avec Messi, et c'est à ce moment que j'apparais... »).
- **Mesure de contrôle dans le moteur** :
  - Fréquence de complétion des `ScheduledRun` de type `third_man` : cible **8 à 15 combinaisons réussies par match** en tactique possession / jeu court.

### 2.7 Sécurité Défensive Préventive (Rest Defense)
- **Grandeur physique** : Nombre de joueurs maintenant une position défensive derrière la ligne du ballon lors des phases d'attaque $N_{\text{rest}}$, et leur distance de sécurité $\Delta x_{\text{rest}}$ (m).
- **Ordre de grandeur réel** :
  - Structure de rest defense : **4 à 5 joueurs** (formules 3+2 ou 2+3 : 2 défenseurs centraux + 1 latéral + 2 milieux, ou 3 défenseurs + 2 pivots ; Christian Gourcuff : « 5 joueurs en sécurité : 2 défenseurs et 3 milieux » ; Olivier Dall'Oglio : « 4 éléments qui restent derrière pour assurer l'équilibre »).
  - Écartement en retrait : **16 à 25 m** derrière le ballon.
- **Mesure de contrôle dans le moteur** :
  - Sonde sur phase `FINITION` ou `PROGRESSION` adverse : nombre de joueurs de l'équipe attaquante situés dans leur propre moitié de terrain ou à $> 20\,\text{m}$ en retrait du ballon toujours $\ge 4$.

### 2.8 Occupation des Cinq Couloirs Verticaux (Juego de Posición)
- **Grandeur physique** : Répartition de l'occupation spatiale sur les 5 couloirs de 13.6 m :
  - Couloir Extérieur Gauche ($y \in [0, 13.6]$)
  - Demi-Espace Gauche ($y \in [13.6, 27.2]$)
  - Axe Central ($y \in [27.2, 40.8]$)
  - Demi-Espace Droit ($y \in [40.8, 54.4]$)
  - Couloir Extérieur Droit ($y \in [54.4, 68]$)
- **Ordre de grandeur réel** :
  - Règle de Pep Guardiola / Paco Seirul-lo : au moins 1 joueur et au plus 2 joueurs dans chaque couloir vertical en phase d'attaque placée ; présence obligatoire d'au moins un relais dans chaque demi-espace.
- **Mesure de contrôle dans le moteur** :
  - Taux de présence simultanée dans les deux demi-espaces en phase `PROGRESSION` $\ge 80\,\%$ du temps de possession.

### 2.9 Tempo et Distribution des Passes
- **Grandeur physique** : Temps moyen de conservation du ballon par porteur $T_{\text{hold}}$ (s) et distribution des longueurs de passe $L_{\text{pass}}$ (m).
- **Ordre de grandeur réel** :
  - Tempo rapide (Tiki-taka / Sarri-ball) : $T_{\text{hold}} \in [0.8, 1.4]\,\text{s}$ ; passes courtes ($< 16\,\text{m}$) représentant $65\text{–}75\,\%$ du volume.
  - Tempo lent / temporisation : $T_{\text{hold}} \in [1.8, 2.6]\,\text{s}$.
  - Jeu direct : passes moyennes et longues ($> 25\,\text{m}$) représentant $40\text{–}50\,\%$ du volume.
- **Mesure de contrôle dans le moteur** :
  - Médiane de durée de possession par touche de balle ; distribution des passes émises catégorisées court/moyen/long conforme aux profils définis dans `roleProfiles.ts:preferredPassLengths`.

### 2.10 Sélectivité du Tir (Shot Quality & xG Threshold)
- **Grandeur physique** : xG moyen par tir tenté $\overline{\text{xG}}$, distance médiane des tirs $D_{\text{shot}}$ (m) et volume total de tirs par équipe $N_{\text{tirs}}$.
- **Ordre de grandeur réel** :
  - Volume par équipe : **11 à 15 tirs par match** (soit 22 à 30 tirs cumulés par match).
  - xG moyen par tir : **0.09 à 0.12**.
  - Tirs depuis l'intérieur de la surface : **60 à 68 %** du total (seuls 32 à 40 % de tirs lointains).
- **Mesure de contrôle dans le moteur** :
  - Somme des xG par match $\in [2.2, 3.2]$ cumulé sur 90 minutes ; ratio tirs dans la boîte / total $\ge 60\,\%$.

---

## 3. Cartographie contre le Code Existant : (a) Modélisé, (b) Partiel/Mal modélisé, (c) Absent

Cette section dresse l'inventaire rigoureux des briques tactiques dans l'arborescence du moteur.

```
+---------------------------------------------------------------------------------------------------------+
| DIMENSION TACTIQUE          | CLASSEMENT | FICHIER ET SYMBOLE CLÉ                                       |
+-----------------------------+------------+--------------------------------------------------------------+
| Hauteur de ligne défensive  | (b)        | decision/offBall/defensiveLine.ts:resolveLineHeight          |
| Ballon couvert / découvert  | (c)        | ABSENT : aucun lien pressureOn -> recul défensif             |
| Compacité du bloc           | (a)        | team/defensiveBlockShape.ts, offBall/compactness.ts          |
| Pressing contextuel score   | (a)        | tactics/contextualPressing.ts:pressingIntensity              |
| Dynamic line height legacy  | (b)        | tactics/contextualPressing.ts:dynamicLineHeight (axe Y faux) |
| Déclencheurs de pressing    | (a)        | decision/offBall/applyOffBallDecisions.ts:applySyncedPress   |
| Piège de pressing / nids    | (c)        | ABSENT : pas d'action coordonnée à 3 joueurs                 |
| Contre-pressing (Gegenpress)| (b)        | decision/offBall/cohesion.ts:applyCounterpressTransition     |
| Rest defense                | (a)        | decision/offBall/cohesion.ts:applyRestDefense                |
| Demi-espaces & 5 couloirs   | (a)        | tactics/patterns/centralPenetration.ts:isHalfSpace           |
| Grille zonale coach         | (a)        | tactics/zonalGrid/tables/                                    |
| Offre de passe proactive    | (b)/(c)    | PARTIELLE : offTheBall en rupture, absent en soutien (base) |
| Combinaison troisième homme | (b)        | tactics/patterns/thirdManRun.ts:detectThirdManOpportunity    |
| Cascade décision porteur    | (a)        | decision/onBall/decideCarrier.ts:selectFromMenu              |
| Mentalité d'équipe (FM24)   | (b)        | PONT SEUL : traducteurTactique.ts:308 (absente moteur local) |
| Doctrine de tir (FM24)      | (b)        | SPÉCIFIÉE : shotStyle.ts:13, en attente dans shotDecision    |
| Sortie de balle courte 6m   | (b)        | tactics/buildOutShape.ts (manque décrochage du 6)            |
| Rôles et devoirs (Duties)   | (a)        | roleProfiles.ts:defaultDutyForRole (defend/support/attack)   |
| Coups de pied arrêtés       | (a)        | setPieces/choreography.ts, corner.ts, freeKick.ts            |
| Erreur de passe multi-attr. | (b)        | actions/ (passErr trop uniforme, manque composure/vision)    |
+---------------------------------------------------------------------------------------------------------+
```

### Détail des composants audités

#### (a) Ce que le moteur modélise déjà fidèlement
1. **Compacité du bloc** :
   - Fichiers : `src/modules/matchEngine/team/defensiveBlockShape.ts`, `src/modules/matchEngine/decision/offBall/compactness.ts`.
   - Fonctionnement : `computeDefensiveBlockShape` calcule l'enveloppe géométrique du bloc ($x_{\text{max}} - x_{\text{min}}$) et `applyCompactness` applique une force de rappel élastique vers le barycentre du bloc pour éviter l'étirement excessif.
2. **Coups de pied arrêtés (Set Pieces)** :
   - Fichiers : `src/modules/matchEngine/setPieces/choreography.ts`, `corner.ts`, `freeKick.ts`, `penalty.ts`.
   - Fonctionnement : Routines complètes de placement zonal vs individuel sur corners, distances réglementaires respectées (9.15 m), et prise en compte des attributs de tireur/sauteur.
3. **Sécurité défensive (Rest Defense)** :
   - Fichiers : `src/modules/matchEngine/decision/offBall/cohesion.ts:applyRestDefense`, `src/modules/matchEngine/decision/offBall/restDefCoverSlide.ts`.
   - Fonctionnement : Épingle les joueurs les plus reculés de l'équipe attaquante pour contrer les départs en contre-attaque.
4. **Découpage en couloirs et Grille Zonale** :
   - Fichiers : `src/modules/matchEngine/tactics/patterns/centralPenetration.ts:isHalfSpace`, `src/modules/matchEngine/tactics/zonalGrid/`.
   - Fonctionnement : Les 5 bandes verticales de 13.6 m sont formalisées et les grilles zonales de coachs réels (Bielsa, Guardiola, Sarri, Simeone) projettent les décalages spatiaux.
5. **Profils de Rôles et Devoirs** :
   - Fichiers : `src/modules/matchEngine/roleProfiles.ts:defaultDutyForRole`.
   - Fonctionnement : Modulateurs comportementaux (`shootBias`, `passBias`, `attackPush`, `defensePull`, `pressRadius`, `appelBias`) par rôle et dérivation des devoirs *defend / support / attack*.

#### (b) Ce que le moteur modélise mal ou partiellement
1. **Hauteur de ligne défensive & Dynamic Line Height** :
   - Fichiers : `src/modules/matchEngine/decision/offBall/defensiveLine.ts`, `src/modules/matchEngine/tactics/contextualPressing.ts`.
   - Défaut prouvé : `contextualPressing.ts:dynamicLineHeight` calcule un offset sur l'axe Y (largeur) alors que la profondeur est sur l'axe X ! Il a été repéré et court-circuité par `defensiveLine.ts:20` (« NE PAS importer dynamicLineHeight : orphelin, axe Y legacy »). De son côté, `defensiveLine.ts` n'intègre aucune notion de pression sur le porteur : il recule selon une fonction purement géométrique de `ballX`.
2. **Mentalité d'équipe (FM24) — Branchée du mauvais côté** :
   - Fichiers : `src/modules/matchEngine/traducteurTactique.ts:308`, `src/modules/matchEngine/injecterDansLeurMatch.ts`, `src/modules/matchEngine/tactics/types.ts:162`.
   - Défaut prouvé : Contrairement à l'annotation obsolète de `coachOrders.ts:51`, `mentality` n'a pas « zéro lecteur » : elle est fidèlement lue par le traducteur vers le moteur de référence (`MENTALITE_VERS_LEUR`) pour être injectée dans la simulation adverse. En revanche, **notre propre moteur local l'ignore totalement**. Le champ unifié `mentalityScalar` déclaré dans `tactics/types.ts:162` est figé à `0.5` dans `gameStateContext.ts:256`. Le lecteur canonique doit vivre dans `gameStateContext.ts` pour alimenter à la fois le pont et nos propres règles décisionnelles.
3. **Doctrine de tir (FM24) — Spécifiée mais non raccordée** :
   - Fichiers : `src/modules/matchEngine/cognition/intent/technique/shotStyle.ts:13`, `archive/cahiers-realises/Cahier_ShotStyle_PassStyle.md:270`, `src/modules/matchEngine/decision/onBall/shotDecision.ts`.
   - Défaut prouvé : Documentée formellement comme une « gate de fréquence, PAS le geste », elle n'a pas été insérée dans `shotDecision.ts` pour moduler le seuil `minShootXG` et les chances de tenter sa chance de loin versus temporiser dans la surface.
4. **Combinaisons à trois et une-deux (Third-man run)** :
   - Fichiers : `src/modules/matchEngine/tactics/patterns/thirdManRun.ts`, `src/modules/matchEngine/decision/onBall/decideCarrier.ts`.
   - Défaut prouvé : Bien que le bug de seuil de probabilité ait été résolu, `detectThirdManOpportunity` bride les séquences à 1 seul bid actif par équipe (`state.scheduledRuns`), interdisant à deux doublettes de combiner simultanément sur le terrain. Le receveur subit en outre un retard d'un tick avant d'enclencher sa course.
5. **Contre-pressing (Gegenpressing)** :
   - Fichiers : `src/modules/matchEngine/decision/offBall/cohesion.ts:applyCounterpressTransition`.
   - Défaut prouvé : Le contre-pressing est bridé par la constante `CCOH_MAX_BALL_COMMITTED = 2`. Deux joueurs seulement peuvent chasser le ballon à la perte, interdisant le regroupement compact à 4 ou 5 joueurs théorisé par Zeidler et Schmidt.
6. **Sortie de balle sur six mètres (Build-up)** :
   - Fichiers : `src/modules/matchEngine/tactics/buildOutShape.ts`.
   - Défaut prouvé : Les deux centraux s'écartent bien (`W_CB >= 35m`), mais le milieu défensif ne s'intercale pas systématiquement en sortie basse (Salida La Volpe incomplète) et les passes latérales courtes dans la surface manquent de fluidité face à un pressing haut étagé.
7. **Qualité de passe et déchet (Badpass)** :
   - Fichiers : `src/modules/matchEngine/actions/` (`passErr`).
   - Défaut prouvé : 76,8 % des ballons perdus sont des passes manquées. L'erreur balistique ne discrimine pas assez le sang-froid (`composure`) sous pression vive par rapport à une passe libre non contestée.
8. **Offre de passe proactive en soutien direct** :
   - Fichiers : `src/modules/matchEngine/decision/offBall/base.ts:40`.
   - Défaut prouvé : L'attribut `offTheBall` est exploité dans les filtres d'appel lointain (`positionalStructure.ts`), mais le soutien court face au porteur en possession médiane se résume à une dérive passive vers le ballon.

#### (c) Ce qui est totalement absent
1. **Règle « Ballon couvert / Ballon découvert » sur la profondeur** :
   - Constat : Aucun calcul dans `defensiveLine.ts` n'évalue si le porteur adverse a levé la tête sans opposition pour commander le recul immédiat de la charnière centrale, ni ne commande de remonter d'un pas quand le porteur est sous pression vive.
2. **Pièges de pressing collectifs orientés (Pressing traps)** :
   - Constat : Pas d'orientation tactique coordonnée où l'attaquant bloque l'axe et les milieux étouffent les lignes de passe latérales pour forcer l'adversaire à concéder une perte sur la ligne de touche.
3. **Démarquage court d'évitement d'ombre** :
   - Constat : Absence d'algorithme déplaçant activement un partenaire de 1 à 3 mètres hors du cône d'interception d'un adversaire lors du jeu de passe court.

---

## 4. Les Sources : Ouvrages, Chapitres et Fondements Doctrinaires

Toutes les grandeurs et règles opérationnelles énoncées ci-dessus proviennent des deux ouvrages de référence de Julien et du corpus de doctrine interne.

### 4.0 Inventaire exhaustif du dépôt de référence (`JulienDelquignies/book`)

Le dépôt cloné (`https://github.com/JulienDelquignies/book`, commit unique `e3c907501a81f1d0e98159df4b361c171da5e17a`) contient **exactement deux fichiers texte**, totalisant **21 860 lignes** et **1,46 Mo** de texte brut. Aucun autre document n'est présent sur ce dépôt.

Voici l'inventaire matériel précis et le niveau d'exploitation de chaque fichier :

1. **`685975367-Comment-regarder-un-match-de-foot-Les-cahiers-du-football-FOOTBALL-etc-z-lib-org.txt`**
   - **Titre** : *Comment regarder un match de foot ?*
   - **Auteurs** : Raphaël Cosmidis, Gilles Juan, Christophe Kuchly, Julien Momont (Les Cahiers du football). Préface de Christian Gourcuff.
   - **Format & Volume** : Fichier texte brut UTF-8 (`.txt`), 736 673 octets, 11 103 lignes (~120 000 mots).
   - **Niveau d'exploitation** :
     - *Lecture intégrale et approfondie* :
       - Préface de Christian Gourcuff (vision systémique, bloc compact, réduction des espaces).
       - Chapitre « Les systèmes de jeu » (4-4-2, 4-3-3, 3-5-2, le faux 9, Denoueix, Benítez).
       - Chapitre « Ne pas avoir le ballon — La défense placée » (Principes généraux, les 4 référentiels de René Maric, « Coulisser en bloc », règle cardinale « Ballon couvert / Ballon découvert » de Guy Lacombe et Stéphane Moulin, « La couverture » et modèle 1+3 Sacchi/Gourcuff).
       - Chapitre « Avoir le ballon » (La conducción théorisée par Guardiola, L'Homme libre expliqué par Xavi/Lillo, Le Troisième homme démontré par Cruyff/Xavi, La Salida Volpiana de Ricardo La Volpe et Sergio Busquets, Occupation de la largeur et analogie directe avec Football Manager).
       - Chapitre « Les coups de pied arrêtés » (Corners en zone vs marquage individuel).
     - *Survol / Parcours thématique* : Chapitres historiques (WM, Catenaccio, 4-2-4) et portraits de carrières individuelles.

2. **`698397464-Comment-gagner-un-match-de-foot-Raphael-Cosmidis-et-Collectif.txt`**
   - **Titre** : *Comment gagner un match de foot ?*
   - **Auteurs** : Raphaël Cosmidis, Gilles Juan, Christophe Kuchly, Julien Momont et Collectif. Préface d'André Villas-Boas.
   - **Format & Volume** : Fichier texte brut UTF-8 (`.txt`), 724 726 octets, 10 757 lignes (~115 000 mots).
   - **Niveau d'exploitation** :
     - *Lecture intégrale et approfondie* :
       - **Partie III : « Comment gagner la bataille tactique ? »** (lignes 6853 à 7630) analysée in extenso, chapitre par chapitre :
         * « Construire depuis l'arrière » par Olivier Dall'Oglio (sorties 6m, fixation par les deux 6, décrochages en appui-remise, rest defense à 4).
         * « Battre un pressing haut » par Ludovic Batelli (le gardien comme 3e défenseur, dézonage intérieur des latéraux).
         * « Attaquer un bloc bas » par Alain Casanova (patience, étirement, attaques des demi-espaces).
         * « Le contre-pressing » par Peter Zeidler (différence pressing/gegenpress, le premier sprint, ne pas faire faute, 6-7 joueurs mobilisés, gestion de la dernière ligne).
         * « Défendre en zone » par Christian Gourcuff (distances métriques de 10-15 m en bloc médian et 6 m en bloc bas, mécanique de l'oblique, rest defense à 5).
         * « Le pressing » par Roger Schmidt (pressing haut en infériorité apparente, timing calé sur le temps de passe adverse, rôle pivot des deux milieux offensifs dans les demi-espaces).
         * « Mener une contre-attaque » par Luka Elsner (les 3 zones d'entrée de surface, tenue préventive des excentrés).
       - **Partie II : « Comment préparer la rencontre ? »** (analyse vidéo, micro-principes de séance, animation sectorielle).
     - *Survol / Parcours thématique* : Partie I (« Construire la culture club » — causeries et management) et Partie IV (« Optimiser la performance » — préparation athlétique, nutrition, sommeil).

3. **Complément doctrinal interne : `Cahier_Beau_Jeu.md`**
   - **Origine** : Branche `docs/cahier-beau-jeu` du dépôt de simulation, par Julien Delquignies.
   - **Niveau d'exploitation** : Lecture intégrale (les 3 faits mesurés réfutant l'égoïsme du porteur, les 3 leviers orthogonaux A/B/C, et les contraintes de non-régression anti-Goodhart).

### 4.1 *Comment regarder un match de foot ?* (Raphaël Cosmidis, Gilles Juan, Christophe Kuchly, Julien Momont — Éditions Solar)
- **Compacité et bloc court (25 mètres)** :
  - *Chapitre « Ne pas avoir le ballon — La défense placée », section « Coulisser en bloc »* (l. 4830–4852) : Citation directe de Marcelo Bielsa (« Avoir une équipe courte avec pas plus de 25 mètres de l'attaque à la défense ») et de Christian Gourcuff (« Le bloc-équipe doit être court et étroit »).
- **Règle du ballon couvert / découvert (Contrôle de la profondeur)** :
  - *Section « Coulisser en bloc »* (l. 4862–4880) : Guy Lacombe (« C'est le ballon qui déclenche la montée, si le porteur est cadré ou pas ») et Stéphane Moulin (« Si le porteur est libre, l'ensemble du bloc doit reculer pour se prémunir d'un ballon par-dessus ; s'il est bien pressé, le bloc peut remonter »).
- **Organisation en zone et principes de cadrage/couverture** :
  - *Section « Principes généraux »* (l. 4751–4792) : Typologie de René Maric (les 4 référentiels : position, homme, espace, ballon) et méthode Raynald Denoueix (« prendre, faire prendre, lâcher... »).
  - *Section « La couverture »* (l. 4887–4920) : Modèle du « 1+3 » d'Arrigo Sacchi et Christian Gourcuff (un joueur sort au cadrage, trois assurent la couverture oblique).
- **Juego de Posición : Conducción, Homme libre et Troisième homme** :
  - *Chapitre « Avoir le ballon »* (l. 6664–6738) :
    - *La Conducción* : Pep Guardiola sur la fixation par le défenseur central pour attirer l'adversaire et libérer un coéquipier.
    - *L'Homme Libre* : Xavi Hernández sur l'obtention de l'homme libre entre les lignes (Zwischenlinienraum).
    - *Le Troisième Homme* : Démonstration intégrale par Xavi (« Piqué veut jouer avec moi, je suis marqué... Messi décroche, devient le deuxième homme... Piqué joue avec Messi qui lui remet, et j'apparais... impossible à défendre »).
- **La Salida Lavolpiana (Sortie basse avec décrochage du 6)** :
  - *Chapitre « Avoir le ballon »* (l. 6740–6790) : Ricardo La Volpe, Juanma Lillo et Pep Guardiola sur le décrochage systématique de Sergio Busquets entre les centraux pour créer le 3 contre 2 initial.
- **Référence explicite à Football Manager** :
  - *Chapitre « Avoir le ballon »* (l. 6795–6800) : Consignes d'écartement du jeu et de largeur de terrain directement comparées aux mécanismes de Football Manager.

### 4.2 *Comment gagner un match de foot ?* (Raphaël Cosmidis, Gilles Juan, Christophe Kuchly, Julien Momont — Éditions Solar)
- **Construction depuis l'arrière & Sorties sur six mètres** :
  - *Partie III, Chapitre « Construire depuis l'arrière » par Olivier Dall'Oglio* (l. 6920–7087) : Fixation par les deux 6 dans l'axe, latéraux qui s'écartent, décrochages en appui-remise dos au but, et responsabilité des 4 joueurs de couverture défensive (rest defense).
- **Contre-pressing (Gegenpressing à la perte)** :
  - *Partie III, Chapitre « Le contre-pressing » par Peter Zeidler* (l. 7233–7334) : Différence entre pressing et contre-pressing ; importance capitale du premier sprint à fond sans faire faute ; mobilisation de 6 à 7 joueurs avancés ; ligne défensive haute qui anticipe sans spéculer sur le hors-jeu passif.
- **Défense en zone, distances métriques et oblique** :
  - *Partie III, Chapitre « Défendre en zone » par Christian Gourcuff* (l. 7336–7464) : **Les distances mesurées exactes** : 10 à 15 m entre les lignes en bloc médian, réduit à 6 m en bloc bas ; mécanique de l'oblique sur changement d'aile ; rest defense avec 5 joueurs en sécurité (structure 3+2 ou 2+3).
- **Pressing haut et étagement dans les demi-espaces** :
  - *Partie III, Chapitre « Le pressing » par Roger Schmidt* (l. 7499–7586) : Pressing en infériorité numérique apparente ; déclenchement sur le temps de passe adverse ; rôle crucial des deux milieux offensifs positionnés dans les demi-espaces.
- **Sortie de pressing haut par le gardien** :
  - *Partie III, Chapitre « Battre un pressing haut » par Ludovic Batelli* (l. 7088–7110) : Utilisation du gardien comme 3e défenseur pour libérer un surnombre au milieu.
- **Contre-attaque structurée** :
  - *Partie III, Chapitre « Mener une contre-attaque » par Luka Elsner* (l. 7587–7630) : Occupation systématique des 3 zones d'entrée de surface ; placement préventif des excentrés.

### 4.3 *Cahier du Beau Jeu* (Julien Delquignies — Doctrine interne)
- Réfutation empirique de l'égoïsme du porteur (la passe gagne 75 à 83 % des arbitrages).
- Confirmation du blocage complet des combinaisons (0 occurrence réelle).
- Établissement des trois leviers orthogonaux A, B, C et de la méthodologie anti-Goodhart (balance-check $N=30$, conservation des garde-fous de progressivité des passes).

---

## 5. Ce qui n'a pas pu être tranché, et ce qu'il faudrait pour le trancher

Quatre questions architecturales et empiriques restent ouvertes. Elles ne doivent pas être résolues au doigt mouillé mais par des protocoles d'expérimentation stricts.

### 5.1 Grille zonale rigide (`ME_ZONAL_GRID`) versus Dynamique émergente
- **L'incertitude** : Dans `src/modules/matchEngine/decision/offBall/base.ts:93`, la grille zonale (`resolveZonalTarget`) impose des offsets géométriques tabulés par coach (`sarri`, `pep`, `simeone`, `bielsa`). Cette grille stabilise l'occupation du terrain mais risque de figer les joueurs et d'étouffer l'offre de passe dynamique (Levier A).
- **Ce qu'il faut pour trancher** : Lancer un banc de test comparatif ($N=50$ matchs) avec `ME_ZONAL_GRID=1` contre `ME_ZONAL_GRID=0` en mesurant :
  1. Le volume d'offres de passe réussies (`passOffersCompleted`),
  2. L'espace vide concédé au centre du terrain (sonde `probeShape`).
  Si la grille zonale réduit de plus de 25 % les lignes de passe ouvertes par le Levier A, elle devra être rétrogradée en simple pondération initiale plutôt qu'en ancre rigide.

### 5.2 Plafond de joueurs au contre-pressing (`CCOH_MAX_BALL_COMMITTED = 2`)
- **L'incertitude** : Le code bride l'engagement à 2 joueurs maximum au ballon pour éviter le syndrome de la « fourmilière » où 10 joueurs s'agglutinent sur la balle. Or, Peter Zeidler et Roger Schmidt insistent sur le fait qu'un contre-pressing efficace requiert 3 à 5 joueurs convergeant immédiatement pour étouffer le receveur.
- **Ce qu'il faut pour trancher** : Créer un test unitaire instrumenté évaluant une dérogation temporaire : autoriser jusqu'à 4 joueurs commis au ballon *uniquement* pendant la fenêtre de transition de contre-pressing ($t \le 2.5\,\text{s}$ après la perte) sous consigne `gegenpress`, tout en mesurant si le cône de tir adverse en cas d'échec reste couvert par la rest defense.

### 5.3 Dégradation de la compacité par la fatigue cumulée
- **L'incertitude** : Christian Gourcuff rappelle que maintenir un bloc à 10-15 m d'écartement est épuisant cognitivement et physiquement. Actuellement, la compacité mesurée reste quasi identique entre la 10e et la 85e minute. Faut-il relier la compacité d'équipe (`defensiveBlockShape.ts`) au niveau moyen de stamina et à la fatigue cognitive (`cognitiveDegradation.ts`) ?
- **Ce qu'il faut pour trancher** : Mesurer sur 20 matchs complets la dérive réelle des distances inter-lignes en fin de rencontre. Si les équipes fatiguées ne s'étirent pas de 3 à 6 mètres supplémentaires, introduire une modulation de compacité indexée sur `stamina < 40%`.

### 5.4 Synchronisation temporelle de l'appel : pré-décision versus frame-tick
- **L'incertitude** : Dans la réalité, un ailier ou un relayeur démarre son appel de rupture 200 à 400 ms *avant* la passe (dès que le porteur lève la tête ou oriente ses épaules). Dans le moteur cadencé à $\Delta t = 0.1\,\text{s}$, exposer l'intention top-k du porteur à la frame courante peut générer du bégaiement cinématique (*jitter*) si le porteur change d'avis au tick suivant sous pression vive.
- **Ce qu'il faut pour trancher** : Évaluer par télémétrie si l'engagement du porteur (`CARRIER_COMMITMENT = 0.25s`) suffit à verrouiller la trajectoire de l'appelant sans provoquer d'appels fantômes brusquement avortés.

---

### Résumé pour l'orchestration
Le présent brief fournit le cahier des charges exécutable pour aligner le moteur de match sur la référence footballistique et FM24. Les agents implémenteurs doivent traiter les chantiers dans l'ordre strict 1 à 5, en validant chaque palier par un balance-check sans régression sur la progressivité existante.
