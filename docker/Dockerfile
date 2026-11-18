# Imagem do n8n com ffmpeg, fontes e yt-dlp, necessaria para os fluxos de cortes.
# A imagem oficial do n8n nao traz ffmpeg.
#
# Atencao: versoes recentes da imagem oficial usam base endurecida, sem gerenciador
# de pacotes (apk). Nesse caso este Dockerfile nao funciona; use a abordagem de
# binarios estaticos montados por volume descrita em docs/infraestrutura.md.

FROM n8nio/n8n:latest

USER root

RUN apk add --no-cache \
      ffmpeg \
      fontconfig \
      ttf-dejavu \
      yt-dlp \
    && fc-cache -f \
    && mkdir -p /opt/fonts /home/node/files \
    && ln -sf /usr/share/fonts/dejavu/DejaVuSans-Bold.ttf /opt/fonts/DejaVuSans-Bold.ttf \
    && chown node:node /home/node/files \
    && ffmpeg -version | head -1

# Os fluxos gravam temporarios em /home/node/files e leem fontes de /opt/fonts.
ENV N8N_RESTRICT_FILE_ACCESS_TO=/home/node/files

USER node
