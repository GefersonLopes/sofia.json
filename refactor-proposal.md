# Proposta de refatoracao - Sofia Latest Agents

## 1. Diagnostico do JSON atual

Arquivo analisado: `Sofia Latest Agents.json`

| Tipo | Quantidade atual | Quantidade proposta |
|---|---:|---:|
| Start | 1 | 1 |
| Condition | 5 | 2 |
| ConditionAgent | 2 | 1 |
| CustomFunction | 26 | 12 |
| LLM | 45 | 19 |
| Agent | 27 | 13 |
| DirectReply | 40 | 1 |
| Total | 146 | 50 |

Principais duplicacoes:

| Duplicacao | Evidencia | Acao proposta |
|---|---|---|
| `Gerente_regular` e `Gerente_primeira_execucao` | Mesmos 16 cenarios, mesmo input e prompt quase identico | Substituir por `Gerente_semantico` unico, seguido de limpeza de `selected_scenario` |
| Blocos `regular`, `primeira_execucao` e `ativo` | Cadeias equivalentes para `seleciona_tratamento`, exames, consultas, notas e replies | Reaproveitar os handlers `regular` como implementacao canonica |
| CustomFunction de copia | Nodes apenas copiam output para `notes`, `img_*_memory`, `contexto_conversa`, `notas_de_pos_consulta` | Mover para `Update Flow State` dos nodes LLM/Agent que produzem o dado; manter CustomFunctions curtos apenas para normalizacao e selecao de cenario |
| DirectReply repetidos | 37 replies retornam `{{ $flow.state.last_response }}` | Unificar em `resposta_final` |

Riscos atuais:

| Risco | Impacto |
|---|---|
| Duas fontes de verdade para roteamento | Divergencia silenciosa entre primeira execucao e execucoes seguintes |
| Start com 36 chaves, muitas runtime | Dificulta manutencao e mistura estado persistente com payload de chamada |
| Copias por JS/CustomFunction | Aumenta latencia e torna efeitos colaterais menos rastreaveis |
| Muitas chamadas LLM antes da resposta | Maior latencia, custo e variancia |
| `last_response` usado por replies apos alguns LLM sem update explicito | Risco de resposta final desatualizada |

## 2. Plano de refatoracao

Nodes removidos:

| Grupo | Remocao |
|---|---|
| `primeira_execucao_*` | Remover copias de handlers, notes e replies |
| `ativo_*` | Remover copias de `seleciona_tratamento` |
| Replies duplicados | Remover todos exceto `resposta_final` |
| CustomFunction de copia | Remover `set_notes_*`, `set_img_*`, `set_contexto_pc_*`, `set_notas_pos_consulta_*` |
| Conditions antigas de entrada | Remover `first_run` e `is_active_logic` como roteadores principais |

Nodes unificados:

| Atual | Novo |
|---|---|
| `Gerente_regular` + `Gerente_primeira_execucao` | `Gerente_semantico` |
| 3 switches de `seleciona_tratamento` | `seleciona_tratamento` unico |
| 40 DirectReply | `resposta_final` |
| 26 CustomFunction | `normaliza_entrada`, `normaliza_cenario_semantico`, setters curtos de `selected_scenario` e `set_last_response_seleciona_else` |

States persistentes mantidos no Start:

| State | Motivo |
|---|---|
| `first_run` | Compatibilidade historica |
| `contexto_conversa` | Memoria resumida usada pelos handlers |
| `todo_consulta` | Contexto funcional de consulta |
| `notes` | Notas operacionais acumuladas |
| `img_consulta_memory` | Memoria de imagem de consulta |
| `img_exame_memory` | Memoria de imagem de exame |
| `notas_de_pos_consulta` | Nota especifica de pos-consulta |
| `debug_scenario` | Compatibilidade com debug |
| `last_response` | Saida final padronizada |
| `selected_scenario` | Cenario limpo e rastreavel apos roteamento |
| `runtime_context` | Snapshot JSON dos dados recebidos via `overrideConfig.vars` |
| `postura_ativa_case` | Compatibilidade com postura ativa |
| `message_count` | Compatibilidade com contagem |
| `runtime_user_message` | Mensagem normalizada da chamada; todos os prompts downstream usam este state em vez de `question` |

