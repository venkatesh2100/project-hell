# Instahyre Auto-Apply (Resume-Aware)

A browser-console script that auto-applies to jobs on [Instahyre](https://instahyre.com) (Opportunities page). Built for **Venkatesh S**.

## What it does

- Works through ALL "View job" cards one after another.
- Applies to every job — no relevance filter, nothing skipped on purpose.
- If an apply modal shows extra questions, fills them from your resume (text fields, dropdowns, Yes/No radios) before clicking Apply. No phone number is ever entered.
- Doesn't quit on the first quiet tick: waits a few idle rounds, scrolls for more jobs, then finishes.

## Usage

1. Log in to Instahyre in Chrome and open the Opportunities page.
2. Open DevTools (`F12` or `Ctrl+Shift+I`) → **Console**.
3. Paste the entire script below and press `Enter`.
4. Watch the console logs. Stop any time with `stopInstahyreAutoApply()`.

> Chrome may ask you to type `allow pasting` before it accepts a pasted script in the console.
> Edit the `CONFIG` object marked TODO (CTC, notice period) before running.

## Script

```js
/**
 * Instahyre Auto-Apply (resume-aware) — Venkatesh S
 * Run inside Chrome DevTools Console on Instahyre (Opportunities page).
 *
 * Changes vs. the original:
 *  - Works through ALL "View job" cards one after another (original only opened the first)
 *  - Applies to every job — no relevance filter, nothing skipped on purpose
 *  - If an apply modal shows extra questions, fills them from your resume
 *    (text fields, dropdowns, Yes/No radios) before clicking Apply. No phone number is ever entered.
 *  - Doesn't quit on the first quiet tick: waits a few idle rounds, scrolls for more jobs, then finishes
 *
 * Stop any time with:  stopInstahyreAutoApply()
 */
(() => {
  // ───────────────────────── CONFIG (edit me) ─────────────────────────
  const CONFIG = {
    yearsExperience: '1',
    currentLocation: 'Hyderabad, India',
    noticePeriod: 'Immediate',   // TODO: confirm
    currentCTC: '',              // TODO: fill, otherwise left blank
    expectedCTC: '',             // TODO: fill, otherwise left blank
    email: 'mail.venkatesh2100@gmail.com',
    linkedin: 'https://linkedin.com/in/venkatesh2100',
    github: 'https://github.com/venkatesh2100',
    portfolio: 'https://venkatesh2100.me',
    tickMs: 2500,
    maxIdleTicks: 4,             // quiet ticks (no buttons found, no new jobs) before finishing
  };

  const SUMMARY =
    'Software Engineer with ~1 year of production experience in backend, full-stack and GenAI. ' +
    'At Find Me LLC I build AI/LLM features with Python, FastAPI, LangChain, AWS Bedrock, RAG/Hybrid RAG and agentic workflows ' +
    '(Langfuse for tracing), on PostgreSQL/Redis backends with React frontends, Docker, Terraform, AWS (EC2/ECS) and GitHub Actions CI/CD. ' +
    'I also built and run Growman (growman.live), a live plant-commerce platform with AI plant identification and a React Native app with 300+ downloads.';

  const FIELD_RULES = [
    [/notice/i, () => CONFIG.noticePeriod],
    [/current.*(ctc|salary|compensation)|(ctc|salary).*current/i, () => CONFIG.currentCTC],
    [/expected.*(ctc|salary|compensation)|(ctc|salary|compensation).*expect/i, () => CONFIG.expectedCTC],
    [/(years?|yrs?).*(experience|exp)|experience.*(years?|yrs?)/i, () => CONFIG.yearsExperience],
    [/linkedin/i, () => CONFIG.linkedin],
    [/github/i, () => CONFIG.github],
    [/portfolio|website|personal site/i, () => CONFIG.portfolio],
    [/e-?mail/i, () => CONFIG.email],
    [/location|city|based/i, () => CONFIG.currentLocation],
    [/why|cover letter|about (you|yourself)|describe|tell us|introduce|summary|relevant/i, () => SUMMARY],
  ];

  // Tech NOT on the resume: experience questions about these get "No"
  const NOT_ON_RESUME = ['java ', 'java,', 'java?', 'kubernetes', 'k8s', '.net', 'c#', 'php', 'angular', 'golang', 'ruby', 'scala', 'kotlin', 'swift', 'flutter', 'spark', 'hadoop', 'azure', 'gcp', 'django', 'flask', 'mongodb', 'solidity', 'web3'];
  const PREFERRED_OPTIONS = ['hyderabad', 'india', 'remote', 'immediate', 'python'];

  // ───────────────────────── STATE ─────────────────────────
  let applyCount = 0;
  let idleTicks = 0;
  let busy = false;
  const lower = (s) => (s || '').toLowerCase();
  const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
  const visible = (el) => !!(el && (el.offsetWidth || el.offsetHeight || el.getClientRects().length));

  console.log('%c🚀 Instahyre auto-apply started (resume-aware, no skipping)', 'color:green;font-weight:bold;');

  // ───────────────────────── MODAL FORM FILLING ─────────────────────────
  const getModal = () =>
    [...document.querySelectorAll('.modal.in, .modal.show, .modal[style*="display: block"], [role="dialog"]')].find(visible) || null;

  // Angular listens for input/change events, so dispatch both
  const setValue = (el, value) => {
    const proto = el.tagName === 'TEXTAREA' ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
    Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, value);
    el.dispatchEvent(new Event('input', { bubbles: true }));
    el.dispatchEvent(new Event('change', { bubbles: true }));
  };

  const fillModal = async () => {
    const modal = getModal();
    if (!modal) return;

    // Text inputs / textareas
    modal.querySelectorAll('textarea, input[type="text"], input[type="number"], input[type="url"], input[type="email"], input:not([type])')
      .forEach((el) => {
        if (!visible(el) || el.disabled || el.readOnly || el.value.trim()) return;
        const label = el.getAttribute('aria-label') || el.placeholder || el.name ||
          el.closest('label')?.innerText || el.parentElement?.innerText || el.parentElement?.parentElement?.innerText || '';
        const rule = FIELD_RULES.find(([re]) => re.test(label));
        const value = rule ? rule[1]() : '';
        if (value) setValue(el, value);
      });

    // Native selects
    modal.querySelectorAll('select:not([disabled])').forEach((sel) => {
      const opts = [...sel.options].filter((o) => o.value && !o.disabled && !/^\s*(select|choose|--)/i.test(o.text));
      if (!opts.length || (sel.selectedIndex > 0 && sel.value)) return;
      const pick = opts.find((o) => PREFERRED_OPTIONS.some((k) => lower(o.text).includes(k))) || opts[0];
      Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, 'value').set.call(sel, pick.value);
      sel.dispatchEvent(new Event('change', { bubbles: true }));
      console.log(`%c🔽 Selected: ${pick.text.trim()}`, 'color:#FFB300;');
    });

    // react-select style dropdowns (opened with mousedown)
    for (const control of [...modal.querySelectorAll('[class*="select__control"]')]) {
      if (control.querySelector('[class*="single-value"], [class*="multi-value"]')) continue;
      control.dispatchEvent(new MouseEvent('mousedown', { bubbles: true, cancelable: true, view: window, button: 0 }));
      await sleep(450);
      const options = [...document.querySelectorAll('[class*="select__option"]')].filter((o) => !/no options|loading/i.test(o.innerText));
      const pick = options.find((o) => PREFERRED_OPTIONS.some((k) => lower(o.innerText).includes(k))) || options[0];
      if (pick) { pick.click(); console.log(`%c🔽 Selected: ${pick.innerText.trim()}`, 'color:#FFB300;'); }
      await sleep(500);
    }

    // Radio groups: Yes by default, No for tech that isn't on the resume
    const groups = {};
    modal.querySelectorAll('input[type="radio"]').forEach((r) => { (groups[r.name || r.getAttribute('ng-model') || 'g'] ||= []).push(r); });
    Object.values(groups).forEach((radios) => {
      if (radios.some((r) => r.checked)) return;
      const labelOf = (r) => lower(r.closest('label')?.innerText || r.parentElement?.innerText || r.value);
      const question = lower(radios[0].closest('fieldset, .form-group, .question, div')?.parentElement?.innerText || '');
      const yes = radios.find((r) => /^\s*yes\b/.test(labelOf(r)));
      const no = radios.find((r) => /^\s*no\b/.test(labelOf(r)));
      const missing = NOT_ON_RESUME.some((k) => question.includes(k)) && /experience|worked|proficien|hands-on/.test(question);
      (missing && no ? no : yes || radios[0]).click();
    });
  };

  // ───────────────────────── JOB NAVIGATION ─────────────────────────
  // Finds the next un-clicked "View job" button
  const nextViewButton = () =>
    [...document.querySelectorAll('button.btn-interested, button.btn-success, button')]
      .find((b) => visible(b) && /view job/i.test(b.textContent) && !b.dataset.autoClicked);

  const openNextJob = () => {
    const btn = nextViewButton();
    if (!btn) return false;
    btn.dataset.autoClicked = '1';
    btn.scrollIntoView({ behavior: 'smooth', block: 'center' });
    btn.click();
    console.log('%c🔍 Opened next job via "View job"', 'color:green;font-weight:bold;');
    return true;
  };

  const finish = () => {
    clearInterval(intervalId);
    console.log('%c🎉 All jobs processed!', 'color:limegreen;font-size:15px;font-weight:bold;');
    console.log(`%c🧾 Total applications sent: ${applyCount}`, 'color:#2196F3;font-weight:bold;');
  };

  // ───────────────────────── MAIN LOOP ─────────────────────────
  const tick = async () => {
    if (busy) return;
    busy = true;
    try {
      const bulkBtn = document.querySelector('button[ng-click="applyBulk()"]');
      const normalBtn =
        document.querySelector('.apply[ng-click*="submitChoice"] .new-btn') ||
        document.querySelector('button[ng-click*="submitChoice"]') ||
        document.querySelector('button[ng-click*="apply"]');

      const target = [bulkBtn, normalBtn].find((b) => b && visible(b) && !b.disabled);
      if (target) {
        idleTicks = 0;
        await fillModal();           // answer any extra questions first
        await sleep(500);
        // Re-query in case Angular re-rendered the button after filling
        const btn = [bulkBtn, normalBtn].find((b) => b && document.contains(b) && !b.disabled) || target;
        btn.click();
        applyCount++;
        const kind = btn === bulkBtn ? 'Bulk' : 'Normal';
        console.log(`%c✅ [${applyCount}] ${kind} Apply clicked at ${new Date().toLocaleTimeString()}`, 'color:#0080FF;font-weight:bold;');
        return;
      }

      // No apply button: open the next job, or scroll for more
      if (openNextJob()) { idleTicks = 0; return; }

      idleTicks++;
      console.log(`%c📜 Nothing to click — scrolling for more (${idleTicks}/${CONFIG.maxIdleTicks})`, 'color:#FF9800;');
      window.scrollTo(0, document.body.scrollHeight);
      if (idleTicks >= CONFIG.maxIdleTicks) finish();
    } finally {
      busy = false;
    }
  };

  const intervalId = setInterval(tick, CONFIG.tickMs);
  window.stopInstahyreAutoApply = () => { clearInterval(intervalId); console.log('🛑 Stopped.'); };
  tick();
})();
```

## Notes

- The script runs entirely in your browser — no data is sent anywhere except to Instahyre when it submits an application.
- It depends on Instahyre's current DOM structure and selectors, so it may break when the site changes.
- Recruiters see these applications as coming from you. Review the notes and answers before running it in bulk, and use automation at your own discretion under Instahyre's terms of service.
