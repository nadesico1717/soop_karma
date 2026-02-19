import React, { useState, useEffect } from 'react';
import { 
  Plus, Trash2, Check, Trophy, Gamepad2, Mic2, Shirt, 
  AlertCircle, Sparkles, Settings, X, Minus, Tag, User, 
  Edit2, Save, Users, UserPlus, ChevronRight, AlertTriangle,
  Lock, Unlock, Key, LogOut, ShieldCheck, Link as LinkIcon, ExternalLink,
  Share2, Info, ArrowRight
} from 'lucide-react';

// --- Firebase Imports ---
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, // 미리보기 환경용 인증 추가
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  doc, 
  addDoc, 
  updateDoc, 
  deleteDoc, 
  setDoc,
  onSnapshot
} from 'firebase/firestore';

// --- ⚠️ GitHub Pages 배포 설정 영역 ⚠️ ---
// 1. Firebase 콘솔 -> 프로젝트 설정 -> 내 앱 -> SDK 설정 및 구성의 내용을 아래에 복사하세요.
const ghPagesConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID"
};

// 2. 환경 감지 및 설정 선택
// Canvas 미리보기 환경에서는 __firebase_config를 사용하고,
// GitHub Pages 등 외부 환경에서는 위에서 정의한 ghPagesConfig를 사용합니다.
const firebaseConfig = typeof __firebase_config !== 'undefined'
  ? JSON.parse(__firebase_config)
  : ghPagesConfig;

// 3. 앱 식별자 (데이터베이스에 저장될 최상위 폴더명, 원하는 영문으로 변경 가능)
const APP_ID = typeof __app_id !== 'undefined' ? __app_id : 'my-karma-dashboard';

