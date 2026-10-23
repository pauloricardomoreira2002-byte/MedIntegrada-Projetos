---
name: orchestration-medical-docs
description: "Arquitetura de orquestração multi-agente para documentos médicos via Orca: livros persistidos em Livros/ na raiz (apenas livros), downloads de artigos/diretrizes em projeto/downloads/, verificação prévia de duplicatas, download completo antes da leitura, validação visual de SVGs"
metadata:
  node_type: memory
  type: project
  originSessionId: b905c881-9cc0-4efc-92a0-cfb9bf222cc8
  modified: 2026-10-22T23:39:12.848Z
---
## Visão Geral

Orquestração para gerar documentos médicos completos usando múltiplos agentes especializados coordenados **via skill `orchestration` (runtime Orca)**. O fluxo é: **Coordenador → Pesquisadores + Downloader → Escritor → Ilustrador → Verificador de SVGs → Integração Final**. Todos os agentes rodam dentro do Orca; não usar o Agent tool comum fora dele.

### Princípios Fundamentais

- **Hierarquia de Fontes**: Diretrizes clínicas &gt; Livros-texto renomados &gt; Artigos científicos recentes
- **Separação Estrita de Arquivos**:
  - `Livros/` na raiz (`G:\Meu Drive\MedIntegrada\Livros`): **APENAS livros-texto completos**. Nenhum artigo, diretriz ou arquivo parcial pode ser salvo aqui.
  - `projeto-{tema}/downloads/artigos/`: artigos científicos baixados por pesquisadores.
  - `projeto-{tema}/downloads/diretrizes/`: diretrizes clínicas baixadas por pesquisadores.
- **Verificação Prévia de Duplicatas**: Antes de baixar qualquer livro, o pesquisador **DEVE** listar o conteúdo de `G:\Meu Drive\MedIntegrada\Livros/` e verificar se já existe um arquivo correspondente (por título ou ISBN). Se existir, usa o local; se não, solicita download ao Downloader.
- **Download Completo Obrigatório**: Livros devem ser baixados inteiros via `mcp__libgen__download`. O pesquisador deve **aguardar o download terminar completamente** antes de tentar ler o arquivo. A ferramenta `mcp__libgen__read` sozinha **não persiste o arquivo**; use-a apenas para extrair capítulos de um livro já baixado localmente.
- **Fluxo Correto de Leitura**: Download completo → Verificar arquivo no disco → Ler páginas/capítulos pertinentes via `mcp__libgen__read` com caminho local OU ferramentas de leitura de PDF → Produzir pesquisa.md e fontes.md.
- **Checkpoints**: Salvar estado após cada fase para recuperação de falhas
- **Validação Visual Obrigatória**: Todo SVG deve ser convertido em PNG e revisado antes da integração final
- **Compilação Incremental**: Montar o documento final conforme os pesquisadores entregam suas partes

## Estrutura de Pastas

```
G:\Meu Drive\MedIntegrada\
├── Livros/                          # BIBLIOTECA COMPARTILHADA (raiz) — APENAS livros-texto completos
│   ├── harrison-complete.pdf
│   ├── goldman-cecil-26e.pdf
│   └── campbell-operative-orthopaedics-14e.pdf
│
└── projeto-{tema}/
    ├── escritor/                    # Planejamento, compilação final, pedidos de SVG
    │   ├── planejamento.md
    │   ├── secoes/
    │   │   ├── secao-1.md
    │   │   └── secao-2.md
    │   ├── pedidos-svg.md
    │   └── documento-final.md
    ├── pesquisadores/               # Um subdiretório por tópico/subtema
    │   ├── topico-1/
    │   │   ├── pesquisa.md
    │   │   └── fontes.md
    │   └── topico-2/
    ├── downloads/                   # Artigos e diretrizes (NUNCA livros)
    │   ├── artigos/                 # Artigos científicos baixados
    │   │   └── artigo-Z.pdf
    │   └── diretrizes/              # Diretrizes clínicas baixadas
    │       └── diretriz-SBD-2025.pdf
    ├── ilustracoes/                 # SVGs gerados e PNGs de validação
    │   ├── svg/
    │   │   └── ilustracao-1.svg
    │   └── validacao/
    │       └── ilustracao-1.png
    └── checkpoints/                 # Estado salvo após cada fase
        ├── fase-1-planejamento.json
        ├── fase-2-pesquisa.json
        └── fase-3-escrita.json
```

## Agentes e Modelos


