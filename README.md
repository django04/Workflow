Migration auth OTLP : API Token → Workload Identity Federation
25 sept. 2026 · @Django$
Résumé exécutif
Nous proposons de remplacer les API tokens statiques utilisés pour envoyer la télémétrie OpenTelemetry (OTLP) vers Dynatrace par la Workload Identity Federation (WIF). Chaque workload s'authentifie alors avec l'identité qu'il possède déjà : son ServiceAccount Kubernetes, ou la Managed Identity de la VM Azure.
• Gain : plus aucun secret Dynatrace longue durée stocké dans les clusters, les VM ou les pipelines, et plus de rotation manuelle.
• Sécurité de la bascule : les API tokens actuels restent actifs en fallback pendant toute la migration. Ils ne sont révoqués qu'après validation formelle de la solution, sur décision du comité.
• Séquencement : par jalons Go/No-Go, en commençant par une vérification auprès de l'éditeur sur la compatibilité avec notre ActiveGate, puis les prérequis plateforme (notamment OpenShift).
• Point d'attention : la WIF Dynatrace est aujourd'hui en statut Preview. D'où une démarche pilote, puis une généralisation progressive avec rollback immédiat possible.
Décision attendue : valider le lancement du pilote hors production et la mobilisation des équipes CaaS et Cloud Azure.
Contexte et bénéfices
Aujourd'hui, les OTel Collectors s'authentifient auprès de Dynatrace avec des API tokens longue durée. Ces tokens sont stockés dans des Secrets Kubernetes ou dans des fichiers de configuration sur les VM. Ils sont partagés, rarement renouvelés, et leur fuite donnerait un accès d'ingestion jusqu'à leur révocation manuelle.
Avec la WIF, le workload présente un jeton court (1 h en Kubernetes), émis et renouvelé automatiquement par son propre fournisseur d'identité. Dynatrace le vérifie à chaque requête et l'associe à un compte de service aux droits limités à l'ingestion.
Axe
Aujourd'hui (API token)
Cible (WIF)
Secret stocké
Oui, longue durée
Aucun
Rotation
Manuelle
Automatique (jeton court)
Traçabilité
Token partagé entre workloads
Identité par cluster / VM
Révocation en incident
Retrouver et révoquer chaque token
Désactivation en un appel par émetteur
Droits
Scopes du token
Scopes d'ingestion uniquement, par compte de service
La migration répond aussi aux attentes de conformité : pas de secret en clair dans les manifestes ni les images, et le principe du moindre privilège.
Périmètre
Le périmètre couvre les flux OTLP (métriques, et logs/traces lorsqu'ils existent) envoyés vers le tenant Dynatrace par les OTel Collectors.
Inclus
• Clusters Kubernetes : AKS, EKS, OpenShift
• VM Azure équipées d'un OTel Collector
• Environnements hors production (pilote) puis production
Exclus
• Authentification des OneAgent et de l'Operator Dynatrace (mécanisme distinct)
• Autres usages d'API tokens (automatisation, Monaco, intégrations)
• Applications qui exportent directement depuis leur SDK sans Collector : elles seront d'abord raccordées à un Collector, dans un lot séparé si nécessaire
Routage via ActiveGate : le trafic OTLP transite aujourd'hui par un ActiveGate. La documentation WIF ne montre que l'export direct vers l'endpoint SaaS, et la documentation OTLP ne décrit, pour l'ActiveGate, que l'authentification par token. La compatibilité WIF à travers l'ActiveGate doit donc être confirmée par l'éditeur avant le pilote (prérequis du jalon J2).
Principe de coexistence : l'API token reste le filet de sécurité
Aucun API token actuel n'est révoqué avant la validation formelle de la solution WIF au jalon J5. Jusque-là, chaque périmètre peut revenir à l'API token en moins de 15 minutes.
Concrètement
• Token conservé actif : le token reste valide côté Dynatrace et stocké à son emplacement actuel (Secret Kubernetes, coffre ou fichier VM), mais n'est plus utilisé par le Collector une fois basculé.
• Rollback par configuration : la configuration du Collector garde les deux modes, sélectionnables par un paramètre (valeur Helm/GitOps côté Kubernetes, variable côté VM). Revenir en arrière consiste à rebasculer ce paramètre et redéployer, sans intervention Dynatrace.
• Surveillance du fallback : la date de dernière utilisation de chaque token est suivie. Un token censé être inactif qui redevient utilisé signale un rollback ou un workload oublié.
• Durcissement pendant la coexistence : chaque token conservé est réduit aux seuls scopes d'ingestion et reçoit une date d'expiration alignée sur la fin prévue du projet, plus une marge.
Condition de décommissionnement : révocation uniquement après le Go du comité au jalon J5, puis au terme d'une période d'observation sans rollback (voir la section Critères).
Planning et jalons
La migration se déroule en sept phases, rythmées par cinq jalons Go/No-Go. Les dates seront fixées au jalon J1.
Phase
Contenu
Jalon de sortie
Cadrage
Inventaire des API tokens et de leurs consommateurs, validation sécurité du statut Preview, conventions
J1 – Go lancement
Check éditeur – ActiveGate
Ticket support Dynatrace : confirmation écrite de la compatibilité WIF pour l'ingestion OTLP routée via ActiveGate (conditions, version d'ActiveGate requise). En cas de réponse négative : arbitrage management (export direct vers le SaaS ou report)
Réponse éditeur, prérequis de J2
Socle Dynatrace
Service users dédiés, trust policies et service user mappings versionnées, procédures de rollback
—
Prérequis plateformes
Émetteur OIDC par cluster (OpenShift = chantier le plus long), identité Entra et Managed Identities des VM
J2 – Prérequis prêts
Pilote hors production
1 cluster + 1 lot de VM en WIF via l'ActiveGate (ou en direct selon la réponse éditeur), API token en fallback, test de rollback réel
J3 – Pilote validé
Généralisation
Tout le hors production, puis la production par vagues (cluster par cluster, VM par lot)
J4 – 100 % des flux en WIF
Observation et décommission
2 semaines minimum sans rollback, puis révocation progressive des API tokens
J5 – Validation finale, autorisation de décommission
Chemin critique : la réponse de l'éditeur sur l'ActiveGate, qui conditionne l'architecture cible, et l'exposition publique de l'émetteur OIDC des clusters OpenShift. Si J2 n'est pas tenu pour OpenShift, les clusters AKS/EKS et les VM avancent seuls et OpenShift rejoint une vague ultérieure.
Organisation et responsabilités
L'administrateur Dynatrace porte le projet, avec le support de l'éditeur Dynatrace sur la fonctionnalité. Les équipes CaaS et Cloud Azure réalisent les changements sur leurs plateformes.
Légende : R = réalise, A = responsable final, C = consulté, I = informé.
Activité
Admin Dynatrace
Éditeur Dynatrace
Équipe CaaS (K8s)
Équipe Cloud Azure
Sécurité
Management
Cadrage, inventaire des tokens
A/R
—
C
C
C
I
Validation du statut Preview
R
C
I
I
A
C
Confirmation de la compatibilité WIF via ActiveGate
A
R
I
I
I
I
Configuration Dynatrace (trust policies, service user mappings, service users)
A/R
C
C
C
C
I
Émetteur OIDC des clusters
C
—
A/R
—
C
I
Identité Entra et Managed Identities
C
—
—
A/R
C
I
Bascule des Collectors et rollback
C
—
A/R (K8s)
A/R (VM)
I
I
Support et escalade des incidents produit WIF
A
R
C
C
I
I
Décisions Go/No-Go (J1 à J5)
R
I
C
C
C
A
Révocation des API tokens
R
—
C
C
C
A
Risques et mitigations
Le risque principal est une perte de télémétrie pendant la bascule. Il est couvert par le maintien de l'API token en fallback et par un déploiement par vagues.
Risque
Impact
Probabilité
Mitigation
Fonctionnalité WIF en Preview (évolution, incident éditeur)
Élevé
Moyenne
Validation sécurité, pilote hors prod, API token conservé jusqu'à J5
WIF non supportée à travers l'ActiveGate (non documenté à ce jour)
Élevé
À confirmer
Confirmation éditeur avant J2 ; sinon arbitrage : export direct vers le SaaS (ouverture réseau) ou report de la migration
Perte de télémétrie lors d'une bascule
Élevé
Faible
Vagues successives, contrôle des volumes avant/après, rollback < 15 min
Émetteur OIDC OpenShift non exposable publiquement (exigence Dynatrace : HTTPS, adresse non privée)
Moyen
Moyenne
Chantier plateforme lancé dès le cadrage ; OpenShift en vague séparée si besoin
Erreurs d'authentification peu explicites (erreur 401 générique)
Moyen
Moyenne
Runbook de diagnostic, alerte sur les échecs d'export des Collectors
Workloads oubliés à l'inventaire
Moyen
Moyenne
Suivi de la dernière utilisation de chaque API token avant révocation
Décommission trop précoce
Élevé
Faible
Révocation seulement après Go J5 et 2 semaines sans rollback
Critères de validation et de décommission
La solution est déclarée validée au jalon J5 uniquement si tous les critères ci-dessous sont remplis. La révocation des API tokens se fait ensuite, périmètre par périmètre.
Validation du pilote (J3)
• Ingestion continue en WIF pendant au moins 5 jours ouvrés, sans écart significatif de volume par rapport à la période sous API token
• Zéro erreur d'authentification persistante côté Collector
• Renouvellement automatique des jetons constaté, sans redémarrage
• Rollback vers l'API token testé réellement et chronométré (< 15 min)
Validation finale (J5)
• 100 % des flux du périmètre en WIF
• 2 semaines consécutives sans rollback ni incident d'ingestion lié à l'authentification
• Aucune utilisation récente des API tokens concernés
• Runbook d'exploitation remis aux équipes (diagnostic, désactivation d'urgence, ajout d'un cluster ou d'une VM)
• Avis favorable de la Sécurité, notamment sur le statut de la fonctionnalité chez l'éditeur
Décommission (après J5)
1. Révocation des tokens d'un premier périmètre, puis 48 h d'observation
2. Révocation des périmètres suivants, vague par vague
3. Suppression des Secrets, fichiers et entrées de coffre associés
4. Clôture du projet et bilan
Décisions attendues et prochaines étapes
Cinq décisions sont nécessaires pour lancer le projet.
1. Valider le lancement du pilote hors production malgré le statut Preview, sous réserve de l'avis Sécurité.
2. Valider le principe de coexistence : API tokens maintenus actifs en fallback jusqu'au jalon J5.
3. Mobiliser les équipes CaaS et Cloud Azure selon le RACI, avec un référent nommé par équipe.
4. Arbitrer le traitement d'OpenShift : vague commune ou vague séparée si l'émetteur OIDC public n'est pas prêt à J2.
5. Si l'éditeur ne confirme pas la WIF via ActiveGate : choisir entre un export direct vers le SaaS (ouverture réseau à valider) et le report de la migration.
Prochaines étapes après accord
[ ] Fixer la date de lancement (S1) et les dates des jalons
[ ] Lancer l'inventaire des API tokens d'ingestion et de leurs consommateurs
[ ] Obtenir l'avis Sécurité sur l'usage d'une fonctionnalité Preview
[ ] Identifier le cluster et le lot de VM du pilote
Sources
• Workload identity federation — Dynatrace Docs
• Workload identity federation with the OTel Collector — Dynatrace Docs
• Dynatrace OTLP API endpoints — Dynatrace DocsWorkflow 
