# Agente Biográfico

> Aplicação web que conduz uma pessoa por perguntas sobre sua vida e gera um livro autobiográfico em DOCX ao final.

**URL:** https://keok-netzsch.github.io/agente-biografico/  
**Repositório:** https://github.com/keok-netzsch/agente-biografico  
**Autor:** Kelvin Okuda — Netzsch  
**Status:** Em desenvolvimento ativo (Mai/2025)

---

## Contexto

Projeto desenvolvido para uma apresentação interna de 2 semanas na Netzsch, com dois objetivos:

1. **Reflexão sobre legado e memória** — apresentar a brevidade da vida e o esquecimento geracional como motivação para preservar histórias
2. **Demonstração prática de IA** — mostrar ao time o que é possível construir com LLMs, promovendo adoção interna de IA

**Semana 1:** Reflexão + demo ao vivo do agente + explicação da arquitetura técnica  
**Semana 2:** Entrega dos DOCXs individuais gerados + dados agregados/anônimos + aprendizados técnicos

---

## O que o app faz

```
Usuário seleciona idioma
       ↓
Insere nome (salvo para o livro)
       ↓
Responde perguntas de 8 capítulos
       ↓  ← IA gera pergunta de aprofundamento após cada resposta
Pode adicionar fotos por capítulo (sépia + mosaico automático)
       ↓
Tela de revisão: escolhe/gera capa do livro
       ↓  ← IA gera imagem via Pollinations.ai ou usuário faz upload
Clica "Gerar meu livro"
       ↓  ← IA transforma respostas em narrativa literária por capítulo
Baixa o DOCX formatado
```

---

## Estrutura de capítulos

| # | Capítulo | Foco |
|---|----------|------|
| 1 | Quem sou eu | Nome, origem, cidade natal |
| 2 | Infância e família | Casa, pessoas próximas, memórias |
| 3 | Escola e juventude | Aprendizados, professores, amigos |
| 4 | Trabalho e carreira | Primeiro emprego, conquistas |
| 5 | Amor e relacionamentos | Família, parceiro, amizades |
| 6 | Momentos que me formaram | Viradas, perdas, surpresas |
| 7 | O que aprendi com a vida | Crenças, lições |
| 8 | Mensagem para as próximas gerações | Legado, como quer ser lembrado |

Cada capítulo tem 2 perguntas base. Após cada resposta, a IA gera 1 pergunta de aprofundamento personalizada — totalizando até 48 interações por sessão.

---

## Arquitetura

### Princípios de design

- **Zero backend** — tudo roda no browser. Hospedagem estática no GitHub Pages.
- **Zero dependências de runtime** — o DOCX é gerado em JS puro (sem bibliotecas CDN)
- **Portabilidade** — arquivo único `index.html` (~75KB). Pode ser aberto localmente sem servidor.
- **Multi-idioma** — PT, EN, DE, ES, FR. Idioma selecionado no início; toda a UI e prompts de IA se adaptam.

### Diagrama de fluxo

```
┌─────────────────────────────────────────────────────────┐
│                     index.html                          │
│                                                         │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  Estado  │    │   Screens    │    │  AI Providers │  │
│  │ (JS obj) │◄──►│  (DOM puro) │    │               │  │
│  │          │    │              │    │  Groq API     │  │
│  │ lang     │    │ 0. Lang sel  │    │  (Llama 3.3)  │  │
│  │ provider │    │ 1. Setup     │    │               │  │
│  │ apiKey   │    │ 2. Questions │───►│  Gemini API   │  │
│  │ answers[]│    │ 3. Review    │    │  (Flash 2.0)  │  │
│  │ photos{} │    │ 4. Final     │    │               │  │
│  │ coverB64 │    └──────────────┘    │  Claude API   │  │
│  └──────────┘                        │  (Sonnet 4)   │  │
│       │                              └───────────────┘  │
│       │         ┌────────────────┐                      │
│       └────────►│  Output layer  │                      │
│                 │                │                      │
│                 │ Canvas API     │  ← sepia + mosaico   │
│                 │ Pollinations   │  ← capa IA           │
│                 │ DOCX builder   │  ← ZIP puro JS       │
│                 │ localStorage   │  ← auto-backup       │
│                 └────────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

### Providers de IA suportados

| Provider | Modelo | Custo | Chave hardcodada |
|----------|--------|-------|-----------------|
| Groq | llama-3.3-70b-versatile | Gratuito (14.4k req/dia) | Sim |
| Gemini | gemini-2.0-flash | Gratuito (1.5k req/dia) | Sim |
| Claude | claude-sonnet-4-20250514 | Pago (~$0.10/sessão) | Não (usuário insere) |

O usuário seleciona o provider na tela de setup. Groq é o padrão.

### Chamadas à API por sessão

```
Por pergunta respondida:
  → 1x getFollowupQuestion()     (max_tokens: 600)

