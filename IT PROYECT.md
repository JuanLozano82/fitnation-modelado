/**
 * FitNation — Script maestro de generación de documentos
 * Genera: index.html (GitHub Pages) + presentación.pptx + modelado.docx
 * Versión: 1.2 — Junio 2026
 * 
 * USO: node build_fitnation.js
 * Para agregar ideas: edita la sección DATA al final del archivo
 */

const fs   = require('fs');
const path = require('path');

// ─── COLORES GLOBALES ───────────────────────────────────────────
const C = {
  brand:      '#1A3567',
  brandMid:   '#2B4FA0',
  brandLight: '#E8EEF8',
  teal:       '#0F6B5C',
  tealLight:  '#E0F4F0',
  amber:      '#B45309',
  amberLight: '#FEF3C7',
  red:        '#991B1B',
  redLight:   '#FEE2E2',
  green:      '#166534',
  greenLight: '#DCFCE7',
  gray:       '#374151',
  grayMid:    '#6B7280',
  grayLight:  '#F3F4F6',
  white:      '#ffffff',
  text:       '#111827',
  text2:      '#4B5563',
};

// ─── DATOS CENTRALES ─────────────────────────────────────────────
// ✏️  PARA AGREGAR IDEAS: edita este objeto y vuelve a ejecutar el script
const DATA = {
  version: '1.2',
  fecha: 'Junio 2026',
  empresa: 'Fit Concept Store, C.A.',
  proyecto: 'FitNation — Modelado de Procesos',

  actores: [
    { nombre: 'ODOO (Lixie)', rol: 'Backoffice · Núcleo del negocio', color: C.brand,
      items: ['Página web y e-commerce','POS y caja en sede','Pasarelas de pago (C2P, tarjeta)','Facturación y marco fiscal venezolano','Contabilidad — asientos automáticos','Suscripciones y ciclo de cobro','Módulo de mercadeo y promociones','Preventa con plan congelado'],
      regla: 'Dueño exclusivo de toda transacción económica. NO toca acceso físico.', sistema: 'Odoo v18 · ISO 27001' },
    { nombre: 'EVO / W12', rol: 'Gestión operativa del gimnasio', color: C.teal,
      items: ['Control de acceso — torniquetes','Biométrico facial — kiosco en sede','App del socio — reserva de clases','Seguimiento deportivo','Envío de contratos y credenciales','Membresías (solo recibe estado PAGADO)','Multisede — opera en Venezuela'],
      regla: 'Dueño exclusivo de todo acceso físico. NO toca cobros.', sistema: 'API REST entrante · Webhook saliente' },
    { nombre: 'MegaSoft', rol: 'Pasarela complementaria', color: C.amber,
      items: ['Cashea — método digital venezolano','Pago Móvil C2P (complementario)','Métodos no nativos de Odoo','Confirmación de cobro a Odoo'],
      regla: 'Extiende métodos de pago. No gestiona membresías ni acceso.', sistema: 'Pendiente confirmar alcance contractual' },
  ],

  planes: [
    { nombre: 'Mensual',    dias: 30  },
    { nombre: 'Trimestral', dias: 90  },
    { nombre: 'Semestral',  dias: 180 },
    { nombre: 'Anual',      dias: 365 },
    { nombre: 'Por sesión', dias: 1   },
  ],

  formasPago: [
    { nombre: 'Pago Móvil C2P',     online: true,  sede: true,  procesador: 'Odoo / MegaSoft' },
    { nombre: 'Tarjeta déb/créd',    online: true,  sede: true,  procesador: 'Odoo' },
    { nombre: 'Cashea',             online: true,  sede: true,  procesador: 'MegaSoft' },
    { nombre: 'Transferencia',       online: true,  sede: false, procesador: 'Odoo (manual)' },
    { nombre: 'Efectivo',            online: false, sede: true,  procesador: 'POS Odoo' },
  ],

  flujoInscripcion: [
    { num:'01', titulo:'Descubrimiento web',     sistema:'Odoo web',        desc:'Usuario visita la web, navega planes y hace clic en "Inscribirme".' },
    { num:'02', titulo:'Registro y duplicados',  sistema:'Odoo + EVO',      desc:'Odoo solicita datos y consulta silenciosamente a EVO para evitar duplicados.' },
    { num:'03', titulo:'Selección de pago',      sistema:'Odoo',            desc:'Confirma plan, ingresa código promo si aplica y elige forma de pago.' },
    { num:'04', titulo:'Procesamiento del cobro',sistema:'Odoo + MegaSoft', desc:'Pasarela retorna: APROBADO / RECHAZADO / PENDIENTE. Factura generada automáticamente.' },
    { num:'05', titulo:'Activación en EVO',      sistema:'Odoo → EVO API',  desc:'Odoo envía instrucción a EVO. EVO activa membresía y envía contrato + credenciales + instrucciones facial.' },
    { num:'06', titulo:'Registro facial (kiosco)',sistema:'EVO — sede',     desc:'PC + cámara en sede. Socio captura biométrico facial vinculado a su perfil.' },
    { num:'07', titulo:'Acceso por torniquete',  sistema:'EVO',             desc:'EVO verifica facial + membresía activa + sin bloqueos en tiempo real.' },
  ],

  preventa: {
    descripcion: 'El sistema debe permitir pagos ANTES de la inauguración. El cobro se procesa de inmediato pero la membresía queda congelada en Odoo hasta el día de apertura.',
    escenarios: [
      { caso: 'Paga 30 días antes',    cobro: 'Inmediato', odoo: 'Congelada',    evo: 'Día de apertura' },
      { caso: 'Paga 1 día antes',      cobro: 'Inmediato', odoo: 'Congelada',    evo: 'Día de apertura' },
      { caso: 'Paga el día de apertura', cobro: 'Inmediato', odoo: 'Activa',    evo: 'Inmediata' },
      { caso: 'Paga después',          cobro: 'Inmediato', odoo: 'Activa',       evo: 'Inmediata' },
    ],
    pendiente: '¿Odoo v18 soporta fecha de inicio diferida de la fecha de cobro? Si no, ¿cuánto desarrollo adicional?'
  },

  moraEscala: [
    { dias: 'Día 0 (vencido)',  odoo: 'Por vencer',          evo: 'Acceso activo',          accion: 'Renovar plan' },
    { dias: 'Días 1–3',         odoo: 'En gracia',            evo: 'Activo + aviso pantalla', accion: 'Pagar' },
    { dias: 'Días 3–5',         odoo: 'Suspendida',           evo: 'BLOQUEADO automático',   accion: 'Pagar deuda' },
    { dias: 'Más de 5 días',    odoo: 'Cancelada por mora',   evo: 'BLOQUEADO permanente',   accion: 'Nuevo plan' },
  ],

  promociones: [
    { tipo: 'GENERAL',     titulo: 'Descuento automático', desc: 'Sin código. Se aplica a todos al cumplir condiciones de sede, plan y fecha.' },
    { tipo: 'CORPORATIVO', titulo: 'Convenio empresarial', desc: 'Código de convenio (ej: CORP-COCACOLA). Uso múltiple. Factura a nombre de empresa.' },
    { tipo: 'COMERCIAL',   titulo: 'Alianza con marcas',   desc: 'Alianzas externas. Puede o no requerir código. Wizard de 7 pasos en Odoo.' },
    { tipo: 'UPGRADE',     titulo: 'Migración de plan',    desc: 'Solo socios activos. Migra a plan superior. Flujo pendiente de documentar completamente.' },
  ],

  pendientes: [
    { nivel: 'CRÍTICO', titulo: 'Preventa — fecha inicio diferida en Odoo v18',   desc: '¿Es configuración nativa o requiere desarrollo adicional?' },
    { nivel: 'CRÍTICO', titulo: 'Flujo exacto de Cashea (MegaSoft)',               desc: '¿El pago es interno en Odoo o redirección a página externa?' },
    { nivel: 'CRÍTICO', titulo: 'Alcance contractual de MegaSoft',                 desc: '¿Lo gestiona Lixie o FitNation contrata por separado?' },
    { nivel: 'CRÍTICO', titulo: 'Proveedor de torniquetes',                        desc: 'EVO figura como "Proveedor" en checklist, no como ejecutor.' },
    { nivel: 'TÉCNICO', titulo: 'Validación automática menor de edad en Odoo',     desc: '¿Bloquea en frontend o requiere lógica de negocio?' },
    { nivel: 'TÉCNICO', titulo: 'Diferenciación corporativo vs individual',        desc: '¿El usuario lo selecciona o el código lo activa automáticamente?' },
    { nivel: 'TÉCNICO', titulo: 'Sección T.I. del checklist vacía (ítems 1–5)',   desc: 'Requerimientos técnicos sin contenido — completar antes de parametrizar.' },
    { nivel: 'TÉCNICO', titulo: 'Flujo completo del Upgrade de plan',              desc: '¿Descuento sobre diferencia? ¿Días restantes se acreditan?' },
    { nivel: 'OPERATIVO', titulo: 'Tablero de seguimiento (Trello/Notion)',        desc: '¿Quién lo configura: Lixie o FitNation?' },
    { nivel: 'OPERATIVO', titulo: 'Precios visibles en web pública',               desc: 'Marketing debe definir si se muestran antes o después del clic.' },
  ],

  comunicaciones: [
    { evento: 'Confirmación pago en preventa',      origen: 'Odoo',       destinatario: 'Socio' },
    { evento: 'Recordatorio 3 días antes apertura', origen: 'Odoo',       destinatario: 'Socio' },
    { evento: 'Activación el día de apertura',      origen: 'Odoo + EVO', destinatario: 'Socio' },
    { evento: 'Compra exitosa + bienvenida',        origen: 'Odoo',       destinatario: 'Socio' },
    { evento: 'Contrato y credenciales app',        origen: 'EVO',        destinatario: 'Socio' },
    { evento: 'Instrucciones registro facial',      origen: 'EVO',        destinatario: 'Socio' },
    { evento: 'Confirmación registro facial',       origen: 'EVO',        destinatario: 'Socio' },
    { evento: 'Aviso 5 días antes de vencer',       origen: 'Odoo',       destinatario: 'Socio' },
    { evento: 'Plan vencido + link de pago',        origen: 'Odoo',       destinatario: 'Socio' },
    { evento: 'Aviso día 3 de mora',                origen: 'Odoo',       destinatario: 'Socio' },
    { evento: 'Acceso suspendido',                  origen: 'Odoo + EVO', destinatario: 'Socio' },
    { evento: 'Reactivación exitosa',               origen: 'EVO',        destinatario: 'Socio' },
    { evento: 'Confirmación cancelación',           origen: 'Odoo',       destinatario: 'Socio' },
  ],
};

