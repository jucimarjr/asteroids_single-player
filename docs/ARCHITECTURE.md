# ARCHITECTURE

Projeto: asteroids_single-player  
Disciplina: Desenvolvimento de Jogos Multiplayer  

---

## 1. Objetivo

Este documento descreve a arquitetura **atual** do projeto (single-player),
com foco em didática, manutenção, coesão e baixo acoplamento.

Ele também registra uma **evolução planejada** para multiplayer, mas sem
confundir com a estrutura que existe hoje.

Regras gerais do projeto:

- PEP 8 (linhas <= 79 caracteres)
- Código didático (clareza > micro-otimização)
- Evitar import circular
- Separar “orquestração” (Game) de “regras” (World)

---

## 2. Estrutura Atual do Repositório

Estrutura existente hoje:

asteroids_single-player/
├── assets/
│   └── sounds/
├── docs/
│   └── ARCHITECTURE.md
├── audio.py
├── commands.py
├── config.py
├── controls.py
├── game.py
├── main.py
├── sprites.py
├── systems.py
└── utils.py

---

## 3. Responsabilidades por Arquivo

### 3.1 main.py

Ponto de entrada.

Responsabilidades:

- Inicializar o jogo chamando `Game().run()`
- Não conter regras de jogo

---

### 3.2 game.py

Camada de orquestração (loop).

Responsabilidades:

- Loop principal (clock, dt, eventos do pygame)
- Estados de tela (ex.: menu / playing / game over, se existirem)
- Capturar input via `controls.py`
- Chamar `World.update(dt, commands)` (em `systems.py`)
- Desenhar a cena (via pygame) e HUD
- Tocar sons chamando `audio.py` (sem regras no áudio)

Não deve:

- Implementar colisões/score (isso é do World)
- Criar dependências “de volta” para `systems.py`/`sprites.py`

---

### 3.3 systems.py

Núcleo de regras: contém o `World` (fonte da verdade).

Responsabilidades típicas do World:

- Manter o estado do jogo (entidades, score, vidas, ondas)
- Aplicar comandos do jogador (intenção)
- Atualizar simulação (movimento, TTL, spawn)
- Detectar e resolver colisões
- Decidir destruições e pontuação

Regras:

- Evitar código de interface (menus, textos, layout)
- Evitar tocar sons diretamente (preferir “sinalizar” para o Game)

---

### 3.4 sprites.py

Entidades do jogo (modelagem).

Responsabilidades típicas:

- Definir classes de entidades (Ship, Asteroid, Bullet, UFO etc.)
- Estado: posição, velocidade, ângulo, raio/rect
- Atualização por `update(dt)`

Observação importante:

- Se as entidades tiverem `draw()`, isso acopla ao pygame. Hoje isso é
  aceitável no single-player, mas deve ser extraído para um renderer
  quando preparar multiplayer.

---

### 3.5 controls.py

Mapeamento de input → comando.

Responsabilidades:

- Traduzir teclado (`pygame.key.get_pressed()`) em `PlayerCommand`
- Centralizar o “layout” de controles (esquerda/direita/impulso/tiro)

Não deve:

- Tomar decisões de regra (cooldown, limite de tiros, colisões)

---

### 3.6 commands.py

Contrato de comando do jogador (intenção).

Responsabilidades:

- Definir `PlayerCommand` (ex.: left/right/thrust/shoot)
- Ser pequeno, serializável e fácil de enviar pela rede no futuro

---

### 3.7 audio.py

Efeitos sonoros.

Responsabilidades:

- Carregar sons a partir de `assets/sounds/`
- Expor funções/métodos simples para tocar sons
- Não conter regras (por exemplo: não decidir quando toca, apenas tocar)

---

### 3.8 utils.py

Ferramentas auxiliares.

Responsabilidades típicas:

- Vetores, wrap de tela, conversão de ângulo
- Funções puras que não dependem de estado global

---

### 3.9 config.py

Configuração central.

Responsabilidades:

- Constantes do jogo (tamanhos, velocidades, TTL, limites, caminhos)
- Um único lugar para ajustar “game feel”

---

### 3.10 assets/sounds/

Recursos de áudio (WAV) usados pelo jogo.

---

## 4. Regras de Dependência (Imports)

Dependências esperadas (direção saudável):

- `main.py` → `game.py`
- `game.py` → `systems.py`, `controls.py`, `audio.py`, `config.py`
- `systems.py` → `sprites.py`, `commands.py`, `config.py`, `utils.py`
- `sprites.py` → `config.py`, `utils.py`
- `controls.py` → `commands.py`

Dependências a evitar:

- `systems.py` importar `game.py`
- `sprites.py` importar `game.py` ou `audio.py`
- qualquer import circular

---

## 5. Por que esta arquitetura já ajuda no Multiplayer

Mesmo sem rede, já existe uma separação importante:

- `commands.py` define “intenção” (o que o jogador pediu)
- `systems.py` (World) decide a “verdade” (o que aconteceu)
- `game.py` orquestra entrada/saída (input, render, som)

No multiplayer, a ideia é:

- Cliente envia `PlayerCommand`
- Servidor roda `World.update()`
- Servidor envia “estado” (snapshot) para clientes

---

## 6. Evolução Planejada (Sem Confundir com a Estrutura Atual)

Quando for preparar multiplayer, a estrutura recomendada é separar pastas:

- `core/` (regras puras, sem pygame)
- `client/` (pygame: input, render, áudio)

Plano típico de evolução:

1) Extrair desenho para um `renderer.py` (tirar `draw()` de entidades)
2) Garantir que `World` não toca som diretamente
3) Adicionar `snapshot()` serializável no `World`
4) Adicionar `entity_id`, `player_id` e `tick`
5) Criar `asteroids_multi-player/` reutilizando o núcleo (`core/`)

Importante: esta seção é “alvo futuro”. A estrutura existente hoje é a
mostrada na Seção 2.
