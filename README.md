<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>البحير نت - التواصل الاجتماعي</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-storage-compat.js"></script>

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">

    <style>
        :root {
            --primary-color: #1da1f2;
            --whatsapp-color: #25D366;
            --bg-color: #f5f8fa;
        }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: var(--bg-color); margin: 0; padding: 0; }
        .container { max-width: 600px; margin: auto; padding: 15px; }
        
        /* شاشة تسجيل الدخول */
        #auth-container { background: white; padding: 25px; border-radius: 20px; box-shadow: 0 10px 25px rgba(0,0,0,0.1); margin-top: 30px; text-align: center; }
        .logo { font-size: 28px; font-weight: bold; color: var(--primary-color); margin-bottom: 20px; }
        
        input { width: 100%; padding: 14px; margin: 10px 0; border: 1.5px solid #eee; border-radius: 12px; box-sizing: border-box; outline: none; transition: 0.3s; }
        input:focus { border-color: var(--primary-color); }
        
        .btn { width: 100%; padding: 14px; border: none; border-radius: 12px; cursor: pointer; font-size: 16px; font-weight: bold; transition: 0.3s; margin-bottom: 10px; }
        .btn-primary { background: var(--primary-color); color: white; }
        .btn-whatsapp { background: var(--whatsapp-color); color: white; display: flex; align-items: center; justify-content: center; gap: 10px; }
        .btn-whatsapp:hover { opacity: 0.9; }

        /* شاشة التطبيق */
        #main-app { display: none; }
        .nav { background: white; padding: 15px; position: sticky; top: 0; z-index: 100; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #eee; }
        
        #post-form { background: white; padding: 15px; border-radius: 15px; margin-bottom: 20px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
        textarea { width: 100%; height: 100px; border: none; padding: 10px; resize: none; font-size: 16px; box-sizing: border-box; outline: none; }
        
        .post-card { background: white; padding: 15px; border-radius: 15px; margin-bottom: 15px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); }
        .post-header { font-weight: bold; color: #333; margin-bottom: 8px; font-size: 14px; }
        .post-img { width: 100%; border-radius: 12px; margin-top: 10px; max-height: 400px; object-fit: cover; }
        
        #recaptcha-container { margin: 10px 0; }
    </style>
</head>
<body>

    <div class="container">
        <div id="auth-container">
            <div class="logo">البحير نت</div>
            <p style="color: #666;">سجل دخولك للتواصل مع الأصدقاء</p>
            
            <div id="login-fields">
                <input type="tel" id="phone" placeholder="+249XXXXXXXXX">
                <div id="recaptcha-container"></div>
                <button id="send-code-btn" class="btn btn-whatsapp" onclick="sendOTP()">
                    <i class="fab fa-whatsapp"></i> تسجيل برقم الواتساب
                </button>
                
                <div id="otp-section" style="display:none; border-top: 1px solid #eee; padding-top: 10px;">
                    <p style="font-size: 13px; color: #888;">أدخل الكود المكون من 6 أرقام</p>
                    <input type="text" id="otp" placeholder="000000">
                    <button class="btn btn-primary" onclick="verifyOTP()">تأكيد الكود</button>
                </div>

                <div style="margin: 20px 0; color: #ccc;">أو</div>

                <input type="email" id="email" placeholder="البريد الإلكتروني">
                <button class="btn btn-primary" style="background: #555;" onclick="loginWithEmail()">دخول عبر رابط البريد</button>
            </div>
        </div>

        <div id="main-app">
            <div class="nav">
                <div style="font-weight: bold; font-size: 20px; color: var(--primary-color);">البحير نت</div>
                <button onclick="logout()" style="background:none; border:1px solid #ff4d4d; color:#ff4d4d; padding:5px 12px; border-radius:8px; cursor:pointer;">خروج</button>
            </div>

            <div id="post-form">
                <textarea id="post-text" placeholder="اكتب شيئاً للدنيا..."></textarea>
                <div style="display: flex; justify-content: space-between; align-items: center; border-top: 1px solid #eee; padding-top: 10px;">
                    <input type="file" id="post-image" accept="image/*" style="width: auto; border: none; padding: 0;">
                    <button class="btn btn-primary" style="width: 100px; margin: 0;" onclick="uploadPost()">نشر</button>
                </div>
            </div>

            <div id="posts-container">
                </div>
        </div>
    </div>

    <script>
        // تكوين Firebase (ضع بياناتك هنا)
        const firebaseConfig = {
            apiKey: "ضع_مفتاحك_هنا",
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

        // إعداد المحقق المرئي
        window.recaptchaVerifier = new firebase.auth.RecaptchaVerifier('recaptcha-container', { 'size': 'invisible' });

        // وظائف تسجيل الدخول
        function sendOTP() {
            const phoneNumber = document.getElementById('phone').value;
            if(!phoneNumber.startsWith('+')) return alert("يرجى كتابة الرقم بالصيغة الدولية مثل +249");
            
            auth.signInWithPhoneNumber(phoneNumber, window.recaptchaVerifier)
                .then(confirmationResult => {
                    window.confirmationResult = confirmationResult;
                    document.getElementById('otp-section').style.display = 'block';
                    document.getElementById('send-code-btn').style.display = 'none';
                }).catch(error => alert("خطأ: " + error.message));
        }

        function verifyOTP() {
            const code = document.getElementById('otp').value;
            confirmationResult.confirm(code).then(() => {
                // سيتم التحديث تلقائياً عبر onAuthStateChanged
            }).catch(() => alert("الكود غير صحيح"));
        }

        function loginWithEmail() {
            const email = document.getElementById('email').value;
            auth.sendSignInLinkToEmail(email, { url: 'https://albehir.github.io', handleCodeInApp: true })
                .then(() => {
                    localStorage.setItem('emailForSignIn', email);
                    alert('افحص بريدك الآن، أرسلنا لك رابط الدخول!');
                }).catch(error => alert(error.message));
        }

        // مراقبة الجلسة
        auth.onAuthStateChanged(user => {
            if (user) {
                document.getElementById('auth-container').style.display = 'none';
                document.getElementById('main-app').style.display = 'block';
                loadPosts();
            } else {
                document.getElementById('auth-container').style.display = 'block';
                document.getElementById('main-app').style.display = 'none';
            }
        });

        // نظام المنشورات
        async function uploadPost() {
            const text = document.getElementById('post-text').value;
            const file = document.getElementById('post-image').files[0];
            if (!text && !file) return;

            let imageUrl = "";
            if (file) {
                const ref = storage.ref(`posts/${Date.now()}_${file.name}`);
                await ref.put(file);
                imageUrl = await ref.getDownloadURL();
            }

            await db.collection("posts").add({
                text: text,
                image: imageUrl,
                user: auth.currentUser.phoneNumber || auth.currentUser.email,
                time: firebase.firestore.FieldValue.serverTimestamp()
            });
            document.getElementById('post-text').value = "";
            document.getElementById('post-image').value = "";
        }

        function loadPosts() {
            db.collection("posts").orderBy("time", "desc").onSnapshot(snap => {
                const container = document.getElementById('posts-container');
                container.innerHTML = "";
                snap.forEach(doc => {
                    const p = doc.data();
                    container.innerHTML += `
                        <div class="post-card">
                            <div class="post-header"><i class="fa-regular fa-user"></i> ${p.user}</div>
                            <div>${p.text}</div>
                            ${p.image ? `<img src="${p.image}" class="post-img">` : ""}
                        </div>`;
                });
            });
        }

        function logout() { auth.signOut(); }
    </script>
</body>
</html>