// ══════════════════════════════════════════════════════════════════
// 1. HTML — GitHub Pages
// ══════════════════════════════════════════════════════════════════
function buildHTML() {
  const html = `<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>${DATA.proyecto} v${DATA.version}</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
<style>
:root{
  --brand:#1A3567;--brand-mid:#2B4FA0;--brand-light:#E8EEF8;--brand-xlight:#F4F7FD;
  --teal:#0F6B5C;--teal-light:#E0F4F0;--amber:#B45309;--amber-light:#FEF3C7;
  --red:#991B1B;--red-light:#FEE2E2;--green:#166534;--green-light:#DCFCE7;
  --gray:#374151;--gray-mid:#6B7280;--gray-light:#F3F4F6;
  --border:#E5E7EB;--white:#fff;--text:#111827;--text2:#4B5563;
  --sans:'Inter',sans-serif;--mono:'JetBrains Mono',monospace;
}
*{box-sizing:border-box;margin:0;padding:0;}
body{font-family:var(--sans);background:#F8FAFC;color:var(--text);height:100vh;overflow:hidden;display:flex;flex-direction:column;}

/* TOPBAR */
.topbar{height:52px;background:var(--brand);display:flex;align-items:center;padding:0 20px;gap:12px;flex-shrink:0;}
.topbar-brand{color:#fff;font-size:15px;font-weight:600;}
.topbar-brand span{opacity:.55;font-weight:300;font-size:12px;margin-left:8px;}
.topbar-ver{font-family:var(--mono);font-size:10px;background:rgba(255,255,255,.12);color:rgba(255,255,255,.8);padding:3px 8px;border-radius:4px;margin-left:auto;}

/* LAYOUT */
.workspace{display:flex;flex:1;overflow:hidden;}

/* SIDEBAR */
.sidebar{width:252px;background:var(--white);border-right:1px solid var(--border);overflow-y:auto;flex-shrink:0;}
.sb-sec{padding:14px 12px 6px;font-size:10px;font-weight:600;letter-spacing:1.2px;color:var(--gray-mid);text-transform:uppercase;}
.sb-div{height:1px;background:var(--border);margin:6px 12px;}
.nav-item{display:flex;align-items:center;gap:10px;padding:8px 14px;cursor:pointer;border-radius:6px;margin:1px 8px;font-size:13px;color:var(--text2);position:relative;transition:background .15s;}
.nav-item:hover{background:var(--gray-light);}
.nav-item.active{background:var(--brand-light);color:var(--brand);font-weight:500;}
.nav-item.active::before{content:'';position:absolute;left:0;top:4px;bottom:4px;width:3px;background:var(--brand);border-radius:0 2px 2px 0;margin-left:-8px;}
.nav-icon{width:18px;height:18px;border-radius:4px;display:flex;align-items:center;justify-content:center;font-size:11px;flex-shrink:0;background:var(--gray-light);color:var(--gray-mid);}
.nav-item.active .nav-icon{background:var(--brand);color:#fff;}
.nav-chevron{margin-left:auto;font-size:10px;color:var(--gray-mid);transition:transform .2s;}
.nav-item.open .nav-chevron{transform:rotate(90deg);}
.nav-sub{overflow:hidden;max-height:0;transition:max-height .25s ease;}
.nav-sub.open{max-height:500px;}
.nav-sub-item{padding:7px 14px 7px 42px;font-size:12px;color:var(--text2);cursor:pointer;border-radius:6px;margin:1px 8px;transition:background .15s;}
.nav-sub-item:hover{background:var(--gray-light);}
.nav-sub-item.active{background:var(--brand-light);color:var(--brand);font-weight:500;}

/* MAIN */
.main{flex:1;display:flex;overflow:hidden;}

/* DETAIL PANEL */
.detail-panel{width:360px;background:var(--white);border-right:1px solid var(--border);overflow-y:auto;flex-shrink:0;transition:width .2s;}
.detail-panel.hidden{width:0;overflow:hidden;}
.dh{padding:18px 18px 12px;border-bottom:1px solid var(--border);}
.dh-bc{font-size:10px;font-family:var(--mono);color:var(--gray-mid);margin-bottom:5px;}
.dh-title{font-size:15px;font-weight:600;color:var(--brand);}
.dh-desc{font-size:12px;color:var(--text2);margin-top:4px;line-height:1.6;}
.db{padding:14px 16px;}

/* SUBPANEL */
.subpanel{flex:1;background:#F8FAFC;overflow-y:auto;}
.sp-empty{display:flex;flex-direction:column;align-items:center;justify-content:center;height:100%;color:var(--gray-mid);font-size:13px;gap:8px;}
.sp-header{padding:18px 22px 12px;border-bottom:1px solid var(--border);background:var(--white);}
.sp-bc{font-size:10px;font-family:var(--mono);color:var(--gray-mid);margin-bottom:3px;}
.sp-title{font-size:15px;font-weight:600;color:var(--brand);}
.sp-body{padding:18px 22px;}

/* CARDS */
.card{background:var(--white);border:1px solid var(--border);border-radius:8px;padding:12px 14px;margin-bottom:8px;cursor:pointer;transition:box-shadow .15s,border-color .15s;position:relative;}
.card:hover{box-shadow:0 2px 10px rgba(26,53,103,.1);border-color:var(--brand-mid);}
.card.active{border-color:var(--brand);box-shadow:0 0 0 2px rgba(43,79,160,.15);}
.card-tag{font-size:10px;font-family:var(--mono);font-weight:500;padding:2px 7px;border-radius:3px;display:inline-block;margin-bottom:5px;}
.card-title{font-size:13px;font-weight:600;color:var(--text);margin-bottom:2px;}
.card-desc{font-size:12px;color:var(--text2);line-height:1.5;}
.card-arr{position:absolute;right:10px;top:50%;transform:translateY(-50%);color:var(--gray-mid);}
.card:hover .card-arr,.card.active .card-arr{color:var(--brand);}

/* ALERTS */
.alert{border-radius:7px;padding:10px 12px;margin:8px 0;display:flex;gap:8px;align-items:flex-start;}
.alert-icon{font-size:13px;flex-shrink:0;}
.a-title{font-size:12px;font-weight:600;margin-bottom:2px;}
.a-text{font-size:11px;line-height:1.6;}
.alert-warn{background:var(--amber-light);border-left:3px solid var(--amber);}
.alert-warn .a-title{color:var(--amber);}
.alert-info{background:var(--brand-light);border-left:3px solid var(--brand-mid);}
.alert-info .a-title{color:var(--brand);}
.alert-crit{background:var(--red-light);border-left:3px solid var(--red);}
.alert-crit .a-title{color:var(--red);}
.alert-ok{background:var(--green-light);border-left:3px solid var(--green);}
.alert-ok .a-title{color:var(--green);}

/* TABLES */
.tw{overflow-x:auto;margin:8px 0;}
table{width:100%;border-collapse:collapse;font-size:12px;}
th{background:var(--brand);color:#fff;padding:7px 11px;text-align:left;font-size:11px;font-weight:500;}
td{padding:6px 11px;border-bottom:1px solid var(--border);color:var(--text2);}
tr:last-child td{border-bottom:none;}
tr:nth-child(even) td{background:var(--gray-light);}

/* STEP LIST */
.sl{list-style:none;}
.si{display:flex;gap:9px;padding:7px 0;border-bottom:1px solid var(--border);}
.si:last-child{border-bottom:none;}
.sn{font-family:var(--mono);font-size:10px;font-weight:500;background:var(--brand-light);color:var(--brand);padding:2px 6px;border-radius:3px;flex-shrink:0;margin-top:1px;}
.st{font-size:12px;color:var(--text2);line-height:1.6;}
.st b{display:block;font-size:13px;color:var(--text);margin-bottom:1px;}

/* SPEC BLOCK */
.spec{background:var(--brand-xlight);border-radius:6px;padding:10px 12px;margin:8px 0;}
.sr{display:flex;justify-content:space-between;padding:4px 0;border-bottom:1px solid var(--border);font-size:12px;}
.sr:last-child{border-bottom:none;}
.sk{color:var(--text2);}
.sv{font-weight:500;color:var(--text);font-family:var(--mono);font-size:11px;}

/* FLOW STEPS */
.pf{margin:8px 0;}
.pfs{display:flex;gap:9px;align-items:flex-start;padding:6px 0;border-bottom:1px solid var(--border);}
.pfs:last-child{border-bottom:none;}
.pfd{width:6px;height:6px;border-radius:50%;background:var(--brand-mid);flex-shrink:0;margin-top:5px;}
.pft{font-size:12px;color:var(--text2);}
.pft b{color:var(--text);}

/* PROMO GRID */
.pg{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-top:8px;}
.pc{border-radius:6px;padding:10px 12px;border:1px solid var(--border);}
.pt{font-size:9px;font-family:var(--mono);font-weight:600;margin-bottom:4px;text-transform:uppercase;letter-spacing:.5px;}
.pn{font-size:12px;font-weight:600;margin-bottom:3px;}
.pd{font-size:11px;color:var(--text2);}

/* PENDING */
.pi{border-left:3px solid var(--amber);padding:9px 11px;background:var(--amber-light);border-radius:0 6px 6px 0;margin-bottom:8px;}
.pi.crit{border-color:var(--red);background:var(--red-light);}
.pi.ok{border-color:var(--green);background:var(--green-light);}
.piq{font-size:12px;font-weight:600;color:var(--amber);margin-bottom:2px;}
.pi.crit .piq{color:var(--red);}
.pid{font-size:11px;color:var(--text2);}

/* SEC LABEL */
.sl2{font-size:10px;font-weight:600;letter-spacing:1px;text-transform:uppercase;color:var(--gray-mid);margin:12px 0 6px;}

/* MOCKUP */
.mk{background:var(--white);border:1px solid var(--border);border-radius:8px;overflow:hidden;margin:10px 0;}
.mb{background:var(--gray-light);padding:5px 9px;display:flex;align-items:center;gap:5px;border-bottom:1px solid var(--border);}
.md1{width:6px;height:6px;border-radius:50%;background:#ff5f57;}
.md2{width:6px;height:6px;border-radius:50%;background:#febc2e;}
.md3{width:6px;height:6px;border-radius:50%;background:#28c840;}
.mu{flex:1;background:var(--white);border-radius:3px;padding:2px 7px;font-size:10px;font-family:var(--mono);color:var(--gray-mid);border:1px solid var(--border);}
.mbd{padding:10px;}
.mhero{background:linear-gradient(135deg,var(--brand),var(--brand-mid));color:#fff;border-radius:5px;padding:12px;margin-bottom:8px;}
.mhero h3{font-size:12px;font-weight:600;margin-bottom:2px;}
.mhero p{font-size:10px;opacity:.75;}
.mplans{display:grid;grid-template-columns:repeat(3,1fr);gap:6px;margin-bottom:8px;}
.mplan{border:1px solid var(--border);border-radius:5px;padding:6px;text-align:center;}
.mplan.hl{border-color:var(--brand-mid);background:var(--brand-xlight);}
.mpn{font-size:10px;font-weight:600;color:var(--brand);margin-bottom:1px;}
.mpd{font-size:9px;color:var(--gray-mid);}
.mcta{background:var(--brand);color:#fff;border:none;border-radius:4px;padding:7px;font-size:11px;font-weight:500;width:100%;cursor:pointer;font-family:var(--sans);}
.mnav{display:flex;gap:10px;padding:0 0 8px;border-bottom:1px solid var(--border);margin-bottom:8px;}
.mnav a{font-size:10px;color:var(--text2);text-decoration:none;}
.mnav a.active{color:var(--brand);font-weight:600;border-bottom:2px solid var(--brand);padding-bottom:3px;}

/* TAG ROW */
.tr2{display:flex;gap:5px;flex-wrap:wrap;margin-top:8px;}
.tag{font-size:10px;font-family:var(--mono);padding:2px 7px;border-radius:3px;}
.t-odoo{background:var(--brand-light);color:var(--brand);}
.t-evo{background:var(--teal-light);color:var(--teal);}
.t-mega{background:var(--amber-light);color:var(--amber);}
.t-pend{background:var(--red-light);color:var(--red);}

@media(max-width:768px){.sidebar{width:0;overflow:hidden;}.detail-panel{width:0;overflow:hidden;}}
</style>
</head>
<body>
<div class="topbar">
  <div class="topbar-brand">FitNation <span>Modelado de Procesos</span></div>
  <div class="topbar-ver">v${DATA.version} · ${DATA.fecha}</div>
</div>
<div class="workspace">
<aside class="sidebar">
  <div class="sb-sec">Ecosistema</div>
  <div class="nav-item" onclick="openSec('actores',this)"><div class="nav-icon">⚙</div>Actores del sistema<span class="nav-chevron">›</span></div>
  <div class="nav-sub" id="sub-actores">
    <div class="nav-sub-item" onclick="openSub('actores','odoo')">Odoo (Lixie)</div>
    <div class="nav-sub-item" onclick="openSub('actores','evo')">EVO / W12</div>
    <div class="nav-sub-item" onclick="openSub('actores','megasoft')">MegaSoft</div>
    <div class="nav-sub-item" onclick="openSub('actores','integracion')">Integración API / Webhooks</div>
  </div>
  <div class="nav-item" onclick="openSec('web',this)"><div class="nav-icon">🌐</div>Página web · Marketing<span class="nav-chevron">›</span></div>
  <div class="nav-sub" id="sub-web">
    <div class="nav-sub-item" onclick="openSub('web','hero')">Hero y propuesta de valor</div>
    <div class="nav-sub-item" onclick="openSub('web','planes')">Sección de planes</div>
    <div class="nav-sub-item" onclick="openSub('web','comprar')">Flujo: clic en Comprar</div>
    <div class="nav-sub-item" onclick="openSub('web','registro')">Formulario de registro</div>
    <div class="nav-sub-item" onclick="openSub('web','pagos')">Formas de pago</div>
    <div class="nav-sub-item" onclick="openSub('web','confirmacion')">Confirmación y activación</div>
  </div>
  <div class="nav-item" onclick="openSec('pagos',this)"><div class="nav-icon">💳</div>Formas de pago<span class="nav-chevron">›</span></div>
  <div class="nav-sub" id="sub-pagos">
    <div class="nav-sub-item" onclick="openSub('pagos','c2p')">Pago Móvil C2P</div>
    <div class="nav-sub-item" onclick="openSub('pagos','tarjeta')">Tarjeta déb / créd</div>
    <div class="nav-sub-item" onclick="openSub('pagos','cashea')">Cashea (MegaSoft)</div>
    <div class="nav-sub-item" onclick="openSub('pagos','transferencia')">Transferencia bancaria</div>
    <div class="nav-sub-item" onclick="openSub('pagos','efectivo')">Efectivo (sede)</div>
  </div>
  <div class="sb-div"></div>
  <div class="sb-sec">Ciclo del socio</div>
  <div class="nav-item" onclick="openSec('flujo',this)"><div class="nav-icon">→</div>Flujo de inscripción<span class="nav-chevron">›</span></div>
  <div class="nav-sub" id="sub-flujo">
    <div class="nav-sub-item" onclick="openSub('flujo','registro')">Registro y duplicados</div>
    <div class="nav-sub-item" onclick="openSub('flujo','bloqueantes')">Bloqueantes</div>
    <div class="nav-sub-item" onclick="openSub('flujo','activacion')">Activación en EVO</div>
    <div class="nav-sub-item" onclick="openSub('flujo','facial')">Registro facial (kiosco)</div>
    <div class="nav-sub-item" onclick="openSub('flujo','acceso')">Acceso por torniquete</div>
  </div>
  <div class="nav-item" onclick="openSec('preventa',this)"><div class="nav-icon">⏳</div>Preventa y apertura<span class="nav-chevron">›</span></div>
  <div class="nav-sub" id="sub-preventa">
    <div class="nav-sub-item" onclick="openSub('preventa','concepto')">Concepto plan congelado</div>
    <div class="nav-sub-item" onclick="openSub('preventa','escenarios')">Escenarios y ejemplos</div>
    <div class="nav-sub-item" onclick="openSub('preventa','apertura')">Día de apertura</div>
  </div>
  <div class="nav-item" onclick="openSec('cobro',this)"><div class="nav-icon">📅</div>Cobro, mora y renovación<span class="nav-chevron">›</span></div>
  <div class="nav-sub" id="sub-cobro">
    <div class="nav-sub-item" onclick="openSub('cobro','ciclo')">Ciclo de cobro</div>
    <div class="nav-sub-item" onclick="openSub('cobro','mora')">Escala de mora</div>
    <div class="nav-sub-item" onclick="openSub('cobro','renovacion')">Renovación manual</div>
    <div class="nav-sub-item" onclick="openSub('cobro','cancelacion')">Suspensión y cancelación</div>
  </div>
  <div class="sb-div"></div>
  <div class="sb-sec">Comercial</div>
  <div class="nav-item" onclick="openSec('marketing',this)"><div class="nav-icon">🎯</div>Módulo de marketing<span class="nav-chevron">›</span></div>
  <div class="nav-sub" id="sub-marketing">
    <div class="nav-sub-item" onclick="openSub('marketing','tipos')">Tipos de promoción</div>
    <div class="nav-sub-item" onclick="openSub('marketing','corporativo')">Convenio corporativo</div>
    <div class="nav-sub-item" onclick="openSub('marketing','upgrade')">Upgrade de plan</div>
    <div class="nav-sub-item" onclick="openSub('marketing','estados')">Estados de promoción</div>
  </div>
  <div class="sb-div"></div>
  <div class="sb-sec">Gestión</div>
  <div class="nav-item" onclick="openSec('pendientes',this)"><div class="nav-icon">!</div>Pendientes para Lixie<span class="nav-chevron">›</span></div>
  <div class="nav-sub" id="sub-pendientes">
    <div class="nav-sub-item" onclick="openSub('pendientes','criticos')">Críticos</div>
    <div class="nav-sub-item" onclick="openSub('pendientes','tecnicos')">Técnicos</div>
    <div class="nav-sub-item" onclick="openSub('pendientes','operativos')">Operativos</div>
  </div>
  <div class="nav-item" onclick="openSec('resumen',this)"><div class="nav-icon">✓</div>Resumen ejecutivo<span class="nav-chevron">›</span></div>
  <div class="nav-sub" id="sub-resumen">
    <div class="nav-sub-item" onclick="openSub('resumen','flujo')">Flujo completo (12 pasos)</div>
    <div class="nav-sub-item" onclick="openSub('resumen','comms')">Comunicaciones automáticas</div>
  </div>
</aside>
<div class="main">
  <div class="detail-panel hidden" id="detailPanel">
    <div class="dh"><div class="dh-bc" id="dBC"></div><div class="dh-title" id="dTitle"></div><div class="dh-desc" id="dDesc"></div></div>
    <div class="db" id="dBody"></div>
  </div>
  <div class="subpanel" id="subPanel">
    <div class="sp-empty" id="spEmpty"><div style="font-size:28px;opacity:.25">←</div><div>Selecciona una sección del menú</div><div style="font-size:11px;opacity:.6">FitNation · Modelado de Procesos v${DATA.version}</div></div>
    <div id="spContent" style="display:none">
      <div class="sp-header"><div class="sp-bc" id="spBC"></div><div class="sp-title" id="spTitle"></div></div>
      <div class="sp-body" id="spBody"></div>
    </div>
  </div>
</div>
</div>
<script>
const SECS = {
  actores:{title:'Actores del ecosistema',desc:'Tres sistemas interconectados. Cada uno es dueño exclusivo de su dominio funcional.',
    cards:[
      {id:'odoo',tag:'BACKOFFICE',tc:'t-odoo',title:'ODOO · Lixie.io',desc:'Núcleo del negocio. Web, cobros, facturación, contabilidad, suscripciones.'},
      {id:'evo',tag:'OPERATIVO',tc:'t-evo',title:'EVO / W12',desc:'Gestión del gimnasio. Torniquetes, facial, app del socio, reserva de clases.'},
      {id:'megasoft',tag:'PASARELA',tc:'t-mega',title:'MegaSoft',desc:'Pasarela complementaria para Cashea y métodos no nativos de Odoo.'},
      {id:'integracion',tag:'TÉCNICO',tc:'t-odoo',title:'Integración API + Webhooks',desc:'Comunicación bidireccional. API REST (Odoo→EVO) y Webhooks (EVO→Odoo).'},
    ]},
  web:{title:'Página web · Especificación Marketing',desc:'Cómo Marketing define la web y cómo impacta cada elemento en el sistema.',
    cards:[
      {id:'hero',tag:'SECCIÓN',tc:'t-odoo',title:'Hero y propuesta de valor',desc:'Headline, subline, CTA principal "Únete ahora". Primer impacto visual.'},
      {id:'planes',tag:'SECCIÓN',tc:'t-odoo',title:'Planes y membresías',desc:'Tarjetas con precio, beneficios, duración. Botón "Comprar" por plan.'},
      {id:'comprar',tag:'FLUJO CRÍTICO',tc:'t-mega',title:'Flujo: clic en Comprar',desc:'Desde el botón hasta el checkout de Odoo — 8 pasos detallados.'},
      {id:'registro',tag:'FORMULARIO',tc:'t-odoo',title:'Formulario de registro',desc:'Datos personales, validación duplicados, bloqueantes.'},
      {id:'pagos',tag:'CHECKOUT',tc:'t-mega',title:'Selección de pago',desc:'C2P, tarjeta, Cashea, transferencia. Flujo de cada método.'},
      {id:'confirmacion',tag:'POST-PAGO',tc:'t-evo',title:'Confirmación y activación',desc:'Pantalla éxito, emails automáticos, instrucciones app y facial.'},
    ]},
  pagos:{title:'Formas de pago',desc:'Cinco métodos disponibles. Online y en sede. Cada uno con su flujo de confirmación.',
    cards:[
      {id:'c2p',tag:'ONLINE+SEDE',tc:'t-odoo',title:'Pago Móvil C2P',desc:'Banco origen + número teléfono. Confirmación casi inmediata.'},
      {id:'tarjeta',tag:'ONLINE+SEDE',tc:'t-odoo',title:'Tarjeta débito / crédito',desc:'Formulario seguro Odoo. Respuesta en segundos.'},
      {id:'cashea',tag:'ONLINE+SEDE',tc:'t-mega',title:'Cashea · MegaSoft',desc:'Método digital venezolano. Flujo exacto pendiente confirmar.'},
      {id:'transferencia',tag:'ONLINE',tc:'t-odoo',title:'Transferencia bancaria',desc:'Activación manual tras confirmación del admin.'},
      {id:'efectivo',tag:'SOLO SEDE',tc:'t-evo',title:'Efectivo en sede',desc:'Solo en recepción vía POS Odoo. No aplica para compra online.'},
    ]},
  flujo:{title:'Flujo de inscripción del socio',desc:'Desde que el usuario hace clic en "Inscribirme" hasta que accede al gimnasio.',
    cards:[
      {id:'registro',tag:'PASO 02',tc:'t-odoo',title:'Registro y duplicados',desc:'Datos personales. Odoo consulta EVO para evitar duplicados.'},
      {id:'bloqueantes',tag:'VALIDACIÓN',tc:'t-pend',title:'Bloqueantes del proceso',desc:'Menor de edad exige responsable. Corporativo exige datos fiscales.'},
      {id:'activacion',tag:'PASO 05',tc:'t-evo',title:'Activación en EVO',desc:'Odoo envía instrucción API. EVO activa y envía contrato + credenciales.'},
      {id:'facial',tag:'PASO 06',tc:'t-evo',title:'Registro facial en kiosco',desc:'PC + cámara en sede. Biométrico vinculado al perfil del socio.'},
      {id:'acceso',tag:'PASO 07',tc:'t-evo',title:'Acceso por torniquete',desc:'EVO valida facial + membresía activa + sin bloqueos en tiempo real.'},
    ]},
  preventa:{title:'Preventa y apertura de sede',desc:'Requisito funcional obligatorio. Confirmado en checklist y plan F-PRY-26.',
    cards:[
      {id:'concepto',tag:'REQUISITO',tc:'t-pend',title:'Plan congelado',desc:'Cobro inmediato al pagar. Activación en EVO solo el día de apertura.'},
      {id:'escenarios',tag:'EJEMPLOS',tc:'t-odoo',title:'Escenarios y tabla',desc:'Casos: paga antes, mismo día, después de la apertura.'},
      {id:'apertura',tag:'DÍA D',tc:'t-evo',title:'Día de apertura',desc:'Odoo activa en lote. EVO habilita accesos. Emails masivos.'},
    ]},
  cobro:{title:'Cobro, mora y renovación',desc:'Ciclo financiero del plan. Notificaciones automáticas y escala de mora.',
    cards:[
      {id:'ciclo',tag:'CICLO',tc:'t-odoo',title:'Ciclo de cobro',desc:'Avisos: -5 días, día 0, día 3. Sin cobro automático. Manual.'},
      {id:'mora',tag:'CRÍTICO',tc:'t-pend',title:'Escala de mora y bloqueo',desc:'Días 3–5: acceso bloqueado. Más de 5: plan cancelado.'},
      {id:'renovacion',tag:'FLUJO',tc:'t-evo',title:'Renovación manual',desc:'Portal, app o POS. Aprobado → EVO reactiva acceso de inmediato.'},
      {id:'cancelacion',tag:'FLUJO',tc:'t-pend',title:'Suspensión y cancelación',desc:'Por mora, administrativa o webhook desde EVO. Bloqueo inmediato.'},
    ]},
  marketing:{title:'Módulo de mercadeo y promociones',desc:'Wizard de 7 pasos en Odoo. Cuatro tipos de promoción configurables.',
    cards:[
      {id:'tipos',tag:'WIZARD',tc:'t-odoo',title:'Tipos de promoción',desc:'General, Corporativo, Comercial, Upgrade.'},
      {id:'corporativo',tag:'B2B',tc:'t-evo',title:'Convenio corporativo',desc:'Código CORP-EMPRESA. Uso múltiple. Factura a nombre de empresa.'},
      {id:'upgrade',tag:'PENDIENTE',tc:'t-pend',title:'Upgrade de plan',desc:'Solo socios activos. Flujo técnico completo pendiente de documentar.'},
      {id:'estados',tag:'ESTADOS',tc:'t-odoo',title:'Estados de una promoción',desc:'Borrador → Programada → Activa → Pausada → Finalizada.'},
    ]},
  pendientes:{title:'Pendientes para Lixie',desc:'Puntos que deben resolverse en la próxima reunión de alineación de alcances.',
    cards:[
      {id:'criticos',tag:'BLOQUEAN ENTREGA',tc:'t-pend',title:'Pendientes críticos (4)',desc:'Preventa, Cashea, MegaSoft contractual, proveedor torniquetes.'},
      {id:'tecnicos',tag:'DESARROLLO',tc:'t-odoo',title:'Pendientes técnicos (4)',desc:'Validación edad, corporativo, sección T.I. vacía, flujo Upgrade.'},
      {id:'operativos',tag:'OPERACIÓN',tc:'t-evo',title:'Pendientes operativos (2)',desc:'Tablero de seguimiento, precios en web pública.'},
    ]},
  resumen:{title:'Resumen ejecutivo',desc:'Vista completa del ciclo del socio y comunicaciones automáticas del sistema.',
    cards:[
      {id:'flujo',tag:'12 PASOS',tc:'t-odoo',title:'Flujo completo',desc:'Desde preventa y apertura hasta cancelación definitiva.'},
      {id:'comms',tag:'13 EMAILS',tc:'t-evo',title:'Comunicaciones automáticas',desc:'Todos los correos del sistema: Odoo, EVO y ambos.'},
    ]},
};

const SUBS = {
  'actores.odoo':{title:'ODOO · Backoffice FitNation',bc:'Actores › Odoo',html:\`
    <div class="sl2">Responsabilidades</div>
    <ul class="sl">
      <li class="si"><span class="sn">WEB</span><div class="st">Página web y e-commerce — portal principal de ventas online</div></li>
      <li class="si"><span class="sn">POS</span><div class="st">Punto de venta en sede — cobro físico de membresías y productos</div></li>
      <li class="si"><span class="sn">PAY</span><div class="st">Pasarelas de pago: C2P, tarjeta, transferencia. Cobro y conciliación</div></li>
      <li class="si"><span class="sn">FAC</span><div class="st">Facturación y retenciones según normativa fiscal venezolana</div></li>
      <li class="si"><span class="sn">CON</span><div class="st">Contabilidad: asientos automáticos y reportes financieros</div></li>
      <li class="si"><span class="sn">SUB</span><div class="st">Suscripciones: ciclo de cobro, vencimientos, notificaciones</div></li>
      <li class="si"><span class="sn">MKT</span><div class="st">Módulo de mercadeo: wizard de 7 pasos para promociones</div></li>
    </ul>
    <div class="alert alert-info"><span class="alert-icon">i</span><div><div class="a-title">Regla de oro</div><div class="a-text">Odoo es el dueño exclusivo de toda transacción económica. Nunca toca acceso físico.</div></div></div>
    <div class="spec">
      <div class="sr"><span class="sk">Proveedor</span><span class="sv">Lixie.io · Caracas, VE</span></div>
      <div class="sr"><span class="sk">Versión</span><span class="sv">Odoo v18</span></div>
      <div class="sr"><span class="sk">Seguridad</span><span class="sv">ISO 27001</span></div>
      <div class="sr"><span class="sk">Comunicación saliente</span><span class="sv">API REST → EVO</span></div>
    </div>\`},
  'actores.evo':{title:'EVO / W12 · Operaciones del gimnasio',bc:'Actores › EVO',html:\`
    <div class="sl2">Responsabilidades</div>
    <ul class="sl">
      <li class="si"><span class="sn">ACC</span><div class="st">Control de acceso físico — torniquetes en todas las sedes</div></li>
      <li class="si"><span class="sn">BIO</span><div class="st">Biométrico facial — captura, almacenamiento y verificación en tiempo real</div></li>
      <li class="si"><span class="sn">APP</span><div class="st">App móvil del socio — reserva de clases, historial deportivo</div></li>
      <li class="si"><span class="sn">MEM</span><div class="st">Membresías — solo recibe estado PAGADO/ACTIVO desde Odoo</div></li>
      <li class="si"><span class="sn">CON</span><div class="st">Envío automático de contratos y credenciales al socio por correo</div></li>
    </ul>
    <div class="alert alert-info"><span class="alert-icon">i</span><div><div class="a-title">Regla de oro</div><div class="a-text">EVO es el dueño exclusivo de todo acceso físico. Nunca toca cobros ni contabilidad.</div></div></div>
    <div class="spec">
      <div class="sr"><span class="sk">Opera en Venezuela</span><span class="sv">Sí</span></div>
      <div class="sr"><span class="sk">Multisede</span><span class="sv">Sí</span></div>
      <div class="sr"><span class="sk">Comunicación saliente</span><span class="sv">Webhook → Odoo</span></div>
      <div class="sr"><span class="sk">Comunicación entrante</span><span class="sv">API REST desde Odoo</span></div>
    </div>\`},
  'actores.megasoft':{title:'MegaSoft · Pasarela complementaria',bc:'Actores › MegaSoft',html:\`
    <div class="sl2">Qué resuelve</div>
    <ul class="sl">
      <li class="si"><span class="sn">CS</span><div class="st"><b>Cashea</b> — método de pago digital venezolano</div></li>
      <li class="si"><span class="sn">C2P</span><div class="st"><b>Pago Móvil C2P</b> — complemento donde Odoo no alcanza nativamente</div></li>
    </ul>
    <div class="alert alert-warn"><span class="alert-icon">?</span><div><div class="a-title">Pendiente crítico</div><div class="a-text">¿MegaSoft ya está integrado o es desarrollo nuevo? ¿Quién gestiona la relación contractual?</div></div></div>
    <div class="alert alert-warn"><span class="alert-icon">?</span><div><div class="a-title">Flujo de Cashea sin confirmar</div><div class="a-text">¿El usuario paga dentro del portal Odoo o es redirigido a una página externa de MegaSoft?</div></div></div>\`},
  'actores.integracion':{title:'Integración API REST + Webhooks',bc:'Actores › Integración',html:\`
    <div class="sl2">Flujo A — Odoo → EVO (API REST — activo)</div>
    <ul class="sl">
      <li class="si"><span class="sn">A1</span><div class="st">Alta de nuevo socio al completar el pago</div></li>
      <li class="si"><span class="sn">A2</span><div class="st">Activación de membresía: estado PAGADO/ACTIVO + fechas</div></li>
      <li class="si"><span class="sn">A3</span><div class="st">Suspensión: Odoo ordena bloqueo de acceso por mora</div></li>
      <li class="si"><span class="sn">A4</span><div class="st">Cancelación: Odoo ordena revocar el acceso definitivamente</div></li>
      <li class="si"><span class="sn">A5</span><div class="st">Reactivación: Odoo ordena habilitar tras renovación</div></li>
    </ul>
    <div class="sl2" style="margin-top:10px;">Flujo B — EVO → Odoo (Webhook — pasivo)</div>
    <ul class="sl">
      <li class="si"><span class="sn">B1</span><div class="st">Excepción operativa: cancelación manual desde EVO</div></li>
      <li class="si"><span class="sn">B2</span><div class="st">Odoo recibe webhook, actualiza estado y detiene facturación</div></li>
    </ul>
    <div class="spec">
      <div class="sr"><span class="sk">Credenciales API EVO</span><span class="sv">Entregadas ✓</span></div>
      <div class="sr"><span class="sk">Webhooks salientes EVO</span><span class="sv">Pendiente habilitar</span></div>
      <div class="sr"><span class="sk">Reintentos automáticos</span><span class="sv">Responsabilidad Lixie</span></div>
    </div>\`},
  'web.hero':{title:'Hero y propuesta de valor',bc:'Web · Marketing › Hero',html:\`
    <div class="sl2">Elementos requeridos por Marketing</div>
    <ul class="sl">
      <li class="si"><span class="sn">H1</span><div class="st"><b>Headline</b> potente y corto enfocado en transformación, no en el gimnasio</div></li>
      <li class="si"><span class="sn">H2</span><div class="st"><b>Subline</b> con beneficio diferencial en máx. 2 líneas</div></li>
      <li class="si"><span class="sn">CTA</span><div class="st"><b>Botón primario</b> "Únete ahora" o "Ver planes" — abre sección de planes</div></li>
      <li class="si"><span class="sn">VID</span><div class="st"><b>Visual</b> video de fondo o imagen hero del gimnasio real</div></li>
      <li class="si"><span class="sn">SOC</span><div class="st"><b>Social proof</b> número de socios activos o logos de empresas convenio</div></li>
    </ul>
    <div class="mk"><div class="mb"><div class="md1"></div><div class="md2"></div><div class="md3"></div><div class="mu">fitnation.com.ve</div></div>
    <div class="mbd"><div class="mhero"><h3>Tu mejor versión<br>empieza aquí.</h3><p>Acceso ilimitado · Clases · App · Reconocimiento facial</p><div style="margin-top:8px;display:flex;gap:6px;"><button style="background:rgba(255,255,255,.9);color:#1A3567;border:none;border-radius:4px;padding:5px 12px;font-size:10px;font-weight:600;cursor:pointer;">Ver planes</button><button style="background:rgba(255,255,255,.15);color:#fff;border:1px solid rgba(255,255,255,.3);border-radius:4px;padding:5px 12px;font-size:10px;cursor:pointer;">Conocer más</button></div></div></div></div>
    <div class="alert alert-info"><span class="alert-icon">i</span><div><div class="a-title">Impacto en sistema</div><div class="a-text">El CTA "Ver planes" debe enlazar a la sección de suscripciones de Odoo. Confirmar implementación con Lixie.</div></div></div>\`},
  'web.planes':{title:'Sección de planes',bc:'Web · Marketing › Planes',html:\`
    <div class="sl2">Estructura de tarjetas</div>
    <ul class="sl">
      <li class="si"><span class="sn">P1</span><div class="st"><b>Nombre del plan</b> visible y grande</div></li>
      <li class="si"><span class="sn">P2</span><div class="st"><b>Precio</b> en Bs. y/o USD referencial</div></li>
      <li class="si"><span class="sn">P3</span><div class="st"><b>Lista de beneficios</b> incluidos</div></li>
      <li class="si"><span class="sn">P4</span><div class="st"><b>Badge "Más popular"</b> en el plan destacado</div></li>
      <li class="si"><span class="sn">P5</span><div class="st"><b>Botón "Comprar"</b> inicia el flujo de inscripción en Odoo</div></li>
    </ul>
    <div class="mk"><div class="mb"><div class="md1"></div><div class="md2"></div><div class="md3"></div><div class="mu">fitnation.com.ve/#planes</div></div>
    <div class="mbd"><div class="mnav"><a href="#">Inicio</a><a href="#" class="active">Planes</a><a href="#">Clases</a><a href="#">Contacto</a></div>
    <div class="mplans"><div class="mplan"><div class="mpn">Mensual</div><div class="mpd">30 días</div></div><div class="mplan hl"><div style="font-size:9px;background:#1A3567;color:#fff;border-radius:3px;padding:1px 5px;display:inline-block;margin-bottom:2px;">Popular</div><div class="mpn">Trimestral</div><div class="mpd">90 días</div></div><div class="mplan"><div class="mpn">Anual</div><div class="mpd">365 días</div></div></div>
    <button class="mcta">Comprar plan →</button></div></div>
    <div class="alert alert-warn"><span class="alert-icon">?</span><div><div class="a-title">A definir con Marketing</div><div class="a-text">¿Se muestran precios en la web pública o solo después de hacer clic?</div></div></div>\`},
  'web.comprar':{title:'Flujo: clic en "Comprar"',bc:'Web · Marketing › Comprar',html:\`
    <div class="sl2">8 pasos desde el botón hasta la activación</div>
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st"><b>Clic en "Comprar"</b> en la tarjeta del plan seleccionado</div></li>
      <li class="si"><span class="sn">2</span><div class="st"><b>¿Tiene cuenta?</b> Login si ya es socio / Registro si es nuevo</div></li>
      <li class="si"><span class="sn">3</span><div class="st"><b>Formulario de registro</b> — datos personales + validaciones</div></li>
      <li class="si"><span class="sn">4</span><div class="st"><b>Resumen del plan</b> — nombre, duración, precio, impuestos</div></li>
      <li class="si"><span class="sn">5</span><div class="st"><b>Código promocional</b> — campo opcional. Odoo aplica el descuento si es válido</div></li>
      <li class="si"><span class="sn">6</span><div class="st"><b>Selección de forma de pago</b> — C2P, tarjeta, Cashea, transferencia</div></li>
      <li class="si"><span class="sn">7</span><div class="st"><b>Procesamiento del cobro</b> — según el método elegido</div></li>
      <li class="si"><span class="sn">8</span><div class="st"><b>Pantalla de éxito</b> — "¡Bienvenido a FitNation! Revisa tu correo."</div></li>
    </ul>\`},
  'web.registro':{title:'Formulario de registro',bc:'Web · Marketing › Registro',html:\`
    <div class="sl2">Campos del formulario</div>
    <div class="spec">
      <div class="sr"><span class="sk">Nombre completo</span><span class="sv">Requerido</span></div>
      <div class="sr"><span class="sk">Cédula / pasaporte</span><span class="sv">Requerido · verifica EVO</span></div>
      <div class="sr"><span class="sk">Correo electrónico</span><span class="sv">Requerido · verifica EVO</span></div>
      <div class="sr"><span class="sk">Teléfono</span><span class="sv">Requerido</span></div>
      <div class="sr"><span class="sk">Fecha de nacimiento</span><span class="sv">Requerido · valida edad</span></div>
      <div class="sr"><span class="sk">RIF empresa (corporativo)</span><span class="sv">Condicional</span></div>
      <div class="sr"><span class="sk">Razón social empresa</span><span class="sv">Condicional</span></div>
    </div>
    <div class="sl2">Validaciones automáticas</div>
    <div class="pf">
      <div class="pfs"><div class="pfd"></div><div class="pft">Odoo consulta EVO por cédula y correo — evita duplicados</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Si &lt;18 años → bloquea y exige responsable legal</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Si plan corporativo → muestra campos de empresa. Factura a nombre de empresa.</div></div>
    </div>\`},
  'web.pagos':{title:'Formas de pago en checkout',bc:'Web · Marketing › Pagos',html:\`
    <div class="sl2">Métodos disponibles online</div>
    <ul class="sl">
      <li class="si"><span class="sn">C2P</span><div class="st"><b>Pago Móvil C2P</b> — banco origen + teléfono. Vía Odoo/MegaSoft.</div></li>
      <li class="si"><span class="sn">TRJ</span><div class="st"><b>Tarjeta déb/créd</b> — formulario seguro Odoo. Segundos.</div></li>
      <li class="si"><span class="sn">CSH</span><div class="st"><b>Cashea</b> — flujo MegaSoft. Pendiente: ¿interno o externo?</div></li>
      <li class="si"><span class="sn">TRF</span><div class="st"><b>Transferencia</b> — manual. Activación tras confirmación admin.</div></li>
    </ul>
    <div class="alert alert-warn"><span class="alert-icon">⚠</span><div><div class="a-title">Efectivo no disponible online</div><div class="a-text">Solo en sede vía POS. No debe mostrarse en el checkout web.</div></div></div>\`},
  'web.confirmacion':{title:'Confirmación y activación post-pago',bc:'Web · Marketing › Confirmación',html:\`
    <div class="sl2">Pantalla de éxito (Odoo)</div>
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st">Mensaje: "¡Bienvenido a FitNation! Tu plan está activo."</div></li>
      <li class="si"><span class="sn">2</span><div class="st">"Revisa tu correo para obtener tus credenciales y el contrato."</div></li>
      <li class="si"><span class="sn">3</span><div class="st">Botón "Descarga la app" con links App Store / Play Store</div></li>
    </ul>
    <div class="sl2" style="margin-top:10px;">Correos automáticos</div>
    <ul class="sl">
      <li class="si"><span class="sn">M1</span><div class="st"><b>Odoo:</b> confirmación de compra + factura adjunta</div></li>
      <li class="si"><span class="sn">M2</span><div class="st"><b>EVO:</b> contrato personalizado + credenciales de app</div></li>
      <li class="si"><span class="sn">M3</span><div class="st"><b>EVO:</b> instrucciones para registro facial en kiosco</div></li>
    </ul>
    <div class="alert alert-warn"><span class="alert-icon">⏳</span><div><div class="a-title">Si es preventa</div><div class="a-text">M1 debe decir: "Tu plan se activa el [fecha de apertura]". M2 y M3 llegan el día de apertura, no ahora.</div></div></div>\`},
  'pagos.c2p':{title:'Pago Móvil C2P',bc:'Formas de pago › C2P',html:\`
    <div class="spec"><div class="sr"><span class="sk">Disponibilidad</span><span class="sv">Online + Sede (POS)</span></div><div class="sr"><span class="sk">Procesado por</span><span class="sv">Odoo / MegaSoft</span></div><div class="sr"><span class="sk">Confirmación</span><span class="sv">Casi inmediata</span></div></div>
    <div class="sl2">Flujo</div>
    <div class="pf">
      <div class="pfs"><div class="pfd"></div><div class="pft">Usuario selecciona C2P en el checkout</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Ingresa banco origen + teléfono + monto</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Odoo envía solicitud a la pasarela C2P</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Pasarela retorna: APROBADO / RECHAZADO / TIMEOUT</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Si APROBADO → Odoo activa suscripción y notifica a EVO</div></div>
    </div>\`},
  'pagos.tarjeta':{title:'Tarjeta débito / crédito',bc:'Formas de pago › Tarjeta',html:\`
    <div class="spec"><div class="sr"><span class="sk">Disponibilidad</span><span class="sv">Online + Sede (POS)</span></div><div class="sr"><span class="sk">Procesado por</span><span class="sv">Odoo (pasarela bancaria)</span></div><div class="sr"><span class="sk">Confirmación</span><span class="sv">Segundos</span></div></div>
    <div class="sl2">Flujo</div>
    <div class="pf">
      <div class="pfs"><div class="pfd"></div><div class="pft">Usuario selecciona "Tarjeta" en el checkout</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Formulario seguro de Odoo: número, vencimiento, CVV</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Odoo procesa vía pasarela bancaria configurada</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Respuesta APROBADO / RECHAZADO en segundos</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Si APROBADO → factura generada + EVO notificado</div></div>
    </div>\`},
  'pagos.cashea':{title:'Cashea · MegaSoft',bc:'Formas de pago › Cashea',html:\`
    <div class="spec"><div class="sr"><span class="sk">Disponibilidad</span><span class="sv">Online + Sede</span></div><div class="sr"><span class="sk">Procesado por</span><span class="sv">MegaSoft</span></div><div class="sr"><span class="sk">Confirmación</span><span class="sv">Pendiente definir</span></div></div>
    <div class="alert alert-warn"><span class="alert-icon">?</span><div><div class="a-title">Flujo exacto pendiente confirmar</div><div class="a-text">¿El usuario paga dentro del portal Odoo (iframe) o es redirigido a MegaSoft/Cashea externamente?</div></div></div>
    <div class="sl2">Flujo conocido</div>
    <div class="pf">
      <div class="pfs"><div class="pfd"></div><div class="pft">Usuario selecciona "Cashea" en el checkout de Odoo</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Odoo redirige o muestra el flujo de MegaSoft para Cashea</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Usuario completa el pago en la interfaz de Cashea</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">MegaSoft notifica a Odoo: APROBADO o RECHAZADO</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Si APROBADO → Odoo activa suscripción y notifica a EVO</div></div>
    </div>\`},
  'pagos.transferencia':{title:'Transferencia bancaria',bc:'Formas de pago › Transferencia',html:\`
    <div class="spec"><div class="sr"><span class="sk">Disponibilidad</span><span class="sv">Online (manual)</span></div><div class="sr"><span class="sk">Procesado por</span><span class="sv">Odoo + revisión admin</span></div><div class="sr"><span class="sk">Confirmación</span><span class="sv">Manual — horas o días</span></div></div>
    <div class="sl2">Flujo</div>
    <div class="pf">
      <div class="pfs"><div class="pfd"></div><div class="pft">Sistema muestra datos bancarios de FitNation</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Usuario realiza la transferencia desde su banco</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Suscripción queda en estado PENDIENTE. Acceso bloqueado.</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Admin confirma manualmente en Odoo</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Odoo activa suscripción → EVO habilita acceso</div></div>
    </div>
    <div class="alert alert-warn"><span class="alert-icon">⚠</span><div><div class="a-title">Riesgo operativo</div><div class="a-text">El socio puede presentarse antes de que el admin confirme. Recepción debe poder ver el estado "Pendiente" en Odoo.</div></div></div>\`},
  'pagos.efectivo':{title:'Efectivo en sede',bc:'Formas de pago › Efectivo',html:\`
    <div class="spec"><div class="sr"><span class="sk">Disponibilidad</span><span class="sv">Solo sede (NO online)</span></div><div class="sr"><span class="sk">Procesado por</span><span class="sv">POS Odoo en recepción</span></div><div class="sr"><span class="sk">Confirmación</span><span class="sv">Inmediata</span></div></div>
    <div class="sl2">Flujo</div>
    <div class="pf">
      <div class="pfs"><div class="pfd"></div><div class="pft">Personal abre el POS de Odoo y busca el perfil del socio</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Selecciona el plan y registra el pago en efectivo</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Odoo genera la factura y activa la suscripción</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Odoo notifica a EVO vía API → acceso habilitado inmediatamente</div></div>
    </div>
    <div class="alert alert-ok"><span class="alert-icon">i</span><div><div class="a-title">Mismo flujo técnico que online</div><div class="a-text">Desde el punto de vista de Odoo y EVO, un pago en efectivo vía POS es idéntico a un pago online aprobado.</div></div></div>\`},
  'flujo.registro':{title:'Registro y verificación de duplicados',bc:'Flujo › Registro',html:\`
    <div class="spec"><div class="sr"><span class="sk">Nombre completo</span><span class="sv">Requerido</span></div><div class="sr"><span class="sk">Cédula / pasaporte</span><span class="sv">Requerido · verifica EVO</span></div><div class="sr"><span class="sk">Correo electrónico</span><span class="sv">Requerido · verifica EVO</span></div><div class="sr"><span class="sk">Teléfono</span><span class="sv">Requerido</span></div><div class="sr"><span class="sk">Fecha de nacimiento</span><span class="sv">Requerido · valida edad</span></div></div>
    <div class="sl2">Verificación de duplicados</div>
    <div class="pf">
      <div class="pfs"><div class="pfd"></div><div class="pft">Odoo consulta silenciosamente a EVO por cédula y correo</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Si existe en EVO → vincula el perfil. No crea duplicado.</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Si no existe → crea perfil nuevo en Odoo y en EVO</div></div>
    </div>\`},
  'flujo.bloqueantes':{title:'Bloqueantes del proceso',bc:'Flujo › Bloqueantes',html:\`
    <div class="alert alert-crit"><span class="alert-icon">⚠</span><div><div class="a-title">Menor de edad (&lt;18 años)</div><div class="a-text">El sistema bloquea el proceso y exige la presencia y firma de un responsable legal antes de activar la membresía.</div></div></div>
    <div class="alert alert-warn"><span class="alert-icon">🏢</span><div><div class="a-title">Plan corporativo / empresarial</div><div class="a-text">Muestra campos adicionales: RIF y razón social. La factura y el contrato salen a nombre de la empresa, no del individuo.</div></div></div>
    <div class="alert alert-warn"><span class="alert-icon">?</span><div><div class="a-title">Pendiente técnico</div><div class="a-text">¿Puede Odoo validar automáticamente la edad con la fecha de nacimiento y mostrar el bloqueo antes del pago?</div></div></div>\`},
  'flujo.activacion':{title:'Activación automática en EVO',bc:'Flujo › Activación',html:\`
    <div class="sl2">Datos que Odoo envía a EVO</div>
    <div class="spec"><div class="sr"><span class="sk">Identificación del socio</span><span class="sv">Cédula / correo</span></div><div class="sr"><span class="sk">Plan adquirido</span><span class="sv">Nombre y tipo</span></div><div class="sr"><span class="sk">Fecha de inicio</span><span class="sv">Hoy o día de apertura</span></div><div class="sr"><span class="sk">Fecha de vencimiento</span><span class="sv">Según duración</span></div><div class="sr"><span class="sk">Estado</span><span class="sv">PAGADO / ACTIVO</span></div></div>
    <div class="sl2" style="margin-top:10px;">Acciones de EVO al recibir</div>
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st">Activa la membresía del socio en el sistema</div></li>
      <li class="si"><span class="sn">2</span><div class="st">Envía contrato personalizado (nombre, plan, fechas, condiciones)</div></li>
      <li class="si"><span class="sn">3</span><div class="st">Envía credenciales de acceso a la app móvil</div></li>
      <li class="si"><span class="sn">4</span><div class="st">Envía instrucciones para registro facial en el kiosco</div></li>
    </ul>\`},
  'flujo.facial':{title:'Registro facial en kiosco de sede',bc:'Flujo › Facial',html:\`
    <div class="spec"><div class="sr"><span class="sk">Hardware</span><span class="sv">PC + cámara en sede</span></div><div class="sr"><span class="sk">Sistema</span><span class="sv">EVO / W12</span></div><div class="sr"><span class="sk">Momento</span><span class="sv">Primera visita a la sede</span></div></div>
    <div class="sl2">Flujo en el kiosco</div>
    <div class="pf">
      <div class="pfs"><div class="pfd"></div><div class="pft">Socio ingresa cédula o correo en el kiosco</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Sistema verifica que tiene membresía activa</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">EVO solicita captura de imagen facial desde la cámara</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">EVO procesa y almacena el biométrico en el perfil</div></div>
      <div class="pfs"><div class="pfd"></div><div class="pft">Confirmación: "Registro exitoso. Ya puedes acceder por los torniquetes."</div></div>
    </div>
    <div class="alert alert-ok"><span class="alert-icon">i</span><div><div class="a-title">Antes de registrar</div><div class="a-text">Recepción puede verificar membresía en EVO y permitir acceso manual hasta que el socio complete el kiosco.</div></div></div>\`},
  'flujo.acceso':{title:'Acceso diario por torniquete',bc:'Flujo › Acceso',html:\`
    <div class="sl2">Verificación en tiempo real (EVO)</div>
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st">Torniquete activa la cámara de reconocimiento</div></li>
      <li class="si"><span class="sn">2</span><div class="st">EVO verifica: rostro registrado ✓</div></li>
      <li class="si"><span class="sn">3</span><div class="st">EVO verifica: membresía activa y vigente ✓</div></li>
      <li class="si"><span class="sn">4</span><div class="st">EVO verifica: sin bloqueos activos ✓</div></li>
      <li class="si"><span class="sn">OK</span><div class="st"><b>Todo OK</b> → torniquete se abre. EVO registra fecha/hora.</div></li>
    </ul>
    <div class="tw"><table><thead><tr><th>Condición</th><th>Torniquete</th></tr></thead><tbody>
      <tr><td>Membresía activa</td><td>✓ Abre</td></tr>
      <tr><td>Vencido día 0</td><td>✗ Renovar</td></tr>
      <tr><td>Mora 1–3 días</td><td>✓ Abre + aviso</td></tr>
      <tr><td>Mora 3–5 días</td><td>✗ Suspendido</td></tr>
      <tr><td>Cancelado</td><td>✗ Nuevo plan</td></tr>
    </tbody></table></div>\`},
  'preventa.concepto':{title:'Concepto: plan congelado',bc:'Preventa › Concepto',html:\`
    <p style="font-size:13px;color:var(--text2);margin-bottom:12px;">El cobro se realiza y confirma de forma inmediata, pero el plan no empieza a contar hasta el día oficial de apertura de la sede.</p>
    <ul class="sl">
      <li class="si"><span class="sn">PAY</span><div class="st"><b>Cobro procesado</b> — dinero cobrado y factura emitida en el momento del pago</div></li>
      <li class="si"><span class="sn">HOLD</span><div class="st"><b>Suscripción congelada</b> — estado "Pagada / En espera de activación". EVO no recibe nada aún.</div></li>
      <li class="si"><span class="sn">ACT</span><div class="st"><b>Activación el día D</b> — Odoo activa en lote y notifica a EVO</div></li>
    </ul>
    <div class="alert alert-warn"><span class="alert-icon">?</span><div><div class="a-title">Pendiente crítico para Lixie</div><div class="a-text">¿Odoo v18 soporta nativamente una fecha de inicio del plan diferente a la fecha del cobro? Si no, ¿cuánto desarrollo adicional?</div></div></div>\`},
  'preventa.escenarios':{title:'Escenarios de preventa',bc:'Preventa › Escenarios',html:\`
    <div class="tw"><table><thead><tr><th>Escenario</th><th>Cobro</th><th>Odoo</th><th>EVO activa</th></tr></thead><tbody>
      <tr><td>Paga 30 días antes</td><td>Inmediato</td><td>Congelada</td><td>Día apertura</td></tr>
      <tr><td>Paga 1 día antes</td><td>Inmediato</td><td>Congelada</td><td>Día apertura</td></tr>
      <tr><td>Paga el día de apertura</td><td>Inmediato</td><td>Activa</td><td>Inmediata</td></tr>
      <tr><td>Paga después</td><td>Inmediato</td><td>Activa</td><td>Inmediata</td></tr>
    </tbody></table></div>
    <div class="alert alert-ok"><span class="alert-icon">✓</span><div><div class="a-title">Regla clave</div><div class="a-text">En todos los casos de preventa, el plan de N días comienza a contar desde el día de apertura. El socio no pierde ni un día.</div></div></div>\`},
  'preventa.apertura':{title:'Día de apertura — activación masiva',bc:'Preventa › Apertura',html:\`
    <div class="sl2">Secuencia del día de inauguración</div>
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st">Odoo tiene configurada la fecha oficial de apertura</div></li>
      <li class="si"><span class="sn">2</span><div class="st">Odoo activa en lote todas las suscripciones en estado "Congelada"</div></li>
      <li class="si"><span class="sn">3</span><div class="st">Para cada una, Odoo envía instrucción a EVO vía API REST</div></li>
      <li class="si"><span class="sn">4</span><div class="st">EVO habilita el acceso de cada socio. Torniquetes listos.</div></li>
      <li class="si"><span class="sn">5</span><div class="st">EVO envía a cada socio: contrato + credenciales + instrucciones facial</div></li>
      <li class="si"><span class="sn">6</span><div class="st">Socios llegan, registran facial en el kiosco y acceden</div></li>
    </ul>
    <div class="alert alert-info"><span class="alert-icon">📬</span><div><div class="a-title">3 días antes de apertura</div><div class="a-text">Odoo envía recordatorio masivo a todos los socios en preventa con instrucciones para el día de apertura.</div></div></div>\`},
  'cobro.ciclo':{title:'Ciclo de cobro',bc:'Cobro › Ciclo',html:\`
    <ul class="sl">
      <li class="si"><span class="sn">-5d</span><div class="st"><b>Aviso vencimiento próximo</b> — link para renovar</div></li>
      <li class="si"><span class="sn">D0</span><div class="st"><b>Plan vencido</b> — aviso + link de pago. Acceso aún activo.</div></li>
      <li class="si"><span class="sn">D3</span><div class="st"><b>Último aviso mora</b> — acceso aún activo (período de gracia)</div></li>
      <li class="si"><span class="sn">D3-5</span><div class="st"><b>Suspensión automática</b> — Odoo suspende + notifica EVO. Acceso BLOQUEADO.</div></li>
      <li class="si"><span class="sn">D5+</span><div class="st"><b>Cancelación por mora</b> — plan cancelado. Requiere nuevo plan.</div></li>
    </ul>
    <div class="alert alert-crit"><span class="alert-icon">🔒</span><div><div class="a-title">El cobro NO es automático</div><div class="a-text">El socio DEBE pagar manualmente. Todo bloqueo activa de forma INMEDIATA y AUTOMÁTICA el cierre del torniquete.</div></div></div>\`},
  'cobro.mora':{title:'Escala de mora',bc:'Cobro › Mora',html:\`
    <div class="tw"><table><thead><tr><th>Días en mora</th><th>Estado Odoo</th><th>EVO / Torniquete</th></tr></thead><tbody>
      <tr><td>Día 0 (vencido)</td><td>Por vencer</td><td>Acceso activo</td></tr>
      <tr><td>Días 1–3</td><td>En gracia</td><td>Activo + aviso en pantalla</td></tr>
      <tr><td>Días 3–5</td><td>Suspendida</td><td>BLOQUEADO automático</td></tr>
      <tr><td>Más de 5 días</td><td>Cancelada</td><td>BLOQUEADO — nuevo plan</td></tr>
    </tbody></table></div>\`},
  'cobro.renovacion':{title:'Renovación manual del plan',bc:'Cobro › Renovación',html:\`
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st">Socio accede al portal web o app móvil</div></li>
      <li class="si"><span class="sn">2</span><div class="st">Selecciona "Renovar mi plan" o "Ponerme al día"</div></li>
      <li class="si"><span class="sn">3</span><div class="st">Puede mantener el mismo plan o cambiar a otro</div></li>
      <li class="si"><span class="sn">4</span><div class="st">Selecciona forma de pago y completa el cobro</div></li>
      <li class="si"><span class="sn">5</span><div class="st">Pago aprobado → Odoo notifica a EVO → acceso reactivado de inmediato</div></li>
    </ul>
    <div class="alert alert-ok"><span class="alert-icon">🏠</span><div><div class="a-title">Renovación en sede</div><div class="a-text">Personal procesa desde el POS de Odoo. Mismo flujo: cobro en Odoo → activación automática en EVO.</div></div></div>\`},
  'cobro.cancelacion':{title:'Suspensión y cancelación',bc:'Cobro › Cancelación',html:\`
    <div class="sl2">A — Por mora (automática)</div>
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st">Odoo detecta mora mayor a 3–5 días → suspende y notifica EVO</div></li>
      <li class="si"><span class="sn">2</span><div class="st">EVO bloquea el acceso facial de inmediato</div></li>
    </ul>
    <div class="sl2" style="margin-top:10px;">B — Administrativa (manual desde Odoo)</div>
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st">Admin suspende manualmente (fraude, solicitud del socio)</div></li>
      <li class="si"><span class="sn">2</span><div class="st">Odoo notifica EVO → acceso bloqueado. Se registra motivo y responsable.</div></li>
    </ul>
    <div class="sl2" style="margin-top:10px;">C — Excepción desde EVO (webhook)</div>
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st">EVO emite webhook automático a Odoo</div></li>
      <li class="si"><span class="sn">2</span><div class="st">Odoo actualiza estado y detiene facturación</div></li>
    </ul>\`},
  'marketing.tipos':{title:'Tipos de promoción',bc:'Marketing › Tipos',html:\`
    <div class="pg">
      <div class="pc" style="background:var(--brand-light)"><div class="pt" style="color:var(--brand)">GENERAL</div><div class="pn">Descuento automático</div><div class="pd">Sin código. Aplica a todos al cumplir condiciones de sede, plan y fecha.</div></div>
      <div class="pc" style="background:var(--teal-light)"><div class="pt" style="color:var(--teal)">CORPORATIVO</div><div class="pn">Convenio empresarial</div><div class="pd">Código de convenio. Uso múltiple. Factura a nombre de empresa.</div></div>
      <div class="pc" style="background:var(--amber-light)"><div class="pt" style="color:var(--amber)">COMERCIAL</div><div class="pn">Alianza con marcas</div><div class="pd">Alianzas externas. Puede o no requerir código. Wizard de 7 pasos.</div></div>
      <div class="pc" style="background:var(--green-light)"><div class="pt" style="color:var(--green)">UPGRADE</div><div class="pn">Migración de plan</div><div class="pd">Solo socios activos. Migra a plan superior. Flujo pendiente.</div></div>
    </div>\`},
  'marketing.corporativo':{title:'Convenio corporativo',bc:'Marketing › Corporativo',html:\`
    <ul class="sl">
      <li class="si"><span class="sn">1</span><div class="st">Marketing crea el convenio en Odoo con el wizard</div></li>
      <li class="si"><span class="sn">2</span><div class="st">Se genera código único (ej: CORP-COCACOLA)</div></li>
      <li class="si"><span class="sn">3</span><div class="st">La empresa distribuye el código entre sus empleados</div></li>
      <li class="si"><span class="sn">4</span><div class="st">El empleado ingresa el código en el checkout de la web</div></li>
      <li class="si"><span class="sn">5</span><div class="st">Odoo valida, aplica descuento y solicita datos de empresa</div></li>
      <li class="si"><span class="sn">6</span><div class="st">Factura emitida a nombre de la empresa, no del individuo</div></li>
    </ul>
    <div class="alert alert-warn"><span class="alert-icon">?</span><div><div class="a-title">Pendiente definir</div><div class="a-text">¿Los códigos tienen cupo máximo de usos o son ilimitados?</div></div></div>\`},
  'marketing.upgrade':{title:'Upgrade de plan activo',bc:'Marketing › Upgrade',html:\`
    <div class="alert alert-crit"><span class="alert-icon">!</span><div><div class="a-title">Sub-flujo pendiente de documentar</div><div class="a-text">El tipo Upgrade existe en el módulo de Marketing de Odoo pero el flujo técnico completo no está definido para FitNation.</div></div></div>
    <ul class="sl">
      <li class="si"><span class="sn">REQ</span><div class="st">Solo disponible para socios con membresía activa</div></li>
      <li class="si"><span class="sn">DESC</span><div class="st">Descuento sobre la diferencia entre plan actual y plan superior</div></li>
      <li class="si"><span class="sn">DÍAS</span><div class="st">Pendiente: ¿los días restantes del plan anterior se acreditan?</div></li>
      <li class="si"><span class="sn">EVO</span><div class="st">Pendiente: ¿EVO recibe instrucción especial de "upgrade" o es nueva membresía?</div></li>
    </ul>\`},
  'marketing.estados':{title:'Estados de una promoción',bc:'Marketing › Estados',html:\`
    <ul class="sl">
      <li class="si"><span class="sn">BOR</span><div class="st"><b>Borrador</b> — en creación, no visible ni activa</div></li>
      <li class="si"><span class="sn">PRG</span><div class="st"><b>Programada</b> — se activará en una fecha futura. Útil para promos de apertura.</div></li>
      <li class="si"><span class="sn">ACT</span><div class="st"><b>Activa</b> — visible y aplicable en el checkout ahora</div></li>
      <li class="si"><span class="sn">PAU</span><div class="st"><b>Pausada</b> — temporalmente desactivada. Existentes se respetan.</div></li>
      <li class="si"><span class="sn">FIN</span><div class="st"><b>Finalizada</b> — venció o se agotó el cupo</div></li>
    </ul>
    <div class="alert alert-info"><span class="alert-icon">i</span><div><div class="a-title">Conexión con preventa</div><div class="a-text">Las promos de apertura pueden crearse en estado "Programada" y activarse automáticamente el día de inauguración.</div></div></div>\`},
  'pendientes.criticos':{title:'Pendientes críticos',bc:'Pendientes › Críticos',html:\`
    <div class="pi crit"><div class="piq">¿Odoo v18 soporta fecha de inicio diferida del cobro?</div><div class="pid">Si no es nativo → desarrollo adicional. Determina si preventa es configuración o sprint extra.</div></div>
    <div class="pi crit"><div class="piq">Flujo exacto de Cashea — ¿interno o externo?</div><div class="pid">¿El usuario paga dentro de Odoo (iframe) o es redirigido a MegaSoft? Afecta UX y tiempo de confirmación.</div></div>
    <div class="pi crit"><div class="piq">Alcance contractual de MegaSoft</div><div class="pid">¿Lixie lo integra dentro de su alcance o FitNation contrata a MegaSoft por separado?</div></div>
    <div class="pi crit"><div class="piq">Proveedor de torniquetes</div><div class="pid">En el checklist EVO figura como "Proveedor" (no ejecutor). Identificar al proveedor real.</div></div>\`},
  'pendientes.tecnicos':{title:'Pendientes técnicos',bc:'Pendientes › Técnicos',html:\`
    <div class="pi"><div class="piq">Validación automática de menor de edad en Odoo</div><div class="pid">¿Frontend al ingresar fecha de nacimiento o lógica de negocio en Odoo?</div></div>
    <div class="pi"><div class="piq">Diferenciación corporativo vs individual en el sistema</div><div class="pid">¿El usuario lo selecciona manualmente o el código corporativo lo activa automáticamente?</div></div>
    <div class="pi"><div class="piq">Sección T.I. del checklist vacía (ítems 1–5)</div><div class="pid">Requerimientos técnicos sin contenido. Deben completarse antes de la parametrización.</div></div>
    <div class="pi"><div class="piq">Flujo técnico completo del Upgrade de plan</div><div class="pid">¿Descuento sobre diferencia? ¿Los días restantes se acreditan al nuevo plan?</div></div>\`},
  'pendientes.operativos':{title:'Pendientes operativos',bc:'Pendientes › Operativos',html:\`
    <div class="pi"><div class="piq">Tablero de seguimiento (Trello / Notion)</div><div class="pid">FitNation necesita ver en tiempo real qué está hecho, bloqueado o pendiente. ¿Quién lo configura?</div></div>
    <div class="pi"><div class="piq">¿Se muestran precios en la web pública?</div><div class="pid">Marketing debe definir si los precios son visibles antes de hacer clic en "Comprar".</div></div>\`},
  'resumen.flujo':{title:'Flujo completo — 12 pasos',bc:'Resumen › Flujo',html:\`
    <ul class="sl">
      <li class="si"><span class="sn">PRE</span><div class="st"><b>Pago en preventa</b> — cobro inmediato, membresía congelada en Odoo</div></li>
      <li class="si"><span class="sn">APE</span><div class="st"><b>Día de apertura</b> — Odoo activa en lote, EVO habilita todos los accesos</div></li>
      <li class="si"><span class="sn">01</span><div class="st"><b>Visita web</b> — usuario navega planes y hace clic en "Inscribirme"</div></li>
      <li class="si"><span class="sn">02</span><div class="st"><b>Registro</b> — datos personales, verificación de duplicados con EVO</div></li>
      <li class="si"><span class="sn">03</span><div class="st"><b>Selección de pago</b> — plan, código promo opcional, método de pago</div></li>
      <li class="si"><span class="sn">04</span><div class="st"><b>Cobro</b> — pasarela retorna APROBADO / RECHAZADO / PENDIENTE</div></li>
      <li class="si"><span class="sn">05</span><div class="st"><b>Activación EVO</b> — API REST. Contrato + credenciales + instrucciones facial</div></li>
      <li class="si"><span class="sn">06</span><div class="st"><b>Facial en kiosco</b> — biométrico capturado y vinculado al perfil</div></li>
      <li class="si"><span class="sn">07</span><div class="st"><b>Acceso diario</b> — torniquete verifica facial + membresía en tiempo real</div></li>
      <li class="si"><span class="sn">08</span><div class="st"><b>Ciclo de cobro</b> — avisos -5d, D0, D3. Sin cobro automático.</div></li>
      <li class="si"><span class="sn">09</span><div class="st"><b>Renovación manual</b> — portal, app o POS. Acceso reactivado de inmediato.</div></li>
      <li class="si"><span class="sn">10</span><div class="st"><b>Suspensión por mora</b> — días 3–5. Acceso bloqueado automáticamente.</div></li>
      <li class="si"><span class="sn">11</span><div class="st"><b>Cancelación</b> — acceso revocado. Historial conservado en ambos sistemas.</div></li>
    </ul>\`},
  'resumen.comms':{title:'Comunicaciones automáticas — 13 emails',bc:'Resumen › Comunicaciones',html:\`
    <div class="tw"><table><thead><tr><th>Evento</th><th>Origen</th></tr></thead><tbody>
      ${DATA.comunicaciones.map(c=>`<tr><td>${c.evento}</td><td>${c.origen}</td></tr>`).join('')}
    </tbody></table></div>\`},
};

let _sec=null;
function openSec(id,el){
  const same=_sec===id;
  document.querySelectorAll('.nav-item').forEach(i=>i.classList.remove('active','open'));
  document.querySelectorAll('.nav-sub').forEach(s=>s.classList.remove('open'));
  document.querySelectorAll('.nav-sub-item').forEach(s=>s.classList.remove('active'));
  if(same){_sec=null;document.getElementById('detailPanel').classList.add('hidden');document.getElementById('spEmpty').style.display='flex';document.getElementById('spContent').style.display='none';return;}
  _sec=id;el.classList.add('active','open');document.getElementById('sub-'+id).classList.add('open');
  const d=SECS[id];if(!d)return;
  document.getElementById('detailPanel').classList.remove('hidden');
  document.getElementById('dBC').textContent='FitNation · v${DATA.version}';
  document.getElementById('dTitle').textContent=d.title;
  document.getElementById('dDesc').textContent=d.desc;
  document.getElementById('dBody').innerHTML=d.cards.map(c=>\`<div class="card" onclick="openSub('${id ? '' : ''}'+arguments[0].replace(/openSub\\\\('\\\\w+',/,'').replace(/'\\\\)/,''),this)" onclick="openSub('${id}','${'' /*placeholder*/}'+'')" data-sub="\${c.id}" onclick="openSubC('${id}','\${c.id}',this)"><span class="card-tag \${c.tc}">\${c.tag}</span><div class="card-title">\${c.title}</div><div class="card-desc">\${c.desc}</div><span class="card-arr">›</span></div>\`).join('');
  // fix onclick with proper closure
  document.getElementById('dBody').querySelectorAll('.card').forEach(card=>{
    const sub=card.dataset.sub;
    card.onclick=()=>openSub(id,sub,card);
  });
  document.getElementById('spEmpty').style.display='flex';document.getElementById('spContent').style.display='none';
}
function openSub(secId,subId,cardEl){
  const key=secId+'.'+subId;const d=SUBS[key];
  document.querySelectorAll('.nav-sub-item').forEach(s=>s.classList.remove('active'));
  document.querySelectorAll('#sub-'+secId+' .nav-sub-item').forEach(s=>{if(s.getAttribute('onclick')&&s.getAttribute('onclick').includes("'"+subId+"'"))s.classList.add('active');});
  if(cardEl){document.querySelectorAll('.card').forEach(c=>c.classList.remove('active'));cardEl.classList.add('active');}
  if(!d)return;
  document.getElementById('spEmpty').style.display='none';document.getElementById('spContent').style.display='block';
  document.getElementById('spBC').textContent=d.bc;document.getElementById('spTitle').textContent=d.title;
  document.getElementById('spBody').innerHTML=d.html;
}
<\/script>
</body>
</html>`;
  fs.writeFileSync('/mnt/user-data/outputs/index.html', html);
  console.log(`✅ HTML generado: ${(html.length/1024).toFixed(0)} KB`);
}

