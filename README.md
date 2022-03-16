Here we’ll set up Algosdk for Sveltekit which uses Vite under the hood for running development server and packaging for browser.

The configuration is described in Step 2 can be used for all others on Vite like Vue, vite-react, or the new Solidjs.

This tutorial achieves the following goals:

- create new accounts and accounts from seed phrase
- create payment transactions between them
- search for transactions using:
--	date ranges
--	transaction ID
--	accounts involved
--3	note field
Gitcoin bounty: https://gitcoin.co/issue/algorandfoundation/grow-algorand/135/100027549
Country: India
Wallet Address: GX2TD2AZBBBZ3MZ2K3GX4U6KQMDGLC5WI6ZSQYSKXA7EJCOMD4SPMSFQNA

# 1. Initialize Svelte 🎉

Run the `npm init svelte@next` command and use the options given below it when prompted.

```
✔ Where should we create your project?
(leave blank to use current directory) …
✔ Which Svelte app template? › Skeleton project
✔ Use TypeScript? … No
✔ Add ESLint for code linting? … No
✔ Add Prettier for code formatting? … No
✔ Add Playwright for browser testing? … No
```

Add algosdk with `npm install algosdk`

Start running the dev server with  `npm run dev` and open http://localhost:3000/

# 2. Vite Configuration

Add  `npm install @esbuild-plugins/node-globals-polyfill path-browserify` packages.

For SvelteKit the vite config is in `vite` object under `kit` inside svelte.config.js file:

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-auto';
import {NodeGlobalsPolyfillPlugin} from '@esbuild-plugins/node-globals-polyfill';

/** @type {import('@sveltejs/kit').Config} */
const config = {
	kit: {
		adapter: adapter(),
		vite: {
			resolve: {
				alias: {
					path: 'path-browserify',
				},
			},
			optimizeDeps: {
				esbuildOptions: {
					// Node.js global to browser globalThis
					define: {
						global: 'globalThis'
					},
					// Enable esbuild polyfill plugins
					plugins: [
						NodeGlobalsPolyfillPlugin({
							buffer: true
						})
					]
				}
			},
		}
	}
};

export default config;

```
For other vite projects add the config inside vite.config.js as:

```js
// vite.config.js
import { defineConfig } from 'vite'
import Vue from '@vitejs/plugin-vue'

import NodeGlobalsPolyfillPlugin from '@esbuild-plugins/node-globals-polyfill'

export default defineConfig({
  resolve: {
    alias: {
      path: 'path-browserify',
    },
  },
  optimizeDeps: {
    esbuildOptions: {
      // Node.js global to browser globalThis
      define: {
          global: 'globalThis'
      },
      // Enable esbuild polyfill plugins
      plugins: [
          NodeGlobalsPolyfillPlugin({
              buffer: true
          })
      ]
    }
  },
})

```

Restart the dev server by pressing Ctrl+C and  running `npm run dev` again.

**Edit:** You can also use the [instructions](https://github.com/algorand/js-algorand-sdk/blob/develop/FAQ.md#it-says-error-cant-resolve-in-the-sdk) provided on the js-algorand-sdk github but the above configuration is recommended because you don't need to manually work with the window object. This might be a problem if you plan to use SvelteKit's SSR mechanics in future.

# 3. Adding API Keys

But first get API tokens from [Purestake](https://developer.purestake.io/home) and add them in to get clients. We initiate the algod client and indexer client.

```
// src/routes/index.svelte

<script>
  import algosdk from 'algosdk';

  const baseServer = 'https://testnet-algorand.api.purestake.io/ps2';
  const port = '';
  const token = { 'X-API-Key': 'YOUR API KEY' };
  const client = new algosdk.Algodv2(token, baseServer, port);

  const indexerServer = 'https://testnet-algorand.api.purestake.io/idx2';
  const indexerClient = new algosdk.Indexer(token, indexerServer, port);
</script>

```
To test if API keys work, let's get status of the node.
```
<!-- Template -->
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
```
Here clicking the button gives the status promise into `status` variable. This promise is awaited and th value is passed to `response`. The response is displayed to the user after converting it from JSON to string inside `<code></code>` blocks.

# 4. Creating Account

Here we'll create and add accounts in src/routes/index.svelte below the above code.

```
// ...
// Script

  let created;
  let accounts = [];

  const createAccount = () => {
    created = algosdk.generateAccount();
    accounts = [...accounts, created];
  };
<script>

<!-- Template -->

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

```
Note using `accounts.push(created)` instead of `accounts = [...accounts, created]` as only an assignment will cause Svelte to update. [More Info](https://svelte.dev/tutorial/updating-arrays-and-objects).

Here an account is created and seed phrase is displayed to user. It is also added to the list of all accounts. The `Create Account` button runs `createAccount` function, giving value to `created` variable and thus displaying the address and seed phrase. Clicking the `Clear` button gives `created` a `null` value causing the address and seed phrase to disappear.

Do note the algosdk functions used. The `algosdk.generateAccount()` gives an object with `addr` (account address) and `sk` (secret key) values. The secret key is converted into a 24 letter seed phrase using `algosdk.secretKeyToMnemonic()`.

# 5. Restoring Account

```
// Script

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


<!-- Template -->

<h1>Restore Account</h1>
<input
  bind:value="{restoreMnemonic}"
  type="text"
  placeholder="Seed Phrase" /><button on:click="{restoreAccount}"
  >Restore</button>

