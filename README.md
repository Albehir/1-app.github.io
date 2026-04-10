<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>البحير نت - تسجيل بسيط</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-storage-compat.js"></script>

    <style>
        :root { --primary: #1da1f2; --bg: #f0f2f5; }
        body { font-family: 'Segoe UI', sans-serif; background-color: var(--bg); margin: 0; }
        .container { max-width: 500px; margin: auto; padding: 15px; }
        
        .card { background: white; padding: 20px; border-radius: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); margin-bottom: 20px; text-align: center; }
        input, select, textarea { width: 100%; padding: 12px; margin: 8px 0; border: 1px solid #ddd; border-radius: 10px; box-sizing: border-box; }
        
        .btn { width: 100%; padding: 12px; border: none; border-radius: 10px; cursor: pointer; font-weight: bold; background: var(--primary); color: white; transition: 0.3s; }
        
        #auth-screen, #profile-setup, #main-app { display: none; }
        
        .post { background: white; padding: 15px; border-radius: 12px; margin-bottom: 15px; box-shadow: 0 2px 5px rgba(0,0,0,0.05); }
        .user-meta { font-size: 12px; color: #65676b; margin-bottom: 10px; border-bottom: 1px solid #eee; padding-bottom: 5px; }
    </style>
</head>
<body>

<div class="container">
    <div id="auth-screen" class="card" style="display: block;">
        <h2 style="color:var(--primary)">البحير نت</h2>
        <p>أدخل بياناتك للانضمام إلينا</p>
        <input type="text" id="username" placeholder="اختر اسم مستخدم">
        <input type="tel" id="phone" placeholder="رقم الهاتف (+249...)">
        <div id="recaptcha-container"></div>
        <button class="btn" onclick="sendOTP()">دخول</button>
        
        <div id="otp-area" style="display:none; margin-top: 15px;">
            <input type="text" id="otp" placeholder="كود التحقق">
            <button class="btn" onclick="verifyOTP()" style="background:#333;">تأكيد الكود</button>
        </div>
    </div>

    <div id="profile-setup" class="card">
        <h3>أكمل بياناتك الشخصية</h3>
        <input type="number" id="user-age" placeholder="العمر">
        <select id="user-city">
            <option value="">اختر مدينتك...</option>
            <option value="الخرطوم">الخرطوم</option>
            <option value="نيالا">نيالا</option>
            <option value="بورتسودان">بورتسودان</option>
            <option value="ود مدني">ود مدني</option>
            <option value="أخرى">مدينة أخرى</option>
        </select>
        <select id="user-status">
            <option value="أعزب">أعزب</option>
            <option value="متزوج">متزوج</option>
            <option value="خاطب">خاطب</option>
        </select>
        <button class="btn" onclick="saveProfile()">حفظ البيانات والدخول</button>
    </div>

    <div id="main-app">
        <div class="card">
            <textarea id="post-text" placeholder="ماذا تريد أن تنشر اليوم؟"></textarea>
            <input type="file" id="post-img" accept="image/*">
            <button class="btn" onclick="createPost()">نشر الآن</button>
        </div>
        <div id="feed"></div>
    </div>
</div>

<script>
    // إعدادات Firebase
    const firebaseConfig = {
        apiKey: " 
  apiKey: "AIzaSyAHbXMLT3F0OpnMZQHHPz-kAOlZBi1hhs4",
  authDomain: "albehirsocial.firebaseapp.com",
  databaseURL: "https://albehirsocial-default-rtdb.firebaseio.com",
  projectId: "albehirsocial",
  storageBucket: "albehirsocial.firebasestorage.app",
  messagingSenderId: "302307189713",
  appId: "1:302307189713:web:abc71f50c9f87113c0f6f0",
  measurementId: "G-0Y1D7BFD37" ",
        authDomain: "albehir-net.firebaseapp.com",
        projectId: "albehir-net",
        storageBucket: "albehir-net.appspot.com",
        messagingSenderId: "...",
        appId: "..."
    };

    firebase.initializeApp(firebaseConfig);
    const auth = firebase.auth();
    const db = firebase.firestore();
    const storage = firebase.storage();

    window.recaptchaVerifier = new firebase.auth.RecaptchaVerifier('recaptcha-container', { 'size': 'invisible' });

    function sendOTP() {
        const phone = document.getElementById('phone').value;
        const user = document.getElementById('username').value;
        if(!user) return alert("يرجى كتابة اسم المستخدم أولاً");
        
        auth.signInWithPhoneNumber(phone, window.recaptchaVerifier).then(result => {
            window.confirmationResult = result;
            document.getElementById('otp-area').style.display = 'block';
            alert("تم إرسال كود التحقق لهاتفك");
        }).catch(err => alert("خطأ: " + err.message));
    }

    function verifyOTP() {
        const code = document.getElementById('otp').value;
        confirmationResult.confirm(code).then(res => checkUser(res.user));
    }

    async function checkUser(user) {
        const doc = await db.collection("users").doc(user.uid).get();
        if (doc.exists) {
            updateActivity();
            showMain();
        } else {
            document.getElementById('auth-screen').style.display = 'none';
            document.getElementById('profile-setup').style.display = 'block';
        }
    }

    async function saveProfile() {
        const user = auth.currentUser;
        const data = {
            username: document.getElementById('username').value,
            age: document.getElementById('user-age').value,
            city: document.getElementById('user-city').value,
            socialStatus: document.getElementById('user-status').value,
            lastSeen: firebase.firestore.FieldValue.serverTimestamp()
        };
        await db.collection("users").doc(user.uid).set(data);
        showMain();
    }

    function updateActivity() {
        if (auth.currentUser) {
            db.collection("users").doc(auth.currentUser.uid).update({
                lastSeen: firebase.firestore.FieldValue.serverTimestamp()
            });
        }
    }

    function showMain() {
        document.getElementById('auth-screen').style.display = 'none';
        document.getElementById('profile-setup').style.display = 'none';
        document.getElementById('main-app').style.display = 'block';
        loadPosts();
    }

    async function createPost() {
        const text = document.getElementById('post-text').value;
        const file = document.getElementById('post-img').files[0];
        const userDoc = await db.collection("users").doc(auth.currentUser.uid).get();
        const userData = userDoc.data();

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
    }

    function loadPosts() {
        db.collection("posts").orderBy("time", "desc").onSnapshot(snap => {
            const feed = document.getElementById('feed');
            feed.innerHTML = "";
            snap.forEach(doc => {
                const p = doc.data();
                const timeStr = p.time ? new Date(p.time.seconds*1000).toLocaleString('ar-EG') : 'الآن';
                feed.innerHTML += `
                    <div class="post">
                        <div class="user-meta">
                            <strong>${p.username}</strong> | ${p.city} | ${timeStr}
                        </div>
                        <p>${p.text}</p>
                        ${p.url ? `<img src="${p.url}" style="width:100%; border-radius:8px;">` : ""}
                    </div>`;
            });
        });
    }

    setInterval(updateActivity, 60000);
</script>

</body>
</html>
