import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  doc, 
  onSnapshot, 
  addDoc, 
  deleteDoc,
  serverTimestamp 
} from 'firebase/firestore';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, 
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  Calendar as CalendarIcon, 
  Utensils, 
  MapPin, 
  Camera, 
  Plus, 
  X, 
  ChevronLeft, 
  ChevronRight, 
  DollarSign, 
  Clock,
  Pin,
  Heart,
  Paperclip,
  Trash2,
  Link as LinkIcon,
  ExternalLink
} from 'lucide-react';

// --- Firebase Configuration ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'core-4-hub';

// --- Constants ---
const MONTHS = [
  "January", "February", "March", "April", "May", "June",
  "July", "August", "September", "October", "November", "December"
];
const DAYS = ["S", "M", "T", "W", "T", "F", "S"];

// Cleaned up icons, vintage color mappings
const CORE_4 = [
  { name: "Rejea", icon: "☕", color: "text-amber-900", bg: "bg-amber-100", border: "border-amber-200" },
  { name: "Maevi", icon: "🍵", color: "text-green-900", bg: "bg-green-100", border: "border-green-200" },
  { name: "Shahanna", icon: "🐱", color: "text-stone-800", bg: "bg-stone-200", border: "border-stone-300" },
  { name: "Ann", icon: "💄", color: "text-red-900", bg: "bg-red-100", border: "border-red-200" }
];

