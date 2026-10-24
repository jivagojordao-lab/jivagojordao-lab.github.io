# Plano de Implementação — Bot de WhatsApp da Clínica

**Stack:** YCloud (Modo Coexistência) · Python (FastAPI + asyncio) · Gemini 3.5 Transcribe · Claude API · Google Calendar (Service Account) · VPS Ubuntu + Cloudflare Tunnel

**Status:** plano revisado a partir do rascunho original (5 etapas). Mantém a arquitetura proposta, mas corrige **bloqueadores** encontrados no código e nas premissas antes de qualquer paciente real falar com o robô.

---

## 1. Resumo executivo

A ideia central do rascunho está correta e é boa:

1. Paciente manda mensagem → o robô **espera** (5 min no expediente, 10 s fora) agrupando mensagens picadas.
2. Se a secretária responder pelo celular nesse intervalo, o evento *echo* da Coexistência **cancela o timer** e pausa a IA por 12 h para aquele paciente (custo zero de IA).
3. Se ninguém responder, o Claude assume, consulta/agenda no Google Calendar e responde numa única mensagem.
4. Áudios são transcritos pelo Gemini; PDFs/imagens e pedidos sensíveis vão direto para a recepção.

Porém, o código como está **não deve ir para produção**. Os problemas mais graves:

| # | Problema | Impacto |
|---|---|---|
| 1 | Alertas para `RECEPTION_ALERT_NUMBER` são texto livre enviado fora da janela de 24 h | A Meta **rejeita** a mensagem; a recepção nunca é avisada |
| 2 | Em "dor no peito"/"falta de ar" o paciente **não recebe resposta nenhuma** (o regex só pausa e alerta) | Risco clínico real |
| 3 | Webhook sem verificação de assinatura | Qualquer um que descubra a URL injeta mensagens falsas, agenda consultas e gasta créditos |
| 4 | Webhook faz trabalho lento (download + transcrição) antes de responder | Timeouts → YCloud reenvia → mensagens duplicadas |
| 5 | `history[-10:]` corta pares `tool_use`/`tool_result` | Erro 400 da API do Claude após algumas conversas |
| 6 | `claude-haiku-4-5` tem mínimo de **4096 tokens** para cache; o prompt tem ~1 000 | O "desconto de 90%" do Prompt Caching **nunca acontece** |
| 7 | Duas execuções do Claude podem rodar ao mesmo tempo para o mesmo paciente; secretária assumindo durante a execução não impede o envio | Respostas duplicadas/contraditórias |
| 8 | Gemini no plano **gratuito** pode usar o conteúdo para treinar modelos e ter revisão humana | Áudio de paciente é **dado sensível (LGPD)** |
| 9 | Agendamento não revalida conflito e ignora eventos de dia inteiro | Consultas duplicadas no mesmo horário |
| 10 | Estado (histórico, cestas) só em memória | Reinício do serviço apaga contexto das conversas |

O plano abaixo mantém as 5 etapas originais, acrescenta uma **Fase 0** (decisões) e uma **fase de piloto em modo sombra**, e detalha cada correção.

---

## 2. Decisões pendentes (precisam de resposta antes da Fase 3)

| Decisão | Opções | Recomendação |
|---|---|---|
| **Canal de alerta da recepção** | (a) Template "utility" aprovado na Meta, enviado pelo número da clínica; (b) Bot do Telegram; (c) E-mail | **(b) Telegram**: grátis, instantâneo, sem aprovação de template. (a) custa ~R$ 0,04–0,08 por alerta e exige aprovar o template |
| **Gemini pago ou gratuito** | Gratuito (dados podem ser usados pelo Google) vs. faturamento ativo | **Ativar faturamento** no projeto do Gemini. Custo baixo para o volume de uma clínica |
| **Modelo do Claude** | `claude-haiku-4-5` (rascunho) vs. `claude-haiku-5-5` | **`claude-haiku-5-5`**: 10× mais barato ($0,10/$0,50 por MTok vs $1/$5) e o cache funciona a partir de 512 tokens |
| **Pausa fora do expediente** | Pausar 12 h mesmo de madrugada vs. responder "a recepção retorna às 8h" | Fora do expediente, **não pausar silenciosamente**: enviar mensagem automática informando o horário de retorno |
| **Duração do modo sombra** | 0 / 3 / 7 dias | **3–5 dias úteis** |
| **Dados da clínica para o prompt** | Valores, convênios, endereço, preparo de exames, médicos, intervalo de almoço, feriados | Coletar antes da Fase 3 |