Na geração final:
  → 8x generateNarrative()       (max_tokens: 1000, uma por capítulo)
  → 1x fetchCoverImage()         (Pollinations.ai — sem tokens)

Total típico: ~24 chamadas de API por sessão completa
Custo estimado (Claude Sonnet): < $0.10 por sessão
```

### Geração do DOCX

O DOCX é um arquivo ZIP contendo XML. A geração é feita **inteiramente em JS nativo**, sem bibliotecas:

```
generateDocx()
  ├── buildZip()                  ← monta o ZIP manualmente
  │     ├── deflateRaw()          ← compressão via CompressionStream API
  │     └── crc32()               ← checksum manual
  ├── makeParaXml()               ← gera XML de parágrafos Word
  ├── makeImageXml()              ← embute imagens via r:embed
  └── arquivos no ZIP:
        ├── [Content_Types].xml
        ├── _rels/.rels
        ├── word/document.xml     ← corpo do livro
        ├── word/styles.xml       ← tipografia (Georgia, headings)
        ├── word/footer1.xml      ← nome + número de página
        ├── word/_rels/document.xml.rels
        └── word/media/imageN.jpeg  ← fotos embutidas em base64
```

**Formato:** A4 (11906 × 16838 EMU), margens 2.5cm, fonte Georgia 12pt, parágrafo com recuo de primeira linha.

### Processamento de imagens

```
fileToSepiaBase64(file, maxPx)
  ├── FileReader → Image
  ├── Canvas: redimensiona para maxPx (padrão 1200px)
  ├── getImageData → aplica matriz sépia pixel a pixel
  └── toDataURL('image/jpeg', 0.88)

composeMosaic(b64list, targetW)
  ├── 1 foto  → retorna diretamente
  ├── 2 fotos → grid 2×1
  ├── 3-4     → grid 2×2
  └── 5+      → grid 3×N

fetchCoverImage(authorName, chap1Summary)
  └── GET https://image.pollinations.ai/prompt/{prompt}?width=800&height=1100
```

### Persistência de dados

```
localStorage['bio_backup'] = {
  lang, authorName, answers[], savedAt
}
```

Salvo automaticamente após cada resposta. Banner de restauração aparece na tela inicial se houver backup.  
**Nota:** `photos{}` e `coverImageB64` **não** são persistidos no localStorage (dados binários grandes). Apenas as respostas de texto são backup-eadas.

---

## Idiomas suportados

Toda a UI, perguntas e prompts de IA estão traduzidos para:

- 🇧🇷 Português (pt)
- 🇺🇸 English (en)
- 🇩🇪 Deutsch (de)
- 🇪🇸 Español (es)
- 🇫🇷 Français (fr)

Para adicionar um idioma: incluir uma entrada no objeto `LANGS` em `index.html` com as chaves `flag`, `label`, `ui{}` e `chapters[]`.

---

## Segurança e limitações conhecidas

| Item | Status | Observação |
|------|--------|------------|
| Chaves de API no HTML público | ⚠️ Risco baixo | Groq e Gemini têm rate limit diário. Para produção, usar repositório privado ou proxy. |
| Sem autenticação | ✅ Intencional | App público, sem dados sensíveis no servidor (zero backend). |
| Sem armazenamento de respostas | ✅ Privacidade | Tudo fica no browser do usuário. Nada sai para servidor além das chamadas de API. |
| CompressionStream | ✅ Compatível | Chrome 80+, Edge 80+, Safari 16.4+, Firefox 113+. |
| Tamanho do DOCX | ⚠️ Depende das fotos | Com muitas fotos em alta resolução, pode chegar a 10–20MB. |

---

## Como atualizar o app

1. Edite `index.html` localmente
2. Acesse https://github.com/keok-netzsch/agente-biografico
3. Clique em `index.html` → ✏️ (editar) → cole o novo conteúdo → **Commit changes**
4. Aguarde ~1 minuto para o GitHub Pages republicar

Ou via Git:
```bash
git add index.html
git commit -m "descrição da mudança"
git push origin main
```

---

## Prompt de retomada do desenvolvimento

> Cole este bloco no início de uma nova conversa com Claude para retomar o desenvolvimento exatamente onde parou.

```
Contexto do projeto — Agente Biográfico

