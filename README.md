import React, { useState, useEffect, useMemo, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, 
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  getFirestore, 
  doc, 
  setDoc, 
  onSnapshot, 
  collection, 
  updateDoc, 
  deleteDoc, 
  addDoc 
} from 'firebase/firestore';
import { 
  Truck, Package, MapPin, CheckCircle, Clock, Wallet, User, Star, LogOut, 
  Navigation, Phone, ShieldCheck, IndianRupee, Bell, X, Check, CreditCard, 
  ChevronRight, AlertCircle, RefreshCw, ShieldAlert, MessageSquare, Map, 
  Send, TrendingUp, Sparkles, Volume2, VolumeX, Shield, UserCheck, HelpCircle
} from 'lucide-react';

const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {
  apiKey: "demo-api-key",
  authDomain: "demo.firebaseapp.com",
  projectId: "demo-project",
  storageBucket: "demo.appspot.com",
  messagingSenderId: "12345678",
  appId: "1:1234567:web:demo"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'live-transport-v1';

export default function LiveTransportApp() {
  const [user, setUser] = useState(null);
  const [profile, setProfile] = useState(null);
  const [orders, setOrders] = useState([]);
  const [usersInfo, setUsersInfo] = useState({});
  const [loading, setLoading] = useState(true);
  const [notifications, setNotifications] = useState([]);
  const [showNotifications, setShowNotifications] = useState(false);
  const [audioEnabled, setAudioEnabled] = useState(true);
  const [simulationRole, setSimulationRole] = useState(null);
  const [firebaseError, setFirebaseError] = useState(null);

  const playSoundEffect = (type) => {
    if (!audioEnabled) return;
    try {
      const AudioCtx = window.AudioContext || window.webkitAudioContext;
      if (!AudioCtx) return;
      const ctx = new AudioCtx();
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      osc.connect(gain);
      gain.connect(ctx.destination);

      if (type === 'beep') {
        osc.frequency.setValueAtTime(880, ctx.currentTime);
        gain.gain.setValueAtTime(0.05, ctx.currentTime);
        osc.start();
        osc.stop(ctx.currentTime + 0.1);
      } else if (type === 'success') {
        osc.frequency.setValueAtTime(523.25, ctx.currentTime); // C5
        osc.frequency.setValueAtTime(659.25, ctx.currentTime + 0.1); // E5
        osc.frequency.setValueAtTime(783.99, ctx.currentTime + 0.2); // G5
        gain.gain.setValueAtTime(0.05, ctx.currentTime);
        osc.start();
        osc.stop(ctx.currentTime + 0.35);
      } else if (type === 'alert') {
        osc.frequency.setValueAtTime(330, ctx.currentTime);
        osc.frequency.setValueAtTime(220, ctx.currentTime + 0.15);
        gain.gain.setValueAtTime(0.08, ctx.currentTime);
        osc.start();
        osc.stop(ctx.currentTime + 0.3);
      }
    } catch (e) {
      console.warn("Audio Context blocked or unsupported:", e);
    }
  };

  const speakNotification = (text) => {
    if (!audioEnabled) return;
    try {
      if ('speechSynthesis' in window) {
        window.speechSynthesis.cancel();
        const utterance = new SpeechSynthesisUtterance(text);
        utterance.rate = 0.95;
        utterance.pitch = 1.0;
        // Attempt to select Hindi or polite Indian accent if available
        const voices = window.speechSynthesis.getVoices();
        const targetVoice = voices.find(v => v.lang.includes('hi') || v.lang.includes('IN'));
        if (targetVoice) utterance.voice = targetVoice;
        window.speechSynthesis.speak(utterance);
      }
    } catch(e) {
      console.warn("TTS Failed:", e);
    }
  };

  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth Error:", err);
        setFirebaseError("Authentication error. Using offline/sandboxed storage modes.");
        setLoading(false);
      }
    };
    initAuth();

    const unsubscribe = onAuthStateChanged(auth, (u) => {
      setUser(u);
      if (!u) setLoading(false);
    });
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    
    // Listen to users information
    const usersRef = collection(db, 'artifacts', appId, 'public', 'data', 'lt_users');
    const unsubUsers = onSnapshot(usersRef, (snapshot) => {
      const uData = {};
      snapshot.docs.forEach(doc => {
        uData[doc.id] = doc.data();
      });
      setUsersInfo(uData);
      if (uData[user.uid]) {
        setProfile(uData[user.uid]);
      }
      setLoading(false);
    }, (err) => {
      console.error("Users Firestore Error:", err);
      // Fallback fallback profiles mock data in case of FireStore security limits during initial load
      setLoading(false);
    });

    // Listen to orders
    const ordersRef = collection(db, 'artifacts', appId, 'public', 'data', 'lt_orders');
    const unsubOrders = onSnapshot(ordersRef, (snapshot) => {
      const oData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setOrders(oData.sort((a, b) => b.createdAt - a.createdAt));
    }, (err) => {
      console.error("Orders Firestore Error:", err);
    });

    return () => {
      unsubUsers();
      unsubOrders();
    };
  }, [user]);

  useEffect(() => {
    if (!user || orders.length === 0) return;

    // Real-time check for expired orders (5 minutes = 300,000 ms)
    const interval = setInterval(() => {
      const now = Date.now();
      orders.forEach(order => {
        if (order.status === 'pending' && (now - order.createdAt > 300000)) {
          // Cancel the order
          const orderRef = doc(db, 'artifacts', appId, 'public', 'data', 'lt_orders', order.id);
          updateDoc(orderRef, { 
            status: 'cancelled', 
            cancelReason: 'Sorry, koi transporter available nahi mila.' 
          }).catch(err => console.error(err));
          
          triggerLocalAlert("Order Timeout", `Order ${order.id} was auto-cancelled due to lack of local transporter.`);
        }
      });
    }, 5000); 

    return () => clearInterval(interval);
  }, [user, orders]);

  const triggerLocalAlert = (title, message) => {
    const newNotif = {
      id: Date.now(),
      title,
      message,
      time: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }),
      read: false
    };
    setNotifications(prev => [newNotif, ...prev]);
    playSoundEffect('alert');
    speakNotification(`${title}: ${message}`);
  };

  // Determine active view based on profile settings or simulator controls
  const activeRole = simulationRole || profile?.role || 'guest';

  if (loading) {
    return (
      <div className="min-h-screen flex flex-col items-center justify-center bg-slate-950 text-white">
        <div className="relative mb-6">
          <div className="animate-ping absolute inset-0 bg-blue-500 rounded-full opacity-20"></div>
          <div className="bg-gradient-to-tr from-blue-600 to-indigo-700 p-6 rounded-full text-white shadow-xl relative">
            <Truck size={48} className="animate-pulse" />
          </div>
        </div>
        <h2 className="text-xl font-black tracking-widest text-slate-100">LIVE TRANSPORT</h2>
        <p className="text-xs font-semibold uppercase tracking-wider text-blue-400 mt-2">Connecting India's Freight Infrastructure...</p>
        <div className="w-16 h-1.5 bg-indigo-900 rounded-full mt-4 overflow-hidden">
          <div className="h-full bg-blue-500 rounded-full animate-infinite-loading w-8"></div>
        </div>
      </div>
    );
  }

  if (user && activeRole === 'guest') {
    return <LoginRegistration user={user} triggerLocalAlert={triggerLocalAlert} />;
  }

  return (
    <div className="min-h-screen bg-slate-900 flex flex-col items-center justify-start font-sans">
      
      {/* SIMULATOR COMPONENT FOR REAL-TIME FLOW TESTING */}
      <div className="w-full max-w-lg bg-indigo-950 text-white px-4 py-3 text-xs shadow-2xl border-b border-indigo-800 z-50 rounded-b-xl">
        <div className="flex justify-between items-center">
          <div className="flex items-center gap-1.5">
            <span className="w-2.5 h-2.5 rounded-full bg-emerald-500 animate-pulse"></span>
            <span className="font-extrabold tracking-wide text-[11px] text-indigo-200">🛠️ PLATFORM OWNER TESTING CONSOLE</span>
          </div>
          <span className="bg-blue-600 text-[10px] px-2.5 py-0.5 rounded-full font-black">ACTIVE VIEW: {activeRole.toUpperCase()}</span>
        </div>
        <div className="grid grid-cols-4 gap-1.5 mt-2">
          <button 
            onClick={() => { setSimulationRole('client'); playSoundEffect('beep'); }} 
            className={`py-1.5 rounded font-bold text-center transition ${activeRole === 'client' ? 'bg-blue-500 text-white shadow-md' : 'bg-indigo-900/60 text-indigo-200 hover:bg-indigo-900'}`}
          >
            📦 Client UI
          </button>
          <button 
            onClick={() => { setSimulationRole('transporter'); playSoundEffect('beep'); }} 
            className={`py-1.5 rounded font-bold text-center transition ${activeRole === 'transporter' ? 'bg-blue-500 text-white shadow-md' : 'bg-indigo-900/60 text-indigo-200 hover:bg-indigo-900'}`}
          >
            🚚 Driver UI
          </button>
          <button 
            onClick={() => { setSimulationRole('admin'); playSoundEffect('beep'); }} 
            className={`py-1.5 rounded font-bold text-center transition ${activeRole === 'admin' ? 'bg-blue-500 text-white shadow-md' : 'bg-indigo-900/60 text-indigo-200 hover:bg-indigo-900'}`}
          >
            👑 Owner UI
          </button>
          <button 
            onClick={() => { setSimulationRole(null); playSoundEffect('beep'); }}
            className="py-1.5 rounded font-bold bg-slate-800 hover:bg-slate-700 text-center text-slate-300"
          >
            🔄 Reset
          </button>
        </div>
        <p className="text-[10px] text-indigo-300/80 mt-1.5 text-center">
          💡 **Note:** Owner access automatically defaults to <strong className="text-yellow-400">ssskgane@gmail.com</strong>
        </p>
      </div>

      {/* CORE MOBILE CONTAINER FRAME */}
      <div className="w-full max-w-lg bg-slate-950 shadow-2xl relative flex flex-col min-h-[90vh] md:min-h-[85vh] md:my-4 md:rounded-3xl border border-slate-800 overflow-hidden">
        
        {/* HEADER BRANDING */}
        <header className="bg-slate-900/90 backdrop-blur-md border-b border-slate-800 p-4 flex justify-between items-center z-20 sticky top-0">
          <div className="flex items-center gap-2.5">
            <div className="bg-gradient-to-tr from-blue-600 to-indigo-700 p-2 rounded-xl text-white shadow-lg shadow-blue-500/20">
              <Truck size={22} className="animate-pulse" />
            </div>
            <div>
              <h1 className="text-base font-black tracking-tight text-white leading-tight flex items-center gap-1">
                Live Transport <Sparkles size={14} className="text-yellow-400 fill-yellow-400" />
              </h1>
              <span className="text-[10px] text-slate-400 font-medium tracking-wide">Bharat Logistics Network • Phase 1</span>
            </div>
          </div>
          
          <div className="flex items-center gap-2">
            {/* Audio Toggle */}
            <button 
              onClick={() => setAudioEnabled(!audioEnabled)}
              className="p-1.5 hover:bg-slate-800 text-slate-400 hover:text-white rounded-lg transition"
              title={audioEnabled ? "Disable Voice Guidance" : "Enable Voice Guidance"}
            >
              {audioEnabled ? <Volume2 size={18} className="text-emerald-400" /> : <VolumeX size={18} />}
            </button>

            {/* Notification Center */}
            <button 
              onClick={() => {
                setShowNotifications(!showNotifications);
                setShowNotifications(prev => {
                  if (!prev) setNotifications(n => n.map(item => ({...item, read: true})));
                  return !prev;
                });
                playSoundEffect('beep');
              }} 
              className="relative p-2 hover:bg-slate-800 text-slate-300 hover:text-white rounded-full transition"
            >
              <Bell size={20} />
              {notifications.some(n => !n.read) && (
                <span className="absolute top-1 right-1 w-2.5 h-2.5 bg-rose-500 rounded-full ring-2 ring-slate-900 animate-pulse"></span>
              )}
            </button>

            <button 
              onClick={() => { 
                auth.signOut(); 
                window.location.reload(); 
              }} 
              className="p-2 hover:bg-slate-800 text-slate-400 hover:text-rose-400 rounded-full transition"
              title="Logout"
            >
              <LogOut size={18} />
            </button>
          </div>
        </header>

        {/* FLOATING NOTIFICATION BOX */}
        {showNotifications && (
          <div className="absolute top-[68px] left-4 right-4 bg-slate-900 border border-slate-800 rounded-2xl shadow-2xl z-50 max-h-72 overflow-y-auto divide-y divide-slate-800/60 animate-fade-in">
            <div className="p-3 bg-slate-900/90 flex justify-between items-center sticky top-0 backdrop-blur-sm">
              <span className="font-extrabold text-xs text-slate-200 tracking-wider uppercase">Live Activity Alert</span>
              <button onClick={() => setShowNotifications(false)} className="text-slate-400 hover:text-white"><X size={16} /></button>
            </div>
            {notifications.length === 0 ? (
              <div className="p-8 text-center text-xs text-slate-500">No dispatch notifications currently.</div>
            ) : (
              notifications.map(n => (
                <div key={n.id} className="p-3 hover:bg-slate-800/50 flex gap-2.5 transition">
                  <div className="w-2 h-2 rounded-full bg-blue-500 mt-1.5 shrink-0 animate-pulse"></div>
                  <div>
                    <p className="text-xs font-extrabold text-slate-200">{n.title}</p>
                    <p className="text-[11px] text-slate-400 mt-0.5">{n.message}</p>
                    <span className="text-[9px] text-slate-500 block mt-1 font-semibold">{n.time}</span>
                  </div>
                </div>
              ))
            )}
          </div>
        )}

        {/* CORE SCROLLABLE CONTENT BODY */}
        <div className="flex-1 overflow-y-auto pb-10 bg-slate-950">
          {activeRole === 'admin' && <AdminDashboard orders={orders} />}
          {activeRole === 'client' && <ClientDashboard user={user} profile={profile} orders={orders} triggerLocalAlert={triggerLocalAlert} usersInfo={usersInfo} />}
          {activeRole === 'transporter' && <TransporterDashboard user={user} profile={profile} orders={orders} triggerLocalAlert={triggerLocalAlert} />}
        </div>
      </div>
    </div>
  );
}

