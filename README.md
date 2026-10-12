# sb-duplex

Transferência de arquivos **por áudio** entre dois PCs, por cabo: codifica o arquivo em som (**OFDM** com
correção de erros), envia pela saída de áudio de um PC e reconstrói na entrada do outro. Com dois cabos é
**bidirecional**: o receptor confirma e o transmissor reenvia só o que faltou. Os dois lados usam o mesmo
programa, por linha de comando. Inclui **calibração automática** do enlace nos dois sentidos.

> **Versão beta, só Windows.** O áudio usa WASAPI (modo exclusivo). Linux e macOS ficam para versões
> futuras.

## Requisitos

- Windows 10 ou 11 (WASAPI)
- Python 3.10 ou mais novo
- Dois PCs e cabos de áudio P2 (3,5 mm); para notebooks com entrada combo, um cabo P3 e um adaptador fone +
  microfone (veja as ligações abaixo)

## Instalação

```bash
python -m pip install --pre sb-duplex
```

Enquanto a versão for beta, o `--pre` é necessário (ou a versão explícita: `sb-duplex==1.3.0b0`).
O pacote não cria um executável: tudo roda com `python -m soundbridge`.

```bash
python -m soundbridge --version
python -m soundbridge --help
python -m soundbridge --list-devices
```

## Como usar

Cada PC usa uma **saída** (`--out`) e uma **entrada** (`--in`) de áudio, ligadas ao outro PC por cabo:
a saída de um vai na entrada do outro. Com um cabo, a transferência é **unidirecional** (`uni`); com
dois cabos cruzados, pode ser **bidirecional** (`--bi`, com confirmação e reenvio).

