// ==UserScript==
// @name         PTA 题目导出助手
// @version      12.8.0
// @description  导出PTA题目为文本(.txt)格式，自动分类单选/多选/判断，适配佛脚刷题软件导入格式。
// @author       Eliauk
// @match        https://pintia.cn/problem-sets/*/exam/problems/*
// @grant        none
// @license      MIT
// ==/UserScript==

(function () {
    'use strict';

    // 防止重复注入
    if (document.getElementById('question-extractor')) return;

    // --- UI 构建 (保持原版古风界面) ---
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
                width: 290px;
                border: 2px solid #5d4037;
                border-top: 6px solid #b22c2c;
                position: relative;">

                <div style="position:absolute; top:4px; left:4px; right:4px; bottom:4px; border: 1px dashed #dcd8c8; pointer-events:none;"></div>

                <h4 style="margin: 0 0 12px 0; color: #5d4037; font-size: 18px; font-weight: bold; text-align: center; letter-spacing: 2px;">PTA 题目导出工具</h4>

                <div style="background: rgba(0,0,0,0.03); padding: 10px; border-radius: 4px; margin-bottom: 12px; font-size: 13px; color: #5d4037; line-height: 1.6; border-left: 3px solid #b22c2c;">
                    <div style="font-weight:bold; margin-bottom:4px; color:#b22c2c;">📜 使用通牒：</div>
                    1. <b>入题集</b>：进入页面，确保题目加载。<br>
                    2. <b>解封印</b>：若题目已锁，点<b style="color:#795548">解除封印</b>，<b style="color:#d32f2f">修正答案</b>。<br>
                    3. <b>点誊抄</b>：导出 TXT 文本。<br>
                    <span style="color:#888; font-size:12px;">* 适配通用导入格式</span>
                </div>

                <div id="status-msg" style="font-size:14px; color:#666; margin-bottom:15px; text-align: center; min-height: 20px;">
                    笔墨已备，静候指令...
                </div>

                <div style="display: flex; justify-content: space-between; gap: 8px;">
                     <button class="export-btn" id="btn-unlock-core" style="
                        background: #795548;
                        color: #fff;
                        font-family: 'Kaiti', 'STKaiti', serif;
                        font-size: 15px;
                        font-weight: bold;
                        padding: 8px 0;
                        border: none;
                        border-radius: 4px;
                        cursor: pointer;
                        box-shadow: 0 3px 6px rgba(121, 85, 72, 0.3);
                        display: flex; align-items: center; justify-content: center;
                        flex: 1;
                        transition: all 0.3s ease;">
                        <span>🔓 解除封印</span>
                    </button>

                    <button class="export-btn" id="btn-export-core" style="
                        background: #b22c2c;
                        color: #fff;
                        font-family: 'Kaiti', 'STKaiti', serif;
                        font-size: 15px;
                        font-weight: bold;
                        padding: 8px 0;
                        border: none;
                        border-radius: 4px;
                        cursor: pointer;
                        box-shadow: 0 3px 6px rgba(178, 44, 44, 0.3);
                        display: flex; align-items: center; justify-content: center;
                        flex: 1;
                        transition: all 0.3s ease;">
                        <span>📜 导出文本</span>
                    </button>
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
            .export-btn:hover { transform: translateY(-1px); filter: brightness(1.1); }
            .export-btn:active { transform: scale(0.96); }
            #main-btn:hover {
                transform: translateY(-3px) scale(1.05);
                background: #c62828;
                box-shadow: 0 8px 20px rgba(178, 44, 44, 0.6);
            }
            #main-btn:active { transform: scale(0.95); }
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
    document.getElementById('btn-export-core').onclick = runExportProcess;
    document.getElementById('btn-unlock-core').onclick = unlockRestrictions;

    // --- 解锁功能 ---
    function unlockRestrictions() {
        statusMsg.textContent = "正在破除题目禁锢...";
        let count = 0;
        const disabledInputs = document.querySelectorAll('input[disabled], textarea[disabled], button[disabled]');
        disabledInputs.forEach(el => { el.removeAttribute('disabled'); el.classList.remove('disabled'); count++; });
        const lockedDivs = document.querySelectorAll('[style*="pointer-events: none"], .disabled');
        lockedDivs.forEach(el => { el.style.pointerEvents = 'auto'; el.classList.remove('disabled'); });
        document.querySelectorAll('label').forEach(l => l.style.pointerEvents = 'auto');

        statusMsg.innerHTML = count > 0 ? `🔓 封印已除。<br>请<b>手动勾选</b>正确答案。` : `🍵 页面似无枷锁。<br>若无法点击，请刷新重试。`;
        if(count > 0) statusMsg.style.color = "#d84315";
    }

    // --- 导出逻辑 (文本版) ---
    function runExportProcess() {
        statusMsg.textContent = "正在研磨墨汁，整理试卷...";
        statusMsg.style.color = "#8d6e63";

        const divs = document.querySelectorAll('div[id]');
        const candidates = [];
        divs.forEach(div => {
            const id = div.id;
            if (id.length > 8 && !isNaN(parseInt(id.substring(0, 3))) && !id.includes('app')) {
                if ((div.querySelector('div.rendered-markdown') || div.innerText.includes('分)'))) {
                    candidates.push(div);
                }
            }
        });
        const uniqueDivs = [...new Set(candidates)];

        if (uniqueDivs.length === 0) {
            statusMsg.innerHTML = "⚠️ 未寻得有效考题。<br><span style='font-size:12px'>请确保页面已加载题目列表。</span>";
            return;
        }

        const problems = { '单选题': [], '多选题': [], '判断题': [] };

        uniqueDivs.forEach((div) => {
            const prob = parseProblemStructure(div);
            if (prob) {
                if (prob.type === '判断题') problems['判断题'].push(prob);
                else if (prob.type === '多选题') problems['多选题'].push(prob);
                else problems['单选题'].push(prob);
            }
        });

        // 生成文本内容
        let txtContent = "";

        // 格式化函数
        const appendSection = (title, items) => {
            if (items.length === 0) return;
            // 题型标识
            txtContent += `${title}\n`;

            items.forEach((item, index) => {
                // 去除题目中的换行符，确保一行
                const cleanTitle = item.title.replace(/[\r\n]+/g, ' ').trim();

                if (item.type === '判断题') {
                    // 判断题格式：1. 题目内容(正确)
                    // 注意：答案在 parseProblemStructure 中已经被处理为 "正确" 或 "错误"
                    txtContent += `${index + 1}. ${cleanTitle} (${item.answer})\n`;
                } else {
                    // 选择题格式：
                    // 1. 题目内容
                    // A. 选项...
                    // 答案：A
                    txtContent += `${index + 1}. ${cleanTitle}\n`;
                    if (item.optionsStr) {
                        // 选项可以有空格分隔，也可以换行，这里使用空格分隔以紧凑，或者原样输出
                        txtContent += `${item.optionsStr}\n`;
                    }
                    txtContent += `答案：${item.answer}\n`;
                }
                // 题目间空一行
                txtContent += `\n`;
            });
            // 题型间分隔
            txtContent += `\n`;
        };

        appendSection('单选题', problems['单选题']);
        appendSection('多选题', problems['多选题']);
        appendSection('判断题', problems['判断题']);

        const totalCount = problems['单选题'].length + problems['多选题'].length + problems['判断题'].length;

        if (totalCount > 0) {
            statusMsg.innerHTML = `✅ 誊抄完毕。<br>共录得 <b>${totalCount}</b> 道试题。`;
            statusMsg.style.color = "#388e3c";
            const dateStr = new Date().toISOString().slice(0,10).replace(/-/g,"");
            downloadTxt(txtContent, `PTA_题集_x${totalCount}_${dateStr}.txt`);
        } else {
            statusMsg.textContent = "⚠️ 导出失败，请重试。";
        }
    }

    // --- 核心修复：RichText 解析 ---
    function getRichText(element) {
        if (!element) return "";
        const clone = element.cloneNode(true);

        const visualGarbage = [
            '.katex-html', '.MathJax_Preview', '.mjx-chtml', '.mjx-container',
            'style', 'script', 'noscript'
        ];
        clone.querySelectorAll(visualGarbage.join(',')).forEach(el => el.remove());

        const mathSelectors = ['.katex', '.MathJax', 'script[type^="math/tex"]'];
        clone.querySelectorAll(mathSelectors.join(',')).forEach(mathEl => {
            let latex = "";
            const annotation = mathEl.querySelector('annotation');
            if (annotation) latex = annotation.textContent;
            else if (mathEl.querySelector('.katex-mathml')) latex = mathEl.querySelector('.katex-mathml').textContent;
            else latex = mathEl.textContent;

            if (latex) {
                latex = latex.replace(/\\over/g, '/')
                             .replace(/\\frac\{(.+?)\}\{(.+?)\}/g, '($1/$2)')
                             .replace(/\\(left|right)/g, '')
                             .replace(/\\([a-zA-Z]+)/g, '$1')
                             .trim();
                const textNode = document.createTextNode(" " + latex + " ");
                mathEl.replaceWith(textNode);
            } else {
                mathEl.remove();
            }
        });

        clone.querySelectorAll('img').forEach(img => {
            if (img.src || img.getAttribute('data-src')) {
                img.parentNode.replaceChild(document.createTextNode(` [图片] `), img);
            }
        });

        clone.querySelectorAll('.sr-only').forEach(el => el.remove());
        // 补充空格而不是换行，确保文本流
        clone.querySelectorAll('p, br, div, li').forEach(el => el.appendChild(document.createTextNode(' ')));

        let text = clone.innerText || clone.textContent;

        text = text.replace(/\^\{(.+?)\}/g, '^($1)')
                   .replace(/\^(\w+)/g, '^$1')
                   .replace(/_\{(\w+)\}/g, '_$1')
                   .replace(/[\r\n]+/g, ' ') // 核心：强制所有换行为空格
                   .replace(/\s+/g, ' ')
                   .trim();

        text = text.replace(/\\cdots/g, '...').replace(/\\dots/g, '...');

        return text;
    }

    // --- 题目结构解析 (文本适配版) ---
    function parseProblemStructure(el) {
        let cleanTitle = "";
        const markdownDiv = el.querySelector('.rendered-markdown');

        if (markdownDiv) {
            cleanTitle = getRichText(markdownDiv);
        } else {
            let tempClone = el.cloneNode(true);
            tempClone.querySelectorAll('label').forEach(l => l.remove());
            cleanTitle = getRichText(tempClone);
        }
        // 移除题目开头的编号 (如 1-1)，并强制移除所有换行
        cleanTitle = cleanTitle.replace(/^\d+-\d+\s*/, '').replace(/[\r\n]+/g, ' ').trim();

        const codeBlock = el.querySelector('pre code');
        if (codeBlock) cleanTitle += ` [包含代码片段]`;

        const labels = el.querySelectorAll('label');
        let optionArr = [], selectedAnswers = [], problemType = '单选题';

        let isTF = false;
        let isMulti = false;
        if (labels.length > 0) {
            let tfCount = 0;
            labels.forEach(lbl => { if (['T','F','TRUE','FALSE','正确','错误','对','错'].includes(lbl.innerText.trim().toUpperCase())) tfCount++; });
            if (tfCount === labels.length) isTF = true;
            if (el.querySelector('input[type="checkbox"]')) isMulti = true;
        } else {
            if (el.querySelectorAll('input[type="radio"]').length === 2) isTF = true;
        }

        if (isTF) problemType = '判断题';
        else if (isMulti) problemType = '多选题';
        else problemType = '单选题';

        if (labels.length > 0) {
            labels.forEach((lbl, i) => {
                const letter = String.fromCharCode(65 + i);
                const input = lbl.querySelector('input');
                // 选项提取：确保单行
                let optionTxt = getRichText(lbl).trim().replace(/^[A-Za-z][\.\s]\s*/, "").replace(/[\r\n]+/g, ' ');

                if (input && input.checked) {
                    if (isTF) {
                        // 判断题：转换为 "正确" 或 "错误"
                        const isTrue = ['T','TRUE','正确','对'].some(k => optionTxt.toUpperCase().includes(k));
                        selectedAnswers.push(isTrue ? '正确' : '错误');
                    } else {
                        selectedAnswers.push(letter);
                    }
                }

                if (!isTF) {
                    // 选项格式：A.选项内容 B.选项内容 (这里用换行符方便后续处理，或者空格)
                    // 文本格式要求选项间有空格或换行。这里使用空格分隔更符合"单选多选格式要求"示例
                    optionArr.push(`${letter}.${optionTxt}`);
                }
            });
        }

        let answerStr = selectedAnswers.join('');
        if (!answerStr) answerStr = "无";

        return {
            type: problemType,
            title: cleanTitle,
            answer: answerStr,
            // 选项用空格拼接，符合紧凑格式
            optionsStr: isTF ? "" : optionArr.join('  ')
        };
    }

    // --- 导出 TXT ---
    function downloadTxt(content, filename) {
        const blob = new Blob([content], { type: 'text/plain;charset=utf-8' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = filename;
        a.click();
        URL.revokeObjectURL(url);
    }

    console.log(
    '%c PTA 题目收割机 %c 谈笑间，题库已灰飞烟灭 ',
    'background: #5d4037; color: #fff; padding: 4px; border-radius: 4px 0 0 4px;',
    'background: #333; color: #f1c40f; font-weight: bold; padding: 4px; border-radius: 0 4px 4px 0;'
);
})();
