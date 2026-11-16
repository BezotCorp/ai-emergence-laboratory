# Laboratoire IA multimodal — conception technique V0

**Statut :** cahier de conception expérimental — 16 novembre 2026
**Projet :** expérience locale et potentiellement diffusée sur Twitch, indépendante du SaaS commercial BezotCorp
**Principe :** observer un système de modèles spécialisés et de mémoires évolutives, sans écrire à l'avance ses répliques ni lui attribuer une personnalité prédéfinie.

## 1. Intention

Une personne interagit avec le laboratoire par du **texte, de la voix, des images ou de la vidéo**. Elle peut changer d'interlocuteur simulé (Rémi, Fred, Sophie, etc.). Les composants conservent des expériences, se transmettent des informations et influencent les productions futures. Le système peut émettre **du texte, une voix/un son, une image, ou rien**, y compris initier une sortie sans nouvelle demande humaine.

On cherche à **observer** les comportements, pas à obtenir un assistant fiable, performant ou poli. Une confusion, un oubli, un silence ou une intervention inopportune peuvent être des résultats intéressants. Aucun comportement spectaculaire n'est garanti, et une phrase telle que « Rémi me manque » ne constituerait pas une preuve de sentiment ou de conscience.

**Limites explicites de la V0 :** le modèle d'origine de chaque modalité reste inchangé. Les composants mémoriels évoluent ; la modification des poids des modèles de perception/génération est une expérience ultérieure, pas un préalable. Le laboratoire ne doit pas avoir accès librement au dépôt Git, aux commandes système ou à des comptes tiers.

## 2. Plateforme et diffusion

**Choix retenu : application de bureau Rust + Dioxus, exécutée localement sur Kubuntu.**

- Dioxus sert à l'interface d'observation et de contrôle ; le traitement reste dans les services Rust natifs. Son rendu desktop s'appuie sur une WebView, sans exiger de serveur web public.
- Ollama sert de moteur local aux modèles qu'il peut effectivement exécuter ; les moteurs audio, génération d'images, apprentissage et mémoire peuvent être des processus ou bibliothèques distincts si Ollama ne couvre pas leurs fonctions.
- OBS capture la fenêtre du laboratoire et les flux audio de l'application et du microphone sur des voies distinctes ; **ne pas réinjecter le son des haut-parleurs dans le microphone** comme solution d'architecture.
- Les spectateurs observent le comportement : ils n'ont pas besoin d'une application web dans la V0. Une interface web reste une possibilité future, non retenue ici.

**Matériel de référence :** RTX 5060 Ti 16 Go de VRAM, 32 Go de RAM. Les tailles ci-dessous sont des **cibles exploratoires**, non des garanties de disponibilité ou de cohabitation en mémoire.

## 3. Topologie expérimentale

```text
              [Chat / microphone / image / vidéo]
                              |
                   [Acquisition / événements]
                              |
            +-----------------+-----------------+
            |                 |                 |
      [Modèle NLP]      [Modèle vocal]     [Modèle visuel]
       ~1 Go cible        ~1 Go cible        ~1 Go cible
            |                 |                 |
      [Mémoire NLP]     [Mémoire vocale]   [Mémoire visuelle]
      ~500 Mo cible      ~500 Mo cible      ~500 Mo cible
            \                 |                /
             +---- [Mémoire / liens transversaux] ----+
                         ~1 Go cible
                              |
                  [Sélection / arbitration]
                       poids ajustables
                              |
                   [Texte | son | image | rien]
                              |
                  [Observation + rétroaction]
                              +------> mémoires / liens
```

Ce dessin représente des **responsabilités et des flux**, pas l'obligation que chaque composant soit un LLM. Les modèles visuel et vocal doivent réellement supporter leur modalité. Une représentation interne n'est pas forcément du texte, un fichier image ou un fichier son.

### Modèles de base

