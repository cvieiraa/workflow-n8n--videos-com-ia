# Gerador de Vídeos Seedance (n8n)

Automação n8n para gerar vídeos curtos via IA (Seedance / fal.ai + modelos de linguagem para roteiro/cenas/efeitos sonoros).

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

## Webhook (Formulário) — como enviar uma requisição de teste

O node `Formulário` tem um `webhookId`. Para testar o workflow:

### 1. Via interface web do n8n:
- Abra o workflow no n8n
- Clique no node `Formulário`
- Clique em "Test URL" para obter o link do webhook
- Acesse o link no navegador e preencha o formulário

### 2. Via requisição HTTP direta:
```bash
curl -X POST https://SEU_N8N_URL/webhook/SEU_WEBHOOK_ID \
  -H "Content-Type: application/json" \
  -d '{
    "História": "Uma aventura épica no espaço",
    "Número de Cenas": 5,
    "Proporção": "16:9",
    "Resolução": "1080p",
    "Duração das Cenas": 5
  }'
```

**Substitua:**
- `SEU_N8N_URL` → domínio da sua instância n8n
- `SEU_WEBHOOK_ID` → ID do webhook (visível no node `Formulário`)

### 3. Via PowerShell (Windows):
```powershell
$body = @{
    "História" = "Uma aventura épica no espaço"
    "Número de Cenas" = 5
    "Proporção" = "16:9"
    "Resolução" = "1080p"
    "Duração das Cenas" = 5
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://SEU_N8N_URL/webhook/SEU_WEBHOOK_ID" `
    -Method Post `
    -ContentType "application/json" `
    -Body $body
```

---

## Estrutura do fluxo de trabalho

```
Formulário (webhook)
    ↓
Edit Fields
    ↓
Criar Roteiro (LLM)
    ↓
Gerar Cenas (LLM)
    ↓
Tamanho Correto? (validação)
    ↓
Dividir as Cenas
    ↓
Loop: Para cada cena
    ↓
Criar Cena (LLM - detalhamento)
    ↓
Gerar Vídeo (Seedance/fal.ai)
    ↓
Gerar Áudio (mmaudio-v2)
    ↓
Aguardar conclusão
    ↓
Juntar Áudio e Vídeo
    ↓
Compilar Clipes (ffmpeg)
    ↓
