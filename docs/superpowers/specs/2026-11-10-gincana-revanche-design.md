# Gincana "A Revanche" — design (talkflow03)

Spec dado pelo Filipe em 10/11/2026, em duas partes (a segunda é um ajuste de
modo de operação sobre a primeira). Transcrito verbatim abaixo.

## Parte 1 — spec original

Adicione a GINCANA ao app `talkflow03` que já está neste repositório.

ANTES DE COMEÇAR: leia o `talkflow03` inteiro, e também o `overflow-awards`,
o `talkflow` e o `velhosfins`. Esta é uma adição ao app existente, não um app
novo. A noite inteira roda num link só.

═══════════════════════════════════════════
PARTE A — AJUSTE NA PREMIAÇÃO (já existe)
═══════════════════════════════════════════

A seção Premiação hoje carrega um JSON estático exportado do `overflow-awards`.
Isso tem que mudar: a votação fica aberta até a véspera, então a revelação
precisa ler os dados AO VIVO do Realtime Database do `overflow-awards`.

- Apontar para a mesma base do app de votação
- Calcular o vencedor de cada categoria na hora da revelação
- Manter tudo o mais como está: troféu, confete, fanfarra, painel de controle
- Manter a possibilidade de o operador corrigir um vencedor manualmente antes
  de revelar

═══════════════════════════════════════════
PARTE B — A GINCANA "A REVANCHE"
═══════════════════════════════════════════

Emenda direto depois da premiação, no mesmo app, sem trocar de link.

ARQUITETURA

Três papéis, todos sincronizados via Firebase:

  ?view=telao     → projetado (já existe)
  ?view=painel    → o operador (já existe)
  ?view=time      → o celular de cada equipe (NOVO)

Cada equipe tem UM celular. Entra escaneando um QR que aparece no telão,
escolhe a cor da equipe numa lista, e pronto. Sem login, sem nome.
Se o celular recarregar, volta no mesmo estado (id salvo no navegador).

AS 10 EQUIPES E O PLACAR INICIAL

O placar NÃO começa do zero: começa com a pontuação final do retiro.

  AZUL      805
  AMARELO   795
  CINZA     760
  VERMELHO  755
  VERDE     755
  LARANJA   750
  BRANCO    745
  PRETO     740
  LILÁS     735
  ROSA      730

Cada equipe tem uma cor correspondente, usada como identificação visual nas
barras e nos blocos do telão. O fundo e a tipografia seguem o design system do
Flow, não a cor da equipe.

ESTRUTURA: 4 BLOCOS + 1 FINAL

  Bloco 1 — O QUE FICOU        · 3 rodadas · memória do retiro
  Bloco 2 — CRONOS             · 3 rodadas · ordenar 5 itens
  Bloco 3 — QUATRO POR QUATRO  · 3 rodadas · achar 4 grupos de 4
  Bloco 4 — PISTAS             · 3 rodadas · adivinhar por pistas progressivas
  FINAL   — EZEQUIEL EMBARALHADO · 1 rodada · montar o texto na ordem

A APOSTA — é o coração da mecânica

Antes de cada bloco, cada equipe aposta. A aposta é POR BLOCO, não por rodada.

  - Teto de 60 pontos por bloco (0 a 60)
  - Na FINAL o teto é 120
  - Acertou 2 das 3 rodadas do bloco → DOBRA o que apostou
  - Acertou 0 ou 1 → PERDE METADE do que apostou
  - Na final: acertou → dobra; errou → perde metade

  O telão anuncia o MODELO do bloco ("agora vem CRONOS") mas nunca o conteúdo.
  A equipe aposta sabendo o tipo de jogo, não as perguntas.
  Tempo de aposta: 45 segundos, com cronômetro no telão.

BÔNUS DE VELOCIDADE

Quem acerta PRIMEIRO em cada rodada ganha +5 pontos fixos no placar, fora da
aposta. São 13 rodadas, 65 pontos em jogo.