function LoginRegistration({ user, triggerLocalAlert }) {
  const [role, setRole] = useState('client');
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [mobile, setMobile] = useState('');
  const [city, setCity] = useState('Deoghar');
  
  // Verification details
  const [vehicle, setVehicle] = useState('Bike');
  const [dlNumber, setDlNumber] = useState('');
  const [rcNumber, setRcNumber] = useState('');
  const [error, setError] = useState('');
  const [submitting, setSubmitting] = useState(false);

  const handleRegister = async (e) => {
    e.preventDefault();
    setError('');

    if (!name || !email || !mobile) {
      return setError("All essential fields (Name, Email, Mobile) are required!");
    }
    
    setSubmitting(true);

    // Auto-assigned owner checks based on prompt instructions
    let assignedRole = role;
    if (email.trim().toLowerCase() === 'ssskgane@gmail.com') {
      assignedRole = 'admin';
    }

    if (assignedRole === 'transporter' && (!dlNumber || !rcNumber)) {
      setSubmitting(false);
      return setError("Valid Driving License (DL) and RC verification are mandatory for onboarding.");
    }

    const avatarSeed = name.replace(/\s+/g, '-').toLowerCase();
    const photoUrl = `https://api.dicebear.com/7.x/bottts/svg?seed=${avatarSeed}`;

    const userData = {
      uid: user.uid,
      name: name.trim(),
      email: email.trim().toLowerCase(),
      mobile: mobile.trim(),
      role: assignedRole,
      photoUrl,
      city: city,
      joinedAt: Date.now()
    };

    if (assignedRole === 'transporter') {
      userData.vehicleType = vehicle;
      userData.status = 'online';
      userData.dlNumber = dlNumber.trim().toUpperCase();
      userData.rcNumber = rcNumber.trim().toUpperCase();
      userData.deliveries = 0;
      userData.rating = 5.0;
    }

    try {
      const userRef = doc(db, 'artifacts', appId, 'public', 'data', 'lt_users', user.uid);
      await setDoc(userRef, userData);
      triggerLocalAlert("Registration Complete", `Welcome ${name}! Setup completed as ${assignedRole}.`);
    } catch (err) {
      console.error(err);
      setError("Registration error. Ensure you fill out the details correctly.");
      setSubmitting(false);
    }
  };

  return (
    <div className="min-h-screen bg-slate-950 flex items-center justify-center p-4 w-full">
      <div className="max-w-md w-full bg-slate-900 border border-slate-800 rounded-3xl shadow-2xl overflow-hidden">
        <div className="bg-gradient-to-r from-blue-700 to-indigo-900 p-6 text-center text-white relative">
          <div className="absolute top-2 right-2 flex items-center gap-1.5 text-[10px] bg-black/30 px-2 py-0.5 rounded-full font-bold">
            <Shield size={10} className="text-yellow-400" /> Secure Onboarding
          </div>
          <Truck size={44} className="mx-auto mb-2 text-white animate-pulse" />
          <h2 className="text-2xl font-black tracking-tight uppercase">Live Transport</h2>
          <p className="text-blue-200 text-xs mt-1">India's Peer-to-Peer Logistics Network</p>
        </div>
        
        <form onSubmit={handleRegister} className="p-6 space-y-4 max-h-[75vh] overflow-y-auto">
          {error && (
            <p className="text-rose-400 text-xs font-bold bg-rose-500/10 p-3 rounded-xl border border-rose-500/20 flex items-center gap-1.5 animate-pulse">
              <AlertCircle size={14} /> {error}
            </p>
          )}

          {email.trim().toLowerCase() === 'ssskgane@gmail.com' && (
            <div className="bg-yellow-500/10 border border-yellow-500/20 p-3 rounded-xl text-xs text-yellow-300 font-semibold flex items-center gap-2">
              <Sparkles size={14} className="text-yellow-400 shrink-0" />
              Recognized Platform Owner Address. Automatically setting Admin role!
            </div>
          )}
          
          <div>
            <label className="block text-[10px] font-black text-slate-400 uppercase tracking-wider mb-2">Chose Your Profile Role</label>
            <div className="grid grid-cols-2 gap-3">
              <button 
                type="button" 
                onClick={() => setRole('client')} 
                className={`py-3 px-4 rounded-xl border transition-all duration-300 flex flex-col items-center gap-1 ${role === 'client' ? 'bg-indigo-600/20 border-indigo-500 text-indigo-300 font-extrabold shadow-lg shadow-indigo-500/10' : 'border-slate-800 text-slate-500 hover:bg-slate-800/40'}`}
              >
                <Package size={22} className={role === 'client' ? 'text-indigo-400' : 'text-slate-600'} />
                <span className="text-xs">📦 Client (Saman Bhejein)</span>
              </button>
              <button 
                type="button" 
                onClick={() => setRole('transporter')} 
                className={`py-3 px-4 rounded-xl border transition-all duration-300 flex flex-col items-center gap-1 ${role === 'transporter' ? 'bg-indigo-600/20 border-indigo-500 text-indigo-300 font-extrabold shadow-lg shadow-indigo-500/10' : 'border-slate-800 text-slate-500 hover:bg-slate-800/40'}`}
              >
                <Truck size={22} className={role === 'transporter' ? 'text-indigo-400' : 'text-slate-600'} />
                <span className="text-xs">🚚 Transporter (Earning)</span>
              </button>
            </div>
          </div>

          <div className="space-y-3">
            <div>
              <label className="block text-[10px] font-black text-slate-400 uppercase tracking-wider mb-1">Pura Naam (Full Name)</label>
              <input type="text" value={name} onChange={e=>setName(e.target.value)} required className="w-full px-4 py-2 bg-slate-950 border border-slate-800 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none text-slate-100 text-xs" placeholder="Ramesh Kumar" />
            </div>
            
            <div>
              <label className="block text-[10px] font-black text-slate-400 uppercase tracking-wider mb-1">Email ID</label>
              <input type="email" value={email} onChange={e=>setEmail(e.target.value)} required className="w-full px-4 py-2 bg-slate-950 border border-slate-800 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none text-slate-100 text-xs" placeholder="ramesh@gmail.com" />
              <p className="text-[10px] text-slate-500 mt-1">💡 Owner Dashboard demo ke liye <strong className="text-yellow-400 font-bold">ssskgane@gmail.com</strong> enter karein.</p>
            </div>

            <div>
              <label className="block text-[10px] font-black text-slate-400 uppercase tracking-wider mb-1">Mobile Number</label>
              <input type="tel" value={mobile} onChange={e=>setMobile(e.target.value)} required className="w-full px-4 py-2 bg-slate-950 border border-slate-800 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none text-slate-100 text-xs" placeholder="+91 XXXXX XXXXX" />
            </div>

            <div>
              <label className="block text-[10px] font-black text-slate-400 uppercase tracking-wider mb-1 font-semibold">Shehar (Operating City)</label>
              <select value={city} onChange={e=>setCity(e.target.value)} className="w-full px-3 py-2 bg-slate-950 border border-slate-800 rounded-xl text-slate-100 text-xs focus:ring-2 focus:ring-indigo-500">
                <option value="Deoghar">Deoghar</option>
                <option value="Patna">Patna</option>
                <option value="Ranchi">Ranchi</option>
                <option value="Delhi">Delhi</option>
                <option value="Mumbai">Mumbai</option>
              </select>
              <p className="text-[9px] text-slate-500 mt-1">💡 City-matching system filters orders by matching operating city.</p>
            </div>
          </div>

          {role === 'transporter' && (
            <div className="bg-slate-950 p-4 rounded-xl border border-slate-800 space-y-3 animate-slide-up">
              <p className="text-xs font-bold text-slate-300 border-b border-slate-800 pb-1 flex items-center gap-1">
                <UserCheck size={14} className="text-blue-400" /> Transporter Credentials Verification
              </p>
              
              <div>
                <label className="block text-[9px] font-bold text-slate-400 uppercase tracking-wide mb-1">Vehicle Type</label>
                <select value={vehicle} onChange={e=>setVehicle(e.target.value)} className="w-full px-3 py-1.5 bg-slate-900 border border-slate-800 rounded-lg text-slate-100 text-xs">
                  <option value="Bike">🏍️ Bike (Standard Earning x1.0)</option>
                  <option value="Auto">🛺 Auto (Heavy load x1.5)</option>
                  <option value="Pickup Van">🛻 Pickup Van (Logistics x2.0)</option>
                  <option value="Mini Truck">🚚 Mini Truck (Freight x2.5)</option>
                  <option value="Heavy Truck">🚛 Heavy Truck (Maxi Freight x3.5)</option>
                </select>
              </div>

              <div>
                <label className="block text-[9px] font-bold text-slate-400 uppercase tracking-wide mb-1">Driving License (DL) Number</label>
                <input type="text" value={dlNumber} onChange={e=>setDlNumber(e.target.value)} required placeholder="DL-JH15A2024XXXX" className="w-full px-3 py-1.5 bg-slate-900 border border-slate-800 rounded-lg text-xs outline-none text-slate-100 uppercase" />
              </div>

              <div>
                <label className="block text-[9px] font-bold text-slate-400 uppercase tracking-wide mb-1">Vehicle RC Registration Number</label>
                <input type="text" value={rcNumber} onChange={e=>setRcNumber(e.target.value)} required placeholder="BR-15X-XXXX / JH-15A-XXXX" className="w-full px-3 py-1.5 bg-slate-900 border border-slate-800 rounded-lg text-xs outline-none text-slate-100 uppercase" />
              </div>
            </div>
          )}

          <button 
            type="submit" 
            disabled={submitting}
            className="w-full bg-gradient-to-r from-blue-600 to-indigo-700 text-white font-extrabold py-3.5 rounded-xl shadow-lg transition-all duration-300 hover:from-blue-700 hover:to-indigo-800 hover:shadow-indigo-500/20 flex justify-center items-center gap-1.5"
          >
            {submitting ? <RefreshCw className="animate-spin" size={18} /> : <CheckCircle size={18} />}
            Setup My Account
          </button>
        </form>
      </div>
    </div>
  );
}

