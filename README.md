# Roncon Studio — documentação do projeto Pilates

**Base da análise:** branch `main`, commit `39b2cf8` de `LucasArts2020/projeto-pilates`. O código foi inspecionado em uma cópia isolada do remoto. As regras e limitações descritas resultam de análise estática. O aplicativo e o instalador não foram executados nesta análise.

## Visão geral

Roncon Studio é um aplicativo desktop para administrar alunos de Pilates, organizar horários e chamadas, acompanhar a quitação da mensalidade do mês, agendar reposições e emitir uma ficha de matrícula em Word. Os dados operacionais ficam no perfil local do aplicativo. A implementação analisada não inclui servidor de aplicação, sincronização entre computadores, autenticação ou integração com meios de pagamento.

| Camada | Tecnologia e função |
| --- | --- |
| Interface | React 18, TypeScript, Tailwind CSS 4 e React Router 7. |
| Desktop | Electron 40, com processo principal separado da interface e comunicação por IPC. |
| Banco | SQLite em memória via SQL.js 1.13 e WebAssembly, com exportação para arquivo local. |
| Documentos | `docx` cria o arquivo e `file-saver` inicia o salvamento. |
| Desenvolvimento e entrega | Vite 7, ESLint 8, TypeScript e Electron Builder 26. As versões efetivas vêm de `package-lock.json`. |

## Executar e empacotar

Se estiver usando a pasta local que contém somente o Git inicializado, clone o projeto em uma pasta de código antes de rodar os comandos.

```powershell
git clone https://github.com/LucasArts2020/projeto-pilates.git
cd projeto-pilates
npm ci
npm run dev
```

Use uma versão do Node.js compatível com o Vite 7, como Node.js 22.12 ou superior na linha 22. O modo de desenvolvimento integra o servidor Vite ao Electron. A interface depende de `window.api`, criado pelo preload. Uma aba do navegador aberta diretamente no servidor Vite não fornece essa ponte.

| Comando | Resultado |
| --- | --- |
| `npm run dev` | Inicia o ambiente integrado Vite/Electron. |
| `npm run lint` | Analisa arquivos TS/TSX com ESLint e falha com avisos. |
| `npm run build` | Executa TypeScript, gera os bundles com Vite e chama o Electron Builder. |
| `npm run preview` | Serve a interface compilada para inspeção. Os fluxos com IPC exigem a janela Electron. |

O empacotamento usa o nome **Roncon Studio**, versão inicial `0.0.0`, arquivo ASAR e saída em `release/${version}`. Os alvos declarados são NSIS x64 no Windows, DMG no macOS e AppImage no Linux. O NSIS permite escolher o diretório de instalação e mantém os dados do perfil na desinstalação. O `appId` ainda é `YourAppID`. Os alvos representam a configuração do projeto e dependem do ambiente de build da plataforma.

O processo principal usa `VITE_DEV_SERVER_URL` em desenvolvimento e carrega `dist/index.html` no pacote. Ele deriva `APP_ROOT` e `VITE_PUBLIC` internamente. O binário `sql-wasm.wasm` é lido de `public/` em desenvolvimento e de `dist/` no pacote. A inicialização do banco falha se esse arquivo estiver ausente. Há um `.env` versionado na referência analisada. Seu conteúdo foi omitido desta documentação.

## Mapa do código

| Caminho | Responsabilidade |
| --- | --- |
| `src/main.tsx`, `src/App.tsx` | Entrada React, rotas por hash e layout com menu lateral e cabeçalho. |
| `src/pages/` | Telas de cadastro, alunos, turmas, histórico, reposições e configurações. |
| `src/components/cadastro/` | Formulário por etapas e controles de navegação. |
| `src/components/listagem/` | Tabela, busca, filtros e modais de detalhes, edição e emissão de documento. |
| `src/components/common/`, `src/components/layout/` | Componentes compartilhados, ícones, etapas, layout, wrappers e pop-ups. |
| `src/services/` | Chamadas ao preload, montagem de dados e geração local do DOCX. |
| `src/types/typeCadastro.tsx`, `src/utils/options.ts` | Tipos de aluno, agenda e chamada, além das opções padrão de horário, plano e pagamento. |
| `electron/main.ts` | Ciclo de vida da aplicação, janela, inicialização do banco, protocolo de mídia e handlers IPC. |
| `electron/preload.ts` | Expõe funções específicas em `window.api` por meio de `contextBridge`. |
| `electron/repositories/` | Consultas e mutações de alunos, pagamentos, chamada, capacidade e reposições. |
| `electron/database/DatabaseManager.ts` | Abre o banco, cria tabelas, aplica alterações de esquema e grava o arquivo. |
| `public/`, `dist-electron/` | Binário WASM de origem e bundles do processo principal/preload. Os bundles também aparecem versionados no commit analisado. |
| `vite.config.ts`, `electron-builder.json5` | Compilação e empacotamento. |

