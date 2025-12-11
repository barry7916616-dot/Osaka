# Osaka
import React, { useState, useEffect, useCallback } from 'react';

// Firebase Imports
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, doc, setDoc, query, onSnapshot, getDoc } from 'firebase/firestore';

// --- 設定 Firebase/Firestore (必須) ---
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-travel-app-id';
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

let app, db, auth;
let currentUserId = null;

// Initialize Firebase
if (Object.keys(firebaseConfig).length > 0) {
    app = initializeApp(firebaseConfig);
    db = getFirestore(app);
    auth = getAuth(app);
}

// 輔助函式: 根據 Firestore 安全規則建立用戶專屬的路徑
const getPrivateDocRef = (docName) => {
    if (!db || !currentUserId) return null;
    return doc(db, 'artifacts', appId, 'users', currentUserId, 'travel_data', docName);
};

// ------------------------------------------

// --- LLM 助手功能: 模擬天氣和導遊分析 ---

// 模擬天氣數據
const simulatedWeather = {
    '2025-01-30': { temp: '5°C/2°C', icon: '❄️', condition: '晴朗，微雪' },
    '2025-01-31': { temp: '8°C/3°C', icon: '☀️', condition: '晴朗，乾燥' },
    '2025-02-01': { temp: '7°C/4°C', icon: '☁️', condition: '多雲，偶有陣雨' },
    '2025-02-02': { temp: '6°C/1°C', icon: '🌧️', condition: '下雨' },
    '2025-02-03': { temp: '9°C/5°C', icon: '🌤️', condition: '多雲轉晴' },
    '2025-02-04': { temp: '10°C/6°C', icon: '☀️', condition: '晴朗' },
};

// 導遊職責分析 (使用模擬 LLM 功能來強調文字)
const highlightGuideText = (text) => {
    if (!text) return null;

    // 定義要高亮的關鍵字及其 Tailwind 樣式
    const highlightRules = [
        { regex: /必吃美食/g, color: 'bg-red-100 text-red-700 font-bold' },
        { regex: /必點菜單/g, color: 'bg-amber-100 text-amber-700 font-bold' },
        { regex: /必買伴手禮/g, color: 'bg-green-100 text-green-700 font-bold' },
        { regex: /重要預約代號:\s*(\w+)/g, color: 'bg-indigo-100 text-indigo-700 font-bold' },
    ];

    let segments = [{ text: text, key: 0 }];
    let segmentKey = 1;

    highlightRules.forEach(rule => {
        const newSegments = [];
        segments.forEach(seg => {
            if (seg.highlight) {
                newSegments.push(seg);
                return;
            }

            const parts = seg.text.split(rule.regex);
            if (parts.length > 1) {
                let matchIndex = 0;
                let highlightText = seg.text.match(rule.regex);

                parts.forEach((part, index) => {
                    if (part) {
                        newSegments.push({ text: part, key: segmentKey++ });
                    }
                    if (index < parts.length - 1 && highlightText[matchIndex]) {
                        // 處理預約代號的特殊情況，只高亮代碼
                        const isBookingCode = highlightText[matchIndex].includes('重要預約代號:');
                        const matchedText = highlightText[matchIndex];
                        
                        if (isBookingCode) {
                            const codeMatch = matchedText.match(/(\w+)$/);
                            if (codeMatch) {
                                // 高亮 "重要預約代號:" 部分
                                newSegments.push({ text: matchedText.substring(0, matchedText.length - codeMatch[1].length), key: segmentKey++, highlight: 'bg-gray-200 text-gray-700 font-semibold' });
                                // 高亮代碼本身
                                newSegments.push({ text: codeMatch[1], key: segmentKey++, highlight: rule.color });
                            } else {
                                newSegments.push({ text: matchedText, key: segmentKey++, highlight: rule.color });
                            }
                        } else {
                            newSegments.push({ text: matchedText, key: segmentKey++, highlight: rule.color });
                        }
                        matchIndex++;
                    }
                });
            } else {
                newSegments.push(seg);
            }
        });
        segments = newSegments;
    });

    return (
        <p className="mt-2 text-sm text-gray-600 leading-relaxed">
            {segments.map((seg) => (
                seg.highlight ? (
                    <span key={seg.key} className={`inline-block px-1 py-0.5 rounded-md text-xs mr-1 mb-1 ${seg.highlight}`}>
                        {seg.text}
                    </span>
                ) : (
                    <span key={seg.key}>{seg.text}</span>
                )
            ))}
        </p>
    );
};

