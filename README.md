// ==UserScript==
// @name         **EdayLotto ตัดเลขปิด (ระบบสมาชิก Firebase + นับเวลาเรียลไทม์ + แจ้งเตือนกลางจอ)**
// @namespace    http://tampermonkey.net/
// @version      6.4
// @description  ระบบตัดเลขปิดพร้อมระบบสมาชิก, Firebase Realtime, ป้องกันล็อกอินซ้อน พร้อมแจ้งเตือนกลางจอ
// @author       You
// @match        *://*.edaylotto.com/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    const FIREBASE_DB_URL = 'https://binnnn-4397e-default-rtdb.asia-southeast1.firebasedatabase.app';

    const STORAGE_KEY_POS = 'edaylotto_cut_panel_pos';
    const STORAGE_KEY_PIN = 'edaylotto_cut_panel_pin';
    const STORAGE_KEY_TEXT = 'edaylotto_cut_panel_text';
    const STORAGE_KEY_SESSION = 'edaylotto_user_session';

    let currentUser = null;
    let currentExpireData = null;
    let currentSessionId = 'sess_' + Math.random().toString(36).substring(2, 15);
    let heartbeatInterval = null;
    let countdownInterval = null;

    function showCenterAlert(message, onConfirm) {
        if (document.getElementById('custom-center-alert')) return;

        let modal = document.createElement('div');
        modal.id = 'custom-center-alert';
        modal.style.cssText = `
            position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
            background: rgba(0, 0, 0, 0.75); z-index: 999999; display: flex;
            justify-content: center; align-items: center; font-family: sans-serif;
            backdrop-filter: blur(4px);
        `;

        modal.innerHTML = `
            <div style="
                background: #ffffff; padding: 30px; border-radius: 20px; width: 340px;
                text-align: center; box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
                border: 3px solid #ff4757; animation: alertBounce 0.3s ease;
            ">
                <div style="font-size: 45px; margin-bottom: 10px;">⚠️</div>
                <h3 style="margin: 0 0 10px 0; color: #ff4757; font-size: 18px;">แจ้งเตือนจากระบบ</h3>
                <p style="color: #4a4a4a; font-size: 14px; margin-bottom: 20px; line-height: 1.5; font-weight: bold;">
                    ${message}
                </p>
                <button id="alert-confirm-btn" style="
                    width: 100%; padding: 12px; background: #ff4757; color: white;
                    border: none; border-radius: 12px; font-weight: bold; font-size: 15px;
                    cursor: pointer; box-shadow: 0 4px 12px rgba(255, 71, 87, 0.4);
                ">ตกลง</button>
            </div>
            <style>
                @keyframes alertBounce {
                    0% { transform: scale(0.7); opacity: 0; }
                    100% { transform: scale(1); opacity: 1; }
                }
            </style>
        `;
        document.body.appendChild(modal);

        document.getElementById('alert-confirm-btn').addEventListener('click', () => {
            modal.remove();
            if (onConfirm) onConfirm();
        });
    }

    function loadFirebaseSDK() {
        return new Promise((resolve) => {
            if (window.firebase && window.firebase.database) {
                resolve(window.firebase);
                return;
            }
            let script1 = document.createElement('script');
            script1.src = 'https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js';
            script1.onload = () => {
                let script2 = document.createElement('script');
                script2.src = 'https://www.gstatic.com/firebasejs/8.10.1/firebase-database.js';
                script2.onload = () => {
                    if (!window.firebase.apps.length) {
                        window.firebase.initializeApp({ databaseURL: FIREBASE_DB_URL });
                    }
                    resolve(window.firebase);
                };
                document.head.appendChild(script2);
            };
            document.head.appendChild(script1);
        });
    }

    async function initAuth() {
        await loadFirebaseSDK();
        let savedSession = localStorage.getItem(STORAGE_KEY_SESSION);

        if (savedSession) {
            try {
                let sessionData = JSON.parse(savedSession);
                await verifyAndLogin(sessionData.username, sessionData.password, true);
            } catch (e) {
                showLoginModal();
            }
        } else {
            showLoginModal();
        }
    }

    function showLoginModal(errorMsg = '') {
        if (document.getElementById('edaylotto-login-modal')) {
            if (errorMsg) document.getElementById('login-err').innerText = errorMsg;
            return;
        }

        let modal = document.createElement('div');
        modal.id = 'edaylotto-login-modal';
        modal.style.cssText = `
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.5); z-index: 10000; display: flex;
            justify-content: center; align-items: center; font-family: sans-serif;
        `;

        modal.innerHTML = `
            <div style="background: white; padding: 25px; border-radius: 20px; width: 320px; box-shadow: 0 10px 25px rgba(255,107,129,0.2); border: 2px solid #ffdde1;">
                <h3 style="margin-top: 0; color: #ff6b81; text-align: center; font-family: sans-serif;">🔐 เข้าสู่ระบบตัดเลขปิด</h3>
                <div id="login-err" style="color: #d32f2f; font-size: 12px; text-align: center; margin-bottom: 10px; font-weight: bold;">${errorMsg}</div>
                <div style="margin-bottom: 12px;">
                    <label style="font-size: 13px; font-weight: bold; display: block; margin-bottom: 4px; color: #666;">ชื่อผู้ใช้งาน (Username)</label>
                    <input type="text" id="login-user" style="width: 100%; padding: 10px; box-sizing: border-box; border: 2px solid #ffccd5; border-radius: 12px; outline: none; background: #fff9fa;">
                </div>
                <div style="margin-bottom: 15px;">
                    <label style="font-size: 13px; font-weight: bold; display: block; margin-bottom: 4px; color: #666;">รหัสผ่าน (Password)</label>
                    <input type="password" id="login-pass" style="width: 100%; padding: 10px; box-sizing: border-box; border: 2px solid #ffccd5; border-radius: 12px; outline: none; background: #fff9fa;">
                </div>
                <button id="login-btn" style="width: 100%; padding: 12px; background: #ff6b81; color: white; border: none; border-radius: 12px; font-weight: bold; cursor: pointer; font-size: 15px; box-shadow: 0 4px 10px rgba(255,107,129,0.3);">เข้าสู่ระบบ</button>
            </div>
        `;
        document.body.appendChild(modal);

        document.getElementById('login-btn').addEventListener('click', async () => {
            let u = document.getElementById('login-user').value.trim();
            let p = document.getElementById('login-pass').value.trim();
            if (!u || !p) {
                document.getElementById('login-err').innerText = 'กรุณากรอกข้อมูลให้ครบถ้วน';
                return;
            }
            document.getElementById('login-btn').innerText = 'กำลังตรวจสอบ...';
            await verifyAndLogin(u, p, false);
        });
    }

    async function verifyAndLogin(username, password, isAutoLogin) {
        let db = window.firebase.database();
        let snapshot = await db.ref('users/' + username).once('value');
        let userData = snapshot.val();
        let loginModal = document.getElementById('edaylotto-login-modal');

        if (!userData || userData.password !== password) {
            if (loginModal) { document.getElementById('login-btn').innerText = 'เข้าสู่ระบบ'; showLoginModal('ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง'); }
            if (isAutoLogin) localStorage.removeItem(STORAGE_KEY_SESSION);
            return;
        }

        let now = new Date().getTime();
        let expireVal = userData.expireDate;
        if (expireVal !== "FOREVER") {
            let expNum = Number(expireVal);
            if (now > expNum) {
                if (loginModal) { document.getElementById('login-btn').innerText = 'เข้าสู่ระบบ'; showLoginModal('บัญชีนี้หมดอายุการใช้งานแล้ว ⚠️'); }
                localStorage.removeItem(STORAGE_KEY_SESSION);
                return;
            }
        }

        currentUser = username;
        currentExpireData = expireVal;
        localStorage.setItem(STORAGE_KEY_SESSION, JSON.stringify({ username, password }));
        if (loginModal) loginModal.remove();

        db.ref('users/' + username + '/currentSession').set(currentSessionId);
        startHeartbeat(username);
        createFloatingPanel();
    }

    function startHeartbeat(username) {
        let db = window.firebase.database();

        db.ref('users/' + username + '/currentSession').on('value', (snapshot) => {
            let activeSess = snapshot.val();
            if (activeSess && activeSess !== currentSessionId) {
                let panel = document.getElementById('cut-both-panel');
                if (panel) panel.style.display = 'none';

                showCenterAlert('มีการเข้าซ้อนกันระบบจะทำการหยุดใช้งาน', () => {
                    localStorage.removeItem(STORAGE_KEY_SESSION);
                    location.reload();
                });
            }
        });

        db.ref('users/' + username + '/expireDate').on('value', (snapshot) => {
            let newExp = snapshot.val();
            if (newExp !== null) {
                currentExpireData = newExp;
                let now = new Date().getTime();
                if (newExp !== "FOREVER" && now > Number(newExp)) {
                    showCenterAlert('⚠️ บัญชีของคุณหมดอายุการใช้งานแล้ว', () => {
                        localStorage.removeItem(STORAGE_KEY_SESSION);
                        location.reload();
                    });
                }
            }
        });

        if (heartbeatInterval) clearInterval(heartbeatInterval);
        heartbeatInterval = setInterval(() => {
            db.ref('users/' + username + '/lastActive').set(Date.now());
        }, 10000);
    }

    function createFloatingPanel() {
        if (document.getElementById('cut-both-panel')) return;

        let panel = document.createElement('div');
        panel.id = 'cut-both-panel';

        let savedPos = localStorage.getItem(STORAGE_KEY_POS);
        let savedPin = localStorage.getItem(STORAGE_KEY_PIN) === 'true';
        let savedText = localStorage.getItem(STORAGE_KEY_TEXT) || '';

        let initialStyle = `
            position: fixed; z-index: 9999; background: #ffffff;
            border: 2px solid #ff6b81; border-radius: 12px; padding: 12px;
            width: 285px; box-shadow: 0 4px 15px rgba(0,0,0,0.2);
            font-family: sans-serif; user-select: none;
        `;

        if (savedPos) {
            try {
                let pos = JSON.parse(savedPos);
                initialStyle += ` left: ${pos.left}px; top: ${pos.top}px; bottom: auto; right: auto;`;
            } catch(e) { initialStyle += ` bottom: 20px; right: 20px;`; }
        } else { initialStyle += ` bottom: 20px; right: 20px;`; }

        panel.style.cssText = initialStyle;

        panel.innerHTML = `
            <div id="panel-header" style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; cursor: move; background: #fff0f5; padding: 6px 10px; border-radius: 8px;">
                <div>
                    <span style="font-weight: bold; font-size: 13px; color: #ff6b81;">✂️ ตัดเลขปิด (${currentUser})</span>
                    <div id="panel-countdown" style="font-size: 11px; color: #2e7d32; font-weight: bold; margin-top: 2px;">กำลังโหลดเวลา...</div>
                </div>
                <div>
                    <span id="pin-panel-btn" title="ล็อก/ปลดล็อกตำแหน่ง" style="cursor: pointer; font-size: 13px; margin-right: 6px;">${savedPin ? '🔒' : '📍'}</span>
                    <span id="minimize-cut-panel" title="ย่อ/ขยาย" style="cursor: pointer; font-size: 12px; color: #666; font-weight: bold;">[—]</span>
                </div>
            </div>

            <div id="panel-content">
                <textarea id="target-both-numbers" rows="3" placeholder="กรอกเลขปิด (กด Enter เพื่อตัด)" style="
                    width: 100%; padding: 8px; box-sizing: border-box; border: 1px solid #ccc;
                    border-radius: 8px; font-size: 13px; resize: vertical; margin-bottom: 8px; outline: none;
                ">${savedText}</textarea>

                <div style="display: flex; gap: 6px;">
                    <button id="execute-cut-both-btn" style="
                        flex: 1; padding: 8px; background-color: #ff6b81; color: white;
                        border: none; border-radius: 8px; font-size: 13px; font-weight: bold; cursor: pointer;
                    ">ตัดตัวปิด</button>
                    <button id="clear-cut-both-btn" style="
                        padding: 8px 10px; background-color: #6c757d; color: white;
                        border: none; border-radius: 8px; font-size: 13px; font-weight: bold; cursor: pointer;
                    " title="ล้างตัวปิด">ล้าง</button>
                </div>
            </div>
        `;

        document.body.appendChild(panel);
        startCountdownDisplay();

        let isPinned = savedPin;
        let header = document.getElementById('panel-header');
        let pinBtn = document.getElementById('pin-panel-btn');
        if (isPinned) { pinBtn.style.opacity = '0.5'; header.style.cursor = 'default'; }

        let isDragging = false, startX, startY, initialLeft, initialTop;
        header.addEventListener('mousedown', (e) => {
            if (isPinned) return;
            isDragging = true;
            startX = e.clientX; startY = e.clientY;
            let rect = panel.getBoundingClientRect();
            panel.style.left = rect.left + 'px'; panel.style.top = rect.top + 'px';
            panel.style.bottom = 'auto'; panel.style.right = 'auto';
            initialLeft = rect.left; initialTop = rect.top;
            document.addEventListener('mousemove', onMouseMove);
            document.addEventListener('mouseup', onMouseUp);
        });

        function onMouseMove(e) {
            if (!isDragging) return;
            panel.style.left = (initialLeft + (e.clientX - startX)) + 'px';
            panel.style.top = (initialTop + (e.clientY - startY)) + 'px';
        }

        function onMouseUp() {
            if (!isDragging) return;
            isDragging = false;
            document.removeEventListener('mousemove', onMouseMove);
            document.removeEventListener('mouseup', onMouseUp);
            let rect = panel.getBoundingClientRect();
            localStorage.setItem(STORAGE_KEY_POS, JSON.stringify({ left: rect.left, top: rect.top }));
        }

        pinBtn.addEventListener('click', () => {
            isPinned = !isPinned;
            localStorage.setItem(STORAGE_KEY_PIN, isPinned);
            pinBtn.style.opacity = isPinned ? '0.5' : '1';
            header.style.cursor = isPinned ? 'default' : 'move';
            pinBtn.innerText = isPinned ? '🔒' : '📍';
        });

        let content = document.getElementById('panel-content');
        let minimizeBtn = document.getElementById('minimize-cut-panel');
        minimizeBtn.addEventListener('click', () => {
            let isHidden = content.style.display === 'none';
            content.style.display = isHidden ? 'block' : 'none';
            minimizeBtn.innerText = isHidden ? '[—]' : '[+]';
        });

        let textarea = document.getElementById('target-both-numbers');
        textarea.addEventListener('input', () => localStorage.setItem(STORAGE_KEY_TEXT, textarea.value));
        document.getElementById('execute-cut-both-btn').addEventListener('click', processCutFlow);
        document.getElementById('clear-cut-both-btn').addEventListener('click', () => {
            textarea.value = '';
            localStorage.removeItem(STORAGE_KEY_TEXT);
        });
        textarea.addEventListener('keydown', (e) => {
            if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); processCutFlow(); }
        });
    }

    function startCountdownDisplay() {
        if (countdownInterval) clearInterval(countdownInterval);

        countdownInterval = setInterval(() => {
            let cdElem = document.getElementById('panel-countdown');
            if (!cdElem) return;

            if (currentExpireData === "FOREVER") {
                cdElem.innerHTML = `♾️ สถานะ: ใช้งานถาวร`;
                cdElem.style.color = '#8e24aa';
                return;
            }

            let now = new Date().getTime();
            let expNum = Number(currentExpireData);
            let diffMs = expNum - now;

            if (diffMs <= 0) {
                cdElem.innerHTML = `⚠️ บัญชีหมดอายุแล้ว`;
                cdElem.style.color = '#d32f2f';
                return;
            }

            let totalSeconds = Math.floor(diffMs / 1000);
            let days = Math.floor(totalSeconds / (3600 * 24));
            let hours = Math.floor((totalSeconds % (3600 * 24)) / 3600);
            let minutes = Math.floor((totalSeconds % 3600) / 60);
            let seconds = totalSeconds % 60;

            if (days > 0) {
                cdElem.innerHTML = `⏳ เหลือเวลา: ${days}วัน ${hours}ชม. ${minutes}นาที`;
            } else {
                cdElem.innerHTML = `⏳ เหลือเวลา: ${hours}ชม. ${minutes}น. ${seconds}วิ`;
            }
            cdElem.style.color = '#2e7d32';
        }, 1000);
    }

    function generatePermutations(str) {
        if (str.length <= 1) return [str];
        let allPerms = new Set();
        for (let i = 0; i < str.length; i++) {
            let char = str[i];
            let remainingStr = str.slice(0, i) + str.slice(i + 1);
            for (let subPerm of generatePermutations(remainingStr)) {
                allPerms.add(char + subPerm);
            }
        }
        return Array.from(allPerms);
    }

    async function processCutFlow() {
        let textarea = document.getElementById('target-both-numbers');
        let inputVal = textarea.value.trim();
        if (!inputVal) return;

        let rawList = inputVal.split(/[\s,]+/).filter(item => item.length > 0);
        let targetSet = new Set();

        rawList.forEach(num => {
            targetSet.add(num);
            if (num.length === 2 && num[0] !== num[1]) {
                targetSet.add(num[1] + num[0]);
            } else if (num.length === 3) {
                let perms = generatePermutations(num);
                perms.forEach(p => targetSet.add(p));
            }
        });

        let targetList = Array.from(targetSet);
        if (targetList.length === 0) return;

        let removeCount = 0;
        removeCount += cutFromActiveButtons(targetList);

        let processedRows = new Set();
        let hasMoreRows = true;

        while (hasMoreRows) {
            hasMoreRows = false;
            let tableRows = document.querySelectorAll('table tbody tr');

            for (let row of tableRows) {
                if (processedRows.has(row)) continue;
                let cols = row.querySelectorAll('td');
                if (cols.length < 2) continue;

                let numberColText = '';
                for (let i = 1; i < cols.length - 2; i++) {
                    numberColText += ' ' + cols[i].innerText;
                }
                if (!numberColText.trim() && cols[1]) {
                    numberColText = cols[1].innerText;
                }

                let hasTarget = targetList.some(num => new RegExp(`\\b${num}\\b`).test(numberColText));

                if (hasTarget) {
                    processedRows.add(row);
                    hasMoreRows = true;

                    let editBtn = row.querySelector('button[class*="edit"], a[class*="edit"], i.fa-edit, i.fa-pencil, .fa-edit, .fa-pencil-alt');
                    if (!editBtn) {
                        row.querySelectorAll('button, a, div').forEach(b => {
                            if (b.children.length === 0 && (b.innerText.includes('edit') || b.className.includes('blue'))) {
                                editBtn = b;
                            }
                        });
                    }

                    if (editBtn) {
                        editBtn.click();
                        await new Promise(r => setTimeout(r, 300));
                        removeCount += cutFromActiveButtons(targetList);
                        await new Promise(r => setTimeout(r, 200));

                        let addBillBtn = null;
                        document.querySelectorAll('button, input[type="button"], input[type="submit"]').forEach(b => {
                            if ((b.innerText || b.value || '').trim().includes('เพิ่มบิล')) {
                                addBillBtn = b;
                            }
                        });

                        if (addBillBtn) {
                            addBillBtn.click();
                            await new Promise(r => setTimeout(r, 400));
                        }
                    }
                    break;
                }
            }
        }

        let btn = document.getElementById('execute-cut-both-btn');
        let originalText = btn.innerHTML;
        btn.innerHTML = `✅ ตัดแล้ว (${removeCount})`;
        btn.style.backgroundColor = '#28a745';
        setTimeout(() => {
            btn.innerHTML = originalText;
            btn.style.backgroundColor = '#ff6b81';
        }, 1500);
    }

    function cutFromActiveButtons(targetList) {
        let count = 0;
        document.querySelectorAll('button, div, span').forEach(el => {
            let text = el.innerText.trim();
            if (el.children.length === 0 && targetList.includes(text)) {
                el.click();
                count++;
            }
        });
        return count;
    }

    initAuth();
})();
