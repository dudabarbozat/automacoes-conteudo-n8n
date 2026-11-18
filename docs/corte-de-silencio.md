# Corte de silêncio

Presente nos dois fluxos de cortes, entre `Salvar vídeo no disco` e `ffmpeg: extrair áudio`:

```
Salvar vídeo no disco
  -> Mapear silêncios          (ffmpeg silencedetect: só decodifica, não encoda)
  -> Montar corte de silêncio  (Code: lê a análise e monta o comando)
  -> ffmpeg: remover silêncio  (uma passada, substitui o arquivo original)
  -> ffmpeg: extrair áudio
```

## Por que no vídeo de origem

Cortar silêncio em cada clipe, depois da seleção, quebraria três coisas:

1. A duração mudaria depois de definida (um corte de 90 s com 15% de pausa viraria 76 s). Cortando antes, o mínimo de 75 s passa a ser de fala real.
2. Os textos posicionados por tempo (meio em 42%, pergunta nos últimos 6 s) sairiam de lugar.
3. A transcrição e os clipes sairiam de uma origem maior, com mais custo de Whisper e de encode.

## Implementação

Uma passada, sem arquivos intermediários: `select`/`aselect` mantêm as janelas de fala e `setpts`/`asetpts` recompõem a linha do tempo.

```
-vf "select='between(t,0,3.15)+between(t,5.85,9.65)+...',setpts=N/FRAME_RATE/TB"
-af "aselect='between(t,0,3.15)+between(t,5.85,9.65)+...',asetpts=N/SR/TB"
```

No áudio a expressão é `N/SR/TB`. Com outra variável o ffmpeg falha com "Error while parsing expression" e gera arquivo vazio.

Teste com vídeo sintético de 20,16 s com três silêncios: saída de 12,52 s, zero silêncios detectados na saída, diferença de cerca de 100 ms entre áudio e vídeo (granularidade de quadro AAC, não deriva acumulada).

## Parâmetros

No nó `Montar corte de silêncio`:

| Constante | Valor | Função |
|---|---|---|
| `MARGEM` | 0.15 | segundos de silêncio mantidos em cada ponta |
| `MIN_TRECHO` | 0.25 | descarta trechos menores (ruído solto) |
| `MAX_TRECHOS` | 220 | teto de janelas, para não estourar o tamanho do comando |
| `LIMITE_SEGURANCA` | 40 | se o corte passaria de 40% do vídeo, não corta nada |
| `MAX_MINUTOS` | 75 | acima disso segue sem cortar silêncio, para não segurar o worker |

No nó `Mapear silêncios`: `silencedetect=noise=-32dB:d=0.6`. Valor de `noise` mais negativo corta menos; `d` menor corta mais e mais picado.

## Trava de segurança

Se o áudio estiver baixo, o detector enxerga silêncio em tudo. Em vez de publicar um vídeo destruído, o nó devolve, por exemplo:

```
houve_corte: false
motivo: cortaria 87.9% do vídeo (teto de 40%)
```

e o fluxo segue com o vídeo intacto. Vídeo com música de fundo contínua não tem silêncio detectado e passa sem encode extra.

## Leitura da primeira execução

O nó expõe `trechos`, `duracao_antes`, `duracao_depois` e `percentual_cortado`.

- 10 a 20%: faixa esperada para narração.
- abaixo de 5%: detector exigente demais ou vídeo com música; tente `noise=-28dB` ou `d=0.4`.
- acima de 30%: agressivo; aumente `MARGEM` para 0.25 ou `d` para 0.8.