// --- 行程與工具的模擬資料 (已加入圖片 URL) ---

const itineraryData = [
    {
        date: '2025-01-30',
        day: '週四 (啟程)',
        items: [
            { type: 'flight', name: '桃園機場 (TPE) -> 關西機場 (KIX)', time: '07:30 - 11:00', detail: 'CI172 華航。請確認重要預約代號: C2A3X9' },
            { type: 'transport', name: 'KIX -> 大阪市區 (南海電鐵)', time: '12:00', detail: '搭乘南海電鐵 Rapi:t，車程約 38 分鐘。請預先購買車票。' },
            { type: 'accommodation', name: '大阪難波飯店 Check-in', time: '14:30', detail: '預計在難波車站附近Check-in並放下行李。' },
            { 
                type: 'attraction', 
                name: '道頓堀 (Dotonbori)', 
                time: '16:00', 
                detail: '欣賞固力果跑跑人招牌，感受大阪的熱情。晚餐必吃美食：章魚燒、大阪燒。',
                imageUrl: 'https://placehold.co/400x200/FAD3CF/222?text=[道頓堀+夜景]' // 模擬圖片 URL
            },
            { type: 'restaurant', name: '一蘭拉麵 (道頓堀屋台館)', time: '19:00', detail: '必點菜單：天然豚骨拉麵，加點半熟鹽味蛋。' },
        ],
    },
    {
        date: '2025-01-31',
        day: '週五 (大阪 -> 京都)',
        items: [
            { 
                type: 'attraction', 
                name: '大阪城公園', 
                time: '09:00', 
                detail: '欣賞壯觀的天守閣。必買伴手禮：大阪城限定紀念品。',
                imageUrl: 'https://placehold.co/400x200/B2D7D0/222?text=[大阪城]'
            },
            { type: 'transport', name: '大阪 -> 京都 (自駕)', time: '11:00', detail: '取車並開往京都，車程約 1 小時。' },
            { type: 'accommodation', name: '京都飯店 Check-in', time: '14:00', detail: '在京都車站附近 check-in。' },
            { 
                type: 'attraction', 
                name: '清水寺', 
                time: '16:00', 
                detail: '欣賞清水舞台的壯麗景色。必買伴手禮：七味家本舖的七味粉。',
                imageUrl: 'https://placehold.co/400x200/964B00/fff?text=[清水寺+舞台]'
            },
            { type: 'restaurant', name: '京の燒肉處 弘 (千本三條本店)', time: '19:30', detail: '必吃美食：國產牛壽喜燒風味燒肉。' },
        ],
    },
    {
        date: '2025-02-01',
        day: '週六 (京都：樂迷清單)',
        items: [
            { 
                type: 'attraction', 
                name: '伏見稻荷大社', 
                time: '09:00', 
                detail: '走訪綿延的千本鳥居。',
                imageUrl: 'https://placehold.co/400x200/E36414/fff?text=[伏見稻荷+鳥居]' 
            },
            { 
                type: 'attraction', 
                name: '嵐山竹林之道', 
                time: '14:00', 
                detail: '享受竹林中的寧靜，避開人潮建議早點到達。必吃美食：中村屋總本店的可樂餅。',
                imageUrl: 'https://placehold.co/400x200/81B622/fff?text=[嵐山+竹林]' 
            },
            { type: 'transport', name: '嵐山 -> 祇園 (自駕)', time: '17:00', detail: '將車停在祇園附近的停車場，準備晚餐和夜間散步。' },
            { type: 'restaurant', name: '祇園麵処 むらじ', time: '19:00', detail: '必點菜單：雞湯拉麵或黑醬油拉麵。' },
        ],
    },
    {
        date: '2025-02-02',
        day: '週日 (京都：文化體驗)',
        items: [
            { 
                type: 'attraction', 
                name: '金閣寺', 
                time: '09:30', 
                detail: '欣賞金光閃閃的舍利殿倒映在鏡湖池中。',
                imageUrl: 'https://placehold.co/400x200/FFD700/000?text=[金閣寺]'
            },
            { type: 'attraction', name: '西陣織會館', time: '11:30', detail: '觀賞和服秀，了解傳統織布工藝。' },
            { type: 'restaurant', name: '瓢亭 (米其林三星)', time: '13:00', detail: '重要預約代號: ZYX987。必點菜單：招牌瓢亭玉子（半熟蛋）。' },
            { type: 'attraction', name: '哲學之道', time: '16:30', detail: '沿著運河散步，感受悠閒氣氛。' },
        ],
    },
    {
        date: '2025-02-03',
        day: '週一 (京都 -> 關西機場周邊)',
        items: [
            { type: 'transport', name: '京都 -> 臨空城 (Rinku Town) 自駕', time: '10:00', detail: '開車前往機場附近的臨空城，準備購物。車程約 1.5 小時。' },
            { 
                type: 'attraction', 
                name: '臨空城 Premium Outlets', 
                time: '12:00', 
                detail: '最後一站購物天堂。必買伴手禮：電器、藥妝。',
                imageUrl: 'https://placehold.co/400x200/C8B8A6/222?text=[臨空城+Outlets]'
            },
            { type: 'restaurant', name: 'Outlets 美食廣場', time: '15:00', detail: '快速用餐。' },
            { type: 'accommodation', name: '關西機場附近飯店 Check-in', time: '17:30', detail: '為隔天搭機做準備。' },
        ],
    },
    {
        date: '2025-02-04',
        day: '週二 (返台)',
        items: [
            { type: 'transport', name: '飯店 -> 關西機場', time: '08:00', detail: '退房並開車至 KIX 還車。' },
            { type: 'flight', name: '關西機場 (KIX) -> 桃園機場 (TPE)', time: '12:00 - 14:00', detail: 'CI173 華航。請確認重要預約代號: D4B4Y0' },
        ],
    },
];

