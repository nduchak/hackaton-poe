<template>
  <div class="w-full p-4 flex flex-col">
    <h1 class="mb-4">Your Aepp</h1>

    <div class="border">
      <div class="bg-green w-full flex flex-row font-mono border border-b">
        <div class="p-2 w-1/4">
          Public Key <small>(from Wallet Aepp)</small>
        </div>
        <div v-if="addressResponse" class="p-2 w-3/4 bg-grey-lightest break-words">
          {{addressResponse | responseToString}}
        </div>
        <div v-else class="p-2 w-3/4 bg-grey-lightest break-words text-grey">
          Requesting Public Key from AE Wallet...
        </div>
      </div>
      <div v-if="heightResponse" class="bg-green w-full flex flex-row font-mono border border-b">
        <div class="p-2 w-1/4">
          Height
        </div>
        <div class="p-2 w-3/4 bg-grey-lightest">
          {{heightResponse | responseToString}}
        </div>
      </div>

      <template v-if="nodeInfoResponse">
        <div v-if="nodeInfoResponse.error" class="bg-green w-full flex flex-row font-mono border border-b">
          <div class="p-2 w-1/4">
            NodeInfo error
          </div>
          <div class="p-2 w-3/4 bg-grey-lightest break-words">
            {{nodeInfoResponse.error}}
          </div>
        </div>
        <div
          v-for="(value, name) in nodeInfoResponse.result"
          v-if="['url', 'name', 'nodeNetworkId', 'version'].includes(name)"
          class="bg-green w-full flex flex-row font-mono border border-b"
        >
          <div class="p-2 w-1/4 capitalize">
            {{name.replace('nodeNetworkId', 'NetworkId')}}
          </div>
          <div class="p-2 w-3/4 bg-grey-lightest">
            {{value}}
          </div>
        </div>
      </template>

      <div v-if="compilerVersionResponse" class="bg-green w-full flex flex-row font-mono border border-b">
        <div class="p-2 w-1/4">
          Compiler version
        </div>
        <div class="p-2 w-3/4 bg-grey-lightest">
          {{compilerVersionResponse | responseToString}}
        </div>
      </div>

    </div>

    <h2 class="mt-4">Claim nickname for games</h2>
    <h3 v-if="claimedName"  class="mt-4">Current name:{{ claimedName.tx.decodedPayload.data.nickname }} </h3>

    <div class="border mt-4 rounded">
      <div class="bg-grey-lightest w-full flex flex-row font-mono">
        <div class="p-2 w-1/4">
          Your nickname
        </div>
        <div class="p-2 w-3/4 bg-white break-words">
          <input
            class="bg-black text-white border-b border-black p-2 w-full"
            v-model="spendPayload"
          />
        </div>
      </div>
      <button
        v-if="client"
        class="w-32 rounded rounded-full bg-purple text-white py-2 px-4 pin-r mr-8 mt-4 text-xs"
        @click="createNameForGames"
      >
        Create
      </button>
    </div>

    <h2 class="mt-4">Use for KYC third-party</h2>
    <h3 class="mt-4">This allow User to create an id for KYC third party </h3>
<!--    <h3 v-if="claimedName"  class="mt-4">Current name:{{ claimedName.tx.decodedPayload.data.nickname }} </h3>-->

    <div class="border mt-4 rounded">
      <div class="bg-grey-lightest w-full flex flex-row font-mono">
        <div class="p-2 w-1/4">
          Third-Party Account ID
        </div>
        <div class="p-2 w-3/4 bg-white break-words">
          <input
            class="bg-black text-white border-b border-black p-2 w-full"
            v-model="kycAddress"
          />
        </div>
      </div>
      <button
        v-if="client"
        class="w-32 rounded rounded-full bg-purple text-white py-2 px-4 pin-r mr-8 mt-4 text-xs"
        @click="createKYCProof"
      >
        Create
      </button>
    </div>

    <div v-if="spendResponse" class="border mt-4 mb-8 rounded">
      <div class="bg-green w-full flex flex-row font-mono border border-b">
        <div class="p-2 w-1/4">
          Send result
        </div>
        <div
          class="p-2 w-3/4 bg-grey-lightest break-words whitespace-pre-wrap"
        >{{ spendResponse | responseToFormattedJSON }}</div>
      </div>
    </div>

  </div>
</template>

