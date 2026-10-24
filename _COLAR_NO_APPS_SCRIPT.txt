// ═══════════════════════════════════════════════════════
// Your Vacation — Qualificador de Captação
// Google Apps Script Backend — v21
// Cole este código em: Planilha > Extensões > Apps Script
// Depois: Implantar > Gerenciar implantações > Editar > Nova versão > Implantar
// ═══════════════════════════════════════════════════════

const SALA_SHEETS = {
  'Alta Vista':       'Alta Vista',
  'Atrium':           'Alta Vista',
  'Marina':           'Alta Vista',
  'Toulon':           'Toulon',       // aba dedicada (regras iguais ao Alta Vista no frontend)
  'Externo':          'Alta Vista',
  'Externo CN':       'Alta Vista',
  'Thermas SP':       'São Pedro',
  'SPTR':             'São Pedro',
  'São Pedro':        'São Pedro',   // nome de aba legado
  'São Pedro Resort': 'São Pedro',   // legado
  'Externo SP':       'São Pedro',
  'Atibaia':          'Atibaia',
  'Porta da sala':    'Atibaia',
  'Vest Casa':        'Atibaia',
  'Entrada Outlet':   'Atibaia',
  'Corredor Outlet':  'Atibaia',
};
const DEFAULT_SHEET = 'Alta Vista'; // fallback — evita criação de abas novas

const COLUNAS = [
  'ID','Data','Hora','Captador','Sala','Nome','Telefone','Cidade','Resultado','Tipo','Renda','Modo','EmSala','TipoSala','Venda',
  // Campos detalhados das respostas
  'PontoCaptacao','IdadeTitular','ProfissaoTitular','IdadeConjuge','ProfissaoConjuge',
  'Carro','Casa','Cartao','Viagens','Modalidade',
  // v2.21 — input numérico do carro (ano de 4 dígitos)
  'AnoCarro',
  // v2.23 — pergunta sobre apresentação de multipropriedade
  'ConheceuMultiprop',
  // v21 — quantas multipropriedades (1, 2 ou 3mais)
  'QtdMultiprop',
];
const N_COLS  = COLUNAS.length;
const COL_WIDTHS = [180,100,80,150,150,150,130,130,90,150,130,80,80,100,80,150,120,150,120,150,120,80,80,80,120,100,140];

function aplicarCabecalho(sheet) {
  const header = sheet.getRange(1, 1, 1, N_COLS);
  header.setValues([COLUNAS]);
  header.setBackground('#1a2340');
  header.setFontColor('#c9a84c');
  header.setFontWeight('bold');
  sheet.setFrozenRows(1);
  COL_WIDTHS.forEach((w, i) => sheet.setColumnWidth(i + 1, w));
}

function getOrCreateSheetByName(sheetName) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(sheetName);
  if (!sheet) {
    sheet = ss.insertSheet(sheetName);
    aplicarCabecalho(sheet);
    return sheet;
  }
  const cabecalhoAtual = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  const cabecalhoCorreto = COLUNAS.every((col, i) => cabecalhoAtual[i] === col);
  if (!cabecalhoCorreto) {
    sheet.getRange(1, 1, 1, N_COLS).setValues([COLUNAS]);
    const header = sheet.getRange(1, 1, 1, N_COLS);
    header.setBackground('#1a2340');
    header.setFontColor('#c9a84c');
    header.setFontWeight('bold');
    sheet.setFrozenRows(1);
    COL_WIDTHS.forEach((w, i) => sheet.setColumnWidth(i + 1, w));
  }
  return sheet;
}

// v2.29: sala digitada como "São Pedro" (qualquer acento/maiúscula/espaço) é gravada como SPTR.
// Protege também celulares que ainda estão com versão antiga do app.
function normalizarSalaDigitada(sala) {
  const n = String(sala || '').trim();
  const k = n.normalize('NFD').replace(/[\u0300-\u036f]/g, '').toLowerCase().replace(/\s+/g, ' ');
  return k === 'sao pedro' ? 'SPTR' : n;
}

function resolverSheet(sala) {
  return getOrCreateSheetByName(SALA_SHEETS[sala] || DEFAULT_SHEET);
}

// v14: índice de IDs por aba — a coluna de IDs de cada aba é lida uma vez por execução
// e guardada num Map (ID → primeira linha onde aparece).
const _indiceIds = {};
function _indiceDaAba(sheet) {
  const nome = sheet.getName();
  if (_indiceIds[nome]) return _indiceIds[nome];
  const mapa = new Map();
  const lastRow = sheet.getLastRow();
  if (lastRow >= 2) {
    sheet.getRange(2, 1, lastRow - 1, 1).getValues().forEach((r, i) => {
      const id = String(r[0]);
      if (!mapa.has(id)) mapa.set(id, i + 2);
    });
  }
  _indiceIds[nome] = mapa;
  return mapa;
}
function findRowById(sheet, id) {
  const linha = _indiceDaAba(sheet).get(String(id));
  return linha ? linha : -1;
}

// ── PI — armazenado em PropertiesService (invisível na planilha) ──
function getPIs() {
  const raw = PropertiesService.getScriptProperties().getProperty('pi_store');
  return raw ? JSON.parse(raw) : [];
}
function savePIs(arr) {
  // PropertiesService suporta até 500KB por propriedade — suficiente para meses de PI
  PropertiesService.getScriptProperties().setProperty('pi_store', JSON.stringify(arr));
}

// ── DDP — armazenado na aba "DDP" do Sheets ──
const DDP_SHEET_NAME = 'DDP';
const DDP_COLS = ['hotel','dateISO','ciPool','ciCot','ciConv','ciGrupos','salaPool','salaCot','salaConv','vendPool','vendCot','vendConv','visitantes'];

function getDDPSheet() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(DDP_SHEET_NAME);
  if (!sheet) {
    sheet = ss.insertSheet(DDP_SHEET_NAME);
    const header = sheet.getRange(1, 1, 1, DDP_COLS.length);
    header.setValues([DDP_COLS]);
    header.setBackground('#1a2340');
    header.setFontColor('#c9a84c');
    header.setFontWeight('bold');
    sheet.setFrozenRows(1);
    sheet.setColumnWidth(1, 160); // hotel
    sheet.setColumnWidth(2, 120); // dateISO
    for (let c = 3; c <= DDP_COLS.length; c++) sheet.setColumnWidth(c, 90);
  }
  return sheet;
}

function getDDPStore() {
  const sheet = getDDPSheet();
  const lastRow = sheet.getLastRow();
  if (lastRow < 2) return [];
  const rows = sheet.getRange(2, 1, lastRow - 1, DDP_COLS.length).getValues();

  function _normDate(v) {
    if (!v) return '';
    if (v instanceof Date) {
      const y = v.getFullYear();
      const m = String(v.getMonth() + 1).padStart(2, '0');
      const d = String(v.getDate()).padStart(2, '0');
      return y + '-' + m + '-' + d;
    }
    const s = String(v).trim().replace(/^DDP_/, '');
    if (/^\d{4}-\d{2}-\d{2}$/.test(s)) return s;
    const dt = new Date(s);
    if (!isNaN(dt)) {
      return dt.getFullYear() + '-' + String(dt.getMonth()+1).padStart(2,'0') + '-' + String(dt.getDate()).padStart(2,'0');
    }
    return s;
  }

  return rows
    .filter(r => r[0] && r[1]) // hotel e dateISO preenchidos
    .map(r => {
      const obj = {};
      DDP_COLS.forEach((col, i) => {
        obj[col] = (i >= 2) ? (Number(r[i]) || 0) : String(r[i] || '').trim();
      });
      obj.dateISO = _normDate(r[1]);
      return obj;
    })
    .filter(r => r.dateISO); // descarta linhas com data inválida
}

function saveDDPEntry(entry) {
  const sheet = getDDPSheet();
  const lastRow = sheet.getLastRow();
  const dateISO = String(entry.dateISO || '').replace(/^DDP_/, '');

  // Normaliza valor de data vindo do Sheets (pode ser Date object ou string)
  function _normSheetDate(v) {
    if (!v) return '';
    if (v instanceof Date) {
      const y = v.getFullYear();
      const m = String(v.getMonth() + 1).padStart(2, '0');
      const d = String(v.getDate()).padStart(2, '0');
      return y + '-' + m + '-' + d;
    }
    const s = String(v).trim().replace(/^DDP_/, '');
    if (/^\d{4}-\d{2}-\d{2}$/.test(s)) return s;
    const dt = new Date(s);
    if (!isNaN(dt)) {
      const y = dt.getFullYear();
      const mo = String(dt.getMonth() + 1).padStart(2, '0');
      const d = String(dt.getDate()).padStart(2, '0');
      return y + '-' + mo + '-' + d;
    }
    return s;
  }

  // Busca linha existente com mesmo hotel+dateISO
  let targetRow = -1;
  if (lastRow >= 2) {
    const hotelCol = sheet.getRange(2, 1, lastRow - 1, 1).getValues().flat();
    const dateCol  = sheet.getRange(2, 2, lastRow - 1, 1).getValues().flat();
    for (let i = 0; i < hotelCol.length; i++) {
      if (String(hotelCol[i]) === entry.hotel && _normSheetDate(dateCol[i]) === dateISO) {
        targetRow = i + 2;
        break;
      }
    }
  }

  // Monta rowData forçando dateISO como string (apóstrofo evita parse automático pelo Sheets)
  const rowData = DDP_COLS.map(col => {
    if (col === 'dateISO') return dateISO; // string pura
    if (col === 'hotel')   return entry.hotel || '';
    return entry[col] !== undefined ? Number(entry[col]) : 0;
  });

  if (targetRow > 0) {
    sheet.getRange(targetRow, 1, 1, DDP_COLS.length).setValues([rowData]);
  } else {
    sheet.appendRow(rowData);
  }
  // Força formato texto na célula dateISO para evitar conversão futura
  const writtenRow = targetRow > 0 ? targetRow : sheet.getLastRow();
  sheet.getRange(writtenRow, 2).setNumberFormat('@STRING@');
}

// ── CRUZAMENTO CHECK-INS × DDP ──
// Aba 'Checkins' (preenchida manualmente): Data | Hotel | Pool | Conv
//   Hotel deve ser 'Alta Vista', 'Atrium' ou 'Marina' (Caldas Novas)
// Cruza com a aba 'DDP' (gerada automaticamente) por data + hotel
// Grava o resultado em 'Cruzamento' (cria se não existir)
const CRUZAMENTO_SHEET = 'Cruzamento';
const CRUZAMENTO_COLS = ['dateISO','hotel','ciPool','ciCot','checkinsPool','checkinsConv','penetracao','atualizadoEm'];
function getCruzamentoSheet() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(CRUZAMENTO_SHEET);
  if (!sheet) {
    sheet = ss.insertSheet(CRUZAMENTO_SHEET);
    const header = sheet.getRange(1, 1, 1, CRUZAMENTO_COLS.length);
    header.setValues([CRUZAMENTO_COLS]);
    header.setBackground('#1a2340').setFontColor('#c9a84c').setFontWeight('bold');
    sheet.setFrozenRows(1);
    sheet.setColumnWidth(1, 110);
    sheet.setColumnWidth(2, 140);
    sheet.setColumnWidth(3, 80);
    sheet.setColumnWidth(4, 80);
    sheet.setColumnWidth(5, 110);
    sheet.setColumnWidth(6, 110);
    sheet.setColumnWidth(7, 110);
    sheet.setColumnWidth(8, 160);
  }
  return sheet;
}

// Lê a aba 'Checkins' e retorna dict {(dateISO, hotel): {pool, conv}}
function lerCheckins() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName('Checkins');
  if (!sheet) return {};
  const last = sheet.getLastRow();
  if (last < 2) return {};
  const values = sheet.getRange(2, 1, last - 1, 4).getValues();
  const map = {};
  values.forEach(r => {
    const data = String(r[0] || '').trim();
    const hotel = String(r[1] || '').trim();
    const pool = Number(r[2]) || 0;
    const conv = Number(r[3]) || 0;
    if (data && hotel) {
      const key = data + '|' + hotel;
      map[key] = { pool, conv };
    }
  });
  return map;
}

function atualizarCruzamentoCheckins() {
  const checkins = lerCheckins();
  if (Object.keys(checkins).length === 0) {
    Logger.log('⚠️  Aba "Checkins" vazia ou não existe. Preencha ela primeiro (Data | Hotel | Pool | Conv).');
    return;
  }
  const ddp = getDDPStore();
  const sheet = getCruzamentoSheet();
  // Apaga dados antigos
  const last = sheet.getLastRow();
  if (last > 1) sheet.getRange(2, 1, last - 1, CRUZAMENTO_COLS.length).clearContent();
  // Monta saída
  const out = [];
  const agora = new Date();
  Object.keys(checkins).forEach(key => {
    const [data, hotel] = key.split('|');
    const ddpEntry = ddp.find(d => d.dateISO === data && d.hotel === hotel);
    const ciPool  = ddpEntry ? Number(ddpEntry.ciPool  || 0) : 0;
    const ciCot   = ddpEntry ? Number(ddpEntry.ciCot   || 0) : 0;
    const c       = checkins[key];
    const totalCI = ciPool + ciCot;
    const totalCk = c.pool + c.conv;
    const penetracao = totalCI > 0 ? Math.round(totalCk / totalCI * 1000) / 10 : 0;
    out.push([
      data, hotel,
      ciPool, ciCot,
      c.pool, c.conv,
      penetracao,
      Utilities.formatDate(agora, 'America/Sao_Paulo', 'yyyy-MM-dd HH:mm:ss')
    ]);
  });
  // Ordena por data desc
  out.sort((a, b) => String(b[0]).localeCompare(String(a[0])));
  if (out.length > 0) {
    sheet.getRange(2, 1, out.length, CRUZAMENTO_COLS.length).setValues(out);
  }
  Logger.log('Cruzamento atualizado: ' + out.length + ' linhas');
}