const initialToolsData = {
    flight: {
        outbound: 'CI172 (TPE 07:30 - KIX 11:00)',
        return: 'CI173 (KIX 12:00 - TPE 14:00)',
        pnr: 'C2A3X9 / D4B4Y0',
        note: '華航經濟艙，去程/回程皆已線上報到。'
    },
    accommodation: [
        { name: '大阪難波飯店', checkin: '1/30', checkout: '1/31', code: 'OSK123' },
        { name: '京都車站周邊公寓', checkin: '1/31', checkout: '2/03', code: 'KYT456' },
        { name: 'KIX 臨空城飯店', checkin: '2/03', checkout: '2/04', code: 'RINKO789' },
    ],
    emergency: [
        { name: '台灣駐大阪辦事處', tel: '+81-6-6110-6688' },
        { name: '日本警察局 (通用)', tel: '110' },
        { name: '急救/火災 (通用)', tel: '119' },
    ]
};

// 初始預算資料 (會從 Firestore 加載)
const initialBudget = {
    totalBudget: 50000,
    entries: [
        { id: 1, type: '支出', category: '機票', amount: 15000, note: 'TPE-KIX 來回' },
        { id: 2, type: '支出', category: '住宿', amount: 12000, note: '5晚總計' },
    ],
};

// --- UI 組件 ---

