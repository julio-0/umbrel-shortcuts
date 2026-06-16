# Checklist de Implementação — notebooklm-mcp + OpenClaw

**Projeto:** PleasePrompto/notebooklm-mcp → bridge HTTP → OpenClaw (umbrelOS)
**Referência:** https://github.com/PleasePrompto/notebooklm-mcp

---

## Fase 1 — Preparar o ambiente e subir o container

Objetivo: ter o container rodando e o endpoint MCP respondendo, **antes** de tocar no OpenClaw.

- [ ] Criar a pasta do projeto no host do umbrelOS:
  ```bash
  mkdir -p ~/notebooklm-mcp/data/chrome-profile
  cd ~/notebooklm-mcp
  ```

- [ ] Copiar o `docker-compose.yml` para essa pasta.

- [ ] Confirmar que as portas `3000` e `6080` estão livres no host:
  ```bash
  ss -tlnp | grep -E '3000|6080'
  ```

- [ ] Subir os containers:
  ```bash
  docker compose up -d
  ```

- [ ] Acompanhar os logs durante a inicialização (aguardar o `npx` baixar o pacote, ~1-2 min na primeira vez):
  ```bash
  docker compose logs -f notebooklm-mcp
  ```

- [ ] Verificar que o healthcheck passa (status deve mudar de `starting` para `healthy`):
  ```bash
  docker compose ps
  ```

- [ ] Testar manualmente que o endpoint MCP está respondendo:
  ```bash
  curl -s http://localhost:3000/mcp
  # Esperado: resposta JSON ou 405 Method Not Allowed — qualquer coisa menos "connection refused"
  ```

**✅ Gate da Fase 1:** `docker compose ps` mostra `notebooklm-mcp` como `healthy`.

---

## Fase 2 — Autenticação Google (setup único)

Objetivo: fazer login na conta Google via noVNC e persistir o perfil Chrome.

> ⚠️ Este passo precisa ser feito só uma vez. O perfil fica salvo em `./data/chrome-profile/`.

- [ ] Acessar o noVNC no browser:
  ```
  http://<IP-do-umbrel>:6080/vnc.html
  ```
  Clicar em "Connect" (sem senha por padrão).

- [ ] No terminal do container, chamar o `setup_auth` via ferramenta MCP.
  A maneira mais fácil é usar o `curl` para enviar uma chamada initialize + setup_auth ao endpoint HTTP.

  **Alternativa mais simples:** entrar no container e rodar com display virtual:
  ```bash
  docker exec -it notebooklm-mcp bash

  # Dentro do container:
  DISPLAY=:99 xvfb-run -a npx notebooklm-mcp@latest
  # Em outro terminal, chamar setup_auth via MCP ou aguardar o Chrome abrir no noVNC
  ```

- [ ] Observar o Chrome abrindo no noVNC e fazer o login Google normalmente (email + senha + 2FA se necessário).

- [ ] Após o login, fechar o Chrome / encerrar o comando de setup. O perfil foi salvo.

- [ ] Confirmar que o volume foi populado:
  ```bash
  ls ~/notebooklm-mcp/data/chrome-profile/
  # Deve conter subpastas do Chrome (Default/, Singleton*, etc.)
  ```

- [ ] Reiniciar o container em modo normal (sem Xvfb extra):
  ```bash
  docker compose restart notebooklm-mcp
  ```

- [ ] Verificar nos logs que o servidor sobe sem pedir login:
  ```bash
  docker compose logs notebooklm-mcp | tail -20
  ```

**✅ Gate da Fase 2:** Nenhum prompt de login nos logs; `./data/chrome-profile/` tem conteúdo.

---

## Fase 3 — Validar o servidor MCP manualmente

Objetivo: confirmar que o servidor MCP processa requests corretamente antes de integrar com OpenClaw.

- [ ] Enviar um request `initialize` via curl para confirmar o handshake MCP:
  ```bash
  curl -X POST http://localhost:3000/mcp \
    -H "Content-Type: application/json" \
    -d '{
      "jsonrpc": "2.0",
      "id": 1,
      "method": "initialize",
      "params": {
        "protocolVersion": "2024-11-05",
        "capabilities": {},
        "clientInfo": { "name": "test", "version": "0.0.1" }
      }
    }'
  ```
  Esperado: JSON com `"result"` contendo `serverInfo` e lista de capacidades.

- [ ] Pegar o `Mcp-Session-Id` retornado no header da resposta acima.

- [ ] Listar as ferramentas disponíveis (usando o session ID obtido):
  ```bash
  SESSION_ID="<valor do header Mcp-Session-Id>"

  curl -X POST http://localhost:3000/mcp \
    -H "Content-Type: application/json" \
    -H "Mcp-Session-Id: $SESSION_ID" \
    -d '{
      "jsonrpc": "2.0",
      "id": 2,
      "method": "tools/list",
      "params": {}
    }'
  ```
  Esperado: lista de ferramentas como `ask_question`, `list_notebooks`, `setup_auth`, etc.