![Ligação unidirecional: a saída de áudio do PC A vai na entrada (microfone ou linha) do PC B](https://raw.githubusercontent.com/Cafecanudo/sb-duplex/main/docs/img/ligacao-unidirecional.svg)

![Ligação bidirecional: dois cabos cruzados, dados de A para B e ACK de B para A](https://raw.githubusercontent.com/Cafecanudo/sb-duplex/main/docs/img/ligacao-bidirecional.svg)

Notebook com uma só entrada **combo** (P3, fone e microfone no mesmo conector): um cabo P3 de 4 polos leva
os dois sentidos e, no outro PC, um adaptador P3 → fone + microfone é ligado **cruzado**.

![Ligação bidirecional com cabo P3 combo e adaptador fone + microfone cruzado no PC B](https://raw.githubusercontent.com/Cafecanudo/sb-duplex/main/docs/img/ligacao-combo-p3.svg)

Nos exemplos, os IDs são fictícios: troque pelos do seu `--list-devices`.

| PC | Saída (`--out`) | Entrada (`--in`) |
|---|---|---|
| PC A | `k7p2` | `m3xa` |
| PC B | `d9fr` | `t4wq` |

### Listar dispositivos

```bash
# saídas e entradas de áudio com ID curto, mais os perfis e apelidos cadastrados
python -m soundbridge --list-devices

# o mesmo em JSON (um documento: outputs, inputs, aliases, profiles, config)
python -m soundbridge --list-devices --json

# só os IDs das entradas, para scripts (com jq)
python -m soundbridge --list-devices --json | jq -r '.inputs[].id'


# apelido para um device (use depois em --out/--in); "dados=" remove
python -m soundbridge --alias dados=k7p2
python -m soundbridge --alias retorno=m3xa
python -m soundbridge --alias dados=
```

Os devices aceitam ID curto (`k7p2`), apelido (`dados`), nome ou trecho do nome
(`"Alto Falante"`) e índice (que muda ao replugar). O ID curto vem do nome do device e não muda.

### Perfis de enlace

Um **perfil** guarda as configurações de um enlace com outro PC: placas, pasta de recepção, MCS,
compressão, calibração. **Não há padrões globais:** ou o comando usa um perfil (`--profile NOME`), ou
informa tudo na linha de comando. O mesmo nome é usado nas duas pontas, e cada PC guarda no próprio
arquivo de configuração o que é dele. Um PC com várias placas tem um perfil por enlace; perfis diferentes
podem repetir entradas e saídas.

```bash
# criar ou atualizar (um ou vários campos de uma vez; só os informados mudam)
python -m soundbridge --save-profile casa --out k7p2 --in m3xa --dir D:\recebidos
python -m soundbridge --save-profile casa --mcs 8 --zip-level 9

# apagar campos do perfil, ou o perfil inteiro
python -m soundbridge --save-profile casa --unset mcs,peak
python -m soundbridge --delete-profile casa

# usar: completa o que não for informado (o informado sempre vence)
python -m soundbridge --tx C:\relatorio.pdf --profile casa
python -m soundbridge --tx C:\relatorio.pdf --profile casa --out ggxj      # troca só a saída
python -m soundbridge --rx --profile casa --keep                            # pasta do perfil
```

Campos do perfil (os nomes das opções, sem `--`): `out`, `in`, `dir`, `bi`, `mcs`/`auto`,
`zip-level`/`no-zip`, `mono`, `band-high`, `peak`, `ack-peak`, `ack-band`, `shared`, `keep` e `force-repl`.
Ficam de fora a ação (`--tx`/`--rx`), a apresentação (`--json`, `--verbose`, `--no-progress`) e o que é de
cada envio (`--dest`, `--more`). `--auto` informado anula o `mcs` do perfil, e `--no-zip`, o `zip-level`.
Um campo ligado no perfil (ex.: `mono`) só sai com `--unset` ou outro perfil.

Nome do perfil: letras minúsculas, dígitos, `_` ou `-`, começando por letra (até 32). Perfil inexistente é
recusado com a lista dos cadastrados. Os padrões da versão anterior (`--save-defaults`) viram, uma vez, o
perfil `padrao`.

**O TX manda o perfil e o RX escolhe a pasta.** Com `--tx --profile casa`, o nome `casa` vai no manifesto do
arquivo. O RX grava na pasta escolhida nesta ordem, e o `--dest` do TX fica sempre relativo a ela:

1. a pasta explícita do RX (`--rx D:\fixo`);
2. o `dir` do perfil `casa` **deste PC** (vence o perfil de escuta: a mesma entrada pode receber de vários TX);
3. o `dir` do perfil com que o RX escuta (`--rx --profile lab`);
4. a pasta atual.

Se o perfil mandado não existir no RX (ou não tiver pasta), o arquivo cai na regra 3 ou 4 e o resultado avisa:
`perfil 'casa' não existe neste PC; gravado em …`. O TX só escolhe um nome; o caminho é sempre o que o dono
do RX cadastrou. Sem perfil no TX, a pasta segue como antes. Um RX de versão anterior ignora o perfil.

```bash
# PC B escuta numa entrada que recebe de dois PCs; cada um cai na sua pasta
PC B:  python -m soundbridge --save-profile notebook --dir D:\recebidos\notebook
PC B:  python -m soundbridge --save-profile desktop --dir D:\recebidos\desktop
PC B:  python -m soundbridge --rx --in t4wq --keep
PC A:  python -m soundbridge --tx C:\relatorio.pdf --profile notebook --dest docs   # D:\recebidos\notebook\docs
```

### Calibrar o enlace

Cada par de placas e cabos se comporta de um jeito: uma saída de linha aguenta estéreo em banda cheia e
modulação alta; uma entrada de microfone de notebook pode somar os canais (só mono), cortar acima de ~8 kHz,
saturar com sinal fraco e ter ganho automático (AGC). Achar à mão o `--mono`, o `--band-high`, o `--peak` e
o `--mcs` de cada sentido exige medir com tons e tentativas. A calibração faz isso sozinha, nos **dois
sentidos**, em poucos minutos.

**Como funciona.** Os dois cabos precisam estar ligados e o outro PC escutando com `--rx` e `--out`.

1. O PC que roda `--calibrate` envia uma **sondagem** (~5 s): uma escada de níveis de sinais conhecidos,
   diferentes no canal esquerdo e no direito.
2. O outro PC mede o que chegou e devolve o resultado pelo 2º cabo:
   - qual nível chega limpo, sem saturar;
   - se a entrada soma os canais (então mono);
   - até que frequência o sinal passa (banda);
   - o SNR, que define o MCS.

   Em seguida, ele sonda o sentido de volta, e o primeiro PC mede.
3. Com **`--calibrate-test`**, a escolha de cada sentido é **validada com transferências reais** (descartadas
   no RX). Se menos de 80% passar, a configuração desce um degrau (MCS menor, depois banda menor, depois
   mono) e é testada de novo. Os tamanhos de teste vêm de uma lista: arquivos pequenos validam rápido;
   arquivos grandes pegam problemas que só aparecem em quadros longos (AGC, rajadas da placa).
4. Com **`--profile NOME --calibrate-save`**, o resultado é gravado no **perfil** (pico, pico do ACK, mono,
   banda, MCS e os testes). O nome do perfil vai no relatório, e o outro PC grava o sentido dele no perfil
   **de mesmo nome** (se existir lá), senão no perfil com que está escutando (`--rx --profile NOME`). Daí em diante, `--tx --profile NOME` usa pico, mono, banda e MCS do perfil, e
   `--rx --profile NOME` usa o pico do ACK, quando você não os informa. O que for passado na linha de comando
   sempre vence.

Sem `--calibrate-save`, nada é gravado: a calibração só mostra o resultado e os comandos prontos.

```bash
# 1. Só medir e ver a recomendação (nada é testado nem salvo)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr --keep
PC A:  python -m soundbridge --calibrate --out k7p2 --in m3xa

# 2. Medir, validar com 10 transferências de 5 KB por sentido e salvar no perfil "casa" dos dois PCs
PC B:  python -m soundbridge --rx --profile casa --keep
PC A:  python -m soundbridge --calibrate --profile casa --calibrate-test --calibrate-save

# 3. Validar em vários tamanhos (do menor ao maior; a escolha final passa em todos)
PC A:  python -m soundbridge --calibrate --profile casa --calibrate-test 5k,200k,1mb --calibrate-save

# 4. Depois de salvo, enviar e receber sem informar pico, mono, banda nem MCS
PC B:  python -m soundbridge --rx --profile casa --keep
PC A:  python -m soundbridge --tx C:\relatorio.pdf --profile casa --bi
```

Exemplo de resultado (PC A com saída de linha; PC B com entrada de microfone de notebook):

```
este PC → outro: mono · 5 kHz · MCS 1 (QPSK r34) · pico 0.0055 · SNR 14 dB · testes 5k 10/10 OK
  aviso: ganho automático (AGC) ou compressão na entrada: desligue o AGC/melhorias de áudio
outro → este PC: estéreo · 22 kHz · MCS 9 (1024-QAM r34) · pico 0.2 · SNR 41 dB · testes 5k 10/10 OK
comandos:
  enviar daqui:        python -m python -m soundbridge --tx ARQUIVO --out k7p2 --in m3xa --bi --mono --band-high 5000 --peak 0.0055 --mcs 1
  receber no outro PC: python -m python -m soundbridge --rx PASTA --out <saída do outro PC> --in <entrada do outro PC> --keep --ack-peak 0.2
  enviar do outro PC:  python -m python -m soundbridge --tx ARQUIVO --out <saída do outro PC> --in <entrada do outro PC> --bi --band-high 22000 --peak 0.2 --mcs 9
  receber aqui:        python -m python -m soundbridge --rx PASTA --out k7p2 --in m3xa --keep --ack-peak 0.0055
calibração gravada no perfil 'casa' (e no outro PC, se ele estiver em --rx --profile)
```

**Tamanhos de teste** (`--calibrate-test`): `2k`, `5k`, `10k`, `20k`, `50k`, `100k`, `200k`, `500k`, `1mb`,
`5mb`, `10mb`; sem lista, `5k`. São 10 repetições até `100k`, 3 até `1mb` e 1 em `5mb` e `10mb`. Os grandes
levam minutos por tentativa (um `10mb` leva ~15 min no melhor caso), então use-os quando o objetivo for
enviar arquivos grandes.

**Avisos.** "O menor nível já chega saturado" pede para reduzir o volume de gravação ou desligar o
reforço (boost) da entrada. "AGC" indica ganho automático na entrada: a calibração ainda funciona, mas
limita o MCS e quadros longos podem falhar; o melhor é desligar o AGC/melhorias de áudio da entrada, ou
usar uma entrada de linha. Recalibre sempre que trocar de placa, porta, cabo ou volume.

### Calibrar só a ida (um cabo)

Com um só cabo, ou quando a volta não é confiável, `--calibrate-uni` mede **só o sentido de ida** e o resultado
aparece **na tela do PC que recebe**. Você digita os parâmetros no PC que envia.

1. O PC que recebe fica em `--rx` como sempre (o `--out` é opcional).
2. O PC que envia roda `--calibrate-uni`: toca a sondagem (~5 s) e termina.
3. O PC que recebe mede e mostra a escolha e os parâmetros para o TX.
4. No PC que envia, use esses parâmetros no `--tx`.

```bash
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq
PC A:  python -m soundbridge --calibrate-uni --out k7p2
```

No PC B:

```
calibração (só de ida) recebida: mono · 7 kHz · MCS 1 (QPSK r34) · pico 0.003 · SNR 26 dB
  aviso: ganho automático (AGC) ou compressão na entrada: desligue o AGC/melhorias de áudio
  no PC que enviou, use: --mono --band-high 7000 --peak 0.003 --mcs 1
```

E no PC A:

```bash
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2 --mono --band-high 7000 --peak 0.003 --mcs 1
```

Não há validação com transferências nem gravação de perfil: sem a volta, o PC A não sabe o que chegou.

### Enviar dados

O RX sobe primeiro (fica esperando); depois o TX envia. **PC A envia → PC B recebe.**

```bash
# 1. Um arquivo, um cabo (unidirecional)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2

# 2. Um arquivo, dois cabos (bidirecional: o RX confirma e o TX reenvia o que faltou)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2 --in m3xa --bi

# 3. Vários arquivos (lista entre aspas, separada por ";"; aceita curingas)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr
PC A:  python -m soundbridge --tx "C:\relatorio.pdf;C:\notas.txt;C:\fotos\*.jpg" --out k7p2 --in m3xa --bi

# 4. Uma pasta inteira, recursiva, dentro de uma subpasta do RX
#    C:\docs\a.txt → D:\recebidos\backup\a.txt ; C:\docs\sub\b.txt → D:\recebidos\backup\sub\b.txt
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr
PC A:  python -m soundbridge --tx C:\docs --dest backup --out k7p2 --in m3xa --bi

# 5. Arquivo numa subpasta do RX
PC A:  python -m soundbridge --tx C:\foto.jpg --dest fotos/2026 --out k7p2 --in m3xa --bi

# 6. Vários envios em sequência sem --keep no RX (--more avisa que vem outro)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr
PC A:  python -m soundbridge --tx C:\a.pdf --out k7p2 --in m3xa --bi --more
PC A:  python -m soundbridge --tx C:\b.pdf --out k7p2 --in m3xa --bi

# 7. MCS fixo (padrão: --auto)
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2 --in m3xa --bi --mcs 8

# 8. Sem compressão, ou compressão mais leve (padrão: comprime no nível 9)
PC A:  python -m soundbridge --tx C:\video.mp4 --out k7p2 --no-zip
PC A:  python -m soundbridge --tx C:\logs --out k7p2 --zip-level 6

# 9. Para uma entrada de microfone (mono, banda estreita, ganho alto) do outro lado
#    --mono, --band-high e --peak só no TX: o RX lê o modo e a banda no header
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2 --mono --band-high 10000 --peak 0.05 --mcs 8

# 10. Com perfil nos dois PCs (sem --out/--in nem pasta; ver "Perfis de enlace")
PC B:  python -m soundbridge --rx --profile casa --keep
PC A:  python -m soundbridge --tx C:\relatorio.pdf --profile casa --bi

# 11. Calibrar os dois sentidos, testar e salvar no perfil (ver "Calibrar o enlace")
PC B:  python -m soundbridge --rx --profile casa --keep
PC A:  python -m soundbridge --calibrate --profile casa --calibrate-test 5k,200k --calibrate-save
PC A:  python -m soundbridge --tx C:\relatorio.pdf --profile casa --bi             # usa a calibração do perfil

# 12. Saída para scripts e diagnóstico
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2 --json            # uma linha JSON por evento
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2 --verbose         # barra + uma linha por round
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2 --no-progress     # sem barra
```

### Receber dados

Do ponto de vista de quem recebe. **PC B recebe ← PC A envia.**

```bash
# 1. Receber um arquivo e sair (um cabo: só unidirecional)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2

# 2. Receber com confirmação (dois cabos: aceita uni e bi; o modo é detectado sozinho)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2 --in m3xa --bi

# 3. Ficar recebendo vários arquivos até Ctrl+C
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr --keep
PC A:  python -m soundbridge --tx C:\a.pdf --out k7p2 --in m3xa --bi
PC A:  python -m soundbridge --tx "C:\b.pdf;C:\c.zip" --out k7p2 --in m3xa --bi

# 4. Receber na pasta do perfil (ou na pasta atual, sem perfil)
PC B:  python -m soundbridge --save-profile casa --dir D:\recebidos
PC B:  python -m soundbridge --rx --profile casa --in t4wq --out d9fr

# 5. Receber de quem envia para entrada de microfone (o RX descobre mono e banda pelo header)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq
PC A:  python -m soundbridge --tx C:\relatorio.pdf --out k7p2 --mono --band-high 10000 --peak 0.05

# 6. Receber vários envios encadeados sem --keep (o TX usa --more; o RX sai no último)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr
PC A:  python -m soundbridge --tx "C:\a.pdf;C:\b.pdf;C:\c.pdf" --out k7p2 --in m3xa --bi

# 7. Responder a uma calibração do outro lado (o RX só precisa estar escutando, com --out)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr
PC A:  python -m soundbridge --calibrate --out k7p2 --in m3xa

# 8. Registrar a recepção em JSON (log para scripts)
PC B:  python -m soundbridge --rx D:\recebidos --in t4wq --out d9fr --keep --json > recepcao.jsonl

# 9. Com perfil (devices e pasta)
PC B:  python -m soundbridge --save-profile casa --out d9fr --in t4wq --dir D:\recebidos
PC B:  python -m soundbridge --rx --profile casa --keep
```

O arquivo é gravado com o mesmo nome (mais a subpasta do `--dest` do TX) e, sem `--force-repl` no TX, nunca sobrescreve:
`relatorio (1).pdf`. Ctrl+C encerra o RX (código de saída 130).

---

## Parâmetros da linha de comando

Uma ação por vez (`--tx`, `--rx`, `--calibrate`, `--calibrate-uni`, `--list-devices`, `--alias`, `--save-profile`,
`--delete-profile`). As tabelas abaixo agrupam os parâmetros por quem os usa.

### TX (envio): `python -m soundbridge --tx ...`

| Parâmetro | Descrição |
|---|---|
| `--tx "ARQ1;ARQ2;..."` | Envia um ou mais arquivos e/ou pastas, em sequência: um item, ou uma lista entre aspas separada por `;` (ex.: `--tx "C:\a.jpg;C:\b.pdf;C:\fotos\*.jpg;C:\docs"`). Aceita curingas, expandidos pelo próprio SoundBridge. **Pasta** envia todos os arquivos dentro dela, recursivamente, e o RX recria a estrutura de subpastas **a partir do conteúdo** (sem o nome da própria pasta): com `--tx C:\docs --dest backup`, `C:\docs\a.txt` vai para `<pasta do RX>\backup\a.txt` e `C:\docs\sub\b.txt` para `<pasta do RX>\backup\sub\b.txt`. Pastas vazias são ignoradas. Todos menos o último levam `--more`, então o RX recebe a lista inteira mesmo sem `--keep`. Tudo é conferido antes de começar; no fim, um resumo (`3/3 arquivos enviados`). Exige `--out`; com `--bi`, exige também `--in` (informados ou do perfil, `--profile`). |
| `--bi` | Bidirecional com ARQ: o RX confirma cada round e o TX reenvia só o que faltou. Exige os dois cabos (e o RX com `--out`). Padrão: unidirecional (um cabo, sem retransmissão, protegido por paridade Reed-Solomon). O RX detecta o modo sozinho. |
| `--auto` | Escolha automática do MCS (padrão). Uni: o maior MCS até 6 cujo áudio estimado cabe em 3 s, senão MCS 3 (na prática, MCS 6 até ~50 KB). Bi: MCS adaptativo começando em 7, que sobe e desce conforme as perdas de cada round. |
| `--mcs 0-9` | Fixa o MCS (desliga a adaptação no bi). Não combina com `--auto`. Tabela abaixo. |
| `--dest SUBPASTA` | Subpasta, relativa à pasta do RX, onde os arquivos serão gravados: `--tx C:\foto.jpg --dest ado/oash` com o RX em `--rx D:\recebidos` grava `D:\recebidos\ado\oash\foto.jpg`. Aceita `/` ou `\`; recusa `..` e drive. |
| `--more` | Avisa o RX que outro arquivo vem em seguida (o RX continua escutando mesmo sem `--keep` e encerra no primeiro arquivo sem `--more`, ou se o próximo não chegar em 2 min). Automático numa lista do `--tx`; manual para encadear vários comandos. |
| `--force-repl` | No RX, substitui o arquivo de mesmo nome (no mesmo `--dest`) em vez de gravar como `nome (1).ext`. Vai no manifesto; um RX de versão anterior ignora o pedido e grava com o sufixo. |
| `--no-zip` | Envia sem compressão. Padrão: todo arquivo é comprimido (deflate) antes de enviar e o RX grava o original, com o mesmo nome e hash. Texto, código e logs ficam 2–3× menores; JPG, ZIP, DOCX e vídeo quase não mudam. |
| `--zip-level 0-9` | Nível de compressão. Padrão 9 (máximo: o canal é lento e a CPU gasta é desprezível); 0 só embala, sem comprimir. |
| `--mono` | Dados em mono (o mesmo sinal nos dois canais), para entrada ou cabo mono. Metade da velocidade. O RX detecta pelo header, sem opção. |
| `--peak 0-1` | Pico do sinal de dados. Padrão: 0,9 no uni, 0,5 no bi (no bi a placa opera em full-duplex e distorce em nível alto). Use o valor recomendado por `--calibrate`. |

No TX, `--out` leva os dados e `--in` recebe o ACK (só no `--bi`).

### RX (recepção): `python -m soundbridge --rx ...`

| Parâmetro | Descrição |
|---|---|
| `--rx [PASTA]` | Recebe arquivos na pasta (criada se não existir) e espera até Ctrl+C. Sem pasta: o `dir` do perfil (`--profile`) ou, sem perfil, a pasta atual. O modo (uni/bi), o mono e a compressão são detectados sozinhos. O arquivo é gravado na pasta (mais a subpasta do `--dest` do TX) e não sobrescreve: `nome (1).ext`. Exige `--in`. |
| `--keep` | Continua escutando depois de cada arquivo. Uma transmissão nova encerra a anterior se ela ficou incompleta. |

No RX, `--in` recebe os dados e `--out` envia o ACK. Sem `--out`, o RX só recebe transferências unidirecionais.

### Comuns (TX e RX)

| Parâmetro | Descrição |
|---|---|
| `--out DEVICE` | Saída de áudio (TX: dados; RX: ACK). Aceita ID curto, apelido, nome (exato ou trecho único) ou índice. Sem ela, usa a do perfil (`--profile`). |
| `--in DEVICE` | Entrada de áudio (TX: ACK; RX: dados). Mesmos formatos de `--out`. Sem ela, usa a do perfil (`--profile`). |
| `--shared` | Usa o WASAPI em modo compartilhado. Padrão: exclusivo (evita o processamento de áudio do Windows, que degrada o sinal). |
| `--band-high HZ` | Limite superior da banda dos dados. Padrão: 22000. Valores da tabela: 5000, 7000, 8000, 10000, 14000, 18000, 22000. No TX define a banda; o RX procura todas as da tabela e lê a do header, sem precisar do parâmetro. Valor fora da tabela: o RX só acha o quadro se usar o mesmo `--band-high`. |
| `--ack-band HZ` | Limite superior da banda do ACK. Padrão: 7000 (entradas de microfone cortam em ~8 kHz). **Igual nos dois lados.** |
| `--ack-peak 0-1` | Pico do ACK. Padrão: o do perfil (gravado pela calibração); sem perfil, 0,02 (baixo porque entradas de microfone têm ganho alto). |
| (padrão) | Barra de progresso numa linha atualizada no lugar: percentual, blocos, KB/s, tempo restante (pela taxa recente) e round/MCS. Avisos (clip) aparecem acima dela. Só em terminal (inclui o Git Bash); com a saída redirecionada, volta às linhas por round. |
| `--verbose` | Mostra também uma linha por round (TX) e por quadro (RX). |
| `--no-progress` | Sem barra: uma linha por round/quadro (o formato anterior). |
| `--json` | Saída em JSON para scripts e integrações: uma linha por evento, todas com `"schema": 2` (versão do formato) e `event`. No TX e no RX, o evento `progress` substitui a barra a cada 0,5 s (`fraction`, `done`/`total` em blocos, `size`, `kib_per_s`, `eta_s`, `status`); `--no-progress` o desliga. O `result` do RX traz `profile` (o perfil que o TX mandou), `dir_source` (`explicit`, `manifest`, `listening` ou `cwd`) e `note` (aviso de perfil inexistente ou sem pasta). No `--list-devices`, um único documento com `schema`, `outputs`, `inputs` (cada device com `id`, `name`, `channels`, `samplerate`, `index`), `aliases`, `profiles` e `config`. |
| `-h`, `--help` | Mostra a ajuda. |
| `--version` | Mostra a versão instalada (ex.: `sb-duplex 1.3.0b0`) e sai. |

### Configuração e diagnóstico

| Parâmetro | Descrição |
|---|---|
| `--list-devices` | Lista saídas e entradas de áudio com ID curto e nome, os perfis e os apelidos. |
| `--alias NOME=DEVICE` | Cadastra um apelido (ex.: `--alias dados=k7p2`) para usar em `--out`/`--in`. `NOME=` remove. Nome: letras minúsculas, dígitos, `_` ou `-`, começando por letra. Fica no arquivo de configuração do usuário (`%APPDATA%\soundbridge\config.json` no Windows; `SOUNDBRIDGE_CONFIG` troca o caminho). |
| `--save-profile NOME` | Cria ou atualiza o perfil NOME com as opções informadas (um ou vários campos: `--out`, `--in`, `--dir`, `--mcs`, `--zip-level`...; devices validados antes de gravar). Só os informados mudam. Ver "Perfis de enlace". |
| `--unset CAMPOS` | Com `--save-profile`: remove campos do perfil (lista separada por vírgula, ex.: `mcs,peak`). |
| `--delete-profile NOME` | Remove o perfil. |
| `--profile NOME` | Usa o perfil NOME: completa as opções não informadas (o informado sempre vence). Vale em `--tx`, `--rx`, `--calibrate` e `--calibrate-uni`. No `--tx`, o nome vai ao RX, que grava na pasta do perfil de mesmo nome dele (ver "Perfis de enlace"). |
| `--dir PASTA` | Com `--save-profile`: pasta onde o `--rx` grava os arquivos. |
| `--calibrate` | Calibra o enlace nos **dois sentidos** contra um RX em escuta (`--rx` com `--out`; um RX sem `--out` só mede e mostra o resultado, como na `--calibrate-uni`): uma sondagem de ~5 s em cada sentido mede nível, mono/estéreo, banda e SNR e recomenda `--peak`, `--mono`, `--band-high` e `--mcs` para cada um, com os comandos de TX e RX. Avisa entrada saturada e AGC. Exige `--out` e `--in`. |
| `--calibrate-test [TAMANHOS]` | Com `--calibrate`: valida a escolha de cada sentido com transferências reais em bi (descartadas no RX), nos tamanhos da lista separada por vírgula: `2k`, `5k`, `10k`, `20k`, `50k`, `100k`, `200k`, `500k`, `1mb`, `5mb`, `10mb` (ex.: `--calibrate-test 5k,10k,10mb`; sem lista: `5k`). Repetições: 10 até 100k, 3 até 1mb, 1 em 5mb e 10mb. Testa do menor ao maior; aceita acima de 80% em cada tamanho; na 1ª reprovação desce um degrau (MCS, depois banda, depois mono) e recomeça, até 4 tentativas. Tamanhos grandes levam minutos (10mb ≈ 15 min por tentativa no melhor caso). |
| `--calibrate-uni` | Calibra **só a ida** (um cabo): toca a sondagem pela `--out`; o RX do outro PC (em `--rx`, com ou sem `--out`) mede e mostra na tela dele a escolha e os parâmetros (`--mono`, `--band-high`, `--peak`, `--mcs`) para usar no TX. Sem validação nem gravação. Exige `--out`. |
| `--calibrate-save` | Com `--calibrate` e `--profile`: grava o resultado no perfil (pico, pico do ACK, mono, banda, MCS e testes); o outro PC grava o sentido dele no perfil de mesmo nome (se existir lá) ou no de escuta. Sem ele, a calibração só mostra o resultado. |

### MCS (`--mcs`)

| MCS | Modulação | Código | MCS | Modulação | Código |
|---|---|---|---|---|---|
| 0 | QPSK | r1/2 | 5 | 64-QAM | r3/4 |
| 1 | QPSK | r3/4 | 6 | 256-QAM | r2/3 |
| 2 | 16-QAM | r1/2 | 7 | 256-QAM | r3/4 |
| 3 | 16-QAM | r3/4 | 8 | 1024-QAM | r2/3 |
| 4 | 64-QAM | r2/3 | 9 | 1024-QAM | r3/4 |

### Comportamento

- O arquivo recebido é gravado na pasta do RX (mais a subpasta do `--dest`, se houver) e não
  sobrescreve: `nome (1).ext`. O caminho recebido é saneado: drive, raiz, `.` e `..` são descartados e o
  resultado sempre fica dentro da pasta do RX.
- Com compressão (padrão), o `--auto` do uni escolhe o MCS pelo tamanho comprimido, e a velocidade
  mostrada é a efetiva (tamanho original ÷ tempo).
- O PC não suspende durante a transferência. O RX avisa quando a entrada satura (clip).
- Códigos de saída: 0 ok, 1 transferência falhou, 2 uso incorreto, 3 erro de device, 130 Ctrl+C.

---

## Limitações conhecidas (beta)

- Só Windows (WASAPI).
- Entradas de **microfone** (sobretudo em notebooks) costumam somar os canais, cortar acima de ~8 kHz e ter
  ganho automático (AGC): funcionam, mas bem mais devagar (mono, banda estreita, MCS baixo). Prefira entrada
  de linha e rode a calibração.
- A calibração com tamanhos grandes (`--calibrate-test 50k` ou mais) num sentido lento pode levar dezenas de
  minutos.

## Licença

MIT.
