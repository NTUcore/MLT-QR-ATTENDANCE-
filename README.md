<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>نظام حضور جامعة التقنية الشمالية - الإصدار السريع</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
    <script src="https://unpkg.com/html5-qrcode"></script>
    
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+Arabic:wght@400;700;900&display=swap');
        body { font-family: 'Noto Sans Arabic', sans-serif; -webkit-tap-highlight-color: transparent; scroll-behavior: smooth; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .qr-container canvas, .qr-container img { margin: 0 auto; border-radius: 1rem; border: 4px solid #fff; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        #reader { width: 100% !important; border: none !important; }
        #reader video { border-radius: 1.5rem; object-fit: cover !important; width: 100% !important; height: 100% !important; }
        .animate-in { animation: fadeIn 0.15s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(5px); } to { opacity: 1; transform: translateY(0); } }
        #admin-modal, #manual-modal { display: none; }
        #admin-modal.active, #manual-modal.active { display: flex; }
    </style>
</head>
<body class="bg-slate-50 text-slate-900 min-h-screen pb-10">

    <header class="p-4 bg-white shadow-sm border-b text-center sticky top-0 z-50">
        <h1 class="text-blue-700 font-bold text-lg">Northern Technical University</h1>
        <h2 class="text-red-600 font-semibold text-xs">كلية التقنيات الصحية والطبية - كركوك</h2>
    </header>

    <main class="max-w-xl mx-auto p-4" id="app-root">
        <div class="flex items-center justify-center h-64">
            <div class="animate-spin rounded-full h-10 w-10 border-t-2 border-blue-600"></div>
        </div>
    </main>

    <!-- Admin Login Modal -->
    <div id="admin-modal" class="fixed inset-0 z-[100] bg-slate-900/60 backdrop-blur-sm items-center justify-center p-4">
        <div class="bg-white rounded-[2rem] p-8 w-full max-sm shadow-2xl animate-in">
            <h3 class="text-xl font-black mb-6 text-center text-slate-800">دخول الإدارة</h3>
            <div class="space-y-4">
                <input id="adm-user" type="text" placeholder="اسم المستخدم" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 font-bold">
                <input id="adm-pass" type="password" placeholder="كلمة المرور" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 font-bold">
                <button id="adm-confirm" class="w-full py-4 bg-blue-600 text-white font-black rounded-2xl shadow-lg active:scale-95 transition-all">متابعة</button>
                <button id="adm-cancel" class="w-full py-2 text-slate-400 font-bold text-sm">إلغاء</button>
            </div>
        </div>
    </div>

    <!-- Manual Add Modal -->
    <div id="manual-modal" class="fixed inset-0 z-[100] bg-slate-900/60 backdrop-blur-sm items-center justify-center p-4">
        <div class="bg-white rounded-[2rem] p-6 w-full max-sm shadow-2xl animate-in border border-slate-100">
            <h3 class="text-lg font-black mb-4 text-center text-slate-800 border-b pb-2">تسجيل حضور يدوي</h3>
            <div class="space-y-4">
                <div class="bg-blue-50 p-3 rounded-xl">
                    <p class="text-[10px] text-blue-400 font-bold">المادة الحالية</p>
                    <p id="manual-subject-display" class="font-black text-blue-800 text-sm"></p>
                </div>
                <div>
                    <label class="block text-xs font-bold text-slate-400 mb-2">اختر الطالب من القائمة</label>
                    <select id="manual-student-select" class="w-full p-4 bg-slate-50 rounded-xl border-none font-bold text-sm outline-none focus:ring-2 ring-blue-500/20">
                        <option value="">جاري تحميل الطلاب...</option>
                    </select>
                </div>
                <button id="manual-save-btn" class="w-full py-4 bg-green-600 text-white font-black rounded-2xl shadow-lg active:scale-95 transition-all">تأكيد الحضور السحابي</button>
                <button id="manual-close-btn" class="w-full py-3 text-slate-400 font-bold text-sm bg-slate-50 rounded-xl">إغلاق</button>
            </div>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getFirestore, collection, doc, setDoc, getDoc, getDocs, addDoc, onSnapshot, query, where, deleteDoc, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";

        const firebaseConfig = {
            apiKey: "AIzaSyBZfsS5xjUyunwIt_cHaPWHQI3H_lzYXoo",
            authDomain: "ntumlt.firebaseapp.com",
            projectId: "ntumlt",
            storageBucket: "ntumlt.firebasestorage.app",
            messagingSenderId: "658799327665",
            appId: "1:658799327665:web:8841af44d52aa7caa12590"
        };

        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);
        const appId = "ntumlt-attendance";

        const SUBJECTS = ["طرائق البحث", "الحشرات طبية (نظري)", "الحشرات طبية (عملي)", "نشاطات لا صفية", "ساعة ارشاد", "الكيمياء الحياتية السريرية (نظري)", "الكيمياء الحياتية (عملي)", "الفطريات (نظري)", "الفطريات (عملي)", "امراض الدم (نظري)", "امراض الدم (عملي)", "اللغه إنكليزية"];

        let userRole = localStorage.getItem('role') || null;
        let userData = JSON.parse(localStorage.getItem('userData')) || null;
        let html5QrCode = null;
        let isProcessing = false;
        let membersCache = {}; 

        const render = (html) => {
            const root = document.getElementById('app-root');
            root.innerHTML = `<div class="animate-in">${html}</div>`;
            lucide.createIcons();
        };

        const notify = (msg, isError = false) => {
            const toast = document.createElement('div');
            toast.className = `fixed top-24 left-1/2 -translate-x-1/2 z-[110] px-6 py-3 rounded-2xl shadow-2xl text-white font-bold animate-in flex items-center gap-2 ${isError ? 'bg-red-500' : 'bg-green-600'}`;
            toast.innerHTML = isError ? `<i data-lucide="alert-circle" size="18"></i> ${msg}` : `<i data-lucide="check-circle" size="18"></i> ${msg}`;
            document.body.appendChild(toast);
            lucide.createIcons();
            setTimeout(() => { toast.style.opacity = '0'; setTimeout(() => toast.remove(), 300); }, 1500);
        };

        const goBackToHome = async () => {
            if (html5QrCode && html5QrCode.isScanning) {
                try { await html5QrCode.stop(); } catch(e) {}
            }
            try { await signOut(auth); } catch(e) {}
            localStorage.clear();
            sessionStorage.clear();
            userRole = null; userData = null;
            window.location.reload(); 
        };

        const loadMembers = () => {
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'members'), (snap) => {
                membersCache = {};
                snap.forEach(doc => { membersCache[doc.id] = doc.data().name; });
            });
        };

        onAuthStateChanged(auth, (user) => {
            if (!user) signInAnonymously(auth);
            showView();
        });

        const showView = () => {
            if (!userRole) return showLogin();
            if (userRole === 'admin') { loadMembers(); return showAdmin(); }
            if (userRole === 'student') return showStudent();
        };

        const showLogin = () => {
            render(`
                <div class="mt-8 bg-white p-8 rounded-[2.5rem] shadow-xl border border-slate-100 animate-in">
                    <h3 class="text-2xl font-black text-center mb-8 text-slate-800">بوابة الدخول</h3>
                    <div class="space-y-4">
                        <input id="stu-name" type="text" placeholder="اسم الطالب الثلاثي" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 text-lg font-bold">
                        <div class="grid grid-cols-2 gap-2">
                            <input id="stu-pin" type="password" placeholder="رمز الدخول" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 font-bold">
                            <input id="stu-confirm" type="password" placeholder="تأكيد الرمز" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 font-bold">
                        </div>
                        <button id="btn-stu-login" class="w-full py-4 bg-blue-600 text-white font-black rounded-2xl shadow-lg active:scale-95 transition-all text-lg">دخول الطالب</button>
                    </div>
                    <div class="mt-10 pt-6 border-t">
                        <button id="open-admin-modal" class="w-full py-3 bg-slate-50 text-slate-500 rounded-xl font-bold text-sm flex items-center justify-center gap-2 border border-slate-100 hover:bg-slate-50">
                            <i data-lucide="shield-check" size="16"></i> دخول المسؤول (إدارة النظام)
                        </button>
                    </div>
                </div>
            `);

            document.getElementById('btn-stu-login').onclick = async () => {
                const name = document.getElementById('stu-name').value.trim();
                const pin = document.getElementById('stu-pin').value;
                if (name.split(' ').length < 3) return notify('يرجى كتابة الاسم الثلاثي', true);
                if (pin !== document.getElementById('stu-confirm').value || pin.length < 4) return notify('تأكد من الرموز', true);
                
                const student = { uid: auth.currentUser.uid, name, role: 'student', stage: 'الثالثة', department: 'مختبرات طبية', pin: pin };
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'members', student.uid), student);
                localStorage.setItem('role', 'student');
                localStorage.setItem('userData', JSON.stringify(student));
                userRole = 'student'; userData = student; showStudent();
            };

            document.getElementById('open-admin-modal').onclick = () => document.getElementById('admin-modal').classList.add('active');
        };

        document.getElementById('adm-cancel').onclick = () => document.getElementById('admin-modal').classList.remove('active');
        document.getElementById('adm-confirm').onclick = () => {
            const u = document.getElementById('adm-user').value;
            const p = document.getElementById('adm-pass').value;
            if (u === "NTU" && p === "NTU_mlt12#45@56") {
                document.getElementById('admin-modal').classList.remove('active');
                localStorage.setItem('role', 'admin'); userRole = 'admin'; loadMembers(); showAdmin();
            } else notify('البيانات خاطئة', true);
        };

        const showStudent = () => {
            if (!userData) { goBackToHome(); return; }
            render(`
                <div class="bg-white rounded-[2.5rem] shadow-2xl border border-slate-100 overflow-hidden animate-in">
                    <div class="p-6 text-center border-b bg-slate-50/50">
                        <h2 class="text-xl font-black text-blue-800 mb-4">${userData.name}</h2>
                        <div class="flex justify-between items-center px-4">
                            <div class="text-right">
                                <span class="text-[10px] text-slate-400 block font-bold">المرحلة</span>
                                <span class="text-sm font-black text-slate-700">${userData.stage}</span>
                            </div>
                            <div class="text-left">
                                <span class="text-[10px] text-slate-400 block font-bold">القسم</span>
                                <span class="text-sm font-black text-slate-700">${userData.department}</span>
                            </div>
                        </div>
                    </div>
                    <div class="p-8 text-center flex flex-col items-center">
                        <div id="qrcode" class="qr-container mb-8"></div>
                        <p class="text-[10px] text-slate-400 font-bold mb-4 flex items-center gap-1"><i data-lucide="scan-line" size="12"></i> الباركود جاهز للمسح</p>
                        <button id="back-home-stu" class="w-full py-4 text-blue-600 border border-blue-50 rounded-2xl font-black flex items-center justify-center gap-2 bg-blue-50/30 active:scale-95 transition-all">
                            <i data-lucide="home"></i> عودة للقائمة الرئيسية
                        </button>
                    </div>
                </div>
            `);
            setTimeout(() => {
                const qrContainer = document.getElementById("qrcode");
                if (qrContainer) {
                    qrContainer.innerHTML = "";
                    new QRCode(qrContainer, { text: userData.uid, width: 220, height: 220, colorDark: "#1e40af", correctLevel: QRCode.CorrectLevel.L });
                }
            }, 50);
            document.getElementById('back-home-stu').onclick = goBackToHome;
        };

        const showAdmin = () => {
            let currentTab = 'scan';
            const draw = () => {
                render(`
                    <nav class="flex bg-white p-2 rounded-3xl shadow-xl gap-2 mb-6 sticky top-20 z-50 border border-slate-50">
                        <button id="tab-scan" class="flex-1 py-4 rounded-2xl font-black transition-all flex items-center justify-center gap-2 ${currentTab === 'scan' ? 'bg-blue-600 text-white shadow-lg' : 'text-slate-400'}"><i data-lucide="scan" size="18"></i> المسح</button>
                        <button id="tab-logs" class="flex-1 py-4 rounded-2xl font-black transition-all flex items-center justify-center gap-2 ${currentTab === 'logs' ? 'bg-indigo-600 text-white shadow-lg' : 'text-slate-400'}"><i data-lucide="list-checks" size="18"></i> الحضور</button>
                        <button id="tab-members" class="flex-1 py-4 rounded-2xl font-black transition-all flex items-center justify-center gap-2 ${currentTab === 'members' ? 'bg-green-600 text-white shadow-lg' : 'text-slate-400'}"><i data-lucide="users" size="18"></i> الطلاب</button>
                    </nav>
                    <div id="admin-content" class="bg-white p-6 rounded-[2.5rem] shadow-xl border border-slate-50 min-h-[480px]"></div>
                    <button id="back-home-admin" class="w-full mt-8 py-4 bg-white text-blue-600 rounded-2xl font-black flex justify-center gap-2 items-center border border-blue-100 active:scale-95 transition-all shadow-sm hover:bg-blue-50">
                        <i data-lucide="home"></i> عودة للقائمة الرئيسية
                    </button>
                `);
                document.getElementById('tab-scan').onclick = () => { currentTab = 'scan'; stopScanner(); draw(); };
                document.getElementById('tab-logs').onclick = () => { currentTab = 'logs'; stopScanner(); draw(); };
                document.getElementById('tab-members').onclick = () => { currentTab = 'members'; stopScanner(); draw(); };
                document.getElementById('back-home-admin').onclick = goBackToHome;

                if (currentTab === 'scan') drawScanner();
                else if (currentTab === 'logs') drawLogs();
                else drawMembers();
                lucide.createIcons();
            };

            const stopScanner = () => { if (html5QrCode) { try { html5QrCode.stop(); } catch(e) {} html5QrCode = null; } };

            const registerStudentAttendance = async (sId, subject) => {
                const realName = membersCache[sId];
                if (!realName) { notify("هذا الطالب غير مسجل في النظام!", true); return false; }

                const todayDate = new Date().toISOString().split('T')[0];
                const attendanceRef = collection(db, 'artifacts', appId, 'public', 'data', 'attendance');
                
                const q = query(attendanceRef, where("studentId", "==", sId), where("subject", "==", subject), where("date", "==", todayDate));
                const snapshot = await getDocs(q);

                if (!snapshot.empty) { notify(`مسجل مسبقاً: ${realName}`, true); return true; }

                notify(`تم الحضور: ${realName}`);
                if (navigator.vibrate) navigator.vibrate(80);

                await addDoc(attendanceRef, { 
                    studentId: sId, studentName: realName, subject: subject, 
                    date: todayDate, time: new Date().toLocaleTimeString('ar-EG'), timestamp: serverTimestamp() 
                });
                return true;
            };

            const drawScanner = () => {
                document.getElementById('admin-content').innerHTML = `
                    <div class="space-y-4">
                        <select id="sel-sub" class="w-full p-4 bg-slate-50 rounded-2xl border-none font-bold text-blue-700 outline-none focus:ring-2 ring-blue-500/20">
                            ${SUBJECTS.map(s => `<option>${s}</option>`).join('')}
                        </select>
                        <div id="reader-container" class="w-full aspect-square bg-slate-900 rounded-[2.5rem] overflow-hidden shadow-2xl border-4 border-white relative">
                             <div id="reader"></div>
                             <div class="absolute inset-0 border-[3rem] border-slate-900/40 pointer-events-none flex items-center justify-center">
                                <div class="w-64 h-64 border-2 border-white/50 rounded-3xl"></div>
                             </div>
                        </div>
                        <button id="start-scan" class="w-full py-5 bg-blue-600 text-white font-black rounded-2xl shadow-xl active:scale-95 transition-all text-lg flex items-center justify-center gap-2">
                             <i data-lucide="camera"></i> تشغيل الماسح الذكي
                        </button>
                        <button id="stop-scan" class="w-full py-5 bg-slate-100 text-slate-500 font-black rounded-2xl hidden flex items-center justify-center gap-2">إيقاف</button>
                    </div>
                `;
                lucide.createIcons();
                const btnStart = document.getElementById('start-scan');
                const btnStop = document.getElementById('stop-scan');

                btnStart.onclick = async () => {
                    const currentSub = document.getElementById('sel-sub').value;
                    btnStart.innerText = "جاري التحميل..."; btnStart.disabled = true;
                    btnStart.classList.add('hidden'); btnStop.classList.remove('hidden');
                    
                    html5QrCode = new Html5Qrcode("reader");
                    // Turbo Speed: 120 FPS for maximum responsiveness
                    await html5QrCode.start(
                        { facingMode: "environment" }, { fps: 120, qrbox: { width: 280, height: 280 }, aspectRatio: 1.0 }, 
                        async (decodedText) => {
                            if (isProcessing) return; isProcessing = true;
                            await registerStudentAttendance(decodedText, currentSub);
                            setTimeout(() => { isProcessing = false; }, 400); // Reduced delay for faster consecutive scans
                        }
                    ).catch(() => { btnStart.disabled = false; btnStart.classList.remove('hidden'); btnStop.classList.add('hidden'); notify("تعذر فتح الكاميرا", true); });
                };
                btnStop.onclick = () => { stopScanner(); btnStart.classList.remove('hidden'); btnStop.classList.add('hidden'); btnStart.disabled = false; btnStart.innerHTML = '<i data-lucide="camera"></i> تشغيل الماسح الذكي'; lucide.createIcons(); };
            };

            const drawLogs = () => {
                const date = new Date().toISOString().split('T')[0];
                document.getElementById('admin-content').innerHTML = `
                    <div class="space-y-4">
                        <div class="flex justify-between items-center mb-2 gap-2">
                             <h4 class="font-black text-slate-800 text-sm">سجل الحضور</h4>
                             <div class="flex gap-2">
                                <button id="btn-download" class="px-3 py-2 bg-green-50 text-green-600 rounded-xl text-xs font-black border border-green-100 flex items-center gap-1 active:scale-95 transition-transform">
                                    <i data-lucide="download" size="14"></i> تنزيل حضور
                                </button>
                                <button id="btn-manual-add" class="px-3 py-2 bg-blue-50 text-blue-600 rounded-xl text-xs font-black border border-blue-100 flex items-center gap-1 active:scale-95 transition-transform">
                                    <i data-lucide="plus-circle" size="14"></i> إضافة حضور
                                </button>
                             </div>
                        </div>
                        <div class="grid grid-cols-2 gap-2">
                            <select id="log-sub" class="p-3 bg-slate-50 rounded-xl border-none text-[10px] font-black outline-none">${SUBJECTS.map(s => `<option>${s}</option>`).join('')}</select>
                            <input id="log-date" type="date" value="${date}" class="p-3 bg-slate-50 rounded-xl border-none text-[10px] font-black outline-none">
                        </div>
                        <div id="attendance-list" class="space-y-2 max-h-[420px] overflow-y-auto no-scrollbar pb-10"></div>
                    </div>
                `;
                lucide.createIcons();

                document.getElementById('btn-manual-add').onclick = () => {
                    const sub = document.getElementById('log-sub').value;
                    document.getElementById('manual-subject-display').innerText = sub;
                    const select = document.getElementById('manual-student-select');
                    select.innerHTML = '<option value="">-- اختر الطالب --</option>';
                    Object.entries(membersCache).sort(([,a], [,b]) => a.localeCompare(b, 'ar')).forEach(([id, name]) => {
                        select.innerHTML += `<option value="${id}">${name}</option>`;
                    });
                    document.getElementById('manual-modal').classList.add('active');

                    document.getElementById('manual-save-btn').onclick = async () => {
                        const sId = select.value;
                        if (!sId) return notify("يرجى اختيار طالب", true);
                        document.getElementById('manual-save-btn').innerText = "جاري التسجيل السحابي...";
                        await registerStudentAttendance(sId, sub);
                        document.getElementById('manual-save-btn').innerText = "تأكيد الحضور السحابي";
                        document.getElementById('manual-modal').classList.remove('active');
                    };
                };
                document.getElementById('manual-close-btn').onclick = () => document.getElementById('manual-modal').classList.remove('active');

                const update = () => {
                    const sub = document.getElementById('log-sub').value;
                    const d = document.getElementById('log-date').value;
                    onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'attendance'), (snap) => {
                        const items = snap.docs.map(doc => ({id: doc.id, ...doc.data()}))
                            .filter(a => a.subject === sub && a.date === d).sort((a,b) => b.timestamp - a.timestamp);
                        
                        document.getElementById('btn-download').onclick = () => {
                            if (!items.length) return notify("لا توجد بيانات للتنزيل", true);
                            const bom = "\uFEFF"; 
                            const csvContent = bom + "اسم الطالب,الوقت,التاريخ,المادة\n" + 
                                items.map(e => `${e.studentName},${e.time},${e.date},${e.subject}`).join("\n");
                            const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
                            const url = URL.createObjectURL(blob);
                            const link = document.createElement("a");
                            link.setAttribute("href", url);
                            link.setAttribute("download", `حضور_${sub}_${d}.csv`);
                            document.body.appendChild(link);
                            link.click();
                            document.body.removeChild(link);
                            notify("تم تنزيل الحضور بنجاح");
                        };

                        document.getElementById('attendance-list').innerHTML = items.length ? items.map((a, i) => `
                            <div class="p-4 bg-slate-50 rounded-2xl flex justify-between items-center border-r-4 border-indigo-500 animate-in" style="animation-delay: ${i*0.05}s">
                                <div><p class="font-bold text-sm text-slate-800">${a.studentName}</p><p class="text-[10px] text-slate-400 font-bold flex items-center gap-1"><i data-lucide="clock" size="10"></i> ${a.time}</p></div>
                                <button class="del-log text-red-300 hover:text-red-500 p-2 transition-colors" data-id="${a.id}"><i data-lucide="trash-2" size="18"></i></button>
                            </div>
                        `).join('') : '<div class="text-center py-20 text-slate-300 text-sm font-bold italic flex flex-col items-center gap-2"><i data-lucide="clipboard-x" size="24"></i> القائمة فارغة</div>';
                        lucide.createIcons();
                        document.querySelectorAll('.del-log').forEach(b => b.onclick = () => deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'attendance', b.dataset.id)));
                    });
                };
                document.getElementById('log-sub').onchange = update;
                document.getElementById('log-date').onchange = update;
                update();
            };

            const drawMembers = () => {
                document.getElementById('admin-content').innerHTML = `<div class="space-y-4"><h4 class="font-black text-slate-800 flex items-center gap-2"><i data-lucide="users" class="text-green-600"></i> الطلاب المسجلين</h4><div id="members-list" class="space-y-2 max-h-[420px] overflow-y-auto no-scrollbar"></div></div>`;
                onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'members'), (snap) => {
                    const list = snap.docs.map(d => ({id: d.id, ...d.data()})).sort((a,b) => a.name.localeCompare(b.name, 'ar'));
                    document.getElementById('members-list').innerHTML = list.map(m => `
                        <div class="p-4 bg-slate-50 rounded-2xl flex justify-between items-center border hover:border-green-200 transition-colors">
                            <p class="font-bold text-sm text-slate-800">${m.name}</p>
                            <button class="del-mem text-red-300 hover:text-red-500 p-2" data-id="${m.id}"><i data-lucide="trash-2" size="18"></i></button>
                        </div>
                    `).join('') || '<p class="text-center py-10 text-slate-300 italic">لا يوجد طلاب</p>';
                    lucide.createIcons();
                    document.querySelectorAll('.del-mem').forEach(b => b.onclick = () => confirm('حذف الطالب نهائياً؟') && deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'members', b.dataset.id)));
                });
            };
            draw();
        };
    </script>
</body>
</html>
