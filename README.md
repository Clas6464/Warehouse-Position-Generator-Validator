# Warehouse-Position-Generator-Validator
STECK:Google Apps Script, JavaScript, HTML5, CSS3.  

function onOpen() {
  const ui = SpreadsheetApp.getUi();
  ui.createMenu('🤖 Generator Pozic V2')
      .addItem('Open Generator', 'showSidebar')
      .addToUi();
}


function saveToSheet(dataArray) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName("NOVEPOZICE");
  
  if (!sheet) {
    sheet = ss.insertSheet("NOVEPOZICE");
    sheet.appendRow(["OLD POZICE ", "NEW POZICE (VOLNA POZICE)"]);
    sheet.getRange("A1:B1").setFontWeight("bold").setBackground("#f3f3f3");
  }
  
  if (dataArray && dataArray.length > 0) {
    sheet.getRange(sheet.getLastRow() + 1, 1, dataArray.length, 2).setValues(dataArray);
    return "UŽ MAŠ V  NOVEPOZICE!";
  }
  return "NÍS NE MAŠ.";
}


function findFreePositions(generatedList) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName("VOLNE POZICE");
  
  if (!sheet) return "ERROOR NE MAŠ TABULKU VOLNE POZICE!";
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  const posIdx = headers.indexOf("position_EAN");
  const secIdx = headers.indexOf("sector");
  
  if (posIdx === -1 || secIdx === -1) return "ERROR NENÍ!";

  const firstPos = generatedList[0]; 
  const sectorMatch = firstPos.match(/^(\d+)([A-Z])/);
  
  if (!sectorMatch) return "ŠPATNY FORMAT!";
  
  const targetSector = String(sectorMatch[1]);
  const avoidLetter = String(sectorMatch[2]);
  
  let freePositions = [];
  for (let i = 1; i < data.length; i++) {
    let currentPosEAN = String(data[i][posIdx]);
    let currentSector = String(data[i][secIdx]);
    if (currentSector === targetSector && !currentPosEAN.includes(avoidLetter)) {
      freePositions.push(currentPosEAN);
    }
  }
  
  return { free: freePositions, original: generatedList };
}