---

## 3. Arquitetura revisada

```
Paciente (WhatsApp)
      │
      ▼
YCloud ──webhook (assinado HMAC)──► Cloudflare Tunnel ──► FastAPI 127.0.0.1:8000
                                                             │
                     ┌───────────────────────────────────────┤ responde 200 em < 1 s
                     ▼                                       │ (processa em background)
             verificar assinatura → deduplicar (id da msg) → roteador de eventos
                     │
     ┌───────────────┼──────────────────────────┬──────────────────────────────┐
     ▼               ▼                          ▼                              ▼
 echo (secretária) inbound texto/áudio      inbound doc/imagem            emergência (regex)
 pausa 12 h,       transcrição Gemini →     pausa + alerta +              resposta FIXA imediata
 cancela timer,    cesta + timer (lock      confirmação ao paciente       (SAMU 192) + alerta
 grava no histórico por paciente)                                         + pausa
                     │
                     ▼ (timer expira e não está pausado)
              Claude (tools: horários livres, agendar, transferir)
                     │
                     ▼ checa pausa de novo → envia via YCloud
                                         
Estado persistente: SQLite (pausas, histórico só-texto, ids processados)
Alertas: Telegram (ou template Meta)
```

### Estrutura de arquivos

```
~/bot-clinica/
├── app/
│   ├── main.py            # FastAPI, rota do webhook, /health
│   ├── config.py          # leitura do .env e validação na inicialização
│   ├── security.py        # verificação da assinatura YCloud
│   ├── state.py           # SQLite: pausas, histórico, dedupe
│   ├── buffer.py          # cestas, timers, locks por paciente
│   ├── agent.py           # loop do Claude + execução das tools
│   ├── prompts.py         # SYSTEM_PROMPT e textos fixos (emergência, fora do horário)
│   ├── calendar_tools.py  # freebusy, cálculo de slots, agendamento com revalidação
│   ├── transcribe.py      # download da mídia (com X-API-Key) + Gemini
│   ├── whatsapp.py        # envio via YCloud
│   ├── notify.py          # alertas (Telegram ou template)
│   └── phones.py          # normalização E.164 e variante do 9º dígito
├── tests/                 # pytest com payloads reais capturados na Fase 1
├── data/bot.db            # SQLite (fora do git)
├── google_credentials.json  (chmod 600, fora do git)
├── .env                     (chmod 600, fora do git)
└── requirements.txt       # versões fixadas
```

---

## 4. Fases de implementação

### Fase 0 — Pré-requisitos (½ dia)

- [ ] Responder às decisões da seção 2.
- [ ] Coletar as informações da clínica para o prompt (seção 2, última linha).
- [ ] VPS Ubuntu 22.04/24.04 com **1 GB de RAM** (o rascunho fala em 30 MB; o realista é ~100–150 MB com FastAPI + SDKs do Google e da Anthropic).
- [ ] Domínio na Cloudflare (para o túnel).
- [ ] Conferir que o celular da clínica tem WhatsApp Business **2.24.17+** e que a conta do Facebook é admin do portfólio.

**Aceite:** todas as decisões registradas; dados da clínica em um documento.

### Fase 1 — YCloud em Coexistência + captura de payloads reais (1 dia)

Passos do rascunho (válidos): criar conta → WhatsApp → Numbers → Add Number → *Connect your existing WhatsApp Business App (Coexistence)* → login no Facebook → código no celular → sincronizar histórico → cartão no Billing Hub da Meta → copiar `YCLOUD_API_KEY`.

**Acréscimos:**

