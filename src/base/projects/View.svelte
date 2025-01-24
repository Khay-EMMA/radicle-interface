<script lang="ts">
  import { Link } from 'svelte-routing';
  import type { Config } from '@app/config';
  import * as proj from '@app/project';
  import Loading from '@app/Loading.svelte';
  import Modal from '@app/Modal.svelte';
  import Avatar from '@app/Avatar.svelte';
  import { Org } from '@app/base/orgs/Org';

  import Browser from './Browser.svelte';

  export let urn: string;
  export let org = "";
  export let commit = "";
  export let config: Config;
  export let path: string;

  let projectRoot = proj.path({ urn, org, commit });
  let getProject = new Promise<string | null>(resolve => {
    if (org) {
      Org.getProfile(org, config).then(p => resolve(p?.seed || null));
    } else {
      resolve(null);
    }
  }).then(async (seed) => {
    const cfg = seed ? config.withSeed(seed) : config;
    const info = await proj.getInfo(urn, cfg);
    return { project: info, config: cfg };
  });

  const back = () => window.history.back();
</script>

<style>
  main {
    width: 100%;
    max-width: var(--content-max-width);
    min-width: var(--content-min-width);
    padding: 4rem 0;
  }
  main > header {
    padding: 0 2rem 0 8rem;
  }

  @media (max-width: 800px) {
    main > header {
      padding-left: 2rem;
    }
  }

  .title {
    font-size: 2.25rem;
    margin-bottom: 0.5rem;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .urn {
    font-family: var(--font-family-monospace);
    font-size: 0.75rem;
    color: var(--color-foreground-faded);
  }
  .description {
    margin: 1rem 0 1.5rem 0;
  }
  .maintainers {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  .maintainer {
    display: inline-block;
    width: 1rem;
    height: 1rem;
    margin-left: 0.5rem;
  }
</style>

<main>
  {#await getProject}
    <header>
      <Loading center />
    </header>
  {:then result}
    <header>
      <div class="title bold">
        <Link to={projectRoot}>{result.project.meta.name}</Link>
        <span class="maintainers">
          {#each result.project.meta.maintainers as user}
            <span class="maintainer">
              <Avatar source={user} />
            </span>
          {/each}
        </span>
      </div>
      <div class="urn">{urn}</div>
      <div class="description">{result.project.meta.description}</div>
    </header>
    <Browser {urn} {org} {path}
      commit={commit || result.project.head}
      config={result.config} />
  {:catch}
    <Modal subtle>
      <span slot="title">🏜️</span>
      <span slot="body">
        <p class="highlight"><strong>{urn}</strong></p>
        <p>This project was not found.</p>
      </span>
      <span slot="actions">
        <button on:click={back}>
          Back
        </button>
      </span>
    </Modal>
  {/await}
</main>