O caminho de uma operação com persistência é **tela → serviço → `window.api` → IPC → repositório → SQL.js → arquivo local**. A emissão de DOCX roda na interface. A leitura das fotos passa pelo protocolo `media:` registrado no processo principal.

## Funcionalidades e regras

| Rota | Fluxo |
| --- | --- |
| `/`, `/lista` | Entrada do aplicativo. Lista alunos, oferece busca e filtros de horário, dia, plano e vencimento. Abre detalhes, edição, exclusão com confirmação, confirmação de pagamento e emissão de documento. |
| `/cadastro` | Cadastro em três etapas para dados pessoais, agenda e informações financeiras. A agenda atual guarda pares de dia e horário. A foto opcional é salva separadamente e seu caminho segue no cadastro. |
| `/turmas` | Mostra grupos por dia e horário, combina alunos fixos com reposições pendentes da data correspondente e abre a chamada. |
| `/historico` | Mostra aulas registradas com data, turma, total e presentes. Abre a relação nominal com status e permite excluir a aula. |
| `/reposicao` | Lista até 50 registros recentes de falta, identifica os agendados e permite consultar vagas e reservar uma reposição por data e horário. |
| `/configuracoes` | Permite adicionar, remover e ajustar limites de ocupação por horário. |

### Alunos, agenda e mensalidade

O cadastro armazena nome, RG, CPF único, nascimento, contatos, endereço, foto, turma, agenda, matrícula, plano, mensalidade, forma de pagamento e vencimento. O plano pode ser mensal, bimestral, trimestral, semestral ou anual. As formas oferecidas são PIX, boleto, cartão e dinheiro. Esses valores são dados cadastrais.

Na confirmação de pagamento, o repositório usa o valor atual da mensalidade e grava um pagamento por aluno e mês de referência. A referência `YYYY-MM` é calculada em UTC e uma nova confirmação substitui o registro do mesmo mês. A listagem consulta esse mês e deriva `pago` e `dataPagamento`. O sistema registra a confirmação manualmente.

A agenda atual é gravada como JSON em `diasSemana`, no formato `[{"dia":"Seg","horario":"07:00"}]`. Alguns pontos ainda leem o formato antigo de siglas e o campo `horarioAula`. O serviço de documentos usa o cadastro para gerar `Ficha_<nome do aluno>.docx` a partir do modelo codificado em `DocumentGenerator.ts`.

### Chamada e histórico

O operador marca presença ou falta e finaliza a chamada. Alunos sem marcação são enviados como falta. O banco também aceita o status `justificado`, enquanto a tela de chamada apresenta presença e falta. O salvamento cria uma nova aula e insere os registros individuais em uma transação SQLite. O histórico agrega os registros por aula e consulta os detalhes por ID. Ao excluir uma aula, o repositório remove os registros de chamada e a aula em uma transação.

### Capacidade e reposição

A disponibilidade é calculada por data e horário, somando alunos fixos daquele dia e reposições ainda pendentes. As vagas correspondem a `max(0, limite - ocupação)`. A consulta de reposições usa os horários configurados quando há registros e recorre aos horários padrão quando a configuração está vazia, com limite padrão de seis vagas por horário.

A reserva vincula aluno, falta de origem, data, horário e observação opcional. A chamada marca como concluídas as reposições dos alunos presentes na mesma data. As reposições concluídas deixam de aparecer na lista de pendências da data e de ocupar vaga na consulta de disponibilidade.

## Persistência e modelo de dados

O Electron resolve o diretório de dados com `app.getPath("userData")`. O caminho efetivo do banco é exibido no log de inicialização em **Salvando DB em**. Os arquivos relevantes são:

- `dados.sqlite` contém os registros da operação.
- `images/` contém fotos com nomes UUID e extensão de origem. O banco guarda seus caminhos absolutos.
- Os DOCX são salvos pelo fluxo de exportação do usuário e ficam separados do banco.

| Tabela | Conteúdo e vínculos |
| --- | --- |
| `students` | Cadastro principal. `id` é a chave, `cpf` é único e `diasSemana` contém JSON. |
| `payments` | Pagamentos por `student_id`, mês, valor e data. Restrição única em aluno e mês. |
| `classes` | Instância de aula, com turma, data e descrição opcional. |
| `attendance` | Status de um aluno em uma aula. Referencia `students` e `classes`. |
| `class_config` | Capacidade por horário. `horario` é a chave e `limite` guarda a lotação. |
| `replacements` | Agendamento por aluno, com referência lógica à falta em `attendance_reference_id`, data, horário, observação e estado de conclusão. |

