<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { onMounted, watch } from 'vue'
import {TonConnect} from '@tonconnect/sdk'
import QRCodeStyling from './QRCodeStyling.vue';
import { ConnectedWalletFromAPI, useWalletStore } from '../stores'

const store = useWalletStore()

const walletConnectionSource = {
  tonkeeper: {
    universalLink: 'https://app.tonkeeper.com/ton-connect',
    bridgeUrl: 'https://bridge.tonapi.io/bridge',
  },
  tonhub: {
    universalLink: "https://tonhub.com/ton-connect",
    bridgeUrl: "https://connect.tonhubapi.com/tonconnect",
  },
};

async function postData(url = "", data = {}) {
  const response = await fetch(url, {
    method: "POST",
    cache: "no-cache",
    body: JSON.stringify(data),
  });
  return response;
}


const connect = async (wallets: 'tonkeeper' | 'tonhub') => {
  // console.log(new Address(rawAddress).toString('base64', { bounceable: true }));
  const d = await postData('https://ton-dapp-backend.systemdesigndao.xyz/ton-proof/generatePayload');
  const { payload } = await d.json();

  if (store?.entity?.connected) {
    await store.entity.disconnect();
  }
    
  const link = store.entity?.connect(walletConnectionSource[wallets], { tonProof: payload });

  store.setConnecting({
    link,
  })
}

const sendTx = async () => {
  const transaction = {
      validUntil: Date.now() + 1000000,
      messages: [
          {
            address: store.wallet?.address.raw!,
            amount: "60000000", 
          }
      ]
  }

  try {
      await store.entity?.sendTransaction(transaction);

      alert('Transaction was sent successfully');
  } catch (e) {
      console.error(e);
  }
}

const disconnect = async () => {
  if (store?.entity?.connected) {
      await store?.entity?.disconnect();
      store.setWallet(undefined);
      localStorage.removeItem('wallet');
  }
}


onMounted(async () => {  
  const tonConnect = new TonConnect({
    manifestUrl: 'https://about.systemdesigndao.xyz/ton-connect.manifest.json',
  });

  store.setEntity(tonConnect);

  const unsubscribe = tonConnect.onStatusChange(async wallet => {
			if (!wallet) {
				return;
			}

			const tonProof = wallet.connectItems?.tonProof;

			if (tonProof) {
				if ('proof' in tonProof) {
          const obj = {
              proof: {
                ...tonProof.proof,
                state_init: wallet.account.walletStateInit,
              },
              network: wallet.account.chain,
              address: wallet.account.address
          };
          try {
            const { token } = await (await postData('https://ton-dapp-backend.systemdesigndao.xyz/ton-proof/checkProof', obj)).json();
            const data = await (await fetch(`https://ton-dapp-backend.systemdesigndao.xyz/dapp/getAccountInfo?network=${obj.network}`, {
                    headers: {
                      Authorization: `Bearer ${token}`,
                      'Content-Type': 'application/json',
                    }
            })).json() as ConnectedWalletFromAPI;

            localStorage.setItem('wallet', JSON.stringify(data));

            store.setWallet(data);
          } catch (err) {
            console.error(err);
          }
					return;
				}

				console.error(tonProof.error);
			}
	});

  return unsubscribe;
})

watch(() => store?.entity?.connected, async () => {
  if (store?.entity?.connected === false) {
    store.entity.restoreConnection();
  }

  if (store?.entity?.connected === true) {
    const localWallet = localStorage.getItem('wallet');
    if (localWallet) {
      const data = JSON.parse(localWallet);
      store.setWallet(data);
      store.setLoading(false);
    }
  }
})

const { wallet, connecting, loading } = storeToRefs(store)
</script>

<template>
  <div v-if="loading === false">
    <div v-if="wallet === undefined" class="p-1">
      <div class="flex justify-center">
        <button class=" bg-main-light-4 w-40 h-10 rounded-full font-sans border-none" @click="connect('tonkeeper')"><span class="text-white-1">Connect tonkeeper</span></button>
        <button class=" bg-main-light-4 w-40 h-10 rounded-full font-sans border-none ml-1" @click="connect('tonhub')"><span class="text-white-1">Connect tonhub</span></button>
      </div>
      <div class="flex justify-center">
        <QRCodeStyling v-if="connecting?.link" :text="connecting.link" />
      </div>
    </div>
    <div v-else class="flex flex-col">
      <pre>{{wallet}}</pre>
      <button @click="sendTx">Send tx</button>
      <button @click="disconnect">Disconnect</button>
    </div>
  </div>
  <div v-else class="flex justify-center items-center h-screen">
    loading wallet state...
  </div>
</template>
