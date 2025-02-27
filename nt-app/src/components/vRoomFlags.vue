<template>
    <vModal ref="vFlagsModal">
        <h1 slot="header">Game Options</h1>
        <template slot="body" class="flags-body">
            <div class="presetContainer">
              (Beta)
              <v-button @click="loadPresets" :disabled="!isHost">Load Preset</v-button>
              <v-button @click="savePresets">Save as Preset</v-button>
            </div>
            <h2>
                <span>Death Penalty</span>
                <vTooltip>
                    <span>{{ tooltip("NT_death_penalty", true) }}</span>
                </vTooltip>
            </h2>
            <select
                class="slot-selector"
                :disabled="!isHost"
                v-model="tempFlags.NT_death_penalty.value"
            >
                <option
                    v-for="(value, key) in deathFlags"
                    :key="key"
                    :value="key"
                >
                    {{ value.name }}
                </option>
            </select>
            <h2>Run Options</h2>
            <div class="switches">
                <vSwitch
                    v-for="entry in booleanFlags"
                    :key="entry.id"
                    v-model="tempFlags[entry.id].value"
                    :disabled="!isHost"
                >
                    <span>{{ flagInfo(entry.id).name }}</span>
                    <vTooltip>
                        <span>{{ tooltip(entry.id, false) }}</span>
                    </vTooltip>
                </vSwitch>
            </div>

            <div class="value-inputs">
              <!--If !tempFlags.NT_sync_orbs.value, add a disabled flag-->
              <div
                  class="orb-count-sync-group"
                  v-if="tempFlags.NT_sync_orbs.value"
              >
                <h2>
                  <span>Orb Sync Count</span>
                  <vTooltip>
                    <span>{{ tooltip("NT_sync_orb_count", false) }}</span>
                  </vTooltip>
                </h2>
                <div class="orb-input">
                  <vInput
                      :disabled="!isHost || !tempFlags.NT_sync_orbs.value"
                      :min=0
                      type="number"
                      v-model.number="tempFlags.NT_sync_orb_count.value"
                      ref="orbInput"
                  ></vInput>
                </div>
              </div>
              <div class="world-seed-sync-group">
                <h2>
                  <span>World seed</span>
                  <vTooltip>
                    <span>{{ tooltip("NT_sync_world_seed", false) }}</span>
                  </vTooltip>
                </h2>
                <div class="world-seed">
                  <vInput
                      :disabled="!isHost"
                      :min=0
                      type="number"
                      v-model.number="tempFlags.NT_sync_world_seed.value"
                      ref="seedInput"
                  ></vInput>
                  <vButton :disabled="!isHost" @click="randomizeSeed"
                  >Random</vButton
                  >
                </div>
              </div>
            </div>
        </template>
        <div slot="footer" class="centered">
            <vButton @click="applyFlags">Apply</vButton>
            <vButton @click="close">Cancel</vButton>
        </div>
    </vModal>
</template>

<script>
import vSwitch from "../components/vSwitch.vue"
import vModal from "../components/vModal.vue"
import vButton from "../components/vButton.vue"
import vTooltip from "../components/vTooltip.vue"
import vInput from "../components/vInput.vue"
import { flagInfo } from "../store/index.js"
import { remote } from 'electron'
import fs from "fs";
const dialog = remote.dialog

