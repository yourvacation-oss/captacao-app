# QualifiQ (captacao-app) - Guia do Claude

Leia antes de qualquer tarefa neste repositório.
Dono: Erick Faleiro (CEO Your Vacation). Fuso: America/Sao_Paulo.
Erick não é programador: explicar em linguagem simples e resumida, e executar sozinho tudo que não depender dele.

ATENÇÃO: este repositório é PÚBLICO. Nunca registrar aqui senhas, tokens, dados de clientes ou descrições de falhas de segurança.

## Estrutura
- index.html: o app inteiro (HTML + CSS + JS em um arquivo, ~6.800 linhas).
- apps-script-backend.js e _COLAR_NO_APPS_SCRIPT.txt: backend Google Apps Script (mesmo conteúdo).
- sw.js, manifest.json, ícones: PWA.
- logo-yourvacation.png: logo oficial usado na tela de entrada.
- Hospedagem: GitHub Pages, branch main. Push na main publica em 1 a 2 min.

## Endereço do app
- Endereço oficial: https://yourvacation-oss.github.io/captacao-app/ (repositório yourvacation-oss/captacao-app). Toda manutenção é feita aqui.
- O repositório antigo erickfaleirosilva-oss/captacao-app (erickfaleirosilva-oss.github.io/captacao-app) é só a PONTE: leva quem ainda abre o endereço antigo para o novo, junto com as pendências do celular. NÃO alterar, não fazer push nem apagar esse repositório.
- O código da ponte está no index.html (bloco "PONTE DE ENDEREÇO", QQ_ENDERECO_NOVO / QQ_HOST_ANTIGO). Só age quando o host é o antigo; no endereço novo fica inativo. O endereço novo recebe os dados pelo #qqmig= da URL. Não remover esse bloco nem a leitura do #qqmig sem combinar com o Erick.
- deploy.sh e LEIA-ME-TI.md ainda citam o endereço antigo.

## Seções do JavaScript (index.html)
- CONFIG: captadores, salas, regras, webhookUrl.
- API: post(lead) e getLeads().
- calcResult(): lógica Q / NQ / PARCIAL. Ribeirão Preto tem regra própria em calcRibeiraoPreto().
- goTo(tab): navegação entre home, hist, cdp, equipe, ddp, timerange.
- Telas: showLogin, showEquipe, showCdp, showHist, showDdp, showTimeRange.

## Cargos
captador, sublider, lider, supervisor_adm (leitura total, sem ações destrutivas), ceo.

## Operações
São Pedro, Caldas Novas, Atibaia, Ribeirão Preto.

## Fluxo de edição
1. Ler o trecho relevante antes de editar.
2. Editar com strings únicas.
3. Validar o JS: extrair o conteúdo do <script> e rodar node --check.
4. Versão: subir APP_VERSAO (index.html), "versao" em version.json e CACHE_NAME (sw.js). Sem isso os celulares não recebem o aviso "Atualizar".
5. Commit com mensagem clara e push na main.
6. Conferir no ar depois de 2 min (aba anônima) em yourvacation-oss.github.io/captacao-app.