Variaveis movidas para `overrideConfig.vars`:

`conversation_id`, `channel_id`, `contact_id`, `beneficiary_id`, `beneficiary_name`, `beneficiary_document`, `beneficiary_document_type`, `company_name`, `user_organization`, `contextType`, `has_digital_hospital`, `dependents_context`, `dependents_instruction`, `wallets_context`, `working_hours_instruction`, `is_working_hours`, `current_datetime`, `status_internacao`, `user_id`, `user_email`, `organization_secondary_name`, `dependents_count`, `notes`, `contexto_conversa`, `todo_consulta`, `attachment_context`, `has_upload`, `upload_type`, `list_conexa_appointments_url`.

Regra corrigida: `$vars` aparece somente em `normaliza_entrada`. Conditions, `ConditionAgentInput`, prompts de LLM, prompts de Agent e configuracoes de tool passam a usar apenas `$flow.state`.

Compatibilidade:

| Contrato atual | Compatibilidade |
|---|---|
| `question` | Mantido como mensagem textual principal |
| `streaming` | Mantido no payload da Prediction API |
| `overrideConfig.sessionId` | Mantido para separar conversas |
| States antigos | Continuam existindo quando usados internamente; runtime deve entrar por `vars` |
| Cenarios | Nomes preservados sem mudanca |

## 3. Novo desenho do Agentflow

Ordem:

1. `Start`
2. `normaliza_entrada`
3. `roteador_deterministico`
4. `Gerente_semantico` somente se nenhuma regra deterministica resolver
5. Handler unico por cenario
6. `resposta_final`

Nodes finais:

| Node | Tipo | Responsabilidade | Le estados | Escreve states |
|---|---|---|---|---|
| `Start` | Start | Inicializar estado persistente minimo | - | startState |
| `normaliza_entrada` | CustomFunction | Montar `runtime_context` com `overrideConfig.vars` e popular Flow State essencial | `question`, `$vars`, `$flow.sessionId` | `runtime_context`, `runtime_user_message`, dados externos, `first_run=false` |
| `roteador_deterministico` | Condition | Resolver regras obvias antes do LLM | `runtime_user_message`, `contextType`, dados de upload, contrato | - |
| `Gerente_semantico` | ConditionAgent | Classificar semanticamente quando regra deterministica nao resolveu | `runtime_user_message`, sinais leves, contexto resumido | - |
| `normaliza_cenario_semantico` | CustomFunction | Extrair string limpa se o gerente retornar JSON interno, por exemplo `{"output":"transferencia_enfermeira"}` | output do gerente | `selected_scenario` |
| `set_selected_*` | CustomFunction | Registrar cenario escolhido por regra deterministica | - | `selected_scenario` |
| `switch_cenario` | Condition | Enviar `selected_scenario` para o handler correspondente | `selected_scenario` | - |
| `seleciona_tratamento` | Condition | Separar `POST_CONSULTATION`, `HOSPITALIZATION`, `LAST_CONSULTATION` | `contextType` | - |
| Handlers `*_regular_*` | Agent/LLM | Executar comportamento funcional de cada cenario | dados normalizados e memoria | `last_response`, `notes`, memorias especificas |
| `set_last_response_seleciona_else` | CustomFunction | Resposta fallback do switch de tratamento | - | `last_response` |
| `resposta_final` | DirectReply | Responder `last_response` | `last_response` | - |

Edges principais:

