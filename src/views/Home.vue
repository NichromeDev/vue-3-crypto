<template>
  <div class="home">
    <img alt="Vue logo" src="../assets/logo.png" />
    <template v-if="loading"> socket connecting... </template>
    <template v-else>
      <ul>
        <li v-for="(string, index) in strings" :key="index">
          <hr />
          {{ string }}
        </li>
      </ul>
    </template>
    <form @submit.prevent="submit">
      <input type="number" v-model="balanceToSend" />
      <button>
        <template v-if="paymentLoading">...loading</template>
        <template v-else>send coins from Alice to BOB</template>
      </button>
    </form>
  </div>
</template>

<script lang="ts" setup>
import { ApiPromise, WsProvider } from "@polkadot/api";
import { onMounted, onUnmounted, ref } from "vue";
import { Keyring } from "@polkadot/keyring";

// * constants
const Alice = "5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY";
const BOB = "5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty";

// * states
const loading = ref(false);
const paymentLoading = ref(false);

const strings = ref<string[]>([]);

const balanceToSend = ref(0);

// * vars
let api: ApiPromise;

// * local utils
const pushString = (string: string) =>
  (strings.value = [...strings.value, string]);

// * handlers
const submit = async () => {
  try {
    paymentLoading.value = true;
    const keyring = new Keyring({ type: "sr25519" });

    // Add Alice to our keyring with a hard-deived path (empty phrase, so uses dev)
    const alice = keyring.addFromUri("//Alice");

    // Create a extrinsic, transferring 12345 units to Bob
    const transfer = api.tx.balances.transfer(BOB, balanceToSend.value);

    // Sign and send the transaction using our account
    const hash = await transfer.signAndSend(alice);

    pushString(`Transfer sent with hash ${hash.toHex()}`);
    paymentLoading.value = false;
  } catch (e) {
    pushString(e);
    paymentLoading.value = false;
  }
};

// * lifecycle
onMounted(async () => {
  try {
    loading.value = true;
    const wsProvider = new WsProvider("wss://rpc.polkadot.io");
    api = await ApiPromise.create({ provider: wsProvider });

    let {
      data: { free: previousFree },
      nonce: previousNonce,
    } = await api.query.system.account(Alice);

    pushString(
      `${Alice} has a balance of ${previousFree}, nonce ${previousNonce}`
    );
    pushString(
      `You may leave this example running and start example 06 or transfer any value to ${Alice}`
    );

    // Here we subscribe to any balance changes and update the on-screen value
    api.query.system.account(
      Alice,
      ({ data: { free: currentFree }, nonce: currentNonce }) => {
        // Calculate the delta
        const change = currentFree.sub(previousFree);

        // Only display positive value changes (Since we are pulling `previous` above already,
        // the initial balance change will also be zero)

        const pushBalance = () =>
          pushString(`New balance change of ${change}, nonce ${currentNonce}`);

        pushBalance();
        if (!change.isZero()) {
          pushBalance();

          previousFree = currentFree;
          previousNonce = currentNonce;
        }
      }
    );

    loading.value = false;
  } catch (e) {
    pushString(e);
    loading.value = false;
  }
});

onUnmounted(() => api?.disconnect());
</script>