// Cria gatilho diário que roda às 22h (atualiza o cruzamento automaticamente)
function criarGatilhoCruzamento() {
  // Remove gatilhos antigos desta função
  ScriptApp.getProjectTriggers().forEach(t => {
    if (t.getHandlerFunction() === 'atualizarCruzamentoCheckins') {
      ScriptApp.deleteTrigger(t);
    }
  });
  ScriptApp.newTrigger('atualizarCruzamentoCheckins')
    .timeBased()
    .atHour(22)
    .everyDays(1)
    .create();
  Logger.log('Gatilho diário criado: atualiza o cruzamento às 22h');
}

// ── ADICIONAR COLUNAS FALTANTES — sem sobrescrever dados ──
// Adiciona na aba especificada (ou em todas as abas da planilha) as colunas
// que estão no array COLUNAS mas faltam no cabeçalho da aba.
// Uso: executar uma vez após reimplantar o backend, para garantir que
// abas existentes tenham todas as 27 colunas antes do primeiro lead novo.
function adicionarColunasFaltantes() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheetsAlvo = ['Alta Vista', 'São Pedro', 'Atibaia', 'Thermas SP (Parque)'];
  let totalAdicoes = 0;
  sheetsAlvo.forEach(nomeAba => {
    const sheet = ss.getSheetByName(nomeAba);
    if (!sheet) return;
    const headerAtual = sheet.getRange(1, 1, 1, Math.max(1, sheet.getLastColumn())).getValues()[0];
    const faltam = COLUNAS.filter(col => headerAtual.indexOf(col) < 0);
    if (faltam.length === 0) {
      Logger.log('[' + nomeAba + '] já tem todas as ' + COLUNAS.length + ' colunas');
      return;
    }
    const startCol = sheet.getLastColumn() + 1;
    const newHeaders = faltam.map((col, i) => [col]);
    sheet.getRange(1, startCol, faltam.length, 1).setValues(newHeaders);
    // Aplica cor de cabeçalho pra ficar consistente
    const range = sheet.getRange(1, startCol, faltam.length, 1);
    range.setBackground('#1a2340').setFontColor('#c9a84c').setFontWeight('bold');
    sheet.setFrozenRows(1);
    Logger.log('[' + nomeAba + '] adicionadas ' + faltam.length + ' colunas: ' + faltam.join(', '));
    totalAdicoes += faltam.length;
  });
  Logger.log('===== TOTAL: ' + totalAdicoes + ' colunas adicionadas em todas as abas =====');
}

// v9: uma escrita por vez. Dois envios ao mesmo tempo esperam a vez em vez de
// os dois acharem "não existe" e gravarem a mesma pesquisa duas vezes.
function comTrava(fn) {
  const lock = LockService.getScriptLock();
  if (!lock.tryLock(30000)) return jsonResponse({ status: 'busy' }); // app reenvia depois
  try { return fn(); } finally { lock.releaseLock(); }
}

// Abas que guardam pesquisas (sem repetir)
function abasDePesquisas() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  return Array.from(new Set(Object.values(SALA_SHEETS))).map(n => ss.getSheetByName(n)).filter(Boolean);
}

// ── CRACHÁ (v15) ──
// O login devolve um crachá assinado aqui (HMAC-SHA256 com segredo nas Propriedades do
// script: CRACHA_SEGREDO). Leituras e gravações exigem o crachá; cada um só lê e grava a
// própria operação (CEO e supervisor_adm leem tudo; a pessoa sempre lê/grava o que é dela).
const CRACHA_VALIDADE_MS = 30 * 24 * 60 * 60 * 1000; // renovado a cada abertura do app
// Operação de cada sala/hotel ("Externo" existe em Caldas e em São Pedro: vai pela operação do captador)
const SALA_OPERACAO = {
  'Alta Vista': 'Caldas Novas', 'Atrium': 'Caldas Novas', 'Marina': 'Caldas Novas', 'Toulon': 'Caldas Novas', 'Externo CN': 'Caldas Novas',
  'Thermas SP': 'São Pedro', 'SPTR': 'São Pedro', 'São Pedro': 'São Pedro', 'São Pedro Resort': 'São Pedro', 'Externo SP': 'São Pedro', 'Thermas SP (Parque)': 'São Pedro',
  'Atibaia': 'Atibaia', 'Porta da sala': 'Atibaia', 'Vest Casa': 'Atibaia', 'Entrada Outlet': 'Atibaia', 'Corredor Outlet': 'Atibaia',
  'Frente da showroom': 'Ribeirão Preto', 'Praça de alimentação': 'Ribeirão Preto', 'Estacionamento': 'Ribeirão Preto', 'Carrossel': 'Ribeirão Preto', 'Parquinho': 'Ribeirão Preto',
};
function _segredoCracha() {
  const props = PropertiesService.getScriptProperties();
  let seg = props.getProperty('CRACHA_SEGREDO');
  if (!seg) { seg = Utilities.getUuid() + Utilities.getUuid(); props.setProperty('CRACHA_SEGREDO', seg); }
  return seg;
}
function _b64url(bytesOuTexto) { return Utilities.base64EncodeWebSafe(bytesOuTexto).replace(/=+$/, ''); }
function _assinarCracha(corpo) { return _b64url(Utilities.computeHmacSha256Signature(corpo, _segredoCracha())); }
function emitirCracha(m) {
  const agora = Date.now();
  const corpo = _b64url(JSON.stringify({ n: m.nome, c: m.cargo, o: m.operacao || '', iat: agora, exp: agora + CRACHA_VALIDADE_MS }));
  return corpo + '.' + _assinarCracha(corpo);
}
function _iguais(a, b) { // comparação de tempo constante
  if (a.length !== b.length) return false;
  let d = 0; for (let i = 0; i < a.length; i++) d |= a.charCodeAt(i) ^ b.charCodeAt(i);
  return d === 0;
}
// Devolve a pessoa (dados atuais da aba Equipe) ou null
function lerCracha(cracha) {
  const t = String(cracha || '');
  const partes = t.split('.');
  if (partes.length !== 2 || !partes[0] || !partes[1]) return null;
  if (!_iguais(_assinarCracha(partes[0]), partes[1])) return null;
  let p;
  try {
    const pad = partes[0] + '==='.slice((partes[0].length + 3) % 4);
    p = JSON.parse(Utilities.newBlob(Utilities.base64DecodeWebSafe(pad)).getDataAsString());
  } catch (e) { return null; }
  if (!p || !p.n || !p.exp || Date.now() > p.exp) return null;
  const m = _acharMembro(p.n);
  if (!m || !m.ativo) return null;
  if (m.resetEm && m.resetEm > new Date(p.iat).toISOString()) return null; // senha redefinida depois
  return { nome: m.nome, cargo: m.cargo, operacao: m.operacao || '' };
}
// Emergência: rodar desligarCracha() faz o backend aceitar pedidos sem crachá (como antes)
function desligarCracha() { PropertiesService.getScriptProperties().setProperty('CRACHA_DESLIGADO', 'SIM'); Logger.log('Crachá DESLIGADO: pedidos sem crachá são aceitos.'); }
function ligarCracha()    { PropertiesService.getScriptProperties().deleteProperty('CRACHA_DESLIGADO'); Logger.log('Crachá LIGADO: leituras e gravações exigem crachá.'); }
function _autenticar(cracha) {
  if (cracha) {
    const u = lerCracha(cracha);
    return u ? { ok: true, user: u } : { ok: false, code: 'cracha_invalido' };
  }
  if (PropertiesService.getScriptProperties().getProperty('CRACHA_DESLIGADO') === 'SIM')
    return { ok: true, user: { nome: '', cargo: 'ceo', operacao: '' } };
  return { ok: false, code: 'sem_cracha' };
}
// Operação de uma pesquisa: pela sala; se não der (ex.: "Externo", sala digitada), pela do captador
const _opCaptadorCache = {};
function _opDoCaptador(nome) {
  // v19: lê a aba Equipe uma vez só por execução (antes lia a aba inteira para cada captador)
  if (!_opCaptadorCache.__carregado) {
    (lerEquipe() || []).forEach(x => { _opCaptadorCache[_chaveNome(x.nome)] = x.operacao || ''; });
    _opCaptadorCache.__carregado = true;
  }
  const k = _chaveNome(nome);
  return k in _opCaptadorCache ? _opCaptadorCache[k] : '';
}
function _opDaPesquisa(sala, captador) { return SALA_OPERACAO[String(sala || '').trim()] || _opDoCaptador(captador); }
function _podeLer(u, op, captador) {
  if (u.cargo === 'ceo' || u.cargo === 'supervisor_adm') return true;
  if (captador && _chaveNome(captador) === _chaveNome(u.nome)) return true;
  return !!op && op === u.operacao;
}
function _podeGravar(u, op, captador) {
  if (u.cargo === 'ceo') return true;
  if (captador && _chaveNome(captador) === _chaveNome(u.nome)) return true;
  return !!op && op === u.operacao;
}
// Sala e Captador de uma linha (colunas lidas uma vez por aba por execução)
const _colunasCache = {};
function _salaECaptador(sheet, row) {
  // v19: lê só a linha pedida (antes lia as colunas Sala e Captador inteiras, ~25 mil linhas)
  const nome = sheet.getName();
  if (!_colunasCache[nome]) {
    const cab = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    _colunasCache[nome] = { iS: cab.indexOf('Sala'), iC: cab.indexOf('Captador'), n: cab.length };
  }
  const c = _colunasCache[nome];
  if (row < 2) return { sala: '', captador: '' };
  const v = sheet.getRange(row, 1, 1, c.n).getValues()[0];
  return { sala: c.iS >= 0 ? (v[c.iS] || '') : '', captador: c.iC >= 0 ? (v[c.iC] || '') : '' };
}
function _podeMarcar(u, sheet, row) { const x = _salaECaptador(sheet, row); return _podeGravar(u, _opDaPesquisa(x.sala, x.captador), x.captador); }

function doPost(e) {
  let d = null;
  try { d = JSON.parse(e.postData.contents); } catch (err) { return jsonResponse({ status: 'error', message: 'json' }); }
  // Entrada e sessão: não exigem crachá e não esperam a trava
  if (d._action === 'equipe_login')  return equipeLogin(d);
  if (d._action === 'equipe_sessao') return equipeSessao(d);
  if (d._action === 'equipe_cracha') return equipeCrachaPorHash(d);
  // Gestão da equipe: conferida pela senha de quem gerencia (por + porSenha)
  if (/^equipe_/.test(String(d._action || ''))) return comTrava(() => _doPost(e, null));
  const a = _autenticar(d.cracha);
  if (!a.ok) return jsonResponse({ status: 'error', code: a.code });
  // v14: marcações de sala (só atualizam células de linhas que já existem) rodam sem a trava
  if (d._action === 'update_sala' || d._action === 'update_sala_batch') return _doPost(e, a.user);
  return comTrava(() => _doPost(e, a.user));
}

