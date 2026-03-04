<!doctype html>
<html lang="fr">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Suivi Réactifs — labo</title>
<style>
:root{--bg:#071018;--card:#0f1720;--muted:#9aa6b2;--danger:#d14343;--ok:#16a34a}
*{box-sizing:border-box}body{margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial;background:var(--bg);color:#e6eef6;padding:14px}
h1{margin:0 0 6px;font-size:20px} .muted{color:var(--muted);font-size:13px;margin-bottom:10px}
.card{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));padding:12px;border-radius:10px;margin-bottom:12px;border:1px solid rgba(255,255,255,0.03)}
.row{display:flex;gap:8px;align-items:center}
label{display:block;font-size:13px;margin:6px 0;color:var(--muted)}
input,select,button{font-size:14px}
input[type=text],input[type=date],select{width:100%;padding:8px;border-radius:8px;border:1px solid rgba(255,255,255,0.03);background:#0b1620;color:#e6eef6}
button{padding:8px 10px;border-radius:8px;border:0;background:#1f2937;color:#fff;cursor:pointer}
button.primary{background:var(--ok)} button.warn{background:var(--danger)}
.grid{display:grid;gap:8px}
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
  <h1>Suivi réactifs / contrôles / calibrants</h1>
  <div class="muted">Ajoute fiche MLE via photo • Modifie / Supprime • Expirés en rouge</div>

  <div class="card">
    <div style="display:flex;gap:10px;flex-wrap:wrap">
      <div style="flex:1;min-width:160px">
        <label>Type</label>
        <select id="type">
          <option>Réactif</option>
          <option>Contrôle</option>
          <option>Calibrant</option>
        </select>
      </div>
      <div style="flex:1;min-width:160px">
        <label>Fiche MLE (texte)</label>
        <input id="mle" type="text" placeholder="Référence MLE / description">
      </div>
      <div style="flex:1;min-width:140px">
        <label>Numéro de lot</label>
        <input id="lot" type="text" placeholder="ex: L-2026-123">
      </div>
      <div style="flex:1;min-width:160px">
        <label>Date d'expiration</label>
        <input id="expiry" type="date">
      </div>
    </div>

    <label>Fiche MLE (photo)</label>
    <!-- accept camera on mobile: capture="environment" helps open rear camera -->
    <input id="photoInput" type="file" accept="image/*" capture="environment">

    <div class="controls">
      <button id="addBtn" class="primary">➕ Ajouter</button>
      <button id="exportCSV">⬇️ Exporter CSV</button>
      <button id="exportJSON">⬇️ Exporter JSON (avec images)</button>
      <button id="importJSON">⬆️ Importer JSON</button>
      <button id="clearAll" class="warn">Tout supprimer</button>
      <input id="search" placeholder="Rechercher MLE / lot" style="padding:8px;border-radius:8px;background:#07161d;color:#e6eef6;border:1px solid rgba(255,255,255,0.03)">
    </div>
  </div>

  <div class="card">
    <table class="table" id="table">
      <thead><tr><th>Type</th><th>MLE</th><th>Lot</th><th>Exp.</th><th>Photo</th><th></th></tr></thead>
      <tbody id="tbody"></tbody>
    </table>
    <div id="empty" class="small" style="display:none">Aucun enregistrement.</div>
  </div>

  <div class="modal" id="modal"><div class="modal-content" id="modalContent"></div></div>

  <div class="footer">Données stockées localement. Sauvegarde régulière recommandée (Exporter JSON).</div>

<script>
(function(){
  const KEY='labo_reactifs_v1';
  let db = JSON.parse(localStorage.getItem(KEY) || '[]');

  // elements
  const typeEl=document.getElementById('type'),mleEl=document.getElementById('mle'),lotEl=document.getElementById('lot'),expiryEl=document.getElementById('expiry'),
        photoInput=document.getElementById('photoInput'),addBtn=document.getElementById('addBtn'),tbody=document.getElementById('tbody'),
        empty=document.getElementById('empty'),search=document.getElementById('search'),
        exportCSV=document.getElementById('exportCSV'),exportJSON=document.getElementById('exportJSON'),
        importJSON=document.getElementById('importJSON'),clearAll=document.getElementById('clearAll'),
        modal=document.getElementById('modal'),modalContent=document.getElementById('modalContent');

  let editingIndex = null;
  let lastImageData = null; // base64 of selected image

  function saveDB(){ localStorage.setItem(KEY, JSON.stringify(db)); }

  function isExpired(dateStr){
    if(!dateStr) return false;
    const d = new Date(dateStr); d.setHours(23,59,59,999);
    return d < new Date();
  }

  function render(){
    tbody.innerHTML='';
    if(db.length===0){ empty.style.display='block'; return; } else empty.style.display='none';
    const q = (search.value||'').toLowerCase();
    db.forEach((row, i) => {
      if(q){
        const s=(row.type+' '+(row.mle||'')+' '+(row.lot||'')).toLowerCase();
        if(!s.includes(q)) return;
      }
      const tr = document.createElement('tr');
      if(isExpired(row.expiry)) tr.className='expired';
      tr.innerHTML = `
        <td>${row.type||''}</td>
        <td>${row.mle?escapeHTML(row.mle):''}</td>
        <td>${row.lot?escapeHTML(row.lot):''}</td>
        <td>${row.expiry?row.expiry:''}</td>
        <td>${row.photo?'<img src="'+row.photo+'" class="thumb" data-i="'+i+'">':''}</td>
        <td>
          <button data-edit="${i}">✏️</button>
          <button data-del="${i}" class="warn">🗑️</button>
        </td>`;
      tbody.appendChild(tr);
    });
    // attach listeners for edit/delete and image click
    tbody.querySelectorAll('button[data-edit]').forEach(b=>b.onclick=e=>{
      const i=+b.getAttribute('data-edit'); startEdit(i);
    });
    tbody.querySelectorAll('button[data-del]').forEach(b=>b.onclick=e=>{
      const i=+b.getAttribute('data-del'); if(confirm('Supprimer cet enregistrement ?')){ db.splice(i,1); saveDB(); render(); }
    });
    tbody.querySelectorAll('img.thumb').forEach(img=>{
      img.onclick=e=>{
        const idx=+img.getAttribute('data-i');
        openModal(db[idx].photo || '');
      };
    });
  }

  function startEdit(i){
    const r = db[i];
    editingIndex = i;
    typeEl.value = r.type || 'Réactif';
    mleEl.value = r.mle || '';
    lotEl.value = r.lot || '';
    expiryEl.value = r.expiry || '';
    lastImageData = r.photo || null;
    addBtn.textContent = '💾 Enregistrer modification';
    window.scrollTo({top:0,behavior:'smooth'});
  }

  function clearForm(){
    typeEl.value='Réactif'; mleEl.value=''; lotEl.value=''; expiryEl.value=''; photoInput.value=''; lastImageData=null;
    editingIndex=null; addBtn.textContent='➕ Ajouter';
  }

  function escapeHTML(s){ return (s||'').replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c])); }

  // handle photo input -> read as data URL (base64)
  photoInput.addEventListener('change', (ev)=>{
    const f = ev.target.files && ev.target.files[0];
    if(!f) return;
    const reader = new FileReader();
    reader.onload = e => { lastImageData = e.target.result; };
    reader.readAsDataURL(f);
  });

  addBtn.onclick = ()=>{
    const rec = {
      type: typeEl.value,
      mle: mleEl.value.trim(),
      lot: lotEl.value.trim(),
      expiry: expiryEl.value || '',
      photo: lastImageData || null,
      created: new Date().toISOString()
    };
    // validation minimal
    if(!rec.mle){ if(!confirm('La fiche MLE est vide. Continuer ?')===false) return; }
    if(editingIndex===null){
      db.unshift(rec); // latest on top
    } else {
      db[editingIndex] = rec;
    }
    saveDB(); clearForm(); render();
  };

  search.addEventListener('input', ()=>render());

  // export CSV (without images) - for reporting
  exportCSV.onclick = ()=>{
    let csv = 'type,mle,lot,expiry,created\\n';
    db.slice().reverse().forEach(r=>{
      csv += `${quoteCSV(r.type)},${quoteCSV(r.mle)},${quoteCSV(r.lot)},${quoteCSV(r.expiry)},${quoteCSV(r.created)}\\n`;
    });
    downloadBlob(csv, 'reactifs_labo.csv', 'text/csv');
  };

  // export JSON with images
  exportJSON.onclick = ()=>{
    const json = JSON.stringify(db);
    downloadBlob(json, 'reactifs_labo.json', 'application/json');
  };

  importJSON.onclick = ()=>{
    const inp = document.createElement('input');
    inp.type='file'; inp.accept='application/json';
    inp.onchange = e=>{
      const f = e.target.files[0];
      if(!f) return;
      const r = new FileReader();
      r.onload = ev => {
        try{
          const arr = JSON.parse(ev.target.result);
          if(!Array.isArray(arr)) throw 'Format incorrect';
          if(confirm('Importer et remplacer la base locale ? OK pour remplacer, Annuler pour ajouter au début')){
            db = arr;
          } else {
            db = arr.concat(db);
          }
          saveDB(); render(); alert('Import terminé');
        }catch(err){ alert('Impossible d\'importer : '+err); }
      };
      r.readAsText(f);
    };
    inp.click();
  };

  clearAll.onclick = ()=>{
    if(confirm('Supprimer toute la base locale ?')){ db=[]; saveDB(); render(); }
  };

  function quoteCSV(s){ if(s==null) s=''; return '"'+String(s).replace(/"/g,'""')+'"'; }
  function downloadBlob(content, name, type){
    const blob = new Blob([content], {type: type});
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = name;
    a.click();
    URL.revokeObjectURL(a.href);
  }

  // modal
  function openModal(img){
    modalContent.innerHTML = img ? '<img src="'+img+'" class="bigimg">' : '<div class="small">Pas d\'image</div>';
    modal.style.display='flex';
  }
  modal.onclick = (e)=>{ if(e.target===modal) modal.style.display='none'; }

  // initial render
  render();
})();
</script>
</body>
</html># Prog
