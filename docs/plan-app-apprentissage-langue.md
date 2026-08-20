# Plan — App d'apprentissage de langue (iOS + macOS)

Deux features :
1. **Agent conversationnel** pour travailler à l'oral une liste de vocabulaire fournie par l'utilisateur.
2. **Quiz / flashcards intrusifs** qui apparaissent sur l'écran (à l'ouverture d'une app, ou aléatoirement) et qu'il faut résoudre pour les faire disparaître.

Ce document se concentre sur la **feature 2**, qui est celle qui porte tout le risque technique.

---

## 1. Le verdict, tout de suite

La feature 2 telle qu'imaginée — « un quiz s'affiche par-dessus tout, je ne peux rien faire tant que je n'ai pas répondu » — est :

| Plateforme | Faisable ? | Comment |
|---|---|---|
| **macOS** | ✅ Oui, littéralement | Fenêtre overlay au-dessus de tout, UI libre, détection d'app native |
| **iOS** | ⚠️ Oui, mais dans une boîte très étroite | API Screen Time (*Family Controls*), UI imposée par le système |

Sur iOS, **aucune app ne peut dessiner par-dessus une autre app**. Il n'y a pas d'équivalent au `SYSTEM_ALERT_WINDOW` d'Android. Le seul mécanisme qui interpose quelque chose entre l'utilisateur et une app qu'il ouvre est le **« shield » de Screen Time** — et ce shield est dessiné par iOS, pas par toi. Tu ne fournis que des paramètres.

C'est la contrainte structurante de tout le projet. Le reste du plan consiste à en tirer le maximum.

---

## 2. Les 5 canaux d'interruption possibles

Il faut arrêter de penser « une seule feature » et penser **cinq canaux** de nature différente, à combiner. Je les classe par force d'interruption.

### Canal A — Shield Screen Time (iOS) — *le vrai blocage*

**Frameworks :** `FamilyControls` (autorisation), `ManagedSettings` (application du blocage), `DeviceActivity` (planification), `ManagedSettingsUI` (apparence du shield), `ShieldAction` (réaction aux boutons).

**Ce que ça fait :** l'utilisateur ouvre Instagram → iOS affiche un écran de blocage plein écran à la place. Ton extension `ShieldConfigurationExtension` fournit l'apparence, ton extension `ShieldActionExtension` réagit aux appuis.

**Ce que tu contrôles exactement** (`ShieldConfiguration`) — et rien d'autre :
- `backgroundColor`, `backgroundBlurStyle`
- `icon` (une `UIImage`)
- `title`, `subtitle` (texte + couleur)
- `primaryButtonLabel`, `primaryButtonBackgroundColor`
- `secondaryButtonLabel`

**Ce que tu ne peux pas faire :** du SwiftUI arbitraire, un champ de saisie, un timer, du son, plus de deux boutons, une liste de 4 réponses.

**Le design qui rentre dans la boîte** — et il est meilleur qu'il n'en a l'air :

```
        ┌──────────────────────────────┐
        │      [icône générée]         │
        │                              │
        │   « der Schlüssel »          │  ← title
        │   Question 2 / 3             │  ← subtitle
        │                              │
        │  ┌────────────────────────┐  │
        │  │      la clé            │  │  ← primaryButtonLabel
        │  └────────────────────────┘  │
        │        la porte              │  ← secondaryButtonLabel
        └──────────────────────────────┘
```

Le QCM binaire (bonne réponse / distracteur) tient parfaitement. Et surtout, `ShieldActionResponse.defer` **redessine le shield immédiatement** en rappelant `configuration(shielding:)`. C'est le mécanisme clé : il permet **d'enchaîner plusieurs questions** dans un même shield.

Boucle complète :

| Événement | Réponse de l'extension | Effet |
|---|---|---|
| Bonne réponse, questions restantes | `.defer` | Question suivante s'affiche |
| Mauvaise réponse | `.defer` | « Faux — la clé. Question suivante » (et la carte est replanifiée) |
| 3/3 réussies | retirer le token du `ManagedSettingsStore`, puis `.defer` | Shield disparaît, app accessible N minutes |
| Bouton « abandonner » | `.close` | L'app bloquée se ferme |