function ClientDashboard({ user, profile, orders, triggerLocalAlert, usersInfo }) {
  const [tab, setTab] = useState('new'); // tabs: 'new' or 'history'
  
  const myOrders = useMemo(() => orders.filter(o => o.clientId === user.uid), [orders, user.uid]);
  const activeOrders = myOrders.filter(o => ['pending', 'accepted'].includes(o.status));
  const pastOrders = myOrders.filter(o => ['delivered', 'cancelled'].includes(o.status));

  return (
    <div className="p-4 space-y-4">
      {/* Visual profile overview */}
      <div className="bg-slate-900 border border-slate-800 rounded-2xl p-4 flex justify-between items-center">
        <div>
          <p className="text-[10px] text-slate-400 uppercase font-bold tracking-wider">Happy shipping,</p>
          <h3 className="font-extrabold text-white text-sm">{profile?.name}</h3>
          <p className="text-[11px] text-indigo-400 font-semibold flex items-center gap-1 mt-0.5">
            <MapPin size={12} /> {profile?.city || 'Deoghar'} Division
          </p>
        </div>
        <div className="w-10 h-10 rounded-full border border-slate-700 bg-slate-800 p-1">
          <img src={profile?.photoUrl || `https://api.dicebear.com/7.x/bottts/svg?seed=client`} alt="Client" className="w-full h-full rounded-full" />
        </div>
      </div>

      <div className="flex bg-slate-900 p-1 rounded-xl border border-slate-800">
        <button onClick={() => setTab('new')} className={`flex-1 py-2 text-xs font-black rounded-lg transition-all duration-300 ${tab === 'new' ? 'bg-gradient-to-r from-blue-600 to-indigo-700 shadow text-white' : 'text-slate-400 hover:text-white'}`}>
          📦 Book Cargo
        </button>
        <button onClick={() => setTab('history')} className={`flex-1 py-2 text-xs font-black rounded-lg transition-all duration-300 ${tab === 'history' ? 'bg-gradient-to-r from-blue-600 to-indigo-700 shadow text-white' : 'text-slate-400 hover:text-white'}`}>
          📋 Bookings Tracker ({myOrders.length})
        </button>
      </div>

      {tab === 'new' ? (
        <CreateOrderForm user={user} profile={profile} triggerLocalAlert={triggerLocalAlert} />
      ) : (
        <div className="space-y-4 animate-slide-up">
          {activeOrders.length > 0 && <h3 className="text-[10px] font-black text-slate-400 uppercase tracking-widest pl-1">Active Deliveries</h3>}
          {activeOrders.map(o => (
            <OrderCard key={o.id} order={o} role="client" triggerLocalAlert={triggerLocalAlert} transporterProfile={usersInfo[o.transporterId]} />
          ))}
          
          {pastOrders.length > 0 && <h3 className="text-[10px] font-black text-slate-400 uppercase tracking-widest pl-1">Past Logistics History</h3>}
          {pastOrders.map(o => (
            <OrderCard key={o.id} order={o} role="client" triggerLocalAlert={triggerLocalAlert} transporterProfile={usersInfo[o.transporterId]} />
          ))}
          
          {myOrders.length === 0 && (
            <div className="text-center py-12 text-slate-500 bg-slate-900 border border-slate-800 rounded-2xl p-6">
              <Package size={48} className="mx-auto mb-3 text-slate-700 animate-bounce-slow" />
              <p className="text-sm font-semibold text-slate-300">No Booking History Found</p>
              <p className="text-xs text-slate-500 mt-1">Aapke sabhi orders aur deliveries yahan live update honge.</p>
            </div>
          )}
        </div>
      )}
    </div>
  );
}

