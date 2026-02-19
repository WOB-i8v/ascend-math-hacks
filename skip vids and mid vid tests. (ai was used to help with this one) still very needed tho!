// ==UserScript==
// @name         hecker stuff
// @namespace    http://tampermonkey.net/
// @version      3.0
// @description  Skip videos, block quizzes, kill idle timeout
// @author       WOB
// @match        https://g.myascendmath.com/Ascend/contentPage.do*
// @match        https://g.myascendmath.com/Ascend/html5NewVideoplayer.html*
// @grant        none
// @run-at       document-start
// @icon         https://www.google.com/s2/favicons?domain=myascendmath.com
// ==/UserScript==

(function() {
    'use strict';

    console.log('ready');

    // ===== GLOBAL STATE =====
    let isQuizBlockingEnabled = false;
    let originalShowQuiz = null;
    let originalShowPopup = null;
    let idleTimers = [];
    let idleIntervals = [];
    const originalSetTimeout = window.setTimeout;
    const originalSetInterval = window.setInterval;

    // ===== IDLE TIMEOUT KILLER (Runs at document-start) =====

    // Override timers that are likely idle monitors (30s, 60s, 5min, 10min)
    window.setTimeout = function(...args) {
        const delay = args[1];
        if (delay === 30000 || delay === 60000 || delay === 300000 || delay === 600000) {
            console.log('Blocked idle timeout setTimeout:', delay + 'ms');
            const dummyId = Math.random();
            idleTimers.push(dummyId);
            return dummyId;
        }
        return originalSetTimeout.apply(this, args);
    };

    window.setInterval = function(...args) {
        const delay = args[1];
        if (delay === 30000 || delay === 60000) {
            console.log('Blocked idle monitoring interval:', delay + 'ms');
            const dummyId = Math.random();
            idleIntervals.push(dummyId);
            return dummyId;
        }
        return originalSetInterval.apply(this, args);
    };

    // Block idle timeout network requests
    const originalFetch = window.fetch;
    window.fetch = function(...args) {
        const url = args[0]?.toString() || '';
        if (url.includes('idleTimeout') || url.includes('checkUserSession')) {
            console.log('Blocked idle request:', url);
            return Promise.resolve({ ok: true, status: 200, json: () => Promise.resolve({ sessionValid: true, timeout: false }) });
        }
        return originalFetch.apply(this, args);
    };

    // Simulate constant user activity (every 45 seconds)
    setInterval(() => {
        ['mousemove', 'keydown', 'click'].forEach(ev => {
            document.dispatchEvent(new Event(ev, { bubbles: true }));
        });
        console.log('Simulated activity');
    }, 45000);

    console.log('Idle timeout protection: ACTIVE');

    // ===== BUTTON INJECTION =====

    function injectButtons() {
        const checkInterval = setInterval(() => {
            const video = document.getElementById('video');
            if (video && video.duration > 0) {
                clearInterval(checkInterval);
                createSkipButton(video);
                createBlockButton();
                createIdleButton();
            }
        }, 500);
        setTimeout(() => clearInterval(checkInterval), 15000);
    }

    // ===== SKIP VIDEO BUTTON =====

    function createSkipButton(video) {
        const btn = document.createElement('button');
        btn.textContent = 'Skip Video';
        btn.id = 'ascend-skip-btn';
        btn.title = 'Ctrl+Shift+S';
        btn.style.cssText = `
            position: fixed; top: 20px; right: 20px; z-index: 999999;
            background: linear-gradient(135deg, #ff4444, #cc0000);
            color: white; border: none; padding: 12px 24px; border-radius: 8px;
            font-weight: bold; cursor: pointer; box-shadow: 0 4px 15px rgba(255,68,68,0.5);
            transition: all 0.3s ease; font-family: 'Segoe UI', sans-serif; font-size: 14px;
        `;

        btn.addEventListener('click', () => {
            btn.disabled = true; btn.textContent = 'Skipping...';
            executeSkip(video);
        });

        document.body.appendChild(btn);
        console.log('Skip button: INJECTED');
    }

    function executeSkip(video) {
        const duration = video.duration;
        const targetTime = duration * 0.91;
        video.currentTime = targetTime;

        if (window.parent?.calcPercentWatched) {
            window.parent.calcPercentWatched(duration * 0.91, targetTime, duration);
            if (window.videoPlayerInstance || window.player) {
                const player = window.videoPlayerInstance || window.player;
                player.secsWatched = duration * 0.91;
            }
        }
        showNotification('✓ Video Skipped!', '#4CAF50');
        setTimeout(resetSkipButton, 2000);
    }

    function resetSkipButton() {
        const btn = document.getElementById('ascend-skip-btn');
        if (btn) {
            btn.disabled = false; btn.textContent = 'Skip Video';
            btn.style.background = 'linear-gradient(135deg, #ff4444, #cc0000)';
        }
    }

    // ===== BLOCK QUIZZES BUTTON =====

    function createBlockButton() {
        const btn = document.createElement('button');
        btn.textContent = 'Quizzes: OFF';
        btn.id = 'ascend-block-btn';
        btn.title = 'Ctrl+Shift+Q';
        btn.style.cssText = `
            position: fixed; top: 80px; right: 20px; z-index: 999999;
            background: linear-gradient(135deg, #ff9800, #f57c00);
            color: white; border: none; padding: 12px 24px; border-radius: 8px;
            font-weight: bold; cursor: pointer; box-shadow: 0 4px 15px rgba(255,152,0,0.5);
            transition: all 0.3s ease; font-family: 'Segoe UI', sans-serif; font-size: 14px;
        `;

        btn.addEventListener('click', () => {
            // Wait for player to be ready
            const checkPlayer = setInterval(() => {
                if (window.videoPlayerInstance || window.player) {
                    clearInterval(checkPlayer);
                    toggleQuizBlocking(btn);
                }
            }, 100);
        });

        document.body.appendChild(btn);
        console.log('Block button: INJECTED');
    }

    function toggleQuizBlocking(button) {
        const player = window.videoPlayerInstance || window.player;

        if (!originalShowQuiz) {
            originalShowQuiz = player.showQuiz.bind(player);
            originalShowPopup = player.showPopup.bind(player);
        }

        isQuizBlockingEnabled = !isQuizBlockingEnabled;

        if (isQuizBlockingEnabled) {
            player.showQuiz = function(quiz) {
                console.log('Quiz blocked:', quiz.question.substring(0, 50) + '...');
                this.quizBlockUntil = this.video.currentTime + 1;
            };
            player.showPopup = function(popup) {
                console.log('Popup blocked:', popup.text.substring(0, 50) + '...');
                this.popupBlockUntil = this.video.currentTime + 1;
                if (this.currentPopup) setTimeout(() => this.hidePopup(), 100);
            };

            player.conceptChecks = [];
            player.popups = [];
            $('#progressMarkers').empty();

            button.textContent = 'Quizzes: ON';
            button.style.background = 'linear-gradient(135deg, #4CAF50, #45a049)';
            showNotification('Quizzes & popups BLOCKED', '#4CAF50');
        } else {
            player.showQuiz = originalShowQuiz;
            player.showPopup = originalShowPopup;
            button.textContent = 'Quizzes: OFF';
            button.style.background = 'linear-gradient(135deg, #ff9800, #f57c00)';
            showNotification('Quizzes & popups ENABLED', '#ff9800');
        }
    }

    // ===== IDLE TIMEOUT BUTTON =====

    function createIdleButton() {
        const btn = document.createElement('button');
        btn.textContent = 'Idle: KILLED';
        btn.id = 'ascend-idle-btn';
        btn.title = 'Click to manually trigger activity';
        btn.style.cssText = `
            position: fixed; top: 140px; right: 20px; z-index: 999999;
            background: linear-gradient(135deg, #000, #333);
            color: #0f0; border: 2px solid #0f0; padding: 10px 20px;
            border-radius: 8px; font-weight: bold; cursor: pointer;
            box-shadow: 0 4px 15px rgba(0,255,0,0.3);
            transition: all 0.3s ease; font-family: monospace; font-size: 13px;
        `;

        btn.addEventListener('click', () => {
            // Trigger activity events
            ['mousemove', 'keydown', 'click'].forEach(ev => {
                document.dispatchEvent(new Event(ev, { bubbles: true }));
            });
            btn.textContent = '✓ Activity Sent!';
            setTimeout(() => btn.textContent = 'Idle: KILLED', 1000);
        });

        document.body.appendChild(btn);
        console.log('Idle button: INJECTED');
    }

    // ===== NOTIFICATION SYSTEM =====

    function showNotification(message, bgColor) {
        const notification = document.createElement('div');
        notification.innerHTML = message;
        notification.style.cssText = `
            position: fixed; top: 200px; right: 20px; z-index: 999999;
            background: ${bgColor}; color: white; padding: 15px 20px;
            border-radius: 10px; font-weight: bold; box-shadow: 0 6px 20px rgba(0,0,0,0.3);
            animation: slideInAscend 0.4s ease; font-family: 'Segoe UI', sans-serif;
        `;

        const style = document.createElement('style');
        style.textContent = `
            @keyframes slideInAscend {
                from { transform: translateX(400px); opacity: 0; }
                to { transform: translateX(0); opacity: 1; }
            }
        `;

        document.head.appendChild(style);
        document.body.appendChild(notification);

        setTimeout(() => {
            notification.style.opacity = '0';
            notification.style.transition = 'opacity 0.5s';
            setTimeout(() => notification.remove(), 500);
        }, 3000);
    }

    // ===== KEYBOARD SHORTCUTS =====

    document.addEventListener('keydown', (e) => {
        if (e.ctrlKey && e.shiftKey) {
            if (e.key === 'S') {
                e.preventDefault();
                const btn = document.getElementById('ascend-skip-btn');
                if (btn && !btn.disabled) btn.click();
            }

            if (e.key === 'Q') {
                e.preventDefault();
                const btn = document.getElementById('ascend-block-btn');
                if (btn) btn.click();
            }

            if (e.key === 'A') {
                e.preventDefault();
                const btn = document.getElementById('ascend-idle-btn');
                if (btn) btn.click();
            }
        }
    });

    // ===== INITIALIZE =====

    injectButtons();
    console.log('everything is up. gl brotha');

})();