// 導航按鈕 (自駕友好)
const NavigationButton = ({ placeName }) => {
    const mapsUrl = `https://www.google.com/maps/search/?api=1&query=${encodeURIComponent(placeName)}`;
    return (
        <a 
            href={mapsUrl} 
            target="_blank" 
            rel="noopener noreferrer" 
            className="flex items-center justify-center space-x-2 px-3 py-1.5 bg-indigo-600 text-white rounded-xl shadow-md hover:bg-indigo-700 transition duration-150 text-sm font-medium"
        >
            <svg xmlns="http://www.w3.org/2000/svg" className="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.828 0l-4.243-4.243a8 8 0 1111.314 0z" />
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M15 11a3 3 0 11-6 0 3 3 0 016 0z" />
            </svg>
            <span>導航 (Google Map)</span>
        </a>
    );
};

// 行程卡片
const ItineraryCard = ({ item }) => {
    let icon, bgColor;
    
    switch (item.type) {
        case 'attraction':
            icon = '⛩️';
            bgColor = 'bg-amber-50'; // 更柔和的米色/金色
            break;
        case 'restaurant':
            icon = '🍜';
            bgColor = 'bg-rose-50'; // 柔和的紅粉色
            break;
        case 'transport':
            icon = '🚄';
            bgColor = 'bg-sky-50'; // 柔和的天藍色
            break;
        case 'accommodation':
            icon = '🏨';
            bgColor = 'bg-fuchsia-50'; // 柔和的紫紅色
            break;
        case 'flight':
            icon = '✈️';
            bgColor = 'bg-emerald-50'; // 柔和的綠色
            break;
        default:
            icon = '📌';
            bgColor = 'bg-gray-50';
    }

    // 判斷是否顯示導航按鈕
    const showNavigation = ['attraction', 'restaurant', 'accommodation'].includes(item.type) && item.name.includes('機場') === false;

    return (
        <div className={`p-4 rounded-xl shadow-lg border border-gray-100 mb-6 transition-all duration-300 ${bgColor} hover:shadow-xl`}>
            
            {/* 景點圖片預覽 (如果存在) */}
            {item.imageUrl && (
                <div className="mb-3 overflow-hidden rounded-lg shadow-inner">
                    <img 
                        src={item.imageUrl} 
                        alt={item.name} 
                        className="w-full h-32 object-cover object-center transition duration-500 ease-in-out hover:scale-[1.03]"
                        // 圖片載入錯誤時的優雅處理
                        onError={(e) => {
                            e.target.onerror = null; 
                            e.target.src = `https://placehold.co/400x128/ccc/666?text=[${item.name} 圖片載入失敗]`;
                        }}
                    />
                </div>
            )}

            <div className="flex justify-between items-start">
                <div className="flex items-start space-x-3">
                    <div className="text-2xl pt-1">{icon}</div>
                    <div>
                        <p className="text-xs text-gray-500 font-medium">{item.time}</p>
                        <h3 className="text-lg font-bold text-gray-800 leading-tight mt-0.5">{item.name}</h3>
                    </div>
                </div>
                
            </div>
            {highlightGuideText(item.detail)}
            
            {showNavigation && (
                <div className="mt-4 flex justify-end">
                    <NavigationButton placeName={item.name} />
                </div>
            )}
        </div>
    );
};