- [ ] (Opcional) Testar um `ask_question` básico com um notebook real:
  ```bash
  curl -X POST http://localhost:3000/mcp \
    -H "Content-Type: application/json" \
    -H "Mcp-Session-Id: $SESSION_ID" \
    -d '{
      "jsonrpc": "2.0",
      "id": 3,
      "method": "tools/call",
      "params": {
        "name": "ask_question",
        "arguments": {
          "question": "Qual é o tema principal deste notebook?",
          "notebook_url": "https://notebooklm.google.com/notebook/SEU_NOTEBOOK_ID"
        }
      }
    }'
  ```

**✅ Gate da Fase 3:** `tools/list` retorna as ferramentas esperadas. `ask_question` retorna uma resposta do NotebookLM.

---

## Fase 4 — Integrar com o OpenClaw

Objetivo: configurar o OpenClaw para consumir o servidor MCP.

- [ ] Decidir a estratégia de rede (escolha uma):

  **Opção A — Rede Docker compartilhada** (mais limpa, usa hostname do container):
  - [ ] Adicionar ao `docker-compose.yml` do OpenClaw:
    ```yaml
    networks:
      notebooklm-net:
        external: true
    ```
  - [ ] Adicionar `notebooklm-net` ao serviço do OpenClaw no compose.
  - [ ] Usar `http://notebooklm-mcp:3000/mcp` como URL.

  **Opção B — IP da LAN** (mais simples, sem mexer nos composes):
  - [ ] Descobrir o IP do umbrelOS na rede local: `hostname -I`
  - [ ] Usar `http://192.168.x.x:3000/mcp` como URL.

- [ ] Editar o `openclaw.json` para adicionar o servidor MCP:
  ```json
  {
    "mcp": {
      "servers": {
        "notebooklm": {
          "url": "http://notebooklm-mcp:3000/mcp",
          "transport": "streamable-http",
          "timeout": 60
        }
      }
    }
  }
  ```

- [ ] Reiniciar o OpenClaw para aplicar a configuração:
  ```bash
  docker compose restart openclaw
  # ou pelo painel do umbrelOS
  ```

- [ ] Verificar nos logs do OpenClaw que o servidor MCP foi descoberto:
  ```bash
  docker compose logs openclaw | grep -i notebooklm
  ```

- [ ] No OpenClaw, pedir para listar as ferramentas MCP disponíveis e confirmar que as do NotebookLM aparecem.

**✅ Gate da Fase 4:** OpenClaw lista ferramentas do `notebooklm` e consegue chamar `ask_question`.

---

## Fase 5 — Hardening e manutenção

Objetivo: deixar o setup robusto para uso contínuo.

- [ ] Desligar o noVNC após concluir o setup (não é necessário em operação normal):
  ```bash
  docker compose stop novnc
  # Para remover permanentemente do compose, comente o serviço novnc no docker-compose.yml
  # e rode: docker compose up -d
  ```

- [ ] Fechar a porta 6080 no firewall do umbrelOS (ou no router) para não expor o noVNC à rede:
  ```bash
  # Exemplo com ufw:
  sudo ufw deny 6080
  ```

- [ ] Verificar limite de uso do NotebookLM: contas gratuitas têm ~50 queries/dia. Contas Google One AI Premium têm limites maiores.

- [ ] Configurar log rotation para não encher o disco:
  ```yaml
  # Adicionar ao serviço notebooklm-mcp no docker-compose.yml:
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "3"
  ```

- [ ] Testar o comportamento após reinicialização do umbrelOS:
  ```bash
  sudo reboot
  # Após voltar:
  docker compose ps   # todos devem subir automaticamente (restart: unless-stopped)
  curl http://localhost:3000/mcp
  ```

- [ ] (Opcional) Se usar conta Google com 2FA, testar se o perfil Chrome persiste a autenticação sem pedir login novamente após ~30 dias.

**✅ Gate da Fase 5:** Setup completo e resiliente. noVNC desligado. Sistema reinicia sozinho.

---

## Referências rápidas

| Ação | Comando |
|---|---|
| Ver logs em tempo real | `docker compose logs -f notebooklm-mcp` |
| Reiniciar o servidor | `docker compose restart notebooklm-mcp` |
| Entrar no container | `docker exec -it notebooklm-mcp bash` |
| Verificar saúde | `docker compose ps` |
| Reautenticar Google | `docker exec -it notebooklm-mcp xvfb-run -a npx notebooklm-mcp@latest` (com noVNC ativo) |
| Ver configuração atual | `docker exec -it notebooklm-mcp npx notebooklm-mcp@latest config get` |
| Atualizar para nova versão | `docker compose pull && docker compose up -d` |

---

## Arquitetura final

```
OpenClaw (umbrelOS app)
    │
    │  POST http://notebooklm-mcp:3000/mcp
    │  transport: streamable-http
    │  header: Mcp-Session-Id
    ▼
notebooklm-mcp container
    ├── Node.js 20
    ├── Chromium (headless)
    ├── Patchright (stealth automation)
    └── Perfil Chrome persistido em ./data/chrome-profile/
                    │
                    │  browser automation (HTTPS)
                    ▼
          notebooklm.google.com
          (Gemini 2.5 backend)
```
