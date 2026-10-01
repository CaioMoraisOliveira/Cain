# Prompt — Fase 1 (MVP) do "Cain" (Claude Code como jogo)

Cole tudo abaixo da linha no Claude Code (com o modelo Sonnet ativo).

---

Vamos construir a FASE 1 (MVP) de um projeto chamado **Cain**: um "jogo" visual no navegador que mostra em tempo real TUDO que o Claude Code está fazendo no meu PC. Cada sessão/agente do Claude Code vira um boneco num mapa dividido por setores (empresas) e salas (departamentos), e o boneco se move de acordo com as ações REAIS que a sessão está executando. Nada é simulado: o jogo só reflete eventos reais.

## Antes de codar
1. Me pergunte, numa única mensagem: (a) lista das minhas empresas e a pasta de cada uma no disco, (b) departamentos de cada empresa (se eu não souber, use "Geral"), (c) meu sistema operacional. Depois disso, não faça mais perguntas a menos que esteja bloqueado.
2. Mostre um plano curto (máx. 15 linhas) e comece.

## Arquitetura (siga isto, sem trocar a stack)
Monorepo Node 20+ com npm workspaces, JavaScript/TypeScript simples, sem frameworks pesados:

- `hook/` — script Node **multiplataforma** (nada de bash/curl, precisa funcionar em Windows e Mac) chamado pelos hooks do Claude Code. Lê o JSON do stdin e faz POST para `http://127.0.0.1:4317/event`. Regras: timeout de 1s, NUNCA bloqueia nem falha o Claude Code (sempre `exit 0`, engole erros), sem dependências.
- `bridge/` — servidor Node (`http` + `ws`) ouvindo SOMENTE em `127.0.0.1:4317`:
  - `POST /event` recebe eventos do hook, normaliza e transmite via WebSocket.
  - Mantém estado em memória: sessões ativas (por `session_id`), agente atual, última ferramenta, status (trabalhando / ocioso / aguardando permissão / finalizado).
  - Ao iniciar, lê `world.config.json` e o roster de agentes de `~/.claude/agents/*.md` e `<pasta-da-empresa>/.claude/agents/*.md`.
  - Ao conectar um cliente, envia o snapshot completo do estado.
- `web/` — Vite + PixiJS v8. Mapa top-down estilo sci-fi pixel (por enquanto formas/retângulos com paleta neon escura; arte de verdade fica pra depois):
  - Hierarquia: **Mundo → Empresa (setor) → Departamento (sala) → Agente (boneco)**.
  - Cada sala tem "estações": Bancada (Edit/Write/MultiEdit), Terminal (Bash), Antena (WebSearch/WebFetch), Biblioteca (Read/Grep/Glob), Portal (Task/Agent = cria subagente), Descanso (ocioso/Stop).
  - Boneco anda até a estação correspondente ao evento atual (tween simples), com balão mostrando a ação (ex.: "Editando src/app.ts").
  - Notification / pedido de permissão → "!" piscando na cabeça.
  - Subagente cria um boneco menor ligado ao pai; some no SubagentStop.
  - HUD estilo terminal (como um "Business Control Surface"): agentes ativos, ações por minuto, feed de eventos ao vivo. Clique num boneco abre painel com sessão, pasta, últimas 20 ações.
  - Zoom/pan com mouse. Mapear sessão → empresa pelo `cwd` (prefixo da pasta configurada); sem match vai pro setor "Outros".
- `world.config.json` — empresas, pastas, departamentos, cores. Crie também `world.config.example.json`.

## Instalação dos hooks
- Crie `scripts/install-hooks.mjs` que faz MERGE (nunca sobrescreve) em `~/.claude/settings.json`, adicionando o hook para: SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, Notification, Stop, SubagentStop, SessionEnd (matcher `*` onde aplicável), com `type: "command"` apontando para o script do `hook/` com caminho absoluto.
- Faça backup antes (`settings.json.bak-<data>`) e crie `scripts/uninstall-hooks.mjs` que remove só o que foi adicionado.
- Me mostre o diff do settings.json e PEÇA CONFIRMAÇÃO antes de gravar.

## Comandos finais
- `npm install` e `npm run dev` sobem bridge + web juntos.
- README curto: instalar, configurar empresas, instalar hooks, rodar, desinstalar.

## Critério de pronto da Fase 1
Com `npm run dev` rodando, abro uma sessão do Claude Code em qualquer pasta de empresa, peço algo simples, e vejo o boneco aparecer no setor certo, andar entre as estações conforme as ferramentas reais, mostrar "!" quando pede permissão e ir descansar quando termina. Teste você mesmo enviando eventos falsos com um script `scripts/fake-events.mjs` (só para teste, não usado em produção).

## Fora do escopo agora (NÃO faça)
Lançar/comandar agentes pelo jogo, aprovar permissões pela tela, XP/níveis, métricas de negócio, arte pixel definitiva. Isso é Fase 2/3.

## Economia de tokens (importante)
- Não explore pastas fora do projeto além do necessário; nunca leia `node_modules`, builds ou arquivos `.jsonl` inteiros.
- Arquivos pequenos e focados; não reescreva arquivos inteiros quando um edit resolve.
- Sem testes elaborados — só o `fake-events.mjs` e uma checagem manual.
- Commits pequenos com mensagens claras em português.
- Ao terminar, pare e me dê um resumo de no máximo 10 linhas: o que foi feito, como rodar, e o que sugere para a Fase 2.
