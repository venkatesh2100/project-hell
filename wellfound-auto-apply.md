# Wellfound Auto-Apply (India)

A browser-console script that auto-applies to jobs on [Wellfound](https://wellfound.com) using a resume-aware application note. Built for **Venkatesh S**.

## What it does

- Opens every job card on the listing page and fills the application note.
- Applies to every job — no skill-match, title or experience filtering.
- Selects **India** as the current location when the application form asks for one (best effort — missing the field does not skip the job).
- Picks one of two notes per job — an AI/GenAI-focused note or a backend/full-stack note — based on keywords in the job description.
- Answers custom radio questions from the question text plus the resume profile (Yes / Intermediate for skills on the resume, No / Beginner for ones that are not).
- Fills the remaining react-select and native dropdowns with preferred options (Remote, Python).
- Handles the relocation questionnaire when the Apply button stays disabled.
- Skips a job only when Apply is still disabled after all answers are filled.
- Scrolls to load more jobs, up to the configured limit, then prints a summary table.

## Usage

1. Log in to Wellfound in Chrome and open a jobs listing page (e.g. the roles you are targeting).
2. Open DevTools (`F12` or `Ctrl+Shift+I`) → **Console**.
3. Paste the entire script below and press `Enter`.
4. Watch the console logs, and review the final `console.table` summary of applied/skipped jobs.

> Chrome may ask you to type `allow pasting` before it accepts a pasted script in the console.

## Configuration

Edit the `CONFIG` object at the top of the script:

| Key | Default | Meaning |
| --- | --- | --- |
| `location` | `India` | Value typed into the location dropdown when the form asks for a current location. |
| `maxScrolls` | `10` | How many scroll attempts to make when no new jobs are found. |

The application notes (`NOTE_AI`, `NOTE_FULLSTACK`) and the resume keyword lists (`AI_KEYWORDS`, `ON_RESUME`, `NOT_ON_RESUME`) are plain constants — update them if your resume changes.

## Script

```js
/**
 * Wellfound Auto-Apply (India) — Venkatesh S
 * Run inside Chrome DevTools Console on a Wellfound jobs listing page.
 *
 * Changes vs. the original:
 *  - Applies to EVERY job: no skill-match, title or experience filtering
 *  - Selects India as the current location when the form asks for one
 *  - Application note rewritten from your resume (AI-focused or backend/full-stack variant per job)
 *  - Custom radio questions are answered from the question text + your resume
 *    (Yes / Intermediate for your skills, No / Beginner for ones not on your resume)
 *  - Relocation questionnaire handler kept from the original script
 *  - Only skips if Apply is still disabled after answering, since it can't be clicked
 */
(async () => {
  const delay = (ms) => new Promise((res) => setTimeout(res, ms));

  // ───────────────────────── CONFIG ─────────────────────────
  const CONFIG = {
    location: 'India',
    maxScrolls: 10,
  };

  const LINKS = `🔗 Links:
Portfolio: https://venkatesh2100.me
GitHub: https://github.com/venkatesh2100
LinkedIn: https://linkedin.com/in/venkatesh2100
Live project: https://growman.live`;

  const NOTE_AI = `Hi there,

I'm Venkatesh S, a Software Engineer focused on building production GenAI and backend systems. I'm currently working on AI/LLM features at Find Me LLC (remote, New York) using Python, FastAPI, LangChain, AWS Bedrock, RAG / Hybrid RAG and agentic workflows, with Langfuse for tracing and evaluation.

• Built RAG and Hybrid RAG pipelines combining semantic retrieval with traditional search
• Built agentic workflows for tool integration and multi-step automation
• FastAPI + PostgreSQL + Redis services, Dockerized, deployed on AWS (EC2/ECS) with Terraform and GitHub Actions CI/CD
• Production observability with Prometheus/Grafana and Playwright end-to-end testing
• Solo-built Growman (growman.live), a live plant-commerce platform with AI plant identification and an AI care assistant (Dootha AI), plus a React Native Android app with 300+ downloads

${LINKS}

Happy to share more detail on any of this. Looking forward to hearing from you!

Best,
Venkatesh S`;

  const NOTE_FULLSTACK = `Hi there,

I'm Venkatesh S, a Software Engineer with experience across backend, full-stack and cloud. At Find Me LLC (remote, New York) I build FastAPI/Python services backed by PostgreSQL and Redis, React frontends, and the CI/CD and infrastructure around them.

• Python/FastAPI, Node.js/Express REST APIs; React, Next.js, React Native, Tailwind
• Docker, AWS (EC2/ECS), Terraform, GitHub Actions; Prometheus/Grafana monitoring
• Unit, API and Playwright end-to-end testing for reliable releases
• Independently designed, shipped and maintain Growman (growman.live): auth, orders, Razorpay payments, invoicing, notifications, AI plant identification and a React Native app with 300+ downloads

${LINKS}

Hope this fits the role. Looking forward to hearing from you!

Best,
Venkatesh S`;

  // ───────────────────────── RESUME PROFILE ─────────────────────────
  const AI_KEYWORDS = ['llm', 'genai', 'gen ai', 'generative', 'rag', 'langchain', 'agentic', 'agent', 'bedrock', 'prompt', 'ai engineer', 'machine learning', 'nlp'];
  const NOT_ON_RESUME = ['java ', 'java,', 'java?', 'kubernetes', 'k8s', '.net', 'c#', 'php', 'angular', 'golang', 'ruby', 'scala', 'kotlin', 'swift', 'flutter', 'spark', 'hadoop', 'azure', 'gcp', 'django', 'flask', 'mongodb', 'solidity', 'web3'];
  const ON_RESUME = ['python', 'fastapi', 'langchain', 'rag', 'llm', 'bedrock', 'react', 'next', 'node', 'express', 'typescript', 'javascript', 'postgres', 'redis', 'docker', 'aws', 'terraform', 'github actions', 'ci/cd', 'rest', 'api', 'playwright', 'prometheus', 'grafana', 'tailwind', 'html', 'css', 'sql', 'linux', 'git'];

  // ───────────────────────── STATE ─────────────────────────
  let appliedCount = 0;
  let skippedCount = 0;
  let scrollCount = 0;
  const processedButtons = new Set();
  const log = [];

  console.log('%c🚀 Starting Wellfound India auto-apply (all jobs)...', 'color: green; font-weight: bold;');

  // ───────────────────────── HELPERS ─────────────────────────
  const lower = (s) => (s || '').toLowerCase();

  const getRoot = () => document.querySelector('[role="dialog"], [data-test*="SlideIn"]') || document;

  const waitForElement = async (selector, timeout = 5000) => {
    const start = Date.now();
    while (Date.now() - start < timeout) {
      const el = document.querySelector(selector);
      if (el) return el;
      await delay(100);
    }
    return null;
  };

  const closeModal = async () => {
    const closeBtn = await waitForElement('button[data-test="closeButton"]', 3000);
    if (closeBtn) closeBtn.click();
    await delay(600);
  };

  // Best-effort job text: the open slide-in/dialog, else the card around the Learn More button
  const getJobText = (applyBtn, learnMoreBtn) => {
    const dialog = applyBtn.closest('[role="dialog"], [data-test*="SlideIn"]');
    if (dialog) return lower(dialog.innerText);
    let el = learnMoreBtn;
    for (let i = 0; i < 4 && el.parentElement; i++) el = el.parentElement;
    return lower(el.innerText);
  };

  // ───────────────────────── LOCATION ─────────────────────────
  const isLocationControl = (control) => {
    const input = control.querySelector('input');
    const context = [
      input?.getAttribute('placeholder'),
      input?.getAttribute('aria-label'),
      input?.getAttribute('name'),
      input?.id,
      control.getAttribute('aria-label'),
      control.parentElement?.innerText,
      control.parentElement?.parentElement?.innerText,
    ].join(' ').toLowerCase();

    return /location|country|based in|current location|where.*live|where.*located/.test(context);
  };

  const selectIndiaReact = async (control) => {
    const input = control.querySelector('input');
    if (!input) return false;

    const selected = control.querySelector('[class*="single-value"]');
    if (selected?.innerText.trim().toLowerCase() === CONFIG.location.toLowerCase()) {
      console.log('%c✅ India already selected', 'color: green');
      return true;
    }

    control.scrollIntoView({ block: 'center', behavior: 'smooth' });
    await delay(300);
    input.focus();

    const setter = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set;
    setter.call(input, CONFIG.location);
    input.dispatchEvent(new Event('input', { bubbles: true }));
    await delay(1000);

    const menu = document.querySelector('[class*="select__menu"]');
    if (!menu) {
      console.log('%c❌ India dropdown suggestions not found', 'color: red');
      return false;
    }

    const options = [...menu.querySelectorAll('[class*="select__option"]')];
    const india = options.find(
      (option) =>
        option.innerText.trim().toLowerCase() === CONFIG.location.toLowerCase() &&
        option.getAttribute('aria-disabled') !== 'true'
    );

    if (!india) {
      console.log('%c❌ Exact India option not found', 'color: red');
      return false;
    }

    india.click();
    await delay(800);

    const verified = control.querySelector('[class*="single-value"]');
    if (verified?.innerText.trim().toLowerCase() === CONFIG.location.toLowerCase()) {
      console.log('%c✅ Location selected: India', 'color: green');
      return true;
    }

    return false;
  };

  const handleLocationDropdown = async () => {
    const root = getRoot();

    // First look for a React Select location field.
    const controls = [...root.querySelectorAll('[class*="select__control"]')];
    const control = controls.find(isLocationControl);
    if (control) return await selectIndiaReact(control);

    // Fallback: native select with a location label.
    const nativeSelects = [...root.querySelectorAll('select:not([disabled])')];
    for (const sel of nativeSelects) {
      const context = [
        sel.name,
        sel.id,
        sel.getAttribute('aria-label'),
        sel.parentElement?.innerText,
        sel.parentElement?.parentElement?.innerText,
      ].join(' ').toLowerCase();

      if (!/location|country|based in/.test(context)) continue;

      const option = [...sel.options].find((o) => o.text.trim().toLowerCase() === CONFIG.location.toLowerCase());
      if (!option) continue;

      const setter = Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, 'value').set;
      setter.call(sel, option.value);
      sel.dispatchEvent(new Event('change', { bubbles: true }));
      await delay(700);

      if (sel.value === option.value) {
        console.log('%c✅ Native location selected: India', 'color: green');
        return true;
      }
    }

    console.log('%c⚠️ Could not identify a location dropdown', 'color: gray');
    return false;
  };

  // ───────────────────────── CUSTOM QUESTIONS ─────────────────────────
  const handleCustomQuestions = () => {
    document.querySelectorAll('[data-test^="RadioGroup-customQuestionAnswers"]').forEach((group) => {
      const radios = [...group.querySelectorAll('input[type="radio"]')];
      const labelOf = (r) => lower(r.closest('label')?.innerText || r.parentElement?.innerText || r.value);
      const question = lower(group.closest('[data-test*="customQuestion"]')?.innerText || group.parentElement?.innerText || '');
      const find = (re) => radios.find((r) => re.test(labelOf(r)));

      const knownMissing = NOT_ON_RESUME.some((k) => question.includes(k));
      const knownHave = ON_RESUME.some((k) => question.includes(k));

      // Yes/No
      const yes = find(/^\s*yes\b/);
      const no = find(/^\s*no\b/);
      if (yes && no) {
        (knownMissing ? no : yes).click();
        return;
      }

      // Skill-level scales (beginner / intermediate / advanced / expert ...)
      const beginner = find(/beginner|basic|novice|entry/);
      const intermediate = find(/intermediate|moderate|proficient|working/);
      if (beginner || intermediate) {
        if (knownMissing && beginner) beginner.click();
        else (intermediate || beginner).click();
        return;
      }

      // Avoid guessing answers to unknown questions.
      console.log('%c⚠️ Unrecognized custom question:', 'color: gray', question.slice(0, 120));
    });
  };

  // ───────────────────────── DROPDOWNS ─────────────────────────
  // Preferred option text when a list is shown (first match wins), else the first option is used
  const PREFERRED_OPTIONS = ['remote', 'python'];

  const openSelect = async (control) => {
    control.scrollIntoView({ block: 'center' });
    // react-select opens on mousedown (a plain .click() does nothing)
    control.dispatchEvent(new MouseEvent('mousedown', { bubbles: true, cancelable: true, view: window, button: 0 }));
    await delay(450);
    let menu = document.querySelector('.select__menu');
    if (!menu) {
      // fallback: keyboard open
      const input = control.querySelector('input');
      if (input) {
        input.focus();
        input.dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowDown', keyCode: 40, which: 40, bubbles: true }));
        await delay(450);
        menu = document.querySelector('.select__menu');
      }
    }
    return menu;
  };

  const handleDropdowns = async () => {
    const root = getRoot();
    let filled = 0;

    for (const control of [...root.querySelectorAll('[class*="select__control"]')]) {
      // already has a value? leave it
      if (control.querySelector('[class*="single-value"], [class*="multi-value"]')) continue;

      // never overwrite the location selection
      if (isLocationControl(control)) continue;

      const menu = await openSelect(control);
      if (!menu) {
        console.log('%c⚠️ Could not open a dropdown', 'color: gray');
        continue;
      }

      const options = [...menu.querySelectorAll('[class*="select__option"]')]
        .filter((o) => !/no options|loading/i.test(o.innerText) && o.getAttribute('aria-disabled') !== 'true');
      const pick = options.find((o) => PREFERRED_OPTIONS.some((k) => lower(o.innerText).includes(k))) || options[0];

      if (pick) {
        const label = pick.innerText.trim();
        pick.click();
        filled++;
        console.log(`%c🔽 Selected dropdown option: ${label}`, 'color: orange');
      } else {
        control.querySelector('input')?.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', keyCode: 27, bubbles: true }));
      }
      await delay(600);
    }

    // native <select> elements (skip the location one, handled above)
    root.querySelectorAll('select:not([disabled])').forEach((sel) => {
      if (sel.value) return;
      const opt = [...sel.options].find((o) => o.value && !o.disabled);
      if (!opt) return;
      Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, 'value').set.call(sel, opt.value);
      sel.dispatchEvent(new Event('change', { bubbles: true }));
      filled++;
      console.log(`%c🔽 Selected native option: ${opt.text.trim()}`, 'color: orange');
    });

    return filled;
  };

  // Relocation questionnaire from the original script (fixed: the id contains dots, so it needs [id="..."])
  const handleRelocationQuestion = async () => {
    try {
      const firstRadio = document.querySelector('input[name="qualification.location.action"]');
      if (firstRadio) {
        firstRadio.click();
        console.log('%c📍 Selected relocation option', 'color: orange');
        await delay(500);
      }
      return (await handleDropdowns()) > 0;
    } catch (err) {
      console.log('%c❌ Error in relocation handler', 'color: red', err);
    }
    return false;
  };

  const skip = async (reason, title) => {
    skippedCount++;
    log.push({ job: title, result: 'skipped', reason });
    console.log(`%c⏭️ Skipped — ${reason}`, 'color: orange');
    await closeModal();
  };

  // ───────────────────────── MAIN BATCH ─────────────────────────
  const processBatch = async () => {
    const buttons = [...document.querySelectorAll('button[data-test="LearnMoreButton"]')].filter((b) => !processedButtons.has(b));
    if (buttons.length === 0) return false;

    for (const learnMoreBtn of buttons) {
      processedButtons.add(learnMoreBtn);

      learnMoreBtn.scrollIntoView({ behavior: 'smooth', block: 'center' });
      await delay(300);
      learnMoreBtn.click();
      console.log(`%c🔍 [${appliedCount + skippedCount + 1}] Opened job modal...`, 'color: blue');

      const applyBtn = await waitForElement('button[data-test="JobDescriptionSlideIn--SubmitButton"]');
      if (!applyBtn) {
        console.log('%c❌ Modal failed to load', 'color: red');
        skippedCount++;
        continue;
      }

      await delay(500); // let the description render
      const jobText = getJobText(applyBtn, learnMoreBtn);
      const title = jobText.split('\n').find((l) => l.trim())?.trim().slice(0, 60) || 'unknown job';

      // 1) Select India first (best effort — keep going if the form has no location field)
      const locationSelected = await handleLocationDropdown();
      if (!locationSelected) console.log('%c⚠️ India not selected; continuing anyway', 'color: gray');
      await delay(700);

      // 2) Fill the application note — Apply is usually disabled until it has text
      const aiHits = AI_KEYWORDS.filter((k) => jobText.includes(k)).length;
      const note = aiHits >= 2 ? NOTE_AI : NOTE_FULLSTACK;
      const fillNote = () => {
        const textarea = getRoot().querySelector('textarea:not([disabled])');
        if (!textarea) return false;
        const setter = Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype, 'value').set;
        setter.call(textarea, note);
        textarea.dispatchEvent(new Event('input', { bubbles: true }));
        textarea.dispatchEvent(new Event('change', { bubbles: true }));
        textarea.dispatchEvent(new Event('blur', { bubbles: true }));
        return true;
      };
      if (fillNote()) console.log(`%c📝 Filled ${aiHits >= 2 ? 'AI' : 'full-stack'} note`, 'color: purple');
      else console.log('%c⚠️ Application textarea not found', 'color: gray');
      await delay(700);

      // 3) Answer custom questions, then any remaining dropdowns in the modal
      handleCustomQuestions();
      await delay(700);
      await handleDropdowns();
      await delay(500);

      // 4) Wait up to ~3s for Apply to enable (re-query: React may re-render the button)
      const getBtn = () => document.querySelector('button[data-test="JobDescriptionSlideIn--SubmitButton"]');
      const waitEnabled = async (ms = 3000) => {
        const t = Date.now();
        while (Date.now() - t < ms) {
          const btn = getBtn();
          if (btn && !btn.disabled && btn.getAttribute('aria-disabled') !== 'true') return true;
          await delay(200);
        }
        return false;
      };
      let enabled = await waitEnabled();

      // 5) Still disabled? Try the relocation questionnaire, re-fill the note, wait again
      if (!enabled) {
        console.log('%c⚠️ Apply disabled. Retrying location and message...', 'color: orange');
        await handleLocationDropdown();
        fillNote();
        handleCustomQuestions();
        await delay(500);
        enabled = await waitEnabled();
      }

      // 6) Last resort: report exactly what is blocking it, then skip (a disabled button can't be clicked)
      if (!enabled) {
        const blockers = [...getRoot().querySelectorAll('textarea, input[required], select[required], [aria-required="true"]')]
          .filter((el) => !el.disabled && !el.value && el.type !== 'radio')
          .map((el) => el.getAttribute('name') || el.getAttribute('placeholder') || el.getAttribute('aria-label') || el.tagName);
        console.log('%c🔎 Empty fields that may block Apply:', 'color: gray', blockers);
        await skip('apply button still disabled', title);
        continue;
      }

      // 7) Apply
      await delay(500);
      const finalBtn = getBtn();
      if (!finalBtn || finalBtn.disabled || finalBtn.getAttribute('aria-disabled') === 'true') {
        await skip('apply became disabled before clicking', title);
        continue;
      }

      finalBtn.click();
      console.log('%c📨 Apply clicked. Waiting for response...', 'color: blue');
      await delay(3000);
      appliedCount++;
      log.push({ job: title, result: 'applied', reason: '' });
      console.log(`%c✅ Applied: ${title}`, 'color: teal');

      await closeModal();
      await delay(1000);
    }
    return true;
  };

  // Infinite-scroll loop
  while (scrollCount < CONFIG.maxScrolls) {
    const found = await processBatch();
    if (!found) {
      window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });
      console.log('%c📜 Scrolling to load more jobs...', 'color: darkcyan');
      scrollCount++;
      await delay(2000);
    } else {
      scrollCount = 0;
    }
  }

  console.log('%c🎉 All done!', 'color: limegreen; font-size: 16px; font-weight: bold;');
  console.log(`%c📌 Applied: ${appliedCount}  |  Skipped: ${skippedCount}`, 'color: #4CAF50; font-weight: bold;');
  console.table(log);
})();
```

## Notes

- The script runs entirely in your browser — no data is sent anywhere except to Wellfound when it submits an application.
- It depends on Wellfound's current `data-test` attributes and modal structure, so it may break when the site changes.
- Recruiters see these applications as coming from you. Review the notes and answers before running it in bulk, and use automation at your own discretion under Wellfound's terms of service.
