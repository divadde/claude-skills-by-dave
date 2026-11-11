# claude-skills-by-dave

Marketplace personale di plugin per Claude Code. Ogni plugin sta in `plugins/<nome>/` ed è elencato in `.claude-plugin/marketplace.json`.

## Plugin disponibili

| Plugin | Cosa fa | Comando |
|---|---|---|
| [`orchestra`](plugins/orchestra) | Opus 5.5 pianifica, delega e verifica; Sonnet 5.5 e Haiku 5.5 eseguono il lavoro pesante | `/orchestra:run <task>` |

## Installazione

In Claude Code:

```
/plugin marketplace add divadde/claude-skills-by-dave
/plugin install orchestra@orchestra-marketplace
/reload-plugins
```

Aggiornamento dopo nuovi push: `/plugin marketplace update orchestra-marketplace`.

Prova senza installare, da un clone locale del repo: `claude --plugin-dir ./plugins/orchestra`.

Requisiti:
- Claude Code aggiornato (`claude update`).
- Accesso ai modelli Opus 5.5, Sonnet 5.5 e Haiku 5.5 sull'API Anthropic. Gli ID completi (`claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`) sono scritti nei file del plugin; su Bedrock o Vertex vanno sostituiti con gli ID del provider o con gli alias `opus`, `sonnet`, `haiku`.

## Uso: orchestra

Avvia la sessione su Opus, così resta Opus per tutta la conversazione:

```
claude --model claude-opus-5-5
```

Poi, dentro la sessione:

```
/orchestra:run migra i test in tests/ da unittest a pytest e verifica che la suite passi
```

La skill è solo manuale (`disable-model-invocation`): parte solo quando la invochi tu.

## Cosa fa orchestra

L'orchestratore (Opus) tiene il giudizio, i worker economici fanno il volume. Il flusso:

1. **Preflight.** Verifica di essere su Opus e che `CLAUDE_CODE_SUBAGENT_MODEL` non sia impostata (altrimenti può sovrascrivere i modelli dei worker).
2. **Piano.** Spezza il task in unità, ciascuna con obiettivo, file in input, output atteso, controllo di accettazione e tier. Se il task è piccolo (circa tre chiamate di tool o meno) lo fa da solo.
3. **Routing.** Assegna ogni unità a un tier:

   | Tier | Cosa gli va |
   |---|---|
   | Opus (orchestratore) | requisiti ambigui, decisioni di design, bug non ovvi, modifiche sensibili, integrazione e revisione finale |
   | `orchestra:worker-sonnet` | implementazione ben specificata, refactor con test, scrittura test, lettura e sintesi di codice o documenti, ricerca web circoscritta, esplorazione read-only |
   | `orchestra:worker-haiku` | solo lavoro meccanico e verificabile: rinomine, conversioni di formato, boilerplate da template, lista esplicita di modifiche, find-and-replace controllabile con grep |

   Haiku non fa mai ricerca aperta (i suoi errori sono silenziosi) né scelte di giudizio.
4. **Delega.** Ogni worker riceve un brief strutturato (obiettivo, contesto come percorsi, cosa fare, cosa non toccare, controllo di accettazione, formato di ritorno). Le unità indipendenti partono in parallelo, su insiemi di file disgiunti.
5. **Verifica.** Non basta il "DONE" del worker: l'orchestratore esegue il controllo di accettazione o ispeziona il diff. Se fallisce, ridelega una volta con l'evidenza, poi prende in carico l'unità. Scala Haiku → Sonnet → Opus.
6. **Report.** Al massimo 10 righe: cosa è stato fatto, come è stato verificato, cosa resta aperto, più la ripartizione (per esempio `units: 2 haiku, 3 sonnet, 1 self`).

Il risparmio viene dallo spostare il volume di token sui tier più economici: l'orchestratore genera pochi token, i worker la maggior parte. Quanto si risparmia dipende da quanto del task è delegabile.

## Limiti noti

- `CLAUDE_CODE_SUBAGENT_MODEL`: se impostata può sovrascrivere il routing per-agent. Controlla con `echo "${CLAUDE_CODE_SUBAGENT_MODEL:-unset}"`.
- Gli agent built-in Explore e general-purpose ereditano il modello principale (Opus): la skill istruisce l'orchestratore a non usarli per il lavoro pesante.
- Il campo `model` nel frontmatter della skill vale per l'esecuzione della skill; per tenere Opus su tutta la sessione usa `claude --model claude-opus-5-5` o `/model claude-opus-5-5`.
- I worker non possono lanciare altri agent.
- Per vedere quale modello ha usato davvero un worker: `/tasks` mentre è attivo.
- Se gli agent del plugin non vengono trovati, la skill ripiega su `general-purpose` con `model` esplicito e lo segnala.

Dettagli aggiuntivi nel [README del plugin](plugins/orchestra/README.md), scritto in inglese.

## Struttura del repo

```
claude-skills-by-dave/
├── .claude-plugin/
│   └── marketplace.json        # catalogo: una voce per plugin
├── README.md
└── plugins/
    └── orchestra/
        ├── .claude-plugin/plugin.json
        ├── skills/run/SKILL.md          # /orchestra:run
        ├── agents/worker-sonnet.md      # orchestra:worker-sonnet
        ├── agents/worker-haiku.md       # orchestra:worker-haiku
        └── README.md
```

## Aggiungere un nuovo plugin

1. Crea `plugins/<nome>/.claude-plugin/plugin.json` e, a seconda del bisogno, `skills/<skill>/SKILL.md`, `agents/*.md`, `commands/*.md`. Le cartelle dei componenti stanno alla radice del plugin, non dentro `.claude-plugin/`.
2. Aggiungi una voce in `.claude-plugin/marketplace.json` con `"source": "./plugins/<nome>"`.
3. Valida: `claude plugin validate .` e `claude plugin validate ./plugins/<nome>`.
4. Commit e push, poi `/plugin marketplace update orchestra-marketplace` e `/plugin install <nome>@orchestra-marketplace`.

Le skill di un plugin si chiamano `/<plugin>:<cartella>`, quindi non collidono con quelle di altri plugin né con le skill personali.
