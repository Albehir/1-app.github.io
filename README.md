<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>البحير نت - الإصدار المطور</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-storage-compat.js"></script>

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">

    <style>
        :root { --primary: #1da1f2; --whatsapp: #25D366; --bg: #f0f2f5; }
        body { font-family: 'Segoe UI', sans-serif; background-color: var(--bg); margin: 0; }
        .container { max-width: 500px; margin: auto; padding: 15px; }
        
        /* البطاقات */
        .card { background: white; padding: 20px; border-radius: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); margin-bottom: 20px; }
        input, select, textarea { width: 100%; padding: 12px; margin: 8px 0; border: 1px solid #ddd; border-radius: 10px; box-sizing: border-box; }
        
        .btn { width: 100%; padding: 12px; border: none; border-radius: 10px; cursor: pointer; font-weight: bold; transition: 0.3s; }
        .btn-main { background: var(--primary); color: white; }
        .btn-wa { background: var(--whatsapp); color: white; }

        /* الشاشات */
        #auth-screen, #profile-setup, #main-app { display: none; }
        
        .post { background: white; padding: 15px; border-radius: 12px; margin-bottom: 15px; }
        .user-info { font-size: 12px; color: #65676b; margin-bottom: 10px; }
        .status-online { color: #2ecc71; font-size: 11px; }
    </style>
</head>
<body>

<div class="container">
    <div id="auth-screen" class="card" style="display: block;">
        <h2>البحير نت</h2>
        <input type="text" id="username" placeholder="اسم المستخدم (Username)">
        <input type="tel" id="phone" placeholder="+249XXXXXXXXX">
        <div id="recaptcha-container"></div>
        <button class="btn btn-wa" onclick="sendOTP()"><i class="fab fa-whatsapp"></i> دخول برقم الواتساب</button>
        
        <div id="otp-area" style="display:none;">
            <input type="text" id="otp" placeholder="أدخل كود التحقق">
            <button class="btn btn-main" onclick="verifyOTP()">تأكيد</button>
        </div>
    </div>

    <div id="profile-setup" class="card">
        <h3>أكمل ملفك الشخصي</h3>
        <input type="number" id="user-age" placeholder="العمر">
        <select id="user-city">
            <option value="">اختر المدينة...</option>
            <optgroup label="السودان">
                <option value="الخرطوم">الخرطوم</option>
                <option value="نيالا">نيالا</option>
                <option value="بورتسودان">بورتسودان</option>
            </optgroup>
            <optgroup label="دول أخرى">
                <option value="القاهرة">القاهرة</option>
                <option value="الرياض">الرياض</option>
                <option value="دبي">دبي</option>
            </optgroup>
        </select>
        <select id="user-status">
            <option value="أعزب">أعزب</option>
            <option value="متزوج">متزوج</option>
            <option value="خاطب">خاطب</option>
        </select>
        <button class="btn btn-main" onclick="saveProfile()">حفظ والذهاب للموقع</button>
    </div>

    <div id="main-app">
        <div class="card">
            <textarea id="post-text" placeholder="ماذا يدور في ذهنك؟"></textarea>
            <input type="file" id="post-img" accept="image/*">
            <button class="btn btn-main" onclick="createPost()">نشر</button>
        </div>
        <div id="feed"></div>
    </div>
</div>

<script>
    // إعدادات Firebase
    const firebaseConfig = {
        apiKey: "AIzaSy...", 
        authDomain: "albehir.firebaseapp.com",
        projectId: "albehir",
        storageBucket: "albehir.appspot.com",
        messagingSenderId: "...",
        appId: "..."
    };

    firebase.initializeApp(firebaseConfig);
    const auth = firebase.auth();
    const db = firebase.firestore();
    const storage = firebase.storage();

    // إعداد ReCaptcha
    window.recaptchaVerifier = new firebase.auth.RecaptchaVerifier('recaptcha-container', { 'size': 'invisible' });

    // الدخول
    function sendOTP() {
        const phone = document.getElementById('phone').value;
        auth.signInWithPhoneNumber(phone, window.recaptchaVerifier).then(result => {
            window.confirmationResult = result;
            document.getElementById('otp-area').style.display = 'block';
        });
    }

    function verifyOTP() {
        const code = document.getElementById('otp').value;
        confirmationResult.confirm(code).then(res => checkUser(res.user));
    }

    // التحقق من المستخدم والبيانات
    async function checkUser(user) {
        const doc = await db.collection("users").doc(user.uid).get();
        if (doc.exists) {
            updateLastSeen();
            showMain();
        } else {
            document.getElementById('auth-screen').style.display = 'none';
            document.getElementById('profile-setup').style.display = 'block';
        }
    }

    // حفظ الملف الشخصي
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

    // تحديث آخر ظهور
    function updateLastSeen() {
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

    // نظام المنشورات مع عرض البيانات الجديدة
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
    }

    function loadPosts() {
        db.collection("posts").orderBy("time", "desc").onSnapshot(snap => {
            const feed = document.getElementById('feed');
            feed.innerHTML = "";
            snap.forEach(doc => {
                const p = doc.data();
                const postDate = p.time ? new Date(p.time.seconds*1000).toLocaleString('ar-EG') : 'الآن';
                feed.innerHTML += `
                    <div class="post card">
                        <div class="user-info">
                            <strong>${p.username}</strong> من ${p.city} <br>
                            <span class="status-online">نُشر في: ${postDate}</span>
                        </div>
                        <p>${p.text}</p>
                        ${p.url ? `<img src="${p.url}" style="width:100%; border-radius:10px;">` : ""}
                    </div>`;
            });
        });
    }

    // تحديث التواجد كل دقيقة
    setInterval(updateLastSeen, 60000);
</script>

</body>
</html>