function showSidebar() {
  const html = `
    <!DOCTYPE html>
    <html>
    <head>
    <style>
      body { 
        font-family: 'Segoe UI', sans-serif; 
        margin: 0; padding: 0; 
        transition: background 0.5s ease; 
        background-size: cover; 
        background-position: center;
        background-attachment: fixed;
      }
      .tabs-nav { display: flex; background: rgba(0, 0, 0, 0.85); padding: 5px 5px 0 5px; }
      .tab-btn { flex: 1; padding: 12px; border: none; background: transparent; color: #888; cursor: pointer; font-weight: bold; font-size: 11px; border-radius: 5px 5px 0 0; outline: none; text-transform: uppercase; }
      .tab-btn.active { background: rgba(255, 255, 255, 0.9); color: #1a73e8; }
      
      .container { display: none; background: rgba(255, 255, 255, 0.9); padding: 15px; margin: 15px; border-radius: 12px; box-shadow: 0 8px 15px rgba(0,0,0,0.3); }
      .container.active { display: block; }
      
      .input-group { margin-bottom: 12px; }
      label { font-weight: 700; display: block; margin-bottom: 5px; font-size: 12px; color: #333; }
      input, textarea { width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: 6px; box-sizing: border-box; font-size: 13px; background: rgba(255,255,255,0.8); }
      
      button { width: 100%; padding: 12px; background: #2ecc71; color: white; border: none; border-radius: 8px; cursor: pointer; font-size: 14px; font-weight: bold; margin-top: 10px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
      button:hover { filter: brightness(1.1); }
      
      .btn-blue { background: #3498db; }
      .btn-orange { background: #e67e22; margin-top: 15px; }
      
      #result, #resultNew { margin-top: 15px; padding: 10px; background: #fff; border: 1px solid #ddd; border-radius: 6px; max-height: 140px; overflow-y: auto; font-family: 'Consolas', monospace; font-size: 11px; color: #2c3e50; }
    </style>
    </head>
    <body id="mainBody">

    <div class="tabs-nav">
      <button id="btnOld" class="tab-btn active" onclick="openTab(event, 'oldVersion')">GENERATOR</button>
      <button id="btnNew" class="tab-btn" onclick="openTab(event, 'newVersion')">PŘIDAT VOLNE POZICE</button>
    </div>

    <div id="oldVersion" class="container active">
      <div class="input-group">
        <label>SEKTOR + LITERA</label>
        <div style="display: flex; gap: 5px;">
          <input type="text" id="num1" placeholder="8" style="flex: 2;">
          <input type="text" id="let" placeholder="P" style="flex: 1;">
        </div>
      </div>
      <div class="input-group"><label>PATRA (Pater)</label><input type="number" id="midMax" value="5"></div>
      <div class="input-group"><label>Z ČÍSLA</label><input type="number" id="num2" value="1"></div>
      <div class="input-group"><label>MAX ČÍSEL</label><input type="number" id="countN" value="2"></div>
      <button onclick="generate()">CREATE</button>
      <div id="result">TY VOLE .... </div>
    </div>

    <div id="newVersion" class="container">
      
      <textarea id="transferArea" rows="3" readonly style="background: #f9f9f9;"></textarea>
      <button class="btn-blue" onclick="getVolne()">SERCH NEW POSITION </button>
      <div id="resultNew">.........</div>
      <button id="saveBtn" class="btn-orange" style="display:none;" onclick="sendToSheet()">ADD IN NOVEPOZICE</button>
    </div>

    <script>
      let lastGenerated = [];
      let pairToSave = [];

      
      const bgGenerator = "url('https://preview.redd.it/i-made-some-phone-wallpapers-from-the-first-4-episodes-v0-xoh71m011woc1.png?width=640&crop=smart&auto=webp&s=b68d76ef341b8d1bba806266c59a953751f21918')"; 
      const bgVolne = "url('https://i.pinimg.com/originals/c3/4e/7b/c34e7bfb2de3d96632e02ec72c3e4fab.jpg')";  

      
      document.body.style.backgroundImage = bgGenerator;

      function openTab(evt, tabName) {
        let i, container, tabBtn;
        container = document.getElementsByClassName("container");
        for (i = 0; i < container.length; i++) container[i].classList.remove("active");
        
        tabBtn = document.getElementsByClassName("tab-btn");
        for (i = 0; i < tabBtn.length; i++) tabBtn[i].classList.remove("active");
        
        document.getElementById(tabName).classList.add("active");
        evt.currentTarget.classList.add("active");

        
        if(tabName === 'oldVersion') {
          document.body.style.backgroundImage = bgGenerator;
        } else {
          document.body.style.backgroundImage = bgVolne;
          if(lastGenerated.length > 0) {
            document.getElementById('transferArea').value = lastGenerated.join('\\n');
          }
        }
      }

      function generate() {
        const n1 = document.getElementById('num1').value;
        const l = document.getElementById('let').value;
        const midMax = parseInt(document.getElementById('midMax').value);
        const startN2 = parseInt(document.getElementById('num2').value);
        const repeatN = parseInt(document.getElementById('countN').value);
        
        lastGenerated = [];
        let htmlRes = "";
        for (let j = 0; j < repeatN; j++) {
          let currentLastNum = startN2 + j;
          for (let i = 1; i <= midMax; i++) {
            let p = n1 + l + "-" + i + "-" + currentLastNum;
            lastGenerated.push(p);
            htmlRes += " " + p + "<br>";
          }
        }
        document.getElementById('result').innerHTML = htmlRes;
      }

      function getVolne() {
        const areaVal = document.getElementById('transferArea').value;
        if (!areaVal) return alert("TY VOLE .. NÍC NE MAŠ DO ČEHO BUDU PŘIDAVAT..!");
        const list = areaVal.split('\\n').filter(Boolean);
        
        document.getElementById('resultNew').innerText = "HLEDAM V TABULCE VOLNE POZICE";
        
        google.script.run.withSuccessHandler(function(res) {
          if (typeof res === 'string') {
            document.getElementById('resultNew').innerText = res;
          } else {
            pairToSave = [];
            let html = "<b>NA ...:</b><br>";
            for(let i = 0; i < res.original.length; i++) {
              let oldP = res.original[i];
              let freeP = res.free[i] || "NÍC NEMA ..";
              pairToSave.push([oldP, freeP]);
              html += "<span style='color:#7f8c8d'>" + oldP + "</span> ➔ <b>" + freeP + "</b><br>";
            }
            document.getElementById('resultNew').innerHTML = html;
            document.getElementById('saveBtn').style.display = 'block';
          }
        }).findFreePositions(list);
      }

      function sendToSheet() {
        if (pairToSave.length === 0) return;
        const btn = document.getElementById('saveBtn');
        btn.innerText = "PÍSU .  . ..";
        btn.disabled = true;
        
        google.script.run.withSuccessHandler(function(msg) {
          alert(msg);
          btn.innerText = "ADD IN NOVEPOZICE";
          btn.disabled = false;
          btn.style.display = 'none';
        }).saveToSheet(pairToSave);
      }
    </script>
    </body>
    </html>
  `;
  const userInterface = HtmlService.createHtmlOutput(html).setTitle('Generator Pozic V2').setWidth(320);
  SpreadsheetApp.getUi().showSidebar(userInterface);
}