FLUXO DE CADA BLOCO

  1. Telão anuncia o bloco e abre a aposta (45s)
     Grid de 10 blocos mostra a aposta de cada equipe chegando
  2. Rodada 1 → rodada 2 → rodada 3
     Durante cada rodada, o grid mostra STATUS de cada equipe:
     aguardando · jogando · ENTREGOU (com o tempo)
     NUNCA mostra a resposta — as equipes copiariam
  3. Ao fim de cada rodada: o telão revela quem acertou, destacando quem
     acertou primeiro (+5)
  4. Ao fim das 3 rodadas: resultado do bloco — quem dobrou, quem perdeu metade
  5. Placar geral atualizado, com as posições trocando em animação

TELA FINAL

Pódio com as três primeiras. Campeão em destaque. E uma linha mostrando a
posição que o campeão tinha ANTES da revanche — a virada é a história da noite.

CONTROLES DO PAINEL

  - Avançar e voltar bloco e rodada
  - Abrir e fechar a aposta manualmente
  - Forçar o fim de uma rodada (se uma equipe travar)
  - Ajustar pontos manualmente por equipe (+/-)
  - Corrigir uma resposta marcada errada
  - Mostrar o placar a qualquer momento
  - Resetar a gincana sem perder o placar inicial
  - Mudo
  - Atalhos de teclado

═══════════════════════════════════════════
O CONTEÚDO — as 13 rodadas
═══════════════════════════════════════════

Tudo em arrays no topo do arquivo, fácil de editar depois.

──────────────────────────────
BLOCO 1 — O QUE FICOU
Múltipla escolha, 4 opções. A equipe toca na resposta. 45 segundos por rodada.
──────────────────────────────

RODADA 1
1. Qual era o tema do OVERFLOW 2026?
   Novos Começos ✓ · Velhos Fins · Rios de Água Viva · Identidade
2. Onde aconteceu o retiro?
   Chácara Aprisco ✓ · Recanto · Sítio Boa Esperança · Chácara Betel
3. A partir de que horas era o silêncio nos dormitórios?
   23h ✓ · 22h · 00h · 1h

RODADA 2
4. Quem pregou no culto de abertura, na sexta à noite?
   Filipe ✓ · Pr. Eli · Pra. Suene · Yuri
5. Quem pregou no culto de encerramento, no domingo de manhã?
   Yuri ✓ · Filipe · Pr. Eli · Pra. Suene
6. Qual foi a primeira música do culto de abertura?
   Tudo é Teu / Nova Criatura / Rede ao Mar ✓ · Glorioso Dia · Estações ·
   A Boa Parte

RODADA 3
7. Quantas músicas de celebração a banda tocou depois do encerramento de
   sábado à noite?
   Três ✓ · Duas · Uma · Quatro
8. Quantas arenas simultâneas teve a Batalha Naval Bíblica?
   5 ✓ · 3 · 8 · 10
9. Quantos times disputaram a gincana do retiro?
   10 ✓ · 8 · 12 · 6

──────────────────────────────
BLOCO 2 — CRONOS
Cinco itens embaralhados. A equipe arrasta para a ordem certa. 3 tentativas.
90 segundos por rodada.
──────────────────────────────

RODADA 1 — A ordem de um culto do retiro
  1. Abertura e oração inicial
  2. Louvor animado
  3. Dinâmica de integração
  4. Palavra
  5. Avisos e oração final

RODADA 2 — A ordem do fim de semana
  1. Confecção das bandeiras dos times
  2. Gincana Bíblica Indoor
  3. Culto de abertura, sexta à noite
  4. Circuito Físico e Batalha Naval
  5. Premiação da gincana, domingo

RODADA 3 — O setlist de sábado à noite
  1. Estações
  2. Quem é Esse
  3. Me Atraiu
  4. Só Quero Ver Você + Jesus, Tu és Belo
  5. Ao que Está Assentado Sobre o Trono

──────────────────────────────
BLOCO 3 — QUATRO POR QUATRO
16 palavras na tela. Achar os 4 grupos de 4. 7 chances. 2 minutos por rodada.
──────────────────────────────