// 每日天氣顯示
const WeatherDisplay = ({ date }) => {
    const [weather, setWeather] = useState(null);
    const [isLoading, setIsLoading] = useState(true);

    const fetchWeather = useCallback(async (d) => {
        setIsLoading(true);
        // 模擬 API 延遲
        await new Promise(resolve => setTimeout(resolve, 500)); 
        
        setWeather(simulatedWeather[d] || { temp: 'N/A', icon: '❓', condition: '無法取得天氣' });
        setIsLoading(false);
    }, []);

    useEffect(() => {
        fetchWeather(date);
    }, [date, fetchWeather]);

    if (isLoading) {
        return (
            <div className="flex items-center justify-center p-3 bg-white/70 backdrop-blur-sm rounded-xl mb-4 animate-pulse shadow-inner border border-gray-100">
                <p className="text-sm text-gray-500">正在準備即時資訊...</p>
            </div>
        );
    }

    return (
        <div className="p-3 bg-white/80 backdrop-blur-sm rounded-xl shadow-lg border border-gray-100 mb-4 flex items-center justify-between transition-all duration-300">
            <div className="flex items-center space-x-3">
                <span className="text-3xl">{weather.icon}</span>
                <div>
                    <p className="text-sm text-gray-500">氣溫範圍</p>
                    <p className="text-xl font-extrabold text-gray-800">{weather.temp}</p>
                </div>
            </div>
            <p className="text-sm text-gray-600">{weather.condition}</p>
        </div>
    );
};

