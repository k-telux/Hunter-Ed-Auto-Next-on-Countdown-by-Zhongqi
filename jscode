// ==UserScript==
// @name         Hunter-Ed Auto Next on Countdown
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  When #countdown-bg reaches 100% width, wait 1s and click Next on hunter-ed course pages.
// @author       Zhongqi
// @match        https://www.hunter-ed.com/course/content*
// @match        https://*.hunter-ed.com/course/content*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    // 配置：触发阈值（百分比）和延迟（ms）
    const TRIGGER_PERCENT = 99.5; // >= 99.5% 视为 100%
    const CLICK_DELAY_MS = 1000;   // 1s 延迟再点击
    const CHECK_INTERVAL_MS = 500; // 如果没找到元素，轮询间隔

    let alreadyClickedForThisCycle = false;
    let lastObservedWidth = null;

    function log(...args) {
        console.debug('[HunterAutoNext]', ...args);
    }

    // 返回 #countdown-bg 元素（如果在其他 frame / shadow DOM 里则需要特殊处理；此处假设在主 document）
    function getCountdownEl() {
        return document.getElementById('countdown-bg');
    }

    // 主检测函数：尝试从 style.width 或实际像素宽度计算百分比
    function getProgressPercent(el) {
        if (!el) return null;

        // 优先尝试 style width（通常是 "34.6667%" 这种）
        const styleWidth = (el.getAttribute && el.getAttribute('style')) || '';
        const match = styleWidth.match(/width\s*:\s*([0-9.]+)%/i);
        if (match && match[1]) {
            const p = parseFloat(match[1]);
            if (!isNaN(p)) return p;
        }

        // fallback：使用 offsetWidth / parent.offsetWidth 计算实际百分比
        const parent = el.parentElement || el.offsetParent;
        if (parent && parent.offsetWidth > 0) {
            const p = (el.offsetWidth / parent.offsetWidth) * 100;
            return p;
        }

        return null;
    }

    // 点击 Next 按钮（更稳妥地选取 rel="next" 的链接或 .btn-success）
    function clickNext() {
        if (alreadyClickedForThisCycle) {
            log('Already clicked in this cycle, skipping.');
            return;
        }

        // 优先使用 rel="next"
        let nextAnchor = document.querySelector('#next-button a[rel="next"], a[rel="next"].btn-success');

        // 备用选择：进度区本身（用户要求“点击这个位置的按钮” — 有些页面把点击事件绑定在 #countdown）
        if (!nextAnchor) {
            nextAnchor = document.querySelector('#countdown, #countdown a, #countdown-bg');
        }

        if (nextAnchor) {
            alreadyClickedForThisCycle = true;
            log('Will click next element in', CLICK_DELAY_MS, 'ms:', nextAnchor);
            setTimeout(() => {
                try {
                    nextAnchor.click();
                    log('Clicked next.');
                } catch (err) {
                    console.error('[HunterAutoNext] click failed:', err);
                }
            }, CLICK_DELAY_MS);
        } else {
            console.warn('[HunterAutoNext] Could not find Next button to click.');
        }
    }

    // 观察器回调：当 style 或子节点变化时检查进度
    function setupObserver() {
        const countdown = getCountdownEl();
        if (!countdown) {
            log('countdown-bg not found. Will retry with interval.');
            // 如果找不到元素，就用轮询持续尝试
            const intervalId = setInterval(() => {
                const c = getCountdownEl();
                if (c) {
                    clearInterval(intervalId);
                    log('Found countdown-bg, setting up observer.');
                    createObserver(c);
                    // 立刻做一次检查
                    checkOnce(c);
                }
            }, CHECK_INTERVAL_MS);
            return;
        }
        createObserver(countdown);
        checkOnce(countdown);
    }

    function createObserver(element) {
        // 如果已有 observer，则先 disconnect（为确保 idempotent）
        if (window._hunterAutoNextObserver) {
            try { window._hunterAutoNextObserver.disconnect(); } catch(e){}
        }

        const observer = new MutationObserver(mutations => {
            // 遍历 mutation，看 style attribute 变化或子List变化
            for (const m of mutations) {
                if (m.type === 'attributes' && m.attributeName === 'style') {
                    checkOnce(element);
                } else if (m.type === 'childList') {
                    checkOnce(element);
                }
            }
        });

        observer.observe(element, {
            attributes: true, // style 变化
            attributeFilter: ['style'],
            childList: true,
            subtree: false
        });

        // 保存全局引用以便后续可断开
        window._hunterAutoNextObserver = observer;
        log('MutationObserver created for #countdown-bg');
    }

    // 单次检查函数（会去重，避免重复触发）
    function checkOnce(element) {
        const p = getProgressPercent(element);
        if (p === null) {
            log('Progress percent unknown yet.');
            return;
        }

        // 若进度变化，记录
        if (lastObservedWidth === null || Math.abs(p - lastObservedWidth) > 0.0001) {
            log('Progress updated:', p.toFixed(4) + '%');
            lastObservedWidth = p;
            // 当进度变小（例如页面跳转回来或重置）时，允许再次点击
            if (p < TRIGGER_PERCENT) {
                alreadyClickedForThisCycle = false;
            }
        }

        if (p >= TRIGGER_PERCENT) {
            log('Progress reached threshold:', p.toFixed(2) + '%, scheduling click.');
            clickNext();
        }
    }

    // 防止在单页应用 / Turbolinks 情况下因导航没生效，重新初始化
    function attachNavigationWatcher() {
        // 网站可能使用 turbolinks 等，监听 DOMContentLoaded 和 pushState/popstate
        window.addEventListener('DOMContentLoaded', () => {
            setTimeout(setupObserver, 300); // 小延迟确保元素被渲染
        });

        // history navigation
        window.addEventListener('popstate', () => {
            setTimeout(setupObserver, 200);
        });

        // 覆盖 pushState/replaceState 以便检测 SPA 导航（常见于 Rails/turbolinks）
        (function(history){
            const pushState = history.pushState;
            history.pushState = function(state) {
                if (typeof history.onpushstate === 'function') {
                    history.onpushstate({state: state});
                }
                const ret = pushState.apply(history, arguments);
                setTimeout(setupObserver, 200);
                return ret;
            };
            const replaceState = history.replaceState;
            history.replaceState = function(state) {
                const ret = replaceState.apply(history, arguments);
                setTimeout(setupObserver, 200);
                return ret;
            };
        })(window.history);
    }

    // 启动
    function start() {
        attachNavigationWatcher();
        // 直接运行一次（页面可能已经加载完）
        setTimeout(setupObserver, 200);
    }

    start();

})();
