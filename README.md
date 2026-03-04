<!doctype html>
<html lang="fr">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Suivi Réactifs — Labo</title>
<style>
:root{--bg:#071018;--card:#0f1720;--muted:#9aa6b2;--danger:#d14343;--ok:#16a34a}
*{box-sizing:border-box}body{margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial;background:var(--bg);color:#e6eef6;padding:14px}
h1{margin:0 0 6px;font-size:20px}.muted{color:var(--muted);font-size:13px;margin-bottom:10px}
.card{background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(255,255,255,0.01));padding:12px;border-radius:10px;margin-bottom:12px;border:1px solid rgba(255,255,255,0.03)}
label{display:block;font-size:13px;margin:6px 0;color:var(--muted)}
input,select,button{font-size:14px}
input[type=text],input[type=date],select{width:100%;padding:8px;border-radius:8px;border:1px solid rgba(255,255,255,0.03);background:#0b1620;color:#e6eef6}
button{padding:8px 10px;border-radius:8px;border:0;background:#1f2937;color:#fff;cursor:pointer}
button.primary{background:var(--ok)}button.warn{background:var(--danger)}
.table{width:100%;border-collapse:collapse;margin-top:8px}
.table th,.table td{padding:8px;border-bottom:1px solid rgba(255,255,255,0.03);font-size:13px;text-align:left}
.thumb{width:56px;height:40px;object-fit:cover;border-radius:6px;border:1px solid rgba(255,255,255,0.03)}
.expired{background:rgba(209,67,67,0.08)}
.controls{display:flex;gap:8px;flex-wrap:wrap;margin-top:8px}
.small{color:var(--muted);font-size:12px}
.modal{position:fixed;inset:0;display:none;align-items:center;justify-content:center;background:rgba(0,0,0,0.6);z-index:40}
.modal-content{background:#071018;padding:12px;border-radius:10px;max-width:95%;max-height:90%;overflow:auto}
.bigimg{max-width:100%;height:auto;border-radius:6px}
.footer{color:var(--muted);font-size:12px;text-align:center;margin-top:10px}
@media(min-width:900px){body{padding:28px}}
</style>
</head>
<body>
<h1>Suivi Réactifs / Contrôles / Calibrants</h1>
<div class="muted">Nom réactif obligatoire • Fiche MLE photo • Expirés en rouge</div>

<div class="card">
  <div style="display:flex;gap:10px;flex-wrap:wrap">
    <div style="flex:1;min-width:150px">
      <label>Type</label>
      <select id="type">
        <option>Réactif</option><option>Contrôle</option><option>Calibrant</option>
      </select>
    </div>
    <div style="flex:1;min-width:160px">
      <label>Nom du réactif *</label>
      <input id="nom" type="text" placeholder="ex: Glucose GOD-PAP">
    </div>
    <div style="flex:1;min-width:160px">
      <label>Fiche MLE (texte)</label>
      <input id="mle" type="text" placeholder="Référence ou code MLE">
    </div>
    <div style="flex:1;min-width:130px">
      <label>Numéro de lot</label>
      <input id="lot" type="text" placeholder="ex: L-2026-123">
    </div>
    <div style="flex:1;min-width:140px">
      <label>Date d'expiration</label>
      <input id="expiry" type="date">
    </div>
  </div>

  <label>Fiche MLE (photo)</label>
  <input id="photoInput" type="file" accept="image/*" capture="environment">

  <div class="controls">
    <button id="addBtn" class="primary">➕ Ajouter</button>
    <button id="exportCSV">⬇️ Exporter CSV</button>
    <button id="exportJSON">⬇️ Exporter JSON</button>
    <button id="importJSON">⬆️ Importer JSON</button>
    <button id="clearAll" class="warn">Tout supprimer</button>
    <input id="search" placeholder="Rechercher nom / MLE / lot" style="padding:8px;border-radius:8px;background:#07161d;color:#e6eef6;border:1px solid rgba(255,255,255,0.03)">
  </div>
</div>

<div class="card">
  <table class="table" id="table">
    <thead><tr><th>Type</th><th>Nom</th><th>MLE</th><th>Lot</th><th>Exp.</th><th>Photo</th><th></th></tr></thead>
    <tbody id="tbody"></tbody>
  </table>
  <div id="empty" class="small" style="display:none">Aucun enregistrement.</div>
</div>

<div class="modal" id="modal"><div class="modal-content" id="modalContent"></div></div>
<div class="footer">Données stockées localement. Sauvegarde recommandée (Exporter JSON).</div>

<script>
(function(){
  const KEY='labo_reactifs_v2';
  let db=JSON.parse(localStorage.getItem(KEY)||'[]');
  const qS=id=>document.getElementById(id);
  const typeEl=qS('type'),nomEl=qS('nom'),mleEl=qS('mle'),lotEl=qS('lot'),expiryEl=qS('expiry'),
        photoInput=qS('photoInput'),addBtn=qS('addBtn'),tbody=qS('tbody'),empty=qS('empty'),
        search=qS('search'),exportCSV=qS('exportCSV'),exportJSON=qS('exportJSON'),
        importJSON=qS('importJSON'),clearAll=qS('clearAll'),modal=qS('modal'),modalContent=qS('modalContent');
  let editing=null,lastImage=null;

  const save=()=>localStorage.setItem(KEY,JSON.stringify(db));
  const expired=d=>d&&new Date(d)<new Date();
  const html=s=>(s||'').replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));

  function render(){
    tbody.innerHTML='';
    if(!db.length){empty.style.display='block';return}else empty.style.display='none';
    const q=(search.value||'').toLowerCase();
    db.forEach((r,i)=>{
      if(q&&!`${r.nom} ${r.mle} ${r.lot}`.toLowerCase().includes(q))return;
      const tr=document.createElement('tr');
      if(expired(r.expiry))tr.className='expired';
      tr.innerHTML=`<td>${r.type}</td
