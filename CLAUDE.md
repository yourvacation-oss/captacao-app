# QualifiQ (captacao-app) - Guia do Claude

Leia antes de qualquer tarefa neste repositório.
Dono: Erick Faleiro (CEO Your Vacation). Fuso: America/Sao_Paulo.
Erick não é programador: explicar em linguagem simples e resumida, e executar sozinho tudo que não depender dele.

ATENÇÃO: este repositório é PÚBLICO. Nunca registrar aqui senhas, tokens, dados de clientes ou descrições de falhas de segurança.

## Estrutura
- index.html: o app inteiro (HTML + CSS + JS em um arquivo, ~6.800 linhas).
- apps-script-backend.js e _COLAR_NO_APPS_SCRIPT.txt: backend Google Apps Script (mesmo conteúdo).
- sw.js, manifest.json, ícones: PWA.
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
- Captador desativado = linha comentada com // em CONFIG.captadores. Reativar = remover o //.
- Atibaia com operacao null em registros antigos é proposital.
- Sala digitada "São Pedro" (qualquer acento/maiúscula) é gravada como SPTR: salaDigitadaCanon() no app e normalizarSalaDigitada() no backend. "São Pedro Resort" é outro caso (vai por captador em normalizarSala).
- Hotel de Caldas digitado em "Outra sala" (lista HOTEIS_CALDAS, tolera 1 letra errada) vira sala Externo (externo-cn, regra de Caldas); o nome digitado vai em pontoCaptacao → coluna PontoCaptacao. Alta Vista/Atrium/Marina/Toulon digitados vão para a própria sala. Função salaDoCampoLivre().
- Backend retorna { status: 'ok' }: checar res.status === 'ok', não res.ok.
- Template literals dentro de innerHTML: erro de sintaxe deixa o app em branco.
- Reimplantar Apps Script: Implantar > Gerenciar implantações > editar > Nova versão. Assim a URL do webhook não muda. "Nova implantação" gera URL nova e exige trocar CONFIG.webhookUrl.
- Usuários podem ver versão antiga por cache: recarregar forçado (Cmd+Shift+R).

## Documentação completa
Fica fora do repositório, na pasta local do Erick (projects/captacao-app): PROMPT-IA-HANDOFF.md, ESTADO-ATUAL.md, RESUMO-HANDOFF-OUTRA-IA.md, PLAYBOOK-QUALIFIQ.md.
