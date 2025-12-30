<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Código Fonte - Sistema de Demandas</title>
    <style>
        body {
            background-color: #0f172a;
            color: #f1f5f9;
            font-family: monospace;
            margin: 0;
            padding: 20px;
            line-height: 1.5;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
            background-color: #1e293b;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
            overflow-x: auto;
        }
        pre {
            white-space: pre-wrap;
            word-wrap: break-word;
        }
        code {
            color: #38bdf8;
        }
        .header {
            margin-bottom: 20px;
            border-bottom: 1px solid #334155;
            padding-bottom: 10px;
        }
    </style>
</head>
<body>
    <div class="header">
        <h2>Visualização do Código Fonte</h2>
        <p>Formato: React JS / Firebase</p>
    </div>
    <div class="container">
<pre><code>
import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  query, 
  onSnapshot, 
  addDoc, 
  updateDoc, 
  deleteDoc, 
  doc, 
  serverTimestamp,
  setDoc,
  getDoc,
  writeBatch
} from 'firebase/firestore';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, 
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  Layout, 
  Plus, 
  CheckCircle2, 
  User, 
  Calendar, 
  LogOut, 
  Send,
  Settings,
  Lock,
  UserPlus, 
  Trash2, 
  ShieldCheck, 
  Search, 
  Edit2, 
  Ban, 
  Unlock, 
  ChevronDown, 
  X, 
  FileUp, 
  AlertCircle, 
  Check, 
  Filter 
} from 'lucide-react';

// Firebase Configuration
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'demandas-v1';

const CATEGORIES = [
  "Meio Ambiente", "Linguagens", "Comunicações", "Edição de Vídeo", 
  "Cultura", "Secretaria", "Esportes", "Presidência", "Informações", "Designer"
];

const ROLES = ["Colaborador", "Estagiário", "Diretor"];

const MASTER_CONFIG = {
  login: "CLX",
  pass: "02072007"
};

