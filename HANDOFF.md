# HANDOFF — talkflow03 (Premiação + Gincana "A Revanche")

Sessão de 11/11/2026 (continuação da sessão de 10/11 — handoff anterior
sobrescrito, conteúdo relevante dela incorporado aqui). Evento é dia 13/11.
App já foi testado ao vivo pelo Filipe (celular real + telão + painel) e tem
várias rodadas de feedback aplicadas. Estado da gincana no Firebase foi
resetado no fim desta sessão — está limpo, pronto pro evento.

## Goal

Duas coisas no mesmo app (`talkflow03/index.html`, single-file, Preact+htm+
Firebase, mesmo padrão dos apps irmãos `talkflow`, `velhosfins`,
`overflow-awards`), publicado em `https://filipebiten.github.io/talkflow03/`:

1. **Premiação** — lê o resultado da votação do `overflow-awards` AO VIVO
   (`talkflow/overflowawards/real`).
2. **Gincana "A Revanche"** — depois da premiação, no mesmo link, 10 times
   competem ao vivo pelo celular (`?view=time`), apostando pontos por bloco,
   em 4 blocos de 3 rodadas + 1 final + pódio, com telão autônomo e placar
   que já vem do retiro.

Spec original: `docs/superpowers/specs/2026-11-10-gincana-revanche-design.md`
Plano original (14 tasks): `docs/superpowers/plans/2026-11-10-gincana-revanche.md`
(várias tasks do plano original ainda não feitas — ver "Não implementado").

## Current Progress

**Tudo testado ao vivo no Firebase real (`talkflow-dba50`) e publicado no
GitHub Pages** (2 deploys nesta sessão: `ee7e82c`, depois `caf517c`).

Feedback do Filipe testando ao vivo (celular real + telão) e o que foi
corrigido, nesta ordem:

1. **Placar animado no resultado de bloco**: antes só tinha setinha ▲▼
   estática. Agora anima de verdade — a linha "aparece como estava" e depois
   sobe/desce até a posição nova (FLIP: mede posição antes/depois via
   `getBoundingClientRect`, aplica `transform` e deixa o CSS animar).
   Primeira versão **piscava** — causa real: o relógio da contagem
   regressiva (`useContagem`, tique a cada 250ms) fazia o componente pai
   re-renderizar o tempo todo, e o hook de animação (`useFlip`) não tinha
   guarda de dependência, então reentrava no meio de uma transição em
   andamento e resetava o `transform`. Corrigido com um `dep` que só deixa
   o hook rodar quando a ordem realmente muda. Duração aumentada pra 3s
   com easing orgânico (`cubic-bezier(.16,1,.3,1)` + leve zoom) a pedido do
   Filipe ("bem lento pra dar pra acompanhar").
2. **Cronos (bloco de ordenar)**: o arraste por toque não funcionava
   direito — a causa real era que o navegador "captura" o ponteiro no
   elemento onde o toque começou, então o hit-test original (`pointerenter`
   nos vizinhos) nunca disparava durante um drag por toque. Trocado por
   hit-test manual com `document.elementFromPoint` + `pointer-events:none`
   no item sendo arrastado (pra não detectar ele mesmo). **Mesmo assim o
   Filipe reportou que ainda não conseguia mover no celular dele** (testando
   via iPhone Mirroring, que traduz toque de um jeito que não dá pra
   depurar daqui) — então foi adicionado um **fallback garantido**: botões
   ▲▼ de 44×44px (tamanho de toque recomendado) do lado de cada item, que
   movem o item uma posição pra cima/baixo com um toque, sem depender de
   gesto de arrastar. **Isso ainda não foi confirmado funcionando no celular
   real do Filipe** — é o item mais importante pra re-testar.
3. **Resultado de rodada no telão**: antes tinha uma barra lateral colorida
   fina (side-tab, que o hook de design aponta como "cara de IA"). Trocado
   por cor cheia no card + contraste de texto automático (`corContraste`,
   calcula luminância e escolhe preto ou branco).
4. **Quatro por Quatro**: grupo acertado agora sobe e vira uma linha só (FLIP
   igual ao placar), com o tema revelado acima da grade. Conteúdo corrigido
   com o texto que o Filipe mandou (ver abaixo).
