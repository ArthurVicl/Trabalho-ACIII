# Sequências S1–S5 — Trabalho Prático: Pipeline Escalar e Superescalar
## Arquitetura de Computadores III

---

## Visão Geral

| Arquivo               | Sequência | Hazard Predominante                     | Objetivo Pedagógico                              |
|-----------------------|-----------|-----------------------------------------|--------------------------------------------------|
| `s1_raw_chain.s`      | S1        | Data hazard RAW (dist. 1)               | Stalls vs. forwarding                            |
| `s2_loop_raw_branch.s`| S2        | RAW em loop + branch hazard             | Interação data hazard / controle; predição       |
| `s3_load_use.s`       | S3        | Load-use (irredutível)                  | Único stall não eliminável por forwarding        |
| `s4_war_waw.s`        | S4        | WAR (anti-dep.) + WAW (dep. de saída)  | Base para renomeação de registradores (Parte B)  |
| `s5_branches.s`       | S5        | Múltiplos desvios condicionais mistos   | Comparação preditores estático / 1-bit / 2-bits  |

Todas as sequências estão em **assembly RISC-V** (compatível com RIPES e gem5).
Versões **MIPS** comentadas no final de cada arquivo (compatível com WinDLX e MARS).

---

## Como Usar no RIPES (Recomendado)

**Instalação:**
```bash
# Linux (AppImage)
wget https://github.com/mortbopet/Ripes/releases/latest/download/Ripes-linux.AppImage
chmod +x Ripes-linux.AppImage && ./Ripes-linux.AppImage

# Windows/macOS: baixar o instalador em
# https://github.com/mortbopet/Ripes/releases
```

**Fluxo de trabalho para cada sequência:**

1. Abrir RIPES → aba **Processor** → selecionar **"5-Stage Processor"**
2. Menu **File > Load Program** → selecionar o arquivo `.s`
3. Executar em modo **Step** para avançar ciclo a ciclo
4. Observar o painel **Pipeline Diagram** (IF/ID/EX/MEM/WB de cada instrução)
5. No painel **Processor Settings**, alternar as opções:
   - ☐ **Enable Forwarding** — desmarque para ver stalls sem forwarding
   - **Branch Predictor** — alternar entre Always Not-Taken, 1-bit, 2-bit

**Métricas a coletar** (painel Statistics):
- Clock cycles (total de ciclos)
- Instructions (instruções efetivas)
- **CPI** = Clock cycles / Instructions
- Stalls (RAW stalls e Branch stalls separadamente)

---

## Como Usar no MARS (MIPS)

1. Abrir o MARS → **File > Open** → selecionar o arquivo `.s`
2. Na seção comentada no final de cada arquivo, copiar o código MIPS
   e colar em um novo arquivo `.s`
3. **Settings > Simulate > Delayed branching** → ativar para ver branch delay slots
4. Executar em modo **Step** e observar o painel **Execute**

**Nota:** O MARS não exibe diagrama de pipeline visualmente.
Use o **Instruction Count** e **Clock Cycles** para calcular o CPI.

---

## Roteiro de Coleta de Dados (Parte A do TP)

Execute cada sequência (S1–S5) nas 5 configurações abaixo e preencha a tabela:

| Config. | Forwarding | Predição de Desvio    |
|---------|------------|-----------------------|
| (a)     | ✗          | Always Not-Taken       |
| (b)     | ✓          | Always Not-Taken       |
| (c)     | ✓          | Estática Always NT     |
| (d)     | ✓          | Dinâmica 1-bit         |
| (e)     | ✓          | Dinâmica 2-bits        |

**Tabela de CPI a preencher:**

```
          │  (a)  │  (b)  │  (c)  │  (d)  │  (e)  │
──────────┼───────┼───────┼───────┼───────┼───────┤
S1        │       │       │       │       │       │
S2        │       │       │       │       │       │
S3        │       │       │       │       │       │
S4        │       │       │       │       │       │
S5        │       │       │       │       │       │
```

---

## Dependências por Sequência (Resumo)

### S1 — RAW em Cadeia
```
addi t0, zero, 10
add  t1, t0, t0     ← RAW em t0 (dist=1)
add  t2, t1, t0     ← RAW em t1 (dist=1)
add  t3, t2, t1     ← RAW em t2 (dist=1)
add  t4, t3, t2     ← RAW em t3 (dist=1)
add  t5, t4, t3     ← RAW em t4 (dist=1)
sub  t6, t5, t4     ← RAW em t5 (dist=1)
```
→ **5 RAW com dist=1** | Stalls sem forward: 10 | Stalls com forward: 0

### S2 — Loop (soma de array)
```
loop:
  lw   x13, 0(x12)  ← produz x13
  add  x10, x10, x13← RAW load-use (dist=1) ★ irredutível
  addi x12, x12, 4
  addi x11, x11, -1 ← RAW em x12 (dist=1)
  bne  x11, zero, loop ← RAW em x11 (dist=1) + branch taken 7×
```
→ **8 iterações** | Padrão de branch: TTTTTTTTN

### S3 — Load-Use
```
lw   x10, 0(x5)     ← carrega
add  x11, x10, x6   ← RAW load-use ★ (stall mesmo com forwarding)
lw   x12, 4(x5)     ← carrega
sub  x13, x12, x6   ← RAW load-use ★
lw   x14, 8(x5)     ← carrega
add  x15, x14, x11  ← RAW load-use ★
```
→ **3 load-use** | 3 stalls irredutíveis com forwarding

### S4 — WAR e WAW
```
I1: add x1, x2, x3   escreve x1
I2: sub x4, x1, x5   RAW I1→I2 (x1)
I3: add x2, x6, x7   WAR I1→I3 (x2); WAW eventual
I4: mul x1, x4, x8   RAW I2→I4 (x4); WAW I1→I4 (x1)
I5: add x5, x1, x2   RAW I4→I5 (x1); RAW I3→I5 (x2); WAR I2→I5 (x5)
I6: sub x4, x5, x9   RAW I5→I6 (x5); WAW I2→I6 (x4); WAR I4→I6 (x4)
```

### S5 — Desvios Mistos (array: -3, 5, 0, 2, -1, 0, 7, -4, 0, 3)
```
blt  x11, zero, neg   padrão: T N N N T N N T N N  (30% taken)
beq  x11, zero, zer   padrão: N T N N T N N T N    (33% taken)
j    fim_class         sempre taken (incondicional)
j    loop              sempre taken (incondicional)
```

---

## Estrutura de Arquivos

```
.
├── README.md
├── s1_raw_chain.s          ← S1: cadeia RAW com dist=1
├── s2_loop_raw_branch.s    ← S2: loop com RAW + branch
├── s3_load_use.s           ← S3: load-use irredutível
├── s4_war_waw.s            ← S4: anti-dep. e dep. de saída
└── s5_branches.s           ← S5: múltiplos desvios condicionais
```

---

## Dependências de Software

| Ferramenta  | Versão recomendada | Link |
|-------------|-------------------|------|
| RIPES       | ≥ 2.2.6           | https://github.com/mortbopet/Ripes/releases |
| MARS        | 4.5               | http://courses.missouristate.edu/KenVollmar/MARS/ |
| gem5        | 23.x+             | https://www.gem5.org |

Para gráficos e análise dos dados coletados:
```bash
pip install matplotlib pandas numpy
```
