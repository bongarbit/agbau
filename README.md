# Agbau

Instrumento eletrônico de percussão afro-brasileira desenvolvido pelo [BongarBit](https://www.instagram.com/bongarbit/).

[![Assista ao vídeo](https://img.youtube.com/vi/48ikO1KuL8A/0.jpg)](https://youtu.be/48ikO1KuL8A)

*Ninho Brown, do Bongarbit, tocando o Agbau no Centro Cultural Grupo Bongar - Guitinho da Xambá em Olinda/PE*

---

## Como funciona

O Agbau é composto por 16 botões (organizados numa matriz 4x4) divididos em **8 botões de percussão** e **8 botões de baixo**. Um microcontrolador Arduino lê os botões e envia mensagens MIDI USB para um computador Bela, que reproduz os samples de áudio em tempo real com envelope ADSR.

```
Botões (4×4)
     │
     ▼
Arduino Pro Micro  ──── MIDI USB ────►  Bela (ARM)  ──►  Saída de Áudio
(ATmega32u4)                            Engine C++
     │
Pedal de Looper
     │
LEDs de Status
```

---

## Estrutura do Repositório

```
agbau/
├── Arduino/
│   ├── agbau/
│   │   └── agbau.ino          # Firmware principal do Arduino
│   └── libraries/             # Bibliotecas necessárias (Keypad, MIDIUSB, Bounce2)
│
├── Bela/
│   ├── agbau-sampler-poly/  # Variante polyfônica (principal)
│   │   ├── render.cpp         # Loop de áudio e callbacks MIDI
│   │   ├── Sampler.cpp/h      # Classe de reprodução de samples com ADSR
│   │   ├── ADSR.cpp/h         # Envelope Attack-Decay-Sustain-Release
│   │   ├── Ramp.cpp/h         # Rampa de interpolação suave
│   │   └── samples/           # Arquivos de áudio .wav
│   │
│   └── agbau-sampler-midi-thru/  # Variante com MIDI passthrough
│
├── Circuito/
│   ├── Agbaixo_PCB.kicad_pcb  # Layout da placa de circuito
│   ├── Agbaixo_PCB.kicad_sch  # Esquemático elétrico
│   └── Agbaixo_BOM.csv        # Lista de componentes (Bill of Materials)
│
└── Estrutura Física/
    ├── PCB_Base/              # Base e pés para a PCB (.stl, .f3d)
    └── Cover_Holders/         # Suportes e tampa do instrumento (.stl, .f3d)
```

---

## Firmware do Arduino (`Arduino/agbau/agbau.ino`)

O firmware roda no **Arduino Pro Micro (ATmega32u4)** e é responsável por:

### Leitura dos botões
A matriz 4×4 é lida com a biblioteca `Keypad` com debounce integrado. Os 16 botões são divididos em:
- **Botões 1–8**: baixos (notas MIDI 60–67)
- **Botões 9–16**: percussão (notas MIDI 68–75)

### Modos de operação

**Modo Nota** (padrão): cada botão disparado envia um `noteOn` e ao soltar envia `noteOff` via MIDI USB no canal 9.

**Modo Repeat (Tap Tempo)**: ativado ao pressionar o botão de modo. O último botão pressionado é repetido automaticamente no BPM calculado pelo intervalo entre dois toques. A divisão rítmica é determinada pelo último botão de baixo pressionado, podendo ser: `2, 3, 4, 6, 8, 12, 16` ou `32` por compasso. Um LED pisca no tempo do BPM.

### Looper MIDI
Controlado por um pedal físico:

| Ação do pedal | Resultado |
|---|---|
| 1º toque | Inicia gravação |
| 2º toque | Para gravação e inicia playback |
| 3º toque | Inicia overdub (nova camada) |
| 4º toque | Para overdub |
| Segurar >1s | Apaga o loop |

- Suporta até **3 camadas de overdub**
- Cada camada armazena até **64 mensagens MIDI** com timestamp
- Um LED indica o status: aceso durante gravação, piscando durante playback

> **Atenção:** o modo de overdub ainda apresenta bugs conhecidos. A contagem de botões pressionados pode ficar inconsistente entre camadas e o `noteOff` pode ser disparado imediatamente após o `noteOn` em alguns casos. Use o overdub com cautela.

---

## Engine de Áudio Bela (`Bela/agbau-sampler-poly/`)

Roda no **Bela** (computador embarcado baseado em ARM + BeagleBone Black) com latência de áudio ultra-baixa (~1ms).

### Mapeamento de samples

| Tipo | MIDI | Arquivos de sample |
|---|---|---|
| Baixo 1–8 | 60–67 | `b1.wav` – `b8.wav` |
| Percussão 1–8 | 68–75 | `p1.*` – `p8.*` |

Algumas percussões possuem múltiplos samples que são selecionados aleatoriamente a cada toque, adicionando variação expressiva.

### Classe `Sampler`
Cada um dos 16 samplers é independente e suporta:
- **Múltiplos arquivos WAV** por nota (seleção aleatória entre takes)
- **Envelope ADSR** com parâmetros configuráveis (attack, decay, sustain, release)
- **Modo Loop**: o sample é repetido enquanto a nota estiver ativa (usado nos ganzás — percussões 3 e 7)
- **Release on Note Off**: configura se o release do ADSR é disparado ao soltar a tecla

### Configuração de cada instrumento

```
Baixos (1–8):     ADSR(0.01s, 0.0s, 1.0, 1.0s) — ataque rápido, sustain total
Percussões (1–8): ADSR(0.01s, 3.0s, 0.0, 3.0s) — ataque rápido, decay longo
Ganzás (3 e 7):   ADSR(0.01s, 0.0s, 1.0, 0.1s) — modo loop ativado
```

---

## Como instalar o firmware no Arduino

1. Baixe e instale o **Arduino IDE 1.8.x** (versão legado) em [arduino.cc/en/software](https://www.arduino.cc/en/software)

2. Copie todas as pastas de `Arduino/libraries/` para o diretório `libraries/` da instalação do Arduino

3. Adicione o suporte à placa SparkFun: vá em `File > Preferences` e adicione a URL abaixo em *Additional Board Manager URLs*:
   ```
   https://raw.githubusercontent.com/sparkfun/Arduino_Boards/main/IDE_Board_Manager/package_sparkfun_index.json
   ```

4. Vá em `Tools > Board > Boards Manager`, busque por **SparkFun AVR Boards** e instale

5. Selecione a placa: `Tools > Board > Arduino Pro Micro`

6. Selecione o processador: `Tools > Processor > ATmega32U4 (5V, 16MHz)`

7. Conecte o Arduino via USB, selecione a porta em `Tools > Port`

8. Abra `Arduino/agbau/agbau.ino` e clique em **Upload**

---

## Como instalar o projeto no Bela

Acesse a interface web do Bela pelo navegador em **`http://bela.local`** (ou `192.168.6.2` / `192.168.7.2` caso o endereço anterior não funcione).

### Opção 1 — Pela IDE web (upload arquivo por arquivo)

1. Na aba **Project Explorer**, clique em **New Project**, selecione **C++** e dê um nome ao projeto
2. Faça upload de cada arquivo de `Bela/agbau-sampler-poly/` usando o botão **Upload file**
3. Crie uma pasta `samples/` no projeto e faça upload dos arquivos `.wav` de `Bela/agbau-sampler-poly/samples/`

### Opção 2 — Via SSH/SCP (recomendado para projetos com muitos arquivos)

```bash
# Copie os arquivos C++ para o projeto no Bela
scp Bela/agbau-sampler-poly/*.{cpp,h} root@bela.local:~/Bela/projects/agbau/

# Copie os samples
scp Bela/agbau-sampler-poly/samples/*.wav root@bela.local:~/Bela/projects/agbau/samples/
```

### Finalizando

Verifique se a porta MIDI está correta em `render.cpp` (use `aconnect -l` no terminal SSH do Bela para confirmar):
```cpp
const char* gMidiPort0 = "hw:1,0,0";
```

Conecte o Arduino ao Bela via USB e clique em **Run** na interface web.

---

## Estrutura física

Os arquivos de impressão 3D estão em `Estrutura Física/` nos formatos `.stl` (pronto para fatiar) e `.f3d` (editável no Fusion 360):

- **`PCB_Base/`**: base que acomoda a placa de circuito, com 4 pés (`pe1`–`pe4`)
- **`Cover_Holders/`**: 6 peças que formam os suportes laterais e tampa do instrumento

> **Em desenvolvimento:** as peças 3D ainda estão em fase de testes. No instrumento atual, tanto a base da placa de circuito quanto o suporte do tampo são feitos manualmente em madeira.

---

## Dependências

| Componente | Tecnologia |
|---|---|
| Firmware | C++ / Arduino IDE 1.8.x |
| Bibliotecas Arduino | Keypad, MIDIUSB, Bounce2 |
| Engine de áudio | C++ / Bela SDK |
| Esquemático | KiCad |
| Modelagem 3D | Autodesk Fusion 360 |