function _doPost(e, user) {
  try {
    const data = JSON.parse(e.postData.contents);
    delete data.cracha;
    if (data.sala !== undefined && data.sala !== null) data.sala = normalizarSalaDigitada(data.sala);

    // saveDDP: salva/atualiza registro de DDP na aba DDP do Sheets
    if (data.action === 'saveDDP') {
      let dateISO = data.dateISO || '';
      if (dateISO.startsWith('DDP_')) dateISO = dateISO.slice(4);
      const hotel = data.hotel || '';
      if (!hotel || !dateISO) return jsonResponse({ status: 'error', message: 'hotel e dateISO obrigatórios' });
      if (!_podeGravar(user, SALA_OPERACAO[hotel] || '', null)) return jsonResponse({ status: 'error', code: 'proibido' });
      const entry = {
        hotel, dateISO,
        ciPool:     Number(data.ciPool)     || 0,
        ciCot:      Number(data.ciCot)      || 0,
        ciConv:     Number(data.ciConv)     || 0,
        ciGrupos:   Number(data.ciGrupos)   || 0,
        salaPool:   Number(data.salaPool)   || 0,
        salaCot:    Number(data.salaCot)    || 0,
        salaConv:   Number(data.salaConv)   || 0,
        vendPool:   Number(data.vendPool)   || 0,
        vendCot:    Number(data.vendCot)    || 0,
        vendConv:   Number(data.vendConv)   || 0,
        visitantes: Number(data.visitantes) || 0,
      };
      saveDDPEntry(entry);
      return jsonResponse({ status: 'ok' });
    }

    // getDDP via POST (alternativa ao GET)
    if (data.action === 'getDDP') {
      const hotel = data.hotel || '';
      const store = getDDPStore().filter(r => _podeLer(user, SALA_OPERACAO[r.hotel] || '', null));
      const rows = hotel ? store.filter(r => r.hotel === hotel) : store;
      return jsonResponse({ status: 'ok', rows });
    }

    // dedup: não mais necessário (Sheets já garante upsert por hotel+dateISO), mantido por compatibilidade
    if (data.action === 'dedup') {
      return jsonResponse({ status: 'ok', removed: 0, message: 'DDP agora usa aba no Sheets — dedup automático no upsert' });
    }

    // Gestão da equipe (v11)
    if (data._action === 'equipe_adicionar')  return equipeAdicionar(data);
    if (data._action === 'equipe_redefinir')  return equipeRedefinirSenha(data);
    if (data._action === 'equipe_desativar')  return equipeDesativar(data);
    if (data._action === 'equipe_minha_senha') return equipeMinhaSenha(data);

    // update_sala: atualiza EmSala, TipoSala e Venda de um lead existente
    if (data._action === 'update_sala') {
      const ss = SpreadsheetApp.getActiveSpreadsheet();
      const sheetsAlvo = abasDePesquisas();
      for (const sheet of sheetsAlvo) {
        const row = findRowById(sheet, data.id);
        if (row > 0) {
          if (!_podeMarcar(user, sheet, row)) return jsonResponse({ status: 'error', code: 'proibido', id: data.id });
          const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
          const set = (col, val) => {
            const i = cabecalho.indexOf(col);
            if (i >= 0) sheet.getRange(row, i + 1).setValue(val);
          };
          set('EmSala',   data.emSala   ? 'SIM' : '');
          set('TipoSala', data.tipoSala || '');
          set('Venda',    data.venda    ? 'SIM' : '');
          if (data.sala !== undefined) set('Sala', data.sala);
          return jsonResponse({ status: 'ok', id: data.id });
        }
      }
      return jsonResponse({ status: 'not_found', id: data.id });
    }

    // update_sala_batch: atualiza N leads em 1 chamada (evita quota exceeded)
    if (data._action === 'update_sala_batch' && Array.isArray(data.updates)) {
      const ss = SpreadsheetApp.getActiveSpreadsheet();
      const sheetsAlvo = abasDePesquisas();
      // Cache de cabeçalhos por sheet (evita releitura)
      const cabecalhoCache = {};
      sheetsAlvo.forEach(s => { cabecalhoCache[s.getName()] = s.getRange(1, 1, 1, s.getLastColumn()).getValues()[0]; });
      const results = { ok: 0, not_found: [], proibidos: [], errors: 0 };
      data.updates.forEach(u => {
        try {
          let found = false;
          for (const sheet of sheetsAlvo) {
            const row = findRowById(sheet, u.id);
            if (row > 0) {
              found = true;
              if (!_podeMarcar(user, sheet, row)) { results.proibidos.push(String(u.id)); break; }
              const cabecalho = cabecalhoCache[sheet.getName()];
              const set = (col, val) => {
                const i = cabecalho.indexOf(col);
                if (i >= 0) sheet.getRange(row, i + 1).setValue(val);
              };
              set('EmSala',   u.emSala   ? 'SIM' : '');
              set('TipoSala', u.tipoSala || '');
              set('Venda',    u.venda    ? 'SIM' : '');
              if (u.sala !== undefined) set('Sala', normalizarSalaDigitada(u.sala));
              results.ok++;
              break;
            }
          }
          if (!found) results.not_found.push(String(u.id));
        } catch (e) { results.errors++; }
      });
      // not_found: quantidade (compatível com versões antigas); not_found_ids: quais marcações não acharam a pesquisa
      return jsonResponse({ status: 'ok', processed: data.updates.length, updated: results.ok, not_found: results.not_found.length, not_found_ids: results.not_found, proibido_ids: results.proibidos, errors: results.errors });
    }

    // register_pi: salva em PropertiesService (sem criar aba na planilha)
    if (data._action === 'register_pi') {
      if (!_podeGravar(user, _opDaPesquisa(data.sala, data.captador), data.captador)) return jsonResponse({ status: 'error', code: 'proibido', id: data.id });
      const pis = getPIs();
      if (pis.some(p => String(p.id) === String(data.id))) {
        return jsonResponse({ status: 'duplicate', id: data.id });
      }
      pis.unshift({
        id:       String(data.id),
        date:     data.date     || '',
        time:     data.time     || '',
        captador: data.captador || '',
        sala:     data.sala     || '',
        nomeParc: data.nomeParc || '',
        dateISO:  data.dateISO  || '',
      });
      // Mantém últimos 1000 PIs
      if (pis.length > 1000) pis.pop();
      savePIs(pis);
      return jsonResponse({ status: 'ok', id: data.id });
    }

    // novo lead — salva na aba da sala correspondente
    // v9: linha sem ID não é gravada (virava linha em branco/sem dono na planilha)
    if (data.id === undefined || data.id === null || String(data.id).trim() === '') {
      return jsonResponse({ status: 'error', code: 'sem_id', message: 'Pesquisa sem ID não foi gravada' });
    }
    if (!_podeGravar(user, _opDaPesquisa(data.sala, data.captador), data.captador)) return jsonResponse({ status: 'error', code: 'proibido', id: data.id });
    // v9: procura o ID em todas as abas de pesquisas (a mesma pesquisa pode chegar com outra sala)
    let sheet = resolverSheet(data.sala || '');
    let existingRow = findRowById(sheet, data.id);
    if (existingRow < 0) {
      for (const outra of abasDePesquisas()) {
        if (outra.getName() === sheet.getName()) continue;
        const r = findRowById(outra, data.id);
        if (r > 0) { sheet = outra; existingRow = r; break; }
      }
    }
    if (existingRow > 0) {
      // Lead já existe — sincroniza EmSala, TipoSala e Venda se presentes no request
      if (data.emSala || data.tipoSala || data.venda) {
        const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
        const set = (col, val) => {
          const i = cabecalho.indexOf(col);
          if (i >= 0) sheet.getRange(existingRow, i + 1).setValue(val);
        };
        if (data.emSala)   set('EmSala',   'SIM');
        if (data.tipoSala) set('TipoSala', data.tipoSala);
        if (data.venda)    set('Venda',    'SIM');
      }
      return jsonResponse({ status: 'duplicate', id: data.id });
    }

    const ans = data._ans || {};
    // Normaliza data para DD/MM/YYYY fixo — evita que o Sheets converta para Date object
    function _fmtDate(v) {
      if (!v) return '';
      const s = String(v).trim();
      // Já está em DD/MM/YYYY
      if (/^\d{2}\/\d{2}\/\d{4}$/.test(s)) return s;
      // YYYY-MM-DD
      const m1 = s.match(/^(\d{4})-(\d{2})-(\d{2})/);
      if (m1) return m1[3] + '/' + m1[2] + '/' + m1[1];
      // Qualquer outro formato JS Date string
      const dt = new Date(s);
      if (!isNaN(dt)) {
        return String(dt.getDate()).padStart(2,'0') + '/' +
               String(dt.getMonth()+1).padStart(2,'0') + '/' + dt.getFullYear();
      }
      return s;
    }
    const dataFmt = _fmtDate(data.date);
    sheet.appendRow([
      data.id || '', dataFmt, "'" + (data.time || ''),
      data.captador || '', data.sala || '', data.nome || '',
      data.tel || '', data.cidade || '', data.verdict || '',
      data.tipo || '', data.renda || '', data.mode || '',
      data.emSala   ? 'SIM' : '',          // EmSala
      data.tipoSala || '',                 // TipoSala
      data.venda    ? 'SIM' : '',          // Venda
      // Campos detalhados
      data.pontoCaptacao || data.sala || '', // PontoCaptacao (nome digitado, ex.: hotel de Caldas em "Externo"; senão igual à Sala)
      ans.idadeMarido      || '',          // IdadeTitular
      ans.profissaoTitular || '',          // ProfissaoTitular
      ans.idadeConjuge     || '',          // IdadeConjuge
      ans.profissaoConjuge || '',          // ProfissaoConjuge
      ans.carro            || '',          // Carro (faixa — retrocompat com regras)
      ans.casa             || '',          // Casa
      ans.cartao           || '',          // Cartao
      ans.viagens          || '',          // Viagens
      data.modalidade      || '',          // Modalidade (pool/convidados/proprietarios)
      // v2.21 — input numérico do carro
      (typeof ans.anoCarro === 'number' ? ans.anoCarro : (ans.anoCarro || '')),  // AnoCarro
      // v2.23 — apresentação prévia de multipropriedade
      ans.conheceuMultiprop || '',         // ConheceuMultiprop
      ans.qtdMultiprop || '',              // QtdMultiprop (v21)
    ]);

    // Aplica cor na linha recém inserida e trava formato da coluna Data como texto
    const row = sheet.getLastRow();
    if (_indiceIds[sheet.getName()] && !_indiceIds[sheet.getName()].has(String(data.id))) _indiceIds[sheet.getName()].set(String(data.id), row);
    const nCols = sheet.getLastColumn();
    const idxData = COLUNAS.indexOf('Data');
    if (idxData >= 0) sheet.getRange(row, idxData + 1).setNumberFormat('@STRING@');
    const range = sheet.getRange(row, 1, 1, nCols);
    if (data.verdict === 'Q')       range.setBackground('#d4edda');
    if (data.verdict === 'PARCIAL') range.setBackground('#fff3cd');
    if (data.verdict === 'NQ')      range.setBackground('#f8d7da');

    return jsonResponse({ status: 'ok', id: data.id });
  } catch (err) {
    return jsonResponse({ status: 'error', message: err.message });
  }
}

function doGet(e) {
  const p = e && e.parameter ? e.parameter : {};
  // A lista de nomes da equipe é pública (a tela de entrada precisa dela antes do login)
  if (p.action === 'listarEquipe') return _doGet(e, null);
  const a = _autenticar(p.cracha);
  if (!a.ok) return jsonResponse({ status: 'error', code: a.code });
  const escreve = p.action === 'saveDDP'; // v14: update_sala roda sem a trava
  return escreve ? comTrava(() => _doGet(e, a.user)) : _doGet(e, a.user);
}

// ── Chave de acesso para ler a lista de pesquisas (v9) ──
// Fica guardada em Propriedades do script (CHAVE_LEADS), nunca no código.
// Sem chave ativa, tudo funciona como antes. Com chave ativa, a leitura da lista
// de pesquisas e das incompletas exige ?k=<chave>.
function chaveLeadsOk(params) {
  const chave = PropertiesService.getScriptProperties().getProperty('CHAVE_LEADS');
  return !chave || String(params.k || '') === chave;
}
// Passo 1: gera a chave (ainda não exige). O link para os celulares sai no registro.
function gerarChaveLeads() {
  const chave = Utilities.getUuid().replace(/-/g, '');
  PropertiesService.getScriptProperties().setProperty('CHAVE_LEADS_PENDENTE', chave);
  Logger.log('Chave gerada (ainda NÃO exigida). Link para enviar à equipe:');
  Logger.log('https://yourvacation-oss.github.io/captacao-app/#qqchave=' + chave);
}
// Passo 2: quando os celulares já tiverem a chave, passa a exigir.
function ativarChaveLeads() {
  const props = PropertiesService.getScriptProperties();
  const chave = props.getProperty('CHAVE_LEADS_PENDENTE') || props.getProperty('CHAVE_LEADS');
  if (!chave) { Logger.log('Rode gerarChaveLeads primeiro.'); return; }
  props.setProperty('CHAVE_LEADS', chave);
  props.deleteProperty('CHAVE_LEADS_PENDENTE');
  Logger.log('Chave ATIVA: a lista de pesquisas só é lida com a chave.');
}
// Volta a funcionar sem chave (emergência)
function desativarChaveLeads() {
  PropertiesService.getScriptProperties().deleteProperty('CHAVE_LEADS');
  Logger.log('Chave desativada: leitura liberada como antes.');
}

// ── EQUIPE CENTRALIZADA (v10) ──
// Aba "Equipe": a lista de quem entra no app. O app lê por ?action=listarEquipe
// e guarda uma cópia no celular. Nunca apagar linhas: desativar = Ativo NÃO.
const EQUIPE_SHEET = 'Equipe';
const EQUIPE_COLUNAS = ['Nome','Cargo','Operacao','Sala','SenhaHash','Ativo','ResetEm','CriadoPor','CriadoEm','SenhaOrigem'];
// SenhaOrigem: 'migração' (senha inicial migrada do app; vazio conta como migração), 'pessoal' (criada pela pessoa) ou 'redefinida' (pelo líder/CEO)
const APP_URL_INDEX = 'https://yourvacation-oss.github.io/captacao-app/index.html';

function _isoTexto(v) {
  if (!v) return '';
  if (Object.prototype.toString.call(v) === '[object Date]') return isNaN(v) ? '' : v.toISOString();
  return String(v).trim();
}
// Devolve null se a aba ainda não existe
function lerEquipe() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(EQUIPE_SHEET);
  if (!sheet || sheet.getLastRow() < 2) return null;
  const cab = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  const ix = {}; EQUIPE_COLUNAS.forEach(c => { ix[c] = cab.indexOf(c); });
  const g = (r, c) => ix[c] >= 0 ? r[ix[c]] : '';
  return sheet.getRange(2, 1, sheet.getLastRow() - 1, sheet.getLastColumn()).getValues()
    .filter(r => String(g(r, 'Nome')).trim())
    .map(r => ({
      nome:      String(g(r, 'Nome')).trim(),
      cargo:     String(g(r, 'Cargo')).trim(),
      operacao:  String(g(r, 'Operacao')).trim() || null,
      sala:      String(g(r, 'Sala')).trim() || null,
      ativo:     String(g(r, 'Ativo')).trim().toUpperCase() !== 'NÃO' && String(g(r, 'Ativo')).trim().toUpperCase() !== 'NAO',
    }));
}

