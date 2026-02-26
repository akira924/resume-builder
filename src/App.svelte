<script lang="ts">
  // Resume builder: 3-part layout

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

  let career: CareerEntry[] = [
    { companyName: '', date: '', location: '', bulletCount: 3 },
  ];
  let education: EducationEntry[] = [
    { institution: '', degreeMajor: '', date: '', location: '' },
  ];

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
    <textarea
      class="panel"
      placeholder="Column 2 – row 1"
      spellcheck="false"
    ></textarea>
    <textarea
      class="panel"
      placeholder="Column 2 – row 2"
      spellcheck="false"
    ></textarea>
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
  }
</style>
