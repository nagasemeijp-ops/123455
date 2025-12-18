import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  query, 
  doc, 
  deleteDoc,
  updateDoc
} from 'firebase/firestore';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, 
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  Plus, 
  Search, 
  Trash2, 
  ShoppingBag, 
  X,
  Coffee,
  Sparkles,
  Upload,
  Palette,
  Layers,
  ChevronRight,
  ChevronLeft,
  Heart,
  Truck,
  Hourglass,
  Store,
  Globe,
  Gift,
  PackageCheck,
  Edit3,
  Filter,
  Tag
} from 'lucide-react';

// --- Firebase 配置 ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'muji-wardrobe-v12';

const CATEGORIES = ["上身", "下身", "外套", "洋裝", "鞋履", "配件"];

const SUB_CATEGORIES = {
  "上身": ["短袖", "長袖", "背心", "毛衣", "帽T"],
  "下身": ["短褲", "長褲", "短裙", "長裙"],
  "配件": ["戒指", "手鍊", "項鍊", "耳環", "帽子", "圍巾", "包包"],
  "鞋履": ["涼鞋", "靴子", "跟鞋", "球鞋"],
  "外套": ["薄外套", "厚大衣", "羽絨", "西裝外套"],
  "洋裝": ["短洋裝", "長洋裝", "連身褲"]
};

const SEASONS = ["春日", "夏日", "秋日", "冬日", "全季節"];
const SOURCES = [
  { id: "store", label: "實體門市", icon: <Store size={14}/> },
  { id: "online", label: "網路商店", icon: <Globe size={14}/> },
  { id: "gift", label: "親友贈送", icon: <Gift size={14}/> }
];
const STATUS_OPTIONS = [
  { id: "received", label: "已到家 (現貨)", icon: <PackageCheck size={14}/> },
  { id: "shipping", label: "現貨配送中", icon: <Truck size={14}/> },
  { id: "preorder", label: "預購等待中", icon: <Hourglass size={14}/> }
];

const CURRENCIES = [
  { label: "台幣", code: "TWD", symbol: "NT$" },
  { label: "日幣", code: "JPY", symbol: "¥" },
  { label: "韓元", code: "KRW", symbol: "₩" },
  { label: "港幣", code: "HKD", symbol: "HK$" },
  { label: "美金", code: "USD", symbol: "US$" }
];

const PRESET_COLORS = [
  { name: "無/透明", value: "transparent" },
  { name: "墨黑", value: "#1A1A1A" },
  { name: "杏色", value: "#F5F5DC" },
  { name: "原木", value: "#D2B48C" },
  { name: "亞麻", value: "#E1D9D1" },
  { name: "炭灰", value: "#4A4A4A" },
  { name: "藏青", value: "#1B263B" },
  { name: "橄欖綠", value: "#556B2F" },
  { name: "酒紅", value: "#800000" },
  { name: "純白", value: "#FFFFFF" },
  { name: "霧藍", value: "#778899" },
  { name: "淺藍", value: "#ADD8E6" },
  { name: "焦糖", value: "#8B4513" }
];

const INITIAL_FORM = {
  name: '',
  purchaseDate: new Date().toISOString().split('T')[0],
  brand: '',
  originalPrice: '',
  originalCurrency: 'TWD',
  purchasePrice: '',
  purchaseCurrency: 'TWD',
  category: '上身',
  subCategory: '短袖',
  season: '全季節',
  images: [],
  notes: '',
  color1: '#F5F5DC',
  color2: 'transparent',
  color3: 'transparent',
  isFavorite: false,
  source: 'store',
  status: 'received'
};

