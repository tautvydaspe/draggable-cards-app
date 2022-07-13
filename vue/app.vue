<template>
  <div>
    <form v-if="canBeLoaded === true">
      <div v-for="card in cardsData" :key="card.place">
        <input
          type="checkbox"
          :value="card.isVisible"
          v-model="card.isVisible"
          @change="select"
        />
        <label>
          {{ card.title }}
        </label>
      </div>
    </form>
    <draggable
      v-if="canBeLoaded === true"
      v-model="cardsData"
      @start="drag = true"
      @end="drag = false"
      @change="setNewOrder"
      :key="rerender"
    >
      <card
        class="card"
        v-for="card in cardsData"
        v-show="card.isVisible === true"
        :key="card.place"
        :data="card"
      ></card>
    </draggable>
  </div>
</template>

<script>
import draggable from 'vuedraggable'
import card from '../components/CardDraggable.vue'

export default {
  components: {
    draggable,
    card
  },
  data () {
    return {
      sysinfo: [],
      lastCPUTime: null,
      cpuPercentage: 100,
      memPercentage: 100,
      interfaces: [],
      syslog: [],
      dmesg: [],
      cardsData: [],
      configData: [],
      canBeLoaded: false,
      rerender: true,
      isFirstLoad: true
    }
  },
  timers: {
    updateSystemData: { time: 3000, autostart: true, immediate: true, repeat: true },
    loadConfigData: { time: 1000, autostart: true, immediate: true, repeat: true },
    dataSet: { time: 500, autostart: true },
    deleteExcessiveInterface: { time: 800, autostart: true },
    afterDataSet: { time: 1200, autostart: true }
  },
  methods: {
    // System data update (CPU usage, Memory usage, Firmware Version, Local Time, Uptime)
    updateSystemData () {
      this.$system.getInfo().then(({ release, localtime, uptime, memory }) => {
        this.sysinfo = [
          [this.$t('home.Firmware Version'), release.revision],
          [this.$t('home.Local Time'), new Date(localtime * 1000).toString()],
          [this.$t('Uptime'), '%t'.format(uptime)]
        ]
        this.memPercentage = Math.floor(
          ((memory.total - memory.free) / memory.total) * 100
        )
      })
      this.$rpc.call('system', 'cpu_time').then((times) => {
        if (!this.lastCPUTime) {
          this.cpuPercentage = 0
          this.lastCPUTime = times
          return
        }
        const idle1 = this.lastCPUTime[3]
        const idle2 = times[3]
        let total1 = 0
        let total2 = 0
        this.lastCPUTime.forEach((t) => {
          total1 += t
        })
        times.forEach((t) => {
          total2 += t
        })
        this.cpuPercentage = Math.floor(
          ((total2 - total1 - (idle2 - idle1)) / (total2 - total1)) * 100
        )
        this.lastCPUTime = times
      })
    },
    // Method to load config data
    loadConfigData () {
      if (this.isFirstLoad) {
        this.$spin(true)
        this.isFirstLoad = false
      }
      this.$uci.load('draggable').then(() => {
        this.configData = this.$uci.sections('draggable', 'draggable')
      })
    },
    systemInfoSet () {
      this.cardsData[0] = {
        title: 'System',
        place: '0',
        isVisible: '1',
        name: [
          this.sysinfo[2][0],
          this.sysinfo[1][0],
          this.sysinfo[0][0],
          'Memory usage',
          'CPU usage'
        ],
        data: [
          this.sysinfo[2][1],
          this.sysinfo[1][1],
          this.sysinfo[0][1],
          this.memPercentage,
          this.cpuPercentage
        ]
      }
    },
    systemEventsInfoSet () {
      this.cardsData[Object.keys(this.cardsData).length] = {
        title: 'Recent system events',
        place: '1',
        isVisible: '0',
        name: [
          this.syslog[this.syslog.length - 1].datetime,
          this.syslog[this.syslog.length - 2].datetime,
          this.syslog[this.syslog.length - 3].datetime,
          this.syslog[this.syslog.length - 4].datetime,
          this.syslog[this.syslog.length - 5].datetime
        ],
        data: [
          this.syslog[this.syslog.length - 1].msg,
          this.syslog[this.syslog.length - 2].msg,
          this.syslog[this.syslog.length - 3].msg,
          this.syslog[this.syslog.length - 4].msg,
          this.syslog[this.syslog.length - 5].msg
        ]
      }
    },
    networkEventsInfoSet () {
      this.cardsData[Object.keys(this.cardsData).length] = {
        title: 'Recent network events',
        place: '2',
        isVisible: '1',
        name: [
          this.dmesg[this.dmesg.length - 1].time,
          this.dmesg[this.dmesg.length - 2].time,
          this.dmesg[this.dmesg.length - 3].time,
          this.dmesg[this.dmesg.length - 4].time,
          this.dmesg[this.dmesg.length - 5].time
        ],
        data: [
          this.dmesg[this.dmesg.length - 1].msg,
          this.dmesg[this.dmesg.length - 2].msg,
          this.dmesg[this.dmesg.length - 3].msg,
          this.dmesg[this.dmesg.length - 4].msg,
          this.dmesg[this.dmesg.length - 5].msg
        ]
      }
    },
    interfacesInfoSet () {
      // Reading data from all interfaces displayed in Network/Interfaces
      for (let i = 0; i < this.interfaces.length; i++) {
        if (
          this.interfaces[i].name &&
          this.interfaces[i].getIPv4Addrs().length !== 0
        ) {
          this.cardsData[Object.keys(this.cardsData).length] = {
            title: this.interfaces[i].name.toUpperCase(),
            place: '',
            isVisible: false,
            name: ['type', 'ip address'],
            data: [
              this.interfaces[i].name,
              this.interfaces[i].getIPv4Addrs().join(',')
            ]
          }
        } else if (
          this.interfaces[i].name &&
          this.interfaces[i].getIPv4Addrs().length === 0
        ) {
          this.cardsData[Object.keys(this.cardsData).length] = {
            title: this.interfaces[i].name.toUpperCase(),
            place: '',
            isVisible: false,
            name: ['type', 'ip address'],
            data: [this.interfaces[i].name, '-']
          }
        }
      }
    },
    dataSet () {
      this.systemInfoSet()

      this.systemEventsInfoSet()

      this.networkEventsInfoSet()

      this.interfacesInfoSet()

      // newInterfacePlacePlus is used when there are multiple new interfaces
      let newInterfacePlacePlus = 0

      for (let i = 0; i < this.cardsData.length; i++) {
        for (let j = 0; j < this.configData.length; j++) {
          if (this.cardsData[i].title === this.configData[j].name) {
            this.cardsData[i].place = this.configData[j].place
            if (this.configData[j].isVisible === '1') {
              this.cardsData[i].isVisible = true
              break
            } else {
              this.cardsData[i].isVisible = false
              break
            }
          }
          if (j === this.configData.length - 1) {
            this.cardsData[i].place =
              this.configData.length.toString() + newInterfacePlacePlus
            this.addNewInterface(
              this.cardsData[i].title,
              newInterfacePlacePlus
            )
            newInterfacePlacePlus++
            break
          }
        }
      }
      this.cardsData.sort((a, b) => a.place - b.place)
    },
    afterDataSet () {
      this.canBeLoaded = true
      this.$spin(false)
    },
    // Triggered, when drag and drop action is executed
    setNewOrder () {
      // New card places is set to config
      for (let i = 0; i < this.configData.length; i++) {
        for (let j = 0; j < this.cardsData.length; j++) {
          if (this.cardsData[j].title === this.configData[i].name && this.configData[i].place !== j.toString()
          ) {
            this.$uci.set('draggable', this.configData[i]['.name'], 'place', j.toString())
            break
          }
        }
      }
      this.$uci.save().then(() => {
        this.$uci.apply().then(() => {
          this.loadConfigData()
        })
      })
    },
    // Triggered, when filter select action is executed
    select () {
      for (let i = 0; i < this.configData.length; i++) {
        for (let j = 0; j < this.cardsData.length; j++) {
          if (this.cardsData[j].title === this.configData[i].name) {
            if (this.cardsData[j].isVisible === true && this.configData[i].isVisible === '0') {
              this.$uci.set('draggable', this.configData[i]['.name'], 'isVisible', '1')
              break
            } else if (this.cardsData[j].isVisible === false && this.configData[i].isVisible === '1') {
              this.$uci.set('draggable', this.configData[i]['.name'], 'isVisible', '0')
              break
            }
          }
        }
      }
      this.rerender = !this.rerender
      this.$uci.save().then(() => {
        this.$uci.apply().then(() => {
          this.loadConfigData()
        })
      })
    },
    async addNewInterface (name, newInterfacePlacePlus) {
      await this.$uci.load('draggable').then(() => {
        const newInterfacePlace =
          this.configData.length + newInterfacePlacePlus
        const newSid = this.$uci.add('draggable', 'draggable')
        this.$uci.set('draggable', newSid, 'name', name)
        this.$uci.set('draggable', newSid, 'place', newInterfacePlace.toString())
        this.$uci.set('draggable', newSid, 'isVisible', '0')
        this.$uci.save().then(() => {
          this.$uci.apply().then(() => {
            this.loadConfigData()
          })
        })
      })
    },
    deleteExcessiveInterface () {
      const previousName = []
      for (let i = 0; i < this.configData.length; i++) {
        for (let j = 0; j < this.cardsData.length; j++) {
          if (previousName.includes(this.configData[i].name)) {
            this.$uci.del('draggable', this.configData[i]['.name'])
            break
          }
          if (this.cardsData[j].title === this.configData[i].name) {
            previousName.push(this.configData[i].name)
            break
          }
          if (j === this.cardsData.length - 1) {
            const place = this.configData[i].place
            this.$uci.del('draggable', this.configData[i]['.name'])
            for (let z = 0; z < this.configData.length; z++) {
              if (this.configData[z].place > place) {
                this.$uci.set('draggable', this.configData[z]['.name'], 'place', parseInt(this.configData[z].place) - 1)
                this.configData[z].place =
                  parseInt(this.configData[z].place) - 1
                this.configData[z].place.toString()
              }
            }
            break
          }
        }
      }
      this.$uci.save().then(() => {
        this.$uci.apply().then(() => {
          this.loadConfigData()
        })
      })
    }
  },
  created () {
    this.$rpc.call('system', 'syslog').then(({ log }) => {
      this.syslog = log.map((v, i) => {
        v.key = i
        return v
      })
    })
    this.$rpc.call('system', 'dmesg').then(({ log }) => {
      this.dmesg = log.map((v, i) => {
        v.key = i
        return v
      })
    })
    this.$network.load().then(() => {
      this.interfaces = this.$network.getInterfaces()
    })
  }
}
</script>
<style>
.card-header {
  margin: -10px 0px 10px 0px;
  padding: 0px 0px 5px 0px;
  font-size: 1.15em;
  font-weight: 400;
  letter-spacing: -1px;
  border-bottom: 1px solid rgb(223, 223, 223);
  text-transform: uppercase;
}
.card-text-header {
  font-weight: 600;
  letter-spacing: -1px;
  text-transform: uppercase;
}
.card-text-content {
  font-size: 0.85em;
  border-bottom: 1px solid rgb(202, 202, 202);
  padding-bottom: 5px;
  margin-bottom: 5px;
}
#last {
  border: none;
}
.card {
  display: inline-block;
  border: 1px solid rgb(202, 202, 202);
  border-radius: 5px;
  margin: 1%;
  vertical-align: top;
  box-sizing: border-box;
  /* height: 300px; */
  width: 28%;
  height: 302px;
  overflow-x: hidden;
  overflow-y: auto;
  text-align: justify;
}
.card-div {
  height: 300px;
}
</style>