function CreateOrderForm({ user, profile, triggerLocalAlert }) {
  const [item, setItem] = useState('');
  const [weight, setWeight] = useState('');
  const [pickup, setPickup] = useState('');
  const [drop, setDrop] = useState('');
  const [distance, setDistance] = useState('');
  const [calculatingDist, setCalculatingDist] = useState(false);
  
  // Payment Simulator Step
  const [paymentStep, setPaymentStep] = useState(false);
  const [paymentMethod, setPaymentMethod] = useState('GPay');
  const [paymentProcessing, setPaymentProcessing] = useState(false);
  const [error, setError] = useState('');

  // Auto Distance Simulator based on random but realistic values
  const triggerDistanceSimulation = () => {
    if (!pickup || !drop) return;
    setCalculatingDist(true);
    setError('');
    
    setTimeout(() => {
      // Create random but sensible delivery route values (3 - 58 KM)
      const generatedDist = Math.floor(Math.random() * 52) + 4;
      setDistance(generatedDist.toString());
      setCalculatingDist(false);
      triggerLocalAlert("Distance Calculated", `Estimated Route: ${generatedDist} KM between pickup and delivery drop.`);
    }, 1100);
  };

  // Standard Formula matching instructions: Base (30) + distance (km * 5) + weight (kg * 2)
  const distNum = parseFloat(distance) || 0;
  const weightNum = parseFloat(weight) || 0;
  const baseCharge = 30;
  const distCharge = distNum * 5;
  const weightCharge = weightNum * 2;
  const totalPrice = baseCharge + distCharge + weightCharge;

  const handleOrderSubmit = async () => {
    if (distNum < 1) return setError("Minimum delivery distance must be at least 1 KM.");
    if (distNum > 60) return setError("Maximum delivery distance 60 KM allowed.");
    if (weightNum <= 0) return setError("Please enter a valid cargo weight.");

    setPaymentProcessing(true);

    setTimeout(async () => {
      const orderId = 'ORD' + Math.floor(100000 + Math.random() * 900000);
      const otp = Math.floor(1000 + Math.random() * 9000).toString();

      const orderData = {
        clientId: user.uid,
        clientName: profile?.name || 'Local Customer',
        clientCity: profile?.city || 'Deoghar',
        clientMobile: profile?.mobile || '9999999999',
        item,
        pickup,
        drop,
        distance: distNum,
        weight: weightNum,
        totalAmount: totalPrice,
        appCommission: totalPrice * 0.10, // 10% App Fee
        transporterEarning: totalPrice * 0.90, // 90% Driver Share
        status: 'pending',
        createdAt: Date.now(),
        otp: otp,
        paymentStatus: 'paid',
        paymentMethod: paymentMethod,
        rated: false
      };

      try {
        const orderRef = doc(db, 'artifacts', appId, 'public', 'data', 'lt_orders', orderId);
        await setDoc(orderRef, orderData);
        
        // Clear States
        setItem(''); setWeight(''); setPickup(''); setDrop(''); setDistance('');
        setPaymentStep(false);
        setPaymentProcessing(false);
        triggerLocalAlert("Order Dispatched", `Order ${orderId} posted in ${profile?.city || 'Deoghar'} matching hub. Finding local drivers...`);
      } catch (err) {
        setError("Error dispatching order. Please check connection and try again.");
        setPaymentProcessing(false);
      }
    }, 1500);
  };

  return (
    <div className="bg-slate-900 p-5 rounded-3xl border border-slate-800 shadow-xl space-y-4 animate-slide-up">
      {!paymentStep ? (
        <form onSubmit={(e) => { e.preventDefault(); setPaymentStep(true); }} className="space-y-4">
          <div className="flex items-center gap-1.5 border-b border-slate-800 pb-3">
            <Package size={20} className="text-blue-500" />
            <h2 className="text-sm font-black text-white uppercase tracking-wider">Book New Cargo Shipment</h2>
          </div>

          {error && <div className="bg-rose-500/15 text-rose-400 p-2.5 rounded-xl text-xs font-semibold border border-rose-500/20">{error}</div>}
          
          <div className="space-y-3">
            <div>
              <label className="text-[10px] font-black text-slate-400 uppercase tracking-wider block mb-1">Item Details (Kya Bhejna Hai?)</label>
              <input type="text" required value={item} onChange={e=>setItem(e.target.value)} placeholder="e.g., Office Files, Cotton Fabric Boxes, Books" className="w-full bg-slate-950 border border-slate-800 focus:border-indigo-500 outline-none p-2.5 rounded-xl text-xs text-white" />
            </div>

            <div className="grid grid-cols-2 gap-4">
              <div>
                <label className="text-[10px] font-black text-slate-400 uppercase tracking-wider block mb-1">Cargo Weight (Wazan - KG)</label>
                <input type="number" required min="1" value={weight} onChange={e=>setWeight(e.target.value)} placeholder="e.g., 15" className="w-full bg-slate-950 border border-slate-800 focus:border-indigo-500 outline-none p-2.5 rounded-xl text-xs text-white" />
              </div>
              <div>
                <label className="text-[10px] font-black text-slate-400 uppercase tracking-wider block mb-1">Distance Status</label>
                <div className="relative">
                  <input type="text" required readOnly value={distance ? `${distance} KM` : ''} placeholder="Auto Calculated" className="w-full bg-slate-950/80 border border-slate-800 p-2.5 rounded-xl text-xs text-slate-300 font-bold text-center" />
                </div>
              </div>
            </div>

            <div>
              <label className="text-[10px] font-black text-slate-400 uppercase tracking-wider block mb-1">Pickup Address</label>
              <div className="flex items-center gap-2 bg-slate-950 border border-slate-800 p-2.5 rounded-xl">
                <MapPin size={16} className="text-emerald-500 shrink-0" />
                <input type="text" required value={pickup} onChange={e=>setPickup(e.target.value)} onBlur={triggerDistanceSimulation} placeholder="e.g., Tower Chowk Landmark, Deoghar" className="w-full bg-transparent text-xs text-white outline-none" />
              </div>
            </div>

            <div>
              <label className="text-[10px] font-black text-slate-400 uppercase tracking-wider block mb-1">Drop Destination Address</label>
              <div className="flex items-center gap-2 bg-slate-950 border border-slate-800 p-2.5 rounded-xl">
                <MapPin size={16} className="text-rose-500 shrink-0" />
                <input type="text" required value={drop} onChange={e=>setDrop(e.target.value)} onBlur={triggerDistanceSimulation} placeholder="e.g., Jasidih Station Road, Deoghar" className="w-full bg-transparent text-xs text-white outline-none" />
              </div>
            </div>
          </div>

          {calculatingDist && (
            <div className="flex items-center gap-2 text-xs text-blue-400 font-bold justify-center py-2 bg-blue-500/10 rounded-xl border border-blue-500/20">
              <RefreshCw size={14} className="animate-spin text-blue-500" /> Auto calculating best logistics route...
            </div>
          )}

          {(!calculatingDist && distNum > 0 && weightNum > 0 && distNum <= 60) && (
            <div className="bg-slate-950 p-4 rounded-2xl border border-slate-800 space-y-2">
              <h4 className="text-[10px] font-black text-indigo-400 tracking-wider uppercase">Live Booking Pricing Estimate</h4>
              <div className="text-xs text-slate-400 flex justify-between"><span>Standard Base Fare:</span> <span className="text-slate-200">₹30</span></div>
              <div className="text-xs text-slate-400 flex justify-between"><span>Distance Fare ({distNum} km x ₹5):</span> <span className="text-slate-200">₹{distCharge}</span></div>
              <div className="text-xs text-slate-400 flex justify-between"><span>Weight Allowance ({weightNum} kg x ₹2):</span> <span className="text-slate-200">₹{weightCharge}</span></div>
              <div className="border-t border-slate-800/80 my-1.5"></div>
              <div className="text-sm font-black flex justify-between text-yellow-400"><span>Estimated Payable Total:</span> <span>₹{totalPrice}</span></div>
            </div>
          )}

          {distNum > 60 && (
            <div className="bg-rose-500/10 text-rose-400 p-3 rounded-xl text-xs font-bold flex gap-2 items-center border border-rose-500/20">
              <ShieldAlert size={16} className="shrink-0" />
              <span>"Maximum delivery distance 60 KM allowed."</span>
            </div>
          )}

          <button 
            type="submit" 
            disabled={!distance || distNum > 60 || calculatingDist} 
            className="w-full bg-gradient-to-r from-blue-600 to-indigo-700 hover:from-blue-700 hover:to-indigo-800 disabled:from-slate-800 disabled:to-slate-800 text-white font-black py-3.5 rounded-xl shadow-lg transition-all flex justify-center items-center gap-1 text-xs"
          >
            Review Charges & Book Now <ChevronRight size={16} />
          </button>
        </form>
      ) : (
        <div className="space-y-4 animate-slide-up">
          <div className="flex justify-between items-center border-b border-slate-800 pb-3">
            <h2 className="text-sm font-black text-white flex items-center gap-1.5 uppercase">
              <CreditCard size={18} className="text-emerald-400" /> Secure Instant Payment checkout
            </h2>
            <button onClick={() => setPaymentStep(false)} className="text-xs text-slate-400 hover:text-white font-bold bg-slate-850 px-2.5 py-1 rounded">Edit</button>
          </div>

          <div className="bg-slate-950 p-4 rounded-xl border border-slate-850 flex justify-between items-center">
            <div>
              <p className="text-[10px] text-slate-400 uppercase font-bold tracking-wider">Total Booking Value</p>
              <h3 className="text-2xl font-black text-white">₹{totalPrice}</h3>
            </div>
            <div className="bg-emerald-500/15 text-emerald-400 px-3 py-1 rounded-full text-xs font-black border border-emerald-500/20">100% Secured</div>
          </div>

          <div className="space-y-2">
            <label className="text-[10px] font-black text-slate-400 uppercase tracking-wider">Select UPI / Wallet Source</label>
            <div className="grid grid-cols-2 gap-2">
              <button 
                onClick={() => setPaymentMethod('GPay')} 
                className={`py-3 px-3 border rounded-xl flex items-center justify-center gap-2 text-xs font-black transition-all ${paymentMethod === 'GPay' ? 'border-indigo-500 bg-indigo-500/10 text-white' : 'border-slate-800 hover:bg-slate-850 text-slate-400'}`}
              >
                <span className="w-5 h-5 rounded-full bg-blue-600 text-[10px] text-white flex items-center justify-center font-bold">G</span> Google Pay
              </button>
              <button 
                onClick={() => setPaymentMethod('PhonePe')} 
                className={`py-3 px-3 border rounded-xl flex items-center justify-center gap-2 text-xs font-black transition-all ${paymentMethod === 'PhonePe' ? 'border-purple-500 bg-purple-500/10 text-white' : 'border-slate-800 hover:bg-slate-850 text-slate-400'}`}
              >
                <span className="w-5 h-5 rounded-full bg-purple-600 text-[10px] text-white flex items-center justify-center font-bold">P</span> PhonePe
              </button>
              <button 
                onClick={() => setPaymentMethod('Paytm')} 
                className={`py-3 px-3 border rounded-xl flex items-center justify-center gap-2 text-xs font-black transition-all ${paymentMethod === 'Paytm' ? 'border-sky-500 bg-sky-500/10 text-white' : 'border-slate-800 hover:bg-slate-850 text-slate-400'}`}
              >
                <span className="w-5 h-5 rounded-full bg-sky-600 text-[10px] text-white flex items-center justify-center font-bold">Py</span> Paytm
              </button>
              <button 
                onClick={() => setPaymentMethod('Card')} 
                className={`py-3 px-3 border rounded-xl flex items-center justify-center gap-2 text-xs font-black transition-all ${paymentMethod === 'Card' ? 'border-emerald-500 bg-emerald-500/10 text-white' : 'border-slate-800 hover:bg-slate-850 text-slate-400'}`}
              >
                <CreditCard size={14} className="text-emerald-400" /> NetBanking
              </button>
            </div>
          </div>

          {paymentProcessing ? (
            <div className="flex flex-col items-center justify-center py-6 text-indigo-400 gap-2">
              <RefreshCw size={36} className="animate-spin text-blue-500" />
              <p className="text-xs font-bold animate-pulse">Processing gateway payment. Please do not close...</p>
            </div>
          ) : (
            <button 
              onClick={handleOrderSubmit} 
              className="w-full bg-gradient-to-r from-emerald-600 to-green-600 hover:from-emerald-700 hover:to-green-700 text-white font-black py-4 rounded-xl shadow-lg transition text-xs uppercase tracking-wider"
            >
              Pay Now & Confirm Shipment
            </button>
          )}
        </div>
      )}
    </div>
  );
}

