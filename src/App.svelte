<script>
  let multisig = require("@iota/multisig");
  let converter = require("@iota/converter");
  let txconverter = require("@iota/transaction-converter");
  let bundle_validator = require("@iota/bundle-validator");
  let {
    channelRoot,
    createChannel,
    createMessage,
    parseMessage,
    mamAttach,
    mamFetch,
    mamFetchAll
  } = require("@iota/mam.js");
  let { composeAPI } = require("@iota/core");

  let api = composeAPI({ provider: "https://altnodes.devnet.iota.org:443" });
  let mode = "private";
  let generatedAddress = "no address";
  let account = 0;
  function changeAccount() {
    if (account == 1) {
      account = 0;
    } else {
      account = 1;
    }
  }
  let walletStatus = "Idle";
  //start with -1 because it gets increased before address generation
  let index = -1;

  let seed =
    "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA";
  let mamSeed =
    "TGCEJGAOUFRKJANOWUJOURJTLVANSGCJRTNRCTISSNTESNGFDKJFATAEJGNOVNUKJAQIWUJOUQJXLKANS";

  let addressGenerationRequest = false;
  let addressGenerated = false;
  let addresses = [];
  let channelState = createChannel(mamSeed, 2, mode);
  let balance = 0;
  let payoutAddress = "";
  let payoutValue = 0;
  let transactionApprovalRequest = false;
  let transactionSent = false;
  let transactionHash = "";
  let mwm = 9;

  let initChannelState = createChannel(mamSeed, 2, mode);
  let initialRoot = channelRoot(initChannelState);
  let latestRoot = initialRoot;
  console.log("initialRoot", initialRoot);
  let mamRoot = "";
  let lastFetchedMessage;

  setInterval(() => fetchMam(), 10000);
  let fetchMAMrunning = false;
  async function fetchMam() {
    if (fetchMAMrunning) {
      console.log("fetchMAM runs already");
      return;
    }
    fetchMAMrunning = true;
    await fetchLatestMam();
    if (typeof lastFetchedMessage == "undefined") {
      fetchMAMrunning = false;
      return;
    }
    index = lastFetchedMessage.index;
    if (lastFetchedMessage.type == "addressRequest" && account == 1) {
      walletStatus = "Waiting for address generation approval";
      addressGenerationRequest = true;
    }
    if (lastFetchedMessage.account == account) {
      console.log("Waiting for the other account");
    }
    if (lastFetchedMessage.type == "payoutRequest" && account == 1) {
      walletStatus = "Waiting for transaction approval";
      transactionApprovalRequest = true;
    }
    if (lastFetchedMessage.type == "payoutResponse" && account == 0) {
      transactionApprovalRequest = false
      transactionSent = true;
      transactionHash = lastFetchedMessage.txHash;
    }
    fetchMAMrunning = false;
  }

  function fetchLatestMam() {
    return new Promise(async (resolve, reject) => {
      console.log("Fetching from tangle, please wait...");
      let fetchedMessages = [];
      let fetched = "";
      while (typeof fetched != "undefined") {
        fetched = await mamFetch(api, latestRoot, "private");
        if (fetched) {
          console.log("fetched", fetched);
          let fetchedMessage = JSON.parse(
            converter.trytesToAscii(fetched.message)
          );
          console.log("Fetchedmessage", fetchedMessage);
          latestRoot = fetched.nextRoot;
          fetchedMessages.push(fetched.message);
          //update last fetched message
          lastFetchedMessage = fetchedMessage;
          //add address to array
          if (lastFetchedMessage.type == "address") {
            addressGenerated = true;
            addresses = addresses.concat(lastFetchedMessage.address);
            console.log("addresses", addresses);
          }
          //update channel state
          channelState = fetchedMessage.channelState;
        } else {
          console.log("No tx found with root:", latestRoot);
          resolve(fetchedMessages);
        }
      }
    });
  }

  async function generateAddressRequest() {
    try {
      await fetchMam();
      if (typeof lastFetchedMessage != "undefined") {
        if (lastFetchedMessage.account == account) {
          console.log("You need to wait for the other account");
          return;
        }
      }
      //increase index to get a new address
      index++;
      let digest = multisig.getDigest(seed, index, 2);
      let message = {
        type: "addressRequest",
        index,
        digest
      };
      walletStatus = "Sending addressRequest to MAM";
      await sendMessageToMAM(message);
      walletStatus = "Waiting for address response";
    } catch (e) {
      console.log(e);
    }
  }

  async function generateAddress() {
    try {
      index = lastFetchedMessage.index;
      console.log("index", index);
      let digest = multisig.getDigest(seed, index, 2);
      let multisigaddress = multisig.getAddress([
        lastFetchedMessage.digest,
        digest
      ]);
      console.log("generated address:", multisigaddress);
      generatedAddress = multisigaddress;
      let message = {
        type: "address",
        index,
        digest: digest,
        address: multisigaddress
      };
      walletStatus = "Sending address response to MAM";
      await sendMessageToMAM(message);
      //reset state
      walletStatus = "Idle";
      addressGenerationRequest = false;
    } catch (e) {
      console.log(e);
    }
  }
  function sendMessageToMAM(messageObject) {
    return new Promise(async (resolve, reject) => {
      //copy channel state and create future channel state
      let copiedChannelState = JSON.parse(JSON.stringify(channelState));
      copiedChannelState.start++;
      //add future channel state
      messageObject.channelState = copiedChannelState;
      //add account number
      messageObject.account = account;
      let finalMamMessage = createMessage(
        channelState,
        converter.asciiToTrytes(JSON.stringify(messageObject))
      );
      let tx = await mamAttach(api, finalMamMessage, 3, mwm, "INIT");
      console.log("https://devnet.thetangle.org/transaction/" + tx[0].hash);
      resolve(tx);
    });
  }

  function getBalance() {
    if (addresses.length == 0) {
      return;
    }
    api
      .getBalances(addresses, 100)
      .then(({ balances }) => {
        balance = balances.reduce((a, b) => {
          return a + b;
        });
        console.log("Balance: " + balance + "i");
      })
      .catch(err => {
        console.log(err);
      });
  }

  async function sendTransactionRequest() {
    walletStatus = "Send transaction Request";
    console.log("addresses", addresses);
    let balanceInfo = await api.getBalances(addresses, 100);
    console.log("balanceInfo", balanceInfo);
    let totalBalance = balanceInfo.balances.reduce((a, b) => {
      return a + b;
    });
    let inputAddress = "";
    let inputIndex = 0;
    for(let index = 0; index < balanceInfo.balances.length; index++){
    // for ([index, value] of balanceInfo.balances.entries()) {
      if (balanceInfo.balances[index] > 0) {
        inputAddress = addresses[index];
        inputIndex = index;
      }
    }
    if (payoutValue > totalBalance) {
      console.log("Not enough value");
      walletStatus = "Idle";
      return;
    }

    if (payoutValue != totalBalance) {
      console.log("Request remainder address");
      //todo
      return;
    }
    let transfers = [{ address: payoutAddress, value: payoutValue }];
    let multisigInput = {
      address: inputAddress,
      balance: totalBalance,
      securitySum: 4
    };
    let initBundle = multisig.createBundle(multisigInput, transfers);
    let firstSigned = multisig.addSignature(
      initBundle,
      inputAddress,
      multisig.getKey(seed, inputIndex, 2)
    );
    let message = {
      type: "payoutRequest",
      inputAddress,
      outputAddress: payoutAddress,
      value: payoutValue,
      bundle: firstSigned,
      index,
      inputIndex
    };
    walletStatus = "Sending address response to MAM";
    await sendMessageToMAM(message);
    walletStatus = "Waiting for transaction response";
  }

  async function approvePayout() {
    walletStatus = "Sending transaction";
    let secondSigned = multisig.addSignature(
      lastFetchedMessage.bundle,
      lastFetchedMessage.inputAddress,
      // multisig.getKey(seed, lastFetchedMessage.inputIndex, 2)
      multisig.getKey(seed, 0, 2)
    );
    let finaltxobjs = secondSigned.map(tx =>
      txconverter.asTransactionObject(tx)
    );
    console.log(finaltxobjs);
    let validBundle = bundle_validator.default(finaltxobjs);
    console.log("valid bundle:", bundle_validator.default(finaltxobjs));
    if (validBundle == false) {
      return;
    }
    let bundle = await api.sendTrytes(secondSigned, 3, mwm);
    transactionSent = true;
    transactionHash = bundle[0].hash;
    console.log(
      "Transaction sent: https://thetangle.org/transaction/" + transactionHash
    );
    //Raw trytes should be sent back
    let message = {
      type: "payoutResponse",
      index,
      txHash: transactionHash
    };
    await sendMessageToMAM(message);
    transactionApprovalRequest = false
    walletStatus = "Idle";
  }
