# Projeto Forzy — Sprint 1
## Documento de Decisões de Padronização e Justificativas Técnicas

**Versão:** 1.0.0  
**Data:** 2025-01  
**Responsável:** Equipe Forzy — Challenge FIAP  
**Domínio:** Manutenção Preditiva de Motores Elétricos Industriais

---

## 1. Contexto e Motivação

O Projeto Forzy implementa um Digital Twin para motores elétricos industriais. O módulo de PLN precisa processar textos heterogêneos oriundos de quatro fontes distintas:

| Fonte | Características | Desafios |
|-------|----------------|----------|
| **Placa do equipamento** (nameplate) | Misto PT/EN, abreviações densas, unidades coladas ao valor (`440V`, `1800rpm`) | Alta densidade de abreviações; sem espaços entre valor e unidade |
| **Manual técnico** | EN predominante, terminologia normalizada por normas IEC/NEMA | Termos compostos (`insulation resistance`, `locked rotor current`) |
| **Ficha de cadastro** (ERP/CMMS) | PT com campos semi-estruturados, preenchimento manual inconsistente | Erros ortográficos, truncamentos, campos mesclados |
| **Log de operação** | PT coloquial, siglas de processo, timestamps embutidos | Linguagem informal, TAG de equipamento misturado ao texto |

---

## 2. Decisões de Padronização Terminológica

### 2.1 Estrutura do Glossário

**Decisão:** Adotar estrutura hierárquica com 5 categorias semânticas.

**Justificativa:** A categorização permite filtros semânticos na consulta e facilita o treinamento de modelos de NER (Sprint 2). As categorias refletem os domínios de conhecimento do técnico de manutenção:

1. `parametro_eletrico` — grandezas mensuráveis: tensão, corrente, potência, frequência
2. `componente_mecanico` — partes físicas: rolamento, eixo, acoplamento, ventoinha
3. `componente_eletrico` — partes elétricas: estator, rotor, enrolamento, capacitor
4. `tipo_falha` — modos de falha: curto-circuito, desgaste, desalinhamento, sobretensão
5. `parametro_construtivo` — características de projeto: classe de isolamento, IP, polos, IE

### 2.2 Campos Obrigatórios do Cadastro do Ativo

**Decisão:** Definir 9 campos obrigatórios com tipo e formato estrito.

| Campo | Tipo | Formato | Exemplo |
|-------|------|---------|---------|
| `tag` | string | `{PLANTA}-{AREA}-{TIPO}{SEQ}` | `USI01-COMP-MT001` |
| `descricao_curta` | string | Max 80 chars, maiúsculas | `MOTOR WEG W22 15KW 4P 380V` |
| `descricao_longa` | string | Texto livre, PT, max 500 chars | Texto detalhado da especificação |
| `fabricante` | enum | Lista controlada | `WEG`, `SIEMENS`, `ABB`, `GE`, `BALDOR` |
| `modelo` | string | Alfanumérico sem espaço | `W22-15KW-4P` |
| `localizacao` | string | `{PLANTA}.{AREA}.{POSICAO}` | `USI01.COMP.L3-P2` |
| `especificacao_tecnica` | dict | JSON de pares chave-valor | `{"potencia_kw": 15, "tensao_v": 380}` |
| `fonte` | enum | `placa`, `manual`, `cadastro_erp`, `log_operacao` |
| `idioma` | enum | `pt`, `en`, `misto` |

**Justificativa:** O campo `tag` segue o padrão ISA-5.1 adaptado. A separação entre `descricao_curta` e `descricao_longa` permite uso em interfaces com diferentes espaços disponíveis. O campo `especificacao_tecnica` como dict facilita queries estruturadas.

### 2.3 Fabricantes — Vocabulário Controlado

**Decisão:** Normalizar todas as variações de nome de fabricante para forma canônica.

| Variações aceitas no input | Forma canônica |
|---------------------------|----------------|
| `WEG`, `Weg`, `weg`, `WEG S.A.` | `WEG` |
| `Siemens`, `SIEMENS`, `siemens AG` | `SIEMENS` |
| `ABB`, `Abb`, `A.B.B.` | `ABB` |
| `GE`, `General Electric`, `GE Motors` | `GE` |
| `Baldor`, `BALDOR`, `Baldor Electric` | `BALDOR` |
| `Voith`, `VOITH` | `VOITH` |
| `Nidec`, `NIDEC`, `US Motors` | `NIDEC` |

### 2.4 TAGs de Equipamento

**Decisão:** Formato `{PLANTA}-{AREA}-{TIPO}{SEQ:03d}` com tipos padronizados.

| Código | Tipo de Equipamento |
|--------|-------------------|
| `MT` | Motor elétrico (Motor Trifásico) |
| `MB` | Motor de bomba |
| `MC` | Motor de compressor |
| `MV` | Motor de ventilador |
| `GE` | Gerador elétrico |

---

## 3. Decisões do Pipeline de Pré-processamento

### 3.1 Ordem das Etapas

**Decisão:** Pipeline sequencial de 6 etapas com rastreabilidade por etapa.

