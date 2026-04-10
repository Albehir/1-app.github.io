<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>شبكة البحير الاجتماعية - توثيق الهوية</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.15.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.15.0/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.15.0/firebase-database-compat.js"></script>

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/intl-tel-input/17.0.8/css/intlTelInput.css"/>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/intl-tel-input/17.0.8/js/intlTelInput.min.js"></script>

    <style>
        :root { --primary: #0055d4; --bg: #f0f2f5; --white: #fff; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); margin: 0; padding: 0; }
        
        #auth-screen { display: flex; justify-content: center; align-items: center; min-height: 100vh; padding: 20px; box-sizing: border-box; }
        .auth-card { background: var(--white); padding: 30px; border-radius: 15px; box-shadow: 0 10px 25px rgba(0,0,0,0.1); text-align: center; width: 100%; max-width: 400px; }
        .auth-card input { width: 100%; padding: 12px; margin-bottom: 15px; border-radius: 8px; border: 1px solid #ddd; box-sizing: border-box; font-size: 1rem; }
        
        .iti { width: 100%; margin-bottom: 15px; }
        #main-app { display: none; }
        header { background: var(--primary); color: white; padding: 15px 20px; display: flex; justify-content: space-between; align-items: center; position: sticky; top: 0; z-index: 1000; }
        
        .container { max-width: 500px; margin: 20px auto; padding: 10px; }
        .post-box { background: white; padding: 15px; border-radius: 12px; margin-bottom: 20px; box-shadow: 0 2px 5px rgba(0,0,0,0.05); }
        textarea { width: 100%; height: 70px; border: 1px solid #ddd; border-radius: 8px; padding: 10px; resize: none; width: 100%; box-sizing: border-box; }
        
        .btn-post { background: var(--primary); color: white; border: none; padding: 12px; border-radius: 8px; cursor: pointer; width: 100%; font-weight: bold; margin-top: 5px; }
        .post { background: white; padding: 15px; border-radius: 12px; margin-bottom: 15px; border: 1px solid #e0e0e0; }
        .username { font-weight: bold; color: var(--primary); }
        #recaptcha-container { margin: 10px 0; }
    </style>
</head>
<body>

    <div id="auth-screen">
        <div class="auth-card">
            <h2 style="color: var(--primary);">البحير نت</h2>
            
            <div id="step-1">
                <p>سجل عبر البريد أو الهاتف</p>
                <input type="email" id="email-input" placeholder="البريد الإلكتروني">
                <p style="margin: 5px 0;">أو</p>
                <input type="tel" id="phone-input" placeholder="اكتب رقمك">
                <div id="recaptcha-container"></div>
                <button onclick="handleAuth()" class="btn-post">إرسال رمز التحقق</button>
            </div>

            <div id="step-2" style="display: none;">
                <p>أدخل الكود المرسل لهاتفك</p>
                <input type="number" id="verification-code" placeholder="000000">
                <button onclick="verifySmsCode()" class="btn-post">تأكيد الدخول</button>
            </div>
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
        const firebaseConfig = {
apiKey: "AIzaSyAHbXMLT3F0OpnMZQHHPz-kAOlZBi1hhs4",
  authDomain: "albehirsocial.firebaseapp.com",
  databaseURL: "https://albehirsocial-default-rtdb.firebaseio.com",
  projectId: "albehirsocial",
  storageBucket: "albehirsocial.firebasestorage.app",
  messagingSenderId: "302307189713",
  appId: "1:302307189713:web:abc71f50c9f87113c0f6f0",
  measurementId: "G-0Y1D7BFD37"
};

// Initialize Firebase
const app = initializeApp(firebaseConfig);
const analytics = getAnalytics(app);
        };

        firebase.initializeApp(firebaseConfig);
        const auth = firebase.auth();
        const database = firebase.database();

        const phoneInputField = document.querySelector("#phone-input");
        const phoneInput = window.intlTelInput(phoneInputField, {
            utilsScript: "https://cdnjs.cloudflare.com/ajax/libs/intl-tel-input/17.0.8/js/utils.js",
        });

        window.recaptchaVerifier = new firebase.auth.RecaptchaVerifier('recaptcha-container', {
            'size': 'invisible'
        });

        let confirmationResult = null;
        let currentUserLabel = "";

        async function handleAuth() {
            const email = document.getElementById('email-input').value.trim();
            const phone = phoneInput.getNumber();

            try {
                if (email) {
                    const actionCodeSettings = {
                        url: window.location.href,
                        handleCodeInApp: true
                    };
                    await auth.sendSignInLinkToEmail(email, actionCodeSettings);
                    window.localStorage.setItem('emailForSignIn', email);
                    alert("تم إرسال رابط التحقق إلى بريدك الإلكتروني!");
                } 
                else if (phoneInput.isValidNumber()) {
                    const appVerifier = window.recaptchaVerifier;
                    confirmationResult = await auth.signInWithPhoneNumber(phone, appVerifier);
                    document.getElementById('step-1').style.display = 'none';
                    document.getElementById('step-2').style.display = 'block';
                    alert("تم إرسال كود الـ SMS");
                } 
                else {
                    alert("يرجى إدخال إيميل صحيح أو رقم هاتف كامل");
                }
            } catch (error) {
                alert("خطأ: " + error.message);
                console.error(error);
            }
        }

        async function verifySmsCode() {
            const code = document.getElementById('verification-code').value;
            try {
                const result = await confirmationResult.confirm(code);
                completeLogin(result.user);
            } catch (error) {
                alert("الكود غير صحيح!");
            }
        }

        if (auth.isSignInWithEmailLink(window.location.href)) {
            let email = window.localStorage.getItem('emailForSignIn');
            if (!email) email = window.prompt('يرجى تأكيد بريدك الإلكتروني:');
            auth.signInWithEmailLink(email, window.location.href)
                .then((result) => {
                    window.localStorage.removeItem('emailForSignIn');
                    completeLogin(result.user);
                })
                .catch((error) => alert("خطأ في التحقق: " + error.message));
        }

        function completeLogin(user) {
            currentUserLabel = user.email || user.phoneNumber;
            document.getElementById('auth-screen').style.display = 'none';
            document.getElementById('main-app').style.display = 'block';
            document.getElementById('user-info').innerHTML = `👤 ${currentUserLabel}`;
            
            database.ref('users/' + user.uid).update({
                identity: currentUserLabel,
                lastSeen: Date.now()
            });
            listenForPosts();
        }

        function sendPost() {
            const text = document.getElementById('post-text').value;
            if (text.trim() === "") return;
            database.ref('posts').push({
                username: currentUserLabel,
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
                                <span class="username">${post.username} <span style="color:#28a745; font-size:0.7rem;">● نشط</span></span>
                                <div class="post-content">${post.content}</div>
                                <div class="actions" style="margin-top:10px; border-top:1px solid #eee; padding-top:5px;">
                                    <button style="background:none; border:none; color:var(--primary); cursor:pointer;" onclick="addLike('${key}', ${post.likes || 0})">❤️ تفاعل (${post.likes || 0})</button>
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