function TransporterDashboard({ user, profile, orders, triggerLocalAlert }) {
  const [tab, setTab] = useState('available'); // available, active, past
  const [rejectedOrders, setRejectedOrders] = useState([]);

  const handleToggleStatus = async () => {
    const newStatus = profile.status === 'online' ? 'offline' : 'online';
    const userRef = doc(db, 'artifacts', appId, 'public', 'data', 'lt_users', user.uid);
    await updateDoc(userRef, { status: newStatus });
    triggerLocalAlert("Status Updated", `Aap ab ${newStatus === 'online' ? 'Online ✅' : 'Offline ❌'} hain.`);
  };

  const handleRejectOrder = (orderId) => {
    setRejectedOrders(prev => [...prev, orderId]);
    triggerLocalAlert("Order Rejected", `Cargo ${orderId} rejected from your job queue.`);
  };

  // Filter orders matching transporter city and max 60km limit & not rejected locally
  const availableOrders = orders.filter(o => 
    o.status === 'pending' && 
    o.distance <= 60 && 
    o.clientCity?.toLowerCase() === profile.city?.toLowerCase() &&
    !rejectedOrders.includes(o.id)
  );

  const myActiveOrders = orders.filter(o => o.transporterId === user.uid && o.status === 'accepted');
  const myPastOrders = orders.filter(o => o.transporterId === user.uid && o.status === 'delivered');

  const todayEarnings = myPastOrders
    .filter(o => new Date(o.createdAt).toDateString() === new Date().toDateString())
    .reduce((sum, o) => sum + o.transporterEarning, 0);

  return (
    <div className="p-4 space-y-4">
      {/* Transporter status panel */}
      <div className="bg-slate-900 border border-slate-800 p-4 rounded-2xl flex items-center justify-between shadow-lg">
        <div className="flex items-center gap-3">
          <div className="relative">
            <img 
              src={profile?.photoUrl || "https://api.dicebear.com/7.x/bottts/svg?seed=driver"} 
              alt="Transporter" 
              className="w-12 h-12 rounded-full border border-slate-700 bg-slate-800 object-cover"
            />
            <span className={`absolute bottom-0 right-0 w-3 h-3 rounded-full border-2 border-slate-900 ${profile?.status === 'online' ? 'bg-green-500' : 'bg-rose-500'}`}></span>
          </div>
          <div>
            <div className="flex items-center gap-1.5">
              <h3 className="font-extrabold text-white text-sm leading-tight">{profile?.name}</h3>
              <span className="flex items-center text-[10px] text-amber-400 font-bold bg-amber-500/10 px-1.5 py-0.5 rounded-md border border-amber-500/20">
                <Star size={10} className="fill-current mr-0.5" /> {profile?.rating || '5.0'}
              </span>
            </div>
            <p className="text-xs text-slate-400 mt-0.5">{profile?.vehicleType} • {profile?.city} Division</p>
          </div>
        </div>
        
        <div className="flex flex-col items-end gap-1.5">
          <button 
            onClick={handleToggleStatus}
            className={`px-3 py-1 rounded-full text-[10px] font-black transition-all ${profile?.status === 'online' ? 'bg-emerald-500/15 text-emerald-400 border border-emerald-500/20' : 'bg-rose-500/15 text-rose-400 border border-rose-500/20'}`}
          >
            {profile?.status === 'online' ? '🟢 Online' : '🔴 Offline'}
          </button>
          <p className="text-[10px] font-bold text-slate-300">
            <span className="text-slate-400 font-medium">Earned Today:</span> <span className="text-emerald-400 font-black">₹{todayEarnings.toFixed(0)}</span>
          </p>
        </div>
      </div>

      <div className="flex bg-slate-900 p-1 rounded-xl border border-slate-800">
        <button onClick={() => setTab('available')} className={`flex-1 py-2 text-xs font-black rounded-lg transition-all ${tab === 'available' ? 'bg-gradient-to-r from-blue-600 to-indigo-700 text-white' : 'text-slate-400'}`}>
          Available ({availableOrders.length})
        </button>
        <button onClick={() => setTab('active')} className={`flex-1 py-2 text-xs font-black rounded-lg transition-all ${tab === 'active' ? 'bg-gradient-to-r from-blue-600 to-indigo-700 text-white' : 'text-slate-400'}`}>
          Active Deliveries ({myActiveOrders.length})
        </button>
        <button onClick={() => setTab('past')} className={`flex-1 py-2 text-xs font-black rounded-lg transition-all ${tab === 'past' ? 'bg-gradient-to-r from-blue-600 to-indigo-700 text-white' : 'text-slate-400'}`}>
          My Profile & Earnings
        </button>
      </div>

      <div className="space-y-4">
        {tab === 'available' && (
          profile?.status === 'offline' ? (
            <div className="text-center py-12 bg-slate-900 rounded-3xl border border-slate-800 p-4">
              <Truck size={48} className="mx-auto mb-3 text-slate-600" />
              <p className="text-slate-300 font-bold text-sm">Aap Abhi Offline Hain!</p>
              <p className="text-xs text-slate-500 mt-1">Naye available bookings dekhne ke liye online status active toggle karein.</p>
            </div>
          ) : availableOrders.length > 0 ? (
            availableOrders.map(o => (
              <OrderCard key={o.id} order={o} role="transporter" user={user} onReject={handleRejectOrder} triggerLocalAlert={triggerLocalAlert} />
            ))
          ) : (
             <div className="text-center py-12 text-slate-500 bg-slate-900 border border-slate-800 rounded-2xl p-6">
               <Navigation size={48} className="mx-auto mb-3 text-slate-700 animate-pulse" />
               <p className="text-sm font-semibold text-slate-300">No Orders in {profile?.city}</p>
               <p className="text-xs text-slate-500 mt-1">Aapke region ke naye bookings aate hi automatically update ho jayenge.</p>
             </div>
          )
        )}

        {tab === 'active' && (
           myActiveOrders.length > 0 ? (
             myActiveOrders.map(o => <OrderCard key={o.id} order={o} role="transporter" user={user} triggerLocalAlert={triggerLocalAlert} />)
           ) : (
             <div className="text-center py-12 bg-slate-900 rounded-2xl border border-slate-800 p-4 text-slate-500">
               <Package size={40} className="mx-auto mb-2 text-slate-700" />
               <p className="text-xs text-slate-400">Filhal aapke paas koi active shipment load nahi hai.</p>
             </div>
           )
        )}

        {tab === 'past' && (
          <div className="space-y-4 animate-slide-up">
            <div className="bg-gradient-to-r from-emerald-600 to-indigo-800 p-5 rounded-3xl text-white shadow-xl relative overflow-hidden">
              <div className="absolute right-0 bottom-0 opacity-10 transform translate-x-2 translate-y-2"><Truck size={140} /></div>
              <p className="text-[10px] opacity-90 uppercase font-black tracking-wider">Verified Driver Net Share (90%)</p>
              <h2 className="text-3xl font-black flex items-center mt-1">₹{myPastOrders.reduce((s, o) => s + o.transporterEarning, 0).toFixed(0)}</h2>
              <p className="text-[10px] mt-2 opacity-80">🎉 Safe checkout processing. Platform charge (10%) auto deducted.</p>
            </div>

            <div className="bg-slate-900 p-4 rounded-2xl border border-slate-800 space-y-3">
              <h4 className="text-[10px] font-black text-slate-400 uppercase tracking-wider border-b border-slate-800 pb-2">Verification Registry</h4>
              <div className="text-xs flex justify-between">
                <span className="text-slate-400">Driver License Registry:</span> 
                <span className="font-mono font-black text-slate-200">{profile?.dlNumber || 'REGISTERED_VERIFIED'}</span>
              </div>
              <div className="text-xs flex justify-between">
                <span className="text-slate-400">Vehicle RC Record:</span> 
                <span className="font-mono font-black text-slate-200">{profile?.rcNumber || 'REGISTERED_VERIFIED'}</span>
              </div>
              <div className="text-xs flex justify-between">
                <span className="text-slate-400">Total Complete Logistics Trips:</span> 
                <span className="font-bold text-emerald-400">{myPastOrders.length} Deliveries</span>
              </div>
            </div>

            <h3 className="text-[10px] font-black text-slate-400 uppercase tracking-wider pl-1">Completed Jobs Logs</h3>
            {myPastOrders.map(o => (
              <OrderCard key={o.id} order={o} role="transporter" user={user} triggerLocalAlert={triggerLocalAlert} />
            ))}
          </div>
        )}
      </div>
    </div>
  );
}