1. **Premissa de custo corrigida.** O modelo de "1 000 conversas grátis/mês" foi substituído pela cobrança por mensagem. Hoje, respostas em texto livre **dentro da janela de 24 h** aberta pelo paciente são gratuitas; o que custa é **template** (marketing, utility, autenticação). O robô só responde a quem escreveu, então o custo de Meta para ele é zero — mas alertas e futuros lembretes de consulta por template **são cobrados**. Conferir a tabela oficial da Meta para o Brasil antes de orçar.
2. **Capturar payloads reais antes de escrever lógica.** Subir um endpoint provisório que só grava o JSON bruto em arquivo e cadastrar os eventos `whatsapp.inbound_message.received` e `whatsapp.smb.message.echoes`. Gerar, a partir de um celular pessoal:
   - texto, áudio, imagem, PDF;
   - resposta da secretária pelo **app** e pelo **WhatsApp Web**;
   - reação 🤖 da secretária;
   - uma mensagem enviada **pela API** (para confirmar que ela **não** volta como echo — se voltar, o robô pausaria a si mesmo).
3. Com os payloads em mãos, confirmar: nome exato dos campos (`whatsappMessage.to`, formato da reação, `audio.link` vs `audio.id`, `mime_type`), formato dos números (com/sem `+`, com/sem 9º dígito) e o nome do header de assinatura.
4. Ao criar o endpoint de webhook na YCloud, **guardar o `secret`** retornado (usado na verificação de assinatura).

**Aceite:** pasta `tests/payloads/` com um JSON real de cada caso acima.

### Fase 2 — Credenciais (½ dia)

Passos do rascunho para Service Account do Google Calendar, chave do Gemini e chave da Anthropic continuam válidos. Acréscimos:

- [ ] **Gemini:** ativar faturamento no projeto (dados de pacientes não podem cair nos termos de "serviços não pagos"). Enviar a chave no header `x-goog-api-key`, não na URL (a URL aparece em logs).
- [ ] **Anthropic:** criar uma *workspace* só para o bot e definir **limite de gasto mensal** no console.
- [ ] **Alertas:** criar bot no Telegram (@BotFather), adicionar a secretária e anotar `TELEGRAM_BOT_TOKEN` e `TELEGRAM_CHAT_ID` — ou, se a decisão for template, submeter o template utility na Meta (aprovação pode levar horas).
- [ ] **Google Calendar:** além de compartilhar a agenda com a service account, anotar o **ID da agenda** e criar um evento de teste para validar acesso.

`.env` revisado:

```env
YCLOUD_API_KEY=...
YCLOUD_WEBHOOK_SECRET=...          # novo: secret do endpoint de webhook
CLINIC_WHATSAPP_NUMBER=+5534999999999
ANTHROPIC_API_KEY=sk-ant-...
CLAUDE_MODEL=claude-haiku-5-5      # alterado
GEMINI_API_KEY=...
GEMINI_TRANSCRIBE_MODEL=gemini-3.5-transcribe
GEMINI_FALLBACK_MODEL=...          # modelo Gemini geral para quando o transcribe vier vazio
GOOGLE_CALENDAR_ID=...
TELEGRAM_BOT_TOKEN=...             # ou RECEPTION_ALERT_TEMPLATE=...
TELEGRAM_CHAT_ID=...
BOT_MODE=shadow                    # shadow | live
```

**Aceite:** script `check_credentials.py` que testa cada chave (YCloud lista números, Calendar lê eventos, Gemini transcreve um áudio de teste, Claude responde "ok", Telegram envia mensagem).

### Fase 3 — Código (3–4 dias)

Implementar o fluxo do rascunho **com as correções abaixo**. Cada item vira um commit e, quando aplicável, um teste.

#### 3.1 Webhook: assinatura, resposta rápida e deduplicação (bloqueadores 3 e 4)

A YCloud assina com o header no formato `t=<timestamp>,s=<assinatura>`, onde a assinatura é HMAC-SHA256 de `"{t}." + corpo_bruto` com o secret do endpoint. Confirmar o nome exato do header nos payloads da Fase 1.

```python
def verify_signature(raw: bytes, header: str) -> bool:
    try:
        parts = dict(p.split("=", 1) for p in header.split(","))
        t, s = parts["t"], parts["s"]
    except (ValueError, KeyError):
        return False
    if abs(time.time() - int(t)) > 300:  # 5 min de tolerância contra replay
        return False
    expected = hmac.new(WEBHOOK_SECRET.encode(), f"{t}.".encode() + raw, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, s)


@app.post("/webhook/ycloud")
async def ycloud_webhook(request: Request):
    raw = await request.body()
    if not verify_signature(raw, request.headers.get("YCloud-Signature", "")):
        raise HTTPException(status_code=401)
    body = json.loads(raw)
    if state.already_processed(body.get("id")):
        return {"status": "duplicate"}
    spawn(handle_event(body))  # create_task guardando referência; responde já
    return {"status": "accepted"}
```