export default function App() {
  const [user, setUser] = useState(null); 
  const [currentUser, setCurrentUser] = useState(null); 
  const [isLoggedIn, setIsLoggedIn] = useState(false);
  const [tasks, setTasks] = useState([]);
  const [systemUsers, setSystemUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  
  // UI States
  const [taskToEdit, setTaskToEdit] = useState(null);
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [isAdminPanelOpen, setIsAdminPanelOpen] = useState(false);
  const [loginForm, setLoginForm] = useState({ user: '', pass: '' });
  const [loginError, setLoginError] = useState('');
  
  // Search States
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedColaborador, setSelectedColaborador] = useState(null);

  // Admin User Filter States
  const [adminUserFilter, setAdminUserFilter] = useState({ category: 'Todos', role: 'Todos', search: '' });

  const [newUser, setNewUser] = useState({ login: '', pass: '', category: 'Secretaria', role: 'Colaborador', blocked: false });
  const [editingUser, setEditingUser] = useState(null);

  // Bulk Import States
  const [importText, setImportText] = useState('');
  const [importStatus, setImportStatus] = useState({ loading: false, message: '', type: '' });

  // Privileges
  const isMaster = useMemo(() => currentUser?.isMaster, [currentUser]);
  
  const isManagement = useMemo(() => {
    if (!currentUser) return false;
    return currentUser.isMaster || currentUser.category === 'Presidência';
  }, [currentUser]);

  const isDirector = useMemo(() => currentUser?.role === 'Diretor', [currentUser]);

  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          try {
            await signInWithCustomToken(auth, __initial_auth_token);
          } catch (tokenErr) {
            await signInAnonymously(auth);
          }
        } else {
          await signInAnonymously(auth);
        }
      } catch (error) {
        console.error("Auth error:", error);
      } finally {
        setLoading(false);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!isLoggedIn || !user) return;

    const tasksRef = collection(db, 'artifacts', appId, 'public', 'data', 'tasks');
    const unsubTasks = onSnapshot(query(tasksRef), (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setTasks(data.sort((a, b) => (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0)));
    }, (error) => console.error("Tasks error:", error));

    const usersRef = collection(db, 'artifacts', appId, 'public', 'data', 'users');
    const unsubUsers = onSnapshot(query(usersRef), (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setSystemUsers(data);
    }, (error) => console.error("Users error:", error));

    return () => {
      unsubTasks();
      unsubUsers();
    };
  }, [isLoggedIn, user, appId]);

  const handleLogin = async (e) => {
    e.preventDefault();
    if (!user) return;
    setLoginError('');

    const inputLogin = loginForm.user.trim().toLowerCase();

    if (inputLogin === MASTER_CONFIG.login.toLowerCase() && loginForm.pass === MASTER_CONFIG.pass) {
      setCurrentUser({ login: MASTER_CONFIG.login, category: 'MASTER', role: 'Administrador', isMaster: true });
      setIsLoggedIn(true);
      return;
    }

    try {
      const userRef = doc(db, 'artifacts', appId, 'public', 'data', 'users', inputLogin);
      const userDoc = await getDoc(userRef);
      if (userDoc.exists()) {
        const data = userDoc.data();
        if (data.blocked) { setLoginError('Acesso bloqueado.'); return; }
        if (data.pass === loginForm.pass) {
          setCurrentUser({ ...data, isMaster: false });
          setIsLoggedIn(true);
        } else { setLoginError('Senha incorreta.'); }
      } else { setLoginError('Usuário não encontrado.'); }
    } catch (err) { 
      setLoginError('Erro ao validar acesso.'); 
    }
  };

  const handleLogout = () => {
    setIsLoggedIn(false);
    setCurrentUser(null);
    setLoginForm({ user: '', pass: '' });
    setLoginError('');
    setSelectedColaborador(null);
    setSearchQuery('');
  };

  // CORREÇÃO: Função auxiliar para normalizar strings para comparação
  const normalize = (str) => (str || "").trim().toLowerCase();

  const searchableUsers = useMemo(() => {
    if (!currentUser) return [];
    
    // Filtramos primeiro para não mostrar o MASTER na lista de busca
    let list = systemUsers.filter(u => normalize(u.login) !== normalize(MASTER_CONFIG.login));

    if (isManagement) {
      return list;
    }
    
    if (isDirector) {
      // Se for diretor, só vê quem é da mesma categoria (normalizada)
      return list.filter(u => normalize(u.category) === normalize(currentUser.category));
    }
    
    return [];
  }, [systemUsers, currentUser, isManagement, isDirector]);

  const adminFilteredUsers = useMemo(() => {
    return systemUsers
      .filter(u => normalize(u.login) !== normalize(MASTER_CONFIG.login))
      .filter(u => {
        const userCategory = normalize(u.category);
        const filterCategory = normalize(adminUserFilter.category);
        const userRole = normalize(u.role);
        const filterRole = normalize(adminUserFilter.role);

        const matchCategory = filterCategory === 'todos' || userCategory === filterCategory;
        const matchRole = filterRole === 'todos' || userRole === filterRole;
        const matchSearch = !adminUserFilter.search || normalize(u.login).includes(normalize(adminUserFilter.search));
        
        return matchCategory && matchRole && matchSearch;
      });
  }, [systemUsers, adminUserFilter]);

  const filteredTasks = useMemo(() => {
    if (!currentUser) return [];
    let baseList = tasks;
    
    if (selectedColaborador) {
       return baseList.filter(t => normalize(t.assignedTo) === normalize(selectedColaborador.login));
    }

    if (isManagement) return baseList;
    
    if (isDirector) {
      return baseList.filter(t => normalize(t.category) === normalize(currentUser.category));
    }

    // Colaborador comum vê apenas as suas tarefas no seu setor
    return baseList.filter(t => 
      normalize(t.category) === normalize(currentUser.category) && 
      normalize(t.assignedTo) === normalize(currentUser.login)
    );
  }, [tasks, currentUser, isManagement, isDirector, selectedColaborador]);

  const saveTask = async (taskData) => {
    if (!user) return;
    try {
      if (taskToEdit) {
        await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'tasks', taskToEdit.id), { ...taskData, updatedAt: serverTimestamp() });
      } else {
        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'tasks'), {
          ...taskData,
          status: 'pendente',
          createdBy: currentUser.login,
          createdAt: serverTimestamp(),
          comments: []
        });
      }
      setTaskToEdit(null);
    } catch (err) { console.error(err); }
  };

  const deleteTask = async (id) => {
    if (!user) return;
    try { await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'tasks', id)); } catch (err) { console.error(err); }
  };

  const saveUser = async (e) => {
    e.preventDefault();
    if (!newUser.login || !isMaster || !user) return;
    
    const normalizedId = newUser.login.trim().toLowerCase();
    
    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', normalizedId), { 
        ...newUser,
        login: newUser.login.trim(),
        category: newUser.category.trim(),
        role: newUser.role.trim(),
        updatedAt: new Date().toISOString() 
      });
      setNewUser({ login: '', pass: '', category: 'Secretaria', role: 'Colaborador', blocked: false });
      setEditingUser(null);
    } catch (err) { console.error(err); }
  };

  const processBulkImport = async () => {
    if (!importText.trim() || !user || !isMaster) return;
    setImportStatus({ loading: true, message: 'Processando...', type: 'info' });

    try {
      const lines = importText.split('\n');
      const batch = writeBatch(db);
      let count = 0;

      for (let line of lines) {
        if (!line.trim()) continue;
        const parts = line.split(/\t/).map(s => s.trim());
        
        if (parts.length >= 2) {
          const login = parts[0];
          const birthday = parts[1];
          let categoryInput = parts[2] || 'Secretaria';
          let role = parts[3] || 'Colaborador';

          // Tentar encontrar a categoria correta ignorando case/espaços
          const matchedCategory = CATEGORIES.find(c => normalize(c) === normalize(categoryInput)) || categoryInput;

          const userRef = doc(db, 'artifacts', appId, 'public', 'data', 'users', login.toLowerCase());
          
          batch.set(userRef, {
            login: login,
            pass: birthday.replace(/\//g, '').replace(/-/g, ''), 
            category: matchedCategory,
            role: role,
            blocked: false,
            updatedAt: new Date().toISOString()
          });
          count++;
        }
      }

      await batch.commit();
      setImportStatus({ loading: false, message: `${count} usuários processados com sucesso!`, type: 'success' });
      setImportText('');
      setTimeout(() => setImportStatus({ loading: false, message: '', type: '' }), 4000);
    } catch (err) {
      console.error(err);
      setImportStatus({ loading: false, message: 'Erro ao processar importação.', type: 'error' });
    }
  };

  const toggleUserBlock = async (userData) => {
    if (!user || !isMaster) return;
    try { await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', userData.login.toLowerCase()), { blocked: !userData.blocked }); } catch (err) { console.error(err); }
  };

  const deleteUser = async (login) => {
    if (!user || !isMaster) return;
    try { await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', login.toLowerCase())); } catch (err) { console.error(err); }
  };

  if (loading) return &lt;div className="h-screen flex items-center justify-center bg-slate-950 text-white font-bold tracking-widest animate-pulse uppercase text-xs"&gt;Carregando Sistema...&lt;/div&gt;;

  if (!isLoggedIn) {
    return (
      &lt;div className="min-h-screen bg-slate-950 flex items-center justify-center p-4"&gt;
        &lt;div className="bg-slate-900 border border-slate-800 w-full max-w-md rounded-[2.5rem] p-10 shadow-2xl text-slate-100"&gt;
          &lt;div className="flex flex-col items-center mb-8 text-center"&gt;
            &lt;div className="bg-indigo-600 p-5 rounded-3xl mb-4 shadow-lg shadow-indigo-500/20"&gt;&lt;Lock className="text-white" size={32} /&gt;&lt;/div&gt;
            &lt;h1 className="text-2xl font-black uppercase tracking-tighter"&gt;Portal Interno&lt;/h1&gt;
            &lt;p className="text-slate-500 text-sm font-medium"&gt;Gestão de Demandas&lt;/p&gt;
          &lt;/div&gt;
          &lt;form onSubmit={handleLogin} className="space-y-4"&gt;
            &lt;input type="text" required placeholder="Utilizador" className="w-full bg-slate-800 border border-slate-700 rounded-2xl px-6 py-4 outline-none focus:ring-2 focus:ring-indigo-500 font-bold uppercase" value={loginForm.user} onChange={e =&gt; setLoginForm({...loginForm, user: e.target.value})} /&gt;
            &lt;input type="password" required placeholder="Palavra-passe" className="w-full bg-slate-800 border border-slate-700 rounded-2xl px-6 py-4 outline-none focus:ring-2 focus:ring-indigo-500 font-bold" value={loginForm.pass} onChange={e =&gt; setLoginForm({...loginForm, pass: e.target.value})} /&gt;
            {loginError && &lt;div className="p-3 bg-red-500/10 text-red-400 text-xs font-bold rounded-xl text-center border border-red-500/20"&gt;{loginError}&lt;/div&gt;}
            &lt;button type="submit" className="w-full bg-indigo-600 hover:bg-indigo-500 text-white font-black py-5 rounded-2xl transition-all shadow-lg shadow-indigo-600/20 active:scale-95 uppercase text-xs tracking-widest"&gt;Aceder à Plataforma&lt;/button&gt;
          &lt;/form&gt;
        &lt;/div&gt;
      &lt;/div&gt;
    );
  }

  return (
    &lt;div className="min-h-screen bg-slate-50 text-slate-900 pb-20 font-sans"&gt;
      &lt;header className="bg-white border-b border-slate-200 sticky top-0 z-30 shadow-sm"&gt;
        &lt;div className="max-w-7xl mx-auto px-6 h-20 flex items-center justify-between"&gt;
          &lt;div className="flex items-center gap-3 font-black text-2xl text-indigo-700"&gt;
            &lt;div className="bg-indigo-600 p-2 rounded-xl text-white shadow-md shadow-indigo-200"&gt;&lt;Layout size={24} /&gt;&lt;/div&gt;
            &lt;span className="tracking-tighter uppercase"&gt;Demandas&lt;/span&gt;
          &lt;/div&gt;
          &lt;div className="flex items-center gap-4"&gt;
            &lt;div className="text-right hidden sm:block"&gt;
              &lt;p className="text-[10px] uppercase font-black text-indigo-500 leading-none tracking-widest mb-1"&gt;{currentUser.role} • {currentUser.category}&lt;/p&gt;
              &lt;p className="text-sm font-bold text-slate-700 uppercase"&gt;{currentUser.login}&lt;/p&gt;
            &lt;/div&gt;
            {isMaster && (
              &lt;button onClick={() =&gt; setIsAdminPanelOpen(true)} className="p-3 bg-slate-100 hover:bg-indigo-600 text-slate-600 hover:text-white rounded-2xl transition-all shadow-sm"&gt;
                &lt;Settings size={20} /&gt;
              &lt;/button&gt;
            )}
            &lt;button onClick={handleLogout} className="p-3 bg-red-50 hover:bg-red-500 text-red-500 hover:text-white rounded-2xl transition-all shadow-sm"&gt;
              &lt;LogOut size={20} /&gt;
            &lt;/button&gt;
          &lt;/div&gt;
        &lt;/div&gt;
      &lt;/header&gt;

      &lt;main className="max-w-7xl mx-auto px-6 py-10"&gt;
        {(isManagement || isDirector) && (
          &lt;div className="mb-10 p-6 bg-white border border-slate-100 rounded-[2rem] shadow-sm"&gt;
            &lt;div className="flex flex-col md:flex-row gap-4 items-center"&gt;
              &lt;div className="flex-1 w-full relative"&gt;
                &lt;Search className="absolute left-5 top-1/2 -translate-y-1/2 text-slate-400" size={18} /&gt;
                &lt;input 
                  type="text" 
                  placeholder="Pesquisar colaborador para ver demandas..." 
                  className="w-full pl-14 pr-6 py-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold outline-none focus:ring-2 focus:ring-indigo-500 uppercase"
                  value={searchQuery}
                  onChange={(e) =&gt; setSearchQuery(e.target.value)}
                /&gt;
                
                {searchQuery && (
                  &lt;div className="absolute top-full left-0 right-0 mt-2 bg-white border border-slate-200 rounded-2xl shadow-2xl z-40 max-h-60 overflow-y-auto"&gt;
                    {searchableUsers
                      .filter(u =&gt; normalize(u.login).includes(normalize(searchQuery)))
                      .map(u =&gt; (
                        &lt;button 
                          key={u.login}
                          onClick={() =&gt; { setSelectedColaborador(u); setSearchQuery(''); }}
                          className="w-full flex items-center justify-between p-4 hover:bg-indigo-50 border-b border-slate-50 last:border-0 transition-colors"
                        &gt;
                          &lt;div className="flex items-center gap-3"&gt;
                            &lt;div className="bg-indigo-100 p-2 rounded-lg text-indigo-600"&gt;&lt;User size={16}/&gt;&lt;/div&gt;
                            &lt;div className="text-left"&gt;
                              &lt;p className="font-bold text-slate-800 uppercase text-xs"&gt;{u.login}&lt;/p&gt;
                              &lt;p className="text-[10px] text-slate-400 font-bold uppercase tracking-widest"&gt;{u.role} • {u.category}&lt;/p&gt;
                            &lt;/div&gt;
                          &lt;/div&gt;
                          &lt;ChevronDown className="-rotate-90 text-slate-300" size={16} /&gt;
                        &lt;/button&gt;
                      ))}
                    {searchableUsers.filter(u =&gt; normalize(u.login).includes(normalize(searchQuery))).length === 0 && (
                        &lt;div className="p-6 text-center text-slate-400 text-xs font-bold uppercase"&gt;Nenhum colaborador encontrado&lt;/div&gt;
                    )}
                  &lt;/div&gt;
                )}
              &lt;/div&gt;
              
              {selectedColaborador && (
                &lt;div className="flex items-center gap-3 bg-indigo-600 text-white px-6 py-4 rounded-2xl shadow-lg shadow-indigo-100 animate-in fade-in slide-in-from-right-2"&gt;
                  &lt;div className="text-left"&gt;
                    &lt;p className="text-[9px] font-black uppercase tracking-tighter opacity-70 leading-none mb-1"&gt;Filtrando por&lt;/p&gt;
                    &lt;p className="font-black uppercase text-sm"&gt;{selectedColaborador.login}&lt;/p&gt;
                  &lt;/div&gt;
                  &lt;button onClick={() =&gt; setSelectedColaborador(null)} className="p-1 hover:bg-white/20 rounded-lg transition-all"&gt;
                    &lt;X size={18} /&gt;
                  &lt;/button&gt;
                &lt;/div&gt;
              )}
            &lt;/div&gt;
          &lt;/div&gt;
        )}

        &lt;div className="flex flex-col sm:flex-row justify-between items-start sm:items-center mb-12 gap-6"&gt;
          &lt;div&gt;
            &lt;h2 className="text-4xl font-black text-slate-900 tracking-tight mb-2 uppercase"&gt;
              {selectedColaborador ? 'Demandas de ' + selectedColaborador.login : 'Painel de Controle'}
            &lt;/h2&gt;
            &lt;p className="text-slate-500 font-medium"&gt;
              {selectedColaborador 
                ? `Visualizando tarefas atribuídas a ${selectedColaborador.login.toUpperCase()}` 
                : `Setor atual: ${isManagement ? 'Visão Geral' : currentUser.category}`
              }
            &lt;/p&gt;
          &lt;/div&gt;
          {(isManagement || isDirector) && (
            &lt;button onClick={() =&gt; { setTaskToEdit(null); setIsModalOpen(true); }} className="bg-indigo-600 hover:bg-indigo-700 text-white px-10 py-5 rounded-[2rem] font-black flex items-center gap-3 shadow-2xl shadow-indigo-200 w-full sm:w-auto justify-center transition-all active:scale-95 text-xs tracking-widest uppercase"&gt;
              &lt;Plus size={20} /&gt; Nova Demanda
            &lt;/button&gt;
          )}
        &lt;/div&gt;

        &lt;div className="grid grid-cols-1 gap-6"&gt;
          {filteredTasks.length === 0 ? (
            &lt;div className="bg-white border-2 border-dashed border-slate-200 rounded-[3rem] p-32 text-center"&gt;
              &lt;Search className="mx-auto mb-6 text-slate-100" size={80} /&gt;
              &lt;p className="text-slate-400 font-black uppercase text-xs tracking-widest"&gt;Nenhuma demanda encontrada&lt;/p&gt;
            &lt;/div&gt;
          ) : (
            filteredTasks.map(task =&gt; (
              &lt;TaskCard 
                key={task.id} 
                task={task} 
                currentUser={currentUser} 
                isManagement={isManagement}
                db={db} 
                appId={appId} 
                onEdit={() =&gt; { setTaskToEdit(task); setIsModalOpen(true); }}
                onDelete={() =&gt; deleteTask(task.id)}
              /&gt;
            ))
          )}
        &lt;/div&gt;
      &lt;/main&gt;

      {isAdminPanelOpen && isMaster && (
        &lt;div className="fixed inset-0 z-50 flex items-center justify-center p-4 sm:p-6 bg-slate-900/80 backdrop-blur-xl"&gt;
          &lt;div className="bg-white rounded-[3rem] w-full max-w-6xl shadow-2xl overflow-hidden flex flex-col max-h-[90vh] animate-in zoom-in-95 duration-200 border border-white/20 text-slate-900"&gt;
            &lt;div className="flex justify-between items-center p-8 sm:p-10 border-b bg-slate-50"&gt;
              &lt;div&gt;
                &lt;h2 className="text-2xl font-black flex items-center gap-3 text-slate-800 uppercase tracking-tighter"&gt;
                  &lt;ShieldCheck className="text-indigo-600" size={28} /&gt; Gestão Master
                &lt;/h2&gt;
                &lt;p className="text-xs text-slate-400 font-bold uppercase tracking-widest"&gt;Painel administrativo&lt;/p&gt;
              &lt;/div&gt;
              &lt;button onClick={() =&gt; { setIsAdminPanelOpen(false); setEditingUser(null); }} className="bg-slate-200 p-3 rounded-full text-slate-500 hover:bg-red-500 hover:text-white transition-all shadow-sm"&gt;
                &lt;Plus size={24} className="rotate-45" /&gt;
              &lt;/button&gt;
            &lt;/div&gt;
            
            &lt;div className="p-8 sm:p-10 overflow-y-auto flex-1 custom-scrollbar"&gt;
              &lt;div className="grid grid-cols-1 lg:grid-cols-2 gap-10 mb-12"&gt;
                &lt;div className="p-8 bg-white border border-slate-100 rounded-[2.5rem] shadow-sm"&gt;
                  &lt;h3 className="text-xs font-black uppercase tracking-widest text-indigo-600 mb-6 flex items-center gap-2"&gt;
                    &lt;UserPlus size={16}/&gt; Cadastro Individual
                  &lt;/h3&gt;
                  &lt;form onSubmit={saveUser} className="space-y-4"&gt;
                    &lt;div className="grid grid-cols-2 gap-4"&gt;
                      &lt;input 
                        required 
                        placeholder="Login (pode conter espaços)" 
                        disabled={!!editingUser} 
                        className="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold outline-none focus:ring-2 focus:ring-indigo-500 disabled:opacity-50 uppercase" 
                        value={newUser.login} 
                        onChange={e =&gt; setNewUser({...newUser, login: e.target.value})} 
                      /&gt;
                      &lt;input required placeholder="Senha" className="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold outline-none focus:ring-2 focus:ring-indigo-500" value={newUser.pass} onChange={e =&gt; setNewUser({...newUser, pass: e.target.value})} /&gt;
                    &lt;/div&gt;
                    &lt;div className="grid grid-cols-2 gap-4"&gt;
                      &lt;select className="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold outline-none focus:ring-2 focus:ring-indigo-500 text-xs uppercase" value={newUser.category} onChange={e =&gt; setNewUser({...newUser, category: e.target.value})}&gt;
                        {CATEGORIES.map(c =&gt; &lt;option key={c} value={c}&gt;{c}&lt;/option&gt;) || "Não definido"}
                      &lt;/select&gt;
                      &lt;select className="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold outline-none focus:ring-2 focus:ring-indigo-500 text-xs uppercase" value={newUser.role} onChange={e =&gt; setNewUser({...newUser, role: e.target.value})}&gt;
                        {ROLES.map(r =&gt; &lt;option key={r} value={r}&gt;{r}&lt;/option&gt;)}
                      &lt;/select&gt;
                    &lt;/div&gt;
                    &lt;button type="submit" className="w-full bg-slate-900 text-white p-4 rounded-2xl font-black hover:bg-indigo-600 transition-all text-[10px] uppercase tracking-widest shadow-lg"&gt;
                      {editingUser ? 'Atualizar Colaborador' : 'Adicionar Colaborador'}
                    &lt;/button&gt;
                    {editingUser && &lt;button type="button" onClick={() =&gt; {setEditingUser(null); setNewUser({login:'', pass:'', category:'Secretaria', role:'Colaborador', blocked:false})}} className="w-full text-slate-400 text-[10px] font-black uppercase"&gt;Cancelar Edição&lt;/button&gt;}
                  &lt;/form&gt;
                &lt;/div&gt;

                &lt;div className="p-8 bg-indigo-50/50 border border-indigo-100 rounded-[2.5rem] shadow-sm"&gt;
                  &lt;h3 className="text-xs font-black uppercase tracking-widest text-indigo-600 mb-2 flex items-center gap-2"&gt;
                    &lt;FileUp size={16}/&gt; Importar do Excel
                  &lt;/h3&gt;
                  &lt;p className="text-[9px] text-indigo-400 font-bold uppercase mb-4 leading-relaxed"&gt;
                    Copie e cole as colunas (separadas por TAB):&lt;br/&gt;
                    &lt;span className="bg-white/50 px-1 rounded font-mono"&gt;NOME  |  NASCIMENTO  |  SETOR  |  CARGO&lt;/span&gt;
                  &lt;/p&gt;
                  &lt;textarea 
                    placeholder="Ex: Ana Maria	15051995	Cultura	Estagiário"
                    className="w-full h-32 p-4 bg-white border border-indigo-100 rounded-2xl font-mono text-[10px] outline-none focus:ring-2 focus:ring-indigo-500 mb-4 shadow-inner"
                    value={importText}
                    onChange={e =&gt; setImportText(e.target.value)}
                  /&gt;
                  {importStatus.message && (
                    &lt;div className={`p-3 mb-4 rounded-xl text-[10px] font-black uppercase flex items-center gap-2 ${importStatus.type === 'success' ? 'bg-emerald-100 text-emerald-600' : importStatus.type === 'error' ? 'bg-red-100 text-red-600' : 'bg-blue-100 text-blue-600'}`}&gt;
                      {importStatus.type === 'success' ? &lt;Check size={14}/&gt; : &lt;AlertCircle size={14}/&gt;}
                      {importStatus.message}
                    &lt;/div&gt;
                  )}
                  &lt;button 
                    disabled={!importText.trim() || importStatus.loading}
                    onClick={processBulkImport}
                    className="w-full bg-indigo-600 text-white p-4 rounded-2xl font-black hover:bg-indigo-700 transition-all text-[10px] uppercase tracking-widest disabled:opacity-50 shadow-lg shadow-indigo-100"
                  &gt;
                    {importStatus.loading ? 'Importando...' : 'Processar Lista'}
                  &lt;/button&gt;
                &lt;/div&gt;
              &lt;/div&gt;

              &lt;div className="mb-8 p-6 bg-slate-50 border border-slate-100 rounded-3xl"&gt;
                &lt;div className="flex flex-col md:flex-row gap-4"&gt;
                  &lt;div className="flex-1 relative"&gt;
                    &lt;Search className="absolute left-4 top-1/2 -translate-y-1/2 text-slate-400" size={16} /&gt;
                    &lt;input 
                      type="text" 
                      placeholder="Pesquisar colaborador..." 
                      className="w-full pl-12 pr-4 py-3 bg-white border border-slate-200 rounded-xl text-xs font-bold uppercase outline-none focus:ring-2 focus:ring-indigo-500"
                      value={adminUserFilter.search}
                      onChange={e =&gt; setAdminUserFilter({...adminUserFilter, search: e.target.value})}
                    /&gt;
                  &lt;/div&gt;
                  &lt;div className="flex gap-4"&gt;
                    &lt;select 
                      className="pl-4 pr-10 py-3 bg-white border border-slate-200 rounded-xl text-[10px] font-black uppercase outline-none focus:ring-2 focus:ring-indigo-500"
                      value={adminUserFilter.category}
                      onChange={e =&gt; setAdminUserFilter({...adminUserFilter, category: e.target.value})}
                    &gt;
                      &lt;option value="Todos"&gt;Todas Categorias&lt;/option&gt;
                      {CATEGORIES.map(c =&gt; &lt;option key={c} value={c}&gt;{c}&lt;/option&gt;)}
                    &lt;/select&gt;
                    &lt;select 
                      className="pl-4 pr-10 py-3 bg-white border border-slate-200 rounded-xl text-[10px] font-black uppercase outline-none focus:ring-2 focus:ring-indigo-500"
                      value={adminUserFilter.role}
                      onChange={e =&gt; setAdminUserFilter({...adminUserFilter, role: e.target.value})}
                    &gt;
                      &lt;option value="Todos"&gt;Todos Cargos&lt;/option&gt;
                      {ROLES.map(r =&gt; &lt;option key={r} value={r}&gt;{r}&lt;/option&gt;)}
                    &lt;/select&gt;
                  &lt;/div&gt;
                &lt;/div&gt;
                &lt;div className="mt-4 flex items-center justify-between"&gt;
                  &lt;p className="text-[10px] font-black text-slate-400 uppercase tracking-widest"&gt;
                    Total: {adminFilteredUsers.length} usuários filtrados
                  &lt;/p&gt;
                &lt;/div&gt;
              &lt;/div&gt;
              
              &lt;div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-5 pb-10"&gt;
                {adminFilteredUsers.map(u =&gt; (
                  &lt;div key={u.id} className={`flex items-center justify-between p-6 border rounded-[2rem] transition-all bg-white border-slate-100 hover:shadow-xl group ${u.blocked ? 'bg-red-50/30 grayscale opacity-60' : ''}`}&gt;
                    &lt;div className="flex items-center gap-4"&gt;
                      &lt;div className={`p-3 rounded-xl ${u.blocked ? 'bg-red-100 text-red-600' : 'bg-indigo-50 text-indigo-600'}`}&gt;&lt;User size={20} /&gt;&lt;/div&gt;
                      &lt;div&gt;
                        &lt;p className="font-black text-slate-800 tracking-tight text-sm uppercase"&gt;{u.login}&lt;/p&gt;
                        &lt;p className="text-[9px] text-slate-400 font-bold uppercase tracking-widest"&gt;{(u.category || "Sem Setor")} • {(u.role || "Sem Cargo")}&lt;/p&gt;
                      &lt;/div&gt;
                    &lt;/div&gt;
                    &lt;div className="flex gap-2 opacity-0 group-hover:opacity-100 transition-all scale-90 group-hover:scale-100"&gt;
                      &lt;button onClick={() =&gt; toggleUserBlock(u)} title="Bloquear/Desbloquear" className={`p-2 rounded-lg ${u.blocked ? 'text-emerald-500 bg-emerald-50' : 'text-amber-500 bg-amber-50'}`}&gt;
                        {u.blocked ? &lt;Unlock size={16} /&gt; : &lt;Ban size={16} /&gt;}
                      &lt;/button&gt;
                      &lt;button onClick={() =&gt; { setEditingUser(u); setNewUser({...u}); }} title="Editar" className="p-2 text-indigo-500 bg-indigo-50 rounded-lg"&gt;
                        &lt;Edit2 size={16} /&gt;
                      &lt;/button&gt;
                      &lt;button onClick={() =&gt; deleteUser(u.login)} title="Excluir" className="p-2 text-red-500 bg-red-50 rounded-lg"&gt;
                        &lt;Trash2 size={16} /&gt;
                      &lt;/button&gt;
                    &lt;/div&gt;
                  &lt;/div&gt;
                ))}
              &lt;/div&gt;
            &lt;/div&gt;
          &lt;/div&gt;
        &lt;/div&gt;
      )}

      {isModalOpen && (
        &lt;ModalTask 
          onClose={() =&gt; { setIsModalOpen(false); setTaskToEdit(null); }} 
          onSave={saveTask} 
          currentUser={currentUser} 
          isManagement={isManagement}
          systemUsers={systemUsers}
          categories={CATEGORIES} 
          initialData={taskToEdit}
        /&gt;
      )}
    &lt;/div&gt;
  );
}

function TaskCard({ task, currentUser, isManagement, db, appId, onEdit, onDelete }) {
  const [comment, setComment] = useState('');
  const canManage = isManagement || (currentUser.role === 'Diretor' && normalize(task.category) === normalize(currentUser.category));

  const normalize = (str) =&gt; (str || "").trim().toLowerCase();

  const toggleStatus = () =&gt; updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'tasks', task.id), { status: task.status === 'concluída' ? 'pendente' : 'concluída' });
  const addComment = (e) =&gt; {
    e.preventDefault();
    if (!comment.trim()) return;
    updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'tasks', task.id), { 
      comments: [...(task.comments || []), { id: Date.now(), user: currentUser.login, text: comment, date: new Date().toISOString() }] 
    });
    setComment('');
  };

  return (
    &lt;div className={`bg-white border rounded-[2.5rem] p-8 transition-all ${task.status === 'concluída' ? 'border-emerald-100 bg-emerald-50/10' : 'border-slate-100 shadow-sm hover:shadow-2xl hover:shadow-slate-200'}`}&gt;
      &lt;div className="flex gap-8"&gt;
        &lt;button onClick={toggleStatus} className={`mt-1 w-10 h-10 rounded-2xl border-2 flex items-center justify-center transition-all flex-shrink-0 ${task.status === 'concluída' ? 'bg-emerald-500 border-emerald-500 text-white' : 'border-slate-200 hover:border-indigo-500 text-transparent'}`}&gt;
          &lt;CheckCircle2 size={24} /&gt;
        &lt;/button&gt;
        &lt;div className="flex-1"&gt;
          &lt;div className="flex items-center justify-between mb-4"&gt;
            &lt;div className="flex gap-2"&gt;
              &lt;span className="text-[9px] font-black uppercase text-indigo-500 bg-indigo-50 px-4 py-1.5 rounded-full tracking-widest border border-indigo-100"&gt;{task.category}&lt;/span&gt;
              &lt;span className="text-[9px] font-black uppercase text-slate-400 bg-slate-50 px-4 py-1.5 rounded-full tracking-widest border border-slate-100"&gt;Para: {task.assignedTo}&lt;/span&gt;
            &lt;/div&gt;
            {canManage && (
              &lt;div className="flex gap-2"&gt;
                &lt;button onClick={onEdit} className="p-2 text-slate-300 hover:text-indigo-500 transition-all"&gt;&lt;Edit2 size={18} /&gt;&lt;/button&gt;
                &lt;button onClick={onDelete} className="p-2 text-slate-300 hover:text-red-500 transition-all"&gt;&lt;Trash2 size={18} /&gt;&lt;/button&gt;
              &lt;/div&gt;
            )}
          &lt;/div&gt;
          &lt;h3 className={`font-black text-2xl mb-2 tracking-tight uppercase ${task.status === 'concluída' ? 'line-through text-slate-400' : 'text-slate-800'}`}&gt;{task.title}&lt;/h3&gt;
          &lt;p className="text-slate-500 text-base mb-8 leading-relaxed font-medium"&gt;{task.description}&lt;/p&gt;
          
          &lt;div className="flex flex-wrap gap-5 text-[10px] font-black text-slate-400 mb-8 bg-slate-50 p-4 rounded-2xl w-fit border border-slate-100 uppercase tracking-widest"&gt;
            &lt;span className="flex items-center gap-2"&gt;&lt;User size={14} className="text-indigo-500"/&gt; Autor: {task.createdBy}&lt;/span&gt;
            &lt;span className="flex items-center gap-2"&gt;&lt;Calendar size={14} className="text-indigo-500"/&gt; {task.createdAt?.toDate ? task.createdAt.toDate().toLocaleDateString('pt-PT') : 'Hoje'}&lt;/span&gt;
          &lt;/div&gt;

          &lt;div className="bg-slate-50/50 p-6 rounded-[2rem] border border-slate-100"&gt;
            &lt;div className="space-y-3 mb-6 max-h-48 overflow-y-auto custom-scrollbar pr-2"&gt;
              {(task.comments || []).length === 0 ? (
                &lt;p className="text-[10px] italic text-slate-300 font-bold uppercase tracking-[0.2em] ml-2"&gt;Sem notas técnicas&lt;/p&gt;
              ) : (
                (task.comments || []).map(c =&gt; (
                  &lt;div key={c.id} className="text-sm p-4 bg-white rounded-2xl shadow-sm border border-slate-100"&gt;
                    &lt;div className="flex justify-between items-center mb-2"&gt;
                      &lt;span className="font-black text-indigo-600 uppercase text-[10px] tracking-widest"&gt;{c.user}&lt;/span&gt;
                      &lt;span className="text-[9px] text-slate-300 font-bold uppercase"&gt;{new Date(c.date).toLocaleTimeString()}&lt;/span&gt;
                    &lt;/div&gt;
                    &lt;p className="text-slate-600 font-medium"&gt;{c.text}&lt;/p&gt;
                  &lt;/div&gt;
                ))
              )}
            &lt;/div&gt;
            &lt;form onSubmit={addComment} className="flex gap-3"&gt;
              &lt;input value={comment} onChange={e =&gt; setComment(e.target.value)} placeholder="Inserir observação..." className="flex-1 text-sm bg-white border border-slate-200 p-4 rounded-2xl outline-none focus:ring-2 focus:ring-indigo-500 font-medium" /&gt;
              &lt;button className="bg-indigo-600 text-white px-6 rounded-2xl shadow-lg shadow-indigo-100 hover:scale-105 active:scale-95 transition-all"&gt;&lt;Send size={18} /&gt;&lt;/button&gt;
            &lt;/form&gt;
          &lt;/div&gt;
        &lt;/div&gt;
      &lt;/div&gt;
    &lt;/div&gt;
  );
}

