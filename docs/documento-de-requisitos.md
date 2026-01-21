# Documento de Requisitos — Sistema de Gestão de Clínica (MVP)

## 1. Visão geral
Construir um sistema web para gestão de clínica com:

- Base de dados de médicos e suas agendas.
- Fila online em tempo real por unidade e especialidade.
- Pré-cadastro feito pela secretária para acelerar atendimento e agendamento.

## 2. Perfis de usuário e responsabilidades
### 2.1 Admin
- Cadastra unidades, médicos, especialidades, convênios.
- Define permissões e regras do sistema.
- Acessa relatórios e auditoria.

### 2.2 Secretária
- Realiza pré-cadastro e atualização de dados.
- Cria e gerencia agendamentos.
- Faz check-in e gerencia fila.
- Move paciente entre etapas (triagem, médico, finalizado).

### 2.3 Médico
- Visualiza sua fila e agenda.
- Chama próximo paciente.
- Inicia e finaliza atendimento.
- Registra dados do atendimento (prontuário opcional).

### 2.4 Triagem (opcional)
- Classifica prioridade e sinais básicos.
- Ajusta prioridade na fila conforme protocolo.

## 3. Conceitos centrais do domínio
### 3.1 Médico
- Identificação: nome, CRM, UF, especialidade(s).
- Operacional: unidades onde atende, salas, tempo médio de consulta.
- Agenda: dias e horários disponíveis, exceções, bloqueios.

### 3.2 Paciente
- Identificação: nome, data nascimento, documento (quando aplicável).
- Contato: telefone, WhatsApp, e-mail.
- Convênio: convênio, número da carteirinha.
- Observações: alergias, necessidades especiais, prioridade.

### 3.3 Pré-cadastro
- Nome e telefone.
- Motivo da consulta, especialidade desejada.
- Preferência de horário e unidade.
- Convênio (se houver).
- Status: rascunho, confirmado, convertido, descartado.

### 3.4 Agendamento
- Médico, unidade, data e hora.
- Status: pendente, confirmado, cancelado, remarcado, concluído.
- Origem: pré-cadastro, retorno, encaixe.

### 3.5 Fila online
Fila de execução do atendimento no dia. A fila é “tempo real”, enquanto o agendamento é “planejamento”.

### 3.6 Item de fila
- Paciente ou pré-cadastro convertido.
- Unidade, especialidade, médico (fixo ou “primeiro disponível”).
- Prioridade: normal, preferencial, urgência.
- Estado atual (ver máquina de estados).
- Timestamps: entrou na fila, chamado, iniciou, finalizou.

## 4. Jornada e máquina de estados da fila
### 4.1 Estados
- AGUARDANDO_CHECKIN
- AGUARDANDO_TRIAGEM
- EM_TRIAGEM
- AGUARDANDO_MEDICO
- CHAMADO
- EM_ATENDIMENTO
- FINALIZADO
- CANCELADO
- NAO_COMPARECEU

### 4.2 Transições permitidas
- Check-in realizado: AGUARDANDO_CHECKIN → AGUARDANDO_TRIAGEM ou AGUARDANDO_MEDICO
- Triagem inicia: AGUARDANDO_TRIAGEM → EM_TRIAGEM
- Triagem finaliza: EM_TRIAGEM → AGUARDANDO_MEDICO
- Médico chama: AGUARDANDO_MEDICO → CHAMADO
- Atendimento inicia: CHAMADO → EM_ATENDIMENTO
- Atendimento finaliza: EM_ATENDIMENTO → FINALIZADO
- Cancelamento: qualquer estado “antes do atendimento” → CANCELADO
- Não compareceu: AGUARDANDO_MEDICO ou CHAMADO → NAO_COMPARECEU

### 4.3 Regras de ordenação da fila
Ordenação recomendada:
1. Prioridade (urgência > preferencial > normal).
2. Horário agendado (dentro do seu grupo de prioridade).
3. Ordem de chegada (timestamp de check-in).

Regras adicionais:
- Permitir “encaixe” com tag e auditoria.
- Reordenação manual apenas por usuários autorizados, sempre gerando log.

## 5. Regras do pré-cadastro
### 5.1 Deduplicação
Ao criar pré-cadastro, o sistema deve procurar paciente existente usando:
- Telefone (principal).
- CPF (quando informado).
- E-mail (quando informado).

