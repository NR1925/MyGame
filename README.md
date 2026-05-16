<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"/>
    <title>音樂事奉-雲端同步版 V6.0</title>
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/qrcode/build/qrcode.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/animate.css/4.1.1/animate.min.css"/>
    
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700;900&family=Noto+Sans+TC:wght@400;700;900&display=swap');
        body { font-family: 'Inter', 'Noto Sans TC', sans-serif; background: #f8fafc; color: #0f172a; margin: 0; }
        .projection-card { background: white; border-top: 12px solid #d97706; box-shadow: 0 30px 60px rgba(0,0,0,0.1); border-radius: 2.5rem; }
        .btn-primary { background: linear-gradient(135deg, #fbbf24, #d97706); color: white; transition: all 0.2s; }
        .btn-primary:active { transform: scale(0.95); }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const RELAY_SERVICE = "https://ntfy.sh";
        // 隨機產生一個房間 ID，避免與他人重複
        const ROOM_ID = "worship_" + Math.random().toString(36).substring(2, 10);

        function App() {
            const [mode, setMode] = React.useState(() => new URLSearchParams(window.location.search).get("mode") || "home");
            const [logs, setLogs] = React.useState([]);
            const [answer, setAnswer] = React.useState("");
            const [currentItem, setCurrentItem] = React.useState(null);
            const [isLocal, setIsLocal] = React.useState(window.location.protocol === 'file:');

            React.useEffect(() => {
                if (mode === "teacher") {
                    const targetRoom = new URLSearchParams(window.location.search).get("room") || ROOM_ID;
                    const fetchMsgs = async () => {
                        try {
                            const res = await fetch(`${RELAY_SERVICE}/${targetRoom}/json?poll=1`);
                            const text = await res.text();
                            const lines = text.split('\n').filter(l => l.trim());
                            const newMsgs = lines.map(line => {
                                const data = JSON.parse(line);
                                return { id: data.id, text: data.message, time: new Date(data.time * 1000).toLocaleTimeString() };
                            });
                            if (newMsgs.length > 0) {
                                setLogs(prev => {
                                    const ids = new Set(prev.map(p => p.id));
                                    const filtered = newMsgs.filter(m => !ids.has(m.id));
                                    return [...filtered, ...prev];
                                });
                            }
                        } catch (e) { console.log("同步中..."); }
                    };
                    const timer = setInterval(fetchMsgs, 3000);
                    return () => clearInterval(timer);
                }
            }, [mode]);

            React.useEffect(() => {
                const canvas = document.getElementById("qr-canvas");
                if (canvas) {
                    // 自動判斷網址：如果是本機檔案，QR會報錯，若是線上則正常
                    const currentUrl = window.location.href.split('?')[0];
                    const room = new URLSearchParams(window.location.search).get("room") || ROOM_ID;
                    const inviteUrl = `${currentUrl}?mode=student&room=${room}`;
                    QRCode.toCanvas(canvas, inviteUrl, { width: 240, margin: 2 });
                }
            }, [mode]);

            const handleSend = async () => {
                if (!answer.trim()) return;
                const room = new URLSearchParams(window.location.search).get("room");
                try {
                    await fetch(`${RELAY_SERVICE}/${room}`, { method: 'POST', body: answer.trim() });
                    setMode("success");
                } catch (e) { alert("發送失敗，請檢查網路！"); }
            };

            if (mode === "home") return (
                <div className="min-h-screen flex flex-col items-center justify-center p-6 text-center animate__animated animate__fadeIn">
                    <h1 className="text-4xl font-black mb-2 italic">音樂事奉分享系統</h1>
                    <p className="text-amber-600 font-bold mb-8 uppercase text-xs tracking-widest">Version 6.0 雲端直連版</p>
                    
                    <div className="bg-white p-8 rounded-[3rem] shadow-2xl mb-10 border border-slate-100">
                        {isLocal ? (
                            <div className="w-64 py-10 bg-red-50 rounded-2xl border-2 border-dashed border-red-200">
                                <p className="text-red-500 font-bold px-4">⚠️ 偵測到本機檔案</p>
                                <p className="text-slate-500 text-xs mt-2 px-6 leading-relaxed">手機無法連進您的電腦硬碟。請將代碼貼到 <span className="font-black text-blue-600">CodePen.io</span> 使用！</p>
                            </div>
                        ) : (
                            <canvas id="qr-canvas" className="mx-auto"></canvas>
                        )}
                        <p className="mt-4 font-black text-slate-800">手機掃描開始分享</p>
                    </div>

                    <div className="flex flex-col gap-4 w-full max-w-xs">
                        <button onClick={() => setMode("teacher")} className="btn-primary py-5 rounded-3xl font-black text-xl shadow-xl">進入大螢幕模式</button>
                    </div>
                </div>
            );

            if (mode === "student") return (
                <div className="min-h-screen p-6 flex flex-col bg-white">
                    <h2 className="text-2xl font-black mb-6 flex justify-between items-center">
                        📝 心靈分享 
                        <span className="text-[10px] bg-green-100 text-green-600 px-2 py-1 rounded-full uppercase">Online</span>
                    </h2>
                    <textarea 
                        className="flex-1 w-full p-6 rounded-[2rem] text-xl bg-slate-50 border-2 border-transparent focus:border-amber-500 outline-none shadow-inner" 
                        placeholder="請輸入您的感動..." 
                        value={answer}
                        onChange={e => setAnswer(e.target.value)}
                    />
                    <button onClick={handleSend} className="btn-primary w-full py-6 rounded-full font-black text-2xl mt-8 shadow-2xl">確認送出</button>
                </div>
            );

            if (mode === "teacher") return (
                <div className="min-h-screen bg-slate-900 text-white p-8 flex flex-col">
                    <div className="flex justify-between items-center mb-10">
                        <h1 className="text-3xl font-black">🎵 現場分享牆</h1>
                        <span className="bg-amber-600 px-4 py-1 rounded-full text-xs font-bold">收到 {logs.length} 則</span>
                    </div>
                    <div className="flex-1 flex flex-col justify-center items-center">
                        <div className="projection-card p-12 w-full max-w-4xl min-h-[400px] flex flex-col justify-center items-center text-center">
                            {currentItem ? (
                                <div className="animate__animated animate__fadeInUp">
                                    <h2 className="text-slate-900 text-5xl font-black leading-tight">{currentItem.text}</h2>
                                    <p className="text-slate-300 mt-6 text-sm">{currentItem.time}</p>
                                </div>
                            ) : (
                                <p className="text-slate-200 text-6xl font-black opacity-20 italic">WAITING...</p>
                            )}
                        </div>
                        <button 
                            onClick={() => setCurrentItem(logs[Math.floor(Math.random() * logs.length)])}
                            className="btn-primary mt-12 px-20 py-8 rounded-[3rem] text-4xl font-black shadow-2xl"
                        >
                            🎲 隨機抽選
                        </button>
                    </div>
                </div>
            );

            if (mode === "success") return (
                <div className="min-h-screen flex flex-col items-center justify-center p-6 text-center animate__animated animate__zoomIn">
                    <div className="w-20 h-20 bg-green-500 text-white rounded-full flex items-center justify-center mb-6 text-3xl">✓</div>
                    <h2 className="text-3xl font-black mb-10">已傳送到大螢幕！</h2>
                    <button onClick={() => setMode("student")} className="bg-slate-900 text-white px-12 py-4 rounded-full font-bold">再次填寫</button>
                </div>
            );

            return null;
        }

        const root = ReactDOM.createRoot(document.getElementById("root"));
        root.render(<App />);
    </script>
</body>
</html>