Baixar Vídeo Pronto
```

---

## Parâmetros de configuração

### Proporções suportadas:
- `16:9` — formato paisagem (YouTube, TV)
- `9:16` — formato retrato (TikTok, Reels, Stories)
- `1:1` — formato quadrado (Instagram)
- `4:3` — formato clássico

### Resoluções disponíveis:
- `480p` — SD (rápido, menor qualidade)
- `720p` — HD (equilíbrio)
- `1080p` — Full HD (melhor qualidade)

### Duração das cenas:
- Recomendado: 5-10 segundos por cena
- Mínimo: 3 segundos
- Máximo: 15 segundos

---

## Troubleshooting

### Erro de credenciais
- Verifique se todas as credenciais estão configuradas corretamente
- Confirme que as API keys estão válidas e com créditos
- Teste as credenciais individualmente em cada node

### Número de cenas incorreto
- O node `Tamanho Correto?` valida isso
- Ajuste o prompt no LLM se necessário para garantir o número correto
- Verifique se o JSON retornado pelo LLM está bem formatado

### Vídeo não compila
- Verifique se todos os clipes foram gerados com sucesso
- Confirme que os áudios foram gerados corretamente
- Revise os logs do node `Compilar Clipes`
- Verifique se os URLs dos clipes estão acessíveis

### Timeout nas requisições
- Considere aumentar o timeout nos nodes HTTP
- Vídeos em alta resolução podem demorar mais para processar
- Configure retry logic nos nodes críticos

### Erro ao gerar áudio
- Verifique se o endpoint `mmaudio-v2` está funcionando
- Confirme que a credencial fal.ai está correta
- Teste com descrições de áudio mais simples

---

## Customização

### Modificar prompts do LLM
- Edite os nodes `Criar Roteiro`, `Gerar Cenas` e `Criar Cena`
- Ajuste os prompts para o estilo desejado
- Adicione instruções específicas sobre tom, gênero, etc.

### Alterar parâmetros de vídeo
- Modifique os valores padrão no node `Formulário`
- Ajuste resoluções/proporções disponíveis nos dropdowns
- Adicione novos campos personalizados

### Usar outro provedor LLM
- Desabilite o provedor atual
- Habilite e configure o provedor desejado (OpenRouter/Gemini)
- Ajuste as credenciais correspondentes
- Teste os prompts com o novo provedor

### Adicionar pós-processamento
- Adicione nodes após `Compilar Clipes`
- Implemente filtros, efeitos, legendas
- Configure upload automático para plataformas

---

## Otimização de performance

### Reduzir tempo de processamento:
- Use resoluções menores (480p, 720p) para testes
- Reduza o número de cenas
- Use durações de cena menores

### Reduzir custos de API:
- Escolha modelos LLM mais econômicos para desenvolvimento
- Cache resultados de roteiros/cenas similares
- Agrupe múltiplas gerações em batch

### Melhorar qualidade:
- Use prompts mais detalhados e específicos
- Aumente a resolução para vídeos finais
- Adicione mais contexto nas descrições de cena

---

## Exemplos de uso

### Vídeo educacional:
```json
{
  "História": "Explicar o ciclo da água de forma simples e visual",
  "Número de Cenas": 4,
  "Proporção": "16:9",
  "Resolução": "1080p",
  "Duração das Cenas": 8
}
```

### Vídeo para redes sociais:
```json
{
  "História": "Um dia na vida de um cachorro feliz no parque",
  "Número de Cenas": 6,
  "Proporção": "9:16",
  "Resolução": "1080p",
  "Duração das Cenas": 5
}
```

### Trailer curto:
```json
{
  "História": "Uma aventura de ficção científica no ano 2150",
  "Número de Cenas": 8,
  "Proporção": "16:9",
  "Resolução": "1080p",
  "Duração das Cenas": 3
}
```

---

## Monitoramento e logs

### Verificar execuções:
- No n8n, vá em **Executions** para ver histórico
- Filtre por status (sucesso/erro)
- Analise os dados de entrada/saída de cada node

### Debug de problemas:
- Ative "Save execution data" no workflow
- Use nodes de `Set` para debug de variáveis
- Adicione nodes `Stop and Error` para validações

---

## Integrações futuras

Possíveis expansões do workflow:

- Upload automático para YouTube/TikTok
- Geração de legendas/transcrições
- Tradução automática para múltiplos idiomas
- Análise de sentimento do conteúdo
- Geração de thumbnails personalizadas
- Integração com banco de dados de vídeos
- Sistema de aprovação/revisão
- Agendamento de publicações

---

## Requisitos técnicos

### n8n:
- Versão mínima: 1.0.0
- Recursos necessários:
  - RAM: 2GB mínimo, 4GB recomendado
  - Armazenamento: 10GB para temporários
  - Conexão estável com internet

### APIs utilizadas:
- **fal.ai / Seedance**: geração de vídeo
  - Endpoint: `queue.fal.run/fal-ai/seedance`
- **mmaudio-v2**: geração de áudio
  - Endpoint: `queue.fal.run/fal-ai/mmaudio-v2`
- **ffmpeg API**: compilação de clipes
  - Endpoint: `queue.fal.run/fal-ai/ffmpeg`

---

## FAQ

**P: Quanto tempo leva para gerar um vídeo?**
R: Depende da resolução e número de cenas. Em média: 5-15 minutos para 5 cenas em 1080p.

**P: Qual o custo aproximado por vídeo?**
R: Varia conforme o provedor. Estimativa: $0.50-$2.00 por vídeo de 30-60 segundos.

**P: Posso usar em produção?**
R: Sim, mas configure tratamento de erros robusto e monitore os custos de API.

**P: Suporta outros idiomas?**
R: Sim, ajuste os prompts do LLM para o idioma desejado.

**P: Posso adicionar música de fundo?**
R: Sim, adicione um node após `Compilar Clipes` para mixar áudio adicional.

---

## Suporte e comunidade

- **Documentação n8n**: https://docs.n8n.io
- **Documentação fal.ai**: https://fal.ai/docs
- **Comunidade n8n**: https://community.n8n.io
- **Issues**: Reporte problemas no repositório GitHub

---

## Changelog

### v1.0.0 (2025-11-27)
- Versão inicial do workflow
- Suporte para múltiplos provedores LLM
- Geração de vídeo com Seedance
- Geração de áudio com mmaudio-v2
- Compilação automática de clipes
- Interface de formulário web

---

## Licença

Este projeto é fornecido como está, para fins educacionais e de automação pessoal.

---

## Contribuições

Sinta-se à vontade para:
- Reportar bugs via issues
- Sugerir melhorias e novas features
- Fazer fork e customizar para suas necessidades
- Compartilhar seus workflows modificados

---

## Agradecimentos

- **n8n** — plataforma de automação open-source
- **fal.ai / Seedance** — geração de vídeos por IA
- **OpenAI / LLMs** — geração de roteiros e descrições criativas
- **mmaudio-v2** — geração de efeitos sonoros imersivos
- Comunidade n8n pelos exemplos e suporte

---

## Avisos importantes

⚠️ **Custos de API**: Este workflow consome créditos de APIs pagas. Monitore seu uso.

⚠️ **Rate limits**: Respeite os limites de requisições dos provedores.

⚠️ **Direitos autorais**: Verifique os termos de uso das APIs sobre conteúdo gerado.

⚠️ **Privacidade**: Não envie dados sensíveis através do webhook público.

---

**Desenvolvido com ❤️ para automação de criação de conteúdo por IA**
