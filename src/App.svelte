<script lang="ts">
  // Resume builder: 3-part layout
</script>

<main class="layout">
  <section class="column-first">
    <textarea
      class="panel"
      placeholder="Column 1 – main content"
      spellcheck="false"
    ></textarea>
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
