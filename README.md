# automacoes-conteudo-n8n

Fluxos n8n para produção automatizada de cortes verticais a partir de vídeos longos: transcrição, seleção de trechos, edição com ffmpeg e publicação no YouTube, mais um fluxo que busca a matéria-prima e outro que limpa temporários.

![Demonstração da edição gerada pelo ffmpeg](docs/img/demo-edicao.gif)

Demonstração com vídeo de teste sintético: legenda karaokê, textos fixos e fundo desfocado em 9:16.

## Fluxos

| Arquivo | Gatilho | O que resolve |
|---|---|---|
| `workflows/cortes-documentario-youtube.json` | arquivo novo numa pasta do Drive | transforma um vídeo longo em 2 a 5 cortes 9:16 com textos na tela e publica no YouTube |
| `workflows/cortes-futebol-youtube.json` | arquivo novo em outra pasta do Drive | mesma base, com legenda karaokê, tradução de áudio em outro idioma e regras de corte para debate esportivo |
| `workflows/cacador-de-pautas.json` | agenda diária | escolhe um vídeo Creative Commons inédito no YouTube, baixa e envia para a pasta monitorada |
| `workflows/faxina-temporarios.json` | a cada hora | apaga temporários de vídeo esquecidos por execuções que falharam |

### Cortes (documentário e futebol)

```
Drive (vídeo novo)
  -> criar pasta "Cortes - <nome>" no Drive
  -> baixar -> gravar em disco
  -> ffmpeg silencedetect -> remover silêncio (trava: nunca mais de 40%)
  -> ffmpeg: extrair áudio mono 16 kHz
  -> Whisper: transcrição com timestamps
  -> [futebol] idioma != pt -> GPT traduz segmentos, tempos preservados
  -> AI Agent: trechos + hook + título + pergunta final + descrição (JSON)
  -> Code: alinha início/fim à pontuação da fala, monta .ass e comando ffmpeg
  -> para cada corte:
       ffmpeg 1080x1920 (fundo desfocado + legendas ASS)
       -> salvar no Drive -> upload no YouTube (privado) -> esperar 5 min
  -> apagar temporários
```

### Caçador de pautas

```
agenda diária (6h)
  +-- YouTube: em alta no Brasil
  +-- termo do dia -> YouTube: busca (só Creative Commons) -> detalhes
  +-- histórico de vídeos já usados (n8n Data Table)
  -> filtrar (licença, duração, views, idioma, categoria, título) e ranquear por views/minuto
  -> escolher o melhor ainda não usado
  -> yt-dlp -> Drive (pasta monitorada pelo fluxo de cortes) -> registrar no histórico
```

### Faxina

```
a cada hora -> find /home/node/files (*.mp4, *.mp3, *.ass com mais de 3 h) -delete
```

## Onde entra IA

| Etapa | Modelo | Fluxo |
|---|---|---|
| Transcrição com timestamps (segmento; palavra no futebol) | OpenAI `whisper-1` | cortes |
| Seleção dos trechos e escrita de hook, título, texto do meio, pergunta final e descrição | OpenAI `gpt-4o` via nó AI Agent | cortes |
| Tradução de segmentos para português | OpenAI `gpt-4.1-mini` | cortes de futebol |

A saída do modelo não é aplicada direto: o código realinha os tempos à pontuação da transcrição, limita duração e quantidade de cortes e valida a tradução (falha se faltar mais de 20% dos segmentos). Descoberta e filtragem do caçador de pautas são código, sem modelo.

## Requisitos

- n8n 2.x self-hosted, com Execute Command habilitado, banco persistente (o nó Wait mantém execuções pausadas) e Data Tables para o caçador
- ffmpeg com libass, fonte DejaVu Sans Bold em `/opt/fonts`
- yt-dlp (só para o caçador de pautas)
- Docker
- Contas e APIs: Google Drive API, YouTube Data API v3 (OAuth2), OpenAI API

Variáveis de ambiente do n8n em `.env.example`. Instalação de ffmpeg, yt-dlp e fontes, com e sem imagem customizada, em [docs/infraestrutura.md](docs/infraestrutura.md).

## Importação e configuração

1. Prepare o ambiente: `docker build -t n8n-cortes ./docker` ou a montagem por volumes descrita em `docs/infraestrutura.md`.
2. No n8n: Workflows > Import from File, um arquivo de `workflows/` por vez.
3. Crie as credenciais Google Drive (OAuth2), YouTube (OAuth2) e OpenAI e associe-as aos nós. Os IDs de credencial no JSON são placeholders (`CREDENCIAL_*`).
4. Substitua os placeholders de pasta:
   - `ID_DA_PASTA_MONITORADA` no trigger do fluxo de documentário e no nó de upload do caçador;
   - `TROCAR_PELO_ID_DA_PASTA_DE_FUTEBOL` no trigger do fluxo de futebol.
   Cada fluxo de cortes precisa monitorar uma pasta diferente.
5. No caçador, crie a Data Table `videos_usados` e selecione-a nos dois nós de histórico (`SELECIONE_A_TABELA`).
6. Rode cada fluxo manualmente com um vídeo curto antes de ativar.

Detalhes por fluxo:

- [docs/cortes-documentario.md](docs/cortes-documentario.md)
- [docs/cortes-futebol.md](docs/cortes-futebol.md)
- [docs/cacador-de-pautas.md](docs/cacador-de-pautas.md)
- [docs/corte-de-silencio.md](docs/corte-de-silencio.md)
- [docs/tipografia-e-layout.md](docs/tipografia-e-layout.md)

## Limites conhecidos

- Cota do YouTube: 10.000 unidades por dia e 1.600 por upload. Com 5 cortes, um vídeo de origem por dia.
- Whisper aceita até 25 MB por arquivo; vídeos muito longos exigem dividir o áudio.
- O crédito de licença gerado pelo caçador não é repassado ao fluxo de cortes.
- Partes não validadas contra serviços reais: envio simultâneo de `timestamp_granularities[]` segment e word ao Whisper, e os nós de Data Table após importação.

## Estrutura

```
.
├── README.md
├── LICENSE
├── .env.example
├── .gitignore
├── docker/
│   └── Dockerfile
├── docs/
│   ├── cacador-de-pautas.md
│   ├── corte-de-silencio.md
│   ├── cortes-documentario.md
│   ├── cortes-futebol.md
│   ├── infraestrutura.md
│   ├── tipografia-e-layout.md
│   └── img/
└── workflows/
    ├── cacador-de-pautas.json
    ├── cortes-documentario-youtube.json
    ├── cortes-futebol-youtube.json
    └── faxina-temporarios.json
```

## Licença

MIT. Ver [LICENSE](LICENSE).
