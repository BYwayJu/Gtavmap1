[index.html](https://github.com/user-attachments/files/27319311/index.html)
<!doctype html>
<html lang="zh-Hant">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>私人 GTA 地圖標點</title>
  <style>
    *{box-sizing:border-box}
    body{margin:0;background:#0b1220;color:#fff;font-family:system-ui,-apple-system,"Noto Sans TC",sans-serif;overflow:hidden}
    .app{display:grid;grid-template-columns:320px 1fr;height:100vh}
    .side{padding:16px;background:#111827;border-right:1px solid #334155;overflow:auto;z-index:5}
    .mapWrap{position:relative;overflow:hidden;background:#263b43;touch-action:none;user-select:none}
    .map{position:absolute;left:0;top:0;width:100%;height:100%;background:url('map-dark.jpg') center/contain no-repeat;transform-origin:0 0;cursor:grab}
    .map.dragging{cursor:grabbing}
    .marker{position:absolute;transform:translate(-50%,-100%);font-size:26px;border:0;background:transparent;cursor:pointer;filter:drop-shadow(0 3px 5px #000);z-index:2;width:auto;margin:0;padding:0}
    input,textarea,select,button{width:100%;border-radius:12px;border:1px solid #334155;background:#0f172a;color:white;padding:10px;margin:6px 0}
    button{cursor:pointer;background:#0891b2;border:0;font-weight:700}.btn2{background:#334155}.danger{background:#991b1b}
    .row{display:flex;gap:8px}.row>*{flex:1}
    .card{background:#0f172a;border:1px solid #334155;border-radius:14px;padding:12px;margin:10px 0}
    .small{font-size:12px;color:#cbd5e1}.modal{position:fixed;inset:0;background:rgba(0,0,0,.7);display:none;place-items:center;padding:16px;z-index:10}.box{width:min(420px,100%);background:#111827;border:1px solid #334155;border-radius:18px;padding:18px}.list button{text-align:left;background:#1e293b}.ok{color:#86efac}.warn{color:#fde68a}
    @media(max-width:800px){.app{grid-template-columns:1fr;grid-template-rows:auto 1fr}.side{max-height:48vh}}
  </style>
</head>
<body>
<div class="app">
  <aside class="side">
    <h2>🗺️ GTA 私人地圖</h2>
    <div class="card">
      <b>GitHub 資料庫設定</b>
      <p class="small">repo 裡要有 <code>markers.json</code>。朋友要能修改，請把他加成 Collaborator，並讓他自己填 GitHub Token。</p>
      <input id="owner" placeholder="GitHub 帳號，例如 BYwayJu">
      <input id="repo" placeholder="Repo 名稱，例如 Gtavmap1">
      <input id="path" value="markers.json" placeholder="資料檔路徑 markers.json">
      <input id="branch" value="main" placeholder="分支 main">
      <input id="token" type="password" placeholder="GitHub Token，只存在自己瀏覽器">
      <div class="row"><button onclick="saveConfig()">儲存設定</button><button class="btn2" onclick="loadFromGitHub()">讀取</button></div>
      <div id="status" class="small warn">尚未連線</div>
    </div>

    <div class="card">
      <b>新增標點</b>
      <input id="player" placeholder="你的名字">
      <select id="type">
        <option value="🔴">🔴 搶劫</option><option value="🟢">🟢 毒品</option><option value="🔵">🔵 NPC</option><option value="🟡">🟡 倉庫</option><option value="⚫">⚫ 任務</option><option value="⭐">⭐ 自訂</option>
      </select>
      <input id="customIcon" placeholder="自訂圖案，例如 💎，沒填就用上面">
      <div class="row"><button onclick="zoomIn()">放大</button><button onclick="zoomOut()">縮小</button><button class="btn2" onclick="resetView()">重置</button></div>
      <p class="small">滑鼠滾輪可縮放；按住地圖可拖曳；點地圖新增標點；點標點或清單可編輯。</p>
    </div>

    <div class="card"><b>標點清單</b><div id="count" class="small"></div><div id="list" class="list"></div></div>
  </aside>
  <main id="wrap" class="mapWrap"><div id="map" class="map"></div></main>
</div>

<div id="modal" class="modal">
  <div class="box">
    <h3 id="modalTitle">標點</h3>
    <input id="mName" placeholder="名稱，例如 海戰 NPC">
    <textarea id="mNote" placeholder="備註，例如 需要通行證"></textarea>
    <div class="row"><button onclick="saveMarker()">儲存到 GitHub</button><button class="btn2" onclick="closeModal()">取消</button></div>
    <button id="deleteBtn" class="danger" onclick="deleteMarker()">刪除</button>
  </div>
</div>

<script>
let markers=[], editing=null, pendingPoint=null, sha=null;
let zoom=1, panX=0, panY=0;
let dragging=false, dragMoved=false, startX=0, startY=0, startPanX=0, startPanY=0;
const $=id=>document.getElementById(id);
const wrap=$('wrap'), map=$('map');
function enc(s){return btoa(unescape(encodeURIComponent(s)))}
function dec(s){return decodeURIComponent(escape(atob(s)))}
function cfg(){return {owner:$('owner').value.trim(),repo:$('repo').value.trim(),path:$('path').value.trim()||'markers.json',branch:$('branch').value.trim()||'main',token:$('token').value.trim()}}
function applyTransform(){map.style.transform=`translate(${panX}px, ${panY}px) scale(${zoom})`}
function zoomAt(clientX, clientY, nextZoom){const rect=wrap.getBoundingClientRect();const oldZoom=zoom;nextZoom=Math.max(0.5,Math.min(5,nextZoom));const mx=clientX-rect.left;const my=clientY-rect.top;panX=mx-(mx-panX)*(nextZoom/oldZoom);panY=my-(my-panY)*(nextZoom/oldZoom);zoom=nextZoom;applyTransform()}
function screenToPercent(clientX, clientY){const rect=wrap.getBoundingClientRect();return {x:((clientX-rect.left-panX)/zoom/rect.width)*100,y:((clientY-rect.top-panY)/zoom/rect.height)*100}}

wrap.addEventListener('wheel',e=>{e.preventDefault();zoomAt(e.clientX,e.clientY,zoom*(e.deltaY<0?1.12:0.88))},{passive:false});
wrap.addEventListener('pointerdown',e=>{if(e.target.classList.contains('marker'))return;dragging=true;dragMoved=false;startX=e.clientX;startY=e.clientY;startPanX=panX;startPanY=panY;map.classList.add('dragging');wrap.setPointerCapture(e.pointerId)});
wrap.addEventListener('pointermove',e=>{if(!dragging)return;const dx=e.clientX-startX,dy=e.clientY-startY;if(Math.abs(dx)+Math.abs(dy)>5)dragMoved=true;panX=startPanX+dx;panY=startPanY+dy;applyTransform()});
wrap.addEventListener('pointerup',e=>{if(!dragging)return;dragging=false;map.classList.remove('dragging');if(!dragMoved){const p=screenToPercent(e.clientX,e.clientY);if(p.x>=0&&p.x<=100&&p.y>=0&&p.y<=100){pendingPoint=p;editing=null;$('mName').value='';$('mNote').value='';$('deleteBtn').style.display='none';$('modalTitle').textContent='新增標點';$('modal').style.display='grid'}}});

function saveConfig(){['owner','repo','path','branch','token','player'].forEach(id=>localStorage.setItem('gta_'+id,$(id).value)); setStatus('設定已儲存','ok')}
function loadConfig(){['owner','repo','path','branch','token','player'].forEach(id=>$(id).value=localStorage.getItem('gta_'+id)||$(id).value)}
function setStatus(t,cls='warn'){$('status').className='small '+cls;$('status').textContent=t}
async function apiGet(){const c=cfg(); const r=await fetch(`https://api.github.com/repos/${c.owner}/${c.repo}/contents/${c.path}?ref=${c.branch}`,{headers:{Authorization:`Bearer ${c.token}`,Accept:'application/vnd.github+json'}}); if(!r.ok) throw new Error(await r.text()); return r.json()}
async function apiPut(message){const c=cfg(); const body={message,content:enc(JSON.stringify(markers,null,2)),branch:c.branch}; if(sha) body.sha=sha; const r=await fetch(`https://api.github.com/repos/${c.owner}/${c.repo}/contents/${c.path}`,{method:'PUT',headers:{Authorization:`Bearer ${c.token}`,Accept:'application/vnd.github+json'},body:JSON.stringify(body)}); if(!r.ok) throw new Error(await r.text()); return r.json()}
async function loadFromGitHub(){try{saveConfig(); setStatus('讀取中...'); const data=await apiGet(); sha=data.sha; markers=JSON.parse(dec(data.content.replace(/\n/g,''))||'[]'); render(); setStatus('已從 GitHub 讀取','ok')}catch(e){console.error(e);setStatus('讀取失敗：請確認 repo/token/markers.json 權限')}}
async function pushGitHub(msg){try{setStatus('寫入 GitHub 中...'); const data=await apiPut(msg); sha=data.content.sha; render(); setStatus('已儲存到 GitHub','ok')}catch(e){console.error(e); setStatus('寫入失敗：Token 需要 Contents Read and Write 權限')}}
function render(){map.querySelectorAll('.marker').forEach(x=>x.remove()); markers.forEach(m=>{const b=document.createElement('button');b.className='marker';b.style.left=m.x+'%';b.style.top=m.y+'%';b.textContent=m.icon;b.title=m.name;b.onclick=e=>{e.stopPropagation();openEdit(m.id)};map.appendChild(b)}); $('count').textContent=`共 ${markers.length} 個標點`; $('list').innerHTML=markers.map(m=>`<button onclick="openEdit('${m.id}')"><b>${m.icon} ${escapeHtml(m.name)}</b><div class="small">${escapeHtml(m.owner||'匿名')}｜${escapeHtml(m.note||'')}</div></button>`).join('')}
function escapeHtml(s){return String(s).replace(/[&<>"]/g,a=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[a]))}
function openEdit(id){editing=markers.find(m=>m.id===id); if(!editing)return; $('mName').value=editing.name; $('mNote').value=editing.note||''; $('deleteBtn').style.display='block'; $('modalTitle').textContent='編輯標點'; $('modal').style.display='grid'}
function closeModal(){$('modal').style.display='none'; editing=null; pendingPoint=null}
function saveMarker(){const name=$('mName').value.trim(); if(!name)return alert('請輸入名稱'); if(editing){editing.name=name; editing.note=$('mNote').value; editing.updatedAt=Date.now(); pushGitHub('update marker: '+name)}else{const sel=$('type').value; const ci=$('customIcon').value.trim(); markers.push({id:crypto.randomUUID(),x:pendingPoint.x,y:pendingPoint.y,icon:ci||sel,name,note:$('mNote').value,owner:$('player').value||'匿名',createdAt:Date.now()}); pushGitHub('add marker: '+name)} closeModal()}
function deleteMarker(){if(!editing)return; const name=editing.name; markers=markers.filter(m=>m.id!==editing.id); closeModal(); pushGitHub('delete marker: '+name)}
function zoomIn(){const r=wrap.getBoundingClientRect();zoomAt(r.left+r.width/2,r.top+r.height/2,zoom*1.2)}
function zoomOut(){const r=wrap.getBoundingClientRect();zoomAt(r.left+r.width/2,r.top+r.height/2,zoom/1.2)}
function resetView(){zoom=1;panX=0;panY=0;applyTransform()}
loadConfig(); applyTransform(); render();
</script>
</body>
</html>