Na inicialização, `DatabaseManager` carrega o arquivo existente ou cria um banco novo. Usa `CREATE TABLE IF NOT EXISTS` e tenta adicionar `attendance_reference_id` e `concluida` para compatibilidade com bases anteriores. As mutações exportam o banco em memória e regravam o arquivo inteiro.

Para backup ou transferência, feche o aplicativo e copie o banco e a pasta de imagens juntos. Em outra conta ou máquina, os caminhos absolutos das fotos podem precisar de ajuste. O banco, as fotos e os DOCX contêm dados pessoais e precisam de acesso restrito.

## Contrato IPC

| Grupo | Funções de `window.api` → canais |
| --- | --- |
| Alunos | `listCadastros` → `get-alunos`; `createCadastro` → `add-aluno`; `updateCadastro` → `update-aluno`; `deleteCadastro` → `delete-aluno`. |
| Arquivo e mensalidade | `saveImage` → `save-image`; `confirmarPagamento` → `confirmar-pagamento`. |
| Chamada | `saveAttendance` → `save-attendance`; `getAttendanceHistory` → `get-attendance-history`; `getClassDetails` → `get-class-details`; `deleteClass` → `delete-class`. |
| Horários | `getLimits` → `get-limits`; `saveLimit` → `save-limit`; `deleteLimit` → `delete-limit`. |
| Reposição | `getAbsences` → `get-absences`; `checkAvailability` → `check-availability`; `scheduleReplacement` → `schedule-replacement`; `getReplacementsByDate` → `get-replacements-date`. |

Mutações de repositório normalmente retornam `{ success, error? }`. Leituras devolvem listas ou um mapa de horários. O envio de imagem recebe nome e bytes e retorna o caminho gravado.

## Pontos identificados na análise

| Área | Comportamento observado |
| --- | --- |
| Gravação | `DatabaseManager.save()` registra erros sem propagá-los. Uma operação pode informar sucesso mesmo quando a escrita em disco falha. A substituição do arquivo é direta. Duas instâncias com cópias em memória podem sobrescrever as alterações uma da outra. |
| Integridade | Não foi encontrado `PRAGMA foreign_keys = ON`. A exclusão de aluno remove pagamentos e cadastro, deixando chamadas e reposições sem limpeza explícita. A exclusão de aula não limpa referências de reposição à falta removida. |
| Capacidade | O agendamento faz um `INSERT` sem revalidar vaga, aluno, falta de origem ou duplicidade no repositório. Agendas antigas de dias como strings deixam de contar como ocupação na consulta de reposição. |
| Conclusão de reposição | A atualização usa aluno e data. Se houver mais de uma reposição desse aluno no mesmo dia, uma presença pode concluir todas, inclusive as de outro horário. |
| Duplicidade de aula | Cada salvamento insere outra aula. O esquema não impõe unicidade por turma e data. |
| Histórico | O serviço aceita filtros de datas e o repositório sabe aplicá-los, mas `preload.ts` invoca o canal sem encaminhar esses filtros. |
| Agenda | O filtro de horário da lista usa `horarioAula`. Horários gravados somente nos pares de `diasSemana` podem ficar fora do resultado. |
| Validação do cadastro | A etapa de agenda tem um retorno de bloqueio comentado. A validação do processo principal é limitada e parte das regras depende da interface. |
| Segurança local | Algumas consultas interpolam valores recebidos via IPC diretamente no SQL. O protocolo `media:` converte caminhos em acessos a arquivos sem restringi-los ao diretório de imagens. |
| Arquivos | Salvar foto e gravar cadastro são operações distintas. Troca, falha de cadastro ou exclusão podem deixar fotos sem referência. |
| Configuração | A atualização de limites chama exclusões e gravações separadas, sujeitas a alterações parciais se uma etapa falhar. |
| Entrega | O ícone referenciado em `main.ts` não aparece nos arquivos versionados da referência. O README original é o modelo do Vite. Não há suíte de testes ou script de testes no snapshot. |

## Roteiro de verificação funcional

1. Iniciar pelo Electron, cadastrar um aluno com foto e dois dias em horários diferentes, editar o cadastro e reiniciar para verificar persistência.
2. Conferir busca e filtros da lista. Confirmar pagamento duas vezes para o mesmo mês e verificar o estado exibido.
3. Emitir a ficha DOCX e conferir nome do arquivo e dados preenchidos.
4. Criar uma chamada com presença, falta e aluno sem marcação. Conferir o histórico e seus detalhes.
5. Definir um limite de horário, consultar vagas para uma falta, agendar a reposição e registrar a presença na data agendada. Verificar se a reposição sai das pendências.
6. Em uma base descartável, testar exclusão de aula e aluno com dependências e observar os efeitos sobre histórico, reposições e fotos.
7. Gerar um pacote da plataforma alvo, instalar e conferir abertura do banco, carregamento do WASM e leitura de fotos.