<script>
  //  is a webpack alias present in webpack.config.js
  import { Aepp } from '@aeternity/aepp-sdk/es'
  import * as Crypto  from '@aeternity/aepp-sdk/es/utils/crypto'
  import Http from '@aeternity/aepp-sdk/es/utils/http'

  const GAME_NAME_TAG = 1
  const KYC_TAG = 2
  const PROOF_TAG = 3
  const errorAsField = async fn => {
    try {
      return { result: await fn }
    } catch (error) {
      return { error }
    }
  }

  export default {
    data () {
      return {
        runningInFrame: window.parent !== window,
        client: null,
        mdwUrl: 'https://testnet.mdw.aepps.com/',
        mdwAPI: null,
        zeroTransactions: [],
        claimedName: null,
        addressResponse: null,
        kycResponse: null,
        kycAddress: null,
        heightResponse: null,
        compilerVersionResponse: null,
        nodeInfoResponse: null,
        spendTo: null,
        spendAmount: null,
        spendPayload: null,
        spendResponse: null,
        contractCode: `contract Identity =
      entrypoint main(x : int) = x`,
        compileBytecodeResponse: null,
        contractInitState: [],
        deployResponse: null,
        callResponse: null
      }
    },
    filters: {
      responseToString: response => `${response.error ? 'Error: ' : ''}${response.result || response.error}`,
      responseToFormattedJSON: response => response.error
        ? `Error: ${response.error}`
        : JSON.stringify(response.result, null, 4),
    },
    methods: {
      async sendProof (data) {
        const address = this.addressResponse.result
        this.spendResponse = null
        this.spendResponse = await errorAsField(this.client.spend(
          0,
          address, {
            payload: this.serializePayload(data)
          }
        ));
      },
      async createNameForGames () {
        const nickname = this.spendPayload
        const data = { tag: GAME_NAME_TAG, data: { nickname } }
        await this.sendProof(data)
        this.claimedName = {
          ...this.spendResponse.result,
          ... { tx: { decodedPayload: this.deserializePayload(this.spendResponse.result.tx.payload), ...this.spendResponse.result.tx } } }
      },
      async createKYCProof () {
        await this.sendProof({ tag: KYC_TAG, data: { id: this.kycAddress }})
      },
      async createProof (hashOfSomething) {
        await this.sendProof({ tag: PROOF_TAG, data: { hash: hashOfSomething }})
      },
      async getZeroAmountTransactions (address) {
        const { transactions } = await this.mdwAPI.get(`/middleware/transactions/account/${address}/to/${address}`)
        return transactions.reduce(
          (acc, transaction) => {
            const { tx } = transaction
            if (tx.type !== 'SpendTx' || tx.amount !== 0) return acc
            try {
              transaction.tx.decodedPayload = this.deserializePayload(tx.payload)
              return [...acc, transaction]
            } catch (e) {
              return acc
            }
          }, [])
      },
      async getUserNickName (address) {
        return (await this.getZeroAmountTransactions(address))
          .filter(tx => tx.tx.decodedPayload.tag === GAME_NAME_TAG)
          .sort((a, b) => a.block_height === b.blockHeight ? 0 : a.block_height > b.blockHeight ? -1 : 1)[0]
      },
      // @Todo compress data
      serializePayload ({ tag, data }) {
        return JSON.stringify({ tag, data })
      },
      deserializePayload (compressedData) {
        return JSON.parse(Crypto.decodeBase64Check(compressedData.split('_')[1]).toString())
      },
      async getReverseWindow() {
        const iframe = document.createElement('iframe')
        iframe.src = prompt('Enter wallet URL', 'http://localhost:9000')
        iframe.style.display = 'none'
        document.body.appendChild(iframe)
        await new Promise(resolve => {
          const handler = ({ data }) => {
            if (data.method !== 'ready') return
            window.removeEventListener('message', handler)
            resolve()
          }
          window.addEventListener('message', handler)
        })
        return iframe.contentWindow
      }
    },
    async created () {
      this.client = await Aepp({
        parent: this.runningInFrame ? window.parent : await this.getReverseWindow()
      })
      this.addressResponse = await errorAsField(this.client.address())
      this.heightResponse = await errorAsField(this.client.height())
      this.nodeInfoResponse = await errorAsField(this.client.getNodeInfo())
      this.mdwAPI = Http({ baseUrl: this.mdwUrl })
      this.claimedName = await this.getUserNickName(this.addressResponse.result)
    }
  }
</script>
