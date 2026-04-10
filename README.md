<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>البحير نت - النسخة الكاملة</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-storage-compat.js"></script>

    <style>
        :root { --primary: #007bff; --accent: #6f42c1; --bg: #f0f2f5; --white: #ffffff; --text: #1c1e21; }
        body { font-family: 'Segoe UI', sans-serif; background-color: var(--bg); margin: 0; color: var(--text); padding-bottom: 30px; }
        header { background: linear-gradient(135deg, var(--primary), var(--accent)); color: white; padding: 15px; text-align: center; position: sticky; top: 0; z-index: 1000; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .container { max-width: 550px; margin: auto; padding: 10px; }
        .card { background: var(--white); border-radius: 15px; padding: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.08); margin-bottom: 20px; }
        
        textarea { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 10px; font-size: 16px; outline: none; box-sizing: border-box; resize: none; }
        .file-label { display: inline-block; background: #e4e6eb; padding: 8px 15px; border-radius: 20px; cursor: pointer; margin-top: 10px; font-size: 14px; font-weight: 600; }
        .btn-post { width: 100%; padding: 12px; border: none; border-radius: 10px; cursor: pointer; font-weight: bold; background: var(--primary); color: white; font-size: 16px; margin-top: 10px; }

        .post { background: var(--white); border-radius: 15px; padding: 15px; margin-bottom: 15px; position: relative; }
        .post-header { display: flex; align-items: center; margin-bottom: 10px; justify-content: space-between; }
        .user-meta { display: flex; align-items: center; }
        .avatar { width: 40px; height: 40px; border-radius: 50%; background: #ccc; margin-left: 10px; display: flex; align-items: center; justify-content: center; color: white; font-weight: bold; background: linear-gradient(45deg, var(--primary), var(--accent)); }
        .post-time { font-size: 11px; color: #65676b; display: block; }

        .post-content { font-size: 15px; margin: 10px 0; white-space: pre-wrap; }
        .media-box img, .media-box video { width: 100%; border-radius: 10px; margin-top: 5px; max-height: 400px; object-fit: cover; }

        .reactions-bar { display: flex; border-top: 1px solid #eee; margin-top: 10px; padding-top: 10px; justify-content: space-around; flex-wrap: wrap; gap: 5px; }
        .react-btn { background: #f0f2f5; border: none; border-radius: 15px; padding: 5px 10px; cursor: pointer; font-size: 12px; transition: 0.2s; }
        .react-btn.active { background: #e7f3ff; color: var(--primary); }

        .post-footer { display: flex; border-top: 1px solid #eee; margin-top: 10px; padding-top: 10px; gap: 10px; }
        .footer-btn { flex: 1; background: none; border: none; color: #65676b; font-size: 13px; font-weight: 600; cursor: pointer; padding: 5px; border-radius: 5px; }
        .footer-btn:hover { background: #f2f2f2; }

        .menu-dots { cursor: pointer; padding: 5px; font-weight: bold; }
        #loading-screen { text-align: center; padding: 50px; font-weight: bold; color: var(--primary); }
    </style>
</head>
<body>

<header><h2>البحير نت 🌐</h2></header>

<div class="container">
    <div id="loading-screen">جاري التحميل...</div>

    <div id="main-app" style="display:none;">
        <div class="card">
            <textarea id="post-text" placeholder="ماذا يدور في ذهنك؟"></textarea>
            <label class="file-label">📸 اختر صورة أو فيديو
                <input type="file" id="post-media" accept="image/*,video/*" style="display:none">
            </label>
            <button class="btn-post" id="post-btn" onclick="createPost()">نشر الآن</button>
        </div>
        <div id="feed"></div>
    </div>
</div>

<script>
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

    // دخول تلقائي
    auth.onAuthStateChanged(async (user) => {
        if (!user) {
            await auth.signInAnonymously();
        } else {
            document.getElementById('loading-screen').style.display = 'none';
            document.getElementById('main-app').style.display = 'block';
            loadPosts();
        }
    });

    // النشر (دعم الصور والفيديو)
    async function createPost() {
        const text = document.getElementById('post-text').value;
        const file = document.getElementById('post-media').files[0];
        if(!text && !file) return;

        const btn = document.getElementById('post-btn');
        btn.disabled = true;
        btn.innerText = "جاري الرفع...";

        let url = "";
        let type = "";

        if(file) {
            type = file.type.split('/')[0]; // image or video
            const ref = storage.ref(`media/${Date.now()}_${file.name}`);
            await ref.put(file);
            url = await ref.getDownloadURL();
        }

        await db.collection("posts").add({
            text, url, mediaType: type,
            uid: auth.currentUser.uid,
            reactions: { like: [], haha: [], support: [], hate: [] },
            time: firebase.firestore.FieldValue.serverTimestamp()
        });

        document.getElementById('post-text').value = "";
        document.getElementById('post-media').value = "";
        btn.disabled = false;
        btn.innerText = "نشر الآن";
    }

    // التفاعلات
    async function handleReact(postId, rType, currentR) {
        const myUid = auth.currentUser.uid;
        let r = currentR || { like: [], haha: [], support: [], hate: [] };
        for (let k in r) { r[k] = r[k].filter(id => id !== myUid); }
        r[rType].push(myUid);
        await db.collection("posts").doc(postId).update({ reactions: r });
    }

    // حذف المنشور
    async function deletePost(id) {
        if(confirm("هل أنت متأكد من حذف المنشور؟")) await db.collection("posts").doc(id).delete();
    }

    // تعديل المنشور
    async function editPost(id, oldText) {
        const newText = prompt("تعديل المنشور:", oldText);
        if(newText) await db.collection("posts").doc(id).update({ text: newText });
    }

    // نسخ الرابط والمشاركة
    function copyLink(id) {
        const link = window.location.href + "?post=" + id;
        navigator.clipboard.writeText(link);
        alert("تم نسخ رابط المنشور!");
    }

    function sharePost(text) {
        if (navigator.share) {
            navigator.share({ title: 'البحير نت', text: text, url: window.location.href });
        } else {
            alert("المشاركة غير مدعومة في متصفحك، استخدم نسخ الرابط.");
        }
    }

    // تحميل المنشورات
    function loadPosts() {
        db.collection("posts").orderBy("time", "desc").onSnapshot(snap => {
            const feed = document.getElementById('feed');
            feed.innerHTML = "";
            snap.forEach(doc => {
                const p = doc.data();
                const id = doc.id;
                const myUid = auth.currentUser.uid;
                const time = p.time ? new Date(p.time.seconds*1000).toLocaleString('ar-EG') : 'الآن';
                const r = p.reactions || { like: [], haha: [], support: [], hate: [] };

                feed.innerHTML += `
                    <div class="post">
                        <div class="post-header">
                            <div class="user-meta">
                                <div class="avatar">👤</div>
                                <div><b>مستخدم البحير</b> <span class="post-time">${time}</span></div>
                            </div>
                            <div class="menu-dots" onclick="showActions('${id}', '${p.text}', ${p.uid === myUid})">⋮</div>
                        </div>
                        <div class="post-content">${p.text}</div>
                        <div class="media-box">
                            ${p.url ? (p.mediaType === 'video' ? `<video controls src="${p.url}"></video>` : `<img src="${p.url}">`) : ""}
                        </div>
                        
                        <div class="reactions-bar">
                            <button class="react-btn ${r.like.includes(myUid)?'active':''}" onclick='handleReact("${id}", "like", ${JSON.stringify(r)})'>👍 ${r.like.length}</button>
                            <button class="react-btn ${r.haha.includes(myUid)?'active':''}" onclick='handleReact("${id}", "haha", ${JSON.stringify(r)})'>😂 ${r.haha.length}</button>
                            <button class="react-btn ${r.support.includes(myUid)?'active':''}" onclick='handleReact("${id}", "support", ${JSON.stringify(r)})'>💪 ${r.support.length}</button>
                            <button class="react-btn ${r.hate.includes(myUid)?'active':''}" onclick='handleReact("${id}", "hate", ${JSON.stringify(r)})'>😡 ${r.hate.length}</button>
                        </div>

                        <div class="post-footer">
                            <button class="footer-btn" onclick="copyLink('${id}')">🔗 نسخ</button>
                            <button class="footer-btn" onclick="sharePost('${p.text}')">📤 مشاركة</button>
                            ${p.uid === myUid ? `
                                <button class="footer-btn" onclick="editPost('${id}', '${p.text}')">📝 تعديل</button>
                                <button class="footer-btn" style="color:red" onclick="deletePost('${id}')">🗑️ حذف</button>
                            ` : ''}
                        </div>
                    </div>`;
            });
        });
    }
</script>
</body>
</html>