#### 3.2 Emergência e transferências sempre respondem ao paciente (bloqueador 2)

Separar o regex em dois:

- **`EMERGENCY_REGEX`** (dor no peito, falta de ar, desmaio, sangramento, convulsão…): enviar **imediatamente** texto fixo, sem passar pelo Claude:
  > "Pelo que você descreveu, procure agora o Pronto-Socorro mais próximo ou ligue para o SAMU (192). Já avisei nossa equipe."

  Depois: alerta urgente + pausa.
- **`HUMAN_REGEX`** (atendente, nota fiscal, reembolso, laudo, receita, atestado…): enviar confirmação curta ("Vou passar sua mensagem para a recepção") + alerta + pausa. Fora do expediente, informar o horário de retorno.
- Normalizar acentos antes do regex (`unicodedata`) para pegar "urgencia", "secretaria" etc.
- Documentos/imagens: mesma lógica — confirmar recebimento ao paciente, alertar e pausar.

#### 3.3 Alertas que realmente chegam (bloqueador 1)

`notify.py` com uma única função `alert(texto)`, implementada via Telegram (`sendMessage`) ou template Meta, conforme a decisão. Nunca texto livre do número da clínica para o número da secretária.

#### 3.4 Histórico só-texto e modelo atualizado (bloqueadores 5 e 6)

- Persistir no SQLite **apenas turnos de texto**: `user` (mensagens do paciente, já agrupadas) e `assistant` (resposta final enviada). Os blocos de ferramenta e de raciocínio vivem só dentro da execução atual. Assim o corte "últimos N turnos" nunca separa `tool_use` de `tool_result` e sempre começa por `user`.
- Isso também é necessário no `claude-haiku-5-5`: ele raciocina por padrão e recusa (400) blocos de raciocínio reenviados depois que o histórico anterior foi editado/cortado. Como o histórico salvo não contém esses blocos, o corte é seguro.
- Gravar também as mensagens da secretária (echo) como `"[Recepção]: ..."`, para que o Claude tenha contexto quando a IA for destravada.
- `claude-haiku-5-5`: **não** enviar `temperature`; definir `output_config={"effort": "low"}` (padrão é `medium`; para atendimento simples `low` costuma bastar — validar no modo sombra); `max_tokens` ≈ 4000 (o limite cobre raciocínio + resposta; 600 corta respostas); tratar `stop_reason == "refusal"` e `"max_tokens"` com mensagem padrão + transferência.
- O prefixo cacheado é `tools` + `system` (~1 000 tokens) e fica acima do mínimo de 512 tokens do Haiku 5.5. A data/hora vai na mensagem do usuário (depois do ponto de cache), como no rascunho — correto. Conferir `usage.cache_read_input_tokens > 0` a partir da 2ª chamada.

Esqueleto:

```python
async def run_claude_agent(phone: str, combined_text: str):
    async with locks[phone]:
        history = state.load_text_history(phone, max_turns=10)
        now_str = datetime.now(TZ).strftime("%d/%m/%Y %H:%M (%A)")
        messages = history + [{"role": "user", "content": f"[Data/Hora atual: {now_str}]\nPaciente: {combined_text}"}]

        for _ in range(5):  # limite de voltas de ferramenta
            resp = await anthropic_client.messages.create(
                model=CLAUDE_MODEL,
                max_tokens=4000,
                output_config={"effort": "low"},
                system=[{"type": "text", "text": SYSTEM_PROMPT, "cache_control": {"type": "ephemeral"}}],
                tools=CLAUDE_TOOLS,
                messages=messages,
            )
            if resp.stop_reason != "tool_use":
                break
            messages.append({"role": "assistant", "content": resp.content})
            results = [await run_tool(b, phone) for b in resp.content if b.type == "tool_use"]
            messages.append({"role": "user", "content": results})  # todos os resultados numa única mensagem

        final_text = "".join(b.text for b in resp.content if b.type == "text").strip()
        if resp.stop_reason != "end_turn" or not final_text:
            final_text = FALLBACK_TEXT  # "Vou passar para a recepção..."
            await transfer_to_reception(phone, f"stop_reason={resp.stop_reason}")
        elif state.is_paused(phone) and not state.paused_by_bot(phone):
            return  # a secretária assumiu durante a execução: não enviar
        state.append_turns(phone, combined_text, final_text)
        await send_whatsapp_message(phone, final_text)
```

