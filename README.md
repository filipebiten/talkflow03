# TalkFlow 03 — Premiação & Revanche

App da noite do TalkFlow 03 (13/11, pós-retiro): Overflow Awards (resultado
da votação) + A Revanche (as equipes do retiro disputando pontos extra).
App irmão do [TalkFlow 01](https://github.com/filipebiten/talkflow) e
[Velhos Fins — TalkFlow 02](https://github.com/filipebiten/velhosfins):
mesma arquitetura — single-file HTML, Preact + htm via ESM (esm.sh),
sincronização via Firebase Realtime Database (projeto `talkflow-dba50`,
path `talkflow/talkflow03`).

## As duas telas

- `?view=telao` — projetado, sem controles.
- `?view=painel` — operador, notebook ou celular.

Painel escreve o estado, telão só lê e reage (igual aos apps anteriores).

## Deploy no GitHub Pages

1. Commit e push do `index.html` (e este README).
2. Settings → Pages → Deploy from branch.

## Navegação geral

Seletor no topo do painel: **Premiação | Revanche | Placar**. Dá pra trocar
a qualquer momento — o telão segue o painel.

## Operar a Premiação

1. Na abertura, carregue o JSON exportado pelo app de votação
   (botão **Carregar JSON da votação**) — ou preencha na mão.
2. **Começar premiação** → categoria 1 no telão (emoji + nome do prêmio,
   suspense).
3. **Revelar** → telão mostra troféu, nome do vencedor, confete, fanfarra.
4. **Mostrar ranking (top 3)** opcional, pra disputa acirrada.
5. **Próxima categoria** avança. **Pular categoria** avança sem revelar.
   Espaço no teclado = revelar/próxima.
6. Depois da 15ª, telão mostra o grid de fechamento com os 15 vencedores.
7. **Lista das 15 categorias** no painel deixa pular direto pra qualquer
   uma e editar vencedor/votos a qualquer momento.

## Operar a Revanche

1. **Placar do Overflow** é a primeira tela — mostra o placar que veio do
   retiro antes de qualquer prova.
2. **Começar as provas** → prova 1 no telão (nome + descrição).
3. **Lançar resultado** abre a lista de equipes com posição (1º a 10º) —
   vem pré-preenchida com a ordem atual do placar, só ajustar quem mudou.
4. **Confirmar resultado** calcula os pontos (tabela de colocação da
   prova) e mostra o **placar atualizado** no telão, com setas indicando
   quem subiu/desceu.
5. **Próxima prova** segue. **Ir direto pro resultado final** pula pro
   pódio a qualquer momento.
6. **Mostrar placar ao vivo** (ou aba **Placar** no topo) chama o ranking
   geral no telão a qualquer momento, sem interromper o fluxo da prova.
7. **Resetar revanche** zera os pontos de volta pro placar inicial do
   Overflow (as provas lançadas são desfeitas, o placar inicial não muda).
8. **Ajustar pontos / nomes das equipes** sempre visível embaixo, pra
   correção manual.

## Editar categorias, equipes e provas

No topo do `<script>` do `index.html`:
- `CATEGORIAS_PREMIACAO` — as 15 categorias com emoji.
- `EQUIPES_INICIAIS` — nome, cor e pontos de cada equipe.
- `PROVAS_INICIAIS` — 3 provas de exemplo (nome/descrição editáveis também
  pelo painel); `pontos` é a tabela de colocação (1º lugar leva o primeiro
  valor, etc). Edite ou adicione provas direto no array.

## Testado

Fluxo completo no navegador (telão + painel em abas separadas): abertura →
revelar categoria com troféu/confete/fanfarra → fechamento com os 15
vencedores; placar do Overflow → lançar resultado de prova → placar
atualizado com setas de subida/descida → resultado final com pódio e
posição anterior do campeão → placar ao vivo.

Não testado ainda: importação do JSON real da votação (só editado na mão),
atalho de teclado (espaço), botão "Resetar revanche".