</script>

<style>
  pre {
    color: rgb(182, 28, 182);
  }
  input {
    display: block;
    width: 900px;
    max-width: 100%;
  }
  :global(body) {
    background: #272727;
    color: white;
  }
</style>

<h1>MAM-Multisig-Wallet PoC</h1>
<button on:click={changeAccount}>Account: {account}</button>
<span>Multisig seed:</span>
<input bind:value={seed} placeholder="SEED" />
<span>MAM seed:</span>
<input bind:value={mamSeed} placeholder="MAM SEED" />

<button on:click={fetchMam}>Fetch MAM</button>
<input bind:value={mamRoot} placeholder="MAM ROOT" />

<div>Balance: {balance}</div>
<button on:click={getBalance}>Get balance</button>
<div>Wallet Status: {walletStatus}</div>

<pre>Multisig</pre>
<div>{generatedAddress}</div>
{#if account == 0}
  <button on:click={generateAddressRequest}>Generate address</button>
{/if}

{#if account == 1 && addressGenerationRequest}
  <div>Address generation request</div>
  <button on:click={generateAddress}>Generate address</button>
{/if}

{#if addressGenerated}
  <div>Generated addresses:</div>
  {#each addresses as address, index}
    <div>Index {index}: {address}</div>
  {/each}
{/if}

{#if transactionSent}
  <div>Transaction sent:</div>
  <a href="https://devnet.thetangle.org/transaction/{transactionHash}">
    https://devnet.thetangle.org/transaction/{transactionHash}
  </a>
{/if}

{#if transactionApprovalRequest}
  <div>Transaction request:</div>
  <div>
    Send {lastFetchedMessage.value} from {lastFetchedMessage.inputAddress} to {lastFetchedMessage.outputAddress}
  </div>
  <button on:click={approvePayout}>Approve transaction</button>
{/if}

{#if account == 0}
<div>Send:</div>
<input bind:value={payoutAddress} placeholder="address" />
<input bind:value={payoutValue} placeholder="value" />
<button on:click={sendTransactionRequest}>Send transaction</button>
{/if}