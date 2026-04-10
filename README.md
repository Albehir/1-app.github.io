<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>البحير نت - نسخة مفتوحة</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-storage-compat.js"></script>

    <style>
        :root { --primary: #007bff; --accent: #6f42c1; --bg: #f0f2f5; --white: #ffffff; }
        body { font-family: 'Segoe UI', sans-serif; background-color: var(--bg); margin: 0; padding-bottom: 20px; }
        header { background: linear-gradient(135deg, var(--primary), var(--accent)); color: white; padding: 15px; text-align: center; position: sticky; top: 0; z-index: 1000; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .container { max-width: 550px; margin: auto; padding: 15px; }
        .card { background: var(--white); border-radius: 15px; padding: 20px; box-shadow: 0 4px 12px rgba(0,0,0,0.08); margin-bottom: 20px; }
        input, textarea { width: 100%; padding: 12px; margin: 8px 0; border: 1px solid #ddd; border-radius: 10px; font-size: 16px; outline: none; box-sizing: border-box; }
        .btn { width: 100%; padding: 12px; border: none; border-radius: 10px; cursor: pointer; font-weight: bold; background: var(--primary); color: white; font-size: 16px; }
        
        .post { background: var(--white); border-radius: 15px; padding: 15px; margin-bottom: 15px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); }
        .post-header { display: flex; align-items: center; margin-bottom: 10px; }
        .avatar { width: 40px; height: 40px; border-radius: 50%; background: var(--accent); color: white; display: flex; align-items: center; justify-content: center; font-weight: bold; margin-left: 10px; }
        .user-info b { display: block; font-size: 15px; }
        .post-content { font-size: 15px; line-height: 1.5; }
        .post-img { width: 100%; border-radius: 10px; margin-top: 10px; }
        .reactions-bar { display: flex; border-top: 1px solid #eee; margin-top: 10px; padding-top: 10px; justify-content: space-between; overflow-x: auto; gap: 5px; }
        .react-btn { background: #f8f9fa; border: 1px solid #eee; border-radius: 20px; padding: 5px 10px; cursor: pointer; font-size: 12px; }
    </style>
</head>
<body>

<header><h2>البحير نت (نسخة مفتوحة) 🌐</h2></header>

<div class="container">
    <div class="card">
        <h3>انشر الآن بدون حساب</h3>
        <input type="text" id="post-username" placeholder="اكتب اسمك المستعار هنا...">
        <textarea id="post-text" placeholder="ماذا يدور في ذهنك؟"></textarea>
        <input type="file" id="post-img" accept="image/*">
        <button class="btn" id="post-btn" onclick="createPost()">نشر كضيف</button>
    </div>

    <div id="feed"></div>
</div>

<script>
    const firebaseConfig = {
        apiKey: "AIzaSyAHbXMLT3F0OpnMZQHHPz-kAOlZBi1hhs4",
        authDomain: "albehirsocial.firebaseapp.com",
        projectId: "albehirsocial",
        storageBucket: "albehirsocial.firebasestorage.app",
        messagingSenderId: "302307189713",
        appId: "1:302307189713:web:abc71f50c9f87113c0f6f0"
    };

    firebase.initializeApp(firebaseConfig);
    const db = firebase.firestore();
    const storage = firebase.storage();

    // تشغيل جلب المنشورات فوراً عند فتح الموقع
    window.onload = loadPosts;

    async function createPost() {
        const username = document.getElementById('post-username').value.trim() || "زائر مجهول";
        const text = document.getElementById('post-text').value;
        const file = document.getElementById('post-img').files[0];
        if(!text && !file) return alert("اكتب شيئاً للنشر");

        const btn = document.getElementById('post-btn');
        btn.disabled = true;
        btn.innerText = "جاري النشر...";

        let url = "";
        if(file) {
            const ref = storage.ref(`public_posts/${Date.now()}`);
            await ref.put(file);
            url = await ref.getDownloadURL();
        }

        await db.collection("posts").add({
            text, url,
            username: username,
            reactions: { like: 0, haha: 0, support: 0, hate: 0 },
            time: firebase.firestore.FieldValue.serverTimestamp()
        });

        document.getElementById('post-text').value = "";
        document.getElementById('post-img').value = "";
        btn.disabled = false;
        btn.innerText = "نشر كضيف";
    }

    async function handleReact(postId, type, currentCount) {
        let updateData = {};
        updateData[`reactions.${type}`] = (currentCount || 0) + 1;
        await db.collection("posts").doc(postId).update(updateData);
    }

    function loadPosts() {
        db.collection("posts").orderBy("time", "desc").onSnapshot(snap => {
            const feed = document.getElementById('feed');
            feed.innerHTML = "";
            snap.forEach(doc => {
                const p = doc.data();
                const id = doc.id;
                const r = p.reactions || { like: 0, haha: 0, support: 0, hate: 0 };
                const time = p.time ? new Date(p.time.seconds*1000).toLocaleString('ar-EG') : 'الآن';

                feed.innerHTML += `
                    <div class="post">
                        <div class="post-header">
                            <div class="avatar">${p.username.charAt(0)}</div>
                            <div class="user-info"><b>${p.username}</b><span>${time}</span></div>
                        </div>
                        <div class="post-content">${p.text}</div>
                        ${p.url ? `<img src="${p.url}" class="post-img">` : ""}
                        
                        <div class="reactions-bar">
                            <button class="react-btn" onclick="handleReact('${id}', 'like', ${r.like})">👍 (${r.like})</button>
                            <button class="react-btn" onclick="handleReact('${id}', 'haha', ${r.haha})">😂 (${r.haha})</button>
                            <button class="react-btn" onclick="handleReact('${id}', 'support', ${r.support})">💪 (${r.support})</button>
                            <button class="react-btn" onclick="handleReact('${id}', 'hate', ${r.hate})">😡 (${r.hate})</button>
                        </div>
                    </div>`;
            });
        });
    }
</script>
</body>
</html>