function ModalTask({ onClose, onSave, currentUser, isManagement, systemUsers, categories, initialData }) {
  const [data, setData] = useState(initialData || { 
    title: '', description: '', assignedTo: '', 
    category: isManagement ? 'Presidência' : currentUser.category 
  });

  const normalize = (str) =&gt; (str || "").trim().toLowerCase();

  const categoryTeam = useMemo(() =&gt; {
    if (!data.category) return [];
    const targetCat = normalize(data.category);
    return systemUsers.filter(u =&gt; normalize(u.category) === targetCat && normalize(u.login) !== normalize(MASTER_CONFIG.login));
  }, [systemUsers, data.category]);

  return (
    &lt;div className="fixed inset-0 z-50 flex items-center justify-center p-6 bg-slate-900/70 backdrop-blur-xl"&gt;
      &lt;div className="bg-white rounded-[3.5rem] w-full max-w-xl p-12 shadow-2xl animate-in zoom-in-95 duration-200 border border-white/20"&gt;
        &lt;h2 className="font-black text-slate-900 uppercase text-xs tracking-[0.3em] mb-10 flex items-center gap-3"&gt;
          &lt;Plus className="text-indigo-600" size={18}/&gt; {initialData ? 'Editar Solicitação' : 'Registar Demanda'}
        &lt;/h2&gt;
        &lt;div className="space-y-6"&gt;
          &lt;div&gt;
            &lt;label className="text-[10px] font-black text-slate-400 uppercase ml-3 mb-1 block tracking-widest"&gt;Assunto&lt;/label&gt;
            &lt;input required placeholder="Título da atividade..." className="w-full p-5 border bg-slate-50 border-slate-100 rounded-[1.5rem] font-bold outline-none focus:ring-2 focus:ring-indigo-500 transition-all shadow-inner uppercase" value={data.title} onChange={e =&gt; setData({...data, title: e.target.value})} /&gt;
          &lt;/div&gt;
          &lt;div&gt;
            &lt;label className="text-[10px] font-black text-slate-400 uppercase ml-3 mb-1 block tracking-widest"&gt;Instruções&lt;/label&gt;
            &lt;textarea placeholder="Detalhe o que precisa de ser executado..." className="w-full p-6 border bg-slate-50 border-slate-100 rounded-[1.5rem] h-36 resize-none outline-none focus:ring-2 focus:ring-indigo-500 font-medium text-sm transition-all shadow-inner" value={data.description} onChange={e =&gt; setData({...data, description: e.target.value})} /&gt;
          &lt;/div&gt;
          
          &lt;div className="grid grid-cols-2 gap-5"&gt;
            &lt;div&gt;
              &lt;label className="text-[10px] font-black text-slate-400 uppercase ml-3 mb-1 block tracking-widest"&gt;Setor Alvo&lt;/label&gt;
              &lt;div className="relative"&gt;
                &lt;select className="w-full p-5 border bg-slate-50 border-slate-100 rounded-[1.5rem] font-black text-xs outline-none focus:ring-2 focus:ring-indigo-500 appearance-none cursor-pointer uppercase" 
                  value={data.category} 
                  disabled={!isManagement} 
                  onChange={e =&gt; setData({...data, category: e.target.value, assignedTo: ''})}
                &gt;
                  {isManagement ? (
                    categories.map(c =&gt; &lt;option key={c} value={c}&gt;{c}&lt;/option&gt;)
                  ) : (
                    &lt;option value={currentUser.category}&gt;{currentUser.category}&lt;/option&gt;
                  )}
                &lt;/select&gt;
                {isManagement && &lt;ChevronDown className="absolute right-4 top-1/2 -translate-y-1/2 text-slate-300" size={16} /&gt;}
              &lt;/div&gt;
            &lt;/div&gt;
            &lt;div&gt;
              &lt;label className="text-[10px] font-black text-slate-400 uppercase ml-3 mb-1 block tracking-widest"&gt;Responsável&lt;/label&gt;
              &lt;div className="relative"&gt;
                &lt;select 
                  className="w-full p-5 border bg-slate-50 border-slate-100 rounded-[1.5rem] font-black text-xs outline-none focus:ring-2 focus:ring-indigo-500 appearance-none cursor-pointer uppercase"
                  value={data.assignedTo}
                  onChange={e =&gt; setData({...data, assignedTo: e.target.value})}
                &gt;
                  &lt;option value=""&gt;Escolher...&lt;/option&gt;
                  {categoryTeam.map(u =&gt; (
                    &lt;option key={u.login} value={u.login}&gt;{u.login.toUpperCase()}&lt;/option&gt;
                  ))}
                &lt;/select&gt;
                &lt;ChevronDown className="absolute right-4 top-1/2 -translate-y-1/2 text-slate-300" size={16} /&gt;
              &lt;/div&gt;
            &lt;/div&gt;
          &lt;/div&gt;

          &lt;div className="pt-8"&gt;
            &lt;button onClick={() =&gt; { if(data.title && data.assignedTo) { onSave(data); onClose(); } }} className="w-full bg-indigo-600 text-white font-black py-6 rounded-[2rem] shadow-2xl shadow-indigo-200 uppercase text-xs tracking-[0.2em] hover:bg-indigo-700 hover:-translate-y-1 transition-all active:scale-95"&gt;
              {initialData ? 'Atualizar Dados' : 'Enviar Demanda'}
            &lt;/button&gt;
            &lt;button onClick={onClose} className="w-full text-slate-300 font-black py-4 text-[10px] uppercase tracking-widest mt-2 hover:text-slate-500 transition-colors"&gt;Cancelar Operação&lt;/button&gt;
          &lt;/div&gt;
        &lt;/div&gt;
      &lt;/div&gt;
    &lt;/div&gt;
  );
}
</code></pre>
    </div>
</body>
</html>