Se encontrar:
- Sugerir vínculo ao paciente existente.
- Evitar criar paciente duplicado.

### 5.2 Conversão
Pré-cadastro pode virar:
- Paciente + Agendamento (quando já marca uma data).
- Paciente + Fila (quando é atendimento do dia).
- Apenas Paciente (quando só quer cadastrar e marcar depois).

## 6. Lógica do app por telas
### 6.1 App da Secretária
**Tela: Dashboard do dia**
- Objetivo: visão do fluxo do dia por unidade e especialidade.
- Mostra: total aguardando triagem, total aguardando médico, chamados/em atendimento, tempo médio de espera.
- Ações: criar pré-cadastro, criar agendamento, fazer check-in, gerenciar fila.

**Tela: Pré-cadastro**
- Fluxo: preencher dados mínimos, escolher unidade/especialidade, opcionalmente escolher médico, salvar como rascunho ou confirmar.
- Ações: converter em agendamento, converter em fila do dia, atualizar dados e observações.

**Tela: Check-in**
- Opções: check-in a partir de agendamento do dia, paciente existente, pré-cadastro confirmado.
- Resultado: cria item de fila no estado apropriado e dispara atualização em tempo real.

**Tela: Gerenciador de fila**
- Lista por estados, estilo kanban ou lista com filtros (unidade, especialidade, médico).
- Ações: mover entre estados permitidos, ajustar prioridade com justificativa, cancelar ou marcar não compareceu, reordenar (se permitido) com log.

### 6.2 App do Médico
**Tela: Minha fila**
- Mostra: próximos pacientes com prioridade e tempo de espera; dados essenciais (nome, idade, motivo, convênio, observações).
- Ações: chamar próximo, iniciar atendimento, finalizar atendimento, pausar ou pular com justificativa.
- Regra: médico só vê filas vinculadas a ele ou ao setor ("primeiro disponível").

**Tela: Agenda**
- Mostra: agenda do dia e próximos dias.
- Ações: bloquear horários, abrir encaixes, configurar tempo padrão de consulta.

### 6.3 Painel do paciente (opcional)
- TV na clínica ou link via WhatsApp.
- Mostra: senha ou iniciais, status e previsão aproximada, chamada do próximo.

## 7. Eventos e tempo real
### 7.1 Eventos do sistema
- pre_cadastro_criado
- pre_cadastro_convertido
- agendamento_criado
- checkin_realizado
- fila_item_atualizado
- paciente_chamado
- atendimento_iniciado
- atendimento_finalizado

### 7.2 Quem recebe atualizações
- Secretária: sempre.
- Médico: apenas suas filas.
- Painel/TV: apenas chamada e próximos.
- Paciente: posição e chamado (se habilitado).

## 8. Banco de dados (visão lógica)
Tabelas mínimas:
- medicos
- especialidades
- unidades
- medico_unidade
- pacientes
- pre_cadastros
- agendamentos
- filas
- fila_itens
- checkins
- auditoria_logs

Relações importantes:
- medico N:N unidade
- medico N:N especialidade
- paciente 1:N agendamentos
- agendamento 0:1 checkin
- checkin 1:1 fila_item

## 9. Permissões e auditoria
Permissões (exemplos):
- Secretária: criar/editar pré-cadastro, check-in, alterar fila.
- Médico: chamar e atender, editar apenas seu atendimento.
- Admin: tudo.

Auditoria obrigatória:
- Reordenar fila.
- Alterar prioridade.
- Cancelar item.
- Mover item para estados críticos.

## 10. Requisitos não funcionais
- Desempenho: fila em tempo real sem atrasos perceptíveis.
- Consultas rápidas com índices no banco para: unidade, data, estado, médico.
- Disponibilidade: backups diários, logs de erro, monitoramento básico.
- Segurança: autenticação forte, trilhas de auditoria, segregação por unidade quando necessário.

## 11. Entregáveis do MVP
- Cadastro de médicos, especialidades e unidades.
- Pré-cadastro com deduplicação simples por telefone.
- Agendamento básico.
- Check-in e criação de fila do dia.
- Fila em tempo real para secretária e médico.
- Painel simples de chamada.

