# Automações de conteúdo em n8n

Fluxos n8n que transformam vídeos longos em cortes verticais publicados no YouTube: transcrição, seleção de trechos por IA, edição com ffmpeg e upload, mais a busca da matéria-prima e a limpeza de temporários.

![Demonstração da edição gerada pelo ffmpeg](docs/img/demo-edicao.gif)

Demonstração com vídeo de teste sintético: legenda karaokê, textos fixos e fundo desfocado em 9:16.

## Fluxos

| Arquivo | Gatilho | O que faz |
|---|---|---|
| `cortes-documentario-youtube.json` | vídeo novo numa pasta do Drive | gera 2 a 5 cortes 9:16 com textos na tela e publica no YouTube |
| `cortes-futebol-youtube.json` | vídeo novo em outra pasta | mesma base, com legenda karaokê, tradução de áudio e regras para debate esportivo |
| `cacador-de-pautas.json` | agenda diária | escolhe um vídeo Creative Commons inédito, baixa e envia para a pasta monitorada |
| `faxina-temporarios.json` | a cada hora | apaga temporários de execuções que falharam |

```
Drive (vídeo novo) -> remover silêncio (ffmpeg) -> Whisper (timestamps)
  -> [futebol] tradução -> AI Agent: trechos, hook, título, descrição (JSON)
  -> código alinha tempos à fala e monta legendas ASS
  -> ffmpeg 1080x1920 -> Drive -> YouTube (privado)
```

## Onde entra IA

| Etapa | Modelo |
|---|---|
| Transcrição com timestamps | OpenAI `whisper-1` |
| Seleção dos trechos e textos dos cortes | OpenAI `gpt-4o` (AI Agent) |
| Tradução para português | OpenAI `gpt-4.1-mini` |

A saída do modelo não é aplicada direto: o código realinha os tempos à transcrição, limita duração e quantidade de cortes e valida a tradução.

## Requisitos

n8n 2.x self-hosted com Execute Command, ffmpeg com libass, yt-dlp, Docker e as APIs Google Drive, YouTube Data v3 e OpenAI.

## Como usar

1. Monte o ambiente com `docker build -t n8n-cortes ./docker` ([docs/infraestrutura.md](docs/infraestrutura.md)).
2. Importe os arquivos de `workflows/` no n8n e associe as credenciais Google Drive, YouTube e OpenAI.
3. Troque os placeholders de pasta e de Data Table e rode cada fluxo manualmente com um vídeo curto antes de ativar.

Detalhes por fluxo em [`docs/`](docs/).

## Limites conhecidos

- Cota do YouTube: com 5 cortes, um vídeo de origem por dia.
- Whisper aceita até 25 MB por arquivo; vídeos muito longos exigem dividir o áudio.

## Licença

[MIT](LICENSE).
