# Infraestrutura: ffmpeg, yt-dlp, fontes e pasta de trabalho

Os fluxos dependem de quatro coisas fora do n8n:

| Item | Caminho esperado no container | Usado por |
|---|---|---|
| `ffmpeg` / `ffprobe` | no `PATH` | fluxos de cortes |
| `yt-dlp` | no `PATH` | caçador de pautas |
| fonte DejaVu Sans Bold | `/opt/fonts` | fluxos de cortes (libass) |
| pasta de trabalho gravável | `/home/node/files` | todos |

A imagem oficial do n8n não traz ffmpeg. Há dois caminhos.

## Opção A: imagem customizada

Válida quando a imagem base ainda tem gerenciador de pacotes. Ver `docker/Dockerfile`:

```bash
docker build -t n8n-cortes ./docker
```

## Opção B: binários estáticos montados por volume

Necessária quando a imagem usada é endurecida (sem `apk`) ou quando a orquestração (Docker Swarm, por exemplo) não usa imagens locais. Os binários ficam no host e entram no container como volumes somente leitura, o que sobrevive a atualizações do n8n.

No host:

```bash
mkdir -p /opt/n8n-bin /opt/n8n-fonts /opt/n8n-files

# ffmpeg estático (troque amd64 por arm64 em servidores ARM; confira com uname -m)
cd /tmp
curl -L -o ffmpeg.tar.xz https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz
tar -xf ffmpeg.tar.xz
cp ffmpeg-*-amd64-static/ffmpeg ffmpeg-*-amd64-static/ffprobe /opt/n8n-bin/
rm -rf ffmpeg.tar.xz ffmpeg-*-amd64-static

# yt-dlp estático
curl -L -o /opt/n8n-bin/yt-dlp https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp_linux

# fonte
curl -L -o /opt/n8n-fonts/DejaVuSans-Bold.ttf \
  https://github.com/dejavu-fonts/dejavu-fonts/raw/master/ttf/DejaVuSans-Bold.ttf

chmod 755 /opt/n8n-bin/*
chmod 644 /opt/n8n-fonts/*.ttf
chown -R 1000:1000 /opt/n8n-files   # usuário node do container
```

Nos serviços do n8n (editor e worker, se houver modo fila), acrescente:

```yaml
    environment:
      - N8N_RESTRICT_FILE_ACCESS_TO=/home/node/files
      - EXECUTIONS_TIMEOUT=7200
    volumes:
      - /opt/n8n-bin/ffmpeg:/usr/local/bin/ffmpeg:ro
      - /opt/n8n-bin/ffprobe:/usr/local/bin/ffprobe:ro
      - /opt/n8n-bin/yt-dlp:/usr/local/bin/yt-dlp:ro
      - /opt/n8n-fonts:/opt/fonts:ro
      - /opt/n8n-files:/home/node/files
```

Verificação dentro de cada container:

```sh
ffmpeg -version
yt-dlp --version
ls -la /opt/fonts/
touch /home/node/files/teste && echo ok
```

Teste do download antes de ativar o caçador:

```sh
yt-dlp --no-playlist -f "b[height<=480]" --download-sections "*0-20" \
  -o /home/node/files/teste.mp4 "<URL de um vídeo qualquer>"
```

## Por que `/home/node/files` e a variável de ambiente

A partir do n8n 2.0, `N8N_RESTRICT_FILE_ACCESS_TO` limita onde o nó Read/Write Files from Disk pode ler e gravar. Caminhos fora da lista são recusados com "is not writable" antes de qualquer acesso ao disco, mesmo que o Execute Command consiga gravar no mesmo lugar: são verificações diferentes, uma do kernel e outra da aplicação. Um volume montado também precisa estar declarado na variável.

A pasta `.n8n` é outra coisa: guarda o estado interno do n8n e é bloqueada por `N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES`.

## Modo fila e pasta compartilhada

Em modo fila, o nó Wait pausa a execução e a reenfileira; a retomada pode acontecer em outro processo (worker em vez do editor). Sem pasta compartilhada, o vídeo baixado por um container não existe para o outro e a segunda parte falha depois de a primeira funcionar. Montar o mesmo diretório do host em editor e worker resolve em servidor de nó único. Em cluster com vários nós, é preciso armazenamento de rede.

## Problemas comuns

| Sintoma | Causa provável |
|---|---|
| `ffmpeg: not found` | volume não aplicado no serviço que executou |
| `ffmpeg: Permission denied` | binário sem permissão de execução |
| `Error while opening file /opt/fonts/...` | nome do arquivo de fonte diferente do esperado |
| `is not writable` | `N8N_RESTRICT_FILE_ACCESS_TO` ausente ou container não recriado |
| `Permission denied` ao gravar em `/home/node/files` | dono do diretório no host diferente de 1000 |
| execução abortada no meio | `EXECUTIONS_TIMEOUT` menor que a duração total com as esperas |

## Faxina

`workflows/faxina-temporarios.json` roda a cada hora e apaga de `/home/node/files` os arquivos `.mp4`, `.mp3` e `.ass` com mais de 3 horas (`-mmin +180`). Cobre execuções que falharam antes do nó de limpeza. O filtro por extensão é proposital, para não apagar arquivos de outros workflows na mesma pasta.
