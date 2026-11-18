# Cortes de documentário: Drive para YouTube

Arquivo: `workflows/cortes-documentario-youtube.json`

Um vídeo longo entra numa pasta do Google Drive; o fluxo transcreve, escolhe de 2 a 5 trechos, edita cada um em 9:16 com textos na tela e publica no YouTube em intervalos de 5 minutos.

## Sequência de nós

```
Drive: novo vídeo                      (trigger: arquivo criado na pasta monitorada)
  -> Garantir pasta de trabalho        (mkdir -p /home/node/files)
  -> Drive: criar pasta dos cortes     ("Cortes - <nome do vídeo>", na raiz do Drive)
  -> Drive: baixar vídeo
  -> Salvar vídeo no disco
  -> Mapear silêncios                  (ffmpeg silencedetect, só análise)
  -> Montar corte de silêncio          (Code: janelas a manter + trava de segurança)
  -> ffmpeg: remover silêncio
  -> ffmpeg: extrair áudio             (mp3 mono 16 kHz, 64 kbps)
  -> Ler áudio (.mp3)
  -> Whisper: transcrever              (verbose_json, timestamps por segmento)
  -> Montar transcrição c/ timestamps  (calcula quantos cortes o vídeo comporta)
  -> AI Agent + OpenAI Chat Model      (devolve os cortes em JSON)
  -> Montar cortes + comando de edição (alinha tempos à fala, gera .ass e comando ffmpeg)
  -> Loop: 1 corte por vez
       -> ffmpeg: editar corte 9x16 + legenda
       -> Ler clipe editado -> Drive: salvar corte na pasta
       -> Reler clipe -> YouTube: subir corte (privado)
       -> Esperar 5 minutos -> volta ao loop
  -> Limpar arquivos temporários
```

## Onde entra IA

| Etapa | Modelo | Papel |
|---|---|---|
| Transcrição | `whisper-1` (API OpenAI, via HTTP Request) | texto com timestamps por segmento |
| Seleção de cortes | `gpt-4o` (nó AI Agent) | escolhe início e fim de cada trecho e escreve os textos |

O agente devolve, para cada corte: `inicio_seg`, `fim_seg`, `hook` (até 28 caracteres, fixo na tela), `titulo` (até 42, usado só no YouTube), `texto_meio` (opcional, só se o trecho tiver uma segunda virada), `pergunta_final` e `descricao`.

Regras principais do prompt:

- nunca cortar uma história pela metade; preferir menos cortes e mais longos;
- duração entre 75 e 165 segundos, sem sobreposição;
- hook segue uma de cinco fórmulas (choque numérico, paradoxo, confissão, promessa específica, pergunta provocativa) e não promete o que o trecho não entrega;
- imperativo ("olha", "espera") só quando aponta para algo concreto.

### O código corrige o modelo

Prompt é instrução, não garantia. O nó `Montar cortes + comando de edição` ajusta os tempos devolvidos pelo agente usando os segmentos do Whisper:

- início recua até o começo de uma frase (segmento anterior terminado em pontuação final);
- fim avança até um segmento que termina em `.`, `!` ou `?`, dentro da janela de duração;
- folga de 1,2 s depois da última palavra.

A quantidade de cortes é `duração / 150 s`, limitada entre 2 e 5. O teto de 5 cabe na cota diária do YouTube (ver abaixo).

## Edição (ffmpeg)

Cada clipe sai em 1080x1920, 30 fps, AAC, `+faststart`:

- fundo: o próprio vídeo ampliado com `gblur`, sem barras pretas;
- vídeo original centralizado na largura total;
- selo `PARTE N/T`, hook em amarelo fixo do início ao fim, texto do meio por volta de 42% da duração e pergunta final nos últimos 6 s;
- textos renderizados como legenda ASS (libass), com caixa semitransparente e contorno.

O comando é montado em JavaScript, não em expressão do n8n, para evitar problemas de escape. As constantes de estética ficam no topo do nó (`LARGURA`, `ALTURA`, tamanhos de fonte, posições, cores, `CAIXA`, `TEXTO_TOPO_FIXO`). Detalhes de layout em [tipografia-e-layout.md](tipografia-e-layout.md).

Corte preciso: `-ss` + `-t` com reencode, então o corte é exato no segundo pedido, não no keyframe mais próximo.

## Corte de silêncio

Ver [corte-de-silencio.md](corte-de-silencio.md). Roda sobre o vídeo de origem, antes da transcrição, para que a duração mínima de 75 s seja de fala real e os textos não saiam de posição.

## Configuração após importar

1. Associe as credenciais Google Drive, YouTube e OpenAI aos nós (os IDs no JSON são placeholders).
2. No nó `Drive: novo vídeo`, selecione a pasta monitorada (placeholder `ID_DA_PASTA_MONITORADA`).
3. A pasta de cortes é criada na raiz do Drive. Nunca aponte para dentro da pasta monitorada: o trigger trataria os clipes como vídeos novos e o fluxo entraria em loop.
4. Os uploads saem como `private`. Troque `privacyStatus` quando confiar nos cortes.

## Restrições conhecidas

- **Binário só viaja para o nó seguinte.** Não insira nada entre `Drive: baixar vídeo` e `Salvar vídeo no disco`; nós como Execute Command devolvem só JSON e descartam o binário. Por isso as referências ao ID do vídeo apontam para o trigger (`$('Drive: novo vídeo')`). Se renomear o trigger, atualize as expressões.
- **Pasta de trabalho.** Desde o n8n 2.0, o nó Read/Write Files from Disk rejeita caminhos fora de `N8N_RESTRICT_FILE_ACCESS_TO` com a mensagem "is not writable", mesmo quando o sistema de arquivos permite. O fluxo usa `/home/node/files`; ver [infraestrutura.md](infraestrutura.md).
- **Cota do YouTube.** 10.000 unidades por dia; cada upload custa 1.600. Cinco cortes consomem 8.000, ou seja, um vídeo de origem por dia. Os clipes ficam no Drive, então um upload recusado por cota pode ser feito depois.
- **Limite do Whisper.** 25 MB por arquivo. O áudio mono 16 kHz a 64 kbps cobre cerca de 50 minutos; vídeos mais longos exigem dividir o áudio ou reduzir o bitrate.
- **Espera persistente.** O nó Wait mantém a execução em estado "waiting" por 20 minutos ou mais. Exige banco persistente (Postgres ou SQLite em volume) e `EXECUTIONS_TIMEOUT` compatível.
- **Custo de CPU.** O blur do fundo é a etapa mais pesada. Se ficar lento, troque `-preset veryfast` por `ultrafast` ou reduza o `sigma` do blur.

## Teste manual

1. Suba um vídeo curto (2 a 3 minutos) na pasta monitorada.
2. Execute manualmente e acompanhe nó a nó.
3. Confira o primeiro clipe no Drive antes de ativar o fluxo.
4. Se o ffmpeg falhar, leia o `stderr` do nó: normalmente é fonte ausente ou caractere inesperado no texto.
