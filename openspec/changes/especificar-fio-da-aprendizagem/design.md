# Design

## Context

Ver `proposal.md` para motivação e escopo. O repositório inspecionado contém os documentos v1.2 e OpenSpec, sem aplicação, manifesto de dependências ou testes existentes. O design é necessário por envolver autorização, persistência, temporalidade e dez capacidades relacionadas. Não há migração de sistema existente nem integração escolar nesta mudança.

Fontes: `docs/visao-do-produto.md` e `docs/requisitos.md`, versões 1.2. As decisões abaixo complementam suas ambiguidades. Em dois pontos, refinam explicitamente a redação original: ano reaberto continua elegível para herança; fechamento bloqueia escrita pedagógica no período, mas não ações atuais de atenção manual. A ocultação de grupos pequenos aplica-se aos dois perfis. Os documentos-base permanecem intactos nesta fase; sua conciliação documental está prevista nas tarefas.

## Goals / Non-Goals

**Goals:** uma aplicação web com autoridade de escrita no servidor, persistência transacional, regras determinísticas compartilhadas e rastreabilidade suficiente para explicar cada resultado. Usar a mesma interpretação de contexto em ficha, painel e verificação.

**Non-Goals:** microsserviços, infraestrutura de eventos distribuídos, reconstrução de todo o sistema como era conhecido em qualquer instante e mecanismo genérico de políticas configuráveis. As dez specs são fronteiras de comportamento, não dez serviços.

## Decisions

### D01 — Aplicação única e regras compartilhadas

Adotar uma aplicação web modular com interface, operações de servidor e armazenamento transacional. Separar conceitualmente autorização, registros e cálculos para que os mesmos cálculos alimentem painel, ficha e casos sintéticos. Alternativa descartada: microserviços e cálculos duplicados em cada tela, que acrescentariam sincronização e divergências sem necessidade do domínio.

Este design não prescreve framework, linguagem ou fornecedor. A escolha de bibliotecas concretas deve implementar esta arquitetura e os contratos, sem mudar regras ou tarefas; não criar serviços extras para materializar cada spec.

### D02 — Contexto pedagógico separado da matrícula

Um contexto pedagógico identifica aluno, ano letivo e etapa cursada, com catálogo fixado; matrículas registram a movimentação entre turmas. No mesmo ano/etapa, a transferência referencia o contexto existente e não copia avaliações. A origem da resposta é preservada. Uma mudança excepcional de etapa abre outro contexto. Alternativa descartada: usar exclusivamente a matrícula como identidade da avaliação, que duplicaria o registro na transferência.

Registros bimestrais pertencem ao contexto e período; respostas de habilidade e escrita possuem autoria e versão próprias. Observações são entradas independentes por autora, com limite compartilhado de 4.000 caracteres nos textos vigentes do aluno/bimestre. Revisões antigas não consomem esse limite de texto vigente.

### D03 — Autoria por resposta e autorização atual

Uma única resposta vigente por habilidade/contexto/bimestre e uma de escrita. Titular e substituta podem preencher campos ainda ausentes, mas não substituir resposta alheia. Correção requer autora, vínculo atual com o aluno e período aberto; não exige que a turma atual seja a original. Divergências ficam em observações próprias. Sem autora disponível, retificação é observação no período atual aberto, sem modificar avaliação ou indicador antigo.

Alternativa descartada: permitir que a sucessora sobrescreva respostas, por violar a autoria. Não há votação, média ou arbitragem automática entre professoras.

### D04 — Revisões preservadas e concorrência explícita

Gravar resposta/evento e sua revisão em uma transação, preservando versão anterior, autora, instante e contexto. Usar controle de versão para detectar tela desatualizada; verificar autorização, período, unicidade e limite de observações no salvamento. Em disputa pelo campo ausente, primeiro salvamento válido cria a resposta e o segundo recebe conflito. Não usar última gravação como substituição silenciosa.

Avaliações comuns recebem revisões; consolidação e reavaliação de lacuna são eventos pedagógicos distintos, com versões próprias. A trilha não exige infraestrutura genérica de event sourcing. Textos pedagógicos ficam fora de logs técnicos comuns.

### D05 — Tempo pedagógico e ordem de registro

Ordenar marcações pelo período pedagógico e depois pela ordem de registro dos eventos distintos no mesmo período. Revisões preservam a posição original do registro e apenas substituem sua versão vigente. Usar ordenação total persistente para evitar empate de relógio. Alternativa descartada: ordenar pelo instante da última edição, que faria uma correção antiga superar evidência posterior.

