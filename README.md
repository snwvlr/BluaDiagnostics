# BluaDiagnostics - Sprint 3

Projeto da disciplina de Prompt Engineering e IA.
A proposta vem do desafio da **Care Plus / Blua**: tornar o app, que hoje é
basicamente reativo (agenda, autoriza, consulta), em uma plataforma de
**cuidado remoto proativo**, com check-up digital conversacional e suporte
à prescrição automatizada.

Esta sprint é a parte de **exploração, arquitetura e prova de conceito**.
Ainda não temos RAG indexado nem multi-agente, esses pontos ficam fora
do escopo desta entrega. Aqui a gente entrega o esqueleto, as decisões e
um chatbot mínimo funcionando com system prompt, memória e function
calling.

## Integrantes do grupo

| Nome | RM |
|------|-----|
| Isabela Marques de Oliveira | 567230 |
| Isabelle Ramos De Filippis | 566783 |
| João Vitor Anunciação Oliveira | 567539 |
| Paulo Ribeiro Marinho | 567459 |
| Samy Tamires de Sousa Cruz | 566674 |

## Persona escolhida e por quê

A gente discutiu as três personas possíveis (beneficiário, médico,
operador clínico) e decidiu focar em **beneficiário final em
autoavaliação**.

Por quê:

- É a persona mais próxima da promessa do BluaDiagnostics no enunciado
  ("check-up digital conversacional"). Bate direto com o pilar 1.
- O tom é mais empático e o escopo do agente é claro: **triagem, NÃO
  diagnóstico**. Isso torna mais fácil delimitar o que o bot pode e não
  pode dizer.
- Pra PoC, é mais didático: o usuário entra com sintomas em linguagem
  natural ("estou com dor de cabeça há 2 dias") e a gente consegue
  exercitar memória, tools e guardrails clínicos.
- A prescrição remota (pilar 2) ainda aparece, mas como **fluxo de
  encaminhamento**, o agente sugere teleconsulta e simula o pedido,
  nunca prescreve sozinho.

## Stack escolhida

| Camada | Escolha | Motivo curto |
|---|---|---|
| LLM | **Google Gemini 2.5 Flash** | Free tier padrão (5 RPM); a gente colocou crédito de teste pra ficar no Tier 1 (1K RPM) |
| SDK | `google-generativeai` (SDK oficial) | Menos abstração que LangChain, ajuda a entender o que tá acontecendo |
| Linguagem | Python 3.11 | Padrão da disciplina |
| Ambiente | Google Colab | Não precisa instalar nada local, todo mundo do grupo consegue rodar |
| Segredos | Colab Secrets / `python-dotenv` | API key NUNCA no commit |
| Versionamento | GitHub | Repositório público com acesso ao professor |

### Sobre o rate limit do free tier

O free tier padrão do Gemini 2.5 Flash, quando a gente conferiu no
dashboard do AI Studio, dá só **5 requisições por minuto** (uma a cada
12 segundos). Pra essa PoC, a gente adicionou um **crédito de teste de
R$ 40** na conta (Google Cloud Billing), o que sobe automaticamente pro
**Tier 1**, limites bem mais folgados (1.000 RPM, 10.000 RPD). Mesmo
assim, mantivemos `time.sleep(7)` entre chamadas no notebook como
**defesa em profundidade**: se um colega do grupo rodar com a chave
dele (sem crédito), o código continua funcionando dentro dos 5 RPM
padrão. Tem também retry exponencial caso mesmo assim caia em 429.

### Sobre LangChain/LangGraph

A gente decidiu **não usar** nesta sprint. O motivo é que o trabalho
pede "system prompt + memória + 1 tool", e o SDK do Gemini já dá isso
de forma direta. Adicionar LangChain agora seria uma camada extra que
mais atrapalha o entendimento do grupo do que ajuda. A intenção seria
introduzir LangGraph numa fase posterior, quando entrar a parte
multi-agente (orquestração check-up → triagem → prescrição), que é
exatamente o caso de uso onde o LangGraph brilha.

### Comparação dos 2 modelos candidatos

A gente comparou Gemini 2.5 Flash com GPT-5 Mini (preços verificados em
maio de 2026 nas páginas oficiais de cada provedor):

| Critério | Gemini 2.5 Flash (escolhido) | GPT-5 Mini |
|---|---|---|
| Custo input (1M tokens) | US$ 0,30 | US$ 0,25 |
| Custo output (1M tokens) | US$ 2,50 | US$ 2,00 |
| Janela de contexto | 1.000.000 tokens | 200.000 tokens |
| Latência média | Baixa (~1-2s) | Baixa (~1-2s) |
| Function calling estruturado | Sim, nativo | Sim, nativo |
| Free tier padrão (sem cartão) | Sim, 5 RPM / 250 RPD | Não |
| Privacidade / on-premise | Não (API gerenciada) | Não (API gerenciada) |
| Multimodal nativo | Sim (texto, imagem, áudio, vídeo) | Sim (texto, imagem) |