```

The script section is to be added between the `<script></script>` tags under `createAccount` code. In template we ask the user to input their seed phrase and stores it in `restoreMnemonic`. `algosdk.mnemonicToSecretKey()` returns account object same as last one. All errors are stored in `restored.error` with `restored.status` of error. We also add it to the accounts list.

```
<!-- Template -->

{#if restored?.status === 'ok'}
  Account <code>{restored.addr}</code> restored successfully
  <button
    on:click="{() => {
      restored = null;
      restoreMnemonic = null;
    }}">Clear</button>
{:else if restored?.status === 'error'}
  Restore unsuccessful <code>{restored.error}</code>
  <button
    on:click="{() => {
      restored = null;
      restoreMnemonic = null;
    }}">Clear</button>
{/if}

```
The above template code displays restored address if `restored.status` is `ok` or displays the error code inside `restored.error` if status is `error`.

# 6. Account List

```
<!-- Template -->

<h1>Account List</h1>

{#each accounts as account}
  <label>
    <input type="radio" name="account" value="{account}" bind:group="{defaultAcc}" />
    <code>{account.addr}</code>
  </label>
{/each}

```
Here we list all the accounts using each svelte tag with a radio button infront of it. On clicking it, the `account` value is given to `defaultAcc`.

```
<h1>Account List</h1>
{#if defaultAcc}
  Default account:<code>{defaultAcc.addr}</code>
{:else if accounts.length}
  Select default account from below
{:else}
  Create or add accounts first
{/if}

```
This can be added before listing accounts under `h1` to prompt to select default or to display the default account.

To improve layout and display address in newlines add the following at bottom.
```
<style>
  label {
    display: block;
  }
</style>
```

## Checking Progress

Your `src/index.svelte` should look like this by now
```
<script>
  import algosdk from 'algosdk';

  const algodServer = 'https://testnet-algorand.api.purestake.io/ps2';
  const port = '';
  const token = { 'X-API-Key': 'YOUR API KEY' };
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

<style>
  label {
    display: block;
  }
</style>

```

# 7. Show Balance

This can be done automatically when a new account is made default through svelte's reactive statements.

```
// Script
  $: balance = defaultAcc
    ? client.accountInformation(defaultAcc.addr).do()
    : null;

```
This will update balance with a promise of account information from `client.accountInformation().do()` if a default account exists or will assign null.

```
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

```
The `{#if defaultAcc}` will create a refresh button that will do the same as the reactive statement. The `{#if balance}` will display balence by awaiting balence variable if it is not null.

When the balence promise is resolved, the `balence` is taken from the account information. All algo amount values is the sdk are in microAlgos so, here it is converted to algos to be displayed. The catch block as usual displays any errors while awaiting.

# 8. Create Transaction

```
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

```
Here we acquire values for Address, Note and Algos from user. Then on submit `createTxn()` is invoked which we will define.

```
const encode = (str) => new TextEncoder().encode(str);
  let  reciever, note, amount, transaction;
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

```
A brief owerview of this `createTxn()` function is that we first get 'suggested transaction parameters' and then make a payment transaction with those `suggestedParams` and from, to, amount, and note fields. We then sign the transaction with our secret keys and send the transaction to the bloackchain.

We wait for it to be confirmed using `algosdk.waitForConfirmation()` passing in our client, transaction id and the number of rounds to wait for our transaction to be verified (2 here).

In the code we display the result of transaction after awaiting it.

```
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

```

We then return the confirmed transaction and it's transaction id. For more info on transaction code check out this [tutorial]().

# 8. Searching Blockchain Using Indexer

```
// Script
  const filters = [
    ['acc', 'Account ID'],
    ['start', 'Start Date'],
    ['end', 'End Date'],
    ['txId', 'Transaction ID'],
    ['note', 'Note Field'],
  ];
  let selectedFilters = [];

<!-- Template -->

<h1>Search for Transactions</h1>
<h3>Filters</h3>
{#each filters as [value, title]}
  <label>
    <input type="checkbox" value="{value}" bind:group="{selectedFilters}" />
    {title}
  </label>
{/each}
```
This again uses Svelte's each tag to display checkboxes with input values from first value of nested array and the text beside checkbox from the second value of it.

```
// Script

let selectedValues = { acc: '', start: '', end: '', txId: '', note: '' };

<!-- Template -->

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

```
This shows additional input options based on the checkbox options ticked. It checks if the `selectedFilters` array has something and displays the additional option for it.

It stores the value of all the input valus for additional options in `selectedValues` object with a key for each additional option's value.

```
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

```

This is the search function that returns a promise on clicking `Select` button.

Here for seach filter we have selected a method chaining happens. For each value in `selectedFilters`, we chain a method to the initial `indexerClient.searchForTransactions()` we provide to the array reduce. For example if `selectedFilters` is `["acc","start","txId"]` then res will have the value of `indexerClient.searchForTransactions().address().afterTime().afterTime()`.

Also each filter has subparameters which are passed via `selectedValues` object like `selectedValues.txId` in `.txid(selectedValues.txId)`. The values of `afterTime` and `beforeTime` are datetime values that must be formated into rfc339 format. The `rfc3339` function does that.

Finally the transactions from this search function is returned which is awaited and displayed.
```
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

```

# 9. Final Codebase

The final code is avalable on [GitHub](https://github.com/NoelJacob/algorand-svelte). Check it out. Make issues or pull request if it needs it.