Estou desenvolvendo um projeto pessoal/profissional chamado "Agente Biográfico" — 
um app web que conduz uma pessoa por perguntas sobre sua vida e gera um livro 
autobiográfico em DOCX ao final.

Repositório: https://github.com/keok-netzsch/agente-biografico
URL pública: https://keok-netzsch.github.io/agente-biografico/

Decisões já tomadas e implementadas:
- Arquivo único index.html (~75KB), hospedado no GitHub Pages (estático, zero backend)
- Multi-idioma: PT, EN, DE, ES, FR (selecionado na tela inicial)
- Output: DOCX gerado em JS puro (sem CDN), usando CompressionStream + XML Word manual
- Providers de IA: Groq (padrão, gratuito), Gemini (gratuito), Claude (pago)
  - Chaves de Groq e Gemini hardcodadas no HTML
  - Claude requer input do usuário
- Design: tipografia Lora + Source Sans 3, paleta âmbar/creme, caloroso/familiar
- 8 capítulos biográficos, 2 perguntas cada
- Após cada resposta: IA gera pergunta de aprofundamento personalizada
- Upload de fotos por capítulo: sépia via Canvas API + mosaico automático
- Capa do livro: gerada por IA via Pollinations.ai, ou upload próprio, ou sem capa
- Auto-backup das respostas em localStorage + botão de exportar JSON
- Botão ← Voltar para reeditar pergunta anterior
- Botão 💾 Salvar respostas visível em todas as telas de pergunta

Estrutura de capítulos:
1. Quem sou eu — origem, cidade natal, ano de nascimento
2. Infância e família
3. Escola e juventude
4. Trabalho e carreira
5. Amor e relacionamentos
6. Momentos que me formaram
7. O que aprendi com a vida
8. Mensagem para as próximas gerações

Contexto maior:
Apresentação interna de 2 semanas na Netzsch (empresa alemã de equipamentos industriais).
Semana 1: reflexão sobre brevidade da vida + esquecimento geracional + demo + arquitetura.
Semana 2: entrega dos DOCXs individuais + dados agregados/anônimos + aprendizados.
Objetivo secundário: promover adoção de IA no time.

Próximo passo: [DESCREVA AQUI O QUE QUER FAZER A SEGUIR]
```

---

## Histórico de decisões técnicas

| Decisão | Alternativa descartada | Motivo |
|---------|----------------------|--------|
| Zero backend | Node.js/Python API | GitHub Pages é gratuito e suficiente; sem manutenção |
| DOCX puro JS | docx.js via CDN | CDN bloqueado no painel lateral Claude; independência de rede |
| Groq como padrão | Claude API | Gratuito para testes; qualidade suficiente para o caso de uso |
| Pollinations.ai para capa | Gemini Imagen, DALL-E | Zero configuração, sem API key, gratuito |
| localStorage para backup | IndexedDB | Simplicidade; dados são apenas texto |
| Arquivo único | Multi-arquivo + build | GitHub Pages simples, zero toolchain |