RODADA 1
  Cores de equipe do retiro — MARINHO · OLIVA · LILÁS · MARROM
  Ezequiel 36 — PEDRA · CARNE · ÁGUA · ESPÍRITO
  Provas do OVERFLOW — BANDEIRA · ESGRIMA · BAMBOLÊ · POST-IT
  A visão do Flow — POSTURA · IDENTIDADE · OBEDIÊNCIA · RIO

RODADA 2
  Pregaram no retiro — FILIPE · ELI · SUENE · YURI
  Bandas do setlist — MORADA · FHOP · DUNAMIS · HILLSONG
  Gincanas do retiro — BANDEIRAS · BÍBLICA · CIRCUITO · NAVAL
  Vocais da escala — LARY · NATANAEL · DUDA · CECÍLIA

RODADA 3
  Instrumentos da escala — BATERIA · GUITARRA · VIOLÃO · TECLADO
  Músicas de sábado à noite — ESTAÇÕES · ME ATRAIU · QUEM É ESSE · TRONO
  Regras de convivência — SILÊNCIO · PONTUALIDADE · RESPEITO · CELULAR
  Dirigiram os cultos — ANAHY · RAFA · LUAN · FRAN

──────────────────────────────
BLOCO 4 — PISTAS
As pistas aparecem no TELÃO, uma a uma, para todas as equipes ao mesmo tempo.
A equipe responde no celular quando quiser arriscar. Quem arrisca cedo e acerta
vale mais. Errou, está fora da rodada.
Valor: pista 1 = 10 · pista 2 = 8 · pista 3 = 6 · pista 4 = 4 · pista 5 = 2
──────────────────────────────

RODADA 1 — Resposta: EZEQUIEL
  1. Ele era sacerdote, mas nunca chegou a servir no templo
  2. Profetizou no exílio, longe de casa
  3. Deus mandou ele falar com um vale de ossos secos
  4. O livro dele foi o texto do nosso retiro
  5. "Removerei seu coração de pedra"

RODADA 2 — Resposta: NICODEMOS
  1. Era mestre respeitado em Israel
  2. Procurou Jesus de noite
  3. Jesus se espantou que ele não soubesse de uma coisa
  4. Ouviu que era necessário nascer de novo
  5. A conversa está em João 3

RODADA 3 — Resposta: JOÃO 7:38
  1. Aconteceu no último dia de uma festa de oito dias
  2. Em todos os outros dias derramavam água sobre o altar — nesse, não
  3. Jesus se levantou e gritou no meio do templo
  4. "Se alguém tem sede, venha a mim e beba"
  5. É o versículo que dá nome à nossa rede

──────────────────────────────
FINAL — EZEQUIEL EMBARALHADO
Doze frases embaralhadas. A equipe arrasta para a ordem certa. 3 minutos.
Aposta de até 120. Acertou tudo → dobra. Qualquer erro → perde metade.
──────────────────────────────

  1.  Então aspergirei sobre vocês água pura,
  2.  e ficarão limpos.
  3.  Eu os purificarei de sua impureza
  4.  e sua adoração a ídolos.
  5.  Eu lhes darei um novo coração
  6.  e colocarei em vocês um novo espírito.
  7.  Removerei seu coração de pedra
  8.  e lhes darei coração de carne.
  9.  Porei dentro de vocês meu Espírito,
  10. para que sigam meus decretos
  11. e tenham o cuidado de obedecer a meus estatutos.
  12. Vocês serão o meu povo, e eu serei o seu Deus.

  Ezequiel 36:25-28 (NVT). Quando a primeira equipe acertar, o telão mostra o
  texto inteiro montado, com a referência.

═══════════════════════════════════════════
REQUISITOS POR VIEW
═══════════════════════════════════════════

TELÃO
  - Legível a 20 metros
  - Grid de 10 blocos durante as rodadas, com a cor de cada equipe
  - Durante a aposta: as apostas aparecendo conforme chegam
  - Durante a rodada: status e tempo, nunca conteúdo de resposta
  - No bloco PISTAS: a pista ocupa a tela, grande
  - Nenhum controle, nenhum cursor
  - Placar sempre discreto num canto

