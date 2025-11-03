# Jogo de Terminal em Go (single-player + RPC multiplayer)

Este projeto é um jogo de terminal em Go que agora suporta modo multiplayer via RPC.
O cliente mantém toda a lógica do jogo e o servidor centraliza o estado dos jogadores (posições, vidas).

Principais pontos:

- Cliente: interface, lógica de movimentação, polling periódico para `GetState` no servidor.
- Servidor: mantém lista de jogadores e deduplicação exactly-once por ClientID+Seq; não contém lógica de movimentação nem mapa.

## Controles (single-player / cliente)

| Tecla | Ação                |
| ----- | ------------------- |
| W     | Mover para cima     |
| A     | Mover para esquerda |
| S     | Mover para baixo    |
| D     | Mover para direita  |
| E     | Interagir           |
| ESC   | Sair do jogo        |

## Como rodar (modo RPC multiplayer)

### Requisitos

- Go 1.18+ instalado
- Terminal com suporte a cores (para a interface do jogo)

### Opção 1: Usando o script automatizado (macOS/Linux)

O jeito mais rápido para testar o multiplayer:

**Terminal 1 - Servidor:**

```bash
./run_multiplayer.sh
```

**Terminal 2 - Cliente A:**

```bash
RPC_ADDR="127.0.0.1:12345" CLIENTID_FILE="clientA.clientid" go run .
```

**Terminal 3 - Cliente B:**

```bash
RPC_ADDR="127.0.0.1:12345" CLIENTID_FILE="clientB.clientid" go run .
```

### Opção 2: Passo a passo manual (qualquer sistema)

**1. Compilar o projeto (uma vez):**

```bash
go build ./...
```

**2. Iniciar o servidor (Terminal 1):**

```bash
# macOS/Linux
go run -tags server .

# Windows (PowerShell)
go run -tags server .
```

O servidor deve mostrar: `[SERVER] RPC server listening on :12345`

**3. Iniciar o Cliente A (Terminal 2):**

```bash
# macOS/Linux
RPC_ADDR="127.0.0.1:12345" CLIENTID_FILE="clientA.clientid" go run .

# Windows (PowerShell)
$env:RPC_ADDR="127.0.0.1:12345"; $env:CLIENTID_FILE="clientA.clientid"; go run .
```

**4. Iniciar o Cliente B (Terminal 3):**

```bash
# macOS/Linux
RPC_ADDR="127.0.0.1:12345" CLIENTID_FILE="clientB.clientid" go run .

# Windows (PowerShell)
$env:RPC_ADDR="127.0.0.1:12345"; $env:CLIENTID_FILE="clientB.clientid"; go run .
```

**O que você deve ver:**

- ✅ No **Terminal do Servidor**: logs mostrando `Registered player` e `Received GetState`
- ✅ Nos **Clientes** (jogo na tela):
  - Barra de status mostra: `Jogadores Online: 2`
  - Um **☺ amarelo** representa o outro jogador
  - Quando você move um cliente (WASD), o outro vê o movimento em tempo real

### Opção 3: Compilar binários (para distribuição)

**Servidor:**

```bash
# macOS/Linux
go build -tags server -o gameserver .
./gameserver

# Windows
go build -tags server -o gameserver.exe .
.\gameserver.exe
```

**Cliente:**

```bash
# macOS/Linux
go build -o jogo .
RPC_ADDR="127.0.0.1:12345" CLIENTID_FILE="clientA.clientid" ./jogo

# Windows
go build -o jogo.exe .
$env:RPC_ADDR="127.0.0.1:12345"; $env:CLIENTID_FILE="clientA.clientid"; .\jogo.exe
```

### Variáveis de ambiente úteis

### Variáveis de ambiente úteis

**Cliente:**

- `RPC_ADDR` ou `SERVER_ADDR`: Endereço do servidor (padrão: `127.0.0.1:12345`)
- `CLIENTID_FILE`: Arquivo para persistir o ID do cliente (padrão: `.clientid`)
- `CLIENT_ID`: Forçar um ID específico (ignora arquivo)
- `POLL_MS`: Intervalo de polling em milissegundos (padrão: `300`)
- `DEBUG`: Ativar logs detalhados no stderr (`DEBUG=1`)
- `DEBUG_PANEL`: Mostrar logs em painel de debug na tela (`DEBUG_PANEL=1`)
- `CLIENT_LOG`: Gravar logs em arquivo (ex: `CLIENT_LOG=client.log`)

**Servidor:**

- `GAME_PORT`: Porta do servidor (padrão: `12345`)

### Troubleshooting

**Problema: "address already in use"**

```bash
# Matar processo na porta 12345
lsof -ti:12345 | xargs kill -9
```

**Problema: Clientes não conectam**

1. Verifique se o servidor está rodando
2. Execute cliente com `DEBUG=1` para ver erros de conexão
3. Confirme que `RPC_ADDR` está correto

**Problema: Não vejo outros jogadores**

- Verifique os logs do servidor: deve mostrar `Registered player` para cada cliente
- Nos clientes: barra de status deve mostrar `Jogadores Online: N`
- Se aparecer `Jogadores Online: 1`, apenas 1 cliente conectou

### Testes automatizados

```bash
# Executar todos os testes
go test ./...

# Executar testes com cobertura
go test -cover ./...

# Executar teste específico de RPC
go test -v -run TestExactlyOnce
```

### Scripts de ajuda

**macOS/Linux:**

- `run_multiplayer.sh` — Inicia o servidor com instruções para os clientes

**Windows (PowerShell):**

- Scripts em `scripts/`:
  - `start_server.ps1` — inicia o servidor (aceita parâmetro `-Port`)
  - `start_clients.ps1` — abre múltiplas instâncias do cliente em terminais novos

Notas finais

- O servidor NÃO guarda o mapa nem a lógica de movimentação — isso continua sendo responsabilidade do cliente.
- A persistência de `Seq` no cliente é atômica (escreve em arquivo temporário e renomeia), garantindo resiliência contra crashes durante a escrita.

## Estado atual em relação aos requisitos do trabalho

Resumo curto:

- O servidor gerencia a sessão e o estado dos jogadores (posições, vidas). ✔
- O servidor não mantém o mapa nem a lógica de movimentação (fica no cliente). ✔
- Comunicação sempre iniciada pelos clientes; servidor apenas responde. ✔
- Cliente possui goroutine de polling para `GetState`. ✔
- Chamadas RPC têm retries/backoff implementados no cliente. ✔
- Exactly-once (deduplicação por ClientID+Seq) implementado no servidor com TTL e limpeza. ✔

Notas/pequenas recomendações: Persistência de Seq agora realizada de forma atômica no cliente; recomenda-se adicionar testes de falhas de rede.

## Estrutura do projeto (resumida)

- `server_main.go` — main do servidor (build tag `server`).
- `server.go` — implementação do `GameServer`, deduplicação e `GetState`.
- `client_rpc.go` — cliente RPC com retries e persistência de Seq.
- `main.go` — cliente/jogo com loop principal e integração RPC.
- `jogo.go`, `personagem.go`, `interface.go` — lógica do jogo local e UI.
- `rpc_types.go` — tipos compartilhados (PlayerInfo, CommandArgs, etc.).
- `server_rpc_test.go` — teste que valida exactly-once e GetState via RPC.

---

Se quiser, eu adiciono um passo-a-passo mais enxuto para apresentação (1 slide) ou crio scripts `start_server.ps1` / `start_clients.ps1` para automatizar demos locais — quer que eu crie isso agora?