export default function App() {
  const [user, setUser] = useState(null);
  const [items, setItems] = useState([]);
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [editingId, setEditingId] = useState(null);
  const [viewerItem, setViewerItem] = useState(null);
  const [activeImageIdx, setActiveImageIdx] = useState(0);
  
  // 篩選狀態
  const [filterCategory, setFilterCategory] = useState("全部");
  const [filterSubCategory, setFilterSubCategory] = useState("全部");
  const [filterSeason, setFilterSeason] = useState("全部");
  const [filterColor, setFilterColor] = useState("全部");
  const [searchTerm, setSearchTerm] = useState("");
  
  const fileInputRef = useRef(null);
  const [formData, setFormData] = useState(INITIAL_FORM);

  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    const q = collection(db, 'artifacts', appId, 'users', user.uid, 'clothes');
    const unsubscribe = onSnapshot(q, (snapshot) => {
      const docs = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setItems(docs);
    }, (error) => console.error("Firestore 錯誤:", error));
    return () => unsubscribe();
  }, [user]);

  // 當主類別改變，自動重設子類別為該類別的第一個選項
  useEffect(() => {
    if (SUB_CATEGORIES[formData.category]) {
        const availableSubs = SUB_CATEGORIES[formData.category];
        if (!availableSubs.includes(formData.subCategory)) {
            setFormData(prev => ({ ...prev, subCategory: availableSubs[0] }));
        }
    }
  }, [formData.category]);

  const handleFiles = (e) => {
    const files = Array.from(e.target.files);
    files.forEach(file => {
      const reader = new FileReader();
      reader.onloadend = () => {
        setFormData(prev => ({ ...prev, images: [...prev.images, reader.result] }));
      };
      reader.readAsDataURL(file);
    });
  };

  const moveImage = (index, direction) => {
    const newImages = [...formData.images];
    const newIndex = index + direction;
    if (newIndex < 0 || newIndex >= newImages.length) return;
    [newImages[index], newImages[newIndex]] = [newImages[newIndex], newImages[index]];
    setFormData({ ...formData, images: newImages });
  };

  const removeImage = (index) => {
    setFormData(prev => ({ ...prev, images: prev.images.filter((_, i) => i !== index) }));
  };

  const toggleFavorite = async (e, item) => {
    e.stopPropagation();
    if (!user) return;
    const itemRef = doc(db, 'artifacts', appId, 'users', user.uid, 'clothes', item.id);
    await updateDoc(itemRef, { isFavorite: !item.isFavorite });
  };

  const handleEdit = (e, item) => {
    e.stopPropagation();
    setEditingId(item.id);
    setFormData({ ...item });
    setIsModalOpen(true);
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!user) return;
    try {
      const payload = {
        ...formData,
        originalPrice: Number(formData.originalPrice) || 0,
        purchasePrice: Number(formData.purchasePrice) || 0,
        updatedAt: new Date().toISOString()
      };

      if (editingId) {
        await updateDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'clothes', editingId), payload);
      } else {
        await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'clothes'), {
          ...payload,
          createdAt: new Date().toISOString()
        });
      }
      
      setIsModalOpen(false);
      setEditingId(null);
      setFormData(INITIAL_FORM);
    } catch (err) { console.error("儲存錯誤:", err); }
  };

  const getCurrencySymbol = (code) => CURRENCIES.find(c => c.code === code)?.symbol || "$";

  const filteredItems = items.filter(item => {
    const matchCategory = filterCategory === "全部" || item.category === filterCategory;
    const matchSubCategory = filterSubCategory === "全部" || item.subCategory === filterSubCategory;
    const matchSeason = filterSeason === "全部" || item.season === filterSeason;
    const matchColor = filterColor === "全部" || 
      [item.color1, item.color2, item.color3].includes(filterColor);
    const searchLow = searchTerm.toLowerCase();
    const matchSearch = (item.name || "").toLowerCase().includes(searchLow) || 
                       (item.brand || "").toLowerCase().includes(searchLow);
    
    return matchCategory && matchSubCategory && matchSeason && matchColor && matchSearch;
  });

  const getActiveColors = (item) => {
    return [item.color1, item.color2, item.color3].filter(c => c && c !== 'transparent');
  };

  return (
    <div className="min-h-screen bg-[#FDFBF9] text-[#5D5550] pb-24 md:pb-10 p-4 md:p-10 font-sans selection:bg-[#E8DED3]">
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700&family=Quicksand:wght@400;600;700&display=swap');
        :root { font-family: 'Quicksand', 'Noto Sans TC', sans-serif; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .smart-img { width: 100%; height: 100%; object-fit: cover; object-position: center; }
        @keyframes heart-pulse { 0% { transform: scale(1); } 50% { transform: scale(1.1); } 100% { transform: scale(1); } }
        .favorite-pulse { animation: heart-pulse 1.5s infinite; }
        /* 適配手機底部安全區域 */
        .safe-area-inset-bottom { padding-bottom: env(safe-area-inset-bottom); }
      `}</style>

      <div className="max-w-7xl mx-auto">
        {/* 標題欄 */}
        <header className="flex flex-col md:flex-row justify-between items-center mb-10 gap-6 mt-4 md:mt-0">
          <div className="text-center md:text-left">
            <h1 className="text-3xl font-bold text-[#4A443F] flex items-center justify-center md:justify-start gap-3 tracking-tight">
              <Coffee className="text-[#A69080]" /> 暖心衣櫥紀錄
            </h1>
            <p className="text-[#A69080] text-[10px] tracking-[0.3em] mt-1 opacity-75 uppercase font-semibold">Minimalist Wardrobe Manager</p>
          </div>
          <button 
            onClick={() => { setEditingId(null); setFormData(INITIAL_FORM); setIsModalOpen(true); }}
            className="hidden md:flex items-center gap-2 bg-[#A69080] text-white px-8 py-3.5 rounded-full hover:bg-[#8E796B] transition-all shadow-md font-bold active:scale-95 text-sm"
          >
            <Plus size={18} /> 新增單品
          </button>
        </header>

        {/* 篩選工具列 */}
        <div className="bg-white rounded-[24px] p-4 md:p-6 mb-8 shadow-sm border border-[#F2EDE8] space-y-4 md:space-y-6">
          <div className="flex flex-col lg:flex-row gap-4 md:gap-6 items-center">
            {/* 關鍵字搜尋 */}
            <div className="flex-1 relative w-full">
              <Search className="absolute left-4 top-1/2 -translate-y-1/2 text-[#D1C7BD]" size={16} />
              <input 
                type="text" placeholder="搜尋名稱或品牌..." 
                className="w-full pl-11 pr-4 py-3 bg-[#F9F6F2] rounded-full outline-none text-sm focus:ring-2 focus:ring-[#A69080]/10"
                value={searchTerm} onChange={(e) => setSearchTerm(e.target.value)}
              />
            </div>
            
            {/* 主類別篩選 */}
            <div className="flex gap-2 overflow-x-auto no-scrollbar w-full lg:w-auto px-1 lg:px-0">
              <span className="text-[10px] font-bold text-[#A69080] uppercase flex items-center gap-1 shrink-0 mr-1"><Tag size={12}/> 類別:</span>
              {["全部", ...CATEGORIES].map(cat => (
                <button 
                  key={cat} onClick={() => { setFilterCategory(cat); setFilterSubCategory("全部"); }}
                  className={`px-5 py-2 rounded-full text-[11px] font-bold transition-all whitespace-nowrap border ${
                    filterCategory === cat ? 'bg-[#A69080] text-white border-[#A69080] shadow-sm' : 'bg-white text-[#A69080] border-[#E8DED3]'
                  }`}
                >{cat}</button>
              ))}
            </div>
          </div>

          {/* 子類別篩選 (連動) */}
          {filterCategory !== "全部" && (
            <div className="flex gap-2 overflow-x-auto no-scrollbar pt-2 border-t border-[#F9F6F2]">
              <span className="text-[10px] font-bold text-[#A69080] uppercase flex items-center gap-1 shrink-0 mr-1"><Layers size={12}/> 細分:</span>
              {["全部", ...(SUB_CATEGORIES[filterCategory] || [])].map(sub => (
                <button 
                  key={sub} onClick={() => setFilterSubCategory(sub)}
                  className={`px-4 py-1.5 rounded-full text-[10px] font-bold transition-all whitespace-nowrap border ${
                    filterSubCategory === sub ? 'bg-[#A69080]/20 text-[#A69080] border-[#A69080]' : 'bg-white text-[#D1C7BD] border-[#F2EDE8]'
                  }`}
                >{sub}</button>
              ))}
            </div>
          )}

          <div className="flex flex-col md:flex-row md:items-center gap-4 md:gap-6 pt-2 border-t border-[#F9F6F2]">
            {/* 季節篩選 */}
            <div className="flex gap-2 overflow-x-auto no-scrollbar flex-1 px-1">
              <span className="text-[10px] font-bold text-[#A69080] uppercase flex items-center gap-1 shrink-0 mr-1"><Filter size={12}/> 季節:</span>
              {["全部", ...SEASONS].map(s => (
                <button 
                  key={s} onClick={() => setFilterSeason(s)}
                  className={`px-5 py-2 rounded-full text-[11px] font-bold transition-all whitespace-nowrap border ${
                    filterSeason === s ? 'bg-[#8C7B6E] text-white border-[#8C7B6E] shadow-sm' : 'bg-white text-[#8C7B6E] border-[#E8DED3]'
                  }`}
                >{s}</button>
              ))}
            </div>

            {/* 色系篩選 */}
            <div className="flex gap-3 overflow-x-auto no-scrollbar md:w-auto py-1 px-1">
              <span className="text-[10px] font-bold text-[#A69080] uppercase flex items-center gap-1 shrink-0 mr-1"><Palette size={12}/> 色系:</span>
              <button 
                onClick={() => setFilterColor("全部")}
                className={`px-4 py-1.5 rounded-full text-[10px] font-bold transition-all border shrink-0 ${
                  filterColor === "全部" ? 'bg-[#A69080] text-white border-[#A69080]' : 'bg-white text-[#A69080] border-[#E8DED3]'
                }`}
              >全部</button>
              {PRESET_COLORS.filter(c => c.value !== 'transparent').map(color => (
                <button
                  key={color.value}
                  onClick={() => setFilterColor(color.value)}
                  className={`group relative w-7 h-7 rounded-full border-2 transition-all shrink-0 flex items-center justify-center ${
                    filterColor === color.value ? 'border-[#A69080] scale-110 shadow-md' : 'border-white shadow-sm hover:border-[#E8DED3]'
                  }`}
                  style={{ backgroundColor: color.value }}
                >
                  {filterColor === color.value && (
                    <div className={`w-1.5 h-1.5 rounded-full ${['#FFFFFF', '#F5F5DC', '#E1D9D1', '#ADD8E6'].includes(color.value) ? 'bg-[#A69080]' : 'bg-white'}`} />
                  )}
                </button>
              ))}
            </div>
          </div>
        </div>

        {/* 物品列表 */}
        <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 xl:grid-cols-5 gap-4 md:gap-6">
          {filteredItems.map(item => (
            <div key={item.id} className="group bg-white rounded-[24px] md:rounded-[28px] overflow-hidden shadow-sm hover:shadow-xl transition-all border border-[#F2EDE8] flex flex-col relative">
              <div className="aspect-[3/4] relative overflow-hidden bg-[#F9F6F2] cursor-zoom-in" onClick={() => { setViewerItem(item); setActiveImageIdx(0); }}>
                {item.images?.[0] ? (
                  <img src={item.images[0]} className="smart-img group-hover:scale-105 transition-transform duration-700" alt={item.name} />
                ) : (
                  <div className="w-full h-full flex flex-col items-center justify-center opacity-30 text-[#A69080]"><Sparkles size={32} /></div>
                )}
                
                {item.status !== 'received' && (
                  <div className="absolute top-2 left-2 md:top-3 md:left-3 flex gap-2 z-10">
                    <span className={`flex items-center gap-1 px-2 py-0.5 md:px-2.5 md:py-1 rounded-full text-[8px] md:text-[9px] font-bold text-white shadow-sm backdrop-blur-md ${
                      item.status === 'shipping' ? 'bg-[#D2B48C]/90' : 'bg-[#A69080]/80'
                    }`}>
                      {item.status === 'shipping' ? <Truck size={10}/> : <Hourglass size={10}/>}
                      {STATUS_OPTIONS.find(s => s.id === item.status)?.label.split(' ')[0]}
                    </span>
                  </div>
                )}

                <button 
                  onClick={(e) => toggleFavorite(e, item)}
                  className={`absolute top-2 right-2 md:top-3 md:right-3 p-1.5 md:p-2 rounded-full backdrop-blur-md transition-all z-10 ${
                    item.isFavorite ? 'bg-red-500 text-white' : 'bg-white/60 text-[#A69080]'
                  }`}
                >
                  <Heart size={14} fill={item.isFavorite ? "currentColor" : "none"} className={item.isFavorite ? "favorite-pulse" : ""} />
                </button>

                <div className="absolute bottom-3 left-3 flex flex-col gap-1.5">
                  <div className="flex -space-x-1.5">
                    {getActiveColors(item).map((c, i) => (
                      <div key={i} className="w-3.5 h-3.5 md:w-4 md:h-4 rounded-full border-2 border-white shadow-sm" style={{ backgroundColor: c }}></div>
                    ))}
                  </div>
                </div>
              </div>

              <div className="p-4 md:p-5 flex flex-col flex-1">
                <div className="flex justify-between items-start mb-1">
                  <div className="flex flex-col flex-1 pr-2">
                    <h3 className="font-bold text-[#4A443F] text-xs md:text-sm line-clamp-1">{item.name || '未命名'}</h3>
                    <p className="text-[9px] md:text-[10px] text-[#A69080] font-medium mt-0.5">{item.category} · {item.subCategory}</p>
                  </div>
                  <div className="flex gap-2 shrink-0">
                    <button onClick={(e) => handleEdit(e, item)} className="text-[#D1C7BD] hover:text-[#A69080] p-1"><Edit3 size={12}/></button>
                  </div>
                </div>
                
                <div className="flex flex-wrap items-center gap-1.5 mt-1">
                  <span className="text-[9px] md:text-[10px] font-bold text-[#A69080] bg-[#F9F6F2] px-1.5 py-0.5 rounded-md">{item.brand || 'Personal'}</span>
                </div>
                
                <div className="mt-auto pt-3 md:pt-4 border-t border-[#F9F6F2] flex justify-between items-end">
                  <div className="flex flex-col">
                    <span className="text-[8px] md:text-[9px] text-[#D1C7BD] line-through font-medium leading-none mb-0.5">
                      {getCurrencySymbol(item.originalCurrency)} {item.originalPrice?.toLocaleString()}
                    </span>
                    <span className="text-xs md:text-sm font-bold text-[#8C7B6E] leading-none">
                      {getCurrencySymbol(item.purchaseCurrency)} {item.purchasePrice?.toLocaleString()}
                    </span>
                  </div>
                  <span className="text-[8px] md:text-[9px] bg-[#FDFBF9] text-[#A69080] px-2 py-0.5 md:py-1 rounded-full font-bold border border-[#F2EDE8]">
                    {item.season}
                  </span>
                </div>
              </div>
            </div>
          ))}
        </div>

        {/* 手機版底部懸浮按鈕 */}
        <button 
          onClick={() => { setEditingId(null); setFormData(INITIAL_FORM); setIsModalOpen(true); }}
          className="md:hidden fixed bottom-8 right-6 w-14 h-14 bg-[#A69080] text-white rounded-full flex items-center justify-center shadow-2xl z-40 active:scale-90 transition-transform"
        >
          <Plus size={28} />
        </button>

        {/* 圖片檢視器 */}
        {viewerItem && (
          <div className="fixed inset-0 z-[100] bg-black/95 backdrop-blur-xl flex flex-col animate-in fade-in duration-300">
            <div className="flex justify-between items-center p-6 text-white safe-area-inset-bottom">
              <div className="flex flex-col">
                <h2 className="text-xl font-bold tracking-tight">{viewerItem.name}</h2>
                <p className="text-xs opacity-60 font-medium">
                  {viewerItem.brand} · {viewerItem.category} ({viewerItem.subCategory})
                </p>
              </div>
              <button onClick={() => setViewerItem(null)} className="p-3 bg-white/10 rounded-full"><X/></button>
            </div>
            
            <div className="flex-1 relative flex items-center justify-center p-4">
              <div className="relative max-w-4xl w-full h-full flex items-center justify-center">
                <img src={viewerItem.images[activeImageIdx]} className="max-w-full max-h-full object-contain rounded-lg shadow-2xl" alt="View" />
                {viewerItem.images.length > 1 && (
                  <>
                    <button onClick={() => setActiveImageIdx(prev => (prev === 0 ? viewerItem.images.length - 1 : prev - 1))} className="absolute left-0 md:left-4 p-4 text-white/50"><ChevronLeft size={32}/></button>
                    <button onClick={() => setActiveImageIdx(prev => (prev === viewerItem.images.length - 1 ? 0 : prev + 1))} className="absolute right-0 md:right-4 p-4 text-white/50"><ChevronRight size={32}/></button>
                  </>
                )}
              </div>
            </div>

            <div className="p-6 md:p-8 flex flex-col items-center">
              <div className="flex gap-2.5 mb-6">
                {viewerItem.images?.map((_, idx) => (
                  <button key={idx} onClick={() => setActiveImageIdx(idx)} className={`h-1.5 rounded-full transition-all ${activeImageIdx === idx ? 'bg-white w-8' : 'bg-white/20 w-3'}`} />
                ))}
              </div>
              <div className="max-w-lg w-full bg-white/10 rounded-[28px] md:rounded-[32px] p-5 md:p-6 text-white text-center">
                <p className="text-xs md:text-sm italic font-medium">"{viewerItem.notes || '暫無穿搭隨手記'}"</p>
              </div>
            </div>
          </div>
        )}

        {/* 表單彈窗 */}
        {isModalOpen && (
          <div className="fixed inset-0 z-50 flex items-center justify-center p-0 md:p-4 bg-[#4A443F]/60 backdrop-blur-md overflow-y-auto">
            <div className="bg-white w-full max-w-2xl h-full md:h-auto md:max-h-[90vh] md:rounded-[40px] shadow-2xl flex flex-col animate-in slide-in-from-bottom md:zoom-in-95 duration-300">
              <div className="flex justify-between items-center p-6 md:p-8 border-b border-[#F9F6F2] mt-8 md:mt-0">
                <h2 className="text-xl md:text-2xl font-bold text-[#4A443F] flex items-center gap-3">
                  <ShoppingBag className="text-[#A69080]"/> {editingId ? '編輯單品' : '紀錄新單品'}
                </h2>
                <button onClick={() => setIsModalOpen(false)} className="text-[#A69080] p-2 bg-[#F9F6F2] rounded-full"><X size={20}/></button>
              </div>
              
              <form onSubmit={handleSubmit} className="flex-1 p-6 md:p-8 space-y-8 overflow-y-auto no-scrollbar pb-24 md:pb-8">
                {/* 圖片上傳 */}
                <div>
                  <label className="text-[10px] font-bold text-[#A69080] mb-4 block uppercase tracking-[0.2em]">圖片 排序封面 ✨</label>
                  <div className="flex flex-wrap gap-3 md:gap-4">
                    {formData.images.map((img, idx) => (
                      <div key={idx} className="flex flex-col gap-2">
                        <div className="w-20 h-20 md:w-24 md:h-24 rounded-xl md:rounded-2xl overflow-hidden relative group border-2 border-[#F9F6F2]">
                          <img src={img} className="smart-img" alt="upload" />
                          <button type="button" onClick={() => removeImage(idx)} className="absolute top-1 right-1 bg-red-400 p-1 rounded-full text-white"><X size={10}/></button>
                          {idx === 0 && <div className="absolute bottom-0 inset-x-0 bg-[#A69080]/80 text-white text-[8px] font-bold text-center py-0.5 uppercase">Cover</div>}
                        </div>
                        <div className="flex justify-between px-0.5">
                          <button type="button" disabled={idx === 0} onClick={() => moveImage(idx, -1)} className={`p-1 rounded bg-[#F9F6F2] text-[#A69080] ${idx === 0 ? 'opacity-10' : ''}`}><ChevronLeft size={12}/></button>
                          <button type="button" disabled={idx === formData.images.length - 1} onClick={() => moveImage(idx, 1)} className={`p-1 rounded bg-[#F9F6F2] text-[#A69080] ${idx === formData.images.length - 1 ? 'opacity-10' : ''}`}><ChevronRight size={12}/></button>
                        </div>
                      </div>
                    ))}
                    <button type="button" onClick={() => fileInputRef.current.click()} className="w-20 h-20 md:w-24 md:h-24 rounded-xl md:rounded-2xl border-2 border-dashed border-[#E8DED3] flex flex-col items-center justify-center text-[#A69080] hover:bg-[#F9F6F2]">
                      <Upload size={20}/><span className="text-[9px] mt-1 font-bold uppercase">Add</span>
                    </button>
                    <input type="file" multiple ref={fileInputRef} className="hidden" onChange={handleFiles} accept="image/*" />
                  </div>
                </div>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-x-8 gap-y-6">
                  {/* 愛用標記 */}
                  <div className="md:col-span-2">
                    <button type="button" onClick={() => setFormData({...formData, isFavorite: !formData.isFavorite})} className={`w-full py-4 rounded-2xl border-2 flex items-center justify-center gap-2 font-bold transition-all ${formData.isFavorite ? 'border-red-400 bg-red-50 text-red-500' : 'border-[#F2EDE8] text-[#D1C7BD]'}`}>
                      <Heart size={18} fill={formData.isFavorite ? "currentColor" : "none"} />
                      {formData.isFavorite ? "愛用中" : "標記為愛用單品？"}
                    </button>
                  </div>

                  {/* 獲取來源 */}
                  <div className="md:col-span-2 space-y-3">
                    <label className="text-[10px] font-bold text-[#A69080] uppercase tracking-widest">獲取來源</label>
                    <div className="grid grid-cols-3 gap-2 md:gap-3">
                      {SOURCES.map(s => (
                        <button key={s.id} type="button" onClick={() => setFormData({...formData, source: s.id})} className={`flex flex-col items-center gap-1.5 p-3 rounded-2xl border-2 transition-all ${formData.source === s.id ? 'border-[#A69080] bg-[#A69080]/5 text-[#A69080]' : 'border-[#F9F6F2] text-[#D1C7BD] opacity-60'}`}>
                          {s.icon} <span className="text-[10px] font-bold">{s.label}</span>
                        </button>
                      ))}
                    </div>
                  </div>

                  <div className="md:col-span-2 space-y-2">
                    <label className="text-[10px] font-bold text-[#A69080] uppercase tracking-widest">名稱</label>
                    <input type="text" required className="w-full p-4 bg-[#F9F6F2] rounded-2xl outline-none text-sm" value={formData.name} onChange={e => setFormData({...formData, name: e.target.value})} />
                  </div>

                  {/* 類別連動 */}
                  <div className="md:col-span-2 grid grid-cols-2 gap-4">
                    <div className="space-y-2">
                      <label className="text-[10px] font-bold text-[#A69080] uppercase tracking-widest">主類別</label>
                      <select className="w-full p-4 bg-[#F9F6F2] rounded-2xl outline-none text-sm font-medium" value={formData.category} onChange={e => setFormData({...formData, category: e.target.value})}>
                        {CATEGORIES.map(c => <option key={c} value={c}>{c}</option>)}
                      </select>
                    </div>
                    <div className="space-y-2">
                      <label className="text-[10px] font-bold text-[#A69080] uppercase tracking-widest">細分種類</label>
                      <select className="w-full p-4 bg-[#F9F6F2] rounded-2xl outline-none text-sm font-medium" value={formData.subCategory} onChange={e => setFormData({...formData, subCategory: e.target.value})}>
                        {(SUB_CATEGORIES[formData.category] || []).map(sub => (
                          <option key={sub} value={sub}>{sub}</option>
                        ))}
                      </select>
                    </div>
                  </div>

                  <div className="space-y-2">
                    <label className="text-[10px] font-bold text-[#A69080] uppercase tracking-widest">品牌</label>
                    <input type="text" className="w-full p-4 bg-[#F9F6F2] rounded-2xl outline-none text-sm" value={formData.brand} onChange={e => setFormData({...formData, brand: e.target.value})} />
                  </div>

                  <div className="space-y-2">
                    <label className="text-[10px] font-bold text-[#A69080] uppercase tracking-widest">推薦季節</label>
                    <select className="w-full p-4 bg-[#F9F6F2] rounded-2xl outline-none text-sm font-medium" value={formData.season} onChange={e => setFormData({...formData, season: e.target.value})}>
                      {SEASONS.map(s => <option key={s} value={s}>{s}</option>)}
                    </select>
                  </div>

                  <div className="md:col-span-2 space-y-2 p-4 bg-[#F9F6F2]/50 rounded-[24px] border border-[#F2EDE8]">
                    <label className="text-[10px] font-bold text-[#8C7B6E] uppercase tracking-widest">購入實付</label>
                    <div className="flex gap-2 mt-1">
                      <select className="flex-[0.4] bg-white p-3 rounded-xl outline-none text-[10px] font-bold border border-[#E8DED3]" value={formData.purchaseCurrency} onChange={e => setFormData({...formData, purchaseCurrency: e.target.value})}>
                        {CURRENCIES.map(c => <option key={c.code} value={c.code}>{c.code}</option>)}
                      </select>
                      <input type="number" className="flex-1 bg-white p-3 rounded-xl outline-none text-sm font-bold border border-[#E8DED3]" value={formData.purchasePrice} onChange={e => setFormData({...formData, purchasePrice: e.target.value})} />
                    </div>
                  </div>

                  <div className="md:col-span-2 space-y-4">
                    <label className="text-[10px] font-bold text-[#A69080] uppercase tracking-widest flex items-center gap-2"><Palette size={14}/> 色彩組合</label>
                    <div className="grid grid-cols-3 gap-3">
                      {[1, 2, 3].map(num => (
                        <div key={num} className="bg-[#FDFBF9] p-2 rounded-xl border border-[#F2EDE8] flex flex-col gap-1.5">
                          <input type="color" className="w-full h-8 rounded-lg cursor-pointer" value={formData[`color${num}`] === 'transparent' ? '#ffffff' : formData[`color${num}`]} onChange={(e) => setFormData({...formData, [`color${num}`]: e.target.value})} />
                          <select className="w-full bg-transparent text-[8px] outline-none font-bold text-[#A69080]" onChange={(e) => setFormData({...formData, [`color${num}`]: e.target.value})} value={formData[`color${num}`]}>
                            {PRESET_COLORS.map(c => <option key={c.value} value={c.value}>{c.name}</option>)}
                          </select>
                        </div>
                      ))}
                    </div>
                  </div>
                </div>

                <div className="space-y-2">
                  <label className="text-[10px] font-bold text-[#A69080] uppercase tracking-widest">隨手記</label>
                  <textarea rows="3" className="w-full p-5 bg-[#F9F6F2] rounded-[24px] outline-none text-sm leading-relaxed" value={formData.notes} onChange={e => setFormData({...formData, notes: e.target.value})} />
                </div>

                <div className="flex gap-4">
                    {editingId && (
                        <button type="button" onClick={() => { if(confirm('確定刪除？')) deleteDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'clothes', editingId)); setIsModalOpen(false); }} className="flex-1 py-5 bg-red-50 text-red-500 rounded-[24px] font-bold">刪除單品</button>
                    )}
                    <button type="submit" className="flex-[2] py-5 bg-[#A69080] text-white rounded-[24px] font-bold shadow-lg shadow-[#A69080]/20 flex items-center justify-center gap-2">
                    {editingId ? '儲存更改' : '加入衣櫥'} <ChevronRight size={18}/>
                    </button>
                </div>
              </form>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