- **NLP** : comprend et produit du langage. Le modèle chargé depuis Ollama n'est pas modifié.
- **Voix / audio** : reçoit des signaux audio et produit des représentations ; une réponse audible exige une véritable capacité de synthèse vocale ou de génération audio. La reconnaissance vocale et la synthèse vocale ne sont pas automatiquement la même capacité.
- **Vision** : reçoit des images ou des images extraites d'une vidéo ; produire une image exige un générateur d'images, distinct d'un modèle qui se contente de comprendre des images.
- Pas de *persona* programmé dans un prompt système. Contrôler toutefois les instructions et gabarits de conversation éventuellement embarqués dans les modèles et moteurs : « aucun prompt personnalisé » ne veut pas dire « aucune structure technique d'entrée ».

### Mémoires propres à chaque modalité

Chaque modalité possède un **système mémoriel entraînable / réorganisable** dont la représentation lui appartient. Les ~500 Mo expriment une enveloppe envisagée, **pas une quantité de souvenirs garantie**. La nature du composant mémoriel reste à choisir par expérimentation (poids entraînables, embeddings, connexions associatives, stockage récupérable, ou combinaison). Ne pas présenter un simple historique avec recherche comme un modèle continuellement fine-tuné.

### Mémoire transversale

- Relie des événements, observations et associations de plusieurs modalités.
- Permet qu'un stimulus visuel réactive des informations linguistiques ou acoustiques, ou inversement.
- Ne présuppose pas un format unique pour les données visuelles, vocales et textuelles.
- Conserve la provenance des observations et une distinction entre **observation**, **interprétation**, **hypothèse** et **résultat d'apprentissage**, afin de pouvoir voir ce qui dérive.
- Ne partage pas aveuglément des données privées avec un autre interlocuteur, même dans l'hypothèse d'une mémoire associative commune.

### Arbitrage et expression

Le mécanisme qui choisit **si**, **quand**, **comment** et **à qui** répondre peut comporter des poids adaptatifs. Ce n'est pas nécessairement un quatrième LLM. Il peut sélectionner ou combiner du texte, du son, une image, plusieurs sorties, ou le silence. La possibilité d'initiative et d'interruption existe dans la conception ; elle ne doit pas se réduire à une phrase déclenchée aléatoirement à intervalles réguliers.

**Point technique non résolu :** un poids ne « s'ajuste » pas tout seul. Il faut concevoir et documenter les signaux qui le modifient (écarts entre prédictions et observations, cohérence entre modalités, résultats de l'interaction, etc.). L'absence d'une récompense humaine explicite n'est pas l'absence de règles d'apprentissage. Le choix de ces règles fait partie de l'expérience, pas d'une décision déjà validée.

## 4. Interlocuteurs et identité

- L'interface permet de choisir l'émetteur d'un message parmi plusieurs personnages **fictifs** et le participant réel volontaire.
- Tous les échanges peuvent alimenter une mémoire associative commune, **sans imposer des silos cognitifs étanches**.
- Chaque événement conserve l'identité de son émetteur et la nature réelle ou simulée de sa source. Cela permet de mesurer les confusions entre personnes, sans les programmer.
- La présence d'un personnage fictif ne doit pas être assimilée à une personne réellement entendue ou observée.
- Pour un live public, seules les données personnelles explicitement autorisées sont utilisées. Les conversations et souvenirs privés ne sont pas automatiquement diffusés.

## 5. Chaîne de traitement d'un événement

1. **Acquérir** : message de chat, segment audio, image ou trame vidéo horodatée.
2. **Attribuer une provenance** : source, interlocuteur déclaré, modalité, identifiant de l'événement.
3. **Percevoir** : chaque modèle compétent traite sa modalité et fournit un résultat ou une représentation, sans imposer une conversion globale en texte.
4. **Associer** : mémoire propre puis mémoire transversale relient l'événement aux expériences précédentes.
5. **Réinjecter** : les représentations utiles sont transformées en entrées compréhensibles par le ou les modèles destinataires. Pour un LLM texte, il faut une sérialisation compatible avec son API ; une représentation visuelle opaque ne se transforme pas magiquement en information compréhensible.
6. **Évaluer l'expression** : arbitrage adaptatif ; possibilité d'initiative, interruption ou silence.
7. **Produire** : texte, synthèse/génération audio, image, ou absence de sortie.
8. **Observer et apprendre** : enregistrer les signaux réellement disponibles, mettre à jour les mémoires/composants entraînables et rendre leur activité visible.