O painel histórico exibe a versão corrigida do período com revisão identificada e acesso ao original. Eventos pedagógicos de períodos posteriores não alteram esse indicador; podem aparecer como avanços posteriores. Não haverá opção adicional de reconstruir todo o painel como era conhecido antes da correção.

### D06 — Calendário único e fechamento

Um calendário por escola/ano, quatro bimestres, datas civis interpretadas em America/Sao_Paulo. Período passado usa último dia como referência, aberto ou reaberto; período em andamento usa hoje. Futuro não aceita registro nem indicadores. Fechamento é escolar, não por turma, aceita ausências e registra justificativa. Fechar ano exige quatro bimestres fechados; reabrir bimestre de ano fechado exige reabrir o ano.

Registrar que o ano já foi encerrado separadamente de seu estado atual permite reabrir para correção sem desfazer matrículas posteriores ou herança. Alternativa descartada: derivar elegibilidade apenas do estado fechado atual, que retiraria pendências durante uma correção administrativa.

### D07 — Atenção manual independente

Atenção manual tem identidade, justificativa original, autora, situação e eventos de comentário, encerramento e reabertura. Ações atuais são permitidas ao docente atualmente vinculado mesmo se o bimestre de abertura estiver fechado; conservam data atual e não reescrevem o passado. Somente a autora autorizada corrige o texto original. Coordenação consulta. Melhoras automáticas ou transições escolares não encerram atenção manual.

Alternativa descartada: armazenar somente um campo booleano de atenção no aluno, que perderia motivos, autoria e ciclo de vida.

### D08 — Ausência e catálogo preservados

Ausência é ausência, nunca nível numérico zero. Herança inclui a habilidade nunca avaliada em ano elegível, mas mostra a natureza documental da pendência. IDs estáveis deduplicam repetência. Catálogo é imutável por contexto; versão nova não reescreve texto ou denominador antigo. CAT-v1 usa exatamente o inventário da seção 4 dos Requisitos v1.2. O catálogo de cinco habilidades é fixture separada.

Alternativa descartada: equivalência por nome ou troca global do catálogo, que mudaria a interpretação histórica.

### D09 — Indicadores calculados na consulta

Calcular a partir dos registros persistidos com seleção explícita de contexto e referência. Reconsultar painel e ficha após confirmação de mutação. Manter conjuntos distintos para respostas do bimestre, resumo anual e herança; não preencher bimestre ausente com resposta passada.

Arredondamento por inteiros: para n/d não negativos e d>0, décimos percentuais q=floor((2000*n+d)/(2*d)); exibir q/10 e a fração original. Barras usam a fração sem arredondamento. Alternativa descartada: armazenar percentuais derivados como fonte de verdade, que exigiria invalidar valores a cada revisão.

Não é necessário cache persistente no primeiro desenho. A consulta deve ser medida conforme RNF-06 antes de acrescentar otimizações que alterem a arquitetura.

### D10 — Limitações de informação e apresentação

Histórico externo indisponível é aviso separado e não cria novo estado de atenção. Com escrita adequada e nenhum motivo conhecido, texto: Sem sinal de atenção nos dados disponíveis. Sem escrita e sem outro motivo, comparação automática incompleta. Um motivo conhecido basta para Em atenção, preservando avisos adicionais.

Ocultar agregados pedagógicos de N<5 em ambos os perfis, mantendo identificação, quantidade e consulta individual autorizada. Não tratar essa apresentação como anonimização ou barreira de autorização.

### D11 — Verificação por contratos e prioridade

P0: fluxo persistente turma → aluno → avaliação → histórico, lacunas, atenção, consulta individual da coordenação e indicadores essenciais (escrita, pendências, atenção e cobertura necessária à interpretação). Autorizações, revisões básicas, prevenção de perdas e acessibilidade dos fluxos acompanham P0. P1: operação de gestão, fechamento/reabertura completos, demais indicadores, tela de regras e verificação não funcional completa. P1 não é opcional para declarar MVP completo.

Três camadas de verificação: cálculos determinísticos; integração de autorização/persistência/concorrência; interface e demonstração. A tela executa fixtures sem gravar dados pedagógicos. Simulação de política na tela não prova autorização do servidor; CR-21 exige integração. Casos executados fora da tela têm resultado e evidência no relatório, sem sucesso fictício.

### D12 — Rastreabilidade e critérios críticos

Os IDs dos documentos-base permanecem válidos. A tabela associa contratos às specs; intervalos são inclusivos. Todos os CA continuam critérios de aceitação. O subconjunto crítico abaixo explicita a prioridade de verificação e não dispensa os demais requisitos.