// 預算記帳工具
const BudgetTracker = ({ userId, isAuthReady }) => {
    const [budget, setBudget] = useState(initialBudget);
    const [isSaving, setIsSaving] = useState(false);
    const [newEntry, setNewEntry] = useState({ type: '支出', category: '餐飲', amount: '', note: '' });

    // Firestore 數據訂閱
    useEffect(() => {
        if (!isAuthReady || !userId || !db) return;

        const budgetRef = getPrivateDocRef('budget');

        const unsubscribe = onSnapshot(budgetRef, (docSnap) => {
            if (docSnap.exists()) {
                const data = docSnap.data();
                setBudget(prev => ({
                    ...prev,
                    totalBudget: data.totalBudget || prev.totalBudget,
                    entries: data.entries || prev.entries,
                }));
            } else {
                 // 如果文件不存在，寫入初始數據
                 setDoc(budgetRef, initialBudget, { merge: true }).catch(console.error);
            }
        }, (error) => {
            console.error("Firestore Snapshot Error:", error);
        });

        return () => unsubscribe();
    }, [userId, isAuthReady]);

    // 保存數據到 Firestore
    const saveBudget = useCallback(async (newBudget) => {
        if (!db || !userId) return;
        setIsSaving(true);
        try {
            await setDoc(getPrivateDocRef('budget'), newBudget, { merge: true });
        } catch (error) {
            console.error("Error saving budget:", error);
        } finally {
            setIsSaving(false);
        }
    }, [userId]);


    // 處理新增記帳項目
    const handleAddEntry = (e) => {
        e.preventDefault();
        const amount = parseFloat(newEntry.amount);
        if (isNaN(amount) || amount <= 0) return;

        const newEntryData = {
            id: Date.now(), // 簡單的唯一 ID
            ...newEntry,
            amount: amount,
        };

        const updatedEntries = [...budget.entries, newEntryData];
        const newBudget = { ...budget, entries: updatedEntries };
        setBudget(newBudget);
        saveBudget(newBudget);
        
        setNewEntry({ type: '支出', category: '餐飲', amount: '', note: '' });
    };

    const totalSpent = budget.entries
        .filter(e => e.type === '支出')
        .reduce((sum, entry) => sum + entry.amount, 0);

    const remainingBudget = budget.totalBudget - totalSpent;
    const isOverBudget = remainingBudget < 0;

    return (
        <div className="space-y-6">
            {/* 總預算概覽 */}
            <div className="p-4 rounded-xl bg-stone-100 shadow-inner border border-stone-200">
                <div className="flex justify-between items-center mb-2">
                    <p className="text-sm text-gray-600">總預算 (JPY)</p>
                    <p className="text-xl font-bold text-gray-800">¥{budget.totalBudget.toLocaleString()}</p>
                </div>
                <div className="flex justify-between items-center mb-4 border-t pt-2">
                    <p className="text-sm text-gray-600">已花費</p>
                    <p className="text-xl font-bold text-red-600">¥{totalSpent.toLocaleString()}</p>
                </div>
                <div className="flex justify-between items-center">
                    <p className="text-base font-semibold">剩餘預算</p>
                    <p className={`text-2xl font-extrabold ${isOverBudget ? 'text-red-700' : 'text-green-600'}`}>
                        ¥{remainingBudget.toLocaleString()}
                    </p>
                </div>
            </div>

            {/* 新增記帳表單 */}
            <div className="p-4 bg-white rounded-xl shadow-lg border border-gray-100">
                <h4 className="text-lg font-semibold mb-3 border-b pb-2">新增消費</h4>
                <form onSubmit={handleAddEntry} className="space-y-3">
                    <div className="flex space-x-3">
                        <select 
                            value={newEntry.type}
                            onChange={(e) => setNewEntry({ ...newEntry, type: e.target.value })}
                            className="w-1/3 p-2 border rounded-lg bg-white focus:ring-rose-500 focus:border-rose-500"
                        >
                            <option>支出</option>
                            <option>收入</option>
                        </select>
                        <select 
                            value={newEntry.category}
                            onChange={(e) => setNewEntry({ ...newEntry, category: e.target.value })}
                            className="w-2/3 p-2 border rounded-lg bg-white focus:ring-rose-500 focus:border-rose-500"
                        >
                            <option>餐飲</option>
                            <option>交通</option>
                            <option>住宿</option>
                            <option>門票</option>
                            <option>購物</option>
                            <option>其他</option>
                        </select>
                    </div>
                    <input 
                        type="number" 
                        placeholder="金額 (¥)"
                        value={newEntry.amount}
                        onChange={(e) => setNewEntry({ ...newEntry, amount: e.target.value })}
                        className="w-full p-2 border rounded-lg focus:ring-rose-500 focus:border-rose-500"
                        min="0.01"
                        step="0.01"
                        required
                    />
                    <input 
                        type="text" 
                        placeholder="備註/用途"
                        value={newEntry.note}
                        onChange={(e) => setNewEntry({ ...newEntry, note: e.target.value })}
                        className="w-full p-2 border rounded-lg focus:ring-rose-500 focus:border-rose-500"
                    />
                    <button 
                        type="submit" 
                        className={`w-full p-2 rounded-xl text-white font-semibold transition-colors shadow-md ${isSaving ? 'bg-gray-400' : 'bg-rose-600 hover:bg-rose-700'}`}
                        disabled={isSaving}
                    >
                        {isSaving ? '儲存中...' : '新增記錄'}
                    </button>
                </form>
            </div>
            
            {/* 交易記錄列表 */}
            <div className="p-4 bg-white rounded-xl shadow-lg border border-gray-100">
                <h4 className="text-lg font-semibold mb-3 border-b pb-2">所有交易 ({budget.entries.length})</h4>
                <ul className="space-y-2 max-h-60 overflow-y-auto">
                    {budget.entries.slice().reverse().map(entry => (
                        <li key={entry.id} className="flex justify-between items-center text-sm p-2 border-b last:border-b-0 last:pb-0">
                            <div className="flex flex-col">
                                <span className="font-medium text-gray-800">{entry.category} - {entry.note}</span>
                                <span className="text-xs text-gray-500">{new Date(entry.id).toLocaleDateString('zh-TW', { month: '2-digit', day: '2-digit' })}</span>
                            </div>
                            <span className={`font-bold ${entry.type === '支出' ? 'text-red-500' : 'text-green-500'}`}>
                                {entry.type === '支出' ? '-' : '+'}{entry.amount.toLocaleString()}
                            </span>
                        </li>
                    ))}
                </ul>
            </div>
        </div>
    );
};

// ----------------------------------------------------
// --- 主頁面元件 ---
// ----------------------------------------------------