| Origem | Destino |
|---|---|
| `Start` | `normaliza_entrada` |
| `normaliza_entrada` | `roteador_deterministico` |
| `roteador_deterministico` regra DEBUG | `set_selected_debug_tool_post` |
| `roteador_deterministico` regra CNPJ azul | `set_selected_programas_azul` |
| `roteador_deterministico` regra Conexa | `set_selected_cancelamento_conexa` |
| `roteador_deterministico` regra `contextType` | `set_selected_seleciona_tratamento` |
| `roteador_deterministico` regra humano/enfermeira | `set_selected_transferencia_enfermeira` |
| `roteador_deterministico` regra agradecimento | `set_selected_agradecimento_no_final` |
| `roteador_deterministico` regra imagem exame | `set_selected_analise_imagem_exame` |
| `roteador_deterministico` regra imagem consulta | `set_selected_analise_imagem_consultas` |
| `roteador_deterministico` regra ping | `set_selected_pong` |
| `roteador_deterministico` sem match | `Gerente_semantico` |
| `Gerente_semantico` outputs 0-15 | `normaliza_cenario_semantico` |
| `normaliza_cenario_semantico` | `switch_cenario` |
| `set_selected_*` | `switch_cenario` |
| `switch_cenario` outputs 0-15 | Handler do cenario correspondente |
| Todo handler terminal | `resposta_final` |

## 4. Contrato de entrada via Prediction API

```json
{
  "question": "Mensagem textual do usuario",
  "streaming": true,
  "overrideConfig": {
    "sessionId": "sanus-livechat-conversation-id",
    "vars": {
      "conversation_id": "conv_123",
      "channel_id": "livechat",
      "contact_id": "contact_123",
      "beneficiary_id": "benef_123",
      "beneficiary_name": "Maria Silva",
      "beneficiary_document": "12345678900",
      "beneficiary_document_type": "CPF",
      "company_name": "Sanus",
      "user_organization": "Empresa Cliente",
      "contextType": "POST_CONSULTATION",
      "notes": "",
      "contexto_conversa": "",
      "todo_consulta": "",
      "dependents_context": "",
      "wallets_context": "",
      "working_hours_instruction": "",
      "is_working_hours": "true",
      "current_datetime": "2026-05-12T10:30:00-03:00",
      "has_digital_hospital": "false",
      "status_internacao": "",
      "has_upload": "false",
      "upload_type": "",
      "attachment_context": ""
    }
  },
  "uploads": []
}
```

Com imagem/arquivo:

```json
{
  "question": "Segue a foto do meu exame",
  "streaming": true,
  "overrideConfig": {
    "sessionId": "sanus-livechat-conversation-id",
    "vars": {
      "conversation_id": "conv_123",
      "contextType": "",
      "has_upload": "true",
      "upload_type": "image",
      "attachment_context": "exame"
    }
  },
  "uploads": [
    {
      "data": "data:image/png;base64,...",
      "type": "file",
      "name": "exame.png",
      "mime": "image/png"
    }
  ]
}
```

## 5. Contrato interno de variaveis

