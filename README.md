#import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  deleteDoc, 
  doc, 
  serverTimestamp 
} from 'firebase/firestore';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, 
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  Plus, 
  Trash2, 
  Wallet, 
  ArrowRightLeft, 
  Clock,
  CheckCircle2,
  Send,
  User,
  Sun,
  History
} from 'lucide-react';

// --- איורים מינימליסטיים ---

const PyramidIcon = () => (
  <svg viewBox="0 0 100 100" className="w-8 h-8 drop-shadow-sm">
    <path d="M50 15 L85 85 L15 85 Z" className="fill-amber-500" />
    <path d="M50 15 L85 85 L65 90 L50 15" className="fill-amber-600" />
  </svg>
);

const CamelIcon = () => (
  <svg viewBox="0 0 100 100" className="w-8 h-8 drop-shadow-sm">
    <path 
      d="M20 75 L25 75 Q27 60 30 60 L35 60 Q40 45 50 45 Q55 45 60 50 Q70 45 80 50 Q90 50 93 40 Q95 35 100 35 L100 45 Q95 45 93 55 Q90 65 80 65 L70 65 Q70 75 65 85 L60 85 L60 65 L45 65 L45 85 L40 85 L40 65 L25 65 L20 75 Z" 
      className="fill-orange-600" 
    />
  </svg>
);

// הגדרות Firebase
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'egypt-trip-2026';

const FRIENDS = [
  'בני ממטרה',
  'יוסי הגבר',
  'מוחמד אליעזר',
  'שמעון הצדיק',
  'מנדי הסובל'
];

