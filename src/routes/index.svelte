<script>
  import algosdk from 'algosdk';

  const algodServer = 'https://testnet-algorand.api.purestake.io/ps2';
  const port = '';
  const token = { 'X-API-Key': 'ADoROavzga25nBO0Q4OEKasmr7bkz72l9coQgXN8' };
  const client = new algosdk.Algodv2(token, algodServer, port);

  const indexerServer = 'https://testnet-algorand.api.purestake.io/idx2';
  const indexerClient = new algosdk.Indexer(token, indexerServer, port);

  let status;

  let created;
  let accounts = [];

  const createAccount = () => {
    created = algosdk.generateAccount();
    accounts = [...accounts, created];
  };

  let restoreMnemonic, restored;
  const restoreAccount = () => {
    try {
      restored = algosdk.mnemonicToSecretKey(restoreMnemonic);
    } catch (err) {
      restored.error = err;
      restored.status = 'error';
      return;
    }
    restored.status = 'ok';
    accounts = [...accounts, restored];
  };

  let defaultAcc;

  $: balance = defaultAcc
    ? client.accountInformation(defaultAcc.addr).do()
    : null;

  const encode = (str) => new TextEncoder().encode(str);

  let reciever, note, amount, transaction;
  const createTxn = async () => {
    const suggestedParams = await client.getTransactionParams().do();
    const txnOptions = {
      from: defaultAcc.addr,
      to: reciever,
      amount: algosdk.algosToMicroalgos(amount),
      note: encode(note),
      suggestedParams,
    };
    const txn = algosdk.makePaymentTxnWithSuggestedParamsFromObject(txnOptions);
    const signedTxn = txn.signTxn(defaultAcc.sk);
    await client.sendRawTransaction(signedTxn).do();
    const txId = txn.txID().toString();

    const tx = await algosdk.waitForConfirmation(client, txId, 2);

    return { tx, txId };
  };

  const filters = [
    ['acc', 'Account ID'],
    ['start', 'Start Date'],
    ['end', 'End Date'],
    ['txId', 'Transaction ID'],
    ['note', 'Note Field'],
  ];
  let selectedFilters = [];

  let selectedValues = { acc: '', start: '', end: '', txId: '', note: '' };

  let searchResult;

  const rfc3339 = (date) =>
    new Date(date).toISOString().slice(0, -5) + '+00:00';

  const search = async () => {
    const reducer = (prev, val) => {
      if (val === 'acc') return prev.address(selectedValues.acc);
      else if (val === 'start')
        return prev.afterTime(rfc3339(selectedValues.start));
      else if (val === 'end')
        return prev.beforeTime(rfc3339(selectedValues.end));
      else if (val === 'txId') return prev.txid(selectedValues.txId);
      else if (val === 'note')
        return prev.notePrefix(encode(selectedValues.note));
    };

    const res = selectedFilters.reduce(
      reducer,
      indexerClient.searchForTransactions()
    );

    return (await res.do()).transactions;
  };
</script>

<h1>Check Status</h1>
<button
  on:click="{() => {
    status = client.status().do();
  }}">Get Status</button>
{#if status}
  {#await status}
    Quering...
  {:then response}
    <details>
      <summary>Success</summary>
      <code>{JSON.stringify(response)}</code>
    </details>
  {:catch error}
    <details>
      <summary>Failed</summary>
      <code>{JSON.stringify(error)}</code>
    </details>
  {/await}
{/if}

<h1>Create Account</h1>
<button on:click="{createAccount}">Create Account</button>
{#if created}
  <ul>
    <li>
      Address: <code>{created.addr}</code>
    </li>
    <li>
      Seed Phrase: <code>{algosdk.secretKeyToMnemonic(created.sk)}</code>
    </li>
  </ul>
  <button
    on:click="{() => {
      created = null;
    }}">Clear</button>
{/if}

<h1>Restore Account</h1>
<input
  bind:value="{restoreMnemonic}"
  type="text"
  placeholder="Seed Phrase" /><button on:click="{restoreAccount}"
  >Restore</button>
{#if restored?.status === 'ok'}
  Account <code>{restored.addr}</code> restored successfully
  <button
    on:click="{() => {
      restored = null;
      restoreMnemonic = null;
    }}">Clear</button>
{:else if restored?.status === 'error'}
  Retore unsuccessful <code>{restored.error}</code>
  <button
    on:click="{() => {
      restored = null;
      restoreMnemonic = null;
    }}">Clear</button>
{/if}

