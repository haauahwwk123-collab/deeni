# -*- coding: utf-8 -*-
"""
ديني - تطبيق إسلامي متكامل: قرآن، حديث، مواقيت صلاة، أذكار
Flask single-file app. Made with love by hassan_ahmed.
"""
import json
import time
import threading
import urllib.request
from flask import Flask, render_template_string, request, jsonify, Response

app = Flask(__name__)

# ---------------- API helpers (in-memory cache) ----------------
_cache = {}
_cache_lock = threading.Lock()

def fetch_json(url, cache_key=None, ttl=3600):
    """Fetch JSON with simple in-memory caching."""
    now = time.time()
    if cache_key:
        with _cache_lock:
            hit = _cache.get(cache_key)
            if hit and now - hit[0] < ttl:
                return hit[1]
    try:
        req = urllib.request.Request(url, headers={"User-Agent": "deen-app/1.0"})
        with urllib.request.urlopen(req, timeout=20) as r:
            data = json.loads(r.read().decode("utf-8"))
    except Exception:
        return None
    if cache_key:
        with _cache_lock:
            _cache[cache_key] = (now, data)
    return data

QURAN_API = "https://api.alquran.cloud/v1"
HADITH_URL = "https://cdn.jsdelivr.net/gh/fawazahmed0/hadith-api@1/editions/ara-nawawi.min.json"
ALADHAN_API = "https://api.aladhan.com/v1"
AUDIO_BASE = "https://cdn.islamic.network/quran/audio-surah/128/ar.alafasy"

