<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="theme-color" content="#1e40af">
    <title>نظام حضور NTU - الإصدار النهائي</title>
    
    <!-- Scripts -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
    <script src="https://unpkg.com/html5-qrcode"></script>
    
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+Arabic:wght@400;700;900&display=swap');
        body { font-family: 'Noto Sans Arabic', sans-serif; -webkit-tap-highlight-color: transparent; background-color: #f8fafc; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .qr-container canvas, .qr-container img { margin: 0 auto; border-radius: 1rem; border: 4px solid #fff; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        .animate-in { animation: fadeIn 0.2s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        #admin-modal { display: none; }
        #admin-modal.active { display: flex; }
    </style>
</head>
<body class="min-h-screen pb-10">

    <header class="p-4 bg-white shadow-sm border-b text-center sticky top-0 z-50">
        <h1 class="text-blue-700 font-bold text-lg">Northern Technical University</h1>
        <h2 class="text-red-600 font-semibold text-xs">كلية التقنيات الصحية والطبية - كركوك</h2>
    </header>

    <main class="max-w-xl mx-auto p-4" id="app-root">
        <!-- Content will be injected here -->
    </main>

    <!-- Admin Login Modal -->
    <div id="admin-modal" class="fixed inset-0 z-[100] bg-slate-900/60 backdrop-blur-sm items-center justify-center p-4">
        <div class="bg-white rounded-[2rem] p-8 w-full max-w-sm shadow-2xl animate-in">
            <h3 class="text-xl font-black mb-6 text-center text-slate-800">دخول الإدارة</h3>
            <div class="space-y-4">
                <input id="adm-user" type="text" placeholder="اسم المستخدم" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 font-bold">
                <input id="adm-pass" type="password" placeholder="كلمة المرور" class="w-full p-4 bg-slate-50 rounded-2xl border-none outline-none focus:ring-2 ring-blue-500/20 font-bold">
                <button id="adm-confirm" class="w-full py-4 bg-blue-600 text-white font-black rounded-2xl shadow-lg active:scale-95 transition-all">متابعة</button>
                <button id="adm-cancel" class="w-full py-2 text-slate-400 font-bold text-sm">إلغاء</button>
            </div>
        </div>
    </div>

    <script>
        // --- Database Logic ---
        const DB_NAME = 'NTU_Attendance_LocalDB_V6';
        let db;

        const initDB = () => {
            return new Promise((resolve) => {
                const request = indexedDB.open(DB_NAME, 1);
                request.onupgradeneeded = (e) => {
                    const database = e.target.result;
                    if (!database.objectStoreNames.contains('members')) database.createObjectStore('members', { keyPath: 'uid' });
                };
                request.onsuccess = (e) => { db = e.target.result; resolve(); };
            });
        };

        const dbOps = {
            addMember: (data) => new Promise(res => {
                const tx = db.transaction('members', 'readwrite');
                tx.objectStore('members').put(data);
                tx.oncomplete = () => res();
            })
        };

        let userRole = localStorage.getItem('role') || null;
        let userData = JSON.parse(localStorage.getItem('userData')) || null;

        const render = (html) => {
            document.getElementById('app-root').innerHTML = `<div class="animate-in">${html}</div>`;
            if (window.lucide) lucide.createIcons();
        };

        const showLogin = () => {
            render(`
                <div class="mt-8 bg-white p-8 rounded-[2.5rem] shadow-xl border border-slate-100 text-center">
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
                        <button id="open-admin-modal-btn" class="w-full py-3 bg-slate-50 text-slate-500 rounded-xl font-bold text-sm flex items-center justify-center gap-2 border border-slate-100">
                            <i data-lucide="shield-check" size="16"></i> دخول المسؤول (إدارة النظام)
                        </button>
                    </div>
                </div>
            `);

            // Fixing Student Login Logic
            document.getElementById('btn-stu-login').addEventListener('click', async () => {
                const name = document.getElementById('stu-name').value.trim();
                const pin = document.getElementById('stu-pin').value;
                const confirm = document.getElementById('stu-confirm').value;

                if (name.split(' ').length < 3) return alert('يرجى كتابة الاسم الثلاثي');
                if (!pin || pin !== confirm) return alert('الرموز غير متطابقة أو فارغة');
                
                const uid = 'STU_' + Math.random().toString(36).substr(2, 9);
                const student = { uid, name, role: 'student', stage: 'الثالثة', department: 'مختبرات طبية' };
                await dbOps.addMember(student);
                localStorage.setItem('role', 'student');
                localStorage.setItem('userData', JSON.stringify(student));
                window.location.reload();
            });

            // Fixing Admin Modal Open
            document.getElementById('open-admin-modal-btn').onclick = () => {
                document.getElementById('admin-modal').classList.add('active');
            };
        };

        const showStudent = () => {
            render(`
                <div class="bg-white rounded-[2.5rem] shadow-2xl border border-slate-100 overflow-hidden text-center">
                    <div class="p-6 border-b bg-slate-50/50">
                        <h2 class="text-xl font-black text-blue-800 mb-2">${userData.name}</h2>
                        <p class="text-sm font-bold text-slate-500">${userData.department} - ${userData.stage}</p>
                    </div>
                    <div class="p-8">
                        <div id="qrcode" class="qr-container mb-6"></div>
                        <p class="text-[11px] text-slate-400 font-bold mb-6 italic">الباركود يعمل بدون إنترنت</p>
                        <button id="logout-btn" class="w-full py-3 text-red-400 font-bold text-sm bg-red-50 rounded-xl">تسجيل خروج</button>
                    </div>
                </div>
            `);
            new QRCode(document.getElementById("qrcode"), { text: userData.uid, width: 220, height: 220, colorDark: "#1e40af" });
            document.getElementById('logout-btn').onclick = () => { localStorage.clear(); window.location.reload(); };
        };

        // Admin Modal Controls (Fixing Cancel & Confirm)
        document.getElementById('adm-cancel').onclick = () => {
            document.getElementById('admin-modal').classList.remove('active');
        };

        document.getElementById('adm-confirm').onclick = () => {
            const u = document.getElementById('adm-user').value;
            const p = document.getElementById('adm-pass').value;
            if (u === "NTU" && p === "NTU_mlt12#45@56") {
                localStorage.setItem('role', 'admin');
                window.location.reload();
            } else alert('خطأ في بيانات المسؤول');
        };

        window.onload = async () => {
            await initDB();
            if (userRole === 'admin') {
                render('<div class="p-10 text-center font-bold">لوحة الإدارة قيد التشغيل... تم تسجيل الدخول بنجاح.</div>');
                // Note: The admin dashboard code would go here in a full app
            } else if (userRole === 'student') {
                showStudent();
            } else {
                showLogin();
            }
        };
    </script>
</body>
</html>
