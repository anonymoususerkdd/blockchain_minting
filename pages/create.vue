<template>
  <v-row>
    <v-col cols="12" md="6">
      <v-card>
        <v-card-title> Information </v-card-title>
        <v-select v-model="university" :items="university_items" label="University"></v-select>
        <v-card-text>
          <v-file-input
            v-model="input"
            accept="image/*"
            label="Image input"
          ></v-file-input>
          <v-text-field v-model="upperTitle" label="Name"></v-text-field>
          <v-text-field v-model="lowerTitle" label="Surname"></v-text-field>
          <v-select  v-model="rarity" :items="rarity_items" label="Rarity"></v-select>
          <v-text-field v-model="age" label="Age"></v-text-field>
          <v-text-field v-model="rank" label="Rank"></v-text-field>
          <v-text-field v-model="chair" label="Chair"></v-text-field>
          <v-text-field v-model="hindex" label="H-index"></v-text-field>
          <v-divider class="my-6"></v-divider>
        </v-card-text>
        <v-card-actions>
          <v-btn @click="create" color="primary"
            >Create Metacards on
            {{
              chainIdName[chainId] ? chainIdName[chainId] : 'Unknown'
            }}!</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-col>
    <v-col cols="12" md="6">
      <div id="capture" class="metacard-preview-container" :style="`background-image: url('/${university}.png');`">
        <img class="preview-image" :src="image" />
        <div class="name">{{ upperTitle }}</div>
        <div class="surname">{{ lowerTitle }}</div>
        <div class="age">{{ age }}</div>
        <div class="university">{{ university }}</div>
        <div class="rank">{{ rank }}</div>
        <div class="chair">{{ chair }}</div>
        <div class="hindex">{{ hindex }}</div>
      </div>
    </v-col>
    <v-dialog v-model="loading" persistent width="500px">
      <v-card dark>
        <v-card-title>{{
          !error ? 'Creating your Metacard...' : 'There was an error...'
        }}</v-card-title>
        <v-card-text>
          <v-timeline dense>
            <v-slide-x-reverse-transition group hide-on-leave>
              <v-timeline-item
                v-for="task in tasks.slice(0, currentTask).reverse()"
                :key="task.id"
                fill-dot
                color="white"
              >
                <template v-slot:icon>
                  <v-icon v-if="error" color="red" dark large>
                    mdi-exclamation-thick
                  </v-icon>
                  <v-progress-circular
                    v-else-if="task.id == currentTask"
                    indeterminate
                    color="primary"
                  ></v-progress-circular>

                  <v-icon v-else dark large color="green">
                    mdi-check-bold
                  </v-icon>
                </template>
                {{ task.description }}
              </v-timeline-item>
            </v-slide-x-reverse-transition>
          </v-timeline>
        </v-card-text>
        <v-card-actions>
          <v-btn v-if="error" @click="loading = false"> Close </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog v-model="success" width="500px">
      <v-card dark>
        <v-card-title>Your cards were created!</v-card-title>
        <v-card-text>
          <v-btn :to="`/${newLockAddress}`" nuxt> Mint your cards here </v-btn>
          <div class="py-1"></div>
          <v-btn
            href="https://app.unlock-protocol.com/dashboard"
            target="_blanc"
          >
            View your cards on Unlock Protocol</v-btn
          >
        </v-card-text>
        <v-card-actions>
          <v-btn @click="success = false" color="primary"> Close </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import { ethers } from 'ethers'
import html2canvas from 'html2canvas'
import { NFTStorage, File } from 'nft.storage'
import abis from '~/assets/abis'
import connectProvider from '~/services/provider'
import { unlockAddresses, chainIdName } from '~/assets/networks'

const Crypto = require('crypto')

const apiKey =
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJkaWQ6ZXRocjoweEE5MzRCNGZiNTREOGFCMDk5NzY4NzU5YjU5OWYwYWI5Mjc3NGYyOUMiLCJpc3MiOiJuZnQtc3RvcmFnZSIsImlhdCI6MTYyOTA0NzcyMzk0MSwibmFtZSI6ImFzZGZzYWYifQ.sFlkePbfV7ayYupL6JJER2utyQLY5UrSelHxWPHv7FM'
const client = new NFTStorage({ token: apiKey })

