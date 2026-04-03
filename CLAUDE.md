# Bomberman — CLAUDE.md

## Visão geral

Jogo de Bomberman single-file em HTML5 Canvas puro (sem frameworks, sem dependências externas).
Arquivo principal: `bomberman_v2.html`

---

## Arquitetura

Todo o código está em um único arquivo HTML com três blocos:
- **CSS** (`<style>`) — layout, HUD, botões touch, overlay de menu
- **HTML** — canvas, HUD fixo, controles touch, overlays de menu/resultado
- **JavaScript** (`<script>`) — toda a lógica do jogo (sem módulos, escopo global)

### Seções do JS (separadas por comentários `// ═══`)

| Seção | Responsabilidade |
|---|---|
| CANVAS & RESIZE | Tamanho do canvas responsivo via `TILE = min(vw/COLS, vh/ROWS)` |
| MAP | Geração procedural do mapa (`generateMap`), tile values 0/1/2/3 |
| ENTIDADE BASE | Interpolação suave de movimento (`makeEntity`, `visualPos`, `startMove`, `stepEntity`) |
| PLAYERS | Estado dos jogadores, spawn, respawn com invencibilidade |
| ENEMIES | IA simples (preferência de direção atual + shuffle aleatório) |
| BOMBS & EXPLOSIONS | Colocação, timer, chain reaction, sliding de bombas |
| INPUT P1 | Teclado (WASD/setas) + joystick touch virtual |
| INPUT P2 | Gamepad API (DualSense mapeado) |
| UPDATE | Loop de atualização: players, inimigos, bombas deslizando, explosões, colisões, power-ups |
| DRAW | Renderização Canvas 2D pixel-art (sem sprites externos) |
| HUD | Atualização do DOM (vidas, bombas, fogo, ícones mine/kick) |
| GAME OVER | Lógica de vitória/derrota por modo |
| START GAME | Inicialização/reset completo de estado |

---

## Mapa

- Grid: **13 colunas × 11 linhas** (`COLS=13, ROWS=11`)
- Tile values:
  - `0` — livre
  - `1` — parede inquebrável (borda + grade `r%2==1 && c%2==1`)
  - `2` — bloco destrutível (42% de chance, pode esconder power-up)
  - `3` — tile marcado com bomba
- Power-ups ficam invisíveis dentro do bloco (`visible:false`) e aparecem ao destruí-lo

---

## Entidades e movimento

Todas as entidades (jogadores, inimigos) usam o sistema de **interpolação por grid**:
- `gridR/gridC` — posição lógica atual
- `fromR/fromC` — posição de origem do movimento
- `progress` — 0.0 a 1.0 com easing quadrático
- `moving` — flag que bloqueia novo input até chegar ao destino

Velocidades: `PLAYER_SPEED = 4.5` tiles/s, `ENEMY_SPEED = 2.2` tiles/s

---

## Power-ups

Quatro tipos, com 50% de chance de estar em cada bloco destrutível:

| Tipo | Efeito |
|---|---|
| `bomb` | +1 bomba simultânea |
| `fire` | +1 de alcance da explosão |
| `mine` | Habilita bombas-mina (explodem ao detonar manualmente) |
| `kick` | Permite chutar bombas (bomba desliza até obstáculo) |

Power-ups **persistem após a morte** do jogador (ficam visíveis no mapa).

---

## Bombas

- Timer padrão: **2500ms**
- Bombas-mina: `timer=Infinity`, explodem via `detonateMines()`
- Chain reaction: explosão com timer≤1ms em cascata
- Sliding: `BOMB_SLIDE_SPEED = 7` tiles/s, para ao atingir obstáculo ou outra bomba
- Explosão usa `lethalTimer=300ms` (janela letal) dentro de `timer=700ms` (visual)

---

## Modos de jogo

| Modo | Descrição |
|---|---|
| 1 Jogador - Teclado | P1 com WASD/setas + Space (bomba) + B/E (detonar). Controles touch visíveis |
| 1 Jogador - Controle | P1 via Gamepad API (DualSense: botão 0=bomba, 2=detonar, d-pad/analógico) |
| 2 Jogadores | P1 teclado + P2 gamepad. Versus sem timer. Inimigos também aparecem |

- **P1 spawn:** (1,2) — canto superior esquerdo
- **P2 spawn:** (ROWS-2, COLS-3) — canto inferior direito
- **Inimigos spawn:** 4 cantos restantes, se o tile estiver livre

---

## Controles

### P1 — Teclado
- Movimento: `WASD` ou `Setas`
- Bomba: `Space`
- Detonar mina: `B` ou `E`

### P1/P2 — Gamepad (DualSense)
- Movimento: D-pad ou analógico esquerdo (dead zone `GP_DEAD=0.4`)
- Bomba: Botão 0 (X)
- Detonar mina: Botão 2 (Quadrado)

### P1 — Touch (modo 1P/teclado)
- Joystick virtual no canto esquerdo
- Botão 💣 (bomb) e 💥 (detonar) no canto direito

---

## Renderização

Tudo desenhado via Canvas 2D — **sem imagens externas**:
- Chão: xadrez verde escuro
- Paredes: azul escuro com highlight
- Blocos destrutíveis: marrom com textura de tijolos
- Personagens: formas geométricas + cabeça + pernas animadas + olhos direcionais
- Inimigos: verde (`#22c55e`)
- Explosão: gradiente radial branco→laranja com fade por `globalAlpha`
- Bombas-mina: roxo com cruz piscante

---

## Loop principal

```
requestAnimationFrame(loop)
  → update(dt)
      → updatePlayer × N
      → stepEntity inimigos + thinkEnemy
      → updateSlidingBombs
      → timers de bombas e explosões
      → checkCollisions (explosão/inimigo → player, player → power-up)
      → gameOver check
      → pollGamepadButtons
      → updateHUD
  → draw()
      → drawMap → drawPowerups → drawBombs → drawExplosions → inimigos → players
```

`dt` é clampeado em 80ms para evitar saltos em abas inativas.

---

## Pontos de atenção ao modificar

- **Tile `map[r][c]=3`** é reservado para bombas; limpar com `map[r][c]=0` ao remover bomba.
- **`safe` set** no `generateMap` usa encoding `r*100+c` — funciona só para mapas ≤100 colunas.
- **Joystick** é re-inicializado a cada `startGame` quando no modo touch — não há cleanup dos listeners anteriores.
- **Gamepad polling** é manual no `update()`, não usa eventos `gamepadconnected`.
- **Power-ups** são armazenados separados do mapa; ao destruir bloco, `pu.visible=true` mas `map[r][c]` vira `0`.