function AdminDashboard({ orders }) {
  const deliveredOrders = orders.filter(o => o.status === 'delivered');
  const activeOrders = orders.filter(o => ['pending', 'accepted'].includes(o.status));
  
  // Daily Profit Calculation
  const today = new Date().toDateString();
  const todaysCommission = deliveredOrders
    .filter(o => new Date(o.createdAt).toDateString() === today)
    .reduce((sum, o) => sum + o.appCommission, 0);

  const totalCommission = deliveredOrders.reduce((sum, o) => sum + o.appCommission, 0);

  // Helper trigger to add fully populated realistic mock orders for testing purposes
  const injectMockOrders = async () => {
    const randomItems = ['Industrial Pipe Fittings', 'Handloom Clothes Cargo', 'Supermarket Dairy Crates', 'Residential Sofa Set', 'Pooja Items Pack'];
    const pickupLocations = ['Shiv Mandir Chowk, Deoghar', 'Baidyanath Dham Railway Gate, Deoghar', 'Nandan Pahar Road, Deoghar'];
    const dropLocations = ['Jasidih Bypass Junction, Deoghar', 'Kunda Bypass Road, Deoghar', 'Castairs Town Chowk, Deoghar'];
    
    const randomItem = randomItems[Math.floor(Math.random() * randomItems.length)];
    const randomPickup = pickupLocations[Math.floor(Math.random() * pickupLocations.length)];
    const randomDrop = dropLocations[Math.floor(Math.random() * dropLocations.length)];
    
    const distNum = Math.floor(Math.random() * 40) + 10;
    const weightNum = Math.floor(Math.random() * 25) + 5;
    
    const baseCharge = 30;
    const distCharge = distNum * 5;
    const weightCharge = weightNum * 2;
    const totalPrice = baseCharge + distCharge + weightCharge;
    
    const orderId = 'ORD' + Math.floor(100000 + Math.random() * 900000);
    const otp = Math.floor(1000 + Math.random() * 9000).toString();

    const mockOrder = {
      clientId: "mock-client-id",
      clientName: "Demo Client (Deoghar Hub)",
      clientCity: "Deoghar",
      clientMobile: "9876543210",
      item: randomItem,
      pickup: randomPickup,
      drop: randomDrop,
      distance: distNum,
      weight: weightNum,
      totalAmount: totalPrice,
      appCommission: totalPrice * 0.10,
      transporterEarning: totalPrice * 0.90,
      status: 'pending',
      createdAt: Date.now(),
      otp: otp,
      paymentStatus: 'paid',
      paymentMethod: 'UPI'
    };

    try {
      const orderRef = doc(db, 'artifacts', appId, 'public', 'data', 'lt_orders', orderId);
      await setDoc(orderRef, mockOrder);
    } catch (e) {
      console.error(e);
    }
  };

  return (
    <div className="p-4 space-y-4 animate-slide-up">
      <div className="bg-gradient-to-br from-indigo-950 to-slate-900 border border-slate-800 p-6 rounded-3xl shadow-2xl relative overflow-hidden">
        <div className="absolute -right-6 -top-6 opacity-5"><ShieldCheck size={140} /></div>
        <div className="flex justify-between items-center mb-2">
          <span className="bg-yellow-400 text-slate-950 text-[9px] font-black px-2.5 py-0.5 rounded-full uppercase">PLATFORM SUPER ADMIN</span>
          <span className="text-[10px] text-indigo-300 font-semibold">Live Feed</span>
        </div>
        <h2 className="text-lg font-black text-white">Live Transport Super Panel</h2>
        <p className="text-slate-400 text-xs mt-1">Authorized Owner Access: <strong className="text-yellow-400 font-semibold">ssskgane@gmail.com</strong></p>
        
        <div className="grid grid-cols-2 gap-3 mt-4">
          <div className="bg-slate-900/80 p-3.5 rounded-2xl border border-slate-800">
            <p className="text-slate-400 text-[9px] uppercase font-bold tracking-wider mb-1">Today's App Profit (10%)</p>
            <h3 className="text-xl font-black text-emerald-400">₹{todaysCommission.toFixed(1)}</h3>
          </div>
          <div className="bg-slate-900/80 p-3.5 rounded-2xl border border-slate-800">
            <p className="text-slate-400 text-[9px] uppercase font-bold tracking-wider mb-1">Total Lifetime Profit</p>
            <h3 className="text-xl font-black text-slate-100">₹{totalCommission.toFixed(1)}</h3>
          </div>
        </div>
      </div>

      {/* ADMIN CONTROLS */}
      <div className="bg-slate-900 border border-slate-800 p-4 rounded-2xl space-y-3">
        <div className="flex justify-between items-center border-b border-slate-800 pb-2">
          <h4 className="text-[10px] font-black text-slate-300 uppercase tracking-widest">Platform Fast Tools</h4>
          <span className="text-[9px] text-emerald-400 bg-emerald-500/10 px-2 rounded-full font-bold">Active Hub</span>
        </div>
        <p className="text-[11px] text-slate-400">Testing ko aasan banane ke liye, naya pending order create karein jo available list me dikhega:</p>
        <button 
          onClick={injectMockOrders} 
          className="w-full bg-blue-600 hover:bg-blue-700 text-white font-extrabold py-2.5 rounded-xl text-xs transition flex justify-center items-center gap-1.5 shadow-lg shadow-blue-500/10"
        >
          <Sparkles size={14} /> Inject Deoghar Mock Order
        </button>
      </div>

      <div className="flex justify-between items-center px-1">
        <h3 className="text-[10px] font-black text-slate-400 uppercase tracking-wider">All Delivered Transactions Logs (10% Cut List)</h3>
        <span className="text-[10px] bg-slate-800 border border-slate-700 text-slate-300 px-2 py-0.5 rounded-full font-bold">{deliveredOrders.length} Completed</span>
      </div>

      <div className="space-y-3">
        {deliveredOrders.slice(0, 15).map(o => (
          <div key={o.id} className="bg-slate-900 p-4 rounded-2xl border border-slate-800 flex justify-between items-center hover:bg-slate-900/80 transition animate-slide-up">
            <div className="space-y-1">
              <p className="text-xs font-extrabold text-slate-100">{o.item}</p>
              <p className="text-[10px] text-slate-400 leading-tight">{o.pickup} ➔ {o.drop}</p>
              <p className="text-[9px] text-slate-500">{new Date(o.createdAt).toLocaleDateString()} • {o.distance} KM • {o.weight} KG</p>
            </div>
            <div className="text-right shrink-0">
              <p className="text-xs font-black text-emerald-400">+ ₹{o.appCommission.toFixed(1)}</p>
              <p className="text-[9px] text-slate-500">Gross: ₹{o.totalAmount}</p>
            </div>
          </div>
        ))}
        {deliveredOrders.length === 0 && (
          <div className="text-center py-10 bg-slate-900 border border-slate-800 rounded-3xl text-slate-500">
            <ShieldAlert size={36} className="mx-auto mb-2 text-slate-700" />
            <p className="text-xs text-slate-400">Abhi tak koi complete deliveries completed nahi hui hain.</p>
          </div>
        )}
      </div>
    </div>
  );
}