const ItineraryView = () => (
    <div className="space-y-6">
        <h2 className="text-2xl font-bold text-gray-800 pt-4 px-4 sm:px-6">行程總覽 (1/30 - 2/4)</h2>
        {itineraryData.map(dayData => (
            <div key={dayData.date} className="px-4 sm:px-6">
                <div className="sticky top-0 z-10 bg-stone-50/90 backdrop-blur-sm pt-2 pb-2 rounded-xl shadow-sm">
                    <h3 className="text-xl font-extrabold text-gray-700 border-l-4 border-rose-500 pl-3 leading-tight">
                        {new Date(dayData.date).toLocaleDateString('zh-TW', { month: '2-digit', day: '2-digit' })} ({dayData.day})
                    </h3>
                </div>
                
                <div className="mt-4">
                    <WeatherDisplay date={dayData.date} />
                    {/* 這裡使用一個柔和的邊界線來模擬時間軸 */}
                    <div className="pl-2 border-l-2 border-rose-100 space-y-4">
                        {dayData.items.map((item, index) => (
                            <ItineraryCard key={index} item={item} />
                        ))}
                    </div>
                </div>
            </div>
        ))}
    </div>
);

const ToolsView = ({ isAuthReady, userId }) => (
    <div className="space-y-6 p-4 sm:p-6">
        <h2 className="text-2xl font-bold text-gray-800 pt-4 pb-2 border-b">其他旅遊工具</h2>

        {/* 航班資訊 */}
        <div className="p-4 bg-white rounded-xl shadow-lg border border-gray-100">
            <h3 className="text-xl font-semibold mb-3 flex items-center">
                <span className="text-2xl mr-2">✈️</span> 航班資訊
            </h3>
            <div className="space-y-2 text-gray-700">
                <p><strong>去程:</strong> {initialToolsData.flight.outbound}</p>
                <p><strong>回程:</strong> {initialToolsData.flight.return}</p>
                <p><strong>PNR/訂位代號:</strong> <span className="text-rose-600 font-bold">{initialToolsData.flight.pnr}</span></p>
                <p className="text-sm text-gray-500 border-t pt-2 mt-2">{initialToolsData.flight.note}</p>
            </div>
        </div>

        {/* 住宿資訊 */}
        <div className="p-4 bg-white rounded-xl shadow-lg border border-gray-100">
            <h3 className="text-xl font-semibold mb-3 flex items-center">
                <span className="text-2xl mr-2">🏨</span> 住宿資訊
            </h3>
            <ul className="space-y-3">
                {initialToolsData.accommodation.map((acc, index) => (
                    <li key={index} className="border-b pb-3 last:border-b-0 last:pb-0">
                        <p className="font-semibold text-lg text-gray-800">{acc.name}</p>
                        <p className="text-sm text-gray-600">入住: {acc.checkin} | 退房: {acc.checkout}</p>
                        <p className="text-sm text-indigo-600 font-medium">預約代號: {acc.code}</p>
                    </li>
                ))}
            </ul>
        </div>

        {/* 緊急聯絡電話 */}
        <div className="p-4 bg-white rounded-xl shadow-lg border border-gray-100">
            <h3 className="text-xl font-semibold mb-3 flex items-center">
                <span className="text-2xl mr-2">📞</span> 緊急聯絡電話
            </h3>
            <ul className="space-y-2">
                {initialToolsData.emergency.map((contact, index) => (
                    <li key={index} className="flex justify-between items-center text-gray-700">
                        <span>{contact.name}</span>
                        <a href={`tel:${contact.tel}`} className="text-red-500 font-mono font-semibold hover:underline">
                            {contact.tel}
                        </a>
                    </li>
                ))}
            </ul>
        </div>
        
        {/* 記帳/預算表 */}
        <div className="pt-2">
            <h3 className="text-xl font-semibold mb-4 flex items-center">
                <span className="text-2xl mr-2">💰</span> 記帳/預算表
            </h3>
            <BudgetTracker userId={userId} isAuthReady={isAuthReady} />
        </div>
    </div>
);