// Lê os membros ativos de CONFIG.captadores no index.html (linhas comentadas com // ficam de fora)
function _membrosDoConfig(html) {
  const ini = html.indexOf('captadores: [');
  if (ini < 0) throw new Error('CONFIG.captadores não encontrado no app');
  const fim = html.indexOf('\n  ],', ini);
  const membros = [];
  html.slice(ini, fim).split('\n').forEach(linha => {
    if (!/^\s*\{\s*nome:/.test(linha)) return; // ignora comentadas e outras linhas
    const campo = re => { const m = linha.match(re); return m ? m[1] : null; };
    const nome = campo(/nome:\s*'([^']*)'/);
    if (!nome) return;
    membros.push({
      nome: nome.trim(),
      cargo: campo(/cargo:\s*'([^']*)'/) || 'captador',
      operacao: campo(/operacao:\s*'([^']*)'/),
      sala: campo(/sala:\s*'([^']*)'/),
      senhaHash: campo(/senhaHash:\s*'([0-9a-f]+)'/) || '',
    });
  });
  return membros;
}

// RODAR UMA VEZ (Apps Script > Executar): cria a aba Equipe com os membros ativos do app.
// Pode rodar de novo: só acrescenta quem ainda não estiver na aba.
function criarAbaEquipe() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(EQUIPE_SHEET);
  if (!sheet) {
    sheet = ss.insertSheet(EQUIPE_SHEET);
    const h = sheet.getRange(1, 1, 1, EQUIPE_COLUNAS.length);
    h.setValues([EQUIPE_COLUNAS]); h.setBackground('#1a2340'); h.setFontColor('#c9a84c'); h.setFontWeight('bold');
    sheet.setFrozenRows(1);
  }
  // ResetEm e CriadoEm como texto (o Sheets não converte para data)
  sheet.getRange('G:G').setNumberFormat('@STRING@');
  sheet.getRange('I:I').setNumberFormat('@STRING@');
  const html = UrlFetchApp.fetch(APP_URL_INDEX + '?t=' + Date.now(), { muteHttpExceptions: true }).getContentText();
  const membros = _membrosDoConfig(html);
  if (membros.length < 10) throw new Error('Li só ' + membros.length + ' membros do app — nada foi gravado.');
  const jaTem = new Set((lerEquipe() || []).map(m => m.nome.toLowerCase()));
  const agora = new Date().toISOString();
  const novos = membros.filter(m => !jaTem.has(m.nome.toLowerCase()))
    .map(m => [m.nome, m.cargo, m.operacao || '', m.sala || '', m.senhaHash, 'SIM', '', 'migração (app)', agora, 'migração']);
  if (novos.length) sheet.getRange(sheet.getLastRow() + 1, 1, novos.length, EQUIPE_COLUNAS.length).setValues(novos);
  Logger.log('Aba Equipe: ' + novos.length + ' membros acrescentados (' + membros.length + ' ativos no app).');
  return novos.length;
}

// ── GESTÃO DA EQUIPE PELO APP (v11) ──
// Toda ação vem com quem pede (por) e a senha dele (porSenha), conferida aqui.
// CEO gerencia todas as operações; líder e sublíder só a própria.
const OPERACOES_EQUIPE = ['Caldas Novas', 'São Pedro', 'Atibaia', 'Ribeirão Preto'];
const CARGOS_EQUIPE = ['captador', 'sublider', 'lider', 'supervisor_adm', 'ceo'];
const CARGOS_GESTORES = ['ceo', 'lider', 'sublider'];

function _sha256Hex(t) {
  return Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, String(t), Utilities.Charset.UTF_8)
    .map(b => ((b + 256) % 256).toString(16).padStart(2, '0')).join('');
}
function _chaveNome(t) { return String(t || '').normalize('NFD').replace(/[\u0300-\u036f]/g, '').toLowerCase().replace(/\s+/g, ' ').trim(); }
function _abaEquipe() { return SpreadsheetApp.getActiveSpreadsheet().getSheetByName(EQUIPE_SHEET); }
// Linha (1-based) e dados de um membro pelo nome (ignora acento/maiúscula)
function _acharMembro(nome) {
  const sheet = _abaEquipe();
  if (!sheet || sheet.getLastRow() < 2) return null;
  const cab = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  const ix = {}; EQUIPE_COLUNAS.forEach(c => { ix[c] = cab.indexOf(c); });
  const linhas = sheet.getRange(2, 1, sheet.getLastRow() - 1, sheet.getLastColumn()).getValues();
  const alvo = _chaveNome(nome);
  for (let i = 0; i < linhas.length; i++) {
    if (_chaveNome(linhas[i][ix.Nome]) === alvo) {
      const r = linhas[i], ativo = String(r[ix.Ativo]).trim().toUpperCase();
      return { sheet, ix, linha: i + 2, nome: String(r[ix.Nome]).trim(), cargo: String(r[ix.Cargo]).trim(),
        operacao: String(r[ix.Operacao]).trim(), senhaHash: String(r[ix.SenhaHash]).trim(), ativo: ativo !== 'NÃO' && ativo !== 'NAO',
        resetEm: ix.ResetEm >= 0 ? _isoTexto(r[ix.ResetEm]) : '',
        origem: (ix.SenhaOrigem >= 0 ? String(r[ix.SenhaOrigem]).trim() : '') || 'migração' };
    }
  }
  return null;
}
function _gravar(m, col, valor) {
  if (m.ix[col] < 0 && col === 'SenhaOrigem') m.ix[col] = _garantirColunaOrigem(m.sheet);
  if (m.ix[col] >= 0) m.sheet.getRange(m.linha, m.ix[col] + 1).setValue(valor);
}
// Cria a coluna SenhaOrigem na aba Equipe se ainda não existir; devolve o índice (0-based)
function _garantirColunaOrigem(sheet) {
  const cab = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  let i = cab.indexOf('SenhaOrigem');
  if (i < 0) { i = cab.filter(String).length; sheet.getRange(1, i + 1).setValue('SenhaOrigem'); }
  return i;
}
const CARGOS_SEM_SENHA_MIGRADA = ['ceo', 'lider', 'sublider', 'supervisor_adm'];
function _senhaMigrada(m) { return !m.origem || m.origem === 'migração' || m.origem === 'provisória'; }
// Gestores com a senha inicial migrada não entram nem gerenciam: precisam de redefinição pelo CEO
function _gestorComSenhaMigrada(m) { return (!m.origem || m.origem === 'migração') && CARGOS_SEM_SENHA_MIGRADA.indexOf(m.cargo) >= 0; }
function _registrarHistorico(por, acao, alvo, detalhe) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let h = ss.getSheetByName('Equipe-Histórico');
  if (!h) { h = ss.insertSheet('Equipe-Histórico'); h.getRange(1, 1, 1, 5).setValues([['Quando', 'Quem fez', 'Ação', 'Membro', 'Detalhe']]); h.setFrozenRows(1); }
  h.appendRow([new Date().toISOString(), por, acao, alvo, detalhe || '']);
}
// Confere a senha de alguém. Depois de 5 erros seguidos, bloqueia por 15 min.
function _conferirSenha(nome, senha) {
  const cache = CacheService.getScriptCache(), k = 'erros_' + _chaveNome(nome);
  const erros = Number(cache.get(k) || 0);
  if (erros >= 5) return { ok: false, code: 'bloqueado' };
  const m = _acharMembro(nome);
  if (!m || !m.ativo || !senha || _sha256Hex(senha) !== m.senhaHash) {
    cache.put(k, String(erros + 1), 900);
    return { ok: false, code: 'senha_nao_confere' };
  }
  cache.remove(k);
  return { ok: true, membro: m };
}
// Quem pede pode mexer neste membro/operação?
function _autorizarGestor(data, operacaoAlvo, cargoAlvo) {
  const c = _conferirSenha(data.por, data.porSenha);
  if (!c.ok) return c;
  const g = c.membro;
  if (_senhaMigrada(g)) return { ok: false, code: 'redefinir_pelo_ceo' }; // v17: provisória também não gerencia (precisa criar a pessoal antes)
  if (CARGOS_GESTORES.indexOf(g.cargo) < 0) return { ok: false, code: 'sem_permissao' };
  if (g.cargo !== 'ceo') {
    if (!operacaoAlvo || operacaoAlvo !== g.operacao) return { ok: false, code: 'outra_operacao' };
    if (cargoAlvo === 'ceo' || cargoAlvo === 'supervisor_adm') return { ok: false, code: 'sem_permissao' };
  }
  return { ok: true, gestor: g };
}
function _respostaEquipe(extra) { return jsonResponse(Object.assign({ status: 'ok', equipe: lerEquipe() || [] }, extra || {})); }

function equipeAdicionar(data) {
  const nome = String(data.nome || '').replace(/\s+/g, ' ').trim();
  const cargo = String(data.cargo || '').trim(), operacao = String(data.operacao || '').trim();
  const sala = String(data.sala || '').trim(), senha = String(data.senhaProvisoria || '');
  if (nome.length < 3) return jsonResponse({ status: 'error', code: 'nome_invalido' });
  if (CARGOS_EQUIPE.indexOf(cargo) < 0) return jsonResponse({ status: 'error', code: 'cargo_invalido' });
  if (OPERACOES_EQUIPE.indexOf(operacao) < 0) return jsonResponse({ status: 'error', code: 'operacao_obrigatoria' });
  if (senha.length < 4) return jsonResponse({ status: 'error', code: 'senha_curta' });
  const a = _autorizarGestor(data, operacao, cargo);
  if (!a.ok) return jsonResponse({ status: 'error', code: a.code });
  if (!_abaEquipe()) return jsonResponse({ status: 'error', code: 'sem_aba' });
  const agora = new Date().toISOString();
  const existente = _acharMembro(nome);
  if (existente && existente.ativo) return jsonResponse({ status: 'error', code: 'ja_existe' });
  if (existente) { // estava desativado: reativa com os dados novos
    if (a.gestor.cargo !== 'ceo' && existente.operacao && existente.operacao !== a.gestor.operacao) return jsonResponse({ status: 'error', code: 'outra_operacao' });
    _gravar(existente, 'Cargo', cargo); _gravar(existente, 'Operacao', operacao); _gravar(existente, 'Sala', sala);
    _gravar(existente, 'SenhaHash', _sha256Hex(senha)); _gravar(existente, 'Ativo', 'SIM'); _gravar(existente, 'ResetEm', agora);
    _gravar(existente, 'SenhaOrigem', 'redefinida');
    _registrarHistorico(a.gestor.nome, 'reativar', existente.nome, cargo + ' · ' + operacao + (sala ? ' · ' + sala : ''));
    return _respostaEquipe({ reativado: true });
  }
  const sheet = _abaEquipe();
  _garantirColunaOrigem(sheet);
  sheet.getRange(sheet.getLastRow() + 1, 1, 1, EQUIPE_COLUNAS.length)
    .setValues([[nome, cargo, operacao, sala, _sha256Hex(senha), 'SIM', '', a.gestor.nome, agora, 'redefinida']]);
  _registrarHistorico(a.gestor.nome, 'adicionar', nome, cargo + ' · ' + operacao + (sala ? ' · ' + sala : ''));
  return _respostaEquipe();
}
function equipeRedefinirSenha(data) {
  const senha = String(data.senhaProvisoria || '');
  if (senha.length < 4) return jsonResponse({ status: 'error', code: 'senha_curta' });
  const alvo = _acharMembro(data.nome);
  if (!alvo) return jsonResponse({ status: 'error', code: 'nao_encontrado' });
  const a = _autorizarGestor(data, alvo.operacao, alvo.cargo);
  if (!a.ok) return jsonResponse({ status: 'error', code: a.code });
  if (_sha256Hex(senha) === alvo.senhaHash) return jsonResponse({ status: 'error', code: 'senha_igual' });
  _gravar(alvo, 'SenhaHash', _sha256Hex(senha));
  _gravar(alvo, 'ResetEm', new Date().toISOString());
  _gravar(alvo, 'SenhaOrigem', 'redefinida');
  _registrarHistorico(a.gestor.nome, 'redefinir senha', alvo.nome, '');
  return _respostaEquipe();
}
function equipeDesativar(data) {
  const alvo = _acharMembro(data.nome);
  if (!alvo) return jsonResponse({ status: 'error', code: 'nao_encontrado' });
  const a = _autorizarGestor(data, alvo.operacao, alvo.cargo);
  if (!a.ok) return jsonResponse({ status: 'error', code: a.code });
  if (_chaveNome(alvo.nome) === _chaveNome(a.gestor.nome)) return jsonResponse({ status: 'error', code: 'nao_pode_si_mesmo' });
  _gravar(alvo, 'Ativo', 'NÃO');
  _registrarHistorico(a.gestor.nome, 'desativar', alvo.nome, '');
  return _respostaEquipe();
}
// A pessoa troca a própria senha (ao criar a senha pessoal ou em "Alterar senha")
function equipeMinhaSenha(data) {
  const nova = String(data.senhaNova || '');
  if (nova.length < 4) return jsonResponse({ status: 'error', code: 'senha_curta' });
  const c = _conferirSenha(data.nome, data.senhaAtual);
  if (!c.ok) return jsonResponse({ status: 'error', code: c.code });
  if (_gestorComSenhaMigrada(c.membro)) return jsonResponse({ status: 'error', code: 'redefinir_pelo_ceo' });
  if (_sha256Hex(nova) === c.membro.senhaHash) return jsonResponse({ status: 'error', code: 'senha_igual' });
  _gravar(c.membro, 'SenhaHash', _sha256Hex(nova));
  _gravar(c.membro, 'SenhaOrigem', 'pessoal');
  _registrarHistorico(c.membro.nome, 'trocar a própria senha', c.membro.nome, '');
  return _respostaEquipe({ cracha: emitirCracha(_acharMembro(c.membro.nome)) });
}