# ---------------- Base template ----------------
BASE = """<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="theme-color" content="#0b1512">
<title>{{ title }} | ديني</title>
<link rel="manifest" href="/manifest.json">
<link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='0.9em' font-size='90'>🕌</text></svg>">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Amiri:ital,wght@0,400;0,700;1,400&family=Scheherazade+New:wght@400;500;700&family=Tajawal:wght@300;400;500;700;800&display=swap" rel="stylesheet">
<style>
:root{
  --bg:#0b1512; --bg2:#0f1d18; --card:#122419; --card2:#173022;
  --gold:#d4af6a; --gold2:#e8c987; --green:#2e8b6a; --green2:#3ba37d;
  --text:#e9f2ec; --muted:#9db8a8; --border:#234534;
}
*{margin:0;padding:0;box-sizing:border-box;-webkit-tap-highlight-color:transparent}
body{background:radial-gradient(1200px 600px at 50% -10%,#16352a 0%,var(--bg) 55%);color:var(--text);
  font-family:'Tajawal',sans-serif;min-height:100vh;padding-bottom:40px}
a{color:inherit;text-decoration:none}
.wrap{max-width:720px;margin:0 auto;padding:0 16px}
/* header */
.site-head{text-align:center;padding:34px 0 10px}
.site-head .moon{font-size:34px;filter:drop-shadow(0 0 18px rgba(212,175,106,.45))}
.site-head h1{font-family:'Amiri',serif;font-size:34px;font-weight:700;color:var(--gold2);
  letter-spacing:.5px;margin-top:4px}
.site-head p{color:var(--muted);font-size:14px;margin-top:4px}
.ornament{display:flex;align-items:center;gap:12px;margin:18px auto;max-width:340px}
.ornament .line{flex:1;height:1px;background:linear-gradient(to left,transparent,var(--gold),transparent)}
.ornament .star{color:var(--gold);font-size:14px}
/* top bar for inner pages */
.topbar{position:sticky;top:0;z-index:20;background:rgba(11,21,18,.92);backdrop-filter:blur(10px);
  border-bottom:1px solid var(--border);padding:12px 0;margin-bottom:18px}
.topbar .wrap{display:flex;align-items:center;gap:12px}
.topbar .back{width:36px;height:36px;border-radius:12px;background:var(--card);border:1px solid var(--border);
  display:flex;align-items:center;justify-content:center;font-size:16px;color:var(--gold)}
.topbar h2{font-family:'Amiri',serif;font-size:20px;color:var(--gold2);font-weight:700;flex:1;text-align:center}
.topbar .spacer{width:36px}
/* cards grid */
.grid{display:grid;grid-template-columns:1fr 1fr;gap:14px;padding:8px 0}
.card{background:linear-gradient(160deg,var(--card2),var(--card));border:1px solid var(--border);
  border-radius:22px;padding:24px 14px;text-align:center;position:relative;overflow:hidden;
  transition:transform .15s ease, border-color .2s}
.card:active{transform:scale(.97)}
.card::before{content:"";position:absolute;inset:0;
  background:radial-gradient(200px 90px at 50% -20%,rgba(212,175,106,.14),transparent);pointer-events:none}
.card .ic{font-size:30px;margin-bottom:10px;filter:drop-shadow(0 2px 8px rgba(0,0,0,.4))}
.card h3{font-size:17px;font-weight:700;color:var(--text)}
.card p{font-size:12px;color:var(--muted);margin-top:5px;line-height:1.5}
/* list rows */
.list{display:flex;flex-direction:column;gap:10px;padding:6px 0 20px}
.row{display:flex;align-items:center;gap:12px;background:linear-gradient(160deg,var(--card2),var(--card));
  border:1px solid var(--border);border-radius:16px;padding:13px 14px;transition:border-color .2s}
.row:active{border-color:var(--gold)}
.num{min-width:38px;height:38px;flex:0 0 38px;display:flex;align-items:center;justify-content:center;
  color:var(--gold);font-weight:700;font-size:15px;
  background:radial-gradient(circle,#1b3527 62%,transparent 63%);border:1px solid var(--border);border-radius:50%}
.row .meta{flex:1;min-width:0}
.row .meta b{display:block;font-family:'Amiri',serif;font-size:19px;color:var(--text)}
.row .meta span{font-size:12px;color:var(--muted)}
.row .tag{font-size:11px;color:var(--gold);background:rgba(212,175,106,.1);border:1px solid rgba(212,175,106,.25);
  padding:3px 9px;border-radius:20px;white-space:nowrap}
/* search */
.search{position:relative;margin:4px 0 16px}
.search input{width:100%;background:var(--card);border:1px solid var(--border);border-radius:14px;
  padding:13px 44px 13px 14px;color:var(--text);font-size:15px;font-family:'Tajawal',sans-serif;outline:none}
.search input:focus{border-color:var(--gold)}
.search input::placeholder{color:var(--muted)}
.search .ic{position:absolute;right:14px;top:50%;transform:translateY(-50%);color:var(--muted);font-size:15px}
/* quran page */
.bismillah{text-align:center;font-family:'Scheherazade New',serif;font-size:27px;color:var(--gold2);
  padding:20px 0 8px}
.surah-head{text-align:center;padding:8px 0 4px}
.surah-head .sname{font-family:'Amiri',serif;font-size:26px;color:var(--gold2)}
.surah-head .sinfo{font-size:12.5px;color:var(--muted);margin-top:4px}
.quran-text{font-family:'Scheherazade New',serif;font-size:calc(24px * var(--rf,1));line-height:2.15;text-align:justify;
  background:linear-gradient(160deg,var(--card2),var(--card));border:1px solid var(--border);
  border-radius:20px;padding:22px 18px;margin-top:12px}
.aya{color:var(--text)}
.aya-num{display:inline-flex;align-items:center;justify-content:center;min-width:30px;height:30px;
  margin:0 4px;font-size:14px;color:var(--gold);border:1px solid rgba(212,175,106,.45);border-radius:50%;
  font-family:'Tajawal',sans-serif;vertical-align:middle;line-height:1}
.pagenav{display:flex;gap:10px;margin-top:18px}
.pagenav a{flex:1;text-align:center;background:var(--card);border:1px solid var(--border);
  border-radius:14px;padding:12px;font-size:14px;color:var(--gold2)}
.pagenav a.dis{opacity:.35;pointer-events:none}
.audio-bar{display:flex;align-items:center;gap:12px;background:var(--card);border:1px solid var(--border);
  border-radius:16px;padding:12px 14px;margin-top:12px}
.audio-bar audio{width:100%;height:38px}
.audio-bar .who{font-size:12px;color:var(--muted);white-space:nowrap}
/* hadith */
.hcard{background:linear-gradient(160deg,var(--card2),var(--card));border:1px solid var(--border);
  border-radius:18px;padding:18px;margin-bottom:12px}
.hcard h4{font-family:'Amiri',serif;font-size:18px;color:var(--gold2);margin-bottom:10px;
  display:flex;align-items:center;gap:10px}
.hcard h4 .num{min-width:32px;height:32px;flex-basis:32px;font-size:13px}
.hcard p{font-family:'Amiri',serif;font-size:calc(17.5px * var(--rf,1));line-height:1.95;color:var(--text);text-align:justify}
/* prayer */
.times{display:flex;flex-direction:column;gap:10px;margin-top:14px}
.time-row{display:flex;align-items:center;justify-content:space-between;background:var(--card);
  border:1px solid var(--border);border-radius:16px;padding:15px 18px}
.time-row .pname{font-size:16px;font-weight:700}
.time-row .ptime{font-size:16px;color:var(--gold2);font-weight:700;font-variant-numeric:tabular-nums}
.time-row.next{background:linear-gradient(160deg,#1d4030,#173022);border-color:var(--green2);
  box-shadow:0 0 24px rgba(46,139,106,.18)}
.time-row.next .pname::after{content:" · الصلاة القادمة";font-size:11px;color:var(--green2);font-weight:400}
.hijri{text-align:center;color:var(--muted);font-size:13.5px;margin-top:10px}
.hijri b{color:var(--gold2)}
.city-form{display:flex;gap:8px;margin-top:6px}
.city-form input{flex:1;background:var(--card);border:1px solid var(--border);border-radius:12px;
  padding:11px 14px;color:var(--text);font-size:14px;font-family:'Tajawal',sans-serif;outline:none}
.city-form input:focus{border-color:var(--gold)}
.btn{background:linear-gradient(160deg,var(--gold),#b8934f);color:#1a1305;border:none;border-radius:12px;
  padding:11px 18px;font-size:14px;font-weight:700;font-family:'Tajawal',sans-serif;cursor:pointer}
/* adhkar */
.tabs{display:flex;gap:8px;margin:6px 0 16px;background:var(--card);border:1px solid var(--border);
  border-radius:14px;padding:5px}
.tabs a{flex:1;text-align:center;padding:10px;border-radius:10px;font-size:14px;color:var(--muted);font-weight:500}
.tabs a.on{background:linear-gradient(160deg,#1d4030,#173022);color:var(--gold2);border:1px solid var(--green2)}
.dhikr{background:linear-gradient(160deg,var(--card2),var(--card));border:1px solid var(--border);
  border-radius:18px;padding:18px;margin-bottom:12px;position:relative}
.dhikr p{font-family:'Amiri',serif;font-size:calc(18px * var(--rf,1));line-height:2;text-align:justify;color:var(--text)}
.dhikr .count{position:absolute;top:12px;left:12px;font-size:11.5px;color:var(--gold);
  background:rgba(212,175,106,.1);border:1px solid rgba(212,175,106,.3);padding:3px 10px;border-radius:20px}
.dhikr .fadl{font-size:12.5px;color:var(--muted);margin-top:10px;line-height:1.7;font-family:'Tajawal',sans-serif}
/* tasbih */
.pills{display:flex;gap:8px;flex-wrap:wrap;margin-bottom:14px}
.pill{background:var(--card);border:1px solid var(--border);border-radius:22px;padding:9px 16px;
  font-size:14px;color:var(--muted);cursor:pointer;font-family:'Tajawal',sans-serif;transition:all .15s}
.pill.on{background:linear-gradient(160deg,#1d4030,#173022);color:var(--gold2);border-color:var(--green2)}
.tasbih-stage{text-align:center;margin-top:6px}
.dhikr-display{font-family:'Amiri',serif;font-size:30px;color:var(--gold2);min-height:48px;
  margin:10px 0 2px;line-height:1.6}
.tap-circle{width:250px;height:250px;border-radius:50%;margin:18px auto;cursor:pointer;user-select:none;
  background:radial-gradient(circle at 35% 30%,#1e4432,#122419 70%);border:2px solid var(--green);
  display:flex;flex-direction:column;align-items:center;justify-content:center;
  box-shadow:0 0 60px rgba(46,139,106,.25),inset 0 0 40px rgba(0,0,0,.35);transition:transform .08s}
.tap-circle:active{transform:scale(.96)}
.tap-circle .cnt{font-size:72px;font-weight:800;color:var(--gold2);font-variant-numeric:tabular-nums;line-height:1.1}
.tap-circle .of{font-size:14px;color:var(--muted);margin-top:4px}
.tap-circle .hint{font-size:12px;color:var(--green2);margin-top:10px}
.tasbih-actions{display:flex;gap:10px;justify-content:center;margin-top:8px}
.tasbih-actions .btn.ghost{background:var(--card);color:var(--gold2);border:1px solid var(--border)}
.progress-line{height:6px;background:var(--card);border-radius:6px;overflow:hidden;margin:16px auto 0;max-width:340px}
.progress-line i{display:block;height:100%;width:0%;background:linear-gradient(to left,var(--gold),var(--green2));transition:width .2s}
.done-msg{display:none;margin:16px auto 0;max-width:340px;background:linear-gradient(160deg,#1d4030,#173022);
  border:1px solid var(--green2);border-radius:16px;padding:16px;color:var(--text);font-size:15px;line-height:1.8}
.custom-row{display:none;gap:8px;margin-bottom:14px}
.custom-row input{flex:1;background:var(--card);border:1px solid var(--border);border-radius:12px;
  padding:10px 14px;color:var(--text);font-size:14px;font-family:'Tajawal',sans-serif;outline:none}
.custom-row input:focus{border-color:var(--gold)}
/* offline prep */
.prep{background:linear-gradient(160deg,var(--card2),var(--card));border:1px solid var(--border);
  border-radius:16px;padding:14px 16px;margin-bottom:16px}
.prep .ptxt{font-size:13.5px;color:var(--muted);line-height:1.7}
.prep .ptxt b{color:var(--gold2)}
.prep .prow{display:flex;align-items:center;gap:10px;margin-top:10px}
.prep .pbtn{background:linear-gradient(160deg,var(--gold),#b8934f);color:#1a1305;border:none;
  border-radius:12px;padding:9px 18px;font-size:13.5px;font-weight:700;cursor:pointer;
  font-family:'Tajawal',sans-serif;white-space:nowrap}
.prep .pbtn:disabled{opacity:.5;cursor:default}
.prep .ptrack{flex:1;height:6px;background:#0d1b16;border-radius:6px;overflow:hidden}
.prep .ptrack i{display:block;height:100%;width:0%;background:linear-gradient(to left,var(--gold),var(--green2));transition:width .2s}
.prep.ready{border-color:var(--green2)}
.prep.ready .ptxt{color:var(--green2)}
/* font controls */
.font-ctrl{position:fixed;bottom:18px;left:14px;z-index:50;display:flex;gap:6px;
  background:rgba(11,21,18,.92);border:1px solid var(--border);border-radius:22px;padding:5px;
  backdrop-filter:blur(8px);box-shadow:0 4px 20px rgba(0,0,0,.4)}
.font-ctrl button{width:40px;height:36px;border-radius:16px;border:1px solid var(--border);
  background:var(--card);color:var(--gold2);font-size:15px;font-weight:800;cursor:pointer;
  font-family:'Tajawal',sans-serif}
.font-ctrl button:active{background:var(--green);color:#fff}
/* footer / misc */
.foot{text-align:center;color:var(--muted);font-size:12.5px;margin-top:34px;padding-top:18px;
  border-top:1px solid var(--border)}
.foot b{color:var(--gold)}
.dedic{font-family:'Amiri',serif;font-size:13.5px;color:var(--gold);opacity:.8;margin-top:7px;line-height:1.7}
.err{background:var(--card);border:1px solid var(--border);border-radius:16px;padding:22px;
  text-align:center;color:var(--muted);font-size:14.5px;line-height:1.8;margin-top:14px}
@media (min-width:560px){.grid{grid-template-columns:repeat(4,1fr)}.card h3{font-size:15px}}
</style>
</head>
<body>
{% block body %}{% endblock %}
<script>
if ('serviceWorker' in navigator) { navigator.serviceWorker.register('/sw.js').catch(function(){}); }
(function(){
  var rf = 1;
  try { rf = parseFloat(localStorage.getItem('deen_rf')) || 1; } catch(e) {}
  function apply(){ document.documentElement.style.setProperty('--rf', rf); }
  window.deenFont = function(d){
    rf = Math.min(1.8, Math.max(0.7, Math.round((rf + d) * 10) / 10));
    try { localStorage.setItem('deen_rf', rf); } catch(e) {}
    apply();
  };
  apply();
})();
</script>
</body>
</html>"""

