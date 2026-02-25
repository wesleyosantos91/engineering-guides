# 23. Bloom Filters

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Avançado — diferencial em entrevistas de System Design  
> **Complexidade:** Média

---

## Definição

**Bloom Filter** é uma estrutura de dados **probabilística** e **space-efficient** que responde à pergunta **"este elemento está no set?"** com duas respostas possíveis:

- ✅ **"Definitivamente NÃO está"** — 100% de certeza (zero falsos negativos)
- ⚠️ **"Provavelmente está"** — pode ser um falso positivo

Nunca dá **falso negativo**, mas pode dar **falso positivo**. Em troca, usa **ordens de magnitude menos memória** do que armazenar os elementos reais.

---

## Por Que é Importante?

- **Evita lookups desnecessários** em disco, banco de dados ou rede
- **Usado massivamente em Big Techs** — Google Chrome, Cassandra, HBase, Medium
- **Space-efficient** — milhões de elementos em poucos KB/MB
- **Conceito frequente em entrevistas** — demonstra conhecimento de trade-offs probabilísticos

---

## Como Funciona

### Estrutura Básica

```
Bloom Filter = Bit Array de tamanho m + k hash functions

Inicialização (tudo zero):
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9  10  11
```

### Operação: Add (Inserir)

```
add("hello") com k=3 hash functions:
  h1("hello") % 12 = 1
  h2("hello") % 12 = 5
  h3("hello") % 12 = 9

┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 0 │ 0 │ 0 │ 1 │ 0 │ 0 │ 0 │ 1 │ 0 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1↑  2   3   4   5↑  6   7   8   9↑ 10  11

add("world"):
  h1("world") % 12 = 3
  h2("world") % 12 = 5  ← já era 1
  h3("world") % 12 = 11

┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 0 │ 1 │ 0 │ 1 │ 0 │ 0 │ 0 │ 1 │ 0 │ 1 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3↑  4   5   6   7   8   9  10  11↑
```

### Operação: Check (Verificar)

```
check("hello"):
  h1("hello") % 12 = 1  → bit[1] = 1 ✓
  h2("hello") % 12 = 5  → bit[5] = 1 ✓
  h3("hello") % 12 = 9  → bit[9] = 1 ✓
  → Todos 1 → "PROVAVELMENTE SIM" ✓

check("foo"):
  h1("foo") % 12 = 2   → bit[2] = 0 ✗
  → Pelo menos um 0 → "DEFINITIVAMENTE NÃO" ✓

check("bar"):
  h1("bar") % 12 = 1   → bit[1] = 1 ✓  (setado por "hello")
  h2("bar") % 12 = 3   → bit[3] = 1 ✓  (setado por "world")
  h3("bar") % 12 = 11  → bit[11] = 1 ✓ (setado por "world")
  → Todos 1 → "PROVAVELMENTE SIM"
  → Mas "bar" NUNCA foi adicionado! ← FALSO POSITIVO
```

---

## Parâmetros e Matemática

### Fórmula do False Positive Rate (FPR)

$$FPR \approx \left(1 - e^{-kn/m}\right)^k$$

Onde:
- $m$ = tamanho do bit array
- $k$ = número de hash functions
- $n$ = número de elementos inseridos

### Número Ótimo de Hash Functions

$$k_{optimal} = \frac{m}{n} \cdot \ln 2 \approx 0.693 \cdot \frac{m}{n}$$

### Tamanho Necessário do Bit Array

$$m = -\frac{n \cdot \ln(FPR)}{(\ln 2)^2}$$

### Tabela de Referência

| Elementos (n) | FPR Target | Bits/elem (m/n) | Hash funcs (k) | Memória Total |
|---------------|------------|------------------|-----------------|---------------|
| 1 milhão | 1% | 9.6 bits | 7 | ~1.2 MB |
| 1 milhão | 0.1% | 14.4 bits | 10 | ~1.8 MB |
| 10 milhões | 1% | 9.6 bits | 7 | ~12 MB |
| 100 milhões | 1% | 9.6 bits | 7 | ~120 MB |
| 1 bilhão | 1% | 9.6 bits | 7 | ~1.2 GB |