// RODAR UMA VEZ (Apps Script > Executar) antes de usar a v16: cria a coluna SenhaOrigem e
// preenche pelo histórico (última troca da própria senha = pessoal; redefinição/cadastro = redefinida;
// o resto fica migração). Mostra no registro quem ainda está com a senha inicial migrada.
function prepararOrigemDasSenhas() {
  const sheet = _abaEquipe();
  if (!sheet) throw new Error('Aba Equipe não encontrada');
  const iO = _garantirColunaOrigem(sheet);
  const cab = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  const iN = cab.indexOf('Nome'), iC = cab.indexOf('Cargo'), n = sheet.getLastRow() - 1;
  if (n < 1) return;
  const ultimo = {};
  const h = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Equipe-Histórico');
  if (h && h.getLastRow() > 1) {
    h.getRange(2, 1, h.getLastRow() - 1, 4).getValues().forEach(r => {
      const acao = String(r[2]), alvo = _chaveNome(r[3]);
      if (acao === 'trocar a própria senha') ultimo[alvo] = 'pessoal';
      else if (/^(redefinir senha|adicionar|reativar)/.test(acao)) ultimo[alvo] = 'redefinida';
    });
  }
  const linhas = sheet.getRange(2, 1, n, sheet.getLastColumn()).getValues();
  const origens = linhas.map(r => [String(r[iO] || '').trim() || ultimo[_chaveNome(r[iN])] || 'migração']);
  sheet.getRange(2, iO + 1, n, 1).setValues(origens);
  const conta = {}; origens.forEach(o => { conta[o[0]] = (conta[o[0]] || 0) + 1; });
  const gestores = linhas.map((r, i) => ({ nome: r[iN], cargo: r[iC], o: origens[i][0] })).filter(x => x.o === 'migração' && CARGOS_SEM_SENHA_MIGRADA.indexOf(String(x.cargo)) >= 0);
  Logger.log('SenhaOrigem: ' + JSON.stringify(conta));
  Logger.log('Gestores ainda com a senha inicial migrada (precisam de redefinição): ' + (gestores.map(g => g.nome + ' (' + g.cargo + ')').join(', ') || 'nenhum'));
}
// RODAR NO EDITOR (só quem é dono da planilha): dá senha provisória nova para cada gestor
// (CEO, líder, sublíder, supervisão) que ainda está com a senha inicial migrada.
// As senhas aparecem só no registro desta execução: passe para cada um pessoalmente.
function redefinirGestoresComSenhaMigrada() {
  const sheet = _abaEquipe();
  if (!sheet) throw new Error('Aba Equipe não encontrada');
  _garantirColunaOrigem(sheet);
  const nomes = (lerEquipe() || []).filter(x => x.ativo && CARGOS_SEM_SENHA_MIGRADA.indexOf(x.cargo) >= 0).map(x => x.nome);
  const agora = new Date().toISOString(); let feitos = 0;
  nomes.forEach(nome => {
    const m = _acharMembro(nome);
    if (!m || !_senhaMigrada(m)) return;
    const num = parseInt(Utilities.getUuid().replace(/-/g, '').slice(0, 10), 16) % 1000000;
    const prov = String(num).padStart(6, '0');
    _gravar(m, 'SenhaHash', _sha256Hex(prov)); _gravar(m, 'ResetEm', agora); _gravar(m, 'SenhaOrigem', 'redefinida');
    _registrarHistorico('editor do Apps Script', 'redefinir senha', m.nome, 'gestor com senha migrada');
    Logger.log(m.nome + ' (' + m.cargo + '): senha provisória ' + prov);
    feitos++;
  });
  Logger.log(feitos + ' gestor(es) com senha provisória nova. No 1º acesso o app pede para criar a senha pessoal.');
}

// Entrada no app: responde só ok ou erro (mesmo limite de tentativas)
function equipeLogin(data) {
  const c = _conferirSenha(data.nome, data.senha);
  if (!c.ok) return jsonResponse({ status: 'error', code: c.code });
  // Senha inicial migrada: gestor não entra (redefinição pelo CEO); os demais só criam a senha pessoal (sem crachá)
  if (_gestorComSenhaMigrada(c.membro)) return jsonResponse({ status: 'error', code: 'redefinir_pelo_ceo' });
  if (_senhaMigrada(c.membro)) return jsonResponse({ status: 'ok', criarSenha: true });
  return jsonResponse({ status: 'ok', cracha: emitirCracha(c.membro) });
}
// Celular já logado: troca a senha pessoal guardada nele (em código) por um crachá, sem digitar de novo.
// Mesmo limite de tentativas do login.
function equipeCrachaPorHash(data) {
  const cache = CacheService.getScriptCache(), k = 'erros_' + _chaveNome(data.nome);
  const erros = Number(cache.get(k) || 0);
  if (erros >= 5) return jsonResponse({ status: 'error', code: 'bloqueado' });
  const m = _acharMembro(data.nome);
  const h = String(data.hash || '').toLowerCase();
  if (!m || !m.ativo || !/^[0-9a-f]{64}$/.test(h) || !_iguais(h, m.senhaHash)) {
    cache.put(k, String(erros + 1), 900);
    return jsonResponse({ status: 'error', code: 'senha_nao_confere' });
  }
  // Só senha criada pela pessoa ou redefinida depois da migração (a inicial migrada não vale aqui)
  if (_senhaMigrada(m)) return jsonResponse({ status: 'error', code: 'senha_migrada' });
  if (m.resetEm && String(data.desde || '') < m.resetEm) return jsonResponse({ status: 'error', code: 'refazer_login' });
  cache.remove(k);
  return jsonResponse({ status: 'ok', cracha: emitirCracha(m) });
}
// O celular logado ainda vale? Com crachá: confere e devolve um crachá novo (renova a validade).
// Sem crachá (versões antigas do app): confere por nome + data do último login.
function equipeSessao(data) {
  if (data.cracha) {
    const u = lerCracha(data.cracha);
    if (u) return jsonResponse({ status: 'ok', cracha: emitirCracha(_acharMembro(u.nome)) });
    // crachá vencido ou inválido: segue para a conferência por nome + data (sem crachá novo)
  }
  const m = _acharMembro(data.nome);
  if (!m || !m.ativo) return jsonResponse({ status: 'error', code: 'refazer_login' });
  if (m.resetEm && String(data.desde || '') < m.resetEm) return jsonResponse({ status: 'error', code: 'refazer_login' });
  return jsonResponse({ status: 'ok' });
}

function _doGet(e, user) {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const params = e && e.parameter ? e.parameter : {};
    const salaFiltro = params.sala || null;

    // ?_action=update_sala&id=...&sala=... — atualiza coluna Sala via GET
    if (params._action === 'update_sala') {
      const sheetsAlvo = abasDePesquisas();
      for (const sheet of sheetsAlvo) {
        const row = findRowById(sheet, params.id);
        if (row > 0) {
          if (!_podeMarcar(user, sheet, row)) return jsonResponse({ status: 'error', code: 'proibido', id: params.id });
          const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
          const set = (col, val) => {
            const i = cabecalho.indexOf(col);
            if (i >= 0) sheet.getRange(row, i + 1).setValue(val);
          };
          if (params.sala !== undefined) set('Sala', normalizarSalaDigitada(params.sala));
          if (params.emSala !== undefined) set('EmSala', params.emSala === 'true' ? 'SIM' : '');
          if (params.tipoSala !== undefined) set('TipoSala', params.tipoSala);
          if (params.venda !== undefined) set('Venda', params.venda === 'true' ? 'SIM' : '');
          return jsonResponse({ status: 'ok', id: params.id });
        }
      }
      return jsonResponse({ status: 'not_found', id: params.id });
    }

    // ?action=listarEquipe — lista da aba Equipe (o app guarda uma cópia no celular)
    if (params.action === 'listarEquipe') {
      if (!chaveLeadsOk(params)) return jsonResponse({ status: 'unauthorized' });
      const equipe = lerEquipe();
      return jsonResponse(equipe ? { status: 'ok', equipe } : { status: 'sem_aba', equipe: [] });
    }

    // ?_type=pi — retorna pesquisas incompletas do PropertiesService
    if (params._type === 'pi') {
      if (!chaveLeadsOk(params)) return jsonResponse({ status: 'unauthorized' });
      return jsonResponse({ status: 'ok', pis: getPIs().filter(x => _podeLer(user, _opDaPesquisa(x.sala, x.captador), x.captador)) });
    }

    // ?action=getDDP&hotel=SPTR — retorna registros DDP
    if (params.action === 'getDDP') {
      const hotel = params.hotel || '';
      const store = getDDPStore().filter(r => _podeLer(user, SALA_OPERACAO[r.hotel] || '', null));
      const rows = hotel ? store.filter(r => r.hotel === hotel) : store;
      return jsonResponse({ status: 'ok', rows });
    }

    // ?action=saveDDP&hotel=...&dateISO=...&ciPool=...  — salva via GET (POST não sobrevive ao redirect)
    if (params.action === 'saveDDP') {
      const hotel   = params.hotel   || '';
      const dateISO = (params.dateISO || '').replace(/^DDP_/, '');
      if (!hotel || !dateISO) return jsonResponse({ status: 'error', message: 'hotel e dateISO obrigatórios' });
      if (!_podeGravar(user, SALA_OPERACAO[hotel] || '', null)) return jsonResponse({ status: 'error', code: 'proibido' });
      const entry = {
        hotel, dateISO,
        ciPool:     Number(params.ciPool)     || 0,
        ciCot:      Number(params.ciCot)      || 0,
        ciConv:     Number(params.ciConv)     || 0,
        ciGrupos:   Number(params.ciGrupos)   || 0,
        salaPool:   Number(params.salaPool)   || 0,
        salaCot:    Number(params.salaCot)    || 0,
        salaConv:   Number(params.salaConv)   || 0,
        vendPool:   Number(params.vendPool)   || 0,
        vendCot:    Number(params.vendCot)    || 0,
        vendConv:   Number(params.vendConv)   || 0,
        visitantes: Number(params.visitantes) || 0,
      };
      saveDDPEntry(entry);
      return jsonResponse({ status: 'ok' });
    }

    if (!chaveLeadsOk(params)) return jsonResponse({ status: 'unauthorized' });
    // v9: cada aba uma vez só (antes a mesma aba era lida várias vezes e repetia pesquisas)
    const sheetsAlvo = salaFiltro
      ? [ss.getSheetByName(SALA_SHEETS[salaFiltro] || salaFiltro)].filter(Boolean)
      : abasDePesquisas();

    // ?dateISO=2026-04-24 filtra por dia exato; ?dateFrom=...&dateTo=... filtra por range
    const dateISOFiltro = params.dateISO || null;
    const dateFrom = params.dateFrom || null;
    const dateTo   = params.dateTo   || null;

    // Normaliza qualquer formato de data para YYYY-MM-DD
    function _normRowDate(v) {
      if (!v) return '';
      if (v instanceof Date) {
        const y = v.getFullYear();
        const m = String(v.getMonth() + 1).padStart(2, '0');
        const d = String(v.getDate()).padStart(2, '0');
        return y + '-' + m + '-' + d;
      }
      const s = String(v).trim();
      // Já está em YYYY-MM-DD
      if (/^\d{4}-\d{2}-\d{2}/.test(s)) return s.slice(0, 10);
      // DD/MM/YYYY
      const m1 = s.match(/^(\d{2})\/(\d{2})\/(\d{4})/);
      if (m1) return m1[3] + '-' + m1[2] + '-' + m1[1];
      // "Thu Apr 24 2026 ..." ou qualquer formato JS Date string
      const dt = new Date(s);
      if (!isNaN(dt)) {
        const y = dt.getFullYear();
        const mo = String(dt.getMonth() + 1).padStart(2, '0');
        const d = String(dt.getDate()).padStart(2, '0');
        return y + '-' + mo + '-' + d;
      }
      return s;
    }

    const leads = [];
    sheetsAlvo.forEach(sheet => {
      const lastRow = sheet.getLastRow();
      if (lastRow < 2) return;
      const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
      const idx = {};
      COLUNAS.forEach(col => { idx[col] = cabecalho.indexOf(col); });
      const nCols = sheet.getLastColumn();
      // v19: com filtro de data, lê de baixo para cima em blocos e para quando um bloco inteiro
      // é anterior ao período (as pesquisas entram em ordem). Antes lia a aba inteira.
      let rows;
      const limiteInf = dateISOFiltro || dateFrom;
      if (limiteInf && idx['Data'] >= 0) {
        rows = [];
        const BLOCO = 2000;
        let fim = lastRow;
        while (fim >= 2) {
          const ini = Math.max(2, fim - BLOCO + 1);
          const parte = sheet.getRange(ini, 1, fim - ini + 1, nCols).getValues();
          let temNoPeriodo = false;
          for (let k = 0; k < parte.length; k++) {
            if (!parte[k][0]) continue;
            const d = _normRowDate(parte[k][idx['Data']]);
            if (!d || d >= limiteInf) { temNoPeriodo = true; break; }
          }
          rows = parte.concat(rows);
          if (!temNoPeriodo) break;
          fim = ini - 1;
        }
      } else {
        rows = sheet.getRange(2, 1, lastRow - 1, nCols).getValues();
      }
      rows.forEach(r => {
        if (!r[0]) return;
        // Filtro por data — dia exato ou range
        // Leads sem data (rowDateISO='') são sempre incluídos para não perder registros
        if (dateISOFiltro || dateFrom || dateTo) {
          const rowDateISO = idx['Data'] >= 0 ? _normRowDate(r[idx['Data']]) : '';
          if (dateISOFiltro && rowDateISO && rowDateISO !== dateISOFiltro) return;
          if (dateFrom && rowDateISO && rowDateISO < dateFrom) return;
          if (dateTo   && rowDateISO && rowDateISO > dateTo)   return;
        }
        const get = (col) => idx[col] >= 0 ? r[idx[col]] : '';
        if (!_podeLer(user, _opDaPesquisa(get('Sala'), get('Captador')), get('Captador'))) return;
        leads.push({
          id: String(get('ID')), date: String(get('Data')), time: (function(){ var v = get('Hora'); if (!v) return ''; if (v instanceof Date) return Utilities.formatDate(v, 'America/Sao_Paulo', 'HH:mm'); return String(v); })(),
          captador: get('Captador'), sala: get('Sala'), pontoCaptacao: get('PontoCaptacao'), nome: get('Nome'),
          tel: get('Telefone'), cidade: get('Cidade'), verdict: get('Resultado'),
          tipo: get('Tipo'), renda: get('Renda'), mode: get('Modo'),
          emSala:   get('EmSala')   === 'SIM',
          tipoSala: get('TipoSala') || null,
          venda:    get('Venda')    === 'SIM',
          _ans: {
            idadeMarido:      get('IdadeTitular')      || null,
            profissaoTitular: get('ProfissaoTitular')  || null,
            idadeConjuge:     get('IdadeConjuge')      || null,
            profissaoConjuge: get('ProfissaoConjuge')  || null,
            carro:            get('Carro')             || null,
            casa:             get('Casa')              || null,
            cartao:           get('Cartao')            || null,
            viagens:          get('Viagens')           || null,
            // v2.21 — input numérico do carro (vazio para leads antigos)
            anoCarro:         (function(){ var v = get('AnoCarro'); if (v === '' || v == null) return null; var n = parseInt(v, 10); return isNaN(n) ? null : n; })(),
            // v2.23 — apresentação prévia de multipropriedade
            conheceuMultiprop: get('ConheceuMultiprop') || null,
            qtdMultiprop:      String(get('QtdMultiprop') || '') || null,
          },
        });
      });
    });

    return jsonResponse({ status: 'ok', leads });
  } catch (err) {
    return jsonResponse({ status: 'error', message: err.message });
  }
}

