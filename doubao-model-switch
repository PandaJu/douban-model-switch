// ==UserScript==
// @name         豆包模型记忆
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  记住豆包上次选择的模型模式，下次访问自动切换
// @match        https://www.doubao.com/chat/*
// @grant        none
// ==/UserScript==

(function () {
  'use strict';

  const STORAGE_KEY = 'doubao_saved_model';
  const MODEL_NAMES = ['快速', '思考', '专家'];
  const BUTTON_XPATH = '/html/body/div/div/div/main/div/div[2]/div[5]/div/div[2]/div/div[2]/div[2]/div[1]/div[1]/div/button';

  function log(...args) {
    console.log('[豆包模型记忆]', ...args);
  }

  /** 通过 XPath 获取按钮 */
  function getButtonByXPath() {
    const result = document.evaluate(
      BUTTON_XPATH, document, null,
      XPathResult.FIRST_ORDERED_NODE_TYPE, null
    );
    return result.singleNodeValue;
  }

  /** 从按钮文本中提取当前模型名 */
  function getCurrentModel(button) {
    const text = button.textContent.trim();
    return MODEL_NAMES.find(name => text.includes(name)) || null;
  }

  /** 保存模型到 localStorage */
  function saveModel(name) {
    localStorage.setItem(STORAGE_KEY, name);
    log('已保存模型:', name);
  }

  /** 读取已保存的模型 */
  function getSavedModel() {
    return localStorage.getItem(STORAGE_KEY);
  }

  /** 点击按钮打开下拉菜单，选择目标模型 */
  /** 模拟真实的鼠标点击（兼容 React 合成事件） */
  function simulateClick(el) {
    const rect = el.getBoundingClientRect();
    const x = rect.left + rect.width / 2;
    const y = rect.top + rect.height / 2;
    const opts = { bubbles: true, cancelable: true, view: window, clientX: x, clientY: y };

    el.dispatchEvent(new MouseEvent('pointerdown', opts));
    el.dispatchEvent(new MouseEvent('mousedown', opts));
    el.dispatchEvent(new MouseEvent('pointerup', opts));
    el.dispatchEvent(new MouseEvent('mouseup', opts));
    el.dispatchEvent(new MouseEvent('click', opts));
  }

  /** 等待下拉菜单中的 menuitem 出现 */
  async function waitForMenuItems(timeout = 3000) {
    const start = Date.now();
    while (Date.now() - start < timeout) {
      const items = document.querySelectorAll('[role="menuitem"]');
      if (items.length > 0) return items;
      await sleep(100);
    }
    return document.querySelectorAll('[role="menuitem"]');
  }

  async function switchModel(button, targetName) {
    log('尝试切换到:', targetName);

    // 模拟真实点击打开下拉菜单
    simulateClick(button);

    // 轮询等待 menuitem 出现
    const menuItems = await waitForMenuItems();
    log('找到菜单项数量:', menuItems.length);
    menuItems.forEach((item, i) => {
      log(`  菜单项[${i}]:`, JSON.stringify(item.textContent.trim()));
    });

    for (const item of menuItems) {
      if (item.textContent.includes(targetName)) {
        simulateClick(item);
        log('已切换到:', targetName);
        return true;
      }
    }

    log('未找到目标模型:', targetName);
    // 点击空白处关闭菜单
    document.body.click();
    return false;
  }

  function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  /** 监听按钮文本变化，当用户手动切换时保存 */
  function observeModelChange(button) {
    const observer = new MutationObserver(() => {
      const current = getCurrentModel(button);
      if (current && current !== getSavedModel()) {
        saveModel(current);
      }
    });

    observer.observe(button, {
      childList: true,
      subtree: true,
      characterData: true
    });

    log('已开始监听模型变化');
    return observer;
  }

  /** 等待页面完全初始化（网络请求基本结束） */
  function waitForPageStable() {
    return new Promise(resolve => {
      let lastChange = Date.now();
      const observer = new MutationObserver(() => { lastChange = Date.now(); });
      observer.observe(document.body, { childList: true, subtree: true });

      const check = setInterval(() => {
        // 距离上次 DOM 变化超过 1.5 秒，认为页面稳定
        if (Date.now() - lastChange > 1500) {
          clearInterval(check);
          observer.disconnect();
          resolve();
        }
      }, 300);

      // 最长等 15 秒
      setTimeout(() => { clearInterval(check); observer.disconnect(); resolve(); }, 15000);
    });
  }

  /** 切换模型并验证，失败则重试 */
  async function switchModelWithRetry(button, targetName, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
      log(`第 ${i + 1} 次尝试切换到: ${targetName}`);
      await switchModel(button, targetName);
      await sleep(800);

      // 重新获取按钮（DOM 可能已更新）
      const btn = getButtonByXPath();
      const current = btn ? getCurrentModel(btn) : null;
      if (current === targetName) {
        log('切换验证成功:', targetName);
        return btn; // 返回最新的按钮引用
      }

      log(`切换被还原，当前为: ${current}，等待后重试...`);
      await sleep(1500);
    }
    log('多次重试后仍未成功');
    return getButtonByXPath();
  }

  /** 主入口 */
  async function main() {
    log('脚本已加载，等待按钮出现...');

    // 轮询等待按钮出现
    let button = await new Promise(resolve => {
      const timer = setInterval(() => {
        const btn = getButtonByXPath();
        if (btn) {
          clearInterval(timer);
          resolve(btn);
        }
      }, 500);
      // 60 秒超时
      setTimeout(() => { clearInterval(timer); resolve(null); }, 60000);
    });

    if (!button) {
      log('超时：未找到模型切换按钮');
      return;
    }

    log('按钮已找到，等待页面完全初始化...');
    await waitForPageStable();
    log('页面已稳定');

    // 页面稳定后重新获取按钮引用（避免旧引用失效）
    button = getButtonByXPath() || button;

    const currentModel = getCurrentModel(button);
    const savedModel = getSavedModel();

    log('当前模型:', currentModel, '| 已保存模型:', savedModel);

    // 如果有保存的模型且与当前不同，自动切换（带重试）
    if (savedModel && currentModel !== savedModel) {
      button = await switchModelWithRetry(button, savedModel);
    } else if (currentModel) {
      // 首次使用，保存当前模型
      saveModel(currentModel);
    }

    // 开始监听用户手动切换
    observeModelChange(button);
  }

  // SPA 页面，等待 DOM 加载完成后启动
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', main);
  } else {
    main();
  }
})();
