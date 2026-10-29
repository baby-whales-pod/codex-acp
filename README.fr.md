# Codex ACP dans une sandbox SBX

Ce projet exécute Codex et son serveur ACP dans une sandbox Docker Sandboxes.
Zed, AionUI ou tout autre client ACP reste sur l'hôte et lance le serveur stdio
avec `sbx exec -i`. Aucun port n'est publié : ACP utilise JSON-RPC sur stdin et
stdout.

## Contenu

```text
codex-acp/
├── sbxenv.yaml
├── bin/codex-acp-sbx
├── configs/zed-settings.jsonc
├── kits/codex-acp/codex-acp.yaml
└── workspace/
```

`sbxenv.yaml` compose le kit Codex officiel épinglé et le mixin SBX v3 local.
Le mixin installe `@agentclientprotocol/codex-acp@2.3.0`, qui inclut une version
compatible du CLI Codex. Le wrapper hôte conserve stdin avec `-i` et n'alloue
pas de pseudo-terminal, car un TTY corromprait le flux JSON-RPC ACP.

## Prérequis

- Docker Desktop avec Docker Sandboxes et la commande `sbx` ;
- une session `sbx login` valide ;
- Zed ou AionUI installé sur l'hôte ;
- un abonnement ChatGPT compatible ou une clé API OpenAI.

Les commandes suivantes partent de ce dossier :

```sh
cd docker-demos/codex-acp
```

## 1. Authentifier OpenAI

La méthode recommandée avec un abonnement ChatGPT est le stockage OAuth de
SBX :

```sh
sbx secret set openai --oauth
```

Pour une clé API, laissez SBX la demander sans l'écrire dans l'historique du
shell :

```sh
sbx secret set openai
```

Le secret reste dans le credential store de SBX. Le kit Codex fournit au
processus la variable proxy-gérée attendue ; la vraie valeur n'est pas copiée
dans ce dépôt. `NO_BROWSER=1` masque volontairement le login navigateur depuis
la sandbox, puisque l'authentification se fait sur l'hôte.

## 2. Créer l'environnement

```sh
sbx env create
```

La commande affiche son plan et demande confirmation. Pour une exécution non
interactive :

```sh
sbx env create --force
```

Vérifiez ensuite les binaires :

```sh
sbx exec codex-acp npm list --global --depth=0 @agentclientprotocol/codex-acp@2.3.0
sbx exec codex-acp command -v codex-acp
sbx exec codex-acp codex --version
```

## 3. Vérifier le transport ACP

Le serveur ACP attend un client sur stdin ; il est donc normal que cette
commande reste ouverte sans afficher d'interface :

```sh
./bin/codex-acp-sbx
```

Interrompez ce test avec `Ctrl-C`. Pour diagnostiquer la sandbox :

```sh
sbx exec codex-acp sh -lc 'id; pwd; command -v codex-acp; command -v codex'
sbx policy log
sbx exec codex-acp sh -lc 'ls -la /home/agent/.codex/logs'
```

## Zed

Zed doit lancer `codex-acp` dans la sandbox avec `sbx exec`, et non utiliser
son Codex local du registre ACP.

1. Récupérez le chemin absolu du workspace :

   ```sh
   realpath ./workspace
   ```

2. Dans Zed, lancez `agent: open settings`, ouvrez **External Agents**, puis
   **Add Agent > Add Custom Agent**.
3. Reprenez l'entrée de `configs/zed-settings.jsonc` dans le `settings.json` de
   Zed. Remplacez le chemin `/ABSOLUTE/PATH/TO/...` par le résultat de l'étape
   précédente.
4. Ouvrez le dossier `workspace` dans Zed.
5. Dans l'Agent Panel, choisissez **Codex SBX** et créez un thread.

Configuration complète :

```json
{
  "agent_servers": {
    "Codex SBX": {
      "type": "custom",
      "command": "sbx",
      "args": [
        "exec",
        "-i",
        "-w",
        "/chemin/absolu/codex-acp/workspace",
        "codex-acp",
        "codex-acp"
      ],
      "env": {}
    }
  }
}
```

La commande correspond exactement à :

```sh
sbx exec -i -w /chemin/absolu/codex-acp/workspace codex-acp codex-acp
```

La première occurrence de `codex-acp` est le nom de la sandbox ; la seconde est
la commande exécutée dedans. `-i` conserve stdin ouvert pour ACP. N'ajoutez pas
`-t` : un pseudo-terminal perturberait le flux JSON-RPC.

Le wrapper `bin/codex-acp-sbx` reste disponible si vous préférez centraliser le
nom et le workspace :

```json
{
  "agent_servers": {
    "Codex SBX (wrapper)": {
      "type": "custom",
      "command": "/chemin/absolu/codex-acp/bin/codex-acp-sbx",
      "args": [],
      "env": {
        "CODEX_ACP_SANDBOX": "codex-acp",
        "CODEX_ACP_WORKDIR": "/chemin/absolu/codex-acp/workspace"
      }
    }
  }
}
```

