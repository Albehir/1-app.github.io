<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>البحير نت</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-storage-compat.js"></script>

    <style>
        :root { --primary: #1da1f2; --bg: #f0f2f5; }
        body { font-family: 'Segoe UI', sans-serif; background-color: var(--bg); margin: 0; padding: 0; }
        .container { max-width: 500px; margin: auto; padding: 15px; }
        .card { background: white; padding: 20px; border-radius: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); margin-bottom: 20px; text-align: center; }
        input, select, textarea { width: 100%; padding: 12px; margin: 8px 0; border: 1px solid #ddd; border-radius: 10px; box-sizing: border-box; font-size: 16px; outline: none; }
        .btn { width: 100%; padding: 12px; border: none; border-radius: 10px; cursor: pointer; font-weight: bold; background: var(--primary); color: white; font-size: 16px; transition: 0.3s; }
        .btn:hover { background: #1991db; }
        #auth-screen, #profile-setup, #main-app { display: none; }
        .post { background: white; padding: 15px; border-radius: 12px; margin-bottom: 15px; box-shadow: 0 2px 5px rgba(0,0,0,0.05); text-align: right; }
        .user-meta { font-size: 12px; color: #65676b; margin-bottom: 8px; border-bottom: 1px solid #eee; padding-bottom: 5px; }
        img.post-img { width: 100%; border-radius: 10px; margin-top: 10px; }
    </style>
</head>
<body>

<div class="container">
    <div id="auth-screen" class="card" style="display: block;">
        <h2 style="color:var(--primary); margin:0 0 10px 0;">البحير نت</h2>
        <p>ادخل أي اسم مستخدم وأي كلمة سر</p>
        <input type="text" id="login-username" placeholder="اسم المستخدم">
        <input type="text" id="login-password" placeholder="كلمة المرور (أي شيء)">
        <button class="btn" onclick="handleAuth()">دخول / تسجيل</button>
    </div>

    <div id="profile-setup" class="card">
        <h3>بياناتك الشخصية</h3>
        <input type="number" id="user-age" placeholder="العمر">
        <select id="user-city">
            <option value="">اختر مدينتك...</option>
            <option value="الخرطوم">الخرطوم</option>
            <option value="نيالا">نيالا</option>
            <option value="بورتسودان">بورتسودان</option>
            <option value="أخرى">مدينة أخرى</option>
        </select>
        <select id="user-status">
            <option value="أعزب">أعزب</option>
            <option value="متزوج">متزوج</option>
        </select>
        <button class="btn" onclick="saveProfile()">حفظ وافتتاح الحساب</button>
    </div>

    <div id="main-app">
        <div class="card">
            <textarea id="post-text" placeholder="ماذا يدور في ذهنك؟"></textarea>
            <input type="file" id="post-img" accept="image/*">
            <button class="btn" onclick="createPost()">نشر</button>
        </div>
        <div id="feed"></div>
    </div>
</div>

<script>
    // تم تحديث هذه القيم بناءً على طلبك
    const firebaseConfig = {
        apiKey: "AIzaSyAHbXMLT3F0OpnMZQHHPz-kAOlZBi1hhs4",
        authDomain: "albehirsocial.firebaseapp.com",
        databaseURL: "https://albehirsocial-default-rtdb.firebaseio.com",
        projectId: "albehirsocial",
        storageBucket: "albehirsocial.firebasestorage.app",
        messagingSenderId: "302307189713",
        appId: "1:302307189713:web:abc71f50c9f87113c0f6f0"
    };

    firebase.initializeApp(firebaseConfig);
    const auth = firebase.auth();
    const db = firebase.firestore();
    const storage = firebase.storage();

    function handleAuth() {
        const user = document.getElementById('login-username').value.trim();
        let pass = document.getElementById('login-password').value;

        if(!user || !pass) return alert("يرجى ملء الحقول");

        // خدعة برمجية لتخطي شرط الـ 6 خانات
        if(pass.length < 6) pass = pass.padEnd(6, "0"); 

        const fakeEmail = user + "@albehir.net";

        auth.signInWithEmailAndPassword(fakeEmail, pass)
            .catch(error => {
                if (error.code === 'auth/user-not-found' || error.code === 'auth/wrong-password') {
                    auth.createUserWithEmailAndPassword(fakeEmail, pass).catch(e => alert("خطأ: " + e.message));
                }
            });
    }

    auth.onAuthStateChanged(async (user) => {
        if (user) {
            const doc = await db.collection("users").doc(user.uid).get();
            if (doc.exists) { showScreen('main-app'); loadPosts(); } 
            else { showScreen('profile-setup'); }
        } else { showScreen('auth-screen'); }
    });

    async function saveProfile() {
        const user = auth.currentUser;
        const username = document.getElementById('login-username').value;
        await db.collection("users").doc(user.uid).set({
            username: username,
            age: document.getElementById('user-age').value,
            city: document.getElementById('user-city').value,
            status: document.getElementById('user-status').value,
            lastSeen: firebase.firestore.FieldValue.serverTimestamp()
        });
        showScreen('main-app');
    }

    async function createPost() {
        const text = document.getElementById('post-text').value;
        const file = document.getElementById('post-img').files[0];
        if(!text && !file) return;

        try {
            const userData = (await db.collection("users").doc(auth.currentUser.uid).get()).data();
            let url = "";
            if(file) {
                const ref = storage.ref(`posts/${Date.now()}`);
                await ref.put(file);
                url = await ref.getDownloadURL();
            }

            await db.collection("posts").add({
                text, url,
                username: userData.username,
                city: userData.city,
                time: firebase.firestore.FieldValue.serverTimestamp()
            });
            document.getElementById('post-text').value = "";
            document.getElementById('post-img').value = "";
        } catch (e) { alert("تأكد من تفعيل Firestore و Storage في Firebase"); }
    }

    function loadPosts() {
        db.collection("posts").orderBy("time", "desc").onSnapshot(snap => {
            const feed = document.getElementById('feed');
            feed.innerHTML = "";
            snap.forEach(doc => {
                const p = doc.data();
                const time = p.time ? new Date(p.time.seconds*1000).toLocaleString('ar-EG') : 'الآن';
                feed.innerHTML += `
                    <div class="post">
                        <div class="user-meta"><strong>${p.username}</strong> من ${p.city} | ${time}</div>
                        <div>${p.text}</div>
                        ${p.url ? `<img src="${p.url}" class="post-img">` : ""}
                    </div>`;
            });
        });
    }

    function showScreen(id) {
        document.getElementById('auth-screen').style.display = 'none';
        document.getElementById('profile-setup').style.display = 'none';
        document.getElementById('main-app').style.display = 'none';
        document.getElementById(id).style.display = 'block';
    }
</script>

</body>
</html>