from jinja2 import DictLoader
app.jinja_loader = DictLoader({"base": BASE})

from markupsafe import Markup
FOOT = Markup("""<div class="foot">صُنع بـ ❤️ بواسطة <b>hassan_ahmed</b>\n<div class="dedic">صدقة جارية على روح ستّي وسيّدي الله يرحمهم</div></div>""")

READCTRL = Markup("""<div class="font-ctrl">
<button onclick="deenFont(-0.1)" title="تصغير الخط">أ−</button>
<button onclick="deenFont(0.1)" title="تكبير الخط">أ+</button>
</div>""")

# ---------------- Home ----------------
HOME = """{% extends "base" %}
{% block body %}
<div class="wrap">
  <div class="site-head">
    <div class="moon">🕌</div>
    <h1>ديني</h1>
    <p>رفيقك اليومي: قرآن، حديث، مواقيت الصلاة، وأذكار</p>
    <div class="ornament"><div class="line"></div><div class="star">✦</div><div class="line"></div></div>
  </div>
  <div class="grid">
    <a class="card" href="/quran"><div class="ic">📖</div><h3>القرآن الكريم</h3><p>المصحف كاملًا بتلاوة صوتية، ويمكن تجهيزه كاملًا للقراءة بدون نت</p></a>
    <a class="card" href="/hadith"><div class="ic">📜</div><h3>مكتبة الحديث</h3><p>صحيح البخاري ومسلم كاملًا + الأربعون النووية والأحاديث القدسية</p></a>
    <a class="card" href="/prayer"><div class="ic">🕋</div><h3>مواقيت الصلاة</h3><p>مواقيت دقيقة لمدينتك مع التاريخ الهجري</p></a>
    <a class="card" href="/adhkar"><div class="ic">📿</div><h3>الأذكار</h3><p>أذكار الصباح والمساء مرتبة وسهلة القراءة</p></a>
    <a class="card" href="/tasbih"><div class="ic">☝️</div><h3>السبحة</h3><p>سبحة إلكترونية بعدّاد وأهداف، تنبّهك لما تخلّص</p></a>
    <a class="card" href="/duas"><div class="ic">🤲</div><h3>الأدعية</h3><p>أدعية مأثورة لكل مواقف يومك، من النوم للسفر</p></a>
  </div>
  {{ foot }}
</div>
{% endblock %}"""

@app.route("/")
def home():
    return render_template_string(HOME, base=BASE, title="الرئيسية", foot=FOOT)

# ---------------- Quran ----------------
def get_surah_list():
    data = fetch_json(QURAN_API + "/surah", cache_key="surah_list", ttl=86400)
    if data and data.get("code") == 200:
        return data["data"]
    return None

def get_surah(n):
    data = fetch_json(QURAN_API + "/surah/%d/quran-uthmani" % n, cache_key="surah_%d" % n, ttl=86400)
    if data and data.get("code") == 200:
        return data["data"]
    return None

QURAN_INDEX = """{% extends "base" %}
{% block body %}
<div class="topbar"><div class="wrap">
  <a class="back" href="/">←</a><h2>القرآن الكريم</h2><div class="spacer"></div>
</div></div>
<div class="wrap">
  <div class="search"><input id="q" type="text" placeholder="ابحث عن سورة..." oninput="filt()"><span class="ic">🔍</span></div>
  <div class="prep" id="prepBox">
    <div class="ptxt" id="prepTxt"></div>
    <div class="prow">
      <button class="pbtn" id="prepBtn" onclick="startPrep()"></button>
      <div class="ptrack"><i id="prepBar"></i></div>
    </div>
  </div>
  {% if surahs %}
  <div class="list" id="list">
    {% for s in surahs %}
    <a class="row" href="/quran/{{ s.number }}" data-name="{{ s.name }} {{ s.englishName }}">
      <div class="num">{{ s.number }}</div>
      <div class="meta"><b>{{ s.name }}</b><span>{{ s.englishName }} · {{ s.numberOfAyahs }} آية</span></div>
      <div class="tag">{{ 'مكية' if s.revelationType == 'Meccan' else 'مدنية' }}</div>
    </a>
    {% endfor %}
  </div>
  {% else %}
  <div class="err">تعذّر تحميل فهرس السور حاليًا.<br>تأكد من اتصالك بالإنترنت وحاول مرة أخرى 🔄</div>
  {% endif %}
  {{ foot }}
</div>
<script>
function filt(){
  var q = document.getElementById('q').value.trim();
  document.querySelectorAll('#list .row').forEach(function(r){
    r.style.display = r.dataset.name.indexOf(q) > -1 ? '' : 'none';
  });
}
var PREP_TOTAL = 114, prepBusy = false;
function prepLoad(){
  try { return JSON.parse(localStorage.getItem('deen_prep')) || {done: 0, ready: false}; }
  catch(e) { return {done: 0, ready: false}; }
}
function prepSave(s){ try { localStorage.setItem('deen_prep', JSON.stringify(s)); } catch(e) {} }
function prepUI(){
  var s = prepLoad(), box = document.getElementById('prepBox'),
      txt = document.getElementById('prepTxt'), btn = document.getElementById('prepBtn'),
      bar = document.getElementById('prepBar');
  box.classList.toggle('ready', !!s.ready);
  bar.style.width = Math.round(s.done / PREP_TOTAL * 100) + '%';
  if (s.ready) {
    txt.innerHTML = '<b>✅ القرآن كامل جاهز للقراءة بدون إنترنت</b><br>كل الـ 114 سورة محفوظة على جهازك';
    btn.style.display = 'none';
  } else if (prepBusy) {
    txt.innerHTML = '<b>⏳ جاري تجهيز القرآن للقراءة بدون نت...</b><br>تم تحميل ' + s.done + ' من ' + PREP_TOTAL + ' سورة - خليك على الصفحة دي';
    btn.disabled = true; btn.textContent = 'جاري التحميل...';
  } else if (s.done > 0) {
    txt.innerHTML = '<b>⚠️ التجهيز وقف عند سورة ' + s.done + '</b><br>وصّل الإنترنت ودوس استكمال عشان يكمّل من مكانه';
    btn.disabled = false; btn.textContent = '▶ استكمال';
  } else {
    txt.innerHTML = '<b>📥 جهّز القرآن للقراءة بدون نت</b><br>تحميل مرة واحدة (حوالي 15 ميجا) وبعدها تقرأ أي سورة حتى لو النت مقطوع';
    btn.disabled = false; btn.textContent = 'تحميل';
  }
}
async function startPrep(){
  if (prepBusy) return;
  prepBusy = true; prepUI();
  var s = prepLoad();
  try {
    for (var i = s.done + 1; i <= PREP_TOTAL; i++) {
      var ctrl = new AbortController();
      var to = setTimeout(function(){ ctrl.abort(); }, 15000);
      var resp = await fetch('/quran/' + i, {signal: ctrl.signal});
      clearTimeout(to);
      if (!resp.ok) throw new Error('http ' + resp.status);
      var body = await resp.text();
      // make sure we really got the surah page, not an offline fallback
      // (marker split so this script's own source can never match it)
      var ayaMark = '<span class="ay' + 'a"';
      if (body.indexOf(ayaMark) < 0) throw new Error('not surah content');
      s.done = i; prepSave(s); prepUI();
    }
    s.ready = true; prepSave(s);
  } catch(e) {
    // offline or server hiccup: state is saved, user resumes later
  }
  prepBusy = false; prepUI();
}
prepUI();
</script>
{% endblock %}"""