Le traitement peut continuer à partir d'événements internes, sans message humain supplémentaire. Des limites de ressources et un arrêt d'urgence restent nécessaires pour éviter les boucles infinies ou l'emballement d'émissions audio.

## 6. Interface Twitch / laboratoire

L'interface n'est pas une conversation textuelle déguisée. Elle expose **ce que le système reçoit, active et produit** :

- Zone de discussion, sélection de l'interlocuteur, bouton d'envoi de message.
- Entrées locales : micro, image, caméra/vidéo ; activation explicite pour chaque source.
- Sorties : texte défilant, lecture audio, galerie d'images produites, silence visible comme absence de sortie.
- Vue des trois modalités et de leurs mémoires ; activité, liens sollicités, mise à jour, latences et erreurs réelles.
- Vue du système transversal : associations activées, source des informations et choix d'arbitrage lorsqu'ils sont observables.
- Contrôles d'expérience : activer/couper l'accès aux mémoires, suspendre les mises à jour, effacer une session de simulation **sur commande explicite**, arrêter le système.
- Affichage des états réels : « modèle indisponible », « génération impossible avec ce modèle », « mémoire inaccessible » ; **ne pas fabriquer des pensées internes ou une explication causale que le système ne fournit pas**.

La coupure d'une mémoire permet de comparer le comportement avec/sans cette mémoire. Une perte d'accès n'implique pas que ses données ont été effacées.

## 7. Contraintes de faisabilité à ne pas masquer

| Sujet | Réalité technique |
| --- | --- |
| Modèle ~1 Go | Taille visée ; capacité, qualité, format et vitesse à mesurer sur les vrais modèles |
| Trois « cerveaux » ~500 Mo | Pas de preuve qu'une mémoire entraînable utile puisse être obtenue par cette seule taille |
| Intermodalité | Les sorties d'un modèle ne sont pas automatiquement interprétables par un autre ; adaptateurs/liaisons nécessaires |
| Vidéo | Pour la V0, traitement de trames échantillonnées et horodatées plutôt qu'une compréhension vidéo continue garantie |
| Audio | Micro, reconnaissance et génération vocale demandent des capacités distinctes ; temps réel à vérifier |
| Image produite | Nécessite un modèle ou moteur de génération d'images ; un VLM d'analyse ne suffit pas |
| Apprentissage continu | Demande une règle de mise à jour et peut dégrader les mémoires ; le comportement est un résultat expérimental |
| VRAM / RAM | Prévoir chargement à la demande et mesurer le pic d'usage, pas seulement additionner la taille des fichiers modèles |
| « Pensée » observable | Une image, un texte ou un score affiché est une sortie/trace, pas une preuve d'expérience intérieure |
| Twitch | Pas de publication directe des souvenirs privés ; autorisations des sources audio/vidéo et contrôle des sorties |

## 8. Parcours de réalisation, sans transformer le loisir en chantier de plusieurs années

Ce découpage sert à **produire une expérience observable rapidement**, pas à limiter le concept définitif.

**Essai A — continuité comportementale.** Application desktop, un modèle NLP Ollama immuable, deux interlocuteurs fictifs et Rémi, une mémoire évolutive accessible/inaccessible à la demande, historique et traces d'injection visibles. Vérifier si une expérience antérieure influe sur une réponse ultérieure sans écrire les réponses attendues.

