<!doctype html> 

<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>MMQR — Static → Dynamic QR</title>
  <style>
    body{font-family:Arial,Helvetica,sans-serif;max-width:1000px;margin:20px auto;padding:20px;line-height:1.4;color:#111}
    h1{font-size:22px;margin-bottom:10px}
    .card{border:1px solid #ddd;padding:12px;border-radius:8px;margin-bottom:16px}
    label{display:block;margin:6px 0;font-weight:600}
    input,textarea{width:100%;padding:6px;border:1px solid #ccc;border-radius:4px;font-size:14px}
    textarea{height:80px;font-family:monospace}
    button{margin:6px 4px 0 0;padding:6px 12px;border:0;border-radius:6px;background:#0b6efd;color:#fff;cursor:pointer}
    .qr-box{display:flex;align-items:center;justify-content:center;height:260px;margin-top:10px}
    .hidden{display:none}
  </style>
</head>
<body>
<h1>MMQR — Static → Dynamic QR</h1>
<div class="card">
  <label>Upload QR image</label>
  <input id="fileInput" type="file" accept="image/*" />
  <label>Or paste EMV string</label>
  <textarea id="rawInput" placeholder="000201..."></textarea>
  <button id="btnParse">Parse</button>
</div>
<div class="card">
  <label>54 — Amount</label>
  <input id="input54" type="text" placeholder="5232.67" />
  <label>62.01 — Bill Number</label>
  <input id="input6201" type="text" placeholder="auto timestamp" />
  <label>62.02 — Phone</label>
  <input id="input6202" type="text" placeholder="0943023332" />
  <label>62.05 — Merchant Ref</label>
  <input id="input6205" type="text" placeholder="auto REFID+10digit" />
  <label>62.08 — Purpose</label>
  <input id="input6208" type="text" placeholder="random name" />
  <button id="btnGenerate">Generate Dynamic QR</button>
  <button id="btnToggleQR">Hide QR</button>
  <button id="btnDownload">Download PNG</button>
</div>
<div class="card">
  <label>Dynamic QR String</label>
  <textarea id="resultString" readonly></textarea>
  <div class="qr-box hidden" id="qrBox"></div>
</div><script src="https://cdn.jsdelivr.net/npm/jsqr@1.4.0/dist/jsQR.min.js"></script><script src="https://cdn.jsdelivr.net/npm/qrious@4.0.2/dist/qrious.min.js"></script><script>
const $=id=>document.getElementById(id);
let lastParsed=null;

function parseEMV(str){
  const out=[];let i=0;
  while(i<str.length){
    const tag=str.substr(i,2);i+=2;
    const len=parseInt(str.substr(i,2),10);i+=2;
    const val=str.substr(i,len);i+=len;
    const item={tag,len,value:val};
    if(tag==='26'||tag==='62'||tag==='64'){item.children=parseEMV(val);}
    out.push(item);
  }
  return out;
}
function childrenToMap(children){const m={};children.forEach(c=>m[c.tag]=c.value);return m;}
function pad2(n){return n<10?'0'+n:String(n);}

function CRC16CCITT_Unicode(data){
  const polynomial=0x1021;let crc=0xFFFF;
  const buffer=new TextEncoder().encode(data);
  for(let byte of buffer){
    crc^=byte<<8;
    for(let i=0;i<8;i++){
      crc=(crc&0x8000)?((crc<<1)^polynomial):(crc<<1);
      crc&=0xFFFF;
    }
  }
  return crc.toString(16).toUpperCase().padStart(4,'0');
}
function timestampString(){const d=new Date();const z=n=>String(n).padStart(2,'0');return `${d.getFullYear()}${z(d.getMonth()+1)}${z(d.getDate())}${z(d.getHours())}${z(d.getMinutes())}${z(d.getSeconds())}`;}
function randomDigits(n){let s='';for(let i=0;i<n;i++)s+=Math.floor(Math.random()*10);return s;}
function randomName(){const names=['ShopOne','SuperMart','GoldenStore','LuckyMart','BestBuy','GrandShop'];return names[Math.floor(Math.random()*names.length)];}

$('fileInput').addEventListener('change',async ev=>{
  const f=ev.target.files[0];if(!f)return;
  const img=await fileToImage(f);const raw=await decodeImageToString(img);
  if(raw){$('rawInput').value=raw;}else{alert('No QR found in image');}
});

$('btnParse').addEventListener('click',()=>{
  const raw=$('rawInput').value.trim();if(!raw){alert('Paste EMV string or upload QR');return;}
  lastParsed=parseEMV(raw);
  $('resultString').value=raw;
});

$('btnGenerate').addEventListener('click',()=>{
  if(!lastParsed){alert('Parse QR first');return;}
  applyEdits();
  const s=rebuildWithCRC(lastParsed);
  $('resultString').value=s;
  renderQR(s);
  $('qrBox').classList.remove('hidden');
});

$('btnToggleQR').addEventListener('click',()=>{
  $('qrBox').classList.toggle('hidden');
});

$('btnDownload').addEventListener('click',()=>{
  const img=$('qrBox').querySelector('img');if(!img){alert('No QR');return;}
  const a=document.createElement('a');a.href=img.src;a.download='mmqr.png';a.click();
});

function applyEdits(){
  const amt=$('input54').value.trim()||'5232.67';
  // Remove existing 54 if any
  lastParsed=lastParsed.filter(p=>p.tag!=='54');
  // Ensure tag 53 exists (currency)
  const idx53=lastParsed.findIndex(p=>p.tag==='53');
  if(idx53>=0){
    // Insert 54 right after 53
    lastParsed.splice(idx53+1,0,{tag:'54',len:amt.length,value:amt});
  } else {
    // If no 53, push at end
    lastParsed.push({tag:'54',len:amt.length,value:amt});
  }

  let p62=lastParsed.find(p=>p.tag==='62');
  if(!p62){p62={tag:'62',len:0,value:'',children:[]};lastParsed.push(p62);}
  const childMap=p62.children?childrenToMap(p62.children):{};
  childMap['01']=$('input6201').value.trim()||timestampString();
  childMap['02']=$('input6202').value.trim()||'0943023332';
  childMap['05']=$('input6205').value.trim()||'REFID'+randomDigits(10);
  childMap['08']=$('input6208').value.trim()||randomName();
  const newChildren=Object.keys(childMap).sort().map(k=>({tag:k,len:childMap[k].length,value:childMap[k]}));
  p62.children=newChildren;p62.value=newChildren.map(c=>`${c.tag}${pad2(c.value.length)}${c.value}`).join('');p62.len=p62.value.length;
  const idxCRC=lastParsed.findIndex(p=>p.tag==='63');if(idxCRC>=0)lastParsed.splice(idxCRC,1);
}

function buildEMV(parsed){
  function buildItem(item){
    if(item.children){
      const childStr=item.children.map(c=>`${c.tag}${pad2(c.value.length)}${c.value}`).join('');
      return `${item.tag}${pad2(childStr.length)}${childStr}`;
    } else {
      return `${item.tag}${pad2(item.value.length)}${item.value}`;
    }
  }
  return parsed.map(buildItem).join('');
}

function rebuildWithCRC(parsed){
  const without63=buildEMV(parsed);
  const toCrc=without63+'6304';
  const crc=CRC16CCITT_Unicode(toCrc);
  return toCrc+crc;
}

function renderQR(text){
  const qrBox=$('qrBox');qrBox.innerHTML='';
  const qr=new QRious({value:text,size:300});
  const img=document.createElement('img');img.src=qr.toDataURL();qrBox.appendChild(img);
}

function fileToImage(file){return new Promise((res,rej)=>{const fr=new FileReader();fr.onload=()=>{const img=new Image();img.onload=()=>res(img);img.onerror=rej;img.src=fr.result;};fr.onerror=rej;fr.readAsDataURL(file);});}
function decodeImageToString(img){const c=document.createElement('canvas');c.width=img.naturalWidth;c.height=img.naturalHeight;const ctx=c.getContext('2d');ctx.drawImage(img,0,0);const imgData=ctx.getImageData(0,0,c.width,c.height);const code=jsQR(imgData.data,c.width,c.height);return code?code.data:null;}
</script></body>
</html>