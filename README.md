# TRABALHO-CAMPUS
<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<meta name="theme-color" content="#101827">
<title>Campus AR — Localização em Realidade Aumentada</title>
<style>
:root{font-family:system-ui,-apple-system,Segoe UI,Roboto,sans-serif;color:#f8fafc;background:#101827}
*{box-sizing:border-box}body{margin:0;min-height:100vh}
#overlay{position:fixed;inset:0;pointer-events:none;z-index:5}
header{padding:18px 18px 12px;background:linear-gradient(135deg,#17243a,#101827);border-bottom:1px solid #334155;pointer-events:auto}
h1{font-size:1.25rem;margin:0 0 4px}header p{margin:0;color:#cbd5e1;font-size:.88rem}
.panel{pointer-events:auto;margin:14px;padding:14px;border:1px solid #334155;border-radius:16px;background:rgba(15,23,42,.94);box-shadow:0 10px 28px #0005}
label{display:block;font-size:.88rem;color:#cbd5e1;margin-bottom:6px}
select,button{width:100%;padding:13px;border-radius:10px;border:1px solid #475569;font:inherit}
select{background:#f8fafc;color:#0f172a;margin-bottom:10px}
button{background:#dc2626;color:white;border:0;font-weight:700;margin:4px 0;cursor:pointer}
button.secondary{background:#334155}button:disabled{opacity:.5}
#status{font-size:.9rem;line-height:1.45;color:#e2e8f0;margin-top:8px;min-height:2.6em}
#points{font-size:.83rem;color:#cbd5e1;line-height:1.55;margin-top:10px}
.hint{font-size:.78rem;color:#94a3b8;margin-top:10px;line-height:1.4}
#scene{position:fixed;inset:0;z-index:1}
footer{pointer-events:auto;padding:0 18px 16px;color:#94a3b8;font-size:.75rem}
@media(min-width:700px){.panel{max-width:420px;margin:20px}header{padding-left:24px}}
</style>
<script type="importmap">
{"imports":{"three":"https://unpkg.com/three@0.160.0/build/three.module.js"}}
</script>
</head>
<body>
<div id="scene"></div>
<div id="overlay">
<header>
  <h1>Campus AR</h1>
  <p>Localização dos prédios em Realidade Aumentada Markerless</p>
</header>
<section class="panel">
  <label for="place">Selecione o ponto que deseja posicionar</label>
  <select id="place">
    <option value="Predio 1">Prédio 1</option>
    <option value="Predio 2">Prédio 2</option>
    <option value="Predio 3">Prédio 3</option>
    <option value="Predio 4">Prédio 4</option>
    <option value="Anfiteatro">Anfiteatro</option>
  </select>
  <button id="start">Iniciar experiência RA</button>
  <button id="placeBtn" class="secondary" disabled>Posicionar ponto selecionado</button>
  <button id="reset" class="secondary">Limpar pontos desta sessão</button>
  <div id="status">Toque em “Iniciar experiência RA”, permita o uso da câmera e aponte para uma superfície plana.</div>
  <div id="points">Pontos posicionados: 0/5</div>
  <div class="hint">Regra do trabalho: o aplicativo verifica se cada novo ponto está a pelo menos 50 metros dos pontos já posicionados. A medição usa o espaço local rastreado pelo aparelho durante esta sessão.</div>
</section>
<footer>Requer navegador e aparelho compatíveis com WebXR AR (normalmente Android com serviços ARCore). Use em ambiente bem iluminado.</footer>
</div>

<script type="module">
import * as THREE from 'three';

const sceneRoot = document.getElementById('scene');
const statusEl = document.getElementById('status');
const pointsEl = document.getElementById('points');
const startBtn = document.getElementById('start');
const placeBtn = document.getElementById('placeBtn');
const resetBtn = document.getElementById('reset');
const selectEl = document.getElementById('place');
const MIN_DISTANCE = 50;
const placed = new Map();

let renderer, scene, camera, reticle, hitTestSource = null, localSpace = null;
let currentSession = null;
let latestHitMatrix = null;

function status(message){ statusEl.textContent = message; }
function refreshPoints(){
  pointsEl.textContent = `Pontos posicionados: ${placed.size}/5` +
    (placed.size ? ' — ' + [...placed.keys()].join(', ') : '');
}
function makeLabel(text){
  const canvas = document.createElement('canvas');
  canvas.width = 512; canvas.height = 128;
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = '#991b1b'; ctx.roundRect(8,8,496,112,24); ctx.fill();
  ctx.strokeStyle = '#fff'; ctx.lineWidth = 6; ctx.stroke();
  ctx.fillStyle = '#fff'; ctx.font = 'bold 46px Arial';
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle'; ctx.fillText(text,256,66);
  const texture = new THREE.CanvasTexture(canvas);
  const sprite = new THREE.Sprite(new THREE.SpriteMaterial({map:texture, transparent:true, depthTest:false}));
  sprite.scale.set(0.9,0.225,1);
  sprite.position.set(0,0.62,0);
  return sprite;
}
function makePin(name){
  const group = new THREE.Group();
  const stem = new THREE.Mesh(
    new THREE.CylinderGeometry(0.025,0.025,0.28,12),
    new THREE.MeshStandardMaterial({color:0x991b1b})
  );
  stem.position.y=0.15; group.add(stem);
  const head = new THREE.Mesh(
    new THREE.SphereGeometry(0.12,20,16),
    new THREE.MeshStandardMaterial({color:0xef4444, roughness:.35, metalness:.05})
  );
  head.position.y=0.34; group.add(head);
  const label=makeLabel(name); group.add(label);
  return group;
}
function initThree(){
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera();
  scene.add(new THREE.HemisphereLight(0xffffff,0x667788,1.5));
  const light = new THREE.DirectionalLight(0xffffff,1.1); light.position.set(1,4,2); scene.add(light);
  renderer = new THREE.WebGLRenderer({antialias:true,alpha:true});
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2));
  renderer.setSize(window.innerWidth,window.innerHeight);
  renderer.xr.enabled=true;
  sceneRoot.appendChild(renderer.domElement);
  reticle = new THREE.Mesh(
    new THREE.RingGeometry(0.10,0.14,32),
    new THREE.MeshBasicMaterial({color:0x22c55e})
  );
  reticle.rotation.x=-Math.PI/2; reticle.matrixAutoUpdate=false; reticle.visible=false; scene.add(reticle);
  window.addEventListener('resize',()=>{
    if(!renderer)return;
    renderer.setSize(window.innerWidth,window.innerHeight);
  });
}
async function startAR(){
  if(!navigator.xr){status('Este navegador não disponibiliza WebXR. Tente abrir o projeto no Chrome de um aparelho Android compatível com ARCore.');return;}
  if(!await navigator.xr.isSessionSupported('immersive-ar')){
    status('O aparelho ou navegador não oferece suporte a sessões WebXR immersive-ar.');return;
  }
  try{
    if(!renderer)initThree();
    const session=await navigator.xr.requestSession('immersive-ar',{
      requiredFeatures:['hit-test'],
      optionalFeatures:['dom-overlay'],
      domOverlay:{root:document.getElementById('overlay')}
    });
    currentSession=session;
    renderer.xr.setReferenceSpaceType('local');
    await renderer.xr.setSession(session);
    localSpace=await session.requestReferenceSpace('local');
    const viewerSpace=await session.requestReferenceSpace('viewer');
    hitTestSource=await session.requestHitTestSource({space:viewerSpace});
    session.addEventListener('select',()=>{
      if(latestHitMatrix) placeCurrentPoint();
      else status('Nenhuma superfície detectada. Mova lentamente o aparelho até aparecer o círculo verde.');
    });
    session.addEventListener('end',()=>{
      hitTestSource=null;localSpace=null;currentSession=null;latestHitMatrix=null;
      reticle.visible=false;placeBtn.disabled=true;startBtn.disabled=false;
      status('Sessão encerrada. Os pontos 3D são válidos somente durante a sessão atual.');
    });
    startBtn.disabled=true;
    placeBtn.disabled=false;
    status('RA ativa. Mova o celular até o círculo verde aparecer no chão e toque na tela ou em “Posicionar ponto”.');
    renderer.setAnimationLoop((time,frame)=>{
      if(frame && hitTestSource){
        const results=frame.getHitTestResults(hitTestSource);
        if(results.length){
          const pose=results[0].getPose(localSpace);
          reticle.visible=true;
          reticle.matrix.fromArray(pose.transform.matrix);
          latestHitMatrix=pose.transform.matrix;
        }else{
          reticle.visible=false;latestHitMatrix=null;
        }
      }
      renderer.render(scene,camera);
    });
  }catch(err){
    status('Não foi possível iniciar a RA: '+err.message+'. Verifique permissões da câmera e compatibilidade do aparelho.');
  }
}
function positionFromMatrix(matrix){
  const p=new THREE.Vector3().setFromMatrixPosition(new THREE.Matrix4().fromArray(matrix));
  return p;
}
function placeCurrentPoint(){
  if(!latestHitMatrix || !scene){status('Aponte para uma superfície até o círculo verde aparecer.');return;}
  const name=selectEl.value;
  if(placed.has(name)){status(name+' já está posicionado. Escolha outro ponto ou limpe a sessão.');return;}
  const pos=positionFromMatrix(latestHitMatrix);
  for(const [otherName,other] of placed.entries()){
    const distance=pos.distanceTo(other.position);
    if(distance<MIN_DISTANCE){
      status(`Ponto não colocado: ${name} está a ${distance.toFixed(1)} m de ${otherName}. A distância mínima exigida é ${MIN_DISTANCE} m.`);
      return;
    }
  }
  const pin=makePin(name);
  pin.position.copy(pos);
  scene.add(pin);
  placed.set(name,{position:pos.clone(),object:pin});
  refreshPoints();
  status(name+' posicionado com sucesso. Selecione o próximo local.');
}
startBtn.addEventListener('click',startAR);
placeBtn.addEventListener('click',placeCurrentPoint);
resetBtn.addEventListener('click',()=>{
  for(const item of placed.values())scene?.remove(item.object);
  placed.clear();refreshPoints();
  status('Pontos removidos da sessão atual. Você pode posicioná-los novamente.');
});
</script>
</body>
</html>