// ══════════════════════════════════════════════════════════════════
// 2. PPTX — Presentación ejecutiva
// ══════════════════════════════════════════════════════════════════
async function buildPPTX() {
  const pptxgen = require('pptxgenjs');
  const pres = new pptxgen();
  pres.layout = 'LAYOUT_WIDE'; // 13.3 x 7.5
  pres.title = DATA.proyecto;
  pres.author = 'Director de TI · FitNation';

  const BRAND  = '1A3567';
  const BRANDM = '2B4FA0';
  const TEAL   = '0F6B5C';
  const AMBER  = 'B45309';
  const RED    = '991B1B';
  const GREEN  = '166534';
  const WHITE  = 'FFFFFF';
  const LGRAY  = 'F3F4F6';
  const DGRAY  = '374151';

  function titleSlide(s, title, sub, pill='') {
    s.background = { color: BRAND };
    s.addShape(pres.ShapeType.rect, { x:0, y:0, w:13.3, h:7.5, fill:{color:'1A3567'} });
    // accent line
    s.addShape(pres.ShapeType.rect, { x:0, y:6.8, w:13.3, h:0.7, fill:{color:BRANDM} });
    if(pill) s.addText(pill, { x:0.5, y:0.4, w:3, h:0.35, fontSize:9, fontFace:'Courier New', color:'AABBDD', bold:true });
    s.addText(title, { x:0.5, y:1.6, w:12, h:2.2, fontSize:38, fontFace:'Arial', color:WHITE, bold:true });
    s.addText(sub, { x:0.5, y:3.9, w:10, h:1, fontSize:16, fontFace:'Arial', color:'AABBDD' });
    s.addText(`${DATA.empresa} · ${DATA.fecha}`, { x:0.5, y:6.9, w:12, h:0.4, fontSize:9, fontFace:'Arial', color:'AABBDD' });
  }

  function sectionHeader(s, num, title, color=BRAND) {
    s.background = { color: LGRAY };
    s.addShape(pres.ShapeType.rect, { x:0, y:0, w:0.12, h:7.5, fill:{color:color} });
    s.addText(num, { x:0.3, y:0.3, w:1.2, h:0.5, fontSize:11, fontFace:'Courier New', color:color, bold:true });
    s.addText(title, { x:0.3, y:0.85, w:12, h:1.2, fontSize:32, fontFace:'Arial', color:'111827', bold:true });
  }

  function footer(s) {
    s.addText(`FitNation · ${DATA.proyecto} · v${DATA.version} · ${DATA.fecha}`, {
      x:0, y:7.2, w:13.3, h:0.3, fontSize:8, fontFace:'Arial', color:'9CA3AF', align:'center'
    });
  }

  // ── PORTADA ──
  { const s = pres.addSlide();
    titleSlide(s, DATA.proyecto, `Ecosistema Odoo – EVO/W12 · Ciclo completo del socio`, `v${DATA.version}`);
  }

  // ── AGENDA ──
  { const s = pres.addSlide();
    sectionHeader(s, 'AGENDA', 'Contenido de la presentación');
    const items = ['01  Actores del ecosistema','02  Planes y formas de pago','03  Flujo de inscripción del socio','04  Preventa y apertura de sede','05  Ciclo de cobro y mora','06  Módulo de marketing','07  Pendientes para Lixie'];
    s.addText(items.map((t,i)=>({text:t,options:{bullet:false,breakLine:i<items.length-1,color:i===0?BRAND:DGRAY}})),
      {x:1.5,y:1.8,w:10,h:4.5,fontSize:18,fontFace:'Arial',lineSpacingMultiple:1.6});
    footer(s);
  }

  // ── ACTORES ──
  { const s = pres.addSlide();
    sectionHeader(s, '01', 'Actores del ecosistema');
    const cols = [
      {x:0.3, color:BRAND, label:'ODOO (Lixie)', sub:'Backoffice · Núcleo del negocio', items:['Web y e-commerce','POS y caja en sede','Pasarelas de pago','Facturación fiscal VE','Contabilidad','Suscripciones','Módulo de marketing']},
      {x:4.65, color:TEAL, label:'EVO / W12', sub:'Gestión operativa', items:['Torniquetes · acceso físico','Biométrico facial','App del socio','Reserva de clases','Contratos y credenciales','Seguimiento deportivo','Multisede · Venezuela']},
      {x:9.0, color:AMBER, label:'MegaSoft', sub:'Pasarela complementaria', items:['Cashea (pago digital VE)','Pago Móvil C2P','Métodos no nativos Odoo','Notifica cobros a Odoo','⚠ Alcance pendiente','confirmar contractual','con Lixie o FitNation']},
    ];
    cols.forEach(c=>{
      s.addShape(pres.ShapeType.rect, {x:c.x,y:1.4,w:4.15,h:5.7,fill:{color:c.color},line:{color:c.color},rounding:true});
      s.addText(c.label, {x:c.x+0.15,y:1.5,w:3.85,h:0.45,fontSize:14,fontFace:'Arial',color:WHITE,bold:true});
      s.addText(c.sub,   {x:c.x+0.15,y:1.95,w:3.85,h:0.3,fontSize:9,fontFace:'Arial',color:'CCDDF8'});
      s.addShape(pres.ShapeType.rect,{x:c.x,y:2.3,w:4.15,h:4.8,fill:{color:'F9FAFB'},line:{color:'E5E7EB'}});
      s.addText(c.items.map((t,i)=>({text:'→  '+t,options:{bullet:false,breakLine:i<c.items.length-1}})),
        {x:c.x+0.15,y:2.45,w:3.85,h:4.5,fontSize:11,fontFace:'Arial',color:DGRAY,lineSpacingMultiple:1.5});
    });
    s.addText('API REST  ←→  Odoo notifica a EVO activo  |  EVO notifica a Odoo via Webhook pasivo', {x:0.3,y:7.1,w:12.7,h:0.3,fontSize:9,fontFace:'Courier New',color:BRANDM,align:'center'});
    footer(s);
  }

  // ── PLANES Y PAGOS ──
  { const s = pres.addSlide();
    sectionHeader(s, '02', 'Planes y formas de pago');
    // Planes
    s.addText('Membresías disponibles', {x:0.3,y:1.6,w:6,h:0.35,fontSize:11,fontFace:'Arial',color:DGRAY,bold:true});
    DATA.planes.forEach((p,i)=>{
      const x = 0.3 + i*2.4;
      s.addShape(pres.ShapeType.rect,{x,y:2.05,w:2.25,h:1.4,fill:{color:i===2?BRAND:'F3F4F6'},line:{color:i===2?BRAND:'E5E7EB'},rounding:true});
      s.addText(p.nombre, {x,y:2.1,w:2.25,h:0.55,fontSize:12,fontFace:'Arial',color:i===2?WHITE:BRAND,bold:true,align:'center'});
      s.addText(`${p.dias} días`, {x,y:2.65,w:2.25,h:0.35,fontSize:10,fontFace:'Courier New',color:i===2?'CCDDF8':DGRAY,align:'center'});
    });
    // Pagos
    s.addText('Formas de pago', {x:0.3,y:3.7,w:6,h:0.35,fontSize:11,fontFace:'Arial',color:DGRAY,bold:true});
    const headers = ['Método','Online','Sede','Procesador'];
    const colW = [3.5,1.6,1.6,2.2];
    const startX = [0.3,3.8,5.4,7.0];
    headers.forEach((h,i)=>{
      s.addShape(pres.ShapeType.rect,{x:startX[i],y:4.1,w:colW[i],h:0.4,fill:{color:BRAND}});
      s.addText(h,{x:startX[i]+0.05,y:4.12,w:colW[i]-0.1,h:0.35,fontSize:10,fontFace:'Arial',color:WHITE,bold:true});
    });
    DATA.formasPago.forEach((p,r)=>{
      const y=4.55+r*0.44;
      const fill=r%2===0?'FFFFFF':'F9FAFB';
      const row=[p.nombre,p.online?'✓':'—',p.sede?'✓':'—',p.procesador];
      row.forEach((v,i)=>{
        s.addShape(pres.ShapeType.rect,{x:startX[i],y,w:colW[i],h:0.4,fill:{color:fill},line:{color:'E5E7EB'}});
        s.addText(v,{x:startX[i]+0.05,y:y+0.05,w:colW[i]-0.1,h:0.3,fontSize:10,fontFace:'Arial',color:DGRAY});
      });
    });
    s.addText('⚠  Cobro NO es automático — el socio debe renovar manualmente.', {x:0.3,y:7.1,w:12,h:0.3,fontSize:9,fontFace:'Arial',color:AMBER,bold:true});
    footer(s);
  }

  // ── FLUJO DE INSCRIPCIÓN ──
  { const s = pres.addSlide();
    sectionHeader(s, '03', 'Flujo de inscripción del socio');
    DATA.flujoInscripcion.forEach((step,i)=>{
      const x = 0.3 + (i%4)*3.2;
      const y = i<4 ? 1.6 : 4.3;
      const color = [BRAND,BRAND,BRAND,BRAND,TEAL,TEAL,TEAL][i]||BRAND;
      s.addShape(pres.ShapeType.rect,{x,y,w:3.0,h:2.2,fill:{color:'F9FAFB'},line:{color:color},rounding:true});
      s.addShape(pres.ShapeType.rect,{x,y,w:3.0,h:0.45,fill:{color:color},rounding:true});
      s.addText(step.num,{x,y:y+0.02,w:3.0,h:0.4,fontSize:10,fontFace:'Courier New',color:WHITE,bold:true,align:'center'});
      s.addText(step.titulo,{x:x+0.1,y:y+0.5,w:2.8,h:0.45,fontSize:11,fontFace:'Arial',color:'111827',bold:true});
      s.addText(step.desc,{x:x+0.1,y:y+0.98,w:2.8,h:1.1,fontSize:9,fontFace:'Arial',color:DGRAY,lineSpacingMultiple:1.4});
      s.addText(`[${step.sistema}]`,{x:x+0.1,y:y+2.0,w:2.8,h:0.18,fontSize:8,fontFace:'Courier New',color:color.length===6?color:BRAND});
    });
    s.addText('🔒  Todo bloqueo (mora, cancelación, suspensión) = acceso facial bloqueado INMEDIATA y AUTOMÁTICAMENTE.', {x:0.3,y:7.1,w:12.7,h:0.3,fontSize:9,fontFace:'Arial',color:RED,bold:true});
    footer(s);
  }

  // ── PREVENTA ──
  { const s = pres.addSlide();
    sectionHeader(s, '04', 'Preventa y apertura de sede', AMBER);
    s.addShape(pres.ShapeType.rect,{x:0.3,y:1.55,w:12.7,h:1.1,fill:{color:'FEF3C7'},line:{color:AMBER},rounding:true});
    s.addText('REQUISITO FUNCIONAL OBLIGATORIO',{x:0.5,y:1.6,w:4,h:0.35,fontSize:9,fontFace:'Courier New',color:AMBER,bold:true});
    s.addText(DATA.preventa.descripcion,{x:0.5,y:1.96,w:12.3,h:0.6,fontSize:12,fontFace:'Arial',color:'92400E'});
    // Tabla escenarios
    const headers=['Escenario','Cobro','Estado Odoo','EVO activa'];
    const cw=[4.0,2.2,2.5,2.5];const sx=[0.3,4.3,6.5,9.0];
    headers.forEach((h,i)=>{s.addShape(pres.ShapeType.rect,{x:sx[i],y:2.9,w:cw[i],h:0.4,fill:{color:BRAND}});s.addText(h,{x:sx[i]+0.05,y:2.92,w:cw[i]-0.1,h:0.35,fontSize:10,fontFace:'Arial',color:WHITE,bold:true});});
    DATA.preventa.escenarios.forEach((e,r)=>{
      const y=3.35+r*0.44;const fill=r%2===0?'FFFFFF':'F9FAFB';
      const row=[e.caso,e.cobro,e.odoo,e.evo];
      row.forEach((v,i)=>{s.addShape(pres.ShapeType.rect,{x:sx[i],y,w:cw[i],h:0.4,fill:{color:fill},line:{color:'E5E7EB'}});s.addText(v,{x:sx[i]+0.05,y:y+0.05,w:cw[i]-0.1,h:0.3,fontSize:10,fontFace:'Arial',color:DGRAY});});
    });
    s.addText('✅ Regla clave: el plan de N días comienza a contar desde el día de apertura. El socio no pierde ni un día.',{x:0.3,y:5.55,w:12.7,h:0.4,fontSize:11,fontFace:'Arial',color:GREEN,bold:true});
    s.addShape(pres.ShapeType.rect,{x:0.3,y:6.05,w:12.7,h:0.9,fill:{color:'FEE2E2'},line:{color:RED},rounding:true});
    s.addText(`⚠ PENDIENTE CRÍTICO: ${DATA.preventa.pendiente}`,{x:0.5,y:6.1,w:12.3,h:0.75,fontSize:10,fontFace:'Arial',color:RED,lineSpacingMultiple:1.4});
    footer(s);
  }

  // ── MORA ──
  { const s = pres.addSlide();
    sectionHeader(s, '05', 'Ciclo de cobro y escala de mora');
    const headers=['Días en mora','Estado Odoo','EVO / Torniquete','Acción del socio'];
    const cw=[2.8,3.0,3.5,3.0];const sx=[0.3,3.1,6.1,9.6];
    headers.forEach((h,i)=>{s.addShape(pres.ShapeType.rect,{x:sx[i],y:1.6,w:cw[i],h:0.42,fill:{color:BRAND}});s.addText(h,{x:sx[i]+0.05,y:1.62,w:cw[i]-0.1,h:0.37,fontSize:10,fontFace:'Arial',color:WHITE,bold:true});});
    const rowColors=['FFFFFF','FFFBEB','FEF3C7','FEE2E2'];
    DATA.moraEscala.forEach((e,r)=>{
      const y=2.07+r*0.52;const fill=rowColors[r]||'FFFFFF';
      const row=[e.dias,e.odoo,e.evo,e.accion];
      row.forEach((v,i)=>{s.addShape(pres.ShapeType.rect,{x:sx[i],y,w:cw[i],h:0.48,fill:{color:fill},line:{color:'E5E7EB'}});s.addText(v,{x:sx[i]+0.05,y:y+0.06,w:cw[i]-0.1,h:0.36,fontSize:10,fontFace:'Arial',color:DGRAY,bold:r>=2&&i===2});});
    });
    s.addShape(pres.ShapeType.rect,{x:0.3,y:4.25,w:12.7,h:1.05,fill:{color:'FEE2E2'},line:{color:RED},rounding:true});
    s.addText('🔒 REGLA CRÍTICA',{x:0.5,y:4.3,w:3,h:0.35,fontSize:10,fontFace:'Arial',color:RED,bold:true});
    s.addText('Cualquier suspensión o cancelación (mora automática, administrativa o webhook desde EVO) bloquea el acceso facial de forma INMEDIATA y AUTOMÁTICA. Sin excepciones.',{x:0.5,y:4.65,w:12.3,h:0.55,fontSize:10,fontFace:'Arial',color:RED,lineSpacingMultiple:1.3});
    s.addShape(pres.ShapeType.rect,{x:0.3,y:5.5,w:12.7,h:1.2,fill:{color:'E8EEF8'},line:{color:BRAND},rounding:true});
    s.addText('Notificaciones automáticas al socio',{x:0.5,y:5.55,w:5,h:0.35,fontSize:10,fontFace:'Arial',color:BRAND,bold:true});
    s.addText([
      {text:'−5 días → Aviso de vencimiento próximo + link de renovación',options:{breakLine:true}},
      {text:'Día 0 → Plan vencido + link de pago',options:{breakLine:true}},
      {text:'Día 3 → Último aviso antes de suspensión de acceso',options:{breakLine:false}},
    ],{x:0.5,y:5.92,w:12.3,h:0.7,fontSize:10,fontFace:'Arial',color:DGRAY,lineSpacingMultiple:1.4});
    footer(s);
  }

  // ── MARKETING ──
  { const s = pres.addSlide();
    sectionHeader(s, '06', 'Módulo de mercadeo y promociones');
    const promoColors=[BRAND,TEAL,AMBER,GREEN];
    DATA.promociones.forEach((p,i)=>{
      const x=0.3+(i%2)*6.4;const y=i<2?1.6:4.2;
      s.addShape(pres.ShapeType.rect,{x,y,w:6.1,h:2.3,fill:{color:'F9FAFB'},line:{color:promoColors[i]},rounding:true});
      s.addShape(pres.ShapeType.rect,{x,y,w:6.1,h:0.45,fill:{color:promoColors[i]},rounding:true});
      s.addText(p.tipo,{x:x+0.12,y:y+0.04,w:5.86,h:0.37,fontSize:10,fontFace:'Courier New',color:WHITE,bold:true});
      s.addText(p.titulo,{x:x+0.12,y:y+0.52,w:5.86,h:0.45,fontSize:13,fontFace:'Arial',color:'111827',bold:true});
      s.addText(p.desc,{x:x+0.12,y:y+1.0,w:5.86,h:1.15,fontSize:10,fontFace:'Arial',color:DGRAY,lineSpacingMultiple:1.4});
    });
    s.addText('Estados: Borrador → Programada → Activa → Pausada → Finalizada',{x:0.3,y:7.0,w:12.7,h:0.3,fontSize:9,fontFace:'Arial',color:DGRAY,align:'center'});
    footer(s);
  }

  // ── PENDIENTES ──
  { const s = pres.addSlide();
    sectionHeader(s, '07', 'Pendientes para Lixie', RED);
    const criticos=DATA.pendientes.filter(p=>p.nivel==='CRÍTICO');
    const tecnicos=DATA.pendientes.filter(p=>p.nivel==='TÉCNICO');
    const operativos=DATA.pendientes.filter(p=>p.nivel==='OPERATIVO');
    [[criticos,'CRÍTICOS (bloquean entrega)',RED,0.3,'FEE2E2'],
     [tecnicos,'TÉCNICOS / DESARROLLO',AMBER,4.65,'FEF3C7'],
     [operativos,'OPERATIVOS',GREEN,9.0,'DCFCE7']].forEach(([items,label,color,x,bg])=>{
      s.addShape(pres.ShapeType.rect,{x,y:1.55,w:4.0,h:0.4,fill:{color:color}});
      s.addText(label,{x:x+0.08,y:1.57,w:3.84,h:0.36,fontSize:9,fontFace:'Arial',color:WHITE,bold:true});
      s.addShape(pres.ShapeType.rect,{x,y:1.95,w:4.0,h:5.0,fill:{color:bg},line:{color:color}});
      items.forEach((p,i)=>{
        s.addText(`? ${p.titulo}`,{x:x+0.1,y:1.98+i*1.0,w:3.8,h:0.35,fontSize:9,fontFace:'Arial',color:color,bold:true});
        s.addText(p.desc,{x:x+0.1,y:2.35+i*1.0,w:3.8,h:0.5,fontSize:8,fontFace:'Arial',color:DGRAY,lineSpacingMultiple:1.3});
        if(i<items.length-1)s.addShape(pres.ShapeType.line,{x:x+0.1,y:2.9+i*1.0,w:3.8,h:0,line:{color:'D1D5DB',width:0.5}});
      });
    });
    footer(s);
  }

  // ── CIERRE ──
  { const s = pres.addSlide();
    titleSlide(s,'Próximos pasos','1. Reunión de alineación con Lixie para validar alcances\n2. Confirmar pendientes críticos (preventa, Cashea, torniquetes)\n3. Activar tablero de seguimiento (Trello/Notion)\n4. Modelado final v2.0 para presentar a FitNation','CIERRE');
  }

  await pres.writeFile({ fileName: '/mnt/user-data/outputs/FitNation_Presentacion_v1.2.pptx' });
  console.log('✅ PPTX generado');
}

