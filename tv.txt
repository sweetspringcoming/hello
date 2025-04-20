// ==UserScript==
// @name         camera/study 页面协同监控
// @namespace    http://tampermonkey.net/
// @version      3.5
// @description  自动监控
// @match        https://gxejbx.ok99ok99.com/stu/camera.aspx*
// @match        https://gxejbx.ok99ok99.com/stu/study_new_v3.aspx*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(function () {
    'use strict';

    const url = location.href;

    function playBeep(times = 1) {
        let count = 0;
        const beep = () => {
            const ctx = new AudioContext();
            const osc = ctx.createOscillator();
            osc.type = "sine";
            osc.frequency.setValueAtTime(880, ctx.currentTime);
            osc.connect(ctx.destination);
            osc.start();
            osc.stop(ctx.currentTime + 0.4);
            count++;
            if (count < times) {
                setTimeout(beep, 500);
            }
        };
        beep();
    }

    function getValueByXPath(path) {
        const result = document.evaluate(path, document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null);
        const node = result.singleNodeValue;
        return node ? node.textContent.trim() : null;
    }

    function getElementByXPath(path) {
        return document.evaluate(path, document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
    }

    // camera 
    if (url.includes("camera.aspx")) {
        const xpath = '//*[@id="divpage"]/div/div';
        const monitorInterval = 30000;
        let lastTotal = localStorage.getItem("lastTotalValue");
        if (lastTotal !== null) lastTotal = parseInt(lastTotal);

        function extractTotal(text) {
            const match = text.match(/共(\d+)条/);
            return match ? parseInt(match[1]) : null;
        }

        function monitor() {
            const rawText = getValueByXPath(xpath);
            if (!rawText) return;
            const currentTotal = extractTotal(rawText);
            console.log("当前共x条:", currentTotal);

            if (currentTotal !== null && lastTotal !== null && currentTotal !== lastTotal) {
                console.log(`“共x条”发生变化: ${lastTotal} -> ${currentTotal}`);
                playBeep(6);
                localStorage.setItem("needStudyAction", "1");
            }

            lastTotal = currentTotal;
            localStorage.setItem("lastTotalValue", currentTotal);
        }

        setTimeout(() => {
            monitor();
            setTimeout(() => {
                location.href = location.href;
            }, monitorInterval);
        }, 2000);
    }

    // study
    else if (url.includes("study_new_v3.aspx")) {
        const planXPath = '/html/body/div[1]/div[1]/div[1]/div[1]/div[2]/p[1]/label';
        const totalXPath = '/html/body/div[1]/div[1]/div[1]/div[1]/div[2]/p[2]/label';
        const actionButtonXPath = '/html/body/div[1]/div[1]/div[1]/div[2]/button';
        const continueBtnSelector = '#TB_ajaxContent > div > div.layer_btn.p_20.text_center > a';
        const authInputXPath = '/html/body/div[10]/div[2]/div/div[2]/div/p[2]/input';

        function getTimeSeconds(xpath) {
            const text = getValueByXPath(xpath);
            if (!text) return 0;
            const parts = text.split(":").map(Number);
            if (parts.length !== 3) return 0;
            return parts[0] * 3600 + parts[1] * 60 + parts[2];
        }

        function isWechatAuthRequired() {
            const input = getElementByXPath(authInputXPath);
            if (!input) return false;
            const prev = input.previousElementSibling;
            return prev && prev.textContent.includes("认证码");
        }

        function clickContinueButton() {
            const btn = document.querySelector(continueBtnSelector);
            if (btn) {
                btn.click();
                console.log("✅ ");
            } else {
                console.warn("❌ 未找到");
            }
        }

        function observeForContinueButton() {
            const observer = new MutationObserver((mutations) => {
                const btn = document.querySelector(continueBtnSelector);
                if (btn && btn.offsetParent !== null) {
                    console.log("✅ ");
                    observer.disconnect();
                    clickContinueButton();
                }
            });

            observer.observe(document.body, {
                childList: true,
                subtree: true,
            });
        }

        function submitAfterPlanTime() {
            const button = getElementByXPath(actionButtonXPath);
            if (!button) return;

            button.click();
            console.log("✅ 提交");

            setTimeout(() => {
                playBeep(2);
            }, 1000);

            setTimeout(() => {
                if (isWechatAuthRequired()) {
                    console.log("⚠️ 检测认证码");
                    playBeep(60);
                } else {
                    console.log("🔁 刷新页面");
                    location.reload();

                    setTimeout(() => {
                        observeForContinueButton();
                    }, 2000);
                }
            }, 2000);
        }

        setInterval(() => {
            const needAction = localStorage.getItem("needStudyAction");
            if (needAction !== "1") return;

            const plan = getTimeSeconds(planXPath);
            const total = getTimeSeconds(totalXPath);
            console.log(`🎯 计划时间: ${plan}s, 当前累计: ${total}s`);

            if (total > plan) {
                localStorage.setItem("needStudyAction", "0");
                submitAfterPlanTime();
            }
        }, 5000);

        observeForContinueButton();
    }
})();