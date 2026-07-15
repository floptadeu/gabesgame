# Sprint: som posicional por peça, progresso ao vivo, arrastar com o mouse, e letreiro interativo

**Período:** 2026-07-15
**Arquivo principal:** `index.html`

## Objetivo

Deixar a experiência de jogo mais rica e "viva": som que muda de acordo com o tamanho da peça e o tipo de movimento (rolando vs. arrastando), um indicador de progresso pra quem está jogando manualmente (não só durante a demonstração automática), a possibilidade de mover peças arrastando com o mouse (além de clicar + setas), e um letreiro 3D que reage à câmera e ao clique do jogador.

---

## 1. Som diferenciado por tipo de movimento × tamanho da peça

- `applyMove` agora calcula `shouldRoll(b, dx, dy)` **uma única vez** por jogada e repassa esse resultado tanto pra animação (`setMeshPos`/`startTween`) quanto pro som (`playMoveSFX(b, roll)`), eliminando o cálculo duplicado que existia antes.
- `tierOf(b)` classifica cada peça em `'big'` (bloco 2×2), `'medium'` (H e os V0-V3) ou `'small'` (as 4 casinhas 1×1), reaproveitando `pieceType(b)`.
- Dois sons por tier, com parâmetros em `SFX_TIER`:
  - `playThud(tier)` — o baque de "pedra tombando" que já existia, agora parametrizado (frequência/duração/ganho variam por tier); usado quando a peça rola.
  - `playScrape(tier)` — som novo de "pedra arrastando/raspando": ruído filtrado sustentado com um LFO no filtro (textura de atrito), sem o baque percussivo do thud; usado quando a peça desliza. O tier grande ganha um retumbo grave extra por baixo.
- Só 4 das 6 combinações (tier × movimento) são alcançáveis de fato jogando — o bloco grande nunca rola e as casinhas 1×1 nunca deslizam (regra do `shouldRoll` já existente) — então o ajuste fino por ouvido focou nessas 4.

## 2. Indicador ao vivo de progresso (jogo manual)

- Elemento novo `#movesRemaining`, texto simples abaixo da barra de progresso da demonstração automática — não reaproveita essa barra porque "quantos passos faltam" pode **subir** se você fizer uma jogada ruim, e uma barra de preenchimento sugeriria progresso sempre crescente.
- Depois de cada jogada manual (`move(dir)`), agenda (via `setTimeout(...,0)`, com debounce) uma chamada a `solveFromCurrent()` — o solucionador BFS que já existia — e mostra "Melhor solução daqui: N passos".
- Fica em branco durante a demonstração automática (que já tem sua própria barra) e mostra "Resolvido! 🎉" quando o jogo termina.
- Atualiza também em `resetBoard()` e uma vez ao carregar a página.

## 3. Arrastar peças com o mouse (arrasto = 1 célula)

- `pointerdown` agora já faz um raycast contra as peças; se acertou uma, guarda o id (`dragCandidate`) e desativa `controls.enabled` temporariamente (senão o OrbitControls "rouba" o gesto pra girar a câmera).
- `pointerup`: se havia uma peça sob o ponteiro **e** o arrasto passou de um limiar mínimo (`PIECE_DRAG_PX_THRESHOLD = 18`, maior que o limiar de clique `CLICK_PX_THRESHOLD = 6`, pra não confundir clique tremido com arrasto), calcula a direção do arrasto e chama o `move(dir)` de sempre — reaproveita 100% a lógica de colisão, som e progresso que já existia.
- **Mapeamento de direção**: como a câmera pode ser girada livremente, não existe um "pra cima" fixo em relação ao tabuleiro. A cada soltar do arrasto, as 4 direções do tabuleiro são projetadas pela câmera **atual** e a direção escolhida é a que mais se alinha com o vetor de arrasto na tela (`screenDeltaToBoardDir`) — funciona certo em qualquer ângulo de câmera.
- Listener de `pointercancel` como rede de segurança, sempre reativando `controls.enabled` caso o navegador perca o evento de soltar no meio do arrasto.
- O fluxo antigo (clicar pra selecionar + setas) continua funcionando sem alteração.