SURAH_PAGE = """{% extends "base" %}
{% block body %}
<div class="topbar"><div class="wrap">
  <a class="back" href="/quran">←</a><h2>{{ s.name }}</h2><div class="spacer"></div>
</div></div>
<div class="wrap">
  <div class="surah-head">
    <div class="sname">{{ s.name }}</div>
    <div class="sinfo">{{ 'مكية' if s.revelationType == 'Meccan' else 'مدنية' }} · {{ s.numberOfAyahs }} آية</div>
  </div>
  <div class="audio-bar">
    <audio controls preload="none" src="{{ audio }}"></audio>
    <span class="who">🎧 مشاري العفاسي</span>
  </div>
  {% if show_bismillah %}<div class="bismillah">بِسْمِ اللَّهِ الرَّحْمَٰنِ الرَّحِيمِ</div>{% endif %}
  <div class="quran-text">
    {%- for a in ayahs -%}
      <span class="aya">{{ a.text }}</span><span class="aya-num">{{ a.numberInSurah }}</span>
    {%- endfor -%}
  </div>
  <div class="pagenav">
    <a href="/quran/{{ s.number - 1 }}" class="{{ 'dis' if s.number == 1 }}">‹ السورة السابقة</a>
    <a href="/quran/{{ s.number + 1 }}" class="{{ 'dis' if s.number == 114 }}">السورة التالية ›</a>
  </div>
  {{ readctrl }}{{ foot }}
</div>
{% endblock %}"""

BISMILLAH_PREFIX = "بِسْمِ ٱللَّهِ ٱلرَّحْمَٰنِ ٱلرَّحِيمِ"

@app.route("/quran")
def quran_index():
    return render_template_string(QURAN_INDEX, base=BASE, title="القرآن الكريم",
                                  surahs=get_surah_list(), foot=FOOT)

@app.route("/quran/<int:n>")
def quran_surah(n):
    if n < 1 or n > 114:
        return render_template_string(QURAN_INDEX, base=BASE, title="القرآن الكريم",
                                      surahs=get_surah_list(), foot=FOOT), 404
    s = get_surah(n)
    if not s:
        return render_template_string("""{% extends "base" %}{% block body %}
<div class="topbar"><div class="wrap"><a class="back" href="/quran">←</a><h2>القرآن الكريم</h2><div class="spacer"></div></div></div>
<div class="wrap"><div class="err">تعذّر تحميل السورة حاليًا.<br>تأكد من اتصالك بالإنترنت وحاول مرة أخرى 🔄</div></div>
{% endblock %}""", base=BASE, title="خطأ"), 503
    ayahs = list(s["ayahs"])
    # Remove the duplicated bismillah from the first ayah (we render it separately)
    if n != 1 and n != 9 and ayahs and ayahs[0]["text"].lstrip("﻿").startswith(BISMILLAH_PREFIX[:8]):
        t = ayahs[0]["text"].lstrip("﻿")
        if t.startswith(BISMILLAH_PREFIX):
            ayahs[0] = dict(ayahs[0], text=t[len(BISMILLAH_PREFIX):].strip())
    return render_template_string(SURAH_PAGE, base=BASE, title=s["name"], s=s,
                                  ayahs=ayahs, show_bismillah=(n not in (1, 9)),
                                  audio="%s/%d.mp3" % (AUDIO_BASE, n), foot=FOOT, readctrl=READCTRL)

# ---------------- Hadith (مكتبة الحديث) ----------------
HADITH_BOOKS = {
    "nawawi": {"name": "الأربعون النووية", "desc": "42 حديثًا جامعًا من جوامع الكلم",
               "url": "https://cdn.jsdelivr.net/gh/fawazahmed0/hadith-api@1/editions/ara-nawawi.min.json",
               "paged": False, "note": "جُمعت هذه الأحاديث من صحيحي البخاري ومسلم وغيرهما، اختارها الإمام النووي رحمه الله"},
    "qudsi": {"name": "الأحاديث القدسية", "desc": "40 حديثًا قدسيًا رواها النبي ﷺ عن ربه",
              "url": "https://cdn.jsdelivr.net/gh/fawazahmed0/hadith-api@1/editions/ara-qudsi.min.json",
              "paged": False, "note": "الحديث القدسي: ما رواه النبي ﷺ عن الله عز وجل"},
    "bukhari": {"name": "صحيح البخاري", "desc": "أصح كتاب بعد كتاب الله - كاملًا بالترقيم المعتمد",
                "url": "https://cdn.jsdelivr.net/gh/fawazahmed0/hadith-api@1/editions/ara-bukhari.min.json",
                "paged": True, "note": "صحيح الإمام محمد بن إسماعيل البخاري رحمه الله"},
    "muslim": {"name": "صحيح مسلم", "desc": "صحيح الإمام مسلم بن الحجاج النيسابوري كاملًا",
               "url": "https://cdn.jsdelivr.net/gh/fawazahmed0/hadith-api@1/editions/ara-muslim.min.json",
               "paged": True, "note": "صحيح الإمام مسلم بن الحجاج النيسابوري رحمه الله"},
}
HADITH_PAGE_SIZE = 50

def get_hadiths(book):
    b = HADITH_BOOKS.get(book)
    if not b:
        return None
    data = fetch_json(b["url"], cache_key="hadith_" + book, ttl=86400)
    if data and "hadiths" in data:
        return data["hadiths"]
    return None

HADITH_INDEX = """{% extends "base" %}
{% block body %}
<div class="topbar"><div class="wrap">
  <a class="back" href="/">←</a><h2>مكتبة الحديث</h2><div class="spacer"></div>
</div></div>
<div class="wrap">
  <div class="hijri" style="margin-bottom:14px">كتب الحديث الصحيحة بمتونها المعتمدة 📜</div>
  <div class="list">
    {% for key, b in books.items() %}
    <a class="row" href="/hadith/{{ key }}">
      <div class="num">📖</div>
      <div class="meta"><b>{{ b.name }}</b><span>{{ b.desc }}</span></div>
      <div class="tag">تصفّح</div>
    </a>
    {% endfor %}
  </div>
  {{ foot }}
</div>
{% endblock %}"""

HADITH_PAGE = """{% extends "base" %}
{% block body %}
<div class="topbar"><div class="wrap">
  <a class="back" href="/hadith">←</a><h2>{{ book.name }}</h2><div class="spacer"></div>
</div></div>
<div class="wrap">
  <div class="hijri" style="margin-bottom:14px">{{ book.note }}</div>
  {% if hadiths %}
    {% for h in hadiths %}
    <div class="hcard">
      <h4><span class="num">{{ h.number }}</span> حديث رقم {{ h.number }}</h4>
      <p>{{ h.text }}</p>
    </div>
    {% endfor %}
    {% if book.paged %}
    <div class="pagenav">
      <a href="/hadith/{{ book_key }}?page={{ page - 1 }}" class="{{ 'dis' if page <= 1 }}">‹ الصفحة السابقة</a>
      <a href="/hadith/{{ book_key }}?page={{ page + 1 }}" class="{{ 'dis' if not has_next }}">الصفحة التالية ›</a>
    </div>
    <div class="hijri" style="margin-top:10px">صفحة {{ page }} من {{ total_pages }} · إجمالي {{ total }} حديث</div>
    {% endif %}
  {% else %}
  <div class="err">تعذّر تحميل الأحاديث حاليًا.<br>تأكد من اتصالك بالإنترنت وحاول مرة أخرى 🔄</div>
  {% endif %}
  {{ readctrl }}{{ foot }}
</div>
{% endblock %}"""

@app.route("/hadith")
def hadith():
    return render_template_string(HADITH_INDEX, base=BASE, title="مكتبة الحديث",
                                  books=HADITH_BOOKS, foot=FOOT)

@app.route("/hadith/<book>")
def hadith_book(book):
    b = HADITH_BOOKS.get(book)
    if not b:
        return render_template_string(HADITH_INDEX, base=BASE, title="مكتبة الحديث",
                                      books=HADITH_BOOKS, foot=FOOT), 404
    hadiths = get_hadiths(book)
    page = 1
    has_next = False
    total = total_pages = 0
    if hadiths:
        total = len(hadiths)
        if b["paged"]:
            total_pages = (total + HADITH_PAGE_SIZE - 1) // HADITH_PAGE_SIZE
            try:
                page = max(1, min(total_pages, int(request.args.get("page", "1"))))
            except ValueError:
                page = 1
            s = (page - 1) * HADITH_PAGE_SIZE
            hadiths = hadiths[s:s + HADITH_PAGE_SIZE]
            has_next = page < total_pages
        for h in hadiths:
            h["number"] = h.get("arabicnumber") or h.get("hadithnumber")
    return render_template_string(HADITH_PAGE, base=BASE, title=b["name"], book=b,
                                  book_key=book, hadiths=hadiths, page=page,
                                  has_next=has_next, total=total,
                                  total_pages=total_pages, foot=FOOT, readctrl=READCTRL)