export default {
    //braincells where'd ya go
    name: "vRoomFlags",
    components: {
        vButton,
        vModal,
        vInput,
        vSwitch,
        vTooltip
    },
    beforeMount() {
        // create a "working copy" of the flags - so that we can cancel without
        // affecting the store
        const obj = {}
        for (const flag of this.$store.getters.flags) {
            obj[flag.id] = Object.assign({}, flag)
        }
        this.tempFlags = obj
        this.gamemode = this.$store.getters.roomGamemode
    },
    data() {
        return {
            gamemode: null,
            tempFlags: null
        }
    },
    computed: {
        isHost() {
            return this.$store.getters.isHost
        },
        booleanFlags() {
            return Object.values(this.tempFlags).filter(
                (v) => v.type === "boolean"
            )
        },
        deathFlags() {
            return flagInfo[this.gamemode].NT_death_penalty
        }
    },
    methods: {
        flagInfo(id, val) {
            const gamemode = this.gamemode
            if (!Object.prototype.hasOwnProperty.call(flagInfo, gamemode)) {
                throw new Error(
                    "Invalid flagInfo call: gamemode " +
                        gamemode +
                        " not present"
                )
            }

            /** @type {import('../store/index.js').VueFlag|undefined} */
            const spec = this.tempFlags[id]
            if (!spec) {
                throw new Error(
                    "Invalid flagInfo call: id " + id + " not present"
                )
            }

            switch (spec.type) {
                case "string":
                case "number":
                case "boolean":
                    return flagInfo[gamemode][id]
                case "enum":
                    if (spec.choices.indexOf(val) === -1) {
                        throw new Error(
                            "Invalid flagInfo call: enum value " +
                                val +
                                " not present"
                        )
                    }
                    return flagInfo[gamemode][id][val]
            }
        },
        tooltip(id, isEnum) {
            if (!isEnum) {
                return this.flagInfo(id).tooltip
            }
            const enumValue = (this.tempFlags[id] || { value: null }).value
            return this.flagInfo(id, enumValue).tooltip
        },
        applyFlags() {
          //if tempFlags.NT_sync_orbs.value is false, set the orb count to 0
          if (!this.tempFlags.NT_sync_orbs.value) {
            this.tempFlags.NT_sync_orb_count.value = 0
          }
          this.$emit("applyFlags", Object.values(this.tempFlags))
        },
        loadPresets(){
          dialog.showOpenDialog({
            properties: ['openFile'],
            title: "Load NT Lobby Preset",
            filters: ["json"],
            buttonLabel: "Load"
          })
              .then((dialogReturn)=>{
                if(dialogReturn.canceled) return;
                // fileNames is an array that contains all the selected
                if(dialogReturn.filePaths === undefined || dialogReturn.filePaths.length === 0){
                  console.log("No file selected");
                  return;
                }
                const filePath = dialogReturn.filePaths[0]

                fs.readFile(filePath, 'utf-8', (err, data) => {
                  if(err){
                    alert("An error ocurred reading the file :" + err.message);
                    return;
                  }

                  // Change how to handle the file content
                  console.log("The file content is : " + data);
                  const oldSeed = this.tempFlags.NT_sync_world_seed.value
                  //TODO potential security issue? Validate this input
                  this.tempFlags = JSON.parse(data)
                  this.tempFlags.NT_sync_world_seed.value = oldSeed
                  this.$refs.orbInput.$refs.input.dispatchEvent(new Event("input"))
                  this.$refs.seedInput.$refs.input.dispatchEvent(new Event("input"))
                });
              });
        },
        savePresets(){
          dialog.showSaveDialog({
            properties: ['showOverwriteConfirmation'],
            title: "Save NT Lobby Preset",
            defaultPath: "nt-lobby-preset.json",
            buttonLabel: "Save"
          }).then((dialogReturn) => {
            if(dialogReturn.canceled) return;
            // fileNames is an array that contains all the selected
            if(dialogReturn.filePath === undefined || dialogReturn.filePath.length === 0){
              console.log("No file selected");
              return;
            }
            const filePath = dialogReturn.filePath
            fs.writeFile(filePath, JSON.stringify(this.tempFlags), {}, (err)=>{
              if(err) alert(err)
            })
          })
        },
        randomizeSeed() {
            const seed = Math.floor(Math.random() * 4294967295) + 1
            // not an amazing way to do it
            this.$refs.seedInput.$refs.input.value = seed
            this.$refs.seedInput.$refs.input.dispatchEvent(new Event("input"))
        },
        close() {
            this.$emit("close")
        }
    }
}
</script>

<style>
.flags-body {
    display: flex;
    flex-flow: row wrap;
    width: 100%;
}

.presetContainer{
  width: 100%;
  display: flex;
  flex-flow: row-reverse;
  justify-content: end;
  gap: 8px;
}

.switches {
    display: flex;
    flex-flow: row wrap;
}

.switches > div {
    padding: 0.2em;
    min-width: 180px;
}

.value-inputs{
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.orb-count-sync-group{

}

//if the sync orbs flag is not enabled, disable the group
.orb-count-sync-group[aria-disabled="true"]{
  display: none;
}

.world-seed-sync-group{

}

.orb-count{
  display: flex;
  width: 100%;
}

.world-seed {
    display: flex;
    width: 100%;
}

.world-seed .labeled-input {
    margin-top: auto;
    padding-bottom: 0;
    margin-right: 0;
}
span + .tooltip-wrapper {
    margin-left: 0.5em;
}
</style>
