<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>輕一點 — 自由組合餐</title>
<style>
body { font-family: Arial, sans-serif; background:#f7fafc; margin:0; padding:20px; color:#333; }
.container { max-width:1200px; margin:0 auto; }
.card { background:#fff; border-radius:12px; padding:20px; margin-bottom:20px; box-shadow:0 2px 8px rgba(0,0,0,0.1); }
h1 { font-size:24px; margin-bottom:6px; }
h2 { font-size:20px; margin-bottom:6px; }
p { font-size:14px; margin-top:0; }
.grid { display:grid; gap:12px; }
.grid-cols-1 { grid-template-columns:repeat(1,1fr); }
.grid-cols-2 { grid-template-columns:repeat(2,1fr); }
.label-box { border:1px solid #ddd; border-radius:8px; padding:8px; cursor:pointer; display:flex; flex-direction:column; margin-bottom:4px; }
.label-box:hover { background:#f0f0f0; }
.checkbox { margin-right:6px; }
.tooltip { position:relative; display:inline-block; }
.tooltip .tt { position:absolute; left:50%; transform:translateX(-50%); bottom:calc(100% + 8px); background:rgba(0,0,0,0.8); color:#fff; padding:6px 8px; border-radius:6px; font-size:12px; display:none; white-space:nowrap; z-index:50; }
.tooltip:hover .tt { display:block; }
.scroll-x { overflow-x:auto; }
button { padding:8px 16px; border:none; border-radius:6px; cursor:pointer; font-size:14px; }
button:hover { opacity:0.9; }
.btn-green { background:#38a169; color:white; }
.btn-blue { background:#3182ce; color:white; }
.btn-gray { background:#e2e8f0; color:#333; }
table { width:100%; border-collapse:collapse; font-size:13px; }
th, td { padding:6px 8px; border:1px solid #ddd; }
th { background:#f0f0f0; text-align:left; }
</style>
</head>
<body>
<div class="container">
  <div class="card">
    <h1>輕一點 — 自由組合餐</h1>
    <p>不限數量選擇食材，下方即時計算營養與中醫分析。</p>
  </div>

  <div class="grid grid-cols-2">
    <!-- 左：選單 -->
    <div class="grid grid-cols-1">
      <div class="card">
        <h2>蛋白質</h2>
        <div id="protein-list" style="max-height:300px; overflow:auto;"></div>
      </div>
      <div class="card">
        <h2>主食</h2>
        <div id="carb-list" style="max-height:300px; overflow:auto;"></div>
      </div>
      <div class="card">
        <h2>蔬菜</h2>
        <div id="veg-list" style="max-height:300px; overflow:auto;"></div>
      </div>
      <div class="card">
        <h2>醬汁 / 水果</h2>
        <div id="sauce-list" style="max-height:200px; overflow:auto;"></div>
      </div>
    </div>

    <!-- 右：結果面板 -->
    <div class="card">
      <h2>你的餐盤 — 即時營養 & 中醫分析</h2>
      <p><strong>選取項目：</strong><span id="picked-items">尚未選取</span></p>
      <div class="scroll-x">
        <table>
          <thead>
            <tr>
              <th>食材</th>
              <th>份量</th>
              <th>kcal</th>
              <th>蛋白(g)</th>
              <th>脂肪(g)</th>
              <th>碳水(g)</th>
              <th>中醫屬性</th>
            </tr>
          </thead>
          <tbody id="nutri-rows"></tbody>
        </table>
      </div>
      <p><strong>總熱量：</strong><span id="total-kcal">0</span> kcal</p>
      <p><strong>總 P/F/C（g）：</strong><span id="total-pfc">0 / 0 / 0</span></p>
      <p><strong>P/F/C 熱量佔比：</strong><span id="pfc-pct">0% / 0% / 0%</span></p>
      <div class="card" style="margin-top:10px; padding:10px;">
        <strong>中醫整體分析</strong>
        <p id="tcm-summary">尚無足夠資料。</p>
      </div>
      <div style="margin-top:10px;">
        <button id="add-cart" class="btn-green">加入購物車</button>
        <button id="download-csv" class="btn-blue">下載營養報告（CSV）</button>
        <button id="reset" class="btn-gray">清除選取</button>
      </div>
    </div>
  </div>
</div>

<script>
// 食材模板
const FOOD_TEMPLATES = {
  proteins: {
    '雞胸肉': {portion:'100g', kcal:165,p:31,f:3,c:0,tcm:'平／甘', effect:'補氣健脾、養肌生力'},
    '燻雞': {portion:'100g', kcal:150,p:28,f:3,c:0,tcm:'平／甘鹹', effect:'補氣、開胃'},
    '鮭魚': {portion:'100g', kcal:208,p:22,f:13,c:0,tcm:'溫／甘', effect:'補血養顏、潤膚抗老'},
    '豆腐': {portion:'100g', kcal:80,p:8,f:4,c:2,tcm:'涼／甘', effect:'清熱潤燥、生津'},
    '毛豆': {portion:'100g', kcal:120,p:12,f:6,c:8,tcm:'平／甘', effect:'健脾利濕、補中益氣'}
  },
  carbs: {
    '雜糧飯': {portion:'100g', kcal:220,p:5,f:2,c:45,tcm:'平／甘', effect:'補中益氣、健脾胃'},
    '紫米飯': {portion:'90g', kcal:230,p:5,f:1,c:50,tcm:'溫／甘', effect:'補血養氣、益腎'},
    '玉米飯': {portion:'100g', kcal:200,p:4,f:1,c:42,tcm:'平／甘', effect:'益氣養胃、利濕'},
    '地瓜': {portion:'100g', kcal:115,p:2,f:0,c:27,tcm:'平／甘', effect:'健脾養胃、補中益氣'},
    '馬鈴薯': {portion:'100g', kcal:100,p:2,f:0,c:23,tcm:'平／甘', effect:'補氣健脾、和胃'}
  },
  vegs: {
    '青花菜': {portion:'60g', kcal:35,p:3,f:0,c:7,tcm:'涼／苦甘', effect:'清熱解毒、潤腸'},
    '紅蘿蔔': {portion:'40g', kcal:40,p:1,f:0,c:9,tcm:'平／甘', effect:'補肝明目、健脾'},
    '紫高麗菜': {portion:'40g', kcal:30,p:1,f:0,c:7,tcm:'平／甘', effect:'涼血解毒、促循環'},
    '紫洋蔥': {portion:'30g', kcal:45,p:1,f:0,c:10,tcm:'溫／辛', effect:'活血化瘀、行氣'},
    '番茄': {portion:'40g', kcal:25,p:1,f:0,c:5,tcm:'涼／甘酸', effect:'清熱生津、養顏'},
    '節瓜': {portion:'60g', kcal:10,p:1,f:0,c:2,tcm:'涼／甘', effect:'利水化濕、清熱'},
    '金瓜': {portion:'40g', kcal:50,p:1,f:0,c:12,tcm:'溫／甘', effect:'補中益氣、養胃'},
    '高麗菜': {portion:'60g', kcal:25,p:1,f:0,c:5,tcm:'平／甘', effect:'健胃整腸、調氣'},
    '海帶芽': {portion:'30g', kcal:15,p:1,f:0,c:3,tcm:'涼／鹹', effect:'軟堅散結、降脂'},
    '小黃瓜': {portion:'40g', kcal:8,p:0,f:0,c:2,tcm:'涼／甘', effect:'清熱利水、消腫'},
    '杏鮑菇': {portion:'40g', kcal:40,p:2,f:0,c:8,tcm:'平／甘', effect:'補氣潤燥'},
    '鈕扣菇': {portion:'40g', kcal:20,p:2,f:0,c:4,tcm:'平／甘', effect:'補氣健脾'}
  },
  sauces: {
    '柚香和風': {portion:'15g', kcal:10,p:0,f:0,c:2,tcm:'涼／辛甘', effect:'開胃醒脾'},
    '胡麻醬': {portion:'15g', kcal:80,p:2,f:7,c:2,tcm:'平／甘', effect:'潤燥養血'},
    '青醬': {portion:'15g', kcal:70,p:1,f:6,c:2,tcm:'平／甘', effect:'行氣解鬱'},
    '韓式辣醬': {portion:'15g', kcal:25,p:0,f:0,c:6,tcm:'溫／辛', effect:'活血祛寒'},
    '薑汁和風': {portion:'15g', kcal:10,p:0,f:0,c:2,tcm:'溫／辛', effect:'溫中散寒、促循環'}
  }
};

const selected={protein:[], carb:[], vegs:[], sauce:[]};
const proteinList=document.getElementById('protein-list');
const carbList=document.getElementById('carb-list');
const vegList=document.getElementById('veg-list');
const sauceList=document.getElementById('sauce-list');
const pickedItemsEl=document.getElementById('picked-items');
const nutriRows=document.getElementById('nutri-rows');
const totalKcalEl=document.getElementById('total-kcal');
const totalPfcEl=document.getElementById('total-pfc');
const pfcPctEl=document.getElementById('pfc-pct');
const tcmSummaryEl=document.getElementById('tcm-summary');

function makeOption(name, group, info){
  const wrapper=document.createElement('label');
  wrapper.className='label-box tooltip';
  const input=document.createElement('input');
  input.type='checkbox';
  input.className='checkbox';
  input.addEventListener('change',()=>{
    if(input.checked){ selected[group].push(name); } 
    else{ selected[group]=selected[group].filter(x=>x!==name); }
    refreshSummary();
  });
  wrapper.appendChild(input);
  const div = document.createElement('div');
  div.innerHTML=`${name} (${info.portion})<br>${info.effect} • ${info.tcm}`;
  wrapper.appendChild(div);
  const tooltip=document.createElement('div');
  tooltip.className='tt';
  tooltip.textContent=`${info.tcm} · ${info.effect} · kcal ${info.kcal} · P ${info.p}g / F ${info.f}g / C ${info.c}g`;
  wrapper.appendChild(tooltip);
  return wrapper;
}

Object.entries(FOOD_TEMPLATES.proteins).forEach(([k,v])=>proteinList.appendChild(makeOption(k,'protein',v)));
Object.entries(FOOD_TEMPLATES.carbs).forEach(([k,v])=>carbList.appendChild(makeOption(k,'carb',v)));
Object.entries(FOOD_TEMPLATES.vegs).forEach(([k,v])=>vegList.appendChild(makeOption(k,'vegs',v)));
Object.entries(FOOD_TEMPLATES.sauces).forEach(([k,v])=>sauceList.appendChild(makeOption(k,'sauce',v)));

function refreshSummary(){
  const all=[...selected.protein,...selected.carb,...selected.vegs,...selected.sauce];
  pickedItemsEl.textContent=all.length? all.join('、'):'尚未選取';
  nutriRows.innerHTML='';
  let totalKcal=0,totalP=0,totalF=0,totalC=0;
  let tcmCount={平:0,涼:0,寒:0,溫:0,熱:0};
  all.forEach(name=>{
    const info = FOOD_TEMPLATES.proteins[name]||FOOD_TEMPLATES.carbs[name]||FOOD_TEMPLATES.vegs[name]||FOOD_TEMPLATES.sauces[name];
    nutriRows.innerHTML+=`<tr><td>${name}</td><td>${info.portion}</td><td>${info.kcal}</td><td>${info.p}</td><td>${info.f}</td><td>${info.c}</td><td>${info.tcm}</td></tr>`;
    totalKcal+=info.kcal; totalP+=info.p; totalF+=info.f; totalC+=info.c;
    info.tcm.split(/[／]/).forEach(v=>{if(tcmCount[v]!==undefined) tcmCount[v]++;});
  });
  totalKcalEl.textContent=totalKcal;
  totalPfcEl.textContent=`${totalP} / ${totalF} / ${totalC}`;
  const totalMacroKcal = totalP*4 + totalF*9 + totalC*4;
  pfcPctEl.textContent=totalMacroKcal>0 ? `P:${Math.round(totalP*4/totalMacroKcal*100)}% / F:${Math.round(totalF*9/totalMacroKcal*100)}% / C:${Math.round(totalC*4/totalMacroKcal*100)}%`:'0% / 0% / 0%';
  let tcmMsg=[];
  Object.entries(tcmCount).forEach(([k,v])=>{if(v>0)tcmMsg.push(`${k} ${v}種`);});
  tcmSummaryEl.textContent=tcmMsg.length? tcmMsg.join(' • ') : '尚無足夠資料。';
}

// CSV 下載
document.getElementById('download-csv').addEventListener('click',()=>{
  let csv='食材,份量,kcal,蛋白(g),脂肪(g),碳水(g),中醫屬性\n';
  [...selected.protein,...selected.carb,...selected.vegs,...selected.sauce].forEach(name=>{
    const info = FOOD_TEMPLATES.proteins[name]||FOOD_TEMPLATES.carbs[name]||FOOD_TEMPLATES.vegs[name]||FOOD_TEMPLATES.sauces[name];
    csv+=`${name},${info.portion},${info.kcal},${info.p},${info.f},${info.c},${info.tcm}\n`;
  });
  const blob=new Blob([csv],{type:'text/csv'});
  const url=URL.createObjectURL(blob);
  const a=document.createElement('a');
  a.href=url;
  a.download='營養報告.csv';
  a.click();
  URL.revokeObjectURL(url);
});

// 重置
document.getElementById('reset').addEventListener('click',()=>{
  Object.keys(selected).forEach(k=>selected[k]=[]);
  document.querySelectorAll('input[type="checkbox"]').forEach(i=>i.checked=false);
  refreshSummary();
});

// 加入購物車示例
document.getElementById('add-cart').addEventListener('click',()=>{alert('已加入購物車（示例）');});

refreshSummary();
</script>
</body>
</html>
