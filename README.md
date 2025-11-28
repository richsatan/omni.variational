# omni.variational
自动交易代码，无需下载以及钱包权限，直接f12启动
复制以下代码 到https://omni.variational.io/perpetual/BTC 网站打开即可，按f12,找到控制到 点左上角的清空乱码 粘贴下面代码回车即可运行，盈利和止损/冷却时间/单子存在时间/都可以设置，



 
// ================= V21.0 趋势刷量版 (持仓>10s + 盈利>1%) =================
const CONFIG = {
    direction: 'Buy',        // 只做多
    tradeSizePercent: 80,    // 资金比例
    
    // === 核心时间控制 ===
    minHoldTime: 10,         // 【最少】持仓 10秒 (不到时间绝不平)
    forceCloseTime: 60,      // 【最多】持仓 60秒 (超时强制刷新，防止死拿)
    
    // === 盈亏目标 ===
    targetTP: 1.0,           // 目标盈利：大于 1% 才走
    hardStopLoss: 2.5,       // 硬止损：-2.5% (紧急保命用)
    
    // === 冷却时间 ===
    minWait: 1,              // 平仓后 1秒 开下一单
    maxWait: 5,              
    
    interval: 800
};

// ================= 核心逻辑 =================
let state = 'IDLE'; 
let nextTradeTime = 0; 
let consecutiveLossCount = 0; 
let openOrderStartTime = 0;
let positionStartTime = 0; // 记录持仓开始的那一刻

console.clear();

function log(msg, color = '#00ff00') {
    const time = new Date().toLocaleTimeString();
    console.log(`%c[${time}] ${msg}`, `color: ${color}; font-weight: bold; font-size: 12px;`);
}

function getRandomInt(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
}

function triggerStrongClick(el) {
    if (!el) return;
    el.style.border = "5px solid red";
    setTimeout(() => { el.style.border = ""; }, 200);
    el.focus();
    const opts = { bubbles: true, cancelable: true, view: window };
    el.dispatchEvent(new MouseEvent('mousedown', opts));
    el.dispatchEvent(new MouseEvent('mouseup', opts));
    el.dispatchEvent(new MouseEvent('click', opts));
}

function findElementByText(text) {
    const xpath = `//*[contains(translate(text(), 'abcdefghijklmnopqrstuvwxyz', 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'), '${text.toUpperCase()}')]`;
    const result = document.evaluate(xpath, document, null, XPathResult.ORDERED_NODE_SNAPSHOT_TYPE, null);
    for (let i = 0; i < result.snapshotLength; i++) {
        const el = result.snapshotItem(i);
        if (el.offsetParent && el.tagName !== 'SCRIPT' && el.tagName !== 'STYLE') return el;
    }
    return null;
}

function getPnL() {
    const closeBtn = findElementByText('Close');
    if (!closeBtn) return null;

    let parent = closeBtn.parentElement;
    let foundText = "";
    
    for (let i = 0; i < 6; i++) { 
        if (parent) {
            const txt = parent.innerText;
            if (txt.includes('%') && /\d/.test(txt)) {
                foundText = txt;
            }
            parent = parent.parentElement;
        }
    }

    if (foundText) {
        let match = foundText.match(/\(([+-]?\d+(\.\d+)?)%\)/);
        if (match) return parseFloat(match[1]);
        match = foundText.match(/([+-]?\d+(\.\d+)?)%/);
        if (match) return parseFloat(match[1]);
    }
    return null;
}

function findConfirmBtn() {
    const keywords = ['SELL TO CLOSE', 'BUY TO CLOSE', 'CONFIRM', 'CLOSE POSITION'];
    const els = document.querySelectorAll('button, div[role="button"]');
    for (let el of els) {
        if (!el.offsetParent) continue;
        const txt = el.innerText.toUpperCase().trim();
        if (keywords.some(kw => txt.includes(kw)) && el.offsetWidth > 80) {
            return el;
        }
    }
    return null;
}