**Essai B — modalités supplémentaires.** Ajouter acquisition d'images, puis entrée audio et sortie audio avec moteurs réellement capables. Chaque modalité possède sa mémoire et peut réactiver des associations transversales.

**Essai C — sorties libres.** Introduire génération d'image et arbitrage adaptatif texte/son/image/silence, puis événements internes et interruptions spontanées. Observer et comparer les trajectoires.

**Essai D — débordements expérimentaux.** Tester les apprentissages entre modalités, les dérives, l'oubli, les défauts de reconnaissance et les coupures mémoire. La modification des poids des modèles fondateurs n'appartient pas à la V0 ; une copie expérimentale entraînable pourrait être envisagée ultérieurement tout en préservant l'original.

Ces essais ne supposent ni durée fixe, ni garantie de réponse particulière. On peut s'arrêter dès que l'expérience est amusante et instructive.

## 9. Persistance, observation et reproductibilité

- Les modèles d'origine sont **immuables** ; enregistrer leur nom, leur variante et leur empreinte pour identifier précisément la référence utilisée.
- Les mémoires évolutives conservent leurs propres états et métadonnées ; les instantanés et restaurations, s'ils sont ajoutés, sont explicites.
- Un journal d'événements horodatés conserve les entrées, sorties, mises à jour observables et erreurs ; il n'est pas présenté comme un accès à une « pensée » cachée.
- Les données propres à l'expérience, notamment les voix, images et souvenirs personnels, ne doivent pas être publiées automatiquement avec le code source.
- Éviter les écritures inutiles sur SSD : conserver les traitements transitoires en mémoire et persister uniquement selon une politique explicite de changement, sans garantir une absence totale d'écritures.

## 10. Principes de développement

- Rust pour les responsabilités natives et Dioxus pour l'interface ; les moteurs modèles sont des composants externes ou bibliothèques selon leurs capacités réelles.
- Ne pas introduire d'architecture générique monumentale avant d'avoir une relation fonctionnelle à observer.
- Ne pas confondre configuration d'un modèle, mémoire persistante, récupération de contexte et fine-tuning : chaque mécanisme doit être identifié.
- Ne pas ajouter de personnalité, de script de dialogue ni de réplique « Rémi me manque » prédéfinie.
- Les contraintes de sécurité ne constituent pas une personnalité : pas d'exécution de commandes arbitraires ni d'accès non autorisé aux fichiers/comptes.
- Aucun choix définitif de crate d'entraînement, moteur TTS, modèle visuel ou licence open source n'est imposé dans ce document ; les choisir sur preuve de compatibilité et résultats locaux.
- Ne pas effectuer de commits ou de modifications de dépôt dans le cadre de la présente conception.

## 11. Questions réellement ouvertes

1. Quel mécanisme de représentation et de mise à jour utiliser pour chaque mémoire (poids apprenants, associations, stockage, combinaison) ?
2. Comment créer et réviser des liens intermodaux à partir d'expériences réellement observées ?
3. Quel signal pilote l'arbitrage adaptatif et sa décision de rester silencieux ou d'interrompre ?
4. Quels modèles concrets, dans les tailles visées, comprennent/génèrent effectivement voix, image et langage ?
5. Comment définir une observation mesurable de l'évolution comportementale sans confondre succès expérimental et phrase amusante ?

Ce sont des **questions de conception ou d'expérimentation**, pas des autorisations d'inventer leurs réponses pendant l'implémentation.

## 12. Références techniques (consultées le 16 novembre 2026)

- Dioxus Desktop : <https://dioxuslabs.com/learn/0.7/guides/platforms/desktop/>
- Dioxus, plateformes : <https://dioxuslabs.com/learn/0.7/guides/platforms/>
- Ollama, API : <https://docs.ollama.com/api>
- Ollama, vision : <https://github.com/ollama/ollama/blob/main/docs/capabilities/vision.mdx>
- OBS Studio sous Linux : <https://flathub.org/fr/apps/com.obsproject.Studio>
