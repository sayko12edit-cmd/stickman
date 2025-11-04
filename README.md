<!doctype html>
<html lang="tr">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Stickman Fight — Host / Join (Firebase ile)</title>
  <style>
    /* Basit, okunaklı stil */
    body{font-family:Inter,system-ui,Segoe UI,Arial;margin:0;display:flex;min-height:100vh}
    #ui{width:360px;padding:18px;border-right:1px solid #eee;background:#fafafa}
    #gamewrap{flex:1;display:flex;align-items:center;justify-content:center;background:#111}
    canvas{background:#87ceeb;border-radius:6px;display:block}
    h1{margin:0 0 12px;font-size:18px}
    label{display:block;margin-top:8px;font-size:13px}
    input,button,select{width:100%;padding:8px;margin-top:6px;border-radius:6px;border:1px solid #ccc}
    .row{display:flex;gap:8px}
    .small{flex:1}
    .meta{font-size:12px;color:#666;margin-top:8px}
    .danger{color:#b00}
  </style>
</head>
<body>
  <div id="ui">
    <h1>Stickman Fight — Host / Join</h1>
    <p>Not: Google Sites <strong>JavaScript</strong> çalıştırmayı engeller. Bu dosyayı <strong>GitHub Pages</strong> veya <strong>Netlify</strong> gibi statik hostlara yükleyin. Aşağıda Firebase kullanılarak basit host/join (oda) işlevi sağlanmıştır.</p>

    <label>Firebase config (project web app) - Aşağıya yapıştırın</label>
    <textarea id="fbconfig" placeholder="{ apiKey: '...', authDomain: '...', databaseURL: '...', projectId: '...' }" style="height:90px"></textarea>

    <label>Oda ID (yoksa create'e basın)</label>
    <div class="row">
      <input id="roomId" placeholder="ör: oda-1234" />
      <button id="createBtn">Create</button>
    </div>
    <div class="row">
      <button id="hostBtn" class="small">Host</button>
      <button id="joinBtn" class="small">Join</button>
    </div>

    <label>Kontroller</label>
    <div class="meta">Host: A,S - hareket, D - saldırı. Join: ←,→ - hareket, 0 - saldırı</div>

    <div style="margin-top:12px">
      <div>Oyun durumu: <span id="status">Hazır</span></div>
      <div>Oda linki: <a id="roomLink" href="#">—</a></div>
    </div>

    <div style="margin-top:12px;display:flex;gap:8px">
      <button id="resetBtn">Reset Room</button>
      <button id="deployBtn">Deploy İçin İpuçları</button>
    </div>

    <p class="meta">Bu örnek öğrenme amaçlıdır. Gerçek oyunlar için yetkilendirme, hile önleme ve düzgün sinyal sunucusu gereklidir.</p>
  </div>

  <div id="gamewrap">
    <canvas id="game" width="800" height="400"></canvas>
  </div>

  <!-- Firebase (compat) CDN - Bu sayfada kullanabilmek için internet bağlantısı gerekir -->
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-database-compat.js"></script>

  <script>
  // Basit oyun + Firebase Realtime Database ile oda / host/join senkronizasyonu.
  // Kullanım:
  // 1) Firebase console -> Realtime Database oluşturun (kuralları test için açık tutabilirsiniz)
  // 2) Web app oluşturup config nesnesini alıp üstteki textarea'ya yapıştırın
  // 3) Create ile oda id üretin, Host ya da Join ile katılın.

  // --- Basit yardımcılar ---
  const $ = id => document.getElementById(id);
  let clientId = String(Math.floor(Math.random()*1e9));
  let isHost = false;
  let roomId = null;
  let db = null;
  let roomRef = null;

  // Oyun state
  const cvs = $('game');
  const ctx = cvs.getContext('2d');
  const W = cvs.width, H = cvs.height;

  const state = {
    players: {
      // id: {x,y,dir,health,attack}
    },
    started:false
  };

  // Basit stickman çizici
  function drawPlayer(p){
    ctx.save();
    ctx.translate(p.x,p.y);
    // body
    ctx.beginPath(); ctx.arc(0,-20,8,0,Math.PI*2); ctx.fill(); // head
    ctx.fillRect(-2,-12,4,20); // body
    // arms
    ctx.beginPath(); ctx.moveTo(0,-6); ctx.lineTo(p.dir*18,2); ctx.stroke();
    // legs
    ctx.beginPath(); ctx.moveTo(0,8); ctx.lineTo(-8,22); ctx.moveTo(0,8); ctx.lineTo(8,22); ctx.stroke();
    ctx.restore();
  }

  // Oyun loop
  function gameLoop(timestamp){
    // Update simple physics if host
    if(isHost && roomRef){
      // apply gravity / movement
      for(const id in state.players){
        const p = state.players[id];
        // basic bounds
        if(p.x<20) p.x=20; if(p.x>W-20) p.x=W-20;
        if(p.y>H-40) p.y=H-40;
        // attack cooldown decay
        if(p.attack>0) p.attack = Math.max(0,p.attack-1);
      }
      // collision check: if one attacking and close -> reduce hp
      const ids = Object.keys(state.players);
      if(ids.length>=2){
        const a = state.players[ids[0]], b = state.players[ids[1]];
        const dist = Math.abs(a.x-b.x);
        if(a.attack>0 && dist<50 && a.lastAttackFrame===undefined){ b.health-=10; a.lastAttackFrame=1; }
        if(b.attack>0 && dist<50 && b.lastAttackFrame===undefined){ a.health-=10; b.lastAttackFrame=1; }
      }
      // write state to db every 100ms
      nowWriteStateDebounced();
    }

    // render
    ctx.clearRect(0,0,W,H);
    // ground
    ctx.fillStyle='#274'; ctx.fillRect(0,H-20,W,20);

    // draw players
    ctx.fillStyle='#fff'; ctx.strokeStyle='#fff'; ctx.lineWidth=3;
    for(const id in state.players){
      drawPlayer(state.players[id]);
    }

    // draw HUD
    ctx.fillStyle='#fff'; ctx.font='14px monospace';
    let i=0;
    for(const id in state.players){
      const p = state.players[id];
      ctx.fillText((id===clientId? 'YOU':'OPP')+ ' HP:'+Math.max(0,Math.floor(p.health||0)),10,20+18*i);
      i++;
    }

    requestAnimationFrame(gameLoop);
  }

  // Debounced write
  let writeTimeout = null;
  function nowWriteStateDebounced(){
    if(writeTimeout) return;
    writeTimeout = setTimeout(()=>{
      writeTimeout = null;
      if(roomRef) roomRef.child('state').set(state).catch(e=>console.warn(e));
    },100);
  }

  // Local input handling
  const keys = {};
  window.addEventListener('keydown',e=>{ keys[e.key]=true; handleLocalInput(); });
  window.addEventListener('keyup',e=>{ keys[e.key]=false; handleLocalInput(); });

  function handleLocalInput(){
    const p = state.players[clientId]; if(!p) return;
    let moved=false;
    // host uses a,s,d ; join uses ArrowLeft,ArrowRight,0
    if(isHost){
      if(keys['a']){ p.x -= 6; p.dir=-1; moved=true; }
      if(keys['s']){ p.x += 6; p.dir=1; moved=true; }
      if(keys['d']){ p.attack = 6; moved=true; }
    } else {
      if(keys['ArrowLeft']){ p.x -= 6; p.dir=-1; moved=true; }
      if(keys['ArrowRight']){ p.x += 6; p.dir=1; moved=true; }
      if(keys['0']){ p.attack = 6; moved=true; }
    }
    if(moved && isHost){ nowWriteStateDebounced(); }
    // If not host, push only our input to /rooms/{id}/inputs/{clientId}
    if(!isHost && roomRef){
      roomRef.child('inputs').child(clientId).set({x:p.x,dir:p.dir,attack:p.attack,ts:Date.now()});
    }
  }

  // Firebase init from textarea
  function initFirebaseFromTextarea(){
    const raw = $('fbconfig').value.trim();
    if(!raw) return alert('Firebase config girin');
    let cfg=null;
    try{ cfg = JSON.parse(raw); }catch(e){
      try{ /* allow JS object style */ cfg = eval('('+raw+')'); }catch(e){ return alert('Geçersiz config. JSON formatında yapıştırın.'); }
    }
    const app = firebase.initializeApp(cfg);
    db = firebase.database();
    $('status').innerText = 'Firebase bağlı.';
  }

  // Create room id
  $('createBtn').addEventListener('click',()=>{
    const id = 'room-'+Math.random().toString(36).slice(2,8);
    $('roomId').value = id;
  });

  $('hostBtn').addEventListener('click',async ()=>{
    initFirebaseFromTextarea();
    isHost=true; roomId = $('roomId').value.trim(); if(!roomId) return alert('Oda ID girin');
    roomRef = db.ref('rooms/'+roomId);
    // init room
    state.players = {};
    state.players[clientId] = {x:100,y:H-40,dir:1,health:100,attack:0};
    // wait for joiner
    roomRef.child('meta').set({host:clientId,created:Date.now()});
    roomRef.child('players').child(clientId).set(state.players[clientId]);
    roomRef.child('state').set(state);

    // listen to inputs from joiner
    roomRef.child('inputs').on('value',snap=>{
      const inputs = snap.val()||{};
      for(const id in inputs){ if(id===clientId) continue; const inp = inputs[id];
        // ensure player exists
        if(!state.players[id]) state.players[id] = {x:W-100,y:H-40,dir:-1,health:100,attack:0};
        // apply remote input as authoritative for that client
        state.players[id].x = inp.x;
        state.players[id].dir = inp.dir;
        state.players[id].attack = inp.attack;
      }
    });

    // write state periodically
    setInterval(()=>{ if(roomRef) roomRef.child('state').set(state); }, 200);

    $('status').innerText = 'Host: Oda hazır. Bekleniyor...';
    $('roomLink').innerText = location.href + '?room=' + roomId;
    $('roomLink').href = location.href + '?room=' + roomId;

    requestAnimationFrame(gameLoop);
  });

  $('joinBtn').addEventListener('click',async ()=>{
    initFirebaseFromTextarea();
    isHost=false; roomId = $('roomId').value.trim(); if(!roomId) return alert('Oda ID girin');
    roomRef = db.ref('rooms/'+roomId);
    // add our player locally
    state.players[clientId] = {x:W-100,y:H-40,dir:-1,health:100,attack:0};
    // write our presence
    roomRef.child('players').child(clientId).set(state.players[clientId]);
    $('status').innerText = 'Joined oda: '+roomId;
    $('roomLink').innerText = location.href + '?room=' + roomId;
    $('roomLink').href = location.href + '?room=' + roomId;

    // listen state updates (host authoritative)
    roomRef.child('state').on('value',snap=>{
      const s = snap.val(); if(!s) return;
      // merge authoritative state but keep our own local id if needed
      for(const id in s.players){ state.players[id] = s.players[id]; }
      // ensure our presence if not in authoritative players (network lag)
      if(!state.players[clientId]) state.players[clientId] = {x:W-100,y:H-40,dir:-1,health:100,attack:0};
    });

    requestAnimationFrame(gameLoop);
  });

  $('resetBtn').addEventListener('click',()=>{
    if(!roomRef) return alert('Odaya bağlı değilsiniz');
    roomRef.remove(); $('status').innerText='Oda sıfırlandı.';
  });

  $('deployBtn').addEventListener('click',()=>{
    alert('Google Sites JavaScript çalıştırmaz. Bu dosyayı GitHub Pages veya Netlify/ Vercel gibi servislere yükleyin.\n\n1) Yeni repo oluşturun (ör: stickman-fight).\n2) index.html dosyasını ekleyin.\n3) GitHub Pages -> main branch seçin ve siteyi yayınlayın.\n\nAyrıca Firebase hosting (firebase deploy) ile doğrudan host edebilirsiniz.');
  });

  // Auto-load ?room= query param
  (function(){
    const params = new URLSearchParams(location.search);
    const r = params.get('room'); if(r) $('roomId').value = r;
  })();

  // Başlangıç döngüsü (render başlat)
  requestAnimationFrame(gameLoop);
  </script>
</body>
</html>
