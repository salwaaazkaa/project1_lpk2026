"""
ChemExplorer — app.py
Flask backend untuk website kimia interaktif.

Cara pakai:
  1. pip install flask anthropic requests python-dotenv
  2. Buat file .env berisi: ANTHROPIC_API_KEY=sk-ant-xxxxxxx
  3. python app.py
  4. Buka http://localhost:5000
"""

import os
import json
import textwrap

import anthropic
import requests
from flask import Flask, jsonify, render_template_string, request
from dotenv import load_dotenv

load_dotenv()

app = Flask(__name__)
client = anthropic.Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))
MODEL  = "claude-sonnet-4-20250514"


# ══════════════════════════════════════════════════════════════════
# HTML — satu file, tidak perlu folder templates/
# ══════════════════════════════════════════════════════════════════

HTML = r"""<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>ChemExplorer</title>
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet"/>
<script src="https://cdnjs.cloudflare.com/ajax/libs/3Dmol/2.0.1/3Dmol-min.js"></script>
<style>
:root{--bg:#0d0f14;--bg2:#13161f;--bg3:#1a1e2a;--border:rgba(255,255,255,.08);--border2:rgba(255,255,255,.14);--text:#f0f2f8;--muted:#7a8099;--accent:#7c6af7;--accent2:#5de3c8;--danger:#f76a6a;--success:#5de3a0;--warn:#f7c96a;--r:12px;--rs:8px}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:'Space Grotesk',sans-serif;background:var(--bg);color:var(--text);min-height:100vh}
.shell{display:grid;grid-template-columns:230px 1fr;min-height:100vh}
.sidebar{background:var(--bg2);border-right:1px solid var(--border);padding:2rem 1rem;display:flex;flex-direction:column;gap:4px;position:sticky;top:0;height:100vh;overflow-y:auto}
.logo{font-size:18px;font-weight:600;padding:0 .5rem 1.5rem;display:flex;align-items:center;gap:8px}
.logo-icon{width:32px;height:32px;background:linear-gradient(135deg,var(--accent),var(--accent2));border-radius:8px;display:flex;align-items:center;justify-content:center;font-size:16px}
.nav{display:flex;align-items:center;gap:10px;padding:9px 12px;border-radius:var(--rs);cursor:pointer;font-size:14px;color:var(--muted);border:none;background:none;width:100%;text-align:left;font-family:inherit;transition:all .15s}
.nav:hover{background:var(--bg3);color:var(--text)}
.nav.on{background:rgba(124,106,247,.15);color:var(--accent)}
.main{padding:2rem;max-width:860px}
.panel{display:none}.panel.on{display:block}
.ph{margin-bottom:1.5rem}
.pt{font-size:22px;font-weight:600;margin-bottom:4px}
.ps{font-size:14px;color:var(--muted)}
input,select{background:var(--bg2);border:1px solid var(--border2);color:var(--text);border-radius:var(--rs);padding:10px 14px;font-size:14px;font-family:inherit;outline:none;transition:border .15s;width:100%}
input:focus{border-color:var(--accent)}
button{background:var(--accent);color:#fff;border:none;border-radius:var(--rs);padding:10px 18px;font-size:14px;font-weight:500;font-family:inherit;cursor:pointer;transition:opacity .15s,transform .1s;white-space:nowrap}
button:hover{opacity:.88}button:active{transform:scale(.97)}
.ghost{background:transparent;border:1px solid var(--border2);color:var(--text)}
.ghost:hover{background:var(--bg3);opacity:1}
.sm{padding:6px 14px;font-size:13px}
.card{background:var(--bg2);border:1px solid var(--border);border-radius:var(--r);padding:1.25rem;margin-bottom:1rem}
.ct{font-size:11px;font-weight:600;letter-spacing:.08em;text-transform:uppercase;color:var(--muted);margin-bottom:12px}
.srow{display:flex;gap:8px;margin-bottom:12px}.srow input{flex:1}
.chips{display:flex;gap:6px;flex-wrap:wrap;margin-bottom:1.5rem;align-items:center}
.cl{font-size:12px;color:var(--muted)}
.chip{font-size:12px;padding:4px 12px;border-radius:20px;background:var(--bg3);border:1px solid var(--border);color:var(--muted);cursor:pointer;transition:all .15s}
.chip:hover{border-color:var(--accent);color:var(--accent)}
.rgrid{display:grid;grid-template-columns:1fr auto 1fr;gap:10px;align-items:center;margin-bottom:10px}
.plus{font-size:22px;color:var(--muted);text-align:center;padding-top:20px}
.fl{font-size:12px;color:var(--muted);display:block;margin-bottom:5px}
.pgrid{display:grid;grid-template-columns:repeat(auto-fill,minmax(150px,1fr));gap:10px;margin-bottom:1rem}
.pb{background:var(--bg3);border-radius:var(--rs);padding:10px 12px}
.pl{font-size:11px;color:var(--muted);margin-bottom:4px}
.pv{font-size:14px;font-weight:500}
.badge{display:inline-block;font-size:11px;padding:3px 10px;border-radius:20px;font-weight:500}
.bp{background:rgba(124,106,247,.18);color:var(--accent)}
.bt{background:rgba(93,227,200,.15);color:var(--accent2)}
.br{background:rgba(247,106,106,.15);color:var(--danger)}
.bg{background:rgba(93,227,160,.15);color:var(--success)}
.by{background:rgba(247,201,106,.15);color:var(--warn)}
.fbig{font-family:'JetBrains Mono',monospace;font-size:26px;font-weight:500;color:var(--accent)}
.mono{font-family:'JetBrains Mono',monospace}
.req{font-family:'JetBrains Mono',monospace;font-size:14px;background:var(--bg3);border-radius:var(--rs);padding:12px 14px;line-height:1.8;word-break:break-all;margin-bottom:10px}
.steps{list-style:none;counter-reset:s}
.steps li{counter-increment:s;padding:10px 0 10px 2.5rem;border-bottom:1px solid var(--border);font-size:14px;position:relative;line-height:1.6}
.steps li:last-child{border-bottom:none}
.steps li::before{content:counter(s);position:absolute;left:0;top:10px;width:24px;height:24px;background:rgba(124,106,247,.2);color:var(--accent);border-radius:50%;font-size:12px;font-weight:600;display:flex;align-items:center;justify-content:center}
.sbox{display:flex;align-items:center;gap:1rem;background:var(--bg2);border:1px solid var(--border);border-radius:var(--r);padding:1rem 1.25rem;margin-bottom:1.5rem}
.snum{font-size:28px;font-weight:600;color:var(--accent)}
.slbl{font-size:13px;color:var(--muted)}
.qopts{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin:1rem 0}
.qopt{padding:12px;border:1px solid var(--border2);border-radius:var(--rs);background:var(--bg2);cursor:pointer;font-size:14px;text-align:center;transition:all .15s;color:var(--text);font-family:inherit}
.qopt:hover{border-color:var(--accent);background:rgba(124,106,247,.08)}
.qopt.ok{background:rgba(93,227,160,.12);border-color:var(--success);color:var(--success)}
.qopt.no{background:rgba(247,106,106,.12);border-color:var(--danger);color:var(--danger)}
.div{border:none;border-top:1px solid var(--border);margin:1rem 0}
.fact{background:rgba(124,106,247,.1);border:1px solid rgba(124,106,247,.25);border-radius:var(--rs);padding:10px 14px;font-size:13px;line-height:1.6;color:#c5beff}
.dbox{background:rgba(247,106,106,.1);border:1px solid rgba(247,106,106,.25);border-radius:var(--rs);padding:10px 14px;font-size:13px;line-height:1.6;color:#f9b0b0;margin-bottom:1rem}
.loading{text-align:center;padding:3rem;color:var(--muted);font-size:14px}
.spin{display:inline-block;width:20px;height:20px;border:2px solid var(--border2);border-top-color:var(--accent);border-radius:50%;animation:sp .7s linear infinite;vertical-align:middle;margin-right:8px}
@keyframes sp{to{transform:rotate(360deg)}}
#mol-viewer{width:100%;height:320px;position:relative;border-radius:var(--rs);overflow:hidden}
@media(max-width:680px){
  .shell{grid-template-columns:1fr}
  .sidebar{position:fixed;bottom:0;left:0;right:0;top:auto;height:auto;flex-direction:row;padding:.5rem;border-top:1px solid var(--border);border-right:none;z-index:100;overflow-x:auto}
  .logo{display:none}
  .nav{flex-direction:column;font-size:11px;gap:2px;padding:6px 10px;min-width:64px}
  .main{padding:1rem;padding-bottom:6rem}
  .qopts,.rgrid{grid-template-columns:1fr}
  .plus{padding-top:0}
}
</style>
</head>
<body>
<div class="shell">

<aside class="sidebar">
  <div class="logo"><div class="logo-icon">⚗️</div> ChemExplorer</div>
  <button class="nav on" onclick="sw('search',this)">🔍 Cari Senyawa</button>
  <button class="nav" onclick="sw('reaction',this)">⚡ Reaksi Kimia</button>
  <button class="nav" onclick="sw('viewer3d',this)">🧬 Struktur 3D</button>
  <button class="nav" onclick="sw('quiz',this)">🧠 Kuis</button>
</aside>

<main class="main">

  <!-- ── CARI SENYAWA ── -->
  <div id="p-search" class="panel on">
    <div class="ph"><div class="pt">Cari Senyawa</div><div class="ps">Nama IUPAC, trivial, atau rumus molekul</div></div>
    <div class="srow">
      <input id="ci" placeholder="Contoh: etanol, CH₃COOH, glucose…" onkeydown="if(event.key==='Enter')cari()"/>
      <button onclick="cari()">Cari →</button>
    </div>
    <div class="chips">
      <span class="cl">Contoh:</span>
      <span class="chip" onclick="qs('ethanol')">Etanol</span>
      <span class="chip" onclick="qs('benzene')">Benzena</span>
      <span class="chip" onclick="qs('glucose')">Glukosa</span>
      <span class="chip" onclick="qs('acetic acid')">Asam Asetat</span>
      <span class="chip" onclick="qs('acetone')">Aseton</span>
      <span class="chip" onclick="qs('caffeine')">Kafein</span>
      <span class="chip" onclick="qs('aspirin')">Aspirin</span>
      <span class="chip" onclick="qs('NaCl')">NaCl</span>
    </div>
    <div id="sr"></div>
  </div>

  <!-- ── REAKSI ── -->
  <div id="p-reaction" class="panel">
    <div class="ph"><div class="pt">Simulasi Reaksi</div><div class="ps">Lihat produk, tipe, dan mekanisme reaksi</div></div>
    <div class="card">
      <div class="ct">Masukkan Reaktan</div>
      <div class="rgrid">
        <div><label class="fl">Senyawa</label><input id="rc" placeholder="misal: etanol"/></div>
        <div class="plus">+</div>
        <div><label class="fl">Reagen</label><input id="rr" placeholder="misal: HCl, H₂SO₄"/></div>
      </div>
      <div style="margin:10px 0 12px"><label class="fl">Kondisi tambahan (opsional)</label><input id="rx" placeholder="misal: panas, katalis Pt"/></div>
      <button onclick="reaksi()" style="width:100%">Simulasikan →</button>
    </div>
    <div class="chips">
      <span class="cl">Contoh:</span>
      <span class="chip" onclick="sr2('etanol','HBr','')">Etanol + HBr</span>
      <span class="chip" onclick="sr2('benzena','Cl₂','katalis FeCl₃')">Benzena + Cl₂</span>
      <span class="chip" onclick="sr2('2-butena','H₂O','asam')">2-Butena + H₂O</span>
      <span class="chip" onclick="sr2('propanol','H₂SO₄','panas')">Propanol + H₂SO₄</span>
    </div>
    <div id="rr2"></div>
  </div>

  <!-- ── 3D VIEWER ── -->
  <div id="p-viewer3d" class="panel">
    <div class="ph"><div class="pt">Struktur 3D</div><div class="ps">Visualisasi molekul via PubChem + 3Dmol.js — bisa diputar!</div></div>
    <div class="card">
      <div class="ct">Cari Molekul</div>
      <div class="srow">
        <input id="mi" placeholder="Nama atau rumus…" onkeydown="if(event.key==='Enter')load3D()"/>
        <button onclick="load3D()">Tampilkan 3D →</button>
      </div>
      <div class="chips" style="margin-bottom:0">
        <span class="chip" onclick="m3('ethanol')">Etanol</span>
        <span class="chip" onclick="m3('benzene')">Benzena</span>
        <span class="chip" onclick="m3('aspirin')">Aspirin</span>
        <span class="chip" onclick="m3('glucose')">Glukosa</span>
        <span class="chip" onclick="m3('caffeine')">Kafein</span>
        <span class="chip" onclick="m3('cholesterol')">Kolesterol</span>
      </div>
    </div>
    <div id="3dr"><div class="loading">Masukkan nama senyawa untuk melihat 3D-nya</div></div>
  </div>

  <!-- ── KUIS ── -->
  <div id="p-quiz" class="panel">
    <div class="ph"><div class="pt">Kuis Kimia</div><div class="ps">Tebak nama senyawa dari rumus dan golongannya</div></div>
    <div class="sbox">
      <div><div class="snum" id="sc">0 / 0</div><div class="slbl">Skor kamu</div></div>
      <div style="margin-left:auto;display:flex;gap:8px">
        <button class="ghost sm" onclick="resetKuis()">Reset</button>
        <button onclick="soalBaru()">Soal Baru →</button>
      </div>
    </div>
    <div id="qa"><div class="loading">Tekan "Soal Baru" untuk mulai</div></div>
  </div>

</main>
</div>

<script>
// ── NAV ──
function sw(id,el){
  document.querySelectorAll('.panel').forEach(p=>p.classList.remove('on'));
  document.querySelectorAll('.nav').forEach(n=>n.classList.remove('on'));
  document.getElementById('p-'+id).classList.add('on');
  if(el) el.classList.add('on');
}
function qs(n){document.getElementById('ci').value=n;cari();}
function sr2(c,r,x){document.getElementById('rc').value=c;document.getElementById('rr').value=r;document.getElementById('rx').value=x;reaksi();}
function m3(n){document.getElementById('mi').value=n;load3D();}
function goReaksi(n){document.getElementById('rc').value=n;document.getElementById('rr').value='';sw('reaction',document.querySelectorAll('.nav')[1]);}
function go3D(n){m3(n);sw('viewer3d',document.querySelectorAll('.nav')[2]);}

// ── FETCH ──
function spin(msg){return`<div class="loading"><span class="spin"></span>${msg}</div>`;}
async function api(url,body){
  const r=await fetch(url,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(body)});
  const d=await r.json();
  if(!r.ok) throw new Error(d.error||'Request gagal');
  return d;
}

// ── CARI SENYAWA ──
async function cari(){
  const name=document.getElementById('ci').value.trim();
  if(!name) return;
  const el=document.getElementById('sr');
  el.innerHTML=spin('Menganalisis senyawa...');
  try{const c=await api('/api/compound',{name});renderSenyawa(c,name);}
  catch(e){el.innerHTML=`<div class="card" style="color:var(--danger)">⚠ ${e.message}</div>`;}
}
function renderSenyawa(c,q){
  document.getElementById('sr').innerHTML=`
  <div class="card">
    <div style="display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:1.25rem;flex-wrap:wrap;gap:10px">
      <div>
        <div style="font-size:20px;font-weight:600;margin-bottom:4px">${c.nama_iupac||q}</div>
        <div style="font-size:13px;color:var(--muted)">${c.nama_trivial||''}</div>
        <div style="display:flex;gap:6px;flex-wrap:wrap;margin-top:8px">
          ${c.golongan_senyawa?`<span class="badge bp">${c.golongan_senyawa}</span>`:''}
          ${c.gugus_fungsi?`<span class="badge bt">${c.gugus_fungsi}</span>`:''}
          ${c.polaritas?`<span class="badge by">${c.polaritas}</span>`:''}
        </div>
      </div>
      <div class="fbig">${c.rumus_molekul||''}</div>
    </div>
    <div class="ct">Sifat Fisika & Kimia</div>
    <div class="pgrid" style="margin-bottom:1.25rem">
      <div class="pb"><div class="pl">Massa Molar</div><div class="pv">${c.massa_molar||'-'}</div></div>
      <div class="pb"><div class="pl">Titik Didih</div><div class="pv">${c.titik_didih||'-'}</div></div>
      <div class="pb"><div class="pl">Titik Leleh</div><div class="pv">${c.titik_leleh||'-'}</div></div>
      <div class="pb"><div class="pl">Wujud 25°C</div><div class="pv">${c.wujud_suhu_kamar||'-'}</div></div>
      <div class="pb"><div class="pl">Warna</div><div class="pv">${c.warna||'-'}</div></div>
      <div class="pb"><div class="pl">Bau</div><div class="pv">${c.bau||'-'}</div></div>
      <div class="pb"><div class="pl">Kelarutan</div><div class="pv">${c.kelarutan_air||'-'}</div></div>
      <div class="pb"><div class="pl">pH</div><div class="pv">${c.ph_larutan||'-'}</div></div>
    </div>
    <hr class="div"/>
    <div class="ct">Struktur Bangun</div>
    <div style="font-size:14px;color:var(--muted);line-height:1.7;margin-bottom:1.25rem">${c.rumus_bangun_deskripsi||'-'}</div>
    <div class="ct">Kegunaan</div>
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-bottom:1.25rem">
      <div class="pb"><div class="pl">Industri</div><div style="font-size:13px;line-height:1.6;margin-top:4px">${c.kegunaan_industri||'-'}</div></div>
      <div class="pb"><div class="pl">Laboratorium</div><div style="font-size:13px;line-height:1.6;margin-top:4px">${c.kegunaan_laboratorium||'-'}</div></div>
    </div>
    ${c.sumber_alam?`<div class="ct">Sumber Alam</div><div style="font-size:13px;color:var(--muted);line-height:1.6;margin-bottom:1.25rem">${c.sumber_alam}</div>`:''}
    ${c.bahaya_keamanan?`<div class="dbox">⚠️ <strong>Bahaya:</strong> ${c.bahaya_keamanan}</div>`:''}
    <div class="fact">💡 <strong>Fakta:</strong> ${c.fakta_menarik||'-'}</div>
    <div style="display:flex;gap:8px;margin-top:1.25rem;flex-wrap:wrap">
      <button class="ghost sm" onclick="goReaksi('${(c.nama_iupac||q).replace(/'/g,"\\'")}')">Simulasikan Reaksi →</button>
      <button class="ghost sm" onclick="go3D('${(c.nama_iupac||q).replace(/'/g,"\\'")}')">Lihat 3D →</button>
    </div>
  </div>`;
}

// ── REAKSI ──
async function reaksi(){
  const compound=document.getElementById('rc').value.trim();
  const reagent=document.getElementById('rr').value.trim();
  const conditions=document.getElementById('rx').value.trim();
  if(!compound||!reagent){alert('Isi senyawa dan reagen dulu!');return;}
  const el=document.getElementById('rr2');
  el.innerHTML=spin('Mensimulasikan reaksi...');
  try{const d=await api('/api/reaction',{compound,reagent,conditions});renderReaksi(d);}
  catch(e){el.innerHTML=`<div class="card" style="color:var(--danger)">⚠ ${e.message}</div>`;}
}
function renderReaksi(r){
  const steps=Array.isArray(r.langkah_mekanisme)?r.langkah_mekanisme:[];
  const TB={adisi:'bt',substitusi:'bp',eliminasi:'by',oksidasi:'br',reduksi:'bg',netralisasi:'bg'};
  const tk=(r.tipe_reaksi||'').toLowerCase();
  const bc=Object.keys(TB).find(k=>tk.includes(k))?TB[Object.keys(TB).find(k=>tk.includes(k))]:'bp';
  document.getElementById('rr2').innerHTML=`
  <div class="card">
    <div style="display:flex;gap:6px;flex-wrap:wrap;margin-bottom:1rem">
      <span class="badge ${bc}">${r.tipe_reaksi||'Reaksi'}</span>
      ${r.subtipe?`<span class="badge by">${r.subtipe}</span>`:''}
      ${r.energi?`<span class="badge ${r.energi.toLowerCase().includes('ekso')?'br':'bt'}">${r.energi}</span>`:''}
    </div>
    <div class="ct">Persamaan Reaksi</div>
    <div class="req">${r.persamaan_reaksi||''}</div>
    <div class="pgrid" style="margin-bottom:1.25rem">
      <div class="pb"><div class="pl">Produk Utama</div><div class="pv">${r.produk_utama||'-'}</div></div>
      <div class="pb"><div class="pl">Produk Sampingan</div><div class="pv">${r.produk_sampingan||'-'}</div></div>
      <div class="pb"><div class="pl">Kondisi</div><div class="pv">${r.kondisi_reaksi||'-'}</div></div>
      <div class="pb"><div class="pl">Laju Reaksi</div><div class="pv">${r.laju_reaksi||'-'}</div></div>
    </div>
    <div class="ct">Mekanisme</div>
    <div style="font-size:14px;color:var(--muted);line-height:1.7;margin-bottom:1.25rem">${r.mekanisme_ringkas||'-'}</div>
    ${steps.length?`<div class="ct">Langkah-Langkah</div><ol class="steps" style="margin-bottom:1.25rem">${steps.map(s=>`<li>${s}</li>`).join('')}</ol>`:''}
    ${r.contoh_industri?`<div class="pb" style="margin-bottom:1rem"><div class="pl">Contoh Industri</div><div style="font-size:13px;line-height:1.6;margin-top:4px">${r.contoh_industri}</div></div>`:''}
    ${r.catatan_keamanan?`<div class="dbox">⚠️ ${r.catatan_keamanan}</div>`:''}
  </div>`;
}

// ── 3D ──
async function load3D(){
  const name=document.getElementById('mi').value.trim();
  if(!name) return;
  const el=document.getElementById('3dr');
  el.innerHTML=spin('Mengambil data 3D dari PubChem...');
  try{
    const d=await api('/api/structure3d',{name});
    const{cid,sdf,properties:p}=d;
    el.innerHTML=`
    <div class="card">
      <div style="display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:1rem;flex-wrap:wrap;gap:8px">
        <div>
          <div style="font-size:16px;font-weight:600">${p.IUPACName||name}</div>
          <div style="font-size:12px;color:var(--muted)">PubChem CID: ${cid}</div>
        </div>
        <div class="fbig">${p.MolecularFormula||''}</div>
      </div>
      <div id="mol-viewer"></div>
      <div class="pgrid" style="margin-top:12px">
        <div class="pb"><div class="pl">Massa Molar</div><div class="pv">${p.MolecularWeight||'-'} g/mol</div></div>
        <div class="pb" style="grid-column:span 2"><div class="pl">SMILES</div><div style="font-size:12px;font-family:'JetBrains Mono',monospace;word-break:break-all;margin-top:4px;color:var(--accent2)">${p.IsomericSMILES||'-'}</div></div>
      </div>
      <div style="display:flex;gap:6px;margin-top:10px;flex-wrap:wrap;align-items:center">
        <span style="font-size:12px;color:var(--muted)">Gaya:</span>
        <button class="ghost sm" onclick="gaya('stick')">Stick</button>
        <button class="ghost sm" onclick="gaya('sphere')">Sphere</button>
        <button class="ghost sm" onclick="gaya('line')">Line</button>
      </div>
    </div>`;
    if(sdf){
      setTimeout(()=>{
        const ve=document.getElementById('mol-viewer');
        if(!ve) return;
        window._v=$3Dmol.createViewer(ve,{backgroundColor:'#1a1e2a'});
        window._v.addModel(sdf,'sdf');
        window._v.setStyle({},{stick:{radius:.15,colorscheme:'Jmol'}});
        window._v.zoomTo();window._v.render();
      },100);
    } else {
      document.getElementById('mol-viewer').innerHTML='<div class="loading">Data 3D tidak tersedia</div>';
    }
  }catch(e){el.innerHTML=`<div class="card" style="color:var(--danger)">⚠ ${e.message}</div>`;}
}
function gaya(s){
  if(!window._v) return;
  const m={stick:{stick:{radius:.15,colorscheme:'Jmol'}},sphere:{sphere:{colorscheme:'Jmol'}},line:{line:{linewidth:2,colorscheme:'Jmol'}}};
  window._v.setStyle({},m[s]||m.stick);window._v.render();
}

// ── KUIS ──
const POOL=[
  {name:'Etanol',formula:'C₂H₅OH',type:'Alkohol primer',opts:['Etanol','Metanol','Isopropanol','Butanol']},
  {name:'Asam Asetat',formula:'CH₃COOH',type:'Asam karboksilat',opts:['Asam Asetat','Asam Formiat','Asam Propionat','Asam Butirat']},
  {name:'Benzena',formula:'C₆H₆',type:'Aromatik',opts:['Benzena','Sikloheksana','Toluena','Naftalena']},
  {name:'Aseton',formula:'CH₃COCH₃',type:'Keton',opts:['Aseton','Propanal','Propanol','Dietil Eter']},
  {name:'Glukosa',formula:'C₆H₁₂O₆',type:'Monosakarida',opts:['Glukosa','Fruktosa','Galaktosa','Sukrosa']},
  {name:'Kloroform',formula:'CHCl₃',type:'Haloalkana',opts:['Kloroform','Diklorometan','Tetraklorometana','Kloroetana']},
  {name:'Anilin',formula:'C₆H₅NH₂',type:'Amina aromatik',opts:['Anilin','Fenol','Toluena','Benzaldehida']},
  {name:'Asam Benzoat',formula:'C₆H₅COOH',type:'Asam karboksilat aromatik',opts:['Asam Benzoat','Fenol','Asam Salisilat','Asam Asetat']},
  {name:'Dietil Eter',formula:'C₂H₅OC₂H₅',type:'Eter',opts:['Dietil Eter','Dimetil Eter','THF','Dioksan']},
  {name:'Formaldehida',formula:'HCHO',type:'Aldehida',opts:['Formaldehida','Asetaldehida','Propanaldehida','Butanaldehida']},
  {name:'Propena',formula:'CH₃CH=CH₂',type:'Alkena',opts:['Propena','Propana','Propuna','Butena']},
  {name:'Asetilena',formula:'C₂H₂',type:'Alkuna',opts:['Asetilena','Etena','Etana','Propuna']},
];
let qS=0,qT=0,qA=false,qC=null;

function resetKuis(){qS=0;qT=0;document.getElementById('sc').textContent='0 / 0';document.getElementById('qa').innerHTML='<div class="loading">Tekan "Soal Baru" untuk mulai</div>';}

async function soalBaru(){
  qA=false;
  document.getElementById('qa').innerHTML=spin('Menyiapkan soal...');
  qC=POOL[Math.floor(Math.random()*POOL.length)];
  const opts=[...qC.opts].sort(()=>Math.random()-.5);
  document.getElementById('qa').innerHTML=`
  <div class="card">
    <div class="ct">Senyawa apakah ini?</div>
    <div style="background:var(--bg3);border-radius:var(--rs);padding:2rem;text-align:center;margin-bottom:1rem">
      <div style="font-size:36px;font-family:'JetBrains Mono',monospace;font-weight:500;color:var(--accent);margin-bottom:8px">${qC.formula}</div>
      <div style="font-size:13px;color:var(--muted)">Golongan: ${qC.type}</div>
    </div>
  </div>
  <div class="qopts">${opts.map(o=>`<button class="qopt" onclick="jawab(this,'${o.replace(/'/g,"\\'")}')">${o}</button>`).join('')}</div>
  <div id="qf"></div>`;
}

async function jawab(el,ans){
  if(qA) return;
  qA=true;qT++;
  const ok=ans===qC.name;
  if(ok) qS++;
  document.getElementById('sc').textContent=`${qS} / ${qT}`;
  document.querySelectorAll('.qopt').forEach(o=>{
    if(o.textContent.trim()===qC.name) o.classList.add('ok');
    else if(o===el&&!ok) o.classList.add('no');
    o.style.pointerEvents='none';
  });
  const fb=document.getElementById('qf');
  fb.innerHTML=spin('Memuat penjelasan AI...');
  try{
    const d=await api('/api/quiz-explain',{name:qC.name,formula:qC.formula});
    fb.innerHTML=`
    <div style="background:${ok?'rgba(93,227,160,.1)':'rgba(247,106,106,.1)'};border:1px solid ${ok?'rgba(93,227,160,.3)':'rgba(247,106,106,.3)'};border-radius:var(--rs);padding:14px;margin-top:4px">
      <div style="font-size:13px;font-weight:600;color:${ok?'var(--success)':'var(--danger)'};margin-bottom:8px">${ok?'✓ Benar!':'✗ Jawaban: '+qC.name}</div>
      <div style="font-size:14px;line-height:1.7">${d.explanation}</div>
    </div>
    <button onclick="soalBaru()" style="width:100%;margin-top:10px">Soal Berikutnya →</button>`;
  }catch{fb.innerHTML=`<button onclick="soalBaru()" style="width:100%;margin-top:10px">Soal Berikutnya →</button>`;}
}
</script>
</body>
</html>"""