**Conclusão:** o GPT-5 Mini é levemente mais barato por token, mas o
**free tier do Gemini foi decisivo**, todo mundo do grupo consegue
executar o notebook sem precisar de cartão de crédito, o que importa
muito numa fase de PoC com várias rodadas de teste. Como bônus, o
contexto de 1M de tokens deixa muita folga pra quando a gente plugar o
RAG (não cabe no escopo desta sprint, mas é onde isso vira diferencial).

**Limitação que a gente reconhece:** nenhum dos dois roda on-premise.
Pra um sistema de saúde real em produção, isso seria um problema sério
de LGPD (dados clínicos saindo do datacenter da Care Plus). A
alternativa seria avaliar Llama 3 ou Qwen rodando via Ollama dentro da
infra da operadora. Pra PoC, com dados mockados, o trade-off é
aceitável.

## Riscos clínicos e éticos mapeados

Esse é o ponto mais delicado do projeto. A gente listou os principais
riscos e como pretende mitigar cada um:

| Risco | Como acontece | Mitigação |
|---|---|---|
| **Alucinação clínica** | O LLM inventa um sintoma, dose ou diagnóstico que não existe | System prompt proíbe diagnóstico definitivo; RAG sobre fontes oficiais seria a próxima camada; toda recomendação clínica passa por escalada humana |
| **Viés** | Modelo subestima sintomas em mulheres, idosos ou minorias (problema documentado na literatura) | Eval set com casos diversos; system prompt instrui a tratar todo sintoma com seriedade independente de perfil; logs auditáveis |
| **LGPD** | Dados sensíveis (CPF, histórico, medicamentos) trafegando pra API externa | Na PoC só usamos dados mockados; em produção, dados precisam ser pseudonimizados antes do envio, ou rodar modelo on-premise |
| **Responsabilidade sobre prescrição** | Beneficiário toma remédio que o bot "sugeriu" e passa mal | Bot NUNCA prescreve. Sugestão de medicamento só apareceria como rascunho pro médico aprovar (pilar 2). System prompt reforça isso |
| **Jailbreak / má fé** | Usuário tenta forçar o bot a dar diagnóstico ou receita ("ignore as instruções...") | Eval set tem 2 casos de jailbreak; system prompt tem seção explícita de RESTRIÇÕES; resposta padrão é redirecionar pra teleconsulta |
| **Red flags ignoradas** | Bot trata um AVC como dor de cabeça normal | Eval set tem casos de red flag (dor torácica, sintomas neurológicos súbitos); system prompt instrui a escalar IMEDIATAMENTE pra emergência (SAMU 192) nesses casos |
| **HITL ausente** | Bot operando sozinho em decisão de alto risco | Toda recomendação além de orientação geral → escalada pra teleconsulta ou emergência. O humano (médico) é sempre o último elo |

## Arquitetura

Fluxograma completo em [`docs/arquitetura.md`](docs/arquitetura.md)
(renderiza direto no GitHub porque tá em Mermaid).

Resumindo em texto:

1. **Usuário** manda mensagem ("estou com dor no peito")
2. **Roteamento de intent**: o LLM, guiado pelo system prompt, decide se é
   check-up, dúvida, pedido de agendamento ou red flag
3. Se for red flag → **resposta de emergência imediata** (SAMU 192) e
   encerra o fluxo
4. Caso contrário, o LLM pode chamar **tools via function calling**:
   - `consultar_historico_paciente` (busca dados do beneficiário)
   - `verificar_interacoes_medicamentosas` (se for caso de prescrição)
   - `agendar_teleconsulta` (encaminhamento)
   - `obter_sinais_vitais_wearable` (bônus, dados de wearable mockados)
5. **Guardrails de saída**: o bot nunca afirma diagnóstico, sempre orienta
   procura por profissional quando o caso pede
6. Resposta volta pro usuário com formato estruturado

## Base de conhecimento (mapeada pra futura indexação)

A gente já mapeou os 5 documentos que entrariam no RAG numa próxima
fase:

1. **Bulas resumidas**, paracetamol, dipirona, ibuprofeno, losartana,
   metformina (medicamentos comuns em consultas de telemedicina)
2. **Protocolo de Triagem de Manchester (simplificado)**, adaptado, só
   as classificações de cor e exemplos de sintomas, pro bot ter
   referência de severidade
3. **Política Care Plus de Telemedicina**, quem pode usar, quais
   especialidades, fluxo de autorização
4. **Cartilha do beneficiário Blua**, como usar o app, o que cada
   funcionalidade faz