CELULAR DA EQUIPE
  - Mobile-first. Botões grandes — vai passar de mão em mão
  - Mostra sempre: cor da equipe, pontos atuais, aposta do bloco
  - Arrastar e soltar funcionando bem no toque (Cronos e Final)
  - Funciona com internet ruim: estado local, sincroniza quando volta
  - Se a conexão cair, a equipe continua jogando e envia ao reconectar

PAINEL
  - Denso, sem scroll
  - Mostra: bloco atual, rodada, apostas, quem entregou, placar
  - Botões grandes, clicáveis no escuro

SONS
Web Audio API, sem arquivos externos: abertura de aposta, fim de tempo,
acerto, erro, revelação de resultado, celebração final.

IDENTIDADE VISUAL
Siga o design system dos apps anteriores:
  Preto Flow   #0A0A0A (fundo, nunca preto puro)
  Off-white    #EBE8E0 (texto)
  Teal         #186678 (destaque — tempero, não preenchimento)
  Teal claro   #2589A0
  Bebas Neue   títulos, números, placar, pistas (caixa alta)
  Plus Jakarta Sans  interface e textos de apoio
  Ondas concêntricas como textura de fundo, opacidade baixa
  Cantos arredondados, cubic-bezier(.32,.72,0,1) nas animações

MODO SIMULAÇÃO
Um modo onde eu jogo sozinho, controlando as 10 equipes do mesmo navegador,
para testar o fluxo inteiro antes do evento. Com botão de resetar.

ENTREGA
  - Integrado ao `talkflow03`, mesma estrutura de arquivos
  - README curto: como os times entram, como operar cada bloco, como testar
  - Testar o fluxo completo: premiação ao vivo → QR → 10 times entram →
    4 blocos → final → pódio

## Parte 2 — ajuste de modo de operação (autonomia)

AJUSTE NO MODO DE OPERAÇÃO — a gincana roda sozinha

O painel passa a ser EMERGÊNCIA, não condução. Nada no fluxo normal da noite
pode depender de alguém clicar.

COMO CADA ETAPA AVANÇA

  ABERTURA / ENTRADA DOS TIMES
    O QR fica na tela até todas as 10 equipes terem entrado.
    Assim que a décima entra, espera 5 segundos e começa.
    Se faltar equipe, o operador libera pelo painel.

  APOSTA
    Fecha automaticamente quando TODAS as equipes apostaram,
    ou quando os 45 segundos acabam — o que vier primeiro.
    Equipe que não apostou entra com 0.
    Depois de fechar: 4 segundos mostrando as apostas, e começa a rodada.

  RODADA
    Encerra quando TODAS as equipes entregaram,
    ou quando o tempo da rodada acaba — o que vier primeiro.
    Equipe que não entregou conta como erro.
    IMPORTANTE: o timeout é obrigatório e nunca pode ser desligado. Uma equipe
    que travou, perdeu conexão ou desistiu não pode segurar a noite inteira.

  RESULTADO DA RODADA
    Aparece por 8 segundos — quem acertou, quem acertou primeiro — e passa
    sozinho para a próxima rodada.

  RESULTADO DO BLOCO + PLACAR
    Aparece por 12 segundos, com a animação das posições trocando, e abre a
    aposta do bloco seguinte sozinho.

  FINAL E PÓDIO
    Mesma lógica. O pódio fica na tela até alguém sair dele pelo painel.

CONTAGEM VISÍVEL NAS TRANSIÇÕES

Toda tela que vai passar sozinha mostra uma barra ou um contador discreto no
canto: "próximo em 8…". Assim ninguém é pego de surpresa, e quem está no
microfone sabe quanto tempo tem para falar.

BOTÃO DE PAUSA

Um atalho único e grande no painel: PAUSAR. Congela qualquer contagem em
andamento, em qualquer tela. Apertar de novo retoma de onde parou.
É o que quem conduz usa quando quiser falar entre um bloco e outro, sem
precisar mexer em mais nada.

O QUE O PAINEL CONTINUA PODENDO

(nota: a lista de "o que o painel continua podendo" do spec 2 se sobrepõe
à seção CONTROLES DO PAINEL da parte 1 — tratar como a mesma lista, com a
adição do botão PAUSAR e de "liberar abertura" quando falta equipe.)