| Agente                  | Modelo                                                            | Função                                                                                                                                                                        |
| ----------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Coordenador**         | modelo da sessão (atualmente claude claude-fable-5.1 max)         | Planeja estrutura, despacha pesquisadores via Orca, orquestra o DAG, gerencia checkpoints                                                                                     |
| **Escritor**            | claude claude-fable-5.1 max                                       | Compila .md final com linguagem acessível, lista pedidos detalhados de SVG                                                                                                    |
| **Verificador de SVGs** | claude claude-fable-5.1 high                                      | Converte SVGs em PNG, analisa visualmente legibilidade e sobreposições, aprova ou solicita correções                                                                          |
| **Pesquisadores**       | claude claude-fable-5.1 max                                       | Verifica Livros/ primeiro, baixa livros inteiros se necessário, lê arquivos baixados, pesquisa em medical-mcp seguindo hierarquia de fontes, salva resumos em .md por subtema |
| **Downloader**          | claude claude-opus-5-5 high                                       | Busca livros inteiros no LibGen (prioridade) ou Anna's Archive via MCP, **baixa o arquivo inteiro** via `download` para `Livros/` na raiz, notifica se falhar                 |
| **Ilustrador**          | agy gemini-3.8-flash high (fallback: claude claude-fable-5.1 max) | Gera SVGs a partir do arquivo de pedidos detalhados                                                                                                                           |


> ⚠️ **Nota sobre modelos**: Nesta instalação, o alias `sonnet` resolve para um ID inexistente (erro API 400). Usar `fable` para todos os workers. O coordenador usa o modelo ativo da sessão.

## Fluxo de Trabalho (DAG)

### Fase 1: Planejamento (Coordenador)

1. Recebe o tema do usuário
2. Divide em tópicos e subtemas coerentes
3. Cria pasta do projeto e subpastas (incluindo `downloads/artigos/` e `downloads/diretrizes/`)
4. Garante que `G:\Meu Drive\MedIntegrada\Livros/` exista (`mkdir -p`)
5. Salva planejamento em `escritor/planejamento.md`
6. **Checkpoint**: salva `checkpoints/fase-1-planejamento.json`
7. Despacha pesquisadores via skill `orchestration` (workers Orca) para cada subtema

### Fase 2: Pesquisa Paralela (Pesquisadores + Downloader)

Para cada subtema, o pesquisador segue este fluxo obrigatório:

#### 2.1 Passo Zero: Verificar Biblioteca Local (OBRIGATÓRIO ANTES DE QUALQUER DOWNLOAD)

Antes de iniciar a pesquisa ou solicitar qualquer download de livro:

1. Listar o conteúdo de `G:\Meu Drive\MedIntegrada\Livros/`
2. Identificar se os livros-texto necessários já existem lá (por título, autor ou ISBN)
3. Se existir: usar o arquivo local diretamente, pular o download
4. Se não existir: solicitar ao Downloader

> 🚫 **PROIBIDO**: Baixar um livro que já existe em `Livros/`. Sempre verificar primeiro para evitar duplicatas.

#### 2.2 Hierarquia de Fontes (ordem de prioridade)

1. **Diretrizes Clínicas** (via medical-mcp): SBD, AHA, ESC, NICE, etc. → Salvar em `downloads/diretrizes/`
2. **Livros-Texto Renomados**: Harrison, Goldman, Campbell, Guyton → Verificar em `Livros/` primeiro; se não existir, baixar inteiro para `Livros/`
3. **Artigos Científicos Recentes** (via medical-mcp): Google Scholar, PubMed → Salvar em `downloads/artigos/`

#### 2.3 Fluxo de Download de Livros (OBRIGATÓRIO)

Quando precisa de um livro que NÃO existe em `Livros/`:

1. Solicitar ao Downloader: "Preciso do livro [NOME]"
2. **Downloader** tenta nesta ordem:
   - **LibGen MCP** (prioridade): busca por ISBN ou título completo via `mcp__libgen__search`, depois **baixa o arquivo inteiro** via `mcp__libgen__download` salvando em `G:\Meu Drive\MedIntegrada\Livros/`
   - **Anna's Archive MCP** (fallback): busca alternativa e baixa
3. **Aguardar download completar**: O pesquisador deve esperar o Downloader confirmar sucesso antes de prosseguir
4. Resultado:
   - ✅ **Sucesso**: livro salvo inteiro em `Livros/` (raiz) e pesquisador notificado → prosseguir para leitura
   - ❌ **Falha em ambos**: **notifica o usuário** para download manual (bloqueante)

> 🚫 **PROIBIDO**: O pesquisador **não deve** usar `mcp__libgen__read` como substituto do download. `read` extrai texto sob demanda mas **não persiste o arquivo**. Se precisa de conteúdo de um livro, o fluxo correto é: Downloader baixa → aguardar conclusão → pesquisador lê o arquivo local (via `read` com caminho local ou ferramentas de leitura de PDF).