const App = () => {
    const [currentView, setCurrentView] = useState('itinerary'); // 'itinerary' or 'tools'
    const [isAuthReady, setIsAuthReady] = useState(false);
    const [userId, setUserId] = useState(null);
    const [authError, setAuthError] = useState(null);

    // 1. Firebase 認證及初始化
    useEffect(() => {
        if (!auth) {
            console.warn("Firebase not initialized. Running without persistence.");
            setIsAuthReady(true);
            setUserId(crypto.randomUUID()); // 用隨機 ID 模擬未認證用戶
            return;
        }

        const handleAuth = async () => {
            try {
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Firebase Auth Error:", error);
                setAuthError("身份驗證失敗，部分功能無法使用。");
            }
        };

        const unsubscribe = onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUserId = user.uid; // 設定全局變量
                setUserId(user.uid);
            } else {
                // 如果沒有登入，但我們需要 userId 來存儲 private data
                currentUserId = crypto.randomUUID();
                setUserId(currentUserId);
            }
            setIsAuthReady(true);
        });

        handleAuth();

        return () => unsubscribe();
    }, []);


    const isItinerary = currentView === 'itinerary';

    return (
        <div className="min-h-screen bg-stone-50 flex justify-center font-sans">
            {/* 模擬手機邊框的容器 - 日式極簡風格 */}
            <div className="w-full max-w-md bg-white shadow-2xl relative">
                {/* 狀態提示 */}
                {authError && (
                    <div className="bg-red-100 text-red-700 p-2 text-center text-sm">
                        {authError}
                    </div>
                )}
                {!isAuthReady && (
                    <div className="bg-indigo-100 text-indigo-700 p-2 text-center text-sm">
                        正在準備資料...
                    </div>
                )}
                
                {/* 頂部標題 - 更簡約的風格 */}
                <header className="p-4 border-b border-gray-100 sticky top-0 bg-white z-20 shadow-lg">
                    <h1 className="text-2xl font-extrabold text-gray-900 tracking-tight">
                        關西極簡旅程
                    </h1>
                    <p className="text-sm text-gray-500 mt-1">1/30 (TPE) - 2/4 (KIX)</p>
                </header>

                {/* 主內容區塊 */}
                <main className="pb-20">
                    {isItinerary ? (
                        <ItineraryView isAuthReady={isAuthReady} userId={userId} />
                    ) : (
                        <ToolsView isAuthReady={isAuthReady} userId={userId} />
                    )}
                </main>

                {/* 底部導航 (手機 App 樣式) */}
                <nav className="fixed bottom-0 left-0 right-0 max-w-md mx-auto bg-white border-t border-gray-100 shadow-2xl z-30 p-2 rounded-t-2xl">
                    <div className="flex justify-around items-center">
                        <button
                            onClick={() => setCurrentView('itinerary')}
                            className={`flex flex-col items-center p-2 rounded-xl transition-colors ${
                                isItinerary ? 'text-rose-600 font-bold' : 'text-gray-500 hover:text-rose-500'
                            }`}
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" className="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z" />
                            </svg>
                            <span className="text-xs mt-1">行程總覽</span>
                        </button>
                        
                        <button
                            onClick={() => setCurrentView('tools')}
                            className={`flex flex-col items-center p-2 rounded-xl transition-colors ${
                                !isItinerary ? 'text-rose-600 font-bold' : 'text-gray-500 hover:text-rose-500'
                            }`}
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" className="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.526.323 1.054.516 1.624.516.57 0 1.098-.193 1.624-.516z" />
                                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />
                            </svg>
                            <span className="text-xs mt-1">其他工具</span>
                        </button>
                    </div>
                    
                </nav>
            </div>
        </div>
    );
};

export default App;