Deux astuces utiles :
- L'`icon` étant une `UIImage`, on peut y **rendre une vue SwiftUI** via `ImageRenderer` (barre de progression, streak, phrase d'exemple). C'est un espace petit mais c'est ta seule surface graphique libre.
- Le déblocage temporaire se fait en retirant le token de `store.shield.applications` puis en le remettant via un `DeviceActivitySchedule` ou un timer côté app.

**Limitations à accepter :**
- ❌ **Impossible d'ouvrir ton app depuis le shield.** `ShieldActionResponse` ne propose que `.none`, `.defer`, `.close` — il n'y a pas de `.openParentApp`. Les apps du Store qui semblent le faire utilisent des API privées (risque de rejet/ban) ou une notification locale que l'utilisateur doit taper. **À ne pas faire.** Le quiz doit vivre entièrement dans le shield.
- ❌ Le `FamilyActivityPicker` renvoie des **tokens opaques**. Tu ne sauras jamais que l'utilisateur a choisi « TikTok » — pas de nom, pas de bundle ID, pas d'icône. Les tokens sont comparables et persistables, donc on peut faire « difficulté par app », mais l'utilisateur devra les nommer lui-même.
- ⚠️ Les extensions ont un budget mémoire/temps très serré, et **pas d'accès réseau fiable**. Voir §4 : l'extension ne doit faire que *lire* une question déjà calculée.

**Entitlement — le point bloquant administratif :**
- `com.apple.developer.family-controls` nécessite un **compte Apple Developer payant** (99 $/an). Impossible avec un Apple ID gratuit.
- En développement local, il suffit de l'ajouter au projet.
- Pour **TestFlight ou l'App Store**, il faut une demande d'approbation séparée à Apple — **par bundle ID *et* par extension** (donc 3 demandes : app, shield config, shield action). Délai constaté en 2026 : de 1 jour à plusieurs semaines, parfois sans réponse. À anticiper très tôt si tu veux distribuer.
- Pour un usage strictement personnel, tu peux rester en build de développement : profil valable **1 an** avec un compte payant (7 jours avec un compte gratuit — invivable).

**Le côté « aléatoire » :** `DeviceActivitySchedule` déclenche ton `DeviceActivityMonitor` à intervalles (ex. toutes les heures). Dans le callback `intervalDidStart`, tu tires au sort si le shield est actif pour cette fenêtre. Tu peux aussi déclencher sur seuil d'usage (`eventDidReachThreshold` : « après 10 min sur cette app »), ce qui est souvent plus pertinent que le pur hasard.

---

### Canal B — Automatisation Raccourcis (iOS) — *zéro entitlement, UI libre*

L'utilisateur crée dans l'app Raccourcis une automatisation personnelle : **« Quand [Instagram] est ouverte » → « Exécuter immédiatement » (sans notification) → [ton App Intent]**. Ton `AppIntent` déclaré avec `openAppWhenRun = true` **ouvre ton app au premier plan** avec un quiz complet.

| ✅ Avantages | ❌ Inconvénients |
|---|---|
| Aucune autorisation Apple, marche aujourd'hui | Configuration manuelle, une automatisation par app |
| UI totalement libre : 4 choix, saisie, audio, images | C'est de la **friction**, pas du blocage : l'app cible est ouverte derrière, l'utilisateur peut revenir d'un swipe |
| Fonctionne sur compte développeur gratuit | Un bandeau système peut apparaître ; latence de 1-2 s |
| Le même écran de quiz sert partout ailleurs | L'utilisateur peut désactiver l'automatisation en 3 taps |

**C'est le meilleur point de départ.** Il donne 80 % de la sensation recherchée pour 20 % du travail, et il valide le vrai sujet (est-ce que ce produit est agréable à utiliser ?) avant d'investir dans Screen Time.

---

### Canal C — Notifications locales aléatoires (iOS + macOS)