#### 2.4 Fluxo de Leitura Após Download

Após confirmar que o livro está disponível localmente (seja por verificação prévia ou download recém-concluído):

1. Usar `mcp__libgen__read` com o caminho local do arquivo em `Livros/` para extrair capítulos/páginas pertinentes
2. OU usar ferramentas de leitura de PDF para navegar o arquivo local
3. Extrair apenas o conteúdo relevante para o subtema atribuído
4. Cruzar com diretrizes e artigos baixados em `downloads/`

#### 2.5 Pesquisa com Medical-MCP

O pesquisador usa as ferramentas do medical-mcp seguindo a hierarquia:

- `search-clinical-guidelines`: diretrizes mais recentes → baixar para `downloads/diretrizes/`
- `search-medical-literature`: artigos revisados por pares → baixar para `downloads/artigos/`
- `search-google-scholar`: literatura complementar → baixar para `downloads/artigos/`
- `get-health-statistics`: dados epidemiológicos

> ⚠️ **IMPORTANTE**: Artigos e diretrizes baixados via medical-mcp vão para `projeto-{tema}/downloads/`, **NUNCA** para `Livros/`.

#### 2.6 Entrega do Pesquisador

Para cada subtema:

1. Cruza informações das três fontes hierárquicas
2. Resolve conflitos usando a hierarquia (diretriz &gt; livro &gt; artigo)
3. Salva resultado em `pesquisadores/{topico}/pesquisa.md`
4. Lista fontes usadas em `pesquisadores/{topico}/fontes.md`

#### 2.7 Checkpoint Incremental

Após cada pesquisador terminar:

- **Checkpoint**: atualiza `checkpoints/fase-2-pesquisa.json` com progresso

### Fase 3: Escrita e Compilação (Escritor)

1. Lê todos os .md dos pesquisadores conforme ficam prontos (compilação incremental)
2. Redige cada seção em arquivos separados (`escritor/secoes/secao-X.md`)
3. Identifica locais que precisam de ilustração
4. Insere marcadores de placeholder nos textos: `<!-- SVG: nome-da-ilustracao -->`
5. Documenta pedidos detalhados em `escritor/pedidos-svg.md` com:
   - Nome do SVG
   - Descrição anatômica/clínica precisa
   - Estilo e nível de detalhe desejado
   - Contexto no documento onde será inserido
   - **Especificações técnicas**: dimensões, cores, elementos obrigatórios
6. Compila tudo em `escritor/documento-final.md`
7. **Checkpoint**: salva `checkpoints/fase-3-escrita.json`

#### Diretrizes de Escrita Acessível

O documento final deve ser **muito completo tecnicamente, mas fácil de entender** para quem não é especialista:

- **Linguagem simples**: evitar jargões excessivos; quando necessários, explicar imediatamente
- **Glossário inline**: todo termo técnico que aparecer pela primeira vez deve ter uma explicação entre parênteses ou em nota de rodapé (ex: "osteófitos (pequenos crescimentos ósseos anormais nas articulações)")
- **Analogias e metáforas**: usar comparações do cotidiano para explicar conceitos complexos (ex: "a cartilagem funciona como um amortecedor natural")
- **Parágrafos curtos**: máximo de 4-5 frases por parágrafo para facilitar a leitura
- **Títulos descritivos**: cada seção deve ter um título claro que resuma o conteúdo
- **Resumos ao início**: começar cada capítulo com 2-3 frases resumindo os pontos principais
- **Listas e tabelas**: usar apenas quando realmente facilitarem a compreensão (ex: comparação de doses, critérios diagnósticos); não exagerar — texto corrido bem escrito é preferível na maioria dos casos
- **Tom didático**: escrever como se estivesse explicando para um estudante de medicina ou paciente interessado, não para especialistas

### Fase 4: Ilustração (Gemini)

1. Lê `escritor/pedidos-svg.md`
2. Gera cada SVG em `ilustracoes/svg/`
3. Notifica o Escritor quando terminar

### Fase 5: Validação Visual Obrigatória (Verificador de SVGs)

**Esta fase é crítica e não pode ser pulada.** O Verificador de SVGs (Fable 5.1 high) é um agente dedicado responsável exclusivamente por garantir a qualidade visual das ilustrações.

1. Para cada SVG gerado pelo Ilustrador:
   - Converte SVG para PNG usando ferramenta apropriada (ex: `rsvg-convert`, `inkscape`, ou biblioteca Python como `cairosvg`)
   - Salva PNG em `ilustracoes/validacao/`
   - **Analisa visualmente o PNG**:
     - Verifica se há elementos sobrepostos
     - Confere legibilidade de texto e rótulos
     - Valida proporções anatômicas
     - Checa se cores estão distintas