const App = () => {
  const [user, setUser] = useState(null);
  const [activeTab, setActiveTab] = useState('hub'); 
  const [currentMonth, setCurrentMonth] = useState(0); 
  const [events, setEvents] = useState([]);
  const [wishlist, setWishlist] = useState([]);
  const [meetings, setMeetings] = useState([]);
  const [invites, setInvites] = useState([]);
  const [links, setLinks] = useState([]); 
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [isInviteModalOpen, setIsInviteModalOpen] = useState(false);
  const [isLinkModalOpen, setIsLinkModalOpen] = useState(false); 
  const [selectedDay, setSelectedDay] = useState(null); 
  const [selectedEvent, setSelectedEvent] = useState(null);

  // --- Auth & Data Fetching ---
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) { console.error("Auth error", err); }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;

    const unsubEvents = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'events'), 
      (s) => setEvents(s.docs.map(d => ({ id: d.id, ...d.data() }))), (e) => console.error(e));

    const unsubWish = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'wishlist'), 
      (s) => setWishlist(s.docs.map(d => ({ id: d.id, ...d.data() }))), (e) => console.error(e));

    const unsubMeet = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'meetings'), 
      (s) => setMeetings(s.docs.map(d => ({ id: d.id, ...d.data() })).sort((a,b) => b.timestamp?.seconds - a.timestamp?.seconds)), (e) => console.error(e));

    const unsubInvites = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'invites'), 
      (s) => setInvites(s.docs.map(d => ({ id: d.id, ...d.data() }))), (e) => console.error(e));

    const unsubLinks = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'links'), 
      (s) => setLinks(s.docs.map(d => ({ id: d.id, ...d.data() }))), (e) => console.error(e));

    return () => { unsubEvents(); unsubWish(); unsubMeet(); unsubInvites(); unsubLinks(); };
  }, [user]);

  // --- Helpers ---
  const getDaysInMonth = (m) => new Date(2026, m + 1, 0).getDate();
  const getFirstDayOfMonth = (m) => new Date(2026, m, 1).getDay();

  const handleAddEvent = async (e) => {
    e.preventDefault();
    const fd = new FormData(e.target);
    const mName = fd.get('member');
    const mData = CORE_4.find(m => m.name === mName);
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'events'), {
      title: fd.get('title'),
      description: fd.get('description'),
      date: selectedDay,
      month: currentMonth,
      year: 2026,
      expenses: fd.get('expenses') || "0",
      createdBy: mName,
      creatorIcon: mData?.icon || "✨",
      timestamp: serverTimestamp(),
    });
    setIsModalOpen(false);
  };

  const handleAddInvite = async (e) => {
    e.preventDefault();
    const fd = new FormData(e.target);
    const mName = fd.get('member');
    const mData = CORE_4.find(m => m.name === mName);
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'invites'), {
      title: fd.get('title'),
      date: fd.get('date'),
      time: fd.get('time'),
      location: fd.get('location'),
      author: mName,
      authorIcon: mData?.icon || "✨",
      timestamp: serverTimestamp()
    });
    setIsInviteModalOpen(false);
  };

  const handleAddLink = async (e) => {
    e.preventDefault();
    const fd = new FormData(e.target);
    const mName = fd.get('member');
    const mData = CORE_4.find(m => m.name === mName);
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'links'), {
      title: fd.get('title'),
      url: fd.get('url'),
      author: mName,
      authorIcon: mData?.icon || "✨",
      timestamp: serverTimestamp()
    });
    setIsLinkModalOpen(false);
  };

  const handleAddWish = async (type) => {
    const title = prompt(`Enter ${type} name:`);
    if (!title) return;
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'wishlist'), { title, type, timestamp: serverTimestamp() });
  };

  const CorkBoardBackground = () => (
    <div className="fixed inset-0 pointer-events-none z-0 bg-[#dcbfa6]">
       <div className="absolute inset-0 opacity-50" style={{ 
        backgroundImage: `url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noiseFilter'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.8' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noiseFilter)' opacity='0.4'/%3E%3C/svg%3E")` 
      }}></div>
    </div>
  );

  const WashiTape = ({ color = "bg-[#D6Ceb2]", className = "" }) => (
    <div className={`absolute -top-3 left-1/2 -translate-x-1/2 w-20 h-8 z-20 ${color} opacity-90 backdrop-blur-sm transform -rotate-1 shadow-sm ${className}`} 
         style={{ maskImage: "linear-gradient(45deg, transparent 5px, black 5px), linear-gradient(-45deg, transparent 5px, black 5px)", WebkitMaskImage: "linear-gradient(90deg, transparent 2%, black 5%, black 95%, transparent 98%)" }} />
  );

  return (
    <div className="min-h-screen text-[#3e342a] font-serif selection:bg-stone-300 pb-20">
      <CorkBoardBackground />

      <div className="relative z-10 max-w-6xl mx-auto px-6 pt-10">
        
        {/* Header - Stamped Look */}
        <header className="flex flex-col items-center mb-12 space-y-6">
           <div className="flex gap-4 p-4 bg-[#f9f7f1] rounded-sm border border-[#dcd6c8] shadow-lg transform rotate-1">
            {CORE_4.map(m => (
              <div key={m.name} className="flex flex-col items-center group cursor-default">
                <div className={`w-12 h-12 rounded-full ${m.bg} ${m.border} border-2 flex items-center justify-center text-xl shadow-inner transition-transform group-hover:-translate-y-1`}>
                  {m.icon}
                </div>
                <span className="text-[10px] mt-2 font-mono uppercase tracking-widest text-stone-700 font-bold bg-white/50 px-1 rounded">{m.name}</span>
              </div>
            ))}
          </div>

          <div className="text-center bg-[#fdfbf7]/80 p-6 rounded-lg shadow-xl border border-white/20 backdrop-blur-sm transform -rotate-1">
            <h1 className="text-6xl font-black text-[#2c241b] tracking-tighter opacity-90" style={{ fontFamily: 'Georgia, serif' }}>
              Core 4 Hub
            </h1>
            <div className="flex items-center justify-center gap-2 mt-2">
              <span className="h-px w-8 bg-stone-400"></span>
              <p className="text-xs font-mono tracking-[0.3em] text-stone-600 uppercase font-bold">Est. 2026</p>
              <span className="h-px w-8 bg-stone-400"></span>
            </div>
          </div>
        </header>

        {/* Tab Switcher - Paper Labels pinned to cork */}
        <nav className="flex max-w-md mx-auto mb-10 pl-4 gap-4 relative z-20 justify-center">
          {['hub', 'wishlist', 'meetings'].map(t => (
            <button key={t} onClick={() => setActiveTab(t)}
              className={`px-6 py-2 text-xs font-bold uppercase tracking-widest transition-all shadow-md relative transform hover:-translate-y-1
                ${activeTab === t 
                  ? 'bg-[#fdfbf7] text-stone-900 rotate-1 scale-105' 
                  : 'bg-[#e6dac5] text-stone-600 -rotate-1 hover:bg-[#ede3d1]'}`}>
              <div className="absolute -top-2 left-1/2 -translate-x-1/2 text-red-500 drop-shadow-sm"><Pin size={12} fill="currentColor" /></div>
              {t}
            </button>
          ))}
        </nav>

        <main className="animate-in fade-in duration-700">
          {activeTab === 'hub' && (
            <div className="grid grid-cols-1 lg:grid-cols-12 gap-10 items-start">
              
              {/* LEFT COLUMN: Calendar & Notices */}
              <div className="lg:col-span-7 space-y-12">
                
                {/* Calendar - Classic Paper Calendar pinned to cork */}
                <div className="bg-[#fdfbf7] rounded-sm p-8 shadow-2xl transform -rotate-1 relative border border-[#e8e4dc]">
                  <div className="absolute -top-3 left-1/2 -translate-x-1/2 text-stone-400 drop-shadow-md"><Pin size={24} fill="#555" /></div>
                  
                  <div className="flex items-center justify-between mb-8 border-b-2 border-double border-stone-200 pb-4">
                    <h2 className="text-2xl font-bold text-stone-800 flex items-center gap-3" style={{ fontFamily: 'Georgia, serif' }}>
                      <CalendarIcon size={22} className="text-stone-400" /> {MONTHS[currentMonth]}
                    </h2>
                    <div className="flex gap-2">
                      <button onClick={() => setCurrentMonth(m => Math.max(0, m - 1))} className="p-2 hover:bg-stone-100 rounded-full text-stone-500"><ChevronLeft size={18}/></button>
                      <button onClick={() => setCurrentMonth(m => Math.min(11, m + 1))} className="p-2 hover:bg-stone-100 rounded-full text-stone-500"><ChevronRight size={18}/></button>
                    </div>
                  </div>
                  
                  {/* Calendar Grid */}
                  <div className="grid grid-cols-7 gap-3">
                    {DAYS.map((d, i) => <div key={`dh-${i}`} className="text-center text-[10px] font-mono font-bold text-stone-400 mb-2">{d}</div>)}
                    {Array(getFirstDayOfMonth(currentMonth)).fill(0).map((_, i) => <div key={`be-${i}`} />)}
                    {[...Array(getDaysInMonth(currentMonth))].map((_, i) => {
                      const d = i + 1;
                      const dEv = events.filter(e => e.month === currentMonth && parseInt(e.date) === d);
                      const hasEvent = dEv.length > 0;
                      const isSelected = selectedDay === d;
                      return (
                        <button key={`dc-${d}`} onClick={() => setSelectedDay(d)}
                          className={`aspect-square border flex flex-col items-center justify-center relative hover:bg-[#f0eadd] transition-all text-sm font-serif
                            ${isSelected ? 'bg-yellow-100 border-yellow-300 shadow-inner scale-95' : 
                              hasEvent ? 'bg-[#f4efe6] border-stone-300 text-stone-900' : 'bg-transparent border-transparent text-stone-400'}`}>
                          {d}
                          {hasEvent && (
                            <div className="absolute -bottom-1 -right-1 flex -space-x-1">
                              {dEv.map((ev, idx) => <span key={`ei-${idx}`} className="text-[10px] bg-white rounded-full border border-stone-200 w-4 h-4 flex items-center justify-center shadow-sm z-10">{ev.creatorIcon}</span>)}
                            </div>
                          )}
                        </button>
                      );
                    })}
                  </div>
                </div>

                {/* Notices - Green Felt Board */}
                <div className="bg-[#1e3a29] rounded-lg p-6 shadow-xl relative border-[12px] border-[#5c4033] outline outline-1 outline-[#3e2b22] transform rotate-1">
                   {/* Texture for felt */}
                  <div className="absolute inset-0 opacity-30 pointer-events-none" style={{ 
                      backgroundImage: `url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noise'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='1.2' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noise)' opacity='0.2'/%3E%3C/svg%3E")`
                  }}></div>
                  
                  <div className="flex justify-between items-center mb-6 relative z-10">
                    <h3 className="text-sm font-bold uppercase tracking-widest text-[#e8f5e9] bg-white/10 px-3 py-1 rounded-sm shadow-sm flex items-center gap-2 backdrop-blur-sm border border-white/10">
                      <Pin size={14} /> Group Notices
                    </h3>
                    <button onClick={() => setIsInviteModalOpen(true)} className="text-[10px] font-bold bg-[#f9f7f1] text-stone-800 px-3 py-1.5 rounded-sm hover:shadow-md transition-all border border-[#bdae93] shadow-lg">
                      POST INVITE
                    </button>
                  </div>
                  
                  <div className="grid grid-cols-1 md:grid-cols-2 gap-6 relative z-10">
                    {invites.length === 0 ? (
                      <div className="md:col-span-2 text-center py-10 text-[#81c784] text-xs font-mono italic opacity-70">
                        The board is empty. Pin something here.
                      </div>
                    ) : (
                      invites.map((inv, idx) => (
                        <div key={inv.id} className={`bg-[#fdfbf7] p-5 shadow-[3px_3px_5px_rgba(0,0,0,0.3)] relative transform ${idx % 2 === 0 ? 'rotate-1' : '-rotate-1'}`}>
                          <div className="absolute -top-3 left-1/2 -translate-x-1/2 text-red-700 drop-shadow-md"><Pin size={20} fill="currentColor" /></div>
                          <button onClick={() => deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'invites', inv.id))}
                            className="absolute top-2 right-2 text-stone-300 hover:text-red-400 transition-colors"><X size={12}/></button>
                          <div className="flex items-center gap-2 mb-3 border-b border-stone-200 pb-2">
                            <span className="text-xl">{inv.authorIcon}</span>
                            <span className="text-[9px] font-mono text-stone-400 uppercase tracking-widest">{inv.author} invites you</span>
                          </div>
                          <h4 className="font-serif font-bold text-lg mb-2 text-stone-800 leading-tight">{inv.title}</h4>
                          <div className="space-y-1 text-[11px] font-mono text-stone-600">
                            <div className="flex items-center gap-2"><Clock size={12}/> {inv.date} @ {inv.time}</div>
                            <div className="flex items-center gap-2"><MapPin size={12}/> {inv.location}</div>
                          </div>
                        </div>
                      ))
                    )}
                  </div>
                </div>
              </div>

              {/* RIGHT COLUMN: Sticky Notes */}
              <div className="lg:col-span-5 flex flex-col gap-10">
                
                {/* 1. Day Details Post-it (Yellow) */}
                <div className="bg-[#fff8b5] rounded-sm p-8 shadow-xl relative transform rotate-1 flex flex-col min-h-[400px]">
                  {/* Pin holding the post-it */}
                  <div className="absolute -top-3 left-1/2 -translate-x-1/2 z-20 text-red-600 drop-shadow-md">
                     <Pin size={32} fill="currentColor" />
                  </div>
                  
                  {/* Sticky note gradient */}
                  <div className="absolute inset-x-0 top-0 h-16 bg-gradient-to-b from-white/20 to-transparent pointer-events-none"></div>

                  <h3 className="text-lg font-bold uppercase tracking-widest text-stone-600 mb-6 text-center font-mono border-b-2 border-stone-300/30 pb-4">
                    Day Details
                  </h3>

                  <div className="flex-1 flex flex-col">
                    {!selectedDay ? (
                      <div className="flex-1 flex flex-col items-center justify-center text-stone-400 text-center opacity-70">
                         <CalendarIcon size={48} className="mb-4 text-stone-300" />
                         <p className="font-serif italic">Select a day on the calendar<br/>to see the plans.</p>
                      </div>
                    ) : (
                      <div className="animate-in fade-in slide-in-from-left-4 duration-300">
                         <div className="flex justify-between items-end mb-6">
                           <h2 className="text-4xl font-bold font-serif text-stone-800 leading-none">
                             {MONTHS[currentMonth]} {selectedDay}
                           </h2>
                           <span className="text-xs font-mono text-stone-500">2026</span>
                         </div>

                         <div className="space-y-4 mb-8">
                           {events.filter(e => e.month === currentMonth && parseInt(e.date) === selectedDay).length === 0 ? (
                             <p className="text-stone-500 font-serif italic border-l-2 border-stone-300 pl-4">
                               No plans made for this day yet.
                             </p>
                           ) : (
                             events.filter(e => e.month === currentMonth && parseInt(e.date) === selectedDay).map(event => (
                               <div key={event.id} className="bg-white/60 p-4 rounded-sm border border-stone-200 relative group hover:bg-white/80 transition-colors">
                                 <button 
                                   onClick={() => deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'events', event.id))}
                                   className="absolute top-2 right-2 text-stone-400 hover:text-red-500 opacity-0 group-hover:opacity-100 transition-opacity"
                                 >
                                   <Trash2 size={14} />
                                 </button>
                                 <div className="flex items-center gap-3 mb-2">
                                   <span className="text-2xl">{event.creatorIcon}</span>
                                   <div className="leading-tight">
                                     <h4 className="font-bold text-stone-800 text-sm">{event.title}</h4>
                                     <span className="text-[10px] uppercase font-mono text-stone-500">Planned by {event.createdBy}</span>
                                   </div>
                                 </div>
                                 {event.description && (
                                   <p className="text-xs font-serif text-stone-600 mb-2 italic">"{event.description}"</p>
                                 )}
                                 <div className="text-right">
                                   <span className="text-xs font-mono font-bold bg-green-100 text-green-800 px-2 py-1 rounded-sm border border-green-200">
                                     ${event.expenses}
                                   </span>
                                 </div>
                               </div>
                             ))
                           )}
                         </div>

                         <div className="text-center mt-auto pt-4 border-t border-stone-300/30">
                           <button 
                             onClick={() => setIsModalOpen(true)}
                             className="flex items-center justify-center gap-2 mx-auto text-stone-800 font-bold uppercase tracking-widest text-xs border-2 border-stone-800 px-6 py-3 hover:bg-stone-800 hover:text-[#fff8b5] transition-colors"
                           >
                             <Plus size={16} /> Add Plan
                           </button>
                         </div>
                      </div>
                    )}
                  </div>
                </div>

                {/* 2. Resources Post-it (Pink) */}
                <div className="bg-[#ffdad9] rounded-sm p-8 shadow-xl relative transform -rotate-1 min-h-[300px]">
                   {/* Tape holding the post-it */}
                  <div className="absolute -top-3 left-1/2 -translate-x-1/2 w-20 h-8 bg-[#e8e8e8]/50 backdrop-blur-sm transform rotate-1 shadow-sm" />
                  
                  <div className="flex justify-between items-center mb-6 pb-2 border-b-2 border-red-900/10">
                    <h3 className="text-lg font-bold uppercase tracking-widest text-stone-700 font-mono flex items-center gap-2">
                      <LinkIcon size={16} /> Resources
                    </h3>
                    <button onClick={() => setIsLinkModalOpen(true)} className="p-1 hover:bg-white/20 rounded-full">
                      <Plus size={18} className="text-stone-600" />
                    </button>
                  </div>

                  <div className="space-y-3">
                    {links.length === 0 ? (
                      <div className="text-center py-8 text-stone-500 text-xs font-mono italic">
                         Pin budget sheets or travel links here.
                      </div>
                    ) : (
                      links.map((link) => (
                        <div key={link.id} className="bg-white/80 p-3 shadow-sm relative transform hover:scale-[1.02] transition-transform">
                          <button onClick={() => deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'links', link.id))}
                            className="absolute top-1 right-1 text-stone-300 hover:text-red-500 transition-colors"><X size={12}/></button>
                          
                          <div className="flex items-start gap-3">
                            <div className="mt-1 text-stone-600"><Paperclip size={14} /></div>
                            <div className="flex-1 min-w-0">
                              <h4 className="font-bold text-stone-800 text-sm leading-tight">{link.title}</h4>
                              <a href={link.url} target="_blank" rel="noopener noreferrer" className="text-[10px] text-blue-600 hover:underline flex items-center gap-1 mt-1 truncate w-full block">
                                Open Link <ExternalLink size={8} />
                              </a>
                              <div className="text-[8px] text-stone-400 mt-1 font-mono">
                                Posted by {link.author}
                              </div>
                            </div>
                          </div>
                        </div>
                      ))
                    )}
                  </div>
                </div>

              </div>
            </div>
          )}

          {/* WISHLIST TAB */}
          {activeTab === 'wishlist' && (
            <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
              {[
                { title: 'To Do', type: 'activity', bg: "bg-[#fcf4dd]", tape: "bg-[#e0c986]" },
                { title: 'To Eat', type: 'restaurant', bg: "bg-[#e8f2e4]", tape: "bg-[#aabc9b]" },
                { title: 'To Go', type: 'place', bg: "bg-[#e6eff2]", tape: "bg-[#9dbcc4]" }
              ].map((sec, idx) => (
                <div key={sec.type} className={`p-8 shadow-xl relative ${sec.bg} min-h-[300px]`}>
                  <WashiTape color={sec.tape} className="opacity-80 w-24 h-6 -top-3" />
                  
                  <div className="flex justify-between items-center mb-6 mt-2">
                    <h2 className="text-xl font-serif font-bold text-stone-700">{sec.title}</h2>
                    <button onClick={() => handleAddWish(sec.type)} className="text-stone-400 hover:text-stone-700"><Plus size={20} /></button>
                  </div>
                  
                  <ul className="space-y-4 list-disc pl-4 marker:text-stone-400">
                    {wishlist.filter(w => w.type === sec.type).map(item => (
                      <li key={item.id} className="text-sm font-serif text-stone-700 relative group pl-2">
                        <span className="relative z-10">{item.title}</span>
                        {/* Highlighter effect on hover */}
                        <span className="absolute left-0 bottom-0 w-full h-2 bg-yellow-200/50 -z-0 opacity-0 group-hover:opacity-100 transition-opacity transform -rotate-1"></span>
                        <button onClick={() => deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'wishlist', item.id))} 
                          className="absolute -right-4 top-0 opacity-0 group-hover:opacity-100 text-stone-400 hover:text-red-400"><X size={12}/></button>
                      </li>
                    ))}
                  </ul>
                </div>
              ))}
            </div>
          )}

          {/* LOGS TAB - Polaroids */}
          {activeTab === 'meetings' && (
            <div className="max-w-3xl mx-auto">
               <div className="bg-[#f9f7f1] p-8 rounded-sm shadow-xl border border-[#e8e4dc] mb-16 relative">
                 <div className="absolute -left-2 top-8 -bottom-8 w-4 border-l-2 border-stone-300 flex flex-col justify-between py-2">
                    {[...Array(5)].map((_,i) => <div key={i} className="w-8 h-8 rounded-full border border-stone-300 bg-white -ml-4 shadow-sm"></div>)}
                 </div>
                 
                 <h2 className="text-xl font-serif font-bold text-stone-700 mb-6 text-center border-b border-stone-200 pb-4">Add a Memory</h2>
                 <form onSubmit={async (e) => {
                    e.preventDefault();
                    const fd = new FormData(e.target);
                    const mName = fd.get('member');
                    const mData = CORE_4.find(m => m.name === mName);
                    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'meetings'), {
                      content: fd.get('content'),
                      photoUrl: fd.get('photoUrl') || "https://images.unsplash.com/photo-1517248135467-4c7edcad34c4",
                      month: MONTHS[new Date().getMonth()],
                      timestamp: serverTimestamp(),
                      author: mName,
                      authorIcon: mData?.icon || "✨"
                    });
                    e.target.reset();
                 }} className="space-y-4 pl-6">
                    <div className="flex gap-4">
                      <select name="member" className="bg-transparent border-b border-stone-300 p-2 outline-none font-mono text-xs text-stone-600 w-1/3">
                        {CORE_4.map(m => <option key={m.name} value={m.name}>{m.icon} {m.name}</option>)}
                      </select>
                      <input name="photoUrl" placeholder="Photo URL..." className="bg-transparent border-b border-stone-300 p-2 outline-none font-mono text-xs text-stone-600 w-2/3" />
                    </div>
                    <textarea name="content" required placeholder="Dear Diary..." className="w-full bg-[#f4efe6] border border-stone-200 p-4 outline-none font-serif text-sm text-stone-800 min-h-[100px] shadow-inner" />
                    <button type="submit" className="w-full bg-[#3e342a] text-[#f2e8cf] font-bold py-3 uppercase tracking-widest text-xs hover:bg-[#544a3d] transition-colors shadow-md">Glue In Page</button>
                 </form>
               </div>

               <div className="grid grid-cols-1 md:grid-cols-2 gap-12">
                {meetings.map((log, idx) => (
                  <div key={log.id} className={`bg-white p-3 pb-12 shadow-[5px_5px_15px_rgba(0,0,0,0.15)] relative transform transition-transform hover:scale-[1.02] ${idx % 2 === 0 ? 'rotate-2' : '-rotate-1'}`}>
                    {/* Tape holding photo */}
                    <div className="absolute -top-3 left-1/2 -translate-x-1/2 w-16 h-6 bg-[#e0e0e0]/60 backdrop-blur-sm transform rotate-1"></div>
                    
                    <div className="bg-stone-100 aspect-[4/3] mb-4 overflow-hidden filter sepia-[.2]">
                      <img src={log.photoUrl} className="w-full h-full object-cover" alt="Recap" />
                    </div>
                    
                    <div className="px-2 text-center">
                       <p className="font-serif text-stone-800 text-lg italic leading-tight mb-3">"{log.content}"</p>
                       <div className="flex justify-center items-center gap-2 text-[10px] font-mono text-stone-400 uppercase tracking-widest">
                         <span>{log.authorIcon}</span>
                         <span>{log.month} • {log.author}</span>
                       </div>
                    </div>
                    
                    <button onClick={() => deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'meetings', log.id))} 
                      className="absolute bottom-2 right-2 text-[8px] font-bold text-stone-300 hover:text-red-400 uppercase">Remove</button>
                  </div>
                ))}
               </div>
            </div>
          )}
        </main>
      </div>

      {/* MODALS - PAPER STYLE */}
      {isModalOpen && (
        <div className="fixed inset-0 z-50 flex items-center justify-center p-4 bg-[#3e342a]/80 backdrop-blur-sm animate-in fade-in duration-300">
          <div className="bg-[#fdfbf7] w-full max-w-sm p-8 shadow-2xl relative border-2 border-[#e8e4dc]">
            <div className="absolute -top-4 left-1/2 -translate-x-1/2 w-4 h-4 rounded-full bg-[#3e342a] shadow-sm"></div> {/* Hanging hole */}
            
            <div className="flex justify-between items-center mb-6 border-b border-stone-200 pb-2">
              <h2 className="text-xl font-serif font-bold text-stone-800">{MONTHS[currentMonth]} {selectedDay}</h2>
              <button onClick={() => setIsModalOpen(false)}><X className="text-stone-400" /></button>
            </div>
            
            <form onSubmit={handleAddEvent} className="space-y-5">
              <div className="flex justify-center gap-4 border-b border-stone-100 pb-4">
                {CORE_4.map(m => (
                  <label key={m.name} className="cursor-pointer group">
                    <input type="radio" name="member" value={m.name} required className="hidden peer" />
                    <div className="w-8 h-8 flex items-center justify-center text-lg grayscale group-hover:grayscale-0 peer-checked:grayscale-0 peer-checked:scale-125 transition-all">
                      {m.icon}
                    </div>
                  </label>
                ))}
              </div>
              
              <input name="title" required className="w-full bg-transparent border-b border-stone-300 p-2 text-sm font-bold text-stone-800 outline-none placeholder:text-stone-400" placeholder="Event Name" />
              <textarea name="description" className="w-full bg-[#f4efe6] p-3 text-sm text-stone-700 outline-none h-24 font-serif" placeholder="Details..." />
              
              <div className="flex items-center gap-2">
                <DollarSign size={16} className="text-stone-400" />
                <input name="expenses" type="number" className="w-24 bg-transparent border-b border-stone-300 p-2 text-sm outline-none" placeholder="0.00" />
              </div>
              
              <button type="submit" className="w-full bg-[#8c7b60] text-white py-3 text-xs font-bold uppercase tracking-widest hover:bg-[#7a6a4f]">Save Entry</button>
            </form>
          </div>
        </div>
      )}

      {isInviteModalOpen && (
        <div className="fixed inset-0 z-50 flex items-center justify-center p-4 bg-[#3e342a]/80 backdrop-blur-sm animate-in fade-in duration-300">
          <div className="bg-[#fdfbf7] w-full max-w-sm p-10 shadow-2xl relative" style={{ clipPath: "polygon(0% 0%, 100% 0%, 100% 90%, 50% 100%, 0% 90%)" }}>
             <div className="text-center mb-6">
               <h2 className="text-2xl font-serif font-bold text-stone-800 border-b-2 border-stone-800 inline-block pb-1">INVITATION</h2>
             </div>
             <button onClick={() => setIsInviteModalOpen(false)} className="absolute top-4 right-4 text-stone-400"><X size={18} /></button>
             
             <form onSubmit={handleAddInvite} className="space-y-4">
               <div className="flex justify-center gap-4 mb-4">
                 {CORE_4.map(m => (
                   <label key={m.name} className="cursor-pointer">
                     <input type="radio" name="member" value={m.name} required className="hidden peer" />
                     <span className="text-xl opacity-30 peer-checked:opacity-100 transition-opacity">{m.icon}</span>
                   </label>
                 ))}
               </div>
               
               <input name="title" required className="w-full text-center bg-transparent border-b border-stone-300 p-2 text-sm font-bold outline-none" placeholder="Occasion" />
               <div className="flex gap-2">
                 <input name="date" type="date" required className="w-1/2 bg-transparent border-b border-stone-300 p-2 text-xs font-mono outline-none" />
                 <input name="time" type="time" required className="w-1/2 bg-transparent border-b border-stone-300 p-2 text-xs font-mono outline-none" />
               </div>
               <input name="location" required className="w-full bg-transparent border-b border-stone-300 p-2 text-xs font-mono outline-none text-center" placeholder="@ Location" />
               
               <div className="pt-4 text-center">
                 <button type="submit" className="px-8 py-2 border border-stone-800 text-stone-800 text-[10px] font-bold uppercase tracking-widest hover:bg-stone-800 hover:text-white transition-colors">RSVP Request</button>
               </div>
             </form>
          </div>
        </div>
      )}

      {/* LINK MODAL */}
      {isLinkModalOpen && (
        <div className="fixed inset-0 z-50 flex items-center justify-center p-4 bg-[#3e342a]/80 backdrop-blur-sm animate-in fade-in duration-300">
          <div className="bg-[#fff] w-full max-w-sm p-8 shadow-2xl relative transform rotate-1">
             <div className="absolute -top-3 left-1/2 -translate-x-1/2 text-stone-300"><Paperclip size={24} /></div>
             <div className="text-center mb-6 mt-2">
               <h2 className="text-xl font-serif font-bold text-stone-800">Add Resource</h2>
             </div>
             <button onClick={() => setIsLinkModalOpen(false)} className="absolute top-2 right-2 text-stone-400"><X size={16} /></button>
             
             <form onSubmit={handleAddLink} className="space-y-4">
               <div className="flex justify-center gap-4 mb-2">
                 {CORE_4.map(m => (
                   <label key={m.name} className="cursor-pointer">
                     <input type="radio" name="member" value={m.name} required className="hidden peer" />
                     <span className="text-xl opacity-30 peer-checked:opacity-100 transition-opacity">{m.icon}</span>
                   </label>
                 ))}
               </div>
               
               <input name="title" required className="w-full bg-gray-50 border border-gray-200 p-3 text-sm font-bold outline-none rounded-sm" placeholder="Title (e.g. Budget Sheet)" />
               <input name="url" required type="url" className="w-full bg-gray-50 border border-gray-200 p-3 text-xs font-mono outline-none rounded-sm" placeholder="https://..." />
               
               <button type="submit" className="w-full bg-blue-600 text-white py-3 text-xs font-bold uppercase tracking-widest hover:bg-blue-700 transition-colors rounded-sm">Pin Link</button>
             </form>
          </div>
        </div>
      )}
    </div>
  );
};

export default App;