| Variavel | Origem | Tipo esperado | Obrigatoria | Fallback | Uso |
|---|---|---|---|---|---|
| `question` | Prediction API | string | Sim | `""` | Roteamento e handlers |
| `sessionId` | `overrideConfig.sessionId` | string | Sim | id gerado pelo Flowise | Separacao de conversa |
| `conversation_id` | `overrideConfig.vars` | string | Opcional | `""` | Contexto e debug |
| `channel_id` | `overrideConfig.vars` | string | Opcional | `""` | Contexto |
| `contact_id` | `overrideConfig.vars` | string | Opcional | `""` | Contexto |
| `beneficiary_id` | `overrideConfig.vars` | string | Opcional | `""` | Handlers |
| `beneficiary_name` | `overrideConfig.vars` | string | Opcional | `""` | Personalizacao segura |
| `beneficiary_document` | `overrideConfig.vars` | string | Opcional | `""` | Contratos/programas |
| `beneficiary_document_type` | `overrideConfig.vars` | string | Opcional | `""` | Contratos |
| `company_name` | `overrideConfig.vars` | string | Opcional | `""` | Roteamento/programas |
| `user_organization` | `overrideConfig.vars` | string | Opcional | `""` | Roteamento/programas |
| `contextType` | `overrideConfig.vars` | enum string | Opcional | `""` | `seleciona_tratamento` |
| `notes` | state persistente ou vars | string | Opcional | `""` | Continuidade operacional |
| `contexto_conversa` | state persistente ou vars | string | Opcional | `""` | Contexto resumido |
| `todo_consulta` | state persistente ou vars | string | Opcional | `""` | Consulta/pos-consulta |
| `dependents_context` | `overrideConfig.vars` | string | Opcional | `""` | Contratos/dependentes |
| `wallets_context` | `overrideConfig.vars` | string | Opcional | `""` | Elegibilidade |
| `working_hours_instruction` | `overrideConfig.vars` | string | Opcional | `""` | Transferencia/horarios |
| `is_working_hours` | `overrideConfig.vars` | boolean string | Opcional | `""` | Transferencia |
| `current_datetime` | `overrideConfig.vars` | ISO datetime string | Opcional | `""` | Contexto temporal |
| `has_digital_hospital` | `overrideConfig.vars` | boolean string | Opcional | `"false"` | Internacao |
| `status_internacao` | `overrideConfig.vars` | string | Opcional | `""` | Internacao |
| `has_upload` | `overrideConfig.vars` | boolean string | Opcional | `"false"` | Roteamento deterministico |
| `upload_type` | `overrideConfig.vars` | string | Opcional | `""` | Roteamento deterministico |
| `attachment_context` | `overrideConfig.vars` | string | Opcional | `""` | Roteamento deterministico |
| `last_response` | Flow State | string | Sim interno | `""` | `resposta_final` |

## 6. Prompt refatorado do gerente

```text
Você é o roteador semântico da Sofia no Live Chat da Sanus. Escolha exatamente um cenário. Responda somente o nome do cenário, sem JSON, markdown, aspas, pontuação ou explicação.

Cenários permitidos:
seleciona_tratamento
analise_imagem_exame
exames
analise_imagem_consultas
consultas
transferencia_enfermeira
outras_situacoes
contratos
personalize_saude
agradecimento_no_final
programas_azul
programa_vermelho
pong
cancelamento_conexa
triagem_inicial_novo
debug_tool_post

Prioridade obrigatória:
1. Se a mensagem começar com DEBUG_TOOL_POST, retorne debug_tool_post.
2. Se houver CNPJ 09296295000160, 26203213000104, 04263318000116, 09305994000129 ou 02428624000130, retorne programas_azul.
3. Cancelamento relacionado à Conexa retorna cancelamento_conexa.
4. Escolha/confirmação de tratamento retorna seleciona_tratamento.
5. Pedido de humano, enfermeira ou orientação clínica sensível retorna transferencia_enfermeira.
6. Agradecimento ou despedida isolada retorna agradecimento_no_final.
7. Imagem/arquivo de exame retorna analise_imagem_exame.
8. Imagem/arquivo de consulta retorna analise_imagem_consultas.
9. Dúvida sobre exames sem imagem retorna exames.
10. Dúvida sobre consultas/agendamento sem imagem retorna consultas.
11. Contrato, plano, cobertura, elegibilidade, dependentes ou cadastro retorna contratos.
12. Programa azul retorna programas_azul; programa vermelho retorna programa_vermelho.
13. Perguntas sobre quem é a Sofia, quem é você, identidade, apresentação ou como a Sofia pode ajudar retornam triagem_inicial_novo.
14. Ping/health check retorna pong.
15. Se faltar informação para entender a solicitação, retorne triagem_inicial_novo.
16. Se não houver encaixe seguro, retorne outras_situacoes.
Quando houver dúvida, escolha o cenário mais específico.
```

## 7. Estrategia de performance

| Medida | Resultado esperado |
|---|---|
| `roteador_deterministico` antes do LLM | Evita chamada LLM para DEBUG, contrato azul, Conexa, tratamento por `contextType`, humano, agradecimento, imagem obvia e ping |
| Memory desligada no `Gerente_semantico` | Menos tokens e menor variancia |
| Input leve para roteador | Nao repetir `notes`/contexto pesado no classificador |
| Handlers canonicos | Menos manutencao e menos pontos de divergencia |
| `Update Flow State` no proprio node produtor | Menos CustomFunction e efeitos colaterais |
| Um `DirectReply` final | Saida previsivel e rastreavel via `last_response` |
| Streaming no handler final, nao no roteador | Melhor experiencia sem atrasar classificacao |

