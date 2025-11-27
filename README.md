# Gerador de Vídeos Seedance (n8n)

Automação n8n para gerar vídeos curtos via IA (Seedance / fal.ai + modelos de linguagem para roteiro/cenas/efeitos sonoros).

> Este README foi gerado a partir do workflow `Gerador_de_Videos_Seedance.json`. :contentReference[oaicite:1]{index=1}

---

## Visão geral

O workflow recebe um formulário (webhook) com os parâmetros do vídeo (história, número de cenas, proporção, resolução, duração das cenas), usa um modelo de linguagem para:
1. Criar o roteiro;
2. Dividir em cenas;
3. Gerar descrições detalhadas de cada cena;
4. Enviar cada cena para um gerador de vídeo (Seedance / fal.ai);
5. Gerar efeitos sonoros para cada clipe;
6. Compilar os clipes em um único vídeo final e disponibilizar para download.

---

## Pré-requisitos

- n8n (instância própria ou cloud) com permissão para criar workflows e credenciais HTTP.
- Contas/keys para:
  - OpenAI (ou outro modelo LLM usado no workflow).
  - fal.ai / Seedance (apikeys usados nas requisições HTTP).
  - (Opcional) OpenRouter / Google Gemini — habilite apenas o provedor que desejar.
- Acesso ao repositório GitHub (para guardar o workflow e README).
- Conhecimentos básicos de Git e n8n.

---

## Como importar este workflow para seu n8n

1. Baixe o arquivo `Gerador_de_Videos_Seedance.json` (o export do workflow).
2. No n8n:
   - Vá em **Workflows** → **Import from file**.
   - Selecione o `Gerador_de_Videos_Seedance.json`.
   - Salve e ative (ou mantenha desativado até configurar credenciais).
3. Após importar, abra o workflow e revise cada nó antes de executar.

---

## Configuração de credenciais (passo a passo)

No n8n, crie as credenciais necessárias e associe aos nodes:

### 1. OpenAI / LLM
- Credencial: `OpenAi account` (nome visto no node `OpenAI Chat Model`).
- Vá em **Credentials** → **Create** → escolha OpenAI e cole sua API key.
- Verifique o node `OpenAI Chat Model` ou outro LLM que você pretenda usar. **Deixe somente o provedor que você quer usar habilitado** (os outros devem ficar `disabled` no workflow).

### 2. fal.ai / Seedance (HTTP Header Auth)
- Para os nodes que fazem chamadas a `queue.fal.run` (ex.: `Começando a Gerar o Vídeo`, `Começando a Gerar o Áudio`, `Compilar Clipes`), configure uma credencial do tipo **HTTP Header Auth**:
  - Nome sugerido: `fal.ai`
  - Campo header: `Authorization: Bearer <SUA_API_KEY>`
- No node `credentials` essas credenciais aparecem como `httpHeaderAuth` (conferir nomes idênticos).

### 3. Outros provedores (OpenRouter, Google Gemini)
- Existem nodes preparados para Gemini/OpenRouter — **desative** os que não irá utilizar (no import alguns já vêm `disabled`).

---

## Principais nós do workflow (mapa rápido)

- `Formulário` (formTrigger) — webhook que inicia o fluxo. Campos:
  - `História` (textarea) — texto base da história
  - `Número de Cenas` (number) — opcional, default 5
  - `Proporção` (dropdown) — ex: `16:9`, `9:16`
  - `Resolução` (dropdown) — ex: `480p`, `1080p`
  - `Duração das Cenas` — tempo em segundos (ex.: `5` ou `10`)

- `Edit Fields` — ajusta `number_of_scenes`.
- `Criar Roteiro` — LLM -> gera história detalhada.
- `Gerar Cenas` — LLM -> gera lista de cenas (array).
- `Tamanho Correto?` — valida se o número de cenas bate com o esperado.
- `Dividir as Cenas` -> `Criar Cena` (cada cena vira descrição detalhada: personagens, cenário, movimentos, efeitos sonoros).
- `Começando a Gerar o Vídeo` — chama Seedance / fal.ai text-to-video.
- `Começando a Gerar o Áudio` — chama endpoint `mmaudio-v2` para gerar efeitos sonoros.
- `Loop Over Items` / `Loop Over Items1` — itera sobre clipes/áudios.
- `Juntar Áudio e Vídeo` -> `Compilar Clipes` -> compila com ffmpeg-api do fal.ai.
- `Pegar Vídeo Pronto` / `Baixar Vídeo Pronto` — obtém e baixa o vídeo final.

---
## Abertura do docker n8n

docker run --hostname=a54ab346be99 --user=node --env=WEBHOOK_URL=http://localhost:5678 --env=N8N_DEFAULT_BINARY_DATA_MODE=filesystem --env=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin --env=NODE_VERSION=22.21.0 --env=YARN_VERSION=1.22.22 --env=NODE_ICU_DATA=/usr/local/lib/node_modules/full-icu --env=NODE_ENV=production --env=N8N_RELEASE_TYPE=stable --env=SHELL=/bin/sh --volume=C:\Docker\n8n-data:/home/node/.n8n --network=bridge --workdir=/home/node -p 5678:5678 --restart=no --label='org.opencontainers.image.description=Workflow Automation Tool' --label='org.opencontainers.image.source=https://github.com/n8n-io/n8n' --label='org.opencontainers.image.title=n8n' --label='org.opencontainers.image.url=https://n8n.io' --label='org.opencontainers.image.version=1.121.3' --runtime=runc -d docker.n8n.io/n8nio/n8n

## Webhook (Formulário) — como enviar uma requisição de teste

O node `Formulário` tem um `webhookId`. Exemplo (substitua `https://SEU_N8N_URL` pelo domínio da sua instância):