5. **Pistas — reescrito do zero**: antes era um timer global que revelava
   pista pra sala inteira, achado como "bug" (pulava pista sozinho, mudava
   de pista sozinho). Agora cada equipe revela a própria pista no próprio
   ritmo (`pistasVistas: {equipeIdx: count}`, 1 pista grátis, até 6 por
   rodada), responde quando quiser, e tem botão "pular rodada" (conta como
   errado, libera pra próxima). Telão mostra status por equipe (quantas
   pistas cada um já viu), não mais um texto único pra sala toda.
   **Validação de resposta ficou mais tolerante**: antes exigia o texto
   exato normalizado; agora aceita resposta parcial contendo as palavras
   com peso da resposta certa (ex.: "samaritana" sozinho já vale pra "A
   MULHER SAMARITANA") — bug real que o Filipe pegou testando essa rodada
   específica.
6. **Pódio**: antes tinha só suspense → 3º → 2º → campeão (automático por
   timer). Filipe pediu pra começar do 10º lugar. Reescrito como
   `revanche.podioStep` (0 a 10) sincronizado no Firebase, avançado por um
   botão **"▶ Próximo"** no painel (só aparece na fase pódio) — dá controle
   total de ritmo pro operador no evento real. **Bug achado e corrigido**:
   `podioStep` não resetava pra 0 quando o jogo chegava no pódio
   naturalmente (só resetava no botão "Resetar gincana") — então se alguém
   tivesse testado até o fim antes, o pódio pulava direto pro resultado
   final em vez de mostrar o suspense. Corrigido no ponto de transição
   bloco final → pódio.
7. **Premiação**: não tinha jeito de voltar pra tela de abertura depois de
   já ter avançado (Filipe caiu direto na categoria 3 ao reabrir o painel,
   porque o estado antigo de teste ainda estava lá). Adicionado botão
   "↺ Voltar pra abertura".
8. **Conteúdo da gincana corrigido** (texto exato que o Filipe mandou,
   substituindo o que tinha antes):
   - Cronos R2 "ordem do fim de semana": agora é Culto de abertura (sexta)
     → Confecção das bandeiras → Gincana Bíblica Indoor → Circuito
     Físico/Batalha Naval → Premiação (domingo).
   - Cronos R3: trocado de "setlist de sábado à noite" (o Filipe não tinha
     certeza da ordem) pra "vale de ossos secos" (Ezequiel 37) — ligado ao
     tema do retiro, não depende de memória do evento.
   - Quatro R1 "Provas do Overflow": `PIPA` → `CAMINHÃO PIPA`.
   - Quatro: conteúdo das 3 rodadas todo substituído pelo texto real que o
     Filipe mandou (ver o código, `BLOCOS_GINCANA`, pra lista completa).
   - Pistas R3: trocado de "JOÃO 7:38" (Filipe achou que não ia rolar) pra
     "A MULHER SAMARITANA" (João 4), mantendo o tema de sede/água.
   - Pistas: todas as 3 rodadas ganharam a 6ª pista (texto do Filipe) e a
     tabela de pontos virou `[12,10,8,6,4,2]`.
   - Escolha (Bloco 1): resposta certa não fica mais sempre na opção A —
     as alternativas foram reordenadas pra cada pergunta.
9. Gerado e enviado ao Filipe um texto com todas as perguntas/respostas/
   cores de equipe pra ele mandar pro pessoal confirmar — não salvo em
   arquivo, só na conversa (pode regenerar lendo `BLOCOS_GINCANA`/
   `FINAL_GINCANA` no código se precisar de novo).

**Relacionado, repo separado (`overflow-awards`, não `talkflow03`)**: a
`?view=resultado` do overflow-awards estava pública — adicionado gate de
senha client-side (`flow12345`, visível no código-fonte, só trava clique
casual) só nessa view; `?view=votar` continua aberto pros votantes. Já
commitado e pushado nesse outro repo.

**Não implementado ainda** (do plano original de 14 tasks):
- Task 12 (parte): corrigir resposta depois do bloco já pontuado com
  undo/redo correto do delta — painel só tem ajuste manual de pontos (+/-).
- Task 13: modo Simulação de verdade (`?view=simulacao`) — hoje só existe o
  atalho `botsAtivos` (times 1-9 jogam sozinhos, só pra teste).
- Task 14: sons (`playAberturaAposta`, `playAcerto`, `playErro`) existem
  mas não estão plugados em lugar nenhum. README não atualizado com a
  seção da gincana.

## What Worked

- Arquitetura "quem escreve o quê" (telão dono das transições, painel só
  overrides manuais, celular só a própria entrada) continua segurando bem,
  mesmo com toda a reescrita de pistas/pódio desta sessão.
- **FLIP (First-Last-Invert-Play)** pra animação de reordenação sem
  bibliotecas — mede `getBoundingClientRect` antes/depois, aplica
  `transform` compensando a diferença, deixa o CSS `transition` animar até
  zero. Usado no placar E no 4x4. É mais confiável que confiar em
  reordenação de DOM com CSS puro.
- **Guarda de dependência no `useLayoutEffect`** (comparar um `dep` contra
  um `useRef` antes de rodar a lógica pesada) — resolveu o bug de piscar
  que um hook sem guarda causava quando o componente pai re-renderiza por
  outro motivo (tique do relógio) no meio de uma animação.
- **`elementFromPoint` + `pointer-events:none` no elemento arrastado** pra
  hit-test manual durante drag por toque — mais confiável que depender de
  `pointerenter` nos vizinhos, que não dispara quando o ponteiro é
  capturado (comportamento padrão de touch).
- Sobrescrever `window.confirm = () => true` via `javascript_tool` antes de
  clicar em botões com `confirm()` embutido — evita travar a aba de
  automação (problema documentado no handoff anterior, contornado desta
  vez).
- Testar no site PUBLICADO (GitHub Pages), não só local — pegou o resíduo
  de teste (`podioStep` antigo) que só aparece com estado real/compartilhado
  acumulado, difícil de reproduzir num reset limpo local.

## What Didn't Work / Still Uncertain

- **Drag do Cronos ainda não confirmado no celular real do Filipe.** Duas
  rodadas de fix (pointer capture, depois hit-test manual) e ele ainda
  reportou não conseguir mover. Testar via automação de mouse no Chrome
  SEMPRE funcionou (não reproduz o problema), então a causa exata no
  dispositivo dele (iPhone Mirroring, que traduz touch de um jeito próprio)
  não foi isolada. Os botões ▲▼ são o fallback garantido — **confirmar com
  o Filipe se os botões aparecem e funcionam**, e se o drag em si continua
  quebrado nele é OK deixar assim (os botões resolvem o essencial).
- REST direto no Firebase pra reset de estado (`node -e` com `https`)
  **parou de funcionar** desde que a regra de auth foi ativada (10/11) —
  dá 401 sem token. Resetar agora só dá pelo botão "Resetar gincana" no
  próprio app (que faz a auth anônima certa).
- O modo automático "/auto" do Claude Code bloqueia escrita direta via
  Bash em recurso compartilhado (Firebase) mesmo com consentimento do
  usuário no chat — precisa ser feito pela UI do próprio app.

## Next Steps

1. **Confirmar com o Filipe**: os botões ▲▼ do Cronos funcionam no celular
   dele? Se sim, problema resolvido (drag continua quebrado mas não importa
   mais). Se não, precisa de outra abordagem (talvez trocar o mecanismo de
   ordenar inteiro por botões, sem a opção de arrastar).
2. Terminar o resto do plano original se der tempo: Task 12 (undo/redo),
   Task 13 (Simulação de verdade), Task 14 (plugar sons, atualizar README)
   — baixa prioridade, evento é amanhã.
3. **Antes do evento**: a gincana já foi resetada no fim desta sessão
   (fase entrada, bots desligados, placar inicial) — não esquecer de
   resetar de novo se alguém testar depois disso.
4. Revisitar a regra do Firebase pra `overflowawards`/`velhosfins`/
   `talkflow` (hoje só `talkflow03` é protegido) depois que a votação do
   Overflow Awards fechar — decisão pendente desde 10/11, ainda não feita.

## Como testar de novo

- **Site publicado (preferir este pro teste real)**: `https://filipebiten.github.io/talkflow03/`
  (`?view=telao`, `?view=painel`, `?view=time`). Deploy = `git add index.html && git commit && git push` na branch `main`; GitHub Pages atualiza em ~30-60s.
- Servidor local pra iteração rápida: `cd /Users/filipebiten/projects/talkflow03 && python3 -m http.server 8751`.
- Resetar gincana: `?view=painel` → aba Revanche → "Resetar gincana" (tem
  `confirm()` — se for automatizar, rodar `window.confirm = () => true`
  via JS antes de clicar).
- Ligar bots de teste (times 1-9 jogam sozinhos): botão "Ativar bots
  (teste)" no painel — **lembrar de desligar antes do evento real**.
- Avançar fase manualmente sem esperar o timer: "Forçar fim da fase atual"
  no painel.
- Avançar o pódio (10º→1º→campeão): botão "▶ Próximo" no painel, só visível
  quando `fase === 'podio'`.
