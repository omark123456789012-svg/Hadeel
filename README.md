<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <title>لعبة خمن الأكلة - منصة كيك</title>
    <style>
        body { font-family: 'Segoe UI', sans-serif; background: #0b0e11; color: white; text-align: center; padding: 20px; }
        .setup-box { background: #191b1f; padding: 15px; border-radius: 10px; margin-bottom: 25px; display: inline-block; border: 1px solid #53fc18; }
        input { padding: 10px; font-size: 16px; border-radius: 5px; border: 1px solid #333; background: #0b0e11; color: white; text-align: center; margin-left: 10px; }
        button { padding: 10px 20px; font-size: 16px; border: none; border-radius: 5px; background: #53fc18; color: black; cursor: pointer; font-weight: bold; }
        button:hover { background: #42cb12; }
        #game-box { background: #191b1f; padding: 20px; border-radius: 15px; display: inline-block; width: 450px; box-shadow: 0px 0px 20px rgba(83, 252, 24, 0.2); }
        img { width: 100%; max-height: 280px; border-radius: 10px; object-fit: cover; }
        #winner { color: #53fc18; font-size: 26px; font-weight: bold; margin-top: 15px; }
        #connection-info { font-weight: bold; margin-top: 10px; color: #aaa; }
    </style>
</head>
<body>

    <h1 style="color: #53fc18;">🟢 لعبة خمن الأكلة لبثوث كيك (Kick)</h1>
    
    <!-- صندوق ربط الستريمر -->
    <div class="setup-box">
        <label>أدخل رقم الـ Chatroom ID الخاص بحسابك: </label>
        <input type="text" id="chatroom-id" placeholder="مثال: 123456">
        <button onclick="connectToKick()">ربط شات كيك</button>
        <div id="connection-info">🔴 غير متصل بالبث</div>
        <small style="color: #888; display:block; margin-top:5px;">(يمكن للستريمر معرفة الـ ID الخاص به عبر فتح كود مصدر صفحته على كيك والبحث عن chatroom_id)</small>
    </div>
    <br>
    
    <!-- صندوق اللعبة -->
    <div id="game-box">
        <img id="food-img" src="https://unsplash.com" alt="طعام">
        <h2 id="status">يرجى ربط الشات لبدء التخمين...</h2>
        <div id="winner"></div>
    </div>

    <script>
        const rounds = [
            { image: "https://unsplash.com", answer: "بيتزا" },
            { image: "https://unsplash.com", answer: "برجر" },
            { image: "https://unsplash.com", answer: "سلطة" },
            { image: "https://unsplash.com", answer: "شاورما" },
            { image: "https://unsplash.com", answer: "سوشي" }
        ];

        let currentRound = 0;
        let isRoundActive = false;
        let pusherSocket;

        const imgElement = document.getElementById('food-img');
        const winnerElement = document.getElementById('winner');
        const statusElement = document.getElementById('status');
        const infoDiv = document.getElementById('connection-info');

        function connectToKick() {
            const chatroomId = document.getElementById('chatroom-id').value.trim();
            if (!chatroomId) {
                alert("يرجى إدخال الرقم التعريفي للشات (Chatroom ID)!");
                return;
            }

            infoDiv.innerText = "⏳ جاري الاتصال بخوادم كيك...";
            infoDiv.style.color = "#ffaa00";

            // الاتصال المباشر بخادم Pusher الرسمي الذي تستخدمه منصة كيك للشات
            const kickPusherKey = "32cbd69e4b950bf97679"; // المفتاح الرسمي العام لكيك
            pusherSocket = new WebSocket(`wss://://pusher.com{kickPusherKey}?protocol=7&client=js&version=8.4.0&flash=false`);

            pusherSocket.onopen = () => {
                // الاشتراك في الغرفة الخاصة بالستريمر لقراءة رسائلها
                const subscribeMessage = JSON.stringify({
                    event: "pusher:subscribe",
                    data: { channel: `chatrooms.${chatroomId}.v2` }
                });
                pusherSocket.send(subscribeMessage);
                
                infoDiv.innerText = `🟢 متصل بنجاح بشات كيك! الغرفة: ${chatroomId}`;
                infoDiv.style.color = "#53fc18";
                statusElement.innerText = "خمن ما في الصورة؟ واكتب الإجابة في شات كيك";
                isRoundActive = true;
            };

            pusherSocket.onmessage = (event) => {
                if (!isRoundActive) return;

                const rawData = JSON.parse(event.data);
                
                // التأكد من أن الحدث هو إرسال رسالة شات جديدة
                if (rawData.event === "App\\Events\\ChatMessageEvent") {
                    const chatData = JSON.parse(rawData.data);
                    const userComment = chatData.content.trim(); // نص الرسالة
                    const username = chatData.sender.username;   // اسم المتابع

                    // التحقق من الإجابة الصحيحة
                    if (userComment === rounds[currentRound].answer) {
                        isRoundActive = false;
                        winnerElement.innerText = `🎉 الفائز هو المتابع: ${username}`;
                        statusElement.innerText = `الإجابة الصحيحة كانت: ${rounds[currentRound].answer}`;
                        
                        setTimeout(nextRound, 4000); // الانتقال للصورة التالية بعد 4 ثوانٍ
                    }
                }
            };

            pusherSocket.onerror = () => {
                infoDiv.innerText = "❌ فشل الاتصال بخوادم كيك. تأكد من رقم الغرفة.";
                infoDiv.style.color = "#ff4757";
            };
        }

        function nextRound() {
            currentRound++;
            if (currentRound < rounds.length) {
                imgElement.src = rounds[currentRound].image;
                winnerElement.innerText = "";
                statusElement.innerText = "خمن ما في الصورة؟ واكتب الإجابة في الشات";
                isRoundActive = true;
            } else {
                statusElement.innerText = "انتهت اللعبة! شكراً لجميع المشاركين";
                imgElement.style.display = "none";
                winnerElement.innerText = "";
            }
        }
    </script>
</body>
</html>
