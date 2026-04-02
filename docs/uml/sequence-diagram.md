# Diagramme de séquence — Claude Code v2.1.88

Interactions dynamiques entre les composants pour les scénarios clés.

## Cycle de vie complet d'un appel d'outil

```mermaid
sequenceDiagram
    actor User as Utilisateur
    participant REPL as REPL (main.tsx)
    participant QE as QueryEngine
    participant query as query()
    participant STE as StreamingToolExecutor
    participant Perms as Système de permissions
    participant Tool as Tool.call()
    participant API as API Claude
    participant FS as Système de fichiers

    User->>REPL: Saisie du prompt
    REPL->>QE: submitMessage(prompt, messages, options)

    QE->>query: processUserInput()
    query->>query: fetchSystemPromptParts()
    Note over query: Assembler prompt système<br/>Charger CLAUDE.md<br/>Injecter les outils actifs

    query->>FS: recordTranscript(userMessage)
    query->>query: normalizeMessagesForAPI()

    query->>API: POST /v1/messages (streaming)
    Note over API: modèle + outils + prompt système

    loop Événements de stream
        API-->>query: content_block_delta (texte)
        query-->>QE: yield SDKMessage (texte)
        QE-->>REPL: stream de texte
        REPL-->>User: Afficher le texte
    end

    API-->>query: content_block (tool_use: BashTool)
    API-->>query: message_stop (stop_reason: tool_use)

    query->>STE: execute([toolUseBlock], context)
    STE->>STE: partition(outils concurrent vs série)

    STE->>Tool: validateInput(args, context)
    alt Entrée invalide
        Tool-->>STE: ValidationResult { result: false }
        STE->>query: tool_result(erreur)
    else Entrée valide
        Tool-->>STE: ValidationResult { result: true }
    end

    STE->>Perms: runPreToolUseHooks(toolName, args)
    Note over Perms: Exécuter les scripts shell<br/>définis par l'utilisateur

    alt Hook refuse
        Perms-->>STE: decision: deny
        STE->>query: tool_result(refus de permission)
    else Hook approuve / pas de hook
        Perms-->>STE: decision: approve / passthrough
    end

    STE->>Perms: checkPermissionRules(toolName, args)
    Note over Perms: alwaysAllowRules → auto-approve<br/>alwaysDenyRules → auto-deny<br/>alwaysAskRules → forcer l'invite

    alt Pas de règle correspondante
        Perms->>REPL: showPermissionDialog(toolName, args)
        REPL->>User: Afficher l'invite de permission
        User->>REPL: Autoriser / Refuser
        REPL-->>Perms: décision utilisateur
    end

    alt Permission refusée
        Perms-->>STE: PermissionResult: denied
        STE->>query: tool_result(refus)
    else Permission accordée
        Perms-->>STE: PermissionResult: approved

        STE->>Tool: call(args, context, canUseTool, parentMsg, onProgress)
        Note over Tool: Exécuter la commande shell<br/>Streamer la sortie

        loop Progression de l'outil
            Tool-->>STE: onProgress(BashProgress)
            STE-->>REPL: setToolJSX(progressUI)
            REPL-->>User: Afficher la progression
        end

        Tool-->>STE: ToolResult { data: { stdout, stderr, exitCode } }
        STE->>query: tool_results[]
    end

    query->>FS: recordTranscript(assistantMessage + tool_results)
    query->>query: normalizeMessagesForAPI()
    Note over query: Reboucler vers l'API<br/>avec les résultats d'outils

    query->>API: POST /v1/messages (avec tool_results)
    API-->>query: Réponse finale (stop_reason: end_turn)
    query-->>QE: yield SDKMessage final
    QE-->>REPL: résultat final
    REPL-->>User: Afficher la réponse finale
```

---

## Démarrage et initialisation de la session

```mermaid
sequenceDiagram
    actor User as Utilisateur
    participant CLI as cli.tsx
    participant main as main.tsx
    participant Config as Config / Auth
    participant MCP as MCPConnectionManager
    participant GB as GrowthBook
    participant REPL as REPL.tsx

    User->>CLI: claude [args]
    CLI->>CLI: profileCheckpoint('main_tsx_entry')
    CLI->>CLI: startMdmRawRead() [en parallèle]
    CLI->>CLI: startKeychainPrefetch() [en parallèle]

    CLI->>main: init(options)
    main->>Config: getGlobalConfig()
    main->>Config: checkHasTrustDialogAccepted()

    alt Première utilisation
        main->>REPL: afficher le dialogue de confiance
        User->>REPL: accepter
        main->>Config: saveGlobalConfig()
    end

    main->>Config: applySafeConfigEnvironmentVariables()
    main->>GB: initializeGrowthBook()
    GB-->>main: feature flags chargés

    par Initialisations parallèles
        main->>Config: prefetchAwsCredentials()
    and
        main->>MCP: prefetchOfficialMcpUrls()
    and
        main->>main: prefetchFastModeStatus()
    end

    main->>MCP: connecter les serveurs MCP (depuis settings.json)

    loop Chaque serveur MCP configuré
        MCP->>MCP: spawner/connecter le transport
        MCP->>MCP: lister les outils disponibles
        MCP-->>main: outils MCP enregistrés
    end

    main->>REPL: launchRepl(options, tools, commands)
    REPL-->>User: Afficher l'interface du terminal
```