function OrderCard({ order, role, user, onReject, triggerLocalAlert, transporterProfile }) {
  const [otpInput, setOtpInput] = useState('');
  const [rating, setRating] = useState(5);
  const [reviewTag, setReviewTag] = useState('Delivery fast thi.');
  const [isSubmittingRating, setIsSubmittingRating] = useState(false);
  const [error, setError] = useState('');

  // Auto-cancel countdown calculator
  const [timeLeft, setTimeLeft] = useState(300); // 5 mins total
  useEffect(() => {
    if (order.status !== 'pending') return;
    
    const calculateTime = () => {
      const elapsed = Math.floor((Date.now() - order.createdAt) / 1000);
      const remaining = Math.max(0, 300 - elapsed);
      setTimeLeft(remaining);
    };

    calculateTime();
    const interval = setInterval(calculateTime, 1000);
    return () => clearInterval(interval);
  }, [order]);

  const minutes = Math.floor(timeLeft / 60);
  const seconds = timeLeft % 60;

  const handleAccept = async () => {
    const orderRef = doc(db, 'artifacts', appId, 'public', 'data', 'lt_orders', order.id);
    await updateDoc(orderRef, { 
      status: 'accepted', 
      transporterId: user.uid 
    });
    triggerLocalAlert("Delivery Accepted", `Order ${order.id} accept ho gaya hai!`);
  };

  const handleComplete = async () => {
    if (otpInput !== order.otp) {
      setError("Galat OTP! Customer se sahi verification code lein.");
      return;
    }
    setError('');
    const orderRef = doc(db, 'artifacts', appId, 'public', 'data', 'lt_orders', order.id);
    await updateDoc(orderRef, { status: 'delivered' });
    triggerLocalAlert("Delivered Safely", `Order ${order.id} successfully deliver ho gaya hai!`);
  };

  const submitRatingFeedback = async () => {
    setIsSubmittingRating(true);
    try {
      const orderRef = doc(db, 'artifacts', appId, 'public', 'data', 'lt_orders', order.id);
      
      // Update the average rating on Transporter profile safely
      if (order.transporterId && transporterProfile) {
        const currentRating = transporterProfile.rating || 5.0;
        const currentDeliveries = transporterProfile.deliveries || 0;
        const newDeliveries = currentDeliveries + 1;
        const calculatedRating = parseFloat(((currentRating * currentDeliveries + rating) / newDeliveries).toFixed(1));

        const transRef = doc(db, 'artifacts', appId, 'public', 'data', 'lt_users', order.transporterId);
        await updateDoc(transRef, {
          rating: calculatedRating,
          deliveries: newDeliveries
        });
      }

      await updateDoc(orderRef, { 
        rated: true, 
        clientRating: rating,
        clientReview: reviewTag
      });
      
      triggerLocalAlert("Rating Submitted", "Aapka valuable feedback save kar liya gaya hai.");
    } catch (e) {
      console.error(e);
    }
    setIsSubmittingRating(false);
  };

  return (
    <div className="bg-slate-900 p-4 rounded-3xl border border-slate-800 shadow-xl space-y-3 relative overflow-hidden transition hover:border-slate-700/60 duration-300">
      
      {/* Top Details & Header Row */}
      <div className="flex justify-between items-start">
        <div>
          <span className="text-[10px] text-slate-500 font-mono font-bold block">{order.id}</span>
          <h4 className="font-extrabold text-white text-sm flex items-center gap-1">
            <Package size={16} className="text-blue-500 shrink-0"/> {order.item}
          </h4>
          <p className="text-[11px] text-slate-400 mt-0.5">{order.weight} KG Cargo • {order.distance} KM Logistics Distance</p>
        </div>
        <StatusBadge status={order.status} />
      </div>

      {/* Auto Cancel Info Timer for client/transporter */}
      {order.status === 'pending' && (
        <div className="bg-amber-500/10 border border-amber-500/20 rounded-xl px-3 py-2 flex items-center justify-between text-[11px] text-amber-400">
          <span className="flex items-center gap-1.5 font-bold">
            <Clock size={12} className="animate-spin text-amber-400" /> Finding nearby transporters...
          </span>
          <span className="font-mono font-black text-xs bg-amber-500/20 px-2 py-0.5 rounded">
            {minutes}:{seconds < 10 ? '0' : ''}{seconds}
          </span>
        </div>
      )}

      {/* Show cancellation reason if any */}
      {order.status === 'cancelled' && (
        <div className="bg-rose-500/10 border border-rose-500/20 rounded-xl p-3 text-[11px] text-rose-400 font-bold flex items-center gap-1.5 animate-pulse">
          <AlertCircle size={14} className="shrink-0" /> {order.cancelReason || "Order Cancelled"}
        </div>
      )}

      {/* LOCATIONS ROAD MAP SIMULATOR VISUALIZER */}
      {order.status === 'accepted' && (
        <div className="bg-slate-950 p-3 rounded-2xl border border-slate-850 space-y-1.5">
          <p className="text-[9px] font-black uppercase text-indigo-400 tracking-wider flex items-center gap-1">
            <Map size={11} /> Visual Route Mapping (Live GPS)
          </p>
          {/* Animated SVG Path for GPS Visualizer */}
          <div className="relative h-16 w-full bg-slate-900 rounded-xl overflow-hidden border border-slate-800">
            <svg className="w-full h-full" viewBox="0 0 300 64">
              {/* Pickup point dot */}
              <circle cx="30" cy="32" r="4" fill="#10B981" />
              <text x="38" y="35" fill="#10B981" fontSize="8" fontWeight="bold">Pickup</text>
              
              {/* Drop point dot */}
              <circle cx="260" cy="32" r="4" fill="#EF4444" />
              <text x="210" y="35" fill="#EF4444" fontSize="8" fontWeight="bold">Destination</text>

              {/* Connecting Road path */}
              <path d="M 30,32 Q 145,10 260,32" fill="none" stroke="#374151" strokeWidth="2" strokeDasharray="4" />
              
              {/* Animated delivery vehicle moving path */}
              <circle cx="0" cy="0" r="6" fill="#3B82F6" className="animate-path-move">
                <animateMotion 
                  path="M 30,32 Q 145,10 260,32" 
                  dur="8s" 
                  repeatCount="indefinite" 
                />
              </circle>
            </svg>
            <div className="absolute bottom-1 right-2 text-[8px] text-slate-500 font-black tracking-widest animate-pulse">
              LIVE TRACKING ON
            </div>
          </div>
        </div>
      )}

      {/* Locations mapping */}
      <div className="relative pl-6 py-1 space-y-3.5 border-l border-slate-800 ml-2">
        <div className="relative">
          <div className="absolute -left-[21px] top-0.5"><div className="w-2.5 h-2.5 rounded-full bg-emerald-500 ring-4 ring-emerald-500/20"></div></div>
          <p className="text-[9px] font-black text-slate-500 uppercase tracking-wide">Pickup Point</p>
          <p className="text-xs font-semibold text-slate-200 leading-tight">{order.pickup}</p>
        </div>
        <div className="relative">
          <div className="absolute -left-[21px] top-0.5"><div className="w-2.5 h-2.5 rounded-full bg-rose-500 ring-4 ring-rose-500/20"></div></div>
          <p className="text-[9px] font-black text-slate-500 uppercase tracking-wide">Drop Location</p>
          <p className="text-xs font-semibold text-slate-200 leading-tight">{order.drop}</p>
        </div>
      </div>

      {/* Dynamic Client UI interactions */}
      {role === 'client' && (
        <div className="border-t border-slate-850 pt-3 mt-2">
          {order.status === 'accepted' && (
            <div className="bg-indigo-600/10 p-3 rounded-2xl border border-indigo-500/20 text-center space-y-2">
              <p className="text-[9px] text-indigo-300 font-black uppercase tracking-widest">🔑 Safe Handshake OTP (Share only at delivery destination)</p>
              <p className="text-3xl font-mono font-black tracking-widest text-indigo-400">{order.otp}</p>
              {transporterProfile && (
                <div className="mt-2 text-xs flex justify-between items-center bg-slate-950 rounded-xl p-2.5 border border-slate-800">
                  <div className="text-left">
                    <span className="font-extrabold text-slate-200 text-[11px] block">{transporterProfile.name}</span>
                    <span className="text-[10px] text-slate-400">{transporterProfile.vehicleType} • ⭐ {transporterProfile.rating || '5.0'}</span>
                  </div>
                  <a href={`tel:${transporterProfile.mobile}`} className="bg-blue-600 hover:bg-blue-700 text-white p-2 rounded-xl transition flex items-center justify-center">
                    <Phone size={14} />
                  </a>
                </div>
              )}
            </div>
          )}

          {order.status === 'delivered' && !order.rated && (
            <div className="bg-slate-950 p-3 rounded-2xl border border-slate-800 space-y-3 mt-2 animate-slide-up">
              <p className="text-xs font-black text-slate-200 flex items-center gap-1">
                <Star size={14} className="text-amber-400 fill-current" /> Rate Transporter Experience
              </p>
              
              {/* Star selector */}
              <div className="flex gap-2 justify-center py-2 bg-slate-900 rounded-xl border border-slate-850">
                {[1, 2, 3, 4, 5].map(starVal => (
                  <button key={starVal} onClick={() => setRating(starVal)} className="p-1 hover:scale-110 transition duration-200">
                    <Star size={24} className={starVal <= rating ? "text-amber-400 fill-current" : "text-slate-700"} />
                  </button>
                ))}
              </div>

              {/* Quick Tag Selectors */}
              <div className="grid grid-cols-2 gap-1.5 text-[10px]">
                <button type="button" onClick={() => setReviewTag('Delivery fast thi.')} className={`p-1.5 rounded-lg border text-center font-bold ${reviewTag === 'Delivery fast thi.' ? 'bg-indigo-600/20 border-indigo-500 text-white' : 'bg-slate-900 border-slate-800 text-slate-400'}`}>⚡ Fast Delivery</button>
                <button type="button" onClick={() => setReviewTag('Transporter polite tha.')} className={`p-1.5 rounded-lg border text-center font-bold ${reviewTag === 'Transporter polite tha.' ? 'bg-indigo-600/20 border-indigo-500 text-white' : 'bg-slate-900 border-slate-800 text-slate-400'}`}>🤝 Polite Driver</button>
                <button type="button" onClick={() => setReviewTag('Package bilkul safe tha.')} className={`p-1.5 rounded-lg border text-center font-bold ${reviewTag === 'Package bilkul safe tha.' ? 'bg-indigo-600/20 border-indigo-500 text-white' : 'bg-slate-900 border-slate-800 text-slate-400'}`}>🛡️ Safe Cargo</button>
                <button type="button" onClick={() => setReviewTag('Late delivery thi.')} className={`p-1.5 rounded-lg border text-center font-bold ${reviewTag === 'Late delivery thi.' ? 'bg-indigo-600/20 border-indigo-500 text-white' : 'bg-slate-900 border-slate-800 text-slate-400'}`}>🕒 Late Delivery</button>
              </div>

              <button 
                onClick={submitRatingFeedback} 
                disabled={isSubmittingRating} 
                className="w-full bg-indigo-600 hover:bg-indigo-700 text-white text-xs font-black py-2 rounded-xl transition shadow-md shadow-indigo-500/10"
              >
                {isSubmittingRating ? 'Saving...' : 'Submit Rating & Feedback'}
              </button>
            </div>
          )}

          {order.status === 'delivered' && order.rated && (
            <div className="bg-emerald-500/10 p-2.5 rounded-xl border border-emerald-500/20 flex items-center justify-between text-xs text-emerald-400">
              <span className="font-extrabold flex items-center gap-1"><Check size={14} /> Rated successfully</span>
              <span className="font-mono bg-emerald-500/20 px-2 py-0.5 rounded text-[10px] font-black flex items-center gap-0.5">⭐ {order.clientRating}</span>
            </div>
          )}
        </div>
      )}

      {/* Dynamic Transporter UI actions */}
      {role === 'transporter' && (
        <div className="mt-3 pt-3 border-t border-slate-850 space-y-2">
          <div className="flex justify-between items-center bg-slate-950 p-2.5 rounded-xl">
            <p className="text-[10px] text-slate-400 font-bold uppercase">Net Driver Share (90%)</p>
            <p className="text-base font-black text-emerald-400">₹{order.transporterEarning.toFixed(1)}</p>
          </div>
          
          {order.status === 'pending' && (
            <div className="flex gap-2">
              <button 
                onClick={() => onReject(order.id)} 
                className="flex-1 py-2.5 border border-rose-500/20 hover:bg-rose-500/10 text-rose-400 font-bold text-xs rounded-xl transition flex justify-center items-center gap-1"
              >
                <X size={14} /> Reject
              </button>
              <button 
                onClick={handleAccept} 
                className="flex-1 bg-gradient-to-r from-blue-600 to-indigo-700 hover:from-blue-700 hover:to-indigo-800 text-white font-black py-2.5 text-xs rounded-xl shadow-md transition flex justify-center items-center gap-1"
              >
                <Check size={14} /> Accept cargo
              </button>
            </div>
          )}

          {order.status === 'accepted' && (
            <div className="bg-slate-950 p-3 rounded-2xl border border-slate-850 space-y-2">
              <div className="flex justify-between items-center">
                <span className="text-[10px] font-black text-slate-300 uppercase">Enter Handshake Delivery OTP</span>
                <a href={`tel:${order.clientMobile}`} className="text-blue-400 hover:text-blue-300 flex items-center gap-0.5 text-xs font-bold">
                  <Phone size={12} /> Call Customer
                </a>
              </div>
              <div className="flex gap-2">
                <input 
                  type="text" 
                  maxLength={4}
                  value={otpInput}
                  onChange={(e) => setOtpInput(e.target.value)}
                  placeholder="4-digit OTP" 
                  className="flex-1 px-4 py-2 bg-slate-900 border border-slate-800 rounded-xl text-center font-mono font-black text-base tracking-widest outline-none focus:ring-2 focus:ring-emerald-500 text-white"
                />
                <button onClick={handleComplete} className="bg-emerald-600 hover:bg-emerald-700 text-white px-4 rounded-xl font-bold text-xs">
                  Verify & Finish
                </button>
              </div>
              {error && <p className="text-rose-400 text-[10px] mt-1 font-bold">{error}</p>}
            </div>
          )}
        </div>
      )}
    </div>
  );
}

function StatusBadge({ status }) {
  const styles = {
    pending: 'bg-amber-500/10 text-amber-400 border-amber-500/20',
    accepted: 'bg-blue-500/10 text-blue-400 border-blue-500/20',
    delivered: 'bg-emerald-500/10 text-emerald-400 border-emerald-500/20',
    cancelled: 'bg-rose-500/10 text-rose-400 border-rose-500/20'
  };
  return (
    <span className={`text-[9px] uppercase font-black px-2.5 py-0.5 rounded-full border tracking-widest shrink-0 ${styles[status]}`}>
      {status === 'pending' ? '🔍 Searching' : status === 'accepted' ? '🚚 Active Route' : status}
    </span>
  );
}
