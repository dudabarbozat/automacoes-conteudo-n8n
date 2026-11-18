# Caçador de pautas

Arquivo: `workflows/cacador-de-pautas.json`

Roda uma vez por dia, escolhe um vídeo com licença Creative Commons, baixa e envia para a pasta do Drive monitorada por um dos fluxos de cortes. O fluxo de cortes assume a partir do trigger dele; não há acoplamento direto entre os dois.

```
+- caçador de pautas -----------------+
| 1x por dia: descobre, filtra, baixa |
| envia para a pasta do Drive  ----+  |
+----------------------------------|--+
                                   v
+- fluxo de cortes -------------------+
| detecta vídeo novo na pasta         |
| transcreve -> agente -> corta -> publica
+-------------------------------------+
```

Vantagens do desacoplamento: o fluxo de cortes continua funcionando com uploads manuais se o caçador falhar, cada fluxo tem seu próprio timeout e o caçador pode ser testado olhando só o que chegou na pasta. O custo é banda: o vídeo é baixado, enviado ao Drive e baixado de novo. Se isso incomodar, um nó Execute Workflow liga os dois diretamente.

## Sequência de nós

```
Diariamente às 6h
  +-- YouTube: em alta no Brasil           (videos.list, chart=mostPopular)      1 unidade
  +-- Termos do dia
  |     -> YouTube: buscar por tema        (search.list, videoLicense=creativeCommon) 100 unidades
  |     -> Detalhes dos buscados           (videos.list)                          1 unidade
  +-- Histórico: já usados                 (n8n Data Table)
        |
  Juntar fontes -> Filtrar e ranquear -> Escolher vídeo inédito
  -> Baixar com yt-dlp -> Ler vídeo baixado
  -> Drive: subir na pasta monitorada
  -> Histórico: registrar -> Apagar cópia local
```

Este fluxo não usa modelo de linguagem: descoberta, filtro e ranqueamento são feitos em código.

## Duas fontes

A lista "em alta" do YouTube é dominada por música, gameplay e esporte ao vivo, e sozinha seria rejeitada quase todo dia pelos filtros. Por isso o fluxo soma uma busca por tema, com um termo diferente por dia da semana (nó `Termos do dia`, constante `TERMOS_POR_DIA`). A configuração atual é para futebol:

| Dia | Termo |
|---|---|
| domingo | podcast futebol brasileiro debate |
| segunda | mesa redonda brasileirao analise |
| terça | entrevista jogador brasileirao |
| quarta | debate libertadores analise |
| quinta | bastidores futebol brasileiro podcast |
| sexta | coletiva tecnico brasileirao |
| sábado | programa esportivo debate futebol |

A busca considera vídeos publicados nos últimos 14 dias (`DIAS_PARA_TRAS`).

## Licença

Só entram vídeos Creative Commons, em duas camadas:

- na busca, pelo parâmetro `videoLicense=creativeCommon`;
- no filtro, pelo campo `status.license`, porque `videos.list` (fonte "em alta") não aceita o parâmetro de licença.

Desliga com `SO_CREATIVE_COMMONS = false`. O nó `Escolher vídeo inédito` gera o campo `creditos` (título original, canal, link e licença). Esse crédito ainda não é repassado ao fluxo de cortes.

## Filtros (`Filtrar e ranquear`)

| Constante | Valor | Função |
|---|---|---|
| `SO_CREATIVE_COMMONS` | true | só licença CC |
| `DUR_MIN_MIN` | 12 | vídeo curto não rende 5 cortes |
| `DUR_MAX_MIN` | 75 | vídeo longo encarece a transcrição |
| `VIEWS_MIN` | 3000 | piso baixo de propósito: vídeos CC têm menos audiência |
| `IDIOMAS` | `['pt']` | idiomas aceitos |
| `CATEGORIAS_BLOQUEADAS` | 10, 20, 1 | música, games, filme: sem fala aproveitável |
| `TITULO_BLOQUEADO` | "melhores momentos", "todos os gols", "compacto"... | imagem de partida não tem fala para transcrever |

Ranqueamento por views por minuto, não por views totais, para favorecer vídeo denso em vez de só longo.

Quando nenhum candidato passa, a execução para com a contagem por motivo de recusa, por exemplo:

```
Nenhum dos 50 vídeos passou. Contagem por motivo:
{"licença youtube":41,"curto":4,"poucas views":3,"idioma en":2}
```

A contagem indica qual filtro está barrando. Recusa majoritária por licença pede termos de busca mais amplos, não filtros mais frouxos. Como o fluxo de futebol já traduz áudio, abrir `IDIOMAS` para `['pt','en','es']` é o caminho natural para aumentar o volume de material CC.

## Deduplicação

`Escolher vídeo inédito` compara os candidatos com a tabela `videos_usados` e pega o de maior nota ainda não usado. Se todos já foram usados, para com mensagem explícita em vez de republicar.

### Tabela `videos_usados` (Data Tables do n8n)

| Coluna | Tipo |
|---|---|
| `video_id` | String |
| `titulo` | String |
| `canal` | String |
| `url` | String |
| `nota` | Number |
| `usado_em` | String |

Depois de importar, selecione a tabela nos nós `Histórico: já usados` (operação get) e `Histórico: registrar` (operação insert); o JSON traz o placeholder `SELECIONE_A_TABELA`. A configuração desses dois nós não foi validada após importação e pode exigir reconfiguração manual.

Alternativa sem Data Table, usando dados estáticos do workflow no nó `Escolher vídeo inédito`:

```js
const memoria = $getWorkflowStaticData('global');
memoria.usados = memoria.usados || [];
const usados = new Set(memoria.usados.map(String));
// ... antes do return:
memoria.usados.push(inedito.video_id);
if (memoria.usados.length > 500) memoria.usados = memoria.usados.slice(-500);
```

Nesse caso remova os dois nós de Data Table e ligue `Filtrar e ranquear` direto em `Escolher vídeo inédito`.

## Cota diária do YouTube

| Operação | Custo |
|---|---|
| `videos.list` (em alta) | 1 |
| `search.list` | 100 |
| `videos.list` (detalhes) | 1 |
| 5 uploads no fluxo de cortes | 8.000 |
| **Total** | **8.102 de 10.000** |

Cabe um vídeo de origem por dia.

## Configuração

1. Instale o `yt-dlp` no ambiente do n8n ([infraestrutura.md](infraestrutura.md)).
2. Associe as credenciais YouTube e Google Drive.
3. No nó `Drive: subir na pasta monitorada`, informe a pasta monitorada pelo fluxo de cortes desejado (placeholder `ID_DA_PASTA_MONITORADA`).
4. Crie a tabela `videos_usados` e selecione-a nos dois nós de histórico.
5. Ajuste `TERMOS_POR_DIA` ao nicho.

## Riscos

- **Bloqueio do download.** Downloads a partir de IP de datacenter podem ser recusados ("Sign in to confirm you're not a bot"). Teste o `yt-dlp` no servidor antes de ativar o fluxo.
- **Escolha automática.** Um vídeo pode passar em todos os filtros e não render bons cortes. Rode manualmente nas primeiras vezes.
- **Volume de material CC.** Conteúdo Creative Commons em português é escasso; espere dias sem candidato até calibrar os termos.
- **Binário pelo banco.** Com `binaryDataMode` em `database`, o vídeo inteiro trafega pelo Postgres. Em modo fila, `filesystem` exige volume compartilhado entre editor e worker.