# ---------------- Prayer times ----------------
PRAYER_NAMES = [("Fajr","الفجر"),("Sunrise","الشروق"),("Dhuhr","الظهر"),
                ("Asr","العصر"),("Maghrib","المغرب"),("Isha","العشاء")]
MAIN_PRAYERS = ["Fajr","Dhuhr","Asr","Maghrib","Isha"]

PRAYER_PAGE = """{% extends "base" %}
{% block body %}
<div class="topbar"><div class="wrap">
  <a class="back" href="/">←</a><h2>مواقيت الصلاة</h2><div class="spacer"></div>
</div></div>
<div class="wrap">
  <form class="city-form" method="get" action="/prayer">
    <input name="city" value="{{ city }}" placeholder="اكتب اسم مدينتك...">
    <button class="btn" type="submit">عرض</button>
  </form>
  {% if t %}
  <div class="hijri">📅 <b>{{ t.hijri.weekday.ar }} {{ t.hijri.day }} {{ t.hijri.month.ar }} {{ t.hijri.year }}هـ</b> · الموافق {{ t.greg }}</div>
  <div class="times">
    {% for key, name in rows %}
    <div class="time-row {{ 'next' if key == next_p }}">
      <span class="pname">{{ name }}</span><span class="ptime">{{ to12(t.timings[key]) }}</span>
    </div>
    {% endfor %}
  </div>
  <div class="hijri" style="margin-top:14px">📍 {{ city }}{% if country %}, {{ country }}{% endif %}</div>
  {% else %}
  <div class="err">تعذّر جلب المواقيت لمدينة «{{ city }}».<br>جرّب كتابة اسم المدينة بالإنجليزية، مثل: Cairo 🔄</div>
  {% endif %}
  {{ foot }}
</div>
{% endblock %}"""

def to12(t24):
    """'13:45' -> '1:45 م'"""
    try:
        h, m = t24.split(":")[:2]
        h = int(h)
        suf = "ص" if h < 12 else "م"
        h12 = h % 12 or 12
        return "%d:%s %s" % (h12, m, suf)
    except Exception:
        return t24

def next_prayer(timings):
    from datetime import datetime
    now = datetime.now().strftime("%H:%M")
    for p in MAIN_PRAYERS:
        if timings.get(p, "") >= now:
            return p
    return "Fajr"

app.jinja_env.globals["to12"] = to12

@app.route("/prayer")
def prayer():
    city = request.args.get("city", "Cairo").strip() or "Cairo"
    country = request.args.get("country", "Egypt").strip() or "Egypt"
    data = fetch_json(ALADHAN_API + "/timingsByCity?city=%s&country=%s&method=5" % (
        urllib.request.quote(city), urllib.request.quote(country)),
        cache_key="pt_%s_%s" % (city.lower(), country.lower()), ttl=1800)
    t = None
    if data and data.get("code") == 200:
        d = data["data"]
        t = {"timings": d["timings"], "hijri": d["date"]["hijri"],
             "greg": d["date"]["gregorian"]["date"]}
        for k in list(t["timings"]):
            t["timings"][k] = t["timings"][k].split(" ")[0]
    return render_template_string(PRAYER_PAGE, base=BASE, title="مواقيت الصلاة",
                                  t=t, rows=PRAYER_NAMES, city=city, country=country,
                                  next_p=next_prayer(t["timings"]) if t else None, foot=FOOT)