Erros de ferramenta voltam como `tool_result` com `is_error: True`, nunca são descartados.

#### 3.5 Concorrência por paciente (bloqueador 7)

- Um `asyncio.Lock` por telefone envolve a execução do Claude.
- Mensagens que chegam **durante** a execução vão para uma nova cesta e iniciam novo timer, que só roda depois que o lock é liberado.
- Antes de enviar qualquer resposta, checar `is_paused(phone)` de novo. A pausa guarda a **origem** (`secretaria` ou `bot`): se foi o próprio Claude que chamou `transferir_para_secretaria` nesta execução, a mensagem de despedida ainda é enviada; se foi a secretária, não.
- Ao receber echo: cancelar timer, limpar cesta e marcar pausa (já previsto no rascunho).

#### 3.6 Google Calendar sem conflitos (bloqueador 9)

- Usar `freebusy.query` em vez de listar eventos; considerar eventos de dia inteiro como dia bloqueado.
- **Calcular os slots livres no código** (grade de 30 min, 08h–18h, sem intervalo de almoço, sem sábado/domingo/feriados, sem horários no passado) e devolver ao Claude uma lista pronta: `"Livres em 2026-11-03: 08:00, 08:30, 10:00..."`. O modelo não precisa fazer aritmética de agenda.
- `agendar_consulta`: revalidar o slot com `freebusy` imediatamente antes do `insert`; se ocupado, devolver erro com as alternativas.
- Gravar telefone em `extendedProperties.private` (facilita futuras tools de **cancelar/remarcar** e lembretes).
- Chamadas da biblioteca do Google são síncronas: executar com `await asyncio.to_thread(...)` para não travar o servidor; criar o `service` uma vez e reutilizar.
- Validar os argumentos das tools (formato de data, data futura, nome não vazio) antes de chamar a API.

#### 3.7 Transcrição de áudio robusta (bloqueador 8)

- `gemini-3.5-transcribe` existe, mas está em **preview** (anunciado em 26/08/2026), tem configuração própria de transcrição e há relatos de respostas vazias com HTTP 200 em alguns caminhos REST. Implementar:
  1. Download do áudio **com header `X-API-Key`** da YCloud (o link direto só funciona por poucos minutos; se `audio.link` vier vazio, baixar pelo `audio.id`).
  2. `mime_type` vindo do payload (não fixar `audio/ogg`).
  3. Chamada ao `gemini-3.5-transcribe`; se o texto vier vazio, tentar o `GEMINI_FALLBACK_MODEL`.
  4. Se ainda falhar: responder ao paciente "Não consegui ouvir seu áudio, pode escrever?" em vez de mandar `"[Áudio inaudível]"` para o Claude.
- Validar o formato exato da requisição contra a documentação atual do Gemini durante a implementação.

#### 3.8 Telefones (risco do 9º dígito)

No Brasil, o identificador do WhatsApp às vezes vem **sem** o 9º dígito do celular (`+55 34 9999-9999` vs `+55 34 99999-9999`). Se o `from` do inbound e o `to` do echo vierem em formatos diferentes, a pausa da secretária não encontra o paciente. `phones.normalize()` gera uma chave canônica; usar essa chave em pausas, cestas, timers e histórico. Confirmar com os payloads da Fase 1.

#### 3.9 Destravar a IA

- Manter a reação 🤖 como gatilho principal.
- **Atenção:** o texto `..` é entregue ao paciente (aparece na conversa dele). Se for mantido, documentar isso para a secretária; preferir só a reação.

#### 3.10 Modo sombra

Com `BOT_MODE=shadow`, todo o fluxo roda, mas a resposta do Claude vai **só para o canal de alerta** ("O robô responderia: ..."), nunca para o paciente. Mensagens de emergência continuam indo ao paciente em qualquer modo.