| Capacidade | RN/RF e critérios de origem | CR associados | RNF |
| --- | --- | --- | --- |
| acesso-e-permissoes | RN06/11; RF-11/12; CA-11.4, CA-12.4 | CR-21 | RNF-02/03 |
| gestao-escolar | RN11; RF-12; CA-12.1–12.4 | CR-21 | RNF-01/03/05 |
| catalogo-pedagogico | RN03/10/14; RF-02; CA-02.1/02.3 | CR-22 | RNF-01 |
| calendario-letivo | RN01; RF-01/13; CA-01.2, CA-13.2/13.3 | CR-23 | RNF-01/03 |
| avaliacao-bimestral | RN02–05/14; RF-01–04/13; CA-01.1–01.3, CA-02.1–02.3, CA-03.1/03.2/03.3, CA-04.1–04.3, CA-13.1/13.4/13.5 | CR-19 | RNF-01/03/04/07 |
| historico-pedagogico | RN06; RF-05; CA-05.1–05.3, CA-04.2 | CR-23 | RNF-01/03 |
| lacunas-herdadas | RN07/14; RF-06/07; CA-06.1–06.3, CA-07.1–07.4 | CR-05/06/07/23 | RNF-01 |
| atencao-pedagogica | RN04/08/09; RF-08/09; CA-08.1–08.7, CA-09.1–09.5 | CR-08–10/14–17/23 | RNF-01/03 |
| paineis-e-indicadores | RN12/13; RF-10/11; CA-10.1–10.5, CA-11.1–11.3 | CR-01–04/11–20/24 | RNF-04/06 |
| regras-e-casos-de-referencia | RF-14; CA-14.1–14.4 | CR-01–24 | RNF-04 |

Critérios CA críticos: CA-01.1–01.3; CA-02.1–02.3; CA-03.1–03.3; CA-04.1–04.3; CA-05.1–05.3; CA-06.1–06.3; CA-07.1–07.4; CA-08.1–08.7; CA-09.1–09.5; CA-10.1–10.4; CA-11.3/11.4; CA-12.2–12.4; CA-13.1–13.5; CA-14.2–14.4. CR-01–CR-24 são obrigatórios. RNF-01/02/03/07 são verificados desde P0; RNF-04 acompanha cada fluxo; RNF-05/06 são verificados antes da entrega completa.

Além dos IDs existentes, verificar os cenários novos de titular/substituta, transferência com autoria, limite compartilhado de observações, correção retroativa, eventos no mesmo período, atenção manual após fechamento e reabertura sem retirada da herança. Não renumerar CA/CR para acomodar esses cenários.

## Risks / Trade-offs

- Autoria estrita pode impedir substituir uma resposta da antecessora → oferecer observação própria e retificação contextual, sem contornar a regra.
- Painéis antigos podem mudar após correção autorizada → identificar revisão e conservar versões; distinguir isso de avanço pedagógico posterior.
- Pendência sem avaliação pode ser interpretada como dificuldade → decompor os motivos e mostrar cobertura e ausência explicitamente.
- Um único calendário fecha todas as turmas → informar o alcance à coordenação e preservar a reabertura justificada.
- Referências documentais ainda têm redações anteriores → as decisões desta change e cenários correspondentes prevalecem nesses pontos; conciliar os documentos na implementação.
- Desempenho depende do ambiente → medir 30 operações com 500 alunos e cinco anos, registrar condições e corrigir consultas antes da conclusão.
- Uso real não foi validado → demonstrar somente dados sintéticos; validação pedagógica e definições institucionais permanecem externas.

## Migration Plan

Não há dados legados a migrar. Na futura implementação: preparar armazenamento e conta inicial; carregar CAT-v1 e base fictícia reinicializável; executar verificações; produzir candidata; copiar dados e aplicação; demonstrar restauração antes da entrega. A base de demonstração inclui dois anos, turma com pelo menos cinco e outra com quatro alunos, lacunas avaliadas e ausentes, escrita abaixo e atenção manual.

Entrega local reproduzível é a base operacional; hospedagem, se utilizada, acrescenta conexão protegida, recuperação confiável de acesso e backups diários com retenção técnica de 30 dias. Não declarar hospedagem pronta sem verificação. Reversão da candidata restaura versão de aplicação e cópia compatível dos dados sintéticos, preservando a trilha até o ponto da cópia e identificando o ambiente de demonstração.

## Open Questions

- Horário e formato exigidos pela banca em 18/11/2026 e disponibilidade de participantes para avaliação de usabilidade. Não mudam contratos; sem participantes, relatar apenas avaliação técnica.
- Nome do navegador da instituição e segundo navegador disponível para registrar a matriz de verificação.
- Ferramentas concretas para executar a arquitetura web e armazenamento transacional; não afetam os contratos nem autorizam ampliar escopo. Hospedagem não é condição para a execução local reproduzível.