---

## Protocole de communication inter-agents

```mermaid
sequenceDiagram
    participant Lead as Agent coordinateur
    participant TaskBoard as TaskBoard (fichier)
    participant TeammateA as Coéquipier A
    participant TeammateB as Coéquipier B

    Lead->>TaskBoard: TaskCreateTool(title, description)
    Note over TaskBoard: Écrire task_1.json sur disque

    Lead->>TeammateA: TeamCreateTool(agentType)
    Lead->>TeammateB: TeamCreateTool(agentType)

    par Agents autonomes
        TeammateA->>TaskBoard: TaskListTool(status: pending)
        TaskBoard-->>TeammateA: [task_1, task_2, ...]
        TeammateA->>TaskBoard: TaskUpdateTool(task_1, status: in_progress)
        Note over TeammateA: Revendiquer task_1
    and
        TeammateB->>TaskBoard: TaskListTool(status: pending)
        TaskBoard-->>TeammateB: [task_2, ...]
        TeammateB->>TaskBoard: TaskUpdateTool(task_2, status: in_progress)
        Note over TeammateB: Revendiquer task_2
    end

    TeammateA->>TeammateA: Exécuter task_1 (BashTool, FileEdit, etc.)
    TeammateA->>Lead: SendMessageTool(message: "task_1 terminée")
    Lead-->>TeammateA: SendMessageTool(message: "bien reçu")
    TeammateA->>TaskBoard: TaskUpdateTool(task_1, status: completed)

    TeammateB->>TeammateB: Exécuter task_2
    TeammateB->>TaskBoard: TaskUpdateTool(task_2, status: completed)

    Lead->>TaskBoard: TaskListTool(status: pending)
    TaskBoard-->>Lead: [] (liste vide)
    Lead->>TeammateA: TeamDeleteTool()
    Lead->>TeammateB: TeamDeleteTool()
    Note over Lead: Mission accomplie
```

---

## Flux d'authentification OAuth MCP

```mermaid
sequenceDiagram
    participant Tool as MCPTool
    participant MCP as MCPConnectionManager
    participant Auth as McpAuthTool / OAuth
    participant Server as Serveur MCP distant
    participant User as Utilisateur (UI)

    Tool->>MCP: appeler l'outil mcp__server__action
    MCP->>Server: Requête outil (transport HTTP)

    alt Serveur requiert l'auth
        Server-->>MCP: Erreur 401 / défi OAuth
        MCP->>Auth: démarrer le flux OAuth
        Auth->>User: Afficher l'URL d'autorisation
        User->>Auth: Visiter l'URL + autoriser
        Auth->>Auth: Recevoir le code OAuth
        Auth->>Server: Échanger le code contre un token
        Server-->>Auth: access_token + refresh_token
        Auth->>MCP: stocker les tokens dans le keychain
    else Token valide en cache
        MCP->>MCP: charger le token depuis le keychain
    end

    MCP->>Server: Requête outil + Bearer token
    Server-->>MCP: Résultat de l'outil
    MCP-->>Tool: ToolResult
```

---

## Flux de compaction du contexte

```mermaid
sequenceDiagram
    participant query as query()
    participant compact as services/compact/
    participant API as API Claude
    participant FS as Système de fichiers

    query->>query: calculateTokenWarningState(messages)
    Note over query: Vérifier si le seuil est dépassé

    alt Compaction nécessaire
        query->>compact: buildCompactMessages(messages)
        compact->>compact: getMessagesAfterCompactBoundary()
        Note over compact: Diviser : anciens / récents

        compact->>API: POST /v1/messages\n(prompt de résumé + anciens messages)
        API-->>compact: résumé en texte brut

        compact->>compact: Reconstruire messages[]\n[résumé] + [compact_boundary] + [récents]
        compact-->>query: messages[] compactés

        query->>FS: recordTranscript(compact_boundary_marker)
    end

    query->>query: normalizeMessagesForAPI(messages)
    Note over query: Reprendre la boucle agent normale
```
