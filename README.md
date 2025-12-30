<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>Sistema – Painel Master</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<!-- Firebase compat -->
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-firestore-compat.js"></script>

<style>
body{margin:0;font-family:Arial;background:#f1f5f9}
.hidden{display:none}
.container{max-width:1100px;margin:auto;padding:20px}
header{background:#1e40af;color:#fff;padding:15px}
button{padding:8px 12px;border:none;border-radius:6px;cursor:pointer}
input,select,textarea{width:100%;padding:8px;margin-top:6px}
.card{background:#fff;padding:15px;border-radius:12px;margin-bottom:10px}
.red{background:#ef4444;color:#fff}
.green{background:#22c55e;color:#fff}
.gray{background:#64748b;color:#fff}
.small{font-size:12px;color:#555}
</style>
</head>

<body>

<!-- LOGIN -->
<div id="loginScreen" class="container">
  <h2>Login</h2>
  <input id="loginUser" placeholder="Usuário">
  <input id="loginPass" type="password" placeholder="Senha">
  <button onclick="login()">Entrar</button>
  <p id="loginError" style="color:red"></p>
</div>

<!-- APP -->
<div id="appScreen" class="hidden">
<header>
  <b id="userInfo"></b>
  <button onclick="logout()" style="float:right">Sair</button>
</header>

<div class="container">

<!-- PAINEL MASTER -->
<div id="masterPanel" class="hidden">
  <h2>🔐 Painel Master</h2>

  <div class="card">
    <h3>Cadastrar / Editar Usuário</h3>
    <input id="uLogin" placeholder="Login">
    <input id="uPass" placeholder="Senha">
    <select id="uRole">
      <option>Usuário</option>
      <option>Admin</option>
    </select>
    <button class="green" onclick="saveUser()">Salvar</button>
  </div>

  <div class="card">
    <h3>📥 Importar em Massa</h3>
    <p class="small">Formato: login[TAB]senha[TAB]role</p>
    <textarea id="bulkText" rows="5"></textarea>
    <button onclick="bulkImport()">Importar</button>
  </div>

  <div class="card">
    <h3>👥 Usuários</h3>
    <input id="userSearch" placeholder="Pesquisar..." onkeyup="loadUsers()">
    <div id="usersList"></div>
  </div>
</div>

</div>
</div>

<script>
/* 🔥 FIREBASE CONFIG (SEU) */
const firebaseConfig = {
  apiKey: "AIzaSyB-uGBd-kGVmap9UtwttleUdfn1_Y4TFYM",
  authDomain: "gerenciado-de-renda-mesal.firebaseapp.com",
  projectId: "gerenciado-de-renda-mesal",
  storageBucket: "gerenciado-de-renda-mesal.firebasestorage.app",
  messagingSenderId: "555940377892",
  appId: "1:555940377892:web:e7f32c6ac32cfc52f7afb1"
};

firebase.initializeApp(firebaseConfig);
const db = firebase.firestore();

/* MASTER FIXO */
const MASTER = { login:"master", pass:"1234" };
let currentUser = null;

/* LOGIN */
async function login(){
  const u = loginUser.value.trim().toLowerCase();
  const p = loginPass.value;
  loginError.innerText = "";

  if(u === MASTER.login && p === MASTER.pass){
    currentUser = {login:"MASTER", role:"Master"};
    openApp();
    return;
  }

  const ref = db.collection("users").doc(u);
  const snap = await ref.get();
  if(!snap.exists) return loginError.innerText="Usuário não encontrado";

  const d = snap.data();
  if(d.pass !== p) return loginError.innerText="Senha incorreta";
  if(d.blocked) return loginError.innerText="Usuário bloqueado";

  currentUser = d;
  openApp();
}

function openApp(){
  loginScreen.classList.add("hidden");
  appScreen.classList.remove("hidden");
  userInfo.innerText = currentUser.login;

  if(currentUser.role === "Master"){
    masterPanel.classList.remove("hidden");
    loadUsers();
  }
}

function logout(){ location.reload(); }

/* USERS */
function saveUser(){
  const id = uLogin.value.trim().toLowerCase();
  if(!id) return;

  db.collection("users").doc(id).set({
    login:uLogin.value.trim(),
    pass:uPass.value,
    role:uRole.value,
    blocked:false
  });

  uLogin.value="";uPass.value="";
  loadUsers();
}

function loadUsers(){
  usersList.innerHTML="";
  const q = userSearch.value.toLowerCase();

  db.collection("users").onSnapshot(snap=>{
    usersList.innerHTML="";
    snap.forEach(doc=>{
      const u = doc.data();
      if(!u.login.toLowerCase().includes(q)) return;

      const div = document.createElement("div");
      div.className="card";
      div.innerHTML=`
        <b>${u.login}</b><br>
        <small>${u.role}</small><br><br>
        <button class="gray" onclick="toggleBlock('${doc.id}',${u.blocked})">
          ${u.blocked?'Desbloquear':'Bloquear'}
        </button>
        <button class="red" onclick="delUser('${doc.id}')">Excluir</button>
      `;
      usersList.appendChild(div);
    });
  });
}

function toggleBlock(id,state){
  db.collection("users").doc(id).update({blocked:!state});
}

function delUser(id){
  if(confirm("Excluir usuário?")){
    db.collection("users").doc(id).delete();
  }
}

function bulkImport(){
  bulkText.value.split("\n").forEach(l=>{
    const p=l.split("\t");
    if(p[0]){
      db.collection("users").doc(p[0].toLowerCase()).set({
        login:p[0],
        pass:p[1]||"1234",
        role:p[2]||"Usuário",
        blocked:false
      });
    }
  });
  bulkText.value="";
  loadUsers();
}
</script>

</body>
</html>