export default function App() {
  const [user, setUser] = useState(null);
  const [expenses, setExpenses] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  const [inputs, setInputs] = useState(
    FRIENDS.reduce((acc, name) => ({
      ...acc, 
      [name]: { amount: '', description: '' }
    }), {})
  );

  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          try { await signInWithCustomToken(auth, __initial_auth_token); } 
          catch (e) { await signInAnonymously(auth); }
        } else { await signInAnonymously(auth); }
      } catch (err) { setError("שגיאת חיבור"); }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    const q = collection(db, 'artifacts', appId, 'public', 'data', 'expenses');
    const unsubscribe = onSnapshot(q, (snapshot) => {
      const data = snapshot.docs.map(doc => ({
        id: doc.id, ...doc.data()
      })).sort((a, b) => (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0));
      setExpenses(data);
      setLoading(false);
    }, () => {
      setError("שגיאת טעינה");
      setLoading(false);
    });
    return () => unsubscribe();
  }, [user]);

  const calculations = useMemo(() => {
    const totals = {};
    FRIENDS.forEach(f => totals[f] = 0);
    let grandTotal = 0;
    expenses.forEach(exp => {
      const val = parseFloat(exp.amount) || 0;
      totals[exp.payer] += val;
      grandTotal += val;
    });
    const average = grandTotal / FRIENDS.length;
    const balances = FRIENDS.map(name => ({
      name,
      balance: totals[name] - average,
      totalPaid: totals[name]
    }));
    const settlement = [];
    const debtors = balances.filter(b => b.balance < -0.01).sort((a, b) => a.balance - b.balance);
    const creditors = balances.filter(b => b.balance > 0.01).sort((a, b) => b.balance - a.balance);
    let dIdx = 0, cIdx = 0;
    const tD = debtors.map(d => ({ ...d })), tC = creditors.map(c => ({ ...c }));
    while (dIdx < tD.length && cIdx < tC.length) {
      const val = Math.min(Math.abs(tD[dIdx].balance), tC[cIdx].balance);
      settlement.push({ from: tD[dIdx].name, to: tC[cIdx].name, amount: val });
      tD[dIdx].balance += val; tC[cIdx].balance -= val;
      if (Math.abs(tD[dIdx].balance) < 0.01) dIdx++;
      if (Math.abs(tC[cIdx].balance) < 0.01) cIdx++;
    }
    return { grandTotal, average, balances, settlement };
  }, [expenses]);

  const handleInputChange = (name, field, value) => {
    setInputs(prev => ({ ...prev, [name]: { ...prev[name], [field]: value } }));
  };

  const addExpense = async (name) => {
    const { amount, description } = inputs[name];
    if (!amount || !description || !user) return;
    try {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'expenses'), {
        amount: parseFloat(amount),
        description,
        payer: name,
        createdAt: serverTimestamp(),
        userId: user.uid
      });
      setInputs(prev => ({ ...prev, [name]: { amount: '', description: '' } }));
    } catch (err) { setError("נכשל בעדכון"); }
  };

  const deleteExpense = async (id) => {
    try { await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'expenses', id)); } 
    catch (err) { setError("מחיקה נכשלה"); }
  };

  const ExpenseCard = ({ name }) => (
    <div key={name} className="bg-amber-400 rounded-[2.5rem] shadow-xl p-4 flex flex-col border-b-8 border-amber-600 w-[calc(50%-0.5rem)] sm:w-56 lg:w-64 transition-transform active:scale-95 m-1">
      <div className="text-center mb-4 bg-white/30 py-3 rounded-[1.5rem] shadow-inner border border-white/20">
        <h3 className="text-3xl font-serif font-black text-slate-900 tracking-tight leading-none drop-shadow-sm italic">
          {name}
        </h3>
      </div>
      
      <div className="space-y-3">
        <div className="relative">
          <input 
            type="number" 
            placeholder="סכום ₪"
            inputMode="decimal"
            className="w-full py-3 px-4 bg-white rounded-xl border-none focus:ring-4 focus:ring-orange-500 outline-none text-lg font-black text-slate-900 shadow-inner"
            value={inputs[name].amount}
            onChange={(e) => handleInputChange(name, 'amount', e.target.value)}
          />
        </div>
        <input 
          type="text" 
          placeholder="עבור מה?"
          className="w-full py-3 px-4 bg-white/80 rounded-xl border-none focus:ring-4 focus:ring-orange-500 outline-none text-xs font-bold text-slate-700 shadow-inner"
          value={inputs[name].description}
          onChange={(e) => handleInputChange(name, 'description', e.target.value)}
        />
        <button 
          onClick={() => addExpense(name)}
          disabled={!inputs[name].amount || !inputs[name].description}
          className="w-full py-3.5 bg-slate-950 hover:bg-black disabled:bg-slate-300 disabled:text-slate-500 text-amber-400 text-xs font-black rounded-xl transition-all shadow-lg flex items-center justify-center gap-2"
        >
          <Send className="w-4 h-4" />
          עדכן
        </button>
      </div>
    </div>
  );

  if (loading) return (
    <div className="flex items-center justify-center h-screen bg-sky-50 text-amber-800">
      <div className="flex flex-col items-center gap-4 animate-pulse">
        <Sun className="w-12 h-12 text-orange-500 animate-spin" />
        <p className="text-sm font-bold tracking-widest uppercase">מעמיסים את הגמלים...</p>
      </div>
    </div>
  );

  return (
    <div className="min-h-screen bg-sky-50 text-slate-900 font-sans pb-12 overflow-x-hidden" dir="rtl">
      
      <header className="bg-gradient-to-b from-sky-500 to-sky-400 text-white px-6 pt-10 pb-12 rounded-b-[3.5rem] shadow-xl text-center relative">
        <div className="relative z-10">
          <div className="flex justify-center gap-4 mb-4">
            <PyramidIcon />
            <CamelIcon />
          </div>
          <h1 className="text-4xl font-black tracking-tight drop-shadow-md">
            חוזרים למצרים <span className="text-amber-300 italic font-black">2026</span>
          </h1>
          <p className="text-xs font-bold uppercase tracking-[0.2em] opacity-80 mt-1 text-sky-100">יומן מסע כספי משותף</p>
        </div>
      </header>

      <main className="px-3 max-w-7xl mx-auto -mt-6 space-y-12">
        
        <section className="flex flex-col items-center w-full">
          <div className="flex items-center gap-2 mb-6 px-1 opacity-70">
            <Plus className="w-4 h-4 text-sky-800" />
            <h2 className="text-sm font-black uppercase tracking-widest">עדכון הוצאה אישי</h2>
          </div>
          
          <div className="w-full max-w-6xl">
            <div className="flex flex-wrap justify-center gap-3 mb-3">
              {FRIENDS.slice(0, 3).map(name => (
                <ExpenseCard key={name} name={name} />
              ))}
            </div>
            
            <div className="flex flex-wrap justify-center gap-3">
              {FRIENDS.slice(3, 5).map(name => (
                <ExpenseCard key={name} name={name} />
              ))}
            </div>
          </div>
        </section>

        <div className="max-w-4xl mx-auto space-y-8">
          
          <section className="bg-gradient-to-br from-orange-500 to-amber-600 p-8 rounded-[3rem] text-white shadow-2xl relative overflow-hidden border-2 border-white/20">
            <div className="relative z-10">
              <div className="flex flex-col sm:flex-row justify-between items-center gap-6 mb-10">
                <div className="text-center sm:text-right">
                  <h2 className="flex items-center justify-center sm:justify-start gap-3 text-2xl font-black">
                    <Wallet className="w-7 h-7 text-amber-200" />
                    מצב הקופה
                  </h2>
                  <p className="text-white/60 text-xs font-bold mt-1 uppercase tracking-tight">ממוצע לאדם: ₪{calculations.average.toFixed(0)}</p>
                </div>
                <div className="bg-white/20 backdrop-blur-xl px-8 py-4 rounded-[2rem] border border-white/10 text-center shadow-inner">
                  <p className="text-white/40 text-[10px] font-black uppercase tracking-widest mb-1">סה"כ כללי</p>
                  <p className="text-4xl font-black tracking-tighter text-white">₪{calculations.grandTotal.toLocaleString()}</p>
                </div>
              </div>

              <div className="flex flex-wrap justify-center gap-3">
                {calculations.balances.map(b => (
                  <div key={b.name} className="bg-black/15 p-4 rounded-3xl border border-white/5 text-center flex flex-col justify-between min-h-[110px] w-32 backdrop-blur-sm">
                    <p className="text-[11px] font-serif italic text-white/70 font-black truncate mb-1 leading-none">{b.name}</p>
                    <p className="text-xl font-black text-white">₪{b.totalPaid.toLocaleString()}</p>
                    <div className={`text-[9px] font-black px-2 py-1 rounded-lg mt-2 inline-block self-center ${b.balance >= 0 ? 'bg-sky-400 text-sky-950 shadow-sm' : 'bg-white/10 text-white/50'}`}>
                      {b.balance >= 0 ? `+${b.balance.toFixed(0)}` : b.balance.toFixed(0)}
                    </div>
                  </div>
                ))}
              </div>
            </div>
          </section>

          <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
            
            <section className="bg-white p-6 rounded-[3rem] shadow-xl border border-sky-100 flex flex-col items-center">
              <h2 className="flex items-center gap-3 text-lg font-black mb-6 text-slate-800 uppercase tracking-tighter">
                <ArrowRightLeft className="w-5 h-5 text-sky-600" />
                סיכום חובות
              </h2>
              <div className="w-full space-y-3">
                {calculations.settlement.length === 0 ? (
                  <div className="text-center py-10 bg-sky-50 rounded-3xl border border-dashed border-sky-200">
                    <CheckCircle2 className="w-10 h-10 text-sky-300 mx-auto mb-2 opacity-50" />
                    <p className="text-sky-800 font-bold text-xs italic">אין חובות, כולם פיטים!</p>
                  </div>
                ) : (
                  calculations.settlement.map((s, i) => (
                    <div key={i} className="flex flex-col p-4 bg-sky-50/50 rounded-[2rem] border border-sky-100 transition-all hover:bg-white hover:shadow-md text-center">
                      <div className="flex items-center justify-between mb-2">
                        <span className="text-[10px] font-serif italic font-black text-slate-900 bg-white px-3 py-1.5 rounded-xl shadow-sm border border-sky-50">{s.from}</span>
                        <ArrowRightLeft className="w-3.5 h-3.5 text-sky-300" />
                        <span className="text-[10px] font-serif italic font-black text-white bg-sky-600 px-3 py-1.5 rounded-xl shadow-md">{s.to}</span>
                      </div>
                      <p className="text-slate-800 text-2xl font-black tracking-tighter leading-none mt-1">₪{s.amount.toFixed(0)}</p>
                    </div>
                  ))
                )}
              </div>
            </section>

            <section className="bg-white rounded-[3rem] shadow-xl border border-sky-100 overflow-hidden flex flex-col">
              <div className="p-6 border-b border-sky-50 flex items-center justify-between bg-sky-50/20">
                <h2 className="flex items-center gap-2 text-xs font-black text-slate-500 uppercase tracking-widest">
                  <History className="w-4 h-4" />
                  היסטוריה
                </h2>
                <div className="bg-slate-900 text-amber-400 px-3 py-1 rounded-full text-[10px] font-black shadow-sm">
                  {expenses.length}
                </div>
              </div>
              <div className="flex-1 overflow-y-auto max-h-[400px] p-4 space-y-3">
                {expenses.length === 0 ? (
                  <div className="text-center py-20 text-slate-300 text-xs italic">אין תנועות...</div>
                ) : (
                  expenses.map((exp) => (
                    <div key={exp.id} className="flex items-center justify-between p-3 bg-sky-50/30 hover:bg-white rounded-2xl border border-transparent hover:border-sky-100 transition-all group">
                      <div className="flex-1 min-w-0 pr-1">
                        <p className="font-black text-slate-800 text-sm truncate mb-0.5 tracking-tight leading-tight">{exp.description}</p>
                        <div className="flex items-center gap-1.5 text-[9px] text-amber-700 font-serif italic font-black uppercase">
                          <User className="w-3 h-3 opacity-50" />
                          {exp.payer}
                        </div>
                      </div>
                      <div className="flex items-center gap-3">
                        <span className="font-black text-slate-900 text-base tracking-tighter">₪{exp.amount}</span>
                        <button onClick={() => deleteExpense(exp.id)} className="p-2 text-slate-200 hover:text-red-500 transition-all opacity-0 group-hover:opacity-100">
                          <Trash2 className="w-4 h-4" />
                        </button>
                      </div>
                    </div>
                  ))
                )}
              </div>
            </section>
          </div>
        </div>
      </main>

      <footer className="text-center mt-12 py-12 text-slate-400 text-[10px] font-black uppercase tracking-[0.4em] px-6">
        <p>© 2026 חוזרים למצרים | {appId}</p>
      </footer>
    </div>
  );
}
