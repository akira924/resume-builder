<script lang="ts">
  // Resume builder: 3-part layout

  const DETAILS_STORAGE_KEY = 'resume-builder-details';

  type CareerEntry = {
    companyName: string;
    date: string;
    location: string;
    bulletCount: number;
  };
  type EducationEntry = {
    institution: string;
    degreeMajor: string;
    date: string;
    location: string;
  };

  let fullName = '';
  let location = '';
  let phone = '';
  let email = '';
  let experienceLevel = '';

  let career: CareerEntry[] = [
    { companyName: '', date: '', location: '', bulletCount: 3 },
  ];
  let education: EducationEntry[] = [
    { institution: '', degreeMajor: '', date: '', location: '' },
  ];

  function loadDetailsFromStorage() {
    if (typeof window === 'undefined') return;
    try {
      const raw = localStorage.getItem(DETAILS_STORAGE_KEY);
      if (raw) {
        const data = JSON.parse(raw);
        if (data.fullName !== undefined) fullName = data.fullName;
        if (data.location !== undefined) location = data.location;
        if (data.phone !== undefined) phone = data.phone;
        if (data.email !== undefined) email = data.email;
        if (data.experienceLevel !== undefined) experienceLevel = data.experienceLevel;
        if (Array.isArray(data.career) && data.career.length > 0) career = data.career;
        if (Array.isArray(data.education) && data.education.length > 0) education = data.education;
      }
    } catch (_) {}
  }
  loadDetailsFromStorage();

  $: if (typeof window !== 'undefined') {
    try {
      localStorage.setItem(
        DETAILS_STORAGE_KEY,
        JSON.stringify({
          fullName,
          location,
          phone,
          email,
          experienceLevel,
          career,
          education,
        })
      );
    } catch (_) {}
  }

  function addCareer() {
    career = [
      ...career,
      { companyName: '', date: '', location: '', bulletCount: 3 },
    ];
  }
  function removeCareer(i: number) {
    career = career.filter((_, idx) => idx !== i);
  }

  function addEducation() {
    education = [
      ...education,
      { institution: '', degreeMajor: '', date: '', location: '' },
    ];
  }
  function removeEducation(i: number) {
    education = education.filter((_, idx) => idx !== i);
  }

  const PROMPT_STATIC_AFTER_PROFILE = `
========================================
JOB DESCRIPTION HANDLING RULES
========================================
1. If the job requires any security clearance, DO NOT generate a resume.
2. If the job requires on-site, hybrid work arrangement and there is no possibility to work remotely in candidate's location, DO NOT generate a resume.

========================================
OUTPUT FORMAT (STRICT)
========================================
Return ONE JSON object with the following structure:
{
  "header": {
    "name": "",
    "role": "",
    "address": "",
    "phone": "",
    "email": ""
  },
  "summary": "",
  "skills": [
    { "category": ["Skill1", "Skill2", "..."] }
  ],
  "experience": [
    {
      "company": "",
      "title": "",
      "duration": "",
      "location": "",
      "sentences": [
        "Sentence 1",
        "Sentence 2"
      ]
    }
  ],
  "education": [
    {
      "institution": "",
      "degree": "",
      "date": "",
      "location": ""
    }
  ]
}

========================================
CONTENT RULES
========================================
ROLE
- 2-4 words long
- Appropriate for the job description
- Simple, Concise, and Common in the industry
JOB TITLE RULES:
- 2-4 words long
- Appropriate for the job description
- Simple, Concise, and Common in the industry
- Job titles should be familiar to the career flow
SUMMARY
- 3–4 sentences
- Professional, ATS-optimized, Confident and Concise
- Aligned directly to the job description
SKILLS
- 30–35 total skills
- Categorized
- Must include technologies from the job description
- Only include technologies released during the work period
EXPERIENCE – SENTENCE RULES (VERY IMPORTANT)
- Third-person only without the name, and he or she
- Each sentence must be 150–220 characters and contain detailed, technically rich descriptions of your role, specific contributions, and technologies used.
- Each sentence must end with a period.
- Each experience must reference company industry relevance.
- No bullet symbols.
- No sentence may be vague or generic.
- No special characters except "/" or "-" when required (examples: CI/CD, T-SQL).
- All technologies must be included only after their first version. (or alpha release date)
SENTENCE COUNT PER COMPANY
{{SENTENCE_COUNT_PER_COMPANY}}
EDUCATION DEGREE RULES:
Only modify the degree if it is not related to the job description.
- Each degree should be appropriate for the job description.
- Each degree should be common in the industry.

========================================
FORMATTING RULES
========================================
- JSON ONLY
- ONE code block ONLY
- No markdown outside JSON
- No comments
- No trailing commas
- Valid JSON syntax
- ATS-safe language only

========================================
FINAL VALIDATION
========================================
Before responding, verify:
- All job description technologies are included
- Sentence length requirements are met
- Sentence count requirements are met
- Job titles are aligned to the role
- Output is valid JSON
========================================
JOB DESCRIPTION
========================================

`;

  $: candidateProfileBlock = (() => {
    const lines: string[] = [
      'Name: ' + (fullName.trim() || '(not set)'),
      'Address: ' + (location.trim() || '(not set)'),
      'Phone: ' + (phone.trim() || '(not set)'),
      'Email: ' + (email.trim() || '(not set)'),
    ];
    if (education.length > 0 && (education[0].institution || education[0].degreeMajor || education[0].date)) {
      const ed = education
        .filter((e) => e.institution || e.degreeMajor || e.date)
        .map((e) => [e.degreeMajor, e.institution, e.date, e.location].filter(Boolean).join(', '))
        .join('; ');
      if (ed) lines.push('Education: ' + ed);
    }
    if (experienceLevel.trim()) {
      lines.push('Experience Level: ' + experienceLevel.trim());
    }
    if (career.length > 0 && career.some((c) => c.companyName || c.date)) {
      lines.push(
        'Career History:',
        ...career
          .filter((c) => c.companyName || c.date)
          .map((c) => `${c.companyName}: ${c.date}`.trim())
      );
    }
    return lines.join('\n');
  })();

  $: sentenceCountPerCompany = career
    .filter((c) => c.companyName.trim())
    .map((c) => `- ${c.companyName}: ${c.bulletCount} sentences`)
    .join('\n') || '- (add career entries above)';

  $: aiPromptText =
    `You are an expert resume writer.
Your task is to generate a resume based on the candidate's profile and the job description.
Your output MUST be exactly ONE valid JSON object inside a single code block.
Do NOT include explanations, comments, markdown, headers, or text outside the JSON.
Do NOT generate multiple code blocks.
Do NOT ask questions.

If any rule fails, STOP and return a plain text error message instead of JSON.

========================================
CANDIDATE PROFILE
========================================
${candidateProfileBlock}
` + PROMPT_STATIC_AFTER_PROFILE.replace('{{SENTENCE_COUNT_PER_COMPANY}}', sentenceCountPerCompany);

  let copyButtonJustCopied = false;

  async function copyAiPromptToClipboard() {
    try {
      await navigator.clipboard.writeText(aiPromptText);
      copyButtonJustCopied = true;
      setTimeout(() => {
        copyButtonJustCopied = false;
      }, 3000);
    } catch (_) {}
  }