## 8. Estrategia de validacao

Execute os testes no fluxo antigo e no fluxo novo com o mesmo `sessionId` por caso, mas em ambientes separados. Compare cenario escolhido, resposta final, `last_response`, `notes` e memorias especificas.

| Caso | Entrada | Vars essenciais | Esperado |
|---|---|---|---|
| Saudacao inicial | `Oi` | `contextType=""` | `triagem_inicial_novo` |
| CPF/CNPJ contrato azul | `Meu CNPJ é 09296295000160` | - | `programas_azul` |
| Exames sem imagem | `Quero entender meu exame de sangue` | `has_upload=false` | `exames` |
| Exame com imagem | `Segue foto do meu exame` | `has_upload=true`, `attachment_context=exame` | `analise_imagem_exame` |
| Consulta sem imagem | `Quero marcar consulta` | `has_upload=false` | `consultas` |
| Consulta com imagem | `Segue guia da consulta` | `has_upload=true`, `attachment_context=consulta` | `analise_imagem_consultas` |
| Cancelamento Conexa | `Quero cancelar minha consulta da Conexa` | - | `cancelamento_conexa` |
| Pedido de enfermeira | `Preciso falar com uma enfermeira` | - | `transferencia_enfermeira` |
| Agradecimento final | `Obrigado, era isso` | - | `agradecimento_no_final` |
| Debug tool post | `DEBUG_TOOL_POST teste` | `conversation_id=conv_123` | `debug_tool_post` |
| Tratamento pos-consulta | `Quero escolher meu tratamento` | `contextType=POST_CONSULTATION` | `seleciona_tratamento` branch 0 |
| Tratamento internacao | `Continuar acompanhamento` | `contextType=HOSPITALIZATION` | `seleciona_tratamento` branch 1 |
| Tratamento ultima consulta | `Seguimento da ultima consulta` | `contextType=LAST_CONSULTATION` | `seleciona_tratamento` branch 2 |
| Outras situacoes | `Tenho uma duvida fora desses temas` | - | `outras_situacoes` ou `triagem_inicial_novo` se faltar contexto |

Validacao tecnica:

1. Importar `Sofia Latest Agents.refactored.proposal.json` em um Flowise de homologacao.
2. Confirmar que todos os nodes aparecem e que existe apenas um terminal: `resposta_final`.
3. Rodar a matriz acima com `streaming=false` para comparar logs.
4. Rodar novamente com `streaming=true` para validar Live Chat.
5. Conferir chamada com upload real para exame e consulta.
6. So substituir o fluxo atual apos paridade dos cenarios criticos.

## 9. Entrega

Arquivo gerado: `Sofia Latest Agents.refactored.proposal.json`

Resumo da proposta:

| Metrica | Atual | Proposto |
|---|---:|---:|
| Nodes | 146 | 50 |
| Edges | 145 | 92 |
| Gerentes semanticos | 2 | 1 |
| DirectReply | 40 | 1 |
| CustomFunction | 26 | 12 |
| Terminais | 42 caminhos | 1 node final |

Observacoes:

| Item | Observacao |
|---|---|
| Original | Nao foi alterado |
| JSON proposto | Validado localmente como JSON parseavel, sem ids duplicados, sem edges quebradas e com um unico terminal `resposta_final` |
| Uso de `$vars` | Restrito ao node `normaliza_entrada`; todos os demais nodes usam `$flow.state` |
| `selected_scenario` | Sempre preenchido por setters deterministicos ou por `normaliza_cenario_semantico` antes do `switch_cenario` |
| Importacao Flowise | Deve ser testada em homologacao antes de substituir producao |
| Contrato externo | Mantido com camada de compatibilidade por `overrideConfig.vars` e `sessionId` |
