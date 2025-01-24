<script lang="ts">
  import { onMount } from 'svelte';
  import { navigate } from 'svelte-routing';
  import type { Config } from '@app/config';
  import { session } from '@app/session';
  import Loading from '@app/Loading.svelte';
  import Link from '@app/Link.svelte';
  import Modal from '@app/Modal.svelte';
  import Form from '@app/Form.svelte';
  import type { Field } from '@app/Form.svelte';
  import { assert } from '@app/error';
  import Error from '@app/Error.svelte';
  import { isAddressEqual, isReverseRecordSet } from '@app/utils';

  import { getRegistration } from './registrar';
  import type { EnsRecord } from './resolver';
  import type { Registration } from './registrar';
  import Update from './Update.svelte';

  enum Status {
    Loading,
    Found,
    NotFound,
    Failed,
  }

  type State =
      { status: Status.Loading }
    | { status: Status.NotFound }
    | { status: Status.Found; registration: Registration }
    | { status: Status.Failed; error: string };

  export let subdomain: string;
  export let config: Config;

  subdomain = subdomain.toLowerCase();

  let state: State = { status: Status.Loading };
  let editable = false;
  let fields: Field[] = [];
  let name = `${subdomain}.${config.registrar.domain}`;
  let updateRecords: EnsRecord[] | null = null;

  onMount(() => {
    getRegistration(name, config)
      .then(async r => {
        if (r) {
          let reverseRecord = false;
          if (r.address) {
            reverseRecord = await isReverseRecordSet(r.address, name, config);
          }

          fields = [
            { name: "owner", placeholder: "",
              description: "The owner and controller of this name.",
              value: r.owner, resolve: true, editable: false },
            { name: "address", placeholder: "Ethereum address, eg. 0x4a9cf21...bc91",
              description: "The address this name resolves to. " + (
                reverseRecord
                  ? `The reverse record for this address is set to **${name}**`
                  : "The reverse record for this address is **not set**. "
                  + "For this name to be correctly associated with the address, "
                  + "a reverse record should be set."
              ),
              value: r.address, editable: true },
            { name: "url", label: "URL", placeholder: "https://acme.org",
              description: "A homepage or other URL associated with this name.",
              value: r.url,editable: true },
            { name: "avatar", placeholder: "https://acme.org/avatar.png",
              description: "An avatar or square image associated with this name.",
              value: r.avatar, editable: true },
            { name: "twitter", placeholder: "Twitter username, eg. 'acme'",
              description: "The Twitter handle associated with this name.",
              value: r.twitter, editable: true },
            { name: "github", label: "GitHub", placeholder: "GitHub username, eg. 'acme'",
              description: "The GitHub username associated with this name.",
              value: r.github, editable: true },
            { name: "seed.id", label: "Seed ID", placeholder: "hynkyn...3nrzc@seed.acme.org:8887",
              description: "The ID of a Radicle Link node that hosts entities associated with this name.",
              value: r.seedId, editable: true },
            { name: "seed.api", label: "Seed API", placeholder: "https://seed.acme.org:8888",
              description: "The HTTP address of a node that serves Radicle entities over HTTP.",
              value: r.seedApi, editable: true },
          ];
          state = { status: Status.Found, registration: r };
        } else {
          state = { status: Status.NotFound };
        }
        return r;
      }).catch(err => {
        state = { status: Status.Failed, error: err };
      });
  });

  const onSave = async (event: { detail: { name: string; value: string | null }[] }) => {
    assert(state.status === Status.Found, "registration must be found");

    updateRecords = event.detail
      .filter(f => f.value !== null)
      .map(f => {
        return { name: f.name, value: f.value as string };
      });
  };

  $: isOwner = (registration: Registration): boolean => {
    return $session ? isAddressEqual(registration.owner, $session.address) : false;
  };
</script>

<style>
  main {
    padding: 5rem 0;
  }
  main > header {
    display: flex;
    align-items: center;
    justify-content: left;
    margin-bottom: 2rem;
  }
  main > header > * {
    margin: 0 1rem 0 0;
  }
</style>

{#if state.status === Status.Loading}
  <Loading />
{:else if state.status === Status.Failed}
  <Error title="Registration could not be loaded" on:close={() => navigate('/registrations')}>
    {state.error}
  </Error>
{:else if state.status === Status.NotFound}
  <Modal subtle>
    <span slot="title" class="secondary">
      <div>🍄</div>
      {subdomain}.{config.registrar.domain}
    </span>

    <span slot="body">
      <p>The name <strong>{subdomain}</strong> is not registered.</p>
    </span>

    <span slot="actions">
      <Link to={`/registrations/${subdomain}/form`} primary>Register &rarr;</Link>
    </span>
  </Modal>
{:else if state.status === Status.Found}
  <main>
    <header>
      <h1 class="bold">{subdomain}.{config.registrar.domain}</h1>
      <button
        class="tiny primary" class:active={editable} disabled={!isOwner(state.registration)}
        on:click={() => editable = !editable}>
          Edit
      </button>
      <button class="tiny secondary" disabled>
        Transfer
      </button>
    </header>
    <Form {config} {editable} {fields} on:save={onSave} on:cancel={() => editable = false} />
  </main>

  {#if updateRecords}
    <Update {config} {subdomain} on:close={() => updateRecords = null}
            registration={state.registration} records={updateRecords} />
  {/if}
{/if}