## 12. Casos de uso (resumo)
1. **Admin cadastra unidades, especialidades e médicos**.
2. **Secretária cria pré-cadastro** com dados mínimos e confirma.
3. **Secretária converte pré-cadastro** em agendamento futuro.
4. **Secretária realiza check-in** a partir de agendamento do dia.
5. **Secretária inclui paciente na fila do dia** (sem agendamento).
6. **Triagem ajusta prioridade** e registra sinais.
7. **Médico chama próximo paciente** e inicia atendimento.
8. **Médico finaliza atendimento** e encerra item de fila.
9. **Secretária cancela ou marca não compareceu** com justificativa.
10. **Admin audita mudanças críticas** em fila e prioridade.

## 13. Critérios de aceite (MVP)
### 13.1 Cadastro básico
- É possível cadastrar unidades, especialidades e médicos com associação N:N.
- Admin consegue definir permissões por perfil.

### 13.2 Pré-cadastro
- Secretária cria pré-cadastro com nome e telefone.
- Sistema sugere possível paciente existente por telefone/CPF/e-mail.
- Pré-cadastro pode ser salvo como rascunho ou confirmado.

### 13.3 Agendamento
- Secretária cria agendamento com médico, unidade, data e hora.
- Agendamento pode ser confirmado, remarcado ou cancelado.

### 13.4 Check-in e fila
- Check-in cria item de fila no estado apropriado.
- Fila é atualizada em tempo real para secretária e médico.
- Ordenação respeita prioridade, horário agendado e ordem de chegada.

### 13.5 Atendimento
- Médico consegue chamar próximo, iniciar e finalizar atendimento.
- Sistema registra timestamps de chamado/início/fim.

### 13.6 Auditoria
- Alterações críticas geram log com usuário, data/hora e justificativa.

## 14. Endpoints por recurso (proposta)
### 14.1 Autenticação e usuários
- `POST /auth/login` — login.
- `POST /auth/logout` — logout.
- `GET /usuarios/me` — dados do usuário.

### 14.2 Unidades
- `GET /unidades`
- `POST /unidades`
- `GET /unidades/{id}`
- `PATCH /unidades/{id}`
- `DELETE /unidades/{id}`

### 14.3 Especialidades
- `GET /especialidades`
- `POST /especialidades`
- `GET /especialidades/{id}`
- `PATCH /especialidades/{id}`
- `DELETE /especialidades/{id}`

### 14.4 Médicos
- `GET /medicos`
- `POST /medicos`
- `GET /medicos/{id}`
- `PATCH /medicos/{id}`
- `DELETE /medicos/{id}`
- `POST /medicos/{id}/unidades` — vincular unidades.
- `POST /medicos/{id}/especialidades` — vincular especialidades.

### 14.5 Pacientes
- `GET /pacientes`
- `POST /pacientes`
- `GET /pacientes/{id}`
- `PATCH /pacientes/{id}`
- `DELETE /pacientes/{id}`

### 14.6 Pré-cadastros
- `GET /pre-cadastros`
- `POST /pre-cadastros`
- `GET /pre-cadastros/{id}`
- `PATCH /pre-cadastros/{id}`
- `POST /pre-cadastros/{id}/confirmar`
- `POST /pre-cadastros/{id}/converter` — paciente, agendamento ou fila.

### 14.7 Agendamentos
- `GET /agendamentos`
- `POST /agendamentos`
- `GET /agendamentos/{id}`
- `PATCH /agendamentos/{id}`
- `POST /agendamentos/{id}/cancelar`
- `POST /agendamentos/{id}/remarcar`

### 14.8 Check-ins
- `POST /checkins`
- `GET /checkins/{id}`

### 14.9 Fila e itens
- `GET /filas` — por unidade/especialidade.
- `GET /fila-itens` — filtros por estado, médico, data.
- `POST /fila-itens` — criar item (check-in do dia).
- `PATCH /fila-itens/{id}` — mover estado permitido.
- `POST /fila-itens/{id}/chamar`
- `POST /fila-itens/{id}/iniciar`
- `POST /fila-itens/{id}/finalizar`
- `POST /fila-itens/{id}/cancelar`
- `POST /fila-itens/{id}/nao-compareceu`
- `POST /fila-itens/{id}/reordenar` — com log.
- `POST /fila-itens/{id}/prioridade` — com justificativa.

### 14.10 Auditoria
- `GET /auditoria-logs` — filtros por data, usuário, recurso.

### 14.11 Eventos (tempo real)
- `GET /eventos/stream` — SSE/WebSocket com eventos do sistema.

