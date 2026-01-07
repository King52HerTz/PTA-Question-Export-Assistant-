```js
// ==UserScript==
// @name         PTA 题目导出助手 (指引版)
// @version      12.4.0
// @description  可以导出pta中的判断、单选、多选题到佛脚刷题小程序进行反复练习 
// @author       Eliauk
// @match        https://pintia.cn/problem-sets/*/exam/problems/*
// @grant        none
// @license      MIT
// ==/UserScript==

(function () {
    'use strict';

    // 防止重复注入
    if (document.getElementById('question-extractor')) return;

    // --- UI 构建 (古风界面 + 指引) ---
    const toolHTML = `
        <div id="question-extractor" style="position: fixed; bottom: 40px; right: 30px; z-index: 10000; display: flex; flex-direction: column; align-items: flex-end; font-family: 'Kaiti', 'STKaiti', 'SimSun', serif;">

            <div id="popup" style="
                background: #f9f7f0;
                background-image: linear-gradient(to bottom, #f9f7f0 0%, #f2efe4 100%);
                border-radius: 6px;
                padding: 16px;
                box-shadow: 0 5px 25px rgba(93, 64, 55, 0.4);
                display: none;
                margin-bottom: 15px;
                width: 280px;
                border: 2px solid #5d4037;
                border-top: 6px solid #b22c2c;
                position: relative;">

                <div style="position:absolute; top:4px; left:4px; right:4px; bottom:4px; border: 1px dashed #dcd8c8; pointer-events:none;"></div>

                <h4 style="margin: 0 0 12px 0; color: #5d4037; font-size: 18px; font-weight: bold; text-align: center; letter-spacing: 2px;">PTA 题目导出工具</h4>

                <div style="background: rgba(0,0,0,0.03); padding: 10px; border-radius: 4px; margin-bottom: 12px; font-size: 13px; color: #5d4037; line-height: 1.6; border-left: 3px solid #b22c2c;">
                    <div style="font-weight:bold; margin-bottom:4px; color:#b22c2c;">📜 使用通牒：</div>
                    1. <b>入题集</b>：进入含有题目列表的页面。<br>
                    2. <b>待加载</b>：稍微上下滚动，确保题目已显示。<br>
                    3. <b>点誊抄</b>：点击下方按钮，生成文档。
                </div>

                <div id="status-msg" style="font-size:14px; color:#666; margin-bottom:15px; text-align: center; min-height: 20px;">
                    笔墨已备，静候指令...
                </div>

                <div style="display: flex; justify-content: center;">
                    <button class="export-btn" id="btn-export-core" style="
                        background: #b22c2c;
                        color: #fff;
                        font-family: 'Kaiti', 'STKaiti', serif;
                        font-size: 16px;
                        font-weight: bold;
                        padding: 8px 30px;
                        border: none;
                        border-radius: 4px;
                        cursor: pointer;
                        box-shadow: 0 3px 6px rgba(178, 44, 44, 0.3);
                        display: flex; align-items: center; gap: 8px;
                        transition: all 0.3s ease;">
                        <span>✍️ 开始誊抄</span>
                    </button>
                </div>
                <div style="font-size: 10px; color: #999; text-align: center; margin-top: 10px; opacity: 0.7;">
                    - 极简 · 去重 · 存真 -
                </div>
            </div>

            <button id="main-btn" title="点击展开导出面板" style="
                width: 60px;
                height: 70px;
                border-radius: 12px;
                background: #b22c2c;
                color: #f9f7f0;
                border: 2px solid #fff;
                cursor: pointer;
                box-shadow: 0 6px 15px rgba(178, 44, 44, 0.5);
                transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: center;
                font-family: 'Kaiti', serif;
                font-weight: bold;
                line-height: 1.2;
                overflow: hidden;">

                <span style="font-size: 20px; margin-bottom: 2px; animation: floatArrow 2s infinite ease-in-out;">📥</span>
                <span style="font-size: 16px; writing-mode: vertical-rl; text-orientation: upright; letter-spacing: 2px;">导出</span>
            </button>
        </div>
        <style>
            /* 按钮悬停效果 */
            .export-btn:hover { background: #d32f2f !important; transform: translateY(-1px); box-shadow: 0 5px 10px rgba(178, 44, 44, 0.5) !important; }
            .export-btn:active { transform: scale(0.96); }

            /* 悬浮球特效 */
            #main-btn:hover {
                transform: translateY(-3px) scale(1.05);
                background: #c62828;
                box-shadow: 0 8px 20px rgba(178, 44, 44, 0.6);
            }
            #main-btn:active { transform: scale(0.95); }

            /* 箭头上下浮动动画 */
            @keyframes floatArrow {
                0%, 100% { transform: translateY(0); }
                50% { transform: translateY(-3px); }
            }
        </style>
    `;

    document.body.insertAdjacentHTML('beforeend', toolHTML);

    const mainBtn = document.getElementById('main-btn');
    const popup = document.getElementById('popup');
    const statusMsg = document.getElementById('status-msg');

    mainBtn.onclick = () => popup.style.display = popup.style.display === 'none' ? 'block' : 'none';

    document.getElementById('btn-export-core').onclick = function() {
        runExportProcess();
    };

    // --- 核心逻辑  ---

    function runExportProcess() {
        // 古风状态文案
        statusMsg.textContent = "正在研磨墨汁，解析试卷...";
        statusMsg.style.color = "#8d6e63";

        // 筛选题目容器
        const divs = document.querySelectorAll('div[id]');
        const candidates = [];

        divs.forEach(div => {
            const id = div.id;
            // 过滤非题目ID
            if (id.length > 8 && !isNaN(parseInt(id.substring(0, 3))) && !id.includes('app')) {
                if ((div.querySelector('div.rendered-markdown') || div.innerText.includes('分)'))) {
                    candidates.push(div);
                }
            }
        });

        const uniqueDivs = [...new Set(candidates)];

        if (uniqueDivs.length === 0) {
            statusMsg.innerHTML = "⚠️ <span style='color:#d32f2f'>未寻得有效考题。</span><br><span style='font-size:12px'>请确保页面已加载题目列表。</span>";
            return;
        }

        let outputText = "";
        let count = 0;

        uniqueDivs.forEach((div) => {
            const textFragment = parseProblemStructure(div);
            if (textFragment) {
                outputText += textFragment + '\n';
                count++;
            }
        });

        if (count > 0) {
            statusMsg.innerHTML = `✅ 誊抄完毕。<br>共录得 <b>${count}</b> 道试题。`;
            statusMsg.style.color = "#388e3c";

            const dateStr = new Date().toISOString().slice(0,10).replace(/-/g,"");
            // 文件名保持格式
            downloadTxt(outputText, `PTA_题集_x${count}_${dateStr}.txt`);
        } else {
            statusMsg.textContent = "⚠️ 导出失败，请重试。";
        }
    }

    // --- 文本清洗引擎 (Logic Unchanged) ---
    function getRichText(element) {
        if (!element) return "";
        const clone = element.cloneNode(true);
        const garbageSelectors = ['.katex-mathml', '.MathJax_Preview', '.MathJax_Display', 'script', 'style', 'annotation[encoding="application/x-tex"]'];
        const mathElements = clone.querySelectorAll('.katex');
        mathElements.forEach(mathEl => {
            const annotation = mathEl.querySelector('annotation');
            let latex = annotation ? annotation.textContent : mathEl.innerText;
            mathEl.innerHTML = "";
            mathEl.textContent = " " + latex + " ";
        });
        clone.querySelectorAll(garbageSelectors.join(',')).forEach(el => el.remove());
        clone.querySelectorAll('img').forEach(img => {
            const src = img.src || img.getAttribute('data-src');
            if (src) {
                const textNode = document.createTextNode(` [Image: ${src}] `);
                if (img.parentNode) img.parentNode.replaceChild(textNode, img);
            }
        });
        clone.querySelectorAll('p, br, div, li').forEach(el => el.appendChild(document.createTextNode(' ')));
        let text = clone.innerText;
        text = text.replace(/\^\{(\w+)\}/g, '^$1').replace(/_\{(\w+)\}/g, '_$1').replace(/\\([a-zA-Z]+)/g, '$1');
        text = text.replace(/\b([a-zA-Z0-9])\s*\1\s*\1\b/g, "$1").replace(/\b([a-zA-Z0-9])\s*\1\b/g, "$1");
        text = text.replace(/(O\([^\)]+\))\s*\1\s*\1/gi, "$1").replace(/(O\([^\)]+\))\s*\1/gi, "$1");
        text = text.replace(/O\(N(\d+)\)\s*O\(N\^(\d+)\)/gi, (m, p1, p2) => p1 === p2 ? `O(N^${p2})` : m)
                   .replace(/O\(N\^(\d+)\)\s*O\(N(\d+)\)/gi, (m, p1, p2) => p1 === p2 ? `O(N^${p1})` : m);
        text = text.replace(/(N\^\d+)\s*\1/gi, "$1");
        return text.replace(/[\r\n]+/g, ' ').replace(/\s+/g, ' ').trim();
    }

    function parseProblemStructure(el) {
        let cleanTitle = "";
        const markdownDiv = el.querySelector('.rendered-markdown');
        if (markdownDiv) cleanTitle = getRichText(markdownDiv);
        else {
            let tempClone = el.cloneNode(true);
            tempClone.querySelectorAll('label').forEach(l => l.remove());
            cleanTitle = getRichText(tempClone);
        }
        const codeBlock = el.querySelector('pre code');
        if (codeBlock) cleanTitle += ` [Code Snippet: ${codeBlock.innerText.replace(/[\r\n]+/g, '  ')}]`;

        const labels = el.querySelectorAll('label');
        let optionStr = "", selectedAnswers = [], isTrueFalse = false;
        if (labels.length > 0) {
            let tfCount = 0;
            labels.forEach(lbl => { if (['T','F','TRUE','FALSE','正确','错误','对','错'].includes(lbl.innerText.trim().toUpperCase())) tfCount++; });
            if (tfCount === labels.length) isTrueFalse = true;
        } else {
            if (el.querySelectorAll('input[type="radio"]').length === 2) isTrueFalse = true;
        }

        if (labels.length > 0) {
            labels.forEach((lbl, i) => {
                const letter = String.fromCharCode(65 + i);
                const input = lbl.querySelector('input');
                let optionTxt = getRichText(lbl).trim().replace(/^[A-Za-z][\.\s]\s*/, "");
                if (input && input.checked) selectedAnswers.push(isTrueFalse ? (['T','TRUE','正确','对'].some(k => optionTxt.toUpperCase().includes(k)) ? 'T' : 'F') : letter);
                if (!isTrueFalse) optionStr += `${letter}.${optionTxt}  `;
            });
        } else if (isTrueFalse) {
            const radios = el.querySelectorAll('input[type="radio"]');
            if (radios[0]?.checked) selectedAnswers.push('T');
            else if (radios[1]?.checked) selectedAnswers.push('F');
        }

        let answerPart = selectedAnswers.join('');
        return isTrueFalse ? `${cleanTitle}(${answerPart})` : `${cleanTitle}(${answerPart})\n${optionStr.trim()}`;
    }

    function downloadTxt(text, filename) {
        const blob = new Blob([text], { type: 'text/plain;charset=utf-8' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = filename;
        a.click();
        URL.revokeObjectURL(url);
    }

    // 只有开发者才看得到的低调小彩蛋
    console.log('%c PTA 导出工具 %c 界面重绘完成 ', 'background: #5d4037; color: #fff', 'background: #f4f1e8; color: #5d4037');
})();
```