</script>

<main class="layout">
  <section class="column-first">
    <div class="details-panel">
      <h2 class="details-heading">Details</h2>

      <label class="detail-row">
        <span class="detail-label">Full name</span>
        <input type="text" bind:value={fullName} placeholder="Your full name" />
      </label>
      <label class="detail-row">
        <span class="detail-label">Location</span>
        <input type="text" bind:value={location} placeholder="City, Country" />
      </label>
      <label class="detail-row">
        <span class="detail-label">Phone No</span>
        <input type="text" bind:value={phone} placeholder="+1 234 567 8900" />
      </label>
      <label class="detail-row">
        <span class="detail-label">Email</span>
        <input type="email" bind:value={email} placeholder="you@example.com" />
      </label>
      <label class="detail-row">
        <span class="detail-label">Experience level</span>
        <input
          type="text"
          bind:value={experienceLevel}
          placeholder="e.g. Senior with 10+ years of experience"
        />
      </label>

      <div class="section-block">
        <div class="section-header">
          <h3 class="section-title">Career</h3>
          <button type="button" class="btn-small" on:click={addCareer}
            >+ Add</button
          >
        </div>
        {#each career as entry, i}
          <div class="multi-row multi-row-career">
            <input
              type="text"
              bind:value={entry.companyName}
              placeholder="Company name"
            />
            <input type="text" bind:value={entry.date} placeholder="Date" />
            <input type="text" bind:value={entry.location} placeholder="Location" />
            <input
              type="number"
              bind:value={entry.bulletCount}
              min="0"
              max="20"
              placeholder="#"
              class="input-bullets"
              title="Number of bullet points"
            />
            <button
              type="button"
              class="btn-remove"
              on:click={() => removeCareer(i)}
              title="Remove row"
              disabled={career.length <= 1}
            >−</button>
          </div>
        {/each}
      </div>

      <div class="section-block">
        <div class="section-header">
          <h3 class="section-title">Education</h3>
          <button type="button" class="btn-small" on:click={addEducation}
            >+ Add</button
          >
        </div>
        {#each education as entry, i}
          <div class="multi-row multi-row-4">
            <input
              type="text"
              bind:value={entry.institution}
              placeholder="Institution"
            />
            <input
              type="text"
              bind:value={entry.degreeMajor}
              placeholder="Degree and major"
            />
            <input type="text" bind:value={entry.date} placeholder="Date" />
            <input type="text" bind:value={entry.location} placeholder="Location" />
            <button
              type="button"
              class="btn-remove"
              on:click={() => removeEducation(i)}
              title="Remove row"
              disabled={education.length <= 1}
            >−</button>
          </div>
        {/each}
      </div>
    </div>
  </section>
  <section class="column-second">
    <div class="textarea-wrap">
      <textarea
        class="panel panel-readonly"
        placeholder="AI prompt (synced from Details)"
        spellcheck="false"
        readonly
        value={aiPromptText}
      ></textarea>
      <button type="button" class="btn-icon copy-btn" title={copyButtonJustCopied ? 'Copied!' : 'Copy'} aria-label={copyButtonJustCopied ? 'Copied' : 'Copy'} on:click={copyAiPromptToClipboard}>
        <span class="copy-btn-icon" class:copied={copyButtonJustCopied}>
          <svg class="icon-copy" xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <rect x="9" y="9" width="13" height="13" rx="2" ry="2"/>
            <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/>
          </svg>
          <svg class="icon-check" xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <polyline points="20 6 9 17 4 12"/>
          </svg>
        </span>
      </button>
    </div>
    <div class="textarea-wrap">
      <textarea
        class="panel"
        placeholder="Column 2 – row 2"
        spellcheck="false"
      ></textarea>
      <button type="button" class="btn-icon" title="Download" aria-label="Download">
        <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
          <polyline points="7 10 12 15 17 10"/>
          <line x1="12" y1="15" x2="12" y2="3"/>
        </svg>
      </button>
    </div>
  </section>
</main>

<style>
  .layout {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.5rem;
    height: 100vh;
    width: 100%;
    padding: 0.5rem;
    box-sizing: border-box;
  }

  .column-first {
    display: flex;
    flex-direction: column;
    min-height: 0;
    width: 100%;
    height: 100%;
  }

  .details-panel {
    width: 100%;
    height: 100%;
    min-height: 0;
    padding: 0.75rem;
    border-radius: 8px;
    border: 1px solid rgba(255, 255, 255, 0.15);
    background: rgba(255, 255, 255, 0.05);
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    box-sizing: border-box;
    overflow-y: auto;
  }

  .details-heading {
    margin: 0 0 0.25rem 0;
    font-size: 1.1rem;
    font-weight: 600;
  }

  .detail-row {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .detail-label {
    font-size: 0.85rem;
    color: rgba(255, 255, 255, 0.7);
  }

  .detail-row input,
  .multi-row input {
    padding: 0.5rem 0.6rem;
    border-radius: 6px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    background: rgba(0, 0, 0, 0.2);
    color: inherit;
    font: inherit;
    box-sizing: border-box;
  }

  .detail-row input {
    width: 100%;
  }

  .detail-row input::placeholder,
  .multi-row input::placeholder {
    color: rgba(255, 255, 255, 0.4);
  }

  .section-block {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    padding-top: 0.5rem;
    border-top: 1px solid rgba(255, 255, 255, 0.1);
  }

  .section-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.5rem;
  }

  .section-title {
    margin: 0;
    font-size: 1rem;
    font-weight: 600;
  }

  .btn-small {
    padding: 0.35rem 0.6rem;
    font-size: 0.85rem;
  }

  .multi-row {
    display: grid;
    grid-template-columns: 1fr 1fr 1fr auto;
    gap: 0.4rem;
    align-items: center;
  }

  .multi-row-career {
    grid-template-columns: 1fr 1fr 1fr 3.5rem auto;
  }

  .input-bullets {
    width: 100%;
    min-width: 0;
    text-align: center;
  }

  .multi-row-4 {
    grid-template-columns: 1fr 1fr 1fr 1fr auto;
  }

  .multi-row input {
    min-width: 0;
  }

  .btn-remove {
    padding: 0.4rem 0.6rem;
    min-width: 2rem;
    font-size: 1.1rem;
    line-height: 1;
  }

  .btn-remove:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  @media (prefers-color-scheme: light) {
    .details-panel {
      border-color: rgba(0, 0, 0, 0.12);
      background: rgba(0, 0, 0, 0.03);
    }
    .detail-label {
      color: rgba(0, 0, 0, 0.65);
    }
    .detail-row input,
    .multi-row input {
      border-color: rgba(0, 0, 0, 0.15);
      background: rgba(255, 255, 255, 0.6);
    }
    .detail-row input::placeholder,
    .multi-row input::placeholder {
      color: rgba(0, 0, 0, 0.45);
    }
    .section-block {
      border-top-color: rgba(0, 0, 0, 0.1);
    }
  }

  .column-second {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    min-height: 0;
  }

  .textarea-wrap {
    position: relative;
    flex: 1;
    min-height: 0;
    display: flex;
  }

  .textarea-wrap .panel {
    flex: 1;
  }

  .btn-icon {
    position: absolute;
    top: 0.5rem;
    right: 0.5rem;
    width: 28px;
    height: 28px;
    padding: 0;
    border: none;
    border-radius: 6px;
    background: rgba(255, 255, 255, 0.1);
    color: rgba(255, 255, 255, 0.8);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.15s, color 0.15s;
  }

  .btn-icon:hover {
    background: rgba(255, 255, 255, 0.2);
    color: #fff;
  }

  .btn-icon:active {
    background: rgba(255, 255, 255, 0.15);
  }

  .copy-btn-icon {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    position: relative;
    width: 14px;
    height: 14px;
  }

  .copy-btn-icon .icon-copy,
  .copy-btn-icon .icon-check {
    position: absolute;
    inset: 0;
    margin: auto;
    transition: opacity 0.25s ease;
  }

  .copy-btn-icon .icon-copy {
    opacity: 1;
  }

  .copy-btn-icon .icon-check {
    opacity: 0;
  }

  .copy-btn-icon.copied .icon-copy {
    opacity: 0;
  }

  .copy-btn-icon.copied .icon-check {
    opacity: 1;
  }

  .panel {
    flex: 1;
    min-height: 0;
    width: 100%;
    padding: 0.75rem;
    border-radius: 8px;
    border: 1px solid rgba(255, 255, 255, 0.15);
    background: rgba(255, 255, 255, 0.05);
    color: inherit;
    font: inherit;
    resize: none;
    box-sizing: border-box;
  }

  .column-second .panel {
    flex: 1;
  }

  .panel-readonly {
    cursor: default;
    opacity: 0.95;
  }

  .panel::placeholder {
    color: rgba(255, 255, 255, 0.4);
  }

  @media (prefers-color-scheme: light) {
    .panel {
      border-color: rgba(0, 0, 0, 0.12);
      background: rgba(0, 0, 0, 0.03);
    }
    .panel::placeholder {
      color: rgba(0, 0, 0, 0.45);
    }
    .btn-icon {
      background: rgba(0, 0, 0, 0.06);
      color: rgba(0, 0, 0, 0.7);
    }
    .btn-icon:hover {
      background: rgba(0, 0, 0, 0.12);
      color: #000;
    }
    .btn-icon:active {
      background: rgba(0, 0, 0, 0.08);
    }
  }
</style>
