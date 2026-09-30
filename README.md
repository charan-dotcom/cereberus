<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cereberus | Autonomous AI SOC Platform</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
        import { 
            getAuth, 
            signInWithEmailAndPassword, 
            createUserWithEmailAndPassword,
            signInWithPopup, 
            GoogleAuthProvider, 
            signOut,
            onAuthStateChanged 
        } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-auth.js";

        // Optional: Paste your live Firebase project credentials here
        const firebaseConfig = {
            apiKey: "YOUR_FIREBASE_API_KEY",
            authDomain: "YOUR_PROJECT.firebaseapp.com",
            projectId: "YOUR_PROJECT_ID",
            storageBucket: "YOUR_PROJECT.appspot.com",
            messagingSenderId: "YOUR_SENDER_ID",
            appId: "YOUR_APP_ID"
        };

        let auth = null;
        const isFirebaseConfigured = firebaseConfig.apiKey !== "YOUR_FIREBASE_API_KEY";

        if (isFirebaseConfigured) {
            try {
                const app = initializeApp(firebaseConfig);
                auth = getAuth(app);
                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        setupUserSession(user.email, user.displayName || user.email.split('@')[0], user.photoURL);
                    }
                });
            } catch (err) {
                console.warn("Firebase Init failed, running in sandbox mode:", err);
            }
        }

        // Email / Password Login or Auto-Register
        window.handleEmailAuth = async function() {
            const emailInput = document.getElementById('login-email');
            const passwordInput = document.getElementById('login-password');
            const email = emailInput.value.trim();
            const password = passwordInput.value.trim();
            const errorMsg = document.getElementById('auth-error');
            errorMsg.classList.add('hidden');

            if (!email || !email.includes('@')) {
                showError("Please enter a valid email address.");
                return;
            }
            if (!password || password.length < 6) {
                showError("Password must be at least 6 characters.");
                return;
            }

            if (auth) {
                try {
                    const userCredential = await signInWithEmailAndPassword(auth, email, password);
                    setupUserSession(userCredential.user.email, userCredential.user.email.split('@')[0]);
                } catch (loginError) {
                    if (loginError.code === 'auth/user-not-found' || loginError.code === 'auth/invalid-credential') {
                        try {
                            const newCredential = await createUserWithEmailAndPassword(auth, email, password);
                            setupUserSession(newCredential.user.email, newCredential.user.email.split('@')[0]);
                        } catch (registerError) {
                            showError(registerError.message);
                        }
                    } else {
                        showError(loginError.message);
                    }
                }
            } else {
                // Standalone Sandbox Mode
                setupUserSession(email, email.split('@')[0]);
            }
        };

        // Google Sign-In
        window.handleGoogleAuth = async function() {
            const errorMsg = document.getElementById('auth-error');
            errorMsg.classList.add('hidden');

            if (auth) {
                try {
                    const provider = new GoogleAuthProvider();
                    const result = await signInWithPopup(auth, provider);
                    const user = result.user;
                    setupUserSession(user.email, user.displayName, user.photoURL);
                } catch (err) {
                    showError(err.message);
                }
            } else {
                setupUserSession("analyst.google@cereberus.soc", "Google Analyst", "https://api.dicebear.com/7.x/bottts/svg?seed=cereberus");
            }
        };

        // Log out handler
        window.handleLogout = function() {
            if (auth) {
                signOut(auth).catch(() => {});
            }
            localStorage.removeItem('cereberus_active_session');
            document.getElementById('app-dashboard').classList.add('hidden');
            document.getElementById('login-screen').classList.remove('hidden');
            document.getElementById('login-password').value = '';
        };

        function showError(msg) {
            const errorMsg = document.getElementById('auth-error');
            errorMsg.textContent = msg;
            errorMsg.classList.remove('hidden');
        }

        function setupUserSession(email, displayName, photoUrl) {
            currentUserEmail = email;
            localStorage.setItem('cereberus_active_session', JSON.stringify({ email, name: displayName, photo: photoUrl }));

            document.getElementById('login-screen').classList.add('hidden');
            document.getElementById('app-dashboard').classList.remove('hidden');
            document.getElementById('user-email-display').textContent = email;
            document.getElementById('user-name-display').textContent = displayName || email.split('@')[0];
            
            const avatar = document.getElementById('user-avatar');
            avatar.src = photoUrl || `https://api.dicebear.com/7.x/identicon/svg?seed=${encodeURIComponent(email)}`;

            loadUserNotes(email);
            initCharts();
            checkApiKeyStatus();
            startAlertTicker();
        }

        window.addEventListener('DOMContentLoaded', () => {
            const savedSession = localStorage.getItem('cereberus_active_session');
            if (savedSession && !auth) {
                const user = JSON.parse(savedSession);
                setupUserSession(user.email, user.name, user.photo);
            }
        });
    </script>

    <script>
        tailwind.config = {
            theme: {
                extend: {
                    colors: {
                        cyber: { 
                            bg: '#050B14', 
                            panel: '#0A1526', 
                            border: '#1E3A5F', 
                            accent: '#00F0FF', 
                            alert: '#FF0055', 
                            warn: '#FFB800', 
                            success: '#00FF9D' 
                        }
                    }
                }
            }
        }
    </script>
    <style>
        body { background-color: #050B14; color: #e2e8f0; font-family: 'Inter', system-ui, -apple-system, sans-serif; }
        .glass-panel { background: rgba(10, 21, 38, 0.75); backdrop-filter: blur(12px); border: 1px solid #1E3A5F; }
        .cyber-glow { box-shadow: 0 0 25px rgba(0, 240, 255, 0.15); }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .scrollbar-hide { -ms-overflow-style: none; scrollbar-width: none; }
        #chat-window { transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1); transform: translateY(120%); opacity: 0; pointer-events: none; }
        #chat-window.active { transform: translateY(0); opacity: 1; pointer-events: auto; }
        @keyframes marquee { 0% { transform: translateX(100%); } 100% { transform: translateX(-100%); } }
        .ticker-animate { display: inline-block; white-space: nowrap; animation: marquee 35s linear infinite; }
    </style>
</head>
<body class="h-screen w-screen overflow-hidden flex flex-col bg-[radial-gradient(ellipse_at_top,_var(--tw-gradient-stops))] from-blue-950/25 via-cyber-bg to-cyber-bg">

    <!-- UNIVERSAL LOGIN SCREEN -->
    <div id="login-screen" class="fixed inset-0 z-50 flex items-center justify-center bg-cyber-bg/95 backdrop-blur-md px-4">
        <div class="glass-panel p-8 rounded-2xl w-full max-w-md cyber-glow border-t-2 border-t-cyber-accent">
            <div class="text-center mb-6">
                <div class="inline-flex items-center justify-center w-12 h-12 rounded-xl bg-cyber-accent/10 border border-cyber-accent text-cyber-accent mb-3">
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 15v2m-6 4h12a2 2 0 002-2v-6a2 2 0 00-2-2H6a2 2 0 00-2 2v6a2 2 0 002 2zm10-10V7a4 4 0 00-8 0v4h8z"></path></svg>
                </div>
                <h1 class="text-3xl font-black text-white tracking-widest">CEREBERUS</h1>
                <p class="text-xs text-cyber-accent uppercase tracking-wider mt-1">Autonomous AI SOC Platform</p>
                <p class="text-xs text-gray-400 mt-2">Sign in or register an isolated investigation workspace.</p>
            </div>

            <div id="auth-error" class="hidden mb-4 p-3 rounded bg-cyber-alert/10 border border-cyber-alert text-cyber-alert text-xs text-center"></div>

            <div class="space-y-4">
                <div>
                    <label class="block text-xs uppercase tracking-wider text-gray-400 mb-1">Email Address</label>
                    <input type="email" id="login-email" placeholder="analyst@soc.org" class="w-full bg-cyber-bg border border-cyber-border rounded-lg px-4 py-2.5 text-white text-sm focus:outline-none focus:border-cyber-accent transition-colors">
                </div>
                <div>
                    <label class="block text-xs uppercase tracking-wider text-gray-400 mb-1">Password</label>
                    <input type="password" id="login-password" placeholder="••••••••" class="w-full bg-cyber-bg border border-cyber-border rounded-lg px-4 py-2.5 text-white text-sm focus:outline-none focus:border-cyber-accent transition-colors" onkeypress="if(event.key==='Enter') handleEmailAuth()">
                </div>
                <button onclick="handleEmailAuth()" class="w-full bg-cyber-accent text-cyber-bg font-bold py-2.5 rounded-lg hover:bg-white transition-all uppercase tracking-wider text-sm shadow-[0_0_15px_rgba(0,240,255,0.3)]">
                    Access Workspace
                </button>

                <div class="relative flex items-center py-1">
                    <div class="flex-grow border-t border-cyber-border"></div>
                    <span class="flex-shrink-0 mx-3 text-gray-500 text-xs uppercase tracking-wider">or</span>
                    <div class="flex-grow border-t border-cyber-border"></div>
                </div>

                <button onclick="handleGoogleAuth()" class="w-full glass-panel hover:bg-cyber-panel text-white font-medium py-2.5 rounded-lg border border-cyber-border hover:border-cyber-accent transition-all flex items-center justify-center gap-3 text-sm">
                    <svg class="w-4 h-4" viewBox="0 0 24 24"><path fill="#EA4335" d="M12 5c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 1.5 14.97.5 12 .5 7.5.5 3.63 3.06 1.77 6.84l3.66 2.84C6.3 6.93 8.9 5 12 5z"/><path fill="#4285F4" d="M23.49 12.28c0-.85-.08-1.67-.22-2.45H12v4.64h6.45c-.28 1.48-1.11 2.73-2.36 3.58l3.66 2.84c2.14-1.98 3.74-4.89 3.74-8.61z"/><path fill="#FBBC05" d="M5.43 14.68A7.47 7.47 0 0 1 5.03 12c0-.93.15-1.83.4-2.68L1.77 6.48A11.97 11.97 0 0 0 0 12c0 1.92.46 3.74 1.28 5.36l3.66-2.84z"/><path fill="#34A853" d="M12 23.5c3.24 0 5.96-1.07 7.95-2.91l-3.66-2.84c-1.08.72-2.46 1.15-4.29 1.15-3.1 0-5.7-2.07-6.57-4.88L1.77 16.86C3.63 20.64 7.5 23.5 12 23.5z"/></svg>
                    Continue with Google
                </button>
            </div>
        </div>
    </div>

    <!-- LIVE TELEMETRY TICKER -->
    <div id="live-ticker-bar" class="w-full bg-cyber-bg border-b border-cyber-border py-1 px-4 overflow-hidden flex items-center text-[11px] font-mono text-gray-400 z-30">
        <span class="text-cyber-accent font-bold uppercase tracking-wider mr-4 flex items-center gap-1.5 shrink-0">
            <span class="w-2 h-2 rounded-full bg-cyber-accent animate-ping"></span> Live Signal
        </span>
        <div class="overflow-hidden whitespace-nowrap w-full">
            <div id="ticker-content" class="ticker-animate text-gray-300">
                [ALERT-7401] Inbound SSH brute force detected from 185.220.101.5 • [INGESTION] Palo Alto parser normalized 12,410 eps • [ZERO-TRUST] Anomaly detected on service principal `sp-billing-dev` • [AI-CORRELATION] Incident #992 synthesized from 4 endpoint and cloud signals
            </div>
        </div>
    </div>

    <!-- MAIN DASHBOARD -->
    <div id="app-dashboard" class="hidden flex-1 w-full flex overflow-hidden">
        <!-- Sidebar Navigation -->
        <aside class="w-64 glass-panel h-full flex flex-col border-r border-cyber-border z-10">
            <div class="p-5 border-b border-cyber-border flex items-center justify-between">
                <div>
                    <h1 class="text-lg font-black text-white tracking-widest">CEREBERUS</h1>
                    <p class="text-cyber-accent text-[10px] tracking-widest">AI CORRELATION SOC</p>
                </div>
                <div class="w-2.5 h-2.5 rounded-full bg-cyber-success shadow-[0_0_8px_#00FF9D]"></div>
            </div>

            <!-- Profile Info -->
            <div class="p-4 border-b border-cyber-border bg-cyber-panel/50">
                <div class="flex items-center gap-3">
                    <img id="user-avatar" src="" class="w-9 h-9 rounded-full border border-cyber-accent/40 bg-cyber-bg" alt="Avatar">
                    <div class="overflow-hidden">
                        <p id="user-name-display" class="text-xs font-bold text-white truncate">Analyst</p>
                        <p id="user-email-display" class="text-[10px] text-gray-400 truncate">analyst@soc.org</p>
                    </div>
                </div>
                <button onclick="handleLogout()" class="mt-2.5 w-full py-1 text-[10px] text-cyber-alert border border-cyber-alert/30 hover:bg-cyber-alert hover:text-white rounded transition-colors text-center">
                    Disconnect Session
                </button>
            </div>

            <!-- Navigation Tabs -->
            <nav class="flex-1 p-3 space-y-1.5 overflow-y-auto">
                <button onclick="switchTab('dashboard')" id="nav-dashboard" class="w-full text-left px-3.5 py-2 rounded bg-cyber-accent/10 text-cyber-accent border-l-2 border-cyber-accent text-xs font-medium transition-all">Overview Console</button>
                <button onclick="switchTab('correlations')" id="nav-correlations" class="w-full text-left px-3.5 py-2 rounded text-gray-400 hover:bg-cyber-panel hover:text-white text-xs font-medium transition-all">Active Incidents</button>
                <button onclick="switchTab('topology')" id="nav-topology" class="w-full text-left px-3.5 py-2 rounded text-gray-400 hover:bg-cyber-panel hover:text-white text-xs font-medium transition-all">Threat Topology</button>
                <button onclick="switchTab('playbooks')" id="nav-playbooks" class="w-full text-left px-3.5 py-2 rounded text-gray-400 hover:bg-cyber-panel hover:text-white text-xs font-medium transition-all">Response Playbooks</button>
                <button onclick="switchTab('threatintel')" id="nav-threatintel" class="w-full text-left px-3.5 py-2 rounded text-gray-400 hover:bg-cyber-panel hover:text-white text-xs font-medium transition-all">Threat Intelligence</button>
                <button onclick="switchTab('notes')" id="nav-notes" class="w-full text-left px-3.5 py-2 rounded text-gray-400 hover:bg-cyber-panel hover:text-white text-xs font-medium transition-all flex items-center justify-between">
                    <span>Forensic Notes</span>
                    <span id="notes-badge" class="text-[10px] bg-cyber-border px-1.5 py-0.5 rounded text-gray-300">0</span>
                </button>
                <button onclick="switchTab('pipeline')" id="nav-pipeline" class="w-full text-left px-3.5 py-2 rounded text-gray-400 hover:bg-cyber-panel hover:text-white text-xs font-medium transition-all">Ingestion Telemetry</button>
            </nav>
        </aside>

        <!-- Main Content Area -->
        <main class="flex-1 h-full overflow-y-auto p-6 scrollbar-hide">
            
            <!-- TAB 1: OVERVIEW CONSOLE -->
            <div id="view-dashboard" class="view-section space-y-6">
                <header class="border-b border-cyber-border pb-3 flex justify-between items-center">
                    <div>
                        <h2 class="text-xl font-bold text-white tracking-wide">Threat Operations Overview</h2>
                        <p class="text-xs text-gray-400">Continuous telemetry correlation engine • Zero hallucination guarantee</p>
                    </div>
                    <span class="text-xs font-mono px-3 py-1 rounded bg-cyber-panel border border-cyber-border text-cyber-accent">STATUS: ENFORCING</span>
                </header>

                <div class="grid grid-cols-1 md:grid-cols-4 gap-4">
                    <div class="glass-panel p-4 rounded-xl border-l-4 border-l-cyber-alert">
                        <p class="text-xs text-gray-400">Critical Incidents</p>
                        <p class="text-2xl font-black text-white mt-1">3</p>
                    </div>
                    <div class="glass-panel p-4 rounded-xl border-l-4 border-l-cyber-warn">
                        <p class="text-xs text-gray-400">Active Correlations</p>
                        <p class="text-2xl font-black text-white mt-1">14</p>
                    </div>
                    <div class="glass-panel p-4 rounded-xl border-l-4 border-l-cyber-accent">
                        <p class="text-xs text-gray-400">Ingested EPS</p>
                        <p class="text-2xl font-black text-white mt-1">9,240</p>
                    </div>
                    <div class="glass-panel p-4 rounded-xl border-l-4 border-l-cyber-success">
                        <p class="text-xs text-gray-400">Telemetry Health</p>
                        <p class="text-2xl font-black text-white mt-1">99.9%</p>
                    </div>
                </div>

                <!-- Charts Grid -->
                <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
                    <div class="glass-panel p-5 rounded-xl lg:col-span-2 h-72">
                        <div class="flex justify-between items-center mb-3">
                            <h3 class="text-xs font-bold uppercase tracking-wider text-cyber-accent">Event Velocity (Last 24 Hours)</h3>
                            <span class="text-[11px] text-gray-400">Events / Minute</span>
                        </div>
                        <canvas id="velocityChart"></canvas>
                    </div>
                    <div class="glass-panel p-5 rounded-xl h-72">
                        <div class="flex justify-between items-center mb-3">
                            <h3 class="text-xs font-bold uppercase tracking-wider text-cyber-accent">Severity Breakdown</h3>
                            <span class="text-[11px] text-gray-400">Normalized</span>
                        </div>
                        <canvas id="severityChart"></canvas>
                    </div>
                </div>
            </div>

            <!-- TAB 2: ACTIVE INCIDENTS -->
            <div id="view-correlations" class="view-section hidden space-y-6">
                <header class="border-b border-cyber-border pb-3">
                    <h2 class="text-xl font-bold text-white tracking-wide">Synthesized Incident Timelines</h2>
                    <p class="text-xs text-gray-400">Cross-domain telemetry linked strictly by raw cryptographic hash & log references.</p>
                </header>
                
                <!-- SEV 1 Incident -->
                <div class="glass-panel p-5 rounded-xl border border-cyber-alert relative">
                    <div class="absolute top-0 right-0 bg-cyber-alert text-white text-[10px] px-3 py-1 font-bold rounded-bl-lg tracking-wider">SEV 1 - ACTIVE</div>
                    <h3 class="text-base font-bold text-white">Impossible Travel & Data Exfiltration</h3>
                    <p class="text-xs text-gray-400 mt-0.5 mb-3">Sources: <span class="text-cyber-accent">Okta IDP + CrowdStrike Falcon + AWS CloudTrail</span></p>
                    
                    <div class="space-y-2 border-l-2 border-cyber-border pl-4 my-3 text-xs">
                        <div>
                            <span class="text-cyber-warn font-mono font-bold">14:02:11 UTC</span>
                            <p class="text-gray-300">Push notification exhaustion detected on administrative account `ops-admin@enterprise.internal`.</p>
                        </div>
                        <div>
                            <span class="text-cyber-alert font-mono font-bold">14:08:44 UTC</span>
                            <p class="text-gray-300">Base64 obfuscated PowerShell payload spawned on endpoint `PROD-SRV-04` by child process `svchost.exe`.</p>
                        </div>
                        <div>
                            <span class="text-cyber-alert font-mono font-bold">14:15:02 UTC</span>
                            <p class="text-gray-300">Spike in `s3:GetObject` against restricted bucket `arn:aws:s3:::customer-pii-vault` from external IP `185.220.101.5`.</p>
                        </div>
                    </div>

                    <div class="flex gap-2 justify-end pt-3 border-t border-cyber-border">
                        <button onclick="executeAction('Quarantine Host PROD-SRV-04')" class="px-3 py-1.5 text-xs text-white bg-cyber-panel hover:bg-gray-800 border border-cyber-border rounded">Isolate Endpoint</button>
                        <button onclick="executeAction('Revoke AWS IAM Credentials')" class="px-3 py-1.5 text-xs text-cyber-bg bg-cyber-alert hover:bg-white font-bold rounded">Revoke Access Tokens</button>
                        <button onclick="copyToNotes('Case #101: Investigated impossible travel on ops-admin. Actions: Endpoint isolated & AWS tokens revoked.')" class="px-3 py-1.5 text-xs text-cyber-accent border border-cyber-accent/40 rounded hover:bg-cyber-accent/10">Add to Notes</button>
                    </div>
                </div>

                <!-- SEV 2 Incident -->
                <div class="glass-panel p-5 rounded-xl border border-cyber-warn relative">
                    <div class="absolute top-0 right-0 bg-cyber-warn text-cyber-bg text-[10px] px-3 py-1 font-bold rounded-bl-lg tracking-wider">SEV 2 - SUSPICIOUS</div>
                    <h3 class="text-base font-bold text-white">Internal Kerberoasting & Lateral Movement</h3>
                    <p class="text-xs text-gray-400 mt-0.5 mb-3">Sources: <span class="text-cyber-accent">Active Directory Logs + Suricata NIDS</span></p>
                    <p class="text-xs text-gray-300">Anomalous TGS service ticket request with RC4 cipher encryption originating from workstation `DESKTOP-882J` targeting MSSQL service SPNs.</p>
                    <div class="flex gap-2 justify-end pt-3 border-t border-cyber-border mt-3">
                        <button onclick="executeAction('Flag AD Account for Forced Password Reset')" class="px-3 py-1.5 text-xs text-cyber-bg bg-cyber-warn font-bold rounded">Force Password Reset</button>
                    </div>
                </div>
            </div>

            <!-- TAB 3: THREAT TOPOLOGY MAP -->
            <div id="view-topology" class="view-section hidden space-y-6">
                <header class="border-b border-cyber-border pb-3">
                    <h2 class="text-xl font-bold text-white tracking-wide">Threat Ingress Topology</h2>
                    <p class="text-xs text-gray-400">Visual mapping of live threat actors to localized network infrastructure.</p>
                </header>

                <div class="glass-panel p-6 rounded-xl relative overflow-hidden h-96 flex items-center justify-center">
                    <div class="absolute inset-0 opacity-10 bg-[radial-gradient(#00F0FF_1px,transparent_1px)] [background-size:16px_16px]"></div>
                    
                    <div class="relative w-full max-w-2xl flex items-center justify-between">
                        <!-- Node 1: External Adversary -->
                        <div class="text-center">
                            <div class="w-16 h-16 rounded-full bg-cyber-alert/20 border-2 border-cyber-alert flex items-center justify-center mx-auto mb-2 animate-pulse">
                                <svg class="w-8 h-8 text-cyber-alert" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"/></svg>
                            </div>
                            <p class="text-xs font-bold text-white">Tor Exit Relay</p>
                            <p class="text-[10px] text-cyber-alert font-mono">185.220.101.5</p>
                        </div>

                        <!-- Vector Line -->
                        <div class="flex-1 flex flex-col items-center px-4">
                            <span class="text-[10px] text-cyber-warn font-mono mb-1">Encrypted Payload</span>
                            <div class="w-full h-0.5 bg-gradient-to-r from-cyber-alert via-cyber-warn to-cyber-accent relative">
                                <div class="w-2 h-2 rounded-full bg-white absolute top-[-3px] left-1/2 animate-ping"></div>
                            </div>
                            <span class="text-[10px] text-gray-500 font-mono mt-1">443/TCP TLSv1.3</span>
                        </div>

                        <!-- Node 2: Edge Firewall -->
                        <div class="text-center">
                            <div class="w-16 h-16 rounded-full bg-cyber-accent/20 border-2 border-cyber-accent flex items-center justify-center mx-auto mb-2">
                                <svg class="w-8 h-8 text-cyber-accent" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m5.618-4.016A11.955 11.955 0 0112 2.944a11.955 11.955 0 01-8.618 3.04A12.02 12.02 0 003 9c0 5.591 3.824 10.29 9 11.622 5.176-1.332 9-6.03 9-11.622 0-1.042-.133-2.052-.382-3.016z"/></svg>
                            </div>
                            <p class="text-xs font-bold text-white">Palo Alto Gateway</p>
                            <p class="text-[10px] text-cyber-accent font-mono">FW-EDGE-01</p>
                        </div>

                        <!-- Vector Line -->
                        <div class="flex-1 flex flex-col items-center px-4">
                            <span class="text-[10px] text-cyber-alert font-mono mb-1">Lateral Tunnel</span>
                            <div class="w-full h-0.5 bg-gradient-to-r from-cyber-accent to-cyber-alert"></div>
                            <span class="text-[10px] text-gray-500 font-mono mt-1">RPC / SMB</span>
                        </div>

                        <!-- Node 3: Target Workstation -->
                        <div class="text-center">
                            <div class="w-16 h-16 rounded-full bg-cyber-alert/20 border-2 border-cyber-alert flex items-center justify-center mx-auto mb-2">
                                <svg class="w-8 h-8 text-cyber-alert" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.75 17L9 20l-1 1h8l-1-1-.75-3M3 13h18M5 17h14a2 2 0 002-2V5a2 2 0 00-2-2H5a2 2 0 00-2 2v10a2 2 0 002 2z"/></svg>
                            </div>
                            <p class="text-xs font-bold text-white">Compromised Node</p>
                            <p class="text-[10px] text-cyber-alert font-mono">PROD-SRV-04</p>
                        </div>
                    </div>
                </div>
            </div>

            <!-- TAB 4: RESPONSE PLAYBOOKS -->
            <div id="view-playbooks" class="view-section hidden space-y-6">
                <header class="border-b border-cyber-border pb-3">
                    <h2 class="text-xl font-bold text-white tracking-wide">Automated Response Playbooks</h2>
                    <p class="text-xs text-gray-400">Deterministic SOAR playbooks configured to run mitigation workflows.</p>
                </header>

                <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <div class="glass-panel p-5 rounded-xl border border-cyber-border space-y-3">
                        <div class="flex justify-between items-start">
                            <div>
                                <h3 class="font-bold text-white text-sm">Playbook #01: Ransomware Host Quarantine</h3>
                                <p class="text-xs text-gray-400">Triggered upon mass entropy file change or Shadow Copy deletion.</p>
                            </div>
                            <span class="text-[10px] px-2 py-0.5 rounded bg-cyber-success/20 text-cyber-success">READY</span>
                        </div>
                        <ul class="text-xs text-gray-300 space-y-1 list-disc list-inside">
                            <li>Isolates endpoint network interface cards via EDR API</li>
                            <li>Captures memory dump to S3 for volatility analysis</li>
                            <li>Kills non-system processes with unsigned certificates</li>
                        </ul>
                        <button onclick="executeAction('Ransomware Quarantine Playbook executed')" class="w-full bg-cyber-panel hover:border-cyber-accent border border-cyber-border text-white text-xs font-bold py-2 rounded transition-all">Simulate Run</button>
                    </div>

                    <div class="glass-panel p-5 rounded-xl border border-cyber-border space-y-3">
                        <div class="flex justify-between items-start">
                            <div>
                                <h3 class="font-bold text-white text-sm">Playbook #02: Credential Compromise Revocation</h3>
                                <p class="text-xs text-gray-400">Triggered upon impossible travel or MFA credential fatigue alerts.</p>
                            </div>
                            <span class="text-[10px] px-2 py-0.5 rounded bg-cyber-success/20 text-cyber-success">READY</span>
                        </div>
                        <ul class="text-xs text-gray-300 space-y-1 list-disc list-inside">
                            <li>Invalidates all active OAuth Refresh and IdP sessions</li>
                            <li>Rotates AWS access keys and attaches deny-all IAM inline policy</li>
                            <li>Alerts on-call security engineer via PagerDuty</li>
                        </ul>
                        <button onclick="executeAction('Credential Revocation Playbook executed')" class="w-full bg-cyber-panel hover:border-cyber-accent border border-cyber-border text-white text-xs font-bold py-2 rounded transition-all">Simulate Run</button>
                    </div>
                </div>
            </div>

            <!-- TAB 5: THREAT INTELLIGENCE -->
            <div id="view-threatintel" class="view-section hidden space-y-6">
                <header class="border-b border-cyber-border pb-3">
                    <h2 class="text-xl font-bold text-white tracking-wide">Threat Intelligence Feed</h2>
                    <p class="text-xs text-gray-400">Real-time indicators of compromise (IOCs) ingested from global telemetry partners.</p>
                </header>

                <div class="glass-panel rounded-xl overflow-hidden border border-cyber-border">
                    <table class="w-full text-left text-xs">
                        <thead class="bg-cyber-panel border-b border-cyber-border text-gray-400 font-mono">
                            <tr>
                                <th class="p-3">Indicator (IOC)</th>
                                <th class="p-3">Type</th>
                                <th class="p-3">Threat Actor</th>
                                <th class="p-3">Confidence</th>
                                <th class="p-3">Status</th>
                            </tr>
                        </thead>
                        <tbody class="divide-y divide-cyber-border text-gray-300">
                            <tr>
                                <td class="p-3 font-mono text-cyber-accent">185.220.101.5</td>
                                <td class="p-3">IPv4 (Tor Node)</td>
                                <td class="p-3">APT-29 (Cozy Bear)</td>
                                <td class="p-3 text-cyber-alert font-bold">98%</td>
                                <td class="p-3"><span class="px-2 py-0.5 rounded bg-cyber-alert/20 text-cyber-alert text-[10px]">Blocked</span></td>
                            </tr>
                            <tr>
                                <td class="p-3 font-mono text-cyber-accent">e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855</td>
                                <td class="p-3">SHA-256</td>
                                <td class="p-3">Lazarus Group</td>
                                <td class="p-3 text-cyber-warn font-bold">85%</td>
                                <td class="p-3"><span class="px-2 py-0.5 rounded bg-cyber-warn/20 text-cyber-warn text-[10px]">Monitored</span></td>
                            </tr>
                            <tr>
                                <td class="p-3 font-mono text-cyber-accent">login-update-microsoft365.online</td>
                                <td class="p-3">Phishing Domain</td>
                                <td class="p-3">Storm-0558</td>
                                <td class="p-3 text-cyber-alert font-bold">95%</td>
                                <td class="p-3"><span class="px-2 py-0.5 rounded bg-cyber-alert/20 text-cyber-alert text-[10px]">Sinkholed</span></td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>

            <!-- TAB 6: PERSONAL CASE NOTES (Per-User Isolated) -->
            <div id="view-notes" class="view-section hidden space-y-6">
                <header class="border-b border-cyber-border pb-3">
                    <h2 class="text-xl font-bold text-white tracking-wide">Analyst Forensic Notes</h2>
                    <p class="text-xs text-gray-400">Private investigation records stored specifically for your active email account.</p>
                </header>

                <div class="glass-panel p-4 rounded-xl space-y-3">
                    <textarea id="note-input" rows="3" placeholder="Log an IOC, forensic deduction, or timeline entry..." class="w-full bg-cyber-bg border border-cyber-border rounded-lg p-3 text-white text-xs focus:outline-none focus:border-cyber-accent"></textarea>
                    <button onclick="addCurrentNote()" class="px-4 py-2 bg-cyber-accent text-cyber-bg font-bold rounded-lg text-xs hover:bg-white transition-all uppercase tracking-wider">Save Note to Account</button>
                </div>

                <div id="notes-list" class="space-y-3"></div>
            </div>

            <!-- TAB 7: PIPELINE INGESTION HEALTH -->
            <div id="view-pipeline" class="view-section hidden space-y-6">
                <header class="border-b border-cyber-border pb-3">
                    <h2 class="text-xl font-bold text-white tracking-wide">Telemetry Ingestion Streams</h2>
                    <p class="text-xs text-gray-400">Real-time health and parsing metrics of multi-domain collectors.</p>
                </header>

                <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <div class="glass-panel p-4 rounded-xl flex justify-between items-center">
                        <div>
                            <p class="font-bold text-white text-sm">CrowdStrike Falcon (Endpoint)</p>
                            <p class="text-xs text-gray-400">2,840 events/sec • Zero dropped packets</p>
                        </div>
                        <span class="text-cyber-success text-xs font-semibold flex items-center gap-1.5"><span class="w-2 h-2 rounded-full bg-cyber-success animate-pulse"></span> Streaming</span>
                    </div>
                    <div class="glass-panel p-4 rounded-xl flex justify-between items-center">
                        <div>
                            <p class="font-bold text-white text-sm">AWS CloudTrail (Cloud Auditing)</p>
                            <p class="text-xs text-gray-400">4,190 events/sec • Latency: 110ms</p>
                        </div>
                        <span class="text-cyber-success text-xs font-semibold flex items-center gap-1.5"><span class="w-2 h-2 rounded-full bg-cyber-success animate-pulse"></span> Streaming</span>
                    </div>
                    <div class="glass-panel p-4 rounded-xl flex justify-between items-center">
                        <div>
                            <p class="font-bold text-white text-sm">Okta Identity Engine (IDP)</p>
                            <p class="text-xs text-gray-400">1,210 events/sec • Webhook active</p>
                        </div>
                        <span class="text-cyber-success text-xs font-semibold flex items-center gap-1.5"><span class="w-2 h-2 rounded-full bg-cyber-success animate-pulse"></span> Streaming</span>
                    </div>
                    <div class="glass-panel p-4 rounded-xl flex justify-between items-center">
                        <div>
                            <p class="font-bold text-white text-sm">Palo Alto Networks (Edge Network)</p>
                            <p class="text-xs text-gray-400">998 events/sec • Syslog TLS enabled</p>
                        </div>
                        <span class="text-cyber-success text-xs font-semibold flex items-center gap-1.5"><span class="w-2 h-2 rounded-full bg-cyber-success animate-pulse"></span> Streaming</span>
                    </div>
                </div>
            </div>
        </main>
    </div>

    <!-- AI COPILOT CHAT BUTTON -->
    <button onclick="toggleChat()" class="fixed bottom-6 right-6 w-14 h-14 bg-cyber-accent rounded-full flex items-center justify-center shadow-[0_0_20px_rgba(0,240,255,0.4)] hover:scale-105 transition-all z-40">
        <svg class="w-6 h-6 text-cyber-bg" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 10h.01M12 10h.01M16 10h.01M9 16H5a2 2 0 01-2-2V6a2 2 0 012-2h14a2 2 0 012 2v8a2 2 0 01-2 2h-5l-5 5v-5z"></path></svg>
    </button>

    <!-- FLOATING AI CHAT WINDOW -->
    <div id="chat-window" class="fixed bottom-24 right-6 w-96 h-[500px] glass-panel rounded-2xl shadow-2xl flex flex-col z-40 overflow-hidden border-t-2 border-t-cyber-accent">
        <!-- Header -->
        <div class="bg-cyber-panel p-3 border-b border-cyber-border flex justify-between items-center">
            <div class="flex items-center gap-2">
                <div class="w-2 h-2 bg-cyber-accent rounded-full animate-pulse"></div>
                <h3 class="font-bold text-white text-xs tracking-wider">Cereberus Copilot</h3>
            </div>
            <div class="flex items-center gap-3">
                <button onclick="toggleSettings()" class="text-gray-400 hover:text-cyber-accent transition-colors" title="Configure Gemini API Key">
                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z"></path><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"></path></svg>
                </button>
                <button onclick="toggleChat()" class="text-gray-400 hover:text-white text-lg leading-none">&times;</button>
            </div>
        </div>
        
        <!-- Settings Panel Overlay -->
        <div id="ai-settings-panel" class="hidden absolute inset-0 top-11 bg-cyber-bg/95 backdrop-blur-md p-5 z-50 flex flex-col justify-center border-b border-cyber-border">
            <h4 class="text-white text-sm font-bold mb-2">Connect Your Gemini API Key</h4>
            <p class="text-xs text-gray-400 mb-4">Add your free key from Google AI Studio. It is saved strictly in your local browser storage.</p>
            <input type="password" id="gemini-key-input" placeholder="AIzaSy..." class="w-full bg-cyber-panel border border-cyber-border rounded px-3 py-2 text-white text-xs mb-3 focus:outline-none focus:border-cyber-accent">
            <div class="flex gap-2">
                <button onclick="saveApiKey()" class="flex-1 bg-cyber-accent text-cyber-bg font-bold py-2 rounded text-xs hover:bg-white transition-all">Save Key</button>
                <button onclick="toggleSettings()" class="flex-1 border border-cyber-border text-white py-2 rounded text-xs hover:bg-cyber-panel">Cancel</button>
            </div>
            <p id="key-status-msg" class="text-center text-xs mt-3 hidden"></p>
        </div>
        
        <!-- Messages Flow -->
        <div id="chat-messages" class="flex-1 overflow-y-auto p-4 space-y-3 text-xs">
            <div class="bg-cyber-panel/90 rounded-lg p-3 text-gray-300 mr-6 border border-cyber-border">
                Hello! I am Cereberus. I synthesize alerts across endpoint, identity, and cloud telemetry without hallucinating missing data. What incident are we reviewing?
            </div>
        </div>
        
        <!-- Input Area -->
        <div class="p-3 border-t border-cyber-border bg-cyber-bg/80">
            <div class="flex gap-2">
                <input type="text" id="chat-input" placeholder="Ask about an alert, hash, or IOC..." class="flex-1 bg-cyber-panel border border-cyber-border rounded-lg px-3 py-2 text-white text-xs focus:outline-none focus:border-cyber-accent" onkeypress="if(event.key === 'Enter') sendMessage()">
                <button onclick="sendMessage()" class="bg-cyber-accent text-cyber-bg px-3.5 py-2 rounded-lg font-bold text-xs hover:bg-white transition-colors">Send</button>
            </div>
        </div>
    </div>

    <!-- NOTIFICATION TOAST -->
    <div id="toast-notify" class="fixed top-6 right-6 z-50 hidden glass-panel px-4 py-2.5 rounded-lg border border-cyber-accent text-xs text-white shadow-2xl flex items-center gap-2">
        <span class="w-2 h-2 rounded-full bg-cyber-accent"></span>
        <span id="toast-text">Action executed</span>
    </div>

    <!-- CONTROLLERS -->
    <script>
        let currentUserEmail = "";

        // Tab Navigation
        function switchTab(tabId) {
            document.querySelectorAll('.view-section').forEach(el => el.classList.add('hidden'));
            document.querySelectorAll('nav button').forEach(el => {
                el.classList.remove('bg-cyber-accent/10', 'text-cyber-accent', 'border-l-2', 'border-cyber-accent');
                el.classList.add('text-gray-400');
            });
            document.getElementById(`view-${tabId}`).classList.remove('hidden');
            const activeBtn = document.getElementById(`nav-${tabId}`);
            activeBtn.classList.remove('text-gray-400');
            activeBtn.classList.add('bg-cyber-accent/10', 'text-cyber-accent', 'border-l-2', 'border-cyber-accent');
        }

        // Automated Mitigation Toast
        function executeAction(actionName) {
            const toast = document.getElementById('toast-notify');
            document.getElementById('toast-text').textContent = `SOC Action: ${actionName}`;
            toast.classList.remove('hidden');
            setTimeout(() => toast.classList.add('hidden'), 3500);
        }

        // Personal Isolated Notes
        function getStorageKey() { return `cereberus_notes_${currentUserEmail}`; }

        function loadUserNotes(email) {
            const raw = localStorage.getItem(`cereberus_notes_${email}`);
            const notes = raw ? JSON.parse(raw) : [];
            const list = document.getElementById('notes-list');
            const badge = document.getElementById('notes-badge');
            list.innerHTML = "";
            badge.textContent = notes.length;

            if (notes.length === 0) {
                list.innerHTML = `<div class="p-6 text-center text-gray-500 text-xs glass-panel rounded-xl">No case notes logged yet for ${email}. Record your first finding above.</div>`;
                return;
            }

            notes.forEach((note, index) => {
                const item = document.createElement('div');
                item.className = "glass-panel p-3 rounded-lg flex justify-between items-start text-xs border border-cyber-border";
                item.innerHTML = `
                    <div class="space-y-1">
                        <p class="text-white">${note.text}</p>
                        <span class="text-[10px] text-gray-500 font-mono">${note.timestamp}</span>
                    </div>
                    <button onclick="deleteNote(${index})" class="text-cyber-alert hover:text-white ml-3 text-[10px]">[Delete]</button>
                `;
                list.appendChild(item);
            });
        }

        function addCurrentNote() {
            const input = document.getElementById('note-input');
            const text = input.value.trim();
            if (!text) return;
            const notes = JSON.parse(localStorage.getItem(getStorageKey()) || "[]");
            notes.unshift({ text: text, timestamp: new Date().toLocaleTimeString() + " - " + new Date().toLocaleDateString() });
            localStorage.setItem(getStorageKey(), JSON.stringify(notes));
            input.value = "";
            loadUserNotes(currentUserEmail);
        }

        function copyToNotes(text) {
            const notes = JSON.parse(localStorage.getItem(getStorageKey()) || "[]");
            notes.unshift({ text: text, timestamp: new Date().toLocaleTimeString() + " - " + new Date().toLocaleDateString() });
            localStorage.setItem(getStorageKey(), JSON.stringify(notes));
            loadUserNotes(currentUserEmail);
            switchTab('notes');
        }

        function deleteNote(index) {
            const notes = JSON.parse(localStorage.getItem(getStorageKey()) || "[]");
            notes.splice(index, 1);
            localStorage.setItem(getStorageKey(), JSON.stringify(notes));
            loadUserNotes(currentUserEmail);
        }

        // Alert Ticker Rotation
        function startAlertTicker() {
            const ticker = document.getElementById('ticker-content');
            const alerts = [
                "[ALERT-7401] Inbound SSH brute force detected from 185.220.101.5",
                "[INGESTION] Palo Alto parser normalized 12,410 eps without drop",
                "[ZERO-TRUST] Anomaly detected on service principal `sp-billing-dev`",
                "[AI-CORRELATION] Incident #992 synthesized from 4 telemetry streams",
                "[DEFENSE] EDR isolated endpoint `DEV-WIN-02` following Mimikatz detection"
            ];
            let i = 0;
            setInterval(() => {
                i = (i + 1) % alerts.length;
                ticker.textContent = alerts.join(' • ');
            }, 10000);
        }

        // Charts
        let velocityChart = null;
        let severityChart = null;
        function initCharts() {
            if (velocityChart) return;
            
            // Velocity Chart
            const ctx1 = document.getElementById('velocityChart');
            if (ctx1) {
                velocityChart = new Chart(ctx1.getContext('2d'), {
                    type: 'line',
                    data: {
                        labels: ['00:00', '04:00', '08:00', '12:00', '16:00', '20:00', '24:00'],
                        datasets: [{
                            data: [3200, 2900, 6800, 9240, 7100, 5200, 4100],
                            borderColor: '#00F0FF',
                            backgroundColor: 'rgba(0, 240, 255, 0.08)',
                            borderWidth: 2, tension: 0.35, fill: true
                        }]
                    },
                    options: {
                        responsive: true, maintainAspectRatio: false,
                        plugins: { legend: { display: false } },
                        scales: {
                            y: { grid: { color: 'rgba(30, 58, 95, 0.3)' }, ticks: { color: '#64748b', font: { size: 10 } } },
                            x: { grid: { color: 'rgba(30, 58, 95, 0.3)' }, ticks: { color: '#64748b', font: { size: 10 } } }
                        }
                    }
                });
            }

            // Severity Chart
            const ctx2 = document.getElementById('severityChart');
            if (ctx2) {
                severityChart = new Chart(ctx2.getContext('2d'), {
                    type: 'doughnut',
                    data: {
                        labels: ['Critical (SEV-1)', 'High (SEV-2)', 'Medium (SEV-3)', 'Low/Informational'],
                        datasets: [{
                            data: [3, 14, 45, 120],
                            backgroundColor: ['#FF0055', '#FFB800', '#00F0FF', '#00FF9D'],
                            borderWidth: 0
                        }]
                    },
                    options: {
                        responsive: true, maintainAspectRatio: false,
                        plugins: { legend: { position: 'bottom', labels: { color: '#94a3b8', font: { size: 10 } } } }
                    }
                });
            }
        }

        // Chat & BYOK Logic
        function toggleChat() {
            document.getElementById('chat-window').classList.toggle('active');
        }

        function toggleSettings() {
            const panel = document.getElementById('ai-settings-panel');
            panel.classList.toggle('hidden');
            const savedKey = localStorage.getItem('cereberus_gemini_key');
            if (savedKey) {
                document.getElementById('gemini-key-input').value = savedKey;
            }
        }

        function saveApiKey() {
            const val = document.getElementById('gemini-key-input').value.trim();
            const status = document.getElementById('key-status-msg');
            status.classList.remove('hidden', 'text-cyber-alert', 'text-cyber-success');
            
            if (val.length < 10) {
                status.textContent = "Invalid API Key format.";
                status.classList.add('text-cyber-alert');
                return;
            }
            
            localStorage.setItem('cereberus_gemini_key', val);
            status.textContent = "Key saved securely to browser storage!";
            status.classList.add('text-cyber-success');
            setTimeout(() => { toggleSettings(); status.classList.add('hidden'); }, 1500);
        }

        function checkApiKeyStatus() {
            if (!localStorage.getItem('cereberus_gemini_key')) {
                setTimeout(() => {
                    appendMsg('System', '⚠️ No API Key found. Click the ⚙️ settings icon above to link your free Gemini key and activate the AI.', false);
                }, 1000);
            }
        }

        async function sendMessage() {
            const input = document.getElementById('chat-input');
            const message = input.value.trim();
            if (!message) return;

            appendMsg('You', message, true);
            input.value = '';

            const storedKey = localStorage.getItem('cereberus_gemini_key');
            if (!storedKey) {
                appendMsg('System', 'Authentication Required: Please click the ⚙️ icon in the chat header to supply your Gemini API Key.', false);
                return;
            }

            const typing = appendMsg('Cereberus', 'Correlating telemetry...', false, true);
            const endpoint = `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${storedKey}`;
            
            const payload = {
                contents: [{
                    parts: [{
                        text: `You are Cereberus, an autonomous AI Security Operations Center co-pilot.
                        Your task is to analyze, correlate, and explain security incidents in clear, human-understandable English.
                        Rule: Do NOT invent or hallucinate missing forensic evidence. If data is lacking, explicitly flag it.
                        Query: ${message}`
                    }]
                }]
            };

            try {
                const res = await fetch(endpoint, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });
                
                const data = await res.json();
                typing.remove();
                
                if (data.error) {
                    throw new Error(data.error.message || "API Error");
                }
                
                if (data.candidates && data.candidates[0].content) {
                    appendMsg('Cereberus', data.candidates[0].content.parts[0].text, false);
                } else {
                    appendMsg('Cereberus', "I was unable to correlate that query with known telemetry. Could you provide the specific IOC or log artifact?", false);
                }
            } catch (err) {
                typing.remove();
                appendMsg('System', `Connection Failed: ${err.message}. Please check your API key in settings.`, false);
            }
        }

        function appendMsg(sender, text, isUser, isTyping = false) {
            const box = document.getElementById('chat-messages');
            const div = document.createElement('div');
            
            if (sender === 'System') {
                div.className = 'bg-cyber-alert/20 rounded-lg p-2.5 text-cyber-warn text-center border border-cyber-warn/50 mx-4';
            } else {
                div.className = isUser 
                    ? 'bg-cyber-accent/10 rounded-lg p-2.5 text-white ml-6 border border-cyber-accent/30'
                    : `bg-cyber-panel/90 rounded-lg p-2.5 text-gray-300 mr-6 border border-cyber-border ${isTyping ? 'animate-pulse' : ''}`;
            }
            
            div.innerText = text;
            box.appendChild(div);
            box.scrollTop = box.scrollHeight;
            return div;
        }
    </script>
</body>
</html>