#### 3.11 Testes automatizados

`pytest` usando os payloads reais da Fase 1, com YCloud/Claude/Gemini/Calendar simulados:

- assinatura inválida → 401; mesmo `id` duas vezes → processado uma vez;
- echo durante a espera cancela o timer; echo durante a execução do Claude impede o envio;
- emergência responde em < 1 s com texto fixo;
- cálculo de slots (dia inteiro bloqueado, almoço, fim de semana, passado);
- agendamento em slot ocupado é recusado;
- histórico cortado sempre começa por `user` e não contém blocos de ferramenta.

**Aceite da Fase 3:** testes passando; `check_credentials.py` verde; prompt revisado com dados reais da clínica.

### Fase 4 — Deploy 24/7 (½ dia)

O serviço systemd e o Cloudflare Tunnel do rascunho estão corretos. Ajustes:

- `EnvironmentFile=/home/<user>/bot-clinica/.env` no unit; `chmod 600 .env google_credentials.json`.
- Rodar como usuário sem sudo.
- Instalar o `cloudflared` **como serviço** (o comando do painel já faz isso) para sobreviver a reboot.
- Rota `GET /health` (verifica SQLite e retorna 200) + monitor externo gratuito (ex.: UptimeRobot) avisando no Telegram se cair.
- Logs no `journald` **sem conteúdo de mensagens de pacientes** (só ids, tipos e status).
- Backup diário do `data/bot.db` (cron + cópia para outro local).
- `requirements.txt` com versões fixadas (`pip freeze`).

Webhook na YCloud: URL `https://bot.suaclinica.com.br/webhook/ycloud`, eventos `whatsapp.inbound_message.received` e `whatsapp.smb.message.echoes`.

**Aceite:** reiniciar a VPS e o bot volta sozinho; `/health` acessível pelo túnel; monitor ativo.

### Fase 5 — Testes ponta a ponta e piloto (1 semana)

Executar com um celular pessoal em `BOT_MODE=shadow`, depois `live`:

| # | Teste | Resultado esperado |
|---|---|---|
| T0 | Enviar mensagem pela API para o celular pessoal | **Não** gera echo; o robô não se pausa |
| T1 | "Oi, bom dia" no expediente; secretária responde antes de 5 min | Timer cancelado; Claude não é chamado; pausa registrada |
| T2 | Secretária reage 🤖 | Pausa removida |
| T3 | Áudio "quais horários vocês têm na próxima terça?" sem ninguém responder | Transcrição correta; slots reais da agenda; uma única mensagem |
| T4 | Pedir para agendar num horário livre | Evento criado com nome e telefone |
| T5 | Dois pacientes pedindo o mesmo horário quase juntos | Só um agendamento; o outro recebe alternativas |
| T6 | "Estou com dor no peito" | Texto fixo do SAMU em segundos + alerta + pausa |
| T7 | Enviar PDF | Confirmação ao paciente + alerta + pausa |
| T8 | 4 mensagens picadas em 30 s | Uma única resposta considerando todas |
| T9 | Secretária responde enquanto o Claude está processando | Resposta do robô não é enviada |
| T10 | Reiniciar o serviço no meio de uma conversa | Pausas e histórico preservados |
| T11 | Mensagem às 23h pedindo atendente | Aviso de horário de retorno + alerta |
| T12 | POST no webhook sem assinatura | 401 |

**Piloto em modo sombra (3–5 dias úteis):** a secretária atende normalmente e lê no Telegram o que o robô responderia. Ajustar prompt, regex e tempos de espera com base nisso. Critério para ir ao `live`: nenhuma resposta perigosa ou errada sobre agenda/valores nos últimos 2 dias do piloto.

**Treinamento da recepção (30 min):** como a pausa funciona, como destravar (🤖), onde chegam os alertas, o que fazer se o robô errar.

### Fase 6 — Operação contínua

- Revisão semanal no primeiro mês: conversas transferidas, erros, custo (`usage` do Claude registrado por chamada).
- Próximas funcionalidades (fora do escopo inicial): cancelar/remarcar consulta, lembrete de consulta na véspera (template utility, cobrado), confirmação de presença.

---

## 5. LGPD e segurança