function migrarTodasAsAbas() {
  Object.values(SALA_SHEETS).forEach(nome => getOrCreateSheetByName(nome));
  Logger.log('Migração concluída.');
}

// ── AUDITORIA DE ABAS — apenas loga o que tem em cada aba ──
// Rodar manualmente antes de migrarAbasPorSala() para entender o cenário.
function auditarAbas() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  Logger.log('========== AUDITORIA DE ABAS ==========');
  Logger.log('Mapa oficial SALA_SHEETS:');
  Object.keys(SALA_SHEETS).forEach(sala => {
    Logger.log('  ' + sala + ' → ' + SALA_SHEETS[sala]);
  });
  Logger.log('');
  Logger.log('Abas presentes na planilha:');
  const sheets = ss.getSheets();
  sheets.forEach(s => {
    const last = s.getLastRow();
    if (last < 2) {
      Logger.log('  [' + s.getName() + '] vazia (ou só cabeçalho)');
      return;
    }
    const cabecalho = s.getRange(1, 1, 1, s.getLastColumn()).getValues()[0];
    const idxSala = cabecalho.indexOf('Sala');
    if (idxSala < 0) {
      Logger.log('  [' + s.getName() + '] sem coluna Sala — pulando');
      return;
    }
    // Conta distribuição de salas
    const nCols = s.getLastColumn();
    const rows = s.getRange(2, 1, last - 1, nCols).getValues();
    const contagem = {};
    rows.forEach(r => {
      const sala = String(r[idxSala] || '').trim() || '(vazio)';
      contagem[sala] = (contagem[sala] || 0) + 1;
    });
    Logger.log('  [' + s.getName() + '] ' + rows.length + ' linhas');
    Object.keys(contagem).sort().forEach(sala => {
      const esperado = SALA_SHEETS[sala] || '???';
      const flag = esperado === '???' ? ' ⚠️  NÃO MAPEADA' : (esperado !== s.getName() ? ' ⚠️  ABA ERRADA (esperado: ' + esperado + ')' : '');
      Logger.log('      - ' + sala + ': ' + contagem[sala] + flag);
    });
  });
  Logger.log('========== FIM AUDITORIA ==========');
}

// ── AUDITORIA DDP — diagnostica a taxa de marcação de EmSala / TipoSala / Venda ──
// Roda uma vez para entender quanto do DDP automático está faltando.
// Considera apenas abas que alimentam DDP (Alta Vista, Atrium, Marina, SPTR, Thermas SP)
function auditarDDP() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const SALAS_DDP = ['Alta Vista', 'Atrium', 'Marina', 'SPTR', 'Thermas SP'];
  // Calcula janela: últimos 30 dias
  const hoje = new Date();
  const de30 = new Date(hoje.getTime() - 30 * 24 * 60 * 60 * 1000);
  const isoDe = de30.toISOString().slice(0, 10);
  const isoAte = hoje.toISOString().slice(0, 10);

  Logger.log('========== AUDITORIA DDP ==========');
  Logger.log('Janela: ' + isoDe + ' a ' + isoAte + ' (últimos 30 dias)');
  Logger.log('');

  const totais = { total: 0, emSala: 0, semEmSala: 0, tipoPool: 0, tipoCot: 0, tipoConv: 0, semTipo: 0, venda: 0 };
  const porSala = {};

  SALAS_DDP.forEach(salaNome => {
    // O SALA_SHEETS mapeia tudo — para SPTR/Thermas SP, fica na aba 'São Pedro'
    const abaNome = salaNome === 'SPTR' || salaNome === 'Thermas SP' ? 'São Pedro' : 'Alta Vista';
    const sheet = ss.getSheetByName(abaNome);
    if (!sheet) { Logger.log('⚠️  Aba "' + abaNome + '" não encontrada'); return; }
    const last = sheet.getLastRow();
    if (last < 2) return;
    const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    const idxData = cabecalho.indexOf('Data');
    const idxSala = cabecalho.indexOf('Sala');
    const idxEmSala = cabecalho.indexOf('EmSala');
    const idxTipoSala = cabecalho.indexOf('TipoSala');
    const idxVenda = cabecalho.indexOf('Venda');
    if (idxData < 0 || idxSala < 0) return;
    const nCols = sheet.getLastColumn();
    const rows = sheet.getRange(2, 1, last - 1, nCols).getValues();

    const s = { total: 0, emSala: 0, semEmSala: 0, tipoPool: 0, tipoCot: 0, tipoConv: 0, semTipo: 0, venda: 0 };
    rows.forEach(r => {
      // Filtra por sala (considera nomes legados também)
      const sala = String(r[idxSala] || '').trim();
      const salaNormalizada = (sala === 'São Pedro Resort') ? 'SPTR' : sala;
      if (salaNormalizada !== salaNome) return;
      // Filtra por data
      const dataStr = String(r[idxData] || '').trim();
      const isoMatch = dataStr.match(/^(\d{2})\/(\d{2})\/(\d{4})$/);
      if (!isoMatch) return;
      const dateISO = isoMatch[3] + '-' + isoMatch[2] + '-' + isoMatch[1];
      if (dateISO < isoDe || dateISO > isoAte) return;
      s.total++;
      const emSala = String(r[idxEmSala] || '').trim().toUpperCase() === 'SIM';
      if (emSala) s.emSala++; else s.semEmSala++;
      const tipoSala = String(r[idxTipoSala] || '').trim().toLowerCase();
      if (tipoSala === 'pool') s.tipoPool++;
      else if (tipoSala === 'cotista') s.tipoCot++;
      else if (tipoSala === 'conv' || tipoSala === 'convidado') s.tipoConv++;
      else s.semTipo++;
      const venda = String(r[idxVenda] || '').trim().toUpperCase() === 'SIM';
      if (venda) s.venda++;
    });
    porSala[salaNome] = s;
    totais.total      += s.total;
    totais.emSala     += s.emSala;
    totais.semEmSala  += s.semEmSala;
    totais.tipoPool   += s.tipoPool;
    totais.tipoCot    += s.tipoCot;
    totais.tipoConv   += s.tipoConv;
    totais.semTipo    += s.semTipo;
    totais.venda      += s.venda;
  });

  Logger.log('--- Por sala (últimos 30 dias) ---');
  Object.keys(porSala).forEach(nome => {
    const s = porSala[nome];
    const taxaEmSala  = s.total ? Math.round(s.emSala / s.total * 100) : 0;
    const taxaTipo    = s.emSala ? Math.round((s.tipoPool + s.tipoCot + s.tipoConv) / s.emSala * 100) : 0;
    Logger.log('  [' + nome + '] ' + s.total + ' leads no período');
    Logger.log('      EmSala: ' + s.emSala + ' / ' + s.total + ' (' + taxaEmSala + '%)');
    Logger.log('      Sem EmSala: ' + s.semEmSala + ' (leads que entraram mas não foram marcados como em sala)');
    Logger.log('      TipoSala (só dos marcados em sala): Pool=' + s.tipoPool + ' Cot=' + s.tipoCot + ' Conv=' + s.tipoConv + ' Sem=' + s.semTipo + ' (' + taxaTipo + '% preenchido)');
    Logger.log('      Vendas: ' + s.venda);
  });
  Logger.log('');
  Logger.log('--- TOTAL GERAL ---');
  const taxaEmSalaTotal = totais.total ? Math.round(totais.emSala / totais.total * 100) : 0;
  const taxaTipoTotal   = totais.emSala ? Math.round((totais.tipoPool + totais.tipoCot + totais.tipoConv) / totais.emSala * 100) : 0;
  Logger.log('  Leads no período: ' + totais.total);
  Logger.log('  Com EmSala: ' + totais.emSala + ' (' + taxaEmSalaTotal + '%)');
  Logger.log('  Sem EmSala (gargalo principal): ' + totais.semEmSala);
  Logger.log('  Com TipoSala (dos marcados): ' + (totais.tipoPool + totais.tipoCot + totais.tipoConv) + ' / ' + totais.emSala + ' (' + taxaTipoTotal + '%)');
  Logger.log('  Vendas marcadas: ' + totais.venda);
  Logger.log('');
  Logger.log('--- DIAGNÓSTICO ---');
  if (totais.semEmSala / Math.max(totais.total, 1) > 0.5) {
    Logger.log('  ❌ Mais de 50% dos leads sem EmSala — os captadores não estão marcando (ou o app não está salvando).');
    Logger.log('  Próximo passo: investigar UX do botão "Entrou em sala" no app.');
  } else if (totais.semEmSala / Math.max(totais.total, 1) > 0.2) {
    Logger.log('  ⚠️  20-50% dos leads sem EmSala — problema de adesão parcial.');
    Logger.log('  Próximo passo: backfill dos vazios + melhorias de UX.');
  } else {
    Logger.log('  ✓ Maioria dos leads marcados. Backfill cobre só os antigos.');
  }
  if (totais.emSala > 0 && (totais.tipoPool + totais.tipoCot + totais.tipoConv) / totais.emSala < 0.5) {
    Logger.log('  ❌ Entre os marcados, menos de 50% têm TipoSala.');
  }
  Logger.log('========== FIM AUDITORIA DDP ==========');
}

