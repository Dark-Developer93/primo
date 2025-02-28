<script>
  import { fade } from 'svelte/transition';
  import { createEventDispatcher } from 'svelte';

  export let level = 0
  export let child = false;
  export let disabled = false;
  export let isFirst = false;
  export let isLast = false;
  export let minimal = false;
  export let showDefaultValue = true
  export let showVisibilityOptions = false

  const dispatch = createEventDispatcher();

  function moveItem(direction) {
    dispatch('move', direction);
  }

</script>

<div
  style="margin-left: {level}rem"
  class="field-container"
  class:has-default-value={showDefaultValue}
  class:has-visibility-options={showVisibilityOptions}
  class:child
  class:minimal>
  <label class="type">
    <span>Type</span>
    <slot name="type" />
  </label>
  {#if minimal}
    <slot name="main" />
  {:else}
    <!-- svelte-ignore a11y-label-has-associated-control -->
    <label>
      <span>Label</span>
      <slot name="label" />
    </label>
    <!-- svelte-ignore a11y-label-has-associated-control -->
    <div class="field">
      <label>
        <span>ID</span>
        <slot name="key" />
      </label>
    </div>
    {#if showDefaultValue}
    <div class="field">
      <!-- svelte-ignore a11y-label-has-associated-control -->
      <label>
        <span>Default Value</span>
        <slot name="default-value" />
      </label>
    </div>
    {/if}
    {#if showVisibilityOptions}
      <label class="hide">
        <span>Visible</span>
        <slot name="hide" />
      </label>
    {/if}
  {/if}
  <div class="option-buttons">
    <button disabled={isFirst} title="Move up" on:click={() => moveItem('up')}>
      <i class="fas fa-arrow-up" />
    </button>
    <button
      disabled={isLast}
      title="Move down"
      on:click={() => moveItem('down')}>
      <i class="fas fa-arrow-down" />
    </button>
    <button on:click={() => dispatch('delete')} {disabled} title="delete field">
      <i class="fas fa-trash" />
    </button>
  </div>
</div>

<style lang="postcss">
  .field-container {
    display: grid;
    grid-template-columns: auto 1fr 1fr auto;
    padding: 0.5rem;
    gap: 1rem;

    &.has-default-value {
      grid-template-columns: auto 1fr 1fr 1fr auto;
    }

    &.has-visibility-options {
      grid-template-columns: auto 1fr 1fr minmax(4rem, auto) auto;
    }

    &.has-default-value.has-visibility-options {
      grid-template-columns: auto 1fr 1fr 1fr minmax(4rem, auto) auto;
    }

    &.child {
      margin-left: 1rem;
      padding: 0.25rem 0.5rem;
    }

    .type {
      border-radius: 1px;
      display: flex;
      min-width: 3rem;
    }

    label {
      display: flex;
      flex-direction: column;
      flex: 1;

      span {
        font-weight: 600;
        font-size: var(--font-size-1);
        padding-bottom: 2px;
      }
    }

    .option-buttons {
      /* padding: 0.25rem 0.5rem; */
      color: var(--color-gray-3);
      background: var(--color-gray-900);
      z-index: 10;
      border-radius: var(--primo-border-radius);
      display: flex;
      align-items: center;
      justify-content: flex-end;

      button {
        border-radius: 1px;
        transition: color 0.1s;
        &:hover,
        &:focus {
          color: var(--color-gray-4);
        }
        &:first-child {
          margin-right: 0.25rem;
        }

        &:last-child {
          margin-left: 0.5rem;
          color: var(--color-gray-5);
        }
      }
    }

    &.minimal {
      grid-template-columns: auto 1fr auto;
    }
  }
  button[disabled] {
    color: var(--color-gray-7);
    cursor: default;
  }
  span {
    color: var(--color-gray-3);
  }

</style>