// ══════════════════════════════════════════════════════════════════
// 3. DOCX — Documento Word
// ══════════════════════════════════════════════════════════════════
async function buildDOCX() {
  const {
    Document,Packer,Paragraph,TextRun,Table,TableRow,TableCell,
    HeadingLevel,AlignmentType,BorderStyle,WidthType,ShadingType,
    VerticalAlign,LevelFormat,PageBreak
  } = require('docx');

  const BRAND_HEX  = '1A3567';
  const BRAND_LIGHT= 'E8EEF8';
  const TEAL_HEX   = '0F6B5C';
  const TEAL_LIGHT = 'E0F4F0';
  const AMBER_HEX  = 'B45309';
  const AMBER_LIGHT= 'FEF3C7';
  const RED_HEX    = '991B1B';
  const RED_LIGHT  = 'FEE2E2';
  const GREEN_HEX  = '166534';
  const GREEN_LIGHT= 'DCFCE7';
  const GRAY_HEX   = '374151';
  const GRAY_LIGHT = 'F3F4F6';
  const WHITE      = 'FFFFFF';

  const b = (style=BorderStyle.SINGLE,color='CCCCCC',size=1)=>({style,size,color});
  const borders = {top:b(),bottom:b(),left:b(),right:b()};
  const noBorder = {style:BorderStyle.NONE,size:0,color:'FFFFFF'};
  const noBorders = {top:noBorder,bottom:noBorder,left:noBorder,right:noBorder};

  const h1 = t => new Paragraph({spacing:{before:400,after:160},children:[new TextRun({text:t,bold:true,size:34,color:BRAND_HEX,font:'Arial'})]});
  const h2 = t => new Paragraph({spacing:{before:280,after:100},children:[new TextRun({text:t,bold:true,size:26,color:BRAND_HEX,font:'Arial'})]});
  const h3 = t => new Paragraph({spacing:{before:200,after:80},children:[new TextRun({text:t,bold:true,size:22,color:GRAY_HEX,font:'Arial'})]});
  const body = (t,opts={}) => new Paragraph({spacing:{before:60,after:60},children:[new TextRun({text:t,size:20,font:'Arial',...opts})]});
  const bullet = (t,bold=false) => new Paragraph({numbering:{reference:'bullets',level:0},spacing:{before:40,after:40},children:[new TextRun({text:t,size:20,font:'Arial',bold})]});
  const space = (before=120) => new Paragraph({spacing:{before,after:0},children:[new TextRun('')]});
  const divider = () => new Paragraph({spacing:{before:180,after:180},border:{bottom:{style:BorderStyle.SINGLE,size:4,color:'DDDDDD'}},children:[new TextRun('')]});
  const pageBreak = () => new Paragraph({children:[new PageBreak()]});

  const stepRow = (num,title,system,desc) => new Table({
    width:{size:9360,type:WidthType.DXA},columnWidths:[700,2600,2200,3860],
    rows:[new TableRow({children:[
      new TableCell({borders,width:{size:700,type:WidthType.DXA},shading:{fill:BRAND_HEX,type:ShadingType.CLEAR},margins:{top:80,bottom:80,left:100,right:100},verticalAlign:VerticalAlign.CENTER,children:[new Paragraph({alignment:AlignmentType.CENTER,children:[new TextRun({text:num,bold:true,size:18,color:WHITE,font:'Courier New'})]})]}),
      new TableCell({borders,width:{size:2600,type:WidthType.DXA},shading:{fill:WHITE,type:ShadingType.CLEAR},margins:{top:80,bottom:80,left:120,right:100},children:[new Paragraph({children:[new TextRun({text:title,bold:true,size:20,font:'Arial'})]})]}),
      new TableCell({borders,width:{size:2200,type:WidthType.DXA},shading:{fill:BRAND_LIGHT,type:ShadingType.CLEAR},margins:{top:80,bottom:80,left:120,right:100},children:[new Paragraph({children:[new TextRun({text:system,size:17,color:BRAND_HEX,font:'Courier New'})]})]}),
      new TableCell({borders,width:{size:3860,type:WidthType.DXA},shading:{fill:WHITE,type:ShadingType.CLEAR},margins:{top:80,bottom:80,left:120,right:100},children:[new Paragraph({children:[new TextRun({text:desc,size:19,font:'Arial',color:GRAY_HEX})]})]}),
    ]})]
  });

  const alertBox = (icon,title,lines,fill,titleColor) => new Table({
    width:{size:9360,type:WidthType.DXA},columnWidths:[400,8960],
    rows:[new TableRow({children:[
      new TableCell({borders:noBorders,width:{size:400,type:WidthType.DXA},shading:{fill,type:ShadingType.CLEAR},margins:{top:120,bottom:120,left:120,right:80},verticalAlign:VerticalAlign.CENTER,children:[new Paragraph({alignment:AlignmentType.CENTER,children:[new TextRun({text:icon,size:22,font:'Arial'})]})]}),
      new TableCell({borders:noBorders,width:{size:8960,type:WidthType.DXA},shading:{fill,type:ShadingType.CLEAR},margins:{top:100,bottom:100,left:140,right:120},children:[
        new Paragraph({children:[new TextRun({text:title,bold:true,size:20,font:'Arial',color:titleColor})]}),
        ...lines.map(l=>new Paragraph({spacing:{before:30,after:30},children:[new TextRun({text:l,size:19,font:'Arial'})]}))
      ]}),
    ]})]
  });

  const tblHeader = (...cols) => new TableRow({children:cols.map((c,i)=>new TableCell({
    borders,width:{size:Math.floor(9360/cols.length),type:WidthType.DXA},
    shading:{fill:BRAND_HEX,type:ShadingType.CLEAR},margins:{top:80,bottom:80,left:120,right:80},
    children:[new Paragraph({children:[new TextRun({text:c,bold:true,size:19,color:WHITE,font:'Arial'})]})]
  }))});

  const tblRow = (...vals) => new TableRow({children:vals.map(v=>new TableCell({
    borders,width:{size:Math.floor(9360/vals.length),type:WidthType.DXA},
    shading:{fill:WHITE,type:ShadingType.CLEAR},margins:{top:80,bottom:80,left:120,right:80},
    children:[new Paragraph({children:[new TextRun({text:v,size:19,font:'Arial',color:GRAY_HEX})]})]
  }))});

  const doc = new Document({
    numbering:{config:[{reference:'bullets',levels:[{level:0,format:LevelFormat.BULLET,text:'\u2022',alignment:AlignmentType.LEFT,style:{paragraph:{indent:{left:560,hanging:280}}}}]}]},
    sections:[{
      properties:{page:{size:{width:12240,height:15840},margin:{top:1440,right:1260,bottom:1440,left:1260}}},
      children:[
        // PORTADA
        space(1800),
        new Paragraph({alignment:AlignmentType.CENTER,spacing:{before:0,after:80},children:[new TextRun({text:'FITNATION',bold:true,size:72,color:BRAND_HEX,font:'Arial'})]}),
        new Paragraph({alignment:AlignmentType.CENTER,spacing:{before:0,after:200},children:[new TextRun({text:DATA.empresa,size:26,color:GRAY_HEX,font:'Arial'})]}),
        new Paragraph({alignment:AlignmentType.CENTER,border:{bottom:{style:BorderStyle.SINGLE,size:6,color:BRAND_HEX}},spacing:{before:0,after:320},children:[new TextRun('')]}),
        new Paragraph({alignment:AlignmentType.CENTER,spacing:{before:200,after:80},children:[new TextRun({text:'MODELADO DE PROCESOS',bold:true,size:44,color:BRAND_HEX,font:'Arial'})]}),
        new Paragraph({alignment:AlignmentType.CENTER,spacing:{before:0,after:80},children:[new TextRun({text:'Ecosistema Digital: Odoo – EVO/W12',size:26,color:GRAY_HEX,font:'Arial'})]}),
        new Paragraph({alignment:AlignmentType.CENTER,spacing:{before:200,after:80},children:[new TextRun({text:'Ciclo completo: Inscripción → Activación → Acceso → Cobro → Renovación → Cancelación',bold:true,size:24,color:BRAND_HEX,font:'Arial'})]}),
        new Paragraph({alignment:AlignmentType.CENTER,spacing:{before:1200,after:0},children:[new TextRun({text:`Versión ${DATA.version}  |  ${DATA.fecha}  |  Director de TI – FitNation`,size:18,color:GRAY_HEX,font:'Arial'})]}),
        pageBreak(),

        // 1. ACTORES
        h1('1. Actores del ecosistema'),
        body('Tres sistemas interconectados. Cada uno es dueño exclusivo de su dominio funcional.'),
        space(),
        new Table({width:{size:9360,type:WidthType.DXA},columnWidths:[3120,3120,3120],rows:[
          new TableRow({children:DATA.actores.map(a=>new TableCell({borders,width:{size:3120,type:WidthType.DXA},shading:{fill:a.color.replace('#',''),type:ShadingType.CLEAR},margins:{top:100,bottom:100,left:140,right:120},children:[new Paragraph({children:[new TextRun({text:a.nombre,bold:true,size:22,color:WHITE,font:'Arial'})]})]}))})  ,
          new TableRow({children:DATA.actores.map(a=>new TableCell({borders,width:{size:3120,type:WidthType.DXA},shading:{fill:WHITE,type:ShadingType.CLEAR},margins:{top:100,bottom:100,left:140,right:120},children:[
            ...a.items.map(i=>new Paragraph({numbering:{reference:'bullets',level:0},spacing:{before:30,after:30},children:[new TextRun({text:i,size:18,font:'Arial',color:GRAY_HEX})]})),
            new Paragraph({spacing:{before:80,after:0},children:[new TextRun({text:a.regla,size:17,italic:true,color:GRAY_HEX,font:'Arial'})]}),
          ]}))})
        ]}),
        space(80),
        alertBox('⇄','Integración bidireccional Odoo ↔ EVO',['Flujo A (API REST): Odoo llama a EVO para activar, suspender o cancelar membresías.','Flujo B (Webhook): EVO notifica a Odoo si ocurre una excepción operativa.','Flujo C (Setup): Sincronización inicial de catálogos, sucursales y planes.'],BRAND_LIGHT,BRAND_HEX),
        divider(),

        // 2. PLANES
        h1('2. Planes y membresías'),
        space(60),
        new Table({width:{size:9360,type:WidthType.DXA},columnWidths:[1872,1872,1872,1872,1872],rows:[
          tblHeader(...DATA.planes.map(p=>p.nombre)),
          tblRow(...DATA.planes.map(p=>`${p.dias} días`)),
        ]}),
        space(80),
        alertBox('⚠','Renovación manual obligatoria',['El cobro NO es automático/recurrente. El socio debe pagar manualmente para reactivar su membresía.'],AMBER_LIGHT,AMBER_HEX),
        divider(),

        // 3. FORMAS DE PAGO
        h1('3. Formas de pago'),
        space(60),
        new Table({width:{size:9360,type:WidthType.DXA},columnWidths:[2800,1680,1680,3200],rows:[
          tblHeader('Método','Online','Sede','Procesado por'),
          ...DATA.formasPago.map(p=>tblRow(p.nombre,p.online?'✓':'—',p.sede?'✓':'—',p.procesador)),
        ]}),
        divider(),

        // 4. PREVENTA
        h1('4. Requisito: Preventa y Plan Congelado'),
        body('Requisito funcional obligatorio confirmado en checklist de operaciones y plan F-PRY-26.'),
        space(80),
        alertBox('📄','Base documental',['Checklist Mercadeo ítem 4: "Reporte de ventas – Preventa" como requerimiento.','F-PRY-26 Semana 3: Landing Page con links de pago antes de apertura.','F-PRY-26 Semana 6: "Carga de clientes activos (Suscripciones vigentes)" el día de go-live.'],BRAND_LIGHT,BRAND_HEX),
        space(80),
        h3('Escenarios:'),
        new Table({width:{size:9360,type:WidthType.DXA},columnWidths:[2700,1800,2430,2430],rows:[
          tblHeader('Escenario','Cobro','Estado Odoo','EVO activa'),
          ...DATA.preventa.escenarios.map(e=>tblRow(e.caso,e.cobro,e.odoo,e.evo)),
        ]}),
        space(80),
        alertBox('?','Pendiente crítico para Lixie',[DATA.preventa.pendiente],AMBER_LIGHT,AMBER_HEX),
        divider(),

        // 5. FLUJO
        h1('5. Flujo completo de inscripción'),
        space(80),
        ...DATA.flujoInscripcion.map(s=>stepRow(s.num,s.titulo,s.sistema,s.desc)).flatMap(t=>[t,space(4)]),
        space(80),
        alertBox('🔒','Regla crítica de bloqueo',['Cualquier suspensión o cancelación bloquea el acceso facial de forma INMEDIATA y AUTOMÁTICA.','No hay excepciones operativas. Aplica para mora automática, suspensión administrativa o webhook desde EVO.'],RED_LIGHT,RED_HEX),
        divider(),

        // 6. MORA
        h1('6. Ciclo de cobro y escala de mora'),
        h3('Notificaciones automáticas:'),
        bullet('−5 días: aviso de vencimiento próximo + link de renovación'),
        bullet('Día 0: plan vencido + link de pago'),
        bullet('Día 3: último aviso antes de suspensión'),
        bullet('Días 3–5: acceso BLOQUEADO automáticamente'),
        space(80),
        new Table({width:{size:9360,type:WidthType.DXA},columnWidths:[2000,2787,2787,1786],rows:[
          tblHeader('Días en mora','Estado Odoo','EVO / Torniquete','Acción'),
          ...DATA.moraEscala.map(e=>tblRow(e.dias,e.odoo,e.evo,e.accion)),
        ]}),
        divider(),

        // 7. MARKETING
        h1('7. Módulo de mercadeo y promociones'),
        space(60),
        new Table({width:{size:9360,type:WidthType.DXA},columnWidths:[2340,2340,2340,2340],rows:[
          tblHeader(...DATA.promociones.map(p=>p.tipo)),
          new TableRow({children:DATA.promociones.map((p,i)=>new TableCell({borders,width:{size:2340,type:WidthType.DXA},shading:{fill:[BRAND_LIGHT,TEAL_LIGHT,AMBER_LIGHT,GREEN_LIGHT][i],type:ShadingType.CLEAR},margins:{top:100,bottom:100,left:120,right:100},children:[
            new Paragraph({children:[new TextRun({text:p.titulo,bold:true,size:20,font:'Arial'})]}),
            new Paragraph({spacing:{before:40},children:[new TextRun({text:p.desc,size:18,font:'Arial',color:GRAY_HEX})]}),
          ]}))}),
        ]}),
        divider(),

        // 8. COMUNICACIONES
        h1('8. Comunicaciones automáticas'),
        space(60),
        new Table({width:{size:9360,type:WidthType.DXA},columnWidths:[6240,1680,1440],rows:[
          tblHeader('Evento','Sistema origen','Destinatario'),
          ...DATA.comunicaciones.map(c=>tblRow(c.evento,c.origen,c.destinatario)),
        ]}),
        divider(),

        // 9. PENDIENTES
        pageBreak(),
        h1('9. Pendientes para Lixie'),
        space(80),
        ...DATA.pendientes.map(p=>alertBox(
          p.nivel==='CRÍTICO'?'🔴':p.nivel==='TÉCNICO'?'🟡':'🔵',
          `[${p.nivel}] ${p.titulo}`,[p.desc],
          p.nivel==='CRÍTICO'?RED_LIGHT:p.nivel==='TÉCNICO'?AMBER_LIGHT:BRAND_LIGHT,
          p.nivel==='CRÍTICO'?RED_HEX:p.nivel==='TÉCNICO'?AMBER_HEX:BRAND_HEX
        )).flatMap(a=>[a,space(6)]),

        // 10. RESUMEN
        pageBreak(),
        h1('10. Resumen ejecutivo del flujo'),
        space(60),
        new Table({width:{size:9360,type:WidthType.DXA},columnWidths:[720,3200,2600,2840],rows:[
          tblHeader('Paso','Acción','Sistema','Resultado'),
          tblRow('PRE','Pago en preventa','Odoo','Cobro inmediato · membresía congelada'),
          tblRow('APE','Activación masiva en apertura','Odoo → EVO (lote)','Todos los planes preventa activos'),
          ...DATA.flujoInscripcion.map(s=>tblRow(s.num,s.titulo,s.sistema,s.desc.substring(0,60)+'...')),
          tblRow('08','Ciclo de cobro y avisos','Odoo','Notificaciones -5d, D0, D3'),
          tblRow('09','Renovación manual','Odoo → EVO','Acceso reactivado de inmediato'),
          tblRow('10','Suspensión por mora','Odoo → EVO auto','Acceso bloqueado días 3–5'),
          tblRow('11','Cancelación definitiva','Odoo → EVO','Acceso revocado · historial conservado'),
        ]}),

        space(400),
        new Paragraph({alignment:AlignmentType.CENTER,border:{top:{style:BorderStyle.SINGLE,size:4,color:'DDDDDD'}},spacing:{before:200,after:80},children:[new TextRun({text:`FitNation – ${DATA.empresa}  |  Modelado de Procesos v${DATA.version}  |  ${DATA.fecha}`,size:16,color:GRAY_HEX,font:'Arial'})]}),
        new Paragraph({alignment:AlignmentType.CENTER,children:[new TextRun({text:'Documento confidencial · Uso interno · Director de TI',size:16,color:GRAY_HEX,font:'Arial'})]}),
      ]
    }]
  });

  const buf = await Packer.toBuffer(doc);
  fs.writeFileSync('/mnt/user-data/outputs/FitNation_Modelado_v1.2.docx', buf);
  console.log('✅ DOCX generado');
}

// ══════════════════════════════════════════════════════════════════
// MAIN — corre los tres builds
// ══════════════════════════════════════════════════════════════════
(async () => {
  console.log('🚀 FitNation build script iniciando...\n');
  buildHTML();
  await buildPPTX();
  await buildDOCX();
  console.log('\n✅ Todos los archivos generados:');
  console.log('  → /mnt/user-data/outputs/index.html');
  console.log('  → /mnt/user-data/outputs/FitNation_Presentacion_v1.2.pptx');
  console.log('  → /mnt/user-data/outputs/FitNation_Modelado_v1.2.docx');
  console.log('\n📝 Para actualizar: edita el objeto DATA en build_fitnation.js y corre: node build_fitnation.js');
})();