// ── BACKFILL DDP — seta EmSala=SIM e TipoSala=pool nos leads sem flag ──
// Rodar SOMENTE após auditarDDP() confirmar que vale a pena.
// Afeta apenas os últimos 30 dias. Não toca em leads já marcados.
function backfillEmSala() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const SALAS_DDP = ['Alta Vista', 'Atrium', 'Marina', 'SPTR', 'Thermas SP'];
  const hoje = new Date();
  const de30 = new Date(hoje.getTime() - 30 * 24 * 60 * 60 * 1000);
  const isoDe = de30.toISOString().slice(0, 10);
  const isoAte = hoje.toISOString().slice(0, 10);

  Logger.log('========== BACKFILL DDP ==========');
  Logger.log('Janela: ' + isoDe + ' a ' + isoAte);
  Logger.log('Vai setar EmSala=SIM e TipoSala=pool em leads sem flag.');
  Logger.log('Leads já marcados (EmSala=SIM ou TipoSala preenchido) NÃO serão tocados.');
  Logger.log('');

  let totalAlterados = 0;
  const porSala = {};

  SALAS_DDP.forEach(salaNome => {
    const abaNome = salaNome === 'SPTR' || salaNome === 'Thermas SP' ? 'São Pedro' : 'Alta Vista';
    const sheet = ss.getSheetByName(abaNome);
    if (!sheet) return;
    const last = sheet.getLastRow();
    if (last < 2) return;
    const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    const idxData = cabecalho.indexOf('Data');
    const idxSala = cabecalho.indexOf('Sala');
    const idxEmSala = cabecalho.indexOf('EmSala');
    const idxTipoSala = cabecalho.indexOf('TipoSala');
    if (idxData < 0 || idxSala < 0) return;
    const nCols = sheet.getLastColumn();
    const rows = sheet.getRange(2, 1, last - 1, nCols).getValues();
    const alteracoes = [];
    rows.forEach((r, i) => {
      const sala = String(r[idxSala] || '').trim();
      const salaNormalizada = (sala === 'São Pedro Resort') ? 'SPTR' : sala;
      if (salaNormalizada !== salaNome) return;
      const dataStr = String(r[idxData] || '').trim();
      const isoMatch = dataStr.match(/^(\d{2})\/(\d{2})\/(\d{4})$/);
      if (!isoMatch) return;
      const dateISO = isoMatch[3] + '-' + isoMatch[2] + '-' + isoMatch[1];
      if (dateISO < isoDe || dateISO > isoAte) return;
      const emSalaAtual = String(r[idxEmSala] || '').trim().toUpperCase();
      const tipoSalaAtual = String(r[idxTipoSala] || '').trim();
      // Só altera se AMBOS estiverem vazios (não toca em lead parcialmente marcado)
      if (!emSalaAtual && !tipoSalaAtual) {
        alteracoes.push({ rowIdx: i + 2, captador: r[3], sala, dateISO });
      }
    });
    if (alteracoes.length === 0) {
      Logger.log('  [' + salaNome + '] nenhum lead precisou de backfill');
      return;
    }
    // Aplica as alterações: seta EmSala=SIM e TipoSala=pool nas células
    alteracoes.forEach(a => {
      if (idxEmSala >= 0) sheet.getRange(a.rowIdx, idxEmSala + 1).setValue('SIM');
      if (idxTipoSala >= 0) sheet.getRange(a.rowIdx, idxTipoSala + 1).setValue('pool');
    });
    totalAlterados += alteracoes.length;
    porSala[salaNome] = alteracoes.length;
    Logger.log('  [' + salaNome + '] ' + alteracoes.length + ' leads alterados');
  });

  Logger.log('');
  Logger.log('Total de leads alterados: ' + totalAlterados);
  Logger.log('Distribuição:');
  Object.keys(porSala).forEach(nome => {
    Logger.log('  [' + nome + '] ' + porSala[nome] + ' leads');
  });
  Logger.log('========== FIM BACKFILL ==========');
}

// ── MIGRAÇÃO DE ABAS POR SALA — separa leads mal-posicionados ──
// 1. Identifica a aba oficial de cada operação (a com MAIS LINHAS VÁLIDAS
//    — desconsidera vazias e salas não-mapeadas)
// 2. Move linhas mal-posicionadas para a aba oficial correta baseado na coluna 'Sala'
// 3. Renomeia abas duplicadas (ex: 2x "São Pedro") — a perdedora vira "Nome (legada — vazia)"
// Rodar manualmente UMA VEZ após auditarAbas() confirmar o cenário.
function migrarAbasPorSala() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheets = ss.getSheets();

  // 1. Identifica a aba oficial de cada operação (a com MAIS LINHAS VÁLIDAS)
  //    Linhas válidas = com sala que existe no SALA_SHEETS
  // Considera só abas cujo nome bate com algum dos valores únicos de SALA_SHEETS
  const valoresUnicos = [...new Set(Object.values(SALA_SHEETS))]; // ['Alta Vista', 'São Pedro', 'Atibaia']
  const contagemPorNome = {};
  valoresUnicos.forEach(nome => { contagemPorNome[nome] = []; });
  sheets.forEach(s => {
    if (contagemPorNome.hasOwnProperty(s.getName())) {
      contagemPorNome[s.getName()].push(s);
    }
  });

  // Helper: conta linhas válidas (sala existe no SALA_SHEETS) de uma sheet
  function _countValidas(sheet) {
    const last = sheet.getLastRow();
    if (last < 2) return 0;
    const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    const idxSala = cabecalho.indexOf('Sala');
    if (idxSala < 0) return 0;
    const nCols = sheet.getLastColumn();
    const rows = sheet.getRange(2, 1, last - 1, nCols).getValues();
    let count = 0;
    rows.forEach(r => {
      const sala = String(r[idxSala] || '').trim();
      if (SALA_SHEETS[sala]) count++;
    });
    return count;
  }

  const abaOficial = {}; // operacao (nome único) → Sheet oficial
  Object.keys(contagemPorNome).forEach(nome => {
    const candidatos = contagemPorNome[nome];
    if (candidatos.length === 0) {
      Logger.log('⚠️  Aba oficial "' + nome + '" não existe — criando vazia');
      abaOficial[nome] = getOrCreateSheetByName(nome);
    } else if (candidatos.length === 1) {
      abaOficial[nome] = candidatos[0];
      Logger.log('✓ Aba oficial "' + nome + '": única existente (' + _countValidas(candidatos[0]) + ' linhas válidas de ' + candidatos[0].getLastRow() + ')');
    } else {
      // Mais de uma — escolhe a com mais linhas VÁLIDAS; renomeia as outras
      candidatos.forEach(c => { c._linhasValidas = _countValidas(c); });
      candidatos.sort((a, b) => b._linhasValidas - a._linhasValidas);
      abaOficial[nome] = candidatos[0];
      Logger.log('✓ Aba oficial "' + nome + '": escolhida com ' + candidatos[0]._linhasValidas + ' linhas válidas (de ' + candidatos[0].getLastRow() + ' totais)');
      for (let i = 1; i < candidatos.length; i++) {
        const duplicada = candidatos[i];
        const novoNome = nome + ' (legada — vazia)';
        try {
          duplicada.setName(novoNome);
          Logger.log('  ↻ Aba duplicada renomeada para "' + novoNome + '" (' + duplicada._linhasValidas + ' válidas de ' + duplicada.getLastRow() + ')');
        } catch (e) {
          Logger.log('  ⚠️  Não consegui renomear aba (já existe "' + novoNome + '"?): ' + e.message);
        }
      }
    }
  });

  // 2. Para cada aba (oficial ou não), move linhas para a aba oficial correta
  let totalMovidas = 0;
  const logMovidas = [];
  sheets.forEach(s => {
    const nomeAba = s.getName();
    if (nomeAba.endsWith('(legada — vazia)')) {
      Logger.log('⏭  Pulando aba legada: ' + nomeAba);
      return;
    }
    if (!valoresUnicos.includes(nomeAba)) {
      Logger.log('⏭  Pulando aba fora do escopo: ' + nomeAba);
      return;
    }
    const last = s.getLastRow();
    if (last < 2) return;
    const cabecalho = s.getRange(1, 1, 1, s.getLastColumn()).getValues()[0];
    const idxSala = cabecalho.indexOf('Sala');
    if (idxSala < 0) return;
    const nCols = s.getLastColumn();
    const dados = s.getRange(2, 1, last - 1, nCols).getValues();

    // Coleta linhas que precisam sair desta aba
    const linhasParaMover = []; // { rowIdx (1-based na planilha), sala, destinoAba, valores }
    dados.forEach((r, i) => {
      const sala = String(r[idxSala] || '').trim();
      if (!sala) return;
      const destino = SALA_SHEETS[sala];
      if (!destino) {
        Logger.log('  ⚠️  Linha ' + (i + 2) + ' tem sala "' + sala + '" não mapeada — IGNORANDO');
        return;
      }
      if (destino !== nomeAba) {
        linhasParaMover.push({ rowIdx: i + 2, sala, destinoAba: destino, valores: r });
      }
    });

    if (linhasParaMover.length === 0) {
      Logger.log('✓ Aba "' + nomeAba + '": todas as ' + dados.length + ' linhas estão no lugar certo');
      return;
    }

    Logger.log('⤴  Aba "' + nomeAba + '": ' + linhasParaMover.length + ' linhas para mover');
    linhasParaMover.forEach(m => {
      logMovidas.push({ de: nomeAba, para: m.destinoAba, sala: m.sala, rowIdx: m.rowIdx });
    });
  });

  // 3. Executa as movimentações de baixo para cima em cada aba de origem
  // Agrupa por aba de origem
  const porOrigem = {};
  logMovidas.forEach(m => {
    if (!porOrigem[m.de]) porOrigem[m.de] = [];
    porOrigem[m.de].push(m);
  });
  Object.keys(porOrigem).forEach(nomeOrigem => {
    const sheetOrigem = ss.getSheetByName(nomeOrigem);
    if (!sheetOrigem) return;
    const cabecalho = sheetOrigem.getRange(1, 1, 1, sheetOrigem.getLastColumn()).getValues()[0];
    const idxSala = cabecalho.indexOf('Sala');
    const nCols = sheetOrigem.getLastColumn();
    // Agrupa por destino
    const porDestino = {};
    porOrigem[nomeOrigem].forEach(m => {
      if (!porDestino[m.destinoAba]) porDestino[m.destinoAba] = [];
      porDestino[m.destinoAba].push(m);
    });
    Object.keys(porDestino).forEach(nomeDestino => {
      const sheetDestino = abaOficial[nomeDestino];
      if (!sheetDestino) return;
      // Coleta os valores de cada linha (de baixo para cima para não deslocar índice ao deletar)
      const rowsParaInserir = [];
      const rowIdxs = porDestino[nomeDestino].map(m => m.rowIdx).sort((a, b) => b - a);
      rowIdxs.forEach(rowIdx => {
        const valores = sheetOrigem.getRange(rowIdx, 1, 1, nCols).getValues()[0];
        rowsParaInserir.unshift(valores);
      });
      // Insere no destino (abaixo do cabeçalho)
      sheetDestino.getRange(sheetDestino.getLastRow() + 1, 1, rowsParaInserir.length, nCols).setValues(rowsParaInserir);
      // Reaplica cor baseada em verdict
      rowsParaInserir.forEach((r, i) => {
        const row = sheetDestino.getLastRow() - rowsParaInserir.length + i + 1;
        const verdict = r[cabecalho.indexOf('Resultado')];
        if (typeof verdict === 'string') {
          const range = sheetDestino.getRange(row, 1, 1, nCols);
          if (verdict === 'Q')       range.setBackground('#d4edda');
          else if (verdict === 'PARCIAL') range.setBackground('#fff3cd');
          else if (verdict === 'NQ') range.setBackground('#f8d7da');
        }
      });
      // Remove da origem (de baixo para cima)
      rowIdxs.forEach(rowIdx => {
        sheetOrigem.deleteRow(rowIdx);
      });
      totalMovidas += rowIdxs.length;
      Logger.log('  ⤴  ' + rowIdxs.length + ' linhas: "' + nomeOrigem + '" → "' + nomeDestino + '"');
    });
  });

  Logger.log('========== MIGRAÇÃO CONCLUÍDA ==========');
  Logger.log('Total de linhas movidas: ' + totalMovidas);
  Logger.log('Abas oficiais finais:');
  Object.keys(abaOficial).forEach(nome => {
    Logger.log('  [' + nome + '] ' + abaOficial[nome].getLastRow() + ' linhas');
  });
}

// ── Remove linhas duplicadas por ID, mantendo apenas a ÚLTIMA ocorrência de cada ID ──
// Rodar manualmente no Apps Script quando houver duplicatas
function dedupTodasAsAbas() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const nomeAbas = [...new Set(Object.values(SALA_SHEETS))]; // ['Alta Vista', 'São Pedro', 'Atibaia']
  let totalRemovidas = 0;

  nomeAbas.forEach(nome => {
    const sheet = ss.getSheetByName(nome);
    if (!sheet) { Logger.log(nome + ': aba não encontrada, pulando.'); return; }
    const lastRow = sheet.getLastRow();
    if (lastRow < 2) { Logger.log(nome + ': vazia, pulando.'); return; }

    const nCols = sheet.getLastColumn();
    const allRows = sheet.getRange(2, 1, lastRow - 1, nCols).getValues();

    // Mapeia id → índice da última ocorrência (mais recente = maior índice no array)
    const lastIdx = {};
    allRows.forEach((r, i) => {
      const id = String(r[0]).trim();
      if (id) lastIdx[id] = i;
    });

    // Marca linhas a remover (qualquer linha que NÃO é a última ocorrência do seu ID)
    const linhasParaRemover = []; // índices no array (base 0)
    allRows.forEach((r, i) => {
      const id = String(r[0]).trim();
      if (id && lastIdx[id] !== i) linhasParaRemover.push(i);
    });

    if (!linhasParaRemover.length) {
      Logger.log(nome + ': sem duplicatas.');
      return;
    }

    // Remove de baixo para cima para não deslocar índices
    linhasParaRemover.sort((a, b) => b - a);
    linhasParaRemover.forEach(i => {
      sheet.deleteRow(i + 2); // +2 porque array é base-0 e linha 1 é cabeçalho
    });

    totalRemovidas += linhasParaRemover.length;
    Logger.log(nome + ': ' + linhasParaRemover.length + ' duplicatas removidas. Restam ' + (allRows.length - linhasParaRemover.length) + ' linhas.');
  });

  Logger.log('=== Dedup concluído. Total removidas: ' + totalRemovidas + ' linhas. ===');
}

