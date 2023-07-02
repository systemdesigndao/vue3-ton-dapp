<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { onMounted } from 'vue'
import {TonConnect} from '@tonconnect/sdk';
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
  // const rawAddress = store.entity.connector.wallet.account.address; // like '0:abcdef123456789...'
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

onMounted(() => {
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

const { wallet, connecting } = storeToRefs(store)
</script>

<template>
  <div v-if="wallet === undefined">
    <button  @click="connect('tonkeeper')">Connect tonkeeper</button>
    <button  @click="connect('tonhub')">Connect tonhub</button>
    <QRCodeStyling v-if="connecting?.link" :text="connecting.link" />
  </div>
  <div v-else>
    <pre>{{wallet}}</pre>
  </div>
</template>