```
Comparação de memória:

Armazenar 1M URLs (avg 100 bytes):
  HashSet:      ~100 MB + overhead ≈ 200 MB
  Bloom Filter: ~1.2 MB (para 1% FPR)
  
  → 166x menos memória! ✓
```

---

## Implementação

### Python (do zero)

```python
import math
import mmh3  # MurmurHash3

class BloomFilter:
    def __init__(self, expected_items: int, fp_rate: float = 0.01):
        # Calcula parâmetros ótimos
        self.fp_rate = fp_rate
        self.size = self._optimal_size(expected_items, fp_rate)
        self.num_hashes = self._optimal_hashes(self.size, expected_items)
        self.bit_array = bytearray(math.ceil(self.size / 8))
        self.count = 0
    
    @staticmethod
    def _optimal_size(n: int, p: float) -> int:
        """m = -(n * ln(p)) / (ln(2))^2"""
        return int(-n * math.log(p) / (math.log(2) ** 2))
    
    @staticmethod
    def _optimal_hashes(m: int, n: int) -> int:
        """k = (m/n) * ln(2)"""
        return max(1, int((m / n) * math.log(2)))
    
    def _get_positions(self, item: str) -> list:
        """Gera k posições usando double hashing"""
        positions = []
        for i in range(self.num_hashes):
            # Double hashing: h(i) = h1 + i*h2
            hash_val = mmh3.hash(item, seed=i) % self.size
            positions.append(abs(hash_val))
        return positions
    
    def _set_bit(self, position: int):
        byte_idx = position // 8
        bit_idx = position % 8
        self.bit_array[byte_idx] |= (1 << bit_idx)
    
    def _get_bit(self, position: int) -> bool:
        byte_idx = position // 8
        bit_idx = position % 8
        return bool(self.bit_array[byte_idx] & (1 << bit_idx))
    
    def add(self, item: str):
        """Adiciona item ao filter"""
        for pos in self._get_positions(item):
            self._set_bit(pos)
        self.count += 1
    
    def might_contain(self, item: str) -> bool:
        """Verifica se item PODE estar no filter"""
        return all(self._get_bit(pos) for pos in self._get_positions(item))
    
    def __contains__(self, item: str) -> bool:
        return self.might_contain(item)
    
    def __len__(self) -> int:
        return self.count

# Uso
bf = BloomFilter(expected_items=1_000_000, fp_rate=0.01)

# Adiciona URLs
bf.add("https://example.com/page1")
bf.add("https://example.com/page2")

# Verifica
print("page1" in bf)    # True  (correto)
print("page3" in bf)    # False (correto, provavelmente)
print(f"Size: {len(bf.bit_array)} bytes")  # ~1.2 MB
print(f"Hash functions: {bf.num_hashes}")   # 7
```

### Redis Bloom Filter

```redis
# Redis module: RedisBloom (BF.*)

# Criar com parâmetros específicos
BF.RESERVE user_emails 0.001 10000000
# FPR=0.1%, capacity=10M

# Adicionar
BF.ADD user_emails "user@example.com"
BF.MADD user_emails "a@b.com" "c@d.com" "e@f.com"

# Verificar
BF.EXISTS user_emails "user@example.com"  # (integer) 1
BF.EXISTS user_emails "unknown@test.com"  # (integer) 0 (definitivamente não)
BF.MEXISTS user_emails "a@b.com" "x@y.com"  # 1) 1  2) 0

# Info
BF.INFO user_emails
# Capacity: 10000000
# Size: 17882316 (bytes)
# Number of filters: 1
# Number of items inserted: 4
```

### Guava (Java)

```java
import com.google.common.hash.BloomFilter;
import com.google.common.hash.Funnels;

// Criar bloom filter para 1M strings com 1% FPR
BloomFilter<String> filter = BloomFilter.create(
    Funnels.stringFunnel(Charsets.UTF_8),
    1_000_000,  // expected insertions
    0.01        // false positive rate
);

// Adicionar
filter.put("user@example.com");

// Verificar
filter.mightContain("user@example.com");  // true
filter.mightContain("unknown@test.com");  // false (provavelmente)

// Estimar FPR atual
filter.expectedFpp();  // 0.0099...
```