export default {
  data() {
    return {
      chainIdName,
      loading: false,
      success: false,
      input: null,
      upperTitle: '',
      lowerTitle: '',
      rarity: '',
      age: '',
      university: '',
      rank: '',
      chair: '',
      hindex: '',
      collectionName: '',
      collectionDescription: '',
      quantity: 1,
      price: 0.01,
      currentTask: 1,
      errorAtTask: 0,
      newLockAddress: '',
      provider: null,
      signer: null,
      chainId: 0,
      connected: false,
      university_items: ['TUM', 'LMU'],
      rarity_items: ['Common', 'Limited', 'Rare', 'Super rare', 'Unique'],
      rarityMapping: {"Common": 1, "Limited": 1000, 'Rare': 100, 'Super rare':10, 'Unique': 1},
      tasks: [
        {
          id: 1,
          description: 'Uploading your card to IPFS...',
        },
        {
          id: 2,
          description: 'Creating and uploading metadata...',
        },
        {
          id: 3,
          description: 'Creating your contract...',
        },
        {
          id: 4,
          description: 'Connecting contract to metadata...',
        },
      ],
    }
  },
  middleware: 'ethDetected',
  computed: {
    error() {
      return this.currentTask === this.errorAtTask
    },
    image() {
      if (!this.input) return null
      return URL.createObjectURL(this.input)
    },
  },

  mounted() {
    this.connect()
  },

  methods: {
    async connect() {
      this.connected = false
      const { provider, signer, chainId, updateOnChainChange } =
        await connectProvider()
      this.provider = provider
      this.signer = signer
      this.chainId = chainId
      this.connected = true
      updateOnChainChange()
    },

    errorHandler(e) {
      this.errorAtTask = this.currentTask
    },

    async create() {
      this.loading = true

      const canvas = await html2canvas(document.querySelector('#capture'), {
        allowTaint: true,
      })

      canvas.toBlob(async (blob) => {
        try {
          // Upload image ============
          const cidImage = await client.storeBlob(blob)
          console.log(`image cid ${cidImage}`)
          // =========================

          this.currentTask++

          // Upload metadata ==============

          const fileList = []

          // this.rarityMapping[this.rarity] -> quantity
          for (let i = 1; i <= this.rarityMapping[this.rarity]; i++) {
            const metadataFile = new File(
              [
                JSON.stringify({
                  name: `${this.collectionName} #${i}`,
                  description: this.description,
                  image: `ipfs://${cidImage}`,
                }),
              ],
              String(i) // file name
            )
            fileList.push(metadataFile)
          }
          console.log(`file list ${fileList}`)
          // TODO point to our IPFC cloud
          // mint card for all varieties
          // quanitity depending on class of rarity
          const cidBaseURI = await client.storeDirectory(fileList)

          console.log(`base URI ${cidBaseURI}`)
          // =========================

          this.currentTask++

          // create lock ================
          const unlockContract = new ethers.Contract(
            unlockAddresses[this.chainId],
            abis.Unlock.abi,
            this.provider
          )
          const unlockWithSigner = unlockContract.connect(this.signer)
          const txCreateLock = await unlockWithSigner.createLock(
            31536000, // expiration duration in seconds - now it's a year
            '0x0000000000000000000000000000000000000000', // ERC20 token address - 0 for Ether
            ethers.utils.parseEther(String(this.price)), // price
            this.rarityMapping[this.rarity], // number of keys
            this.collectionName, // name
            '0x' + Crypto.randomBytes(12).toString('hex') // user specific salt
          )
          const res = await txCreateLock.wait()
          const newLockAddress = res.events[0].args.newLockAddress
          this.newLockAddress = newLockAddress
          console.log(`new lock address ${newLockAddress}`)
          // ============================

          this.currentTask++

          // set base uri of lock ================
          const lockContract = new ethers.Contract(
            newLockAddress,
            abis.PublicLock.abi,
            this.provider
          )
          const lockWithSigner = lockContract.connect(this.signer)

          const txChangeURI = await lockWithSigner.setBaseTokenURI(
            `ipfs://${cidBaseURI}/`
          )
          console.log(txChangeURI)
          await txChangeURI.wait()
          // ===========================
          this.loading = false
          this.success = true
        } catch (error) {
          this.errorHandler(error)
        }
      })
    },
  },
}
</script>

<style scoped>
.metacard-preview-container {
  position: absolute;
  margin-top: 2rem;
  margin-left: 5rem;
  background-size: cover;
  background-color: white;
  border-radius: 1rem;
  width: 308px;
  height: 500px;
}

.preview-image {
  position: absolute;
  z-index: 2;
  /*left: 135px;*/
  top: 60px;
  border-radius: 10px;
  width: 308px;
  height: 300px;
}

.name {
  position: relative;
  left: 65px;
  top: 280px;
  font-weight: bold;
  text-align: center;
  z-index: 6;
  padding-top: 1px;
  color: white;
  font-size: 2em;
  width: 170px;
  height: 23px;
}
.surname {
  position: relative;
  left: 65px;
  top: 300px;
  z-index: 6;
  text-align: center;
  color: white;
  font-size: 1.9em;
  width: 170px;
  height: 23px;
}
.rarity {
  position: relative;
  left: 0px;
  top: 0px;
  z-index: 6;
  text-align: center;
  color: white;
  font-size: 1.9em;
  width: 170px;
  height: 23px;
}
.age {
  position: relative;
  left: 35px;
  top: 340px;
  z-index: 6;
  text-align: center;
  color: white;
  font-size: 1.4em;
  width: 10px;
  height: 3px;
}
.university {
  position: relative;
  left: 125px;
  top: 340px;
  z-index: 6;
  text-align: center;
  color: white;
  font-size: 1.4em;
  width: 10px;
  height: 3px;
}
.rank {
  position: relative;
  left: 240px;
  top: 340px;
  z-index: 6;
  text-align: center;
  color: white;
  font-size: 1.4em;
  width: 10px;
  height: 3px;
}
.chair {
  position: relative;
  left: 35px;
  top: 400px;
  z-index: 6;
  text-align: center;
  color: white;
  font-size: 1.4em;
  width: 10px;
  height: 3px;
}
.hindex {
  position: relative;
  left: 220px;
  top: 400px;
  z-index: 6;
  text-align: center;
  color: white;
  font-size: 1.4em;
  width: 10px;
  height: 3px;
}
</style>
