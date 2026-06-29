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
const sleep = ms => new Promise(r => setTimeout(r, ms));
const visible = el => !!el && el.offsetParent !== null;
const text = el => (el?.textContent||'').trim();

function emitClick(el) {
if (!el) return false;
['mousemove','mouseenter','mouseover','mousedown','mouseup','click'].forEach(t =>
el.dispatchEvent(new MouseEvent(t, { bubbles: true, cancelable: true, view: window }))
);
return true;
}

function chatLinks() {
return [...document.querySelectorAll('a[href]')]
.filter(visible)
.filter(a => {
const href = a.getAttribute('href')||'';
if (!href.startsWith('/')) return false;
if (href === '/search') return false;
const txt = (a.getAttribute('aria-label')||'') + ' ' + (a.textContent||'');
return !/search|cerca|settings|help|activity|nuova chat|new chat/i.test(txt);
});
}

function rowFor(a) {
return a.closest('a, li, mat-list-item, .mat-mdc-list-item, [role="listitem"]') || a;
}

function findMenuBtn(row) {
const candidates = [...row.querySelectorAll('button, [role="button"], [aria-label]')].filter(visible);
let btn = candidates.find(el => {
const s = ((el.getAttribute('aria-label')||'') + ' ' + (el.getAttribute('data-test-id')||'') + ' ' + (el.textContent||'')).toLowerCase();
return /menu|more|altro|opzioni|azioni|overflow|dots|three/.test(s);
});
if (btn) return btn;
btn = candidates.find(el => el.querySelector('svg, mat-icon'));
if (btn) return btn;
const rowRect = row.getBoundingClientRect();
return [...document.querySelectorAll('button, [role="button"]')].filter(visible)
.find(el => Math.abs(el.getBoundingClientRect().top - rowRect.top) < 36);
}

function findDeleteMenuItem() {
return [...document.querySelectorAll('[data-test-id="delete-button"], [role="menuitem"], button')]
.filter(visible)
.find(el => /(delete-button|elimina|delete)/i.test(((el.getAttribute('data-test-id')||'')+' '+(el.textContent||'')).toLowerCase()));
}

function findConfirm() {
return [...document.querySelectorAll('button, [role="button"]')].filter(visible)
.find(b => /\b(elimina|delete|conferma)\b/i.test((b.textContent||'').trim()));
}

let deleted = 0;
const max = 2000;
for (let i=0;i<max;i++) {
const links = chatLinks();
if (!links.length) { console.log('No chats found.'); break; }
const a = links[0];
const row = rowFor(a);
try {
row.scrollIntoView({block:'center'});
row.dispatchEvent(new MouseEvent('mouseenter',{bubbles:true}));
await sleep(250);

const menuBtn = findMenuBtn(row);
if (!menuBtn) { console.warn('Menu button not found for row'); break; }

emitClick(menuBtn);
await sleep(500);

const del = findDeleteMenuItem();
if (!del) { console.warn('Delete item not found'); document.body.click(); await sleep(200); break; }

emitClick(del);
await sleep(600);

const confirm = findConfirm();
if (confirm) { emitClick(confirm); deleted++; console.log('Deleted', deleted); await sleep(900); }
else { console.warn('Confirm not found'); break; }
} catch(e) {
console.error('Error on iteration', i, e);
await sleep(1000);
}
} console.log('Finished. Deleted total:', deleted);
})();
```