En cas d'échec, utilisez `dev: open acp logs` dans la palette Zed. Si Zed a été
lancé graphiquement et ne trouve pas `sbx`, utilisez son chemin absolu comme
valeur de `command` ou employez le wrapper.

## AionUI

AionUI sait créer un **Custom Agent** ACP utilisant un processus stdio.

1. Ouvrez **Settings > Agents**, puis l'onglet **Custom Agents**.
2. Créez un agent et renseignez :

   - Name : `Codex SBX`
   - Connection/transport : `stdio` ou `CLI`
   - Command/CLI path : le chemin absolu retourné par
     `realpath ./bin/codex-acp-sbx`
   - Arguments : aucun
   - Environment : `CODEX_ACP_SANDBOX=codex-acp` et
     `CODEX_ACP_WORKDIR=<chemin absolu retourné par realpath ./workspace>`

3. Utilisez **Test Connection**, sauvegardez et activez l'agent.
4. Créez une conversation avec **Codex SBX** et sélectionnez `workspace` comme
   espace de travail.

Selon la version d'AionUI, les champs peuvent être présentés dans un éditeur
JSON avancé. Le contrat reste identique : exécutable = wrapper, arguments =
tableau vide, environnement = les deux variables ci-dessus, transport = stdio.

## Obsidian avec le plugin Agent Client

Le fichier `sbxenv.obsidian.yaml` permet d'utiliser un vault Obsidian comme
workspace de la sandbox. Obsidian et le plugin restent sur l'hôte ; Agent
Client lance le wrapper, qui démarre `codex-acp` dans la sandbox via stdio.

1. Dans `sbxenv.obsidian.yaml`, remplacez la valeur de `workspace:` par le
   chemin absolu de votre vault. Le chemin ne doit pas pointer vers le seul
   dossier `.obsidian`, mais vers la racine du vault :

   ```yaml
   workspace: /chemin/absolu/vers/mon-vault
   ```

2. Créez l'environnement dédié après avoir configuré le secret OpenAI comme
   expliqué plus haut :

   ```sh
   sbx env create sbxenv.obsidian.yaml
   ```

   Cette configuration crée une sandbox nommée `codex-acp-obsidian`.

3. Dans Obsidian, ouvrez **Settings > Community Plugins > Browse**, recherchez
   **Agent Client**, puis installez et activez le plugin.
4. Ouvrez **Settings > Agent Client**, descendez jusqu'à **Custom Agents**, puis
   choisissez **Add custom agent** et renseignez :

   - Agent ID : `codex-sbx`
   - Display name : `Codex SBX`
   - Path : le chemin absolu retourné par
     `realpath ./bin/codex-acp-sbx`
   - Arguments : aucun
   - Environment variables, une par ligne :

     ```text
     CODEX_ACP_SANDBOX=codex-acp-obsidian
     CODEX_ACP_WORKDIR=/chemin/absolu/vers/mon-vault
     ```

5. Cliquez sur l'icône robot dans le ruban Obsidian, sélectionnez **Codex SBX**
   dans la liste des agents et démarrez une conversation. Vous pouvez
   mentionner des notes avec `@` ; Codex peut lire et modifier les fichiers du
   vault monté comme workspace.

Le plugin doit utiliser l'agent personnalisé ci-dessus, et non son preset
Codex local. L'authentification reste gérée par le secret SBX : ne renseignez
pas de clé API dans Agent Client. Gardez la validation des permissions activée
et vérifiez les modifications proposées avant de les accepter.

## Utiliser un autre projet

Modifiez ensemble :

1. `workspace:` dans `sbxenv.yaml` ;
2. `CODEX_ACP_WORKDIR` dans Zed ou AionUI.

Puis recréez l'environnement :

```sh
sbx env rm
sbx env create
```

Vous pouvez aussi conserver ce fichier et remplacer le contenu de `workspace/`.

## Cycle de vie

```sh
sbx env create              # créer
sbx exec -it codex-acp sh   # shell de diagnostic seulement
sbx env rm                  # supprimer la sandbox
```

N'ajoutez jamais `-t` au lancement ACP. Ne placez jamais une clé dans
`sbxenv.yaml`, la configuration Zed/AionUI ou le wrapper.

## Références

- [Agent Client Protocol](https://agentclientprotocol.com/get-started/introduction)
- [Codex ACP](https://github.com/agentclientprotocol/codex-acp)
- [Agents externes Zed](https://zed.dev/docs/ai/external-agents)
- [Documentation officielle du CLI Codex](https://developers.openai.com/codex/cli/)
- [AionUI](https://www.aionui.com/en)
- [Plugin Agent Client pour Obsidian](https://github.com/RAIT-09/obsidian-agent-client)
