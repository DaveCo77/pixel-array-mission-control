<!-- src/App.vue (Mission Control Master Shell) -->
<script setup>
import { ref, onMounted } from 'vue';
import { createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut, onAuthStateChanged } from 'firebase/auth';
import { doc, setDoc, getDoc } from 'firebase/firestore';
import { auth, db } from './firebase';

// Import your Dave-Co Modules
import LedWallBuilder from './modules/LedWallBuilder.vue';
import TimerCo from './modules/TimerCo.vue';

// --- NAVIGATION & EVENT WORKSPACE STATE ---
const activeTab = ref('livedesk'); // 'livedesk' | 'timer' | 'ledwall'
const activeEventId = ref('crown_gala_2026');
const signalFeed = ref([
  { id: 1, time: '08:00:00', level: 'info', msg: 'Mission Control initialized. Connected to Event Room.' }
]);

// Receives automatic 60s warning & 0s overtime alerts emitted by TimerCo.vue!
const handleTimerAlert = (alertObj) => {
  signalFeed.value.unshift({
    id: Date.now(),
    time: new Date().toTimeString().slice(0, 8),
    level: alertObj.level,
    msg: alertObj.msg
  });
};

// --- YOUR EXISTING FIREBASE AUTH LOGIC ---
const currentUser = ref(null);
const accountLevel = ref('free');
const showAuthModal = ref(false);
const authMode = ref('signup');
const authEmail = ref('');
const authPassword = ref('');
const authError = ref('');

onMounted(() => {
  onAuthStateChanged(auth, async (user) => {
    if (user) {
      currentUser.value = user;
      const docRef = doc(db, "users", user.uid);
      const docSnap = await getDoc(docRef);
      accountLevel.value = docSnap.exists() ? (docSnap.data().accountLevel || 'free') : 'free';
    } else {
      currentUser.value = null;
      accountLevel.value = 'free';
    }
  });
});

const handleAuthSubmit = async () => {
  authError.value = '';
  try {
    if (authMode.value === 'signup') {
      const cred = await createUserWithEmailAndPassword(auth, authEmail.value, authPassword.value);
      await setDoc(doc(db, "users", cred.user.uid), {
        email: authEmail.value,
        accountLevel: 'free',
        createdAt: new Date().toISOString()
      });
    } else {
      await signInWithEmailAndPassword(auth, authEmail.value, authPassword.value);
    }
    showAuthModal.value = false;
    authEmail.value = '';
    authPassword.value = '';
  } catch (err) {
    authError.value = err.message;
  }
};

const handleLogout = async () => {
  await signOut(auth);
};

const openDsmPopup = () => {
  window.open(`/dsm.html?event=${activeEventId.value}`, '_blank', 'width=1280,height=720');
};
</script>

