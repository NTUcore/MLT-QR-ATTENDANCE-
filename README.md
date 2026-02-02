<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="theme-color" content="#1e40af">
    <title>نظام حضور NTU - الإصدار المصلح</title>
    
    <!-- Scripts -->
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
                <button id="manual-save-btn" class="w-full py-4 bg-green-600 text-white font-black rounded-2xl shadow-lg active:scale-95 transition-all">تأكيد الحضور</button>
                <button id="manual-close-btn" class="w-full py-3 text-slate-400 font-bold text-sm bg-slate-50 rounded-xl">إغلاق</button>
            </div>
        </div>
    </div>

    <script>
        // --- Database Logic ---
        const DB_NAME = 'NTU_Attendance_LocalDB_V4';
        const DB_VERSION = 1;
        let db;

        const initDB = () => {
            return new Promise((resolve, reject) => {
                const request = indexedDB.open(DB_NAME, DB_VERSION);
                request.onupgradeneeded = (e) => {
                    const database = e.target.result;
                    if (!database.objectStoreNames.contains('members')) database.createObjectStore('members', { keyPath: 'uid' });
                    if (!database.objectStoreNames.contains('attendance')) {
                        const store = database.createObjectStore('attendance', { keyPath: 'id', autoIncrement: true });
                        store.createIndex('by_date_subject', ['date', 'subject'], { unique: false });
                    }
                };
                request.onsuccess = (e) => { db = e.target.result; resolve(db); };
                request.onerror = (e) => reject(e);
            });
        };

        const dbOps = {
            addMember: (data) => new Promise((res) => {
                const tx = db.transaction('members', 'readwrite');
                tx.objectStore('members').put(data);
                tx.oncomplete = () => res();
            }),
            getAllMembers: () => new Promise((res) => {
                const tx = db.transaction('members', 'readonly');
                const req = tx.objectStore('members').getAll();
                req.onsuccess = () => res(req.result);
            }),
            deleteMember: (uid) => new Promise((res) => {
                const tx = db.transaction('members', 'readwrite');
                tx.objectStore('members').delete(uid);
                tx.oncomplete = () => res();
            }),
            addAttendance: (data) => new Promise((res) => {
                const tx = db.transaction('attendance', 'readwrite');
                tx.objectStore('attendance').add(data);
                tx.oncomplete = () => res();
            }),
            getAttendance: (subject, date) => new Promise((res) => {
                const tx = db.transaction('attendance', 'readonly');
                const req = tx.objectStore('attendance').getAll();
                req.onsuccess = () => {
                    const filtered = req.result.filter(a => a.subject === subject && a.date === date);
                    res(filtered);
                };
            }),
            checkAttendanceExist: (studentId, subject, date) => new Promise((res) => {
                const tx = db.transaction('attendance', 'readonly');
                const req = tx.objectStore('attendance').getAll();
                req.onsuccess = () => {
                    const exist = req.result.some(a => a.studentId === studentId && a.subject === subject && a.date === date);
                    res(exist);
                };
            }),
            deleteAttendance: (id) => new Promise((res) => {
                const tx = db.transaction('attendance', 'readwrite');
                tx.objectStore('attendance').delete(id);
                tx.oncomplete = () => res();
            })
        };

        // --- UI Logic ---
        const SUBJECTS = ["طرائق البحث", "الحشرات طبية (نظري)", "الحشرات طبية (عملي)", "نشاطات لا صفية", "ساعة ارشاد", "الكيمياء الحياتية السريرية (نظري)", "الكيمياء الحياتية (عملي)", "الفطريات (نظري)", "الفطريات (عملي)", "امراض الدم (نظري)", "امراض الدم (عملي)", "اللغه إنكليزية"];
        let userRole = localStorage.getItem('role') || null;
        let userData = JSON.parse(localStorage.getItem('userData')) || null;
        let html5QrCode = null;
        let membersCache = {};

        const render = (html) => {
            const root = document.getElementById('app-root');
            root.innerHTML = `<div class="animate-in">${html}</div>`;
            lucide.createIcons();
        };

        const notify = (msg, isError = false) => {
            const toast = document.createElement('div');
            toast.className = `fixed top-24 left-1/2 -translate-x-1/2 z-[110] px-6 py-3 rounded-2xl shadow-2xl text-white font-bold animate-in flex items-center gap-2 ${isError ? 'bg-red-500' : 'bg-green-600'}`;
            toast.innerHTML = `<i data-lucide="${isError ? 'alert-circle' : 'check-circle'}" size="18"></i> ${msg}`;
            document.body.appendChild(toast);
            lucide.createIcons();
            setTimeout(() => { toast.style.opacity = '0'; setTimeout(() => toast.remove(), 300); }, 1500);
        };

        const showLogin = () => {
            render(`
                <div class="mt-8 bg-white p-8 rounded-[2.5rem] shadow-xl border border-slate-100 animate-in text-center">
                    <h3 class="text-2xl font-black mb-1 text-slate-800">بوابة دخول</h3>
                    <p class="text-[11px] text-slate-400 font-bold mb-8 italic">يرجى انشاء حساب او تسجيل الدخول لتتمكن من تسجيل حضور</p>
                    <div class="space-y-4 text-right">
                        <input id="stu-name" type="text" placeholder="اسم الطالب الثلاثي" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 text-lg font-bold">
                        <div class="grid grid-cols-2 gap-2">
                            <input id="stu-pin" type="password" placeholder="رمز الدخول" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 font-bold">
                            <input id="stu-confirm" type="password" placeholder="تأكيد الرمز" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 font-bold">
                        </div>
                        <button id="btn-stu-login" class="w-full py-4 bg-blue-600 text-white font-black rounded-2xl shadow-lg active:scale-95 transition-all text-lg">دخول الطالب</button>
                    </div>
                    <div class="mt-10 pt-6 border-t">
                        <button id="open-admin-modal" class="w-full py-3 bg-slate-50 text-slate-500 rounded-xl font-bold text-sm flex items-center justify-center gap-2 border border-slate-100">
                            <i data-lucide="shield-check" size="16"></i> دخول المسؤول (إدارة النظام)
                        </button>
                    </div>
                </div>
            `);

            document.getElementById('btn-stu-login').onclick = async () => {
                const name = document.getElementById('stu-name').value.trim();
                const pin = document.getElementById('stu-pin').value;
                if (name.split(' ').length < 3) return notify('يرجى كتابة الاسم الثلاثي', true);
                if (pin !== document.getElementById('stu-confirm').value) return notify('الرموز غير متطابقة', true);
                
                const uid = 'STU_' + Math.random().toString(36).substr(2, 9);
                const student = { uid, name, role: 'student', stage: 'الثالثة', department: 'مختبرات طبية', pin: pin };
                await dbOps.addMember(student);
                localStorage.setItem('role', 'student');
                localStorage.setItem('userData', JSON.stringify(student));
                window.location.reload();
            };

            document.getElementById('open-admin-modal').onclick = () => {
                document.getElementById('admin-modal').classList.add('active');
            };
        };

        const showStudent = () => {
            render(`
                <div class="bg-white rounded-[2.5rem] shadow-2xl border border-slate-100 overflow-hidden animate-in">
                    <div class="p-6 text-center border-b bg-slate-50/50">
                        <h2 class="text-xl font-black text-blue-800 mb-4">${userData.name}</h2>
                        <div class="flex justify-between items-center px-4">
                            <div class="text-right"><span class="text-[10px] text-slate-400 block font-bold">المرحلة</span><span class="text-sm font-black text-slate-700">${userData.stage}</span></div>
                            <div class="text-left"><span class="text-[10px] text-slate-400 block font-bold">القسم</span><span class="text-sm font-black text-slate-700">${userData.department}</span></div>
                        </div>
                    </div>
                    <div class="p-8 text-center flex flex-col items-center">
                        <div id="qrcode" class="qr-container mb-8"></div>
                        <p class="text-[10px] text-slate-400 font-bold mb-4 flex items-center gap-1"><i data-lucide="scan-line" size="12"></i> الباركود جاهز للمسح</p>
                        <button id="logout-btn" class="w-full py-4 text-slate-400 border border-slate-200 rounded-2xl font-black flex items-center justify-center gap-2">
                             خروج
                        </button>
                    </div>
                </div>
            `);
            new QRCode(document.getElementById("qrcode"), { text: userData.uid, width: 220, height: 220, colorDark: "#1e40af" });
            document.getElementById('logout-btn').onclick = () => { localStorage.clear(); window.location.reload(); };
        };

        const showAdmin = () => {
            let currentTab = 'scan';
            const draw = () => {
                render(`
                    <nav class="flex bg-white p-2 rounded-3xl shadow-xl gap-2 mb-6 border">
                        <button id="tab-scan" class="flex-1 py-4 rounded-2xl font-black ${currentTab === 'scan' ? 'bg-blue-600 text-white' : 'text-slate-400'}">المسح</button>
                        <button id="tab-logs" class="flex-1 py-4 rounded-2xl font-black ${currentTab === 'logs' ? 'bg-blue-600 text-white' : 'text-slate-400'}">الحضور</button>
                    </nav>
                    <div id="admin-content" class="bg-white p-6 rounded-[2.5rem] shadow-xl min-h-[400px]"></div>
                    <button id="admin-logout" class="w-full mt-4 py-3 text-red-500 font-bold">تسجيل خروج الإدارة</button>
                `);
                document.getElementById('tab-scan').onclick = () => { currentTab = 'scan'; draw(); };
                document.getElementById('tab-logs').onclick = () => { currentTab = 'logs'; draw(); };
                document.getElementById('admin-logout').onclick = () => { localStorage.clear(); window.location.reload(); };

                if (currentTab === 'scan') drawScanner();
                else drawLogs();
            };

            const drawScanner = () => {
                document.getElementById('admin-content').innerHTML = `
                    <div class="space-y-4">
                        <select id="sel-sub" class="w-full p-4 bg-slate-50 rounded-2xl border-none font-bold text-blue-700">${SUBJECTS.map(s => `<option>${s}</option>`).join('')}</select>
                        <div id="reader" class="rounded-3xl overflow-hidden border-4 border-white"></div>
                        <button id="start-scan" class="w-full py-5 bg-blue-600 text-white font-black rounded-2xl shadow-xl">بدء الكاميرا</button>
                    </div>
                `;
                document.getElementById('start-scan').onclick = async () => {
                    html5QrCode = new Html5Qrcode("reader");
                    await html5QrCode.start({ facingMode: "environment" }, { fps: 10, qrbox: 250 }, async (text) => {
                        const sub = document.getElementById('sel-sub').value;
                        const date = new Date().toISOString().split('T')[0];
                        const exists = await dbOps.checkAttendanceExist(text, sub, date);
                        if (!exists) {
                            const members = await dbOps.getAllMembers();
                            const student = members.find(m => m.uid === text);
                            if (student) {
                                await dbOps.addAttendance({ studentId: text, studentName: student.name, subject: sub, date: date, time: new Date().toLocaleTimeString('ar-EG') });
                                notify(`تم تسجيل: ${student.name}`);
                            } else notify('طالب غير مسجل', true);
                        } else notify('مسجل مسبقاً', true);
                    });
                };
            };

            const drawLogs = async () => {
                const sub = SUBJECTS[0];
                const date = new Date().toISOString().split('T')[0];
                const list = await dbOps.getAttendance(sub, date);
                document.getElementById('admin-content').innerHTML = `
                    <div class="space-y-4">
                        <h4 class="font-black text-center border-b pb-2">سجل اليوم</h4>
                        <div id="log-list" class="space-y-2">
                            ${list.map(a => `<div class="p-3 bg-slate-50 rounded-xl flex justify-between"><span>${a.studentName}</span><span class="text-slate-400 text-xs">${a.time}</span></div>`).join('') || 'لا يوجد حضور'}
                        </div>
                    </div>
                `;
            };
            draw();
        };

        // --- Global Listeners ---
        document.getElementById('adm-confirm').onclick = () => {
            const u = document.getElementById('adm-user').value;
            const p = document.getElementById('adm-pass').value;
            if (u === "NTU" && p === "NTU_mlt12#45@56") {
                localStorage.setItem('role', 'admin');
                window.location.reload();
            } else notify('خطأ في البيانات', true);
        };

        // FIX: Cancel button now properly hides the modal
        document.getElementById('adm-cancel').onclick = () => {
            document.getElementById('admin-modal').classList.remove('active');
            // Clear fields
            document.getElementById('adm-user').value = "";
            document.getElementById('adm-pass').value = "";
        };

        window.onload = async () => {
            await initDB();
            if (userRole === 'admin') showAdmin();
            else if (userRole === 'student') showStudent();
            else showLogin();
        };
    </script>
</body>
</html>