- Mensagens e áudios de pacientes são **dados pessoais sensíveis (saúde)**. Necessário: aviso no primeiro contato ("Sou a assistente virtual da clínica; suas mensagens são processadas para agendamento — política de privacidade: <link>"), base legal registrada, prazo de retenção do histórico (ex.: apagar após 90 dias) e lista dos operadores (YCloud, Meta, Anthropic, Google).
- Gemini **somente com faturamento ativo**; Anthropic com limite de gasto e workspace dedicada.
- Segredos fora do git (`.gitignore` com `.env`, `google_credentials.json`, `data/`). **Não versionar o código do bot neste repositório público do GitHub Pages com segredos** — usar um repositório privado para o código.

---

## 6. Estimativa de custos (validar com as tabelas oficiais)

Hipótese: 40 conversas/dia que chegam ao Claude, ~4 chamadas por conversa.

| Item | Estimativa mensal |
|---|---|
| Claude `claude-haiku-5-5` (~$0,10/MTok entrada, ~$0,50/MTok saída, cache reduz entrada repetida) | ~US$ 2–5 |
| Gemini 3.5 Transcribe (pago, poucos minutos de áudio/dia) | baixo — conferir preço do preview |
| YCloud plano gratuito | US$ 0 (confirmar limites do plano) |
| Meta — respostas dentro da janela de 24 h | R$ 0 |
| Meta — templates (só se escolher alerta por template / lembretes) | ~R$ 0,04–0,08 por mensagem utility |
| VPS 1 GB | ~US$ 5 |
| Cloudflare Tunnel, Telegram, UptimeRobot | US$ 0 |

---

## 7. Cronograma resumido

| Fase | Duração | Depende de |
|---|---|---|
| 0 — Decisões e dados | ½ dia | — |
| 1 — YCloud + captura de payloads | 1 dia | 0 |
| 2 — Credenciais | ½ dia | 0 (paralelo à 1) |
| 3 — Código + testes | 3–4 dias | 1, 2 |
| 4 — Deploy | ½ dia | 3 |
| 5 — Testes E2E + modo sombra | ~1 semana | 4 |
| **Total até o live** | **~2 semanas** | |

---

## 8. Pontos a confirmar durante a implementação

- Nome exato do header de assinatura da YCloud e estrutura do payload de echo/reação (Fase 1).
- Se mensagens enviadas pela API geram `whatsapp.smb.message.echoes` (teste T0).
- Formato atual da requisição do `gemini-3.5-transcribe` e se o problema de transcrição vazia persiste.
- Limites do plano gratuito da YCloud e tabela de preços da Meta para o Brasil.
- Parâmetro `output_config` disponível na versão instalada do SDK `anthropic` (usar versão recente).

## Fontes consultadas

- [Gemini 3.5 Transcribe — documentação do modelo](https://ai.google.dev/gemini-api/docs/models/gemini-3.5-transcribe)
- [Relato de transcrição vazia no gemini-3.5-transcribe (fórum Google AI)](https://discuss.ai.google.dev/t/gemini-3-5-transcribe-returns-empty-transcription-http-200-zero-output-tokens-on-all-documented-rest-paths/179937)
- [Termos adicionais da Gemini API (serviços não pagos)](https://ai.google.dev/terms)
- [YCloud — Webhook Integration Guide](https://docs.ycloud.com/reference/webhook-integration-guide)
- [YCloud — Webhook Events & Payloads](https://docs.ycloud.com/reference/webhook-events-payloads)
- [YCloud — Inbound Message Webhook Examples](https://docs.ycloud.com/reference/whatsapp-inbound-message-webhook-examples)
- [YCloud — Business App Sent Message Sync Webhook Examples](https://docs.ycloud.com/reference/whatsapp-business-app-sent-message-sync-webhook-examples)
- [YCloud Go SDK v1.17.0 (renomeação para `whatsapp.smb.message.echoes`)](https://github.com/YCloud-Developers/ycloud-sdk-go/releases/tag/v1.17.0)
- [360dialog — Coexistence Webhooks](https://docs.360dialog.com/partner/onboarding/whatsapp-coexistence/coexistence-webhooks)
- [Preços da WhatsApp API no Brasil (estimativas de terceiros)](https://chatmaxima.com/whatsapp-api-pricing/brazil/)