## Cuidados
- A equipe é gerenciada pela aba Equipe do app (grava na planilha). CONFIG.captadores é só reserva de nomes, sem senhas: nunca colocar senha, hash ou senha padrão no index.html nem em comentários.
- Atibaia com operacao null em registros antigos é proposital.
- Hotéis de Caldas Novas (DDP e regras): Alta Vista, Atrium, Marina e Toulon, todos com calcAltaVista, idade 28–65 e pergunta do apartamento. Ao incluir um hotel novo de Caldas, atualizar DDP_CONFIG.operacoes, DDP_CONFIG.salaParaOperacao, CONFIG.salasDdpAuto, CONFIG.salasCaldas, calcResult(), _cnSala, _isCN e a condição da pergunta 'apto'.
- Sala digitada "São Pedro" (qualquer acento/maiúscula) é gravada como SPTR: salaDigitadaCanon() no app e normalizarSalaDigitada() no backend. "São Pedro Resort" é outro caso (vai por captador em normalizarSala).
- Hotel de Caldas digitado em "Outra sala" (lista HOTEIS_CALDAS, tolera 1 letra errada) vira sala Externo (externo-cn, regra de Caldas); o nome digitado vai em pontoCaptacao → coluna PontoCaptacao. Alta Vista/Atrium/Marina/Toulon digitados vão para a própria sala. Função salaDoCampoLivre().
- Backend retorna { status: 'ok' }: checar res.status === 'ok', não res.ok.
- Backend v9: toda escrita passa por comTrava() (LockService; se ocupado devolve status 'busy' e o app reenvia). Pesquisa sem ID é recusada (code 'sem_id'). Antes de gravar, o ID é procurado em todas as abas (abasDePesquisas()). update_sala_batch devolve not_found_ids; o app guarda essas marcações e tenta de novo (_reenfileirarMarcacaoSemPesquisa).
- Backend v14: findRowById usa índice por aba (_indiceDaAba: coluna de IDs lida uma vez por execução, Map ID → primeira linha). update_sala e update_sala_batch rodam sem a trava (doPost logo após equipe_sessao; no doGet só saveDDP usa trava).
- Crachá (backend v15 / app v2.40): equipe_login devolve um crachá assinado (HMAC-SHA256, segredo CRACHA_SEGREDO nas Propriedades do script, nunca no código) com nome, cargo, operação e validade de 30 dias. Toda leitura e gravação leva o crachá (POST: campo cracha; GET: ?cracha=); só ?action=listarEquipe é público. Cada um lê e grava a própria operação (SALA_OPERACAO; "Externo" e salas digitadas vão pela operação do captador) e sempre o que é dele; CEO grava tudo; CEO e supervisor_adm leem tudo. equipe_sessao com crachá devolve um crachá novo; equipe_cracha troca o hash da senha pessoal do celular por um crachá (mesmo limite de 5 tentativas). Redefinir senha ou desativar invalida o crachá. Emergência: desligarCracha() / ligarCracha() no Apps Script.
- SenhaOrigem (backend v16 / app v2.41): coluna da aba Equipe com migração | pessoal | redefinida (vazio = migração). equipe_cracha só aceita hash de senha pessoal ou redefinida. Login com senha migrada: captador recebe criarSenha (sem crachá; o crachá vem ao criar a senha pessoal); CEO, líder, sublíder e supervisor_adm recebem redefinir_pelo_ceo (não entram nem gerenciam). Senha nova igual à atual = senha_igual (minha_senha e redefinir). Funções do editor: prepararOrigemDasSenhas (cria e preenche a coluna pelo Equipe-Histórico) e redefinirGestoresComSenhaMigrada (provisórias aparecem só no registro da execução).
- Chave de acesso da lista de pesquisas: fica só em Propriedades do script (CHAVE_LEADS), nunca no código. Funções no Apps Script: gerarChaveLeads (gera, ainda não exige, mostra o link #qqchave=), ativarChaveLeads, desativarChaveLeads. O app guarda a chave em yv_chave_leads e manda como ?k=.
- Equipe centralizada (backend v12 / app v2.37): a lista de login vem da aba "Equipe" (Nome, Cargo, Operacao, Sala, SenhaHash, Ativo, ResetEm, CriadoPor, CriadoEm) por ?action=listarEquipe, que devolve só nome, cargo, operacao, sala e ativo. O app guarda cópia no celular (yv_equipe_central); sem cópia, usa CONFIG.captadores como reserva. Use sempre equipeAtual(). criarAbaEquipe() já foi rodado (a aba é a fonte da equipe). Entrada: POST equipe_login (responde só ok/erro; 5 erros = 15 min). Com internet, quem confere é a planilha; sem internet, só a senha pessoal já salva naquele celular. equipe_sessao (ao abrir) diz se o celular precisa entrar de novo (senha redefinida depois do último login ou desativado).
- Gestão da equipe (v2.36 / backend v11): aba Equipe do app grava na planilha via POST _action equipe_adicionar | equipe_redefinir | equipe_desativar | equipe_minha_senha. Toda ação leva por + porSenha (conferida no backend; 5 erros = bloqueio de 15 min). CEO gerencia todas as operações; líder/sublíder só a própria e não mexem em CEO/supervisão; ninguém desativa a si mesmo. Operação obrigatória ao cadastrar. Desativar = Ativo NÃO (nunca apagar). Histórico na aba "Equipe-Histórico". A senha pessoal nova também vai para a planilha (_sincronizarMinhaSenha). Se a planilha ainda tem a senha inicial do líder, o app pede a inicial uma vez para atualizar.
- Tela de entrada (v2.34, mockup aprovado): fundo #172438, dourado #F1D691, fonte Poppins, cores fixas (não seguem o tema claro). Passo 1 = busca (#loginSelectPanel, nomes só com 2+ letras); passo 2 = senha (#pwPanel, "Olá, [nome]!"), separado da busca. voltarParaBusca() = "Não é você? Trocar nome".
- Template literals dentro de innerHTML: erro de sintaxe deixa o app em branco.
- Reimplantar Apps Script: Implantar > Gerenciar implantações > editar > Nova versão. Assim a URL do webhook não muda. "Nova implantação" gera URL nova e exige trocar CONFIG.webhookUrl.
- Usuários podem ver versão antiga por cache: recarregar forçado (Cmd+Shift+R).

## Documentação completa
Fica fora do repositório, na pasta local do Erick (projects/captacao-app): PROMPT-IA-HANDOFF.md, ESTADO-ATUAL.md, RESUMO-HANDOFF-OUTRA-IA.md, PLAYBOOK-QUALIFIQ.md.
