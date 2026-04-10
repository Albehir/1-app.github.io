<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>البحير نت - تفاعلات كاملة</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-storage-compat.js"></script>

    <style>
        :root { --primary: #007bff; --accent: #6f42c1; --bg: #f0f2f5; --white: #ffffff; --danger: #dc3545; }
        body { font-family: 'Segoe UI', sans-serif; background-color: var(--bg); margin: 0; padding-bottom: 50px; }
        header { background: linear-gradient(135deg, var(--primary), var(--accent)); color: white; padding: 15px; text-align: center; position: sticky; top: 0; z-index: 1000; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .container { max-width: 550px; margin: auto; padding: 15px; }
        .card { background: var(--white); border-radius: 15px; padding: 20px; box-shadow: 0 4px 12px rgba(0,0,0,0.08); margin-bottom: 20px; }
        input, select, textarea { width: 100%; padding: 12px; margin: 10px 0; border: 1px solid #ddd; border-radius: 10px; font-size: 16px; outline: none; box-sizing: border-box; }
        .btn { width: 100%; padding: 12px; border: none; border-radius: 10px; cursor: pointer; font-weight: bold; background: var(--primary); color: white; font-size: 16px; transition: 0.3s; }
        
        .post { background: var(--white); border-radius: 15px; padding: 15px; margin-bottom: 15px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); position: relative; }
        .post-header { display: flex; align-items: center; margin-bottom: 10px; }
        .avatar { width: 40px; height: 40px; border-radius: 50%; background: var(--accent); color: white; display: flex; align-items: center; justify-content: center; font-weight: bold; margin-left: 10px; }
        .user-info b { display: block; font-size: 15px; }
        .user-info span { font-size: 11px; color: #888; }
        .post-content { font-size: 15px; line-height: 1.5; margin-bottom: 10px; }
        .post-img { width: 100%; border-radius: 10px; margin-top: 10px; }

        /* أزرار التفاعلات الجديدة */
        .reactions-bar { display: flex; border-top: 1px solid #eee; margin-top: 10px; padding-top: 10px; justify-content: space-between; overflow-x: auto; gap: 5px; }
        .react-btn { background: #f8f9fa; border: 1px solid #eee; border-radius: 20px; padding: 5px 10px; cursor: pointer; font-size: 13px; display: flex; align-items: center; gap: 4px; transition: 0.2s; white-space: nowrap; }
        .react-btn:hover { background: #e9ecef; }
        .react-btn.active { border-color: var(--primary); background: #e7f3ff; color: var(--primary); }
        .delete-btn { position: absolute; left: 15px; top: 15px; color: #ccc; cursor: pointer; font-size: 11px; border: none; background: none; }

        #auth-screen, #profile-setup, #main-app { display: none; }
    </style>
</head>
<body>

<header><h2>البحير نت 🌐</h2></header>

<div class="container">
    <div id="auth-screen" class="card" style="display: block;">
        <h3>دخول</h3>
        <input type="text" id="login-username" placeholder="اسم المستخدم">
        <input type="password" id="login-password" placeholder="كلمة المرور">
        <button class="btn" onclick="handleAuth()">دخول</button>
    </div>

    <div id="profile-setup" class="card">
        <h3>بياناتك</h3>
        <input type="number" id="user-age" placeholder="العمر">
        <select id="user-city">
            <option value="نيالا">نيالا</option>
            <option value="الخرطوم">الخرطوم</option>
            <option value="بورتسودان">بورتسودان</option>
        </select>
        <button class="btn" onclick="saveProfile()">ابدأ</button>
    </div>

    <div id="main-app">
        <div class="card">
            <textarea id="post-text" placeholder="ماذا يدور في ذهنك؟"></textarea>
            <input type="file" id="post-img" accept="image/*">
            <button class="btn" id="post-btn" onclick="createPost()">نشر</button>
        </div>
        <div id="feed"></div>
    </div>
</div>

<script>
    // إعداداتك الخاصة
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
        if(!user || !pass) return alert("املا الحقول");
        if(pass.length < 6) pass = pass.padEnd(6, "0"); 
        const email = user + "@albehir.net";

        auth.signInWithEmailAndPassword(email, pass).catch(e => {
            if (e.code === 'auth/user-not-found') auth.createUserWithEmailAndPassword(email, pass);
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
        await db.collection("users").doc(auth.currentUser.uid).set({
            username: document.getElementById('login-username').value,
            age: document.getElementById('user-age').value,
            city: document.getElementById('user-city').value
        });
        showScreen('main-app');
    }

    async function createPost() {
        const text = document.getElementById('post-text').value;
        const file = document.getElementById('post-img').files[0];
        if(!text && !file) return;

        let url = "";
        if(file) {
            const ref = storage.ref(`posts/${Date.now()}`);
            await ref.put(file);
            url = await ref.getDownloadURL();
        }

        const userData = (await db.collection("users").doc(auth.currentUser.uid).get()).data();
        await db.collection("posts").add({
            text, url,
            uid: auth.currentUser.uid,
            username: userData.username,
            city: userData.city,
            reactions: { like: [], haha: [], support: [], hate: [] }, // نظام التفاعلات الجديد
            time: firebase.firestore.FieldValue.serverTimestamp()
        });

        document.getElementById('post-text').value = "";
        document.getElementById('post-img').value = "";
    }

    async function handleReact(postId, type, currentReactions) {
        const myUid = auth.currentUser.uid;
        let reactions = currentReactions || { like: [], haha: [], support: [], hate: [] };

        // إزالة المستخدم من أي تفاعل آخر أولاً
        for (let key in reactions) {
            reactions[key] = reactions[key].filter(id => id !== myUid);
        }

        // إضافة التفاعل الجديد
        reactions[type].push(myUid);
        
        await db.collection("posts").doc(postId).update({ reactions: reactions });
    }

    async function deletePost(postId) {
        if(confirm("حذف المنشور؟")) await db.collection("posts").doc(postId).delete();
    }

    function loadPosts() {
        db.collection("posts").orderBy("time", "desc").onSnapshot(snap => {
            const feed = document.getElementById('feed');
            feed.innerHTML = "";
            snap.forEach(doc => {
                const p = doc.data();
                const id = doc.id;
                const r = p.reactions || { like: [], haha: [], support: [], hate: [] };
                const myUid = auth.currentUser.uid;

                feed.innerHTML += `
                    <div class="post">
                        ${p.uid === myUid ? `<button class="delete-btn" onclick="deletePost('${id}')">حذف</button>` : ''}
                        <div class="post-header">
                            <div class="avatar">${p.username.charAt(0)}</div>
                            <div class="user-info"><b>${p.username}</b><span>${p.city}</span></div>
                        </div>
                        <div class="post-content">${p.text}</div>
                        ${p.url ? `<img src="${p.url}" class="post-img">` : ""}
                        
                        <div class="reactions-bar">
                            <button class="react-btn ${r.like.includes(myUid)?'active':''}" onclick='handleReact("${id}", "like", ${JSON.stringify(r)})'>
                                👍 لايك (${r.like.length})
                            </button>
                            <button class="react-btn ${r.haha.includes(myUid)?'active':''}" onclick='handleReact("${id}", "haha", ${JSON.stringify(r)})'>
                                😂 هاها (${r.haha.length})
                            </button>
                            <button class="react-btn ${r.support.includes(myUid)?'active':''}" onclick='handleReact("${id}", "support", ${JSON.stringify(r)})'>
                                💪 أدعمه (${r.support.length})
                            </button>
                            <button class="react-btn ${r.hate.includes(myUid)?'active':''}" onclick='handleReact("${id}", "hate", ${JSON.stringify(r)})'>
                                😡 أكرهه (${r.hate.length})
                            </button>
                        </div>
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