# ---------------- Adhkar ----------------
ADHKAR = {
"morning": [
 {"t":"أَعُوذُ بِاللَّهِ مِنَ الشَّيْطَانِ الرَّجِيمِ: اللَّهُ لَا إِلَهَ إِلَّا هُوَ الْحَيُّ الْقَيُّومُ، لَا تَأْخُذُهُ سِنَةٌ وَلَا نَوْمٌ، لَهُ مَا فِي السَّمَاوَاتِ وَمَا فِي الْأَرْضِ، مَنْ ذَا الَّذِي يَشْفَعُ عِنْدَهُ إِلَّا بِإِذْنِهِ، يَعْلَمُ مَا بَيْنَ أَيْدِيهِمْ وَمَا خَلْفَهُمْ، وَلَا يُحِيطُونَ بِشَيْءٍ مِنْ عِلْمِهِ إِلَّا بِمَا شَاءَ، وَسِعَ كُرْسِيُّهُ السَّمَاوَاتِ وَالْأَرْضَ، وَلَا يَئُودُهُ حِفْظُهُمَا، وَهُوَ الْعَلِيُّ الْعَظِيمُ","c":"مرة واحدة","f":"آية الكرسي: من قالها حين يصبح أُجير من الجن حتى يمسي"},
 {"t":"قُلْ هُوَ اللَّهُ أَحَدٌ ۝ اللَّهُ الصَّمَدُ ۝ لَمْ يَلِدْ وَلَمْ يُولَدْ ۝ وَلَمْ يَكُنْ لَهُ كُفُوًا أَحَدٌ","c":"3 مرات","f":"سورة الإخلاص"},
 {"t":"قُلْ أَعُوذُ بِرَبِّ الْفَلَقِ ۝ مِنْ شَرِّ مَا خَلَقَ ۝ وَمِنْ شَرِّ غَاسِقٍ إِذَا وَقَبَ ۝ وَمِنْ شَرِّ النَّفَّاثَاتِ فِي الْعُقَدِ ۝ وَمِنْ شَرِّ حَاسِدٍ إِذَا حَسَدَ","c":"3 مرات","f":"سورة الفلق"},
 {"t":"قُلْ أَعُوذُ بِرَبِّ النَّاسِ ۝ مَلِكِ النَّاسِ ۝ إِلَهِ النَّاسِ ۝ مِنْ شَرِّ الْوَسْوَاسِ الْخَنَّاسِ ۝ الَّذِي يُوَسْوِسُ فِي صُدُورِ النَّاسِ ۝ مِنَ الْجِنَّةِ وَالنَّاسِ","c":"3 مرات","f":"سورة الناس"},
 {"t":"أَصْبَحْنَا وَأَصْبَحَ الْمُلْكُ لِلَّهِ، وَالْحَمْدُ لِلَّهِ، لَا إِلَهَ إِلَّا اللَّهُ وَحْدَهُ لَا شَرِيكَ لَهُ، لَهُ الْمُلْكُ وَلَهُ الْحَمْدُ وَهُوَ عَلَى كُلِّ شَيْءٍ قَدِيرٌ، رَبِّ أَسْأَلُكَ خَيْرَ مَا فِي هَذَا الْيَوْمِ وَخَيْرَ مَا بَعْدَهُ، وَأَعُوذُ بِكَ مِنْ شَرِّ مَا فِي هَذَا الْيَوْمِ وَشَرِّ مَا بَعْدَهُ","c":"مرة واحدة"},
 {"t":"اللَّهُمَّ بِكَ أَصْبَحْنَا، وَبِكَ أَمْسَيْنَا، وَبِكَ نَحْيَا، وَبِكَ نَمُوتُ، وَإِلَيْكَ النُّشُورُ","c":"مرة واحدة"},
 {"t":"اللَّهُمَّ أَنْتَ رَبِّي لَا إِلَهَ إِلَّا أَنْتَ، خَلَقْتَنِي وَأَنَا عَبْدُكَ، وَأَنَا عَلَى عَهْدِكَ وَوَعْدِكَ مَا اسْتَطَعْتُ، أَعُوذُ بِكَ مِنْ شَرِّ مَا صَنَعْتُ، أَبُوءُ لَكَ بِنِعْمَتِكَ عَلَيَّ، وَأَبُوءُ بِذَنْبِي، فَاغْفِرْ لِي فَإِنَّهُ لَا يَغْفِرُ الذُّنُوبَ إِلَّا أَنْتَ","c":"مرة واحدة","f":"سيد الاستغفار: من قالها موقنًا بها فمات من يومه دخل الجنة"},
 {"t":"بِسْمِ اللَّهِ الَّذِي لَا يَضُرُّ مَعَ اسْمِهِ شَيْءٌ فِي الْأَرْضِ وَلَا فِي السَّمَاءِ وَهُوَ السَّمِيعُ الْعَلِيمُ","c":"3 مرات","f":"لم تضره فجأة آفة حتى يمسي"},
 {"t":"رَضِيتُ بِاللَّهِ رَبًّا، وَبِالْإِسْلَامِ دِينًا، وَبِمُحَمَّدٍ ﷺ نَبِيًّا","c":"3 مرات","f":"كان حقًا على الله أن يرضيه يوم القيامة"},
 {"t":"حَسْبِيَ اللَّهُ لَا إِلَهَ إِلَّا هُوَ، عَلَيْهِ تَوَكَّلْتُ وَهُوَ رَبُّ الْعَرْشِ الْعَظِيمِ","c":"7 مرات","f":"كفاه الله ما أهمه من أمر الدنيا والآخرة"},
 {"t":"اللَّهُمَّ إِنِّي أَسْأَلُكَ الْعَفْوَ وَالْعَافِيَةَ فِي الدُّنْيَا وَالْآخِرَةِ","c":"مرة واحدة"},
 {"t":"سُبْحَانَ اللَّهِ وَبِحَمْدِهِ","c":"100 مرة","f":"حُطَّت خطاياه وإن كانت مثل زبد البحر"},
 {"t":"سُبْحَانَ اللَّهِ الْعَظِيمِ وَبِحَمْدِهِ","c":"10 مرات"},
 {"t":"لَا إِلَهَ إِلَّا اللَّهُ وَحْدَهُ لَا شَرِيكَ لَهُ، لَهُ الْمُلْكُ وَلَهُ الْحَمْدُ وَهُوَ عَلَى كُلِّ شَيْءٍ قَدِيرٌ","c":"100 مرة","f":"كانت له عدل عشر رقاب، وكُتبت له مئة حسنة"},
 {"t":"سُبْحَانَ اللَّهِ وَبِحَمْدِهِ عَدَدَ خَلْقِهِ، وَرِضَا نَفْسِهِ، وَزِنَةَ عَرْشِهِ، وَمِدَادَ كَلِمَاتِهِ","c":"3 مرات"},
 {"t":"اللَّهُمَّ صَلِّ وَسَلِّمْ عَلَى نَبِيِّنَا مُحَمَّدٍ","c":"10 مرات"},
],
"evening": [
 {"t":"أَعُوذُ بِاللَّهِ مِنَ الشَّيْطَانِ الرَّجِيمِ: اللَّهُ لَا إِلَهَ إِلَّا هُوَ الْحَيُّ الْقَيُّومُ، لَا تَأْخُذُهُ سِنَةٌ وَلَا نَوْمٌ، لَهُ مَا فِي السَّمَاوَاتِ وَمَا فِي الْأَرْضِ، مَنْ ذَا الَّذِي يَشْفَعُ عِنْدَهُ إِلَّا بِإِذْنِهِ، يَعْلَمُ مَا بَيْنَ أَيْدِيهِمْ وَمَا خَلْفَهُمْ، وَلَا يُحِيطُونَ بِشَيْءٍ مِنْ عِلْمِهِ إِلَّا بِمَا شَاءَ، وَسِعَ كُرْسِيُّهُ السَّمَاوَاتِ وَالْأَرْضَ، وَلَا يَئُودُهُ حِفْظُهُمَا، وَهُوَ الْعَلِيُّ الْعَظِيمُ","c":"مرة واحدة","f":"آية الكرسي: من قالها حين يمسي أُجير من الجن حتى يصبح"},
 {"t":"قُلْ هُوَ اللَّهُ أَحَدٌ ۝ اللَّهُ الصَّمَدُ ۝ لَمْ يَلِدْ وَلَمْ يُولَدْ ۝ وَلَمْ يَكُنْ لَهُ كُفُوًا أَحَدٌ","c":"3 مرات","f":"سورة الإخلاص"},
 {"t":"قُلْ أَعُوذُ بِرَبِّ الْفَلَقِ ۝ مِنْ شَرِّ مَا خَلَقَ ۝ وَمِنْ شَرِّ غَاسِقٍ إِذَا وَقَبَ ۝ وَمِنْ شَرِّ النَّفَّاثَاتِ فِي الْعُقَدِ ۝ وَمِنْ شَرِّ حَاسِدٍ إِذَا حَسَدَ","c":"3 مرات","f":"سورة الفلق"},
 {"t":"قُلْ أَعُوذُ بِرَبِّ النَّاسِ ۝ مَلِكِ النَّاسِ ۝ إِلَهِ النَّاسِ ۝ مِنْ شَرِّ الْوَسْوَاسِ الْخَنَّاسِ ۝ الَّذِي يُوَسْوِسُ فِي صُدُورِ النَّاسِ ۝ مِنَ الْجِنَّةِ وَالنَّاسِ","c":"3 مرات","f":"سورة الناس"},
 {"t":"أَمْسَيْنَا وَأَمْسَى الْمُلْكُ لِلَّهِ، وَالْحَمْدُ لِلَّهِ، لَا إِلَهَ إِلَّا اللَّهُ وَحْدَهُ لَا شَرِيكَ لَهُ، لَهُ الْمُلْكُ وَلَهُ الْحَمْدُ وَهُوَ عَلَى كُلِّ شَيْءٍ قَدِيرٌ، رَبِّ أَسْأَلُكَ خَيْرَ مَا فِي هَذِهِ اللَّيْلَةِ وَخَيْرَ مَا بَعْدَهَا، وَأَعُوذُ بِكَ مِنْ شَرِّ مَا فِيهَا وَشَرِّ مَا بَعْدَهَا","c":"مرة واحدة"},
 {"t":"اللَّهُمَّ بِكَ أَمْسَيْنَا، وَبِكَ أَصْبَحْنَا، وَبِكَ نَحْيَا، وَبِكَ نَمُوتُ، وَإِلَيْكَ الْمَصِيرُ","c":"مرة واحدة"},
 {"t":"اللَّهُمَّ أَنْتَ رَبِّي لَا إِلَهَ إِلَّا أَنْتَ، خَلَقْتَنِي وَأَنَا عَبْدُكَ، وَأَنَا عَلَى عَهْدِكَ وَوَعْدِكَ مَا اسْتَطَعْتُ، أَعُوذُ بِكَ مِنْ شَرِّ مَا صَنَعْتُ، أَبُوءُ لَكَ بِنِعْمَتِكَ عَلَيَّ، وَأَبُوءُ بِذَنْبِي، فَاغْفِرْ لِي فَإِنَّهُ لَا يَغْفِرُ الذُّنُوبَ إِلَّا أَنْتَ","c":"مرة واحدة","f":"سيد الاستغفار: من قالها موقنًا بها فمات من ليلته دخل الجنة"},
 {"t":"بِسْمِ اللَّهِ الَّذِي لَا يَضُرُّ مَعَ اسْمِهِ شَيْءٌ فِي الْأَرْضِ وَلَا فِي السَّمَاءِ وَهُوَ السَّمِيعُ الْعَلِيمُ","c":"3 مرات","f":"لم تضره فجأة آفة حتى يصبح"},
 {"t":"رَضِيتُ بِاللَّهِ رَبًّا، وَبِالْإِسْلَامِ دِينًا، وَبِمُحَمَّدٍ ﷺ نَبِيًّا","c":"3 مرات"},
 {"t":"أَعُوذُ بِكَلِمَاتِ اللَّهِ التَّامَّاتِ مِنْ شَرِّ مَا خَلَقَ","c":"3 مرات","f":"لم تضره حُمَةٌ تلك الليلة"},
 {"t":"حَسْبِيَ اللَّهُ لَا إِلَهَ إِلَّا هُوَ، عَلَيْهِ تَوَكَّلْتُ وَهُوَ رَبُّ الْعَرْشِ الْعَظِيمِ","c":"7 مرات","f":"كفاه الله ما أهمه من أمر الدنيا والآخرة"},
 {"t":"سُبْحَانَ اللَّهِ وَبِحَمْدِهِ","c":"100 مرة"},
 {"t":"اللَّهُمَّ صَلِّ وَسَلِّمْ عَلَى نَبِيِّنَا مُحَمَّدٍ","c":"10 مرات"},
]}

ADHKAR_PAGE = """{% extends "base" %}
{% block body %}
<div class="topbar"><div class="wrap">
  <a class="back" href="/">←</a><h2>الأذكار</h2><div class="spacer"></div>
</div></div>
<div class="wrap">
  <div class="tabs">
    <a href="/adhkar?t=morning" class="{{ 'on' if tab == 'morning' }}">☀️ أذكار الصباح</a>
    <a href="/adhkar?t=evening" class="{{ 'on' if tab == 'evening' }}">🌙 أذكار المساء</a>
  </div>
  {% for d in items %}
  <div class="dhikr">
    <span class="count">{{ d.c }}</span>
    <p>{{ d.t }}</p>
    {% if d.f %}<div class="fadl">✨ {{ d.f }}</div>{% endif %}
  </div>
  {% endfor %}
  {{ readctrl }}{{ foot }}
</div>
{% endblock %}"""

