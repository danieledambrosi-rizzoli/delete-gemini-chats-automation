# delete-gemini-chats-automation
## this is an useful javascript link that deletes all the gemini chats automatically

### Usage
1. Copy the code
2. Open "https://gemini.google.com/app"
3. Open the side menu that shows all the chats
4. Right-click and select "Inspect"
5. Open the console and paste the code inside it
6. Execute the code and wait for it to finish

### Code
```
(async () => {
  'use strict';

  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const txt   = el => (el?.textContent || '').trim();
  const css   = el => { try { return getComputedStyle(el); } catch { return {}; } };
  const hasLayout = el => {
    if (!el) return false;
    if (el.offsetParent !== null) return true;
    const r = el.getBoundingClientRect();
    return r.width > 0 && r.height > 0;
  };
  const isShown = el =>
    hasLayout(el) && css(el).visibility !== 'hidden' && css(el).display !== 'none';
  const forceShow = el => {
    if (!el) return;
    const s = css(el);
    try {
      if (s.display === 'none')       el.style.display = '';
      if (s.visibility === 'hidden')  el.style.visibility = 'visible';
      if (s.opacity === '0')          el.style.opacity = '1';
      if (s.pointerEvents === 'none') el.style.pointerEvents = 'auto';
    } catch {}
  };

  async function waitFor(fn, timeout = 5000, interval = 100) {
    const end = Date.now() + timeout;
    while (Date.now() < end) {
      try { const v = fn(); if (v) return v; } catch {}
      await sleep(interval);
    }
    return null;
  }

  function fire(el, type) {
    if (!el) return;
    const r  = el.getBoundingClientRect();
    const ev = {
      bubbles: true, cancelable: true, view: window,
      clientX: r.left + r.width / 2,
      clientY: r.top + r.height / 2,
      button: 0, buttons: type.endsWith('down') ? 1 : 0,
    };
    try {
      if (type.startsWith('pointer')) {
        el.dispatchEvent(new PointerEvent(type, { ...ev, pointerId: 1, pointerType: 'mouse', isPrimary: true }));
      } else {
        el.dispatchEvent(new MouseEvent(type, ev));
      }
    } catch { el.dispatchEvent(new MouseEvent(type, ev)); }
  }
  const hover = el => ['pointerover','pointerenter','mouseover','mouseenter','mousemove'].forEach(t => fire(el, t));
  async function click(el, { force = false } = {}) {
    if (!el) return false;
    if (force) forceShow(el);
    hover(el);
    ['pointerdown','mousedown','pointerup','mouseup','click'].forEach(t => fire(el, t));
    return true;
  }

  // ---------------- chat rows ----------------
  // Primary: data-test-id="conversation", menu button is actions-menu-button inside.
  // Fallback: any anchor whose path is /app/<id>.
  function findChatRows() {
    const out = [];
    const seen = new Set();

    for (const el of document.querySelectorAll('[data-test-id="conversation"]')) {
      if (!isShown(el)) continue;
      const anchor = el.querySelector('a[href]');
      let id;
      if (anchor) {
        try { id = new URL(anchor.getAttribute('href'), location.origin).pathname; } catch {}
      }
      if (!id) id = 'conv::' + txt(el).slice(0, 60);
      if (seen.has(id)) continue;
      seen.add(id);
      out.push({
        id, row: el, anchor,
        menuButton: el.querySelector('[data-test-id="actions-menu-button"]'),
      });
    }
    if (out.length) return out;

    for (const a of document.querySelectorAll('a[href]')) {
      try {
        const path = new URL(a.getAttribute('href'), location.origin).pathname;
        if (!/^\/app\/[^\/?#]+$/.test(path)) continue;
        if (!isShown(a)) continue;
        if (seen.has(path)) continue;
        seen.add(path);
        const row = a.closest('[data-test-id="conversation"], li, [role="listitem"]') || a.parentElement || a;
        out.push({
          id: path, row, anchor: a,
          menuButton: row.querySelector('[data-test-id="actions-menu-button"]'),
        });
      } catch {}
    }
    return out;
  }

  // ---------------- menu button ----------------
  function findMenuButton(target) {
    if (target.menuButton && isShown(target.menuButton)) return target.menuButton;

    const scope = target.row;
    // 1. exact data-test-id
    const byTestId = scope.querySelector('[data-test-id="actions-menu-button"]');
    if (byTestId) return byTestId;
    // 2. aria-label
    for (const b of scope.querySelectorAll('button, [role="button"]')) {
      const s = `${b.getAttribute('aria-label') || ''} ${b.getAttribute('data-test-id') || ''}`.toLowerCase();
      if (/menu|option|more|altro|opzioni|azioni|overflow|dots|three/.test(s)) return b;
    }
    // 3. very last button in the row
    const all = [...scope.querySelectorAll('button, [role="button"]')];
    return all[all.length - 1] || null;
  }

  // ---------------- delete item ----------------
  function findDeleteItem() {
    // Prefer explicit test id anywhere (that's what Gemini uses)
    const byTestId = [...document.querySelectorAll('[data-test-id="delete-button"]')].find(isShown);
    if (byTestId) return byTestId;

    // Then any open menu / dialog
    const scopes = [
      ...document.querySelectorAll('.cdk-overlay-pane, mat-menu, [role="menu"], .mat-mdc-menu-panel'),
      document,
    ];
    for (const scope of scopes) {
      for (const el of scope.querySelectorAll('[role="menuitem"], button, [data-test-id]')) {
        if (!isShown(el)) continue;
        const s = `${el.getAttribute('data-test-id') || ''} ${txt(el)}`.toLowerCase();
        if (/(delete|elimina|rimuovi|remove)/.test(s) && !/(cancel|annulla|restore|ripristina)/.test(s)) return el;
      }
    }
    return null;
  }

  // ---------------- confirm ----------------
  function findConfirmButton() {
    const scopes = [
      ...document.querySelectorAll('mat-dialog-container, .cdk-overlay-pane, [role="dialog"], [role="alertdialog"]'),
      document,
    ];
    for (const scope of scopes) {
      for (const b of scope.querySelectorAll('button, [role="button"]')) {
        if (!isShown(b)) continue;
        const t = txt(b).toLowerCase();
        if (/^(elimina|delete|conferma|confirm|sì|yes|ok)/.test(t) &&
            !/(annulla|cancel|no|chiudi|close)/.test(t)) return b;
      }
    }
    return null;
  }

  async function dismiss() {
    document.body.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', keyCode: 27, bubbles: true }));
    await sleep(100);
    document.body.dispatchEvent(new KeyboardEvent('keyup',   { key: 'Escape', keyCode: 27, bubbles: true }));
    await sleep(150);
  }

  // ---------------- diagnostics ----------------
  function dumpDiagnostics(stage, target) {
    console.group(`=== gemini-cleaner diagnostics (${stage}) ===`);
    console.log('location:', location.href);
    console.log('body.classes:', [...document.body.classList]);
    console.log('conversations in DOM:', document.querySelectorAll('[data-test-id="conversation"]').length);
    console.log('actions-menu-buttons in DOM:', document.querySelectorAll('[data-test-id="actions-menu-button"]').length);
    if (target) {
      console.log('target.id:', target.id);
      console.log('target.row outerHTML (first 800 chars):');
      console.log(target.row.outerHTML.slice(0, 800));
      const btns = [...target.row.querySelectorAll('button, [role="button"]')].map(b => ({
        testId: b.getAttribute('data-test-id'),
        ariaLabel: b.getAttribute('aria-label'),
        text: txt(b).slice(0, 40),
        visible: isShown(b),
      }));
      console.log('buttons inside target.row:', btns);
    }
    console.groupEnd();
  }

  // ================= MAIN =================
  console.log('[gemini-cleaner] starting');

  const initial = await waitFor(() => { const r = findChatRows(); return r.length ? r : null; }, 15000);
  if (!initial) {
    console.error('[gemini-cleaner] No chat rows detected within 15s.');
    dumpDiagnostics('no-rows');
    return;
  }
  console.log(`[gemini-cleaner] found ${initial.length} chat row(s); starting`);

  const MAX = 2000, MAX_FAIL = 5;
  let deleted = 0, failStreak = 0;

  for (let i = 0; i < MAX; i++) {
    const rows = findChatRows();
    if (rows.length === 0) { console.log('[gemini-cleaner] no chats left'); break; }

    const target = rows[0];
    const id     = target.id;

    try {
      target.row.scrollIntoView({ block: 'center', behavior: 'instant' });
      await sleep(150);
      hover(target.row);
      let p = target.row.parentElement;
      for (let k = 0; k < 3 && p; k++, p = p.parentElement) hover(p);
      await sleep(250);

      const menuBtn = await waitFor(() => findMenuButton(target), 3000);
      if (!menuBtn) { dumpDiagnostics('menu-button-missing', target); throw new Error('menu button not found'); }

      await click(menuBtn, { force: true });
      await sleep(300);

      const delItem = await waitFor(() => findDeleteItem(), 3000);
      if (!delItem) { dumpDiagnostics('delete-item-missing', target); throw new Error('delete menu item not found'); }

      await click(delItem);
      await sleep(400);

      const confirmBtn = await waitFor(() => findConfirmButton(), 3000);
      if (!confirmBtn) { dumpDiagnostics('confirm-missing', target); throw new Error('confirm button not found'); }

      await click(confirmBtn);

      const gone = await waitFor(() => !findChatRows().some(r => r.id === id), 8000);
      if (!gone) throw new Error('row still present after confirm');

      deleted++;
      failStreak = 0;
      console.log(`[gemini-cleaner] ✓ deleted ${deleted}: ${id.slice(0, 80)}`);
      await sleep(300);
    } catch (e) {
      failStreak++;
      console.warn(`[gemini-cleaner] ✗ iteration ${i}: ${e.message} (streak ${failStreak}/${MAX_FAIL})`);
      await dismiss();
      await sleep(400);
      if (failStreak >= MAX_FAIL) {
        console.error('[gemini-cleaner] too many consecutive failures — aborting');
        dumpDiagnostics('aborting', target);
        break;
      }
    }
  }

  console.log(`[gemini-cleaner] done. deleted=${deleted}`);
})();
```
