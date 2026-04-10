
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>شبكة البحير الاجتماعية</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.15.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.15.0/firebase-database-compat.js"></script>

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/intl-tel-input/17.0.8/css/intlTelInput.css"/>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/intl-tel-input/17.0.8/js/intlTelInput.min.js"></script>

    <style>
        :root { --primary: #0055d4; --bg: #f0f2f5; --white: #fff; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); margin: 0; padding: 0; }
        
        /* شاشة التسجيل المعدلة */
        #auth-screen { display: flex; justify-content: center; align-items: center; min-height: 100vh; padding: 20px; box-sizing: border-box; }
        .auth-card { background: var(--white); padding: 30px; border-radius: 15px; box-shadow: 0 10px 25px rgba(0,0,0,0.1); text-align: center; width: 100%; max-width: 400px; }
        .auth-card input { width: 100%; padding: 12px; margin-bottom: 15px; border-radius: 8px; border: 1px solid #ddd; box-sizing: border-box; font-size: 1rem; }
        
        /* تنسيق خاص لحقل الهاتف ليناسب المكتبة */
        .iti { width: 100%; margin-bottom: 15px; }

        #main-app { display: none; }
        header { background: var(--primary); color: white; padding: 15px 20px; display: flex; justify-content: space-between; align-items: center; position: sticky; top: 0; z-index: 1000; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        
        .container { max-width: 500px; margin: 20px auto; padding: 10px; }
        .post-box { background: white; padding: 15px; border-radius: 12px; margin-bottom: 20px; box-shadow: 0 2px 5px rgba(0,0,0,0.05); }
        textarea { width: 100%; height: 70px; border: 1px solid #ddd; border-radius: 8px; padding: 10px; resize: none; box-sizing: border-box; font-family: inherit; font-size: 1rem; }
        
        .btn-post { background: var(--primary); color: white; border: none; padding: 12px; border-radius: 8px; cursor: pointer; margin-top: 10px; width: 100%; font-weight: bold; font-size: 1rem; transition: 0.3s; }
        .btn-post:hover { background: #0044aa; }
        
        .post { background: white; padding: 15px; border-radius: 12px; margin-bottom: 15px; border: 1px solid #e0e0e0; animation: fadeIn 0.4s ease; }
        .username { font-weight: bold; color: var(--primary); display: block; margin-bottom: 5px; }
        .status-online { color: #28a745; font-size: 0.7rem; margin-right: 5px; }
        .post-content { font-size: 1.1rem; color: #333; word-wrap: break-word; line-height: 1.4; }
        
        .actions { border-top: 1px solid #eee; margin-top: 10px; padding-top: 8px; }
        .like-btn { cursor: pointer; color: #666; border: none; background: none; font-size: 0.9rem; font-weight: bold; }
        
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
    </style>
</head>
<body>

    <div id="auth-screen">
        <div class="auth-card">
            <h2 style="color: var(--primary);">البحير نت</h2>
            <p>إنشاء حساب جديد</p>
            <input type="text" id="username-input" placeholder="الاسم المستعار (سيظهر للجميع)">
            <input type="email" id="email-input" placeholder="البريد الإلكتروني">
            <input type="tel" id="phone-input">
            <button onclick="startApp()" class="btn-post">دخول للمنصة</button>
        </div>
    </div>

    <div id="main-app">
        <header>
            <div><strong>شبكة البحير الاجتماعية</strong></div>
            <div id="user-info"></div>
        </header>

        <div class="container">
            <div class="post-box">
                <textarea id="post-text" placeholder="ماذا يدور في ذهنك؟"></textarea>
                <button onclick="sendPost()" class="btn-post">نشر المنشور 🚀</button>
            </div>
            <div id="posts-container"></div>
        </div>
    </div>

    <script>
        // إعدادات Firebase
        const firebaseConfig = {
            apiKey: "AIzaSyAHbXMLT3F0O1A2B3C4D5E6F7G8H9I0", 
            authDomain: "albehirsocial.firebaseapp.com",
            databaseURL: "https://albehirsocial-default-rtdb.firebaseio.com",
            projectId: "albehirsocial",
            storageBucket: "albehirsocial.appspot.com",
            messagingSenderId: "302307189713",
            appId: "1:302307189713:web:5d2f6f8f7a6b5c4d3e2f1"
        };

        firebase.initializeApp(firebaseConfig);
        const database = firebase.database();
        let myUsername = "";

        // تفعيل أرقام الهاتف لجميع الدول
        const phoneInputField = document.querySelector("#phone-input");
        const phoneInput = window.intlTelInput(phoneInputField, {
            preferredCountries: ["sd", "sa", "eg", "ae"],
            utilsScript: "https://cdnjs.cloudflare.com/ajax/libs/intl-tel-input/17.0.8/js/utils.js",
        });

        function startApp() {
            const name = document.getElementById('username-input').value.trim();
            const email = document.getElementById('email-input').value.trim();
            const phone = phoneInput.getNumber();

            if (name === "" || email === "" || !phoneInput.isValidNumber()) {
                return alert("من فضلك أدخل بيانات صحيحة (تأكد من رقم الهاتف)");
            }

            myUsername = name;

            // حفظ بيانات المستخدم في قاعدة البيانات عند التسجيل أول مرة
            database.ref('users/' + myUsername).set({
                username: myUsername,
                email: email,
                phone: phone,
                lastSeen: Date.now()
            });

            document.getElementById('auth-screen').style.display = 'none';
            document.getElementById('main-app').style.display = 'block';
            document.getElementById('user-info').innerHTML = `👤 ${myUsername}`;
            listenForPosts();
        }

        function sendPost() {
            const text = document.getElementById('post-text').value;
            if (text.trim() === "") return;
            database.ref('posts').push({
                username: myUsername,
                content: text,
                timestamp: Date.now(),
                likes: 0
            });
            document.getElementById('post-text').value = "";
        }

        function listenForPosts() {
            database.ref('posts').on('value', (snapshot) => {
                const container = document.getElementById('posts-container');
                container.innerHTML = "";
                const data = snapshot.val();
                if (data) {
                    Object.keys(data).reverse().forEach(key => {
                        const post = data[key];
                        container.innerHTML += `
                            <div class="post">
                                <span class="username">${post.username} <span class="status-online">● نشط الآن</span></span>
                                <div class="post-content">${post.content}</div>
                                <div class="actions">
                                    <button class="like-btn" onclick="addLike('${key}', ${post.likes || 0})">❤️ تفاعل (${post.likes || 0})</button>
                                </div>
                            </div>`;
                    });
                }
            });
        }

        function addLike(postId, currentLikes) {
            database.ref('posts/' + postId).update({ likes: currentLikes + 1 });
        }
    </script>
</body>
</html>