const App = () => {
  // --- 1. Firebase Initialization & Global Vars ---
  const [user, setUser] = useState(null);
  const [db, setDb] = useState(null);
  const [isOnline, setIsOnline] = useState(true);

  // --- 2. Initial Data & State ---
  const initialCategories = [
    { id: 'Game', label: '게임', color: 'blue', icon: 'Gamepad2' },
    { id: 'Dance', label: '댄스', color: 'purple', icon: 'Sparkles' },
    { id: 'Costume', label: '의상', color: 'pink', icon: 'Shirt' },
    { id: 'Food', label: '먹방/벌칙', color: 'orange', icon: 'AlertCircle' },
    { id: 'Etc', label: '기타', color: 'gray', icon: 'Mic2' },
  ];

  // State
  const [categories, setCategories] = useState(initialCategories);
  const [viewers, setViewers] = useState([]);
  const [karmas, setKarmas] = useState([]);
  const [streamerName, setStreamerName] = useState('');
   
  // App Logic State
  const [channelId, setChannelId] = useState('');
  const [isLanding, setIsLanding] = useState(true); 
  const [landingInput, setLandingInput] = useState('');

  const [selectedViewer, setSelectedViewer] = useState('All');
  const [isAdmin, setIsAdmin] = useState(false);
  const [storedPassword, setStoredPassword] = useState('0000'); 
  const [showLoginModal, setShowLoginModal] = useState(false);
  const [showAdminMenu, setShowAdminMenu] = useState(false);
  const [passwordInput, setPasswordInput] = useState('');
  const [newPasswordInput, setNewPasswordInput] = useState('');
  const [isEditingName, setIsEditingName] = useState(false);

  // Input Forms
  const [newKarma, setNewKarma] = useState('');
  const [newCategory, setNewCategory] = useState('Game');
  const [newCount, setNewCount] = useState(1);
  const [targetViewerInput, setTargetViewerInput] = useState('');
  const [newViewerName, setNewViewerName] = useState('');
  const [newViewerUrl, setNewViewerUrl] = useState('');
  const [filter, setFilter] = useState('All');
   
  // UI State
  const [editingId, setEditingId] = useState(null);
  const [editForm, setEditForm] = useState({ title: '', count: 0, initialCount: 0, viewer: '' });
  const [isCatManagerOpen, setIsCatManagerOpen] = useState(false);
  const [customCatName, setCustomCatName] = useState('');
  const [isViewerAdding, setIsViewerAdding] = useState(false);
  const [deleteModal, setDeleteModal] = useState({ isOpen: false, viewerId: null, viewerName: null });

  // --- 3. Firebase Setup & URL Parsing ---
  useEffect(() => {
    // 1. Firebase Init
    const app = initializeApp(firebaseConfig);
    const auth = getAuth(app);
    const firestore = getFirestore(app);
    
    setDb(firestore);

    // 2. Hash Check logic (To handle refresh and deep links)
    const checkHash = () => {
      const hash = window.location.hash.replace('#', '');
      if (hash) {
        const decoded = decodeURIComponent(hash);
        setChannelId(decoded);
        setIsLanding(false);
      }
    };

    checkHash();
    window.addEventListener('hashchange', checkHash);

    // 3. Authentication (Hybrid: Preview or Production)
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          // Preview environment with custom token
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          // GitHub environment with anonymous login
          await signInAnonymously(auth);
        }
      } catch (error) {
        console.error("Auth Error:", error);
        setIsOnline(false);
      }
    };

    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      setUser(u);
      setIsOnline(!!u);
    });

    return () => {
      unsubscribe();
      window.removeEventListener('hashchange', checkHash);
    };
  }, []);

  // --- 4. Firestore Data Listeners ---
  useEffect(() => {
    if (!user || !db || !channelId) return;

    // 1. Karmas Listener
    const karmasRef = collection(db, 'artifacts', APP_ID, 'public', 'data', 'karmas');
    const unsubKarmas = onSnapshot(karmasRef, (snapshot) => {
      const data = snapshot.docs
        .map(doc => ({ id: doc.id, ...doc.data() }))
        .filter(item => item.channelId === channelId);
      data.sort((a, b) => b.createdAt - a.createdAt);
      setKarmas(data);
    }, (err) => console.error("Firestore Error:", err));

    // 2. Viewers Listener
    const viewersRef = collection(db, 'artifacts', APP_ID, 'public', 'data', 'viewers');
    const unsubViewers = onSnapshot(viewersRef, (snapshot) => {
      const data = snapshot.docs
        .map(doc => ({ id: doc.id, ...doc.data() }))
        .filter(item => item.channelId === channelId);
      setViewers(data);
    });

    // 3. Categories Listener
    const catsRef = collection(db, 'artifacts', APP_ID, 'public', 'data', 'categories');
    const unsubCats = onSnapshot(catsRef, (snapshot) => {
      const data = snapshot.docs
        .map(doc => ({ id: doc.id, ...doc.data() }))
        .filter(item => item.channelId === channelId);
      if (data.length > 0) setCategories(data);
      else setCategories(initialCategories);
    });

    // 4. Settings Listener (Channel Settings)
    const settingsRef = collection(db, 'artifacts', APP_ID, 'public', 'data', 'channel_settings');
    const unsubSettings = onSnapshot(settingsRef, (snapshot) => {
        const mySettings = snapshot.docs.find(d => d.id === channelId);
        if (mySettings) {
            const data = mySettings.data();
            if (data.streamerName) setStreamerName(data.streamerName);
            if (data.password) setStoredPassword(data.password);
        } else {
            setStreamerName(channelId);
            setStoredPassword('0000');
        }
    });

    return () => {
      unsubKarmas();
      unsubViewers();
      unsubCats();
      unsubSettings();
    };
  }, [user, db, channelId]);

  // --- 5. Handlers ---
  const handleChannelSubmit = (e) => {
    e.preventDefault();
    if (!landingInput.trim()) return;
    const cleanId = landingInput.trim(); 
    setChannelId(cleanId);
    setIsLanding(false);
    // Update hash for sharing
    window.location.hash = encodeURIComponent(cleanId);
  };

  const updateSettings = async (field, value) => {
    if (!db || !user || !channelId) return;
    await setDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', 'channel_settings', channelId), { 
      [field]: value 
    }, { merge: true });
  };

  const addKarma = async (e) => {
    e.preventDefault();
    if (!newKarma.trim() || !db || !user) return;
    
    let finalViewer = '익명';
    if (selectedViewer !== 'All') {
      finalViewer = selectedViewer;
    } else if (targetViewerInput.trim()) {
      finalViewer = targetViewerInput.trim();
    }

    const countVal = parseInt(newCount) || 1;
    await addDoc(collection(db, 'artifacts', APP_ID, 'public', 'data', 'karmas'), {
      channelId: channelId,
      title: newKarma,
      category: newCategory,
      count: countVal,
      initialCount: countVal,
      viewer: finalViewer,
      completed: false,
      createdAt: Date.now()
    });
    setNewKarma('');
    setNewCount(1);
    setTargetViewerInput('');
  };

  const deleteKarma = async (id) => {
    if (!db || !user) return;
    await deleteDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', 'karmas', id));
  };

  const toggleComplete = async (item) => {
    if (!isAdmin || !db || !user) return;
    await updateDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', 'karmas', item.id), {
      completed: !item.completed
    });
  };

  const adjustCount = async (item, delta) => {
    if (!isAdmin || !db || !user) return;
    const nextCount = Math.max(0, item.count + delta);
    await updateDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', 'karmas', item.id), {
      count: nextCount,
      completed: nextCount === 0
    });
  };

  const saveEdit = async (id) => {
    if (!db || !user) return;
    const nextCount = parseInt(editForm.count) || 0;
    await updateDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', 'karmas', id), {
      title: editForm.title,
      count: nextCount,
      initialCount: parseInt(editForm.initialCount) || 1,
      viewer: editForm.viewer || '익명',
      completed: nextCount === 0
    });
    setEditingId(null);
  };

  const addViewer = async (e) => {
    e.preventDefault();
    if (!newViewerName.trim() || !db || !user) return;
    if (!viewers.some(v => v.name === newViewerName.trim())) {
      await addDoc(collection(db, 'artifacts', APP_ID, 'public', 'data', 'viewers'), {
        channelId: channelId,
        name: newViewerName.trim(),
        url: newViewerUrl.trim()
      });
    } else {
      alert('이미 존재하는 시청자입니다.');
    }
    setNewViewerName('');
    setNewViewerUrl('');
    setIsViewerAdding(false);
  };

  const confirmDeleteViewer = async () => {
    if (!db || !user) return;
    const { viewerId, viewerName } = deleteModal;
    if (viewerId) {
      await deleteDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', 'viewers', viewerId));
      if (selectedViewer === viewerName) setSelectedViewer('All');
    }
    setDeleteModal({ isOpen: false, viewerId: null, viewerName: null });
  };

  const addCategory = async () => {
    if (!customCatName.trim() || !db || !user) return;
    await addDoc(collection(db, 'artifacts', APP_ID, 'public', 'data', 'categories'), {
      channelId: channelId,
      label: customCatName,
      color: 'teal',
      icon: 'Tag',
      id: `custom-${Date.now()}`
    });
    setCustomCatName('');
  };

  const deleteCategory = async (catId) => {
    if (!db || !user) return;
    await deleteDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', 'categories', catId));
    if (newCategory === catId) setNewCategory('Game');
  };

  const handleLoginSubmit = (e) => {
    e.preventDefault();
    if (passwordInput === storedPassword) {
      setIsAdmin(true);
      setShowLoginModal(false);
      setPasswordInput('');
    } else {
      alert('비밀번호가 올바르지 않습니다.');
    }
  };

  const handleChangePassword = (e) => {
    e.preventDefault();
    if (newPasswordInput.length === 4 && !isNaN(newPasswordInput)) {
      updateSettings('password', newPasswordInput);
      setNewPasswordInput('');
      setShowAdminMenu(false);
      alert('비밀번호가 변경되었습니다.');
    } else {
      alert('비밀번호는 4자리 숫자여야 합니다.');
    }
  };

  const copyShareLink = () => {
    const url = window.location.href;
    
    // Fallback function using execCommand
    const copyFallback = () => {
      const textArea = document.createElement("textarea");
      textArea.value = url;
      textArea.style.top = "0";
      textArea.style.left = "0";
      textArea.style.position = "fixed";
      document.body.appendChild(textArea);
      textArea.focus();
      textArea.select();
      try {
        const successful = document.execCommand('copy');
        if (successful) {
          alert(`주소가 복사되었습니다!\n${url}`);
        } else {
          prompt("URL을 복사해주세요:", url);
        }
      } catch (err) {
        prompt("URL을 복사해주세요:", url);
      }
      document.body.removeChild(textArea);
    };

    if (navigator.clipboard && navigator.clipboard.writeText) {
      navigator.clipboard.writeText(url)
        .then(() => alert(`주소가 복사되었습니다!\n${url}`))
        .catch(() => copyFallback());
    } else {
      copyFallback();
    }
  };

  // --- UI Styling Helpers ---
  const getColorTheme = (colorName) => {
    const themes = {
      blue: 'bg-blue-100 text-blue-600 border-blue-200',
      purple: 'bg-purple-100 text-purple-600 border-purple-200',
      pink: 'bg-pink-100 text-pink-600 border-pink-200',
      orange: 'bg-orange-100 text-orange-600 border-orange-200',
      gray: 'bg-gray-100 text-gray-600 border-gray-200',
      green: 'bg-green-100 text-green-600 border-green-200',
      red: 'bg-red-100 text-red-600 border-red-200',
      teal: 'bg-teal-100 text-teal-600 border-teal-200',
    };
    return themes[colorName] || themes.gray;
  };

  const getIconComponent = (iconName, size = 16) => {
    const icons = {
      Gamepad2: <Gamepad2 size={size} />,
      Sparkles: <Sparkles size={size} />,
      Shirt: <Shirt size={size} />,
      AlertCircle: <AlertCircle size={size} />,
      Mic2: <Mic2 size={size} />,
      Tag: <Tag size={size} />
    };
    return icons[iconName] || <Tag size={size} />;
  };

  const filteredByViewer = selectedViewer === 'All' 
    ? karmas 
    : karmas.filter(k => k.viewer === selectedViewer);

  const finalDisplayKarmas = filteredByViewer.filter(k => {
    if (filter === 'Active') return !k.completed;
    if (filter === 'Completed') return k.completed;
    return true;
  });

  const totalCount = filteredByViewer.length;
  const completedCount = filteredByViewer.filter(k => k.completed).length;
  const progress = totalCount === 0 ? 0 : Math.round((completedCount / totalCount) * 100);
  const progressBarStyle = { width: `${progress}%` };

  const getViewerUrl = (viewerName) => {
    const viewer = viewers.find(v => v.name === viewerName);
    return viewer ? viewer.url : '';
  };

  // --- Render ---
   
  if (isLanding) {
    return (
      <div className="min-h-screen bg-slate-50 flex flex-col items-center justify-center p-4 font-sans">
        <div className="w-full max-w-md bg-white rounded-3xl shadow-xl overflow-hidden border border-slate-100 p-8 text-center animate-in zoom-in-95">
          <div className="w-20 h-20 bg-gradient-to-br from-indigo-500 to-purple-600 rounded-3xl flex items-center justify-center mx-auto mb-6 shadow-lg shadow-purple-200">
             <Trophy size={40} className="text-white" />
          </div>
          <h1 className="text-3xl font-bold text-slate-800 mb-2">업보 청산 대시보드</h1>
          <p className="text-slate-500 mb-8 text-sm">
            스트리머를 위한 실시간 업보 관리 서비스<br/>
            채널명을 입력하여 당신만의 업보판을 생성하세요.
          </p>
           
          <form onSubmit={handleChannelSubmit} className="space-y-4">
            <div className="relative">
              <input
                type="text"
                value={landingInput}
                onChange={(e) => setLandingInput(e.target.value)}
                placeholder="채널명 (예: 인정P)"
                className="w-full p-4 pl-6 text-lg bg-slate-50 border border-slate-200 rounded-2xl focus:outline-none focus:ring-4 focus:ring-purple-100 focus:border-purple-400 transition-all font-bold text-slate-700 text-center"
                autoFocus
              />
            </div>
            <button 
              type="submit" 
              className="w-full py-4 bg-slate-800 text-white font-bold text-lg rounded-2xl hover:bg-slate-900 transition-transform active:scale-95 flex items-center justify-center gap-2"
            >
              내 방 만들기 <ArrowRight size={20}/>
            </button>
          </form>

          <div className="mt-8 bg-purple-50 p-4 rounded-xl text-left border border-purple-100">
            <h3 className="text-xs font-bold text-purple-700 mb-2 flex items-center gap-1"><Info size={14}/> 처음 오셨나요?</h3>
            <ul className="text-xs text-slate-600 space-y-1 list-disc list-inside">
              <li>원하는 <strong>채널명</strong>을 입력하면 즉시 생성됩니다.</li>
              <li>초기 관리자 비밀번호는 <strong>0000</strong>입니다.</li>
              <li>접속 후 자물쇠(🔒)를 눌러 <strong>비밀번호를 변경</strong>하세요.</li>
              <li>우측 상단 <strong>공유 버튼</strong>으로 시청자를 초대하세요.</li>
            </ul>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-slate-50 text-slate-800 font-sans flex flex-col md:flex-row relative">
      <aside className="w-full md:w-64 bg-white border-b md:border-r border-slate-200 flex flex-col flex-shrink-0 z-10">
        <div className="p-4 border-b border-slate-100 bg-gradient-to-r from-indigo-500 to-purple-600 text-white flex justify-between items-center">
          <div>
            <h2 className="text-lg font-bold flex items-center gap-2">
              <Users size={20} /> 시청자 목록
            </h2>
            <div className="flex items-center gap-2 mt-1">
              <span className={`w-2 h-2 rounded-full ${isOnline ? 'bg-green-400' : 'bg-red-400'}`}></span>
              <p className="text-xs opacity-80">{isOnline ? '실시간 동기화 중' : '오프라인'}</p>
            </div>
          </div>
        </div>

        <div className="flex-1 overflow-x-auto md:overflow-y-auto p-2 space-x-2 md:space-x-0 md:space-y-1 flex md:block scrollbar-hide">
          <button
            onClick={() => setSelectedViewer('All')}
            className={`flex-shrink-0 w-auto md:w-full text-left px-4 py-3 rounded-xl transition-all flex items-center justify-between group ${
              selectedViewer === 'All' 
                ? 'bg-purple-100 text-purple-700 font-bold shadow-sm' 
                : 'hover:bg-slate-100 text-slate-600'
            }`}
          >
            <span className="flex items-center gap-2">
              <Users size={16} /> 전체 보기
            </span>
            {selectedViewer === 'All' && <ChevronRight size={16} className="hidden md:block" />}
          </button>

          {viewers.map(viewer => (
            <div
              key={viewer.id}
              onClick={() => setSelectedViewer(viewer.name)}
              className={`flex-shrink-0 w-auto md:w-full text-left px-4 py-3 rounded-xl transition-all flex items-center justify-between group relative cursor-pointer ${
                selectedViewer === viewer.name 
                  ? 'bg-purple-100 text-purple-700 font-bold shadow-sm' 
                  : 'hover:bg-slate-100 text-slate-600'
              }`}
              role="button"
            >
              <span className="flex items-center gap-2 pr-6 truncate max-w-[150px]">
                {viewer.url ? <LinkIcon size={14} className="text-purple-400"/> : <User size={16} />} 
                {viewer.name}
              </span>
              {isAdmin && (
                <button 
                  onClick={(e) => {
                    e.stopPropagation();
                    setDeleteModal({ isOpen: true, viewerId: viewer.id, viewerName: viewer.name });
                  }}
                  className="md:opacity-0 group-hover:opacity-100 p-1 hover:bg-red-100 text-slate-400 hover:text-red-500 rounded transition-all absolute right-2"
                  title="목록에서 제거"
                >
                  <Trash2 size={12} />
                </button>
              )}
            </div>
          ))}

          {isAdmin && (
            isViewerAdding ? (
              <form onSubmit={addViewer} className="p-2 bg-slate-50 rounded-xl border border-purple-100 m-1">
                <div className="flex flex-col gap-2">
                  <input
                    autoFocus
                    type="text"
                    value={newViewerName}
                    onChange={(e) => setNewViewerName(e.target.value)}
                    placeholder="닉네임"
                    className="w-full p-2 text-sm border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-200"
                  />
                  <input
                    type="text"
                    value={newViewerUrl}
                    onChange={(e) => setNewViewerUrl(e.target.value)}
                    placeholder="URL (선택)"
                    className="w-full p-2 text-sm border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-200"
                  />
                  <div className="flex gap-2 mt-1">
                    <button type="submit" className="flex-1 bg-purple-600 text-white p-2 rounded-lg text-xs font-bold">추가</button>
                    <button type="button" onClick={() => setIsViewerAdding(false)} className="flex-1 bg-slate-200 text-slate-600 p-2 rounded-lg text-xs font-bold">취소</button>
                  </div>
                </div>
              </form>
            ) : (
              <button
                onClick={() => setIsViewerAdding(true)}
                className="flex-shrink-0 w-auto md:w-full text-left px-4 py-3 rounded-xl text-slate-400 hover:bg-slate-50 hover:text-purple-600 transition-all border border-dashed border-slate-200 hover:border-purple-300 flex items-center justify-center gap-2 mt-2"
              >
                <UserPlus size={16} /> 시청자 추가
              </button>
            )
          )}
        </div>

        <div className="p-4 border-t border-slate-100 mt-auto hidden md:flex justify-between items-center bg-slate-50">
           <span className="text-xs text-slate-400 font-medium">관리자 전용</span>
           <button 
             onClick={() => isAdmin ? setShowAdminMenu(true) : setShowLoginModal(true)}
             className={`p-2 rounded-full transition-all ${isAdmin ? 'bg-purple-100 text-purple-600' : 'bg-slate-200 text-slate-400 hover:bg-slate-300'}`}
             title={isAdmin ? "관리자 설정" : "관리자 로그인"}
           >
             {isAdmin ? <Unlock size={16} /> : <Lock size={16} />}
           </button>
        </div>
      </aside>

      <main className="flex-1 flex flex-col h-screen overflow-hidden">
        <div className="flex-1 overflow-y-auto bg-slate-50 p-4 md:p-8">
          <div className="w-full max-w-4xl mx-auto bg-white rounded-3xl shadow-xl overflow-hidden border border-slate-100 relative">
            <div className="bg-gradient-to-r from-indigo-500 via-purple-500 to-pink-500 p-8 text-white relative">
              <div className="absolute top-4 right-4 flex gap-2">
                <button 
                   onClick={copyShareLink}
                   className="p-2 rounded-full bg-white/20 text-white hover:bg-white/30 backdrop-blur-md transition-all"
                   title="공유 링크 복사"
                >
                  <Share2 size={18} />
                </button>
                <div className="md:hidden">
                  <button 
                    onClick={() => isAdmin ? setShowAdminMenu(true) : setShowLoginModal(true)}
                    className={`p-2 rounded-full backdrop-blur-md transition-all ${isAdmin ? 'bg-white/30 text-white' : 'bg-black/20 text-white/70'}`}
                  >
                    {isAdmin ? <Unlock size={18} /> : <Lock size={18} />}
                  </button>
                </div>
              </div>

              <div className="flex justify-between items-start mb-6">
                <div>
                  <div className="flex items-center gap-2 h-10">
                    {isAdmin && isEditingName ? (
                      <div className="flex items-center gap-2 animate-in fade-in slide-in-from-left-2">
                        <input 
                          className="text-2xl font-bold text-indigo-900 rounded-lg px-3 py-1 w-48 shadow-lg focus:outline-none focus:ring-2 focus:ring-purple-300"
                          value={streamerName}
                          onChange={(e) => setStreamerName(e.target.value)}
                          onKeyDown={(e) => {
                            if (e.key === 'Enter') {
                              updateSettings('streamerName', streamerName);
                              setIsEditingName(false);
                            }
                          }}
                          autoFocus
                        />
                        <button onClick={() => { updateSettings('streamerName', streamerName); setIsEditingName(false); }} className="bg-white/20 p-2 rounded-lg hover:bg-white/30 transition-colors"><Check size={20}/></button>
                      </div>
                    ) : (
                      <h1 
                        onClick={() => isAdmin && setIsEditingName(true)}
                        className={`text-3xl font-bold flex items-center gap-2 transition-opacity select-none ${isAdmin ? 'cursor-pointer hover:opacity-90 group' : ''}`}
                      >
                        {streamerName || channelId} <span className="opacity-80 font-light">|</span> Dashboard
                        {isAdmin && <Edit2 size={18} className="opacity-0 group-hover:opacity-100 transition-opacity text-white/70" />}
                      </h1>
                    )}
                  </div>
                  <p className="opacity-90 mt-1 flex items-center gap-2">
                      {selectedViewer === 'All' ? '전체 업보 현황' : `${selectedViewer}님의 업보`}
                  </p>
                </div>
                <div className="hidden md:flex bg-white/20 backdrop-blur-md px-4 py-2 rounded-2xl flex-col items-center min-w-[80px]">
                  <span className="text-xs font-semibold uppercase tracking-wider opacity-80">진행률</span>
                  <span className="text-2xl font-bold">{progress}%</span>
                </div>
              </div>
              <div className="w-full bg-black/20 rounded-full h-4 mb-2 backdrop-blur-sm overflow-hidden">
                <div className="bg-white h-full rounded-full transition-all duration-700 ease-out shadow-[0_0_10px_rgba(255,255,255,0.5)]" style={progressBarStyle}></div>
              </div>
              <div className="flex justify-between text-sm opacity-90 font-medium px-1">
                <span>남은 항목: {totalCount - completedCount}개</span>
                <span>청산 완료: {completedCount}개</span>
              </div>
            </div>

            {isAdmin && (
              <div className="p-6 border-b border-slate-100 bg-slate-50/50 animate-in slide-in-from-top-4">
                <form onSubmit={addKarma} className="flex flex-col gap-3">
                  <div className="flex flex-wrap gap-2">
                    <div className="relative flex-grow md:flex-grow-0 w-1/3 md:w-1/4 min-w-[120px]">
                      <select 
                        value={newCategory}
                        onChange={(e) => setNewCategory(e.target.value)}
                        className="w-full p-3 pr-8 rounded-xl border border-slate-200 bg-white focus:outline-none focus:ring-2 focus:ring-purple-400 text-sm font-medium appearance-none"
                      >
                        {categories.map(cat => (
                          <option key={cat.id} value={cat.id}>{cat.label}</option>
                        ))}
                      </select>
                      <div className="absolute right-3 top-3.5 pointer-events-none text-slate-400">▼</div>
                    </div>

                    <button
                      type="button"
                      onClick={() => setIsCatManagerOpen(!isCatManagerOpen)}
                      className="p-3 bg-white border border-slate-200 rounded-xl text-slate-500 hover:bg-purple-50 hover:text-purple-600 transition-colors"
                    >
                      <Settings size={18} />
                    </button>
                    
                    <div className="relative w-20 flex-shrink-0">
                        <input 
                        type="number" 
                        min="1"
                        value={newCount}
                        onChange={(e) => setNewCount(e.target.value)}
                        placeholder="횟수"
                        className="w-full p-3 rounded-xl border border-slate-200 bg-white focus:outline-none focus:ring-2 focus:ring-purple-400 text-center font-bold"
                      />
                    </div>

                    {selectedViewer === 'All' ? (
                      <div className="relative flex-1 min-w-[140px]">
                        <div className="absolute left-3 top-3.5 text-slate-400"><User size={16} /></div>
                        <input 
                          type="text" 
                          value={targetViewerInput}
                          onChange={(e) => setTargetViewerInput(e.target.value)}
                          placeholder="닉네임 (비워두면 익명)"
                          className="w-full p-3 pl-9 rounded-xl border border-slate-200 bg-white focus:outline-none focus:ring-2 focus:ring-purple-400"
                        />
                      </div>
                    ) : (
                      <div className="flex items-center px-4 bg-purple-100 text-purple-700 rounded-xl font-bold border border-purple-200">
                        To. {selectedViewer}
                      </div>
                    )}
                  </div>

                  <div className="flex gap-2">
                    <input 
                      type="text" 
                      value={newKarma}
                      onChange={(e) => setNewKarma(e.target.value)}
                      placeholder="새로운 룰렛 업보 내용을 입력하세요..."
                      className="flex-1 p-3 rounded-xl border border-slate-200 bg-white focus:outline-none focus:ring-2 focus:ring-purple-400"
                    />
                    <button type="submit" className="bg-purple-600 hover:bg-purple-700 text-white px-6 py-3 rounded-xl font-bold transition-colors flex items-center justify-center gap-2 shadow-lg shadow-purple-200 whitespace-nowrap">
                      <Plus size={18} /> 추가
                    </button>
                  </div>
                </form>

                {isCatManagerOpen && (
                  <div className="mt-4 p-4 bg-white rounded-xl border border-purple-100 shadow-sm animate-in fade-in slide-in-from-top-2">
                    <div className="flex justify-between items-center mb-3">
                      <h3 className="text-sm font-bold text-slate-700">카테고리 관리</h3>
                      <button onClick={() => setIsCatManagerOpen(false)} className="text-slate-400 hover:text-slate-600"><X size={16}/></button>
                    </div>
                    <div className="flex gap-2 mb-4">
                      <input 
                        type="text" 
                        value={customCatName}
                        onChange={(e) => setCustomCatName(e.target.value)}
                        placeholder="새 이름"
                        className="flex-1 p-2 text-sm border border-slate-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-400"
                        onKeyDown={(e) => e.key === 'Enter' && addCategory()}
                      />
                      <button onClick={addCategory} className="px-4 py-2 bg-slate-800 text-white text-xs font-bold rounded-lg hover:bg-slate-700">생성</button>
                    </div>
                    <div className="flex flex-wrap gap-2">
                      {categories.map(cat => (
                        <div key={cat.id} className={`flex items-center gap-1 pl-3 pr-1 py-1 rounded-lg border text-xs font-bold ${getColorTheme(cat.color)}`}>
                          {cat.label}
                          <button onClick={() => deleteCategory(cat.id)} className="p-1 hover:bg-black/10 rounded-full ml-1"><X size={12} /></button>
                        </div>
                      ))}
                    </div>
                  </div>
                )}
              </div>
            )}

            <div className="flex border-b border-slate-100">
              {['All', 'Active', 'Completed'].map((tab) => (
                <button
                  key={tab}
                  onClick={() => setFilter(tab)}
                  className={`flex-1 py-4 text-sm font-bold transition-colors relative ${
                    filter === tab ? 'text-purple-600' : 'text-slate-400 hover:text-slate-600'
                  }`}
                >
                  {tab === 'All' ? '전체' : tab === 'Active' ? '진행 중' : '청산 완료'}
                  {filter === tab && <div className="absolute bottom-0 left-0 w-full h-0.5 bg-purple-600 rounded-t-full" />}
                </button>
              ))}
            </div>

            <div className="p-6 bg-white min-h-[400px]">
              {finalDisplayKarmas.length === 0 ? (
                <div className="h-64 flex flex-col items-center justify-center text-slate-400">
                  <Trophy size={48} className="mb-4 text-slate-200" />
                  <p>표시할 업보가 없습니다.</p>
                </div>
              ) : (
                <div className="space-y-3">
                  {finalDisplayKarmas.map((item) => {
                    const category = categories.find(c => c.id === item.category) || { color: 'gray', icon: 'Tag', label: '미분류' };
                    const themeClass = getColorTheme(category?.color || 'gray');
                    const viewerUrl = getViewerUrl(item.viewer);
                    const isEditing = editingId === item.id;

                    if (isEditing) {
                      return (
                        <div key={item.id} className="p-4 rounded-2xl border-2 border-purple-400 bg-purple-50 space-y-3">
                          <div className="flex gap-2">
                            <input
                              className="flex-1 p-2 border border-purple-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-400 text-sm font-bold"
                              value={editForm.title}
                              onChange={(e) => setEditForm({...editForm, title: e.target.value})}
                            />
                            <input
                              className="w-1/3 p-2 border border-purple-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-400 text-sm"
                              value={editForm.viewer}
                              onChange={(e) => setEditForm({...editForm, viewer: e.target.value})}
                            />
                          </div>
                          <div className="flex items-center gap-2 justify-between">
                             <div className="flex items-center gap-2 text-sm bg-white p-2 rounded-lg border border-purple-200">
                                <span className="text-slate-500 font-medium pl-1">남은:</span>
                                <input
                                  type="number"
                                  className="w-16 p-1 border border-slate-200 rounded text-center font-bold"
                                  value={editForm.count}
                                  onChange={(e) => setEditForm({...editForm, count: e.target.value})}
                                />
                                <span className="text-slate-400">/</span>
                                <span className="text-slate-500 font-medium">전체:</span>
                                <input
                                  type="number"
                                  className="w-16 p-1 border border-slate-200 rounded text-center"
                                  value={editForm.initialCount}
                                  onChange={(e) => setEditForm({...editForm, initialCount: e.target.value})}
                                />
                             </div>
                             <div className="flex gap-2">
                               <button onClick={() => saveEdit(item.id)} className="p-2 bg-purple-600 text-white rounded-lg hover:bg-purple-700"><Save size={18} /></button>
                               <button onClick={() => setEditingId(null)} className="p-2 bg-white border border-slate-200 text-slate-500 rounded-lg hover:bg-slate-100"><X size={18} /></button>
                             </div>
                          </div>
                        </div>
                      );
                    }

                    return (
                      <div 
                        key={item.id}
                        className={`group flex flex-col md:flex-row md:items-center p-4 rounded-2xl border transition-all duration-300 ${
                          item.completed 
                            ? 'bg-slate-50 border-slate-100 opacity-60' 
                            : 'bg-white border-slate-200 hover:border-purple-300 hover:shadow-md'
                        }`}
                      >
                        <div className="flex items-center w-full">
                          <button
                            onClick={() => toggleComplete(item)}
                            disabled={!isAdmin}
                            className={`flex-shrink-0 w-10 h-10 rounded-full flex items-center justify-center mr-4 transition-all ${
                              item.completed
                                ? 'bg-green-500 text-white scale-95'
                                : isAdmin 
                                  ? 'bg-slate-100 text-slate-300 hover:bg-purple-100 hover:text-purple-400'
                                  : 'bg-slate-50 text-slate-200 cursor-default'
                            }`}
                          >
                            <Check size={20} strokeWidth={3} />
                          </button>

                          <div className="flex-1 min-w-0 mr-4">
                            <div className="flex flex-wrap items-center gap-2 mb-1">
                              <span className={`px-2 py-0.5 rounded-md text-[10px] font-bold uppercase tracking-wider flex items-center gap-1 border ${themeClass}`}>
                                {getIconComponent(category?.icon)}
                                {category?.label}
                              </span>
                              
                              {viewerUrl ? (
                                <a 
                                  href={viewerUrl} 
                                  target="_blank" 
                                  rel="noopener noreferrer"
                                  className="flex items-center gap-1 px-2 py-0.5 rounded-md bg-slate-100 text-slate-500 text-[10px] font-medium border border-slate-200 hover:bg-purple-100 hover:text-purple-600 hover:border-purple-200 transition-colors"
                                >
                                   <User size={10} /> {item.viewer} <ExternalLink size={8} className="ml-0.5 opacity-50"/>
                                </a>
                              ) : (
                                <span className="flex items-center gap-1 px-2 py-0.5 rounded-md bg-slate-100 text-slate-500 text-[10px] font-medium border border-slate-200">
                                   <User size={10} /> {item.viewer}
                                </span>
                              )}

                              {item.completed && <span className="text-[10px] font-bold text-green-600 bg-green-100 px-2 py-0.5 rounded-md">CLEAR</span>}
                            </div>
                            <h3 className={`font-semibold text-lg truncate ${item.completed ? 'line-through text-slate-400' : 'text-slate-800'}`}>
                              {item.title}
                            </h3>
                          </div>

                          <div className={`flex items-center gap-3 bg-slate-50 rounded-xl px-3 py-2 mr-2 ${item.completed ? 'opacity-50 pointer-events-none' : ''}`}>
                             <button 
                               onClick={() => adjustCount(item, -1)} 
                               disabled={!isAdmin} 
                               className={`w-6 h-6 flex items-center justify-center rounded-full border transition-colors ${isAdmin ? 'bg-white border-slate-200 text-slate-500 hover:bg-red-50 hover:text-red-500 hover:border-red-200' : 'bg-slate-100 border-transparent text-slate-300 cursor-default'}`}
                             >
                               <Minus size={14} />
                             </button>
                             <div className="text-center min-w-[3rem]">
                               <span className="text-lg font-bold text-slate-700">{item.count}</span>
                               <span className="text-xs text-slate-400 ml-1">/ {item.initialCount}</span>
                             </div>
                             <button 
                               onClick={() => adjustCount(item, 1)} 
                               disabled={!isAdmin} 
                               className={`w-6 h-6 flex items-center justify-center rounded-full border transition-colors ${isAdmin ? 'bg-white border-slate-200 text-slate-500 hover:bg-blue-50 hover:text-blue-500 hover:border-blue-200' : 'bg-slate-100 border-transparent text-slate-300 cursor-default'}`}
                             >
                               <Plus size={14} />
                             </button>
                          </div>

                          {isAdmin && (
                            <div className="flex gap-1 ml-auto md:ml-0 opacity-100 md:opacity-0 group-hover:opacity-100 transition-opacity">
                               <button onClick={() => setEditingId(item.id) || setEditForm({title: item.title, count: item.count, initialCount: item.initialCount, viewer: item.viewer})} className="p-2 text-slate-300 hover:text-blue-500 hover:bg-blue-50 rounded-lg"><Edit2 size={18} /></button>
                              <button onClick={() => deleteKarma(item.id)} className="p-2 text-slate-300 hover:text-red-500 hover:bg-red-50 rounded-lg"><Trash2 size={18} /></button>
                            </div>
                          )}
                        </div>
                      </div>
                    );
                  })}
                </div>
              )}
            </div>
          </div>
        </div>
      </main>

      {/* Modals */}
      {showLoginModal && (
        <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/50 p-4 animate-in fade-in">
          <div className="bg-white rounded-2xl p-6 w-full max-w-xs shadow-2xl scale-100 animate-in zoom-in-95 relative">
            <button onClick={() => setShowLoginModal(false)} className="absolute right-4 top-4 text-slate-400 hover:text-slate-600"><X size={20}/></button>
            <div className="text-center mb-6">
              <div className="w-12 h-12 bg-purple-100 text-purple-600 rounded-full flex items-center justify-center mx-auto mb-3">
                <Lock size={24} />
              </div>
              <h3 className="text-lg font-bold text-slate-800">관리자 로그인</h3>
              <p className="text-xs text-slate-500 mt-1">이 채널의 비밀번호를 입력하세요.</p>
            </div>
            <form onSubmit={handleLoginSubmit}>
              <input 
                type="password" 
                maxLength="4"
                placeholder="비밀번호 4자리"
                className="w-full p-3 text-center text-xl tracking-[0.5em] font-bold border border-slate-200 rounded-xl focus:outline-none focus:ring-2 focus:ring-purple-400 mb-4"
                value={passwordInput}
                onChange={(e) => setPasswordInput(e.target.value)}
                autoFocus
              />
              <button type="submit" className="w-full py-3 bg-purple-600 text-white font-bold rounded-xl hover:bg-purple-700 transition-colors">로그인</button>
            </form>
          </div>
        </div>
      )}

      {showAdminMenu && (
        <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/50 p-4 animate-in fade-in">
          <div className="bg-white rounded-2xl p-6 w-full max-w-xs shadow-2xl scale-100 animate-in zoom-in-95 relative">
            <button onClick={() => setShowAdminMenu(false)} className="absolute right-4 top-4 text-slate-400 hover:text-slate-600"><X size={20}/></button>
            <div className="text-center mb-6">
              <div className="w-12 h-12 bg-green-100 text-green-600 rounded-full flex items-center justify-center mx-auto mb-3">
                <ShieldCheck size={24} />
              </div>
              <h3 className="text-lg font-bold text-slate-800">관리자 설정</h3>
            </div>
            <div className="space-y-4">
              <div className="bg-slate-50 p-4 rounded-xl border border-slate-100">
                <label className="text-xs font-bold text-slate-500 mb-2 block flex items-center gap-1"><Key size={12}/> 비밀번호 변경</label>
                <form onSubmit={handleChangePassword} className="flex gap-2">
                  <input 
                    type="password" 
                    maxLength="4"
                    placeholder="새 4자리"
                    className="flex-1 p-2 text-center font-bold border border-slate-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-400 text-sm"
                    value={newPasswordInput}
                    onChange={(e) => setNewPasswordInput(e.target.value)}
                  />
                  <button type="submit" className="bg-slate-800 text-white px-3 py-2 rounded-lg text-xs font-bold whitespace-nowrap">변경</button>
                </form>
              </div>
              <button onClick={() => { setIsAdmin(false); setShowAdminMenu(false); }} className="w-full py-3 bg-red-50 text-red-500 font-bold rounded-xl hover:bg-red-100 transition-colors flex items-center justify-center gap-2">
                <LogOut size={18} /> 로그아웃
              </button>
            </div>
          </div>
        </div>
      )}

      {deleteModal.isOpen && (
        <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/50 p-4 animate-in fade-in">
          <div className="bg-white rounded-2xl p-6 w-full max-w-sm shadow-2xl scale-100 animate-in zoom-in-95">
            <h3 className="text-lg font-bold text-slate-800 mb-2 flex items-center gap-2">
              <AlertTriangle className="text-red-500" size={20}/> 시청자 삭제
            </h3>
            <p className="text-slate-600 mb-6">
              정말로 <span className="font-bold text-purple-600">{deleteModal.viewerName}</span>님을 목록에서 삭제하시겠습니까?
            </p>
            <div className="flex gap-2 justify-end">
              <button onClick={() => setDeleteModal({ isOpen: false, viewerId: null, viewerName: null })} className="px-4 py-2 bg-slate-100 text-slate-600 rounded-xl font-medium hover:bg-slate-200">취소</button>
              <button onClick={confirmDeleteViewer} className="px-4 py-2 bg-red-500 text-white rounded-xl font-bold hover:bg-red-600 shadow-lg shadow-red-200">삭제하기</button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

export default App;