@app.route("/adhkar")
def adhkar():
    tab = request.args.get("t", "morning")
    if tab not in ADHKAR:
        tab = "morning"
    return render_template_string(ADHKAR_PAGE, base=BASE, title="الأذكار",
                                  tab=tab, items=ADHKAR[tab], foot=FOOT, readctrl=READCTRL)



# ---------------- Duas (الأدعية اليومية) ----------------
DUAS = [
 {"t":"دعاء الاستيقاظ من النوم","d":"الْحَمْدُ لِلَّهِ الَّذِي أَحْيَانَا بَعْدَ مَا أَمَاتَنَا وَإِلَيْهِ النُّشُورُ"},
 {"t":"دعاء النوم","d":"بِاسْمِكَ اللَّهُمَّ أَمُوتُ وَأَحْيَا"},
 {"t":"دعاء دخول الخلاء","d":"اللَّهُمَّ إِنِّي أَعُوذُ بِكَ مِنَ الْخُبُثِ وَالْخَبَائِثِ"},
 {"t":"دعاء الخروج من الخلاء","d":"غُفْرَانَكَ"},
 {"t":"دعاء الخروج من المنزل","d":"بِسْمِ اللَّهِ، تَوَكَّلْتُ عَلَى اللَّهِ، وَلَا حَوْلَ وَلَا قُوَّةَ إِلَّا بِاللَّهِ","f":"يُقال له: هُديتَ وكُفيتَ ووُقيتَ"},
 {"t":"دعاء دخول المنزل","d":"بِسْمِ اللَّهِ وَلَجْنَا، وَبِسْمِ اللَّهِ خَرَجْنَا، وَعَلَى اللَّهِ رَبِّنَا تَوَكَّلْنَا"},
 {"t":"دعاء قبل الأكل","d":"بِسْمِ اللَّهِ وَعَلَى بَرَكَةِ اللَّهِ"},
 {"t":"دعاء بعد الأكل","d":"الْحَمْدُ لِلَّهِ الَّذِي أَطْعَمَنَا وَسَقَانَا وَجَعَلَنَا مِنَ الْمُسْلِمِينَ"},
 {"t":"دعاء دخول المسجد","d":"أَعُوذُ بِاللَّهِ الْعَظِيمِ، وَبِوَجْهِهِ الْكَرِيمِ، وَسُلْطَانِهِ الْقَدِيمِ، مِنَ الشَّيْطَانِ الرَّجِيمِ، اللَّهُمَّ افْتَحْ لِي أَبْوَابَ رَحْمَتِكَ"},
 {"t":"دعاء الخروج من المسجد","d":"اللَّهُمَّ إِنِّي أَسْأَلُكَ مِنْ فَضْلِكَ"},
 {"t":"دعاء السفر","d":"سُبْحَانَ الَّذِي سَخَّرَ لَنَا هَذَا وَمَا كُنَّا لَهُ مُقْرِنِينَ، وَإِنَّا إِلَى رَبِّنَا لَمُنْقَلِبُونَ، اللَّهُمَّ إِنَّا نَسْأَلُكَ فِي سَفَرِنَا هَذَا الْبِرَّ وَالتَّقْوَى، وَمِنَ الْعَمَلِ مَا تَرْضَى"},
 {"t":"دعاء الركوب","d":"بِسْمِ اللَّهِ، الْحَمْدُ لِلَّهِ، سُبْحَانَ الَّذِي سَخَّرَ لَنَا هَذَا وَمَا كُنَّا لَهُ مُقْرِنِينَ، وَإِنَّا إِلَى رَبِّنَا لَمُنْقَلِبُونَ"},
 {"t":"دعاء لبس الثوب","d":"الْحَمْدُ لِلَّهِ الَّذِي كَسَانِي هَذَا الثَّوْبَ وَرَزَقَنِيهِ مِنْ غَيْرِ حَوْلٍ مِنِّي وَلَا قُوَّةٍ"},
 {"t":"دعاء دخول السوق","d":"لَا إِلَهَ إِلَّا اللَّهُ وَحْدَهُ لَا شَرِيكَ لَهُ، لَهُ الْمُلْكُ وَلَهُ الْحَمْدُ، يُحْيِي وَيُمِيتُ، وَهُوَ حَيٌّ لَا يَمُوتُ، بِيَدِهِ الْخَيْرُ، وَهُوَ عَلَى كُلِّ شَيْءٍ قَدِيرٌ"},
 {"t":"دعاء الكرب","d":"لَا إِلَهَ إِلَّا اللَّهُ الْعَظِيمُ الْحَلِيمُ، لَا إِلَهَ إِلَّا اللَّهُ رَبُّ الْعَرْشِ الْعَظِيمِ، لَا إِلَهَ إِلَّا اللَّهُ رَبُّ السَّمَاوَاتِ وَرَبُّ الْأَرْضِ وَرَبُّ الْعَرْشِ الْكَرِيمِ"},
 {"t":"دعاء الهم والحزن","d":"اللَّهُمَّ إِنِّي أَعُوذُ بِكَ مِنَ الْهَمِّ وَالْحَزَنِ، وَالْعَجْزِ وَالْكَسَلِ، وَالْبُخْلِ وَالْجُبْنِ، وَضَلَعِ الدَّيْنِ وَغَلَبَةِ الرِّجَالِ"},
 {"t":"دعاء قضاء الدين","d":"اللَّهُمَّ اكْفِنِي بِحَلَالِكَ عَنْ حَرَامِكَ، وَأَغْنِنِي بِفَضْلِكَ عَمَّنْ سِوَاكَ"},
 {"t":"دعاء رؤية الهلال","d":"اللَّهُمَّ أَهِلَّهُ عَلَيْنَا بِالْأَمْنِ وَالْإِيمَانِ، وَالسَّلَامَةِ وَالْإِسْلَامِ، رَبِّي وَرَبُّكَ اللَّهُ"},
 {"t":"دعاء الصلاة على النبي ﷺ","d":"اللَّهُمَّ صَلِّ عَلَى مُحَمَّدٍ وَعَلَى آلِ مُحَمَّدٍ، كَمَا صَلَّيْتَ عَلَى إِبْرَاهِيمَ وَعَلَى آلِ إِبْرَاهِيمَ، إِنَّكَ حَمِيدٌ مَجِيدٌ، اللَّهُمَّ بَارِكْ عَلَى مُحَمَّدٍ وَعَلَى آلِ مُحَمَّدٍ، كَمَا بَارَكْتَ عَلَى إِبْرَاهِيمَ وَعَلَى آلِ إِبْرَاهِيمَ، إِنَّكَ حَمِيدٌ مَجِيدٌ"},
 {"t":"دعاء ختم المجلس","d":"سُبْحَانَكَ اللَّهُمَّ وَبِحَمْدِكَ، أَشْهَدُ أَنْ لَا إِلَهَ إِلَّا أَنْتَ، أَسْتَغْفِرُكَ وَأَتُوبُ إِلَيْكَ"},
]

DUAS_PAGE = """{% extends "base" %}
{% block body %}
<div class="topbar"><div class="wrap">
  <a class="back" href="/">←</a><h2>الأدعية اليومية</h2><div class="spacer"></div>
</div></div>
<div class="wrap">
  <div class="hijri" style="margin-bottom:14px">أدعية مأثورة من القرآن والسنة لكل مواقف يومك 🤲</div>
  {% for x in duas %}
  <div class="dhikr">
    <h4 style="font-family:'Amiri',serif;font-size:17px;color:var(--gold2);margin-bottom:10px">🤲 {{ x.t }}</h4>
    <p>{{ x.d }}</p>
    {% if x.f %}<div class="fadl">✨ {{ x.f }}</div>{% endif %}
  </div>
  {% endfor %}
  {{ readctrl }}{{ foot }}
</div>
{% endblock %}"""

@app.route("/duas")
def duas():
    return render_template_string(DUAS_PAGE, base=BASE, title="الأدعية اليومية", duas=DUAS, foot=FOOT, readctrl=READCTRL)