```
ENTRADA (texto bruto)
  ↓
[E1] Detecção de idioma (heurística + langdetect)
  ↓
[E2] Expansão de abreviações industriais (regex com contexto)
  ↓
[E3] Normalização unicode e lowercase
  ↓
[E4] Separação de valores numéricos e unidades coladas (440V → 440 V)
  ↓
[E5] Remoção de stopwords (lista técnica customizada)
  ↓
[E6] Lematização por idioma (spaCy pt_core_news_sm / en_core_web_sm)
  ↓
SAÍDA (lista de lemas normalizados)
```

**Justificativa da ordem:** A expansão de abreviações (E2) precede a normalização (E3) porque o mapeamento de regex usa padrões case-sensitive para maior precisão (`HP` ≠ `hp`). A separação de valores (E4) precede a remoção de stopwords (E5) para não perder unidades de medida que são tokens de domínio.

### 3.2 Abreviações: Regex vs. Lookup Simples

**Decisão:** Usar regex com word boundaries (`\b`) ao invés de substituição simples.

**Justificativa:** Substituição simples em `rot. 1800rpm` poderia corromper substrings. O pattern `\brot\.` garante correspondência apenas quando `rot.` é um token isolado.

### 3.3 Lematização vs. Stemming

**Decisão:** Lematização (spaCy) como método primário; stemming (NLTK RSLP/Porter) como referência comparativa.

**Justificativa:** 
- **Stemming** (RSLP para PT, Porter para EN) é mais rápido mas produz radicais que não são palavras válidas (`rolamento → rolam`), dificultando a interpretabilidade do glossário.
- **Lematização** preserva formas reconhecíveis (`rolamentos → rolamento`), essencial para matching com o glossário técnico.
- Termos de domínio listados em `DOMINIO_TOKENS` são passados sem lematização para preservar forma canônica do glossário.

**Avaliação quantitativa:** Ver Bloco 4 do notebook — comparação TTR e exemplos lado a lado.

### 3.4 Stopwords Técnicas

**Decisão:** Usar lista customizada ao invés de stopwords genéricas do spaCy/NLTK.

**Justificativa:** Stopwords padrão removem palavras como `não` (negação importante em laudos: "não conformidade") e `fase` (termo técnico: "desequilíbrio de fase"). A lista customizada remove apenas artigos, preposições e conjunções que não carregam significado no domínio técnico.

### 3.5 Tratamento de Textos Multilíngues

**Decisão:** Classificar cada documento como `pt`, `en` ou `misto`; para `misto`, aplicar pipeline PT com fallback EN.

**Justificativa:** Placas de equipamentos frequentemente mesclam idiomas (`Motor WEG 50HP tens. 440V rot. 1800rpm`). A detecção heurística por proporção de tokens reconhecidos em cada idioma é mais robusta que modelos de detecção de idioma para textos tão curtos.

---

## 4. Estrutura de Diretórios e Naming Convention

```
forzy_sprint1/
├── notebooks/
│   └── sprint1_pln_corpus.ipynb          # Notebook principal (este arquivo)
├── data/
│   ├── raw/
│   │   └── forzy_corpus_raw.csv          # Dataset sintético original
│   └── processed/
│       ├── forzy_corpus_estruturado.json # Corpus processado completo
│       └── forzy_corpus_stats.json       # Estatísticas do corpus
├── docs/
│   ├── forzy_glossario_tecnico.json      # Glossário PT/EN
│   └── decisoes_padronizacao.md          # Este documento
└── outputs/
    ├── forzy_analise_corpus.png          # Visualizações
    └── forzy_comparacao_lemma_stem.png   # Comparação lematização vs stemming
```

**Convenção de nomenclatura:**
- Prefixo `forzy_` em todos os artefatos do projeto
- Snake_case para nomes de arquivo
- Sem acentos nos nomes de arquivo
- Sufixo de versão `_v{N}` quando houver iterações (`forzy_glossario_tecnico_v2.json`)

---

## 5. Decisões de Qualidade e Validação

### 5.1 Critérios de Aceitação do Corpus

Um documento é aceito no corpus limpo se:
- [ ] Campo `tag` válido (regex: `^[A-Z]{2,6}\d{2}-[A-Z]{2,6}-[A-Z]{2}\d{3}$`)
- [ ] Campo `descricao` com comprimento entre 10 e 500 caracteres
- [ ] Campo `idioma` classificado (não `desconhecido`)
- [ ] Pelo menos 3 lemas após processamento

### 5.2 Métricas de Qualidade do Pipeline

| Métrica | Fórmula | Meta Sprint 1 |
|---------|---------|---------------|
| Cobertura do glossário | termos_glossario_no_corpus / total_termos_glossario | ≥ 60% |
| Taxa de expansão de abreviações | abreviacoes_expandidas / abreviacoes_encontradas | ≥ 90% |
| Type-Token Ratio (TTR) | vocabulario_unico / total_lemas | 0.3 – 0.7 |
| Compressão do pipeline | lemas / tokens_brutos | ≤ 0.8 |

---

## 6. Referências e Normas

- **IEC 60034** — Máquinas Elétricas Girantes (terminologia internacional)
- **NEMA MG 1** — Motors and Generators (padrão norte-americano)
- **NBR 5383** — Máquinas de corrente alternada — terminologia PT
- **ISA-5.1** — Instrumentation Symbols and Identification (padrão de TAGs)
- **ISO 10816** — Avaliação de vibração mecânica em máquinas