<h1>Account List</h1>
{#if defaultAcc}
  Default account:<code>{defaultAcc.addr}</code>
{:else if accounts.length}
  Select default account from below
{:else}
  Create or add accounts first
{/if}

{#each accounts as account}
  <label>
    <input
      type="radio"
      name="account"
      value="{account}"
      bind:group="{defaultAcc}" />
    <code>{account.addr}</code>
  </label>
{/each}

<h1>See Balence</h1>
{#if defaultAcc}
  <button
    on:click="{() => {
      balance = client.accountInformation(defaultAcc.addr).do();
    }}">Refresh</button>
{/if}
{#if balance}
  {#await balance}
    Quering...
  {:then response}
    Amount: <code>{algosdk.microalgosToAlgos(response.amount)}</code>
  {:catch error}
    <details>
      <summary>Failed</summary>
      <code>{JSON.stringify(error)}</code>
    </details>
  {/await}
{/if}

<h1>Create Transaction</h1>
{#if defaultAcc}
  From: <code>{defaultAcc.addr}</code>
  <label>
    To: <input type="text" placeholder="Address" bind:value="{reciever}" />
  </label>
  <label>
    Note: <input type="text" placeholder="Note" bind:value="{note}" />
  </label>
  <label>
    Amount: <input type="number" placeholder="Algos" bind:value="{amount}" />
  </label>
  <button
    on:click="{() => {
      transaction = createTxn();
    }}">Send</button>
{/if}
{#if transaction}
  {#await transaction}
    Sending transaction and awaiting confirmation...
  {:then { tx, txId }}
    Confirmed with transaction ID: <code>{txId}</code> in round {tx[
      'confirmed-round'
    ]}
  {:catch error}
    <details>
      <summary>Failed</summary>
      <code>{error}</code>
    </details>
  {/await}
{/if}

<h1>Search for Transactions</h1>
<h3>Filters</h3>
{#each filters as [value, title]}
  <label>
    <input type="checkbox" value="{value}" bind:group="{selectedFilters}" />
    {title}
  </label>
{/each}
{#if selectedFilters.includes('acc')}
  <label>
    Account: <input
      type="text"
      placeholder="Address"
      bind:value="{selectedValues.acc}" />
  </label>
{/if}
{#if selectedFilters.includes('start')}
  <label>
    Start: <input type="datetime-local" bind:value="{selectedValues.start}" />
  </label>
{/if}
{#if selectedFilters.includes('end')}
  <label>
    End: <input type="datetime-local" bind:value="{selectedValues.end}" />
  </label>
{/if}
{#if selectedFilters.includes('txId')}
  <label>
    Transaction ID: <input
      type="text"
      placeholder="TxId"
      bind:value="{selectedValues.txId}" />
  </label>
{/if}
{#if selectedFilters.includes('note')}
  <label>
    Note or note prefix: <input
      type="text"
      placeholder="Note"
      bind:value="{selectedValues.note}" />
  </label>
{/if}
<button
  on:click="{() => {
    searchResult = search();
  }}">Search</button>
{#if searchResult}
  {#await searchResult}
    Searching...
  {:then response}
    <details>
      <summary>Success</summary>
      <code>{JSON.stringify(response)}</code>
    </details>
  {:catch error}
    <details>
      <summary>Failed</summary>
      <code>{error}</code>
    </details>
  {/await}
{/if}

<style>
  label {
    display: block;
  }
</style>