---

## Variações

### Counting Bloom Filter

```
Problema: Bloom Filter padrão NÃO suporta remoção
  (zerar um bit pode afetar outros elementos)

Solução: Usar CONTADORES em vez de bits

Standard:  [0, 1, 0, 1, 1, 0] — bits
Counting:  [0, 2, 0, 1, 3, 0] — counters

add("X"):    incrementa posições de X
remove("X"): decrementa posições de X

Trade-off: 3-4x mais memória (4 bits por counter vs 1 bit)
```

### Cuckoo Filter

```
Vantagens sobre Bloom Filter:
- Suporta DELETE nativo (sem counting)
- Melhor locality (cache-friendly)
- Melhor FPR para mesma memória quando > 3% FPR target

Desvantagens:
- Inserção pode falhar (tabela cheia)
- Mais complexo de implementar

Usado em: Redis CuckooFilter (CF.*)
```

### Quotient Filter

```
- Suporta merge de dois filters
- Suporta delete
- Locality melhor que Bloom
- Usado em: LSM-tree databases
```

### Scalable Bloom Filter

```
Problema: Bloom Filter tem capacidade fixa. Se ultrapassar n → FPR degrada.

Solução: Chain de Bloom Filters com FPR decrescente:

  BF1: capacity=1M,  FPR=0.5%
  BF2: capacity=2M,  FPR=0.25%  (criado quando BF1 enche)
  BF3: capacity=4M,  FPR=0.125% (criado quando BF2 enche)
  
  Check: verificar em TODOS os filters (OR)
  FPR total: ≤ Σ FPRi ≈ target FPR
```

### Comparação de Variações

| Variação | Delete | Memória | FPR | Merge | Complexidade |
|----------|--------|---------|-----|-------|--------------|
| **Bloom Filter** | Não | Muito baixa | Boa | Sim (OR de bit arrays) | Simples |
| **Counting BF** | Sim | 3-4x Bloom | Boa | Sim | Média |
| **Cuckoo Filter** | Sim | Melhor que Counting | Boa | Não | Média |
| **Quotient Filter** | Sim | Similar a Bloom | Boa | Sim | Alta |
| **Scalable BF** | Não | Dinâmica | Controlada | Não | Média |

---

## Casos de Uso em Big Techs

### Google Chrome — Safe Browsing

```
Problema: Verificar se URL é maliciosa entre bilhões de URLs.
  Fazer lookup no servidor para CADA URL = lento e privacidade.

Solução: Bloom Filter local no browser.

  User navega para "example.com/phishing"
  
  1. Chrome verifica Bloom Filter local (~25 MB)
     ├── "Definitivamente não maliciosa" → ACESSO DIRETO ✓
     └── "Possivelmente maliciosa" → passo 2
  
  2. Consulta servidor Google para confirmação
     ├── Falso positivo → acesso liberado
     └── Realmente maliciosa → AVISO ao usuário ⚠️
  
  Resultado: 99%+ das URLs resolvidas localmente (sem hit no servidor)
```

### Apache Cassandra — SSTable Lookups

```
Problema: Cassandra armazena dados em SSTables (arquivos imutáveis no disco).
  Uma leitura pode precisar checar múltiplos SSTables.

Solução: Bloom Filter por SSTable.

  Read("user:123"):
  
  SSTable 1: BF.check("user:123") → NO  → Skip ✓ (evitou disk I/O!)
  SSTable 2: BF.check("user:123") → NO  → Skip ✓
  SSTable 3: BF.check("user:123") → YES → Read from disk → Found!
  
  Sem Bloom Filter: 3 disk reads
  Com Bloom Filter: 1 disk read + 2 memory checks
  
  Config: bloom_filter_fp_chance = 0.01 (padrão)
  Menor FPR = mais memória, menos disk reads
```

### HBase — Block Index

```
HBase usa Bloom Filters no Block Index:

  RegionServer recebe Get("row:123")
  
  1. Verifica MemStore (memória) → miss
  2. Para cada HFile (SSTable):
     ├── Bloom Filter check: "row:123 neste HFile?"
     ├── NO → skip HFile ✓
     └── MAYBE → busca no block index → read block from disk
  
  bloom_filter_type: ROW (por row key) ou ROWCOL (por row+column)
```

