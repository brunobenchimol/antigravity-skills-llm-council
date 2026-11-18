# LLM Council (Antigravity Skill)

Uma skill nativa para o **Google Antigravity** que transforma uma pergunta difícil, ideia ou decisão estratégica em **5 opiniões independentes de especialistas**, seguidas de **revisão por pares anônima** e um **veredito final sintetizado pelo Chairman**.

Adaptado para o Antigravity a partir da skill criada por [Ole Lehmann](https://x.com/itsolelehmann), baseada na metodologia [LLM Council](https://github.com/karpathy/llm-council) de **Andrej Karpathy**.

---

## O que ela faz

Quando você faz uma pergunta para uma única IA, recebe apenas uma linha de raciocínio — sem saber quais são os pontos cegos dela.

O **LLM Council** executa exatamente a metodologia do Karpathy usando recursos **100% nativos do Antigravity** (sem Python, scripts externos ou dependências adicionais):

1. **Enquadramento (Step 1):** O coordenador lê contexto relevante do seu workspace (se aplicável, como `GEMINI.md`, `AGENTS.md`, `.agents/rules/`, etc.) e formula uma pergunta neutra.
2. **5 Conselheiros em Paralelo (Step 2):** Cinco subagentes isolados analisam a decisão simultaneamente sob lentes opostas:
   - **The Contrarian** — procura falhas fatais, riscos e o que você está evitando enxergar.
   - **The First Principles Thinker** — desmonta premissas e questiona se você está fazendo a pergunta certa.
   - **The Expansionist** — busca o potencial máximo e oportunidades adjacentes subestimadas.
   - **The Outsider** — lê tudo sem contexto prévio, expondo jargões e a "maldição do conhecimento".
   - **The Executor** — foca apenas em viabilidade prática ("o que você faz na segunda-feira de manhã?").
3. **Peer Review Anônimo (Step 3):** As 5 respostas são embaralhadas e rotuladas apenas como `Response A` a `Response E`. Cinco novos subagentes revisores avaliam às cegas qual resposta é mais forte, qual tem o maior ponto cego e o que **todas** deixaram passar.
4. **Veredito do Chairman (Step 4):** Um subagente Chairman recebe tudo desanonimizado e entrega a síntese final no mesmo idioma da sua pergunta:
   - Onde o Conselho Concorda
   - Onde o Conselho Diverge
   - Pontos Cegos Detectados pelo Conselho
   - A Recomendação (direta, sem "depende")
   - A Primeira Coisa a Fazer
5. **Transcrição Opcional (Step 6):** Se você pedir para salvar, o arquivo `council-transcript-[timestamp].md` é gravado na raiz do workspace atual (ou no caminho que você informar) e o agente pergunta se o local está bom ou se prefere mover.

---

## Estrutura do Repositório

```text
antigravity-skills-llm-council/
├── README.md
└── skills/
    ├── llm-council/
    │   └── SKILL.md     # Skill principal (/llm-council + triggers PT/EN)
    └── council/
        └── SKILL.md     # Alias curto (/council)
```

---

## Como instalar no Antigravity

### Opção 1: Instalação Global (Recomendada — funciona em qualquer projeto)

Copie as duas pastas de `skills/` para o diretório global de skills do Antigravity (`~/.gemini/config/skills/`):

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.gemini\config\skills" | Out-Null
Copy-Item -Recurse -Force ".\skills\llm-council" "$env:USERPROFILE\.gemini\config\skills\"
Copy-Item -Recurse -Force ".\skills\council" "$env:USERPROFILE\.gemini\config\skills\"
```

**macOS / Linux:**
```bash
mkdir -p ~/.gemini/config/skills
cp -r skills/llm-council skills/council ~/.gemini/config/skills/
```

### Opção 2: Instalação por Workspace (Projeto específico)

Copie `skills/llm-council` e `skills/council` para `<seu-projeto>/.agents/skills/`.

---

## Como usar no Chat

Você pode acionar tanto por **slash command** quanto por **linguagem natural** (em Português ou Inglês — os conselheiros sempre respondem no mesmo idioma da sua pergunta):

### Via Slash Commands
* `/llm-council <sua pergunta ou decisão>`
* `/council <sua pergunta ou decisão>`

### Via Linguagem Natural (Triggers)
* **Em Português:** *"rodar o conselho sobre..."*, *"passar no conselho:"*, *"sala de guerra nisso:"*, *"estressar essa decisão:"*, *"testar essa ideia:"*, *"me ajude a decidir entre X e Y"*
* **In English:** *"council this:"*, *"run the council on..."*, *"war room this:"*, *"pressure-test this:"*, *"stress-test this:"*, *"debate this:"*

---

## Smoke Test (Perguntas para Testar o Motor)

Logo após instalar, copie e cole qualquer uma das perguntas abaixo em um chat novo para validar que o motor está rodando de ponta a ponta (elas são autossuficientes e têm dilemas reais que forçam o choque entre os 5 conselheiros):

### 1. Produto & Carreira (Português — `/council`)
```text
/council Devo pedir demissão agora para construir meu micro-SaaS B2B em tempo integral com 6 meses de reserva financeira, ou manter o emprego CLT e validar nos fins de semana por 1 ano mesmo avançando devagar?
```

### 2. Tecnologia & Arquitetura (Português — `/llm-council`)
```text
/llm-council Estamos começando um produto do zero com um time de 3 devs: devemos ir direto para microsserviços pensando em escala futura ou construir um monólito modular sabendo que pode dar trabalho refatorar depois?
```

### 3. Estratégia & Precificação (Inglês — `/council`)
```text
/council Should we offer a forever-free tier to drive viral adoption for our developer tool, or charge from day one ($19/month) and accept slower growth?
```

### 4. Trigger Natural sem barra (Português — Descoberta semântica)
```text
Rodar o conselho: Vale mais a pena investir R$ 50 mil para lançar um app nativo completo ou usar esse orçamento para validar demanda com uma landing page e concierge manual via WhatsApp?
```

### O que observar para confirmar que funcionou
1. Nenhum arquivo do seu workspace foi alterado (a menos que você peça para salvar a transcrição).
2. O agente disparou os **5 advisors em paralelo** e, na rodada seguinte, **5 peer reviews anônimos em paralelo**.
3. A resposta final trouxe exatamente os **5 blocos do veredito** no mesmo idioma da pergunta, com uma recomendação firme e **uma única ação inicial**.

---

## Créditos

* Metodologia original: [Andrej Karpathy — LLM Council](https://github.com/karpathy/llm-council)
* Design original da skill (Claude): [Ole Lehmann](https://x.com/itsolelehmann)
* Adaptação nativa para Google Antigravity: Bruno Benchimol
