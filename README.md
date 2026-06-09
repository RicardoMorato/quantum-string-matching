# Quantum String Matching

**Projeto Final — Computação Quântica | Grupo 5 (String Match)**
**CIn-UFPE · 2026.1**
**Equipe:** Ricardo Morato Rocha · Pedro Gabriel Alves da Silva

Implementação do algoritmo de busca de padrão quântico de **Niroula & Nam (2021)** em Qiskit, com análise de ruído em simuladores e execução em hardware IBM Quantum real.

---

## Estrutura do repositório

```
string_matching_project/
├── String_Matching_Quantico.ipynb   # Notebook principal
└── README.md                        # Este arquivo
```

---

## Configuração do ambiente

Crie e ative o ambiente conda antes de executar o notebook:

```bash
conda create -n quantum_string_matching python=3.11 -y
conda activate quantum_string_matching
pip install qiskit==2.4.1 qiskit-ibm-runtime==0.46.1 qiskit-aer==0.17.2 matplotlib pylatexenc
```

Para executar as células de hardware real IBM, exporte seu token de acesso antes de iniciar o Jupyter:

```bash
export IBM_TOKEN="seu_token_aqui"
jupyter notebook
```

O token pode ser gerado em [quantum.ibm.com](https://quantum.ibm.com/) após criar uma conta gratuita.

---

## A Apresentação

### Rodando localmente

Caso você queira ver a apresentação localmente da forma como foi pensada em ser consumida, instale as dependências do diretório `/presentation` com:

```bash
cd presentation
npm install
```

Depois, rode o comando:

```bash
npm run dev
```

Esse comando deve redirecionar você para `http://localhost:3030/1`, após isso, interaja com a apresentação normalmente.

### Em formato PPTX

Existe também um arquivo `.pptx` no diretório `/presentation`. Embora não tenha todas as animações adicionadas pelos autores, pode ser visualizada sem problemas.

---

## O Algoritmo

### Problema

Dados um texto binário `T` de comprimento `N` e um padrão binário `P` de comprimento `M`, o objetivo é encontrar todas as posições `i` onde `T[i..i+M-1] = P`.

- Solução clássica (KMP): O(N + M)
- Solução quântica (Niroula & Nam 2021): **Õ(√N)**: aceleração quadrática via busca de Grover

### Por que Niroula & Nam (2021)?

O artigo de **Ramesh & Vinay (2000)** descreve apenas a complexidade do oráculo, sem fornecer construção de circuito. Já **Niroula & Nam (2021)** apresentam uma implementação explícita em nível de portas quânticas, com o operador de deslocamento cíclico construído via portas CSWAP, o que permite seguir o algoritmo passo a passo.

---

### Arquitetura: Três Registradores

O circuito usa três registradores quânticos:

```
q_idx  (índice):   ⌈log₂(N-M+1)⌉ qubits   — superposição de todas as posições de início i
q_text (texto):    N qubits                  — codifica o texto completo T
q_pat  (padrão):   M qubits                  — codifica o padrão P
```

Para `T="1011"` (N=4) e `P="11"` (M=2), o circuito fica com 8 qubits:

```
q_idx  → ⌈log₂(3)⌉ = 2 qubits
q_text → 4 qubits
q_pat  → 2 qubits
─────────────────────────────
Total  → 8 qubits
```

---

### Pseudocódigo

```
ALGORITMO QuantumStringMatch(T[0..N-1], P[0..M-1]):

1. PREPARAÇÃO DE ESTADO
   a. Codificar T → |T⟩  (porta X em qubit j onde tⱼ = '1')
   b. Codificar P → |P⟩  (porta X em qubit j onde pⱼ = '1')
   c. H⊗k no registrador de índice → superposição uniforme sobre i ∈ {0,...,N-M}

2. DESLOCAMENTO CÍCLICO CONTROLADO
   Para cada bit b do registrador de índice:
     Aplicar deslocamento cíclico esquerdo do texto por 2^b posições,
     controlado no qubit q_idx[b].
   Os primeiros M qubits do texto deslocado correspondem a T[i..i+M-1].

3. COMPARAÇÃO XOR
   CNOT de q_text[j] → q_pat[j]  para j = 0..M-1
   Resultado: q_pat[j] = T[i+j] XOR P[j]
   Se T[i..i+M-1] = P → q_pat = |0...0⟩

4. ORÁCULO DE FASE
   Inverte a fase de |0...0⟩ no registrador de padrão:
     X em todos os M qubits de q_pat
     H → MCX → H no último qubit  (equivalente a multi-controlled Z)
     X em todos os M qubits de q_pat

5. DESCOMPUTA
   Reverter XOR (passo 3): CNOT de q_text[j] → q_pat[j] novamente
   Reverter deslocamento cíclico (passo 2):
     A inversa do deslocamento esquerdo por c é o deslocamento esquerdo por (N-c) % N

6. DIFUSOR DE GROVER no registrador de índice
   H⊗k → X⊗k → MCZ → X⊗k → H⊗k

7. REPETIR passos 2–6 por ⌊π/4 · √(N-M+1)⌋ iterações.

8. MEDIR q_idx → posições onde P ocorre em T.
```

---

### Tabelas-verdade

#### 1. Comparação por XOR (Porta CNOT — Passo 3)

A CNOT usa `q_text[j]` como controle e `q_pat[j]` como alvo. O resultado armazenado em `q_pat[j]` é `T[i+j] XOR P[j]`:

| `q_text[j]` (texto deslocado) | `q_pat[j]` (padrão) | Saída em `q_pat[j]` | Interpretação |
|:---:|:---:|:---:|:---|
| 0 | 0 | **0** | bits iguais (match) |
| 0 | 1 | **1** | bits diferentes (mismatch) |
| 1 | 0 | **1** | bits diferentes (mismatch) |
| 1 | 1 | **0** | bits iguais (match) |

Quando todos os M bits resultam em 0, `q_pat = |0...0⟩`, indicando match completo na posição i.

#### 2. Oráculo de Fase (Passo 4 — padrão de M=2 bits)

O oráculo aplica fase −1 exclusivamente ao estado `|00⟩`, que representa um match:

| Estado de `q_pat` (após XOR) | Fase aplicada | Interpretação |
|:---:|:---:|:---|
| `\|00⟩` | **−1** | match completo em i |
| `\|01⟩` | +1 | mismatch no bit 0 |
| `\|10⟩` | +1 | mismatch no bit 1 |
| `\|11⟩` | +1 | mismatch em ambos os bits |

#### 3. Deslocamento Cíclico Esquerdo (N=4 qubits)

Após o deslocamento por `c`, a posição j do registrador passa a conter `T[(j+c) % N]`. Os primeiros M=2 qubits expõem a janela `T[c..c+M-1]`:

| Shift c | Pos. 0 | Pos. 1 | Pos. 2 | Pos. 3 | Janela `[0..1]` |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 0 | T[0] | T[1] | T[2] | T[3] | T[0] T[1] |
| 1 | T[1] | T[2] | T[3] | T[0] | T[1] T[2] |
| 2 | T[2] | T[3] | T[0] | T[1] | T[2] T[3] |
| 3 | T[3] | T[0] | T[1] | T[2] | T[3] T[0] |

---

### Exemplo completo: `T="1011"`, `P="11"`

#### Verificação clássica das posições

| i | Janela `T[i..i+1]` | Match com `P="11"`? |
|:---:|:---:|:---:|
| 0 | "10" | ✗ |
| 1 | "01" | ✗ |
| 2 | "11" | **✓** |

#### Trace quântico

**Preparação de estado:**
```
q_idx  = H²|00⟩ = (|00⟩ + |01⟩ + |10⟩ + |11⟩) / 2
q_text = |1011⟩   (portas X nos qubits 0, 2, 3)
q_pat  = |11⟩     (portas X nos qubits 0, 1)
```

**Após deslocamento cíclico controlado por i:**

| Componente | Texto deslocado | Janela `[0..1]` |
|:---:|:---:|:---:|
| `\|00⟩` (i=0) | `\|1011⟩` | "10" |
| `\|01⟩` (i=1) | `\|0111⟩` | "01" |
| `\|10⟩` (i=2) | `\|1110⟩` | "11" |
| `\|11⟩` (i=3) | `\|1101⟩` | "11" ← wrap-around, posição inválida |

A componente i=3 existe porque `⌈log₂(3)⌉ = 2` bits geram superposição sobre {0,1,2,3}, mas a posição 3 excede o espaço válido (N-M = 2). O deslocamento por 3 cria um match "falso" por wrap-around.

**Após XOR com `P="11"`:**

| `\|i⟩` | `q_pat` após XOR | Oráculo aplicado? |
|:---:|:---:|:---:|
| `\|00⟩` | "10" XOR "11" = `\|01⟩` | não |
| `\|01⟩` | "01" XOR "11" = `\|10⟩` | não |
| `\|10⟩` | "11" XOR "11" = `\|00⟩` | **sim → fase −1** |
| `\|11⟩` | "11" XOR "11" = `\|00⟩` | sim (espúrio) → fase −1 |

**Correção de posições espúrias:** a componente i=3 recebe um phase-flip adicional via `_phase_flip_state`, cancelando a marca indevida do oráculo.

**Após o difusor de Grover:** a amplitude de `|10⟩` (i=2) é amplificada em relação às demais. A medição retorna `"10"` com alta probabilidade, e `int("10", 2) = 2` — a posição correta do match.

---

### Funções do código

| Função | Célula | Descrição |
|:---|:---:|:---|
| `encode_string(qc, reg, bits)` | 5 | Codifica uma string binária no registrador via portas X |
| `controlled_cyclic_shift(qc, ctrl, reg, c)` | 6 | Versão controlada: substitui cada SWAP por um CSWAP (Fredkin) |
| `build_grover_oracle(qc, pat_reg)` | 7 | Oráculo de fase — inverte a fase de `\|0...0⟩` no registrador de padrão |
| `build_grover_diffuser(qc, idx_reg)` | 8 | Difusor de Grover no registrador de índice |
| `_phase_flip_state(qc, idx_reg, i, n)` | 9 | Inverte a fase do estado `\|i⟩` — usado para cancelar matches espúrios por wrap-around |
| `build_string_match_circuit(text, pattern, n_iter)` | 9 | Monta o circuito completo; com `n_iter=None`, o número de iterações é calculado a partir do número real de matches |
| `interpret_counts(counts, n_idx, search_space)` | 11 | Converte bitstrings do Qiskit em posições inteiras e filtra posições fora do espaço válido |

---

### Detalhes de implementação

#### Inverso do deslocamento cíclico

A inversa de um deslocamento cíclico esquerdo por `c` posições é o deslocamento esquerdo por `(N - c) % N`, não o mesmo deslocamento aplicado novamente. Aplicar os mesmos SWAPs duas vezes só desfaz o deslocamento em ciclos de comprimento 2; em ciclos maiores, o registrador retorna a uma permutação incorreta.

```python
# Descomputa com o inverso correto:
inv_shift = (N - shift_amount) % N
controlled_cyclic_shift(qc, q_idx[bit], q_text, inv_shift)
```

#### Leitura de resultados no Qiskit ≥ 1.0

A partir do Qiskit 1.0, `get_counts()` retorna bitstrings em formato big-endian (bit de maior índice à esquerda). Isso significa que `int(bitstring, 2)` dá diretamente o inteiro correto, sem necessidade de inverter a string. Em versões anteriores do Qiskit (< 1.0), o comportamento era o oposto.

#### Matches espúrios por wrap-around

Quando `N-M+1` não é potência de 2, o registrador de índice em superposição inclui estados inválidos `i ≥ N-M+1`. O deslocamento cíclico por essas posições pode criar matches por wrap-around, que o oráculo marca incorretamente. Para cada posição inválida que produz um match falso, aplicamos `_phase_flip_state` após o oráculo, desfazendo a marca antes do difusor.

---

## Referências

1. Niroula, P., Nam, Y. *A quantum algorithm for string matching.* npj Quantum Information **7**, 37 (2021). https://doi.org/10.1038/s41534-021-00369-3
2. Ramesh, H., Vinay, V. *String matching in Õ(√n + √m) quantum time.* arXiv:quant-ph/0011049 (2000).
3. Grover, L. *A fast quantum mechanical algorithm for database search.* Proc. 28th ACM STOC (1996).