# ══════════════════════════════════════════════════════════════════
# ROUTES
# ══════════════════════════════════════════════════════════════════

@app.route("/")
def index():
    return render_template_string(HTML)


@app.route("/api/compound", methods=["POST"])
def get_compound():
    name = request.json.get("name", "").strip()
    if not name:
        return jsonify({"error": "Nama senyawa kosong"}), 400

    system = textwrap.dedent("""\
        Kamu adalah asisten kimia yang akurat.
        Jawab HANYA dalam format JSON valid, tanpa markdown, tanpa backtick.
        Skema wajib (semua nilai string):
        {
          "nama_iupac": "", "nama_trivial": "", "rumus_molekul": "",
          "massa_molar": "", "titik_didih": "", "titik_leleh": "",
          "kelarutan_air": "", "wujud_suhu_kamar": "", "warna": "", "bau": "",
          "golongan_senyawa": "", "gugus_fungsi": "", "polaritas": "", "ph_larutan": "",
          "kegunaan_industri": "", "kegunaan_laboratorium": "", "bahaya_keamanan": "",
          "rumus_bangun_deskripsi": "", "fakta_menarik": "", "sumber_alam": ""
        }""")

    try:
        resp = client.messages.create(
            model=MODEL, max_tokens=1000, system=system,
            messages=[{"role": "user", "content": f"Data lengkap senyawa: {name}"}]
        )
        text = resp.content[0].text.strip().replace("```json", "").replace("```", "").strip()
        return jsonify(json.loads(text))
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route("/api/reaction", methods=["POST"])
def get_reaction():
    data       = request.json
    compound   = data.get("compound", "").strip()
    reagent    = data.get("reagent", "").strip()
    conditions = data.get("conditions", "standar").strip() or "standar"

    if not compound or not reagent:
        return jsonify({"error": "Senyawa dan reagen wajib diisi"}), 400

    system = textwrap.dedent("""\
        Kamu ahli kimia organik. Jawab HANYA JSON valid tanpa markdown.
        Skema:
        {
          "persamaan_reaksi": "", "tipe_reaksi": "", "subtipe": "",
          "produk_utama": "", "produk_sampingan": "", "kondisi_reaksi": "",
          "energi": "", "laju_reaksi": "", "mekanisme_ringkas": "",
          "langkah_mekanisme": ["langkah 1", "langkah 2", "langkah 3"],
          "contoh_industri": "", "catatan_keamanan": ""
        }""")

    try:
        resp = client.messages.create(
            model=MODEL, max_tokens=1000, system=system,
            messages=[{"role": "user", "content":
                f"Senyawa: {compound}\nReagen: {reagent}\nKondisi: {conditions}"}]
        )
        text = resp.content[0].text.strip().replace("```json", "").replace("```", "").strip()
        return jsonify(json.loads(text))
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route("/api/structure3d", methods=["POST"])
def get_structure3d():
    name = request.json.get("name", "").strip()
    if not name:
        return jsonify({"error": "Nama kosong"}), 400

    try:
        base = "https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound"

        cid_r = requests.get(f"{base}/name/{requests.utils.quote(name)}/cids/JSON", timeout=10)
        if not cid_r.ok:
            return jsonify({"error": "Senyawa tidak ditemukan di PubChem"}), 404
        cid = cid_r.json()["IdentifierList"]["CID"][0]

        sdf_r = requests.get(f"{base}/CID/{cid}/SDF?record_type=3d", timeout=10)
        sdf   = sdf_r.text if sdf_r.ok else None

        props_fields = "IUPACName,MolecularFormula,MolecularWeight,IsomericSMILES"
        prop_r = requests.get(f"{base}/CID/{cid}/property/{props_fields}/JSON", timeout=10)
        props  = prop_r.json()["PropertyTable"]["Properties"][0] if prop_r.ok else {}

        return jsonify({"cid": cid, "sdf": sdf, "properties": props})
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route("/api/quiz-explain", methods=["POST"])
def quiz_explain():
    data    = request.json
    name    = data.get("name", "")
    formula = data.get("formula", "")

    try:
        resp = client.messages.create(
            model=MODEL, max_tokens=200,
            messages=[{"role": "user", "content":
                f"Jelaskan {name} ({formula}) dalam 2-3 kalimat untuk siswa SMA. "
                "Sertakan ciri khas strukturnya dan satu kegunaan utama."}]
        )
        return jsonify({"explanation": resp.content[0].text})
    except Exception as e:
        return jsonify({"error": str(e)}), 500


# ══════════════════════════════════════════════════════════════════
if __name__ == "__main__":
    if not os.environ.get("ANTHROPIC_API_KEY"):
        print("⚠  ANTHROPIC_API_KEY belum di-set!")
        print("   Buat file .env berisi: ANTHROPIC_API_KEY=sk-ant-xxxxxxx")
    app.run(debug=True, port=5000)
