<script lang="ts">
  // Resume builder: 3-part layout
  import { jsPDF } from 'jspdf';

  const DETAILS_STORAGE_KEY = 'resume-builder-details';

  // Expected shape of resume JSON (from AI prompt output)
  type ResumeJson = {
    header?: { name?: string; role?: string; address?: string; phone?: string; email?: string };
    summary?: string;
    skills?: Array<Record<string, string[]>>;
    experience?: Array<{
      company?: string;
      title?: string;
      duration?: string;
      location?: string;
      sentences?: string[];
    }>;
    education?: Array<{
      institution?: string;
      degree?: string;
      date?: string;
      location?: string;
    }>;
  };

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
- Seniority level should be aligned with the candidate's total experience years.
JOB TITLE RULES:
- 2-4 words long
- Appropriate for the job description
- Simple, Concise, and Common in the industry
- Job titles should be familiar to the career flow
SUMMARY
- Professional, ATS-optimized, Confident and Concise
- Aligned directly to the job description
SKILLS
- 30–40 total skills
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

  let jsonInput = '';
  let downloadError = '';

  function extractJsonFromRaw(raw: string): string {
    const trimmed = raw.trim();
    const codeBlock = /^```(?:json)?\s*([\s\S]*?)```\s*$/m.exec(trimmed);
    return codeBlock ? codeBlock[1].trim() : trimmed;
  }

  function parseResumeJson(raw: string): ResumeJson | null {
    try {
      const jsonStr = extractJsonFromRaw(raw);
      if (!jsonStr) return null;
      const data = JSON.parse(jsonStr) as ResumeJson;
      if (!data || typeof data !== 'object') return null;
      return data;
    } catch {
      return null;
    }
  }

  function downloadResumePdf() {
    downloadError = '';
    const data = parseResumeJson(jsonInput);
    if (!data) {
      downloadError = 'Invalid or empty JSON. Paste the resume JSON from the AI output.';
      return;
    }

    const doc = new jsPDF({ unit: 'mm', format: 'letter' });
    const margin = 10.16;
    const pageW = 215.9;
    const pageH = 279.4;
    const contentW = pageW - 2 * margin;
    // Resume styles: Helvetica, 11pt body, 22pt full name, 1.5x line spacing
    const bodyFontSize = 11;
    const nameFontSize = 22;
    const lineSpacing = 1.5;
    const ptToMm = 25.4 / 72;
    const bodyLineHeight = bodyFontSize * lineSpacing * ptToMm;
    // offset baseline so the cap-top of the first text lands at the margin edge
    let y = margin + nameFontSize * 0.72 * ptToMm;
    const nameLineHeight = nameFontSize * 1.15 * ptToMm;
    const sectionGap = bodyLineHeight;

    function nextLine(h = bodyLineHeight) {
      y += h;
    }

    function checkNewPage(needed = 15) {
      if (y > pageH - margin - needed) {
        doc.addPage();
        y = margin;
      }
    }

    function addWrappedText(text: string, fontSize: number, bold = false) {
      doc.setFontSize(fontSize);
      doc.setFont('helvetica', bold ? 'bold' : 'normal');
      const lineHeightMm = fontSize * lineSpacing * ptToMm;
      const lines = doc.splitTextToSize(text, contentW);
      for (const line of lines) {
        checkNewPage(lineHeightMm);
        doc.text(line, margin, y);
        y += lineHeightMm;
      }
    }

    function addSectionTitle(title: string) {
      const sectionTitleLineHeight = bodyFontSize * 2.0 * ptToMm;
      checkNewPage(sectionTitleLineHeight * 1.5);
      y += sectionGap;
      doc.setFontSize(bodyFontSize);
      doc.setFont('helvetica', 'bold');
      doc.text(title, margin, y);
      y += sectionTitleLineHeight;
    }

    // Header
    const h = data.header;
    if (h?.name) {
      doc.setFontSize(nameFontSize);
      doc.setFont('helvetica', 'bold');
      doc.text(h.name, margin, y);
      y += nameLineHeight;
    }
    if (h?.role) {
      doc.setFontSize(bodyFontSize);
      doc.setFont('helvetica', 'normal');
      doc.text(h.role, margin, y);
      nextLine();
    }
    const contactParts = [h?.address, h?.phone, h?.email].filter(Boolean);
    if (contactParts.length) {
      doc.setFontSize(bodyFontSize);
      doc.text(contactParts.join('  |  '), margin, y);
      nextLine();
    }

    // Summary
    if (data.summary) {
      addSectionTitle('PROFESSIONAL SUMMARY');
      addWrappedText(data.summary, bodyFontSize);
    }

    // Skills
    if (data.skills?.length) {
      addSectionTitle('TECHNICAL SKILLS');
      doc.setFontSize(bodyFontSize);
      const skillLineHeight = bodyFontSize * lineSpacing * ptToMm;
      for (const cat of data.skills) {
        for (const [category, list] of Object.entries(cat)) {
          if (list?.length) {
            const categoryLabel = category ? category + ': ' : '';
            const skillsText = list.join(', ');

            if (!categoryLabel) {
              addWrappedText(skillsText, bodyFontSize);
            } else {
              doc.setFont('helvetica', 'bold');
              const catWidth = doc.getTextWidth(categoryLabel);
              doc.setFont('helvetica', 'normal');
              const fullText = categoryLabel + skillsText;
              const lines = doc.splitTextToSize(fullText, contentW);

              checkNewPage(skillLineHeight);
              doc.setFont('helvetica', 'bold');
              doc.text(categoryLabel, margin, y);
              doc.setFont('helvetica', 'normal');
              doc.text(lines[0].substring(categoryLabel.length), margin + catWidth, y);
              y += skillLineHeight;

              for (let i = 1; i < lines.length; i++) {
                checkNewPage(skillLineHeight);
                doc.text(lines[i], margin, y);
                y += skillLineHeight;
              }
            }
          }
        }
      }
    }

    // Experience
    if (data.experience?.length) {
      addSectionTitle('WORK EXPERIENCE');
      for (let ei = 0; ei < data.experience.length; ei++) {
        const exp = data.experience[ei];
        checkNewPage(bodyLineHeight * 2);
        if (exp.title || exp.duration) {
          doc.setFontSize(bodyFontSize);
          doc.setFont('helvetica', 'bold');
          if (exp.title) doc.text(exp.title, margin, y);
          if (exp.duration) doc.text(exp.duration, margin + contentW, y, { align: 'right' });
          nextLine();
        }
        if (exp.company || exp.location) {
          doc.setFontSize(bodyFontSize);
          doc.setFont('helvetica', 'normal');
          if (exp.company) doc.text(exp.company, margin, y);
          if (exp.location) doc.text(exp.location, margin + contentW, y, { align: 'right' });
          nextLine();
        }
        if (exp.sentences?.length) {
          doc.setFontSize(bodyFontSize);
          doc.setFont('helvetica', 'normal');
          const bullet = '\u2022';
          const bulletGap = doc.getTextWidth(bullet + ' ');
          const bulletIndent = contentW - bulletGap;
          for (const sent of exp.sentences) {
            const lines = doc.splitTextToSize(sent, bulletIndent);
            checkNewPage(bodyLineHeight);
            doc.text(bullet, margin, y);
            doc.text(lines[0], margin + bulletGap, y);
            y += bodyLineHeight;
            for (let li = 1; li < lines.length; li++) {
              checkNewPage(bodyLineHeight);
              doc.text(lines[li], margin + bulletGap, y);
              y += bodyLineHeight;
            }
          }
        }
        if (ei < data.experience.length - 1) {
          nextLine(bodyLineHeight * 0.5);
        }
      }
    }

    // Education
    if (data.education?.length) {
      addSectionTitle('EDUCATION');
      doc.setFontSize(bodyFontSize);
      doc.setFont('helvetica', 'normal');
      for (const ed of data.education) {
        checkNewPage(bodyLineHeight);
        const line = [ed.degree, ed.institution, ed.date, ed.location].filter(Boolean).join(', ');
        if (line) addWrappedText(line, bodyFontSize);
      }
    }

    const fileName = (h?.name || 'resume').replace(/\s+/g, '-').replace(/[^\w\-]/g, '') || 'resume';
    doc.save(`${fileName}.pdf`);
  }

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
          <h3 class="section-title">WORK EXPERIENCE</h3>
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
          <h3 class="section-title">EDUCATION</h3>
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
    <div class="textarea-wrap json-column">
      <textarea
        class="panel"
        placeholder="Paste resume JSON here (from AI output). Then click Download to generate PDF."
        spellcheck="false"
        bind:value={jsonInput}
      ></textarea>
      {#if downloadError}
        <p class="download-error" role="alert">{downloadError}</p>
      {/if}
      <button type="button" class="btn-icon download-btn" title="Download PDF" aria-label="Download PDF" on:click={downloadResumePdf}>
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
    flex-direction: column;
  }

  .textarea-wrap.json-column {
    gap: 0.25rem;
  }

  .download-error {
    margin: 0;
    padding: 0.35rem 0.5rem;
    font-size: 0.8rem;
    color: #f87171;
    background: rgba(248, 113, 113, 0.12);
    border-radius: 6px;
    flex-shrink: 0;
  }

  .json-column .panel {
    flex: 1;
    min-height: 0;
  }

  .json-column .download-btn {
    position: absolute;
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
