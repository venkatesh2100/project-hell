# Uplers Auto-Apply (Resume-Aware)

A browser-console script that auto-applies to jobs on the [Uplers](https://uplers.com) jobs page. Built for **Venkatesh S**.

## What it does

- Applies to EVERY job — no relevance filter, nothing is skipped on purpose.
- Fills modal text fields from your profile (no phone number is ever entered).
- Answers Yes/No questions: YES by default, NO only for tech that isn't on your resume.
- Selects dropdowns (native `<select>` and react-select style) automatically.
- Only gives up on a job if the form still refuses to submit after 3 tries.
- Scrolls to load more jobs, up to the configured attempt limit.

## Usage

1. Log in to Uplers in Chrome and open the jobs page.
2. Open DevTools (`F12` or `Ctrl+Shift+I`) → **Console**.
3. Paste the entire script below and press `Enter`.
4. Watch the console logs.

> Chrome may ask you to type `allow pasting` before it accepts a pasted script in the console.
> Edit the `CONFIG` object marked TODO (CTC, notice period) before running — if CTC fields are empty, the job is skipped when asked.

## Script

```js
/**
 * Uplers Auto-Apply (resume-aware) — Venkatesh S
 * Run inside Chrome DevTools Console on the Uplers jobs page.
 *
 * What changed vs. the original:
 *  - Applies to EVERY job (no relevance filter, nothing is skipped on purpose)
 *  - Fills modal text fields from your profile (no phone number is ever entered)
 *  - Yes/No questions: YES by default, NO only for tech that isn't on your resume
 *  - Selects dropdowns (native <select> and react-select style) automatically
 *  - Only gives up on a job if the form still refuses to submit after 3 tries
 *
 * !! Fill in the CONFIG values marked TODO before running. !!
 */
(() => {
  // ───────────────────────── CONFIG (edit me) ─────────────────────────
  const CONFIG = {
    yearsExperience: '1',          // Aug 2025 – Sep 2026 full-time + independent projects
    currentLocation: 'Hyderabad, India',
    noticePeriod: 'Immediate',     // TODO: confirm
    currentCTC: '',                // TODO: fill (leave '' => job is skipped if asked)
    expectedCTC: '',               // TODO: fill (leave '' => job is skipped if asked)
    email: 'mail.venkatesh2100@gmail.com',
    linkedin: 'https://linkedin.com/in/venkatesh2100',
    github: 'https://github.com/venkatesh2100',
    portfolio: 'https://venkatesh2100.me',
    actionDelay: 2500,
    maxScrollAttempts: 4,
  };

  const SUMMARY =
    'Software Engineer with ~1 year of production experience building backend, full-stack and GenAI applications. ' +
    'At Find Me LLC I build AI/LLM features with Python, FastAPI, LangChain, AWS Bedrock, RAG/Hybrid RAG and agentic workflows, ' +
    'with Langfuse for tracing, PostgreSQL/Redis backends, React frontends, Docker, Terraform, AWS (EC2/ECS), GitHub Actions CI/CD, ' +
    'Prometheus/Grafana and Playwright testing. I also independently built and run Growman (growman.live), a live plant-commerce ' +
    'platform with AI plant identification, an AI care assistant (Dootha AI) and a React Native Android app with 300+ downloads.';

  // ───────────────────────── RESUME PROFILE ─────────────────────────
  const MATCH_KEYWORDS = [
    'python', 'fastapi', 'langchain', 'langfuse', 'rag', 'retrieval', 'llm', 'genai', 'gen ai', 'generative ai',
    'agentic', 'agent', 'bedrock', 'prompt engineering', 'ai engineer', 'machine learning', 'nlp',
    'backend', 'back-end', 'back end', 'full stack', 'full-stack', 'fullstack', 'software engineer', 'software developer',
    'react', 'next.js', 'nextjs', 'react native', 'node', 'express', 'typescript', 'javascript',
    'postgres', 'redis', 'rest api', 'api', 'aws', 'docker', 'terraform', 'ci/cd', 'github actions',
    'prometheus', 'grafana', 'playwright', 'microservices',
  ];

  // Tech NOT on the resume — if a yes/no question asks about experience in these, answer NO / skip
  const NOT_ON_RESUME = [
    'java ', 'java,', 'java?', 'kubernetes', 'k8s', '.net', 'c#', 'php', 'angular', 'golang', 'go lang', 'ruby',
    'scala', 'kotlin', 'swift', 'spark', 'hadoop', 'azure', 'gcp ', 'salesforce', 'sap',
  ];

  const YES_PATTERNS = [
    'confirm', 'agree', 'comfortable', 'willing', 'available', 'ok with', 'okay with', 'remote', 'overlap',
    'timezone', 'time zone', 'hours', 'contract', 'full-time', 'full time', 'commit', 'authorized', 'eligible',
    ...MATCH_KEYWORDS,
  ];

  // text-field label -> answer ('' means "unknown; skip job")
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
    [/why|cover letter|about (you|yourself)|describe|tell us|introduce|summary|experience with|relevant/i, () => SUMMARY],
  ];

  // ───────────────────────── STATE ─────────────────────────
  let appliedCount = 0;
  let skippedCount = 0;
  let processing = false;
  let scrollAttempts = 0;
  const skipped = new Set();            // card keys we decided not to apply to
  const attempts = new Map();           // card key -> submit attempts
  const decisions = new Map();          // card key -> { apply: bool, reason }
  const SEL = {
    cards: '.jobListSection .list .jobCardContainer',
    active: '.jobListSection .list .jobCardContainer.active',
    applied: '.oppHeadActionTag',
    submit: '.bottomActions input.primaryBtn.cta[type="submit"]',
    applyNow: '.actionBtns .primary button.applyBtn',
    close: 'button.modalCloseBtn',
    // best-guess selectors for the job detail pane; falls back to the card text
    details: '.jobDetailSection, .jobDetails, .jobDescription, .oppDetails, .opportunityDetails',
  };

  console.log('%c🚀 Uplers Auto-Apply (resume-aware) started', 'color:#00E676;font-weight:bold;font-size:14px;');

  // ───────────────────────── HELPERS ─────────────────────────
  const isVisible = (el) => !!(el && (el.offsetWidth || el.offsetHeight || el.getClientRects().length));
  const lower = (s) => (s || '').toLowerCase();

  const click = (el) => {
    if (el) el.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true, view: window }));
  };

  const setValue = (el, value) => {
    const proto = el.tagName === 'TEXTAREA' ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
    Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, value);
    el.dispatchEvent(new Event('input', { bubbles: true }));
    el.dispatchEvent(new Event('change', { bubbles: true }));
  };

  const cardKey = (card) =>
    (card?.querySelector('.jobTitle')?.textContent || card?.textContent || '').trim().replace(/\s+/g, ' ').slice(0, 120);

  const allCards = () => Array.from(document.querySelectorAll(SEL.cards));

  // ───────────────────────── DROPDOWNS ─────────────────────────
  const PREFERRED_OPTIONS = ['hyderabad', 'india', 'remote', 'immediate', 'python'];

  const handleDropdowns = async () => {
    const root = document.querySelector(SEL.submit)?.closest('form') || document;
    let filled = 0;

    // react-select style (opens on mousedown, not click)
    for (const control of [...root.querySelectorAll('[class*="select__control"]')]) {
      if (control.querySelector('[class*="single-value"], [class*="multi-value"]')) continue;
      control.dispatchEvent(new MouseEvent('mousedown', { bubbles: true, cancelable: true, view: window, button: 0 }));
      await new Promise((r) => setTimeout(r, 450));
      const menu = document.querySelector('[class*="select__menu"]');
      const options = menu ? [...menu.querySelectorAll('[class*="select__option"]')].filter((o) => !/no options|loading/i.test(o.innerText)) : [];
      const pick = options.find((o) => PREFERRED_OPTIONS.some((k) => lower(o.innerText).includes(k))) || options[0];
      if (pick) { pick.click(); filled++; console.log(`%c🔽 Selected: ${pick.innerText.trim()}`, 'color:#FFEB3B;'); }
      await new Promise((r) => setTimeout(r, 500));
    }

    // native <select>
    root.querySelectorAll('select:not([disabled])').forEach((sel) => {
      if (sel.value) return;
      const opt = [...sel.options].find((o) => o.value && !o.disabled);
      if (!opt) return;
      Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, 'value').set.call(sel, opt.value);
      sel.dispatchEvent(new Event('change', { bubbles: true }));
      filled++;
      console.log(`%c🔽 Selected: ${opt.text.trim()}`, 'color:#FFEB3B;');
    });
    return filled;
  };

  // ───────────────────────── MODAL FILLING ─────────────────────────
  // Returns list of questions we could not answer truthfully.
  const fillModal = () => {
    const root = document.querySelector(SEL.submit)?.closest('form') || document;
    const unresolved = [];

    // Yes/No radios
    root.querySelectorAll('.radioFlex').forEach((container) => {
      const question = lower(container.parentElement?.innerText || container.innerText);
      const labels = Array.from(container.querySelectorAll('label.radioInput'));
      const yes = labels.find((l) => lower(l.textContent).includes('yes'));
      const no = labels.find((l) => lower(l.textContent).trim() === 'no');
      const pick = (label) => {
        const input = label.querySelector('input[type="radio"]') || label;
        if (!input.checked) click(input);
      };

      if (NOT_ON_RESUME.some((k) => question.includes(k)) && /experience|worked|proficien|hands-on/.test(question) && no) return pick(no);
      if (yes) return pick(yes);
      if (labels.length) pick(labels[0]);
    });

    // Text inputs / textareas
    root.querySelectorAll('textarea, input[type="text"], input[type="number"], input[type="url"], input[type="email"], input[type="tel"], input:not([type])')
      .forEach((el) => {
        if (!isVisible(el) || el.disabled || el.readOnly || el.value.trim()) return;
        const label =
          el.getAttribute('aria-label') || el.placeholder || el.name ||
          el.closest('label')?.innerText || el.parentElement?.innerText || el.parentElement?.parentElement?.innerText || '';
        const rule = FIELD_RULES.find(([re]) => re.test(label));
        const value = rule ? rule[1]() : '';
        if (value) setValue(el, value);
      });

    return unresolved;
  };

  const closeModal = () => click(document.querySelector(SEL.close));

  // ───────────────────────── NAVIGATION ─────────────────────────
  const selectCard = (card) => {
    card.scrollIntoView({ behavior: 'smooth', block: 'center' });
    click(card.querySelector('.jobTitle') || card.querySelector('.jobCard') || card);
  };

  const advanceToNextCard = (activeCard) => {
    const cards = allCards();
    const idx = cards.indexOf(activeCard);
    const next = cards.slice(idx + 1).find((c) => !c.querySelector(SEL.applied) && !skipped.has(cardKey(c)));

    if (next) {
      scrollAttempts = 0;
      selectCard(next);
      console.log(`%c🔄 Next job: ${cardKey(next).slice(0, 60)}`, 'color:#AB47BC;');
      setTimeout(() => { processing = false; }, 1500);
    } else {
      scrollAndLoadMore();
    }
  };

  const scrollAndLoadMore = () => {
    scrollAttempts++;
    console.log(`%c📜 Loading more jobs (attempt ${scrollAttempts}/${CONFIG.maxScrollAttempts})...`, 'color:#FF9800;font-weight:bold;');

    const before = allCards();
    const list = document.querySelector('.jobListSection .list') || document.querySelector('.jobListSection');
    before[before.length - 1]?.scrollIntoView({ behavior: 'auto', block: 'end' });
    if (list) {
      list.scrollTop = list.scrollHeight;
      list.dispatchEvent(new WheelEvent('wheel', { bubbles: true, cancelable: true, deltaY: 1000, deltaMode: 0 }));
    }
    window.scrollTo(0, document.body.scrollHeight);

    setTimeout(() => {
      const after = allCards();
      const active = document.querySelector(SEL.active);
      if (after.length > before.length) {
        console.log(`%c✨ Loaded ${after.length - before.length} new jobs`, 'color:#00E676;font-weight:bold;');
        scrollAttempts = 0;
        advanceToNextCard(active || after[before.length - 1]);
      } else if (scrollAttempts < CONFIG.maxScrollAttempts) {
        processing = false;
      } else {
        finish();
      }
    }, 3500);
  };

  const finish = () => {
    clearInterval(intervalId);
    console.log('%c🎉 Done — no more jobs to process.', 'color:#00E676;font-size:15px;font-weight:bold;');
    console.log(`%c🧾 Applied: ${appliedCount} | Skipped: ${skippedCount}`, 'color:#fff;background:#1E88E5;padding:6px 12px;border-radius:4px;');
    console.table([...decisions].map(([job, d]) => ({ job, applied: d.apply && !d.skippedLater, reason: d.reason })));
  };

  const skipJob = (card, reason) => {
    const key = cardKey(card);
    skipped.add(key);
    skippedCount++;
    decisions.set(key, { apply: false, reason });
    console.log(`%c⏭️ Skipped "${key.slice(0, 50)}" — ${reason}`, 'color:#FFA726;');
  };

  // ───────────────────────── MAIN LOOP ─────────────────────────
  const step = () => {
    if (processing) return;
    processing = true;

    let card = document.querySelector(SEL.active);

    // 1. Nothing selected yet → pick the first job not applied/skipped
    if (!card) {
      card = allCards().find((c) => !c.querySelector(SEL.applied) && !skipped.has(cardKey(c)));
      if (card) {
        selectCard(card);
        setTimeout(() => { processing = false; }, 1200);
      } else {
        scrollAndLoadMore();
      }
      return;
    }

    const key = cardKey(card);

    // 2. Already applied or previously skipped → move on
    if (card.querySelector(SEL.applied) || skipped.has(key)) return advanceToNextCard(card);

    // 3. Application modal is open → fill everything, submit, verify it actually closed
    const submitBtn = document.querySelector(SEL.submit);
    if (submitBtn && isVisible(submitBtn)) {
      const tries = (attempts.get(key) || 0) + 1;
      attempts.set(key, tries);
      if (tries > 3) {
        closeModal();
        skipJob(card, 'form refused to submit after 3 tries');
        return setTimeout(() => advanceToNextCard(card), 800);
      }
      fillModal();
      handleDropdowns().then(() => setTimeout(() => {
        click(submitBtn);
        setTimeout(() => {
          if (isVisible(document.querySelector(SEL.submit))) { processing = false; return; } // blocked by validation; retry next tick
          appliedCount++;
          console.log(`%c✅ [${appliedCount}] Applied: ${key.slice(0, 60)}`, 'color:#00E676;font-weight:bold;');
          closeModal();
          advanceToNextCard(document.querySelector(SEL.active) || card);
        }, CONFIG.actionDelay);
      }, 800));
      return;
    }

    // 4. Job detail pane → click Apply Now
    const applyBtn = document.querySelector(SEL.applyNow);
    if (applyBtn && isVisible(applyBtn) && !applyBtn.disabled) {
      if (!decisions.has(key)) {
        decisions.set(key, { apply: true, reason: 'applied' });
        console.log(`%c🎯 Applying: ${key.slice(0, 60)}`, 'color:#29B6F6;');
      }
      click(applyBtn);
      return setTimeout(() => { processing = false; }, 1200);
    }

    // 5. Nothing actionable → try loading more
    scrollAndLoadMore();
  };

  const intervalId = setInterval(step, CONFIG.actionDelay);
  window.stopUpllersAutoApply = () => { clearInterval(intervalId); console.log('🛑 Stopped.'); };
})();
```

## Notes

- The script runs entirely in your browser — no data is sent anywhere except to Uplers when it submits an application.
- It depends on Uplers' current DOM structure and selectors, so it may break when the site changes.
- Recruiters see these applications as coming from you. Review the notes and answers before running it in bulk, and use automation at your own discretion under Uplers' terms of service.