### Medium — "Já leu este artigo?"

```
Problema: Não recomendar artigos que o usuário já leu.
  Armazenar set de artigos lidos por user = caro.

Solução: Bloom Filter por usuário.

  User visits article_id=12345
  → BF.add(user_bf, "12345")
  
  Recommendation engine:
  → candidate = "67890"
  → BF.check(user_bf, "67890") → NO → Recomendar ✓
  → candidate = "12345"
  → BF.check(user_bf, "12345") → YES → Skip (já leu)
  
  Falso positivo: ~1% dos artigos NÃO lidos são filtrados
  → Aceitável: user perde 1% de recomendações (não percebe)
```

### Akamai — Cache One-Hit-Wonders

```
Problema: ~75% dos objetos em CDN são acessados apenas 1 vez.
  Cacheá-los desperdiça espaço (evicts de objetos populares).

Solução: Bloom Filter como "admission policy".

  Request para "video.mp4":
  
  1. BF.check("video.mp4")
     ├── NO → primeira vez → NÃO cachear → BF.add("video.mp4")
     └── YES → já visto antes → CACHEAR ✓
  
  Resultado: apenas objetos acessados 2+ vezes entram no cache.
  Redução de ~50% no tamanho do cache sem perda de hit rate.
```

---

## Bloom Filter vs Alternativas

| Approach | Memória | FPR | Delete | Check Time |
|----------|---------|-----|--------|------------|
| **HashSet** | Alta (stores values) | 0% | Sim | O(1) |
| **Bloom Filter** | Muito baixa | >0% | Não | O(k) |
| **Cuckoo Filter** | Baixa | >0% | Sim | O(1) |
| **Sorted Set + Binary Search** | Alta | 0% | Sim | O(log n) |
| **Trie** | Alta | 0% | Sim | O(m) |

---

## Trade-offs

| Aspecto | Prós | Contras |
|---------|------|---------|
| **Memória** | Ordens de magnitude menos | Não armazena os dados reais |
| **Performance** | O(k) lookups (k << n) | k hash computations por check |
| **Certeza** | Zero falsos negativos | Falsos positivos possíveis |
| **Operações** | Add e check são O(k) | Sem delete (standard) |
| **Escalabilidade** | Memória fixa (independe de key size) | Capacidade fixa (degrada se ultrapassar n) |

---

## Perguntas Frequentes em Entrevistas

1. **"O que é um Bloom Filter?"**
   - Estrutura probabilística com bit array + k hash functions
   - "Definitivamente não" ou "provavelmente sim"
   - Space-efficient: milhões de items em poucos MB

2. **"Quando usar Bloom Filter?"**
   - Evitar disk/network lookups desnecessários
   - Cache admission (filtrar one-hit-wonders)
   - Deduplicação aproximada
   - Pre-filter antes de operações caras

3. **"Qual o trade-off?"**
   - Memória vs False Positive Rate
   - Mais memória → menor FPR
   - Fórmula: $m = -\frac{n \cdot \ln(FPR)}{(\ln 2)^2}$

4. **"Bloom Filter suporta delete?"**
   - Standard: NÃO (zerar bit pode afetar outros)
   - Counting Bloom Filter: SIM (usa contadores)
   - Cuckoo Filter: SIM (alternativa moderna)

5. **"Cite exemplos reais"**
   - Chrome Safe Browsing, Cassandra SSTables, HBase blocks
   - Akamai cache admission, Medium recommendations

---

## Referências

- Burton H. Bloom (1970) — *"Space/Time Trade-offs in Hash Coding with Allowable Errors"* — Paper original
- Fan et al. (2014) — *"Cuckoo Filter: Practically Better Than Bloom"*
- Almeida et al. (2007) — *"Scalable Bloom Filters"*
- Broder & Mitzenmacher (2004) — *"Network Applications of Bloom Filters: A Survey"*
- Cassandra Documentation — *"Bloom Filters"*
- Martin Kleppmann — *"Designing Data-Intensive Applications"*, referências a Bloom Filters