# ---------------- Tasbih (السبحة الإلكترونية) ----------------
TASBIH_PAGE = """{% extends "base" %}
{% block body %}
<div class="topbar"><div class="wrap">
  <a class="back" href="/">←</a><h2>السبحة الإلكترونية</h2><div class="spacer"></div>
</div></div>
<div class="wrap">
  <div class="pills" id="dhikrPills">
    <button class="pill on" data-d="سُبْحَانَ اللَّهِ">سبحان الله</button>
    <button class="pill" data-d="الْحَمْدُ لِلَّهِ">الحمد لله</button>
    <button class="pill" data-d="اللَّهُ أَكْبَرُ">الله أكبر</button>
    <button class="pill" data-d="لَا إِلَهَ إِلَّا اللَّهُ">لا إله إلا الله</button>
    <button class="pill" data-d="أَسْتَغْفِرُ اللَّهَ">أستغفر الله</button>
    <button class="pill" data-d="custom">✏️ ذكر خاص</button>
  </div>
  <div class="custom-row" id="customRow">
    <input id="customDhikr" maxlength="60" placeholder="اكتب الذكر اللي تحب تسبّح به...">
    <button class="btn" onclick="setCustom()">تم</button>
  </div>
  <div class="pills" id="targetPills">
    <button class="pill on" data-t="33">هدف: 33</button>
    <button class="pill" data-t="100">هدف: 100</button>
    <button class="pill" data-t="1000">هدف: 1000</button>
    <button class="pill" data-t="0">بدون هدف</button>
  </div>
  <div class="tasbih-stage">
    <div class="dhikr-display" id="dhikrText">سُبْحَانَ اللَّهِ</div>
    <div class="tap-circle" id="tapCircle" onclick="tap()">
      <div class="cnt" id="cnt">0</div>
      <div class="of" id="of">من 33</div>
      <div class="hint">اضغط هنا للتسبيح</div>
    </div>
    <div class="progress-line"><i id="bar"></i></div>
    <div class="done-msg" id="doneMsg">🌟 ما شاء الله، وصلت لهدفك!<br>تقبّل الله منك 🤲</div>
    <div class="tasbih-actions">
      <button class="btn ghost" onclick="resetCount()">🔄 من الأول</button>
      <button class="btn ghost" onclick="nextDhikr()">⏭ الذكر التالي</button>
    </div>
  </div>
  {{ foot }}
</div>
<script>
var count = 0, target = 33, dhikr = 'سُبْحَانَ اللَّهِ';
var dhikrOrder = ['سُبْحَانَ اللَّهِ','الْحَمْدُ لِلَّهِ','اللَّهُ أَكْبَرُ','لَا إِلَهَ إِلَّا اللَّهُ','أَسْتَغْفِرُ اللَّهَ'];
function save(){ try{ localStorage.setItem('tasbih', JSON.stringify({c:count,t:target,d:dhikr})); }catch(e){} }
function load(){ try{ var s = JSON.parse(localStorage.getItem('tasbih')); if(s){ count=s.c||0; target=s.t===undefined?33:s.t; dhikr=s.d||dhikr; } }catch(e){} }
function render(){
  document.getElementById('cnt').textContent = count;
  document.getElementById('dhikrText').textContent = dhikr;
  document.getElementById('of').textContent = target > 0 ? 'من ' + target : 'بدون هدف';
  var bar = document.getElementById('bar');
  bar.style.width = target > 0 ? Math.min(100, count/target*100) + '%' : '0%';
  document.getElementById('doneMsg').style.display = (target > 0 && count >= target) ? 'block' : 'none';
  document.querySelectorAll('#dhikrPills .pill').forEach(function(p){
    p.classList.toggle('on', p.dataset.d === dhikr || (p.dataset.d === 'custom' && dhikrOrder.indexOf(dhikr) < 0));
  });
  document.querySelectorAll('#targetPills .pill').forEach(function(p){
    p.classList.toggle('on', parseInt(p.dataset.t) === target);
  });
  document.getElementById('customRow').style.display = (dhikrOrder.indexOf(dhikr) < 0) ? 'flex' : 'none';
}
function tap(){
  count++;
  if (navigator.vibrate) { navigator.vibrate(target > 0 && count === target ? [60,40,60] : 15); }
  render(); save();
}
function resetCount(){ count = 0; render(); save(); }
function nextDhikr(){
  var i = dhikrOrder.indexOf(dhikr);
  dhikr = dhikrOrder[(i + 1) % dhikrOrder.length];
  count = 0; render(); save();
}
function setCustom(){
  var v = document.getElementById('customDhikr').value.trim();
  if (v) { dhikr = v; count = 0; render(); save(); }
}
document.querySelectorAll('#dhikrPills .pill').forEach(function(p){
  p.onclick = function(){
    if (p.dataset.d === 'custom') {
      document.getElementById('customRow').style.display = 'flex';
      document.getElementById('customDhikr').focus();
    } else { dhikr = p.dataset.d; count = 0; render(); save(); }
  };
});
document.querySelectorAll('#targetPills .pill').forEach(function(p){
  p.onclick = function(){ target = parseInt(p.dataset.t); count = 0; render(); save(); };
});
load(); render();
</script>
{% endblock %}"""

@app.route("/tasbih")
def tasbih():
    return render_template_string(TASBIH_PAGE, base=BASE, title="السبحة", foot=FOOT)

# ---------------- PWA ----------------
MANIFEST = {
    "name": "ديني - قرآن وأذكار ومواقيت الصلاة",
    "short_name": "ديني",
    "start_url": "/",
    "display": "standalone",
    "dir": "rtl",
    "lang": "ar",
    "background_color": "#0b1512",
    "theme_color": "#0b1512",
    "icons": [{
        "src": "/icon.svg", "sizes": "any", "type": "image/svg+xml", "purpose": "any"
    }]
}

SW_JS = """
const CACHE = 'deen-v2';
const SHELL = ['/', '/quran', '/adhkar', '/icon.svg'];
self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(SHELL)).then(() => self.skipWaiting()));
});
self.addEventListener('activate', e => {
  e.waitUntil(caches.keys().then(ks => Promise.all(ks.filter(k => k !== CACHE).map(k => caches.delete(k)))).then(() => self.clients.claim()));
});
self.addEventListener('fetch', e => {
  const url = new URL(e.request.url);
  if (e.request.method !== 'GET') return;
  // Cache-first for Quran pages & surah data, network-first otherwise
  if (url.pathname.startsWith('/quran')) {
    e.respondWith(caches.match(e.request).then(hit => hit || fetch(e.request).then(res => {
      if (res.ok) { const c = res.clone(); caches.open(CACHE).then(ca => ca.put(e.request, c)); }
      return res;
    }).catch(() => caches.match('/quran'))));
    return;
  }
  e.respondWith(fetch(e.request).then(res => {
    if (res.ok && url.origin === location.origin) {
      const c = res.clone(); caches.open(CACHE).then(ca => ca.put(e.request, c));
    }
    return res;
  }).catch(() => caches.match(e.request)));
});
"""

ICON_SVG = """<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512">
<rect width="512" height="512" rx="110" fill="#0b1512"/>
<circle cx="256" cy="230" r="150" fill="none" stroke="#d4af6a" stroke-width="14"/>
<path d="M256 96 A134 134 0 1 0 256 364 A107 134 0 1 1 256 96 Z" fill="#d4af6a"/>
<path d="M330 150 l12 30 30 12 -30 12 -12 30 -12 -30 -30 -12 30 -12 Z" fill="#e8c987"/>
<text x="256" y="445" font-size="52" text-anchor="middle" fill="#e8c987" font-family="serif">ديني</text>
</svg>"""

@app.route("/manifest.json")
def manifest():
    return jsonify(MANIFEST)

@app.route("/sw.js")
def sw():
    return Response(SW_JS, mimetype="application/javascript",
                    headers={"Service-Worker-Allowed": "/", "Cache-Control": "no-cache"})

@app.route("/icon.svg")
def icon():
    return Response(ICON_SVG, mimetype="image/svg+xml", headers={"Cache-Control": "max-age=86400"})

@app.after_request
def no_cache_html(resp):
    if resp.mimetype == "text/html":
        resp.headers["Cache-Control"] = "no-cache"
    return resp

if __name__ == "__main__":
    import os
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
