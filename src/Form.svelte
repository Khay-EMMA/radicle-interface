<script context="module" lang="ts">
  export interface Field {
    name: string;
    value: string | null;
    label?: string;
    placeholder?: string;
    description: string;
    resolve?: boolean;
    editable: boolean;
  }
</script>

<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import marked from 'marked';
  import { capitalize, isUrl, isAddress } from '@app/utils';
  import Address from '@app/Address.svelte';
  import type { Config } from '@app/config';

  export let fields: Field[];
  export let editable = false;
  export let disabled = false;
  export let config: Config;

  const cleanup = (fields: Field[]): { name: string; value: string | null }[] => {
    return fields.filter(field => field.editable).map(field => {
      return {
        name: field.name,
        value: field.value && field.value.trim(),
      };
    });
  };
  const dispatch = createEventDispatcher();
  const save = () => dispatch('save', cleanup(fields));
  const cancel = () => dispatch('cancel');
</script>

<style>
  .fields {
    display: grid;
    grid-template-columns: auto auto;
    grid-gap: 1rem 1.5rem;
  }
  .fields > div {
    justify-self: start;
    align-self: center;
  }

  .field {
    display: flex;
    align-items: center;
    width: 28rem;
    height: 2.125rem;
    border: 1px dashed transparent;
    padding: 0.25rem 1rem;
    margin: 0;
    white-space: nowrap;
  }

  .description {
    padding-left: 1rem;
    max-width: 32rem;
  }
  .description :global(p) {
    padding: 0;
    margin: 0;
  }

  input.field {
    border-radius: 1rem;
    overflow: hidden;
    text-overflow: ellipsis;
    border-color: var(--color-secondary) !important;
  }
  input.field::placeholder {
    color: var(--color-secondary);
    font-style: italic;
  }
  input.field[disabled] {
    color: var(--color-secondary);
  }

  .label {
    border: 1px solid transparent;
    padding: 0.25rem;
    height: 100%;
    display: block;
  }

  .actions {
    margin-top: 2rem;
    text-align: center;
    visibility: hidden;
  }
  .actions button {
    margin-left: 1rem;
  }
  .actions.editable {
    visibility: visible;
  }
</style>

<div class="fields">
  {#each fields as field}
    <div class="label">
      {field.label || capitalize(field.name)}
    </div>
    <div>
      {#if field.editable && editable}
        <input name={field.name} class="field" placeholder={field.placeholder}
               bind:value={field.value} type="text" {disabled} />
      {:else}
        <span class="field">
          {#if field.value}
            {#if isUrl(field.value)}
              <span>
                <a class="link" href="{field.value}" target="_blank">{field.value}</a>
              </span>
            {:else if isAddress(field.value)}
              <Address resolve={field.resolve ?? false} address={field.value} {config} />
            {:else}
              {field.value}
            {/if}
          {:else}
            <span class="subtle not-set">&cross; Not set</span>
          {/if}
        </span>
      {/if}
      <div class="description text-small faded">
        {@html marked(field.description)}
      </div>
    </div>
  {/each}
</div>

<div class="actions" class:editable>
  <button on:click={cancel} {disabled} class="small">
    Cancel
  </button>
  <button on:click={save} {disabled} class="small primary">
    Save
  </button>
</div>