2. Resultado da validação:
   - ✅ **Aprovado**: marca como válido em `checkpoints/fase-4-validacao.json`
   - ❌ **Reprovado**: solicita regeneração ao Ilustrador com feedback específico do problema
3. Loop de validação até todos os SVGs estarem aprovados
4. Notifica o Coordenador quando todos os SVGs forem validados com sucesso

### Fase 6: Integração Final (Escritor)

1. Substitui placeholders pelos SVGs validados
2. Revisão final de formatação e coerência
3. Entrega o documento completo ao usuário
4. **Checkpoint final**: salva `checkpoints/fase-5-entrega.json`

## Sistema de Checkpoints

Cada checkpoint é um arquivo JSON com:

```json
{
  "fase": "nome-da-fase",
  "timestamp": "2026-10-23T01:00:00Z",
  "status": "completa|parcial|falhou",
  "progresso": {
    "itens_totais": 10,
    "itens_completos": 7,
    "itens_pendentes": ["item-8", "item-9", "item-10"]
  },
  "arquivos_gerados": ["caminho/para/arquivo1.md", "caminho/para/arquivo2.md"],
  "erros": []
}
```

### Recuperação de Falhas

Se a orquestração falhar em qualquer ponto:

1. Lê o último checkpoint válido
2. Retoma a partir da fase interrompida
3. Não reprocessa trabalho já concluído

## Validação Visual de SVGs

### Por que é Obrigatória?

SVGs gerados por IA frequentemente têm:

- Elementos sobrepostos que tornam o diagrama ilegível
- Texto truncado ou fora de posição
- Cores muito similares que se confundem
- Proporções anatômicas incorretas

### Processo de Validação

1. **Conversão**: SVG → PNG (preserva renderização exata)
2. **Análise Visual**: Verificador de SVGs lê o PNG usando ferramenta de leitura de imagens
3. **Checklist de Validação**:
   - [ ] Elementos não se sobrepõem
   
      \[ \] Texto é legível e completo
      \[ \] Cores são distintas e apropriadas
      \[ \] Proporções anatômicas estão corretas
      \[ \] Rótulos e setas apontam para estruturas corretas
4. **Feedback para Correção**: Se reprovado, envia descrição específica do problema ao Ilustrador

### Ferramentas de Conversão Sugeridas

- **Python**: `cairosvg` ou `svglib` + `reportlab`
- **CLI**: `rsvg-convert` (Linux/Mac), `inkscape --export-type=png`
- **Online**: APIs de conversão (última opção devido a latência)

## Hierarquia de Fontes

### Ordem de Prioridade (do mais ao menos confiável)

1. **Diretrizes Clínicas Oficiais**
   - Sociedades médicas: SBD, SBP, AHA, ESC, NICE
   - Atualizadas regularmente com base em evidências
   - Consenso de especialistas
2. **Livros-Texto Renomados**
   - Harrison's Principles of Internal Medicine
   - Goldman-Cecil Medicine
   - Campbell-Walsh Urology
   - Guyton and Hall Textbook of Medical Physiology
   - Conteúdo revisado por pares, estável
3. **Artigos Científicos Recentes**
   - Publicados em revistas de alto impacto
   - Revisão por pares rigorosa
   - Podem conter achados muito recentes ainda não incorporados em diretrizes

### Resolução de Conflitos

Quando fontes divergem:

1. **Diretriz vs Livro**: seguir a diretriz (mais recente)
2. **Diretriz vs Artigo**: seguir a diretriz, mas mencionar o artigo como perspectiva alternativa
3. **Livro vs Artigo**: seguir o livro se o artigo for muito recente (&lt;2 anos), senão seguir o artigo

## Como Invocar

Quando quiser usar esta orquestração, diga algo como:

> "Use a orquestração médica para criar um documento sobre [TEMA]"

O coordenador carregará este plano e executará o DAG descrito acima **via skill `orchestration`** (todos os workers dentro do Orca).

### Exemplos de Uso

- "Use a orquestração médica para criar um documento sobre osteoartrite"
- "Use a orquestração médica para criar um guia de tendinopatias"
- "Use a orquestração médica para criar material sobre artrite reumatoide"

## Referências

- [[orca-cli]] — para comandos de worktree e browser
- Skill `orchestration` — coordenação via Orca runtime (obrigatória para esta orquestração)
- MCP `medical-mcp` — para busca de artigos e diretrizes (salvar em `downloads/`)
- MCP `libgen` — para download de livros (salvar em `Livros/` na raiz; usar `download`, não apenas `read`)
- MCP `pdf-mcp`— para ler os livros, artigos e diretrizes
- Anna's Archive — fallback para downloads de livros