async function main() {
    log(`📈 V21.0 趋势策略启动！`, '#ff00ff');
    log(`规则: 必须持仓 >${CONFIG.minHoldTime}s | 盈利 >${CONFIG.targetTP}% 平仓`);

    setInterval(async () => {
        try {
            // 1. 寻找 Close 按钮
            const closeBtn = findElementByText('Close');
            const realCloseBtn = (closeBtn && closeBtn.getBoundingClientRect().top > 300) ? closeBtn : null;

            if (realCloseBtn) {
                // =========== 发现持仓 ===========
                if (state !== 'HOLDING') {
                    state = 'HOLDING';
                    positionStartTime = Date.now(); // 记录持仓起始时间
                    log(`✅ 开单成功！开始计时...`, '#00ffff');
                }

                const pnl = getPnL();
                // 计算持仓秒数
                const holdTime = (Date.now() - positionStartTime) / 1000;

                if (pnl !== null) {
                    const color = pnl > 0 ? '#00ff00' : '#ffff00';
                    
                    // 只有偶尔打印，避免刷屏
                    if (Math.random() > 0.8) {
                        console.log(`%c[监控] PnL:${pnl}% | 持仓:${holdTime.toFixed(1)}s`, `color: ${color}`);
                    }

                    // --- 核心平仓逻辑 ---

                    // 1. 紧急止损 (无视时间，立即跑)
                    if (pnl <= -CONFIG.hardStopLoss) {
                        log(`🛑 紧急止损 (-${pnl}%)`, '#ff0000');
                        await closePosition(realCloseBtn, 'LOSS');
                        return;
                    }

                    // 2. 时间门槛检查
                    if (holdTime < CONFIG.minHoldTime) {
                        // 如果还没到 10秒，跳过后续判断，死拿！
                        return;
                    }

                    // --- 超过10秒后，开始判断平仓 ---

                    // 3. 止盈 (必须大于 1%)
                    if (pnl >= CONFIG.targetTP) {
                        log(`💰 达标止盈 (+${pnl}%)`, '#00ff00');
                        await closePosition(realCloseBtn, 'WIN');
                    }
                    
                    // 4. 超时强平 (60秒了还没到1%，为了刷量，平掉换车)
                    else if (holdTime > CONFIG.forceCloseTime) {
                        const type = pnl > 0 ? 'WIN' : 'TIMEOUT';
                        const logColor = pnl > 0 ? '#00ff00' : '#ff9900';
                        log(`⏰ 超时强平 (${pnl}%)，换下一单`, logColor);
                        await closePosition(realCloseBtn, type);
                    }
                    
                    // 5. 如果是微亏或者微利(<1%)，且没超时，就继续拿！
                }
            } else {
                // =========== 无持仓 ===========
                if (state === 'OPENING') {
                    if (Date.now() - openOrderStartTime > 10000) {
                        log(`⚠️ 开单超时，重置`, '#ff9900');
                        state = 'IDLE';
                    }
                    return; 
                }

                if (state === 'HOLDING') {
                    // 刚平仓完
                    let waitSeconds = getRandomInt(CONFIG.minWait, CONFIG.maxWait);
                    log(`🚀 休息 ${waitSeconds}s 继续...`, '#ffff00');
                    
                    nextTradeTime = Date.now() + (waitSeconds * 1000);
                    state = 'WAITING';
                    return;
                }

                if (state === 'WAITING') {
                    if (Date.now() < nextTradeTime) return;
                    state = 'IDLE';
                }

                if (state === 'IDLE') {
                    await tryToOpenOrder();
                }
            }
        } catch (e) { console.error(e) }
    }, CONFIG.interval);
}

async function closePosition(btn, result) {
    if (result === 'LOSS') consecutiveLossCount++;
    else if (result === 'WIN') consecutiveLossCount = 0; 
    
    triggerStrongClick(btn);
    await new Promise(r => setTimeout(r, 800)); 
    
    const confirm = findConfirmBtn();
    if (confirm) {
        triggerStrongClick(confirm);
    } else {
        const backupBtn = findElementByText('Sell to Close');
        if (backupBtn) triggerStrongClick(backupBtn);
    }
    
    await new Promise(r => setTimeout(r, 4000)); 
}

async function tryToOpenOrder() {
    const buyBtn = findElementByText('Buy BTC');
    const enterSizeBtn = findElementByText('Enter Size');
    const actionBtn = buyBtn || enterSizeBtn;

    if (actionBtn && actionBtn.offsetWidth > 100) {
        if (!actionBtn.innerText.includes('Enter Size')) {
            state = 'OPENING';
            openOrderStartTime = Date.now();
            log(`⚡️ 开单！`, '#00ff00');
            triggerStrongClick(actionBtn);
            await new Promise(r => setTimeout(r, 1000));
        }
    }
}

main();