## 4. Letreiro "GABES GAME": billboard + reposicionamento

- `updateTitleBillboard()`, chamada a cada frame, gira o letreiro só no eixo Y (`rotation.y = Math.atan2(dx, dz)` entre câmera e letreiro) pra ele sempre ficar de frente pro ponto de vista, mantendo a inclinação fixa que já existia no eixo X. Girar só em Y evita que o texto incline de um jeito estranho em ângulos de câmera extremos.
- Reposicionado de `z: -2.9` pra `z: -3.3`, por segurança extra.

## 5. Clique no letreiro: brilho dourado + som de brilhante

- `triggerTitleSparkle()` dispara três coisas juntas: um pulso de brilho no material (`emissiveIntensity` sobe rápido e desce devagar), uma explosão de ~26 partículas douradas (`THREE.Points`, some sozinha em ~0.75s), e `playTitleChime()` — sequência rápida de notas agudas em onda triangular, bem diferente da fanfarra de vitória.
- O teste de clique no letreiro acontece antes do teste de clique nas peças no mesmo listener de `pointerup`, e não mexe na peça atualmente selecionada.

## 6. Validação / testes

Todos os testes usaram Playwright headless (Chromium), seguindo o padrão já estabelecido no projeto:

- Como o script é clássico (não-módulo), só `function` declaradas no topo viram `window.X` — todas as funções novas foram declaradas assim de propósito, pra dar aos testes acesso direto (`tierOf`, `screenDeltaToBoardDir`, `updateMovesRemaining`, etc).
- Pra estado só acessível via `const`/`let` (`blocks`, `camera`, `titleMesh`, `controls`, `renderer`, `audioCtx`), foram adicionadas pequenas funções de apoio só-pra-teste: `getBlockPos`, `getTitleYaw`, `getTitlePosition`, `debugProjectToScreen`, `getCanvasRect`, `getCameraPosition`, `getControlsEnabled`, `getAudioState`.
- Testes de lógica pura via `page.evaluate` (`tierOf` pros 4 tipos de peça).
- Smoke test de áudio: chamou `playThud`/`playScrape`/`playTitleChime` pra todos os tiers e confirmou que nada lança erro.
- Teste de gesto real: usou `debugProjectToScreen` pra calcular a posição de tela real de uma peça e da célula vazia ao lado, e simulou um arrasto de mouse de verdade (`page.mouse.move/down/move/up`) — confirmou que a peça se moveu da célula certa pra célula certa, e que o indicador de progresso atualizou.
- Teste de câmera: simulou um arrasto real fora de qualquer peça (girando a câmera de verdade via OrbitControls) e confirmou que o yaw do letreiro mudou de acordo.
- Teste de clique no letreiro: clicou na posição projetada do letreiro e confirmou visualmente (print) a explosão de partículas.
- Print antes/depois em cada etapa pra conferência visual.

Todos os testes automatizados passaram na primeira rodada completa (depois de corrigir dois bugs nos próprios testes: acessar `window.audioCtx` que não existe — é `let`, não function — e coordenadas de arrasto de câmera fora da área real do canvas).

## Estado final

- Som de movimento agora varia por tamanho da peça e por tipo de movimento (rolando/arrastando), com 4 combinações realmente alcançáveis jogando.
- Jogo manual mostra "quantos passos faltam pra resolver" em tempo real, recalculado a cada jogada.
- Dá pra mover peças arrastando com o mouse, em qualquer ângulo de câmera, sem quebrar o fluxo antigo de clicar + setas.
- O letreiro "GABES GAME" sempre fica de frente pra câmera, e brilha com partículas douradas + som ao ser clicado.

## Próximos passos (sugestões, não feitos nesta sprint)

- Zona morta pra arrastos na diagonal (hoje sempre escolhe a direção com maior pontuação, mesmo que seja por pouco).
- Recalcular o encaixe do letreiro (`fitTitleToView`) a cada frame em vez de só no resize, caso ele chegue a cortar em ângulos de câmera muito extremos.
- Cap de segurança separado (mais baixo) pro `solveFromCurrent` do indicador ao vivo, caso alguma posição do meio do jogo fique perto do limite de estados do BFS.