La flashcard arrive en notification, répondable **sans ouvrir l'app** :
- `UNNotificationAction` → boutons de QCM (compte sur 2 à 4 visibles selon le contexte d'affichage).
- `UNTextInputNotificationAction` → réponse libre au clavier.
- `UNNotificationContentExtension` → UI custom dans la notification déroulée (boutons, switches). Attention : depuis cette UI custom, **on ne peut pas ouvrir l'app**.

**Planification aléatoire :** iOS ne garde que les **64 notifications locales** les plus proches. On planifie donc ~48 tirages sur 2-3 jours (heures aléatoires dans une plage définie par l'utilisateur, ex. 9h-21h, hors réunions), et on regarnit la file à chaque lancement de l'app + via un `BGAppRefreshTask`.

Faible interruption, mais **le canal le plus fiable, sans aucune permission spéciale, et qui marche aussi sur Mac et Apple Watch.**

---

### Canal D — Widgets interactifs & Live Activity (iOS)

Widget écran d'accueil / écran verrouillé affichant une carte, avec boutons `AppIntent` (iOS 17+) : tap sur la bonne traduction → le widget se met à jour sur place. Zéro interruption mais présence permanente. Contraintes : budget de rafraîchissement limité, pas de son, animations quasi nulles. **Bon rapport effet/effort, à faire après le canal C** (même code de rendu de carte).

---

### Canal E — macOS — *là, tu as les mains libres*

C'est la plateforme où la feature 2 peut être exactement ce que tu décris.

- **Détection d'ouverture d'app :** `NSWorkspace.shared.notificationCenter` → `didActivateApplicationNotification`. Tu reçois le `NSRunningApplication` complet (nom, bundle ID, icône). Aucun entitlement.
- **Overlay :** une `NSWindow` borderless, `level = .screenSaver`, `collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]`, contenu SwiftUI arbitraire. Vrai quiz modal plein écran.
- **Rendre l'esquive difficile :** `NSApp.presentationOptions = [.disableProcessSwitching, .hideDock, .disableForceQuit]` bloque Cmd-Tab et le Dock tant que le quiz est affiché.
- **Contrainte :** ces options de présentation et l'observation système s'accommodent mal du sandbox App Store. Distribution recommandée : **hors App Store, en Developer ID notarisé**.

> À retenir : **la feature 2 est nettement plus forte sur Mac que sur iPhone.** Si tu passes beaucoup de temps devant un Mac, c'est là qu'il faut mettre l'effort en priorité.

---

## 3. Récapitulatif de décision

| Canal | Force | Effort | Entitlement | Compte payant | Quand |
|---|---|---|---|---|---|
| C — Notifications | 🟡 Faible | 🟢 1-2 j | Non | Non | **Phase 2** |
| D — Widgets | 🟡 Faible | 🟢 1 j | Non | Non | Phase 2 |
| B — Raccourcis | 🟠 Moyenne | 🟢 2-3 j | Non | Non | **Phase 3** |
| A — Screen Time | 🔴 Forte | 🔴 3-5 j | **Oui** | **Oui** | Phase 4 |
| E — macOS overlay | 🔴 Totale | 🟠 2-3 j | Non | Pour distribuer | **Phase 5** |

---

## 4. Architecture technique

### Structure du projet

```
LangApp/
├── LangCore/                    ← Swift Package, partagé par TOUTES les cibles
│   ├── Model/                   ← Deck, Card, Review, Setting
│   ├── Scheduler/               ← FSRS (répétition espacée)
│   ├── QuizGen/                 ← génération question + distracteurs
│   ├── SharedStore/             ← lecture/écriture App Group
│   └── Import/                  ← CSV / TSV / collage de liste
├── LangApp-iOS/                 ← app principale SwiftUI
├── LangApp-macOS/               ← app + overlay AppKit
├── ShieldConfiguration.appex    ← apparence du shield  (canal A)
├── ShieldAction.appex           ← logique de réponse   (canal A)
├── DeviceActivityMonitor.appex  ← planification         (canal A)
├── NotificationContent.appex    ← UI de notification    (canal C)
└── Widget.appex                 ← widget interactif     (canal D)
```

### La règle d'or des extensions

> **Les extensions ne calculent rien. Elles lisent une question déjà prête.**

Les extensions Screen Time et notification s'exécutent avec très peu de mémoire, très peu de temps, et sans garantie de réseau. Ouvrir SwiftData/CloudKit dedans est la façon la plus sûre de se retrouver avec un shield blanc en production.

**Solution :** l'app principale maintient dans un **App Group** un petit fichier `next_questions.json` — une file de 20 à 50 questions pré-calculées et pré-formatées (énoncé, bonne réponse, distracteur, ID de carte). L'extension :
1. lit le fichier (quelques ms) ;
2. affiche la question ;
3. **écrit** le résultat dans un fichier `pending_reviews.json` ;
4. l'app principale, à son prochain lancement, ingère ces résultats et met à jour le vrai modèle + regarnit la file.

Ce découplage est ce qui fait tenir le tout. À concevoir dès le début, pas après.

### Données et synchronisation

- **Stockage :** SwiftData (rapide à écrire) ou GRDB (plus de contrôle, meilleur pour un accès concurrent depuis les extensions). Recommandation : **GRDB**, précisément à cause du partage App Group.
- **Sync iPhone ↔ Mac :** CloudKit base privée. Tes decks et ta progression suivent.
- **Import de liste :** collage de texte ou fichier `.csv`, format `mot;traduction;exemple;catégorie`. Ajout ultérieur possible : import Anki `.apkg`.
- **Ordonnancement :** **FSRS** (implémentations Swift disponibles) plutôt que SM-2 — meilleurs intervalles, et c'est aujourd'hui le standard de fait.
- **Génération des distracteurs :** aucun LLM nécessaire. On tire dans le même deck, même catégorie grammaticale, longueur comparable, en excluant les quasi-synonymes. Rapide, déterministe, hors ligne.

---

## 5. Feature 1 — l'agent conversationnel

Boucle classique : **micro → transcription → LLM → synthèse vocale**.

| Étage | Choix recommandé | Alternative |
|---|---|---|
| Transcription | `SpeechAnalyzer` / `SpeechTranscriber` (iOS 26+, on-device, gratuit, faible latence) | Whisper API |
| Conversation | **Claude** (`claude-sonnet-5`) via un proxy | — |
| Synthèse vocale | `AVSpeechSynthesizer` (gratuit, hors ligne) | ElevenLabs si tu veux vraiment la qualité |

**Point de sécurité :** ne jamais embarquer une clé API dans l'app. Un petit proxy (Cloudflare Worker, ~30 lignes) qui détient la clé et relaie les appels suffit.

**Le prompt système est le cœur de la feature**, pas la plomberie :

> Tu es un partenaire de conversation en allemand, niveau A2. Tu dois amener naturellement ces 12 mots dans la discussion : […]. Phrases courtes. Si l'utilisateur fait une erreur, reformule correctement sans interrompre le fil. Ne sors jamais du personnage.

**Ce qui relie les deux features — et fait que c'est un produit, pas deux gadgets :**
1. L'agent reçoit en entrée les **mots « dus » du jour** selon FSRS.
2. En fin de session, il renvoie un JSON structuré : quels mots ont été employés, correctement ou non.
3. Ce retour alimente **le même moteur de révision** que les quiz du canal A/B/C.

Un mot bien utilisé à l'oral doit espacer sa prochaine flashcard. C'est ce couplage qui donne sa valeur à l'ensemble.

**Bonus hors ligne :** le **Foundation Models framework** (modèle ~3 Mds de paramètres on-device, iOS 26 / macOS 26, gratuit, avec sortie structurée typée via `@Generable`) est trop limité pour une vraie conversation nuancée, mais **parfait pour noter une réponse libre** dans un quiz (« "la clé" vs "clé" — accepté »), hors ligne et sans coût.

---

## 6. Plan par phases

| Phase | Contenu | Durée |
|---|---|---|
| **0 — Décisions** | Compte Apple payant ? Usage perso ou App Store ? Langue cible ? iPhone ou Mac en premier ? | ½ j |
| **1 — Socle** | `LangCore` : modèle, import CSV, FSRS, génération de quiz, store App Group. Écran de révision classique dans l'app. **Tout le reste en dépend.** | 2-3 j |
| **2 — Ça apparaît tout seul** | Canal C (notifications aléatoires) + canal D (widget). Sans aucun entitlement. Premier moment où l'app devient réelle. | 1-2 j |
| **3 — Ça apparaît quand j'ouvre une app** | Canal B (automatisations Raccourcis) + écran de quiz plein écran. **La feature 2 devient tangible sur iPhone.** | 2-3 j |
| **4 — Le vrai blocage** | Canal A (Screen Time). Demande d'entitlement **à lancer dès la phase 0** si distribution envisagée. | 3-5 j |
| **5 — macOS** | Canal E : overlay plein écran, détection d'app, blocage de Cmd-Tab. Meilleur ratio effort/effet du projet. | 2-3 j |
| **6 — Agent vocal** | Proxy + boucle STT/LLM/TTS + réinjection dans FSRS. | 3-5 j |

**Ordre de grandeur :** phases 1→3 tiennent dans un week-end long et donnent déjà quelque chose d'utilisable tous les jours. L'ensemble représente ~3 semaines à temps partiel.

---

## 7. Risques

| Risque | Impact | Parade |
|---|---|---|
| Entitlement Family Controls jamais approuvé | Canal A mort pour la distribution | Déposer la demande en phase 0 ; les canaux B/C/D ne dépendent pas d'Apple |
| Extension shield qui plante (mémoire/temps) | Écran de blocage vide, app inaccessible | Règle §4 : lecture d'un JSON pré-calculé, jamais de calcul ni de réseau |
| Tokens d'app opaques | Pas de personnalisation par app | Laisser l'utilisateur nommer lui-même chaque token choisi |
| L'utilisateur contourne (désactive l'automatisation, désinstalle) | Feature inefficace | Assumé : c'est un outil pour soi, pas un contrôle parental |
| Profil de dev qui expire | App qui cesse de fonctionner | Compte payant (1 an) ; prévoir la réinstallation |
| Fatigue d'interruption | Abandon en 2 semaines | Budget d'interruptions/jour, plages horaires, mode pause, difficulté adaptative |

---

## 8. Ce que je recommande

1. **Commencer par le canal B (Raccourcis), pas par Screen Time.** Même sensation, aucune dépendance à Apple, et tu sais en 3 jours si le concept te plaît vraiment.
2. **Faire le Mac tôt.** C'est la plateforme où la feature 2 peut être exactement ce que tu décris, sans compromis.
3. **Traiter Screen Time comme une phase d'optimisation**, pas comme le fondement. Si l'entitlement traîne, le produit existe déjà.
4. **Concevoir le partage App Group dès le premier jour.** C'est le seul choix d'architecture réellement coûteux à corriger plus tard.

### Décisions qu'il me faut pour passer à l'implémentation

1. Compte Apple Developer payant (99 $/an) — oui ou non ? *(sans lui : pas de Screen Time, et réinstallation tous les 7 jours)*
2. Usage strictement personnel, ou publication App Store à terme ?
3. Quelle langue cible, et quel niveau approximatif ?
4. iPhone d'abord, ou Mac d'abord ?

---

## Sources

- [Managed Settings UI — Apple Developer Documentation](https://developer.apple.com/documentation/managedsettingsui)
- [Requesting the Family Controls entitlement — Apple Developer Documentation](https://developer.apple.com/documentation/familycontrols/requesting-the-family-controls-entitlement)
- [Open parent app from ShieldAction extension in iOS — Apple Developer Forums](https://developer.apple.com/forums/thread/719905)
- [Allow to open parent app from ShieldActionDelegate: `ShieldActionResponse.openParentApp` — Apple Developer Forums](https://developer.apple.com/forums/thread/820790)
- [Family Controls entitlement request submitted on March 9, 2026 — Apple Developer Forums](https://developer.apple.com/forums/thread/818553)
- [How to Get the Apple Family Controls Entitlement — Newly](https://newly.app/how-to/family-controls-entitlement)
- [Creating a ScreenTime ShieldConfigurationDataSource for iOS FamilyControls API — John Baker](https://medium.com/@B4k3R/creating-a-screentime-shieldconfigurationdatasource-for-ios-familycontrols-api-5ca1079d3188)
- [Setting triggers in Shortcuts on iPhone or iPad — Apple Support](https://support.apple.com/guide/shortcuts/setting-triggers-apde31e9638b/ios)
- [Intro to personal automation in Shortcuts — Apple Support](https://support.apple.com/guide/shortcuts/intro-to-personal-automation-apd690170742/ios)
- [UNNotificationContentExtension — Apple Developer Documentation](https://developer.apple.com/documentation/usernotificationsui/unnotificationcontentextension)
- [UNTextInputNotificationAction — Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/untextinputnotificationaction)
- [Meet the Foundation Models framework — WWDC25](https://developer.apple.com/videos/play/wwdc2025/286/)
- [Apple's Foundation Models framework unlocks new intelligent app experiences — Apple Newsroom](https://www.apple.com/newsroom/2025/09/apples-foundation-models-framework-unlocks-new-intelligent-app-experiences/)