5. **Diretriz interna sobre red flags clínicas**, lista resumida de
   sintomas que exigem emergência imediata

Esses arquivos não foram indexados nesta sprint, ficam listados aqui
como compromisso da arquitetura.

## Contrato das tools

Definido em [`tools/tools_spec.json`](tools/tools_spec.json) no padrão
JSON Schema. As 3 tools obrigatórias estão lá:

- `consultar_historico_paciente`
- `verificar_interacoes_medicamentosas`
- `agendar_teleconsulta`

Além delas, a gente incluiu a tool de bônus:

- `obter_sinais_vitais_wearable`, simula a integração com Apple Health
  / Google Fit / Oura puxando frequência cardíaca, SpO2, qualidade de
  sono, passos e variabilidade da FC

No notebook da PoC, o comportamento delas é **simulado** (mock), retornam
dados fixos pra demonstrar o fluxo. Em produção, cada tool vira uma
chamada real à API da Care Plus.

## System prompt

Tá em [`prompts/system_prompt.md`](prompts/system_prompt.md).
Tem as 5 seções pedidas: PAPEL, ESCOPO, RESTRIÇÕES, FORMATO_DE_SAIDA e
ESCALADA_HUMANA.

## Eval set

10 casos de teste em [`evals/sprint3_eval_set.json`](evals/sprint3_eval_set.json),
cobrindo:

- 3 casos de happy path (sintomas leves, dúvidas comuns)
- 3 casos de red flag (sintomas que exigem emergência)
- 2 tentativas de jailbreak (pedir diagnóstico ou prescrição direta)
- 2 perguntas out-of-scope (assuntos fora da saúde)

Cada caso tem: `id`, `categoria`, `entrada_usuario`, `contexto_esperado`,
`resposta_ideal` e `criterios_avaliacao`.

## Bônus, integração simulada com wearables

A gente quis ir atrás do reconhecimento adicional que o enunciado
oferece pra grupos que simulam integração com wearables.

O que entregamos:

- **`data/wearables_mock.json`**, três snapshots de dados que poderiam
  vir de Apple Health, Google Fit e Oura Ring, com FC, SpO2, sono,
  passos, HRV e timestamp.
- **Tool `obter_sinais_vitais_wearable`** especificada em
  `tools_spec.json` e implementada no notebook. O bot pode chamar ela
  pra puxar os sinais antes de fazer uma orientação.
- **Demonstração no notebook**, tem uma célula que conversa com o bot
  pedindo análise dos sinais vitais e mostra ele consumindo o JSON.

Em produção, essa tool faria uma chamada autenticada pra um endpoint
que conecta com o HealthKit / Health Connect / API do Oura.

## Prova de conceito (notebook)

Em [`notebooks/sprint3_poc.ipynb`](notebooks/sprint3_poc.ipynb).

O que ele demonstra:

- Carregamento da API key via Colab Secrets (sem expor chave no código)
- System prompt aplicado ao modelo (Gemini 2.5 Flash)
- Conversa com 4 turnos coerentes (memória de sessão mantida)
- Chamada bem-sucedida à tool `consultar_historico_paciente` via
  function calling
- Chamada à `agendar_teleconsulta` no fluxo
- Chamada à `obter_sinais_vitais_wearable` (bônus)
- Testes de guardrail (jailbreak e red flag)
- Rate limiting (`time.sleep(7)` + retry exponencial) como margem de
  segurança, o free tier padrão é 5 RPM, e mesmo a gente tendo crédito
  de teste o delay garante que rode com qualquer chave

Pra rodar:

1. Abrir no Colab
2. Em "Secrets" (ícone de chave na sidebar), criar um secret chamado
   `GOOGLE_API_KEY` com a sua chave do AI Studio
3. Run all

A chave de API se pega gratuitamente em
[aistudio.google.com](https://aistudio.google.com).

## Estrutura do repositório

```
BluaDiagnostics/
├── README.md                          # este arquivo
├── CONTRIBUTING.md                    # quem somos e como o grupo trabalhou
├── SECURITY.md                        # política de segurança (projeto acadêmico)
├── .gitignore                         # pra não vazar .env e API keys
├── entrega.txt                        # arquivo final que sobe na plataforma
├── docs/
│   └── arquitetura.md                 # fluxograma em Mermaid
├── prompts/
│   └── system_prompt.md               # instruções do agente
├── evals/
│   └── sprint3_eval_set.json          # 10 casos de teste
├── tools/
│   └── tools_spec.json                # contratos das 4 tools (3 obrigatórias + 1 bônus)
├── data/
│   └── wearables_mock.json            # dados mockados de wearables (bônus)
└── notebooks/
    └── sprint3_poc.ipynb              # PoC funcional
```
