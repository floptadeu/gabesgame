# Sprint: Jogo de Blocos Deslizantes (Huarong Dao / Klotski)

**Período:** 2026-07-10 a 2026-07-13
**Arquivo principal:** `index.html`
**Referências visuais:** `FORMA.png` (layout inicial), `POSICAO FINAL.png` (layout de vitória)

## Objetivo

Adaptar o jogo de blocos deslizantes existente para reproduzir exatamente as
peças e o tabuleiro mostrados nas imagens de referência, e adicionar um botão
que demonstra a solução automaticamente, para o usuário aprender jogando.

---

## 1. Análise das imagens de referência

As imagens `FORMA.png` e `POSICAO FINAL.png` foram analisadas pixel a pixel
(componentes conectados + detecção de linhas de grade) para extrair a posição
exata de cada peça, em vez de estimar visualmente. Isso confirmou que o jogo é
o clássico **Huarong Dao / Klotski** ("Cao Cao foge"), num tabuleiro **4
colunas × 5 linhas**, com 10 peças:

- 1 bloco grande 2×2 (`B`)
- 1 bloco horizontal 2×1 (`H`)
- 4 blocos verticais 1×2 (`V0`–`V3`)
- 4 blocos quadrados 1×1 (`S0`–`S3`)

**Layout inicial** (`FORMA.png`):

```
S0  BB  BB  S2
S1  BB  BB  S3
.   HH  HH  .
V0  V1  V2  V3
V0  V1  V2  V3
```

(dois espaços vazios na linha 2, colunas 0 e 3)

**Layout final / vitória** (`POSICAO FINAL.png`), obtido pela mesma análise de
pixels:

```
.   Q   Q   .
V   Q   Q   V
V   H   H   V
V   B   B   V
V   B   B   V
```

- o bloco grande (`B`) desce até embaixo, centro;
- o retângulo (`H`) termina exatamente em cima dele;
- os 4 quadrados (`Q`) sobem e ocupam o espaço 2×2 que o bloco grande deixou
  vago no topo;
- os 4 verticais (`V`) preenchem as colunas das pontas (2 empilhados de cada
  lado).

## 2. Reconstrução do `index.html`

- O array `blocks` foi reescrito para bater exatamente com o layout de
  `FORMA.png` (antes tinha um layout parecido mas incorreto: blocos verticais
  no lugar errado, quadrados a mais/a menos).
- Confirmado por busca em largura (BFS, script Python descartável) que esse
  layout inicial é **solucionável**.

## 3. Botão "Mostrar solução automática"

Adicionado um solucionador (BFS) em JavaScript, direto no `index.html`:

- Calcula, a partir da posição **atual** do tabuleiro (não só do início), a
  sequência de movimentos de 1 célula até a posição de vitória.
- Anima os movimentos um a um (~450ms cada), destacando em laranja a peça que
  está se mexendo, para o usuário acompanhar e imitar depois.
- Botões novos: **Mostrar solução automática**, **Parar** (interrompe a
  demonstração no meio) e **Reiniciar** (volta ao layout inicial).
- Controles manuais (setas) ficam desabilitados durante a demonstração.

## 4. Bugs encontrados e corrigidos durante o sprint

| # | Problema | Correção |
|---|----------|----------|
| 1 | Condição de vitória checava só a posição do bloco grande (`B`) | Passou a checar também o retângulo (`H`) |
| 2 | Mesmo checando `B` + `H`, o solucionador deixava as outras 8 peças em qualquer posição — não batia com `POSICAO FINAL.png` | Vitória agora exige o **grid completo** (`TARGET_GRID`), célula a célula, comparando pelo *tipo* de peça (`Q`, `V`, `H`, `B`) |
| 3 | Cor do retângulo (`H`) estava branca; devia ser azul como o bloco grande | Trocada para `cyan` |
| 4 | Ao exigir o grid completo, o BFS **estourou a memória** (Node ficou sem heap) | As 4 peças `Q` e as 4 peças `V` são idênticas entre si; o BFS as tratava como distintas, multiplicando o espaço de busca por todas as permutações possíveis entre peças iguais. Corrigido usando uma chave de deduplicação **canônica por tipo** (agrupa e ordena posições de peças do mesmo tipo), mantendo a árvore de movimentos real para a animação |

## 5. Validação / testes

Como o ambiente não tem navegador interativo disponível para o assistente,
todos os testes foram feitos programaticamente, sem depender de "parece
certo":

- **BFS em Python** (script descartável): confirmou que o layout inicial é
  solucionável e que o menor caminho (só com a exigência de `B` no lugar)
  tinha 47 movimentos.
- **Reimplementação em Node.js** da lógica exata do solucionador: replay de
  cada movimento contra o grid de ocupação, validando que nenhum é ilegal e
  que o resultado final bate com o alvo.
- **Teste headless com `jsdom`**: carregou o `index.html` real (arquivo do
  disco, sem cópias), clicou de verdade no botão "Mostrar solução automática"
  e conferiu:
  - mensagem final = "Você venceu!"
  - grid final == `TARGET_GRID` exatamente
  - nenhuma peça sobreposta ou fora do tabuleiro
  - cor do `H` == `cyan`

Esse último teste foi reexecutado a cada correção (condição de vitória, cor,
e o bug de estouro de memória do BFS) até todos os critérios passarem.

## 6. Estado final

- `index.html` reproduz fielmente `FORMA.png` no início e `POSICAO
  FINAL.png` na vitória.
- Botão de solução automática funcional, testado de ponta a ponta.
- Contorno tracejado no tabuleiro mostra visualmente as 5 regiões-alvo (bloco
  grande, retângulo, área dos quadrados, coluna esquerda e coluna direita de
  verticais).

## Próximos passos (sugestões, não feitos neste sprint)

- Suporte a toque/arraste direto nas peças (hoje é clique + setas).
- Contador de movimentos do jogador e comparação com o mínimo teórico.
- Indicador de progresso (dificuldade / nº de jogadas até a solução a partir
  da posição atual).