<template>
  <div class="flex h-screen w-screen bg-slate-950 text-slate-100 overflow-hidden font-sans">

    <!-- FIREBASE AUTH MODAL -->
    <div v-if="showAuthModal" class="fixed inset-0 bg-black/80 backdrop-blur-sm flex items-center justify-center z-50">
      <div class="bg-slate-900 border border-slate-800 rounded-xl p-6 w-full max-w-sm shadow-2xl">
        <h2 class="text-lg font-bold text-center mb-4">{{ authMode === 'signup' ? 'Create Pixel Array Account' : 'Welcome Back' }}</h2>
        <div class="flex gap-2 mb-4 border-b border-slate-800 pb-2">
          <button @click="authMode = 'signup'" :class="authMode === 'signup' ? 'text-cyan-400 border-b-2 border-cyan-400' : 'text-slate-500'" class="flex-1 pb-1 font-bold text-xs">Sign Up</button>
          <button @click="authMode = 'login'" :class="authMode === 'login' ? 'text-cyan-400 border-b-2 border-cyan-400' : 'text-slate-500'" class="flex-1 pb-1 font-bold text-xs">Log In</button>
        </div>
        <input type="email" v-model="authEmail" placeholder="Email Address" class="w-full bg-slate-950 border border-slate-800 rounded px-3 py-2 text-xs text-white mb-2.5 outline-none focus:border-cyan-500">
        <input type="password" v-model="authPassword" placeholder="Password" class="w-full bg-slate-950 border border-slate-800 rounded px-3 py-2 text-xs text-white mb-3 outline-none focus:border-cyan-500">
        <div v-if="authError" class="text-rose-400 text-xs text-center mb-3">{{ authError }}</div>
        <button @click="handleAuthSubmit" class="w-full bg-cyan-600 hover:bg-cyan-500 text-white font-bold py-2.5 rounded text-xs transition">
          {{ authMode === 'signup' ? 'Create Free Account' : 'Log In' }}
        </button>
        <button @click="showAuthModal = false" class="w-full text-slate-500 hover:text-slate-300 text-xs mt-3">Cancel</button>
      </div>
    </div>

    <!-- LEFT MISSION CONTROL DOCK -->
    <aside class="w-60 bg-[#0b0f19] border-r border-slate-800 flex flex-col justify-between shrink-0 z-20">
      <div>
        <!-- Brand Header -->
        <div class="p-4 border-b border-slate-800">
          <div class="flex items-center gap-2">
            <span class="w-2.5 h-2.5 rounded-full bg-cyan-400 shadow-[0_0_10px_#22d3ee]"></span>
            <span class="font-black tracking-widest text-xs uppercase text-white">PIXEL ARRAY</span>
          </div>
          <p class="text-[10px] text-slate-500 font-mono mt-0.5">DAVE-CO MISSION CONTROL</p>
        </div>

        <!-- Active Event ID Room Selector -->
        <div class="p-3 border-b border-slate-800 bg-slate-900/40">
          <label class="text-[10px] uppercase tracking-wider text-slate-400 font-bold block mb-1">Active Event Room ID</label>
          <input v-model="activeEventId" class="w-full bg-slate-950 border border-slate-700 rounded px-2.5 py-1.5 text-xs font-mono text-cyan-300 focus:border-cyan-400 outline-none" placeholder="e.g. crown_gala_2026" />
        </div>

        <!-- Suite Module Switcher -->
        <nav class="p-2 space-y-1">
          <button @click="activeTab = 'livedesk'"
                  :class="activeTab === 'livedesk' ? 'bg-cyan-500/15 text-cyan-300 border border-cyan-500/30' : 'text-slate-400 hover:bg-slate-900 hover:text-white border border-transparent'"
                  class="w-full flex items-center justify-between px-3 py-2.5 rounded-lg text-xs font-bold transition">
            <span>⚡ Live Desk (Split)</span>
            <span class="text-[9px] bg-cyan-500/20 text-cyan-300 px-1.5 py-0.5 rounded">2x2</span>
          </button>

          <button @click="activeTab = 'timer'"
                  :class="activeTab === 'timer' ? 'bg-blue-500/15 text-blue-400 border border-blue-500/30' : 'text-slate-400 hover:bg-slate-900 hover:text-white border border-transparent'"
                  class="w-full flex items-center justify-between px-3 py-2.5 rounded-lg text-xs font-bold transition">
            <span>⏱️ TimerCo Control</span>
            <span class="text-[9px] bg-emerald-500/20 text-emerald-300 px-1.5 py-0.5 rounded">LIVE</span>
          </button>

          <button @click="activeTab = 'ledwall'"
                  :class="activeTab === 'ledwall' ? 'bg-purple-500/15 text-purple-300 border border-purple-500/30' : 'text-slate-400 hover:bg-slate-900 hover:text-white border border-transparent'"
                  class="w-full flex items-center justify-between px-3 py-2.5 rounded-lg text-xs font-bold transition">
            <span>🧱 LED Wall 3D</span>
            <span class="text-[9px] bg-emerald-500/20 text-emerald-300 px-1.5 py-0.5 rounded">v4.5</span>
          </button>
        </nav>
      </div>

      <!-- Bottom User & Quick DSM Pop-out -->
      <div class="p-3 border-t border-slate-800 space-y-2.5 bg-slate-950">
        <button @click="openDsmPopup" class="w-full py-2 px-3 bg-blue-600/20 hover:bg-blue-600/30 text-blue-300 border border-blue-500/30 rounded-lg text-xs font-bold transition flex items-center justify-center gap-1.5">
          <span>🖥️ Launch Stage DSM</span>
        </button>

        <div v-if="currentUser" class="flex items-center justify-between bg-slate-900 px-2.5 py-2 rounded border border-slate-800">
          <div class="truncate pr-2">
            <div class="text-[11px] font-bold text-white truncate">{{ currentUser.email }}</div>
            <span class="text-[9px] px-1.5 py-0.5 rounded font-bold uppercase" :class="accountLevel === 'pro' ? 'bg-purple-600 text-white' : 'bg-slate-700 text-slate-300'">
              {{ accountLevel }}
            </span>
          </div>
          <button @click="handleLogout" class="text-[10px] text-slate-400 hover:text-white border border-slate-700 px-2 py-1 rounded">Out</button>
        </div>
        <button v-else @click="showAuthModal = true" class="w-full py-2 bg-slate-800 hover:bg-slate-700 text-white rounded text-xs font-bold">
          Log In / Sign Up
        </button>
      </div>
    </aside>

    <!-- MAIN WORKSPACE VIEWPORT -->
    <main class="flex-1 h-full overflow-hidden relative">
      
      <!-- 1. LIVE DESK (Split View: TimerCo + SignalCo Feed + 3D Wall) -->
      <div v-if="activeTab === 'livedesk'" class="grid grid-cols-12 grid-rows-2 gap-3 p-3 h-full w-full bg-slate-950">
        <!-- Top Left (7 cols): Live TimerCo Embedded -->
        <div class="col-span-7 row-span-1 border border-slate-800 rounded-xl overflow-hidden shadow-xl">
          <TimerCo :eventId="activeEventId" :isEmbedded="true" @timer-alert="handleTimerAlert" />
        </div>

        <!-- Top Right (5 cols): SignalCo Live Comms Feed (Ready for your SignalCo code next!) -->
        <div class="col-span-5 row-span-1 bg-[#0b0b0b] border border-slate-800 rounded-xl p-4 flex flex-col justify-between">
          <div>
            <div class="flex items-center justify-between border-b border-white/10 pb-2 mb-3">
              <span class="text-xs font-black tracking-widest uppercase text-white">📡 Signal<span class="text-amber-400">Co</span> // Auto-Cue Feed</span>
              <span class="text-[10px] font-mono text-emerald-400">LINKED TO TIMERCO</span>
            </div>
            <div class="space-y-2 overflow-y-auto max-h-48 pr-1">
              <div v-for="item in signalFeed" :key="item.id"
                   class="p-2.5 rounded border text-xs font-mono flex items-start gap-2"
                   :class="item.level === 'crit' ? 'bg-rose-950/40 border-rose-500/40 text-rose-300' : item.level === 'warn' ? 'bg-amber-950/40 border-amber-500/40 text-amber-300' : 'bg-slate-900 border-slate-800 text-slate-300'">
                <span class="text-[10px] text-slate-500 shrink-0">{{ item.time }}</span>
                <span class="font-bold">{{ item.msg }}</span>
              </div>
            </div>
          </div>
          <div class="text-[11px] text-slate-500 italic">
            Drop your SignalCo code in next to unlock full two-way department cueing here.
          </div>
        </div>

        <!-- Bottom Half (12 cols): Live 3D LED Wall Builder -->
        <div class="col-span-12 row-span-1 border border-slate-800 rounded-xl overflow-hidden shadow-xl">
          <LedWallBuilder />
        </div>
      </div>

      <!-- 2. FULLSCREEN TIMERCO CONTROL -->
      <div v-else-if="activeTab === 'timer'" class="h-full w-full">
        <TimerCo :eventId="activeEventId" :isEmbedded="false" @timer-alert="handleTimerAlert" />
      </div>

      <!-- 3. FULLSCREEN 3D LED WALL BUILDER -->
      <div v-else-if="activeTab === 'ledwall'" class="h-full w-full">
        <LedWallBuilder />
      </div>

    </main>
  </div>
</template>