// ── Remove linhas duplicadas por TELEFONE + DATA + CAPTADOR, mantendo a MAIS COMPLETA ──
// Resolve o problema do auto-save parcial antigo que gerava múltiplos leads
// com IDs diferentes mas mesmo tel + data + captador.
// Critério de "mais completa": a que tiver MAIS campos _ans preenchidos.
// Rodar manualmente no Apps Script uma vez para limpar as duplicações antigas.
function dedupPorTelEData() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const nomeAbas = [...new Set(Object.values(SALA_SHEETS))]; // ['Alta Vista', 'São Pedro', 'Atibaia']
  let totalRemovidas = 0;
  let totalMantidas = 0;

  nomeAbas.forEach(nome => {
    const sheet = ss.getSheetByName(nome);
    if (!sheet) { Logger.log(nome + ': aba não encontrada, pulando.'); return; }
    const lastRow = sheet.getLastRow();
    if (lastRow < 2) { Logger.log(nome + ': vazia, pulando.'); return; }

    const nCols = sheet.getLastColumn();
    const cabecalho = sheet.getRange(1, 1, 1, nCols).getValues()[0];
    const idxTel      = cabecalho.indexOf('Telefone');
    const idxData     = cabecalho.indexOf('Data');
    const idxCaptador = cabecalho.indexOf('Captador');
    const idxResultado= cabecalho.indexOf('Resultado');
    if (idxTel < 0 || idxData < 0) { Logger.log(nome + ': colunas Tel/Data não encontradas.'); return; }

    // Lê todas as linhas
    const allRows = sheet.getRange(2, 1, lastRow - 1, nCols).getValues();

    // Agrupa por chave (tel + data + captador)
    const grupos = {}; // chave → [{idx, score, row}]
    allRows.forEach((r, i) => {
      const tel = String(r[idxTel] || '').replace(/\D/g, '').slice(0, 11);
      const data = String(r[idxData] || '').trim();
      const captador = String(r[idxCaptador] || '').trim();
      if (!tel || !data) return; // sem tel ou data — não deduplica
      const key = tel + '|' + data + '|' + captador;
      // Score = quantos campos do _ans estão preenchidos (Q tem campos extras)
      // Para simplificar, conta quantas colunas não-vazias na linha
      let score = 0;
      for (let c = 0; c < nCols; c++) {
        if (c === idxTel) continue; // tel já conta na chave
        const v = r[c];
        if (v !== '' && v !== null && v !== undefined) score++;
      }
      // Bonus para resultado definido (Q > PARCIAL > NQ)
      if (idxResultado >= 0) {
        const v = String(r[idxResultado] || '').toUpperCase();
        if (v === 'Q') score += 10;
        else if (v === 'PARCIAL') score += 5;
      }
      if (!grupos[key]) grupos[key] = [];
      grupos[key].push({ idx: i, score, row: r });
    });

    // Para cada grupo, mantém só a linha de MAIOR score
    const linhasParaRemover = []; // índices no array (base 0)
    Object.keys(grupos).forEach(key => {
      const grupo = grupos[key];
      if (grupo.length <= 1) return; // sem duplicata
      // Ordena por score desc — primeira é a vencedora
      grupo.sort((a, b) => b.score - a.score);
      // Marca as outras para remoção
      for (let i = 1; i < grupo.length; i++) {
        linhasParaRemover.push(grupo[i].idx);
      }
      const vencedor = grupo[0];
      Logger.log('  ' + nome + ' | ' + key + ' → ' + grupo.length + ' cópias, mantendo idx=' + vencedor.idx + ' (score=' + vencedor.score + ')');
    });

    if (!linhasParaRemover.length) {
      Logger.log(nome + ': sem duplicatas por tel+data+captador.');
      return;
    }

    // Remove de baixo para cima para não deslocar índices
    linhasParaRemover.sort((a, b) => b - a);
    linhasParaRemover.forEach(i => {
      sheet.deleteRow(i + 2);
    });

    totalRemovidas += linhasParaRemover.length;
    totalMantidas += (allRows.length - linhasParaRemover.length);
    Logger.log(nome + ': ' + linhasParaRemover.length + ' duplicatas removidas. Restam ' + (allRows.length - linhasParaRemover.length) + ' linhas.');
  });

  Logger.log('=== Dedup por tel+data+captador concluído ===');
  Logger.log('Total removidas: ' + totalRemovidas);
  Logger.log('Total mantidas: ' + totalMantidas);
}

// ── Migra dados do PropertiesService (ddp_store antigo) para a aba DDP ──
// Rodar UMA VEZ manualmente no Apps Script após implantar nova versão
function migrarDDPParaSheets() {
  const raw = PropertiesService.getScriptProperties().getProperty('ddp_store');
  if (!raw) { Logger.log('Nada para migrar — ddp_store vazio.'); return; }
  const store = JSON.parse(raw);
  if (!store || !store.length) { Logger.log('Nada para migrar — lista vazia.'); return; }
  getDDPSheet(); // garante que a aba existe
  let migrados = 0;
  store.forEach(entry => {
    if (!entry.hotel || !entry.dateISO) return;
    let d = entry.dateISO;
    if (d.startsWith('DDP_')) d = d.slice(4);
    saveDDPEntry(Object.assign({}, entry, { dateISO: d }));
    migrados++;
  });
  Logger.log('Migração DDP concluída: ' + migrados + ' registros transferidos para a aba DDP.');
  // NÃO apaga o ddp_store antigo automaticamente — confirme visualmente na planilha antes de limpar
}

// Reaplica cores em todas as linhas existentes (rodar uma vez manualmente)
function reaplCarCores() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  Object.values(SALA_SHEETS).forEach(nome => {
    const sheet = ss.getSheetByName(nome);
    if (!sheet) return;
    const lastRow = sheet.getLastRow();
    if (lastRow < 2) return;
    const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    const resultIdx = cabecalho.indexOf('Resultado');
    if (resultIdx < 0) return;
    const nCols = sheet.getLastColumn();
    const rows = sheet.getRange(2, 1, lastRow - 1, nCols).getValues();
    rows.forEach((r, i) => {
      const verdict = r[resultIdx];
      const range = sheet.getRange(i + 2, 1, 1, nCols);
      if (verdict === 'Q')       range.setBackground('#d4edda');
      else if (verdict === 'PARCIAL') range.setBackground('#fff3cd');
      else if (verdict === 'NQ') range.setBackground('#f8d7da');
    });
    Logger.log(`${nome}: ${lastRow - 1} linhas processadas`);
  });
}

// ── CORREÇÃO ÚNICA (21/10/2026): Sala "São Pedro" → "SPTR" ──
// Antes, faça backup da planilha. 1º rode simularCorrecaoSalaSaoPedro (só conta, não altera).
// Depois rode corrigirSalaSaoPedroParaSPTR. Altera só as colunas Sala e PontoCaptacao
// quando o valor é "São Pedro" (com ou sem acento/maiúscula/espaço). Pode rodar de novo sem risco.
function simularCorrecaoSalaSaoPedro() { return _corrigirSalaSaoPedro(false); }
function corrigirSalaSaoPedroParaSPTR() { return _corrigirSalaSaoPedro(true); }
function _corrigirSalaSaoPedro(aplicar) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const resumo = {};
  ss.getSheets().forEach(sheet => {
    const last = sheet.getLastRow();
    if (last < 2) return;
    const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    ['Sala', 'PontoCaptacao'].forEach(col => {
      const i = cabecalho.indexOf(col);
      if (i < 0) return;
      const range = sheet.getRange(2, i + 1, last - 1, 1);
      const vals = range.getValues();
      let n = 0;
      vals.forEach(v => {
        if (v[0] !== '' && v[0] !== 'SPTR' && normalizarSalaDigitada(v[0]) === 'SPTR') { v[0] = 'SPTR'; n++; }
      });
      if (n > 0) {
        resumo[sheet.getName() + ' / ' + col] = n;
        if (aplicar) range.setValues(vals);
      }
    });
  });
  Logger.log((aplicar ? 'CORRIGIDO' : 'SIMULAÇÃO (nada alterado)') + ': ' + JSON.stringify(resumo));
  return resumo;
}

// ── CORREÇÃO ÚNICA (v20): Sala "São Pedro Resort" / "SP Resort" → "SPTR" ──
// Antes, faça backup da planilha. 1º rode simularSaoPedroResortParaSPTR (só conta, não altera).
// Depois rode corrigirSaoPedroResortParaSPTR. Altera só as colunas Sala e PontoCaptacao das abas
// de pesquisas (não mexe no DDP nem na Equipe). Aceita acento/maiúscula/espaço. Pode rodar de novo sem risco.
function simularSaoPedroResortParaSPTR() { return _corrigirSaoPedroResort(false); }
function corrigirSaoPedroResortParaSPTR() { return _corrigirSaoPedroResort(true); }
function _corrigirSaoPedroResort(aplicar) {
  const alvos = ['sao pedro resort', 'sp resort'];
  const chave = v => String(v).normalize('NFD').replace(/[\u0300-\u036f]/g, '').toLowerCase().replace(/\s+/g, ' ').trim();
  const resumo = {};
  abasDePesquisas().forEach(sheet => {
    const last = sheet.getLastRow();
    if (last < 2) return;
    const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    ['Sala', 'PontoCaptacao'].forEach(col => {
      const i = cabecalho.indexOf(col);
      if (i < 0) return;
      const range = sheet.getRange(2, i + 1, last - 1, 1);
      const vals = range.getValues();
      let n = 0;
      vals.forEach(v => { if (v[0] !== '' && alvos.indexOf(chave(v[0])) >= 0) { v[0] = 'SPTR'; n++; } });
      if (n > 0) {
        resumo[sheet.getName() + ' / ' + col] = n;
        if (aplicar) range.setValues(vals);
      }
    });
  });
  Logger.log((aplicar ? 'CORRIGIDO' : 'SIMULAÇÃO (nada alterado)') + ': ' + JSON.stringify(resumo));
  return resumo;
}

function jsonResponse(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}

// ── AUDITORIA DE CIDADES THERMAS SP — % de São Paulo Capital + top 10 ──
// Lê TODA a aba "São Pedro" (que contém leads de Thermas SP + SPTR + Externo SP),
// filtra só Thermas SP (cobre nomes legados), e calcula:
//   - Total de leads de Thermas SP no histórico
//   - % que vem de "São Paulo" capital exato
//   - Top 10 cidades (em contagem absoluta + % relativo)
function auditarCidadesThermas() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName('São Pedro');
  if (!sheet) { Logger.log('❌ Aba "São Pedro" não encontrada'); return; }
  const last = sheet.getLastRow();
  if (last < 2) { Logger.log('❌ Aba "São Pedro" vazia'); return; }
  const cabecalho = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  const idxSala = cabecalho.indexOf('Sala');
  const idxCidade = cabecalho.indexOf('Cidade');
  if (idxSala < 0 || idxCidade < 0) {
    Logger.log('❌ Coluna "Sala" ou "Cidade" não encontrada');
    return;
  }
  const nCols = sheet.getLastColumn();
  const rows = sheet.getRange(2, 1, last - 1, nCols).getValues();

  // Nomes que devem cair em "Thermas SP" (cobre legado)
  const SALAS_THERMAS = new Set(['Thermas SP', 'Thermas São Pedro', 'THERMAS SÃO PEDRO', 'THERMAS SÃO PEDRO - HOTEL', 'Thermas - UH AP']);

  let total = 0;
  let spCapital = 0;
  const cidades = {}; // contagem de cidades

  rows.forEach(r => {
    const sala = String(r[idxSala] || '').trim();
    if (!SALAS_THERMAS.has(sala)) return;
    total++;
    const cidade = String(r[idxCidade] || '').trim() || '(vazio)';
    cidades[cidade] = (cidades[cidade] || 0) + 1;
    if (cidade.toLowerCase() === 'são paulo') spCapital++;
  });

  Logger.log('========== AUDITORIA THERMAS SP × CIDADE ==========');
  Logger.log('Total de leads do Thermas SP (todo o histórico): ' + total);
  Logger.log('Leads de São Paulo capital: ' + spCapital);
  const pct = total ? Math.round(spCapital / total * 1000) / 10 : 0;
  Logger.log('% de São Paulo capital: ' + pct + '%');
  Logger.log('');
  Logger.log('Top 15 cidades (com % do total):');
  const top = Object.entries(cidades).sort((a, b) => b[1] - a[1]).slice(0, 15);
  top.forEach(([cid, qtd]) => {
    const pctCid = total ? Math.round(qtd / total * 1000) / 10 : 0;
    const flag = cid.toLowerCase() === 'são paulo' ? ' ← SP CAPITAL' : '';
    Logger.log('  ' + cid.padEnd(30) + ' ' + String(qtd).padStart(6) + '  (' + pctCid + '%)' + flag);
  });
  Logger.log('========== FIM AUDITORIA ==========');
}

// Diagnóstico (só leitura): mede o tempo de cada parte da leitura de pesquisas do mês
function medirLeitura() {
  const mede = (rotulo, fn) => { const a = Date.now(); const r = fn(); Logger.log(rotulo + ': ' + (Date.now() - a) + ' ms'); return r; };
  const ceo = { nome: 'Erick Faleiro', cargo: 'ceo', operacao: '' };
  const r = mede('CEO mês inteiro (tudo)', () => _doGet({ parameter: { dateFrom: '2026-10-01', dateTo: '2026-10-24' } }, ceo));
  const txt = mede('gerar texto', () => r.getContent());
  Logger.log('tamanho: ' + Math.round(txt.length / 1024) + ' KB, pesquisas: ' + JSON.parse(txt).leads.length);
  mede('Líder SP mês', () => _doGet({ parameter: { dateFrom: '2026-10-01', dateTo: '2026-10-24' } }, { nome: 'Camila Cristina Pinto', cargo: 'lider', operacao: 'São Pedro' }));
  const sh = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('São Pedro'); const n = sh.getLastRow();
  mede('SP ID+Data (coluna inteira)', () => sh.getRange(2, 1, n - 1, 2).getValues());
  mede('SP 3.400 linhas x todas colunas', () => sh.getRange(n - 3400, 1, 3400, sh.getLastColumn()).getValues());
  mede('ler Equipe', () => _acharMembro('Erick Faleiro'));
}
